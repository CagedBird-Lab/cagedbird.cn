---
title: MiceTimer 诞生记：Android 深度睡眠下的“夺命连环扣”
categories:
  - - 技术实践
  - - 系统工程
  - - 系统编程
  - - Android
tags:
  - Rust
  - Android
  - KernelSU
  - MiceTimer
  - 性能工程
date: 2026-01-02 23:00:00
---

### 序言：索然无味的“稳定”

最近我的 FCM 连接时间稳定得让人有些想打呵欠。Time connected 长期维持在 1.5 小时以上，重连几乎是秒级完成。在玩机圈摸爬滚打这么久，我深知这种“索然无味”背后的含义：**基础设施的边界情况被彻底堵死了。**

但在几天前，情况完全不是这样。

那时候，我还在依赖最原始的 Shell 脚本配合 `while true; sleep 1h` 来更新 FCM Hosts。我本以为这很稳，直到我发现：每当我晚上睡觉放下手机，早起一看，Hosts 的同步记录竟然断了五六个小时。

这时候我才意识到，我掉进了 Android 系统那个深不可测的“休眠黑洞”。

<!-- more -->

---

### 1. 消失的计时：为什么 `sleep` 会“偷懒”？

在普通的桌面 Linux 或服务器上，`sleep 3600` 确实意味着一小时后唤醒。但在 Android 这个为了续航而无所不用其极的怪胎身上，逻辑变了。

当屏幕熄灭，手机进入 **Doze 模式（深度睡眠）** 后，CPU 会被完全挂起（Suspend）。此时，普通脚本里的 `sleep` 进程就像是被冻结在时空缝隙里的倒霉蛋：
*   时钟在流逝，但 CPU 停止了处理中断。
*   如果你设置了 `sleep 1h`，而手机休眠了 4 小时，那么当你点亮屏幕时，你的进程才刚刚意识到：“噢，时间到了，我该起来了。”

**这哪是定时任务？这分明是随缘任务。** 对于需要高频同步、对抗网络剧烈波动的 FCM 优化来说，这种计时精度无异于自杀。

---

### 2. 暴力破解：寻找能在“梦中计时”的时钟

作为一名有着严重技术洁癖的工程师，我无法忍受这种随机性。我需要一个能够跨越休眠（Suspend）阶段的计时方案。

在 Linux 内核中，其实提供了一种叫做 `timerfd` 的利器。它比普通的 `sleep` 高级在于：
1.  **文件句柄化**：它是一个文件描述符，可以被 `epoll` 监听，非常适合做异步调度。
2.  **CLOCK_BOOTTIME**：关键点来了！如果使用这个时钟 ID，它会记录系统从启动至今的时间，**包括系统休眠的时间**。

方向确定了：我需要一个常驻后台的守护进程，用 `timerfd` 开启全局调度。

---

### 3. 技术选型：为什么又是 Rust？

有人问我，写个定时器为什么不用 C 或者 Go？
*   **C 语言**：当然可以，但写起来太累。处理配置解析、信号处理和错误恢复时，样板代码多得让人头秃。
*   **Go 语言**：Runtime 太重。对于一个追求极致性能、要在 Android 后台常驻的组件，我无法接受 10MB+ 的内存占用。
*   **Rust**：完美。它对 Linux 系统调用有近乎原生的封装（通过 `nix` crate），同时它的所有权机制保证了我在处理高并发定时器时绝不会翻车。

于是，**MiceTimer** 诞生了。

---

### 4. 核心逻辑：唤醒锁与事件循环

`MiceTimer` 的核心逻辑其实不到 300 行代码，但它精准地踩在了 Android 内核的痛点上。

#### 计时逻辑
通过 `timerfd` 配合 `CLOCK_BOOTTIME` 监听触发事件。当计时器触发时，`epoll` 会立刻唤醒等待中的守护进程。

#### 唤醒保证（WakeLock）
光能“醒来”还不够。如果在执行任务（比如下载 20MB 的 Hosts）时，手机决定再次睡去，任务就会半路夭折。
我直接从内核里“借”来了尚方宝剑：`/sys/power/wake_lock`。

在任务执行前，MiceTimer 会自动申请一个局部唤醒锁：
```rust
// 任务触发时
fs::write("/sys/power/wake_lock", &lock_name)?;
// 执行任务
Command::new("sh").arg("-c").arg(&unit.exec).status();
// 任务结束后自动释放
fs::write("/sys/power/wake_unlock", &lock_name)?;
```
这确保了即使你在半夜三点同步数据，手机屏幕虽然关着，CPU 也会在任务执行期间保持全速运转。

---

### 5. 架构飞跃：从“散兵游勇”到“中心调度”

在开发 MiceTimer 的过程中，我顺便重构了我的整个 **CagedBird-Lab** 生态。

以前，每个 KSU 模块（比如 FCM-Hosts, DNS-Optimizer）都要自己写一套简陋的 `service.sh` 循环。现在，我引入了 **Skeleton（空壳）架构**：
*   **MiceTimer** 是指挥部（Daemon）：负责所有模块的定时任务。
*   **功能模块** 变成了任务包：安装时只需要在 `/data/adb/timers.d/` 丢一个简单的 TOML 配置文件。

```toml
# 示例：fcm-hosts.toml
Description = "每小时同步一次优选 Hosts"
Exec = "/system/bin/fcm-update"
OnUnitActiveSec = "1h"
WakeLock = true
```

这种架构的优美之处在于：**模块不再关心如何计时，只关心如何执行逻辑。** 所有的日志输出、唤醒逻辑、重试间隔，全部由 MiceTimer 统一接管。

---

### 6. 自动化：拒绝手动搬运

既然有了新玩具，发布流程也必须是“洁癖级”的。
我写了一套 GitHub Actions 流程：
1.  自动交叉编译适用于 Android aarch64 的 Rust 二进制。
2.  自动打包成标准 KSU 模块 zip。
3.  自动计算版本号并更新 OTA `update.json`。
4.  自动部署到 GitHub Pages 分发。

现在，我只需要在本地 `git push`，剩下的事情全部交给云端流水线。

---

### 结语

写完 MiceTimer 的那一刻，我长舒了一口气。那些曾经困扰我的“随机同步中断”、“休眠任务丢失”彻底成为了历史。

看着手机日志里精准到秒的触发记录，那种“一切尽在掌握”的掌控感，正是作为一名系统软件工程师最原始的快乐。虽然现在的 FCM 稳定得有些无趣，但这种无趣，正是我追求的极致。

---

**CagedBird-Lab** 正在不断进化。下一篇，我们来聊聊如何用“空壳模块”重构你的 Android 系统组件。

### 项目链接
- **MiceTimer 核心调度器**: [GitHub - CagedBird-Lab/micetimer](https://github.com/CagedBird-Lab/micetimer)
- **FCM Hosts Optimizer 任务包**: [GitHub - CagedBird-Lab/fcm-hosts-ksu](https://github.com/CagedBird-Lab/fcm-hosts-ksu)
- **Infrastructure 主页**: [GitHub - CagedBird-Lab](https://github.com/CagedBird-Lab)

Stay hungry, stay coding.

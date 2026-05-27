---
title: Arch Linux 休眠修复实录：从 resume 设备到 HibernateMode=platform
tags:
  - Arch Linux
  - Hibernate
  - systemd
  - NVIDIA
  - KDE Plasma
  - 电源管理
categories:
  - - 技术实践
  - - 系统工程
  - - 电源管理
date: 2026-05-28 02:45:00
---

这台笔记本的休眠问题拖了一年多。

它不是那种“点一下没反应”的简单故障，而是每往前推进一步，就暴露下一层问题：先是 systemd 直接拒绝休眠，然后是 NVIDIA 恢复路径出错，再后来是镜像明明写进去了，机器却黑屏、键盘亮、风扇转，死活不真正断电。

最后真正稳定工作的状态反而很朴素：

```text
Arch Linux
KDE Plasma Wayland
2560x1600 @ 240Hz
独立 swap 分区
resume=UUID=<swap-uuid>
HibernateMode=platform
NVIDIA 走 systemd/procfs sleep 路径
NVIDIA 不进 initramfs
```

这篇记录一下完整判断过程。重点不是照抄某个配置，而是把“休眠失败”拆成几个阶段来看。

<!-- more -->

## 第一层：resume 设备必须是真正 active 的 swap

最开始的报错很直接：

```text
Call to Hibernate failed: Specified resume device is missing or is not an active swap device
```

这个阶段不用猜 Wayland、NVIDIA、KWin，也不用怀疑刷新率。systemd 已经把话说清楚了：内核要恢复的 resume 设备不存在，或者它不是当前启用的 swap。

最后的修复方式是单独准备一个用于休眠的 swap 分区，并同时让三处配置对上：

```fstab
# /etc/fstab
UUID=<swap-uuid> none swap defaults,pri=20 0 0
```

```text
# /etc/default/grub
resume=UUID=<swap-uuid>
```

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo mkinitcpio -P
```

验证时不要只看配置文件，要看运行态：

```bash
swapon --show
cat /sys/power/resume
cat /sys/power/resume_offset
```

`swapon --show` 里必须能看到这块 swap。`/sys/power/resume` 也应该指向正确的 block device major/minor。普通 swap 分区一般不需要 resume offset，`resume_offset` 为 `0` 是正常的。

这一层修完后，`systemctl hibernate` 至少不再因为 resume 设备直接失败。

## 第二层：NVIDIA 不要同时走两条休眠路径

下一层问题出在 NVIDIA。

这台机器是混合显卡，桌面环境跑 KDE Plasma，机器上有 NVIDIA 独显。NVIDIA 的休眠恢复路径有一个关键要求：如果开启了保存显存分配的机制，就应该走它提供的 procfs/systemd sleep 接口，而不是让内核 suspend notifier 单独处理。

相关配置是：

```conf
# /etc/modprobe.d/nvidia-sleep.conf
options nvidia NVreg_UseKernelSuspendNotifiers=0 NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
```

同时启用 NVIDIA 的 systemd sleep 服务：

```bash
sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service
```

这一步有一个容易忽略的坑：如果把 NVIDIA 模块放进 initramfs，恢复早期阶段可能会先加载 NVIDIA，然后又撞上 systemd/procfs 这套路径，最后变成“谁来负责恢复显存”的问题。

所以最后选择让 NVIDIA 不进 initramfs：

```bash
# /etc/mkinitcpio.conf
MODULES=()
```

改完后重新生成 initramfs：

```bash
sudo mkinitcpio -P
```

这不是说所有 NVIDIA 机器都必须这么配，而是这台机器上，这样能把路径收敛成一条：NVIDIA 由 systemd 的 `nvidia-hibernate.service` / `nvidia-resume.service` 处理。

## 第三层：镜像写入成功，不代表机器完成了休眠

最误导人的阶段在这里。

当 resume 和 NVIDIA 路径都处理完以后，休眠看上去还是失败：屏幕黑掉，键盘灯还亮，风扇还在转，机器没有真正断电。只能长按电源键强制关机。

但强制关机后再开机，内存状态居然能恢复。

这个现象非常关键。它说明问题不是：

```text
没有写入休眠镜像
resume 找不到镜像
恢复路径完全坏掉
```

恰恰相反，镜像已经写入成功，恢复也基本成立。坏掉的是更靠后的阶段：写完休眠镜像之后，机器没有正确进入最终电源状态。

当时的配置是：

```ini
# /etc/systemd/sleep.conf.d/hibernatemode.conf
[Sleep]
HibernateMode=shutdown
```

`shutdown` 模式在这台机器上的表现就是：写完镜像以后黑屏，但机器不断电。于是最终改成：

```ini
# /etc/systemd/sleep.conf.d/hibernatemode.conf
[Sleep]
HibernateMode=platform
```

`platform` 会让平台固件/ACPI 参与进入 S4 的流程。这台机器真正吃的是这条路。

改完以后再看：

```bash
cat /sys/power/disk
```

可以看到当前选中的模式类似：

```text
[platform] shutdown reboot suspend test_resume
```

这一步是整个问题的转折点。之前一直像是“图形黑屏”，但它其实是“休眠镜像写完以后没有正确断电”。

## Wayland 和 240Hz 不是根因

中间也测试过 X11、Wayland、60Hz、240Hz。

这类变量很容易让人误判。因为最终症状是黑屏，而黑屏很容易被归因到：

```text
Wayland
KWin
NVIDIA
高刷新率
显示器唤醒
```

但后来的验证结果很明确：最终稳定成功的组合就是 Plasma Wayland 加 240Hz。也就是说，Wayland 和 240Hz 不是根因。

它们最多是排障过程中的干扰项。真正决定成败的是：

```text
resume 设备是否正确
NVIDIA sleep 路径是否一致
HibernateMode 是否适合这台机器
```

## 一个可复用的判断顺序

这次之后，我觉得排 Linux 休眠问题不能把“休眠失败”当成一个整体。应该拆成几个阶段：

```text
systemd 是否允许进入休眠
        ↓
resume/swap 是否配置正确
        ↓
休眠镜像是否真正写入
        ↓
写完镜像后机器是否进入正确电源状态
        ↓
恢复后内核、驱动、图形栈是否正常
```

对应到现象，大致可以这样判断：

| 现象 | 优先怀疑 |
| --- | --- |
| `systemctl hibernate` 直接报 resume 设备错误 | swap / resume UUID / initramfs |
| 日志里出现 NVIDIA power management/procfs 相关错误 | NVIDIA sleep 路径冲突 |
| 黑屏但键盘亮、风扇转、机器不关 | `/sys/power/disk` / `HibernateMode` |
| 强制断电再开机能恢复内存 | 镜像写入和 resume 基本成立，重点查最终电源状态 |
| 机器恢复了但桌面黑屏，SSH 可进 | 再看图形栈、KWin、NVIDIA resume、刷新率 |

这次真正的关键证据是：

```text
黑屏时机器没有断电
强制关机再开机后内存能恢复
```

它把问题从“显示恢复失败”重新定位到了“休眠后的电源状态没切对”。

## 最终配置摘要

这台机器最终留下的休眠相关配置大概是这样：

```ini
# /etc/systemd/sleep.conf.d/hibernatemode.conf
[Sleep]
HibernateMode=platform
```

```conf
# /etc/modprobe.d/nvidia-sleep.conf
options nvidia NVreg_UseKernelSuspendNotifiers=0 NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
```

```bash
# /etc/mkinitcpio.conf
MODULES=()
```

```text
# /etc/default/grub
resume=UUID=<swap-uuid>
```

```fstab
# /etc/fstab
UUID=<swap-uuid> none swap defaults,pri=20 0 0
```

再配合 NVIDIA 的 systemd sleep 服务：

```bash
systemctl is-enabled nvidia-suspend.service
systemctl is-enabled nvidia-hibernate.service
systemctl is-enabled nvidia-resume.service
```

验证成功时，日志里能看到类似链路：

```text
PM: hibernation: hibernation entry
System returned from sleep operation 'hibernate'
PM: hibernation: hibernation exit
nvidia-resume.service: Deactivated successfully
```

## 结尾：不要被“黑屏”两个字骗了

这次折腾最久的地方，就是“黑屏”这个症状太宽泛。

显示器黑了，可能是桌面没恢复；可能是显卡没恢复；也可能是机器压根没进入正确的休眠电源状态。它们看起来都叫黑屏，但排查方向完全不同。

最后这个问题能解开，是因为现象被拆细了：

```text
不是单纯黑屏
而是写完休眠镜像后黑屏、键盘亮、风扇不停、机器不断电
```

这句话比任何玄学参数都重要。

对于这台机器，答案最终落在了 `HibernateMode=platform`。但更通用的经验是：先判断休眠失败发生在哪个阶段，再动配置。Linux 休眠不是一个开关，而是一条链路。链路里每一段都可能坏，症状却可能都长得像“黑屏”。

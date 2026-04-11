---
title: 香橙派5 Pro Wi‑Fi修复实录：从 brcmfmac 掉频到 BCMDHD 稳定 5GHz
tags:
  - 香橙派5Pro
  - RK3588S
  - AP6256
  - BCM43456
  - BCMDHD
  - brcmfmac
  - WiFi
  - Ubuntu-Rockchip
categories:
  - - 技术实践
  - - 系统工程
  - - 系统编程
  - - 嵌入式开发
date: 2026-04-12 02:20:00
---

这次记录一场非常典型、也非常折磨人的开发板无线网络排障：**Orange Pi 5 Pro 在 `Joshua-Riek/ubuntu-rockchip` 系统上，开机后短暂能连 5GHz，但几分钟内就会掉回 WORLD / 只剩 2.4GHz，最终断网。**

最后真正的解法，不是继续在路由器、DFS、DNS、翻墙环境上兜圈子，而是把这块板子的 Wi‑Fi 驱动路径从 `brcmfmac` 切到 **`bcmdhd_sdio`**。

<!-- more -->

## 一、问题现象：看起来像路由问题，实际上不是

现场症状一开始很迷惑：

- 开机后能看到并连上 `CMCC-201-5G`
- 过几十秒到几分钟后，5GHz 消失
- `wlan0` 断开，扫描里只剩 2.4GHz
- 有时伴随 regdom 掉回 WORLD

这类现象很容易让人先怀疑：

- 路由器 160MHz / DFS 有问题
- 光猫 DNS 太烂
- 主机共享网络或代理影响了板子
- 5GHz 信号本来就不稳定

这些猜测不能说完全没意义，但都不是根因。

## 二、关键证据：问题不在上层配置，而在驱动路径

### 2.1 当前系统实际走的是 `brcmfmac`

故障系统环境：

- 发行版：`Ubuntu 24.04.1 LTS`
- 内核：`6.1.0-1025-rockchip`
- 项目：`Joshua-Riek/ubuntu-rockchip`

启动日志里能看到：

```text
brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac43456-sdio for chip BCM4345/9
```

也就是说，当前这块板子的 AP6256 / BCM43456，实际走的是 **`brcmfmac` + `brcmfmac43456-sdio`** 这条路径。

而且在更差的时候，会直接在启动阶段报：

```text
ieee80211 phy0: brcmf_bus_started: failed: -52
ieee80211 phy0: brcmf_attach: dongle is not responding: err=-52
brcmfmac: brcmf_sdio_firmware_callback: brcmf_attach failed
```

### 2.2 官方 Orange Pi 镜像的路线并不一样

我对比了 Orange Pi 官方 `jammy + linux6.1.43` 镜像，发现一个很重要的差异：

- 官方路线启用的是 **BCMDHD SDIO**
- 而当前这套 `ubuntu-rockchip` 在这块板子上跑的是 `brcmfmac`

这一点不是猜测，而是来自镜像内容和内核配置对比。

### 2.3 固件替换不是解法

中间也验证过几个常见岔路：

- `brcmfmac43456-sdio.bin`：当前系统 vs 官方镜像，**相同**
- `brcmfmac43456-sdio.txt`：当前系统 vs 官方镜像，**相同**
- 去掉 `clm_blob`：**不能根治**

所以这个问题不是“你 bin / txt 放错了”这么简单。

## 三、排障中途还顺手踩了一个大坑：/lib 被我弄坏了

为了尝试“无刷机切到 vendor 6.1.43 路径”，我一度把 Orange Pi 的根分区弄成了非标准 merged-/usr 布局：

- `/lib` 被错误替换成了真实目录
- 新 SSH 会话一进来直接报：

```text
/bin/bash: No such file or directory
```

最后的救援方法反而很实用：

1. 让开发板进入 **UMS 模式**，把 eMMC 暴露给主机
2. 在主机侧直接挂载根分区
3. 把 `/lib` 恢复成：

```text
/lib -> usr/lib
```

4. 顺手把误放进去的 6.1.43 资源整理回正确位置

这一步虽然是“意外支线”，但也验证了一个经验：

> 对 RK3588 这类板子，**UMS 是非常硬核、非常有效的救援手段**。

## 四、真正的修复：切到 `bcmdhd_sdio`

最终有效方案非常直接：

### 4.1 安装 BCMDHD SDIO DKMS

```bash
sudo apt-get update
sudo apt-get install -y dkms bcmdhd-sdio-dkms
```

### 4.2 为模块指定固件 / NVRAM / 配置路径

```conf
# /etc/modprobe.d/99-bcmdhd-sdio.conf
options bcmdhd_sdio firmware_path=/lib/firmware/fw_bcm43456c5_ag.bin nvram_path=/lib/firmware/nvram_ap6256.txt config_path=/lib/firmware/config.txt op_mode=0 iface_name=wlan0
```

### 4.3 禁用 `brcmfmac`

```conf
# /etc/modprobe.d/99-blacklist-brcmfmac-ap6256.conf
blacklist brcmfmac
blacklist brcmutil
```

### 4.4 关机 / 重启前，先把 AP6256 拉下电

这是这次修复里最像“补足时序缺口”的一刀。

```ini
[Unit]
Description=Power down AP6256 SDIO Wi-Fi before shutdown/reboot
DefaultDependencies=no
Before=shutdown.target reboot.target poweroff.target halt.target kexec.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'modprobe -r bcmdhd_sdio dhd_static_buf_sdio >/dev/null 2>&1 || true; echo 0 > /sys/class/rkwifi/wifi_power >/dev/null 2>&1 || true; echo 0 > /sys/class/rkwifi/wifi_set_carddetect >/dev/null 2>&1 || true'

[Install]
WantedBy=reboot.target
WantedBy=poweroff.target
WantedBy=halt.target
WantedBy=kexec.target
```

同时，我还禁用了原本只针对 `brcmfmac` 的：

```text
ap6256-reboot.service
```

## 五、最终结果：5GHz 稳定回来，而且不是“碰巧”

修完之后，现场验证过两类场景：

### 5.1 冷启动

- `wlan0` 自动连回 `CMCC-201-5G`
- 工作在 `5180 MHz / channel 36`
- 通过了之前的故障窗口（原来几十秒到一两分钟就掉）

### 5.2 软件重启

这是更关键的回归测试。

在保留关机前下电清理逻辑后，软件重启同样恢复正常：

```text
wl_android_wifi_on : Success
Link UP with 48:81:d4:89:db:3f
```

并且连续观测超过旧故障窗口后，依然保持：

- `GENERAL.STATE:100 (connected)`
- `GENERAL.CONNECTION:CMCC-201-5G`
- `freq: 5180.0`
- `signal: -54 dBm`

对外连通性也验证过：

```bash
ping -I wlan0 223.5.5.5
```

结果稳定返回，说明这不是“假连接”。

## 六、最终判断：根因不是 DFS、不是 DNS、不是翻墙环境

这次探索最大的收获，其实不是“把 Wi‑Fi 修好了”，而是把问题讲清楚了：

- **路由器设置会影响体验，但不是根因**
- **AP6256 / BCM43456 在这块板子上的 `brcmfmac` 路径不稳定，才是主因**
- **BCMDHD SDIO 才是更适合这台 Orange Pi 5 Pro 的驱动路径**

换句话说，这不是简单的“家庭网络玄学”，而是**板级驱动选型问题**。

## 七、顺手向上游提了 PR

既然这不是单机玄学，就值得向上游反馈。

我最终给 `Joshua-Riek/ubuntu-rockchip` 提了一个 PR：

- **Use BCMDHD SDIO for Orange Pi 5 Pro AP6256 Wi-Fi**
- PR: <https://github.com/Joshua-Riek/ubuntu-rockchip/pull/1343>

这个 PR 的核心思路也很朴素：

1. 为 Orange Pi 5 Pro 安装 `bcmdhd-sdio-dkms`
2. 对这块板子黑掉 `brcmfmac`
3. 明确 BCMDHD 的 firmware / nvram / config 路径
4. 用“关机前下电”替换原本只对 `brcmfmac` 有意义的 reboot workaround

## 八、结语

很多板子问题最烦的地方，不是它不能修，而是它会让你先去怀疑一堆不相干的东西：

- 路由器是不是垃圾
- 运营商是不是抽风
- DFS 是不是在搞事
- 代理共享是不是把它带歪了

但这次最终证明：

> **真正值得怀疑的，是那条默认看起来“理所当然”的驱动路径。**

当你把 `brcmfmac` 这层雾拨开，看到 BCMDHD 这条更贴板子的路径之后，问题反而变得很朴素：

- 正确的驱动
- 正确的固件入口
- 正确的关机时序

然后，5GHz 就老老实实回来了。

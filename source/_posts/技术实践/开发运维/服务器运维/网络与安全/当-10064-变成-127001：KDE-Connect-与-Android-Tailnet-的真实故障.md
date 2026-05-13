---
title: 当 100.64 变成 127.0.0.1：KDE Connect 与 Android Tailnet 的真实故障
tags:
  - Android
  - KDE Connect
  - sing-box
  - Tailscale
  - Headscale
  - Tailnet
  - 网络排障
  - 开源贡献
categories:
  - - 技术实践
  - - 开发运维
  - - 服务器运维
  - - 网络与安全
date: 2026-05-14 01:20:00
---

我最开始只是想做一件很朴素的事情：让手机和电脑稳定互联。

不是那种“在同一个 Wi-Fi 下面互相发现”的互联，而是无论手机在移动数据、电脑在家里、笔记本在公司，所有设备都能通过一个统一的 tailnet 相互访问。最好 Android 上也不要同时跑 Tailscale、Clash、KDE Connect、各种代理和后台服务。我希望 `sing-box` 既负责代理，也负责 tailnet，而 KDE Connect 只要相信这个 tailnet 就行。

听起来很合理。

真正做起来以后，我才发现，这里面藏着几个非常典型的现代网络软件陷阱：UI 状态不等于数据面状态，`ping` 通不等于 TCP 走对了路，抓包看到 `100.64.x.x` 不等于应用层也能看到 `100.64.x.x`。最后这个问题甚至一路挖到了 KDE Connect Android 的源码，并被整理成了 KDE Bugzilla 和 KDE Invent Merge Request。

<!-- more -->

## 目标：用一个 sing-box 接管代理和 tailnet

我的目标不是“多装几个工具凑合能用”。我真正想要的形态是：

- Linux / macOS 上用自己维护的 `sing-box` fork；
- Android 上用自己的 SFA / sing-box 构建；
- `sing-box` 原生接入 Headscale / Tailscale-compatible tailnet；
- 同时继续承担日常代理和 TUN 接管；
- tailnet 内的 SSH、KDE Connect 等服务都走统一路径。

也就是说，我希望 Android 手机即使只开移动数据，也能通过 tailnet 访问电脑：

```bash
ssh cagedbird@100.64.0.1
```

只要 SSH 能通，KDE Connect 理论上也应该能通。毕竟它本质也是网络连接，只不过多了一套 discovery、identity packet 和 TLS handshake。

现实当然没有这么简单。

## 第一幕：SFA 说 NeedsLogin，但日志说不是

Android 上最开始的迷惑点是 SFA 的 Tailscale endpoint 状态。界面可能显示：

```text
NeedsLogin
```

如果只看 UI，很容易认为 tailnet 根本没登录成功。但日志里又能看到类似：

```text
endpoint/tailscale[tailnet]: tsnet running state path .../tailscaled.state
endpoint/tailscale[tailnet]: tsnet starting with hostname "mice-android"
endpoint/tailscale[tailnet]: LocalBackend state is NeedsLogin; running StartLoginInteractive...
endpoint/tailscale[tailnet]: LocalBackend state is NeedsLogin; force_login enabled; running StartLoginInteractive...
```

更关键的是，当 Termux 里发起 SSH 时，SFA 日志已经能捕获业务流量：

```text
endpoint/tailscale[tailnet]: outbound connection to 100.64.0.1:22
```

这条日志非常重要。它说明 Termux 流量已经进入 Android VPN/TUN，并且 `sing-box` 已经把目标 `100.64.0.1:22` 路由到了 `tailnet` endpoint。

所以这时不能再简单地相信 UI 上的 `NeedsLogin`。UI 文案是一层状态抽象，业务数据面是另一层事实。

这次真正卡住数据面的点，是 Headscale / DERP 侧的证书校验。服务器日志里能看到类似：

```text
TLS handshake error ... remote error: tls: bad certificate
```

最后通过给 DERP map 使用证书指纹固定解决。这个阶段给我的第一个教训是：

> 控制面看起来 online，不等于数据面一定通；UI 显示 NeedsLogin，也不等于业务流量一定没进 tailnet。

网络排障必须沿着数据包走，而不是沿着界面文案走。

## 第二幕：ping 通了，SSH 却死了

Android tailnet 路径打通以后，我又遇到另一个更像玄学的问题：Mac / Linux 节点在 Headscale 里明明在线，`ping` 也通，但 SSH 就是超时。

现象大概是：

```bash
ping 100.64.0.3
# OK

ssh cagedbird@100.64.0.3
# timeout

nc -vz 100.64.0.3 22
# timeout
```

第一反应很容易怀疑：

- Mac 上 sshd 没起来；
- Headscale ACL 有问题；
- DERP 不通；
- 防火墙挡了；
- Tailscale 打洞失败；
- MagicDNS 配错了。

但后来抓包把方向彻底扭转了。

在 Linux 笔记本上抓 `tailscale0`：

```bash
sudo timeout 10 tcpdump -ni tailscale0 'host 100.64.0.3 and tcp and (port 22 or port 34152)' -c 8
```

结果是：

```text
0 packets captured
```

这句话比任何猜测都硬。它说明 TCP/22 根本没有进 Tailscale interface。

真正根因是 Linux 笔记本上的 `sing-box` TUN 配置漏掉了 Tailscale / Headscale 地址段，例如：

```text
100.64.0.0/10
fd7a:115c:a1e0::/48
```

于是普通 TCP 流量被 `sing-box` TUN 劫走，没有进入 `tailscale0`。这就造成了非常迷惑的状态：

- ICMP / TSMP 看起来正常；
- Headscale 控制面在线；
- 但业务 TCP 不通。

第二个教训是：

> `ping` 通不是结论，业务 TCP 走没走进正确 interface 才是结论。

## 第三幕：KDE Connect 不相信我的 tailnet

终于，手机能通过 tailnet SSH 到电脑了。

于是我开始测试 KDE Connect。

我的预期很简单：既然手机已经能访问电脑的 `100.64.0.1`，那在 KDE Connect Android 里通过 “Add devices by IP” 手动添加 `100.64.0.1`，理论上就应该可以配对。

但它不行。

这时最容易被误导到几个方向。

第一个方向是：KDE Connect 依赖 UDP broadcast，而 Tailscale 不支持 broadcast / multicast。

这当然是真的，但不是这次的根因。因为我已经使用了 “Add devices by IP”，目的就是绕过 broadcast discovery。

第二个方向是：KDE Connect Android 1.35.x 曾经有一个和 Tailscale 相关的回归，它把 `100.64.0.0/10` 这类 CGNAT 地址当成非本地地址拒绝。这个问题已经由 KDE 上游的 MR 修复过。

相关链接：

- KDE Bug 515707：<https://bugs.kde.org/show_bug.cgi?id=515707>
- KDE MR 625：<https://invent.kde.org/network/kdeconnect-android/-/merge_requests/625>

但我的场景仍然失败。所以我怀疑这不是同一个问题。

第三个方向是 Android 后台冻结。某些厂商系统确实会冻结后台 app，但我当时在前台疯狂刷新也失败，所以不能把锅全甩给后台管理。

这时只剩下一条路：抓包和读源码。

## 第四幕：包到了，但 Java 看见的不是它

KDE Connect 的默认端口是 `1716`。我先确认 desktop 到 Android 的 TCP 流量确实到了手机 tailnet IP。

抓包能看到类似：

```text
100.64.0.1 -> 100.64.0.3:1716 TCP SYN
100.64.0.3 -> 100.64.0.1 SYN/ACK
100.64.0.1 -> 100.64.0.3 payload
100.64.0.3 ACK
```

这说明：

- 不是 desktop 没发；
- 不是 tailnet 路由没通；
- 不是 TCP/1716 没到手机。

然后我开始在 KDE Connect Android 的 `LanLinkProvider` 里加日志。真正的关键证据出现了：

```text
TCP listener accepted raw socket from /127.0.0.1:xxxxx to /127.0.0.1:1716
TCP connection accepted ... private:false explicitPeer:false
Discarding TCP packet from a non-local IP /127.0.0.1
```

网络层明明是：

```text
100.64.0.1 -> 100.64.0.3:1716
```

但 Java `ServerSocket.accept()` 看到的是：

```text
127.0.0.1 -> 127.0.0.1:1716
```

我又用手工 probe 复现了一次：

```bash
printf 'manual-probe-from-laptop\n' | timeout 5 nc -v 100.64.0.3 1716
adb shell 'ss -tan | grep 1716 || true'
```

Android 侧看到类似：

```text
ESTAB 127.0.0.1:47272 -> 127.0.0.1:1716
ESTAB [::ffff:127.0.0.1]:1716 -> [::ffff:127.0.0.1]:47272
```

这时根因终于清楚了。

`sing-box` / userspace tailnet 在 Android 上把入站 TCP 交给 app 时，app socket 层看到的是 loopback。KDE Connect Android 原逻辑假设 socket source address 就是 peer 的真实地址，于是它做了这样的判断：

```text
127.0.0.1 不是 private LAN
127.0.0.1 也不是用户配置的 100.64.0.1
=> reject before identity packet
```

问题不再是“网络通不通”，而是：

```text
用户配置的 peer identity: 100.64.0.1
网络层真实 peer: 100.64.0.1
应用层观测 peer: 127.0.0.1
```

KDE Connect 原来的 trust gate 假设这三个是一致的。Android userspace tailnet 打破了这个假设。

## 第五幕：补丁必须窄，不能乱信任 loopback

最粗暴的办法当然是：让 KDE Connect 接受 loopback。

但这太危险，也太不优雅。`127.0.0.1` 不能被全局当成可信 peer。真正合理的边界应该是：只有当用户已经配置了 custom devices，也就是已经显式 opt-in 某个 peer 时，才把 loopback inbound 当成 userspace tailnet/proxy 的入口。

我本地验证成功的核心逻辑是：

```java
boolean privateAddress = isPrivateAddress(address);
boolean explicitPeer = isConfiguredCustomDevice(address);
boolean loopbackProxyPeer = isLoopbackCustomDeviceProxy(address);
if (!privateAddress && !explicitPeer && !loopbackProxyPeer) {
    Log.i("LanLinkProvider", "Discarding TCP packet from a non-local IP " + address);
    return;
}

final Pair<NetworkPacket, Boolean> pair = unserializeReceivedIdentityPacket(
        message,
        explicitPeer || loopbackProxyPeer
);
```

辅助函数：

```java
private boolean isLoopbackCustomDeviceProxy(InetAddress address) {
    return address.isLoopbackAddress()
            && !CustomDevicesActivity.getCustomDeviceList(context).isEmpty();
}
```

这不是全局信任 loopback。它只是承认一个现实：

> 在 Android userspace VPN / tailnet 环境里，用户显式配置的远端 peer 可能以 loopback 的形式交付到应用层。

后面 KDE Connect 仍然会继续处理 identity packet 和 TLS handshake。这个补丁只是不在最前面的地址 gate 把连接直接杀掉。

## 第六幕：复现条件为什么这么苛刻

这个 bug 非常容易被 “works for me” 掩盖。

如果手机和电脑在同一个物理 LAN，比如同一个 Wi-Fi，KDE Connect 会走普通 LAN discovery / direct path。peer 在 app 里仍然表现为 `192.168.x.x` 或其他 private address，原逻辑直接通过。

所以同 LAN 测试成功不能证明这个 bug 不存在。

真正的复现条件是：

```text
Android KDE Connect
+ userspace tailnet/VPN inbound path
+ desktop and phone not on same physical LAN
+ manual tailnet IP peer
+ app-visible socket source rewritten to loopback
```

这也是为什么我后来专门在 KDE Invent MR 里补了一条评论：同 Wi-Fi 测试会掩盖问题，必须跨不同物理网络走 userspace tailnet 才能看到这个 failure mode。

## 第七幕：从本地补丁到 KDE Invent

本地修通以后，我没有停在“自己能用就行”。因为这个问题很可能会影响其他使用 Android userspace VPN / tailnet 的 KDE Connect 用户。

于是我先开了 KDE Bugzilla：

- KDE Bug 520110：<https://bugs.kde.org/show_bug.cgi?id=520110>

然后整理了一个干净的上游分支，去掉临时 debug 包名修改和大量诊断日志，只保留最小补丁。

一开始我还尝试开 GitHub PR：

- GitHub mirror PR：<https://github.com/KDE/kdeconnect-android/pull/34>

结果它很快被自动关闭。原因是 KDE 的 GitHub 仓库只是 mirror，真正的贡献入口是 KDE Invent。

于是又走了一遍 KDE Invent：注册账号、添加 SSH key、创建 fork、处理 GitLab API、推分支、开 Merge Request。中间还遇到一个 KDE pre-receive audit：提交 author 不能只是 `Mice`，必须是完整名字。于是我又专门为 Invent MR 分支重写了 author / committer。

最后正式 MR 是：

- KDE Invent MR !650：<https://invent.kde.org/network/kdeconnect-android/-/merge_requests/650>

这条链路最终变成：

```text
Bugzilla 520110
  -> KDE Invent MR !650
  -> public fork branch
  -> clean author commits
  -> build verification
```

本地验证至少包括：

```bash
./gradlew compileDebugJavaWithJavac
```

以及真实设备上的配对成功。

## 小插曲：工具也需要安全带

这几天还遇到过另一个和 `sing-box` 相关的小插曲。

一次配置迁移时，模板错误地把 Headscale CA PEM 渲染成了未加引号的 JSONC 裸值：

```jsonc
"certificate": -----BEGIN CERTIFICATE-----\n...
```

运行 `sing-box format/check` 时，它没有快速报错，而是出现了病态内存增长，最后被 OOM kill。修复当然是把 PEM 正确渲染成 JSON string，但这个事故也提醒我：配置验证这种工具最好套一层护栏。

例如：

```bash
systemd-run --user --scope \
  -p MemoryMax=2G \
  -p MemorySwapMax=0 \
  ...
```

排障时不能假设工具永远优雅失败。坏输入、坏 parser、坏模板叠在一起时，工具也可能把整台机器拖下水。

## 这次真正学到的东西

这次故事里最值钱的不是那几行 Java 代码，而是几个排障原则。

第一，**不要过度相信 UI 状态**。SFA 显示 `NeedsLogin`，不代表业务数据面完全不可用。

第二，**不要把 ping 当成业务连通性证明**。ICMP / TSMP 通，不代表 TCP/22 或 TCP/1716 走进了正确的 interface。

第三，**抓包要抓在正确位置**。只看控制面、只看服务日志、只看 CLI 输出，都可能被局部事实误导。

第四，**应用层看到的地址不一定等于网络层真实地址**。这在 userspace VPN、代理、TUN、透明转发里尤其常见。

第五，**开源贡献最重要的是把玄学变成事实**。维护者未必有你的真实网络环境，但你可以把这个环境里的 failure mode 变成 issue、证据链和最小补丁。

## 结语：把自己的现场经验变成公共资产

这个问题如果只描述成“我的 KDE Connect 不能通过 Tailscale 用”，它很容易被归因到 broadcast、配置、Android 后台、旧版本回归或防火墙。

但真正的根因是：

```text
100.64.x.x 在网络层存在，127.0.0.1 在应用层出现。
```

这中间差了一层 userspace tailnet transport。

我喜欢这类问题，因为它们一开始看起来像玄学，最后却能被拆成非常具体的事实：哪个 interface 没包，哪个 socket 地址不对，哪一行 trust gate 提前 return。

这也是我理解的开源精神：不是每个人都要写大功能，但如果你真实遇到了一个复杂环境里的问题，并愿意多走一步，把它整理成上游能 review 的形式，那这个经验就不再只属于你一个人。

它会变成公共资产。

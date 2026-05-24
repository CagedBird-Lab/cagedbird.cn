---
title: 透明代理下，給 DNS 與探針留一條真網之路
date: 2026-05-24 21:35:00
categories:
  - 网络
tags:
  - TUN
  - sing-box
  - netns
  - DNS
  - Linux
---

昔年玩軟路由，常有一惑：若 Clash TUN 接管全家流量，smartdns 又當如何優選？

DNS 欲優選，須見真網；TUN 欲接管，則要攔流。二者若不分道，smartdns 所見，已非底層網路，而是代理之後的影子。以影測形，必生玄學。

今日終於想明白：問題不在 TUN 與 smartdns 不可共存，而在控制面與數據面混作一處。

## 一、TUN 不是原罪

TUN 所為，不過截流：

```text
app packet -> policy route / nft -> tun0 -> sing-box -> direct/proxy
```

若規則寫得好，國內地址可排除，境外地址可接管，特定域名可直連。這是數據面。

但測量、DNS、探針、優選，另屬控制面。控制面若也入 TUN，便會自噬：

```text
用被代理污染之路，測是否該代理
```

此事一亂，症狀便成：

```text
ping 看似不通
curl 時好時壞
DNS 優選不可信
節點測速似真似假
```

非工具不行，乃路未分。

## 二、要訣在 mark 與路

Linux 之妙，在於 packet 可被標記，可入異表，可行異路。

例如 sing-box 常有 bypass mark：

```text
fwmark 0x2024 -> goto nop
```

若一股流量帶此 mark，便不再入 TUN。此即免代理之符。

故有三法：

```text
mark 其流，使其免 TUN
policy route 分其表，使其走真出口
netns 別其境，使其生於淨土
```

第一法輕，第二法正，第三法最清。

## 三、淨土之形

我後來取第三法：造一個 Linux network namespace。

```text
netflaplab netns
  veth0: 10.250.250.2/30
      |
host veth-netflap: 10.250.250.1/30
      |
NAT + mark 0x2024
      |
wlp4s0 真 Wi-Fi 出口
```

其 nft 規則大意如下：

```text
iifname "veth-netflap" meta mark set 0x2024
ip saddr 10.250.250.0/30 oifname "wlp4s0" masquerade
```

於是，在淨土中執行：

```bash
sudo net-flap-lab run curl -4 --noproxy '*' https://timicc.com
sudo net-flap-lab run ping -c 4 223.5.5.5
sudo net-flap-lab run tracepath -4 172.67.75.65
```

其流向不經 host app OUTPUT，不入 sing-box TUN，不吃代理環境。它只走底層 Wi-Fi 與運營商出口。

此時再測，方可知真相：

```text
host probe 異常，lab probe 正常 -> TUN / 本機策略污染
host probe 異常，lab probe 亦異常 -> 底層網路 / 運營商出口真壞
```

## 四、smartdns 亦同理

若置於 OpenWrt，法亦相同。

smartdns 不應被全局代理吞掉。它該見真網，方能作真優選。

可用：

```text
iptables/nft mark
ip rule + ip route table
專門 netns
或獨立容器網路命名空間
```

核心不是某插件，而是這幾問：

```text
packet 從何來？
經何 hook？
帶何 mark？
查何 route table？
出何 interface？
```

能答此五問，TUN 與 smartdns 便不再相剋。

## 五、為何不直接用容器

Docker、Podman、systemd-nspawn、bubblewrap、unshare 皆可借力，然皆非終局。

因真正要定者，是本機之路：

```text
出口網卡是 wlp4s0
bypass mark 是 0x2024
NAT 網段是 10.250.250.0/30
目標是避開 sing-box TUN
```

通用工具不知此機之法。故只需薄封裝：

```text
ip netns + veth + nft mark + NAT
```

小而明，可查，可刪，可復現。

## 六、今日所悟

昔日在恩山翻帖，常見眾人言：裝此插件，套彼規則，改某 DNS，玄學可解。其實多半未觸其本。

真本不玄：

```text
欲測真網，先造真網之路。
欲免代理污染，須使測量流不入代理。
欲令 DNS 優選可信，須使 DNS 見到未被代理折疊的世界。
```

TUN 可全局接管，smartdns 亦可真網優選。二者可共存，只須分道。

道分，則不亂。

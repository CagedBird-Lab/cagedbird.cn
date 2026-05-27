---
title: 宁波移动 HG5143F 光猫超密获取记录
date: 2026-05-27 21:05:00
categories:
  - 网络
tags:
  - 光猫
  - 中国移动
  - HG5143F
---

记录一次宁波移动 HG5143F 光猫超密获取过程。

设备型号：

```text
宁波移动 HG5143F
```

参考方法：

```text
https://www.bilibili.com/opus/748434378355376185
```

<!-- more -->

## 1. 找到光猫网关

先看本机 ARP 表，确认当前网关地址：

```text
❯ arp -a
? (192.168.1.3) at 48:81:d4:89:db:3c [ether] on wlp4s0
_gateway (192.168.1.1) at 24:b7:da:3b:24:e0 [ether] on wlp4s0
```

这里光猫网关是：

```text
192.168.1.1
```

## 2. Telnet 连接光猫

直接 telnet 到网关：

```text
❯ telnet 192.168.1.1
```

进入后是一个 `#` shell：

```text
#
```

中间尝试过 `su`，但密码不对：

```text
#su
Password:
su: incorrect password
#exit
Connection closed by foreign host.
```

重新 telnet 进去后，不需要 `su`，直接执行 `cfg_cmd` 即可。

## 3. 读取超管用户名

```text
#cfg_cmd get InternetGatewayDevice.DeviceInfo.X_CMCC_TeleComAccount.Username
FHAPI_INIT Error!
argc = 3
argv[0] = cfg_cmd
argv[1] = get
argv[2] = InternetGatewayDevice.DeviceInfo.X_CMCC_TeleComAccount.Username
get success!value=CMCCAdmin
```

得到用户名：

```text
CMCCAdmin
```

## 4. 读取超管密码

```text
#cfg_cmd get InternetGatewayDevice.DeviceInfo.X_CMCC_TeleComAccount.Password
FHAPI_INIT Error!
argc = 3
argv[0] = cfg_cmd
argv[1] = get
argv[2] = InternetGatewayDevice.DeviceInfo.X_CMCC_TeleComAccount.Password
get success!value=<device-specific-password>
```

得到当前设备的随机超管密码。

## 5. 字段汇总

```text
用户名字段：InternetGatewayDevice.DeviceInfo.X_CMCC_TeleComAccount.Username
密码字段：  InternetGatewayDevice.DeviceInfo.X_CMCC_TeleComAccount.Password
用户名：    CMCCAdmin
密码：      <device-specific-password>
```

说明：这里没有贴出我这台设备的真实密码明文，只保留字段和命令。

## 6. 自动化脚本

我把上述流程做成了一个小脚本：

```text
https://github.com/CagedBird-Lab/hg5143f-superpass
```

脚本会自动完成：

```text
读取网关 MAC -> 开启 telnet -> 用 admin/Fh@<MAC 后 6 位> 登录 -> 执行 cfg_cmd 读取超管账号密码
```

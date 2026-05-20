---
title: Arch Linux 上一次 Steam 原生游戏中文字体空白排障小记
tags:
  - Arch Linux
  - Steam
  - Unity
  - 字体
  - fontconfig
  - 游戏排障
categories:
  - - 技术实践
  - - 开发运维
  - - 开发环境
date: 2026-05-20 14:45:00
---

这次遇到的是一个很离谱但又很典型的问题：一个明明有 Linux 原生版本的 Steam 游戏，在 Arch Linux 上切到中文后，主菜单按钮框还在，文字却空了；反而用 Steam Legacy Runtime / Wine 跑 Windows 版时中文正常。

最后修好以后，结论不是“Linux 没装中文字体”，而是更具体的一类坑：**老 Unity Linux 原生版在 Steam Runtime 里直接读了 DejaVu 字体，绕开了宿主系统的 CJK fallback**。

<!-- more -->

## 先排除：系统字体本身没坏

第一步没有急着乱改 Steam，而是先看宿主系统的字体链：

```bash
fc-match 'sans:lang=zh-cn'
fc-match 'serif:lang=zh-cn'
fc-match 'monospace:lang=zh-cn'
```

结果都能正常命中 Noto CJK / 思源系字体。也就是说，桌面环境、浏览器、Qt/GTK 常规应用依赖的 fontconfig 逻辑大体是通的。

真正可疑的点变成了 Steam Runtime 和游戏进程本身。

## 关键证据：游戏进程只在读 DejaVu

看最终游戏进程的文件句柄时，能看到它实际打开的是 Steam Runtime 里的 DejaVu：

```text
/proc/<pid>/fd/... -> /usr/share/fonts/truetype/dejavu/DejaVuSans.ttf
```

这就解释通了：

- 英文 UI 正常，因为 DejaVu 有拉丁字符；
- 中文 UI 为空，因为 DejaVu 没有中文字形；
- Windows 版 / Wine 正常，因为走的是另一套 Windows 字体 fallback；
- 宿主系统明明有 Noto/WQY/微软雅黑，也没用上，因为老 Unity 原生版没有走完整的宿主 fontconfig fallback。

所以这不是“原生 Linux 一定更好”的问题，而是“原生版维护质量”和“运行时隔离”叠加出来的问题。

## 最小修复：只对这个游戏重定向字体

我没有改全局 fontconfig 优先级，而是保留一个很窄的 per-game 修复：

1. 安装老 Unity / Android fallback 常见的字体包：

```bash
sudo pacman -S --needed ttf-droid
```

2. 做一个只给该游戏使用的 `LD_PRELOAD` 小补丁，拦截它打开 DejaVu 的行为：

```text
/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf
/usr/share/fonts/TTF/DejaVuSans.ttf
```

并重定向到 Steam Runtime 里可见的宿主字体：

```text
/run/host/fonts/droid/DroidSansFallbackFull.ttf
```

3. Steam 启动项使用无空格路径：

```bash
LD_PRELOAD=/home/cagedbird/.local/share/Steam/steamapps/common/sor-fontfix/libfont_redirect.so %command%
```

这里有个小坑：一开始补丁目录放在游戏目录 `Streets of Rogue` 下面，路径中有空格，被 Steam / Pressure Vessel 拆坏了。后来挪到 `steamapps/common/sor-fontfix/`，启动项才稳定生效。

## 验证方式

这类问题不要只靠“感觉字体变了”。这次真正有用的验证是三层：

```text
最终游戏进程环境里有 LD_PRELOAD
        ↓
/proc/<pid>/maps 里加载了 libfont_redirect.so
        ↓
/proc/<pid>/fd 里的字体句柄从 DejaVu 变成 DroidSansFallbackFull.ttf
```

最后再由实际画面确认中文 UI 恢复。

## 这次记住的经验

1. Linux 原生游戏不代表本地化路径一定比 Windows/Proton 版更好。
2. Steam Runtime 可能把宿主字体隔离起来，或者让老游戏更偏向 runtime 自带字体。
3. `fc-match` 只能证明宿主系统字体链正常，不能证明游戏进程真的用了这条链。
4. 修字体问题时，优先看最终进程的 `fd` / `maps` / 环境变量，比盲目装一堆字体可靠。
5. 能做 per-game 修复就不要轻易污染全局 fontconfig。

这次的修复本质上是：让老 Unity 原生版仍然以为自己在读 DejaVu，但实际拿到一个包含中文字形的 fallback 字体。范围小、可回滚，也不会把整个系统字体优先级改乱。

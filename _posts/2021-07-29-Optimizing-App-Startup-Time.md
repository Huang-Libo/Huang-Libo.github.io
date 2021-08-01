---
title: "[WIP]【WWDC16】优化 APP 启动（基于 dyld 2）"
categories: [攻城狮, WWDC]
tags: [WWDC16, iOS, APP 性能优化, APP 启动优化, Mach-O, 虚拟内存, dyld 2]
---

![cover.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/cover.jpeg)
<p>
  <h2>目录</h2>
</p>

- [前言](#前言)
  - [1. 谁想看这个 Session](#1-谁想看这个-session)
  - [2. 内容提要](#2-内容提要)
- [Mach-O](#mach-o)
  - [术语](#术语)
  - [Segment](#segment)
  - [Section](#section)
  - [通用的 segment](#通用的-segment)
- [Reference](#reference)

## 前言

> 说明：这个 Session 的链接已打不开，不知为何下掉了，但在 [wwdc.io](https://wwdc.io) 的 macOS 应用中可以查看。

[WWDC16 - Session406: \<Optimizing App Startup Time\>](https://developer.apple.com/videos/play/wwdc2016/406/) 介绍了基于 **dyld 2** 的系统内，*进程*是如何启动的，在 `main()` 之前做了哪些事情，并结合理论和实践给出了优化启动时间的方法。

### 1. 谁想看这个 Session

![audience.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/audience.jpeg)

- 当前开发的 APP 启动太慢
- 想让 APP 保持快速启动
- 想了解系统的运作方式

### 2. 内容提要

- 理论
  - `Mach-O` 格式
  - 虚拟内存 *(Virtual Memory)*的基础
  - `Mach-O` 二进制文件是如何加载和准备的
  - `main()` 函数之前发生的所有事情
- 实践
  - 如何测量 `main()` 函数之前消耗的时间
  - 优化启动时间

## Mach-O

### 术语

![Mach-O-Terminology.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Terminology.jpeg){: .normal width="500"}

`Mach-O` 是 *Mach Object File Format* 的缩写，它包含一系列文件类型：

- **Executable**: APP 或 APP extension 中主要的二进制文件
- **Dylib**: 动态库（aka `DSO` 或 `DLL`）
- **Bundle**: 无法被链接的特殊的 dylib ，只能用 `dlopen()` 打开，比如 macOS 上使用的 plug-ins 。

**Image**: executable 或 dylib 或 bundle 。  
**Framework**: 包含目录的 dylib，目录中有 dylib 需要的*资源文件*和*头文件*。

### Segment

![Mach-O-Segments.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Segments.jpeg){: .normal width="500"}

`Mach-O` 文件由多个 segment 组成，其的名称由*下划线*和*大写字母*构成，如：`__TEXT`，`__DATA`，`__LINKEDIT`。

每个 segment 都是*页大小 (page size)* 的倍数，*页大小*在不同的设备上不一样：

- 在 arm64 上是 **16KB**
- 在其他架构上是 **4KB**

在上图示例中，`__TEXT` 包含 3 个页，`__DATA` 和 `__LINKEDIT` 各包含一个页。

### Section

![Mach-O-Sections.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Sections.jpeg){: .normal width="500"}

section 是编译器忽略的东西，它们只是 segment 的子区域。它们的大小没有被限制为页大小的倍数，但它们是不重叠的 (non-overlapping) 。

其名称由*下划线*和*小写字母*构成，如：`__text`，`__stubs` ，`__const` 等。

### 通用的 segment

- `__TEXT`：位于文件的开头处，包含 `Mach-O header` 、代码、只读的常量；
- `__DATA`：包含所有的可读可写的内容，如：全局变量、静态变量等；
- `__LINKEDIT`：包含关于如何加载这个程序的“元数据”，如：代码签名、符号表等。


## Reference

- 整理的字幕：<https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2016/406_optimizing_app_startup_time/Transcript-Edited.md>

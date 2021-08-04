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
- [Mach-O 简介](#mach-o-简介)
  - [术语](#术语)
  - [Segment](#segment)
  - [Section](#section)
  - [Segment 的类型](#segment-的类型)
  - [Mach-O Univeral Files](#mach-o-univeral-files)
- [虚拟内存简介](#虚拟内存简介)
  - [间接层](#间接层)
- [Reference](#reference)

## 前言

> 说明：这个 Session 的链接已打不开，不知为何下掉了，但在 [wwdc.io](https://wwdc.io) 的 macOS 应用中可以查看。

[WWDC16 - Session406: \<Optimizing App Startup Time\>](https://developer.apple.com/videos/play/wwdc2016/406/) 介绍了基于 **dyld 2** 的系统内，*进程*是如何启动的，在 `main()` 之前做了哪些事情，并结合理论和实践给出了优化启动时间的方法。

### 1. 谁想看这个 Session

![audience.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/audience.jpeg)

Apple 的市场团队做了一些调查，结果显示这 3 个群体将在此 session 中有所收获：

- 当前开发的 APP 启动太慢
- 不想成为前一类人（想让 APP 保持快速启动）
- 想了解系统的运作方式

### 2. 内容提要

- 理论
  - `Mach-O` 格式
  - 虚拟内存 *(Virtual Memory)*的基础
  - `Mach-O` 二进制文件的加载
  - `main()` 函数之前发生的所有事情
- 实践
  - 测量 `main()` 函数之前消耗的时间
  - 优化启动时间

## Mach-O 简介

### 术语

![Mach-O-Terminology.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Terminology.jpeg){: .normal width="500"}

`Mach-O` 是 **Mach O**bject 文件格式的缩写，它包含一系列文件类型：

- **Executable**: APP 或 APP Extension 中主要的二进制文件
- **Dylib**: 动态库（aka `DSO` 或 `DLL`）
- **Bundle**: 无法被链接的特殊的 dylib ，只能用 `dlopen()` 打开，比如 macOS 上使用的 plug-ins 。

**Image**: executable 或 dylib 或 bundle 。  
**Framework**: 包含目录的 dylib，目录中有 dylib 需要的*资源文件*和*头文件*。

### Segment

![Mach-O-Segments.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Segments.jpeg){: .normal width="500"}

`Mach-O` 文件由多个 segment 组成，其名称由*下划线*和*大写字母*构成，如：`__TEXT`，`__DATA`，`__LINKEDIT`。

每个 segment 都是*页大小 (page size)* 的整数倍，*页大小*在不同的设备上不一样：

- 在 arm64 上是 **16KB**
- 在其他架构上是 **4KB**

在上图示例中，`__TEXT` 包含 3 个页，`__DATA` 和 `__LINKEDIT` 各包含一个页。

### Section

![Mach-O-Sections.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Sections.jpeg){: .normal width="500"}

section 是编译器忽略的东西，它们只是 segment 的子区域。它们的大小没有被限制为页大小的整数倍，但它们是不重叠的 (non-overlapping) 。

其名称由*下划线*和*小写字母*构成，如：`__text`，`__stubs` ，`__const` 等。

### Segment 的类型

通用的 segment 有这些：

- `__TEXT`：位于文件的开头处，包含 `Mach-O header` 、代码、只读的常量；
- `__DATA`：包含所有的可读可写的内容，如：全局变量、静态变量等；
- `__LINKEDIT`：包含关于如何加载这个程序的“元数据”，如：代码签名、符号表等。

### Mach-O Univeral Files

![Mach-O-Universal-Files-1.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Universal-Files-1.jpeg){: .normal width="400"}

- 假设我们在构建一个 64 位的 iOS 应用 ，Xcode 会为我们生成一个 arm64 架构的 Mach-O 文件；
- 如果改为也支持 32 位的设备，当我们在 Xcode 内 rebuild 的时候，会生成一个新的 armv7s 架构的 Mach-O 文件。

这两个文件最终会合并为一个新的文件，这个文件就是 *通用 Mach-O 文件 (Mach-O Univeral Files)* ，或称作*通用二进制文件*、*胖二进制文件*。

![Mach-O-Universal-Files-2.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Universal-Files-2.jpeg){: .normal width="500"}

由此可知，*Mach-O Univeral Files* 是包含多个架构的二进制文件，它的起始处有一个 `Fat Header`，里面有个列表记录了所有的架构以及它们在文件中的偏移量。这个 `Fat Header` 的大小是*一个页*。

## 虚拟内存简介

![Virtual-Memory.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Virtual-Memory.jpeg){: .normal width="500"}

你也许会感到疑惑：

- 为什么 segment 的大小需要是*页大小*的整数倍？
- 为什么 `Mach-O Header` 是*一个页*的大小，这浪费了很多空间。

它们基于*页*的原因，与*虚拟内存*有关。

### 间接层

> The adage in software engineering that every problem can be solved by adding a level of indirection

软件工程中有句名言：每个问题都可以通过增加*间接层 (indirection)* 来解决。

*虚拟内存 (Virtual Memory)*解决的问题是，当系统中有很多*进程 (process)* 时，如何管理机器的*物理内存 (physical RAM)* 。

系统设计者的做法就是添加了一个间接层：每个进程都是一个*逻辑地址空间 (logical address space)* ，会被*映射*到一些*物理内存页 (physical page of RAM)* 。

这个映射不一定是一对一的：

- 某个进程中的一些*逻辑地址*可能没有对应的*物理 RAM* ；
- 多个进程的*逻辑地址*可能映射到相同的*物理 RAM* 。

这种设计提供了很多机会。


## Reference

- 整理的字幕：<https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2016/406_optimizing_app_startup_time/Transcript-Edited.md>

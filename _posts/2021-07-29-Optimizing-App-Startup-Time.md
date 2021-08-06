---
title: "[WIP]【WWDC16】优化 APP 启动（基于 dyld 2）"
categories: [攻城狮, WWDC]
tags: [WWDC16, iOS, APP 性能优化, APP 启动优化, Mach-O, 虚拟内存, dyld 2]
---

![cover.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/cover.jpeg){: .normal width="600"}
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
  - [虚拟内存的特性](#虚拟内存的特性)
    - [1. 缺页异常 (page fault)](#1-缺页异常-page-fault)
    - [2. 多个进程共享物理内存](#2-多个进程共享物理内存)
    - [3. File backed mapping](#3-file-backed-mapping)
    - [4. 写时复制 (copy on write)](#4-写时复制-copy-on-write)
    - [5. Dirty page & Clean page](#5-dirty-page--clean-page)
    - [6. 页的权限](#6-页的权限)
- [Mach-O 文件加载到虚拟内存](#mach-o-文件加载到虚拟内存)
  - [示例：第一个进程加载 dylib](#示例第一个进程加载-dylib)
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

*虚拟内存 (Virtual Memory, VM)*解决的问题是，当系统中有很多*进程 (process)* 时，如何管理机器的*物理内存 (physical RAM)* 。

系统设计者的做法就是添加了一个间接层：每个进程都是一个*逻辑地址空间 (logical address space)* ，它们会被*映射*到一些*物理内存页 (physical page of RAM)* 。

这个映射不一定是一对一的：

- 某个进程中的一些*逻辑地址*可能没有映射到*物理内存* ；
- 多个进程的*逻辑地址*可能会映射到相同的*物理内存* 。

这种设计提供了很多机会。

### 虚拟内存的特性

#### 1. 缺页异常 (page fault)

如果**某进程**内的一个*逻辑地址*没有映射到*物理内存* ，当这个进程访问该逻辑地址时，就会发生一次 **缺页异常** 。此时，内核将停止这个进程，并试图找出需要发生什么。

#### 2. 多个进程共享物理内存

**两个进程**的不同的*逻辑地址*可能会映射到相同的*物理页*，也就是说这两个进程可共享相同的*物理内存*。

#### 3. File backed mapping

...（这里需要更详细的解释）

可以通过 `mmap` 调用告诉 `VM` 系统，将文件的*部分内容*映射到进程中的某个地址范围，而不是将整个文件读入。当 page fault 发生的时候，只读取缺失的那一个*页*，也就是对文件执行惰性加载。

因此，Mach-O 的 `__TEXT` 段可以被映射到多个进程，可以被惰性读取，可以在多个进程之间共享。

#### 4. 写时复制 (copy on write)

对于 `__DATA` 段，由于它是可读可写的，因此会对它使用*写时复制*技术，这与 *Apple file system* 上拷贝文件的操作类似。

其机制是，当多个进程只是读取*全局变量*时，会在多个进程之间地共享 `__DATA` 页；当某个进程尝试往 `__DATA` 页写入内容时，就会触发*写时复制* 。

*写时复制*导致内核将该*页*复制到另一段*物理内存* 中，并将*映射*重定向到该*物理内存*中。所以这个进程就有了它自己的那个*页*的副本。

#### 5. Dirty page & Clean page

上面提到的通过 copy 得到的 page 就是 *dirty page* 。

- **Dirty page**：包含特定进程信息的*页*。
- **Clean page**：不包含特定进程信息的*页*。如果需要，内核可以重新生成 *clean page* ，比如重新从磁盘读取。

由此可知，dirty page 比 clean page 昂贵得多。

#### 6. 页的权限

可以对 `Mach-O` 文件的*每个页*设置访问权限。权限可以是 r、w、x 的任意组合。

## Mach-O 文件加载到虚拟内存

在接下来的示例中，将介绍两个进程加载同一个 dylib 的详细流程。（假设这个 dylib 之前还没有被加载过）

### 示例：第一个进程加载 dylib

![Mach-O-Image-Loading-1.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Image-Loading-1.jpeg)

如图所示，这里有一个 dylib 文件，*我们不是在内存中读取它而是在内存中映射它 (rather than reading it in memory we've mapped it in memory)* 。在内存中，这个 dylib 需要 8 页。

这里的节省的不同之处在于这些*填充的 0 (ZeroFills)* 。大部分全局变量的初始值是 0 ，因此，*静态链接器*进行了优化，将所有值为零的全局变量移到末尾，这样就不会占用磁盘空间。在虚拟内存中，当这个*页*第一次被访问时会用 0 填充，因此不需要读取。

![Mach-O-Image-Loading-2.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Image-Loading-2.jpeg)

dyld 要做的第一件事是在进程中、物理内存中查看 `Mach-O header`，此时 `Mach-O header` 还没有加载到物理内存中，因此会发生一次缺页异常。然后，内核将读取 `Mach-O` 文件的第一页并置于物理内存中，再设置好*虚拟内存*到*物理内存*的映射。

![Mach-O-Image-Loading-3.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Image-Loading-3.jpeg)

现在，dyld 就可以开始读取 `Mach-O header` 了。`Mach-O header` 会告知 dyld 需要去 `__LINKEDIT` *页*中读取一些信息，此时，会再发生一次缺页异常。同样，内核会将 `__LINKEDIT` 读入物理内存并设置好映射。

![Mach-O-Image-Loading-4.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Image-Loading-4.jpeg)

接着，这个 `__LINKEDIT` *页*会告诉 dyld 需要对 `__DATA` *页*做一些 Fix-ups ，以使这个 dylib 可运行。读取这个 `__DATA` *页*的流程与之前类似。

![Mach-O-Image-Loading-5.jpeg](/images/WWDC/2016/406-optimizing-app-startup-time/Mach-O-Image-Loading-5.jpeg)

不同的是，dyld 在做 Fix-ups 时，会往这个 `__DATA` *页*写入内容，此时就触发了*写时复制*，这个 page 就变成了 dirty page 。

小结：

- 如果我们只是简单地分配 8 页空间，然后把整个 dylib 读入，会产生 8 个 dirty page ；
- 我们按上述模式按*页*加载，只会产生 1 个 dirty page 和 2 个 clean page 。

## Reference

- 整理的字幕：<https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2016/406_optimizing_app_startup_time/Transcript-Edited.md>

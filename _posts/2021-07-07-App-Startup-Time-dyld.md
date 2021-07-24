---
title: "[WIP]【WWDC17】优化 APP 启动之 dyld"
categories: [攻城狮, WWDC]
tags: [WWDC17, iOS, APP 性能优化, APP 启动优化, dyld, dyld3]
---

![cover.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/cover.jpeg)

<p>
  <h2>目录</h2>
</p>

- [前言](#前言)
- [术语](#术语)
  - [Darwin 里的 dyld 的全称](#darwin-里的-dyld-的全称)
  - [启动时间 (Startup Time)](#启动时间-startup-time)
  - [启动闭包 (Launch Closure)](#启动闭包-launch-closure)
- [回顾 WWDC16：优化 APP 启动的建议](#回顾-wwdc16优化-app-启动的建议)
  - [减少启动阶段的任务](#减少启动阶段的任务)
  - [多使用 Swift](#多使用-swift)
- [Instruments: Static initializer tracing](#instruments-static-initializer-tracing)
- [dyld 简史](#dyld-简史)
  - [dyld 1.0 (1996–2004)](#dyld-10-19962004)
  - [dyld 2.0 (2004–2007)](#dyld-20-20042007)
    - [1. 改进：速度和语义](#1-改进速度和语义)
    - [2. 不足：健全性检验比较有限](#2-不足健全性检验比较有限)
  - [dyld 2.x (2007–2017)](#dyld-2x-20072017)
    - [1. 支持了更多的架构和系统](#1-支持了更多的架构和系统)
    - [2. 提高了安全性](#2-提高了安全性)
    - [3. 优化了性能](#3-优化了性能)
  - [共享缓存 (Shared Cache)](#共享缓存-shared-cache)
    - [优化方式](#优化方式)
      - [1. 重新排列二进制文件 (Rearranges binaries) 以提高加载速度](#1-重新排列二进制文件-rearranges-binaries-以提高加载速度)
      - [2. 预链接动态库 (Pre-links dylibs)](#2-预链接动态库-pre-links-dylibs)
      - [3. 预构建 (Pre-builds) dyld 和 ObjC 使用的数据结构](#3-预构建-pre-builds-dyld-和-objc-使用的数据结构)
    - [共享缓存的生成方式](#共享缓存的生成方式)
  - [dyld 3 (2017)](#dyld-3-2017)

## 前言

[WWDC17 - Session413: \<App Startup Time: Past, Present, and Future\>](https://developer.apple.com/videos/play/wwdc2017/413/) 详细介绍了 `dyld` ，并给出了 APP 启动优化相关的建议。演讲者来自 *dyld Team* 。

## 术语

### Darwin 里的 dyld 的全称

> 一些博文在介绍 `dyld` 时，把它的英文全称写错了，那个全称是 `FreeBSD` 上使用的。这里为了防止大家记成错的，就不提那个名称了。  
>  
> 虽然 `Darwin` 的内核也用了 `FreeBSD` 的一部分，但对 `dyld` 的名称有自己的解释。

在终端使用 `man dyld` 查看文档，可看出其全称是 **the dynamic linker** ：

![man-dyld.jpg](/images/WWDC/Common/man-dyld.jpg)

### 启动时间 (Startup Time)

> 说明：通常意义上的启动时间也包括 `main()` 之后的一部分时间，但本演讲只涉及 `main()` 之前的时间。

这个演讲的*启动时间*是指 `main()` 之前的时间，并讲解如何加快这个阶段的启动速度。

### 启动闭包 (Launch Closure)

启动闭包指**启动一个 APP 所需要的所有信息**。比如：

- APP 使用了哪些 **dylib** 。
- APP 内使用的各种**符号 (symbols) 的偏移量 (offsets)** 。
- APP 的**代码签名 (code signatures)** 的位置。

## 回顾 WWDC16：优化 APP 启动的建议

![wwdc16-advice.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/wwdc16-advice.jpeg)

### 减少启动阶段的任务

- 减少*嵌入的 (Embed)* `dylib` 的数量。
- 减少声明的 `classes/methods` 的数量。
- 少使用 `initializers` 。

### 多使用 Swift

`Swift` 语言的设计避开了 `C` , `C++` 和 `Objective-C` 中的许多隐患。

- `Swift` 没有 `initializer` 。
- `Swift` 改善了大小。
- `Swift` 不允许存在*未对齐的（misaligned）*数据结构，未对齐的数据结构会在启动时消耗额外的时间。

相关 Session ：

- [WWDC16 - Session406: \<Optimizing App Startup Time\>](https://developer.apple.com/videos/play/wwdc2016/406/)
  - (链接已打不开，不知为何下掉了，但在 [wwdc.io](https://wwdc.io) 的 macOS 应用中可以查看)

## Instruments: Static initializer tracing

静态初始化器的代码在 `main()` 之前调用，而开发者对 `main()` 之前发生的事缺乏可视的观测方式。

*iOS 11* 和 *macOS High Sierra* 的内核与 `dyld` 中新增了相关基础设施 (infrastructure) ，因此可使用 Instruments 中新增的 **Static Initializer Calls** 为每个**静态初始化器 (Static Initializer)** 提供精确的计时：

![Instruments-Static-Initializer-Tracing.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/Instruments-Static-Initializer-Tracing.jpeg){: .normal width="350"}

添加方式：  

- Instruments
  - Blank（空模板）
    - 点击右上角的 `+` 按钮
      - 在弹出的页面中搜索 **Static Initializer Calls**，双击即可添加
      - 可搭配 **Time Profiler** 使用（也是搜索后双击添加）

## dyld 简史

### dyld 1.0 (1996–2004)

![dyld-1.0.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-1.0.jpeg)

早在 1996 年，`dyld 1` 就作为 `NeXTStep 3.3` 的一部分发布了。在这之前，`NeXTStep` 使用静态库。

这个时间早于 **POSIX** `dlopen()` 标准颁布的时间，大多数系统也还未使用大型 `C++` 动态库。

当时，由于和其他的 `Unix` 系统的实现不太一样，因此人们在 `macOS 10` 的早期版本上编写*第三方包装程序 (third-party wrappers)* ，以支持标准的 `Unix` 软件。

在发布 *macOS Cheetah (10.0)* 之前，`dyld 1` 中引入了 *Prebinding* 技术，它虽然可以加快启动速度，但有安全隐患，因为 *Prebinding* 会修改应用二进制文件。详情请看 [字幕中 Prebinding 的简介](https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2017/413-App-Startup-Time-Past-Present-and-Future/Transcript-Edited.md#prebinding)。

### dyld 2.0 (2004–2007)

![dyld-2.0.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-2.0.jpeg)

`dyld 2.0` 是作为 `macOS Tiger (10.4)` 的一部分发布的，是对 `dyld` 的完全重写。

#### 1. 改进：速度和语义

- `dyld 2.0` 正确地支持了 `C++` *初始化器*的语义，因此可以得到高效的 `C++` 库支持。
- `dyld 2.0` 有语义正确的、完全 *native* 的 `dlopen` 和 `dlsym` 。
- 由于 `dyld 2.0` 的速度得到了提升，因此减少了 `prebinding`，只有系统库会执行 `prebinding`，且只会在系统升级的时候执行。

#### 2. 不足：健全性检验比较有限

`dyld 2.0` 是为速度提升设计的，因此*健全性检测 (sanity checking)*比较有限，存在一些安全性问题。但那个年代的（针对 Apple 系统的）恶意软件比较少。

### dyld 2.x (2007–2017)

![dyld-2.x.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-2.x.jpeg)

在这个时期，`dyld 2.x` 做了多项显著的改进。

#### 1. 支持了更多的架构和系统

- 支持更多的**架构**：`dyld 2` 最初是在 `PowerPC` 上发行的，后来又增加了对 `x86`, `x86_64`, `arm`, `arm64` 架构的支持。
- 支持更多的**系统**：这个时期发布了 `iOS`，`tvOS`，和 `watchOS` 系统，`dyld` 做了大量相关的适配工作。

#### 2. 提高了安全性

- *Codesigning* ：代码签名。
- *ASLR (Address Space Layout Randomization)* ： 每次加载库时，它可能在不同的地址。
- *Bounds checking* ：检查 `Mach-O Header` 的边界，所以其他人无法用*篡改 (malformed)* 的二进制文件做某些类型的*连接 (attach)* 。

#### 3. 优化了性能

完全用**共享缓存**替代了 prebinding 。

### 共享缓存 (Shared Cache)

![dyld-Shared-Cache.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-Shared-Cache.jpeg)

**共享缓存是包含所有系统 dylib 的单个文件。** 它是在 `iOS 3.1` 和 `macOS Snow Leopard (10.6)` 引入的，并**完全取代了 prebinding** 。

由于所有的系统 `dylib` 被合并成了单个文件，因此系统可以对其做一些优化。

#### 优化方式

##### 1. 重新排列二进制文件 (Rearranges binaries) 以提高加载速度

系统可以**重新排列**这些二进制文件的所有的*文本段 (text segments)* 和所有的*数据段 (data segments)* ，重写他们的整个*符号表 (symbol tables)* ，以减少大小。这样，系统需要在每个进程*挂载 (mount)* 的*区域 (regions)* 更少。

##### 2. 预链接动态库 (Pre-links dylibs)

*Prelinker* 可以**打包二进制段 (pack binary segments)** 来节省内存。开发者不需要做任何改动，就能节省很多内存。在 iOS 系统上，可以在运行时节省 500MB ~ 1GB 的内存。

##### 3. 预构建 (Pre-builds) dyld 和 ObjC 使用的数据结构

它还*预构建 (Pre-builds)* 了 *dyld* 和 *ObjC* 在运行时使用的数据结构，这样我们就不必在启动时做了。这又一次节省了更多的内存和大量的时间。

#### 共享缓存的生成方式

- 在 macOS 上，它是在本地构建的，当你看到 *optimizing system performance* 时，就是系统在更新共享缓存。
- 在所有其他的 Apple 系统上，共享缓存是作为系统的一部分发布的。

### dyld 3 (2017)

![dyld-3](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3.jpeg)

*dyld 3* 是对*动态链接 (dynamic linking)* 的完全重新思考。

- 2017 年，*iOS 11* 的系统自带 APP 开始使用 *dyld 3* ，第三方 APP 还是使用 *dyld 2.x* 。
- 2019 年，*iOS 13* 的系统自动 APP 和第三方 APP 都使用 *dyld 3* 。

（本文写于 2021年7月，此时已是 *iOS 14* ）


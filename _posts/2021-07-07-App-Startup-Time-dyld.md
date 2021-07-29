---
title: "【WWDC17】优化 APP 启动之 dyld"
categories: [攻城狮, WWDC]
tags: [WWDC17, iOS, APP 性能优化, APP 启动优化, dyld, dyld3]
---

![dyld-cover.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-cover.jpeg)

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
    - [1. dyld 2.0 的改进：速度和语义](#1-dyld-20-的改进速度和语义)
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
- [dyld 3 诞生的背景](#dyld-3-诞生的背景)
  - [为什么又重写动态链接器](#为什么又重写动态链接器)
  - [无法使用 XCTest 测试 dyld 2.x 代码的原因](#无法使用-xctest-测试-dyld-2x-代码的原因)
  - [dyld 3 的改进](#dyld-3-的改进)
- [dyld 2 的执行流程](#dyld-2-的执行流程)
  - [1. Parse mach-o headers & Find dependencies](#1-parse-mach-o-headers--find-dependencies)
  - [2. Map mach-o files](#2-map-mach-o-files)
  - [3. Perform symbol lookups](#3-perform-symbol-lookups)
  - [4. Bind and rebase](#4-bind-and-rebase)
  - [5. Run initializers](#5-run-initializers)
- [dyld 2 向 dyld 3 的转变](#dyld-2-向-dyld-3-的转变)
  - [1. 安全敏感组件](#1-安全敏感组件)
  - [2. 可缓存的部分](#2-可缓存的部分)
- [dyld 3 架构简介](#dyld-3-架构简介)
  - [概览：dyld 3 由三部分组成](#概览dyld-3-由三部分组成)
  - [1. 进程外的 mach-o 解析器/编译器](#1-进程外的-mach-o-解析器编译器)
  - [2. 运行启动闭包的进程内引擎](#2-运行启动闭包的进程内引擎)
  - [3. 启动闭包缓存服务](#3-启动闭包缓存服务)
- [为 dyld 3 做准备](#为-dyld-3-做准备)
  - [潜在的问题](#潜在的问题)
  - [__DATA 中未对齐的指针](#__data-中未对齐的指针)
  - [急切的符号解析](#急切的符号解析)
    - [1. dyld 2 执行惰性的符号解析 (Lazy symbol resolution)](#1-dyld-2-执行惰性的符号解析-lazy-symbol-resolution)
    - [2. dyld 3 执行急切的符号解析 (Eager symbol resolution)](#2-dyld-3-执行急切的符号解析-eager-symbol-resolution)
    - [3. dyld 3 对符号缺失的兼容](#3-dyld-3-对符号缺失的兼容)
    - [4. Linker Flag: _bind_at_load](#4-linker-flag-_bind_at_load)
  - [dlopen() / dlsym() / dladdr()](#dlopen--dlsym--dladdr)
  - [dlclose()](#dlclose)
  - [all_image_infos](#all_image_infos)
  - [最佳实践](#最佳实践)
- [Reference](#reference)

## 前言

[WWDC17 - Session413: \<App Startup Time: Past, Present, and Future\>](https://developer.apple.com/videos/play/wwdc2017/413/) 详细介绍了 `dyld` ，并给出了 APP 启动优化相关的建议。演讲者来自 *dyld Team* 。

## 术语

### Darwin 里的 dyld 的全称

> 一些博文在介绍 `dyld` 时，把它的英文全称写错了，那个全称是 `FreeBSD` 上使用的。这里为了防止大家记成错的，就不提那个名称了。  
>  
> 虽然 `Darwin` 的内核也用了 `FreeBSD` 的一部分，但随着版本的迭代，已对 `dyld` 已有了自己的解释。

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
- 少用 `initializers` 。

### 多使用 Swift

`Swift` 语言的设计避开了 `C` , `C++` 和 `Objective-C` 中的许多隐患。

- `Swift` 没有 `initializer` 。
- `Swift` 改善了大小。
- `Swift` 不允许存在*未对齐的（misaligned）*数据结构，未对齐的数据结构会在启动时消耗额外的时间。

相关 Session ：

- [WWDC16 - Session406: \<Optimizing App Startup Time\>](https://developer.apple.com/videos/play/wwdc2016/406/)
  - (链接已打不开，不知为何下掉了，但在 [wwdc.io](https://wwdc.io) 的 macOS 应用中可以查看)

## Instruments: Static initializer tracing

*静态初始化器 (Static Initializer)*的代码在 `main()` 之前调用，而开发者对 `main()` 之前发生的事缺乏可视的观测方式。

*iOS 11* 和 *macOS High Sierra* 的内核与 `dyld` 中新增了相关基础设施 (infrastructure) ，因此可使用 Instruments 中新增的 **Static Initializer Calls** 为每个*静态初始化器* 提供精确的计时：

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

发布 *macOS Cheetah (10.0)* 时，在 `dyld 1` 中引入了 *Prebinding* 技术，它虽然可以加快启动速度，但有安全隐患，因为 *Prebinding* 会修改应用二进制文件。详情请看 [字幕中 Prebinding 的简介](https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2017/413-App-Startup-Time-Past-Present-and-Future/Transcript-Edited.md#prebinding)。

### dyld 2.0 (2004–2007)

![dyld-2.0.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-2.0.jpeg)

`dyld 2.0` 是作为 `macOS Tiger (10.4)` 的一部分发布的，是对 `dyld` 的完全重写。

#### 1. dyld 2.0 的改进：速度和语义

- 正确地支持了 `C++` *初始化器*的语义，因此可以得到高效的 `C++` 库支持。
- 有语义正确的、完全 *native* 的 `dlopen` 和 `dlsym` 。
- 由于速度得到了提升，因此减少了 `prebinding`，只有系统库会执行 `prebinding`，且只会在系统升级的时候执行。

#### 2. 不足：健全性检验比较有限

`dyld 2.0` 是为速度提升设计的，因此*健全性检测 (sanity checking)*比较有限，存在一些安全性问题。但那个年代的（针对 Apple 系统的）恶意软件比较少。

### dyld 2.x (2007–2017)

![dyld-2.x.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-2.x.jpeg)

在这个时期，`dyld 2.x` 做了多项显著的改进。

#### 1. 支持了更多的架构和系统

- 支持更多的**架构**：`dyld 2` 最初是在 `PowerPC` 上发行的，后来又增加了对 `x86`, `x86_64`, `arm`, `arm64` 架构的支持。
- 支持更多的**系统**：这个时期发布了 `iOS`，`tvOS`，和 `watchOS` 系统，`dyld` 做了大量相关的适配工作。

#### 2. 提高了安全性

- *Codesigning* ：*代码签名*。
- *ASLR (Address Space Layout Randomization)* ： *地址空间布局随机化*，每次加载库时，它可能在不同的地址。
- *Bounds checking* ：检查 `Mach-O Header` 的边界，所以其他人无法用*篡改 (malformed)* 的二进制文件来做某些类型的*连接 (attach)* 。

#### 3. 优化了性能

完全用**共享缓存**替代了 prebinding 。下面将简要介绍*共享缓存*。

### 共享缓存 (Shared Cache)

![dyld-2.x-Shared-Cache.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-2.x-Shared-Cache.jpeg)

**共享缓存是包含所有系统 dylib 的单个文件。** 它是在 `iOS 3.1` 和 `macOS Snow Leopard (10.6)` 引入的，并**完全取代了 prebinding** 。

由于所有的系统 `dylib` 被合并成了单个文件，因此系统可以对其做一些优化。

#### 优化方式

##### 1. 重新排列二进制文件 (Rearranges binaries) 以提高加载速度

系统可以**重新排列**这些二进制文件的所有的*文本段 (text segments)* 和所有的*数据段 (data segments)* ，重写他们的整个*符号表 (symbol tables)* ，以减少大小。这样，系统需要在每个进程*挂载 (mount)* 的*区域 (regions)* 更少。

##### 2. 预链接动态库 (Pre-links dylibs)

*Prelinker* 可以**打包二进制段 (pack binary segments)** 来节省内存。开发者不需要做任何改动，就能节省很多内存。在 iOS 系统上，可以在运行时节省 **500MB ~ 1GB** 的内存。

##### 3. 预构建 (Pre-builds) dyld 和 ObjC 使用的数据结构

*预构建 (Pre-builds)* *dyld* 和 *ObjC* 在运行时使用的数据结构，这样我们就不必在启动时做了。这也节省了很多内存和时间。

#### 共享缓存的生成方式

- 在 macOS 上，它是在本地构建的，当你看到 *optimizing system performance* 时，就是系统在更新共享缓存。
- 在所有其他的 Apple 系统上，共享缓存是作为系统的一部分发布的。

注：从 *macOS Big Sur (11.0)* 开始，共享缓存也是作为系统的一部分发布的。

参考：

- <https://iphonedev.wiki/index.php/Dyld_shared_cache>

### dyld 3 (2017)

![dyld-3](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3.jpeg)

*dyld 3* 是对*动态链接 (dynamic linking)* 的完全重新思考。

- 2017 年，*iOS 11* 的系统自带 APP 开始使用 *dyld 3* ，第三方 APP 还是使用 *dyld 2.x* 。
- 2019 年，*iOS 13* 的系统自动 APP 和第三方 APP 都使用 *dyld 3* 。

（注：本文写于 2021年7月，此时已是 *iOS 14* ）

下面将详细介绍 `dyld 3` 。

## dyld 3 诞生的背景

### 为什么又重写动态链接器

1. 性能：理论上，让 APP 运行起来的*最小的任务量*包含哪些任务？
2. 安全：是否可以有更严格的安全检查并预先设计安全？
3. 可测试性和可靠性：能否使 `dyld` 更容易测试？

### 无法使用 XCTest 测试 dyld 2.x 代码的原因

由于 `XCTest` 依赖于动态链接器的底层功能来将它插入进程，所以无法使用 `XCTest` 对 `dyld 2.x` 的代码做测试。因此 `dyld` 的工程师很难对 `dyld` 的*安全性*和*性能*做测试。

### dyld 3 的改进

- 把 `dyld` 复杂的操作从*进程 (process)* 中移出，以**守护进程 (daemon)** 的形式存在。因此，使用 `XCTest` 测试 `dyld` 的代码也变得很容易了。
- 这使得进程中 `dyld` 的其余部分尽可能小，因此也减少了 APP 内的（可被恶意软件利用的）“攻击平台”。
- 另外，这也使 APP 的启动更快了，因为最快的代码就是你从未写过的代码，其次是从未执行的代码。

> 原文：The fastest code is code you never write, followed closely by code you almost never execute.  
> 这让小编想到了在海淀区 Hello World “公园”的建筑上的一句话：No code is faster than no code.

## dyld 2 的执行流程

![dyld-2-launch.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-2-launch.jpeg){: .normal width="300"}

### 1. Parse mach-o headers & Find dependencies

`dyld` 会解析 `Mach-O Headers` ，并寻找 APP 需要的动态库，然后这些动态库可能依赖了其他动态库，`dyld` 就递归地做这个任务，直到获得 APP 的完整的动态库依赖图 (a complete graph of all your dylibs) 。

平均而言，一个 iOS 应用需要加载 300 ~ 600 个系统动态库，数量较多，因此有相当大的工作量。

### 2. Map mach-o files

这个阶段 `dyld` *映射 (map)* 所有的 `Mach-O` 文件到 APP 的*地址空间 (address space)* 。

### 3. Perform symbol lookups

接下来执行*符号查找 (symbol lookups)* 。比如 APP 内使用了 `printf` ，`dyld` 就会去系统动态库中查找它的地址，找到后就把这个地址拷贝到 APP 内的一个*函数指针 (function pointer)* 中。

### 4. Bind and rebase

因为 APP 是在一个随机地址（用了 `ASLR` 技术），因此 APP 内的**所有的指针**都必须把这个基地址值加上。

### 5. Run initializers

最后，`dyld` 调用 APP 内所有的 `initializer` ，完成后就能调用 APP 的 `main()` 函数了。

## dyld 2 向 dyld 3 的转变

![dyld-2-to-dyld-3.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-2-to-dyld-3.jpeg)

`dyld 3` 与 `dyld 2` 的主要区别是将大部分功能从*进程 (process)* 移到了*守护进程 (daemon)* 。

### 1. 安全敏感组件

有安全隐患的阶段：

- **Parsing mach-o headers**
- **Find dependencies**

如果 `Mach-O Headers` 被篡改了，则可被用来做特定类型的攻击。我们的 APP 中可能使用了 `@rpath` ，也就是*搜索路径 (search path)* ，通过篡改它们或者在正确的位置插入库，就能侵入 APP 。

因此，`dyld 3` 将这部分功能从*进程 (process)* 移到了*守护进程 (daemon)* 。

### 2. 可缓存的部分

![dyld-3-compare.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-compare.jpeg)
_dyld 2 与 dyld 3 执行流程的对比_

可缓存的阶段：

- **Parsing mach-o headers**
- **Find dependencies**
- **Perform symbol lookups**

除非*执行了软件更新*或*更改了磁盘上的库*，否则每次启动时：

1. APP 依赖的库不会变。
2. 库中的*符号 (symbols)* 始终处于相同的*偏移量 (offset)* 。

因此，在 `dyld 3` 中，将可缓存的阶段挪到了最前面，并将执行结果以一个*闭包 (closure)* 的形式写入磁盘中，在之后的流程中会用到这个闭包。

## dyld 3 架构简介

### 概览：dyld 3 由三部分组成

![dyld-3-architecture-overview.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-architecture-overview.jpeg)

`dyld 3` 有三个组件：

1. 一个*进程外 (out-of-process)* 的 `mach-o` *解析器/编译器 (parser/complier)*
2. 一个运行*启动闭包*的*进程内 (in-process)* 引擎
3. 一个*启动闭包*缓存服务

大多数启动使用缓存，而不必调用*进程外*的 `mach-o` *解析器或编译器*。

*启动闭包*比 `mach-o` 要简单得多，它们是 **memory map** 文件，因此不需要使用复杂的方式来解析。并且验证*启动闭包*也很简单。*启动闭包*就是为速度而生的。

接下来简要介绍这三个组件。

### 1. 进程外的 mach-o 解析器/编译器

![dyld-3-architecture-1.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-architecture-1.jpeg)

首先，`dyld 3` 包含一个进程外的 `mach-o` 解析器/编译器。

它的主要功能是：

- 解析 (resolve) 所有的启动需要的*搜索路径 (search path)* 、*@rpath* 、*环境变量 (environment variable)* ；
- 解析 (parse) 所有的 `mach-o` 二进制文件；
- 执行*符号查找 (symbol lookup)* ；
- 最后，创建一个带有结果的*闭包 (closure)* 。

这部分功能在 `dyld 3` 中是一个普通的*守护进程*，所以 `dyld` 团队可以使用普通的测试基础设施。

### 2. 运行启动闭包的进程内引擎

![dyld-3-architecture-2.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-architecture-2.jpeg)

然后，`dyld 3` 包含一个小型的进程内引擎，也就是说这部分功能会加载到 APP 的进程中，也是最常被使用到的模块。

它的主要功能是：

- 验证启动闭包是否正确；
- map in 所有的 dylib ；
- *Fix-ups*: *Bind* 和 *Rebase* ；
- 运行所有的 initializer ；
- 跳转到 APP 的 `main()` 函数。

值得注意的是，在这个过程中再也不需要解析 `mach-o header` 或执行*符号查找*，这使 APP 的启动更快了。

### 3. 启动闭包缓存服务

![dyld-3-architecture-3.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-architecture-3.jpeg)

最后，`dyld 3` 包含一个*启动闭包缓存服务 (launch closure caching service)* 。

在 `iOS`、`tvOS`、`watchOS` 中，即使 APP 没有运行过，`启动闭包`就已经被预构建了：

- **系统 APP** 的*启动闭包*內建在*共享缓存*中。
- **第三方 APP** 在安装时或系统升级时构建启动闭包（因为系统升级时，系统动态库可能有变动）。

在 `macOS` 上，第一次启动 APP 时有所不同，但之后再启动就能使用缓存的启动闭包了：

- On macOS, because you can **side load applications**, the **in-process engine can RPC out to the daemon if necessary on first launch**（???）, and then after that it will be able to use a cached closure just like everything else.

## 为 dyld 3 做准备

### 潜在的问题

![dyld-3-potential-issue.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-potential-issue.jpeg)

在切换到 `dyld 3` 时，有一些潜在的问题需要注意：

- `dyld 3` 完全兼容 `dyld 2.x` ，
  - 但是一些旧 API 在 `dyld 3` 上会运行的较慢或者使用回退机制，因此开发者需要避免这种情况。
  - 为 `dyld 2.x` 做的一些的优化可能已经无效了。
- `dyld 3` 有*更严格的链接语义 (Stricter linking semantics)* ，
  - 对于旧的二进制，会兼容旧的行为；
  - 对于新的二进制，会导致 *linker error* 。

### __DATA 中未对齐的指针

![dyld-3-unaligned-pointer-1.jpge](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-unaligned-pointer-1.jpeg)

假如 APP 中有一个*全局*的数据结构指向一个函数或者另一个*全局*的数据结构，那么这个指针需要在 APP 启动前被 *fix up* ，并且这个指针必须对齐、以获得最佳的性能。

*fix up* 未对齐的指针要复杂得多：

- 未对齐的指针会*跨越 (span)* 多个页面，这可能会导致更多的*页面错误 (page fault)* 和其他问题
- 而且它们可能存在与*多处理器 (multiprocessor)* 相关的*原子性问题 (atomicity issues)*

*静态连接器 (static linker)* 会对这种情况发出一个 *ld warning* ：指针没有在某地址上对齐。这通常是对应数据段的地址。

下面将给出一个 C 中未对齐指针的案例。Swift 不允许这么做，因此，请多使用 Swift 。

![dyld-3-unaligned-pointer-2.jpge](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-unaligned-pointer-2.jpeg)

![dyld-3-unaligned-pointer-3.jpge](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-unaligned-pointer-3.jpeg)

### 急切的符号解析

![dyld-3-eager-symbol-resolution-1.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-eager-symbol-resolution-1.jpeg){: .normal width="600"}

#### 1. dyld 2 执行惰性的符号解析 (Lazy symbol resolution)

在 `dyld 2` 中，如果在启动时就查找所有的符号，代价太高。因此 `dyld 2` 执行的是*惰性的符号解析*。

- 每个符号在第一次被调用时执行符号查找 。
- 如果符号缺失，将在首次调用时导致 crash 。

比如，APP 内第一次调用 `printf` 函数时，最初它的函数指针不是指向 `printf` 的实现，而是指向 `dyld` 中的一个函数，这个 `dyld` 的函数会返回指向 `printf` 实现的指针。第二次调用 `printf` 时，就直接调用它的实现了。

#### 2. dyld 3 执行急切的符号解析 (Eager symbol resolution)

在 `dyld 3` 中，由于符号查找的结果缓存在启动闭包中，所以符号查找非常快，可以在启动时就检查所有的符号。也就是说 `dyld 3` 执行*急切的符号解析*。

#### 3. dyld 3 对符号缺失的兼容

由上述内容可知，`dyld 2` 和 `dyld 3` 对**符号缺失**的反应不同。当缺失某个符号时：

- 在使用 `dyld 2` 的系统上，APP 可成功启动，但会在第一次调用缺失的符号时 crash 。
- 在使用 `dyld 3` 的系统上，APP 无法正常启动。

`dyld 3` 目前对这种情况做了兼容，使得其行为与 `dyld 2` 保持一致，这样可以保障缺失符号的旧版 APP 能正常启动。

其兼容方法是在 `dyld 3` 中内置了一个会自动 crash 的符号，如果 APP 启动时未找到符号，就将其绑定到这个符号上，因此，第一次调用时会 crash 。

需要注意的是，这是当前 `dyld 3` 的兼容行为，未来可能会强制所有的符号解析在前面执行。也就是说，启动时如果缺失符号就会 crash ，这使得开发者能在开发时就发现代码的问题。

#### 4. Linker Flag: _bind_at_load

![dyld-3-eager-symbol-resolution-2.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-3-eager-symbol-resolution-2.jpeg)

在使用 `dyld 2` 的系统上，可以在 **Other Linker Flags** 中添加 `_bind_at_load` 参数，可以强制 `dyld 2` 在启动时加载所有符号，这样可以提前发现问题，为 `dyld 3` 的到来做好准备。

要注意的是，由于 `_bind_at_load` 会导致启动很慢，因此只能在 Debug build 中使用，不要在 Release build 中使用。

### dlopen() / dlsym() / dladdr()

![dyld-dlopen-dlsym-dladdr.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-dlopen-dlsym-dladdr.jpeg){: .normal width="600"}

### dlclose()

![dyld-dlclose.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-dlclose.jpeg){: .normal width="600"}

### all_image_infos

![dyld-all_image_infos.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-all_image_infos.jpeg){: .normal width="500"}

### 最佳实践

![dyld-best-practices.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/dyld-best-practices.jpeg)

## Reference

- 整理的字幕：<https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2017/413-App-Startup-Time-Past-Present-and-Future/Transcript-Edited.md>
- 原始的 PDF：<https://devstreaming-cdn.apple.com/videos/wwdc/2017/413fmx92zo14voet8/413/413_app_startup_time_past_present_and_future.pdf?dl=1>

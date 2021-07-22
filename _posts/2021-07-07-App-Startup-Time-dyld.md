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
  - [启动时间（Startup Time）](#启动时间startup-time)
  - [启动闭包（Launch Closure）](#启动闭包launch-closure)
- [回顾 WWDC16：优化 APP 启动时间的建议](#回顾-wwdc16优化-app-启动时间的建议)
  - [减少启动阶段的任务](#减少启动阶段的任务)
  - [多使用 Swift](#多使用-swift)

## 前言

[WWDC17 - Session413: \<App Startup Time: Past, Present, and Future\>](https://developer.apple.com/videos/play/wwdc2017/413/) 详细介绍了 `dyld` ，并给出了 APP 启动优化相关的建议。演讲者来自 *dyld Team* 。

## 术语

### Darwin 里的 dyld 的全称

> 一些博文在介绍 `dyld` 时写错了其英文全称，那个全称是 `FreeBSD` 上使用的，这里为了防止大家记成错的，就不提那个名称了。  
> 虽然 `Darwin` 的内核也用了 `FreeBSD` 的一部分，但对 `dyld` 的名称有自己的解释。

在终端使用 `man dyld` 查看文档，可看出其全称是 **the dynamic linker** ：

![man-dyld.jpg](/images/WWDC/Common/man-dyld.jpg)

### 启动时间（Startup Time）

> 说明：通常意义上的启动时间也包括 `main()` 之后的一部分时间，但本演讲只涉及 `main()` 之前的时间。

这个演讲的*启动时间*是指 `main()` 之前的时间，并讲解如何加快这个阶段的启动速度。

### 启动闭包（Launch Closure）

启动闭包指**启动一个 APP 所需要的所有信息**。比如：

- APP 使用了哪些 **dylib** 。
- APP 内使用的各种**符号（symbols）的偏移量（offsets）**。
- APP 的**代码签名（code signatures）**的位置。

## 回顾 WWDC16：优化 APP 启动时间的建议

![wwdc16-advice.jpeg](/images/WWDC/2017/413-App-Startup-Time-dyld/wwdc16-advice.jpeg)

### 减少启动阶段的任务

- 减少*嵌入的（Embed）* `dylib` 的数量。
- 减少声明的 `classes/methods` 的数量。
- 少使用 `initializers` 。

### 多使用 Swift

`Swift` 避免了 `C` , `C++` 和 `Objective-C` 的许多隐患。

- `Swift` 没有 `initializer` 。
- `Swift` 改善了大小。
- `Swift` 不允许存在*未对齐的（misaligned）*数据结构，未对齐的数据结构会在启动时消耗额外的时间。

相关 Session ：

- [WWDC16 - Session406: \<Optimizing App Startup Time\>](https://developer.apple.com/videos/play/wwdc2016/406/)
  - (链接已打不开，不知为何下掉了，但在 [wwdc.io](https://wwdc.io) 的 macOS 应用中可以查看)

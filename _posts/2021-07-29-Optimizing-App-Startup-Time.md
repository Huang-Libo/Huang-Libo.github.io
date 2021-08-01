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

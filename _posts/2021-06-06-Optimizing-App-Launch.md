---
title: "[WIP]【2019】优化 APP 启动"
categories: [攻城狮, WWDC]
tags: [WWDC 2019, iOS, APP Launch]
---

# 前言

[WWDC 2019 / 423 - Optimizing App Launch](https://developer.apple.com/videos/play/wwdc2019/423/) 介绍了 *APP* 启动的流程，以及测量和优化启动的方法，演讲者来自 *Performance Team* 。内容包含：  

- 什么是启动、启动的类型、启动的阶段；
- 如何正确地测量启动、测量时需要排除的干扰因素，以及要关注老设备的性能表现；
- 使用 *Instruments* 剖析启动的过程，使用 *XCTest* 做自动化的启动统计；
- 使用 *MetricKit* 和 *Xcode Organizer* 长期追踪启动数据。

# 什么是启动

## 暖场小故事

“全球的 *iOS* 设备*每天*要启动各种 *APP* 数十亿次。”  

如果每次启动能节省一毫秒，那么能节省 162 天（即所有 *iOS* 设备在一天内启动 *APP* 时能节省的总时间），这是将🚀发射到火星所需要的时间。  

他们计算的方式应该是这样的：  

```
14*1,000,000,000/1000/60/60/24 ≈ 162天
```

这个小故事和“全国每个人给我一块钱我就有 13 亿块钱了”有异曲同工之，哈哈。  

![](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-162-days-to-mars.jpg)
_把🚀送上火星需要162天_

## 为什么启动很重要

> App launch is a user experience interruption.

“*APP* 启动是用户体验的中断。”  

### 1. 第一印象

启动是用户对 *APP* 的第一印象，因此，它应该是令人愉悦的。  

另外，开发者倾向于关注更新的设备，但我们得保障使用旧机型的用户也有比较好的 *APP* 使用体验。  

### 2. 可以反映整体的代码质量

启动通常包含了大量的代码，包括 *primer coating*、*initialization*、*view creation* 等。

因此，如果启动的体验不好，就暗示着代码中其他模块的使用体验可能也不好。  

### 3. 性能

对于手机来说，启动是一个非常紧张的时刻。它包含了大量的 *CPU* 和 *memory* 的工作。  

因此，开发者应该尝试减少这部分的工作，因为它会影响*系统的性能*和*电池的消耗量*。  

## 启动的类型

![](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-type-1.jpg)

![](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-type-2.jpg)

- *cold launch*，冷启动
- *warm launch*，热启动
- *resume*，不算启动（*isn't quite a launch*）

## 目标

在**400毫秒**内展示第一帧。  

也就是说，在启动动画（*launch animation*）完成前，就应该完成 *UI* 的展示，当启动动画结束时，*APP* 就应该是可交互的（*interactive*）、可响应的（*responsive*）。  

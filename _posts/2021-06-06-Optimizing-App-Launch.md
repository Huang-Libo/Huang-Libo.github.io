---
title: "【WWDC19】优化 APP 启动"
categories: [攻城狮, WWDC]
tags: [WWDC19, iOS, APP 性能优化, APP 启动优化]
---

<p>
  <h2>目录</h2>
</p>

- [前言](#前言)
- [1. 什么是启动](#1-什么是启动)
  - [暖场小故事](#暖场小故事)
  - [为什么启动很重要](#为什么启动很重要)
    - [1. 第一印象](#1-第一印象)
    - [2. 可以反映整体的代码质量](#2-可以反映整体的代码质量)
    - [3. 性能](#3-性能)
  - [启动的类型](#启动的类型)
    - [1. 冷启动](#1-冷启动)
    - [2. 热启动](#2-热启动)
    - [3. Resume](#3-resume)
  - [不同启动类型的对比](#不同启动类型的对比)
  - [启动的目标耗时：400毫秒](#启动的目标耗时400毫秒)
  - [案例：Maps APP 的启动](#案例maps-app-的启动)
  - [启动的 6 个阶段](#启动的-6-个阶段)
    - [1. System Interface](#1-system-interface)
      - [1.1 dyld](#11-dyld)
      - [1.2 libSystemInit](#12-libsysteminit)
    - [2. Static Runtime Initialization](#2-static-runtime-initialization)
    - [3. UIKit Initialization](#3-uikit-initialization)
    - [4. Application Initialization](#4-application-initialization)
      - [APP lifecycle callbacks](#app-lifecycle-callbacks)
      - [UI lifecycle callbacks](#ui-lifecycle-callbacks)
    - [5. First Frame Render](#5-first-frame-render)
    - [6. Extended Phase](#6-extended-phase)
  - [系统侧的优化](#系统侧的优化)
- [2. 正确地测量启动](#2-正确地测量启动)
  - [控制变量](#控制变量)
  - [测量前的准备工作](#测量前的准备工作)
    - [1. 重启设备](#1-重启设备)
    - [2. 消除网络影响](#2-消除网络影响)
    - [3. 消除 iCloud 的影响](#3-消除-icloud-的影响)
    - [4. 使用 release build](#4-使用-release-build)
    - [5. 测量热启动](#5-测量热启动)
    - [6. 准备多份 mock 数据](#6-准备多份-mock-数据)
  - [选择测量的机器](#选择测量的机器)
  - [优化启动的步骤](#优化启动的步骤)
    - [1. Minimize : 减少与启动无关的任务](#1-minimize--减少与启动无关的任务)
    - [2. Prioritize : 确定任务的优先顺序](#2-prioritize--确定任务的优先顺序)
    - [3. Optimize : 优化现有的任务](#3-optimize--优化现有的任务)
- [3. Demo：测量启动的工具](#3-demo测量启动的工具)
  - [使用 Instrument - App Launch Template](#使用-instrument---app-launch-template)
    - [APP 生命周期（APP life cycle）](#app-生命周期app-life-cycle)
    - [线程状态（Thread State）](#线程状态thread-state)
    - [查看启动阶段的详情](#查看启动阶段的详情)
    - [CPU clock & Wall clock](#cpu-clock--wall-clock)
    - [线程的优先级](#线程的优先级)
    - [线程优先级反转的问题](#线程优先级反转的问题)
    - [代码分析](#代码分析)
  - [使用 XCTest 测量启动](#使用-xctest-测量启动)
  - [长期追踪启动数据](#长期追踪启动数据)
    - [1. 使用 Xcode Organizer 收集启动数据](#1-使用-xcode-organizer-收集启动数据)
    - [2. 使用 MetricKit 收集更详细的统计数据](#2-使用-metrickit-收集更详细的统计数据)
- [总结](#总结)
- [相关文章](#相关文章)

## 前言

[WWDC 2019 / 423 - Optimizing App Launch](https://developer.apple.com/videos/play/wwdc2019/423/) 介绍了 *APP* 启动的流程，以及测量和优化启动的方法，演讲者来自 *Performance Team* 。内容包含：  

- 什么是启动、启动的类型、启动的阶段；
- 如何正确地测量启动、测量时需要排除的干扰因素，以及要关注老设备的性能表现；
- 使用 *Instruments* 剖析启动的过程，使用 *XCTest* 做自动化的启动统计；
- 使用 *MetricKit* 和 *Xcode Organizer* 长期追踪启动数据。

相关资源：笔者整理过的[讲稿（视频的字幕）](https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2019/423-Optimizing-App-Launch/Optimizing-App-Launch-Edited.md)。

## 1. 什么是启动

### 暖场小故事

“全球的 *iOS* 设备*每天*要启动各种 *APP* 数十亿次。”  

如果每次启动能节省一毫秒，那么能节省 162 天（即所有 *iOS* 设备在一天内启动 *APP* 可节省的总时间），这是将🚀发射到火星所需要的时间。  

他们计算的方式应该是这样的：  

```plaintext
14*1,000,000,000/1000/60/60/24 ≈ 162天
```

这个小故事和“全国每个人给我一块钱我就有 13 亿块钱了”有异曲同工之妙，哈哈。  

![APP-launch-162-days-to-mars](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-162-days-to-mars.jpg)
_把🚀送上火星需要162天_

### 为什么启动很重要

> App launch is a user experience interruption.

“*APP* 启动是用户体验的中断。”  

#### 1. 第一印象

启动是用户对 *APP* 的第一印象，因此，它应该是令人愉悦的。  

另外，开发者倾向于关注更新的设备，但我们得保障使用旧机型的用户也有比较好的 *APP* 使用体验。  

#### 2. 可以反映整体的代码质量

启动通常包含了大量的代码，包括 *primer coating*（？？）、*initialization*、*view creation* 等。

因此，如果启动的体验不好，就暗示着代码中其他模块的使用体验可能也不好。  

#### 3. 性能

对于手机来说，启动是一个非常紧张的时刻。它包含了大量的 *CPU* 和*内存*的工作。  

因此，开发者应该尝试减少这部分的工作，因为它会影响*系统的性能*和*电池的消耗量*。  

### 启动的类型

![APP-launch-type-1](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-type-1.jpg)

#### 1. 冷启动

冷启动（*cold launch*）发生在重启后（或者 *APP* 很长时间没有被启动过）。

在这种情况下，如果要启动一个 *APP* ，需要：

- 将它从*磁盘*加载到*内存*；
- 启动支撑 *APP* 运行的系统侧服务（*system-side services*）；
- 创建 *APP* 的进程。

#### 2. 热启动

如果 *APP* 最近启动过，下次启动将会是热启动（*warm launch*）。  

**热启动也需要创建进程**，但此时 *APP* 的一部分已经在*内存*中，一些系统侧的服务也已经启动了。  

因此，热启动要比冷启动要快一些。  

#### 3. Resume

如果 *APP* 的进程还存在，当用户从 *Home screen* 或者 *APP switcher* 重新进入 *APP* 时，就是 *resume*。  

*resume* **不算启动**（*isn't quite a launch*），APP 在之前已经完成了启动。所以 *resume* 的速度很快。  

因此要注意：在做启动测量时，不要把启动和 *resume* 混淆了。

### 不同启动类型的对比

![APP-launch-type-2](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-type-2.jpg)

简单翻译一下：  

| 冷启动 | 热启动 | Resume |
|:---:|:---:|:---:|
| 重启后 | 最近被终止 | APP 被挂起 |
| APP 不在内存中 | APP 部分在内存中 | APP 全部在内存中 |
| 进程不存在 | 进程不存在 | 进程存在 |

### 启动的目标耗时：400毫秒

在 **400 毫秒**内展示第一帧。  

也就是说，在启动动画（*launch animation*）完成前，就应该完成 *UI* 的展示；当启动动画结束后，*APP* 就应该是可交互的（*interactive*）、可响应的（*responsive*）。  

### 案例：Maps APP 的启动

用户点击了 *Maps* 的图标，系统开始执行启动：  

![APP-launch-phases-Maps-1](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-Maps-1.jpg){: .normal width="600"}

前 **100 毫秒**（紫色部分），*iOS* 系统会做初始化 *APP* 所需的系统侧工作：  

![APP-launch-phases-Maps-2](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-Maps-2.jpg){: .normal width="600"}

这给了开发者大约 **300 毫秒**（绿色部分）的时间去创建视图、加载内容、生成 *APP* 的第一帧，也就是说，在 **400 毫秒**内完成启动并展示第一帧。  

第一帧不需要是全部完成的状态，可以为异步加载的数据展示 *placeholder* ，但 *APP* 此时应该是可响应、可交互的。这个阶段完成后，用户可以搜索和查看自己的收藏，等等：  

![APP-launch-phases-Maps-3](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-Maps-3.jpg){: .normal width="600"}

在接下来的几百毫秒里，当异步数据加载完成后，显示最终的页面：  

![APP-launch-phases-Maps-4](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-Maps-4.jpg){: .normal width="600"}

### 启动的 6 个阶段

![APP-launch-phases-overall](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-overall.jpg)

#### 1. System Interface

![APP-launch-phases-1](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-1.jpg)

##### 1.1 dyld

*System Interface* 的前半部分是 **dyld（the dynamic linker）**阶段，它的作用是加载 *shared libraries* 和 *frameworks* 。

查看 `dyld` 的文档：  

```bash
man dyld
```

输出：  

![man-dyld.jpg](/images/WWDC/common/man-dyld.jpg)

在 *WWDC17*，*Apple* 推出了 *dyld 3*，为系统添加了令人兴奋的优化。  

- 2017 年，*iOS 11* 的系统自带 APP 开始使用 *dyld 3* ，第三方 APP 还是使用 *dyld 2.x* 。
- 2019 年，*iOS 13* 的系统自动 APP 和第三方 APP 都使用 *dyld 3* 。

在使用 *dyld 3* 后，系统会*缓存运行时依赖项 (caching runtime dependencies)* ，这给*热启动*带来显著的速度改进。  

相关 *Session* ：

- [WWDC17 / 413 - App Startup Time: Past, Present, and Future](https://developer.apple.com/videos/play/wwdc2017/413/)

在 *dyld 3* 阶段的几个建议：  
  
- ✅ 避免链接未用到的 *frameworks* 。
- ✅ 避免在启动时加载动态库，比如 `dlopen` 和 `NSBundle load`。因为这样就失去了 *dyld 3 缓存运行时依赖项*带来的好处。
- ✅ （根据上一条）最后，这意味着要*硬链接 (Hard link)* 所有的依赖，它比以前更快。

##### 1.2 libSystemInit

*System Interface* 的后半部分是 *libSystemInit* 。在这个阶段，系统初始化 APP 内用到的*底层系统组件（low-level system components）*。  

这主要是具有*固定时间开销*的系统端工作，*APP* 开发者不用太关心。  

#### 2. Static Runtime Initialization

![APP-launch-phases-2](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-2.jpg)

这是系统初始化 *APP* 的 **Objective-C 和 Swift 的 Runtime** 的时候。  

一般而言，*APP* 在这个阶段不会做任何工作，除非 *APP* 中包含*静态初始化方法（static initializer method）* ，可能是 *APP* 代码中的，也可能是 *APP* 链接的 *framework* 中的。  

**不建议使用静态初始化方法**。  

下面给两个建议：  

- ✅ *Framework* 的开发者应该考虑暴露初始化的 *API* ，避免使用静态初始化。  
- ✅ 如果一定要使用静态初始化，考虑把 `+load` 方法中的代码移到 `+initialize` 方法中。因为 `+load` 方法在每次启动 APP 的时候就会调用，而 `+initialize` 方法会在第一次使用这个*类*的时候调用。  

#### 3. UIKit Initialization

![APP-launch-phases-3](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-3.jpg)

这是系统初始化 `UIApplication` 和 `UIApplicationDelegate` 的阶段。  

在大多数情况下，这是系统端的工作，设置*事件处理（event processing）*以及与系统集成。

但是如果开发者子类化了 `UIApplication` ，或者在 `UIApplicationDelegate` 的初始化方法中做了额外的工作，将会影响这阶段的耗时。  

因此，如果做了上述定制化的事情，确保以下两点：  

- ✅ 减少 `UIApplication` 子类中的工作。
- ✅ 减少 `UIApplicationDelegate` 初始化方法中的工作。

#### 4. Application Initialization

![APP-launch-phases-4](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-4.jpg)

*Application* 初始化阶段是开发者对启动时间影响最大的阶段。

##### APP lifecycle callbacks

无论是否使用了 `UIScene` 相关的 *API* ，都会先调用 `UIApplicationDelegate` 的 *APP lifecycle callbacks* ：  

```objc
application:willFinishLaunchingWithOptions:
application:didFinishLaunchingWithOptions:
```

##### UI lifecycle callbacks

- 如果没有使用 `UIScene` ，当 *UI* 展示给用户的时候，再调用 `UIApplicationDelegate` 的 *UI lifecycle callbacks* ：  

```objc
applicationDidBecomeActive:
```

- 如果使用了 *iOS 13* 推出的 `UIScene` ，当 *UI* 展现给用户时，会为每个 *scene* 调用 `UISceneDelegate` 的 *UI lifecycle callbacks* ：  

```objc
scene:willConnectToSession:options:
sceneWillEnterForeground:
sceneDidBecomeActive:
```

*Application Initialization* 阶段的建议：  

- ✅ 推迟与 APP 启动无关的工作，或放在子线程执行
- ✅ 如果使用了 `UIScene` ，请在 scene 之间共享资源，避免做重复的工作。

`UIScene` 相关介绍请查看：

- [Apple Article - Managing Your App's Life Cycle](https://developer.apple.com/documentation/uikit/app_and_environment/managing_your_app_s_life_cycle?language=objc)
- [WWDC 2019 / 212 - Introducing Multiple Windows on iPad](https://developer.apple.com/videos/play/wwdc2019/212)

#### 5. First Frame Render

![APP-launch-phases-5](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-5.jpg)

在这个阶段的任务是创建 view 、布局 view 、最后绘制 view 。  

当系统拿到 view 的这些信息，就执行 `commit` 和 `render` ，完成第一帧的展示。  

在这个阶段，会调用这些方法：  

```objc
loadView
viewDidLoad
layoutSubviews
```

建议有：  

- ✅ 减少 view 的层级、懒加载不需要在启动时展示的 view 。
- ✅ 优化 auto layout ，减少不必要的约束。

相关 *Session* :  

- [WWDC 2018 / 220 - High Performance Auto Layout](https://developer.apple.com/videos/play/wwdc2018/220/)

#### 6. Extended Phase

![APP-launch-phases-6](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-phases-6.jpg)

- App-specific period after first frame
- Displays asynchronously loaded data
- App should be interactive and responsive

在这个阶段可异步加载数据，加载完成后，给用户展示最终的页面。但是，在进入这个阶段之前，APP 就应该是*可交互、可响应*的，不能因为数据的加载而阻断 UI 交互。

建议：  

- ✅ 使用 `os_signpost` 来测量这阶段的工作

### 系统侧的优化

*iOS 13* 做了一系列优化，其中一些优化可显著地加快启动速度，如：引入 dyld 3 、缓存 APP 的运行时依赖项、Auto Layout 的优化、Scheduler 的优化、Runtime 的优化，等等。

开发者只需要稍作调整，甚至都不需要做任何事情，启动就会变得更快。

![APP-launch-System-Improvements](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-System-Improvements.jpg)

## 2. 正确地测量启动

### 控制变量

![APP-launch-measure-control-variables](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-measure-control-variables.png)
_测量时要控制变量_

在任何时候，iOS 设备都处于各种不同的状态和条件下，这可能会给启动带来很大的变化。  

所以，当我们分析和比较启动数据时，要控制变量、消除不可控因素，保障启动的*可预测性*和*一致性*。比如，消除*网络的干扰*和*后台进程的干扰*。  

这听起来违反直觉，因为这样启动并不是常规的使用方式，但这是没关系的，在测量启动数据时，更重要的是取得一致的结果，以此来评估进展。  

### 测量前的准备工作

在测量前，需要设置干净和一致的环境，下面给出一些建议。

#### 1. 重启设备

重启设备将清除任何不必要的状态，然后让它在接下来的几分钟内稳定下来，以使系统的启动全部完成。  

#### 2. 消除网络影响

每次网络请求获得的数据量大小可能差别很大，要消除这些变化带来的影响，可以使用飞行模式，或者使用 mock 数据。  

#### 3. 消除 iCloud 的影响

使用同一个 iCloud 账号，并保持其数据内容不变，或者直接退出 iCloud 账号，以消除 iCloud 后台进程对测量的干扰。  

#### 4. 使用 release build

使用 *release build* 可在测量过程中减少不必要的*调试代码*带来的开销，且可以获得编译时的优化。

#### 5. 测量热启动

应该使用*热启动*进行测量，其测量结果是更加一致的（ *more consistent* ），因为 APP 的一部分可能已经在*内存*中，并且一些*系统端服务*可能已经在运行。  

#### 6. 准备多份 mock 数据

开发者可能需要准备多份 mock 数据，来模拟**不同数据规模**对启动的影响。最好做到只加载显示第一帧所需的数据。  

### 选择测量的机器

![APP-launch-older-and-newer-devices](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-older-and-newer-devices.jpg){: .normal width="600"}

在选择要测量机器时，请确保包含*最老支持的系统版本中的最老的设备*。以保障所有的用户都能有较好的 APP 启动体验。  

### 优化启动的步骤

![APP-launch-optimize-steps](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-optimize-steps.jpg)

#### 1. Minimize : 减少与启动无关的任务

- ✅ 延迟执行与展示第一帧不相关的任务
- ✅ 避免阻塞主线程
- ✅ 降低内存的使用

#### 2. Prioritize : 确定任务的优先顺序

- ✅ 为任务确定正确的 *QoS*
- ✅ 利用 *scheduler* 为 APP 启动做的优化（ Utilize *scheduler* optimizations for app launch ）
- ✅ 使用正确的原语保持优先级（ Preserve the priority with the right primitives ）

相关 *Session* :  

- [WWDC17 / 706 - Modernizing Grand Central Dispatch Usage](https://developer.apple.com/videos/play/wwdc2017/706/)

#### 3. Optimize : 优化现有的任务

- ✅ 简化或限制当前任务
- ✅ 优化算法和数据结构
- ✅ 缓存资源和计算结果

## 3. Demo：测量启动的工具

> 演讲人用一个 Demo 项目演示了 *App Launch Template* 的使用方法，这里只做个要点记录，代码详情可去看原视频。  

在测量前，先执行 Profile ( <kbd>cmd</kbd> + <kbd>I</kbd> ) ，此时 Xcode 会以 *Release* 模式重新编译 APP ，编译和安装后，会自动打开 Instrument 。  

### 使用 Instrument - App Launch Template

在 *Xcode 11* 中新增了 *App Launch Template* ，专门用于测量 APP 的启动：  

![APP-launch-instrument-1](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-instrument-1.jpg)

#### APP 生命周期（APP life cycle）

- **紫色部分**发生在 `main` 方法调用前。
- **绿色部分**发生在 `main` 方法调用后的早期阶段，APP 在这个阶段完成启动并绘制第一帧。
- **蓝色部分**对应 *Extended phase* ，做一些异步的任务，拿到数据后再绘制最终的视图。

#### 线程状态（Thread State）

- **灰色**表示线程**被阻塞了（Blocked）**，意味着线程没有做任何工作。
- **红色**表示线程是**可运行的（Runnable）**，意味着有工作计划要完成，但缺乏CPU资源。
- **橙色**表示线程**被抢占了（Preempted）**，也就是说它正在做某项工作，但是被其他具有更高优先级的竞争性任务打断了。
- **蓝色**表示线程**正在运行（Running）**，也就是说它正在使用 CPU 执行任务。

#### 查看启动阶段的详情

**三击**一个启动阶段，可以突出显示该阶段并获得详细信息：  

![APP-launch-instrument-2](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-instrument-2.jpg)

- 在左边，可以看到这段时间内所有任务的详细*堆栈*跟踪。
- 在右边，可以看到一个聚合的*堆栈*跟踪，它列出了按 CPU 样本大小的数量排序的所有*符号*（ all of the symbols ordered by the number of CPU sample size ）。

#### CPU clock & Wall clock

在示例 demo 的 *system interfaces* 阶段， *wall clock*（挂钟，即现实世界的时间）显示有 **149ms**，但 *CPU clock*（CPU 时钟）只有 **6ms**。  

- Wall clock：  

![App-launch-wall-clock.jpg](/images/WWDC/2019/423-Optimizing-App-Launch/App-launch-wall-clock.jpg)

- CPU clock：  

![App-launch-CPU-clock.jpg](/images/WWDC/2019/423-Optimizing-App-Launch/App-launch-CPU-clock.jpg)

这个差异来自测量机制本身的耗时，CPU clock 的数据剔除了这部分时间，因此，我们应该以调用栈中显示的 CPU clock 耗时为准。  

#### 线程的优先级

- Priority 47 ：user interactive QoS ，主线程使用此 QoS 。
- Priority 4 ：background QoS 。
- ... (还有一些其他的 QoS 这里没写)

#### 线程优先级反转的问题

从下图可以看出，主线程被阻塞了（灰色），而子线程有很多任务需要运行，但是缺乏 CPU 资源（红色）。

将菜单切换到 `Events: Thread States` ，可以查看线程的状态：

![App-launch-thread-states-1.jpg](/images/WWDC/2019/423-Optimizing-App-Launch/App-launch-thread-states-1.jpg)

线程的状态：  

![App-launch-thread-states-2.jpg](/images/WWDC/2019/423-Optimizing-App-Launch/App-launch-thread-states-2.jpg)

主线程的状态中有这样一行：

```plaintext
make runnable by __thread_selfid 0x12253 running on Core 1
```

说明主线程是被这个子线程设置为 runnable 的，因此，主线程的阻塞是这个子线程造成的。

子线程的信息：  

- 名称：`__thread_selfid`
- 地址：`0x12253`

点击子线程，可以看到，它的 QoS 是 4 ，也就是 background QoS ：

![App-launch-thread-states-3.jpg](/images/WWDC/2019/423-Optimizing-App-Launch/App-launch-thread-states-3.jpg)

显然，这里存在*优先级反转（priority inversion）*的问题，也就是说，高优先级、高 QoS 的线程，被低优先级、低 QoS 的线程阻塞了。

#### 代码分析

在 `StarDataProvider` 类中，创建了一个串行队列，QoS 是 `.background` 。有两个加载数据的方法：

- `loadDataAsync` ：异步地加载数据
- `loadDataSync` ：同步地加载数据

![APP-launch-priority-inversion-1](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-priority-inversion-1.jpg)

在 `AppDelegate` 中，调用了 `loadDataAsync` 方法异步加载数据。由于这个 APP 的需求是拿到数据后才能执行后续任务，因此又使用**信号量**来阻塞主线程，直到数据加载任务完成后，才继续执行后续任务：  

![APP-launch-priority-inversion-2](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-priority-inversion-2.jpg)  

乍一看没问题，但是要注意，该子线程的 priority 是 4 ，如果有另一个 priority 更高（比如 20）的线程 A 在执行耗 CPU 的任务，那么会导致 priority 为 4 的线程缺乏 CPU 资源去执行任务，只能等线程 A 完成任务腾出 CPU 资源，才能继续执行。**这导致了主线程要等待不相干的线程 A 的任务完成**，实际上这完全是没必要的。  

这里应该使用 `loadDataSync` 方法：

![APP-launch-priority-inversion-3](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-priority-inversion-3.jpg)

**使用 sync 方法后，GCD 会暂时将主线程的优先级（priority=47）传播给这个子线程**，主线程会等子线程完成任务后再恢复到活跃状态，不会被 priority 为 20 的线程 A 所影响，优先级反转问题迎刃而解。

### 使用 XCTest 测量启动

开发者可以利用 *Xcode 11* 中新增的 `XCTest` API 来测量 APP 的启动性能。只需几行代码，*Xcode* 就可以反复启动 APP（默认值是 5 次），然后给出 APP 启动的统计数据。  

XCTest 会做一个*抛弃型的登录（throwaway launch）*，这将消除冷启动带来的差异。

![APP-launch-XCTest](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-XCTest.jpg)

XCTest 的测量结果中会显示*平均启动时间*：  

![APP-launch-demo-optimize-result](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-demo-optimize-result.jpg)

### 长期追踪启动数据

- 开发时就要关注 APP 的性能（**Make performance a development-time priority**）
- 为启动时间设置一个目标值，并用图表展示版本迭代过程中 APP 启动时间的走势

![APP-launch-plot](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-plot.jpg){: .normal width="400"}

#### 1. 使用 Xcode Organizer 收集启动数据

在 *iOS 13* 中，对于选择加入的用户，APP 的性能指标将被收集。

然后，它们将在 24 小时内汇总，并发送给 Xcode Organizer ，开发者可以通过 *APP version* 和 *device version* 的直方图查看这些数据。

![APP-launch-Xcode-Organizer](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-Xcode-Organizer.jpg)

相关 *Session* ：  

- [WWDC 2019 / 417 - Improving Battery Life and Performance](https://developer.apple.com/videos/play/wwdc2019/417/)

#### 2. 使用 MetricKit 收集更详细的统计数据

- 收集自定义的能耗和性能指标
- 汇总结果每 24 小时交付一次

## 总结

- 理解启动的各个阶段
- 使用工具去测量启动的性能，而不是估算
- 在开发的各个阶段都要追踪 APP 的性能

## 相关文章

- <https://xiaozhuanlan.com/topic/4690823715>

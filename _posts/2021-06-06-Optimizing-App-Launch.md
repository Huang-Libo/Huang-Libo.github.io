---
title: "[WIP]【2019】优化 APP 启动"
categories: [攻城狮, WWDC]
tags: [WWDC 2019, iOS, APP 性能优化, APP 启动优化]
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
      - [dyld](#dyld)
      - [libSystemInit](#libsysteminit)
    - [2. Static Runtime Initialization](#2-static-runtime-initialization)
    - [3. UIKit Initialization](#3-uikit-initialization)
    - [4. Application Initialization](#4-application-initialization)
      - [APP Lifecycle Callbacks](#app-lifecycle-callbacks)
      - [UI lifecycle callbacks](#ui-lifecycle-callbacks)
    - [5. First Frame Render](#5-first-frame-render)
    - [6. Extended Phase](#6-extended-phase)
- [2. 正确地地测量 launch](#2-正确地地测量-launch)
  - [控制变量](#控制变量)
  - [测量前的准备工作](#测量前的准备工作)
    - [1. 重启设备](#1-重启设备)
    - [2. 消除网络影响](#2-消除网络影响)
    - [3. 消除 iCloud 的影响](#3-消除-icloud-的影响)
    - [4. 使用 release build](#4-使用-release-build)
    - [5. 测量热启动](#5-测量热启动)
    - [6. 准备多份 mock 数据](#6-准备多份-mock-数据)
  - [选择测量的机器](#选择测量的机器)
  - [使用 XCTest 测量启动](#使用-xctest-测量启动)

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

##### dyld

*System Interface* 的前半部分是 *dyld（the dynamic link editor）*阶段，它的作用是加载 *shared libraries* 和 *frameworks* 。

在 *WWDC 2017*，*Apple* 推出了 *dyld3*，为系统添加了令人兴奋的优化。但直到 2019 年，*iOS 13* 才开始使用 *dyld3* 。  

在使用 *dyld3* 后，系统会*缓存运行时依赖项（caching runtime dependencies）*，这给*热启动*带来显著的速度改进。  

相关 *Session* ：

- [WWDC 2017 / 413 - App Startup Time: Past, Present, and Future](https://developer.apple.com/videos/play/wwdc2017/413/)

在 *dyld3* 阶段的几个建议：  
  
- ✅ 避免链接未用到的 *frameworks* 。
- ✅ 避免在启动时加载动态库，比如 `dlopen` 和 `NSBundle load`。因为这样就失去了 *dyld3 缓存运行时依赖项*带来的好处。
- ✅ 最后，（根据上一条）这意味着要*硬链接（Hard link）*所有的依赖，它比以前更快。

##### libSystemInit

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

##### APP Lifecycle Callbacks

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

## 2. 正确地地测量 launch

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

### 使用 XCTest 测量启动

开发者可以利用 *Xcode 11* 中新增的 `XCTest` API 来测量 APP 的启动性能。只需几行代码，*Xcode* 就可以反复启动 APP ，然后给出 APP 启动的统计数据。  

![APP-launch-XCTest](/images/WWDC/2019/423-Optimizing-App-Launch/APP-launch-XCTest.jpg)

相关 *Session* ：  

[WWDC 2019 / 417 - Improving Battery Life and Performance](https://developer.apple.com/videos/play/wwdc2019/417/)

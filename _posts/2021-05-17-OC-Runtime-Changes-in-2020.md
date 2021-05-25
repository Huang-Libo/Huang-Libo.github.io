---
title: "【iOS 14】Objective-C Runtime 的优化"
categories: [攻城狮, WWDC]
tags: [WWDC, iOS, Objective-C Runtime, class_rw_ext_t, Reletive Method List, Tagged Pointer]
---

# 前言

[WWDC 2020 / 10163](https://developer.apple.com/videos/play/wwdc2020/10163/) 介绍了 *2020* 年 *Objective-C Runtime* 的一些优化，演讲者来自 *Languages and Runtimes team* 。内容包含：  

- *Class Data Structure* 的优化：从 `class_rw_t` 中拆分出一个新类型 `class_rw_ext_t` ；
- 在引入的 *Binary Image* 中使用 **Reletive Method Lists** ；
- *ARM64* 架构上 **Tagged Pointer** 格式的变化。

经过优化后：

- **less memory usage**（减少了内存的使用）
- **smaller binaries**（二进制包更小）

由于疫情影响，*WWDC 2020* 完全是线上举办的，官网上也没有给出演示 *PDF* ，只有[讲稿（即字幕）](https://github.com/Bob-Playground/WWDC-Stuff/blob/master/WWDC-2020/10163-OC-Runtime-Changes/Transcript-Edited.md)。

# 新特性的适配

> With any luck, you won’t need to do anything and your apps will just get faster.  

“运气好的话，你不需要做任何事情，你的应用程序就会变得更快。”  

如果代码中没有依赖“隐藏”在 *Runtime* 中的内部类型（ *internal data structures* ），比如 `class_rw_t`，或直接读取 **Tagged Pointer** 的 *bit* 数据，那就不需要做改动。  

但如果 *APP* 中直接使用了这些“隐藏”的内容，*APP* 在新系统上可能会出现异常。  

这些优化将在下列系统中生效：  

- *macOS Big Sur ( macOS 11 )*
- *iOS 14*
- *tvOS 14*
- *watchOS 7*

# 1. 新增类型：class_rw_ext_t

在 *iOS 14* 的 *Class Data Structure* 中新增了 `class_rw_ext_t` 类型。  

我们先看看 *iOS 13* 上的 *Runtime* 是如何运行的。  

## class_ro_t

类对象( *class object* )中包含了我们最常访问的信息：指向元类的指针( *metaclass* )，指向父类的指针( *superclass* )，以及方法缓存( *method cache* )。  

类对象还有一个指针，指向存有更多信息的类型，叫做 `class_ro_t` ，其中的 **ro** 代表 **read only** 。`class_ro_t` 存储着**类的名称**，以及**实例方法列表**、**协议**、**属性**、**实例变量**等信息。  

*类对象*和 `class_ro_t` 的示意图：  

![](/images/2021/runtime-class_ro_t.jpg)

*Swift 类* 和 *Objective-C 类* 共用这个基础设施( *infrastructure* )，因此每个 *Swift 类* 也有这些结构。  

当*类*第一次从 *disk* 加载到 *memory* 时，他们就是以这种方式展现的，但是当*类*被**使用**时会发生改变。

## Clean memory & Dirty memory

为了理解后续要发生的事，需要了解 *clean memory* 和 *dirty memory* 的区别。

Clean Memory
: *Clean memory* 是加载后就不会改变的 memory ，比如 `class_ro_t` ，因为它是 *read only* 的。
: *Clean memory* 可以从 *memory* 中清除、为其他内容腾出空间。因为如果需要它，系统总是可以从 *disk* 重新加载它。

Dirty Memory
: *Dirty memory* 是当*进程*运行的时候会被改变的 memory ，比如*类对象*。因为 *Runtime* 会为*类对象*创建一个全新的 *method cache* ，并让*类对象*内的一个指针指向 *method cache* 。
: *Dirty memory* 比 *Clean memory* 昂贵得多。它必须在*进程*运行期间一直存在。



*macOS* 可以把 *memory* 内的 *Dirty Memory* 转移至 *swap（交换分区）；而* *iOS* 不使用 *swap* ，所以在 *iOS* 中 *Dirty Memory* 很昂贵。  

## class_rw_t

当*类***第一次使用时**，*Runtime* 创建了一个额外的类型，叫做 `class_rw_t` ，**rw** 代表 **read/write**（可读可写）。  

![](/images/2021/runtime-class_rw_t.jpg)
_绿色部分是 dirty memory ，蓝色部分是 clean memory 。_

在 `class_rw_t` 中，存储在 *Runtime* 期间生成的新信息。  

例如，所有*类*都使用 *First Subclass* 和 *Next Sibling Class* 指针链接到一个树结构中，这允许运行时遍历当前使用的所有类，这对于使方法缓存失效是很有用的。（**这个地方的细节不太了解，作者也没细讲，待研究**）  

`class_ro_t` 中已经有了 *Methods* 和 *Properties* ，为什么又在新创建的 `class_rw_t` 中添加它们呢？  

原因是 *Methods* 和 *Properties* 可以在运行时被改变。  

比如，当一个分类被加载时，它可以向原类中添加新方法。程序员可以使用 *Runtime API* 动态地添加它们。  

## 从 class_rw_t 中拆分出 class_rw_ext_t

> We measured about 30 megabytes of these class_rw_t structures across the system on an iPhone.  

“我们在 iPhone 系统中检测到了大约 **30MB** 的 `class_rw_t` 结构。”  

由于 `class_rw_t` 在运行时会被改变，因此它是 *dirty memory* 。在我们的设备上有很多*类*在被使用中，它们占用了相当多的内存。  

那我们怎样把它们缩小呢？

还记得吗，我们在“读/写部分”需要这些东西，是因为它们可以在运行时更改。

> But examining usage on real devices, we found that only around 10% of classes ever actually have their methods changed.  

“但是通过对实际设备的使用进行分析，我们发现只有大约 **10%** 的*类*真正改变了它们的 *Methods* 。”  

并且 `demangled` 只被 *Swift class* 使用了，并且只有当 *Swift class* 需要知道它们对应的 *Objective-C* 名称时，才需要用到它。

因此，我们可以把 `class_rw_t` 拆成两部分，将不常使用的部分拆分出去、放在名为 `class_rw_ext_t` 的新结构中。  

对于确实需要*附加信息*的类，我们才创建 `class_rw_ext_t` 、并将其放入类中使用。  

![](/images/2021/runtime-class_rw_ext_t-1.jpg)

![](/images/2021/runtime-class_rw_ext_t-2.jpg)

> Approximately 90% of classes never need this extended data, saving around 14 megabytes system wide.  

“大约 90% 的类从不需要这种扩展数据，在系统范围内节省了大约 **14MB** 内存。”  

## 实例

使用 `heap` 命令查看 *macOS Big Sur* 应用中 `class_rw_t` 和 `class_rw_ext_t` 的使用情况。    

### macOS 的 Mail

```bash
heap Mail | egrep 'class_rw|COUNT'
```

![](/images/2021/runtime-class_rw_ext_t-heap-Mail.jpg)

*Mail* 中只有约 **10%** 的类使用了 `class_rw_ext_t` 。

### macOS 版的 WeChat

```bash
heap WeChat | egrep 'class_rw|COUNT'
```

![](/images/2021/runtime-class_rw_ext_t-heap-WeChat.jpg)

*WeChat* 中只有约 **8%** 的类使用了 `class_rw_ext_t` 。

## 使用 Runtime APIs

我们应该使用 *Runtime* 暴露出来的 *API* ，苹果保障它们的稳定性。比如：  

- `class_getName`
- `class_getSuperclass`
- `class_copyMethodList`
- ...

而不要在代码中直接使用 *Runtime* 的这些私有类型，否则新系统更新了这些类型的 *layout*，代码就无法正常运行。  

![](/images/2021/runtime-class_rw_ext_t-error.jpg)

比如，代码直接去 `class_rw_t` 中读取 *Methods* ，在 *iOS 13* 上是可行的，但在 *iOS 14* 上 `class_rw_t` 中已经没有了 *Methods* ，它被挪动到了 `class_rw_ext_t` 中。  
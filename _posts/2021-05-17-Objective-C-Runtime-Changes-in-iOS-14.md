---
title: "【iOS 14】Objective-C Runtime 的优化"
categories: [攻城狮, WWDC]
tags: [WWDC 2020, iOS, Objective-C Runtime, class_rw_ext_t, Reletive Method List, Tagged Pointer]
---

<p>
  <h2>目录</h2>
</p>

- [前言](#前言)
- [新特性的适配](#新特性的适配)
- [1. 新增类型：class_rw_ext_t](#1-新增类型class_rw_ext_t)
  - [class_ro_t](#class_ro_t)
  - [Clean memory & Dirty memory](#clean-memory--dirty-memory)
  - [class_rw_t](#class_rw_t)
  - [从 class_rw_t 中拆分出 class_rw_ext_t](#从-class_rw_t-中拆分出-class_rw_ext_t)
  - [实例](#实例)
    - [macOS 的 Mail](#macos-的-mail)
    - [macOS 版的 WeChat](#macos-版的-wechat)
  - [使用 Runtime APIs](#使用-runtime-apis)
- [2. 相对方法列表（ Reletive Method Lists ）](#2-相对方法列表-reletive-method-lists-)
  - [Objective-C 方法的 3 个部分](#objective-c-方法的-3-个部分)
  - [以 init 方法为例](#以-init-方法为例)
  - [进程中内存的划分](#进程中内存的划分)
  - [使用普通的方法列表( Method Lists )](#使用普通的方法列表-method-lists-)
  - [使用相对方法列表( Reletive Method Lists )](#使用相对方法列表-reletive-method-lists-)
  - [Swizzling relative method lists](#swizzling-relative-method-lists)
  - [Deployment target](#deployment-target)
  - [Mismatched deployment targets](#mismatched-deployment-targets)
  - [使用 Runtime APIs](#使用-runtime-apis-1)
- [3. ARM64 架构上 Tagged Pointer 格式的变化](#3-arm64-架构上-tagged-pointer-格式的变化)
  - [普通的对象指针](#普通的对象指针)
  - [Intel](#intel)
    - [混淆 tagged pointer 的值](#混淆-tagged-pointer-的值)
    - [tag number 和 payload](#tag-number-和-payload)
    - [extended tag](#extended-tag)
    - [Swift 中的 tagged pointer](#swift-中的-tagged-pointer)
  - [ARM64](#arm64)
    - [iOS 13 中 tagged pointer 的格式](#ios-13-中-tagged-pointer-的格式)
    - [iOS 14 中 tagged pointer 格式的变化](#ios-14-中-tagged-pointer-格式的变化)
  - [使用 APIs](#使用-apis)
- [小结](#小结)
  - [相关资料](#相关资料)
- [Reference](#reference)

## 前言

[WWDC 2020 / 10163 - Advancements in the Objective-C runtime](https://developer.apple.com/videos/play/wwdc2020/10163/) 介绍了 *2020* 年 *Objective-C Runtime* 的一些优化，演讲者来自 *Languages and Runtimes Team* 。内容包含：  

1. *Class Data Structure* 的优化：从 `class_rw_t` 中拆分出一个名为 `class_rw_ext_t` 的新类型；
2. 在*二进制映像( Binary Image )* 中使用**相对方法列表( Reletive Method Lists )** ；
3.  *ARM64* 架构上 **Tagged Pointer** 格式的变化。

经过优化后：

- *Better performance and lower memory usage*（更好的性能和更少的内存使用）
- *Smaller binaries*（二进制包更小）
- *ARM64* 架构上的 *Tagged Pointer* 可以存放普通的*对象指针*了。

由于疫情影响，*WWDC 2020* 完全是线上举办的，官网上也没有给出演示 *PDF* ，只有[讲稿（即字幕）](https://github.com/Bob-Playground/WWDC-Stuff/blob/master/2020/10163-OC-Runtime-Changes/Transcript-Edited.md)。

## 新特性的适配

> With any luck, you won’t need to do anything and your apps will just get faster.  

“运气好的话，你不需要做任何事情，你的应用程序就会变得更快。”  

如果代码中没有依赖“隐藏”在 *Runtime* 中的内部类型（ *internal data structures* ），比如 `class_rw_t`，或直接读取 *Tagged Pointer* 的 *bit* 数据，那就不需要做改动。  

但如果 *APP* 中直接使用了这些“隐藏”的内容，*APP* 在新系统上可能会出现异常。  

这些优化将在下列系统中生效：  

- *macOS Big Sur ( macOS 11 )*
- *iOS 14*
- *tvOS 14*
- *watchOS 7*

## 1. 新增类型：class_rw_ext_t

> 在 *iOS 14* 的 *Class Data Structure* 中新增了 `class_rw_ext_t` 类型。  

我们先看看 *iOS 13* 的 *Runtime* 是如何运行的。  

### class_ro_t

类对象( *class object* )中包含了我们最常访问的信息：指向元类的指针( *metaclass* )，指向父类的指针( *superclass* )，以及指向方法缓存的指针( *method cache* )。  

类对象中还有一个 `class_ro_t` 类型的指针，指向存有更多信息的类型。其中的 **ro** 代表 **read only** 。`class_ro_t` 存储着**类的名称**，以及**方法列表**、**协议**、**属性**、**实例变量**等信息。  

*类对象*和 `class_ro_t` 的示意图：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-class_ro_t.jpg)

*Swift 类* 和 *Objective-C 类* 共用这个基础设施( *infrastructure* )，因此每个 *Swift 类* 也有这些结构。  

当*类*第一次从 *disk* 加载到 *memory* 时，他们就是以这种方式呈现的，但是当*类*被**使用**时会发生改变。

### Clean memory & Dirty memory

为了理解后续要发生的事，需要了解 *clean memory* 和 *dirty memory* 的区别。

Clean Memory
: *Clean memory* 是加载后就不会改变的 *memory* ，比如 `class_ro_t` ，因为它是 *read only* 的。
: *Clean memory* 可以从 *memory* 中清除、为其他内容腾出空间。因为如果需要它，系统总是可以从 *disk* 重新加载它。

Dirty Memory
: *Dirty memory* 是当*进程*运行的时候会被改变的 *memory* ，比如*类对象*。因为 *Runtime* 会为*类对象*创建一个全新的 *method cache* ，并让*类对象*内的一个指针指向 *method cache* 。
: *Dirty memory* 比 *Clean memory* 昂贵得多。它必须在*进程*运行期间一直存在。

*macOS* 可以把 *memory* 内的 *Dirty Memory* 转移至 *swap（交换分区）；而* *iOS* 不使用 *swap* ，所以在 *iOS* 中 *Dirty Memory* 很昂贵。  

### class_rw_t

当*类***第一次使用时**，*Runtime* 创建了一个额外的类型，叫做 `class_rw_t` ，**rw** 代表 **read/write**（可读可写）。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-class_rw_t.jpg)
_绿色部分是 dirty memory ，蓝色部分是 clean memory_

在 `class_rw_t` 中，存储在 *Runtime* 期间生成的信息。  

例如，所有*类*都使用 *First Subclass* 和 *Next Sibling Class* 指针链接到一个树结构中，这允许 *Runtime* 遍历当前使用的所有类，这对于使方法缓存失效是很有用的。（**这个地方的细节不太了解，作者也没细讲，待研究**）  

`class_ro_t` 中已经有了 **方法列表**、**协议**、**属性** ，为什么又在新创建的 `class_rw_t` 中添加它们呢？  

原因是它们可以在运行时被改变。  

比如，当一个分类被加载时，它可以向原类中添加新方法。即程序员可以使用 *Runtime API* 动态地添加它们。  

### 从 class_rw_t 中拆分出 class_rw_ext_t

> We measured about 30 megabytes of these class_rw_t structures across the system on an iPhone.  

“我们在 *iPhone* 系统中检测到了大约 **30MB** 的 `class_rw_t` 结构。”  

由于 `class_rw_t` 在运行时会被改变，因此它是 *dirty memory* 。在我们的设备上有很多*类*在被使用中，它们占用了相当多的内存。  

那我们怎样把它们缩小呢？

还记得吗，我们之所以在“读/写部分”需要这些东西，是因为它们可以在*运行时*更改。

> But examining usage on real devices, we found that only around 10% of classes ever actually have their methods changed.  

“但是通过对实际设备的使用进行分析，我们发现只有大约 **10%** 的*类*真正改变了它们的**方法列表** 。”  

并且 `demangled` 只被 *Swift class* 使用了，并且只有当 *Swift class* 查询它们对应的 *Objective-C* 名称时，才需要用到它。

因此，我们可以把 `class_rw_t` 拆成两部分，将不常使用的部分拆分出去、放在名为 `class_rw_ext_t` 的新结构中。  

对于确实需要*附加信息*的*类*，我们才创建 `class_rw_ext_t` 。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-class_rw_ext_t-1.jpg)

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-class_rw_ext_t-2.jpg)

> Approximately 90% of classes never need this extended data, saving around 14 megabytes system wide.  

“大约 90% 的类从不需要这种扩展数据，在系统范围内节省了大约 **14MB** 内存。”  

### 实例

使用 `heap` 命令查看 *macOS Big Sur* 应用中 `class_rw_t` 和 `class_rw_ext_t` 的使用情况。    

#### macOS 的 Mail

```bash
heap Mail | egrep 'class_rw|COUNT'
```

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-class_rw_ext_t-heap-Mail.jpg)

*Mail* 中只有约 **10%** 的类使用了 `class_rw_ext_t` 。

#### macOS 版的 WeChat

```bash
heap WeChat | egrep 'class_rw|COUNT'
```

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-class_rw_ext_t-heap-WeChat.jpg)

*WeChat* 中只有约 **8%** 的类使用了 `class_rw_ext_t` 。

### 使用 Runtime APIs

我们应该使用 *Runtime* 的公开 *API* ，*Apple* 保障它们的稳定性。比如：  

- `class_getName`
- `class_getSuperclass`
- `class_copyMethodList`
- ...

而不要在代码中直接使用 *Runtime* 的这些私有类型，否则，如果新系统更新了这些类型的 *layout*，代码就无法正常运行了。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-class_rw_ext_t-error.jpg)

比如，代码直接去 `class_rw_t` 中读取 *Methods* ，在 *iOS 13* 上是可行的，但在 *iOS 14* 上 `class_rw_t` 中已经没有了 *Methods* ，它被挪动到了 `class_rw_ext_t` 中。  

## 2. 相对方法列表（ Reletive Method Lists ）

> 在 *Binary Image* 中使用 **Reletive Method Lists** 。  

每个*类*都有一个附属的*方法列表( Method List )* 。当你给一个*类*写了一个新方法，它会被添加到这个列表中。  

*Runtime* 使用*方法列表*来解析*消息发送( message sends )* 。

### Objective-C 方法的 3 个部分  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-method-list.jpg)
_每个 Objective-C 方法都由 3 个部分构成_

method's name
: 也称之为 *selector* ，对应 `SEL` 类型。*selector* 就是*字符串( Strings )* ，但是它们是唯一的，所以可以通过*指针*检测是否相同。

method's type encoding
: 是 `char *` 类型的，表示*参数( parameters )* 和*返回值( return types )* 的类型。它不用于*消息发送( message sends )*，但 *Runtime* 内省( *introspection* )和消息转发( *message forwarding* )等事情需要它。

method's implementation
: 对应 `IMP` 类型，表示一个指向*方法实现*的指针( *the actual code for the method* ) 。我们编写的 *Objective-C* 方法会被编译成 *C* 函数，它包含 *Objective-C* 方法的实现，然后*方法列表*中的相应条目指向这个函数。

### 以 init 方法为例

*方法列表*中的这 3 项内容都是*指针*类型。  

这意味着在**64位**系统中，*方法列表*中每一个方法的信息占用**24字节**：  

![Desktop View](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-method-pointer-size-64bit.jpg){: .normal width="400"}

### 进程中内存的划分

上述内容属于 *clean memory* ，但 *clean memory* 也不是“免费”的。它仍然需要从 *disk* 加载，并在使用时占用 *memory* 。

这是进程中*内存*的放大视图：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-normal-binary-images-in-memory.jpg){: .normal width="200"}

这个地址空间很大，需要使用**64位**来寻址。在这个地址空间中，内存被划分成了多个部分：  

- *栈( stack )*
- *堆( heap )*
- *可执行程序( executables )*
- 加载到进程中的*库( libraries )*或*二进制映象( binary images )* (上图**蓝色**部分)

### 使用普通的方法列表( Method Lists )

仍以 `init` 方法为例，我们放大来看其中一个 *binary image* ：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-normal-method-list.jpg)

我们可以看到，方法列表中的 3 个指针指向了*二进制文件*中的地址。这给我们展现了额外的消耗：  

- 一个 *binary image* 可以被加载到 *memory* 的任何地方，具体加载到哪由*动态链接器(dynamic linker)* 决定。  
- 这意味着*动态链接器*需要解析指向 *binary image* 的指针，并且在加载 *binary image* 的时候将指针修改为*方法信息*在内存中的实际地址。这也消耗性能。  

但是请注意，*binary image* 中的*类*的 *method entry* ，只会指向该 *binary image* 中的方法的实现。我们不会把某方法的 metadata 放在一个 binary 中，而把这个方法的实现放在另一个 binary 中。  

这意味着 *method list entries* 实际上不需要引用整个**64位**地址空间的能力。它们只需要能够引用自己的 *binary image* 中的方法，而这些方法总是在附近。  

### 使用相对方法列表( Reletive Method Lists )

因此，*method list entries* 可以在 *binary image* 中使用**32位**的*相对偏移量( relative offset )* ，而不是**64位**的*绝对地址*：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-relative-method-list.jpg)

这种方案有几个优势：

- 首先，无论 *binary image* 被加载到*内存*的哪个位置，*偏移量*总是相同的，所以它们不必在从*磁盘*加载后进行修正。
- 因为它们不需要被修正，它们可以被保存在真正的*只读内存*中，这是更安全的。
- 使用**32位**的*偏移地址*意味着我们在**64位**平台上**需要的内存减少了一半**。

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-method-pointer-size-32bit.jpg){: .normal width="400"}

现在*方法列表*中的每一个方法的信息只占用**12字节**。

> We've measured about 80MB of these methods system wide on a typical iPhone. Since they're half the size, we save 40 megabytes.

“我们在一个典型的 *iPhone* 做了测量，在系统范围内的这些方法大约占用了 **80MB** 。由于我们将它们减半了，所以节省了 **40MB** 。”  

为了体现出优化效果明显，PPT 里还播放了一个震撼的动画。和其他厂的宣传 PPT 不同的是，这个没有搭配震撼的音效 [手动狗头]：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-relative-method-list-saving-memory.jpg){: .normal width="500"}

### Swizzling relative method lists

使用了*相对方法列表*后，就不能使用完整的地址空间了。但是如果对一个方法做了 *Swizzling* ，那么这个方法的实现将会出现在任何地方。而且刚才我们提到要把 *binary image* 的*方法列表*设置为*只读*的。  

为了处理这种场景，*iOS 14* 的 *Runtime* 使用了一个*全局表( global table )* 映射*方法*到它们的 *Swizzling* *实现*。

实际上，*Swizzling* 很少。绝大多数*方法*实际上从来没有被 *Swizzling* ，所以这个 *table* 最终不会变得很大。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-relative-method-list-swizzling.jpg)

> Even better, the table is compact. Memory is dirtied a page at a time.

更好的是，这个 *table* 很小巧。每次只“污染”内存的一*页( page )* 。  

- 在使用旧格式的*方法列表*时，*Swizzling* 一个方法会污染它所在的*全部页( entire page )*，一次 *Swizzling* 会导致*很多KB( kilobytes )* 的脏内存。
- 在使用 *global table* 后，我们只需要付出一个额外的*表条目( table entry )* 的成本。

### Deployment target

如果使用相符的 *minimum deployment target* （至少 *iOS 14*）构建项目，那么 *Xcode* 会自动为构建的二进制包生成 *relative method lists* 。  

如果需要支持旧的系统版本，*Xcode* 会生成旧风格的 *method list* 。

> You still get the benefit from the OS itself being built with the new relative method lists, and the system has no problem with both formats in use in the same app at the same time.

“你仍然可以从使用新的 *relative method lists* 构建的操作系统中获得好处，并且系统在同一 *APP* 中同时使用两种格式没有问题。”   

比如，假设我们的 *APP* 的 *minimum deployment target* 指定的是 *iOS 13* ，并在 *APP* 内引入了两个库：  

1. 系统内建的 `UIKit` ，它在 *iOS 13* 上使用的是普通方法列表，在 *iOS 14* 的 `UIKit` 使用的是相对方法列表；
2. 一个第三方的 *Framework* ，*minimum deployment target* 指定的是 *iOS 13* ，因此它将使用普通方法列表。

在 *iOS 14* 系统上运行这个 *APP* ，使用相对方法列表的 `UIKit` 和使用普通方法列表的 *Framework* 都能正常运行。  

但是，如果把项目的 *minimum deployment target* 指定为今年发布的系统版本( *iOS 14* )，那么生成的二进制包更小、使用时占用的内存更小( **smaller binaries** and **less memory usage** )。这对 *Objective-C* 或 *Swift* 项目都是一个很好的建议。当 *Xcode* 知道它不需要支持旧的系统版本时，它通常可以生成优化地更好的代码或数据。  

### Mismatched deployment targets

假设我们有两个项目：

1. 一个是把 *minimum deployment target* 指定为 *iOS 14* 的 Framework ，Xcode 在构建它时，会使用 *relative method lists* ；
2. 另一个是把 *minimum deployment target* 指定为 *iOS 13* 的 *APP* ，Xcode 在构建它时，会使用旧格式的 *method lists* 。

如果我们把上述 Framework 集成到上述 APP 中，在 iOS 13 系统上运行此 *APP* 会出现问题。由于旧的系统版本没有处理 *relative method lists* 的机制，所以会读取两个**32位**的指针、当成一个**64位**指针使用。  

这意味着两个独立的指针被合并成了一个值无效的指针，使用时肯定会导致 *crash* 。

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-relative-method-list-error.jpg){: .normal width="600"}

### 使用 Runtime APIs

同样地，不要直接使用 *Runtime* 的私有类型，否则在新系统上这些私有类型的 layout 变化后，会导致 *crash* 。应该使用 *Runtime* 提供的公共 *API* 。

- `method_getName`
- `method_getTypeEncoding`
- `method_getImplementation`
- ...

## 3. ARM64 架构上 Tagged Pointer 格式的变化

下文关于 *tagged pointer* 的介绍用到了几个术语，这里先做个解释：  

- **tag bit** : 共 **1 bit**，是 *tagged pointer* 的标志位；
- **tag number** : 共 **3 bit**，代表 *tagged pointer* 的类型；
- **extended tag** : 共 **8 bit**，代表 *tagged pointer* 的拓展类型，当 *tag number* 为 *7(0b111)* 时，就会使用 *extended tag* ；
- **payload** : *tagged pointer* 实际存储的内容。
  - 如果不使用 *extended tag*，*payload* 为 **60 bit**；
  - 如果要使用 *extended tag*，*payload* 为 **52 bit**。

### 普通的对象指针

我们先看看普通的*对象指针( object pointer )* 的结构。  

普通的*对象指针*通常用很大的*十六进制( hexadecimal )* 数字表示。下面我们把它分解成*二进制(binary)* 表示：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-normal-object-pointer.jpg)

在实际的*对象指针*中，只有中间的 bit 被使用了。

- *低 3 位*的值总是 0 ，因为**内存需要对齐：对象的地址值必须是指针大小的整数倍**。( *objects must always be located at an address that's a multiple of the pointer size.* )
- *高 16 位*的值总是 0 ，因为需要的地址空间有限，实际上我们没有一直算到 **2^64** 。

可以看出，普通的*对象指针*的*低 3 位*和*高 16 位*一直是 0 。

### Intel

在**64位** *Intel* 平台的 *Mac* 上，如果把*最低位( bottom bit )* 设置为 1 ，就代表这个指针不是普通的*对象指针*，而是 *tagged pointer* ：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-intel-1.jpg)

在上图的示例中，我们创建了一个值为 42 的 *NSNumber* 。42 对应的二进制数是 101010 ，这个二进制数被放入了 *tagged pointer* 的第 5 至 10 位。  

只要我们（指 *Runtime* 的开发者）教 `NSNumber` 如何读取这些*位*，并教 *Runtime* 正确处理 *tagged pointer* ，系统的其余部分就可以把这些东西当作*对象指针*，永远不知道其中的区别。  

这节省了为较小的数值分配对象的开销，性能肯定会更好。

#### 混淆 tagged pointer 的值  

我们获取到的 *tagged pointer* 的值是做了混淆的。*Runtime* 使用*进程启动时*创建的*随机值*与 *tagged pointer* 的值做了结合。这是一种安全措施，使 *tagged pointer* 难以被伪造。  

所以，如果开发者尝试去读取内存中 *tagged pointer* 的值，获取到的将是被混淆的值。  

#### tag number 和 payload

接下来的 3 *bit* 是 *tag number* ，代表 *tagged pointer* 的*类型*。比如 *3(0b011)* 代表 `NSNumber` ，*6(0b110)* 代表 `NSDate` 。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-intel-2.jpg)

因为 *tag number* 有 3 *bit* ，说明可以表示 8 种类型(0-7)。  

剩下的 *bits* 是 *payload* ，用来存储值。对于 *tagged* `NSNumber` 来说， *payload* 存储的就是实际的数值。  

#### extended tag

*tag number* 等于 *7(0b111)* 是一个特殊的 case ，它代表要使用 *extended tag* 。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-intel-3.jpg)

*extended tag* 使用接下来的 8 *bit* 来编码类型，以更小的 *payload* 为代价、来容纳 256 种新的 *tag* 类型。比如 *tagged* `UIColor`、*tagged* `NSIndexSet`。  

#### Swift 中的 tagged pointer

只有 *Apple Runtime* 的开发人员才能添加 *tagged pointer* 类型。  

但是在 Swift 开发中，可以创建自己的 *tagged pointer* 类型。比如，*Swift Runtime* 将*枚举标识符( enum discriminator )* 存储在*关联值( associated value )* *payload* 的*空余位( spare bits )* 中。  

更重要的是，*Swift* 对*值类型*的使用实际上降低了 *tagged pointer* 的重要性，because values no longer need to be exactly pointer sized（???）。  

例如，一个 *Swift* `UUID` 类型可以是两个单词并且*内联( inline )* 保存，而不是分配一个单独的对象，因为它不适合放在指针中（???）。  

### ARM64

#### iOS 13 中 tagged pointer 的格式

和 *Intel* 平台不同的是，在 *ARM64* 平台上使用*最高位( top bit )* 来标识 *tagged pointer* 。接下来的 *3 bits* 用于标识 tagged pointer 的类型，payload 使用剩下的 *bits* ：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-arm64-overview.jpg)

为什么在 *ARM* 平台上用 *top bit* 代表 *tagged pointer* 呢？  

实际上这是为 `objc_msgSend` 做的一个小优化。  

我们希望 `objc_msgSend` 中最常见的使用方式尽可能快，而最常见的使用方式是一个普通的对象指针调用 `objc_msgSend` 。  

把 *top bit* 设置为 1 ，就能在一个判断中得知这个指针是 *tagged pointer* 或 `nil` ，这简化了普通对象指针调用 `objc_msgSend` 的流程，不用分别去判断 *tagged pointer* 和 `nil`：  

```c
if (ptrValue <= 0) // is tagged or nil
```

说明：  

- `nil` 对应的指针值是 `0x0` ；
- 由于*最高位( top bit )* 是*符号位*，当最高位是 1 是，指针值是一个负数。

所有当指针值小于或等于 0 时，代表指针为 *tagged pointer* 或 `nil` 。  

和 *Intel* *平台类似，ARM* 上的 *tag number* *7(0b111)* 代表要把接下来的 *8 bit* 用做 *extended tag* 。剩下的 *bits* 给 *payload* 使用：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-arm64-old.jpg)

#### iOS 14 中 tagged pointer 格式的变化

在 *iOS 14* 中，*tagged pointer* 的*标志位( tag bit )*还是在*最高位*，因为这个对 `objc_msgSend` 的优化仍然很有用。  

但是 *tag number* 挪到了*低 3 位( bottom three bits )* ， *extended tag*（如果用到了）紧随 *tag bit* 后面。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-arm64-new-1.jpg)

为什么要这样改呢？  

我们再来看看 *tagged pointer* 和普通的*对象指针*的差异。  

- 我们现有的工具，比如动态链接器，由于一个名为 **Top Byte Ignore(TBI)**[^1] 的 *ARM* 特性，会忽略指针的*高 8 位*。所有，我们可以把 *extended tag* 放在 *Top Byte Ignore* 位中，用于标识更多的 *tagged pointer* 类型。  
- 对于一个**对齐的**普通对象指针，它的*低 3 位*总是 0 。将 *tagged pointer* 的 *tag number* 挪到*低 3 位*后 ，*低 3 位*的值为 *7(0b111)* 时，代表要使用 *extended tag* 。  

现在，*tagged pointer* 和普通的*对象指针*格式一致了，这意味着我们可以添加一个 *extended tag pointer* 类型，代表 payload 中存储的是一个指针类型的值。  

因此，**我们可以在 tagged pointer 的 payload 中存储一个普通的指针值了**。  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-arm64-new-2.jpg)

这为什么很有用呢？  

> Well, it opens up the ability for a tagged pointer to refer to constant data in your binary such as strings or other data structures that would otherwise have to occupy dirty memory.

“它使 *tagged pointer* 能够*引用*二进制包中的*常量数据*，如字符串或其他数据结构，否则这些数据结构将不得不占用 *dirty memory* 。”  

### 使用 APIs

不要在代码中依赖 *Runtime* 的内部细节。我们看一个案例：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/runtime-tagged-pointer-on-arm64-error.jpg)

在上图的示例中，将指针值执行位操作、右移 60 ，如果结果为 *0xa(0b1010)* ，说明这个指针是存有 `NSString` 的 *tagged pointer* 。  

在 *iOS 13* 上，这种写法确实没问题。右移后得到的 1010 这个值分为两部分：  

- 1 是指针的最高位，代表这个指针是 *tagged pointer* 
- 010 是 *tag number* ，代表这个指针是 *tagged* `NSString`

但在 *iOS 14* 上，由于 *tag number* 被移到了*低 3 位*，所以上面代码中的判断在新系统上不再准确。  

应该使用 `isKindOfClass` 这种公开的 API 来判断指针类型，而不要依赖 *Runtime* 中的内部细节。  

最后，演讲人又说了两段话来解释为什么不要在代码中依赖 *Runtime* 的内部细节，值得引起（喜欢在生产环境中使用系统私有 API 的）开发者们的注意：  

> We don't want to hide anything and we definitely don't want to break anybody's apps.

“我们不想隐藏任何东西，也绝对不想破坏任何人的 APP 。”

> When these details aren't exposed, it's just because we need to maintain the flexibility to make changes like this, and your apps will keep working just fine, as long as they don't rely on these internal details.

“这些细节没有被公开，只是因为我们需要保持灵活性来做出类似上文中的优化。只要你的 APP 不依赖于这些内部细节，它们将继续正常工作。”

## 小结

以 *PPT* 里的总结来结尾吧：  

![](/images/WWDC/2020/10163-OC-Runtime-Changes/wrap-up.jpg){: .normal width="500"}

### 相关资料

- 笔者整理的 *WWDC* 资料库，含讲稿（字幕）：[https://github.com/Bob-Playground/WWDC-Stuff](https://github.com/Bob-Playground/WWDC-Stuff)

## Reference

[^1]: **Top Byte Ignore(TBI)** : 是 *ARMv8* 引入的一个特性，它通过忽略虚拟地址( *virtual address* )的*高 8 位*来提供内存标记工具。这允许软件使用一个**64位**指针的*高 8 位*作为标签。参考：[https://en.wikichip.org/wiki/arm/tbi](https://en.wikichip.org/wiki/arm/tbi)

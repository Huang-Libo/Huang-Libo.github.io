---
title: "LLDB: Beyond po"
tags: LLDB
---

# 前言

[WWDC 2019 / 429](https://developer.apple.com/videos/play/wwdc2019/429/) 介绍了 Xcode 11 中 `LLDB` 的常用技巧及其原理，本文做一个摘要和总结。  

示例 project：[https://github.com/Bob-Playground/LLDB-Demo](https://github.com/Bob-Playground/LLDB-Demo)  

# LLDB 常用命令一：po

示例代码：

```swift
struct Trip {
    var name: String
    var destinations: [String]
}

let cruise = Trip(
    name: "Mediterranean Cruise",
    destinations: ["Sorrento", "Capri", "Taormina"])
```

## po 的常见用法

`po` 常见用法是打印变量：  

```lldb
(lldb) po cruise
▿ Trip
  - name : "Mediterranean Cruise"
  ▿ destinations : 3 elements
    - 0 : "Sorrento"
    - 1 : "Capri"
    - 2 : "Taormina"
```

`po` 也可以调用对象的方法：  

```lldb
(lldb) po cruise.name.uppercased()
"MEDITERRANEAN CRUISE"
```

```lldb
(lldb) po cruise.destinations.sorted()
▿ 3 elements
  - 0 : "Capri"
  - 1 : "Sorrento"
  - 2 : "Taormina"
```

`po` 还可以计算表达式，等等。  

## po 是 expression 命令的 alias

> `po` 不是 LLDB 中的 first-class 命令。

po 实际上是 `expression` 命令的一个 *alias*，`po cruise` 等效于：

```lldb
expression -O -- cruise
```

参数说明：`-O` 是 `--object-description` 的简写。详细的参数说明可在 LLDB 内通过 `help expression` 查看。

我们也可以创建自己的 *alias*：

```lldb
command alias my_po express -O --
```

使用自己创建的 `my_po`：

```lldb
my_po cruise
```

## po 的原理

po 的执行流程如下，假如用户输入了 `po view`：

![](/images/2021/lldb-po-1.jpg)

此例中，LLDB 为 `po` 生成两次代码并编译、执行，最后展示相应的 description。

如果想自定义 description，可以添加一个遵守 `CustomDebugStringConvertible` 协议的 extension：

```swift
extension Trip: CustomDebugStringConvertible {
    var debugDescription: String { "Trip description" }
}
```

更多信息请查看 `CustomDebugStringConvertible` 协议的相关文档。

# LLDB 常用命令二：p

和 `po` 和类似的命令是 `p`：

```lldb
(lldb) p cruise
(LLDB_Demo.Trip) $R0 = {
  name = "Mediterranean Cruise"
  destinations = 3 values {
    [0] = "Sorrento"
    [1] = "Capri"
    [2] = "Taormina"
  }
}
```

与 `po` 输出不同的是，`p` 的输出中多了一个 `$R0`。像这种`$R`+`数字`的组合，实际上是 LLDB 生成的变量，可以在后续的调试中使用，如：

```lldb
(lldb) po $R0.name
"Mediterranean Cruise"
```

## p 是 expression 命令的 alias

> `p` 不是 LLDB 中的 first-class 命令。

实际上，`p` 也是 `expression` 命令的 *alias*，`p cruise` 等效于：

```lldb
expression cruise
```

## p 的原理

![-w1549](/images/2021/lldb-p-1.jpg)

`p` 的前半部分过程与 `po` 一样，生成获取对象的代码并获取对象。不一样的地方是，`p` 拿到 **result** 后，会对 **result** 做 **Dynamic type resolution（动态类型解析）**。我们来看一个例子，把之前的代码稍作改动：

```swift
protocol Activity {}

struct Trip: Activity {
    var name: String
    var destinations: [String]
}

let cruise: Activity = Trip(
    name: "Mediterranean Cruise",
    destinations: ["Sorrento", "Capri", "Taormina"])
```

我们添加了一个 `Activity` 协议，并让 `Trip` 遵守此协议。创建的 `cruise` 变量声明为遵守 `Activity` 协议的类型。

再输入 `p cruise` 和 `p cruise.name`：

![-w506](/images/2021/lldb-p-2.jpg)

可以看到，`p cruise` 的输出和之前一样，打印出了 `cruise` 的真实类型 `Trip`。这是因为 LLDB 在拿到 **result** 后对其做了**Dynamic type resolution**。

而 `p cruise.name` 报错，是因为在 LLDB **生成代码时**(生成代码是第一步，还没到动态解析)，只能从源码知道 `cruise` 是遵守 `Activity` 的协议的类型，而 `Activity` 协议中并没有 `name` 成员，所以 LLDB 生成的代码编译时会报错。

想要通过 `p` 打印 `name`，就必须先将 `cruise` 强转为 `Trip` 类型：`p (cruise as! Trip).name`。

在流程的最后，**result** 会传递给 **formatter subsystem** 处理，以输出可读性更强的内容。如果想看原始内容，可在 `expression` 命令后使用 `--raw` 参数：

```lldb
expression --raw -- cruise
```

![](/images/2021/lldb-expression-raw.jpg)


# LLDB 常用命令三：v

在上面的例子中，我们需要强制转换 `cruise` 的类型，才能打印 `name`。其实 LLDB 的 `v` 命令，可以更便捷地完成这项任务：

![-w492](/images/2021/lldb-v-1.jpg)

从输出我们可以看到，`v cruise` 的输出和 `p cruise` 类似。但是，`p cruise.name` 报错了，而 `v cruise.name` 能正常打印 name。

## v 是 frame 命令的 alias

实际上，`v` 是 Xcode 10.2 引入的 *alias*，`v cruise` 等效于：

```lldb
frame variable cruise
```

## v 的原理：

![-w1457](/images/2021/lldb-v-2.jpg)

从上图可以看出，`v` 并不生成代码来编译和执行，而是先直接从内存中读取值，再进行 **Dynamic type resolution**。如果有 **subfields**，则循环这两步，直到拿到最终的值。  

与 `p` 相同的是，**result** 会传递给 **formatter subsystem** 处理，以输出可读性更强的内容。  

由于不需要编译和执行代码，`v` 的速度也比 `po` 或 `p` 快很多。但是，这也决定了 `v` 只能读取值，而**无法调用方法或计算表达式**。  

# po，p，v 的使用场景

![-w1569](/images/2021/lldb-compare-po-p-v.jpg)

小结：  

- `po` 和 `p` 可使用语言的全部特性。LLDB 根据用户的输入，生成代码编译运行。可以进行调用方法、计算表达式等操作。
- `po` 可以获得对象的 description，`p` 和 `v` 能使用 **Data formatter subsystem** 处理 **result**。
- `p` 可以对 **result** 做 **Dynamic type resolution**。
- `v` 直接从内存读取变量，速度快，并且可以对读取的值**递归地**做 **Dynamic type resolution**，但不能用于调用方法、计算表达式等。

# 使用 help 查看 LLDB 命令的介绍

在 `lldb` 内，可使用 `help` 命令查询其他命令的用法，如：

```bash
help po
help expression
```

![-w845](/images/2021/lldb-help.jpg)

# Reference

[https://developer.apple.com/videos/play/wwdc2019/429/](https://developer.apple.com/videos/play/wwdc2019/429/)  
[https://xiaozhuanlan.com/topic/2683509174](https://xiaozhuanlan.com/topic/2683509174)
---
title:  "警惕 NSTimer 引起的循环引用"
tags: iOS timer NSTimer 
---

# 典型案例：使用 Target-Action 模式添加 NSTimer

先看看下述代码有什么问题：  

```objc
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(timerTriggered:) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
}

- (void)timerTriggered:(NSTimer *)timer {
    NSLog(@"timerTriggered");
}

- (void)dealloc {
    [self.timer invalidate];
    self.timer = nil;
    NSLog(@"%s", __func__);
}
@end
```

## 问题描述

退出此 ViewController 页面后，`NSLog(@"%s", __func__)` 不会打印，而 `NSLog(@"timerTriggered")` 会一直执行。说明 `dealloc` 方法没有调用。  

显而易见，这里存在内存泄漏。  


## 原因分析

> 计时器会保留其目标对象，等到自身“失效”时再释放此对象。调用 `invalidate` 方法可令计时器失效；执行完相关任务之后，一次性的计时器也会失效。开发者若将计时器设置成重复执行模式，那么必须自己调用 `invalidate` 方法，才能令其停止。[^1]

在上述代码中，`viewController` 强引用了 `timer`，`timer` 又强引用了 `target` (即 `viewController`)，形成了循环引用：

![](/images/2021/NSTimer-circular-reference-1.png)

说明：如果 `repeats` 参数设为 `NO`, 执行完定时任务后，`timer` 会取消对 `target` 的强引用，因此不会形成循环引用。  

**接下来讲讲常见的解决方法。**  

# 方法一：打破循环引用

由上述分析可知，由于存在循环引用，所以 `dealloc` 方法不会执行。我们可以在 `viewWillDisappear` 等 view 事件里主动销毁 timer，这样就能打破循环引用了。  

可以根据项目的需求自行决定在哪个事件里销毁 timer。

# 方法二：使用 iOS 10 添加的新 API（推荐）

在 **iOS 10** 及以上的项目中，可使用 `NSTimer` 新增的 `block` 范式的方法，只要确保 `block` 内没有循环引用即可：

```objc
__weak typeof(self) weakSelf = self;
self.timer = [NSTimer timerWithTimeInterval:1 repeats:YES block:^(NSTimer * _Nonnull timer) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    NSLog(@"timerTriggered");
    // ...
}];
```

# 方法三：使用 dispatch_source 定时器（推荐）

使用 `dispatch_source` 定时器的案例，可参看 [MSWeakTimer](https://github.com/mindsnacks/MSWeakTimer)。  

另外，由于 `NSTimer` 需要添加到 Runloop 中，当 Runloop 繁忙时，`timer` 的事件就不能得到及时执行，会出现不准时的问题。  

而 `dispatch_source` 不依赖 Runloop，所以比 `NSTimer` **更准时**。  

# 方法四：使用 NSProxy 做中间件（推荐，有创意）

> 示例 project：[https://github.com/BOB-Module/NSTimer-with-NSProxy](https://github.com/BOB-Module/NSTimer-with-NSProxy)

使用 `NSProxy` 做中间件执行消息转发。要点：  

1.  把 `timer` 的 `target` 设置成 `proxy`；
2.  `proxy` 弱引用 `viewController`，将 `timer` 发来的定时任务转发给 `viewController`。  

关系图如下：

![](/images/2021/NSTimer-circular-reference-2.png)

先添加 `NSProxy` 的子类 `LBWeakProxy`：  

```objc
@interface LBWeakProxy : NSProxy

/// 弱引用原 Target（在本例中就是 viewController）
@property (nonatomic, weak) id target;

+ (instancetype)proxyWithTarget:(id)target;

@end
```

```objc
@implementation LBWeakProxy

+ (instancetype)proxyWithTarget:(id)target {
    LBWeakProxy *proxy = [LBWeakProxy alloc];
    proxy.target = target;
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    NSMethodSignature *signature = [self.target methodSignatureForSelector:sel];
    return signature;
}

-(void)forwardInvocation:(NSInvocation *)invocation {
    invocation.target = self.target;
    [invocation invoke];
}

@end
```

然后在创建 `timer` 时，将 `timer` 的 `target` 设置为 `proxy`：  

```objc
// 强引用链: self -> timer -> proxy , 而 proxy 弱引用 self, 不会形成循环引用
LBWeakProxy *proxy = [LBWeakProxy proxyWithTarget:self];
self.timer = [NSTimer timerWithTimeInterval:1 target:proxy selector:@selector(timerTriggered:) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
```

## 原理

`NSProxy`：和 `NSObject` 类似，是基类。这里利用 `NSProxy` 的`methodSignatureForSelector:`方法、`forwardInvocation:` 方法做了消息转发。  

在上述例子中，先通过 `proxyWithTarget:` 方法创建 `LBWeakProxy` 的实例 `proxy`，然后将 `timer` 的 `target` 设置为 `proxy`，在 `proxy` 内弱引用 `viewController`、并将消息转发给 `viewController`。   

回到文章开头的案例：退出 `viewController` 页面时，由于 `timer` 没有强引用 `viewController`，所以 `viewController` 的 `dealloc` 方法会执行，所以 `dealloc` 中销毁 `timer` 的方法也就能正常执行了。  

这样，`viewController` 在释放时，`timer` 也释放了。

## 用 NSProxy 做消息转发比 NSObject 高效

也有人使用 `NSObject` 子类做中间件执行消息转发，但实际上效率没有 `NSProxy` 高。因为 `NSObject` 要在执行方法查找，找不到相关方法后，才进入消息转发阶段。  

而 `NSProxy` 如同其名，天生就是做代理的，会直接进入到消息转发阶段。  

# 小结

- `NSTimer` 很常用，但也有它的局限性，用的时候要多留意细节，提防内存泄漏。  
- `dispatch_source` 定时器不受 Runloop 影响，比 `NSTimer` 更准时的。 在对时间精确度要求高的场景中，`NSTimer` 并不适用。  

# 相关资料

[^1]: 《Effective Objective-C 2.0》第 52 条：别忘了 NSTimer 会保留其目标对象.
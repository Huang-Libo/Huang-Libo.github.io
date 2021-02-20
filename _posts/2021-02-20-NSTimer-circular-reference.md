---
title:  "警惕 NSTimer 引起的循环引用"
tags: iOS
---

## 典型案例：使用 Target-Action 模式添加 NSTimer

请先看看下述代码有什么问题：  

```objc
#import "ViewController.h"

@interface ViewController ()
@property (nonatomic, strong) NSTimer *timer;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // self 强引用了 timer，timer 又强引用了 target（self），形成了循环引用。
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

在上述代码中，`self` 强引用了 `timer`，`timer` 又强引用了 `target`，而 `target` 就是 `self`，即 `timer` 又强引用了 `self`，形成了循环引用，所以 `dealloc` 方法不会执行。

说明：如果 `repeats` 参数设为 NO, 则不会产生循环引用，因为这种情况下 `timer` 没有强引用 `target`。  

**接下来讲讲常见的解决方法。**  

## 方法一：打破循环引用

由上述分析可知，由于存在循环引用，所以 `dealloc` 方法不会执行。我们可以在 `viewWillDisappear` 等 view 事件里主动销毁 timer，这样就能打破循环引用了。  

可以根据项目的需求自行决定在哪个事件里销毁 timer。

## 方法二：使用 iOS 10 添加的新 API（推荐）

在 iOS 10 及以上的项目中，可使用 `NSTimer` 新增的 `block` 范式的方法，只要确保 `block` 内没有循环引用即可：

```objc
self.timer = [NSTimer timerWithTimeInterval:1 repeats:YES block:^(NSTimer * _Nonnull timer) {
    NSLog(@"timerTriggered");
}];
```

## 方法三：使用 dispatch_source 定时器（推荐）

使用 `dispatch_source` 定时器的案例，可参看 [MSWeakTimer](https://github.com/mindsnacks/MSWeakTimer)。  

另外，由于 `NSTimer` 需要添加到 Runloop 中，当 Runloop 繁忙时，`timer` 的事件就不能得到及时执行，会出现不准时的问题。  

而 `dispatch_source` 不依赖 Runloop，所以比 `NSTimer` **更准时**。  

## 方法四：使用 NSProxy 做中间件（有创意）

> 示例 project：[https://github.com/BOB-Module/NSTimer-with-NSProxy](https://github.com/BOB-Module/NSTimer-with-NSProxy)

使用 `NSProxy` 做中间件执行消息转发，以避免 `timer` 强引用 `self`。  

先添加 `NSProxy` 的子类 `LBWeakProxy`：  

```objc
@interface LBWeakProxy : NSProxy

/// 弱引用原 Target
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

然后在创建 `timer` 时，将 `timer` 的 `target` 设置为 `LBWeakProxy` 的实例：  

```objc
// 强引用链: self -> timer -> proxy , 而 proxy 弱引用 self, 不会形成循环引用
LBWeakProxy *proxy = [LBWeakProxy proxyWithTarget:self];
self.timer = [NSTimer timerWithTimeInterval:1 target:proxy selector:@selector(timerTriggered:) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
```

### 原理说明

`NSProxy` 和 `NSObject` 类似，是基类。这里利用 `NSProxy` 的`methodSignatureForSelector:`方法、`forwardInvocation:` 方法做了消息转发。  

在上述例子中，先通过 `proxyWithTarget:` 方法创建 `LBWeakProxy` 的实例 `proxy`，然后将 `timer` 的 `target` 设置为 `proxy`。在 `proxy` 内部，`target` 弱引用 `self`，并且会将消息转发给上述 `self`。  

强引用链: `self` -> `timer` -> `proxy`，而 `proxy` 弱引用 `self`，因此不会形成形成循环引用。  

### 用 NSProxy 做消息转发比 NSObject 高效

也有人使用 `NSObject` 子类做中间件执行消息转发，但实际上效率没有 `NSProxy` 高。因为 `NSObject` 要在执行方法查找、且找不到相关方法后，才进入消息转发阶段。  

而 `NSProxy` 如同其名，天生就是做代理的，会直接进入到消息转发阶段。  

## 小结

- `NSTimer` 很常用，但也有它的局限性，用的时候要多留意细节，提防内存泄漏。  
- `dispatch_source` 定时器不受 Runloop 影响，比 `NSTimer` 更准时的。 在对时间精确度要求高的场景中，`NSTimer` 并不适用。  
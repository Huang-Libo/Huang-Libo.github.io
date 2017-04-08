---
layout: post
title:  "改变 UITabBarItem 上 badge(小红点)的大小"
date:   2017-04-08 15:02:15 +0800
categories: iOS
---

## 小红点简介

APP 里的小红点, 对于无数强迫症患者来说, 简直就是一场灾难. 然而, 用户和程序员都不怎么喜欢的小红点, 对产品经理来说却是一个"重要特性". 所以小红点是大部分 app 都需要做的功能, iOS 对小红点有一些原生的支持, 也有一些相关的第三方库, 比如 [RKNotificationHub](https://github.com/cwRichardKim/RKNotificationHub).

## 关于小红点的新需求

> 需求 :  **给 `UITabBarItem` 添加一个不带数字的小红点** . 

### 可能的实现方式一: (使用 iOS 原生的)

这个时候我们最先想到的可能就是使用 `UITabBarItem` 的 `badgeValue` 属性, 因为不需要显示数字, 所以直接给一个空字符串:

```
self.navigationController.tabBarItem.badgeValue = @"";
```

但是这种实现方式有一个很严重的问题, 就是小红点太大了, 以至于我现在都直接称其为**大胖点**. 这里我就不上图了, 不信的同学可以试一试.

**小结: 此方法适用于在 `UITabBarItem` 上显示带数字或者带文字的小红点. 不能满足需求, pass.**

### 可能的实现方式二: (使用第三方库)

系统原生的方法不适用时, 我们秉持着**不重复造轮子**的原则, 先去 `GitHub` 上搜索, 找一找有没有合适的第三方库. 功夫不负有心人, 我找到了文章开头提到的 [RKNotificationHub](https://github.com/cwRichardKim/RKNotificationHub), 当我心情愉悦地准备大干一场时才发现, `RKNotificationHub` 只能给 `UIView` 的子类添加红点, 而 `UITabBarItem` 是继承自 `UIBarItem`, `UIBarItem` 就直接继承自 `NSObject`, 和 `UIView` 一点关系也没有.

之后又在 `Github` 上看了几个库, 都是给 UIView 的子类添加红点的, 想了想这个功能其实挺轻量级的, 要不就自己写吧.

**小结: 此方法适用于给 `UIView` 的子类添加小红点. 不能满足需求, pass.**

### 可能的实现方式三: (写一个 `UITabBar` 的分类)

要自己实现这个功能, 最先想到的方式的给 `UITabBarItem` 写一个分类, 看看能不能改变原生**大胖点**的尺寸. 

但我在 `Google` 和百度都搜了搜, 看到的最多的实现方式是写一个 `UITabBar` 的分类. 当时我就有点没想明白, 给 `UITabBar` 写分类, 如何去取到每一个 `UITabBarItem` 的位置? 想一想就很麻烦. 这里有一个[相关博客](http://blog.csdn.net/lilinoscar/article/details/47103747), 感兴趣的同学可以看看, 想节约时间的同学可以直接看后面的小结.

该博客的解决方式有一个很明显的问题, 就是他使用了一个 `Magic Number`: `0.6`, 我第一眼看到这个 `0.6` 的时候就在想, 这个 `0.6` 是怎么确定的, 到底靠不靠谱? 看了代码我明白了, 这个数字是用来计算 `UITabBarItem` 相对于 `UITabBar` 的位置的, 还真不一定靠谱.

在`iPhone` 上这个实现方式是勉强可以的接受, 因为 `iPhone` 上的各个 `UITabBarItem` 相隔很近, 并且第一个 `UITabBarItem` 和 `UITabBar` 的左侧也是挨着的; 但是, 由于 `iPad` 的屏幕很宽, 各个 `UITabBarItem` 之间有一个不太确定的间距, 并且 `iPad` 上的第一个 `UITabBarItem` 和 `UITabBar` 的左侧有很长的一段间距, **所以如果 APP 要适配 iPad, 这个方法肯定是不适用的**.

**小结: 这个博客用到的方法目前看起来可以满足 iPhone APP 的需求, 但是万一哪天苹果爸爸给 `iPhone` 上的 `UITabBarItem` 之间添加了一个大间距, 小红点的位置就不准了.**


### 最靠谱的实现方式: (写一个 `UITabBarItem` 的分类)

思来想去, 还是觉得给 `UITabBarItem` 写一个分类比较靠谱. 

经过大量搜索, 我在 `GitHub` 的一个角落找到了一个相关的仓库: [UITabBarItem-CustomBadge](https://github.com/enryold/UITabBarItem-CustomBadge), 看名字就能猜到这是一个关于 `UITabBarItem` 的分类. 该作者写的分类的功能和我的需求稍有差别, 所以我做了一点修改, 先上代码:

```objc
//
//  UITabBarItem+Badge.h
//  KNAdministrator
//
//  Created by HuangLibo on 2017/4/6.
//  Copyright © 2017年 KNRT. All rights reserved.
//
//  原理: 给 UITabBarItem 的 badgeValue 赋值为空字符串, 系统会添加一个大胖点, 把大胖点的颜色设置为 clearColor (只适用于 iOS 10, 因为 badgeColor 是 iOS 10 的新属性, iOS 9 及以下的实现方式是通过 KVC 给 _UIBadgeBackground 的 image 属性设置为 nil). 在 UITabBarItem 的 view 属性上有一个 _UIBadgeView 成员变量(就是大胖点), 通过遍历获取大胖点并在其上添加一个子 view, 颜色设置为红色, 设置好 frame 即可.

#import <UIKit/UIKit.h>

@interface UITabBarItem (Badge)

/**
 添加一个比系统 badge 更小的 badge (红点)
 */
- (void)addSmallBadge;

@end
```

 
 
 
```objc
//
//  UITabBarItem+Badge.m
//  KNAdministrator
//
//  Created by HuangLibo on 2017/4/6.
//  Copyright © 2017年 KNRT. All rights reserved.
//

#import "UITabBarItem+Badge.h"

#define SMALL_BADGE_TAG 10000
#define BADGE_DIAMETER 10

@implementation UITabBarItem (Badge)

- (void)addSmallBadge {
    // 获取 UITabBarItem 中的 view: UITabBarButton
    UIView *tabBarButton = [self valueForKey:@"view"];
    
    // 通过设置 badgeValue 的值让系统添加大胖点
    self.badgeValue = @"";
    // 大胖点的颜色设置为透明 iOS 10 才能使用
    if ([UIDevice currentDevice].systemVersion.doubleValue >= 10) {
        self.badgeColor = [UIColor clearColor];
    }
    
    // 遍历 UITabBarItem view 的 subviews
    for(UIView *subview in tabBarButton.subviews) {
        NSString *str = NSStringFromClass([subview class]);
        
        if([str isEqualToString:@"_UIBadgeView"])
        {
            // 此时的 subview 是 _UIBadgeView
            UIView *badgeView = subview;
            
            for(UIView *badgeSubview in badgeView.subviews)
            {
                // 如果已有小红点, 则先移除
                if(badgeSubview.tag == SMALL_BADGE_TAG) {
                    [badgeSubview removeFromSuperview];
                }
                
                // iOS 9 才通过此方式设置大胖点的颜色
                if ([UIDevice currentDevice].systemVersion.doubleValue < 10) {
                    // 设置 _UIBadgeBackgound 的背景色为透明
                    NSString *aViewClassStr = NSStringFromClass([badgeSubview class]);
                    if ([aViewClassStr isEqualToString:@"_UIBadgeBackground"]) {
                        // 将 badgeSubview 的 image 属性设置为 nil, 大胖点就消失了, 所以推测原生的大胖点就是一个 18*18 的图片
                        [badgeSubview setValue:nil forKey:@"image"];
                    }
                }
            }
            
            // 创建一个小红点, kBadgeDiameter 为红点直径
            UIView *customBadge = [[UILabel alloc] initWithFrame:CGRectMake(0, 0, BADGE_DIAMETER, BADGE_DIAMETER)];
            // 设置小红点的颜色
            customBadge.backgroundColor = [UIColor redColor];
            
            // 小红点切圆角
            customBadge.layer.cornerRadius = customBadge.frame.size.height / 2;
            customBadge.layer.masksToBounds = YES;
            
            // 设置 tag
            customBadge.tag = SMALL_BADGE_TAG;
            
            // 将小红点添加到大胖点上
            [badgeView addSubview:customBadge];
        }
    }
}

@end
```

解读: 实现步骤在注释里已经写的比较清楚了, 我主要想说说核心思路. 能通过这个方式来完成此功能, 还是依靠了强大的 KVC, 取出了 `UITabBarButton`, 并操作了相关的私有成员变量: `_UIBadgeView`, `_UIBadgeBackground`.

另外有一点要注意的是, 我写的分类对 iOS 10 及 iOS 10 以下的系统作了区别处理, 主要是因为在 iOS 10 上有 `badgeColor` 这个属性, 可以通过设置其值来把大胖点变为透明, 但是 iOS 10 以下的系统没这个属性, 要适配 iOS 10 以下的系统, 可以通过使用 `KVC` 把 `_UIBadgeBackground` 的 `image` 属性设置为 `nil` 的方式来实现的.

**小结: 靠谱!**

## 总结

以上讲了**给 `UITabBarItem` 添加一个不带数字的小红点**的四个思路, 前两个因为适用场景不同, 所以没采用; 第三种方式比较 tricky, 并且在 iPad 这种大屏设备上不适用; 所以给 `UITabBarItem` 添加一个不带数字的小红点最合适的方式还是第四种: 给 `UITabBarItem` 添加一个分类.














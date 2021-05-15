---
title:  "ObjC 中 selector, method, message 的区别"
categories: [编程, iOS Dev Tips]
tags: [iOS, Runtime]
---

有点晚了, 先占个坑待填.

selector, 方法名  

method, selector 及 method 的实现.

message, selector 及 所需的参数.

```objc
#import "InteractiveSectionController.h"
#import "InteractiveCell.h"

@implementation InteractiveSectionController

#pragma mark - IGListSectionType

- (NSInteger)numberOfItems {
    return 1;
}

- (CGSize)sizeForItemAtIndex:(NSInteger)index {
    return CGSizeMake(self.collectionContext.containerSize.width, 41);
}

- (UICollectionViewCell *)cellForItemAtIndex:(NSInteger)index {
    InteractiveCell *cell = [self.collectionContext dequeueReusableCellOfClass:[InteractiveCell class] forSectionController:self atIndex:index];
    return cell;
}

- (void)didUpdateToObject:(id)object {
    
}

- (void)didSelectItemAtIndex:(NSInteger)index {
    
}

@end
```





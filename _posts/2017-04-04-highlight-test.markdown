---
layout: post
title:  "代码高亮测试"
date:   2017-04-04 02:10:15 +0800
categories: iOS
---

```
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
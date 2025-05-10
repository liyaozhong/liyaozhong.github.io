---
layout: post
title: "iOS UIScrollView一个容易忽略的细节"
subtitle: "UIScrollView与系统右滑手势冲突解决方案"
tags: [iOS, UI]
comments: false
---

最近项目中需要用到横向滚动的UIScrollView，另外页面还要支持系统右滑退出的逻辑。说到这里大家肯定都知道了，我们要解决的是一个手势冲突的问题。解决方案网上很多，我说一下自己的解决方法：
自定义一个UIScrollView，当然要实现``UIGestureRecognizerDelegate``
```
@interface GestureScrollView : UIScrollView<UIGestureRecognizerDelegate>
@end
```
然后重写``- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer``
```
@implementation GestureScrollView

- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer{
    if ([gestureRecognizer isKindOfClass:[UIPanGestureRecognizer class]]  
      && [otherGestureRecognizer isKindOfClass:[UIScreenEdgePanGestureRecognizer class]] 
      && self.contentOffset.x == 0) {
        return YES;
    } else  {
        return NO;
    }
}

@end
```

到此，问题就解决了。
通常，这可能会是一道面试题，通常，问到这里问题就结束了。我喜欢刨根问底，于是会多问一句**"有没有漏掉什么？"**
是啊，总感觉怪怪的，我们只是实现了代理方法，并没有写``xxx.delegate = self``的逻辑啊。
很多面试者会直接说，设置UIScrollView的panGestureRecognizer的delegate指向UIScrollView自己！
看下``UIScrollView.h``：
```
// Use these accessors to configure the scroll view's built-in gesture recognizers.
// Do not change the gestures' delegates or override the getters for these properties.
@property(nonatomic, readonly) UIPanGestureRecognizer *panGestureRecognizer NS_AVAILABLE_IOS(5_0);
```
这货是readonly的，还不能改。

看来这个问题没那么简单，想找到正确答案，我们来做个简单实验。添加如下代码：
```
- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if(self){
        NSLog(@"self = %@, panGestureRecognizer.delegate = %@", self, self.panGestureRecognizer.delegate);
    }
    return self;
}
```
运行log如下：
```
self = <GestureScrollView: 0x145854800; baseClass = UIScrollView; frame = (0 34.5; 375 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x1455d8520>; layer = <CALayer: 0x14551ee40>; contentOffset: {0, 0}; contentSize: {0, 0}>
panGestureRecognizer.delegate = <GestureScrollView: 0x145854800; baseClass = UIScrollView; frame = (0 34.5; 375 568); clipsToBounds = YES; gestureRecognizers = <NSArray: 0x1455d8520>; layer = <CALayer: 0x14551ee40>; contentOffset: {0, 0}; contentSize: {0, 0}>

```
果然如此，UIScrollView在初始化的时候就已经把panGestureRecognizer.delegate设置成了自己。 
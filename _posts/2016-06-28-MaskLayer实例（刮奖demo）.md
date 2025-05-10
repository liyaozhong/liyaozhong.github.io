---
layout: post
title: "MaskLayer实例（刮奖demo）"
subtitle: "iOS中CALayer的mask属性应用"
tags: [iOS]
comments: false
---

>今天在简书上看到了一个[刮刮乐的demo](http://www.jianshu.com/p/eab521dde13f)，作者的思路很有意思,推荐大家去阅读下。

最近的项目要做im，有下面的场景：
![聊天发图片.png](http://upload-images.jianshu.io/upload_images/1415843-af83d42593985280.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个气泡的实现用到了maskLayer，正好可以实现一个刮奖的demo。于是乎...**搞起！**

##maskLayer介绍

``CALayer``有一个``mask``属性，这便是我们今天的主角。看下它是干什么的:
```
/* A layer whose alpha channel is used as a mask to select between the
 * layer's background and the result of compositing the layer's
 * contents with its filtered background. Defaults to nil. When used as
 * a mask the layer's `compositingFilter' and `backgroundFilters'
 * properties are ignored. When setting the mask to a new layer, the
 * new layer must have a nil superlayer, otherwise the behavior is
 * undefined. Nested masks (mask layers with their own masks) are
 * unsupported. */

@property(nullable, strong) CALayer *mask;
```

简单理解就是，如果``mask``不为nil，那么``mask``以内的区域会显示``layer``本身的内容，``mask``以外的区域会显示``layer``后面的内容（相当于透明）。这里需要两点注意：
  * ``mask``必须是一个独立的layer，不能拥有``super layer``
  * 不支持嵌套的``mask``

##上图


![demo.gif](http://upload-images.jianshu.io/upload_images/1415843-40f17fe7f79466d5.gif?imageMogr2/auto-orient/strip)

###View Hierarchy
没用Reveal，大伙凑活看吧。

![view.png](http://upload-images.jianshu.io/upload_images/1415843-def320674c9b370c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![view hierarchy.png](http://upload-images.jianshu.io/upload_images/1415843-69e7c1e35556e203.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要三个View：
* 背景UIImageView--scratch_bg.png（蓝色背景）
* ScratchView--设置``mask``的自定义view
* UILabel--显示刮奖结果，可以根据具体需求改为其他view

###工作原理
如上所示，``mask``的设置在ScratchView中，捕获手指的移动创建``mask``的layer并设置给ScratchView。
这样一来，``mask``区域内显示ScratchView本身的内容（ScratchView的子view），``mask``区域外继续显示ScratchView后面的内容（背景图）。

###如何绘制maskLayer？
首先要明白，``mask``是一个``CALayer``,创建一个不规则的``CALayer``首选``CAShapeLayer`` ；

其次，``CAShapeLayer``通过path来定义形状，我们的目标就是把用户的每一次移动轨迹通过path来表示；

再其次，用户移动轨迹必然不能通过一个path来表示（做path的union操作......想都不敢想），所以我们把每个用户轨迹用一个``CAShapeLayer``表示，然后通过``addSublayer``方法添加到``mask``中。

最后，明白了我们的绘制方法，剩下最后的问题就是如何绘制path。为了体现出用户移动轨迹的圆滑边界和手指宽度，我们需要在每次移动之后绘制一个从上一次起点到此次终点的圆柱型path，如下图：
![绘制path.png](http://upload-images.jianshu.io/upload_images/1415843-718b0dae7542ad54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###Code
``ScratchView.h``定义如下:
```
#import <UIKit/UIKit.h>

IB_DESIGNABLE
@interface ScratchView : UIView
@property (nonatomic) IBInspectable CGFloat scratchLineWidth;
@end
```
``scratchLineWidth``用来表示圆柱形轨迹的宽度。

``ScratchView.m``:
```
#import "ScratchView.h"

@interface ScratchView ()
{
    CGPoint startPoint;
}
@property (nonatomic, strong) CALayer * maskLayer;
@end

@implementation ScratchView

- (void) awakeFromNib
{
    [super awakeFromNib];
    self.layer.mask = [CALayer new];
}

- (void) touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch *touch = [[event allTouches] anyObject];
    CGPoint touchLocation = [touch locationInView:self];
    startPoint = touchLocation;
}

- (void) touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch *touch = [[event allTouches] anyObject];
    CGPoint touchLocation = [touch locationInView:self];
    CAShapeLayer * layer = [CAShapeLayer new];
    layer.path = [self getPathFromPointA:startPoint toPointB:touchLocation].CGPath;
    if(!_maskLayer){
        _maskLayer = [CALayer new];
    }
    [_maskLayer addSublayer:layer];
    
    self.layer.mask = _maskLayer;
    startPoint = touchLocation;
}

- (void) touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch *touch = [[event allTouches] anyObject];
    CGPoint touchLocation = [touch locationInView:self];
    CAShapeLayer * layer = [CAShapeLayer new];
    layer.path = [self getPathFromPointA:startPoint toPointB:touchLocation].CGPath;
    if(!_maskLayer){
        _maskLayer = [CALayer new];
    }
    [_maskLayer addSublayer:layer];
    
    self.layer.mask = _maskLayer;
}

- (UIBezierPath *) getPathFromPointA:(CGPoint)a toPointB : (CGPoint) b
{
    UIBezierPath * path = [UIBezierPath new];
    UIBezierPath * curv1 = [UIBezierPath bezierPathWithArcCenter:a radius:self.scratchLineWidth startAngle:angleBetweenPoints(a, b)+M_PI_2 endAngle:angleBetweenPoints(a, b)+M_PI+M_PI_2 clockwise:b.x >= a.x];
    [path appendPath:curv1];
    UIBezierPath * curv2 = [UIBezierPath bezierPathWithArcCenter:b radius:self.scratchLineWidth startAngle:angleBetweenPoints(a, b)-M_PI_2 endAngle:angleBetweenPoints(a, b)+M_PI_2 clockwise:b.x >= a.x];
    [path addLineToPoint:CGPointMake(b.x * 2 - curv2.currentPoint.x, b.y * 2 - curv2.currentPoint.y)];
    [path appendPath:curv2];
    [path addLineToPoint:CGPointMake(a.x * 2 - curv1.currentPoint.x, a.y * 2 - curv1.currentPoint.y)];
    [path closePath];
    return path;
}

CGFloat angleBetweenPoints(CGPoint first, CGPoint second) {
    CGFloat height = second.y - first.y;
    CGFloat width = first.x - second.x;
    CGFloat rads = atan(height/width);
    return -rads;
}

@end
```

在``- (void) awakeFromNib``中执行``self.layer.mask = [CALayer new];``可以把当前view设置为全透。
``- (UIBezierPath *) getPathFromPointA:(CGPoint)a toPointB : (CGPoint) b``方法负责生成两点之间的圆柱型path。
每当用户移动一小段距离之后，我们便创建一个新的``CAShapeLayer``，添加到``mask``中。

**[以上源码](https://github.com/liyaozhong/ScratchCard)**

##Next

* 因为在``touchesMoved``和``touchesEnded``会创建新对象并且add到``mask``中，无疑会持续消耗内存，还是要考虑添加一些path union之类的策略。准备从``CALayer``的``- (BOOL)containsPoint:(CGPoint)p;``方法入手。 
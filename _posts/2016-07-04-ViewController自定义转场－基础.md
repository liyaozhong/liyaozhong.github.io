---
layout: post
title: "ViewController自定义转场－基础"
subtitle: "iOS7新特性详解"
tags: [iOS, ViewController, Transition, Animation]
comments: false
---

从iOS7开始，苹果更新了自定义ViewController转场的API，这些新增的类和接口让很多人困惑，望而却步。本文就从这些API入口，让读者理清这些API错综复杂的关系。

## 几个protocol

讲自定义转场就离不开这几个protocol：
* ``UIViewControllerContextTransitioning``
* ``UIViewControllerAnimatedTransitioning``
* ``UIViewControllerInteractiveTransitioning``
* ``UIViewControllerTransitioningDelegate``
* ``UINavigationControllerDelegate``
* ``UITabBarControllerDelegate``

乍一看很多，其实很简单，我们可以将其分为三类：

1. 描述ViewController转场的：
    ``UIViewControllerTransitioningDelegate``,``UINavigationControllerDelegate``,``UITabBarControllerDelegate``
2. 定义动画内容的
    ``UIViewControllerAnimatedTransitioning``,``UIViewControllerInteractiveTransitioning``
3. 表示动画上下文的
    ``UIViewControllerContextTransitioning``

### 描述ViewController转场的

细说之前先扯个蛋：
>为什么苹果要引入这一套API？因为在iOS7之前，做转场动画很麻烦，要写一大堆代码在ViewController中。引入这一套API之后，在丰富功能的同时极大程度地降低了代码耦合，实现方式就是将之前在ViewController里面的代码通过protocol分离了出来。

顺着这个思路往下想，实现自定义转场动画首先需要找到ViewController的``delegate``。苹果告诉我们切换ViewController有三种形式：UITabBarController内部切换，UINavigationController切换，present modal ViewController。这三种方式是不是需要不同的protocol呢？

我们分别来看下：
* ``UIViewControllerTransitioningDelegate`` 自定义模态转场动画时使用。
  设置UIViewController的属性transitioningDelegate。
  ``@property (nullable, nonatomic, weak) id <UIViewControllerTransitioningDelegate> transitioningDelegate``
* ``UINavigationControllerDelegate`` 自定义navigation转场动画时使用。
  设置UINavigationController的属性delegate
  ``@property(nullable, nonatomic, weak) id<UINavigationControllerDelegate> delegate``
* ``UITabBarControllerDelegate`` 自定义tab转场动画时使用。
  设置UITabBarController的属性delegate
  ``@property(nullable, nonatomic,weak) id<UITabBarControllerDelegate> delegate``

实际上这三个protocol干的事情是一样的，就是我们"扯淡"的内容，只不过他们的应用场景不同罢了。我们下面以``UINavigationControllerDelegate``为例，其他的类似。

### 定义动画内容的

``UINavigationControllerDelegate``主要包含这两个方法：
```
- (nullable id <UIViewControllerInteractiveTransitioning>)navigationController:(UINavigationController *)navigationController
                          interactionControllerForAnimationController:(id <UIViewControllerAnimatedTransitioning>) animationController NS_AVAILABLE_IOS(7_0);

- (nullable id <UIViewControllerAnimatedTransitioning>)navigationController:(UINavigationController *)navigationController
                                   animationControllerForOperation:(UINavigationControllerOperation)operation
                                                fromViewController:(UIViewController *)fromVC
                                                  toViewController:(UIViewController *)toVC  NS_AVAILABLE_IOS(7_0);

```
两个方法分别返回``UIViewControllerInteractiveTransitioning``和``UIViewControllerAnimatedTransitioning``，它们的任务是描述动画行为（转场动画如何执行，就看它俩的）。
从名字可以看出，这两个protocol的区别在于是否是interactive的。如何理解？****interactive动画可以根据输入信息的变化改变动画的进程。****例如iOS系统为``UINavigationController``提供的默认右滑退出手势就是一个interactive 动画，整个动画的进程由用户手指的移动距离控制。

我们来看下相对简单的``UIViewControllerAnimatedTransitioning``:
```
- (NSTimeInterval)transitionDuration:(nullable id <UIViewControllerContextTransitioning>)transitionContext;
- (void)animateTransition:(id <UIViewControllerContextTransitioning>)transitionContext;
@optional
- (void)animationEnded:(BOOL) transitionCompleted;
```
transitionDuration返回动画的执行时间，animateTransition处理具体的动画，animationEnded是optional，大部分情况下不需要处理。
这里出现了我们要讲的最后一个protocol：``UIViewControllerContextTransitioning``。

### 表示动画上下文的
``UIViewControllerContextTransitioning``也是唯一一个不需要我们实现的protocol。
>Do not adopt this protocol in your own classes, nor should you directly create objects that adopt this protocol. 

``UIViewControllerContextTransitioning``提供了一系列方法，为interactive和非interactive动画提供上下文：
```
//转场动画发生在该View中
- (nullable UIView *)containerView;
//上报动画执行完毕
- (void)completeTransition:(BOOL)didComplete;
//根据key返回一个ViewController。我们通过UITransitionContextFromViewControllerKey找到将被替换掉的ViewController，通过UITransitionContextToViewControllerKey找到将要显示的ViewController
- (nullable __kindof UIViewController *)viewControllerForKey:(NSString *)key;
```
还有一些其他的方法，我们以后用到再说。

下面我们通过一个简单的Demo串联理解下。

## DEMO


![transitionDemo.gif](http://upload-images.jianshu.io/upload_images/1415843-0c6ff627250f6134.gif?imageMogr2/auto-orient/strip)

这是一个缩放同时修改透明度的动画，我们来看下如何实现。
在上面的讲解中，我们通过倒推的方式来理解转场动画中用到的protocol，在Demo 中，我们会从创建动画开始。

### 第一步：创建动画
由上面的解析得知，动画是在``UIViewControllerAnimatedTransitioning``中定义的，所以我们首先创建实现``UIViewControllerAnimatedTransitioning``的对象：``JLScaleTransition``。

``JLScaleTransition.h``
```
@interface JLScaleTransition : NSObject<UIViewControllerAnimatedTransitioning>
@end
```

``JLScaleTransition.m``

```
@implementation JLScaleTransition

- (NSTimeInterval)transitionDuration:(nullable id <UIViewControllerContextTransitioning>)transitionContext
{
    return 0.5f;
}

- (void)animateTransition:(id <UIViewControllerContextTransitioning>)transitionContext
{
    UIViewController *toVC = (UIViewController*)[transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    UIViewController *fromVC = (UIViewController*)[transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    UIView * containerView = [transitionContext containerView];
    UIView * fromView = fromVC.view;
    UIView * toView = toVC.view;
    
    [containerView addSubview:toView];
    
    [[transitionContext containerView] bringSubviewToFront:fromView];
    
    NSTimeInterval duration = [self transitionDuration:transitionContext];
    [UIView animateWithDuration:duration animations:^{
        fromView.alpha = 0.0;
        fromView.transform = CGAffineTransformMakeScale(0.2, 0.2);
        toView.alpha = 1.0;
    } completion:^(BOOL finished) {
        fromView.transform = CGAffineTransformMakeScale(1, 1);
        [transitionContext completeTransition:YES];
    }];
}
```
在``animateTransition``中，我们分别获取两个ViewController的view，将``toView``添加到``containerView``中，然后执行动画。为了理解``containerView``和``fromView``,``toView``的关系，我们添加几个log来分析一下：
```
- (void)animateTransition:(id <UIViewControllerContextTransitioning>)transitionContext
{
    UIViewController *toVC = (UIViewController*)[transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    UIViewController *fromVC = (UIViewController*)[transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    UIView * containerView = [transitionContext containerView];
    UIView * fromView = fromVC.view;
    UIView * toView = toVC.view;
    NSLog(@"startAnimation! fromView = %@", fromView);
    NSLog(@"startAnimation! toView = %@", toView);
    for(UIView * view in containerView.subviews){
        NSLog(@"startAnimation! list container subviews: %@", view);
    }
    
    [containerView addSubview:toView];
    
    [[transitionContext containerView] bringSubviewToFront:fromView];
    
    NSTimeInterval duration = [self transitionDuration:transitionContext];
    [UIView animateWithDuration:duration animations:^{
        fromView.alpha = 0.0;
        fromView.transform = CGAffineTransformMakeScale(0.2, 0.2);
        toView.alpha = 1.0;
    } completion:^(BOOL finished) {
        fromView.transform = CGAffineTransformMakeScale(1, 1);
        [transitionContext completeTransition:YES];
        for(UIView * view in containerView.subviews){
            NSLog(@"endAnimation! list container subviews: %@", view);
        }
    }];
}
```

运行log如下：
```
2016-06-29 13:50:48.512 JLTransition[1970:177922] startAnimation! fromView = <UIView: 0x7aaef4e0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x7aaef5a0>>
2016-06-29 13:50:48.513 JLTransition[1970:177922] startAnimation! toView = <UIView: 0x796ac5a0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x796ac050>>
2016-06-29 13:50:48.513 JLTransition[1970:177922] startAnimation! list container subviews: <UIView: 0x7aaef4e0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x7aaef5a0>>
2016-06-29 13:50:49.017 JLTransition[1970:177922] endAnimation! list container subviews: <UIView: 0x796ac5a0; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x796ac050>>
```
可见，转场执行的时候，``containerView``中只包含``fromView``，转场动画执行完毕之后，``containerView``会将``fromView``移除。因为``containerView``不负责``toView``的添加，所以我们需要主动将``toView``添加到``containerView``中。

>注意！非interactive转场中，动画结束之后需要执行``[transitionContext completeTransition:YES];（如果动画被取消，传NO）``；但是在interactive转场中，动画是否结束是由外界控制的（用户行为或者特定函数），需要在外部调用。

### 第二步：定义转场

在第二部，我们需要实现``UIViewControllerAnimatedTransitioning``，并将第一步创建的``JLScaleTransition``对象返回。

``JLScaleNavControlDelegate.h``
```
@interface JLScaleNavControlDelegate : NSObject<UINavigationControllerDelegate>
@end
```

``JLScaleNavControlDelegate.m``

```
@implementation JLScaleNavControlDelegate
- (nullable id <UIViewControllerAnimatedTransitioning>) navigationController:(UINavigationController *)navigationController animationControllerForOperation:(UINavigationControllerOperation)operation fromViewController:(UIViewController *)fromVC toViewController:(UIViewController *)toVC
{
    return [JLScaleTransition new];
}
@end
```

这一步很简单，实现``UIViewControllerAnimatedTransitioning``对应方法即可。

### 第三步：设置转场

设置转场其实就是设置delegate（还记得我们"扯淡"的内容吧）。
```
    self.navigationController.delegate = id<UINavigationControllerDelegate>
    self.transitioningDelegate = id<UIViewControllerTransitioningDelegate>
    self.tabBarController.delegate = id<UITabBarControllerDelegate>
```
设置delegate有两种方式：通过代码；通过StoryBoard。
#### 通过代码设置

```
@interface ViewController ()
@property (nonatomic, strong) JLScaleNavControlDelegate * scaleNavDelegate;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.scaleNavDelegate = [JLScaleNavControlDelegate new];
}

- (IBAction)triggerTransitionDelegate:(id)sender
{
    self.navigationController.delegate = self.scaleNavDelegate;
    [self.navigationController pushViewController:[TargetViewController new] animated:YES];
}
```
#### 通过StoryBoard设置

在StoryBoard中为``Navigation Bar``添加一个Object，并且声明为``JLScaleNavControlDelegate``（定义见上文）。

![storyboard添加delegate](http://upload-images.jianshu.io/upload_images/1415843-5c102fe0334467c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

按住control，从navigation controller拖线置新添加的object，指定为delegate。

![storyboard设置delegate](http://upload-images.jianshu.io/upload_images/1415843-7c45a8b70b8b135e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### NEXT

今天就到这里，源码放在[github](https://github.com/liyaozhong/JLTransition)，还包括一些复杂的动画，会持续更新，后面有时间专门挑几个效果牛逼的聊聊。 
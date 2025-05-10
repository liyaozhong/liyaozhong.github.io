---
layout: post
title: "iOS App Extension入门"
subtitle: "iOS 10新功能背后的技术基础"
tags: [iOS]
comments: false
---

>iOS 10推出了很多新功能，其中有几个高调的变化：通知栏更加实用，电话可以防骚扰，iMessage变得更加有趣和强大，还有就是新一轮的Siri调戏。这些重大功能让我们更加期待iOS10正式上线！作为开发者，我们也需要不断为自己充电，想把握先机？让我们先来看看它们的基本－***App Extension***


##介绍
应用扩展（App Extension）从iOS 8正式登录iOS平台,开发者可以通过应用扩展为用户提供系统特定的扩展功能。开发者通过扩展点(Extension Point)来指定特定的系统功能，如Today扩展，通知栏扩展，Messages扩展，电话簿扩展等等。

##创建应用扩展
应用扩展必须依附于一个**宿主App**(iOS10有更新，见下文)，在现有工程中，选择File->New->Target－>Application Extension来创建一个应用扩展
![创建Extension.png](http://upload-images.jianshu.io/upload_images/1415843-830fe29a5a8180f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

开发者可以根据自己的业务需求创建不同的应用扩展，也可以在一个App中创建多个应用扩展。
>iOS10开始，Messages扩展可独立于宿主App创建:
![创建Messages Application.png](http://upload-images.jianshu.io/upload_images/1415843-c7b2b9e35a4f1c71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Messages Application将自动创建一个Sticker Pack和Messages Extension。
参考[iMessage Apps and Stickers, Part 1 WWDC 2016](https://developer.apple.com/videos/play/wwdc2016/204/) 第7分钟的介绍。


Xcode会为我们自动生成一些文件,有两种类型：
1. 含用户交互界面的
如：Today Extension
![Today Extension.png](http://upload-images.jianshu.io/upload_images/1415843-1aea7c0146381fc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
包含一个ViewController，一个ui和一个plist。
同类型的还有Message Extension(需要Xcode8),  Share Extension, Action Extension等。
2.不含用户交互界面的
如：Share Link Extension

![Share Link Extension.png](http://upload-images.jianshu.io/upload_images/1415843-fb6d9758d6d3ed89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
包含一个Handler和plist。
同类型的还有Call Directory Extension(需要Xcode8)等。

这两种类型的plist大致相同，除了NSExtension这个属性。
含用户交互界面的应用扩展会定义``NSExtensionMainStoryboard``，通常会是MainInterface；而不含用户交互界面的应用扩展则会定义``NSExtensionPrincipalClass``，通常会是XXXHandler（在Share Link Extension的例子中是RequestHandler）。



##生命周期
了解应用扩展的生命周期，**首先**要了解两个概念：
1.容器App （container app）
2.宿主App （host app）

容器App是包含应用扩展的App，就是应用扩展所依赖的App;
宿主App是提供应用扩展界面显示或者功能的App。

**其次:**
虽然容器App包含应用扩展，但它与应用扩展有着完全不同的生命周期，应用扩展与容器App拥有不同的进程。

![应用扩展life cycle](https://developer.apple.com/library/prerelease/content/documentation/General/Conceptual/ExtensibilityPG/Art/app_extensions_lifecycle_2x.png)

1.用户在宿主App中选择应用扩展（查看其UI或者点击启动其功能）。下面会说明

2.系统启动应用扩展。在此期间，宿主App会调用相关方法启动应用扩展，应用扩展可以通过ExtensionContext的到交互信息。

3.应用扩展执行业务逻辑。有些应用扩展在这个阶段不需要和宿主App进行交互，如Share Link Extension；有些应用扩展则需要和宿主App保持联系以监听用户操作或接受命令，如iMessage Extension。

4.应用扩展执行完毕，系统将其终止。

在应用扩展的生命周期中，甚至不需要与容器App有任何交互。实际上，应用扩展运行时，容器App也不需要处于运行状态（除非主动调用）。

###应用扩展响应宿主App请求

 我们在上文提到了**应用扩展的两种类型**，应用扩展的类型不同，响应请求的方式也不同。
有用户交互界面的应用扩展在宿主App中展示的时候启动，并且加载UI。我们可以在ViewController的``loadView``等方法中通过``self.extensionContext``([NSExtensionContext 类型](https://developer.apple.com/reference/foundation/nsextensioncontext))来获取宿主App传给应用扩展的信息。
无用户交互界面的应用扩展在Handler（``NSExtensionPrincipalClass``指定的Handler）中直接接受来自宿主App的请求,调用方法：
```
- (void)beginRequestWithExtensionContext:(NSExtensionContext *)context;
```

当应用扩展响应宿主App，并请求完业务逻辑之后，需要主动告知宿主App。通过调用``NSExtensionContext``的这两个方法：
```
- (void)completeRequestReturningItems:(nullable NSArray *)items completionHandler:(void(^ __nullable)(BOOL expired))completionHandler;
```
```
- (void)cancelRequestWithError:(NSError *)error;
```
>应用扩展执行``completeRequestReturningItems: ``之后，系统可以随时将其终止，如果是有用户交互界面的应用扩展，执行该方法之后，viewController会立刻dismiss（有用户交互界面的应用扩展通常都是通过``presentViewController``的方式显示的）。

##参考
[App Extension Programming Guide](https://developer.apple.com/library/prerelease/content/documentation/General/Conceptual/ExtensibilityPG/index.html#//apple_ref/doc/uid/TP40014214-CH20-SW1)
[NSExtensionContext Class Reference](https://developer.apple.com/reference/foundation/nsextensioncontext) 
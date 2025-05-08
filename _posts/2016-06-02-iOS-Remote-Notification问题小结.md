---
layout: post
title: iOS Remote Notification问题小结
subtitle: 个推SDK使用过程中的常见问题及解决方案
tags: [iOS, Push, 个推, Remote Notification]
comments: false
---

最近在项目中加入了个推的push sdk，遇到了不少蛋疼的问题，在此记录一下，希望能帮到其它读者。
>笔记用到的是个推SDK的透传消息，个推的逻辑是应用在前台的时候，会接收GeTuiSdkDidReceivePayloadData回调，在回调处理中能拿到后台配置的跳转参数。

- **问题1 当有多条push消息的时候，GeTuiSdkDidReceivePayloadData会调用多次，但是用户只点击了一个消息，如何避免其它push信息的回调。**

解决方案是记录用户点击的push消息。
如何记录呢？我们知道，当用户通过点击push消息启动应用的时候，有两种情况：1.应用在后台运行，此时会走到``- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler``。2.应用未启动，此时会走到``- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(nullable NSDictionary *)launchOptions``。我们需要做的就是在这两个方法里拿到一个类似消息ID的东西，在GeTuiSdkDidReceivePayloadData回调中与消息进行比对，如果不一致，则忽略掉。
>个推自身有很多的消息配置，有些需要后台单独设置，笔者采用的方法是比对消息的内容。虽然看起来比较low，但是大部分case还是能完全cover住的。如果读者担心消息内容一致引发的问题，可以参考个推后台的相关配置，在此不作评述。

代码如下：
首先在AppDelegate中声明一个NSString * clickNotifyMsg。
```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    //初始化...
    NSDictionary* remoteNotification = [launchOptions objectForKey:UIApplicationLaunchOptionsRemoteNotificationKey];
    if(remoteNotification && remoteNotification[@"aps"]){
        NSDictionary * aps = remoteNotification[@"aps"];
        if(aps && aps[@"alert"]){
            clickNotifyMsg = remoteNotification[@"alert"];
        }
    }
    return YES;
}
```
```
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler
{
    if(userInfo && userInfo[@"aps"]){
         NSDictionary * aps = userInfo[@"aps"];
         if(aps && aps[@"alert"]){
             clickNotifyMsg = aps[@"alert"];
         }
     }
     completionHandler(UIBackgroundFetchResultNewData);
}
```
通过个推回调拿到的payloadDic是个字典对象，消息内容的key叫content。
```
if(payloadDic && payloadDic[@"content"] && [clickNotifyMsg isEqualToString:payloadDic[@"content"]]){
  //处理逻辑
}
```

- **问题2 收到push消息，点击图标启动应用仍执行GeTuiSdkDidReceivePayloadData回调逻辑。**

这个问题可以在问题1提出的解决方案基础上解决。大体思路是只有需要处理GeTuiSdkDidReceivePayloadData回调的时候才将clickNotifyMsg赋值，否则置为nil。GeTuiSdkDidReceivePayloadData回调中通过判断clickNotifyMsg是否是nil，就可知道是否需要执行相关逻辑。

我们先来看下``- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler``方法的官方文档

>Use this method to process incoming remote notifications for your app. Unlike the application:didReceiveRemoteNotification: method, which is called only when your app is running in the foreground, the system calls this method when your app is running in the foreground or background. In addition, if you enabled the remote notifications background mode, the system launches your app (or wakes it from the suspended state) and puts it in the background state when a remote notification arrives. 
...
>If the user opens your app from the system-displayed alert, the system may call this method again when your app is about to enter the foreground so that you can update your user interface and display information pertaining to the notification.

实际上，如果应用在后台收到push，``- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler``方法会立刻执行一次。当用户点击push消息启动的时候，会执行第二次；如果是通过点击图标启动应用，则不会执行第二次。
如何判断是第一次执行还是第二次执行呢？幸好IOS提供了UIApplicationState这个状态。
```
typedef NS_ENUM(NSInteger, UIApplicationState) {
    UIApplicationStateActive,
    UIApplicationStateInactive,
    UIApplicationStateBackground
} NS_ENUM_AVAILABLE_IOS(4_0);
```
这是UIApplicationState的定义，在此可以简单理解UIApplicationStateActive是前台状态，UIApplicationStateBackground是后台状态，UIApplicationStateInactive就是第二次调用``- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler``时的状态。（具体含义请参考官方文档，在此只是大致理解下）。
修改``- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler``方法如下：
```
if(application.applicationState == UIApplicationStateInactive){
     if(userInfo && userInfo[@"aps"]){
         NSDictionary * aps = userInfo[@"aps"];
         if(aps && aps[@"alert"]){
             clickNotifyMsg = aps[@"alert"];
         }
     }
 }    
 completionHandler(UIBackgroundFetchResultNewData);
```
>另外，记得在GeTuiSdkDidReceivePayloadData回调处理完跳转逻辑后将clickNotifyMsg置空。

最后，再吐槽下个推，**为什么只能配置一个APNs证书！！！** 
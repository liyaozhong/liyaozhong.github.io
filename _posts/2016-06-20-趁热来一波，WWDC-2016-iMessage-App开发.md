---
layout: post
title: "趁热来一波，WWDC 2016 iMessage App开发"
subtitle: "iOS 10新消息应用开发指南"
tags: [iOS, iMessage, WWDC]
comments: false
---

WWDC 2016关于iMessage App的两个视频已经放出（[iMessage Apps and Stickers, Part1](https://developer.apple.com/videos/play/wwdc2016/204/) [iMessage Apps and Stickers, Part2](https://developer.apple.com/videos/play/wwdc2016/224/)）。从iOS 10开始，消息扩展（Messages Extension）可以独立于宿主App(Container App)开发，并且提供了全新的消息模式，开始支持iap和Apple Pay等等。iMessage App有充足的理由让我们兴奋，用现在流行的话来讲，iMessage App将会是一个"风口"。

>本文主要是对最新的WWDC关于iMessage App视频的翻译和笔者的个人理解，英文好的童鞋还是推荐看WWDC的视频资料。阅读本文之前，需要读者掌握基本的iOS Extension知识，可以参考[iOS App Extension入门](http://www.jianshu.com/p/8cf08db29356)

***
## iMessage App简介
iMessage App是一种全新的应用扩展，载体是iOS系统的Message应用，通过iMessage App，用户可以发送更加丰富的消息内容，享受更具交互性的会话体验。我们来看看它都有什么新鲜玩意：

### 新增三种类型
   * Stickers

   * Interactive Messages

   * 可以发送图片，音视频，文本，链接等等


### Messages App Store
就是这货：

![Messages App Store](http://upload-images.jianshu.io/upload_images/1415843-b38f6b2600b6c9cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

用来干什么的呢？

* 显示iMessage App;

* 为未安装应用的用户提供安装途径(Inline App Attribution);

* 提供iap，Apple Pay和访问相机功能。

iMessage App只能在iOS（10+）系统运行，iWatch和mac可以收到消息，但是不能解析。iWatch可以发送Stickers。

iMessage App本身是一个Extension，但是它可以独立开发，不依赖任何Container App。我们也可以在现有的项目中添加iMessage App，系统会自动将其添加到Messages App Store。
如果不想创建Container App，在Xcode 8中，直接创建一个Messages Application(****File->New->Project->Application->Messages Application****)；在现有项目中添加iMessage App，可以通过添加Extension的方式添加(****File->New->Target->Application Extension->Messages Extension****)。

***
## Stickers

Sticker是iOS 10 iMessage引入的一种新的交互方式，可以当做消息发送，也可以附加在已有消息上，支持PNG,APNG,GIF,JPEG的图片格式（推荐使用PNG或APNG格式），图片大小最大500KB。

Stickers的创建非常简单，首先创建一个Sticker Pack Application(****File->New->Project->Application->Sticker Pack Application****)，然后在``Assets.xcassets``中添加一个``Sticker Pack``，然后添加相应的资源文件即可。Sticker Pack Application不需要任何代码，当用户在Messages App Store中打开我们的Messages App时，系统会自动将图片资源显出。系统为Stickers提供了三种尺寸：Large，Medium，Small，我们可以在``Sticker Pack``中进行设置
![sticker size](http://upload-images.jianshu.io/upload_images/1415843-56829cb4c5726b49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Custom Sticker

自定义Stickeryou 有如下优势:

1.自定义UI

2.可以动态改变Sticker

3.可以使用相机功能

4.支持IAP

自定义Sticker需要创建一个Messages Application。Messages Application会自动生成一个Sticker Pack和一个Messages Extension，如下图：
![Messages Application Project](http://upload-images.jianshu.io/upload_images/1415843-9f46eae02502b093.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


>如果想在已有项目中添加Sticker，只能添加一个Messages Extension，然后用自定义Sticker的方法来实现。

自定义Sticker需要我们自己控制Sticker的显示，所以我们不在``Assets.xcassets``中添加stickers，我们需要修改``MessagesExtension``中的``MessagesViewController``。
``MessagesViewController``是``MSMessagesAppViewController``的子类，后者是iMessage App中一个重要的组建，它的生命周期我们会在后面介绍，在此我们只需要知道``MessagesViewController``就是iMessage App的显示界面。

想要自定义Sticker，需要自定义``MSStickerBrowserViewController``。``MSStickerBrowserViewController``顾名思义，是用来显示Sticker的，我们将``MSStickerBrowserViewController``添加到``MessagesViewController``，这样我们的iMessage App的显示界面就是Sticker的界面了。

在``MSStickerBrowserViewController``中重写如下两个方法：

```
@protocol MSStickerBrowserViewDataSource <NSObject>
- (NSInteger)numberOfStickersInStickerBrowserView:(MSStickerBrowserView *)stickerBrowserView;//返回Sticker数量
- (MSSticker *)stickerBrowserView:(MSStickerBrowserView *)stickerBrowserView stickerAtIndex:(NSInteger)index;//返回MSSticker对象
```
``MSStickerBrowserViewDataSource``的用法跟``UITableViewDataSource``很像，甚至连``reloadData``都很像。``MSStickerBrowserViewController``有一个``stickerBrowserView``对象

```
@property (nonatomic, strong, readonly) MSStickerBrowserView *stickerBrowserView;
```
刷新数据的时候，调用``[stickerBrowserView reloadData]``（是不是跟tableView的reloadData很像！）

```
/*!
 @abstract Asks the Sticker Browser View to reload its data from its data source.
 */
- (void)reloadData;
```

>细心的读者可能已经发现，``MSStickerBrowserViewDataSource``和``UITableViewDataSource``有一个不同的地方：
``- (MSSticker *)stickerBrowserView:(MSStickerBrowserView *)stickerBrowserView stickerAtIndex:(NSInteger)index``返回的是一个``MSSticker``，而``- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath``返回的是一个``UIView``。
其实，这是``MSStickerBrowserViewController``帮我们做的一层额外封装，真正显示的是一个叫``MSStickerView``的view。由此可以发现，自定义Sticker不一定非要用``MSStickerBrowserViewController``，我们也可以用一个普通的``UIViewController``，在显示Sticker的地方，我们只需要用``MSStickerView``就可以了。


附一张视频中的截图，方便大家理解
![custom sticker demo](http://upload-images.jianshu.io/upload_images/1415843-cc9571b08a155105.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***
## Interactive Messages

交互型消息(Interactive Messages)是一种特殊的消息类型，由图片、文字、音视频等组成，用户可以对该消息进行交互操作（由iMessage App定义）。

交互型消息由消息扩展(Message Extension)显示和创建，之后由用户主动发送出去；接受者收到消息之后可以点击查看，进行交互操作，并且可以回复消息。这些步骤都在``MSMessagesAppViewController``中完成。

在我们深入了解之前，先来认识几个关键对象。

* ###MSMessagesAppViewController
我们在上面已经介绍过了``MSMessagesAppViewController``，它用来展示消息扩展(Message Extension)的界面。

  ![MSMessagesAppViewController](http://upload-images.jianshu.io/upload_images/1415843-00f9bc33c766ec11.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


  ``MSMessagesAppViewController``有两种展现方式：

* Compact-MSMessagesAppPresentationStyleCompact
* Expanded-``MSMessagesAppPresentationStyleExpanded``
	
  在Compact模式下，不能访问键盘和相机，同时也不能使用横向滑动操作（Compact模式下，横向操作会由系统捕获），但是可以访问用户输入框。
	![Compact Style](http://upload-images.jianshu.io/upload_images/1415843-35e5fb37da0d6cb6.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  在Expanded模式下，我们不能访问用户输入框，但是可以访问键盘、相机，并且可以使用横向滑动操作。
	![Expanded Style](http://upload-images.jianshu.io/upload_images/1415843-2fc13e24e43732bd.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)	

* ###MSConversation
``MSConversation``指当前的会话，我们可以通过``MSConversation``"发送消息"（****这里的发送消息并不是真正的发送消息，只是把消息添加到输入框中，真正的消息发送是用户点击发送按钮完成的****），获取会话的用户信息等。

  ![MSConversation](http://upload-images.jianshu.io/upload_images/1415843-bb2627708998cfe4.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* ###MSMessage
``MSMessage``是消息体，它包含两个主要部分  
  * ``MSSession``用来描述消息如何发送; 
  * ``MSMessageLayout``用来描述消息如何展示

  ![MSMessage](http://upload-images.jianshu.io/upload_images/1415843-8f7716fc63e26aea.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***
## iMessage App LifeCycle

理解了上面的概念，我们来看下Message App的生命周期。

* 启动过程：
    1. Message Extension启动
	2. ``- (void)didBecomeActiveWithConversation:(MSConversation *)conversation;``
	3. ``- (void)viewWillAppear:(BOOL)animated;``
	4. ``- (void)viewDidAppear:(BOOL)animated;``
	
* 销毁过程:
  1. ``- (void)viewWillDisappear:(BOOL)animated;``
  2. ``- (void)viewDidDisappear:(BOOL)animated;``
  3. ``- (void)willResignActiveWithConversation:(MSConversation *)conversation;``
  4. Message Extension被系统销毁

上面列出的回调方法均出现在``MSMessagesAppViewController``中，可见，iMessage App的生命周期就是``MSMessagesAppViewController``的生命周期。当启动Message Extension之后，系统会立刻调用``- (void)didBecomeActiveWithConversation:(MSConversation *)conversation;``方法，在``MSMessagesAppViewController``被用户关闭或通过其它方式dismiss之后，会执行``- (void)willResignActiveWithConversation:(MSConversation *)conversation;``，之后系统随时可以终止Message Extension（一般情况下会立刻终止）。

除此之外，还有一个特殊的过程：

* 唤起过程：
	1. ``-(void)willTransitionToPresentationStyle:(MSMessagesAppPresentationStyle)presentationStyle;``
	2. ``-(void)didTransitionToPresentationStyle:(MSMessagesAppPresentationStyle)presentationStyle;``
	
唤起过程发生在``MSMessagesAppViewController``的``presentationStyle``发生变化时。

>PS:Message Extension启动的时候，总是以Compact Style的方式显示。

##创建 MSMessage

``MSMessage``就是消息本身，我们可以通过修改``MSMessage``的属性来决定消息的展示内容。消息的展现方式由``MSMessageLayout``决定，不过``MSMessageLayout``是一个抽象类，目前系统只提供了一种展现方法``MSMessageTemplateLayout``，如下图

![MSMessageTemplateLayout](https://docs-assets.developer.apple.com/published/af521ba258/MSMessageTemplateLayout_2x_93d9e9b7-b99c-4def-a8e1-2df50a710a52.png)

我们可以设置``MSMessage``的``URL``属性，用来描述消息内容。因为iMessage App只能在iOS系统运行，mac收到该消息之后不会解析消息内容，如果用户点击该消息并且``URL``是http(s)类型的，系统会通过浏览器打开相应的页面。


##"发送 Message"

我们所说的"发送 Message"实际上是添加消息到用户输入框。这一过程由``MSConversation``完成。在``MSMessagesAppViewController``中我们可以通过成员变量``activeConversation``获取当前的``MSConversation``，随后可调用一下方法插入不同的消息。

```
- (void)insertMessage:(MSMessage *)message localizedChangeDescription:(nullable NSString *)changeDescription completionHandler:(nullable void (^)(NSError * _Nullable))completionHandler;
- (void)insertSticker:(MSSticker *)sticker completionHandler:(nullable void (^)(NSError * _Nullable))completionHandler;
- (void)insertText:(NSString *)text completionHandler:(nullable void (^)(NSError * _Nullable))completionHandler;
- (void)insertAttachment:(NSURL *)URL withAlternateFilename:(nullable NSString *)filename completionHandler:(nullable void (^)(NSError * _Nullable))completionHandler;
```

其中第一个方法添加的是交互型消息，其余三个依次添加Sticker，文本以及音视频。

---

###今天到此为止，WWDC视频还提到了``MSSession``，如何聚合消息，获取会话者信息等，我们以后再聊！建议大家去认真看下原版视频！最后，如果你觉得本文对你有帮助，请点赞😊！ 
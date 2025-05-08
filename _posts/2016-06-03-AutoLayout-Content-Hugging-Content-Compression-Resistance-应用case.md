---
layout: post
title: "[AutoLayout] Content Hugging & Content Compression Resistance 应用case"
subtitle: "iOS AutoLayout中两个优先级属性的实际应用"
tags: [iOS, AutoLayout, UI]
comments: false
---

>AutoLayout是苹果大力推广的view布局方法，但是做过Android的人都会感觉iOS的AutoLayout简直弱爆了。实际上，iOS的AutoLayout还是有它独特的优势，可以轻松实现某些特定的布局。今天我们讨论Content Hugging 和 Content Compression Resistance这两个优先级。

前段时间面试过很多iOS开发，问到自动布局的时候，很多人只会简单使用，甚至都不清楚Content Hugging 和 Content Compression Resistance是干嘛用的。我们不会在本篇文章讨论这两个属性，不熟悉的可以上网搜搜看，英文好的可以看下[这篇文章](http://krakendev.io/blog/autolayout-magic-like-harry-potter-but-real)。我们今天只讨论一个具体case，也是我在面试AotoLayout中必问的一个问题（不幸的是，目前为止还没有一个人的答案令我满意）。
## 亮题：
我需要在一个view中放入两个单行的label，一个居左一个居右，并且文字长度不固定。要求如下：
1.每个label只显示单行文字，超出部分显示...
2.两个label最小间距为10pixel
3.尽量把右边的label显示全
4.用AutoLayout实现

##### 不要着急往下看，请先思考一分钟。#####

![](http://upload-images.jianshu.io/upload_images/1415843-15609a54ca1e5577.gif?imageMogr2/auto-orient/strip)


不少面试者会马上说，label2.leading >= lable1.trailing + 10。毫无疑问，这是有问题的，因为它不满足要求3。当两个label显示的文字都很长时，Layout Engine会优先满足左侧label的显示需求。

此时，我们需要用到Content Hugging 和 Content Compression Resistance。如果你觉得这两个名字很难理解，可以简单理解为***Content Hugging越大，view越难变大，Content Compression Resistance越大，view越难变小***。
为方便理解，我们做一个简单的Demo


![demo1.png](http://upload-images.jianshu.io/upload_images/1415843-2688c6c24463788b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

黄色区域包含两个label，下面的两个TextField分别控制label的文字。

在我们的case里，只存在horizontal的情况，所以不需要考虑这两个属性的vertical值。Content Hugging Priority默认值是250，Content Compression Resistance Priority默认值是750。
本着"***Content Hugging越大，view越难变大，Content Compression Resistance越大，view越难变小***"的原则，我们将label1的Content Hugging Priority设置为251

![label1.png](http://upload-images.jianshu.io/upload_images/1415843-e23990002e8b8497.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时将label2的Content Compression Resistance设置为751

![label2.png](http://upload-images.jianshu.io/upload_images/1415843-b9e8fd5e74dfbb58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行，效果如下：

![demo.gif](http://upload-images.jianshu.io/upload_images/1415843-c2b9e672d924b479.gif?imageMogr2/auto-orient/strip)

##### 欢迎拍砖。 
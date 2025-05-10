---
layout: post
title: "bringToFront 的意外发现"
subtitle: "Android View层级管理中的隐藏细节"
tags: [Android, 源码分析]
comments: false
---

最近在项目用到了``View.bringToFront()``方法，简单看了下其源码，在这儿总结一下。

``bringToFront``方法在SDK中的说明是"Change the view's z order in the tree, so it's on top of other sibling views"，翻译过来就是改变view的z轴，使其处在他父view的顶端。关于bringToFront的实现，网上有很多资料介绍，大体来说就是将这个view从父view中移除，然后再加入父view的顶端。具体实现如何呢？

``bringToFront``的具体实现要参看ViewGroup中的``bringChildToFront``方法。代码很简单，如下：

```
public void bringChildToFront(View child) {
        int index = indexOfChild(child);
        if (index >= 0) {
            removeFromArray(index);
            addInArray(child, mChildrenCount);
            child.mParent = this;
        }
    }
```
分两步，首先remove，然后add，实际上ViewGroup维持了一个View数组，``addInArray``方法会把这个child加入到数组最末端，在draw的时候，将依次画数组中的child，最后一个自然就放到了顶端。
``removeFromArray``方法中有一段奇怪的代码：
```
if (!(mTransitioningViews != null && mTransitioningViews.contains(children[index]))) {
            children[index].mParent = null;
}
```
mTransitioningViews官方解释如下：
>The set of views that are currently being transitioned. This list is used to track views being removed that should not actually be removed from the parent yet because they are being animated.

正如上述，当某个child正在做动画的时候(这里指``android.app.Fragment``和``android.animation.LayoutTransition``移除view的动画)，还不能删除这个child，应该等到动画结束，所以在ViewGroup中暂时保留这个child，直到动画真正结束后再真正删除。ViewGroup有两个成对出现的方法：``startViewTransition``和``endViewTransition``。在``startViewTransition``方法中将child加入mTransitioningViews中，在``endViewTransition``中最终执行``view.dispatchDetachedFromWindow()``，并在函数最后调用``invalidate()``。

值得一提的是，在``removeView``的时候，如果当前child在``mTransitioningViews``中，ViewGroup并不会执行``view.dispatchDetachedFromWindow()``，也不会设置当前view的mParent为空。

没想到分析``bringToFront``方法竟然还意外发现一个mTransitioningViews，由此可以看到一个潜在的问题，如果我们在执行LayoutTransition的DISAPPEARING动画同时removeView，这时子view还并未删除，如果直接将子view加入其它ViewGroup中则会报错"The specified child already has a parent. You must call removeView() on the child's parent first." 因为此时这个view还未从上一个ViewGroup中删除。 
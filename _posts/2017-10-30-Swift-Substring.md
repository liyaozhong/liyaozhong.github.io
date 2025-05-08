---
layout: post
title: "Swift Substring"
subtitle: "Swift字符串切片的实现原理"
tags: [Swift, Memory]
comments: false
---

在swift中，当我们使用``split``方法的时候会返回一个``Substring``数组：
``    public func split(maxSplits: Int = default, omittingEmptySubsequences: Bool = default, whereSeparator isSeparator: (Character) throws -> Bool) rethrows -> [Substring]
``

这跟java或者ObjectC不同，在其他语言中我们将会得到一个``String``数组，为何swift返回一个新的类型``Substring``呢？

###### 解释这个问题之前，先来看另外一个问题：

###### 如何提升``split``方法的效率？

``split``方法就是根据特定字符串将原数组切分成不同的部分，我们知道``String``的存储比较特殊，一般都是在堆上，如果切分之后的子串也都存在堆上，无疑是一个巨大的开销，况且方法调用者并不一定要用到每一个子串。

怎么办呢？

那就**共享内存**！

>**Substring**
When you create a slice of a string, a Substring instance is the result. Operating on substrings is fast and efficient because a substring shares its storage with the original string. The Substring type presents the same interface as String, so you can avoid or defer any copying of the string's contents.

``split``方法返回的每一个字符串子串不额外开辟内存空间，而是使用原数组的地址，这样就可以省下分配空间的开销。这是内部实现，无论是使用``Substring``还是``String``都可以做到，那么回到最初的问题，``Substring``有啥优势？


###### 答案就是，这是一种优秀的编程思想！

考虑下面的case：

```
let bigContent : String = xxxx//获取一大段文字
let partOfIt = splitContent(bigContent)//截取一小部分
summaryLabel.text = partOfIt//设置UILabel
```

根据上面的分析我们知道``Substring``和原字符串是共享内存的，因此当上述逻辑执行完毕之后，只要``summaryLabel``没有销毁，``bigContent``所指向的字符串就不会释放，即使我们只使用了该字符串的一部分。

为避免上述**内存泄漏**情况的出现，我们应当给``summaryLabel``分配一个新的``String``：
```
summaryLabel.text = String(partOfIt)
```

这样``bigContent``该释放就释放，和``summaryLabel``不再相关。

那为什么要用``Substring``？因为如果``split``返回的是``[String]``，粗心的程序员很难会考虑到这么深入的问题，内存泄漏很容易发生。因此在API设计的时候，设置一个新的Type``Substring``,如果写的是``summaryLabel.text = partOfIt``编译器会报错，提示类型转化。

>**Important**
Don't store substrings longer than you need them to perform a specific operation. A substring holds a reference to the entire storage of the string it comes from, not just to the portion it presents, even when there is no other reference to the original string. Storing substrings may, therefore, prolong the lifetime of string data that is no longer otherwise accessible, which can appear to be memory leakage. 
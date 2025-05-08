---
layout: post
title: "Swift Errors"
subtitle: "Swift常见错误解析"
tags: [Swift]
comments: false
---

#### fatal error: unexpectedly found nil while unwrapping an Optional value

>It means your variable is set to nil, but your code is expecting it to not be nil.

什么叫"your code is expecting it to not be nil"呢？来看个例子：
```
var name : String?
name = nil
name!.uppercaseString//fatal error: unexpectedly found nil while unwrapping an Optional value
```
[参考](http://jamesonquave.com/blog/understanding-the-fatal-error-cant-unwrap-optional-none-errors-in-swift/)

---
#### Initializer requirement 'XXXX' can only be satisfied by a  required initializer in the definition of non-final class 'XXXX'

意思就是你声明的'XXXX'这个构造器要么加上```required```关键字，要么把你的类改成```final  class```。

为什么呢？

出现这种情况，一般是在自定义的'XXXX'构造器由于某种原因必须在子类中重写，在子类中重写的构造器必须加```required```关键字。当然如果是```final class```也就不用操这个心了，因为压根不会有子类。

>关于swift构造器可以参考[Swift initializer](http://www.jianshu.com/p/bdb07143d368)

[参考](http://stackoverflow.com/questions/26578664/how-to-satisfy-a-protocol-which-includes-an-initializer/) 
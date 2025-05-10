---
layout: post
title: "Swift tricks-Nonmutating"
subtitle: "Swift中的非可变方法修饰符"
tags: [Swift]
comments: false
---

>***Swift tricks***系列收集Swift牛逼的patterns和让你代码更加Swifty的tricks，持续更新中……

开始今天的话题前，我们先来看几行代码：
```
let unsafemutablepointer : UnsafeMutablePointer<Int>
unsafemutablepointer = UnsafeMutablePointer.alloc(3)
unsafemutablepointer[0] = 1
unsafemutablepointer[1] = 2
unsafemutablepointer[2] = 3
///////////
let intArray : Array<Int> = [1,2,3]
intArray[1] = 3
```

****上面代码有什么问题？****

![](http://upload-images.jianshu.io/upload_images/1415843-a4a8fc9d40f6983f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可能你已经想到了，编译器会报错，因为我们在尝试修改一个常量数组。


>"If you create an instance of a structure and assign that instance to a constant, you cannot modify the instance's properties, even if they were declared as variable properties"
"This behavior is due to structures being value types. When an instance of a value type is marked as a constant, so are all of its properties."

![error!.png](http://upload-images.jianshu.io/upload_images/1415843-1685e557bfb4022f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么，另外一个问题来了：同样是用下标访问，同样是值类型(Value Type)，为什么``UnsafeMutablePointer``不会报错？


``UnsafeMutablePointer``和``Array``都是``struct``，存储的类型也一样都是``Int``，额……

---

Array:
```
public subscript (index: Int) -> Element
```
UnsafeMutablePointer:
```
public subscript (i: Int) -> Memory { get nonmutating set }
```
```nonmutating```是什么东西？！

---

### nonmutating

```nonmutating```是swift的一个关键字，它和```mutating```一样用来修饰方法，只不过```nonmutating```一般用来修饰setter方法。
当用```nonmutating```修饰setter方法的时候，swift编译器会知道该方法并不会改变常量变量，因此不会报错。
***使用nonmutating有一定的危险性，因为它跳过了swift编译器的安全检查机制。不过我们可以在合适的情况下使用它，写出更加灵活的代码***

假设我们想通过下标访问``NSUserDefaults``，我们会这么做：
```
extension NSUserDefaults {
    subscript(key: String) -> AnyObject? {
        get { return objectForKey(key) }
        set { setObject(newValue, forKey: key) }
    }
}

var defaults: NSUserDefaults = NSUserDefaults.standardUserDefaults()
defaults["key"] = "value"
```
不幸的是，```defaults```是mutable的，这就意味着它可以被别人重新赋值，进而引发问题。使用```nonmutating```，我们就可以这样做：
```
protocol NSUserDefaultsSubscipt {
    subscript(key: String) -> AnyObject? { get nonmutating set }
}
extension NSUserDefaults:NSUserDefaultsSubscipt {
    subscript(key: String) -> AnyObject? {
        get { return objectForKey(key) }
        set { setObject(newValue, forKey: key) }
    }
}
let defaults: NSUserDefaults = NSUserDefaults.standardUserDefaults()
defaults["key"] = "value"
```
现在再看```defaults```是不是有点class的感觉？

[参考:The why of nonmutating](https://medium.com/swift-programming/the-why-of-nonmutating-7ecd2cf17ecf#.7pm6l44zy) 
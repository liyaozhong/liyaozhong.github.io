---
layout: post
title: "Swift tricks-Enum Associated Value & Raw Value"
subtitle: "Swift枚举的高级特性"
tags: [Swift]
comments: false
---

>***Swift tricks***系列收集Swift牛逼的patterns和让你代码更加Swifty的tricks，持续更新中……

### Associated Value
Swift的``enum``和Object-C中的``enum``一样都不能拥有属性，但是Swift提供了一个非常强大的功能——Associated Value。它可以让``enum``的``case value``存储不同类型的值。
引用苹果官方的例子:
```
enum Barcode {
   case upc(Int, Int, Int, Int)
   case qrCode(String)
}
```
我们可以这样创建``Barcode``：
```
var productBarcode = Barcode.upc(8, 85909, 51226, 3)
productBarcode = .qrCode("ABCDEFGHIJKLMNOP")
```
我们可以通过``switch``语法将Associated Value取出来：
```
switch productBarcode {
   case .upc(let numberSystem, let manufacturer, let product, let check):
      print("UPC: \(numberSystem), \(manufacturer), \(product), \(check).")
   case .qrCode(let productCode):
      print("QR code: \(productCode).")
}
```
### Raw Value
``enum``可以通过``raw value``对其进行预填充。例如，我们可以用一个Int类型的rawValue来表示硬币的正反面:
```
enum Coin:Int{
    case Head = 1
    case Tail
}
```
``Coin.Head.rawValue``为1，``Coin.Tail.rawValue``为2。
我们也可以用String类型的rawValue来表示Icon:
![rawValue.png](http://upload-images.jianshu.io/upload_images/1415843-3cc06faf5a69e993.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们甚至可以直接通过rawValue初始化``enum``：

![rawValue init.png](http://upload-images.jianshu.io/upload_images/1415843-613cfcbe8eb97ff7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以在定义``enum``的时候指定rawValue的类型（一般会是Int, String, Character,Float等）。如果不指定，那么就没有rawValue属性。如下面的``Coin.Head``就没有rawValue。
>enum case cannot have a raw value if the enum does not have a raw type

```
enum Coin{
    case Head
    case Tail
}
```

rawValue的本质是一个名为``RawRepresentable``的protocol：
```
public protocol RawRepresentable {
    associatedtype RawValue
    public init?(rawValue: Self.RawValue)
    public var rawValue: Self.RawValue { get }
}
```
Int, String, Character, Float等都实现了这个protocol。

---

下面我们会通过一个例子来深入理解``Associated Value``和``Raw Value``

### 举个🌰

假设我们在做一个学生的评分系统，我们需要一个``enum``来表示学生的成绩:
```
enum Score {
    case Fail
    case Pass
    case Good
    case Perfect
}

var input = 70;
let score : Score

if(input < 60){
    score = .Fail
}else if(input < 80){
    score = .Pass
}else if(input < 90){
    score = .Good
}else{
    score = .Perfect
}
```
上面的做法不怎么优雅，我们可以这样做：
```
enum Score2 {
    case Fail
    case Pass
    case Good
    case Perfect
    
    init(_ score : Int){
        if(score < 60){
            self = .Fail
        }else if(score < 80){
            self = .Pass
        }else if(score < 90){
            self = .Good
        }else{
            self = .Perfect
        }
    }
}

let score2 = Score2(85)
```
这样一来，判断逻辑就统一了，也更容易维护。

但是需求往往会比较复杂，例如老师在宣读成绩的时候需要写评语，另外会表扬成绩优秀的学生：
```
func writeComment(score : Int) -> () {
    var comment : String
    switch Score2(score) {
    case .Fail:
        comment = "表现不好，要努力啊!"
    case .Pass:
        comment = "加油！"
    case .Good:
        comment = "不错！"
    case .Perfect:
        comment = "非常好！"
    }
    print("comment:\(comment), score:\(score)")
}

func applauseStudent(score : Int) -> () {
    switch Score2(score) {
    case .Perfect:
        print("你是我的骄傲, 你考了\(score)")
    default: break
    }
}

func announceScore(score : Int) -> () {
    writeComment(score)
    applauseStudent(score)
}
```

``writeComment``和``applauseStudent``两个方法都用到了``score``和``enum``，于是我们对同一个``enum``初始化了两次(虽然不是什么大事，但是我们还是要精益求精)。

改成下面这样?
#### 方案2
```
func writeComment(score : Int, grade : Score2) -> ()
func applauseStudent(score : Int, grade : Score2) -> ()
```
没次传两个参数，解决了问题，但是有点ugly。有没有更Swifty的方法？
#### 方案3
我们可以用``associated value``，将分数存储在``enum``中:
```
enum Score2 {
    case Fail(Int)
    case Pass(Int)
    case Good(Int)
    case Perfect(Int)
    
    init(_ score : Int){
        if(score < 60){
            self = .Fail(score)
        }else if(score < 80){
            self = .Pass(score)
        }else if(score < 90){
            self = .Good(score)
        }else{
            self = .Perfect(score)
        }
    }
}
```
```
func writeComment(score : Score2) -> () {
    var comment : String
    var grade : Int
    switch score {
    case .Fail(let g):
        comment = "表现不好，要努力啊!"
        grade = g
    case .Pass(let g):
        comment = "加油！"
        grade = g
    case .Good(let g):
        comment = "不错！"
        grade = g
    case .Perfect(let g):
        comment = "非常好！"
        grade = g
    }
    print("comment:\(comment), score:\(grade)")
}

func applauseStudent(score : Score2) -> () {
    switch score {
    case .Perfect(let grade):
        print("你是我的骄傲, 你考了\(grade)")
    default: break
    }
}

func announceScore(score : Int) -> () {
    let s = Score2(score)
    writeComment(s)
    applauseStudent(s)
}
```
#### 方案4
方案3看起来很不错，但是我们在switch中做了大量的值提取操作。下面来介绍终极方案：``associated value`` + ``raw value``

```
enum Score3: RawRepresentable{
    case Fail(Int)
    case Pass(Int)
    case Good(Int)
    case Perfect(Int)
    
    typealias RawValue = Int

    var rawValue: RawValue {
        var grade : Int
        switch self {
        case .Fail(let g):
            grade = g
        case .Pass(let g):
            grade = g
        case .Good(let g):
            grade = g
        case .Perfect(let g):
            grade = g
        }
        return grade
    }
    init?(rawValue: RawValue) {
        if(rawValue < 60){
            self = .Fail(rawValue)
        }else if(rawValue < 80){
            self = .Pass(rawValue)
        }else if(rawValue < 90){
            self = .Good(rawValue)
        }else{
            self = .Perfect(rawValue)
        }
    }
}
```
```
func writeComment(score : Score3) -> () {
    var comment : String
    switch score {
    case .Fail:
        comment = "表现不好，要努力啊!"
    case .Pass:
        comment = "加油！"
    case .Good:
        comment = "不错！"
    case .Perfect:
        comment = "非常好！"
    }
    print("comment:\(comment), score:\(score.rawValue)")
}

func applauseStudent(score : Score3) -> () {
    switch score {
    case .Perfect:
        print("你是我的骄傲, 你考了\(score.rawValue)")
    default: break
    }
}

func announceScore(score : Int) -> () {
    let s = Score3(rawValue: score)
    writeComment(s!)
    applauseStudent(s!)
}
```

[参考:Swift: Raw{Not}Representable enum](http://blog.krzyzanowskim.com/2015/03/12/swift-raw-not-representable-enum/) 
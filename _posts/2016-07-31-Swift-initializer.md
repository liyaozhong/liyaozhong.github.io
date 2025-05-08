---
layout: post
title: "Swift initializer"
subtitle: "Swift构造器详解"
tags: [Swift, Initializer]
comments: false
---

今天聊聊Swift中的构造器，无论你是Swfit新手还是混迹论坛多年的老鸟，相信你看完本文之后一定会有所收获。

我们将讨论以下四部分内容：
- designated initializer
- convenience initializer
- required 关键字
- protocol中的initializer

---

### Designated Initializer

当我们创建一个``class``的时候，如果这个``class``拥有****未初始化的非Optional属性****，则必须提供一个构造器(称为designated initializer)。这句话有两个重点：
****1.未初始化；2.非Optional属性。****
比较一下下面几种写法：

```
//A
class Person {
    let name : String = ""
}
```

```
//B
class Person {
    let name : String
}
```

```
//C
class Person {
    var name : String?
}
```

```
//D
class Person {
    let name : String?
}
```
A,C都可以正常编译，B,D会报错(``error: class 'Person' has no initializers``)。因为Swift不允许非Optional变量在没有初始化的情况下使用或者访问，这样会造成潜在的风险，于是Swift编译器对构造器做了严格的限制:类必须提供一个designated initializer，并且初始化所有****未初始化的非Optional属性****

---

 A,B两种写法中，``let``换成``var``结果是一样的。但是D比较特殊，在Swift中，常量在访问和使用之前必须初始化(哪怕是Optional)
```
var p : String?
print(p)
//正常编译
```
```
let p : String?
print(p)
//报错：error: constant 'p' used before being initialized
```

---

由此可见我们上面的描述并不准确，应当是****未初始化的非Optional属性或Optinal常量****。不过一般我们不会声明一个Optional常量，另外这么说也是在太过拗口，所以就简单称之为****未初始化的非Optional属性****吧(下文也会这样表述)。


同样，在子类初始化过程中，Swift也做了严格的限制
```
class Person {
    let name : String
    init(name : String){
        self.name = name
    }
}
class Engineer: Person {
    var title : String
    init(name: String, title: String) {
        self.title = title
        super.init(name: name)
    }
}
```
****首先****，``Engineer``必须调用``super.init``，如果你这样写：
```
    init(name: String, title: String) {
        self.title = title
        self.name = name
    }
```
不好意思，``self.name``是常量，不可修改。即便我们将``let name : String``改为``var name : String``，仍会报错，而且是两个：
- ``error: super.init isn't called on all paths before returning from initializer``
- ``error: use of 'self' in property access 'name' before super.init initializes self``

从这两个错误信息可以看出，在子类的构造器中，父类的成员变量必须通过父类的designated initializer进行初始化。也就是说，我们必须调用``super.init``（第一个error），而且在调用``super.init``之前不能使用和访问任何父类的成员变量（第二个error）。

****其次****,``super.init``必须在子类所有****未初始化的非Optional属性****初始化之后调用。

---

这跟我们以前所了解的不同，例如在JAVA的构造器中我们会先调用``super.init``。Swift之所以这样做，同样是为了安全性。
假设我们要求在初始化一个``Person``之后，立刻输出一些信息用来debug：
```
class Person {
    var name : String
    init(name : String){
        self.name = name
        printDescription()
    }
    
    func printDescription() -> () {
        print("name=\(name)")
    }
}
class Engineer: Person {
    var title : String
    init(name: String, title: String) {
        self.title = title
        super.init(name: name)
    }
    
    override func printDescription() -> () {
        print("name=\(name), title=\(title)")
    }
}
```
如果在``Engineer``的``init``中，我们首先调用``super.init``，那么我们就会在初始化``title``之前执行子类的``printDescription``方法，这是Swift所不允许的。

---

### Convenience Initializer

一个类可以提供多个designated initializer，但是designated initializer不可以调用designated initializer。
如果需要提供一个依赖designated initializer的构造器，需要添加关键字``convenience``，称为convenience initializer。
在上面的例子中，假设我们从文件中读取人的信息，读取到的是****"名字;职位;联系方式"****格式的字符串。我们需要写一个构造器方法从这串文字中解析出相应的信息。
```
class Person {
    let name : String
    init(name : String){
        self.name = name
    }
    
    convenience init(convertFromLiteral value : String){
        var name = ""
        let components = value.componentsSeparatedByString(";")
        for component in components{
            if(name.characters.count == 0){
                name = component
                break
            }
        }
        self.init(name: name)
    }
}
```
因为新的构造器调用了``self.init``，因此需要添加关键字``convenience``，否则会报错：``error: designated initializer for 'Person' cannot delegate (with 'self.init'); did you mean this to be a convenience initializer?``

一个类可以拥有多个convenience initializer，而且convenience initializer可以调用其他convenience initializer。
convenience initializer有以下约束：
- 必须调用当前类的designated initializer，调用父类的designated initializer是不行的。
- 只有当前类可以使用（子类也可以使用，不过需要特殊处理。见下文）。

---

### Required 关键字

如果父类中的构造器需要在子类中重写，可以使用``required``关键字。子类必须重写父类中标记为``required``的构造器方法(不管是designated initializer还是convenience initializer)。
上面的例子中，``init(convertFromLiteral value : String)``并不适合子类，所以我们加上``required``关键字，编译器会提示我们``Engineer``没有实现required initializer。
```
class Engineer: Person {
    var title : String
    
    init(name: String, title: String) {
        self.title = title
        super.init(name: name)
    }
    
    required convenience init(convertFromLiteral value : String){
        var name = ""
        var title = ""
        let components = value.componentsSeparatedByString(";")
        for component in components{
            if(name.characters.count == 0){
                name = component
            }else if(title.characters.count == 0){
                title = component
                break
            }
        }
        self.init(name: name, title: title)
    }
}
```

上文提到了convenience initializer只有当前类才能使用，Swift不允许子类通过父类构造器进行初始化是显而易见的。convenience initializer实际上更像一个方法，如果我们将某个convenience initializer所依赖的designated initializer标记为``required``，就相当于告诉了编译器我们的子类一定重写了这个designated initializer。也就是说，如果我们的子类通过这个convenience initializer进行初始化，最终会执行子类中重写的designated initializer。

说这么一大堆，用代码看的更清楚：
```
class Person {
    let name : String
    required init(name : String){
        self.name = name
    }
    
    convenience init(convertFromLiteral value : String){
        var name = ""
        let components = value.componentsSeparatedByString(";")
        for component in components{
            if(name.characters.count == 0){
                name = component
                break
            }
        }
        self.init(name: name)
    }
}

class Engineer: Person {
    var title : String
    
    required init(name: String) {
        self.title = "unknow"
        super.init(name: name)
    }
    
    init(name: String, title: String) {
        self.title = title
        super.init(name: name)
    }
}

let peter = Person.init(convertFromLiteral: "peter;")
print(peter.name)
let tom = Engineer.init(convertFromLiteral: "tom;")
print(tom.name + ", " + tom.title)
```
将父类中convenience initializer所依赖的designated initializer标记为``required``，我们就可以通过该convenience initializer初始化子类了。

---

### Protocol中的Initializer


Swift提供了很多包涵构造器的``protocol``，如``StringLiteralConvertible``，``DictionaryLiteralConvertible``，``ArrayLiteralConvertible``等。

如果我们想通过如下方式初始化，我们就需要用到``StringLiteralConvertible``
```
let peter : Person = "Peter"
```
代码如下：
```
class Person: StringLiteralConvertible{
    let name: String
    init(name value: String) {
        self.name = value
    }
    
    required convenience init(stringLiteral value: String) {
        self.init(name: value)
    }
    
    required convenience init(extendedGraphemeClusterLiteral value: String) {
        self.init(name: value)
    }
    
    required convenience init(unicodeScalarLiteral value: String) {
        self.init(name: value)
    }
}

let peter : Person = "Peter"
```
是不是很cool？

>需要注意的是，Swfit很好的控制了构造器的作用域（默认情况下，convenience initializer是子类不可见的），但是``protocol``中的方法是子类可见的。所以``protocol``中的构造器方法都必须声明为``required``。
如果实现``protocol``的是一个``final``类，那么则没必要再是``required``，因为它不会有子类。

---

等一下，如果``Person``是第三方库中的类呢？
你可能会说没问题啊，我们可以通过``extension``实现``StringLiteralConvertible``
```
class Person: StringLiteralConvertible{
    let name: String
    init(name value: String) {
        self.name = value
    }
    
}

extension Person: StringLiteralConvertible{
    required convenience init(stringLiteral value: String) {
        self.init(name: value)
    }
    
    required convenience init(extendedGraphemeClusterLiteral value: String) {
        self.init(name: value)
    }
    
    required convenience init(unicodeScalarLiteral value: String) {
        self.init(name: value)
    }
}

let peter : Person = "Peter"
```
真是这样吗？Sorry!!!
- 首先，required initializer必须在类中声明。所以上面的``required``关键字要不得
- 其次，``protocol``中的构造器方法必须声明为``required``，除非``Person``是``final``的。

emmm....

### What Else

``struct``和``enum``同样可以拥有构造器，不过上面提到的designated initializer，convenience initializer，``required`` 关键字都是针对``class``的,不要搞混。
最后上段代码，供大家把玩
```
enum Sex{
    case Male
    case Female
    case Unknow
}
let malesName = ["peter", "tom"]
let femalesName = ["marry", "taylor"]
extension Sex: StringLiteralConvertible{
    init(stringLiteral value: String) {
        if(malesName.contains(value)){
            self = .Male
        }else if(femalesName.contains(value)){
            self = .Female
        }else{
            self = .Unknow
        }
        print(self)
    }
    init(extendedGraphemeClusterLiteral value: String) {
        self.init(stringLiteral value)
    }
    init(unicodeScalarLiteral value: String) {
        self.init(stringLiteral value)
    }
}

var sex : Sex = "tom"
sex = "marry"
sex = "gollum"
``` 
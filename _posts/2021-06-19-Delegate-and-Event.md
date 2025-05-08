---
layout: post
title: "Delegate and Event"
subtitle: "C#委托与事件详解"
tags: [C#, Delegate, Event, Design Pattern]
comments: false
---

在C#中，`delegate`（委托）和`event`（事件）是两个相关但不同的概念。虽然它们常常一起使用，但它们各自有不同的作用和用途。

### Delegate（委托）

**定义**：

- `delegate` 是一种类型，类似于函数指针，可以指向任何与其签名匹配的方法。它定义了方法的签名（返回类型和参数），并允许你存储对这种方法的引用。

**特点**：

- 委托可以在方法内部或外部声明，灵活性较高。
- 委托可以直接调用，且支持多播，即可以存储多个方法并依次调用它们。
- 委托常用于回调机制，允许将方法作为参数传递，并在适当的时间点调用这些方法。

**示例**：

```csharp
// 定义一个委托类型
delegate void MyDelegate(string message);

// 使用委托
MyDelegate del = Console.WriteLine;
del("Hello, World!");  // 直接调用委托
```

### Event（事件）

**定义**：

- `event` 是基于委托的一种特殊机制，主要用于发布/订阅模式。在这种模式下，一个对象（发布者）可以通知多个订阅者某个事件发生了。事件通常用于提供一种更为安全的方式，让外部对象订阅并响应某个动作。

**特点**：

- 事件只能在方法外部声明，通常用于定义外部对象可以订阅的接口。
- 事件只能在声明它的类内部触发，外部代码无法直接调用或触发事件，从而提供了额外的安全性。
- 事件通常用于观察者模式等场景，被观察者在内部声明事件，供外部观察者注册处理程序。

**示例**：

```csharp
// 定义一个委托类型
delegate void MyEventHandler(string message);

// 定义一个事件
class MyClass
{
    public event MyEventHandler MyEvent;

    public void TriggerEvent(string message)
    {
        if (MyEvent != null)
        {
            MyEvent(message);  // 触发事件
        }
    }
}

// 订阅和触发事件
MyClass obj = new MyClass();
obj.MyEvent += Console.WriteLine;  // 订阅事件
obj.TriggerEvent("Hello, Event!");  // 触发事件
```

### 主要区别

1. **声明位置**：
   - `event` 只能在方法外部声明，用于定义一个可以被外部对象监听的触发点。
   - `delegate` 可以在方法内部和外部声明，具有更高的灵活性。

2. **触发限制**：
   - `event` 只能在声明它的类内部触发，外部代码无法直接调用事件。这确保了事件的触发是受控的，通常在特定条件下发生。
   - `delegate` 可以在类的内部和外部触发，没有访问限制，外部代码可以直接调用委托。

3. **使用场景**：
   - `delegate` 常用于回调机制，允许方法作为参数传递，并在需要时调用。
   - `event` 常用于发布/订阅模式，被观察者（发布者）声明事件，供外部观察者（订阅者）响应。

4. **操作符**：
   - `delegate` 可以使用 `=` 直接赋值方法，也可以使用 `+=` 添加方法到调用列表，使用 `-=` 从调用列表中移除方法。
   - `event` 在事件定义的类内部可以使用 `=`、`+=` 和 `-=` 操作符；在类外部只能使用 `+=` 和 `-=` 来添加或移除订阅者，不能直接赋值。这一限制防止了外部代码错误地覆盖或清空事件的处理程序列表。

总结来说，`delegate` 提供了一种灵活的方式来封装和调用方法，而 `event` 则是在委托的基础上，提供了一个安全的、受控的发布/订阅模型。 
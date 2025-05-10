---
layout: post
title: "Swift tricks-Phantom Types"
subtitle: "Swift中的类型安全编程技巧"
tags: [Swift]
comments: false
---

>***Swift tricks***系列收集Swift牛逼的patterns和让你代码更加Swifty的tricks，持续更新中……

## Phantom Types

在项目中，某些业务是需要按照严格的流程和规范进行的，举个🌰
```
func getSecretData() -> String {
    return "secretedata";
}
func encrypt(secretData : String) -> String {
    return "encryptdata";
}
func sendEncryptData(encryptdata : String){
    // send encrypt data
}
//首先获取秘密信息
var secretedata = getSecretData()
//对秘密信息加密
var encryptdata = encrypt(secretedata)
//发送加密信息
sendEncryptData(encryptdata)
```
上面的例子必须严格按照***"获取信息"->"加密"->"发送"***的流程来，否则就会产生安全问题！
但是程序员是人，是人就会犯错误。如果一个粗心的程序员写了下面的代码，那将会产生灾难性的后果:
```
var secretedata = getSecretData()
sendEncryptData(secretedata)
```

---

肿么办？能不能通过代码来保证流程呢？Yes, we can!
一般的做法是酱紫的：
```
struct SecretData {
    let secretedata:String
}

struct EncryptData {
    let encryptdata:String
}

func getSecretData() -> SecretData {
    return SecretData(secretedata:"secretedata");
}
func encrypt(secretData : SecretData) -> EncryptData {
    return EncryptData(encryptdata:"encryptdata");
}
func sendEncryptData(encryptdata : EncryptData){
    // send encrypt data
}
var secretedata = getSecretData()
var encryptdata = encrypt(secretedata)
sendEncryptData(encryptdata)
```
这样我们就能避免粗心程序员造成的错误 。因为当你试图执行``sendEncryptData(secretedata)``的时候，编译器会报错！

---

在上面的方法中，我们定义了两个``struct``，这两个``struct``除了名字不一样外，其他都是一模一样。设想一下，如果``struct``里面的字段稍微多一点，我们的代码将是这样的：
```
struct SecretData {
    let secretedata:String
    let encyptMehod:String
    let encyptKey:String
    let from:String
    let to:String
    ......
}

struct EncryptData {
    let encryptdata:String
    let encyptMehod:String
    let encyptKey:String
    let from:String
    let to:String
    ......
}
```
这……就有点不怎么Swifty了。
肿么办？

---

#### Phantom Types！

```
enum Encrypted {}
enum Decrypted {}

struct SecretData<T> {
    let secretedata:String
    let encyptMehod:String
    let encyptKey:String
    let from:String
    let to:String
}

func getSecretData() -> SecretData<Decrypted> {
    return SecretData(secretedata:"secretedata", encyptMehod:"nb", encyptKey:"xxx",from:"agent",to:"gcd");
}
func encrypt(secretData : SecretData<Decrypted>) -> SecretData<Encrypted> {
    return SecretData(secretedata:"secretedata", encyptMehod:"nb", encyptKey:"xxx",from:"agent",to:"gcd");
}
func sendEncryptData(encryptdata : SecretData<Encrypted>)
{
    // send encrypt data
}
var secretedata = getSecretData()
var encryptdata = encrypt(secretedata)
sendEncryptData(encryptdata)
```
``Encrypted``和``Decrypted``是两个***Phantom Type***，我们通过一个范型``struct``解决了重复定义属性的问题。
>PS:看到没有case的``enum``不要惊讶，这是Phantom Type的精髓

[参考：Functional Snippet #13: Phantom Types](https://www.objc.io/blog/2014/12/29/functional-snippet-13-phantom-types/) 
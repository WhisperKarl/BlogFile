---
title: 'Swift笔记:从Swift再看ARC以及循环引用'
date: 2016-06-15 20:17:06
tags:
---
# 自动引用计数(ARC)
Swift中ARC的原理同OC中是相同的，简单来讲就是当实例不再被使用（引用计数为0）时，实例会被释放。
为确保实例不会被提前销毁而引起程序崩溃，ARC会跟踪和计算每一个实例被多少属性、常量、和变量引用，无论是将实例赋值给属性、常量还是变量，它们都会对实例保持强引用，只要强引用还在，实例就不会被销毁。
<!-- more -->
# 类实例之间的循环引用
当两个实例互相强引用对方的时候，就会造成循环引用，任何一方都无法被正常释放。
举个例子:
``` 
class Person {
let name: String
init(name: String) { self.name = name }
var apartment: Apartment?
deinit{print"\(name) is being deinitialized"}
}
class Apartment {
let number: Int
init(number: Int) { self.number = number }
var tenant: Person?
deinit{print"apartment is being deinitialized"}
}
var john: Person?
var number73: Apartment?
john = Person(name: "John Appleseed

number73 = Apartment(number: 73)
john.apartment = number73
number73.tenant = john
john = nil
number73 = nil
```
运行可以看出，由于`john`有一个指向`Person`的强引用，而number73有一个指向`Apartment`的强引用，所以都无法释放。
强引用关系如图所示：
![53CBCE63-80C5-4E3B-A6D6-816C3FD0E0BD.png](http://upload-images.jianshu.io/upload_images/1642800-26b7f949c21e51f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
置为nil后，如图所示：

![9BE6751A-2355-4DD2-AE2A-3A3300DDC561.png](http://upload-images.jianshu.io/upload_images/1642800-a082fb66b29b1176.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 实例之间强引用的解决方法
- 用`weak`修饰，适用于引用可以为nil的一方。如上面例子中，`apartment`可以为空，因此可以用`weak`修饰`tenant`:
``` 
class Apartment {
let number: Int
init(number: Int) { self.number = number }
weak var tenant: Person?
deinit{print"apartment is being deinitialized"}
}
```
- 用`unowned`修饰，适用于一方的引用不能为 `nil `的情形。如下面的例子，没个信用卡必定会有一个主人，因此信用卡类中的`customer`用`unowned`来修饰以解决循环引用
```
class Customer {
let name: String
var card: CreditCard?
init(name: String) {
self.name = name
}
}

class CreditCard {
let number: UInt64
unowned let customer: Customer
init(number: UInt64, customer: Customer) {
self.number = number
self.customer = customer
}
}
```
- `unowned`和隐式解包的可选属性结合使用，适用于双方都不能为nil都有值的情况。如下面的例子，每个国家都会有城市，每个城市也都会属于某个国家：
``` 
class Country {
let name: String
let capitalCity: City!
init(name: String, capitalName: String) {
self.name = name
self.capitalCity = City(name: capitalName, country: self)
}
}

class City {
let name: String
unowned let country: Country
init(name: String, country: Country) {
self.name = name
self.country = country
}
}
```

# 闭包引起的循环引用
循环引用还会发生在当你将一个闭包赋值给实例的某个属性，而且闭包中又使用了这个实例（类使用OCblock中使用了self）。
产生的原因和类相似，因为闭包同类一样都是引用类型。
```
class HTMLElement{
let name: String
let text: String?

lazy var asHTML: Void -> String = {
if let text = self.text {
return "<\(self.name)>\(text)</\(self.name)>"
}else{
return"<\(self.name)/>"
}
}
init(name: String, text: String? = nil){
self.name = name
self.text = text
}

deinit{
print("\(name) is being deinitialized")
}
}

var paragraph: HTMLElement? = HTMLElement(name: "p", text: "hello,world")
print(paragraph!.asHTML())
paragraph = nil
```
在这个例子中`asHTML`属性持有了闭包的强引用，而闭包中又使用了`self`引用了`self.name`和`self.text`，因此闭包捕获了self，这以为了闭包反过来持有了实例的强引用。这样就产生了循环引用
## 解决闭包引起的循环引用
解决方法是定义闭包的捕获列表，在捕获列表中对对象进行`weak`或者`unowned`修饰。
- 如果闭包有参数列表和返回类型，把捕获列表放在它们前面：
```
lazy var someClosure: (Int, String) -> String = {
[unowned self, weak someInstance] (index: Int, stringToProcess: String) -> String in
// closure body goes here
}  
```
- 如果闭包没有指明参数列表和返回类型，即它们会通过上下文推断，那么可以把捕获列表和关键字`in`写在闭包最开始的地方：
```
lazy var someClosure: Void -> String = {
[unowned self, weak someInstance] in
// closure body goes here
}  
```
对于上面的例子，我们可以这样修改:
```
class HTMLElement{
let name: String
let text: String?

lazy var asHTML: Void -> String = {
[unowned self] in //-->无主值捕获 防止循环引用
if let text = self.text {
return "<\(self.name)>\(text)</\(self.name)>"
}else{
return"<\(self.name)/>"
}
}
init(name: String, text: String? = nil){
self.name = name
self.text = text
}

deinit{
print("\(name) is being deinitialized")
}
}
```
参考资料:[Thte Swift Programming Language](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/index.html#//apple_ref/doc/uid/TP40014097-CH3-ID0])
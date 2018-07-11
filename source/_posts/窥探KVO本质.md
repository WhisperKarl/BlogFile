---
title: 窥探KVO本质
date: 2018-07-11 10:48:36
categories:
- 编程
tags:
- iOS
---
### KVO的实现方式
`KVO` 是我们日常开发经常用到的技术，关于 `KVO` 的实现相信大家也都有一定的了解，我们来看下苹果的对于 [KVO](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOImplementation.html)的描述：
>Automatic key-value observing is implemented using a technique called *isa-swizzling*. 
The `isa` pointer, as the name suggests, points to the object's class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data. 
When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance. 
You should never rely on the `isa` pointer to determine class membership. Instead, you should use the   [class](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Protocols/NSObject/Description.html#//apple_ref/occ/intfm/NSObject/class)  method to determine the class of an object instance.

<!-- more -->
简单说就是苹果利用了`isa-swizzling`(isa 欺诈)技术，替换了实例对象的`isa`指针。当实例对象的某个属性被注册监听后，实例对象的`isa`指针将指向一个临时创建的类，而不是真正的类对象，新创建的类将重写setter方法，从而实现触发通知。由于`isa`指针没有指向真正的类对象，因此通过`object_getClass`(即isa指针)方法获取到的类对象将是错误的，因此苹果建议使用`[Object class]`方法获取实例的类对象。
### 代码验证
下面我们通过简单的代码验证一下以上过程。
首先我们创建一个简单的`Person`对象，有一个`age`属性：
```
@interface Person
@property (nonatomic, assign) int age;
@end

@implementation Person

-(void)setAge:(int)age{
_age = age;
}
@end
```
声明两个Person属性，并给person1对象添加KVO监听：
```
self.person1 = [[Person alloc] init];
self.person2 = [[Person alloc] init];   
[self.person1 addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
```
这时候我们打个断点，打印一下两个person对象的isa指针:
![打印 isa 指针](https://upload-images.jianshu.io/upload_images/1642800-e89447984e832b68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过控制台打印信息我们可以明确的看到，添加了KVO监听的`person1`的`isa`指针指向了一个叫做`NSKVONotifying_Person`的类，而没有添加监听的`person2`的`isa`指向的是`Person`类，这就说明添加KVO以后，实例对象的`isa`指针确实被替换了。

另外，我们可以猜测`NSKVONotifying_Person `类应该是重写了`setter`方法，从而在属性改变的时候发出通知，我们可以打印一下`setAge:`的`IMP`：
![打印 IMP](https://upload-images.jianshu.io/upload_images/1642800-28b9625aa6514fdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过打印信息我们可以很明确的的知道，在添加KVO监听以后，`setAge:`方法被替换为了`_NSSetIntValueAndNotify`，通过方法名我们可以猜测，这个方法的作用就是设置属性值并发出通知。

这样我们就可以初步证明了 KVO 的触发机制。即修改`isa`指针，只想一个新生成的类`NSKVONotifying_XXX`，并修改这个类的 setter 方法为`_NSSetXXXValueAndNotify `，从而发出通知。
### KVO 的触发流程 
我们知道如果想手动触发 KVO，需要先调用`willChangeValueForKey:`再调用`didChangeValueForKey:`，我们来验证下是不是这样：
```
#import "Person.h"

@implementation Person

- (void)setAge:(int)age{
_age = age;
}

- (void)willChangeValueForKey:(NSString *)key{
[super willChangeValueForKey:key];
NSLog(@"will change key : %@",key);
}

- (void)didChangeValueForKey:(NSString *)key{
NSLog(@"did change key : %@ --begin",key);
[super didChangeValueForKey:key];
NSLog(@"did change key : %@ --end",key);
}
@end
```
运行程序，我们可以得到以下打印信息：
```
KVO[80799:10056110] will change key : age
KVO[80799:10056110] did change key : age --begin
KVO[80799:10056110] age-<Person: 0x600000013f40>-{
kind = 1;
new = 1;
old = 11;
}
KVO[80799:10056110] did change key : age --end
```
和我们预想的一样，派生类的 setter 方法的确调用了这两个方法。这里还有一个需要注意的是，如果我们不调用`super`方法，将不会触发通知，这也证明了 KVO 通知的发出确实是依赖这两个方法的调用。
### NSKVONotifying_Person 的结构
KVO 的派生类也是 Class 类型，因此也遵循 `objc_class`的结构：
```
struct objc_class : objc_object {
// Class ISA;
Class superclass;
cache_t cache;             // formerly cache pointer and vtable
class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
class_rw_t *data();
……
}
```
而我们熟知的 methodList、propertyList 等信息都储存在`class_rw_t `类型的`data` 变量中：
```
struct class_rw_t {
// Be warned that Symbolication knows the layout of this structure.
uint32_t flags;
uint32_t version;
const class_ro_t *ro;
method_array_t methods;
property_array_t properties;
protocol_array_t protocols;
……
```
对于这块知识这里不做详细说明，我们继续看`NSKVONotifying_Person`都储存了哪些信息，我们先来打印一下它的 `superClass`:
```
(lldb) p class_getSuperclass(object_getClass(self.person1))
(Class) $11 = Person
(lldb) 
```
这里我们可以看到 KVO 派生类的 类对象的 `superClass`指针指向的是原来的类对象，即KVO 生成的类对象是原来类对象的子类。
另外我们可以通过`class_copyMethodList`来看一下这个派生类实现了哪些对象方法，我们直接看结果：
```
KVO[80860:10060358] NSKVONotifying_Person setAge:, class, dealloc, _isKVOA,
```
`setAge:`即是重写的`setter`方法
`class `  方法是为了返回正确的类对象，即本文开头介绍的苹果推荐的通过`class`方法获取类对象，以免发生不必要的麻烦
`dealloc ` 大概是在对象销毁时做一些释放操作
`_isKVOA ` 大概是为了判断是不是 KVO 生成的派生类

至此我们用代码验证了 KVO 的实现方式。




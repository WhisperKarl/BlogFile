---
title: 从iOS的事件响应链看TableView为什么不响应touchesBegan
date: 2016-09-23 18:17:46
categories:
- 编程
tags:
- iOS
---
>问题还原：当我们需要收起`TextField`的键盘时，通常的做法一般是在`touchBegan`方法中放弃第一响应者或者直接`endEditing`。而当我们把一个`TableView`添加到控制器的`View`上时，`touchBegan`方法会不响应，原因就在于事件被`TableView`拦截了
<!--more-->

### iOS的事件响应链
事件响应链，顾名思义就是由一系列事件响应者构成的一个响应层次。当我们点击了手机屏幕上一点时，系统会通过一系列的方法找到应该由哪一个视图来响应我们的点击事件。系统是通过hitTest由UIWindow一层层向下遍历找到可以响应点击事件的子视图，知道某一个视图没有可以响应事件的子视图时，那么这个视图就是我们所说的第一响应者。我们可以写个例子来看这个过程。

### 事件响应链的形成过程
假设我们有下图的层次结构
![层级关系.png](http://occxq9xco.bkt.clouddn.com/层级关系.png)
UIView有两个方法：
```
- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;   // recursively calls -pointInside:withEvent:. point is in the receiver's coordinate system
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;   // default returns YES if point is in bounds
```
通过注释我们可以看出，第一个方法是递归返回hitTest的View对象，第二个方法是返回点击的点是否在某一个View坐标范围内。我们可以通过给UIView写一个分类来打印hitTest过程：
```
- (BOOL)kr_pointInside:(CGPoint)point withEvent:(UIEvent *)event{

BOOL canAnswer = [self kr_pointInside:point withEvent:event];
NSLog(@"%@ can answer: %d",self.class,canAnswer);
return canAnswer;
}

- (UIView *)kr_hitTest:(CGPoint)point withEvent:(UIEvent *)event{

UIView *answerView = [self kr_hitTest:point withEvent:event];
NSLog(@"hit view :%@",self.class);
return answerView;
}
```
当我们点击ViewC时，我们可以看到打印信息：
```
UIWindow can answer: 1
UIView can answer: 1
hit view :_UILayoutGuide
hit view :_UILayoutGuide
AView can answer: 1
DView can answer: 0
hit view :DView
hit view :UILabel
BView can answer: 1
hit view :UILabel
CView can answer: 1
hit view :UILabel
hit view :CView
hit view :BView
hit view :AView
hit view :UIView
hit view :UIWindow
UIStatusBarWindow can answer: 1
UIStatusBar can answer: 0
UIStatusBarForegroundView can answer: 0
UIStatusBarServiceItemView can answer: 0
UIStatusBarDataNetworkItemView can answer: 0
UIStatusBarBatteryItemView can answer: 0
UIStatusBarTimeItemView can answer: 0
```
从打印信息我们大概可以得到响应者查找顺序为：UIWindow->UIView->AView->DView->BView->CView
所以当点击ViewC时我们可以得到一个响应者栈：

![C.png](http://occxq9xco.bkt.clouddn.com/C.png)
所以第一响应者就是ViewC，如果ViewC不能响应，那么逐级向上查找，如果UIWindow也不响应，事件抛弃。

### TableView为什么不响应touchBegan
回到刚开始的问题，当我们点击TableView时，为什么touchBegan不响应呢？通过响应链我们不难想象到，当我们点击屏幕时，第一响应者应该是UITableView，而我们调用的touchBegan其实是ViewController的View的方法，所以无法被调用。
解决方法也很简单，我们可以给tableView写一个基类，重写tableview的touchBegan方法，通过block或者代理传出，然后继承基类，即可实现touchBegan的响应。

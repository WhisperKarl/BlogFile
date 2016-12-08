---
title: iOS 自定义转场动画
date: 2016-07-19 21:46:47
categories:
- 编程
tags:
- iOS
- 转场动画
---
## 理论基础
从iOS7以后，苹果就提供了自定义转场动画的API，通过API我们可以很容易的实现各种转场动画。这里对转场动画的实现做一个小结。
<!-- more -->
1. 第一步，自定义一个管理转场动画的对象，这个对象需要遵循`<UIViewControllerAnimatedTransitioning>`协议，并实现两个必要方法
``` obj-c 
- (NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext{
//返回动画时间
}
- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext{
//动画具体实现
}```
2. 第二步，在相应的控制器根据具体情况遵循具体协议并返回动画管理对象从而实现转场效果。
-  present控制器的情况: FirstVC(将要推出下一级的VC)遵循`<UIViewControllerTransitioningDelegate>
`协议，把要退出的控制器的代理设为当前控制器：`SecondVC.transitioningDelegate = self`，并在对应方法中返回转场动画对象。
```
-(id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source{
//返回一个prenent动画过度的对象
}
-(id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed{
//返回一个管理dismiss动画过度的对象
}
```
- push控制器的情况：FirstVC遵循`<UINavigationControllerDelegate>`协议，设置`    self.navigationController.delegate = self`，并实现方法:
```obj-c
-(id<UIViewControllerAnimatedTransitioning>)navigationController:(UINavigationController *)navigationController animationControllerForOperation:(UINavigationControllerOperation)operation fromViewController:(UIViewController *)fromVC toViewController:(UIViewController *)toVC{
//返回push动画过度对象
}
```

---
虽然只有两步，但是有好多种协议，区分起来还是挺迷糊的，下面通过一个demo来分析一下。



![转场效果.gif](http://occxq9xco.bkt.clouddn.com/animation.gif)
这个demo是[@叶孤城](http://weibo.com/u/1438670852?refer_flag=1001030101_&is_all=1)在直播时候写的，模仿的是qq音乐的转场动画，他称之为截图大法，掌握这个方法，就可以实现绝大部分的转场效果了。我们来看一下是怎么实现的。
1. 首先创建两个控制器FirstVC和SecondVC，结构都很简单，首页底部一个深色条，在深色条上添加一个小的周杰伦的图片。二级页中间一张大的周杰伦的图片。
2. 创建一个管理动画的类`TransitionAnimationTest`，集成于`NSObject`，遵循`UIViewControllerAnimatedTransitioning`协议:
```
@interface TransitionAnimationTest : NSObject<UIViewControllerAnimatedTransitioning>
```
3. 返回动画时间:
```
-(NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext{
//动画持续时间
return 0.4f;
}
```
3. 最重要的动画实现部分,我在代码里加了备注
```obj-c
-(void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext{
//先拿到fromVC和toVC, fromVC就是要推次级页的控制器,toVC就是被推出的控制器
UINavigationController *nav = (UINavigationController *)[transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
FirstViewController *fromViewController = (FirstViewController *)nav.topViewController;
SecondViewController *toViewController = (SecondViewController *)[transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
//containerView 是承载过度动画过程中各个view的一层View
UIView *containerView = [transitionContext containerView];
NSTimeInterval duration = [self transitionDuration:transitionContext];
//把FirstVC的小周杰伦图片进行截图,然后隐藏原图片,这样我们只需要对截图进行操作就可以
UIView *smallImageSnapshot = [fromViewController.smallZhouJieLun snapshotViewAfterScreenUpdates:YES];
//截图的初始坐标为小图的位置,由于我们要把截图加到containerView上,所以我们要通过这个方法进行坐标转化,把相对于深色条(vackView)的坐标转化为相对于containerView的坐标
smallImageSnapshot.frame = [containerView convertRect:fromViewController.smallZhouJieLun.frame fromView:fromViewController.backView];
fromViewController.smallZhouJieLun.hidden = YES;

toViewController.view.frame = [transitionContext finalFrameForViewController:toViewController];
//先隐藏toVC,动画完成后再显示出来
toViewController.view.alpha = 0;
toViewController.bigImageView.hidden = YES;
//根据图层关系把toVC.view和截图先后加到containerView上
[containerView addSubview:toViewController.view];
[containerView addSubview:smallImageSnapshot];

//开始动画
[UIView animateWithDuration:duration animations:^{
//显示toVC.view
toViewController.view.alpha = 1.0f;
//获取新的frame,即大图的相对于containerView的坐标,并赋给截图
CGRect frame = [containerView convertRect:toViewController.bigImageView.frame fromView:toViewController.view];
smallImageSnapshot.frame = frame;

} completion:^(BOOL finished) {
//显示大图
toViewController.bigImageView.hidden = NO;
//小图要显示出来,否则pop后会消失
fromViewController.smallZhouJieLun.hidden = NO;
//隐藏截图
[smallImageSnapshot removeFromSuperview];
//完成动画
[transitionContext completeTransition:!transitionContext.transitionWasCancelled];
}];
}
```
到这里就完成了转场动画管理类的创建

5. 在控制器里实现动画，我们要使用present，所以我们要遵循`<UIViewControllerTransitioningDelegate>`协议并实现方法:
```obj-c
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source{
return  [[TransitionAnimationTest alloc] init];
}
```
这样就完成了present的动画，dismiss的动画同理也可实现，只不过截图截的是大图，frame变换是有大图到小图，道理是一样的。

### 小结：其实我们只要掌握了转场API的使用方法，具体的动画就靠我们丰富的想象力去实现了。
本文demo地址：https://github.com/WhisperKarl/TransitionAnimation.git
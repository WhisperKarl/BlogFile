---
title: iOS AutoLayout约束的优先级
date: 2016-11-22 18:56:22
categories:
- 编程
tags:
- iOS
---
##  约束的优先级
AutoLayout中添加的约束也有优先级,优先级的数值是1~1000。分为两种情况：
<!--more-->
- 一种情况是我们经常添加的各种约束,默认的优先级是1000，也就是最高级别，条件允许的话系统会满足我们所有的约束需求。
- 另外一种情况就是固有约束(intinsic content size)，严格来说这个这个更像是指UILabel和UIButton控件的一种属性,但是在AutoLayout中这个属性的取值和约束优先级的属性相结合才能完成图形的绘制。
我们知道UIButton和UILabel两种控件可以根据内容长短来控制控件宽度，当展示内容的宽度满足不了约束的要求时(过短或者过宽)，控件就会被拉伸或者压缩，当我们不想控件被拉伸或者压缩时，就需要设置控件的固有约束（intinsic content size）来实现我们的需求。固有约束分为两种：
1 ) Content Hugging Priority
官方文档的解释是
` Returns the priority with which a view resists being made larger than its intrinsic size.`
即表示的是控件的抗拉伸优先级，优先级越高，越不易被拉伸，默认为251
2) Content Compression Resistance Priority
`Returns the priority with which a view resists being made smaller than its intrinsic size.`
这个优先级的字面意思很明确了，是防压缩优先级，优先级越高，越不易被压缩，默认为750

多说无益，举个栗子来感受一下
## 举个栗子
- 我们放置如图所示约束的一个label控件，约束的默认优先级为1000，固有约束 Content Hugging Priority为251,  Content Compression Resistance Priority为750
![1.png](http://occxq9xco.bkt.clouddn.com/autolayout1.png)
运行结果为
![2.png](http://occxq9xco.bkt.clouddn.com/autolayout2.png)
为什么会这样呢，不难想到是由于  左边距+右边距+Label宽度>屏幕宽度，无法满足我们所有的约束需求，而根据优先级的数值，左右边距的有限级（1000）> 抗压缩优先级（750），所以系统优先满足左右边距而选择压缩Label的方案

- Content Compression Resistance Priority的影响
我们把左边距的优先级设置为700，这时候它小于Content Compression Resistance Priority为的优先级（750）
![3.png](http://occxq9xco.bkt.clouddn.com/autolayout3.png)
运行结果变为
![4.png](http://occxq9xco.bkt.clouddn.com/autolayout4.png)
我们看到Label整体向右偏移了一些来保证Label内容能够显示完全，这就是由于Content Compression Resistance Priority优先级大于右边距约束的优先级产生的效果

- Content Hugging Priority的影响
再来看一下抗拉伸优先级的影响，我们把左右边距约束的优先级恢复到1000，左右边距改为50，看一下效果
![5.png](http://occxq9xco.bkt.clouddn.com/autolayout5.png)
和上个🌰对比一下我们不难想到，由于 左右边距+固有宽度<屏幕宽度而Content Hugging Priority(251) < 边距约束的优先级(1000) ,所以系统拉谁了Label本身
如果我们降低右边距的优先级为240， 小于抗拉伸优先级（251），效果如下
![6.png](http://occxq9xco.bkt.clouddn.com/autolayout6.png)
这时候系统优先满足了Label的宽度，而没有满足右边距的需求。
这样，约束的优先级和固有宽度优先级如何相互影响就很明确了，来看一个工作中的例子。

## 再举个🌰
场景还原，工作中我们常常会碰到这种需求，并排放置两个Label，左边的Label宽度根据内容适应，右边的Label距离左边Label有个固定距离，距离屏幕右边有个固定距离，在不设置优先级的情况下，我们会经常遇到奇怪的现象，要么左边的Labe被拉伸，要么被压缩，比如下图
![7.png](http://occxq9xco.bkt.clouddn.com/autolayout7.png)
我们设置左边Label的抗压缩优先级和抗拉伸优先级都大于右边Label，效果如图
![8.png](http://occxq9xco.bkt.clouddn.com/autolayout8.png)
![9.png](http://occxq9xco.bkt.clouddn.com/autolayout9.png)
由于左边Label的抗压缩和抗拉伸优先级都高于右边Label，而且其他约束的优先级（1000）也都高于右边label的固有宽度优先级，所以系统选择拉伸或者压缩了右边的Label，实现了我们的需求。

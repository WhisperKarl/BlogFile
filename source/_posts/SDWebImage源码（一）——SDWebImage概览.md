---
title: SDWebImage源码（一）——SDWebImage概览
date: 2016-12-08 22:15:47
categories:
- 编程
tags:
- iOS
---
SDWebImage是我们经常使用的一个异步图片加载库，使用时只需一行代码就能实现网络图片的异步加载、缓存(内存+磁盘)，非常方便。最近工作稍清闲，就仔细研读了一下它的源码。本篇主要是简单梳理SDWebImage的工作流程，如下图所示：
<!-- more -->

![流程.png](http://upload-images.jianshu.io/upload_images/1642800-bfa9da4a8e180374.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们看到SDWebImage各个部件分工明确，`SDImageCache`负责图片缓存相关的工作，`SDWebImageDownloader`负责图片下载相关的工作，`SDWebImageManager`则是将前两者结合起来完成整个工作流程。

我们就按模块一点点剖析它的各个模块：
- [SDWebImage源码（二）——SDImageCache缓存器](http://www.jianshu.com/p/76424e747c0c)
- [SDWebImage源码（三）——SDWebImageDownloader下载器](http://www.jianshu.com/p/637e0aa8982e)
- [SDWebImage源码（四）——SDWebImageManager](http://www.jianshu.com/p/d00436d71784)

---
title: SDWebImage源码（三）——SDWebImageDownloader下载器
date: 2016-12-10 18:59:45
categories:
- 编程
tags:
- iOS
---
#### SDWebImageDownloader
`SDWebImageDownloader`完成了对网络图片的异步下载工作，准确说这个类是一个下载任务管理器，而真正的网络请求是在继承于`NSOperation`的`SDWebImageDownloaderOperation`类实现的。我们来看一下这两个类的具体实现。
<!-- more -->
`SDWebImageDownloader `的头文件内容比较少，主要是定义了一些基本参数如下载优先级策略、最大并发数、超时时间等，比较简单，这里就不再赘述。`SDWebImageDownloader `中核心的方法是`- downloadImageWithURL:(NSURoptions: progress: completed:
`，在看这个方法之前我们先看另外一个方法:
```

- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock andCompletedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
    // The URL will be used as the key to the callbacks dictionary so it cannot be nil. If it is nil immediately call the completed block with no image or data.
    //如果URL为空，直接执行完成回调block并传入nil参数，结束本次请求
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return;
    }
  //NSMutableDicitonary不是线程安全的，利用GCD的barrier 保证线程安全，确保字典不会同时存取
    dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }

        // Handle single download of simultaneous download request for the same URL
        //为URL创建一个唯一对应的callbacks，并赋值给self.URLCallBacks
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
        }
    });
}
```
上面这段代码猛一看一头雾水，画张结构图一看就非常明晰了

![结构图.png](http://upload-images.jianshu.io/upload_images/1642800-9cf7e9789ea361a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
即`URLCallBacks`字典存储的每个请求的`callbacksForURL`，在这里我有一点不明白的是为什么中间会有一层可变数组来承载`callbacksForURL`字典，因为从代码上看，每个URL请求只会被请求一次的，希望如果有明白的小伙伴留言告知~
走完上面这个函数，我们就能确保每个请求都能和它的`progressBlock `和`completedBlock `回调一一对应，接着回到download方法继续看：
```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageDownloaderCompletedBlock)completedBlock {
    __block SDWebImageDownloaderOperation *operation;
    __weak SDWebImageDownloader *wself = self;

    [self addProgressCallback:progressBlock andCompletedBlock:completedBlock forURL:url createCallback:^{
        //设置请求超时时间
        NSTimeInterval timeoutInterval = wself.downloadTimeout;
        if (timeoutInterval == 0.0) {
            timeoutInterval = 15.0;
        }

        // In order to prevent from potential duplicate caching (NSURLCache + SDImageCache) we disable the cache for image requests if told otherwise
      //初始化URLRequest
        NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
        request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
        request.HTTPShouldUsePipelining = YES;
        if (wself.headersFilter) {
            request.allHTTPHeaderFields = wself.headersFilter(url, [wself.HTTPHeaders copy]);
        }
        else {
            request.allHTTPHeaderFields = wself.HTTPHeaders;
        }

       //初始化SDWebImageDownloaderOperation并处理回调，网络请求以及数据处理在SDWebImageDownloaderOperation类中进行
        operation = [[SDWebImageDownloaderOperation alloc] initWithRequest:request
                                                                   options:options
                                                                  progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                                  //拿到URL对应的progressBlock并赋值
                                                                      SDWebImageDownloader *sself = wself;
                                                                      if (!sself) return;
                                                                      NSArray *callbacksForURL = [sself callbacksForURL:url];
                                                                      for (NSDictionary *callbacks in callbacksForURL) {
                                                                          SDWebImageDownloaderProgressBlock callback = callbacks[kProgressCallbackKey];
                                                                          if (callback) callback(receivedSize, expectedSize);
                                                                      }
                                                                  }
                                                                 completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                                                                     //执行下载完成回调
                                                                     SDWebImageDownloader *sself = wself;
                                                                     if (!sself) return;
                                                                     NSArray *callbacksForURL = [sself callbacksForURL:url];
                                                                     if (finished) {
                                                                         //移除URL对应的callback
                                                                         [sself removeCallbacksForURL:url];
                                                                     }
                                                                     for (NSDictionary *callbacks in callbacksForURL) {
                                                                         SDWebImageDownloaderCompletedBlock callback = callbacks[kCompletedCallbackKey];
                                                                         if (callback) callback(image, data, error, finished);
                                                                     }
                                                                 }
                                                                 cancelled:^{
                                                                     SDWebImageDownloader *sself = wself;
                                                                     if (!sself) return;
                                                                     [sself removeCallbacksForURL:url];
                                                                 }];
        //服务器身份验证，一般用不到
       if (wself.urlCredential) {
            operation.credential = wself.urlCredential;
        } else if (wself.username && wself.password) {
            operation.credential = [NSURLCredential credentialWithUser:wself.username password:wself.password persistence:NSURLCredentialPersistenceForSession];
        }
        
        //设置线程优先级
        if (options & SDWebImageDownloaderHighPriority) {
            operation.queuePriority = NSOperationQueuePriorityHigh;
        } else if (options & SDWebImageDownloaderLowPriority) {
            operation.queuePriority = NSOperationQueuePriorityLow;
        }
      
        //添加任务到串行队列，开启下载任务
        [wself.downloadQueue addOperation:operation];
        //根据配置的执行顺序（先进先出还是先进后出）添加线程依赖
        if (wself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
            // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
            [wself.lastAddedOperation addDependency:operation];
            wself.lastAddedOperation = operation;
        }
    }];
    return operation;
}
```
downloadImageWithURL方法的内容不多，通过一步步的拆分我们可以看的比较明了，下面继续看看`SDWebImageDownloaderOperation `这个类中做了哪些事情
#### SDWebImageDownloaderOperation
`SDWebImageDownloaderOperation`继承于`NSOperation`重写了`start`方法，在`start`方法里创建了`NSURLConnection`链接。先看看对外开放的初始化方法做了哪些工作：
```
- (id)initWithRequest:(NSURLRequest *)request
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock {
    if ((self = [super init])) {
         //在SDWebImageDownloader中创建的NSMutableURLRequest对象
        _request = request;
        //是否解图片
        _shouldDecompressImages = YES;
        //验证挑战
        _shouldUseCredentialStorage = YES;
         //下载策略
        _options = options;
        _progressBlock = [progressBlock copy];
        _completedBlock = [completedBlock copy];
        _cancelBlock = [cancelBlock copy];
        _executing = NO;
        _finished = NO;
        _expectedSize = 0;
        //用来标记response是不是从cache中获取的
        responseFromCached = YES; // Initially wrong until `connection:willCacheResponse:` is called or not called
    }
    return self;
}
```
初始化主要还是对一些参数做了配置，下面看start方法：
```
- (void)start {
    @synchronized (self) {
        if (self.isCancelled) {
        //如果是取消状态，重置参数并设置为完成状态
            self.finished = YES;
            [self reset];
            return;
        }
//开启后台下载任务
#if TARGET_OS_IPHONE && __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_4_0
        //先验证UIApplication实体是否存在
        Class UIApplicationClass = NSClassFromString(@"UIApplication");
        BOOL hasApplication = UIApplicationClass && [UIApplicationClass respondsToSelector:@selector(sharedApplication)];
        //如果存在且允许后台下载
        if (hasApplication && [self shouldContinueWhenAppEntersBackground]) {
            __weak __typeof__ (self) wself = self;
            UIApplication * app = [UIApplicationClass performSelector:@selector(sharedApplication)];
            self.backgroundTaskId = [app beginBackgroundTaskWithExpirationHandler:^{
                //这个回调是后台任务将被系统杀死时执行，这里取消了下载任务
                __strong __typeof (wself) sself = wself;
  
                if (sself) {
                    [sself cancel];

                    [app endBackgroundTask:sself.backgroundTaskId];
                    sself.backgroundTaskId = UIBackgroundTaskInvalid;
                }
            }];
        }
#endif
        
        //构建NSURLConnection向服务器发送请求
        self.executing = YES;
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
        self.thread = [NSThread currentThread];
    }

    [self.connection start];

    if (self.connection) {
        if (self.progressBlock) {
            self.progressBlock(0, NSURLResponseUnknownLength);
        }
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStartNotification object:self];
        });

        //需要手动开启runloop，connection才能正常接收delegate回调
        if (floor(NSFoundationVersionNumber) <= NSFoundationVersionNumber_iOS_5_1) {
            // Make sure to run the runloop in our background thread so it can process downloaded data
            // Note: we use a timeout to work around an issue with NSURLConnection cancel under iOS 5
            //       not waking up the runloop, leading to dead threads (see https://github.com/rs/SDWebImage/issues/466)
            CFRunLoopRunInMode(kCFRunLoopDefaultMode, 10, false);
        }
        else {
            CFRunLoopRun();
        }
      
        //runloop关闭之后才会继续执行下面的代码
        if (!self.isFinished) {
            //请求失败
            [self.connection cancel];
            [self connection:self.connection didFailWithError:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorTimedOut userInfo:@{NSURLErrorFailingURLErrorKey : self.request.URL}]];
        }
    }
    else {
        if (self.completedBlock) {
        //NSURLConnection创建失败，执行Error回调
            self.completedBlock(nil, nil, [NSError errorWithDomain:NSURLErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Connection can't be initialized"}], YES);
        }
    }
//到这里，不管下载成功与否，都注销之前注册的后台任务
#if TARGET_OS_IPHONE && __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_4_0
    Class UIApplicationClass = NSClassFromString(@"UIApplication");
    if(!UIApplicationClass || ![UIApplicationClass respondsToSelector:@selector(sharedApplication)]) {
        return;
    }
    if (self.backgroundTaskId != UIBackgroundTaskInvalid) {
        UIApplication * app = [UIApplication performSelector:@selector(sharedApplication)];
        [app endBackgroundTask:self.backgroundTaskId];
        self.backgroundTaskId = UIBackgroundTaskInvalid;
    }
#endif
}
```
start方法是实现并发`NSOperation`的核心方法，在这个方法里创建了`NSURLConnection`请求并发起请求。在这里有个疑问就是，为什么开启runloop之后的代码一定会等到runloop关闭以后才执行，看来需要仔细研习一下runloop的只是了，或者有明白的小伙伴留言告知一下~
重点看一下`NSURLConnection`的两个代理方法：
```

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    //把新接收到的数据拼接到imageData后面
    [self.imageData appendData:data];
    //如果options值为SDWebImageDownloaderProgressiveDownload，即边下载边显示，而不是下载完成一次洗显示，则需要随着接收到data数据立即进行图片转化，具体转化方法这里不做探究
    if ((self.options & SDWebImageDownloaderProgressiveDownload) && self.expectedSize > 0 && self.completedBlock) {
         // The following code is from http://www.cocoaintheshell.com/2011/05/progressive-images-download-imageio/
        // Thanks to the author @Nyx0uf

        // Get the total bytes downloaded
        
        const NSInteger totalSize = self.imageData.length;

        // Update the data source, we must pass ALL the data, not just the new bytes
        CGImageSourceRef imageSource = CGImageSourceCreateWithData((__bridge CFDataRef)self.imageData, NULL);

        if (width + height == 0) {
            CFDictionaryRef properties = CGImageSourceCopyPropertiesAtIndex(imageSource, 0, NULL);
            if (properties) {
                NSInteger orientationValue = -1;
                CFTypeRef val = CFDictionaryGetValue(properties, kCGImagePropertyPixelHeight);
                if (val) CFNumberGetValue(val, kCFNumberLongType, &height);
                val = CFDictionaryGetValue(properties, kCGImagePropertyPixelWidth);
                if (val) CFNumberGetValue(val, kCFNumberLongType, &width);
                val = CFDictionaryGetValue(properties, kCGImagePropertyOrientation);
                if (val) CFNumberGetValue(val, kCFNumberNSIntegerType, &orientationValue);
                CFRelease(properties);

                // When we draw to Core Graphics, we lose orientation information,
                // which means the image below born of initWithCGIImage will be
                // oriented incorrectly sometimes. (Unlike the image born of initWithData
                // in connectionDidFinishLoading.) So save it here and pass it on later.
                orientation = [[self class] orientationFromPropertyValue:(orientationValue == -1 ? 1 : orientationValue)];
            }

        }

        if (width + height > 0 && totalSize < self.expectedSize) {
            // Create the image
            CGImageRef partialImageRef = CGImageSourceCreateImageAtIndex(imageSource, 0, NULL);

#ifdef TARGET_OS_IPHONE
            // Workaround for iOS anamorphic image
            if (partialImageRef) {
                const size_t partialHeight = CGImageGetHeight(partialImageRef);
                CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
                CGContextRef bmContext = CGBitmapContextCreate(NULL, width, height, 8, width * 4, colorSpace, kCGBitmapByteOrderDefault | kCGImageAlphaPremultipliedFirst);
                CGColorSpaceRelease(colorSpace);
                if (bmContext) {
                    CGContextDrawImage(bmContext, (CGRect){.origin.x = 0.0f, .origin.y = 0.0f, .size.width = width, .size.height = partialHeight}, partialImageRef);
                    CGImageRelease(partialImageRef);
                    partialImageRef = CGBitmapContextCreateImage(bmContext);
                    CGContextRelease(bmContext);
                }
                else {
                    CGImageRelease(partialImageRef);
                    partialImageRef = nil;
                }
            }
#endif

            if (partialImageRef) {
                UIImage *image = [UIImage imageWithCGImage:partialImageRef scale:1 orientation:orientation];
                NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:self.request.URL];
                UIImage *scaledImage = [self scaledImageForKey:key image:image];
                if (self.shouldDecompressImages) {
                    image = [UIImage decodedImageWithImage:scaledImage];
                }
                else {
                    image = scaledImage;
                }
                CGImageRelease(partialImageRef);
                dispatch_main_sync_safe(^{
                    if (self.completedBlock) {
                        self.completedBlock(image, nil, nil, NO);
                    }
                });
            }
        }

        CFRelease(imageSource);
    }
    //下载进度回调
    if (self.progressBlock) {
        self.progressBlock(self.imageData.length, self.expectedSize);
    }
}
```
下载完成的代理方法：
```
- (void)connectionDidFinishLoading:(NSURLConnection *)aConnection {
    SDWebImageDownloaderCompletedBlock completionBlock = self.completedBlock;
    @synchronized(self) {
         //停止runloop 释放资源
        CFRunLoopStop(CFRunLoopGetCurrent());
        self.thread = nil;
        self.connection = nil;
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStopNotification object:self];
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadFinishNotification object:self];
        });
    }
    
    if (![[NSURLCache sharedURLCache] cachedResponseForRequest:_request]) {
        responseFromCached = NO;
    }
    
    if (completionBlock) {
        if (self.options & SDWebImageDownloaderIgnoreCachedResponse && responseFromCached) {
            //如果options选择了SDWebImageDownloaderIgnoreCachedResponse且response是从URLCache中取到的，则成功回调返回空值（为何？）
            completionBlock(nil, nil, nil, YES);
        } else if (self.imageData) {
            UIImage *image = [UIImage sd_imageWithData:self.imageData];
            NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:self.request.URL];
             //根据手机屏幕分辨率对图片进行缩放
            image = [self scaledImageForKey:key image:image];
            
            // Do not force decoding animated GIFs
            if (!image.images) {
                 //如果不是gif图，解压图片
                if (self.shouldDecompressImages) {
                    image = [UIImage decodedImageWithImage:image];
                }
            }
            if (CGSizeEqualToSize(image.size, CGSizeZero)) {
                //如果图片是0像素，返回nil并报错
                completionBlock(nil, nil, [NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Downloaded image has 0 pixels"}], YES);
            }
            else {
                completionBlock(image, self.imageData, nil, YES);
            }
        } else {
            completionBlock(nil, nil, [NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Image data is nil"}], YES);
        }
    }
    //释放资源
    self.completionBlock = nil;
    [self done];
}
```
以上就是`SDWebImageDownloader`的核心内容。

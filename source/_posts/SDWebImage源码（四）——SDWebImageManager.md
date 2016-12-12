---
title: SDWebImage源码（四）——SDWebImageManager
date: 2016-12-12 17:59:08
categories:
- 变成
tags:
- iOS
---
前面我们了解了`SDWebImageDownloader`和`SDImageCache`两个类，这两个都是为`SDWebImageManager`来服务的，`SDWebImageManager `起到了两个类的连接作用。我们先来看一下头文件中的一些方法：
<!-- more -->
```
@interface SDWebImageManager : NSObject

@property (weak, nonatomic) id <SDWebImageManagerDelegate> delegate;

@property (strong, nonatomic, readonly) SDImageCache *imageCache;
@property (strong, nonatomic, readonly) SDWebImageDownloader *imageDownloader;

/**
 * The cache filter is a block used each time SDWebImageManager need to convert an URL into a cache key. This can
 * be used to remove dynamic part of an image URL.
 *
 * The following example sets a filter in the application delegate that will remove any query-string from the
 * URL before to use it as a cache key:
 *
 * @code

[[SDWebImageManager sharedManager] setCacheKeyFilter:^(NSURL *url) {
    url = [[NSURL alloc] initWithScheme:url.scheme host:url.host path:url.path];
    return [url absoluteString];
}];

 * @endcode
 */
//从注释可以看出这个block是对URL格式做一些操作，我们工作中并没有去实现，直接用了图片地址
@property (nonatomic, copy) SDWebImageCacheKeyFilterBlock cacheKeyFilter;

/**
 * Returns global SDWebImageManager instance.
 *
 * @return SDWebImageManager shared instance
 */
+ (SDWebImageManager *)sharedManager;

/**
 * Downloads the image at the given URL if not present in cache or return the cached version otherwise.
 *
 * @param url            The URL to the image
 * @param options        A mask to specify options to use for this request
 * @param progressBlock  A block called while image is downloading
 * @param completedBlock A block called when operation has been completed.
 *
 *   This parameter is required.
 * 
 *   This block has no return value and takes the requested UIImage as first parameter.
 *   In case of error the image parameter is nil and the second parameter may contain an NSError.
 *
 *   The third parameter is an `SDImageCacheType` enum indicating if the image was retrieved from the local cache
 *   or from the memory cache or from the network.
 *
 *   The last parameter is set to NO when the SDWebImageProgressiveDownload option is used and the image is 
 *   downloading. This block is thus called repeatedly with a partial image. When image is fully downloaded, the
 *   block is called a last time with the full image and the last parameter set to YES.
 *
 * @return Returns an NSObject conforming to SDWebImageOperation. Should be an instance of SDWebImageDownloaderOperation
 */
//核心方法，调用了SDImageCache和SDWebImageDownloader中的方法完成了对图片缓存的查询和图片下载，返回了SDWebImageDownloaderOperation类
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;

/**
 * Saves image to cache for given URL
 *
 * @param image The image to cache
 * @param url   The URL to the image
 *
 */
// 缓存图片
- (void)saveImageToCache:(UIImage *)image forURL:(NSURL *)url;

/**
 * Cancel all current operations
 */
//取消所有操作
- (void)cancelAll;

/**
 * Check one or more operations running
 */
//检测是否有运行中的操作
- (BOOL)isRunning;

/**
 *  Check if image has already been cached
 *
 *  @param url image url
 *
 *  @return if the image was already cached
 */
// 检查图片是否缓存过
- (BOOL)cachedImageExistsForURL:(NSURL *)url;

/**
 *  Check if image has already been cached on disk only
 *
 *  @param url image url
 *
 *  @return if the image was already cached (disk only)
 */
//在磁盘内存中查询图片缓存是否存在
- (BOOL)diskImageExistsForURL:(NSURL *)url;

/**
 *  Async check if image has already been cached
 *
 *  @param url              image url
 *  @param completionBlock  the block to be executed when the check is finished
 *  
 *  @note the completion block is always executed on the main queue
 */
//异步查询缓存并回调
- (void)cachedImageExistsForURL:(NSURL *)url
                     completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;

/**
 *  Async check if image has already been cached on disk only
 *
 *  @param url              image url
 *  @param completionBlock  the block to be executed when the check is finished
 *
 *  @note the completion block is always executed on the main queue
 */
// 异步查询磁盘缓存并回调
- (void)diskImageExistsForURL:(NSURL *)url
                   completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;


/**
 *Return the cache key for a given URL
 */
// 调用SDWebImageCacheKeyFilterBlock对URL格式做处理
- (NSString *)cacheKeyForURL:(NSURL *)url;

@end
```
从头文件我们就可以看出，`SDWebImageManager`调用的都是我们之前已经了解过的方法，这里就不再赘述，着重看一下download核心方法：
```
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock {
    // Invoking this method without a completedBlock is pointless
    // completedBlock是必须的
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

    // Very common mistake is to send the URL using NSString object instead of NSURL. For some strange reason, XCode won't
    // throw any warning for this type mismatch. Here we failsafe this error by allowing URLs to be passed as NSString.
    //url类型判断
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }
    // Prevents app crashing on argument type error like sending NSNull instead of NSURL
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }
      //看到这里就懵逼了，怎么又出来一个SDWebImageCombinedOperation，
      /**从声明我们可以看到SDWebImageCombinedOperation也是一个遵循<SDWebImageOperation>协议的NSObject类，
        *实现了cancel方法和一个cacheOperation属性，主要是为了操作SDImageCache的cancel方法，和NSOperation也没什么关系，这里不用太过纠结*/
        
    __block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    __weak SDWebImageCombinedOperation *weakOperation = operation;

    BOOL isFailedUrl = NO;
    @synchronized (self.failedURLs) {
        //self.failedURLs 储存着请求失败过的url
        isFailedUrl = [self.failedURLs containsObject:url];
    }

    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
      //如果url长度为0，或者之前请求失败过而且不允许重新请求，完成操作
        dispatch_main_sync_safe(^{
            NSError *error = [NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil];
            completedBlock(nil, error, SDImageCacheTypeNone, YES, url);
        });
        return operation;
    }

    @synchronized (self.runningOperations) {
      // self.runningOperations  储存着所有运行当中的operation
        [self.runningOperations addObject:operation];
    }
    NSString *key = [self cacheKeyForURL:url];
  
   //调用了SDImageCache种的查询方法，并返回一个Operation，并赋给上面讲的那个combinedOperation，方便做cancle操作
    operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {
        if (operation.isCancelled) {
            @synchronized (self.runningOperations) {
              //如果取消操作，移除
                [self.runningOperations removeObject:operation];
            }
            return;
        }
       //缓存中没有取到image才会执行下面的代码，因为源码中并没有实现delegate，所以 下面的判断相当于是 1&&1 ,如果没有取到图片，一定会执行
        if ((!image || options & SDWebImageRefreshCached) && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url])) {
  
            if (image && options & SDWebImageRefreshCached) {
                dispatch_main_sync_safe(^{
                    // If image was found in the cache but SDWebImageRefreshCached is provided, notify about the cached image
                    // AND try to re-download it in order to let a chance to NSURLCache to refresh it from server.
                    //如果在缓存中找到了图片，执行complete回调，但是如果允许SDWebImageRefreshCached，则继续进行下面的下载操作
                    completedBlock(image, nil, cacheType, YES, url);
                });
            }

            // download if no image or requested to refresh anyway, and download allowed by delegate
            SDWebImageDownloaderOptions downloaderOptions = 0;
            if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
            if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
            if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
            if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
            if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
            if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
            if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
            if (image && options & SDWebImageRefreshCached) {
                // force progressive off if image already cached but forced refreshing
                downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                // ignore image read from NSURLCache if image if cached but force refreshing
                downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
            }
          //创建一个下载线程，这里我们应该了解到自始至终的block回调都是同一个对象
            id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished) {
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (!strongOperation || strongOperation.isCancelled) {
                    // Do nothing if the operation was cancelled
                    // See #699 for more details
                    // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
                    //这里不能调用completedBlock，会覆盖数据（？）
                }
                else if (error) {
                    //进行一些错误处理
                    dispatch_main_sync_safe(^{
                        if (strongOperation && !strongOperation.isCancelled) {
                            completedBlock(nil, error, SDImageCacheTypeNone, finished, url);
                        }
                    });

                    if (   error.code != NSURLErrorNotConnectedToInternet
                        && error.code != NSURLErrorCancelled
                        && error.code != NSURLErrorTimedOut
                        && error.code != NSURLErrorInternationalRoamingOff
                        && error.code != NSURLErrorDataNotAllowed
                        && error.code != NSURLErrorCannotFindHost
                        && error.code != NSURLErrorCannotConnectToHost) {
                        @synchronized (self.failedURLs) {
                            //把请求出错的url添加到failedURLs 数组中
                            [self.failedURLs addObject:url];
                        }
                    }
                }
                else {
                    //没有错误的话，先移除错误url
                    if ((options & SDWebImageRetryFailed)) {
                        @synchronized (self.failedURLs) {
                            [self.failedURLs removeObject:url];
                        }
                    }
                    
                    BOOL cacheOnDisk = !(options & SDWebImageCacheMemoryOnly);

                    if (options & SDWebImageRefreshCached && image && !downloadedImage) {
                        // Image refresh hit the NSURLCache cache, do not call the completion block
                      //如果是从内存取出的图片而不是下载的，不做操作
                    }
                    else if (downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) {  
                //源码中并没有实现delegate方法，所以这一段代码并不会执行
                 dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                            UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];

                            if (transformedImage && finished) {
                                BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                                [self.imageCache storeImage:transformedImage recalculateFromImage:imageWasTransformed imageData:(imageWasTransformed ? nil : data) forKey:key toDisk:cacheOnDisk];
                            }

                            dispatch_main_sync_safe(^{
                                if (strongOperation && !strongOperation.isCancelled) {
                                    completedBlock(transformedImage, nil, SDImageCacheTypeNone, finished, url);
                                }
                            });
                        });
                    }
                    else {
                        if (downloadedImage && finished) {
                           //缓存图片
                            [self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];
                        }

                        dispatch_main_sync_safe(^{
                          //回到主线程执行completedBlock
                            if (strongOperation && !strongOperation.isCancelled) {
                                completedBlock(downloadedImage, nil, SDImageCacheTypeNone, finished, url);
                            }
                        });
                    }
                }

                if (finished) {
                    @synchronized (self.runningOperations) {
                       //完成操作，从runningOperations中移除
                        if (strongOperation) {
                            [self.runningOperations removeObject:strongOperation];
                        }
                    }
                }
            }];
            operation.cancelBlock = ^{
                //取消操作
                [subOperation cancel];
                
                @synchronized (self.runningOperations) {
                    __strong __typeof(weakOperation) strongOperation = weakOperation;
                    if (strongOperation) {
                        [self.runningOperations removeObject:strongOperation];
                    }
                }
            };
        }
        //下面是如果从缓存中取到图片的方法
        else if (image) {
            dispatch_main_sync_safe(^{
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (strongOperation && !strongOperation.isCancelled) {
                    completedBlock(image, nil, cacheType, YES, url);
                }
            });
            @synchronized (self.runningOperations) {
                [self.runningOperations removeObject:operation];
            }
        }
        else {
            // Image not in cache and download disallowed by delegate
            dispatch_main_sync_safe(^{
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (strongOperation && !weakOperation.isCancelled) {
                    completedBlock(nil, nil, SDImageCacheTypeNone, YES, url);
                }
            });
            @synchronized (self.runningOperations) {
                [self.runningOperations removeObject:operation];
            }
        }
    }];

    return operation;
}
```
到此为止，`SDWebImage`的几个重要组件我们都做了了解，我们回到最开始的地方，即`UIImageView+WebCache`分类中看一下:
```
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock {
    [self sd_cancelCurrentImageLoad];
    objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);

    if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
            self.image = placeholder;
        });
    }
    
    if (url) {

        // check if activityView is enabled or not
        if ([self showActivityIndicatorView]) {
            [self addActivityIndicator];
        }

        __weak __typeof(self)wself = self;
        id <SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadImageWithURL:url options:options progress:progressBlock completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            [wself removeActivityIndicator];
            if (!wself) return;
            dispatch_main_sync_safe(^{
                if (!wself) return;
                if (image && (options & SDWebImageAvoidAutoSetImage) && completedBlock)
                {
                    completedBlock(image, error, cacheType, url);
                    return;
                }
                else if (image) {
                    wself.image = image;
                    [wself setNeedsLayout];
                } else {
                    if ((options & SDWebImageDelayPlaceholder)) {
                        wself.image = placeholder;
                        [wself setNeedsLayout];
                    }
                }
                if (completedBlock && finished) {
                    completedBlock(image, error, cacheType, url);
                }
            });
        }];
        [self sd_setImageLoadOperation:operation forKey:@"UIImageViewImageLoad"];
    } else {
        dispatch_main_async_safe(^{
            [self removeActivityIndicator];
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                completedBlock(nil, error, SDImageCacheTypeNone, url);
            }
        });
    }
}
```
到这里这块儿代码相信我们很容易能看懂，最终的completedBlock会回调到这里并做一些UI上的操作，其中有一句话`[self sd_setImageLoadOperation:operation forKey:@"UIImageViewImageLoad"];
 `看一下它的实现：
```
- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key {
    [self sd_cancelImageLoadOperationWithKey:key];
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    [operationDictionary setObject:operation forKey:key];
}
```
`operationDictionary `是一个可变字典，储存着`SDWebImageManager`中核心方法返回的`operation`(就是那个让人不知道做什么用的operation)，以url作为key一一对应，我们看到第一句`[self sd_cancelImageLoadOperationWithKey:key];`，具体代码就补贴了，他是拿到了对应的`operation`并做了一个cancel操作，这样可以保证同样的url同一时间不会被重复执行加载操作，节约了资源，不得不佩服作者的严谨性！

##### 小结
至此我们就完成了SDWebImage源码的阅读工作，但也只是走马观花地了解了一个大概。大到是框架结构，小到代码风格，这个框架有太多的东西值得我们去学习推敲，方方面面都透露着作者以及众多贡献者的智慧，我们读起来也一定会受益良多。另外，在阅读源码过程中，[这个博客](http://www.cnblogs.com/polobymulberry/)给了我很大的帮助，对博客作者桑果同学一并表示感谢。

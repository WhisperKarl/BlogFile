---
title: SDWebImage源码（二）——SDImageCache缓存器
date: 2016-12-09 16:16:58
categories:
- 编程
tags:
- iOS
---
#### SDWebImage的缓存器
`SDImageCache`是`SDWebImage`的重要部件之一，它是一个单例类，完成了对图片的内存缓存、异步磁盘缓存、图片缓存查询等功能，这也是其优秀性能的原因所在，即下载过的图片将被缓存到内存和本地磁盘，当再次请求相同图片时直接从缓存中提取图片，从而大大提高了加载速度。`SDWebImage`的作者对核心方法都做了比较好的注释，这也大大提高了我们的阅读速度，我们先来看看头文件里的内容。
<!-- more -->
```
//缓存类型的枚举
typedef NS_ENUM(NSInteger, SDImageCacheType) {
//禁用缓存，直接从网络下载
SDImageCacheTypeNone,
//从磁盘获取缓存
SDImageCacheTypeDisk,
//从内存获取缓存
SDImageCacheTypeMemory
};

typedef void(^SDWebImageQueryCompletedBlock)(UIImage *image, SDImageCacheType cacheType);

typedef void(^SDWebImageCheckCacheCompletionBlock)(BOOL isInCache);

typedef void(^SDWebImageCalculateSizeBlock)(NSUInteger fileCount, NSUInteger totalSize);

/**
* SDImageCache maintains a memory cache and an optional disk cache. Disk cache write operations are performed
* asynchronous so it doesn’t add unnecessary latency to the UI.
*/
@interface SDImageCache : NSObject

/**
* Decompressing images that are downloaded and cached can improve performance but can consume lot of memory.
* Defaults to YES. Set this to NO if you are experiencing a crash due to excessive memory consumption.
*/
//是否提前解压图片，打开可以提高性能，但会消耗较多的内存，如果遇到内存警告，建议关闭该选项以及内存缓存选项
@property (assign, nonatomic) BOOL shouldDecompressImages;

/**
*  disable iCloud backup [defaults to YES]
*/
//是否启用iCloud
@property (assign, nonatomic) BOOL shouldDisableiCloud;

/**
* use memory cache [defaults to YES]
*/
// 是否启用内存缓存
@property (assign, nonatomic) BOOL shouldCacheImagesInMemory;

/**
* The maximum "total cost" of the in-memory image cache. The cost function is the number of pixels held in memory.
*/
// 内存允许的最大内存容量，该数值是可以存储的最大像素数
@property (assign, nonatomic) NSUInteger maxMemoryCost;

/**
* The maximum number of objects the cache should hold.
*/
// 内存中允许的最大缓存数量
@property (assign, nonatomic) NSUInteger maxMemoryCountLimit;

/**
* The maximum length of time to keep an image in the cache, in seconds
*/
// 磁盘缓存保留的最长时间，以秒计
@property (assign, nonatomic) NSInteger maxCacheAge;

/**
* The maximum size of the cache, in bytes.
*/
// 磁盘缓存的最大容量，以字节计
@property (assign, nonatomic) NSUInteger maxCacheSize;

/**
* Returns global shared cache instance
*
* @return SDImageCache global instance
*/
+ (SDImageCache *)sharedImageCache;

/**
* Init a new cache store with a specific namespace
*
* @param ns The namespace to use for this cache store
*/
//以ns 进行一些初始化操作
- (id)initWithNamespace:(NSString *)ns;

/**
* Init a new cache store with a specific namespace and directory
*
* @param ns        The namespace to use for this cache store
* @param directory Directory to cache disk images in
*/
// 进行一些初始化操作
- (id)initWithNamespace:(NSString *)ns diskCacheDirectory:(NSString *)directory;

//获取磁盘缓存路径
-(NSString *)makeDiskCachePath:(NSString*)fullNamespace;

/**
* Add a read-only cache path to search for images pre-cached by SDImageCache
* Useful if you want to bundle pre-loaded images with your app
*
* @param path The path to use for this read-only cache path
*/
//添加只读路径，不常用
- (void)addReadOnlyCachePath:(NSString *)path;

/**
* Store an image into memory and disk cache at the given key.
*
* @param image The image to store
* @param key   The unique image cache key, usually it's image absolute URL
*/
// 以图片的请求路径作为唯一key保存图片到内存和磁盘缓存中
- (void)storeImage:(UIImage *)image forKey:(NSString *)key;

/**
* Store an image into memory and optionally disk cache at the given key.
*
* @param image  The image to store
* @param key    The unique image cache key, usually it's image absolute URL
* @param toDisk Store the image to disk cache if YES
*/
//以图片的请求路径作为唯一key保存图片到内存和磁盘（可选）缓存中
- (void)storeImage:(UIImage *)image forKey:(NSString *)key toDisk:(BOOL)toDisk;

/**
* Store an image into memory and optionally disk cache at the given key.
*
* @param image       The image to store
* @param recalculate BOOL indicates if imageData can be used or a new data should be constructed from the UIImage
* @param imageData   The image data as returned by the server, this representation will be used for disk storage
*                    instead of converting the given image object into a storable/compressed image format in order
*                    to save quality and CPU
* @param key         The unique image cache key, usually it's image absolute URL
* @param toDisk      Store the image to disk cache if YES
*/
// 直接将图片的NSData（不转为图片对象）存储到内存或者磁盘中，节约空间和CPU占用
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk;

/**
* Store image NSData into disk cache at the given key.
*
* @param imageData The image data to store
* @param key   The unique image cache key, usually it's image absolute URL
*/
// 存储图片的二进制数据到磁盘缓存中
- (void)storeImageDataToDisk:(NSData *)imageData forKey:(NSString *)key;

/**
* Query the disk cache asynchronously.
*
* @param key The unique key used to store the wanted image
*/
//以key值异步查询磁盘中是否有该图片
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock;

/**
* Query the memory cache synchronously.
*
* @param key The unique key used to store the wanted image
*/
//以key值同步查询内存中是否有该图片
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key;

/**
* Query the disk cache synchronously after checking the memory cache.
*
* @param key The unique key used to store the wanted image
*/
// 同步查询磁盘中是否有某张图片
- (UIImage *)imageFromDiskCacheForKey:(NSString *)key;

/**
* Remove the image from memory and disk cache synchronously
*
* @param key The unique image cache key
*/
// 同步移除内存和磁盘中某张图片的缓存
- (void)removeImageForKey:(NSString *)key;


/**
* Remove the image from memory and disk cache asynchronously
*
* @param key             The unique image cache key
* @param completion      An block that should be executed after the image has been removed (optional)
*/
// 异步移除内存和磁盘中的某张图片的缓存,并执行完成回调
- (void)removeImageForKey:(NSString *)key withCompletion:(SDWebImageNoParamsBlock)completion;

/**
* Remove the image from memory and optionally disk cache asynchronously
*
* @param key      The unique image cache key
* @param fromDisk Also remove cache entry from disk if YES
*/
// 异步移除内存和磁盘（可选）中的某张图片的缓存
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk;

/**
* Remove the image from memory and optionally disk cache asynchronously
*
* @param key             The unique image cache key
* @param fromDisk        Also remove cache entry from disk if YES
* @param completion      An block that should be executed after the image has been removed (optional)
*/
//  异步移除内存和磁盘（可选）中的某张图片的缓存,并执行完成回调
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion;

/**
* Clear all memory cached images
*/
// 清除所有内存缓存
- (void)clearMemory;

/**
* Clear all disk cached images. Non-blocking method - returns immediately.
* @param completion    An block that should be executed after cache expiration completes (optional)
*/
// 清除磁盘缓存
- (void)clearDiskOnCompletion:(SDWebImageNoParamsBlock)completion;
- (void)clearDisk;


/**
* Remove all expired cached image from disk. Non-blocking method - returns immediately.
* @param completionBlock An block that should be executed after cache expiration completes (optional)
*/
// 移除磁盘中过期的缓存
- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock;
- (void)cleanDisk;

/**
* Get the size used by the disk cache
*/
// 获取磁盘缓存占用容量
- (NSUInteger)getSize;

/**
* Get the number of images in the disk cache
*/
// 获取磁盘缓存图片数量
- (NSUInteger)getDiskCount;

/**
* Asynchronously calculate the disk cache's size.
*/
// 异步计算磁盘缓存大小
- (void)calculateSizeWithCompletionBlock:(SDWebImageCalculateSizeBlock)completionBlock;

/**
*  Async check if image exists in disk cache already (does not load the image)
*
*  @param key             the key describing the url
*  @param completionBlock the block to be executed when the check is done.
*  @note the completion block will be always executed on the main queue
*/
// 异步查询磁盘中是否有某一张图片的缓存，并执行回调
- (void)diskImageExistsWithKey:(NSString *)key completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;
- (BOOL)diskImageExistsWithKey:(NSString *)key;

/**
*  Get the cache path for a certain key (needs the cache path root folder)
*
*  @param key  the key (can be obtained from url using cacheKeyForURL)
*  @param path the cache path root folder
*
*  @return the cache path
*/
// 以key值获取指定路径下的图片的磁盘缓存路径
- (NSString *)cachePathForKey:(NSString *)key inPath:(NSString *)path;

/**
*  Get the default cache path for a certain key
*
*  @param key the key (can be obtained from url using cacheKeyForURL)
*
*  @return the default cache path
*/
// 以key值获取默认路径下的图片缓存路径
- (NSString *)defaultCachePathForKey:(NSString *)key;

```
虽然方法看上去挺多的但其实核心的方法就是对图片缓存的增、删、查的实现，只不过每种功能可能有多种实现方式，但本质上是一样的，我们只需关注核心的几个方法即可。

`SDWebImage`实现了内存缓存和磁盘缓存，内存缓存是通过`NSCache`实现，磁盘缓存是通过`NSFileManager`来实现文件的存储，磁盘缓存是异步实现的。
#### 初始化
首先定义了一个继承于`NSCache`的类`AutoPurgeCache`，当收到内存警告时，默认清除所有缓存
```
@interface AutoPurgeCache : NSCache
@end

@implementation AutoPurgeCache

- (id)init
{
    self = [super init];
    if (self) {
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(removeAllObjects) name:UIApplicationDidReceiveMemoryWarningNotification object:nil];
}
return self;
}

- (void)dealloc
    {
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationDidReceiveMemoryWarningNotification object:nil];

}
```
初始化`SDImageCache`
```
+ (SDImageCache *)sharedImageCache {
//创建单例类，copy会调用init方法
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}

- (id)init {
    return [self initWithNamespace:@"default"];
}

- (id)initWithNamespace:(NSString *)ns {
    //获取磁盘缓存的默认路径
    NSString *path = [self makeDiskCachePath:ns];
    return [self initWithNamespace:ns diskCacheDirectory:path];
}

- (id)initWithNamespace:(NSString *)ns diskCacheDirectory:(NSString *)directory {
    if ((self = [super init])) {
        //缓存空间的默认名称
        NSString *fullNamespace = [@"com.hackemist.SDWebImageCache." stringByAppendingString:ns];

        // initialise PNG signature data
        //用于验证png格式
        kPNGSignatureData = [NSData dataWithBytes:kPNGSignatureBytes length:8];

        // Create IO serial queue
        //初始化串行队列，用于异步写入磁盘
        _ioQueue = dispatch_queue_create("com.hackemist.SDWebImageCache", DISPATCH_QUEUE_SERIAL);

        // Init default values
        //磁盘缓存存储时间，默认为一周
        _maxCacheAge = kDefaultCacheMaxCacheAge;

        // Init the memory cache
        //初始化内存缓存
        _memCache = [[AutoPurgeCache alloc] init];
        _memCache.name = fullNamespace;

        // Init the disk cache
        //磁盘缓存的默认路径
        if (directory != nil) {
            _diskCachePath = [directory stringByAppendingPathComponent:fullNamespace];
        } else {
            NSString *path = [self makeDiskCachePath:ns];
            _diskCachePath = path;
        }

        //下面是一些属性的初始化值
        _shouldDecompressImages = YES;
        _shouldCacheImagesInMemory = YES;
        _shouldDisableiCloud = YES;

        dispatch_sync(_ioQueue, ^{
          //初始化文件管理器
            _fileManager = [NSFileManager new];
        });

//下面是一些清除缓存的通知
#if TARGET_OS_IOS
        // Subscribe to app events
      //收到内存警告时清除内存缓存
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(clearMemory)
                                                     name:UIApplicationDidReceiveMemoryWarningNotification
                                                   object:nil];
        //每次进程杀死前都清理过期(默认大于一周)的磁盘缓存
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(cleanDisk)
                                                     name:UIApplicationWillTerminateNotification
                                                   object:nil];
      //后台挂起时清理过期磁盘缓存
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(backgroundCleanDisk)
                                                     name:UIApplicationDidEnterBackgroundNotification
                                                   object:nil];
#endif
    }

    return self;
}

```
以上就是`SDImageCache`的初始化过程，主要是初始化内存缓存对象以及初始化磁盘缓存目录，以及对一些默认参数的赋值。
#### 缓存图片
虽然`SDImageCache`有多个存储图片的方法，但都是基于下面这一个方法并传入不同的参数而已：
```
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk {
    if (!image || !key) {
        return;
    }
    // if memory cache is enabled
    //是否允许内存缓存
    if (self.shouldCacheImagesInMemory) {
        //计算图片所占空间
        //NSUInteger SDCacheCostForImage(UIImage *image) {
         // return image.size.height * image.size.width * image.scale * image.scale;}
        NSUInteger cost = SDCacheCostForImage(image);
        //写入内存缓存
        [self.memCache setObject:image forKey:key cost:cost];
    }

    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            NSData *data = imageData;

            if (image && (recalculate || !data)) {
            //如果需要重新计算(recalculate)或者data为0,需要将image转化为二进制数据
#if TARGET_OS_IPHONE
                // We need to determine if the image is a PNG or a JPEG
                // PNGs are easier to detect because they have a unique signature (http://www.w3.org/TR/PNG-Structure.html)
                // The first eight bytes of a PNG file always contain the following (decimal) values:
                // 137 80 78 71 13 10 26 10

                // If the imageData is nil (i.e. if trying to save a UIImage directly or the image was transformed on download)
                // and the image has an alpha channel, we will consider it PNG to avoid losing the transparency

              //这里这段代码做的一个工作是判断图片格式
             //有两种方式：一种方式是判断二进制数据前8位是否是png特定格式(137 80 78 71 13 10 26 10)；一种是判断图片是否有透明度
                int alphaInfo = CGImageGetAlphaInfo(image.CGImage);
                BOOL hasAlpha = !(alphaInfo == kCGImageAlphaNone ||
                                  alphaInfo == kCGImageAlphaNoneSkipFirst ||
                                  alphaInfo == kCGImageAlphaNoneSkipLast);
                //如果有透明度 则证明是png格式
                BOOL imageIsPng = hasAlpha;

                // But if we have an image data, we will look at the preffix
                //如果有data数据，直接判断起始数据是否符合格式
                if ([imageData length] >= [kPNGSignatureData length]) {
                    imageIsPng = ImageDataHasPNGPreffix(imageData);
                }

              //如果是png格式，转为data数据, 否则转为png
                if (imageIsPng) {

                    data = UIImagePNGRepresentation(image);
                }
                else {
                    data = UIImageJPEGRepresentation(image, (CGFloat)1.0);
                }
#else  
              //非iOS平台的处理，看不懂，总之就是获取图片的data数据
                data = [NSBitmapImageRep representationOfImageRepsInArray:image.representations usingType: NSJPEGFileType properties:nil];
#endif
            }
            //把二进制数据存入磁盘
            [self storeImageDataToDisk:data forKey:key];
        });
    }
}
```
以上这段代码比较简单，主要是对传入参数的合法性做一些判断以及对image的格式做判断并获取NSData数据，继续看storeImageDataToDisk:方法：
```
- (void)storeImageDataToDisk:(NSData *)imageData forKey:(NSString *)key {

    if (!imageData) {
        return;
    }

    if (![_fileManager fileExistsAtPath:_diskCachePath]) {
      //如果默认磁盘缓存路径不存在，则创建缓存文件夹
        [_fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
    }

    // get cache Path for image key
    //得到单独文件的路径，这里对URL做了一个MD5加密+扩展名作为文件名，具体代码不贴了，源码里都可以看到
    NSString *cachePathForKey = [self defaultCachePathForKey:key];
    // transform to NSUrl
    // 生成路径URL用以iCloud备份
    NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
    //写入缓存文件
    [_fileManager createFileAtPath:cachePathForKey contents:imageData attributes:nil];

    // disable iCloud backup
    if (self.shouldDisableiCloud) {
        //iCloud备份，没有深入了解
        [fileURL setResourceValue:[NSNumber numberWithBool:YES] forKey:NSURLIsExcludedFromBackupKey error:nil];
    }
}
```
至此，图片缓存的过程就完成了，这时候在沙盒Library/Cache/目录下就会有一份图片的缓存。
下面看一下查询图片的接口，查询内存缓存的接口比较简单，只需要一句话
```
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key {
	return [self.memCache objectForKey:key];
}
```
看一下查询磁盘缓存的接口
```
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock {

// block回调传入image对象还有缓存对象
// typedef void(^SDWebImageQueryCompletedBlock)(UIImage *image, SDImageCacheType cacheType);
    if (!doneBlock) {
        return nil;
    }

    if (!key) {
        doneBlock(nil, SDImageCacheTypeNone);
        return nil;
    }

    // First check the in-memory cache...
    //如果内存缓存中查询到了，执行成功block
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        doneBlock(image, SDImageCacheTypeMemory);
        return nil;
    }

    //内存中没有，异步查询磁盘 
    NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            return;
        }

        @autoreleasepool {
            //对图片做了一系列的解码转换等操作，得到UIImage对象
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.shouldCacheImagesInMemory) {
            //如果允许内存缓存，写入内存以提高下次查询效率
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                //回到主线程，执行成功回调
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });

    return operation;
}

```
磁盘查询也是非常简单的，至于这个接口为什么返回一个`NSOperation`对象，我们在单独讲`SDWebImageManager`时再说。
再来看一下清理缓存的接口，分两种情况， 一种是`clear`一种是`clean`。`clear`比较暴力，直接移除内存缓存的对象或者直接移除磁盘缓存文件夹。着重看一下`cleanDisk`：
```
- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock {
    //异步执行
    dispatch_async(self.ioQueue, ^{
        NSURL *diskCacheURL = [NSURL fileURLWithPath:self.diskCachePath isDirectory:YES];
        //需要获取的文件属性：是否是文件夹、修改时间、文件大小
        NSArray *resourceKeys = @[NSURLIsDirectoryKey, NSURLContentModificationDateKey, NSURLTotalFileAllocatedSizeKey];

        // This enumerator prefetches useful properties for our cache files.
       // 枚举器，遍历磁盘缓存目录
        NSDirectoryEnumerator *fileEnumerator = [_fileManager enumeratorAtURL:diskCacheURL
                                                   includingPropertiesForKeys:resourceKeys
                                                                      options:NSDirectoryEnumerationSkipsHiddenFiles
                                                                 errorHandler:NULL];
        //过期日期：从现在往前推 maxCacheAge，超过这个日期的删除
        NSDate *expirationDate = [NSDate dateWithTimeIntervalSinceNow:-self.maxCacheAge];
        NSMutableDictionary *cacheFiles = [NSMutableDictionary dictionary];
        NSUInteger currentCacheSize = 0;

        // Enumerate all of the files in the cache directory.  This loop has two purposes:
        //
        //  1. Removing files that are older than the expiration date.
        //  2. Storing file attributes for the size-based cleanup pass.
        NSMutableArray *urlsToDelete = [[NSMutableArray alloc] init];
        for (NSURL *fileURL in fileEnumerator) {
            NSDictionary *resourceValues = [fileURL resourceValuesForKeys:resourceKeys error:NULL];

            // Skip directories.
            //如果是文件夹，跳过
            if ([resourceValues[NSURLIsDirectoryKey] boolValue]) {
                continue;
            }

            // Remove files that are older than the expiration date;
            //如果修改时间早于一周前，移除
            NSDate *modificationDate = resourceValues[NSURLContentModificationDateKey];
            if ([[modificationDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
                [urlsToDelete addObject:fileURL];
                continue;
            }

            // Store a reference to this file and account for its total size.
            //统计缓存总大小并暂存文件属性
            NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
            currentCacheSize += [totalAllocatedSize unsignedIntegerValue];
            [cacheFiles setObject:resourceValues forKey:fileURL];
        }

        for (NSURL *fileURL in urlsToDelete) {
            //移除缓存文件
            [_fileManager removeItemAtURL:fileURL error:nil];
        }

        // If our remaining disk cache exceeds a configured maximum size, perform a second
        // size-based cleanup pass.  We delete the oldest files first.
        //如果剩下的文件总大小大于我们定义的总的内存大小，再循环一遍，删除相对较老的文件
        if (self.maxCacheSize > 0 && currentCacheSize > self.maxCacheSize) {
            // Target half of our maximum cache size for this cleanup pass.
            //设定一个期望大小——删到最大内存的一半
            const NSUInteger desiredCacheSize = self.maxCacheSize / 2;

            // Sort the remaining cache files by their last modification time (oldest first).
            //按照文件修改时间由老到新排序
            NSArray *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                            usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                return [obj1[NSURLContentModificationDateKey] compare:obj2[NSURLContentModificationDateKey]];
                                                            }];

            // Delete files until we fall below our desired cache size.
            //对排序后的数组循环删除，知道缓存占用内存降到我们期望的数值，跳出循环
            for (NSURL *fileURL in sortedFiles) {
                if ([_fileManager removeItemAtURL:fileURL error:nil]) {
                    NSDictionary *resourceValues = cacheFiles[fileURL];
                    NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                    currentCacheSize -= [totalAllocatedSize unsignedIntegerValue];

                    if (currentCacheSize < desiredCacheSize) {
                        break;
                    }
                }
            }
        }
        if (completionBlock) {
          //在主线程执行完成回调
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}

```

以上就是SDImageCache的所有内容。

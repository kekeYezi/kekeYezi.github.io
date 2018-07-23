---
layout: post  
title: Objc 源码学习 · AFNetworking 
date: 2018-07-07 
description: Objc 源码学习 · AFNetworking 
tags: 源码学习
---

## 源码学习 · AFNetworking  

前言

AF的框架从学iOS 到现在一直在使用，之前看过2.0 ，3.0 还是第一次完整的看完.其实每次AF 版本变化都有可以学习的东西。这次看的是3.2.0版本。那么记录下我这次学习的心得吧。本文内容基本基于自己的疑问展开。

做技术，凡事多问几个为什么，总不会错。

### AFHTTPSessionManager

* NS_ASSUME_NONNULL_BEGIN，NS_ASSUME_NONNULL_END  nullable

主要的作用就是为了方便多添加 引出了疑问nonnull 和 nullable，作为工具类经常会被swift项目引入。 只有和Swift混编，swift调用oc的时候需要标注属性的 nonnull 和 nullable。这个就是帮助便捷添加nonnull 不为空。

* NSSecureCoding

据说是更安全，那么 安全在哪呢？

```
NSSecureCoding
使用NSCoding来读写用户数据文件的问题在于，把全部的类编码到一个文件里，也就意味着给了这个文件访问你APP里面实例类的权限。
 
NSSecureCoding 除了在解码时要指定key和要解码的对象的类，如果要求的类和从文件中解码出的对象的类不匹配，NSCoder会抛出异常，代表数据已经被篡改了。
```

* NS_DESIGNATED_INITIALIZER & NS_UNAVAILABLE

当一个类有多个初始化方法时，推荐使用这个，编译器会在你使用其他的初始化方法时进行警告，规范代码的一种方法了。

```
+ (instancetype)new NS_UNAVAILABLE;
- (instancetype)init NS_UNAVAILABLE; ///< 直接标记 init 方法不可用
- (instancetype)initWithUserID:(NSNumber *)userID;

// 在调用时给出提示
- (id) init __attribute__((unavailable("Must use initWithFoo: instead.")));

- (instancetype)initWithUserID:(NSNumber *)userID {
    self = [super init];
    if (self) {
        if (userID.integerValue <= 0) {
            // raise: 原因
            // format: 具体描述
            [NSException raise:@"error parameter" format:@"user id can not = %@", userID];
        }
        self.userID = userID;
    }
    return self;
}

```

* DEPRECATED_ATTRIBUTE

提醒调用着 方法即将废弃 不建议使用了。多用于以前是这种方法 做兼容处理



NSURLSessionConfiguration

NSURLSessionDataTask



* NSAssert  & NSParameterAssert

断言辅助debug 定位问题



* @dynamic securityPolicy;

自行实现 get setter 方法



* @throw [NSException exceptionWithName:@"Invalid Security Policy" reason:reason userInfo:nil];

同样是 打印错误信息 和 断言有什么区别呢？



## AFURLSessionManager

NSURLSessionDelegate, NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate



NSOperationQueue

回调放在子线程

```
    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
```



dispatch_queue_t. dispatch_group_t



判断版本号

```
#ifndef NSFoundationVersionNumber_iOS_8_0
#define NSFoundationVersionNumber_With_Fixed_5871104061079552_bug 1140.11
#else
#define NSFoundationVersionNumber_With_Fixed_5871104061079552_bug NSFoundationVersionNumber_iOS_8_0
#endif
```



把delegate swizze block抛出，让业务更连贯， nsurlsession 的给的控制都是代理形式。但是业务用block更好

那我如果不用af 会有什么遗憾？Progress 拿不到了。拿得到  但是在 delegate里面 af把它弄成了 block



url_session_manager_create_task_safely 解决iOS7 的bug

static inline void

_AFURLSessionTaskSwizzling

@interface _AFURLSessionTaskSwizzling : NSObject 为什么用下划线？

```
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
```



@property (readwrite, nonatomic, strong) NSLock *lock;

mutableTaskDelegatesKeyedByTaskIdentifier nonatomic，我换成 atomic 可以么？

```
[self.lock lock];
    [self removeNotificationObserverForTask:task];
    [self.mutableTaskDelegatesKeyedByTaskIdentifier removeObjectForKey:@(task.taskIdentifier)];
    [self.lock unlock];
```

```
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(dataTasks))]) {
            tasks = dataTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(uploadTasks))]) {
            tasks = uploadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(downloadTasks))]) {
            tasks = downloadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(tasks))]) {
            tasks = [@[dataTasks, uploadTasks, downloadTasks] valueForKeyPath:@"@unionOfArrays.self"];
        }

        dispatch_semaphore_signal(semaphore);
    }];

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```

信号量的使用

请求1与请求2要在请求3之前完成 



getTasksWithCompletionHandler



重写方法？ 保证 block这个 参数 不能为空

```
- (BOOL)respondsToSelector:(SEL)selector {
    if (selector == @selector(URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:)) {
        return self.taskWillPerformHTTPRedirection != nil;
    } else if (selector == @selector(URLSession:dataTask:didReceiveResponse:completionHandler:)) {
        return self.dataTaskDidReceiveResponse != nil;
    } else if (selector == @selector(URLSession:dataTask:willCacheResponse:completionHandler:)) {
        return self.dataTaskWillCacheResponse != nil;
    }
#if !TARGET_OS_OSX
    else if (selector == @selector(URLSessionDidFinishEventsForBackgroundURLSession:)) {
        return self.didFinishEventsForBackgroundURLSession != nil;
    }
#endif

    return [[self class] instancesRespondToSelector:selector];
}

```



## AFNetworkReachabilityManager

NSLocalizedStringFromTable

```
static const void * AFNetworkReachabilityRetainCallback(const void *info) {
    return Block_copy(info);
}

static void AFNetworkReachabilityReleaseCallback(const void *info) {
    if (info) {
        Block_release(info);
    }
}

```



## AFSecurityPolicy

```
static NSData * AFSecKeyGetData(SecKeyRef key) {
    CFDataRef data = NULL;

    __Require_noErr_Quiet(SecItemExport(key, kSecFormatUnknown, kSecItemPemArmour, NULL, &data), _out);

    return (__bridge_transfer NSData *)data;

_out:
    if (data) {
        CFRelease(data);
    }

    return nil;
}

```



```
static BOOL AFSecKeyIsEqualToKey(SecKeyRef key1, SecKeyRef key2) {
#if TARGET_OS_IOS || TARGET_OS_WATCH || TARGET_OS_TV
    return [(__bridge id)key1 isEqual:(__bridge id)key2];
#else
    return [AFSecKeyGetData(key1) isEqual:AFSecKeyGetData(key2)];
#endif
}

```



## AFURLRequestSerialization

不直接用 NSMutableURLRequest ，会带来哪些困扰？



参考链接

https://cdn2.jianshu.io/p/856f0e26279d?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation


---
layout: post  
title: iOS App Optimization series 4 耗时统计
date: 2019-05-18 
description: iOS App Optimization series 4 耗时统计
tags: 质量
---

对于耗时统计作用最有价值的莫过于以下几点

* [启动时间优化](https://kekeyezi.github.io/2018/08/iOS-App-optimization-series-2-startup/)

关于启动时间优化之前有文章分析过，如果没有心情细看可以直接使用最终方案。

起点时间

```
+ (BOOL)processInfoForPID:(int)pid procInfo:(struct kinfo_proc*)procInfo
{
    int cmd[4] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, pid};
    size_t size = sizeof(*procInfo);
    return sysctl(cmd, sizeof(cmd)/sizeof(*cmd), procInfo, &size, NULL, 0) == 0;
}

+ (NSTimeInterval)processStartTime
{
    struct kinfo_proc kProcInfo;
    if ([self processInfoForPID:[[NSProcessInfo processInfo] processIdentifier] procInfo:&kProcInfo]) {
        return kProcInfo.kp_proc.p_un.__p_starttime.tv_sec * 1000.0 + kProcInfo.kp_proc.p_un.__p_starttime.tv_usec / 1000.0;
    } else {
        NSAssert(NO, @"无法取得进程的信息");
        return 0;
    }
}
```

终点时间

```
+ (void)load {
    @autoreleasepool {
        __block id<NSObject> obs;
        obs = [[NSNotificationCenter defaultCenter] addObserverForName:UIApplicationDidBecomeActiveNotification
                                                                object:nil queue:nil
                                                            usingBlock:^(NSNotification *note) {
                                                                [self startUp];
                                                                [[NSNotificationCenter defaultCenter] removeObserver:obs];
                                                            }];
    }
}
```

两边的时间差就是一定意义上的启动时间，可以用作统计分析。ps：release模式下才有意义

* 监控各个业务是否给用户带来卡顿的感觉

​        其实就是用户点开页面的响应时间了。从我点击要进行跳转到用户见到他想要的界面并且能够操作 这段时间如果越快就给人一种越流畅的感觉。那么这么时间应该要怎么进行统计呢？很自然的想到 vc的init -> viewDidApper。利用Swizzle 可以hook对应的系统方案达到自动统计的作用。这种方案应该是比较成熟也相对来说门槛不高。但是如果要求更高一些，我需要线上监控某个我开发的需求里面最占用性能的函数是什么，又该如何处理呢？

推荐一个最近发现处理的比较好的库[MTHawkeye](https://github.com/meitu/MTHawkeye/blob/develop/MTHawkeye.podspec)

虽然star数不多，但是基本实现了我一期想要做到的效果。还没有体验的小伙伴可以下载试试他们的功能。

里面做了2块有意义的事情。

1. Hook vc的生命周期做AOP打点监控
2. 利用fishhook+ ARM 64 汇编达到函数级别的监控



直接上代码吧

他将启动流程定义为这几个流程

![image-20190519154419217](/assets/images/2019-05/code_time.png)

定义了一下几个时间点进行Hook 统计

```
typedef NS_ENUM(NSInteger, MTHViewControllerLifeCycleStep) {
    MTHViewControllerLifeCycleStepInitExit = 0,
    MTHViewControllerLifeCycleStepLoadViewEnter,
    MTHViewControllerLifeCycleStepLoadViewExit,
    MTHViewControllerLifeCycleStepViewDidLoadEnter,
    MTHViewControllerLifeCycleStepViewDidLoadExit,
    MTHViewControllerLifeCycleStepViewWillAppearEnter,
    MTHViewControllerLifeCycleStepViewWillAppearExit,
    MTHViewControllerLifeCycleStepViewDidAppearEnter,
    MTHViewControllerLifeCycleStepViewDidAppearExit,
    MTHViewControllerLifeCycleStepUnknown,
};

```

下面两段代码应该见的很多了 而且写法都一样

```
// MARK: replacement objc_msgSend (arm64)
//replacement objc_msgSend (arm64)
// https://blog.nelhage.com/2010/10/amd64-and-va_arg/
// http://infocenter.arm.com/help/topic/com.arm.doc.ihi0055b/IHI0055B_aapcs64.pdf
// https://developer.apple.com/library/ios/documentation/Xcode/Conceptual/iPhoneOSABIReference/Articles/ARM64FunctionCallingConventions.html
#define call(b, value)                            \
    __asm volatile("stp x8, x9, [sp, #-16]!\n");  \
    __asm volatile("mov x12, %0\n" ::"r"(value)); \
    __asm volatile("ldp x8, x9, [sp], #16\n");    \
    __asm volatile(#b " x12\n");

#define save()                      \
    __asm volatile(                 \
        "stp x8, x9, [sp, #-16]!\n" \
        "stp x6, x7, [sp, #-16]!\n" \
        "stp x4, x5, [sp, #-16]!\n" \
        "stp x2, x3, [sp, #-16]!\n" \
        "stp x0, x1, [sp, #-16]!\n" \
                                    \
        "stp q8, q9, [sp, #-32]!\n" \
        "stp q6, q7, [sp, #-32]!\n" \
        "stp q4, q5, [sp, #-32]!\n" \
        "stp q2, q3, [sp, #-32]!\n" \
        "stp q0, q1, [sp, #-32]!\n");

#define load()                    \
    __asm volatile(               \
        "ldp q0, q1, [sp], #32\n" \
        "ldp q2, q3, [sp], #32\n" \
        "ldp q4, q5, [sp], #32\n" \
        "ldp q6, q7, [sp], #32\n" \
        "ldp q8, q9, [sp], #32\n" \
                                  \
        "ldp x0, x1, [sp], #16\n" \
        "ldp x2, x3, [sp], #16\n" \
        "ldp x4, x5, [sp], #16\n" \
        "ldp x6, x7, [sp], #16\n" \
        "ldp x8, x9, [sp], #16\n");

#define link(b, value)                           \
    __asm volatile("stp x8, lr, [sp, #-16]!\n"); \
    __asm volatile("sub sp, sp, #16\n");         \
    call(b, value);                              \
    __asm volatile("add sp, sp, #16\n");         \
    __asm volatile("ldp x8, lr, [sp], #16\n");

#define ret() __asm volatile("ret\n");

```

```
__attribute__((__naked__)) static void hook_Objc_msgSend() {
    // Save parameters.
    save()

        __asm volatile("mov x2, lr\n");
    __asm volatile("mov x3, x4\n");

    // Call our before_objc_msgSend.
    call(blr, &before_objc_msgSend)

        // Load parameters.
        load()

        // Call through to the original objc_msgSend.
        call(blr, orig_objc_msgSend)

        // Save original objc_msgSend return value.
        save()

        // Call our after_objc_msgSend.
        call(blr, &after_objc_msgSend)

        // restore lr
        __asm volatile("mov lr, x0\n");

    // Load original objc_msgSend return value.
    load()

        // return
        ret()
}
```

其实这些代码都出自 戴铭老师之手。从这里可以看到他的[开源项目](https://github.com/ming1016/GCDFetchFeed)只不过关于统计的代码他和其他功能耦合在一块并没有单独提出来。但是MTHawkeye 进行了分离，所以我们能直观的看到和Instruments的time profile一样的功能。

* 手动打点

还有很多并不是通用的逻辑 我们也希望能在线上进行监控统计，那其实就和埋点一样 虽然有无痕埋点的策略，但是还是有少部分需求需要定制化，特殊化处理。比如我们需要监控某个请求链（多个接口返回合并算业务成功）这个时候手动埋点可能就更方便。

* 网络耗时

资料待整理，这块也是利用系统给的回调函数做AOP分析

#### 总结一下：

关于时间耗时统计介绍了3个常用场景。虽然统计和记录了数据，但是怎么上传，怎么分析，怎么通知开发解决问题 每个环节可能都会遇到这样那样的问题。走一步看一步吧～

### 写在最后：

Better Late Than Never ，愿我们都能跨出自己的第一步。

站在巨人的肩膀上，感谢前辈们的文章。

## 参考链接：



<https://github.com/ming1016/GCDFetchFeed>
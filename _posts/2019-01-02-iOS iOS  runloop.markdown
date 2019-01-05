---
layout: post  
title: iOS RunLoop
date: 2019-01-02 
description: 从问题出发解释RunLoop概念
tags: iOS
---

前言：

​	2019年的第一篇博客 。如果说每年的都有一个主题，那么我对自己2019年的要求就是夯实基础。之前很多概念要么很早就了解过，要么知识已经更新迭代。所以导致很多知识不用已经遗忘或者就是当时根本也理解的不够透彻。借着培养写博客的习惯梳理一遍自己的知识点，夯实一下自己的基础。

​	其实RunLoop的知识也不是新鲜知识，很多很好的博客从2015年就从源码层次去分析过了，自己之前对RunLoop的理解也是一些很浅的概念。这次就不从原理讲解了，从文末的资料就可以看到大部分很高质量的博客。我认为有些事情结论，或者结果其实并不重要，经历和路程比较重要。比如一个很高质量的库你看懂了自然是好事，但是他是怎么从一个普通库变成即简洁又实用的库才是比较有意义的。 我们带着问题进入这次博客内容吧～



~~问题1:什么是RunLoop？ RunLoop的存在意义是啥，能不能没有这玩意？~~

问题2:RunLoop是不是很底层，和我们日常开发有关系么？他的应用场景都有哪些？哪些库有用到RunLoop？

~~问题3:为什么总是要把RunLoop和线程放在一起来讲？他们两是啥关系？~~

~~问题4:RunLoop与GCD、Autorelease Pool 是什么关系？~~

~~问题5:那些大神是怎么知道这些知识的？~~

~~是不是这么多问题一时都不好解释？ 没关系我们一点点来，很多东西拆开解决就好很多~~

本来是想一篇文章解释很多内容，后面发现工作量太大了，我们就从实际出发，过程中介绍一下概念神马的。

友情推荐[demo工程](https://github.com/kekeYezi/KKRunLoopDemo)  里面包含了RunLoop源码和运用场景Demo。

### 正文

RunLoop的概念 其实也不是很底层，毕竟核心代码是开源的，并且代码数也就4000行的样子，硬啃还是有希望啃完的。[传送门](http://opensource.apple.com/tarballs/CF/ )（<http://opensource.apple.com/tarballs/CF/> ） 

RunLoop实质上就是一个 do while 循环 (详情源码第2308行)

```
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode)
```

runloop 就是一个循环线程，常驻 一直在跑的，核心功能就是： 引用一段解释：

```
如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。
线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回

总的来讲就是：RunLoop是保证线程不会退出，并且能在不处理消息的时候让线程休眠，节约资源，在接收到消息的时候唤醒线程做出对应处理的消息循环机制。
```



```
从上面的代码可以看出，线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。
```

源码确实有全局字典

```
一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。
```



日常开发运用的场景我这么总结了一下 大概有以下几个场景（可能会漏，从个人觉得价值高低出发）：

1.  主线程卡顿监控 （ps：个人觉得 监控runloop 看卡顿还应该结合 fps cpu 内存等内容综合判定 ）
2. 线程保活
3. [AsyncDisplayKit](https://github.com/facebookarchive/AsyncDisplayKit) + [YYAsyncLayer](https://github.com/ibireme/YYAsyncLayer) 异步渲染
4. NStimer 滑动运行
5. 列表大图加载卡顿。 滑动结束刷新图片。
6. 应用复活



结合demo代码看效果最佳，注释和示例都在demo中

### 主线程卡顿检测

____

https://github.com/kekeYezi/KKRunLoopDemo/blob/master/KKRunLoopDemo/KKRunLoopDemo/ViewControllers/MainThreadMonitorViewController.m



扩展阅读：

https://www.jianshu.com/p/ea36e0f2e7ae 

https://www.jianshu.com/p/c8ee2103ca92

看到介绍多种卡顿检测方法

### 线程保活

____

https://github.com/kekeYezi/KKRunLoopDemo/blob/master/KKRunLoopDemo/KKRunLoopDemo/ViewControllers/ThreadAliveViewController.m



### 异步渲染

____

[YYAsyncLayer](https://github.com/ibireme/YYAsyncLayer) 

sunnyxx的UITableView+FDTemplateLayoutCell利用Observer在界面空闲状态下计算出UITableViewCell的高度并进行缓存。

获取线程或者app空闲的时候

### NSTimer

_____

运用场景太多 也比较简单 不再赘述



### 大图加载

____

暂时没有遇到场景，有需要再研究



### 应用复活

____

https://blog.csdn.net/u011619283/article/details/53673255



RUnLoop的概念可以从 源码对结构体的定义大致看出 RunLoop RunLoopMode 以及Source/Timer/Observer 的概念

一个runloop 有多个mode 一个mode 主要有三个东西 分别是Source/Timer/Observer 

```
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;  /* locked for accessing mode list */
    __CFPort _wakeUpPort;	// used for CFRunLoopWakeUp 内核向该端口发送消息可以唤醒runloop
    Boolean _unused;
    volatile _per_run_data *_perRunData; // reset for runs of the run loop
    pthread_t _pthread;             //RunLoop对应的线程
    uint32_t _winthread;
    CFMutableSetRef _commonModes;    //存储的是字符串，记录所有标记为common的mode
    CFMutableSetRef _commonModeItems;//存储所有commonMode的item(source、timer、observer)
    CFRunLoopModeRef _currentMode;   //当前运行的mode
    CFMutableSetRef _modes;          //存储的是CFRunLoopModeRef
    struct _block_item *_blocks_head;//doblocks的时候用到
    struct _block_item *_blocks_tail;
    CFTypeRef _counterpart;
};


struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;   //mode名称
    Boolean _stopped;    //mode是否被终止
    char _padding[3];
    //几种事件
    CFMutableSetRef _sources0;  //sources0
    CFMutableSetRef _sources1;  //sources1
    CFMutableArrayRef _observers; //通知
    CFMutableArrayRef _timers;    //定时器
    CFMutableDictionaryRef _portToV1SourceMap; //字典  key是mach_port_t，value是CFRunLoopSourceRef
    __CFPortSet _portSet; //保存所有需要监听的port，比如_wakeUpPort，_timerPort都保存在这个数组中
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};

struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};


struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits; //用于标记Signaled状态，source0只有在被标记为Signaled状态，才会被处理
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
        CFRunLoopSourceContext version0;	 /* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	 /* immutable, except invalidation */
    } _context;
};

struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;  //标记fire状态
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;        //添加该timer的runloop
    CFMutableSetRef _rlModes;     //存放所有 包含该timer的 mode的 modeName，意味着一个timer可能会在多个mode中存在
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;	  //理想时间间隔	/* immutable */
    CFTimeInterval _tolerance;    //时间偏差      /* mutable */
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};

```



##总结

复习了一次RunLoop的基础概念，运用的场景也一一列出。很多东西只是你不知道，知道了从搜索引擎还是能探索出很多知识的，今天就到这 祝大家满怀好奇继续学习～



###参考资料

官方文档 有必要看一下

[https://opensource.apple.com/source/CF/CF-635.19/CFRunLoop.c.auto.html](https://opensource.apple.com/source/CF/CF-635.19/CFRunLoop.c.auto.html)

[https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW24](

经典博客内容

[https://v.youku.com/v_show/id_XODgxODkzODI0.html](https://v.youku.com/v_show/id_XODgxODkzODI0.html)

[https://blog.ibireme.com/2015/05/18/runloop/](https://blog.ibireme.com/2015/05/18/runloop/)

[http://mrpeak.cn/blog/ios-runloop/](http://mrpeak.cn/blog/ios-runloop/)

[https://blog.csdn.net/u011619283/column/info/13762](https://blog.csdn.net/u011619283/column/info/13762)

cocoachina 总结

[http://www.cocoachina.com/ios/20180515/23380.html](http://www.cocoachina.com/ios/20180515/23380.html)

使用场景总结

[https://www.cnblogs.com/kenshincui/p/6823841.html](https://www.cnblogs.com/kenshincui/p/6823841.html)

卡顿检测策略

[https://blog.csdn.net/Philm_iOS/article/details/81200524](https://blog.csdn.net/Philm_iOS/article/details/81200524)

[https://www.jianshu.com/p/08e85de54ef6](https://www.jianshu.com/p/08e85de54ef6)

[http://sindrilin.com/2017/03/24/blocking_observe.html](http://sindrilin.com/2017/03/24/blocking_observe.html)

解释为什么是before 和 after wait 判断卡顿

[https://www.jianshu.com/p/6c10ca55d343](https://www.jianshu.com/p/6c10ca55d343)

运用总结

https://www.jianshu.com/p/adf9eb244e81


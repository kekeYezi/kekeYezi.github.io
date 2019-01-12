---
layout: post  
title: iOS App Optimization series 3 内存优化
date: 2019-01-07 
description: iOS App Optimization series 3 内存优化
tags: 优化
---

生在ARC时代我们是幸运的，还记得初期学习OC 就是在iOS5，那个时候还是MRC时代，又是初学，分不清内存怎么管理，各种崩溃各种酸爽。后面引入ARC机制后，对开发者总算友好了很多，但是不注意还是会引起内存问题。实际情况下解决稳定增长的内存问题并不难（毕竟能够复现），难就难在开发并不知道自己开发的功能发生了内存问题，那么我们就需要借助一些工具提醒我们项目里的内存问题。

下面我们从3个优化方向来总结一下 （配合[自制demo](https://github.com/kekeYezi/KKMemoryOptimization)食用更佳）

![memory](/assets/images/2019-01/memory.png)

##减少内存泄露

###定位问题

1.引起 内存泄露的点

* 循环引用
* nstimer
* 大图延迟释放
* ....

2.查看哪里没有dealloc,怎么定位出问题的vc或者类



### 解决问题

循环引用 目前我觉得最好的方案就是 FB的FBMemoryProfiler，轻量级的就是腾讯之前分享的MLeaksFinder。

关于图片内存 imageName 和 imageWithContentOfFile了解一下.



##降低内存使用峰值

记录内存峰值。Xcode的那个毕竟不能release上用，生产用户无法验证。APM里面有一块也是关于记录内存情况的。我们可以抽样记录内存情况然后分析。

在一些使用场景里,比如整个页面初始化,要分配整个使用内存,批量的图片处理（或者变量）,会出现段时间需要加载大量内容,占用过高的内存.需要利用autorelasepool 进行优化

怎么测量内存数值？这是网上找到和Xcode显示内存最接近的函数了。

```
// 获取当前应用的内存占用情况，和Xcode数值相近
+ (double)getMemoryUsage {
    task_vm_info_data_t vmInfo;
    mach_msg_type_number_t count = TASK_VM_INFO_COUNT;
    if(task_info(mach_task_self(), TASK_VM_INFO, (task_info_t) &vmInfo, &count) == KERN_SUCCESS) {
        return (double)vmInfo.phys_footprint / (1024 * 1024);
    } else {
        return -1.0;
    }
}
```



##减少内存异常引用

crash  BAD_EXCESS

这个问题比较复杂，线上的bug 除了exception 就是这个引起的崩溃最多了，也有比较多的优化空间。

搞一个自动增加内存的工具，模拟app高内存 高cpu状态下的情况。模拟线上环境

静态分析代码也能辅助预防一些问题。



###写在最后：

Better Late Than Never ，愿我们都能跨出自己的第一步。

站在巨人的肩膀上，感谢前辈们的文章。



##参考链接：

计算内存问题

[http://www.samirchen.com/ios-app-memory-usage/](http://www.samirchen.com/ios-app-memory-usage/)

总结

[https://github.com/SilongLi/AppPerformance](https://github.com/SilongLi/AppPerformance) 

instrument 方法

[https://blog.csdn.net/clovejq/article/details/78689759](https://blog.csdn.net/clovejq/article/details/78689759)

微信读书的库

[https://wereadteam.github.io/2016/02/22/MLeaksFinder/](https://wereadteam.github.io/2016/02/22/MLeaksFinder/)

**[https://juejin.im/entry/5965f8856fb9a06bad654502](https://juejin.im/entry/5965f8856fb9a06bad654502)**

内存泄露场景

**[https://www.jianshu.com/p/e9d989c12ff8](https://www.jianshu.com/p/e9d989c12ff8)**

[http://www.olinone.com/?p=25](http://www.olinone.com/?p=25)

野指针内存总结

[http://www.cocoachina.com/ios/20180917/24893.html](http://www.cocoachina.com/ios/20180917/24893.html)

自动管理 deallock 释放通知？

[https://cocoapods.org/pods/CYLDeallocBlockExecutor](https://cocoapods.org/pods/CYLDeallocBlockExecutor)

库的总结

[http://cnsue.me/2017/04/03/iOS%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E8%87%AA%E5%8A%A8%E6%A3%80%E6%B5%8B/](http://cnsue.me/2017/04/03/iOS%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E8%87%AA%E5%8A%A8%E6%A3%80%E6%B5%8B/)

自写leakfinder

**[http://yanfeng.life/2017/11/23/YFMemoryLeakDetector-intro/](http://yanfeng.life/2017/11/23/YFMemoryLeakDetector-intro/)**

市面上开源的方案

[http://cnsue.me/2017/04/03/iOS%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E8%87%AA%E5%8A%A8%E6%A3%80%E6%B5%8B/](http://cnsue.me/2017/04/03/iOS%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E8%87%AA%E5%8A%A8%E6%A3%80%E6%B5%8B/)

[https://github.com/facebook/FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)

[http://wereadteam.github.io/2016/07/20/MLeaksFinder2/](http://wereadteam.github.io/2016/07/20/MLeaksFinder2/)

[https://code.facebook.com/posts/583946315094347/automatic-memory-leak-detection-on-ios/](https://code.facebook.com/posts/583946315094347/automatic-memory-leak-detection-on-ios/)

instrument教学

[http://www.cocoachina.com/ios/20161013/17759.html](http://www.cocoachina.com/ios/20161013/17759.html)

介绍野指针

[http://www.cocoachina.com/ios/20180917/24893.html](http://www.cocoachina.com/ios/20180917/24893.html)
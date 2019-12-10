---
layout: post  
title: iOS App Optimization series 2 启动时间优化
date: 2018-08-26 
description: iOS App Optimization series 2 启动时间优化
tags: 优化
---

​	关于启动时间优化网上已经有很多相关资料，但大多数都在翻译和解释WWDC 2016/406 和 2017/413 的内容。苹果爸爸说的当然很对，但是看完可能还是找不到具体的解决方法。所以这里作者从自己实践的角度出发叙述下我从哪些方面进行操作了。当然了，没看过WWDC 这两个session的可以从参考链接进入先看一下概念。

贴心的我还准备了[demo](https://github.com/kekeYezi/iOS-Startup-Optimize-Demo)给大家边运行边看博客～

### 带着问题思考

1. 怎么统计我家app的启动时间？
2. 怎么定位出有问题代码进行针对的优化处理？
3. 异步线程操作会不会影响启动时间？
4. 哪些操作会影响到启动时间呢？

启动时间由 main 之前的启动时间和 main 之后的启动时间两部分组成。

![](/assets/images/2018-08/start-up-4.png) 

图片借鉴： [https://kangzubin.com/ios-app-launch-time/](https://kangzubin.com/ios-app-launch-time/)



### 理论部分：

* main 之前的启动时间:

根据WWDC里介绍的方法 在 Edit scheme -> Run -> Arguments 中将环境变量 DYLD_PRINT_STATISTICS 设为 1，就可以看到 main 之前各个阶段的时间消耗。

如图：![](/assets/images/2018-08/start-up-1.png) 

运行结果如下：

![](/assets/images/2018-08/start-up-2.png) 

还有一个方法获取更详细的时间，只需将环境变量 DYLD_PRINT_STATISTICS_DETAILS 设为 1 就可以。结果如下

![](/assets/images/2018-08/start-up-3.png) 

* main 之后的启动时间:

 对于main之后的时间范围主流的说法有几下几种。

1. main函数启动的时候到didFinishLaunchingWithOptions  最后一行代码结束的时间
2. main函数启动的时候到一个vc 视图 viewDidAppear 
3. main函数启动的时候到applicationDidBecomeActive 

当然以上的说法都是从函数执行到函数之间的时间。而我更喜欢下面的计算规则

从用户点击icon到看到视图的时候。

applicationDidBecomeActive 肯定是比viewDidAppear 要久的，统计applicationDidBecomeActive肯定是要比统计 viewDidAppear 要方便的。个人感觉既然都差不多话的就选方案3吧。

这个时间统计直接打点计算就可以，不过当遇到时间较长需要排查问题时，只统计两个点的时间其实不方便排查，目前见到比较好用的方式就是为把启动任务规范化、粒子化，针对每个任务都有打点统计，这样方便后期问题的定位和优化

那么关于问题1: 统计时间的问题 我们就可以在mian函数打上时间点。再在applicationDidBecomeActive 打上点

时间差值就是启动的时间了。不想自己写可以直接用demo里面用的 [https://github.com/kekeYezi/KKTimeWatch](https://github.com/kekeYezi/KKTimeWatch)

效果如下：

![](/assets/images/2018-08/start-up-5.png) 



那么问题2: 只要分段打出每个函数 就知道问题函数在哪。针对优化了～

为了更好的分段统计，建议大家也一起做下代码优化 处理下 复杂的AppDelegate，什么你说你的AppDelegate 不复杂很简洁？ 嗯 当我没说～



### 实践部分：

先看一下 demo里面  time Profile 跑出来的结果

![](/assets/images/2018-08/start-up-6.png) 

和打点统计出来的对比

![](/assets/images/2018-08/start-up-7.png) 

main 里面的start 和 打点的结果基本吻合，耗时函数也可以精准定位。



介于国产App 都有个奇怪的东西，启动广告。如果启动广告有3s的话，那么绝大部份的App 优化效果都变的不那么明显了，但是如果好好利用这个东西，把不是那么必要在启动加上的操作，放在启动图后面 会有意象不到的收获。那么我们重新定义下 启动顺序。

```
1. main
2. didFinishLaunchingWithOptions start
3. 必要第一梯队函数调用
4. 广告启动
5. 必要第二梯队函数调用
6. 广告结束
7. 显示主界面本地数据
8. 请求网络 刷新主界面网络数据
9. 非必要函数 放在需要的时候调用。
```

### 后续：



代码越多启动越慢，所以我们可以找找方法删除不用的变量，函数，类这些代码。

AppCode 代码审查功能可以帮助我们实现。Xcode 也可以辅助我们做一部分工作。

将不必须在+load方法中做的事情延迟到+initialize中



------------------------------------------------------------------------------------------------------------------------------------------------

2019-2-20 更新

关键词：background fetch

如果提升 短时间内频繁冷启动速度？ 淘宝和微信好像做过优化。ipa包里面的frameworks是个疑问点。

### 写在最后：



优化的事情 没有十足的把握还是不要急于上线，经过充分的测试和评估上线才是正道。如果自己的启动时间还没到急需优化的地步也不一定要纠结那么几百毫秒。过度的优化还是要不得的～



Better Late Than Never ，愿我们都能跨出自己的第一步。

站在巨人的肩膀上，感谢前辈们的文章。



## 参考链接：



[https://developer.apple.com/videos/play/wwdc2016/406](https://developer.apple.com/videos/play/wwdc2016/406)

[https://developer.apple.com/videos/play/wwdc2017/413](https://developer.apple.com/videos/play/wwdc2017/413)

[https://www.jianshu.com/p/c14987eee107](https://www.jianshu.com/p/c14987eee107)

[https://juejin.im/post/5a31190751882559e225a775](https://juejin.im/post/5a31190751882559e225a775)

[https://kangzubin.com/ios-app-launch-time](https://kangzubin.com/ios-app-launch-time)

[https://zhuanlan.zhihu.com/p/38183046?https://chars.tech/blog/ios-app-launch-time-optimize](https://zhuanlan.zhihu.com/p/38183046?https://chars.tech/blog/ios-app-launch-time-optimize)

[https://techblog.toutiao.com/2017/01/17/iosspeed](https://techblog.toutiao.com/2017/01/17/iosspeed)

https://ming1016.github.io/2019/12/07/how-to-analyze-startup-time-cost-in-ios/
---
layout: post  
title: iOS App Optimization series 1 体积优化 
date: 2018-06-03 
description: iOS App Optimization series 1 体积优化  
tags: 优化
---



17年公司有任务要求减少安装包体积，本文为了记录自己优化App，给安装包做瘦身的时候总结的一些思路，方法和脚本。大家可以在这里找到本文用到的一些脚本，我先上一张我解决问题思路的脑图，可以先看也可以回头过来看。有个简单的概念就好。

![](/assets/images/2018-06/App体积优化通用版.png)

### 检测自己的App的表现

知己知彼才能百战不殆。谁家公司还没有个竞品分析的对手呢？首先要知道自己与知名App或者竞争对手App的一个差距才能知道自己在哪方面做的不足。以蚂蚁财富为例：

![ipa分析](/assets/images/2018-06/ipa分析.png)

没有后缀的大部分就是二进制文件，占比有百分之82多。

可以对比下数据看看自己的App的体积构成。看看代码占多数，图片占多数，json，plist占多数。脚本代码：https://github.com/kekeYezi/Starfish

当然二进制的构成也可以根据linkmap（http://blog.cnbang.net/tech/2296/）分析出来。

知道大体的一个概念之后我们就可以进行实际的优化操作了。针对自己App表现不好的地方对症下药才能效果显著。



### 可以优化的方向

往大的分，无非就是资源和代码两个部分。这里我按照自己的经验列出对最终结果会产生较大收益的点，作为一个优先级排序。

#####图片和资源文件

1. 图片压缩

大部分开发人员在接收设计师的图片的时候，往往不注意图片的压缩，所以给到的图片直接就加入工程，日积月累就会导致这块的浪费非常之多。从这里（https://github.com/kekeYezi/Starfish）你可以方便的压缩项目里的所有图片以及统计出压缩比例，采用的是 tinypng（[https://tinypng.com](https://tinypng.com) ）的服务 ，效果如图。

![demo1](/assets/images/2018-06/demo1.png)

使用python脚本这里不做过多赘述。

2. 图片去重检查图片文件的md5 作为重复依据，可参考这个脚本
3. 没用图片 @1x，或者使用<https://github.com/tinymind/LSUnusedResource> 
4. 除图片代码外的资源

比如，mp3 mp4，json，plist等等。往往有些没用到，有些又没有压缩。针对这些文件的压缩处理大家各显神通吧。

5. 动态下载非必须文件

这个得和产品和项目负责人商量哪有有必要哪有不必要。视具体情况而定，但是一股脑都忘App本地塞东西肯定不是什么好的做法。

6. H5页面 远端化

有些页面为了加载速度会把资源和图片都放在本地，但是是否所有的业务都需要这么依赖速度？这个也和上一个情况一样视情况而定吧。

7. 功能一样的图片



## 二进制优化&编译选项优化

代码占据了app体积的绝大部份，所以这方便的优化也是重头戏。

1. 去掉未使用的组件库
2. 根据appCode 和 自己对项目的掌握程度可以知道哪些组件，类，函数是可以删除的。
3. 重复代码检查
4. 是否.a 支持 多余的架构 能删就删




### 编译选项优化

可以参考脑图




## 后续流程如何规范

* 避免重复库的引入
* 新接入图片 习惯性压缩
* 注意平时的开发习惯，废弃模块及早清理。不及时清理也要标注
* 同质的开源库（譬如AFnetworking vs ASIHttpRequest），只接入一种。
* 建立预警机制， 一般上线后，都是脚本打包，除了正常的生成ipa包之外，也要生成分析文件， 列出相对上一次上线包大了多少，类文件增加了多少。这样的机制也有助于防止安装包悄无声息的变成巨物。
*  建立监控通过对LinkMap文件的分析，可以得知每个模块可执行文件占用大小。再对比两个版本，就知道业务模块的增量大小。



TODO：

开发mac版本 方便一键优化，checklist对照



追加图片处理方法：相似度分析

https://www.jianshu.com/p/1a1f3a945380



## 总结

​	这边就只记录对项目帮助到比较大的几个点。其余的大家可以根据脑图一一尝试。相信所有的做下来会对项目有个很大的提升与帮助。

​	在设备越来越好，硬件条件越来越高的时代，可能微不足道的优化或提升并不能体现什么。但是希望每一个工程师都有一份工匠精神，把自己负责的项目也好，模块也好做到极致。这也是我们作为一个优秀程序员的追求吧。



### 写在最后：

Better Late Than Never ，愿我们都能跨出自己的第一步。

站在巨人的肩膀上，感谢前辈们的文章。



### 参考文章

[http://www.infoq.com/cn/articles/clang-plugin-ios-app-size-reducing](http://www.infoq.com/cn/articles/clang-plugin-ios-app-size-reducing)

[https://techblog.toutiao.com/2016/12/27/iphone/ ](https://techblog.toutiao.com/2016/12/27/iphone/ )				

[http://blog.cnbang.net/tech/2296/](http://blog.cnbang.net/tech/2296/)

[http://swift.gg/2016/01/07/app-thinning-appcoda/](http://swift.gg/2016/01/07/app-thinning-appcoda/)

[http://ios.jobbole.com/89725/](http://ios.jobbole.com/89725/)

[https://developer.apple.com/library/content/navigation/#section=Topics&topic=Performance](https://developer.apple.com/library/content/navigation/#section=Topics&topic=Performance)

[http://www.cnblogs.com/oc-bowen/p/7692689.html](http://www.cnblogs.com/oc-bowen/p/7692689.html)

[https://github.com/skyming/iOS-Performance-Optimization](https://github.com/skyming/iOS-Performance-Optimization)




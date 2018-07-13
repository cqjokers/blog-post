---
title: 记录线上程序导致服务器CPU过高的问题排除过程
date: 2018-07-13 17:36:30
tags:
- dump
- thread
- Analysis
categories:
- 错误分析
---
## 故障描述
昨天上午朋友告诉我之前我做的一个外包项目(抓取公共运输整合资讯流通服务平台上的数据)突然无法访问了，通过浏览器访问一直处于等待状态，于是我登录服务器查看CPU使用率发现，CPU占用达到80%以上，而java进程CPU占有率则是达到350%以上。
## 问题查找
查看当前服务器CPU占有率最简单的方法就是通过top命令查看![](https://upload-images.jianshu.io/upload_images/13023122-5480faab09f208ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)根据上图发现pid为27055的进程CPU占有率达到353.2%，于是通过`top -p 27055 -H`命令把该进程的所有线程都显示出来，然后发现几个占用CPU资源特别大的线程tid如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-8cd16dc2bf1d1e96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)根据上图所示发现有4个线程占用CPU非常高
<!--more-->
## 分析线程堆栈
首先通过`jstack 27055 > java-stack.log`命令将堆栈信息导出到日志文件里，然后将之前看到的线程ID转为16进制在日志文件里进行查找发现这4个线程是垃圾回收线程，分析图如下图所示：
![](https://upload-images.jianshu.io/upload_images/13023122-90a35033582f4d69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)也可以将堆栈日志上传这上网上进行分析[Fast thread](http://heaphero.io)如果日志文件太大最好还是使用MAT工具进行分析。
由于之前写的几个并没有出现这样的问题，是最近新添的几个抓取网址才导致出现CPU飙高，通过fastThread进行分析发现有几个线程执行的代码是一样的，但是按照正常流程该段代码只会出现在一个线程，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-c463bf338c18dc4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)上图标红的这几个线程都是执行最近新添的代码，并且发现都卡在了`startHandshake`这里，在网上搜索发现也有其它小伙伴遇见这个问题。由于抓取的数据量非常大所以执行的时间会比较久再加上自己手贱在定时执行线程中又包了一个子线程，所以导致了上面出现的几个线程同时执行同一段代码，通过浏览器不用https访问发现一样可以取到数据，然后果断的将后台上的网址换掉。通过优化代码和更换网址后从昨天到今天暂时未发现CPU占用较高的问题
## 总结
* 通过这次问题发现，线程一定不要乱用，凡是用到多线程的地方一定要谨慎思考
* 对于分析问题也会有不少的收获，一些基本的JVM命令需要去了解并在发现问题时知道怎么用。
* 对于java程序而言，CPU占用过高大多数是因为内存不够，而GC又回收不了导致的

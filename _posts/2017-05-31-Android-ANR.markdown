---
layout:     post
title:      "Android ANR分析"
subtitle:   "Android ANR"
navcolor:   "invert"
date:       2017-05-31
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android 
    - ANR 
---
# 概述

&emsp;&emsp;ANR（Application Not Responding）是Android开发常见的一种问题，尤其是在稳定性测试中非常容易出现。我们可以理解ANR是Android的一种机制，Android应用程序是一个进程，该进程中有个main Thread，这个主线程中通过Looper维护着一个MessageQueue，主要负责响应用户事件和处理UI相关的一些事情。那么，如果用户事件或UI事件响应的很慢，官方这样说：[Generally, 100 to 200ms is the threshold beyond which users will perceive slowness in an application](https://developer.android.com/training/articles/perf-anr.html#Reinforcing)。既然100到200毫秒用户就感觉慢了，如果在主线程处理耗时任务，用户体验是不是就非常差呢，所以，Android官方强烈要求处理耗时任务不能在主线程中。但是，有可能我们在写代码时将一个耗时操作（IO、网络请求、数据操作等）放到了UI线程，或者，多个线程产生了死锁，导致主线程无法继续执行等等，在这种情况下如果没有有效的机制提醒用户并回复进程状态，那么，用户可能认为是系统卡死或者定屏问题，从而怒砸手机。于是，在系统设计中便加入了ANR机制，会弹出一个dialog提示用户。

# 分析ANR需要的LOG

&emsp;&emsp;既然ANR发生了，那么我们就要检测是什么原因触发了ANR，需要的log日志如下：

event log:



logcat 日志



/data/anr/tarces.txt


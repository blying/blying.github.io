---
layout:     post
title:      Android app被系统kill的场景
subtitle:   ""
date:       2018-11-06 14:00:00
author:     "包丽颖"
header-img: "img/post-bg-android.jpg"
tags:
  - Android
---

### 何时发生

当我们的app被切到后台的时候，比如用户按下了home键或者切换到了别的应用，总之是我们的app不再和用户交互了，这个时候对于我们的app来说就是什么事情都可能发生的时候了，因为系统会认为你现在已经不是那么重要了，而和用户正在交互的app的优先级是最高的了，系统会想尽一切办法保证这些app的正常运行，如果这时这些app再申请更多的资源，如内存时，当目前的系统状况无法满足时，系统便会拿后台app开刀，也就是很粗鲁的杀掉整个app的进程，这时你也别指望onDestroy之类的callback会触发，你唯一能指望的就是在切后台的时候onPause、onStop和onSaveInstanceState之类的方法，如果你有状态需要保存那么应该在这些地方处理，不要寄希望于onDestroy，它会让你失望的，在这个地方用来释放资源还是ok的。由于系统要杀进程，那么紧接着的问题就是杀哪个进程，系统为此专门有个模块叫LML（low memory killer），详情可以参考下官方文档，见这里：[processes-and-threads](https://link.jianshu.com?t=http://developer.android.com/guide/components/processes-and-threads.html)。

### 再次切回app的行为

比如你在离开app的时候，已经打开了3个act，分别是A，B，C，C在最顶端，也就是任务栈顶，A是你的main activity，假设在后台期间被系统杀掉进程了，后面如果用户再次回来（通过recent tasks或者直接点击launcher里的app icon），这时展现在你眼前的将会是重建后的C activity，而不是正常情况下启动的A，系统同时也恢复了当初的任务栈，也就是说栈里的内容还是A，B，C，这时如果你按下了back键结束了C，那么系统又会帮我们重建B，A在B结束的时候也是一样的逻辑。这里需要注意一点就是，如果是用户自己杀掉了app，那么再次启动的时候回到的是A而不是C，只要记住是系统的错导致我们被杀的话，那么再次回到的话系统就有责任帮我们重建act。关于重建act的详情，可以参考官方文档[recreating-activity](https://link.jianshu.com?t=http://developer.android.com/training/basics/activity-lifecycle/recreating.html)。切回来之后虽然act是被重建了，但如果你代码里用了单例这样的东西来存一些变量的值，那么很不幸，这时所有单例中的字段全变成默认值了(0, false or null)，因为你想啊，进程都被杀死了啊，所有静态字段等等都没了。stackoverflow有这样的问题，比如这个[静态变量变成null了](https://link.jianshu.com?t=http://stackoverflow.com/questions/9541688/static-variable-null-when-returning-to-the-app?rq=1)。
 目前笔者在维护的代码里有类似的构造，线上也确实出现了些类似的问题，着实蛋疼啊，准备改掉这种单例datakeeper的写法，思路大体有以下几种：

1. 不改现有的单例datakeeper写法，但是增加永久存储支持，比如写到SP中，以后如果发现字段中没值了，那么就去SP中读一发；
2. 数据通过Intent传递，这样也能保证不丢失，因为系统重建act的时候，用的Intent和当时启动act时的Intent是一样的，所以如果你所需要的数据都是以这种方式传递的，那恭喜你，you are safe，你要做的只是从Intent中解析出来你需要的数据；
3. 在onSaveInstanceState中保存数据，在onCreate()/onRestoreInstanceState()中恢复数据；

### 如何模拟

由于系统后台杀进程具有一定的随机性，所以作为开发人员不可能去坐等这种情况发生，我们得有方法能很快速的复现，具体步骤如下：

![模拟后台杀进程的步骤](//upload-images.jianshu.io/upload_images/1737966-ce0935f1ef46c0e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/767/format/webp)



 转自：https://www.jianshu.com/p/0f9ecdd19752

 

 

 

 
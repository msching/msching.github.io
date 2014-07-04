---
layout: post
title: "奇怪的Graphics消耗（iOS 7.1.x）"
subtitle: "Strange Graphics Cost in iOS 7.1.x"
date: 2014-07-04 14:25:31 +0800
comments: true
statement: true
categories: [iOS]
---

近期有部分用户反馈我们的app在使用时有发热现象，在排查原因的过程中发现了一个奇怪的问题。在某个页面推出时`Instruments`的`Core Animation`会有帧数显示，数值在59~60，而此时并没有任何需要消耗Graphics的代码在跑，所有UI都是静止的。这个现象只会在iOS 7.1.x上出现，其他系统包括最新发布的iOS8 beta上均没有出现类似问题。

造成这个现象的页面比较特殊，其展现形式看上去虽然是一个UIViewController把一个UINavigationController用ModalView的形式push出来了，但实际上是把UINavigationController的.view直接add在了UIViewController的.view上，并用一个UIView动画展示页面。

为了弄清楚到底为什么会在UI静止的情况下产生Graphics消耗，我新建了一个空工程用相同的方法实现了一个推出页面的动画，然后profile却没有发现有问题。再次回头看app发现这个推出页面的controller是个UITabBarController，于是为测试工程加上tabBarController后再次profile，问题重现了，Core Animation在页面出现之后显示了帧数，页面隐藏之后帧数显示就消失了，从而可以推断是UITabBarController上的某个UI元素导致了这个问题。

接下来的步骤就是遍历UITabBarController的view上所有的subview并逐个隐藏来进行测试，幸运的是第一个就蒙对了，在我隐藏了UITabBarController的tabbar以后奇怪的帧数就不再出现了。这个现象很快让我想到了iOS 7以后苹果加入的模糊效果，这个效果在UINavigationBar、UITabBar、UIToolBar等UI控件上都有使用，下面把UITabBarController去掉，在view上直接add一个UIToolBar，发现问题同样会出现。至此基本确定是由于这个模糊效果造成的，至于为什么只会在7.1.x上出现这个问题。。可能只有苹果才知道了=。=。

测试程序代码点[这里](https://github.com/msching/TestGPU)

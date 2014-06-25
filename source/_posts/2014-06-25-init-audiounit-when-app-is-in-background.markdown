---
layout: post
title: "初始化AudioUnit的正确姿势"
subtitle: "init audiounit when app is in background"
date: 2014-06-25 13:56:27 +0800
comments: true
statement: true
categories: 
---
在使用AudioUnit的过程中发现当app在后台时调用`extern OSStatus AudioUnitInitialize(AudioUnit inUnit)`方法返回`561015905`错误码，解析成string后是`!pla`，google错误码后毫无收获，于是只能workaround。面对这个问题我的workaround是当出现初始化失败的情况下会在程序进入前台时再尝试调用`AudioUnitInitialize`方法来初始化AudioUnit。至此问题已经在一定程度上得到了解决，只要用户进入前台就可以正确初始化AudioUnit并且播放音乐。

今天在应对某个用户反馈时发现该用户在使用remoteControl过程中无法启动播放的情况正是因为后台init AudioUnit会失败导致程序无法如预期工作。于是灵光一闪，觉得在初始化AudioUnit之前先调用`AudioSessionInitialize`并setActive是否就可以解决问题。尝试之后发现果然可以...（之前都在AudioUnitInitialize成功后才去init audiosession）。
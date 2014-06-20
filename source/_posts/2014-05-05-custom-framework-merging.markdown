---
layout: post
title: "合并生成模拟器和真机通用的framework"
subtitle: "Custom framework merging"
date: 2014-05-05 17:58:00 +0800
comments: true
statement: true
categories: [iOS]
---

在使用[iOS-Universal-Framework](https://github.com/kstenerud/iOS-Universal-Framework)制作framework的过程中经常会遇到编译出来的framework只能被真机使用或者只能被模拟器使用的情况。

造成这个问题的原因是由于在编译时选择的目标设备不同的情况下编译出来framework体系结构不同，选择真机进行编辑时会编译产生`armv7`、`armv7s`、`arm64`下的库文件，而选择模拟器会产生`i386`、`x86_64`下的库文件。
具体查看的方法可以执行下列命令：

```
$ lipo -info /Debug-iphoneos/Someframework.framwork/Someframework
# Architectures in the fat file: Someframework are: armv7 arm64 

$ lipo -info /Debug-iphonesimulator/Someframework.framwork/Someframework
# Architectures in the fat file: Someframework are: i386 x86_64 

```


要同时对模拟器和真机进行支持，只要对两个编译出来的framework进行合并就可以了。

执行如下命令就可以进行合并

```
$ lipo –create /Debug-iphoneos/Someframework.framwork/Someframework Debug-iphonesimulator/Someframework.framwork/Someframework –output Someframework
```

完成后再查看framework的版本

```
$ lipo -info Someframework
# Architectures in the fat file: Someframework are: armv7 arm64 i386 x86_64
```
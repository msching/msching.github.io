---
layout: post
title: "iOS音频播放 (八)：NowPlayingCenter和RemoteControl"
subtitle: "Audio playback in iOS (Part 8) : NowPlayingCenter & RemoteControl"
date: 2014-11-06 13:26:28 +0800
comments: true
statement: true
categories: [iOS,Audio,iOS Audio]
---

距离[上一篇](/blog/2014/09/07/audio-in-ios-7/)博文发布已经有一个月多的时间了，在这其间我一直忙于筹办婚礼以至于这篇博文一直拖到了现在。

在之前[一到六篇](/blog/categories/ios-audio/)中我对iOS下的音频播放流程进行了阐述，在[第七篇](/blog/2014/09/07/audio-in-ios-7/)中介绍了如何播放iPod Lib中的歌曲，至此有关音频播放的话题就已经完结了，在这篇里我将会讲到的`NowPlayingCenter`和`RemoteControl`这两个玩意本身和整个播放流程并没有什么关系，但它们可以让音频播放在iOS系统上获得更加好的用户体验。

<!--more-->

----

# NowPlayingCenter

`NowPlayingCenter`能够显示当前正在播放的歌曲信息，它可以控制的范围包括：

* 锁频界面上所显示的歌曲播放信息和图片
* iOS7之后控制中心上显示的歌曲播放信息
* iOS7之前双击home键后出现的进程中向左滑动出现的歌曲播放信息
* AppleTV，AirPlay中显示的播放信息
* 车载系统中显示的播放信息

这些信息的显示都由`MPNowPlayingInfoCenter`类来控制，这个类的定义非常简单：

```objc
MP_EXTERN_CLASS_AVAILABLE(5_0) @interface MPNowPlayingInfoCenter : NSObject

// Returns the default now playing info center.
// The default center holds now playing info about the current application.
+ (MPNowPlayingInfoCenter *)defaultCenter;

// The current now playing info for the center.
// Setting the info to nil will clear it.
@property (copy) NSDictionary *nowPlayingInfo;

@end
```

使用也同样简单，首先`#import <MediaPlayer/MPNowPlayingInfoCenter.h>`然后调用`MPNowPlayingInfoCenter`的单例方法获取实例，再把需要显示的信息组织成Dictionary并赋值给`nowPlayingInfo`属性就完成了。

`nowPlayingInfo`中一些常用属性被定义在`<MediaPlayer/MPMediaItem.h>`中

```objc
MPMediaItemPropertyAlbumTitle				 //NSString
MPMediaItemPropertyAlbumTrackCount			//NSNumber of NSUInteger
MPMediaItemPropertyAlbumTrackNumber		//NSNumber of NSUInteger
MPMediaItemPropertyArtist					//NSString
MPMediaItemPropertyArtwork					//MPMediaItemArtwork
MPMediaItemPropertyComposer				//NSString
MPMediaItemPropertyDiscCount				//NSNumber of NSUInteger
MPMediaItemPropertyDiscNumber				//NSNumber of NSUInteger
MPMediaItemPropertyGenre					//NSString
MPMediaItemPropertyPersistentID			//NSNumber of uint64_t
MPMediaItemPropertyPlaybackDuration		//NSNumber of NSTimeInterval
MPMediaItemPropertyTitle					//NSString
```

上面这些属性大多比较浅显易懂，基本上按照字面上的意思去理解就可以了，需要稍微解释以下的是`MPMediaItemPropertyArtwork`。这个属性表示的是锁屏界面或者AirPlay中显示的歌曲封面图，`MPMediaItemArtwork`类可以由`UIImage`类进行初始化。

```objc
MP_EXTERN_CLASS_AVAILABLE(3_0) @interface MPMediaItemArtwork : NSObject

// Initializes an MPMediaItemArtwork instance with the given full-size image.
// The crop rect of the image is assumed to be equal to the bounds of the 
// image as defined by the image's size in points, i.e. tightly cropped.
- (instancetype)initWithImage:(UIImage *)image NS_DESIGNATED_INITIALIZER NS_AVAILABLE_IOS(5_0);

// Returns the artwork image for an item at a given size (in points).
- (UIImage *)imageWithSize:(CGSize)size;

@property (nonatomic, readonly) CGRect bounds; // The bounds of the full size image (in points).
@property (nonatomic, readonly) CGRect imageCropRect; // The actual content area of the artwork, in the bounds of the full size image (in points).

@end
```

另外一些附加属性被定义在`<MediaPlayer/MPNowPlayingInfoCenter.h>`中

```objc
// The elapsed time of the now playing item, in seconds.
// Note the elapsed time will be automatically extrapolated from the previously 
// provided elapsed time and playback rate, so updating this property frequently
// is not required (or recommended.)
MP_EXTERN NSString *const MPNowPlayingInfoPropertyElapsedPlaybackTime NS_AVAILABLE_IOS(5_0); // NSNumber (double)

// The playback rate of the now playing item, with 1.0 representing normal 
// playback. For example, 2.0 would represent playback at twice the normal rate.
// If not specified, assumed to be 1.0.
MP_EXTERN NSString *const MPNowPlayingInfoPropertyPlaybackRate NS_AVAILABLE_IOS(5_0); // NSNumber (double)

// The "default" playback rate of the now playing item. You should set this
// property if your app is playing a media item at a rate other than 1.0 in a
// default playback state. e.g., if you are playing back content at a rate of
// 2.0 and your playback state is not fast-forwarding, then the default
// playback rate should also be 2.0. Conversely, if you are playing back content
// at a normal rate (1.0) but the user is fast-forwarding your content at a rate
// greater than 1.0, then the default playback rate should be set to 1.0.
MP_EXTERN NSString *const MPNowPlayingInfoPropertyDefaultPlaybackRate NS_AVAILABLE_IOS(8_0); // NSNumber (double)

// The index of the now playing item in the application's playback queue.
// Note that the queue uses zero-based indexing, so the index of the first item 
// would be 0 if the item should be displayed as "item 1 of 10".
MP_EXTERN NSString *const MPNowPlayingInfoPropertyPlaybackQueueIndex NS_AVAILABLE_IOS(5_0); // NSNumber (NSUInteger)

// The total number of items in the application's playback queue.
MP_EXTERN NSString *const MPNowPlayingInfoPropertyPlaybackQueueCount NS_AVAILABLE_IOS(5_0); // NSNumber (NSUInteger)

// The chapter currently being played. Note that this is zero-based.
MP_EXTERN NSString *const MPNowPlayingInfoPropertyChapterNumber NS_AVAILABLE_IOS(5_0); // NSNumber (NSUInteger)

// The total number of chapters in the now playing item.
MP_EXTERN NSString *const MPNowPlayingInfoPropertyChapterCount NS_AVAILABLE_IOS(5_0); // NSNumber (NSUInteger)
```
其中常用的是`MPNowPlayingInfoPropertyElapsedPlaybackTime`和`MPNowPlayingInfoPropertyPlaybackRate`：

* `MPNowPlayingInfoPropertyElapsedPlaybackTime`表示已经播放的时间，用这个属性可以让`NowPlayingCenter`显示播放进度；
* `MPNowPlayingInfoPropertyPlaybackRate`表示播放速率。通常情况下播放速率为1.0，即真是时间的1秒对应播放时间中的1秒；

这里需要解释的是，`NowPlayingCenter`中的进度刷新并不是由app不停的更新`nowPlayingInfo`来做的，而是根据app传入的`ElapsedPlaybackTime`和`PlaybackRate`进行自动刷新。例如传入ElapsedPlaybackTime=120s，PlaybackRate=1.0，那么`NowPlayingCenter`会显示2:00并且在接下来的时间中每一秒把进度加1秒并刷新显示。如果需要暂停进度，传入PlaybackRate=0.0即可。

所以每次播放暂停和继续都需要更新`NowPlayingCenter`并正确设置`ElapsedPlaybackTime`和`PlaybackRate`否则`NowPlayingCenter`中的播放进度无法正常显示。

### NowPlayingCenter的刷新时机

频繁的刷新`NowPlayingCenter`并不可取，特别是在有Artwork的情况下。所以需要在合适的时候进行刷新。

依照我自己的经验下面几个情况下刷新`NowPlayingCenter`比较合适：

* 当前播放歌曲进度被拖动时
* 当前播放的歌曲变化时
* 播放暂停或者恢复时
* 当前播放歌曲的信息发生变化时（例如Artwork，duration等）

在刷新时可以适当的通过判断app是否active来决定是否必须刷新以减少刷新次数。

### MPMediaItemPropertyArtwork

这是一个非常有用的属性，我们可以利用歌曲的封面图来合成一些图片借此达到美化锁屏界面或者显示锁屏歌词。

----

# RemoteControl

`RemoteComtrol`可以用来在不打开app的情况下控制app中的多媒体播放行为，涉及的内容主要包括：

* 锁屏界面双击Home键后出现的播放操作区域
* iOS7之后控制中心的播放操作区域
* iOS7之前双击home键后出现的进程中向左滑动出现的播放操作区域
* AppleTV，AirPlay中显示的播放操作区域
* 耳机线控
* 车载系统的设置
* ...

### iOS 7.1之后如何处理RemoteControl

iOS 7.1之后Apple提供了`MPRemoteCommandCenter`类来统一管理RemoteControl，并且在MPRemoteCommandCenter定义了比之前更多的RemoteControl操作，我们除了可以操作播放还可以进行一些其他的交互操作，比如收藏、评分等等。

使用过程中我们只要为其中需要用到的command添加一个方法就可以实现RemoteControl。

例如对于播放和暂停来说只要进行如下设置就可以了：

```objc
[[UIApplication sharedApplication] beginReceivingRemoteControlEvents]

//返回值根据需要返回，一般情况下返回MPRemoteCommandHandlerStatusSuccess即可
[[MPRemoteCommandCenter sharedCommandCenter].playCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent * _Nonnull event) {
    // play
    return MPRemoteCommandHandlerStatusSuccess;
}];
    
[[MPRemoteCommandCenter sharedCommandCenter].pauseCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent * _Nonnull event) {
    // pause
    return MPRemoteCommandHandlerStatusSuccess;
}];
    
[[MPRemoteCommandCenter sharedCommandCenter].togglePlayPauseCommand addTargetWithHandler:^MPRemoteCommandHandlerStatus(MPRemoteCommandEvent * _Nonnull event) {
    // play or pause
    return MPRemoteCommandHandlerStatusSuccess;
}];
```
或者

```objc
[[UIApplication sharedApplication] beginReceivingRemoteControlEvents]

[[MPRemoteCommandCenter sharedCommandCenter].playCommand addTarget:self action:@selector(play)];
[[MPRemoteCommandCenter sharedCommandCenter].pauseCommand addTarget:self action:@selector(pause)];
[[MPRemoteCommandCenter sharedCommandCenter].togglePlayPauseCommand addTarget:self action:@selector(playOrPause)];
```
如果你只是临时使用的话记得在使用完成后removeTarget并调用endReceivingRemoteControlEvents，removeTarget的方法根据addTarget的方式不同而不同：

1. 如果是`-addTargetWithHandler:`添加的，需要把返回的target记下来后调用`-removeTarget:`移除
2. 如果是`-addTarget:action:`添加的，使用`-removeTarget:action:`移除

对于播放控制之外的收藏、评分等操作需要注意command的相关属性，在必要的时候使用这些属性以达到需要的效果。例如`MPFeedbackCommand`的`active`属性。

~~另外注意`MPChangePlaybackPositionCommand`是个坑，目前（iOS 9.x）看来还不能得到应用，无法通过公开的接口使用，必须调用私有接口后才能让锁屏界面和控制中心的进度条能够拖动，不知道iOS 10会不会开放。~~

SDK8编译在iOS 10.1之后已经可以使用`MPChangePlaybackPositionCommand`了，而且不知道为什么iOS 8.4也是可以的，其他iOS版本依然不能应用。

SDK8中还出现了两个新的Command，`MPChangeRepeatModeCommand`和`MPChangeShuffleModeCommand`，虽然SDK8中才出现却没有标记版本，说明是iOS7.1就开始有了？这个我不确信。这两个Command在锁屏界面和控制中心中没有UI对应，根据台湾[KKBOX公司的实践](https://medium.com/@zonble/the-issue-about-using-mpchangerepeatmodecommand-811840a39e93#.153icy7qb)，这两个Command是被用在CarPlay中，但使用之后出现了一些兼容性的问题，故不推荐大家使用。

PS：macOS 10.12.1之后macOS也有RemoteControl和NowPlayingInfoCenter了，可以在2016款MBP的TouchBar上进行歌曲信息的展示和操作，其中RemoteControl可以接受耳机线控操作和键盘多媒体键操作（之前是要通过[SPMediaKeyTap](https://github.com/nevyn/SPMediaKeyTap)和[DDHidAppleMikey](https://github.com/Daij-Djan/DDHidLib/blob/master/lib/DDHidAppleMikey.m)实现的），耳机线控操作也不再唤起iTunes了，不过似乎只能在非沙箱环境下有效。

### iOS 7.1之前如何处理RemoteComtrol

#### 在何处处理RemoteComtrol
根据[官方文档](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Remote-ControlEvents/Remote-ControlEvents.html)的描述：

`If your app plays audio or video content, you might want it to respond to remote control events that originate from either transport controls or external accessories. (External accessories must conform to Apple-provided specifications.) iOS converts commands into UIEvent objects and delivers the events to an app. The app sends them to the first responder and, if the first responder doesn’t handle them, they travel up the responder chain.`

当`RemoteComtrol`事件产生时，iOS会以`UIEvent`的形式发送给app，app会首先转发到first responder，如果first responder不处理这个事件的话那么事件就会沿着responder chain继续转发。关于responder chain的相关内容可以查看[这里](https://developer.apple.com/library/IOs/documentation/General/Conceptual/Devpedia-CocoaApp/Responder.html)。

从responder chain文档看来如果之前的所有responder全部不响应`RemoteComtrol`事件的话，最终事件会被转发给Application（如图）。所以我们知道作为responder chain的最末端，在`UIApplication`中实现`RemoteComtrol`的处理是最为合理的，而并非在UIWindow中或者AppDelegate中。

![](/images/iOS-audio/responder chain.jpg)

#### 实现自己的UIApplication

首先新建一个`UIApplication`的子类

```objc
#import <UIKit/UIKit.h>

@interface MyApplication : UIApplication

@end
```

然后找到工程中的`main.m`，可以看到代码如下：

```c
int main(int argc, char * argv[]) 
{
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
在main中调用了`UIApplicationMain`方法

```objc
// If nil is specified for principalClassName, the value for NSPrincipalClass from the Info.plist is used. If there is no
// NSPrincipalClass key specified, the UIApplication class is used. The delegate class will be instantiated using init.
UIKIT_EXTERN int UIApplicationMain(int argc, char *argv[], NSString *principalClassName, NSString *delegateClassName);
```

我们需要做的就是给`UIApplicationMain`方法的第三个参数传入我们的application类名，如下：

```c
#import <UIKit/UIKit.h>
#import "AppDelegate.h"
#import "MyApplication.h"

int main(int argc, char * argv[])
{
    @autoreleasepool {
        return UIApplicationMain(argc, argv, NSStringFromClass([MyApplication class]), NSStringFromClass([AppDelegate class]));
    }
}
```

这样就成功实现了自己的`UIApplication`.

#### 处理RemoteComtrol
了解了应该在何处处理`RemoteComtrol`事件之后，再来看下[官方文档](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Remote-ControlEvents/Remote-ControlEvents.html)中描述的三个必要条件：

* 接受者必须能够成为first responder
* 必须显示地声明接收`RemoteComtrol`事件
* 你的app必须是`Now Playing`app

对于第一条就是要在自己的`UIApplication`中实现`canBecomeFirstResponder`方法:

```objc
#import "MyApplication.h"

@implementation MyApplication

- (BOOL)canBecomeFirstResponder
{
    return YES;
}

@end
```
第二条是要求显示地调用`[[UIApplication sharedApplication] beginReceivingRemoteControlEvents]`，调用的实际一般是在播放开始时；

第三条就是要求占据NowPlayingCenter，这个之前已经提到过了。

满足三个条件后可以在`UIApplication`中实现处理`RemoteComtrol`事件的方法，根据不同的事件实现不同的操作即可。

```objc
#import "MyApplication.h"

@implementation MyApplication

- (BOOL)canBecomeFirstResponder
{
    return YES;
}

- (void)remoteControlReceivedWithEvent:(UIEvent *)event
{
    switch (event.subtype)
    {
        case UIEventSubtypeRemoteControlPlay:
        	  //play
            break;
        case UIEventSubtypeRemoteControlPause:
        	  //pause
            break;
        case UIEventSubtypeRemoteControlStop:
        	  //stop
            break;
        default:
            break;
    }
}

@end
```

关于iOS 7之前的RemoteControl处理，git上有一个关于remotecontrol的小工程供大家参考[ios-audio-remote-control](https://github.com/MosheBerman/ios-audio-remote-control)

<div class="github-card" data-github="MosheBerman/ios-audio-remote-control" data-width="400" data-height="" data-theme="default"></div>
<script src="//cdn.jsdelivr.net/github-cards/latest/widget.js"></script>

----

# 后记

到本篇为止iOS的音频播放话题基本上算是完结了。接下来我会在空余时间去研究一下iOS 8中新加入的`AVAudioEngine`，其功能涵盖播放、录音、混音、音效处理，看上去十分强大，从接口的定义上看像是对`AudioUnit`的高层封装，当研究有了一定的成果之后也会以博文的形式分享出来。

----

# 参考资料

[MPNowPlayingInfoCenter](https://encrypted.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&ved=0CC4QFjAA&url=https%3A%2F%2Fdeveloper.apple.com%2FLibrary%2Fios%2Fdocumentation%2FMediaPlayer%2FReference%2FMPNowPlayingInfoCenter_Class%2Findex.html&ei=8bBhVO_RF4HKmwXBiILIDA&usg=AFQjCNFOziF2zKft-wGQ3ew_cHy7Ivxrvg)

[Remote Control Events](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Remote-ControlEvents/Remote-ControlEvents.html)

[Cocoa Responder Chain](https://developer.apple.com/library/IOs/documentation/General/Conceptual/Devpedia-CocoaApp/Responder.html)

[ios-audio-remote-control](https://github.com/MosheBerman/ios-audio-remote-control)
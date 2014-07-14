---
layout: post
title: "iOS音频播放 (二)：AudioSession"
subtitle: "Audio in iOS (Part 2) : AudioSession"
date: 2014-07-08 13:58:27 +0800
comments: true
statement: true
categories: [iOS,Audio]
---


#前言

本篇为《iOS音频播放》系列的第二篇。

在实施[前一篇](blog/2014/07/07/audio-in-ios/)中所述的7个步骤步之前还必须面对一个麻烦的问题，AudioSession。

----

#AudioSession简介

AudioSession这个玩意的主要功能包括以下几点（图片来自[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40007875)）：

1. 确定你的app如何使用音频（是播放？还是录音？）
2. 为你的app选择合适的输入输出设备（比如输入用的麦克风，输出是耳机、手机功放或者airplay）
3. 协调你的app的音频播放和系统以及其他app行为（例如有电话时需要打断，电话结束时需要恢复，按下静音按钮时是否歌曲也要静音等）

![AudioSession](/images/iOS-audio/audiosession.jpg)

AudioSession相关的类有两个：

1. `AudioToolBox`中的`AudioSession`
2. `AVFoundation`中的`AVAudioSession`

其中AudioSession在SDK 7中已经被标注为depracated，而AVAudioSession这个类虽然iOS 3开始就已经存在了，但其中很多方法和变量都是在iOS 6以后甚至是iOS 7才有的。所以各位可以依照以下标准选择：

* 如果最低版本支持iOS 5，请使用`AudioSession`
* 如果最低版本支持iOS 6及以上，可以考虑使用`AVAudioSession`
* 如果最低版本支持iOS 7及以上，请使用`AVAudioSession`

下面以`AudioSession`类为例来讲述AudioSession相关功能的使用（很不幸我需要支持iOS 5。。T-T，使用`AVAudioSession`的同学可以在其头文件中寻找对应的方法使用即可，需要注意的点我会加以说明）.

**注意：在使用AVAudioPlayer/AVPlayer时可以不用关心AudioSession的相关问题，Apple已经把AudioSession的处理过程封装了，但音乐打断后的响应还是要做的（比如打断后音乐暂停了UI状态也要变化，这个应该通过KVO就可以搞定了吧。。我没试过瞎猜的>_<）。**

----

#初始化AudioSession

使用`AudioSession`类首先需要调用初始化方法：

```objc
extern OSStatus AudioSessionInitialize(CFRunLoopRef inRunLoop, 
										 CFStringRef inRunLoopMode, 
										 AudioSessionInterruptionListener inInterruptionListener,
 										 void *inClientData);
```

前两个参数一般填`NULL`表示AudioSession运行在主线程上（但并不代表音频的相关处理运行在主线程上，只是AudioSession），第三个参数需要传入一个一个`AudioSessionInterruptionListener`类型的方法，作为AudioSession被打断时的回调，第四个参数则是代表打断回调时需要附带的对象（即回到方法中的inClientData，如下所示，可以理解为UIView animation中的context）。

```objc
typedef void (*AudioSessionInterruptionListener)(void * inClientData, UInt32 inInterruptionState);
```

这才刚开始，坑就来了。这里会有两个问题：

第一，AudioSessionInitialize可以被多次执行，但`AudioSessionInterruptionListener`只能被设置一次，这就意味着这个打断回调方法是一个静态方法，一旦初始化成功以后所有的打断都会回调到这个方法，即便下一次再次调用AudioSessionInitialize并且把另一个静态方法作为参数传入，当打断到来时还是会回调到第一次设置的方法上。

这种场景并不少见，例如你的app既需要播放歌曲又需要录音，当然你不可能知道用户会先调用哪个功能，所以你必须在播放和录音的模块中都调用AudioSessionInitialize注册打断方法，但最终打断回调只会作用在先注册的那个模块中，很蛋疼吧。。。所以对于AudioSession的使用最好的方法是生成一个类单独进行管理，统一接收打断回调并发送自定义的打断通知，在需要用到AudioSession的模块中接收通知并做相应的操作。

Apple也察觉到了这一点，所以在AVAudioSession中首先取消了Initialize方法，改为了单例方法`sharedInstance`。在iOS 6上所有的打断都需要通过设置`id<AVAudioSessionDelegate> delegate`并实现回调方法来实现，这同样会有上述的问题，所以在iOS6下仍然需要一个单独管理AudioSession的类存在。在iOS 7以后Apple终于把打断改成了通知的形式。。这下科学了。

第二，AudioSessionInitialize方法的第四个参数inClientData，也就是回调方法的第一个参数。上面已经说了打断回调是一个静态方法，而这个参数的目的是为了能让回调时拿到context（上下文信息），所以这个inClientData需要是一个有足够长生命周期的对象（当然前提是你确实需要用到这个参数），如果这个对象被dealloc了，那么回调时拿到的inClientData会是一个野指针。就这一点来说构造一个单独管理AudioSession的类也是有必要的，因为这个类的生命周期和AudioSession一样长，我们可以把context保存在这个类中。

----

#监听RouteChange事件

如果想要实现类似于“拔掉耳机就把歌曲暂停”的功能就需要监听RouteChange事件：

```objc
extern OSStatus AudioSessionAddPropertyListener(AudioSessionPropertyID inID,
												  AudioSessionPropertyListener inProc,
												  void *inClientData);
												  
typedef void (*AudioSessionPropertyListener)(void * inClientData,
											   AudioSessionPropertyID inID,
											   UInt32 inDataSize,
											   const void * inData);
```

调用上述方法，AudioSessionPropertyID参数传`kAudioSessionProperty_AudioRouteChange`，AudioSessionPropertyListener参数传对应的回调方法。inClientData参数同AudioSessionInitialize方法。

同样作为静态回调方法还是需要统一管理，接到回调时可以把第一个参数inData转换成`CFDictionaryRef`并从中获取kAudioSession_AudioRouteChangeKey_Reason键值对应的value（应该是一个CFNumberRef），得到这些信息后就可以发送自定义通知给其他模块进行相应操作(例如`kAudioSessionRouteChangeReason_OldDeviceUnavailable`就可以用来做“拔掉耳机就把歌曲暂停”)。

```objc
//AudioSession的AudioRouteChangeReason枚举
enum {
		kAudioSessionRouteChangeReason_Unknown = 0,
		kAudioSessionRouteChangeReason_NewDeviceAvailable = 1,
		kAudioSessionRouteChangeReason_OldDeviceUnavailable = 2,
		kAudioSessionRouteChangeReason_CategoryChange = 3,
		kAudioSessionRouteChangeReason_Override = 4,
		kAudioSessionRouteChangeReason_WakeFromSleep = 6,
		kAudioSessionRouteChangeReason_NoSuitableRouteForCategory = 7,
		kAudioSessionRouteChangeReason_RouteConfigurationChange = 8
	};

```

```objc
//AVAudioSession的AudioRouteChangeReason枚举
typedef NS_ENUM(NSUInteger, AVAudioSessionRouteChangeReason)
{
	AVAudioSessionRouteChangeReasonUnknown = 0,
	AVAudioSessionRouteChangeReasonNewDeviceAvailable = 1,
	AVAudioSessionRouteChangeReasonOldDeviceUnavailable = 2,
	AVAudioSessionRouteChangeReasonCategoryChange = 3,
	AVAudioSessionRouteChangeReasonOverride = 4,
	AVAudioSessionRouteChangeReasonWakeFromSleep = 6,
	AVAudioSessionRouteChangeReasonNoSuitableRouteForCategory = 7,
	AVAudioSessionRouteChangeReasonRouteConfigurationChange NS_ENUM_AVAILABLE_IOS(7_0) = 8
}
```

**注意：iOS 6下如果使用了`AVAudioSession`由于`AVAudioSessionDelegate`中并没有定义相关的方法，还是需要用这个方法来实现监听。iOS 7下直接监听AVAudioSession的通知就可以了。**

----

这里附带两个方法的实现，都是基于`AudioSession`类的（使用`AVAudioSession`的同学帮不到你们啦）。

1、判断是否插了耳机：

```objc
+ (BOOL)usingHeadset
{
#if TARGET_IPHONE_SIMULATOR
    return NO;
#endif
    
    CFStringRef route;
    UInt32 propertySize = sizeof(CFStringRef);
    AudioSessionGetProperty(kAudioSessionProperty_AudioRoute, &propertySize, &route);
    
    BOOL hasHeadset = NO;
    if((route == NULL) || (CFStringGetLength(route) == 0))
    {
        // Silent Mode
    }
    else
    {
        /* Known values of route:
         * "Headset"
         * "Headphone"
         * "Speaker"
         * "SpeakerAndMicrophone"
         * "HeadphonesAndMicrophone"
         * "HeadsetInOut"
         * "ReceiverAndMicrophone"
         * "Lineout"
         */
        NSString* routeStr = (__bridge NSString*)route;
        NSRange headphoneRange = [routeStr rangeOfString : @"Headphone"];
        NSRange headsetRange = [routeStr rangeOfString : @"Headset"];
        
        if (headphoneRange.location != NSNotFound)
        {
            hasHeadset = YES;
        }
        else if(headsetRange.location != NSNotFound)
        {
            hasHeadset = YES;
        }
    }
    
    if (route)
    {
        CFRelease(route);
    }
    
    return hasHeadset;
}

```

2、判断是否开了Airplay(来自[StackOverflow](http://stackoverflow.com/questions/13044894/get-name-of-airplay-device-using-avplayer))：

```objc
+ (BOOL)isAirplayActived
{
    CFDictionaryRef currentRouteDescriptionDictionary = nil;
    UInt32 dataSize = sizeof(currentRouteDescriptionDictionary);
    AudioSessionGetProperty(kAudioSessionProperty_AudioRouteDescription, &dataSize, &currentRouteDescriptionDictionary);
    
    BOOL airplayActived = NO;
    if (currentRouteDescriptionDictionary)
    {
        CFArrayRef outputs = CFDictionaryGetValue(currentRouteDescriptionDictionary, kAudioSession_AudioRouteKey_Outputs);
        if(outputs != NULL && CFArrayGetCount(outputs) > 0)
        {
            CFDictionaryRef currentOutput = CFArrayGetValueAtIndex(outputs, 0);
            //Get the output type (will show airplay / hdmi etc
            CFStringRef outputType = CFDictionaryGetValue(currentOutput, kAudioSession_AudioRouteKey_Type);
            
            airplayActived = (CFStringCompare(outputType, kAudioSessionOutputRoute_AirPlay, 0) == kCFCompareEqualTo);
        }
        CFRelease(currentRouteDescriptionDictionary);                
    }
    return airplayActived;
}


```

----

#设置类别

下一步要设置AudioSession的Category，使用`AudioSession`时调用下面的接口

```objc
extern OSStatus AudioSessionSetProperty(AudioSessionPropertyID inID,
										  UInt32 inDataSize,
										  const void *inData);
```

如果我需要的功能是播放，执行如下代码

```objc
UInt32 sessionCategory = kAudioSessionCategory_MediaPlayback;
AudioSessionSetProperty (kAudioSessionProperty_AudioCategory,
						   sizeof(sessionCategory),
						   &sessionCategory);
```

使用`AVAudioSession`时调用下面的接口

```objc
/* set session category */
- (BOOL)setCategory:(NSString *)category error:(NSError **)outError;
/* set session category with options */
- (BOOL)setCategory:(NSString *)category withOptions: (AVAudioSessionCategoryOptions)options error:(NSError **)outError NS_AVAILABLE_IOS(6_0);
```

至于Category的类型在[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/AudioSessionBasics/AudioSessionBasics.html#//apple_ref/doc/uid/TP40007875-CH3-SW1)中都有介绍，我这里也只罗列一下具体就不赘述了，各位在使用时可以依照自己需要的功能设置Category。

```objc
//AudioSession的AudioSessionCategory枚举
enum {
		kAudioSessionCategory_AmbientSound               = 'ambi',
		kAudioSessionCategory_SoloAmbientSound           = 'solo',
		kAudioSessionCategory_MediaPlayback              = 'medi',
		kAudioSessionCategory_RecordAudio                = 'reca',
		kAudioSessionCategory_PlayAndRecord              = 'plar',
		kAudioSessionCategory_AudioProcessing            = 'proc'
	};
```


```objc
//AudioSession的AudioSessionCategory字符串
/*  Use this category for background sounds such as rain, car engine noise, etc.  
 Mixes with other music. */
AVF_EXPORT NSString *const AVAudioSessionCategoryAmbient;
	
/*  Use this category for background sounds.  Other music will stop playing. */
AVF_EXPORT NSString *const AVAudioSessionCategorySoloAmbient;

/* Use this category for music tracks.*/
AVF_EXPORT NSString *const AVAudioSessionCategoryPlayback;

/*  Use this category when recording audio. */
AVF_EXPORT NSString *const AVAudioSessionCategoryRecord;

/*  Use this category when recording and playing back audio. */
AVF_EXPORT NSString *const AVAudioSessionCategoryPlayAndRecord;

/*  Use this category when using a hardware codec or signal processor while
 not playing or recording audio. */
AVF_EXPORT NSString *const AVAudioSessionCategoryAudioProcessing;

```

----

#启用

有了Category就可以启动AudioSession了，启动方法：

```objc
//AudioSession的启动方法
extern OSStatus AudioSessionSetActive(Boolean active);
extern OSStatus AudioSessionSetActiveWithFlags(Boolean active, UInt32 inFlags);

//AVAudioSession的启动方法
- (BOOL)setActive:(BOOL)active error:(NSError **)outError;
- (BOOL)setActive:(BOOL)active withOptions:(AVAudioSessionSetActiveOptions)options error:(NSError **)outError NS_AVAILABLE_IOS(6_0);
```
启动方法调用后必须要判断是否启动成功，启动不成功的情况经常存在，例如一个前台的app正在播放，你的app正在后台想要启动AudioSession那就会返回失败。

一般情况下我们在启动和停止AudioSession调用第一个方法就可以了。但如果你正在做一个即时语音通讯app的话（类似于微信、易信）就需要注意在deactive AudioSession的时候需要使用第二个方法，inFlags参数传入`kAudioSessionSetActiveFlag_NotifyOthersOnDeactivation`（`AVAudioSession`给options参数传入`AVAudioSessionSetActiveOptionNotifyOthersOnDeactivation`）。当你的app deactive自己的AudioSession时系统会通知上一个被打断播放app打断结束（就是上面说到的打断回调），如果你的app在deactive时传入了NotifyOthersOnDeactivation参数，那么其他app在接到打断结束回调时会多得到一个参数`kAudioSessionInterruptionType_ShouldResume`否则就是ShouldNotResume（`AVAudioSessionInterruptionOptionShouldResume`），根据参数的值可以决定是否继续播放。

大概流程是这样的：

1. 一个音乐软件A正在播放；
2. 用户打开你的软件播放对话语音，AudioSession active；
3. 音乐软件A音乐被打断并收到InterruptBegin事件；
4. 对话语音播放结束，AudioSession deactive并且传入NotifyOthersOnDeactivation参数；
5. 音乐软件A收到InterruptEnd事件，查看Resume参数，如果是ShouldResume控制音频继续播放，如果是ShouldNotResume就维持打断状态；

[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/ConfiguringanAudioSession/ConfiguringanAudioSession.html#//apple_ref/doc/uid/TP40007875-CH2-SW1)中有一张很形象的图来阐述这个现象：


![](/images/iOS-audio/audiosession-active.jpg)

然而现在某些语音通讯软件和某些音乐软件却无视了`NotifyOthersOnDeactivation`和`ShouldResume`的正确用法，导致我们经常接到这样的用户反馈：

	你们的app在使用xx语音软件听了一段话后就不会继续播放了，但xx音乐软件可以继续播放啊。

好吧，上面只是吐槽一下。请无视我吧。

**2014.7.14补充：**

发现即使之前已经调用过`AudioSessionInitialize`方法，在某些情况下被打断之后可能出现AudioSession失效的情况，需要再次调用`AudioSessionInitialize`方法来重新生成AudioSession。否则调用`AudioSessionSetActive`会返回560557673（其他AudioSession方法也雷同，所有方法调用前必须首先初始化AudioSession），转换成string后为"!ini"即`kAudioSessionNotInitialized`，这个情况在iOS 5.1.x上尤其频繁，iOS 7.x也偶有发生具体的原因还不知晓。

所以每次在调用`AudioSessionSetActive`时应该判断一下错误码，如果是上述的错误码需要重新初始化一下AudioSession。（示例代码中的代码也更新了）

附上OSStatus转成string的方法：

```objc
#import <Endian.h>

NSString * OSStatusToString(OSStatus status)
{
    size_t len = sizeof(UInt32);
    long addr = (unsigned long)&status;
    char cstring[5];
    
    len = (status >> 24) == 0 ? len - 1 : len;
    len = (status >> 16) == 0 ? len - 1 : len;
    len = (status >>  8) == 0 ? len - 1 : len;
    len = (status >>  0) == 0 ? len - 1 : len;
    
    addr += (4 - len);
    
    status = EndianU32_NtoB(status);        // strings are big endian
    
    strncpy(cstring, (char *)addr, len);
    cstring[len] = 0;
    
    return [NSString stringWithCString:(char *)cstring encoding:NSMacOSRomanStringEncoding];
}
```

----

#打断处理

正常启动AudioSession之后就可以播放音频了，下面要讲的是对于打断的处理。之前我们说到打断的回调在iOS5、6下需要统一管理，在收到打断开始和结束时需要发送自定义的通知。

使用`AudioSession`时打断回调应该首先获取`kAudioSessionProperty_InterruptionType`，然后发送一个自定义的通知并带上对应的参数。

```objc
static void MyAudioSessionInterruptionListener(void *inClientData, UInt32 inInterruptionState)
{
    AudioSessionInterruptionType interruptionType = kAudioSessionInterruptionType_ShouldNotResume;
    UInt32 interruptionTypeSize = sizeof(interruptionType);
    AudioSessionGetProperty(kAudioSessionProperty_InterruptionType,
                            &interruptionTypeSize,
                            &interruptionType);
    
    NSDictionary *userInfo = @{MyAudioInterruptionStateKey:@(inInterruptionState),
                 				 MyAudioInterruptionTypeKey:@(interruptionType)};
                 
    [[NSNotificationCenter defaultCenter] postNotificationName:MyAudioInterruptionNotification object:nil userInfo:userInfo];
}
```

收到通知后的处理方法如下（注意ShouldResume参数）：

```objc
- (void)interruptionNotificationReceived:(NSNotification *)notification
{
    UInt32 interruptionState = [notification.userInfo[MyAudioInterruptionStateKey] unsignedIntValue];
    AudioSessionInterruptionType interruptionType = [notification.userInfo[MyAudioInterruptionTypeKey] unsignedIntValue];
    [self handleAudioSessionInterruptionWithState:interruptionState type:interruptionType];
}

- (void)handleAudioSessionInterruptionWithState:(UInt32)interruptionState type:(AudioSessionInterruptionType)interruptionType
{
    if (interruptionState == kAudioSessionBeginInterruption)
    {
        //控制UI，暂停播放
    }
    else if (interruptionState == kAudioSessionEndInterruption)
    {
        if (interruptionType == kAudioSessionInterruptionType_ShouldResume)
        {
            OSStatus status = AudioSessionSetActive(true);
            if (status == noErr)
            {
                //控制UI，继续播放
            }
        }
    }
}
```

----

#小结

关于AudioSession的话题到此结束（码字果然很累。。）。小结一下：

* 如果最低版本支持iOS 5，请使用`AudioSession`，需要有一个类统一管理AudioSession的所有回调，在接到回调后发送对应的自定义通知；
* 如果最低版本支持iOS 6及以上，可以考虑使用`AVAudioSession`，仍然需要统一管理AudioSession的回调，其中打断回调可以使用`AVAudioSessionDelegate`的方法进行监听；
* 如果最低版本支持iOS 7及以上，请使用`AVAudioSession`，不用统一管理，接AVAudioSession的通知即可；
* 根据app的应用场景合理选择`Category`；
* 在deactive时需要注意app的应用场景来合理的选择是否使用`NotifyOthersOnDeactivation`参数；
* 在处理InterruptEnd事件时需要注意`ShouldResume`的值。

----

#示例代码

[这里](https://github.com/msching/MCAudioSession)有我自己写的`AudioSession`的封装，如果各位需要支持iOS 5和6的话可以使用一下。

----

#下篇预告

~~下一篇将讲述如何使用`AudioFileStreamer`分离音频帧，以及如何使用`AudioQueue`进行播放。~~

下一篇将讲述如何使用`AudioFileStreamer`提取音频文件格式信息和分离音频帧。

----

#参考资料

[AudioSession](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/AudioSessionProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40007875)
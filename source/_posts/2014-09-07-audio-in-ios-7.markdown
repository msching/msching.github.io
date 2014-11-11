---
layout: post
title: "iOS音频播放 (七)：播放iPod Library中的歌曲"
subtitle: "Audio playback in iOS (Part 7) : Access iPod Library"
date: 2014-09-07 15:45:47 +0800
comments: true
statement: true
categories: [iOS,Audio,iOS Audio]
---

#前言

由于最近工作量非常饱和，所以这第七篇来的有点晚（创建时间是9月7日。。说出来都是泪）。

现在市面上的音乐播放器都支持iPod Library歌曲（俗称iPod音乐或者本地音乐）的播放，用户对于iPod音乐播放的需求也一直十分强烈。这篇要讲的是如何来播放iPod Library的歌曲。

----

#概述

根据[官方文档](https://developer.apple.com/library/ios/documentation/audiovideo/conceptual/multimediapg/usingaudio/usingaudio.html#//apple_ref/doc/uid/TP40009767-CH2-SW43)描述Apple从iOS 3.0开始允许开发者访问用户的iPod library来获取用户放在其中的歌曲等多媒体内容。

为此Apple提供了多种方法来访问和播放iPod中的音乐，下面我们来分别列举一下这些方法。

----

#访问MediaLibrary

[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/AboutiPodLibraryAccess/AboutiPodLibraryAccess.html#//apple_ref/doc/uid/TP40008765-CH103-SW9)访问iPod Library的方法有两种，分别是MediaPicker和MediaQuery。

![](/images/iOS-audio/iPodLibraryAccessOverview.jpg)

###MediaPicker

MediaPicker是一个高度封装的iPod Library访问方式，通过使用`MPMediaPickerController`类来访问iPod Library。这是一个UI控件，用户可以根据需要选择其中的音乐。这个类使用时非常方便，只需要生成一个``的实例，设置一下属性和delegate后present出来，接下来只要等待回调即可，在回调时需要手动dismiss picker。

```objc
MPMediaPickerController *picker = [[MPMediaPickerController alloc] initWithMediaTypes:MPMediaTypeAnyAudio];
picker.prompt = @"请选择需要播放的歌曲";
picker.showsCloudItems = NO;
picker.allowsPickingMultipleItems = YES;
picker.delegate = self;
[self presentViewController:picker animated:YES completion:nil];


- (void)mediaPickerDidCancel:(MPMediaPickerController *)mediaPicker
{
    [mediaPicker dismissViewControllerAnimated:YES completion:nil];
}

- (void)mediaPicker:(MPMediaPickerController *)mediaPicker didPickMediaItems:(MPMediaItemCollection *)mediaItemCollection
{
    [mediaPicker dismissViewControllerAnimated:YES completion:nil];
    //do something
}
```

上面的代码将会得到如下的效果：

![](/images/iOS-audio/MediaPicker.jpg)

通过MediaPicker最终可以得到`MPMediaItemCollection`，其中存放着所有在Picker中选中的歌曲，每一个歌曲使用一个`MPMediaItem`对象表示。对于MediaPicker的使用也可以参考[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/UsingtheMediaItemPicker/UsingtheMediaItemPicker.html#//apple_ref/doc/uid/TP40008765-CH104-SW1)。

###MediaQuery

如果你觉得MeidaPicker的功能或者UI不能满足你的要求那么可以使用MediaQuery。MediaQuery可以直接访问iPod Library的DB，并根据需要获取数据。[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/UsingTheiPodLibrary/UsingTheiPodLibrary.html#//apple_ref/doc/uid/TP40008765-CH101-SW1)给出了MediaQuery的示意图。

![](/images/iOS-audio/database_access_classes.jpg)

MediaQuery功能十分强大，它可以根据一个或多个条件查询满足需要的MediaItem。

你可以使用`MPMediaQuery`的类方法来生成一些已经预置了条件的Query

```objc
// Base queries which can be used directly or as the basis for custom queries.
// The groupingType for these queries is preset to the appropriate type for the query.
+ (MPMediaQuery *)albumsQuery;
+ (MPMediaQuery *)artistsQuery;
+ (MPMediaQuery *)songsQuery;
+ (MPMediaQuery *)playlistsQuery;
+ (MPMediaQuery *)podcastsQuery;
+ (MPMediaQuery *)audiobooksQuery;
+ (MPMediaQuery *)compilationsQuery;
+ (MPMediaQuery *)composersQuery;
+ (MPMediaQuery *)genresQuery;
```
也可以自己生成`MPMediaPredicate`设置条件，并把它加到Query中，最后通过items和collections访问查询到的结果，例如：

```objc
MPMediaPropertyPredicate *artistNamePredicate =
[MPMediaPropertyPredicate predicateWithValue:@"Happy the Clown"
                                 forProperty:MPMediaItemPropertyArtist
                              comparisonType:MPMediaPredicateComparisonEqualTo];
                                  
MPMediaQuery *quert = [[MPMediaQuery alloc] init];
[quert addFilterPredicate: artistNamePredicate];
quert.groupingType = MPMediaGroupingArtist;
    
NSArray *itemsFromArtistQuery = [quert items];
NSArray *collectionsFromArtistQuery = [quert collections];
```
这一过程可以表示为（图来自[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/AboutiPodLibraryAccess/AboutiPodLibraryAccess.html#//apple_ref/doc/uid/TP40008765-CH103-SW9)）：

![](/images/iOS-audio/mediaQuery.jpg)

这里对于MediaQuery的用法就不再继续展开，关于这块内容并没有什么晦涩难懂的地方需要解释，大家可以通过阅读[官方文档](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/UsingTheiPodLibrary/UsingTheiPodLibrary.html#//apple_ref/doc/uid/TP40008765-CH101-SW1)来详细了解其用法。

###MediaCollection

`MPMediaCollection`是MediaItem的合集，可以通过访问它的items属性来访问所有的MediaItem。

`MPMediaPlaylist`是一个特殊的`MPMediaCollection`代表用户创建的播放列表，它会比MediaCollection包含更多的信息，比如播放列表的名字等等。这些属性可以通过`MPMediaEntity`的方法访问（MPMediaCollection是MPMediaEntity的子类，MPMediaItem也是）。
```objc
// Returns the value for the given entity property.
// MPMediaItem and MPMediaPlaylist have their own properties
- (id)valueForProperty:(NSString *)property;

// Executes a provided block with the fetched values for the given item properties, or nil if no value is available for a property.
// In some cases, enumerating the values for multiple properties can be more efficient than fetching each individual property with -valueForProperty:.
- (void)enumerateValuesForProperties:(NSSet *)properties usingBlock:(void (^)(NSString *property, id value, BOOL *stop))block NS_AVAILABLE_IOS(4_0);
```

###MediaItem

通过MediaPicker和MediaQuery最终都会得到`MPMediaItem`，这个item中包含了许多信息。这些信息都可以通过`MPMediaEntity`的方法访问，其中参数非常多就不列举了具体可以参照MPMediaItem.h。

----

#使用MPMusicPlayerController

拿到iPod Library中的歌曲后就可以开始播放了。播放的方式有很多种，先介绍一下`MediaPlayer framework`中的`MPMusicPlayerController`类。

通过`MPMusicPlayerController`的类方法可以生成两种播放器，生成方法如下：

```objc
// Playing media items with the applicationMusicPlayer will restore the user's iPod state after the application quits.
+ (MPMusicPlayerController *)applicationMusicPlayer;

// Playing media items with the iPodMusicPlayer will replace the user's current iPod state.
+ (MPMusicPlayerController *)iPodMusicPlayer;
```

这两个方法看似生成了一样的对象，但它们的行为却有很大不同。从Apple写的注释上我们可以很清楚的发现它们的区别。`+applicationMusicPlayer`不会继承来自iOS系统自带的iPod应用中的播放状态，同时也不会覆盖iPod的播放状态。而`+iPodMusicPlayer`完全继承iPod应用的播放状态（甚至是播放时间），对其实例的任何操作也会覆盖到iPod应用。对`+iPodMusicPlayer`方法command+点击后可以看到更详细的注释。

```
	The iPod music player employs the iPod app on your behalf. On instantiation, it takes on the current iPod app state and controls that state as your app runs. Specifically, the shared state includes the following:
	Repeat mode (see “Repeat Modes”)
	Shuffle mode (see “Shuffle Modes”
	Now-playing item (see nowPlayingItem)
	Playback state (see playbackState)
	
	Other aspects of iPod state, such as the on-the-go playlist, are not shared. Music that is playing continues to play when your app moves to the background.
```
说白了，当在使用iPodMusicPlayerv其实并不是你的程序在播放音频，而是你的程序在操纵iPod应用播放音频，即使你的程序crash了或者被kill了，音乐也不会因此停止。

而对于`+applicationMusicPlayer`通过command+点击可以看到：

```
The application music player plays music locally within your app. It does not affect the iPod state. 
When your app moves to the background, the music player stops if it was playing.

```
从注释中可以知道这个方法返回的对象虽然不是调用iPod应用播放的也不会影响到iPod应用，但它有个很大的缺点：无法后台播放，即使你在active了audioSession并且在app的设置中设置了Background Audio同样不会奏效。

综上所述，一般在开发音乐软件时很少用到这两个接口来进行iPod Library的播放，大部分开发者都是用这个类中的volme来调整系统音量的（这个属性在SDK 7中也被deprecate掉了）。如果你想用到这个类进行播放的话，这里需要提个醒，给`MPMusicPlayerController`设置需要播放的音乐时要使用下面两个方法：

```objc
// Call -play to begin playback after setting an item queue source. Setting a query will implicitly use MPMediaGroupingTitle.
- (void)setQueueWithQuery:(MPMediaQuery *)query;
- (void)setQueueWithItemCollection:(MPMediaItemCollection *)itemCollection;
```
而不是这个属性：

```objc
// Returns the currently playing media item, or nil if none is playing.
// Setting the nowPlayingItem to an item in the current queue will begin playback at that item.
@property(nonatomic, copy) MPMediaItem *nowPlayingItem;
```
光看名字很容易被`nowPlayingItem`这个属性迷惑，它的意思其实是说在设置了MediaQuery或者MediaCollection之后再设置这个nowPlayingItem可以让播放器从这个item开始播放，前提是这个item需要在MediaQuery或者MediaCollection的.items集合内。

----

#使用AVAudioPlayer和AVPlayer

除了使用MediaPlayer中的类还有很多其他方法来进行iPod播放，其中做的比较出色的是`AVFoundation`中的`AVAudioPlayer`和`AVPlayer`。

这两个类的都有通过NSURL生成实例的初始化方法：

```objc
//AVAudioPlayer
- (id)initWithContentsOfURL:(NSURL *)url error:(NSError **)outError;

//AVPlayer
+ (id)playerWithURL:(NSURL *)URL;
- (id)initWithURL:(NSURL *)URL;
```

其中的NSURL正是来自于`MPMediaItem`的`MPMediaItemPropertyAssetURL`属性。

```objc
//A URL pointing to the media item,
//from which an AVAsset object (or other URL-based AV Foundation object) can be created, with any options as desired. 
//Value is an NSURL object.
MP_EXTERN NSString *const MPMediaItemPropertyAssetURL;
```
上面讲到`MPMediaItem`时已经提到了它是`MPMediaEntity`子类，可以通过`-valueForProperty:`方法访问其中的属性。通过传入`MPMediaItemPropertyAssetURL`就可以得到当前MediaItem对应的URL（ipod-library://xxxxx），生成Player进行播放。大致代码如下：

```objc
@interface MyClass : NSObject
{
	AVAudioPlayer *_player;
	//AVPlayer *_player;
}

//设置AudioSession
[[AVAudioSession sharedInstance] setActive:YES error:nil];
[[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayback error:nil];

//play
NSError *error = nil;
MPMediaItem *item = ...;
NSURL *url = [item valueForProperty:MPMediaItemPropertyAssetURL];
_player = [[AVAudioPlayer alloc] initWithContentsOfURL:url error:&error];
//_player = [AVPlayer playerWithURL:url];
[_player play];

```

**注意：这里我需要更正一下，之前我在[第二篇](/blog/2014/07/08/audio-in-ios-2/)讲到AudioSession时写了这样一段话`在使用AVAudioPlayer/AVPlayer时可以不用关心AudioSession的相关问题，Apple已经把AudioSession的处理过程封装了...`。这段话不对，我把AVFoundation和Mediaplayer混淆了，在写的时候也没注意，应该是在使用MPMusicPlayerController播放时不需要关心AudioSession的问题。**

----

#读取和导出数据

前面说到使用`MPMediaItem`的`MPMediaItemPropertyAssetURL`属性可以得到一个表示当前MediaItem的NSURL，有了这个NSURL我们使用AVFoundation中的类进行播放。播放只是最基本的需求，有了这个URL我们可以做更多更有趣的事情。

在AVFoundation中还有两个有趣的类：`AVAssetReader`和`AVAssetExportSession`。它们可以把iPod Library中的指定歌曲以指定的音频格式导出到内存中或者硬盘中，这个指定的格式包括PCM。这是一个激动人心的特性，有了PCM数据我们就可以做很多很多其他的事情了。

这部分如果要展开的话还会有相当多的内容，国外的先辈们早在2010年就已经发掘了这两个类的用法，详细参见[这里](http://www.subfurther.com/blog/2010/07/19/from-iphone-media-library-to-pcm-samples-in-dozens-of-confounding-potentially-lossy-steps/)和[这里](http://www.subfurther.com/blog/2010/12/13/from-ipod-library-to-pcm-samples-in-far-fewer-steps-than-were-previously-necessary/)。这两篇讲的比较详细并且附有Sample（其中还涉及了一些Extended Audio File Services的内容），如果里面Sample无法下载可以从点击[MediaLibraryExportThrowaway1.zip](/files/MediaLibraryExportThrowaway1.zip)和[VTM_AViPodReader.zip](/files/VTM_AViPodReader.zip)下载。

**需要注意的是在使用`AVAssetReader`的过程中如果访问系统的相机或者照片可能会使`AVAssetReader`产生`AVErrorOperationInterrupted`错误，此时需要重新生成Reader后调用`-startReading`才可以继续读取数据。**

----

#小结

本篇介绍了一些与iPod Library相关的内容，小结一下：

* Apple提供两种方法来访问iPod Library，它们分别是`MPMediaPickerController`和`MPMediaQuery`；

* `MPMediaPickerController`和`MPMediaQuery`最后输出给开发者的对象是`MPMediaItem`，`MPMediaItem`的属性需要通过`-valueForProperty:`方法获取了；

* `MPMusicPlayerController`可以用来播放`MPMediaItem`，但有很多局限性，使用时需要根据不同的使用场景来决定用哪个类方法生成实例；

* `AVAudioPlayer`和`AVPlayer`也可以用来播放`MPMediaItem`，这两个类的功能比较完善，推荐使用，在使用之前别忘记设置AudioSession；

* `MPMediaItem`可以得到对应的URL，这个URL可以用来做很多事情，例如用`AVAssetReader`和`AVAssetExportSession`可以导出其中的数据；

----

#下篇预告

下一篇会讲一些关于NowPlayingCenter和RemoteControl的内容（就是在锁屏界面和ControlCenter中显示的歌曲信息以及上面的那些播放控制按钮）。

----

#参考资料

[iPod Library Access Programming Guide](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008765)

[About iPod Library Access](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/AboutiPodLibraryAccess/AboutiPodLibraryAccess.html#//apple_ref/doc/uid/TP40008765-CH103-SW9)

[Using the Media Item Picker](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/UsingtheMediaItemPicker/UsingtheMediaItemPicker.html#//apple_ref/doc/uid/TP40008765-CH104-SW1)

[Using the iPod Library](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/UsingTheiPodLibrary/UsingTheiPodLibrary.html#//apple_ref/doc/uid/TP40008765-CH101-SW1)

[Using Media Playback](https://developer.apple.com/library/ios/documentation/Audio/Conceptual/iPodLibraryAccess_Guide/UsingMediaPlayback/UsingMediaPlayback.html#//apple_ref/doc/uid/TP40008765-CH100-SW1)

[From iPhone Media Library to PCM Samples in Dozens of Confounding, Potentially Lossy Steps](http://www.subfurther.com/blog/2010/07/19/from-iphone-media-library-to-pcm-samples-in-dozens-of-confounding-potentially-lossy-steps/)

[From iPod Library to PCM Samples in Far Fewer Steps Than Were Previously Necessary](http://www.subfurther.com/blog/2010/12/13/from-ipod-library-to-pcm-samples-in-far-fewer-steps-than-were-previously-necessary/)
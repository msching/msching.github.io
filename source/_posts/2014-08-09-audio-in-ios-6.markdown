---
layout: post
title: "iOS音频播放 (六)：简单的音频播放器实现"
subtitle: "Audio playback in iOS (Part 6) : Create a simple audio player"
date: 2014-08-09 15:55:22 +0800
comments: true
statement: true
categories: [iOS,Audio,iOS Audio]
---

#前言

在前几篇中我分别讲到了`AudioSession`、`AudioFileStream`、`AudioFile`、`AudioQueue`，这些类的功能已经涵盖了[第一篇](/blog/2014/07/07/audio-in-ios/)中所提到的音频播放所需要的步骤：

1. 读取MP3文件	 `NSFileHandle`
2. 解析采样率、码率、时长等信息，分离MP3中的音频帧  `AudioFileStream`/`AudioFile`
3. 对分离出来的音频帧解码得到PCM数据  `AudioQueue`
4. ~~对PCM数据进行音效处理（均衡器、混响器等，非必须）~~  `省略`
5. 把PCM数据解码成音频信号  `AudioQueue`
6. 把音频信号交给硬件播放	`AudioQueue`
7. 重复1-6步直到播放完成

下面我们就讲讲述如何用这些部件组成一个简单的`本地音乐播放器`，这里我会用到**AudioSession**、**AudioFileStream**、**AudioFile**、**AudioQueue**。

**注意：在阅读本篇请实现阅读并理解前面1-5篇的内容以及2-5篇最后给出的封装类，本篇中的播放器实现将基于前面几篇中给出的[MCAudioSession](https://github.com/msching/MCAudioSession)、[MCAudioFileStream](https://github.com/msching/MCAudioFileStream)、[MCAudioFile](https://github.com/msching/MCAudioFile)和[MCAudioOutputQueue](https://github.com/msching/MCSimpleAudioPlayer/blob/master/MCSimpleAudioPlayerDemo/MCSimpleAudioPlayer/MCAudioOutputQueue.h)进行实现。**

----

#AudioFileStream vs AudioFile

解释一下为什么我要同时使用**AudioFileStream**和**AudioFile**。

第一，`对于网络流播必须有AudioFileStream的支持`，这是因为我们在[第四篇](/blog/2014/07/19/audio-in-ios-4/)中提到过**AudioFile**在Open时会要求使用者提供数据，如果提供的数据不足会直接跳过并且返回错误码，而数据不足的情况在网络流中很常见，故无法使用**AudioFile**单独进行网络流数据的解析；

第二，`对于本地音乐播放选用AudioFile更为合适`，原因如下：

1. **AudioFileStream**的主要是用在流播放中虽然不限于网络流和本地流，但流数据是按顺序提供的所以**AudioFileStream**也是顺序解析的，被解析的音频文件还是需要符合流播放的特性，对于不符合的本地文件**AudioFileStream**会在Parse时返回`NotOptimized`错误；
2. **AudioFile**的解析过程并不是顺序的，它会在解析时通过回调向使用者索要某个位置的数据，即使数据在文件末尾也不要紧，所以**AudioFile**适用于所有类型的音频文件；

基于以上两点我们可以得出这样一个结论：`一款完整功能的播放器应当同时使用AudioFileStream和AudioFile`，用**AudioFileStream**来应对可以进行流播放的音频数据，以达到边播放边缓冲的最佳体验，用**AudioFile**来处理无法流播放的音频数据，让用户在下载完成之后仍然能够进行播放。

本来这个Demo应该做成基于网络流的音频播放，但由于最近比较忙一直过着公司和床两点一线的生活，来不及写网络流和文件缓存的模块，所以就用本地文件代替了，所以最终在Demo会先尝试用**AudioFileStream**解析数据，如果失败再尝试使用**AudioFile**以达到模拟网络流播放的效果。


----

#准备工作

第一件事当然是要创建一个新工程，这里我选择了的模板是SingleView，工程名我把它命名为`MCSimpleAudioPlayerDemo`：

![](/images/iOS-audio/createproject.jpg)

创建完工程之后去到Target属性的`Capabilities`选项卡设置`Background Modes`，把`Audio and Airplay`勾选，这样我们的App就可以在进入后台之后继续播放音乐了：

![](/images/iOS-audio/setBackgroundPlayback.jpg)

接下来我们需要搭建一个简单的UI，在storyboard上创建两个UIButton和一个UISlider，Button用来做播放器的播放、暂停、停止等功能控制，Slider用来显示播放进度和seek。把这些UI组件和ViewController的属性/方法关联上之后简单的UI也就完成了。

![](/images/iOS-audio/simpleUI.jpg)

----

#接口定义

下面来创建播放器类`MCSimpleAudioPlayer`，首先是初始化方法（感谢[@喵神](http://weibo.com/onevcat?topnav=1&wvr=5&topsug=1)的[VVDocumenter](https://github.com/onevcat/VVDocumenter-Xcode)）：

```objc
/**
 *  初始化方法
 *
 *  @param filePath 文件绝对路径
 *  @param fileType 文件类型，作为后续创建AudioFileStream和AudioQueue的Hint使用
 *
 *  @return player对象
 */
- (instancetype)initWithFilePath:(NSString *)filePath fileType:(AudioFileTypeID)fileType;
```
另外播放器作为一个典型的状态机，各种状态也是必不可少的，这里我只简单的定义了四种状态：

```objc
typedef NS_ENUM(NSUInteger, MCSAPStatus)
{
    MCSAPStatusStopped = 0,
    MCSAPStatusPlaying = 1,
    MCSAPStatusWaiting = 2,
    MCSAPStatusPaused = 3,
};
```

再加上一些必不可少的属性和方法组成了`MCSimpleAudioPlayer.h`

```objc
@interface MCSimpleAudioPlayer : NSObject

@property (nonatomic,copy,readonly) NSString *filePath;
@property (nonatomic,assign,readonly) AudioFileTypeID fileType;

@property (nonatomic,readonly) MCSAPStatus status;
@property (nonatomic,readonly) BOOL isPlayingOrWaiting;
@property (nonatomic,assign,readonly) BOOL failed;

@property (nonatomic,assign) NSTimeInterval progress;
@property (nonatomic,readonly) NSTimeInterval duration;

- (instancetype)initWithFilePath:(NSString *)filePath fileType:(AudioFileTypeID)fileType;

- (void)play;
- (void)pause;
- (void)stop;
@end
```
----

#初始化

在init方法中创建一个NSFileHandle的实例以用来读取数据并交给AudioFileStream解析，另外也可以根据生成的实例是否是nil来判断是否能够读取文件，如果返回的是nil就说明文件不存在或者没有权限那么播放也就无从谈起了。

```objc
_fileHandler = [NSFileHandle fileHandleForReadingAtPath:_filePath];
```

通过NSFileManager获取文件大小

```objc
_fileSize = [[[NSFileManager defaultManager] attributesOfItemAtPath:_filePath error:nil] fileSize];
```
初始化方法到这里就结束了，作为一个播放器我们自然不能在主线程进行播放，我们需要创建自己的播放线程。

创建一个成员变量`_started`来表示播放流程是否已经开始，在`-play`方法中如果`_started`为NO就创建线程`_thread`并以`-threadMain`方法作为main，否则说明线程已经创建并且在播放流程中：

```
- (void)play
{
    if (!_started)
    {
        _started = YES;
        _thread = [[NSThread alloc] initWithTarget:self selector:@selector(threadMain) object:nil];
        [_thread start];
    }
    else
    {
        //如果是Pause状态就resume
    }
}
```
接下来就可以在`-threadMain`进行音频播放相关的操作了。

----

#创建AudioSession

iOS音频播放的第一步，自然是要创建`AudioSession`，这里引入[第二篇](/blog/2014/07/08/audio-in-ios-2/)末尾给出的AudioSession封装[MCAudioSession](https://github.com/msching/MCAudioSession)，当然各位也可以使用`AVAudioSession`。

初始化的工作会在调用单例方法时进行，下一步是设置Category。

```objc
//初始化并且设置Category
[[MCAudioSession sharedInstance] setCategory:kAudioSessionCategory_MediaPlayback error:NULL];

```
成功之后启用AudioSession，还有别忘了监听Interrupt通知。

```objc
if ([[MCAudioSession sharedInstance] setCategory:kAudioSessionCategory_MediaPlayback error:NULL])
{
    //active audiosession
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(interruptHandler:) name:MCAudioSessionInterruptionNotification object:nil];
    if ([[MCAudioSession sharedInstance] setActive:YES error:NULL])
    {
        //go on
    }
}
```


----

#读取、解析音频数据

成功创建并启用AudioSession之后就可以进入播放流程了，播放是一个无限循环的过程，所以我们需要一个while循环，在文件没有被播放完成之前需要反复的读取、解析、播放。那么第一步是需要读取并解析数据。按照之前说的我们会先使用`AudioFileStream`，引入[第三篇](/blog/2014/07/09/audio-in-ios-3/)末尾给出的AudioFileStream封装[MCAudioFileStream](https://github.com/msching/MCAudioFileStream)。

创建AudioFileStream，**MCAudioFileStream**的init方法会完成这项工作，如果创建成功就设置delegate作为Parse数据的回调。

```objc
_audioFileStream = [[MCAudioFileStream alloc] initWithFileType:_fileType fileSize:_fileSize error:&error];
if (!error)
{
    _audioFileStream.delegate = self;
}
```

接下来要读取数据并且解析，用成员变量`_offset`表示`_fileHandler`已经读取文件位置，其主要作用是来判断Eof。调用**MCAudioFileStream**的`-parseData:error:`方法来对数据进行解析。

```objc
NSData *data = [_fileHandler readDataOfLength:1000];
_offset += [data length];
if (_offset >= _fileSize)
{
    isEof = YES;
}
[_audioFileStream parseData:data error:&error];
if (error)
{
    //解析失败，换用AudioFile
}

```
解析完文件头之后**MCAudioFileStream**的`readyToProducePackets`属性会被置为YES，此后所有的Parse方法都回触发`-audioFileStream:audioDataParsed:`方法并传递`MCParsedAudioData`的数组来保存解析完成的数据。这样就需要一个buffer来存储这些解析完成的音频数据。

于是创建了`MCAudioBuffer`类来管理所有解析完成的数据：

```objc
@interface MCAudioBuffer : NSObject

+ (instancetype)buffer;

- (void)enqueueData:(MCParsedAudioData *)data;
- (void)enqueueFromDataArray:(NSArray *)dataArray;

- (BOOL)hasData;
- (UInt32)bufferedSize;

- (NSData *)dequeueDataWithSize:(UInt32)requestSize 
 					  packetCount:(UInt32 *)packetCount 
 					 descriptions:(AudioStreamPacketDescription **)descriptions;

- (void)clean;
@end

```

创建一个**MCAudioBuffer**的实例`_buffer`，解析完成的数据都会通过`enqueue`方法存储到**_buffer**中，在需要的使用可以通过`dequeue`取出来使用。

```objc
_buffer = [MCAudioBuffer buffer]; //初始化方法中创建

//AudioFileStream解析完成的数据都被存储到了_buffer中
- (void)audioFileStream:(MCAudioFileStream *)audioFileStream audioDataParsed:(NSArray *)audioData
{
    [_buffer enqueueFromDataArray:audioData];
}
```

如果遇到**AudioFileStream**解析失败的话，转而使用**AudioFile**，引入[第四篇](/blog/2014/07/19/audio-in-ios-4/)末尾给出的AudioFile封装[MCAudioFile](https://github.com/msching/MCAudioFile)（之前没有给出，最近补上的）。

```objc
_audioFileStream parseData:data error:&error];
if (error)
{
    //解析失败，换用AudioFile
    _usingAudioFile = YES;
    continue;
}
```

```objc
if (_usingAudioFile)
{
    if (!_audioFile)
    {
        _audioFile = [[MCAudioFile alloc] initWithFilePath:_filePath fileType:_fileType];
    }
    if ([_buffer bufferedSize] < _bufferSize || !_audioQueue)
    {    
        //AudioFile解析完成的数据都被存储到了_buffer中
        NSArray *parsedData = [_audioFile parseData:&isEof];
        [_buffer enqueueFromDataArray:parsedData];
    }
}

```

使用**AudioFile**时同样需要**NSFileHandle**来读取文件数据，但由于其回获取数据的特性我把FileHandle的相关操作都封装进去了，所以使用**MCAudioFile**解析数据时直接调用Parse方法即可。

----

#播放

有了解析完成的数据，接下来就该AudioQueue出场了，引入[第五篇](/blog/2014/08/02/audio-in-ios-5/)末尾提到的AudioQueue的封装[MCAudioOutputQueue](https://github.com/msching/MCSimpleAudioPlayer/blob/master/MCSimpleAudioPlayerDemo/MCSimpleAudioPlayer/MCAudioOutputQueue.h)。

首先创建AudioQueue，由于AudioQueue需要实现创建重用buffer所以需要事先确定bufferSize，这里我设置的bufferSize是近似0.1秒的数据量，计算bufferSize需要用到的duration和audioDataByteCount可以从**MCAudioFileStream**或者**MCAudioFile**中获取。有了bufferSize之后，加上数据格式format参数和magicCookie（部分音频格式需要）就可以生成AudioQueue了。

```objc
- (BOOL)createAudioQueue
{
    if (_audioQueue)
    {
        return YES;
    }
    
    NSTimeInterval duration = _usingAudioFile ? _audioFile.duration : _audioFileStream.duration;
    UInt64 audioDataByteCount = _usingAudioFile ? _audioFile.audioDataByteCount : _audioFileStream.audioDataByteCount;
    _bufferSize = 0;
    if (duration != 0)
    {
        _bufferSize = (0.1 / duration) * audioDataByteCount;
    }
    
    if (_bufferSize > 0)
    {
        AudioStreamBasicDescription format = _usingAudioFile ? _audioFile.format : _audioFileStream.format;
        NSData *magicCookie = _usingAudioFile ? [_audioFile fetchMagicCookie] : [_audioFileStream fetchMagicCookie];
        _audioQueue = [[MCAudioOutputQueue alloc] initWithFormat:format bufferSize:_bufferSize macgicCookie:magicCookie];
        if (!_audioQueue.available)
        {
            _audioQueue = nil;
            return NO;
        }
    }
    return YES;
}
```
接下来从**_buffer**中读出解析完成的数据，交给AudioQueue播放。如果全部播放完毕了就调用一下`-flush`让AudioQueue把剩余数据播放完毕。这里需要注意的是**MCAudioOutputQueue**的`-playData`方法在调用时如果没有可以重用的buffer的话会阻塞当前线程直到`AudioQueue`回调方法送出可重用的buffer为止。
```
UInt32 packetCount;
AudioStreamPacketDescription *desces = NULL;
NSData *data = [_buffer dequeueDataWithSize:_bufferSize packetCount:&packetCount descriptions:&desces];
if (packetCount != 0)
{
    [_audioQueue playData:data packetCount:packetCount packetDescriptions:desces isEof:isEof];
    free(desces);
    
    if (![_buffer hasData] && isEof)
    {
        [_audioQueue flush];
        break;
    }
}
```

----

#暂停 & 恢复

暂停方法很简单，调用**MCAudioOutputQueue**的`-pause`方法就可以了，但要注意的是需要和`-playData:`同步调用，否则可能引起一些问题（比如触发了pause实际由于并发操作没有真正pause住）。

同步的方法可以采用加锁的方式，也可以通过标志位在`threadMain`中进行Pause，Demo中我使用了后者。

```objc
//pause方法
- (void)pause
{
    if (self.isPlayingOrWaiting)
    {
        _pauseRequired = YES;
    }
}


//threadMain中
- (void)threadMain
{
    ...
	
    //pause
    if (_pauseRequired)
    {
        [self setStatusInternal:MCSAPStatusPaused];
        [_audioQueue pause];
        [self _mutexWait];
        _pauseRequired = NO;
    }
	
    //play
    ...
}
```
在暂停后还要记得阻塞线程。

恢复只要调用**AudioQueue** start方法就可以了，同时记得signal让线程继续跑

```objc
- (void)_resume
{
    //AudioQueue的start方法被封装到了MCAudioOutputQueue的resume方法中
    [_audioQueue resume];
    [self _mutexSignal];
}
```

----

#播放进度 & Seek

对于播放进度我在[第五篇](/blog/2014/08/02/audio-in-ios-5/)讲**AudioQueue**时已经提到过了，使用`AudioQueueGetCurrentTime`方法可以获取`实际播放的时间`如果Seek之后需要根据计算timingOffset，然后根据timeOffset来计算最终的播放进度：

```objc
- (NSTimeInterval)progress
{
    return _timingOffset + _audioQueue.playedTime;
}
```
timingOffset的计算在Seek进行，Seek操作和暂停操作一样需要和其他**AudioQueue**的操作同步进行，否则可能造成一些并发问题。

```objc
//seek方法
- (void)setProgress:(NSTimeInterval)progress
{
    _seekRequired = YES;
    _seekTime = progress;
}
```
在seek时为了防止播放进度跳动，修改一下获取播放进度的方法：

```objc
- (NSTimeInterval)progress
{
    if (_seekRequired)
    {
        return _seekTime;
    }
    return _timingOffset + _audioQueue.playedTime;
}
```
下面是`threadMain`中的Seek操作

```objc
if (_seekRequired)
{
    [self setStatusInternal:MCSAPStatusWaiting];
    
    _timingOffset = _seekTime - _audioQueue.playedTime;
    [_buffer clean];
    if (_usingAudioFile)
    {
        [_audioFile seekToTime:_seekTime];
    }
    else
    {
        _offset = [_audioFileStream seekToTime:&_seekTime];
        [_fileHandler seekToFileOffset:_offset];
    }
    _seekRequired = NO;
    [_audioQueue reset];
}
```
Seek时需要做如下事情：

1. 计算timingOffset
2. 清除之前残余在`_buffer`中的数据
3. 挪动NSFileHandle的游标
4. 清除**AudioQueue**中已经Enqueue的数据
5.  如果有用到音效器的还需要清除音效器里的残余数据

---

#打断

在接到Interrupt通知时需要处理打断，下面是打断的处理方法：

```objc
- (void)interruptHandler:(NSNotification *)notification
{
    UInt32 interruptionState = [notification.userInfo[MCAudioSessionInterruptionStateKey] unsignedIntValue];
    
    if (interruptionState == kAudioSessionBeginInterruption)
    {
        _pausedByInterrupt = YES;
        [_audioQueue pause];
        [self setStatusInternal:MCSAPStatusPaused];
        
    }
    else if (interruptionState == kAudioSessionEndInterruption)
    {
        AudioSessionInterruptionType interruptionType = [notification.userInfo[MCAudioSessionInterruptionTypeKey] unsignedIntValue];
        if (interruptionType == kAudioSessionInterruptionType_ShouldResume)
        {
            if (self.status == MCSAPStatusPaused && _pausedByInterrupt)
            {
                if ([[MCAudioSession sharedInstance] setActive:YES error:NULL])
                {
                    [self play];
                }
            }
        }
    }
}
```

这里需要注意，打断操作我放在了主线程进行而并非放到新开的线程中进行，原因如下：

* 一旦打断开始**AudioSession**被抢占后音频立即被打断，此时**AudioQueue**的所有操作会暂停，这就意味着不会有任何数据消耗回调产生；

* 我这个Demo的线程模型中在向**AudioQueue** Enqueue了足够多的数据之后会阻塞当前线程等待数据消耗的回调才会signal让线程继续跑；

于是就得到了这样的结论：一旦打断开始我创建的线程就会被阻塞，所以我需要在主线程来处理暂停和恢复播放。


----

#停止 & 清理

停止操作也和其他操作一样会放到`threadMain`中执行

```objc
- (void)stop
{
    _stopRequired = YES;
    [self _mutexSignal];
}


//treadMain中
if (_stopRequired)
{
    _stopRequired = NO;
    _started = NO;
    [_audioQueue stop:YES];
    break;
}
```

在播放被停止或者出错时会进入到清理流程，这里需要做一大堆操作，清理各种数据，关闭AudioSession，清除各种标记等等。

```objc
- (void)cleanup
{
    //reset file
    _offset = 0;
    [_fileHandler seekToFileOffset:0];
    
    //deactive audiosession
    [[MCAudioSession sharedInstance] setActive:NO error:NULL];
    [[NSNotificationCenter defaultCenter] removeObserver:self name:MCAudioSessionInterruptionNotification object:nil];
    
    //clean buffer
    [_buffer clean];
    
    _usingAudioFile = NO;
    //close audioFileStream
    [_audioFileStream close];
    
    //close audiofile
    [_audioFile close];
    
    //stop audioQueue
    [_audioQueue stop:YES];
    
    //destory mutex & cond
    [self _mutexDestory];
    
    _started = NO;
    _timingOffset = 0;
    _seekTime = 0;
    _seekRequired = NO;
    _pauseRequired = NO;
    _stopRequired = NO;
    
    //reset status
    [self setStatusInternal:MCSAPStatusStopped];
}

```
----

#连接播放器UI

播放器代码完成后就需要和UI连起来让播放器跑起来了。

在viewDidLoad时创建一个播放器：

```objc

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    if (!_player)
    {
        NSString *path = [[NSBundle mainBundle] pathForResource:@"MP3Sample" ofType:@"mp3"];
        _player = [[MCSimpleAudioPlayer alloc] initWithFilePath:path fileType:kAudioFileMP3Type];
        
        [_player addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionNew context:nil];
    }
    [_player play];
}
```
对播放器的status属性KVO用来操作播放和暂停按钮的状态以及播放进度timer的开启和暂停：

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if (object == _player)
    {
        if ([keyPath isEqualToString:@"status"])
        {
            [self performSelectorOnMainThread:@selector(handleStatusChanged) withObject:nil waitUntilDone:NO];
        }
    }
}

- (void)handleStatusChanged
{
    if (_player.isPlayingOrWaiting)
    {
        [self.playOrPauseButton setTitle:@"Pause" forState:UIControlStateNormal];
        [self startTimer];
        
    }
    else
    {
        [self.playOrPauseButton setTitle:@"Play" forState:UIControlStateNormal];
        [self stopTimer];
        [self progressMove:nil];
    }
}

```

播放进度交给timer来刷新：

```
- (void)startTimer
{
    if (!_timer)
    {
        _timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(progressMove:) userInfo:nil repeats:YES];
        [_timer fire];
    }
}

- (void)stopTimer
{
    if (_timer)
    {
        [_timer invalidate];
        _timer = nil;
    }
}

- (void)progressMove:(id)sender
{
    //在seek时不要刷新slider的thumb位置
    if (!self.progressSlider.tracking)
    {
        if (_player.duration != 0)
        {
            self.progressSlider.value = _player.progress / _player.duration;
        }
        else
        {
            self.progressSlider.value = 0;
        }
    }
}
```

监听slider的两个TouchUp时间来进行seek操作：

```objc
- (IBAction)seek:(id)sender
{
    _player.progress = _player.duration * self.progressSlider.value;
}
```

添加两个按钮的TouchUpInside事件进行播放控制：

```objc
- (IBAction)playOrPause:(id)sender
{
    if (_player.isPlayingOrWaiting)
    {
        [_player pause];
    }
    else
    {
        [_player play];
    }
}

- (IBAction)stop:(id)sender
{
    [_player stop];
}

```

----

#进阶的内容

关于简单播放器的构建就讲这么多，以下是一些音频播放相关的进阶内容，由于我自己也没有摸透它们所以暂时就不做详细介绍了以免误人子弟-_-，各位有兴趣可以研究一下，如果有疑问或者有新发现欢迎大家留言或者在[微博](http://weibo.com/msching)上和我交流共同提高~

1. `AudioConverter`可以实现音频数据的转换，在播放流程中它可以充当解码器的角色，可以把压缩的音频数据解码成为PCM数据；
2. `AudioUnit`作为比`AudioQueue`更底层的音频播放类库，Apple赋予了它更强大的功能，除了一般的播放功能之外它还能使用iPhone自带的多种均衡器对音效进行调节；
3. `AUGraph`为`AudioUnit`提供音效处理功能（这个其实我一点也没接触过0_0）

----

#示例代码

上面所讲述的内容对应的工程已经在github上了（[MCSimpleAudioPlayer](https://github.com/msching/MCSimpleAudioPlayer)），有任何问题可以给我发issue~

----

#下篇预告

下一篇会介绍一下如何播放iOS系统`iPod Library`中的歌曲（俗称iPod音乐或者本地音乐）


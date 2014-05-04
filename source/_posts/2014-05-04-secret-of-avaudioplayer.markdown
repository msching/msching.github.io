---
layout: post
title: "AVAudioPlayer的小秘密 - Secret of AVAudioPlayer"
date: 2014-05-04 14:43:40 +0800
comments: true
statement: true
categories: [iOS,Audio]
---

#问题

前两天公司有一位同事在使用AVAudioPlayer的过程中遇到了这样一个问题：

他需要播放一段网络上的音频，实现策略是把音频下载到本地，然后使用AVAudioPlayer进行播放。代码大致是这样的：

```objective-c
NSString *path = .../xxx.mp3; //mp3 file path
NSData *data = [NSData dataWithContentsOfURL:[NSURL fileURLWithPath:path]];
NSError *error = nil;
AVAudioPlayer *player = [[AVAudioPlayer alloc] initWithData:data error:&error];
```

但他在init AVAudioPlayer时遇到了下面的错误。

```objective-c
Error Domain=NSOSStatusErrorDomain Code=1937337955 "The operation couldn’t be completed. (OSStatus error 1937337955.)"
```

----------

#普遍的解决方法

在google搜索之后发现1937337955这个code并不少见，在[StackOverflow](http://stackoverflow.com/questions/4901709/iphone-avaudioplayer-unsupported-file-type)上有人提问问到这个问题，国内的一些博客中也有提到（例如[@我的桌子](http://zhu340381425.blog.163.com/blog/static/75952514201192021013852)和[@SkyLine](http://blog.sina.com.cn/s/blog_7cb9b3b80101d8ap.html)）。

其中提到的解决方法都一样，就是使用AVAudioPlayer的另一个init方法：

```objective-c
- (id)initWithContentsOfURL:(NSURL *)url error:(NSError **)outError;
```
于是尝试修改了代码：

```objective-c
NSString *path = .../xxx.mp3; //mp3 file path
NSError *error = nil;
AVAudioPlayer *player = [[AVAudioPlayer alloc] initWithContentsOfURL:[NSURL fileURLWithPath:path] error:&error];
```
果然没有出现错误，player成功创建并且能够播放。

----------

#深究

不能播放的问题到这里已经fix了，但这个问题本身还没有完结，为什么使用`-initWithContentsOfURL:error:`方法就可以播呢？这个时候也许有的人会认为这是一个apple的bug，认为`-initWithContentsOfURL:error:`方法比`-initWithData::error:`具有更好的适应性。


	Oh that's very interesting. Perhaps that should be submitted to Apple as a bug? 
	In the end I opted for the saved file anyways because it fit better with what we were trying to do. 
	Thanks for the Tip! –  mtmurdock Mar 19 '11 at 0:26


但凡事遇到错误，都应该先从自身开始找原因。经过搜索发现，1937337955这个Errorcode其实就是`kAudioFileUnsupportedFileTypeError`，一般出现在文件格式不符合规范的情况下。假设apple并没有写出bug的话，那么上述问题中的这个mp3一定在文件格式上有缺陷，最终导致了`-initWithData::error:`方法返回错误，而`-initWithContentsOfURL:error:`采用某种方式规避了这个格式缺陷。

回过头去查看`AVAudioPlayer.h`头文件可以看到SDK7中多了两个init方法：

```objective-c
/* The file type hint is a constant defined in AVMediaFormat.h whose value is a UTI for a file format. e.g. AVFileTypeAIFF. */
/* Sometimes the type of a file cannot be determined from the data, or it is actually corrupt.
The file type hint tells the parser what kind of data to look for so that files which are not self identifying or possibly even corrupt can be successfully parsed. */
- (id)initWithContentsOfURL:(NSURL *)url fileTypeHint:(NSString*)utiString error:(NSError **)outError NS_AVAILABLE(10_9, 7_0);
- (id)initWithData:(NSData *)data fileTypeHint:(NSString*)utiString error:(NSError **)outError NS_AVAILABLE(10_9, 7_0);
```

多出来的这个hint参数和`AudioToolbox`framework中`AudioFileStream`、`AudioFile`两个类的Open方法中所使用的hint参数作用一样，可以辅助系统判定当前的文件格式。

接下来尝试在iOS7系统下使用新的init方法生成AVAudioPlayer：

```objective-c
NSString *path = .../xxx.mp3; //mp3 file path
NSData *data = [NSData dataWithContentsOfURL:[NSURL fileURLWithPath:path]];
NSError *error = nil;
AVAudioPlayer *player = [[AVAudioPlayer alloc] initWithData:data fileTypeHint:AVFileTypeMPEGLayer3 error:&error];
```

结果没有错误产生，没有出现错误，player成功创建并且能够播放。

进而进行另两个实验：

1. 把保存文件时的.mp3后缀去掉后使用`-initWithContentsOfURL:error:`生成对象，结果发现产生了错误。
2. 把保存文件时的.mp3后缀改为.wav后使用`-initWithContentsOfURL:error:`生成对象，结果发现产生了错误。

于是确定`-initWithContentsOfURL:error:`方法是利用后缀名作为hintType对文件的解码进行辅助，而`-initWithData::error:`方法由于没有任何hint并且文件本身格式又有缺陷导致错误的产生。

----------

#更完整的解决方法

基于以上分析可以得出一个更为完整的解决方法，可以有效的规避这一类错误：

1. 对于iOS7以上的系统（含iOS7）,在确定文件格式的情况下可以使用`initWithData:fileTypeHint:error:`和`initWithContentsOfURL:fileTypeHint:error:`生成实例，或者把data保存为对应后缀名的文件后使用`-initWithContentsOfURL:error:`后再生成实例；
2. 对于iOS7以下的系统，在确定文件格式的情况下，最为安全的方法是把data保存为对应后缀名的文件后使用`-initWithContentsOfURL:error:`生成实例；

如果上述方法帮不了你，那么就只能去检查文件格式有没有问题或者采用其他的实现方式进行尝试了（比如`AVPlayer`和`AudioToolBox`）。不管怎么说，客户端所能做的只是尽量减少错误发生的频率，最终解决这类问题还是需要音频文件的提供者确保音频文件的格式符合标准没有错误和缺陷。
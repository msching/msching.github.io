---
layout: post
title: "iOS音频播放 (四)：AudioFile"
subtitle: "Audio playback in iOS (Part 4) : AudioFile"
date: 2014-07-19 13:38:30 +0800
comments: true
statement: true
categories: [iOS,Audio]
---

#前言

接着[第三篇](/blog/2014/07/09/audio-in-ios-3/)的`AudioStreamFile`这一篇要来聊一下`AudioFile`。和`AudioStreamFile`一样`AudioFile`是`AudioToolBox` framework中的一员，它也能够完成[第一篇](/blog/2014/07/07/audio-in-ios/)所述的第2步，读取音频格式信息和进行帧分离，但事实上它的功能远不止如此。

-----

#AudioFile介绍

按照[官方文档](https://developer.apple.com/library/mac/documentation/musicaudio/reference/AudioFileConvertRef/Reference/reference.html#//apple_ref/c/func/AudioFileCreateWithURL)的描述：

`a C programming interface that enables you to read or write a wide variety of audio data to or from disk or a memory buffer.With Audio File Services you can:`

* Create, initialize, open, and close audio files
* Read and write audio files
* Optimize audio files
* Work with user data and global information

这个类可以用来创建、初始化音频文件；读写音频数据；对音频文件进行优化；读取和写入音频格式信息等等，功能十分强大，可见它不但可以用来支持音频播放，甚至可以用来生成音频文件。当然，在本篇文章中只会涉及一些和音频播放相关的内容（打开音频文件、读取格式信息、读取音频数据，其实我也只对这些方法有一点了解，其余的功能没用过。。>_<）.

----

#AudioFile的打开“姿势”

`AudioFile`提供了两个打开文件的方法：

1、 `AudioFileOpenURL`

```objc

enum {
	kAudioFileReadPermission      = 0x01,
	kAudioFileWritePermission     = 0x02,
	kAudioFileReadWritePermission = 0x03
};

extern OSStatus AudioFileOpenURL (CFURLRef inFileRef,
									SInt8 inPermissions,
									AudioFileTypeID inFileTypeHint,
									AudioFileID * outAudioFile);
```

从方法的定义上来看是用来读取本地文件的：

第一个参数，文件路径；

第二个参数，文件的允许使用方式，是读、写还是读写，如果打开文件后进行了允许使用方式以外的操作，就得到`kAudioFilePermissionsError`错误码（比如Open时声明是`kAudioFileReadPermission`但却调用了`AudioFileWriteBytes`）；

第三个参数，和`AudioFileStream`的open方法中一样是一个帮助`AudioFile`解析文件的类型提示，如果文件类型确定的话应当传入；

第四个参数，返回AudioFile实例对应的`AudioFileID`，这个ID需要保存起来作为后续一些方法的参数使用；

返回值用来判断是否成功打开文件（OSSStatus == noErr）。

----

2、 `AudioFileOpenWithCallbacks`

```objc
extern OSStatus AudioFileOpenWithCallbacks (void * inClientData,
											  AudioFile_ReadProc inReadFunc,
											  AudioFile_WriteProc inWriteFunc,
											  AudioFile_GetSizeProc inGetSizeFunc,
											  AudioFile_SetSizeProc inSetSizeFunc,
											  AudioFileTypeID inFileTypeHint,
											  AudioFileID * outAudioFile);
```

看过第一个Open方法后，这个方法乍看上去让人有点迷茫，没有URL的参数如何告诉AudioFile该打开哪个文件？还是先来看一下参数的说明吧：

第一个参数，上下文信息，不再多做解释；

第二个参数，当`AudioFile`需要读音频数据时进行的回调（调用Open和Read方式后`同步`回调）；

第三个参数，当`AudioFile`需要写音频数据时进行的回调（写音频文件功能时使用，暂不讨论）；

第四个参数，当`AudioFile`需要用到文件的总大小时回调（调用Open和Read方式后`同步`回调）；

第五个参数，当`AudioFile`需要设置文件的大小时回调（写音频文件功能时使用，暂不讨论）；

第六、七个参数和返回值同`AudioFileOpenURL`方法；

这个方法的重点在于`AudioFile_ReadProc`这个回调。换一个角度理解，这个方法相比于第一个方法自由度更高，AudioFile需要的只是一个数据源，无论是磁盘上的文件、内存里的数据甚至是网络流只要能在`AudioFile`需要数据时（Open和Read时）通过`AudioFile_ReadProc`回调为AudioFile提供合适的数据就可以了，也就是说使用方法不仅仅可以读取本地文件也可以如`AudioFileStream`一样以流的形式读取数据。

----

下面来看一下`AudioFile_GetSizeProc`和`AudioFile_ReadProc`这两个读取功能相关的回调
```objc
typedef SInt64 (*AudioFile_GetSizeProc)(void * inClientData);

typedef OSStatus (*AudioFile_ReadProc)(void * inClientData,
										 SInt64 inPosition,
										 UInt32 requestCount,
										 void * buffer, 
										 UInt32 * actualCount);
```

首先是`AudioFile_GetSizeProc`回调，这个回调很好理解，返回文件总长度即可，总长度的获取途径自然是文件系统或者httpResponse等等。

接下来是`AudioFile_ReadProc`回调：

第一个参数，上下文对象，不再赘述；

第二个参数，需要读取第几个字节开始的数据；

第三个参数，需要读取的数据长度；

第四个参数，返回参数，是一个数据指针并且其空间已经被分配，我们需要做的是把数据memcpy到buffer中；

第五个参数，实际提供的数据长度，即memcpy到buffer中的数据长度；

返回值，如果没有任何异常产生就返回noErr，如果有异常可以根据异常类型选择需要的error常量返回（一般用不到其他返回值，返回noErr就足够了）；

这里需要解释一下这个回调方法的工作方式。`AudioFile`需要数据时会调用回调方法，需要数据的时间点有两个：

1. Open方法调用时，由于`AudioFile`的Open方法调用过程中就会对音频格式信息进行解析，只有符合要求的音频格式才能被成功打开否则Open方法就会返回错误码（换句话说，Open方法一旦调用成功就相当于`AudioStreamFile`在Parse后返回`ReadyToProducePackets`一样，只要Open成功就可以开始读取音频数据，详见[第三篇](/blog/2014/07/09/audio-in-ios-3/)），所以在Open方法调用的过程中就需要提供一部分音频数据来进行解析；

2. Read相关方法调用时，这个不需要多说很好理解；

通过回调提供数据时需要注意inPosition和requestCount参数，这两个参数指明了本次回调需要提供的数据范围是从inPosition开始requestCount个字节的数据。这里又可以分为两种情况：

1. 有充足的数据：那么我们需要把这个范围内的数据拷贝到buffer中，并且给actualCount赋值requestCount，最后返回noError；

2. 数据不足：没有充足数据的话就只能把手头有的数据拷贝到buffer中，需要注意的是这部分被拷贝的数据必须是从inPosition开始的`连续数据`，拷贝完成后给actualCount赋值实际拷贝进buffer中的数据长度后返回noErr，这个过程可以用下面的代码来表示：

```objc
static OSStatus MyAudioFileReadCallBack(void *inClientData,
                                        SInt64 inPosition,
                                        UInt32 requestCount,
                                        void *buffer,
                                        UInt32 *actualCount)
{
    __unsafe_unretained MyContext *context = (__bridge MyContext *)inClientData;
        
    *actualCount = [context availableDataLengthAtOffset:inPosition maxLength:requestCount];
    if (*actualCount > 0)
    {
        NSData *data = [context dataAtOffset:inPosition length:*actualCount];
        memcpy(buffer, [data bytes], [data length]);
    }
    
    return noErr;
}

```

说到这里又需要分两种情况：

2.1. Open方法调用时的回调数据不足：AudioFile的Open方法会根据文件格式类型分几步进行数据读取以解析确定是否是一个合法的文件格式，其中每一步的inPosition和requestCount都不一样，如果某一步不成功就会直接进行下一步，如果几部下来都失败了，那么Open方法就会失败。简单的说就是在调用Open之前首先需要保证音频文件的格式信息完整，这就意味着`AudioFile`并不能独立用于音频流的读取，在流播放时首先需要使用`AudioStreamFile`来得到`ReadyToProducePackets`标志位来保证信息完整；

2.2. Read方法调用时的回调数据不足：这种情况下inPosition和requestCount的数值与Read方法调用时传入的参数有关，数据不足对于Read方法本身没有影响，只要回调返回noErr，Read就成功，只是实际交给Read方法的调用方的数据会不足，那么就把这个问题的处理交给了Read的调用方；


----

#读取音频格式信息

成功打开音频文件后就可以读取其中的格式信息了，读取用到的方法如下：

```objc
extern OSStatus AudioFileGetPropertyInfo(AudioFileID inAudioFile,
										   AudioFilePropertyID inPropertyID,
										   UInt32 * outDataSize,
										   UInt32 * isWritable);
										   
extern OSStatus AudioFileGetProperty(AudioFileID inAudioFile,
									   AudioFilePropertyID inPropertyID,
									   UInt32 * ioDataSize,
									   void * outPropertyData);	
```
`AudioFileGetPropertyInfo`方法用来获取某个属性对应的数据的大小（outDataSize）以及该属性是否可以被write（isWritable），而`AudioFileGetProperty`则用来获取属性对应的数据。对于一些大小可变的属性需要先使用`AudioFileGetPropertyInfo`获取数据大小才能取获取数据（例如formatList），而有些确定类型单个属性则不必先调用`AudioFileGetPropertyInfo`直接调用`AudioFileGetProperty`即可（比如BitRate），例子如下：

```objc
AudioFileID fileID; //Open方法返回的AudioFileID

//获取格式信息
UInt32 formatListSize = 0;
OSStatus status = AudioFileGetPropertyInfo(_fileID, kAudioFilePropertyFormatList, &formatListSize, NULL);
if (status == noErr)
{
    AudioFormatListItem *formatList = (AudioFormatListItem *)malloc(formatListSize);
    status = AudioFileGetProperty(fileID, kAudioFilePropertyFormatList, &formatListSize, formatList);
    if (status == noErr)
    {
        for (int i = 0; i * sizeof(AudioFormatListItem) < formatListSize; i += sizeof(AudioFormatListItem))
        {
            AudioStreamBasicDescription pasbd = formatList[i].mASBD;
            //选择需要的格式。。                             
        }
    }
    free(formatList);
}

//获取码率
UInt32 bitRate;
UInt32 bitRateSize = sizeof(bitRate);
status = AudioFileGetProperty(fileID, kAudioFilePropertyBitRate, &size, &bitRate);
if (status != noErr)
{
    //错误处理
}
```
可以获取的属性有下面这些，大家可以参考文档来获取自己需要的信息（注意到这里有EstimatedDuration，可以得到Duration了）：

```
enum
{
	kAudioFilePropertyFileFormat				=	'ffmt',
	kAudioFilePropertyDataFormat				=	'dfmt',
	kAudioFilePropertyIsOptimized			=	'optm',
	kAudioFilePropertyMagicCookieData		=	'mgic',
	kAudioFilePropertyAudioDataByteCount		=	'bcnt',
	kAudioFilePropertyAudioDataPacketCount	=	'pcnt',
	kAudioFilePropertyMaximumPacketSize		=	'psze',
	kAudioFilePropertyDataOffset				=	'doff',
	kAudioFilePropertyChannelLayout			=	'cmap',
	kAudioFilePropertyDeferSizeUpdates		=	'dszu',
	kAudioFilePropertyMarkerList				=	'mkls',
	kAudioFilePropertyRegionList				=	'rgls',
	kAudioFilePropertyChunkIDs				=	'chid',
	kAudioFilePropertyInfoDictionary        	=	'info',
	kAudioFilePropertyPacketTableInfo		=	'pnfo',
	kAudioFilePropertyFormatList				=	'flst',
	kAudioFilePropertyPacketSizeUpperBound  	=	'pkub',
	kAudioFilePropertyReserveDuration		=	'rsrv',
	kAudioFilePropertyEstimatedDuration		=	'edur',
	kAudioFilePropertyBitRate				=	'brat',
	kAudioFilePropertyID3Tag					=	'id3t',
	kAudioFilePropertySourceBitDepth			=	'sbtd',
	kAudioFilePropertyAlbumArtwork			=	'aart',
  kAudioFilePropertyAudioTrackCount       	=    'atct',
	kAudioFilePropertyUseAudioTrack			=	'uatk'
};	
```

----

#读取音频数据

读取音频数据的方法分为两类：

1、直接读取音频数据：

```objc
extern OSStatus AudioFileReadBytes (AudioFileID inAudioFile,
									  Boolean inUseCache,
									  SInt64 inStartingByte,
									  UInt32 * ioNumBytes,
									  void * outBuffer);
```

第一个参数，FileID；

第二个参数，是否需要cache，一般来说传false；

第三个参数，从第几个byte开始读取数据

第四个参数，这个参数在调用时作为输入参数表示需要读取读取多少数据，调用完成后作为输出参数表示实际读取了多少数据（即Read回调中的requestCount和actualCount）；

第五个参数，buffer指针，需要事先分配好足够大的内存（ioNumBytes大，即Read回调中的buffer，所以Read回调中不需要再分配内存）；

返回值表示是否读取成功，EOF时会返回`kAudioFileEndOfFileError`；

使用这个方法得到的数据都是没有进行过帧分离的数据，如果想要用来播放或者解码还必须通过`AudioFileStream`进行帧分离；

2、按帧（Packet）读取音频数据：

```objc
extern OSStatus AudioFileReadPacketData (AudioFileID inAudioFile,
										   Boolean inUseCache,
										   UInt32 * ioNumBytes,
										   AudioStreamPacketDescription * outPacketDescriptions,
										   SInt64 inStartingPacket,
										   UInt32 * ioNumPackets,
										   void * outBuffer);
										   

extern OSStatus AudioFileReadPackets (AudioFileID inAudioFile,
										Boolean inUseCache,
										UInt32 * outNumBytes,
										AudioStreamPacketDescription * outPacketDescriptions,
										SInt64 inStartingPacket, 
										UInt32 * ioNumPackets, 
										void * outBuffer);
```
按帧读取的方法有两个，这两个方法看上去差不多，就连参数也几乎相同，但使用场景和效率上却有所不同，[官方文档](https://developer.apple.com/library/mac/documentation/musicaudio/reference/AudioFileConvertRef/Reference/reference.html#//apple_ref/c/func/AudioFileCreateWithURL)中如此描述这两个方法：

* `AudioFileReadPacketData` is memory efficient when reading variable bit-rate (VBR) audio data;
* `AudioFileReadPacketData` is more efficient than `AudioFileReadPackets` when reading compressed file formats that do not have packet tables, such as MP3 or ADTS. This function is a good choice for reading either CBR (constant bit-rate) or VBR data if you do not need to read a fixed duration of audio.
* Use `AudioFileReadPackets` only when you need to read a fixed duration of audio data, or when you are reading only uncompressed audio.

只有当需要读取固定时长音频或者非压缩音频时才会用到`AudioFileReadPackets`，其余时候使用`AudioFileReadPacketData`会有更高的效率并且更省内存；

下面来看看这些参数：

第一、二个参数，同`AudioFileReadBytes`；

第三个参数，对于`AudioFileReadPacketData`来说ioNumBytes这个参数在输入输出时都要用到，在输入时表示outBuffer的size，输出时表示实际读取了多少size的数据。而对`AudioFileReadPackets`来说outNumBytes只在输出时使用，表示实际读取了多少size的数据；

第四个参数，帧信息数组指针，在输入前需要分配内存，大小必须足够存在ioNumPackets个帧信息（ioNumPackets * sizeof(AudioStreamPacketDescription)）；

第五个参数，在输入时表示需要读取多少个帧，在输出时表示实际读取了多少帧；

第六个参数，outBuffer数据指针，在输入前就需要分配好空间，这个参数看上去两个方法一样但其实并非如此。对于`AudioFileReadPacketData`来说只要分配`近似帧大小 * 帧数`的内存空间即可，方法本身会针对给定的内存空间大小来决定最后输出多少个帧，如果空间不够会适当减少出的帧数；而对于`AudioFileReadPackets`来说则需要分配`最大帧大小(或帧大小上界) * 帧数`的内存空间才行（最大帧大小和帧大小上界的区别等下会说）；这也就是为何第三个参数一个是输入输出双向使用的，而另一个只是输出时使用的原因。就这点来说两个方法中前者在使用的过程中要比后者更省内存；

返回值，同`AudioFileReadBytes`；

这两个方法读取后的数据为帧分离后的数据，可以直接用来播放或者解码。

下面给出两个方法的使用代码（以MP3为例）：

```objc
AudioFileID fileID; //Open方法返回的AudioFileID
UInt32 ioNumPackets = ...; //要读取多少个packet
SInt64 inStartingPacket = ...; //从第几个Packet开始读取

UInt32 bitRate = ...; //AudioFileGetProperty读取kAudioFilePropertyBitRate
UInt32 sampleRate = ...; //AudioFileGetProperty读取kAudioFilePropertyDataFormat或kAudioFilePropertyFormatList
UInt32 byteCountPerPacket = 144 * bitRate / sampleRate; //MP3数据每个Packet的近似大小

UInt32 descSize = sizeof(AudioStreamPacketDescription) * ioNumPackets;
AudioStreamPacketDescription * outPacketDescriptions = (AudioStreamPacketDescription *)malloc(descSize);

UInt32 ioNumBytes = byteCountPerPacket * ioNumPackets;
void * outBuffer = (void *)malloc(ioNumBytes);

OSStatus status = AudioFileReadPacketData(fileID, 
											false, 
											&ioNumBytes, 
											outPacketDescriptions, 
											inStartingPacket, 
											&ioNumPackets, 
											outBuffer);
```

```objc
AudioFileID fileID; //Open方法返回的AudioFileID
UInt32 ioNumPackets = ...; //要读取多少个packet
SInt64 inStartingPacket = ...; //从第几个Packet开始读取

UInt32 maxByteCountPerPacket = ...; //AudioFileGetProperty读取kAudioFilePropertyMaximumPacketSize，最大的packet大小
//也可以用：
//UInt32 byteCountUpperBoundPerPacket = ...; //AudioFileGetProperty读取kAudioFilePropertyPacketSizeUpperBound，当前packet大小上界（未扫描全文件的情况下）

UInt32 descSize = sizeof(AudioStreamPacketDescription) * ioNumPackets;
AudioStreamPacketDescription * outPacketDescriptions = (AudioStreamPacketDescription *)malloc(descSize);

UInt32 outNumBytes = 0；
UInt32 ioNumBytes = maxByteCountPerPacket * ioNumPackets;
void * outBuffer = (void *)malloc(ioNumBytes);

OSStatus status = AudioFileReadPackets(fileID,
                                       false,
                                       &outNumBytes,
                                       outPacketDescriptions,
                                       inStartingPacket,
                                       &ioNumPackets,
                                       outBuffer);
```

----

#Seek

seek的思路和之前讲`AudioFileStream`时讲到的是一样的，区别在于AudioFile没有方法来帮助修正seek的offset和seek的时间：

* 使用`AudioFileReadBytes`时需要计算出approximateSeekOffset
* 使用`AudioFileReadPacketData`或者`AudioFileReadPackets`时需要计算出seekToPacket

approximateSeekOffset和seekToPacket的计算方法参见[第三篇](/blog/2014/07/09/audio-in-ios-3/)。

----

#关闭AudioFile

`AudioFile`使用完毕后需要调用`AudioFileClose`进行关闭，没啥特别需要注意的。

```objc
extern OSStatus AudioFileClose (AudioFileID inAudioFile);	
```

----

#小结

本篇针对`AudioFile`的音频读取功能做了介绍，小结一下：

* `AudioFile`有两个Open方法，需要针对自身的使用场景选择不同的方法；

* `AudioFileOpenURL`用来读取本地文件

* `AudioFileOpenWithCallbacks`的使用场景比前者要广泛，使用时需要注意`AudioFile_ReadProc`，这个回调方法在Open方法本身和Read方法被调用时会被`同步`调用

* 必须保证音频文件格式信息可读时才能使用`AudioFile`的Open方法，AudioFile并不能独立用于音频流的读取，需要配合`AudioStreamFile`使用才能读取流（需要用`AudioStreamFile`来判断文件格式信息可读之后再调用Open方法）；

* 使用`AudioFileGetProperty`读取格式信息时需要判断所读取的信息是否需要先调用`AudioFileGetPropertyInfo`获得数据大小后再进行读取；

* 读取音频数据应该根据使用的场景选择不同的音频读取方法，对于不同的读取方法seek时需要计算的变量也不相同；

* `AudioFile`使用完毕后需要调用`AudioFileClose`进行关闭；

----

#示例代码

对于本地文件用AudioFile读取比较简单就不在这里提供demo了，对于流播放中的AudioFile使用推荐大家阅读豆瓣的开源播放器代码[DOUAudioStreamer](https://github.com/douban/DOUAudioStreamer)。

----

#参考资料
[Audio File Services Reference](https://developer.apple.com/library/mac/documentation/musicaudio/reference/AudioFileConvertRef/Reference/reference.html#//apple_ref/c/func/AudioFileCreateWithURL)
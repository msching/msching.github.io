---
layout: post
title: "iOS音频播放 (三)：AudioFileStream"
subtitle: "Audio playback in iOS (Part 3) : AudioFileStream"
date: 2014-07-09 11:31:28 +0800
comments: true
statement: true
categories: [iOS,Audio,iOS Audio]
---

本来说好是要在第三篇中讲`AudioFileStream`和`AudioQueue`，但写着写着发现光`AudioFileStream`就好多内容，最后还是决定分篇介绍，这篇先来说一下`AudioFileStream`，下一篇计划说一下和`AudioFileStream`类似的`AudioFile`，下下篇再来说`AudioQueue`。

在本篇那种将会提到计算音频时长duration和音频seek的方法，这些方法对于CBR编码形式的音频文件可以做到比较精确而对于VBR编码形式的会存在较大的误差（关于CBR和VBR，请看本系列的[第一篇](/blog/2014/07/07/audio-in-ios/)），具体讲到duration和seek时会再进行说明。

<!--more-->

----

#AudioFileStream介绍

在[第一篇](/blog/2014/07/07/audio-in-ios/)中说到`AudioFileStreamer`时提到它的作用是用来读取采样率、码率、时长等基本信息以及分离音频帧。那么在[官方文档](https://developer.apple.com/library/ios/documentation/audiovideo/conceptual/multimediapg/usingaudio/usingaudio.html#//apple_ref/doc/uid/TP40009767-CH2-SW28)中Apple是这样描述的：

`To play streamed audio content, such as from a network connection, use Audio File Stream Services in concert with Audio Queue Services. Audio File Stream Services parses audio packets and metadata from common audio file container formats in a network bitstream. You can also use it to parse packets and metadata from on-disk files`

根据Apple的描述`AudioFileStreamer`用在流播放中，当然不仅限于网络流，本地文件同样可以用它来读取信息和分离音频帧。`AudioFileStreamer`的主要数据是文件数据而不是文件路径，所以数据的读取需要使用者自行实现，

支持的文件格式有：

* MPEG-1 Audio Layer 3, used for .mp3 files
* MPEG-2 ADTS, used for the .aac audio data format
* AIFC
* AIFF
* CAF
* MPEG-4, used for .m4a, .mp4, and .3gp files
* NeXT
* WAVE

上述格式是iOS、MacOSX所支持的音频格式，这类格式可以被系统提供的API解码，如果想要解码其他的音频格式（如OGG、APE、FLAC）就需要自己实现解码器了。

----

#初始化AudioFileStream


第一步，自然是要生成一个`AudioFileStream`的实例：
```objc
extern OSStatus AudioFileStreamOpen (void * inClientData,
									   AudioFileStream_PropertyListenerProc inPropertyListenerProc,
									   AudioFileStream_PacketsProc inPacketsProc,
									   AudioFileTypeID inFileTypeHint,
									   AudioFileStreamID * outAudioFileStream);
```
第一个参数和之前的AudioSession的初始化方法一样是一个上下文对象；

第二个参数`AudioFileStream_PropertyListenerProc`是歌曲信息解析的回调，每解析出一个歌曲信息都会进行一次回调；

第三个参数`AudioFileStream_PacketsProc`是分离帧的回调，每解析出一部分帧就会进行一次回调；

第四个参数`AudioFileTypeID`是文件类型的提示，这个参数来帮助`AudioFileStream`对文件格式进行解析。这个参数在文件信息不完整（例如信息有缺陷）时尤其有用，它可以给与`AudioFileStream`一定的提示，帮助其绕过文件中的错误或者缺失从而成功解析文件。所以在确定文件类型的情况下建议各位还是填上这个参数，如果无法确定可以传入0（原理上应该和[这篇博文](/blog/2014/05/04/secret-of-avaudioplayer/)近似）；

```objc
//AudioFileTypeID枚举
enum {
        kAudioFileAIFFType				= 'AIFF',
        kAudioFileAIFCType				= 'AIFC',
        kAudioFileWAVEType				= 'WAVE',
        kAudioFileSoundDesigner2Type	= 'Sd2f',
        kAudioFileNextType				= 'NeXT',
        kAudioFileMP3Type				= 'MPG3',	// mpeg layer 3
        kAudioFileMP2Type				= 'MPG2',	// mpeg layer 2
        kAudioFileMP1Type				= 'MPG1',	// mpeg layer 1
		  kAudioFileAC3Type				= 'ac-3',
        kAudioFileAAC_ADTSType			= 'adts',
        kAudioFileMPEG4Type            = 'mp4f',
        kAudioFileM4AType              = 'm4af',
        kAudioFileM4BType              = 'm4bf',
		  kAudioFileCAFType				= 'caff',
		  kAudioFile3GPType				= '3gpp',
		  kAudioFile3GP2Type				= '3gp2',		
		  kAudioFileAMRType				= 'amrf'		
};
```

第五个参数是返回的AudioFileStream实例对应的`AudioFileStreamID`，这个ID需要保存起来作为后续一些方法的参数使用；

返回值用来判断是否成功初始化（OSStatus == noErr）。

----

#解析数据

在初始化完成之后，只要拿到文件数据就可以进行解析了。解析时调用方法：

```objc
extern OSStatus AudioFileStreamParseBytes(AudioFileStreamID inAudioFileStream,
											UInt32 inDataByteSize,
											const void* inData,
											UInt32 inFlags);
```
第一个参数`AudioFileStreamID`，即初始化时返回的ID；

第二个参数inDataByteSize，本次解析的数据长度；

第三个参数inData，本次解析的数据；

第四个参数是说本次的解析和上一次解析是否是连续的关系，如果是连续的传入0，否则传入`kAudioFileStreamParseFlag_Discontinuity`。

这里需要插入解释一下何谓“连续”。在第一篇中我们提到过形如MP3的数据都以帧的形式存在的，解析时也需要以帧为单位解析。但在解码之前我们不可能知道每个帧的边界在第几个字节，所以就会出现这样的情况：我们传给AudioFileStreamParseBytes的数据在解析完成之后会有一部分数据余下来，这部分数据是接下去那一帧的前半部分，如果再次有数据输入需要继续解析时就必须要用到前一次解析余下来的数据才能保证帧数据完整，所以在正常播放的情况下传入0即可。目前知道的需要传入`kAudioFileStreamParseFlag_Discontinuity`的情况有两个，一个是在**seek完毕之后**显然seek后的数据和之前的数据完全无关；另一个是开源播放器[AudioStreamer](https://github.com/mattgallagher/AudioStreamer)的作者@Matt Gallagher曾在自己的[blog](http://www.cocoawithlove.com/2008/09/streaming-and-playing-live-mp3-stream.html)中提到过的：

`the Audio File Stream Services hit me with a nasty bug: AudioFileStreamParseBytes will crash when trying to parse a streaming MP3. `


`In this case, if we pass the kAudioFileStreamParseFlag_Discontinuity flag to AudioFileStreamParseBytes on every invocation between receiving kAudioFileStreamProperty_ReadyToProducePackets and the first successful call to MyPacketsProc, then AudioFileStreamParseBytes will be extra cautious in its approach and won't crash.`

Matt发布这篇blog是在2008年，这个Bug年代相当久远了，而且原因未知，究竟是否修复也不得而知，而且由于环境不同（比如测试用的mp3文件和所处的iOS系统）无法重现这个问题，所以我个人觉得还是按照Matt的work around在回调得到`kAudioFileStreamProperty_ReadyToProducePackets`之后，在正常解析第一帧之前都传入`kAudioFileStreamParseFlag_Discontinuity`比较好。

回到之前的内容，`AudioFileStreamParseBytes`方法的返回值表示当前的数据是否被正常解析，如果OSStatus的值不是noErr则表示解析不成功，其中错误码包括：

```objc
enum
{
	kAudioFileStreamError_UnsupportedFileType		= 'typ?',
	kAudioFileStreamError_UnsupportedDataFormat		= 'fmt?',
	kAudioFileStreamError_UnsupportedProperty		= 'pty?',
	kAudioFileStreamError_BadPropertySize			= '!siz',
	kAudioFileStreamError_NotOptimized				= 'optm',
	kAudioFileStreamError_InvalidPacketOffset		= 'pck?',
	kAudioFileStreamError_InvalidFile				= 'dta?',
	kAudioFileStreamError_ValueUnknown				= 'unk?',
	kAudioFileStreamError_DataUnavailable			= 'more',
	kAudioFileStreamError_IllegalOperation			= 'nope',
	kAudioFileStreamError_UnspecifiedError			= 'wht?',
	kAudioFileStreamError_DiscontinuityCantRecover	= 'dsc!'
};

```
大多数都可以从字面上理解，需要提一下的是`kAudioFileStreamError_NotOptimized`，文档上是这么说的：

`It is not possible to produce output packets because the file's packet table or other defining info is either not present or is after the audio data.`

它的含义是说这个音频文件的文件头不存在或者说文件头可能在文件的末尾，当前无法正常Parse，换句话说就是这个文件需要全部下载完才能播放，无法流播。

**注意`AudioFileStreamParseBytes`方法每一次调用都应该注意返回值，一旦出现错误就可以不必继续Parse了。**

----

#解析文件格式信息
在调用`AudioFileStreamParseBytes`方法进行解析时会首先读取格式信息，并同步的进入`AudioFileStream_PropertyListenerProc`回调方法

![](/images/iOS-audio/audiofilestreamParse-1.jpg)

来看一下这个回调方法的定义
```objc
typedef void (*AudioFileStream_PropertyListenerProc)(void * inClientData,
													   AudioFileStreamID inAudioFileStream,
													   AudioFileStreamPropertyID inPropertyID,
													   UInt32 * ioFlags);
```


回调的第一个参数是Open方法中的上下文对象；

第二个参数inAudioFileStream是和Open方法中第四个返回参数`AudioFileStreamID`一样，表示当前FileStream的ID；

第三个参数是此次回调解析的信息ID。表示当前PropertyID对应的信息已经解析完成信息（例如数据格式、音频数据的偏移量等等），使用者可以通过`AudioFileStreamGetProperty`接口获取PropertyID对应的值或者数据结构；

```objc
extern OSStatus AudioFileStreamGetProperty(AudioFileStreamID inAudioFileStream,
											 AudioFileStreamPropertyID inPropertyID,
											 UInt32 * ioPropertyDataSize,
											 void * outPropertyData);

```

第四个参数ioFlags是一个返回参数，表示这个property是否需要被缓存，如果需要赋值`kAudioFileStreamPropertyFlag_PropertyIsCached`否则不赋值（这个参数我也不知道应该在啥场景下使用。。一直都没去理他）；

这个回调会进来多次，但并不是每一次都需要进行处理，可以根据需求处理需要的PropertyID进行处理（PropertyID列表如下）。
```objc
//AudioFileStreamProperty枚举
enum
{
	kAudioFileStreamProperty_ReadyToProducePackets			=	'redy',
	kAudioFileStreamProperty_FileFormat						=	'ffmt',
	kAudioFileStreamProperty_DataFormat						=	'dfmt',
	kAudioFileStreamProperty_FormatList						=	'flst',
	kAudioFileStreamProperty_MagicCookieData					=	'mgic',
	kAudioFileStreamProperty_AudioDataByteCount				=	'bcnt',
	kAudioFileStreamProperty_AudioDataPacketCount			=	'pcnt',
	kAudioFileStreamProperty_MaximumPacketSize				=	'psze',
	kAudioFileStreamProperty_DataOffset						=	'doff',
	kAudioFileStreamProperty_ChannelLayout					=	'cmap',
	kAudioFileStreamProperty_PacketToFrame					=	'pkfr',
	kAudioFileStreamProperty_FrameToPacket					=	'frpk',
	kAudioFileStreamProperty_PacketToByte					=	'pkby',
	kAudioFileStreamProperty_ByteToPacket					=	'bypk',
	kAudioFileStreamProperty_PacketTableInfo					=	'pnfo',
	kAudioFileStreamProperty_PacketSizeUpperBound  			=	'pkub',
	kAudioFileStreamProperty_AverageBytesPerPacket			=	'abpp',
	kAudioFileStreamProperty_BitRate							=	'brat',
  kAudioFileStreamProperty_InfoDictionary                  =    'info'
};
```

这里列几个我认为比较重要的PropertyID：

1、`kAudioFileStreamProperty_BitRate`：

表示音频数据的码率，获取这个Property是为了计算音频的总时长Duration（因为AudioFileStream没有这样的接口。。）。

```objc
UInt32 bitRate;
UInt32 bitRateSize = sizeof(bitRate);
OSStatus status = AudioFileStreamGetProperty(inAudioFileStream, kAudioFileStreamProperty_BitRate, &bitRateSize, &bitRate);
if (status != noErr)
{
    //错误处理
}
```

**2014.8.2 补充：**
发现在流播放的情况下，有时数据流量比较小时会出现`ReadyToProducePackets`还是没有获取到bitRate的情况，这时就需要分离一些拼音帧然后计算平均bitRate，计算公式如下：

```objc
UInt32 averageBitRate = totalPackectByteCount / totalPacketCout;
```


2、`kAudioFileStreamProperty_DataOffset`：

表示音频数据在整个音频文件中的offset（因为大多数音频文件都会有一个文件头之后才使真正的音频数据），这个值在seek时会发挥比较大的作用，音频的seek并不是直接seek文件位置而seek时间（比如seek到2分10秒的位置），seek时会根据时间计算出音频数据的字节offset然后需要再加上音频数据的offset才能得到在文件中的真正offset。
```objc
SInt64 dataOffset;
UInt32 offsetSize = sizeof(dataOffset);
OSStatus status = AudioFileStreamGetProperty(inAudioFileStream, kAudioFileStreamProperty_DataOffset, &offsetSize, &dataOffset);
if (status != noErr)
{
    //错误处理
}
```


3、`kAudioFileStreamProperty_DataFormat`

表示音频文件结构信息，是一个AudioStreamBasicDescription的结构

```objc
struct AudioStreamBasicDescription
{
    Float64 mSampleRate;
    UInt32  mFormatID;
    UInt32  mFormatFlags;
    UInt32  mBytesPerPacket;
    UInt32  mFramesPerPacket;
    UInt32  mBytesPerFrame;
    UInt32  mChannelsPerFrame;
    UInt32  mBitsPerChannel;
    UInt32  mReserved;
};

AudioStreamBasicDescription asbd;
UInt32 asbdSize = sizeof(asbd);
OSStatus status = AudioFileStreamGetProperty(inAudioFileStream, kAudioFileStreamProperty_DataFormat, &asbdSize, &asbd);
if (status != noErr)
{
    //错误处理
}	
```

4、`kAudioFileStreamProperty_FormatList`

作用和`kAudioFileStreamProperty_DataFormat`是一样的，区别在于用这个PropertyID获取到是一个AudioStreamBasicDescription的数组，这个参数是用来支持AAC SBR这样的包含多个文件类型的音频格式。由于到底有多少个format我们并不知晓，所以需要先获取一下总数据大小：

```objc
//获取数据大小
Boolean outWriteable;
UInt32 formatListSize;
OSStatus status = AudioFileStreamGetPropertyInfo(inAudioFileStream, kAudioFileStreamProperty_FormatList, &formatListSize, &outWriteable);
if (status != noErr)
{
    //错误处理
}

//获取formatlist
AudioFormatListItem *formatList = malloc(formatListSize);
OSStatus status = AudioFileStreamGetProperty(inAudioFileStream, kAudioFileStreamProperty_FormatList, &formatListSize, formatList);
if (status != noErr)
{
    //错误处理
}

//选择需要的格式
for (int i = 0; i * sizeof(AudioFormatListItem) < formatListSize; i += sizeof(AudioFormatListItem))
{
    AudioStreamBasicDescription pasbd = formatList[i].mASBD;
    //选择需要的格式。。                             
}
free(formatList);
```

5、`kAudioFileStreamProperty_AudioDataByteCount`

顾名思义，音频文件中音频数据的总量。这个Property的作用一是用来计算音频的总时长，二是可以在seek时用来计算时间对应的字节offset。

```objc
UInt64 audioDataByteCount;
UInt32 byteCountSize = sizeof(audioDataByteCount);
OSStatus status = AudioFileStreamGetProperty(inAudioFileStream, kAudioFileStreamProperty_AudioDataByteCount, &byteCountSize, &audioDataByteCount);
if (status != noErr)
{
    //错误处理
}
```

**2014.8.2 补充：**
发现在流播放的情况下，有时数据流量比较小时会出现`ReadyToProducePackets`还是没有获取到audioDataByteCount的情况，这时就需要近似计算audioDataByteCount。一般来说音频文件的总大小一定是可以得到的（利用文件系统或者Http请求中的contentLength），那么计算方法如下：

```objc
UInt32 dataOffset = ...; //kAudioFileStreamProperty_DataOffset
UInt32 fileLength = ...; //音频文件大小
UInt32 audioDataByteCount = fileLength - dataOffset;
```


5、`kAudioFileStreamProperty_ReadyToProducePackets`

这个PropertyID可以不必获取对应的值，一旦回调中这个PropertyID出现就代表解析完成，接下来可以对音频数据进行帧分离了。

----

#计算时长Duration

获取时长的最佳方法是从ID3信息中去读取，那样是最准确的。如果ID3信息中没有存，那就依赖于文件头中的信息去计算了。

计算duration的公式如下：

```
double duration = (audioDataByteCount * 8) / bitRate
```

音频数据的字节总量audioDataByteCount可以通过`kAudioFileStreamProperty_AudioDataByteCount`获取，码率bitRate可以通过`kAudioFileStreamProperty_BitRate`获取也可以通过Parse一部分数据后计算平均码率来得到。

对于CBR数据来说用这样的计算方法的duration会比较准确，对于VBR数据就不好说了。所以对于VBR数据来说，最好是能够从ID3信息中获取到duration，获取不到再想办法通过计算平均码率的途径来计算duration。

----

#分离音频帧

读取格式信息完成之后继续调用`AudioFileStreamParseBytes`方法可以对帧进行分离，并同步的进入`AudioFileStream_PacketsProc`回调方法。

![](/images/iOS-audio/audiofilestreamParse-2.jpg)

回调的定义：

```objc
typedef void (*AudioFileStream_PacketsProc)(void * inClientData,
											  UInt32 inNumberBytes,
											  UInt32 inNumberPackets,
											  const void * inInputData,
											  AudioStreamPacketDescription * inPacketDescriptions);
```
第一个参数，一如既往的上下文对象；

第二个参数，本次处理的数据大小；

第三个参数，本次总共处理了多少帧（即代码里的Packet）；

第四个参数，本次处理的所有数据；

第五个参数，`AudioStreamPacketDescription`数组，存储了每一帧数据是从第几个字节开始的，这一帧总共多少字节。

```objc
//AudioStreamPacketDescription结构
//这里的mVariableFramesInPacket是指实际的数据帧只有VBR的数据才能用到（像MP3这样的压缩数据一个帧里会有好几个数据帧）
struct  AudioStreamPacketDescription
{
    SInt64  mStartOffset;
    UInt32  mVariableFramesInPacket;
    UInt32  mDataByteSize;
};
```

下面是我按照自己的理解实现的回调方法片段：

```
static void MyAudioFileStreamPacketsCallBack(void *inClientData,
                                             UInt32 inNumberBytes,
                                             UInt32 inNumberPackets,
                                             const void *inInputData,
                                             AudioStreamPacketDescription	*inPacketDescriptions)
{
    //处理discontinuous..
    
    if (numberOfBytes == 0 || numberOfPackets == 0)
    {
        return;
    }
    
    BOOL deletePackDesc = NO;
    if (packetDescriptioins == NULL)
    {
        //如果packetDescriptioins不存在，就按照CBR处理，平均每一帧的数据后生成packetDescriptioins
        deletePackDesc = YES;
        UInt32 packetSize = numberOfBytes / numberOfPackets;
        packetDescriptioins = (AudioStreamPacketDescription *)malloc(sizeof(AudioStreamPacketDescription) * numberOfPackets);
        
        for (int i = 0; i < numberOfPackets; i++)
        {
            UInt32 packetOffset = packetSize * i;
            descriptions[i].mStartOffset = packetOffset;
            descriptions[i].mVariableFramesInPacket = 0;
            if (i == numberOfPackets - 1)
            {
                packetDescriptioins[i].mDataByteSize = numberOfBytes - packetOffset;
            }
            else
            {
                packetDescriptioins[i].mDataByteSize = packetSize;
            }
        }
    }
    
    for (int i = 0; i < numberOfPackets; ++i)
    {
        SInt64 packetOffset = packetDescriptioins[i].mStartOffset;
        UInt32 packetSize   = packetDescriptioins[i].mDataByteSize;
        
        //把解析出来的帧数据放进自己的buffer中
        ...
    }
    
    if (deletePackDesc)
    {
        free(packetDescriptioins);
    }
}
```
inPacketDescriptions这个字段为空时需要按CBR的数据处理。但其实在解析CBR数据时inPacketDescriptions一般也会有返回，因为即使是CBR数据帧的大小也不是恒定不变的，例如CBR的MP3会在每一帧的数据后放1 byte的填充位，这个填充位也并非时时刻刻存在，所以帧的大小会有1 byte的浮动。（比如采样率44.1KHZ，码率160kbps的CBR MP3文件每一帧的大小在522字节和523字节浮动。所以不能因为有inPacketDescriptions没有返回NULL而判定音频数据就是VBR编码的）。

----

#Seek

就音频的角度来seek功能描述为“我要拖到xx分xx秒”，而实际操作时我们需要操作的是文件，所以我们需要知道的是“我要拖到xx分xx秒”这个操作对应到文件上是要从第几个字节开始读取音频数据。

对于原始的PCM数据来说每一个PCM帧都是固定长度的，对应的播放时长也是固定的，但一旦转换成压缩后的音频数据就会因为编码形式的不同而不同了。对于CBR而言每个帧中所包含的PCM数据帧是恒定的，所以每一帧对应的播放时长也是恒定的；而VBR则不同，为了保证数据最优并且文件大小最小，VBR的每一帧中所包含的PCM数据帧是不固定的，这就导致在流播放的情况下VBR的数据想要做seek并不容易。这里我们也只讨论CBR下的seek。

CBR数据的seek一般是这样实现的（参考并修改自[matt的blog](http://www.cocoawithlove.com/2010/03/streaming-mp3aac-audio-again.html)）：

1、近似地计算应该seek到哪个字节

```objc
double seekToTime = ...; //需要seek到哪个时间，秒为单位
UInt64 audioDataByteCount = ...; //通过kAudioFileStreamProperty_AudioDataByteCount获取的值
SInt64 dataOffset = ...; //通过kAudioFileStreamProperty_DataOffset获取的值
double durtion = ...; //通过公式(AudioDataByteCount * 8) / BitRate计算得到的时长

//近似seekOffset = 数据偏移 + seekToTime对应的近似字节数
SInt64 approximateSeekOffset = dataOffset + (seekToTime / duration) * audioDataByteCount;
```

2、计算seekToTime对应的是第几个帧（Packet）

我们可以利用之前Parse得到的音频格式信息来计算PacketDuration。_audioItem.fileFormat.mFramesPerPacket / _audioItem.fileFormat.mSampleRate;

```objc
//首先需要计算每个packet对应的时长
AudioStreamBasicDescription asbd = ...; ////通过kAudioFileStreamProperty_DataFormat或者kAudioFileStreamProperty_FormatList获取的值
double packetDuration = asbd.mFramesPerPacket / asbd.mSampleRate

//然后计算packet位置
SInt64 seekToPacket = floor(seekToTime / packetDuration);
```

3、使用`AudioFileStreamSeek`计算精确的字节偏移和时间

`AudioFileStreamSeek`可以用来寻找某一个帧（Packet）对应的字节偏移（byte offset）：

* 如果ioFlags里有kAudioFileStreamSeekFlag_OffsetIsEstimated说明给出的outDataByteOffset是估算的，并不准确，那么还是应该用第1步计算出来的approximateSeekOffset来做seek；
* 如果ioFlags里没有kAudioFileStreamSeekFlag_OffsetIsEstimated说明给出了准确的outDataByteOffset，就是输入的seekToPacket对应的字节偏移量，我们可以根据outDataByteOffset来计算出精确的seekOffset和seekToTime；

```
SInt64 seekByteOffset;
UInt32 ioFlags = 0;
SInt64 outDataByteOffset;
OSStatus status = AudioFileStreamSeek(audioFileStreamID, seekToPacket, &outDataByteOffset, &ioFlags);
if (status == noErr && !(ioFlags & kAudioFileStreamSeekFlag_OffsetIsEstimated))
{
	//如果AudioFileStreamSeek方法找到了准确的帧字节偏移，需要修正一下时间
	seekToTime -= ((approximateSeekOffset - dataOffset) - outDataByteOffset) * 8.0 / bitRate;
	seekByteOffset = outDataByteOffset + dataOffset;
}
else
{
	seekByteOffset = approximateSeekOffset;
}
```
4、按照seekByteOffset读取对应的数据继续使用`AudioFileStreamParseByte`进行解析

如果是网络流可以通过设置range头来获取字节，本地文件的话直接seek就好了。调用`AudioFileStreamParseByte`时注意刚seek完第一次Parse数据需要加参数`kAudioFileStreamParseFlag_Discontinuity`。

----

#关闭AudioFileStream

`AudioFileStream`使用完毕后需要调用`AudioFileStreamClose`进行关闭，没啥特别需要注意的。

```objc
extern OSStatus AudioFileStreamClose(AudioFileStreamID inAudioFileStream);	
```

----

#小结

本篇关于`AudioFileStream`做了详细介绍，小结一下：

* 使用`AudioFileStream`首先需要调用`AudioFileStreamOpen`，需要注意的是尽量提供inFileTypeHint参数帮助`AudioFileStream`解析数据，调用完成后记录`AudioFileStreamID`；

* 当有数据时调用`AudioFileStreamParseBytes`进行解析，每一次解析都需要注意返回值，返回值一旦出现noErr以外的值就代表Parse出错，其中`kAudioFileStreamError_NotOptimized`代表该文件缺少头信息或者其头信息在文件尾部不适合流播放；

* 使用`AudioFileStreamParseBytes`需要注意第四个参数在需要合适的时候传入`kAudioFileStreamParseFlag_Discontinuity`；

* 调用`AudioFileStreamParseBytes`后会首先同步进入`AudioFileStream_PropertyListenerProc`回调来解析文件格式信息，如果回调得到`kAudioFileStreamProperty_ReadyToProducePackets`表示解析格式信息完成；

* 解析格式信息完成后继续调用`AudioFileStreamParseBytes`会进入`MyAudioFileStreamPacketsCallBack`回调来分离音频帧，在回调中应该将分离出来的帧信息保存到自己的buffer中

* seek时需要先近似的计算seekTime对应的seekByteOffset，然后利用`AudioFileStreamSeek`计算精确的offset，如果能得到精确的offset就修正一下seektime，如果无法得到精确的offset就用之前的近似结果

* `AudioFileStream`使用完毕后需要调用`AudioFileStreamClose`进行关闭；

----

#示例代码

[AudioStreamer](https://github.com/mattgallagher/AudioStreamer)和[FreeStreamer](https://github.com/muhku/FreeStreamer)这两个优秀的开源播放器都用到`AudioFileStream`大家可以借鉴。

<div class="github-card" data-github="mattgallagher/AudioStreamer" data-width="400" data-height="" data-theme="default"></div>
<script src="//cdn.jsdelivr.net/github-cards/latest/widget.js"></script>

<div class="github-card" data-github="muhku/FreeStreamer" data-width="400" data-height="" data-theme="default"></div>
<script src="//cdn.jsdelivr.net/github-cards/latest/widget.js"></script>


我自己也写了一个[简单的AudioFileStream封装](https://github.com/msching/MCAudioFileStream)。

<div class="github-card" data-github="msching/MCAudioFileStream" data-width="400" data-height="" data-theme="default"></div>
<script src="//cdn.jsdelivr.net/github-cards/latest/widget.js"></script>

----

#下篇预告

下一篇将讲述如何使用`AudioFile`。

----

#参考资料

[Using Audio](https://developer.apple.com/library/ios/documentation/audiovideo/conceptual/multimediapg/usingaudio/usingaudio.html#//apple_ref/doc/uid/TP40009767-CH2-SW28)

[Streaming and playing an MP3 stream](http://www.cocoawithlove.com/2008/09/streaming-and-playing-live-mp3-stream.html)

[Streaming MP3/AAC audio again](http://www.cocoawithlove.com/2010/03/streaming-mp3aac-audio-again.html)
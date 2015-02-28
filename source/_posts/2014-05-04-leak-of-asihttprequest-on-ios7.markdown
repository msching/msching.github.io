---
layout: post
title: "ASIHTTPRequest iOS7下内存泄漏问题解决记录"
subtitle: "leak of ASIHTTPRequest on iOS7"
date: 2014-05-04 14:43:40 +0800
comments: true
statement: true
categories: [iOS,ASIHTTPRequest]
---

----------

###这是2013年下半年解决iOS7下ASIHTTPRequest内存泄露时所做的记录，现在搬运过来了。

###现在这个修复方法已经被merge到ASIHTTPRequest的主分支上，经过测试可以通过apple的审核，大家可以直接从主分支fork并使用了。

#发现问题

iOS7发布后，我们对产品进行了iOS7的适配。适配完成之后的某天，我使用Leaks对产品的新版本进行内存泄漏检测时发现ASIHTTPRequest存在内存泄漏问题，当时使用的设备是iTouch5，系统为iOS7.0.2。

**Leaks检测结果**

![Leaks](http://ww3.sinaimg.cn/large/7d97a68cgw1eb8uxr5ui3j20sl068acm.jpg)

(ps:使用的是ASIHTTPRequest iPhoneSample的检测图，结果是一样的)


发现之初，我以为是某处ASIHTTPRequest使用不当导致的泄漏，于是把leaks中的堆栈全部都检查了一边，但没有发现任何产品工程中的代码（其中一处泄漏的堆栈如图）。


**Leaks中StackTrace结果**

![StackTrace](http://ww2.sinaimg.cn/large/7d97a68cgw1eb8uxtkz79j209s0gnmz2.jpg)

由于在iOS7发布之前的所有版本中并未看到类似的内存泄漏，所以我就开始怀疑是ASIHTTPRequest在iOS7才产生的。于是我在iOS5和iOS6的设备上进行了Leak Profile，结果没有发现任何泄漏。

对于这样的结果我仍然不是很确信，因为项目的需要我们对ASIHTTPRequest进行了一定的定制，修改了其中一部分代码。为了确定问题确实是出在ASIHTTPRequest上，我去github上翻出了ASIHTTPRequest的repo，pull了最新的代码，用Leaks在iOS7系统上进行了profile。在profile过程中我对iPhone Sample中的每个Tab以此进行了测试，结果在**Synchronous**和**Queue**上并没有发现内存泄漏，在**Upload**上发现了和之前一样泄漏。随后在iOS5和iOS6上也进行了一样的测试，结果依然是没有任何泄漏。

自此确定了这是ASIHTTPRequest在iOS7下特有的内存泄漏，并且只会出现在有POST body的情况下。

<!--more-->

----------
#寻找解答

发现问题之后，我仔细查看了Leaks Profile的结果，发现泄漏集中在`ASIInputStream`上，应该是在使用的过程中`release`方法在某种情况下没有被调用到。于是我重写了ASIIputStream的release方法并在其中断点，分别在iOS7和iOS6进行调试后发现iOS7比iOS6少了一次Release的调用，堆栈如图所示。

**缺少的Release的断点Stack Trace结果**

![Release](http://ww3.sinaimg.cn/large/7d97a68cgw1eb8uxssluaj20dd0bsab3.jpg)

从堆栈来看似乎是 `CoreReadStreamFromCFReadStream` 这个类的析构函数在iOS7下没有被调用到。当时就感觉没救了，这是私有类，想要强行触发析构似乎是不可能的，能做的就是尽量减少其中的泄漏。于是我们做了如下修改：

```objective-c
- (void)close
{
    [stream close];
    [stream release];
    stream = nil;
}
```

修改了`ASIInputStream`的`close`方法，在close完成后把其中的`NSInputStream`对象release掉，以减少内存泄漏，同时保证ASIHTTPRequest不被重复使用（因为其中的NSInputStream已经被release了无法再使用）。

这样一来泄漏有了一定的减少，如图。

**修改后的Leaks检测结果**

![Leakss](http://ww2.sinaimg.cn/large/7d97a68cgw1eb8uxs8d4kj20su023q3r.jpg)

但这样并不能真正解决问题，泄漏依然存在，但我一时也想不到很好的办法，于是只能给repo发issue期望能够得到原作者的回复（虽然我知道希望不大，这哥们很久没管这事了- -）。
这是我发的issue：<https://github.com/pokeb/asi-http-request/issues/378>


----------
#解决问题

发issue大约一周后的某一天收到github的邮件，说有人回复我的issue了，进去一看有一个好心人这样解答道：     

    @mjohnson12

    The leak is because ASIInputStream is being cast to a CFReadStreamRef but ASIInputStream does not derive from NSInputStream it just wraps it.
    
    My Solution is to get rid of ASIInputStream and create a NSInputStream instead in the ASIHTTPRequest startRequest: method.
    
    It breaks using the metrics that ASIInputStream records but I wasn't using them.
    
    I'm using a fairly old version of ASIHTTPRequest v.1.6.2 so your milage may vary.

于是我按照他的做法，把ASIHTTPRequest里的`ASIInputStream`全部替换成了`NSInputStream`，再用Leaks Profile的时候泄漏果真消失了。也正如这位仁兄所说的，`ASIInputStream`只是把自己伪装成一个`NSInputStream`，并且实现了一些`NSInputStream`的接口，实际都是由其中包含的`NSInputStream`实例完成的。`ASIInputStream`这个类的功能主要是用来做流量限制，如果不需要这个功能的话，直接把`ASIInputStream`替换成`NSInputStream`即可解决问题。

那么如果我把`ASIInputStream`继承自`NSInputStream`的话是不是就能既保留流量限制功能又解决泄漏问题了呢？于是我开始尝试继承`NSInputStream`，其中碰到了一些困难。NSInputStream的init方法都是写在一个Category里的，无法被继承- -！。

```objective-c
@interface NSInputStream (NSInputStreamExtensions)
- (id)initWithData:(NSData *)data;
- (id)initWithFileAtPath:(NSString *)path;
- (id)initWithURL:(NSURL *)url NS_AVAILABLE(10_6, 4_0);

+ (id)inputStreamWithData:(NSData *)data;
+ (id)inputStreamWithFileAtPath:(NSString *)path;
+ (id)inputStreamWithURL:(NSURL *)url NS_AVAILABLE(10_6, 4_0);
@end
```

经过一番google我在git上发现了一个repo：<https://github.com/bjhomer/HSCountingInputStream>

这个repo中实现了对于`NSInputStream`的继承。仔细阅读完成后发现其实所谓的继承也只不过时在类的@interface中声明了一下继承自`NSInputStream`而已，实际的工作还是由类中的一个`NSInputStream`实例完成的，但相比于`ASIInputStream`这个repo里的实现多了几个方法：

```objective-c
//私有的CFRunLoopRef schedule、unschedule方法
- (void)_scheduleInCFRunLoop:(CFRunLoopRef)aRunLoop forMode:(CFStringRef)aMode;
- (BOOL)_setCFClientFlags:(CFOptionFlags)inFlags
                 callback:(CFReadStreamClientCallBack)inCallback
                  context:(CFStreamClientContext *)inContext;
- (void)_unscheduleFromCFRunLoop:(CFRunLoopRef)aRunLoop forMode:(CFStringRef)aMode;

//NSInputStream的代理方法
- (void)stream:(NSStream *)aStream handleEvent:(NSStreamEvent)eventCode; 
```

这些方法的作用大家可以参考链接：<http://blog.octiplex.com/2011/06/how-to-implement-a-corefoundation-toll-free-bridged-nsinputstream-subclass/>。

简单的说就是`NSInputStream`需要同时支持`NSRunLoop`和`CFRunLoopRef`的schedule和unschedule方法，那几个私有方法就是负责`CFRunLoopRef`的schedule，NSInputStream的代理方法则是负责事件的传递。这使我联想到了之前提到的没有被调用的`release`方法，iOS6下它的堆栈中正好有`CFRunLoopRef`的一些方法。莫非这就是`ASIInputStream`内存泄漏的原因所在？

我当时的猜测是，iOS7下的内存泄漏是`ASIInputStream`伪装的不够像而导致的，之所以这么认为因为虽然有泄漏但`ASIInputStream`的功能依然存在，HTTP请求并未因此失效，这说明`ASIInputStream`还是被bridge成了`CFReadStream`，但只是因为少了和`NSInputStream`一样的unschedule方法导致其没有被正常unschedule。如果我把`ASIInputStream`的unschedule方法补上是否就可以解决问题？

我把上述的几个方法在`ASIInputStream`中实现了以后再进行Leaks Profile，内存泄漏果然如预期的那样消失了！在`ASIInputStream`的`release`方法中打断点后发现也能够正常调用了。问题到这里算是解决了。接下来要解决这些私有方法调用可能会碰到的审核不通过问题，正好上面那篇文章里提供了思路，用runtime把三个私有方法重定向到自定义的方法上（方法名只是把"_"去掉了）。

```objective-c
+ (BOOL) resolveInstanceMethod:(SEL) selector
{
    NSString * name = NSStringFromSelector(selector);

    if ( [name hasPrefix:@"_"] )
    {
        name = [name substringFromIndex:1];
        SEL aSelector = NSSelectorFromString(name);
        Method method = class_getInstanceMethod(self, aSelector);

        if ( method )
        {
            class_addMethod(self,
                            selector,
                            method_getImplementation(method),
                            method_getTypeEncoding(method));
            return YES;
        }
    }
    return [super resolveInstanceMethod:selector];
}
```

至此大功告成，问题顺利解决。但这个问题之所以会出现的真正缘由我还是没有弄清楚，Apple在iOS7下究竟做了什么，哪位大神如果知道的话还请告知。。感激不尽 ~_~。


附上修改过后的`ASIInputStream`[代码](https://github.com/OpenFibers/asi-http-request/commit/499a3be1f92d7023e2d2092197dbb71c77cdd330)

----------
#后记

问题没解决的时候其他同事建议我更换成AFNetworking等等其他开源库，因为这些库更新快文档全，ASIHTTPRequest接口复杂、代码繁多而且如今已年久失修无人维护了。确实如此，但由于一些项目上的原因我们无法更换，况且ASIHTTRequest在效率上略好，并且拥有其他基于NSURLConnection的库不具备一些功能。在解决问题的过程中也让我对NSInputStream的工作机制有了更深的理解，可谓一石二鸟。由于鄙人能力有限，其中一些地方可能说的有错误或者有纰漏的话还请大神们指正:)
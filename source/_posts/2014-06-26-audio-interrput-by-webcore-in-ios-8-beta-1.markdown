---
layout: post
title: "iOS8beta1下WebCore可能会打断音频播放"
subtitle: "Audio interrput by WebCore in iOS 8 beta 1"
date: 2014-06-26 16:09:56 +0800
comments: true
statement: true
categories: [iOS,Audio]
---

#问题
前不久在QA发现一个问题，在iOS8 beta1上使用我们的app播放歌曲时进入某些内嵌的web页面（UIWebview实现）时歌曲会暂停播放，但是界面仍然显示为正在播放状态。把真机连上Xcode6调试后发现在进入部分网页时会再console上打印如下log：

	AVAudioSession.mm:623: -[AVAudioSession setActive:withOptions:error:]: Deactivating an audio session that has running I/O. All I/O should be stopped or paused prior to deactivating the audio session.
	
bt后堆栈如下：

```
    frame #1: 0x299632fe libAVFAudio.dylib`-[AVAudioSession setActive:error:] + 26
    frame #2: 0x3551b92e WebCore`WebCore::AudioSession::setActive(bool) + 62
    frame #3: 0x35af2674 WebCore`WebCore::MediaSessionManager::updateSessionState() + 100
    frame #4: 0x35af03b6 WebCore`WebCore::MediaSessionManager::addSession(WebCore::MediaSession&) + 74
    frame #5: 0x35af0002 WebCore`WebCore::MediaSession::MediaSession(WebCore::MediaSessionClient&) + 38
    frame #6: 0x35735a20 WebCore`WebCore::HTMLMediaSession::create(WebCore::MediaSessionClient&) + 20
    frame #7: 0x35724c68 WebCore`WebCore::HTMLMediaElement::HTMLMediaElement(WebCore::QualifiedName const&, WebCore::Document&, bool) + 976
    frame #8: 0x3570ad24 WebCore`WebCore::HTMLAudioElement::create(WebCore::QualifiedName const&, WebCore::Document&, bool) + 36
    frame #9: 0x35718184 WebCore`WebCore::audioConstructor(WebCore::QualifiedName const&, WebCore::Document&, WebCore::HTMLFormElement*, bool) + 56
    frame #10: 0x3571803a WebCore`WebCore::HTMLElementFactory::createElement(WebCore::QualifiedName const&, WebCore::Document&, WebCore::HTMLFormElement*, bool) + 230
    frame #11: 0x3533a26c WebCore`WebCore::HTMLDocument::createElement(WTF::AtomicString const&, int&) + 88
    frame #12: 0x3533a1ae WebCore`WebCore::jsDocumentPrototypeFunctionCreateElement(JSC::ExecState*) + 242
    frame #13: 0x2c1cc4d4 JavaScriptCore`llint_entry + 21380
```

发现是WebCore调用了`AVAudioSession`的setActive方法，并且把active置为了NO。这个过程其实类似于音乐在播放时被其他事件打断（例如电话、siri）一样，audio会被打断，同时会发送`kAudioSessionBeginInterruption`事件通知app音频播放已经被打断，需要修正播放器和UI状态；打断结束后回发送`kAudioSessionEndInterruption`事件通知app恢复播放状态。区别在于WebCore的打断并没有任何通知，所以就导致界面上的播放状态为播放中而实际音乐却被打断。

#适配
接下来就要对这个问题进行适配了：

1. 首先，联系了前段组的同事对出现问题的页面进行检查，之后被告知是某个页面的js中调用了一些播放相关的代码导致了这个问题，这些js是之前版本中使用的，现在已经被废弃但没有及时的删除。在删除这些js后，问题自然就消失了。
2. 客户端本身也应该做一些适配来防止下次再有页面出现类似问题，目前我能想到的办法是做一个`AVAudioSession`的category，method swizzle方法`setActive:withOptions:error:`在设置active值时发送通知来修改UI的状态。
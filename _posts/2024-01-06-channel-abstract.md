---
layout: post
title: webrtc 中 channel 的概览
date: 2024-01-06 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio webrtc channel
---


* content
{:toc}

---


## 0. 前言

**Channel 层**是沟通 **Transport 层**和**媒体引擎层**之间的桥梁，负责 **Transport 层**和**媒体引擎层**之间的数据收发。

1. WebRTC的每路数据的传输流程都封装成一个Channel对象。
2. WebRTC中有很多Channel，最顶层的Channel，实际上就是BaseChannel，其派生类可以是VoiceChannel，VideoChannel或者DataChannel。这些BaseChannel都有一个共同的特点，就是每个都有MediaChannel，我们称之为媒体通道。
3. ChannelManager管理这些Channel。ChannelManager会创建MediaChannel，并创建BaseChannel，将MediaChannel挂在相应的BaseChannel上。ChannelManager是怎样创建MediaChannel的呢？实际上是通过其成员`media_engine_`创建的。这个media_engine_就是MediaEngineInterface，也就是媒体引擎接口，这个接口的最终目的就是创建MediaChannel。可见MediaChannel是由MediaEngine创建。
4. 实现MediaEngineInterface的，是CompositeMediaEngine，这是一个组合的MediaEngine，是由VideoEngineInterface和VoiceEngineInterface构成的。实现VideoEngineInterface的，是WebrtcVideoEngine。
5. 对于MediaChannel而言，其实现类是VideoMediaChannel，最终的实现类是WebrtcVideoChannel。
6. MediaEngine创建MediaChannel，对于video而言，就是WebrtcVideoEngine创建WebrtcVideoChannel。对于Audio，就是WebrtcVoiceEngine创建WebrtcVoiceChannel。BaseChannel有MediaChannel，BaseChannel是由ChannelManager创建。
7. !!! 接收数据流，首先是BaseChannel实现了RtpPacketSinkInterface接口，因此Rtp数据，会通过接口的OnRtpPacket流入到BaseChannel中，BaseChannel再将数据送给MediaChannel，因此最终数据到达了WebrtcVideoChannel的OnPacketReceived方法中。
8. !!!发送数据流，首先是BaseChannel实现了MediaChannel::NetworkInterface接口，会通过MediaChannel再将数据送给BaseChannel，因此最终数据到达了RtpTransportInternal的方法中。



## 1. Channel 类结构

### 1.1 类图

![channel]({{ site.url }}{{ site.baseurl }}/images/channel-abstract.assets/channel-class.webp)



![]({{ site.url }}{{ site.baseurl }}/images/channel-abstract.assets/channel-class2.jpeg)

### 1.2 BaseChannel

- BaseChannel 有三个实现类 VoiceChannel、VideoChannel、RtpDataChannel，分别对应了音频、视频、数据。
- BaseChannel 实现的 MediaChannel::NetworkInterface 接口，是用来接收 MediaChannel 发送的数据。
- BaseChannel 实现的 RtpPacketSinkInterface 接口，就用来就收远端发送过来的数据。
- BaseChannel 有个属性，存放的就是MediaChannel。

Channel部分暴露给外部的操作接口还是ChannelManager类中管理的BaseChannel及其派生类，通过这些类，外部模块可以设置音视频的采集源（如VideoCapturer）、为网络发送过来的音视频数据指定渲染器（如AudioRenderer/VideoRenderer），这些类对MediaChannel及其派生类的基础上再包装了一层，如图所示，BaseChannel实现MediaChannel::NetworkInterface接口完成封装好的RTP/RTCP数据包的发送操作，具体纯数据的网络发送请求最终委托给RtpTransportInternal对象。

BaseChannel的主要职责是发送数据，那么数据究竟是怎么产生的呢？我们知道，无论是音频数据，还是视频数据，都要经过一系列的流程才可以产生，比如采集，编码，分包。这个过程显然并不是BaseChannel的职责，专门有对象负责这些事情，就是MediaChannel，也就是媒体通道。由此可见，Webrtc中的通道很多，但是实际上是各司其职，每个通道都由各自的用途。那么MediaChannel和BaseChannel之间的关系是什么样呢？**实际上也是has-a的关系，即每个BaseChannel中都有一个MediaChannel。**
**MediaChannel及其派生类封装了待传输的编解码、RTP/RTCP封包解包等逻辑**，具体对象由相应的Media Engine类创建。



### 1.3 MediaChannel

- MediaChannel 有三个实现类 VoiceMediaChannel、VideoMediaChannel、DataMediaChannel，分别对应了音频、视频、数据。MediaChannel 可以说是音频、视频引擎的门面。负责流对象的创建、销毁，从 BaseChannel 接收数据输送给引擎，做相关接收处理，比如进入 jitter buffer，解码，渲染等。然后从 RTPSender 接收数据，发送给 BaseChannel。

- WebRtcVoiceMediaChannel 和 WebRtcVideoChannel 实现了 webrtc::Transport 接口，用于接收 RTPSender 发送的数据。

  ```
  media\engine\webrtc_voice_engine.h
  media\engine\webrtc_video_engine.h
  ```

  WebRtcVoiceMediaChannel 实现了 Webrtc::Transport 的接口。 在函数 WebRtcVoiceMediaChannel::AddSendStream 中，将 WebRtcVoiceMediaChannel 注册到 RTPSender，接收 RTPSender 发送的数据。



#### ??? WebRtcVideoChannel

![]({{ site.url }}{{ site.baseurl }}/images/channel-abstract.assets/webrtc-video-channel.png)

当调用WebRtcVideoChannel2的AddSendStream方法时，会创建一个WebRtcVideoSendStream对象，同样，调用AddRecvStream成员方法，会创建一个WebRtcVideoReceiveStream对象。

当外部调用WebRtcVideoChannel2的SetCapturer方法时，会转给WebRtcVideoSendStream来响应，WebRtcVideoSendStream内部将InputFrame成员方法挂接VideoCapturer的SignalVideoFrame信号来接收视频采集器传输过来的视频帧数据。

WebRtcVideoChannel2的AddSendStream和SetCapturer的调用时机这里暂时不考虑，这些涉及到网络连接，等每个节点的内容分析完后，再探讨整个流程。

如图所示，WebRtcVideoSendStream和WebRtcVideoReceiveStream也只是个包装类，内部依赖Call接口创建对应的VideoSendStream接口实现类和VideoReceiveStream接口实现类。在internal命名空间内，分别有一个Call类、VideoSendStream类、VideoReceiveStream类来实现这三个接口，Call类创建关键的VideoEngine对象来管理视频数据发送过程中的一系列处理逻辑。

https://blog.csdn.net/yinshipin007/article/details/125769242



### 1.4 ChannelManager

ChannelManager，就是控制Channel的，究竟是哪个channel呢？就是上面介绍的BaseChannel。再具体一点，实际上上述的三个channel，都是由channelmanager创建出来的。从对象图中我们可以看到这种关系，WebRtcSession有一个成员变量（has-a）ChannelManager，而ChannelManager创建了（create）三种Channel（VoiceChannel，VIdeoChannel以及DataChannel）



## 参考

[[WebRTC 架构分析\] WebRTC 中的 Channel 层分析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/476889105)

[Webrtc中的各种Channel](https://www.jianshu.com/p/d6030e7aa282)

[WebRTC手记Channel概念、WebRtcVideoEngine2模块_webrtcvideochannel_音视频开发老马的博客-CSDN博客](https://blog.csdn.net/yinshipin007/article/details/125769242)

[webrtc发送端-Channel的宏观架构](https://www.jianshu.com/p/326a1c64385a)



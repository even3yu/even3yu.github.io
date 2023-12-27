---
layout: post
title: audio pipeline
date: 2023-12-18 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio
---


* content
{:toc}

---


## 0. Audio Pipeline

![Audio Pipeline]({{ site.url }}{{ site.baseurl }}/images/audio-pipeline.assets/pipeline.png)



### 0.1 结构图

![这里写图片描述]({{ site.url }}{{ site.baseurl }}/images/audio-pipeline.assets/abstract.png)



### 0.2 类图

![WebRTC 系列之音频的那些事]({{ site.url }}{{ site.baseurl }}/images/audio-pipeline.assets/class-1.png)

![img]({{ site.url }}{{ site.baseurl }}/images/audio-pipeline.assets/class-2.png)

### 0.3 发送端调用堆栈

![img]({{ site.url }}{{ site.baseurl }}/images/audio-pipeline.assets/audio-send-pipeline.png)

上图是音频数据发送的核心流程，主要是核心函数的调用及线程的切换。PCM 数据从硬件设备中被采集出来，在采集线程做些简单的数据封装会很快进入 APM 模块做相应的 3A 处理，从流程上看 APM 模块很靠近原始 PCM 数据，这一点对 APM 的处理效果有非常大的帮助，感兴趣的同学可以深入研究下 APM 相关的知识。之后数据就会被封装成一个 Task，投递到一个叫 rtp_send_controller 的线程中，到此采集线程的工作就完成了，采集线程也能尽快开始下一轮数据的读取，这样能最大限度的减小对采集的影响，尽快读取新的 PCM 数据，防止 PCM 数据丢失或带来不必要的延时。

接着数据就到了 rtp_send_controller 线程，rtp_send_controller 线程的在此的作用主要有三个，一是做 rtp 发送的拥塞控制，二是做 PCM 数据的编码，三是将编码后的数据打包成 RtpPacketToSend（RtpPacket）格式。最终的 RtpPacket 数据会被投递到一个叫 RoundRobinPacketQueue 的队列中，至此 rtp_send_controller 线程的工作完成。

后面的 RtpPacket 数据将会在 SendControllerThread 中被处理，SendControllerThread 主要用于发送状态及窗口拥塞的控制，最后数据通过消息的形式（type: MSG_SEND_RTP_PACKET）发送到 Webrtc 三大线程之一的网络线程（Network Thread），再往后就是发送给网络。到此整个发送过程结束。


### 0.4 播放端调用堆栈

![img]({{ site.url }}{{ site.baseurl }}/images/audio-pipeline.assets/audio-recv-pipeline.png)

上图是音频数据接收及播放的核心流程。网络线程（Network Thread）负责从网络接收 RTP 数据，随后异步给工作线程（Work Thread）进行解包及分发。如果接收多路音频，那么就有多个 ChannelReceive，每个的处理流程都一样，最后未解码的音频数据存放在 NetEq 模块的 packet_buffer_ 中。与此同时播放设备线程不断的从当前所有音频 ChannelReceive 获取音频数据（10ms 长度），进而触发 NetEq 请求解码器进行音频解码。对于音频解码，WebRTC 提供了统一的接口，具体的解码器只需要实现相应的接口即可，比如 WebRTC 默认的音频解码器 opus 就是如此。当遍历并解码完所有 ChannelReceive 中的数据，后面就是通过 AudioMixer 混音，混完后交给 APM 模块处理，处理完最后是给设备播放。 


## 1. 音频采集

1. [webrtc 支持外部语音输入（虚拟设备）](https://even3yu.github.io/2023/07/26/virtual-audio-device/)
2. [音频采集](https://even3yu.github.io/2023/12/19/audio-capture/)

![img]({{ site.url }}{{ site.baseurl }}/images/audio-pipeline.assets/start-stop-device.png)

## 2. AudioDevcieModule

1. [AudioDeviceModule](https://even3yu.github.io/2023/07/18/audiodevice/)

## 3. Buffer

### 3.1 FileAudioBuffer

### 3.2 AudioDevcieBuffer



## 4. AudioTransport

```less
AudioTransport
AudioTransportImpl
```





### 4.1 音频数据二次处理或者本地录制

数据回调给app层，做语音二次处理

```less
RecordedDataIsAvailable
NeedMorePlayData
PullRenderData
```



```less
	// 回调采集数据
  // register processed recording sink
  virtual void AddSink(AudioTrackSinkInterface* sink) = 0;

  // 回调用于播放的数据
  // register Mix playout audio sink
  virtual void RegisterAudioPlayoutSink(AudioTrackSinkInterface* sink) {}
```



### 4.2 AudioProcessing

3A处理



## 5. 音频编码

### 5.1 webrtc::Call

### 5.2 AudioSendStream



## 6. 数据发送

### 6.1 RTP/RTCP



## --------



## 7. 数据接收

### 7.1 RTP/RTCP



## 8. 音频解码

### 8.1 webrtc::Call

### 8.2 AudioReceiveStream



## 9. AudioTransport

### 9.1 AudioMixer

## 10. 音频播放





## 参考

[WebRTC音频](https://blog.csdn.net/u010657219/article/details/54931154)

[[WebRTC架构分析]本地音频数据录制和播放](https://zhuanlan.zhihu.com/p/131380603)

[WebRTC 系列之音频的那些事](https://worktile.com/kb/p/5925)

[WebRTC ADM 源码流程分析](https://blog.csdn.net/netease_im/article/details/123088666)


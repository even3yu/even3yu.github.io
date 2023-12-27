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

## 1. 音频采集

1. [webrtc 支持外部语音输入（虚拟设备）](https://even3yu.github.io/2023/07/26/virtual-audio-device/)
2. 

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


---
layout: post
title: 音频完整发送到接收，以及3A处理过程
date: 2023-12-21 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio
---


* content
{:toc}

---



![]({{ site.url }}{{ site.baseurl }}/images/audio-flow.assets/audio-pipeline.png)

1. 发起端通过麦克风进行声音采集
2. 发起端将采集到的声音信号输送给APM模块，进行回声消除AEC，噪音抑制NS，自动增益控制处理AGC
3. 发起端将处理之后的数据输送给编码器进行语音压缩编码
4. 发起端将编码后的数据通过RtpRtcp传输模块发送，通过Internet网路传输到接收端
5. 接收端接受网络传输过来的音频数据，先输送给NetEQ模块进行抖动消除，丢包隐藏解码等操作
6. 接收端将处理过后的音频数据送入声卡设备进行播放



## 参考

[WebRTC系列之音频的那些事_webrtc 音频通道_网易智企的博客-CSDN博客](https://blog.csdn.net/netease_im/article/details/107048860)


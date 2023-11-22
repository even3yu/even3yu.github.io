---
layout: post
title: h264 profile and level
date: 2023-11-07 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

## 1. sdp 媒体协商

在 sdp 媒体协商中，有如下示例(来自 webrtc)：

```c
a=rtpmap:125 H264/90000
a=fmtp:125 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
123
```

如上，packetization-mode=1，profile=42，level=e0，id=1f(固定，不做分析)。下面主要分析 packetization-mode、profile 和 level 三个字段。

### 1.1 packetization-mode(协商打包模式)：

rfc6184 定义了 single nalu mode=0; non-interleaved mode=1; interleaved mode=2。一般 sdp 中只支持 packetization-mode=1 的打包模式，如果没有此项，默认支持 Single NAL Unit mode。

### 1.2 !!! profile

一般 sdp 中会出现三种 profile，分别为 42(baseline profile)、4d(main profile)、64(high profile)，参考 wiki 的定义(https://en.wikipedia.org/wiki/Advanced_Video_Coding)：

BP:  baseline profile，基本画质。支持I/P 帧，只支持无交错（Progressive）和CAVLC；
MP: main profile，主流画质。提供I/P/B 帧，支持无交错（Progressive）和交错（Interlaced）， 也支持CAVLC 和CABAC 的支持；
HiP: high profile，高级画质。在main Profile 的基础上增加了8x8内部预测、自定义量化、 无损视频编码和更多的YUV 格式；
~~Extended profile：进阶画质。支持I/P/B/SP/SI 帧，只支持无交错（Progressive）和CAVLC；(用的少)~~

H.264 Baseline profile、Extended profile和Main profile都是针对8位样本数据、4:2:0格式(YUV)的视频序列。在相同配置情况下，High profile（HP）可以比Main profile（MP）降低10%的码率。

![img]({{ site.url }}{{ site.baseurl }}/images/sdp-about-profile-level-id.assets/profile.png)
可以看到，BP， MP，HIP 三列，对应横向的Chroma formats，三种 profile 都只支持 yuv420 一种源图像格式。baseline profile 不支持 B 帧，其它两种支持 B 帧。
**但是实际项目中，即使协商的 profile 非 baseline，webrtc 等实时音视频系统也不会出现 B 帧，因为 B 帧的解码依赖于前后帧，解码顺序与显示顺序不一致，会造成显示延迟。且发生丢包时，B 帧无法解码的概率很大。还有一个项目上的问题是，与自己系统对接的其它音视频系统也不一定支持 B 帧。**



#### 1.2.1 应用场景

- Baseline profile多应用于实时通信领域，
- Main profile多应用于流媒体领域，
- High profile则多应用于广电和存储领域。

从压缩比例来说，baseline< main < high ，对于带宽比较局限的在线视频，可能会选择high，但有些时候，做个小视频，希望所有的设备基本都能解码（有些低端设备或早期的设备只能解码baseline），那就牺牲文件大小吧，用baseline.



### 1.3 !!! level

参考 wiki 的定义(https://en.wikipedia.org/wiki/Advanced_Video_Coding)：
![img]({{ site.url }}{{ site.baseurl }}/images/sdp-about-profile-level-id.assets/level.png)
以上是一部分截图，可以看到，level 主要影响的是支持的最高分辨率和帧率，如sdp中(16进制为1f 转为 10进制 31, 对应的就是 3.1行)，最高支持 1,280×720@30.0 的视频。



## 2. ffmpeg如何控制profile&level

```bash
ffmpeg -i input.mp4 -profile:v baseline -level 3.0 output.mp4 ffmpeg -i input.mp4 -profile:v main -level 4.2 output.mp4
```

## 3. Apple 设备对不同profile的支持。 

![这里写图片描述]({{ site.url }}{{ site.baseurl }}/images/sdp-about-profile-level-id.assets/apple-prfile.png)

## 参考

[关于H.264 profile-level-id](https://blog.csdn.net/u012587637/article/details/108767639)

https://cloud.tencent.com/developer/article/2020442

[H264编码profile & level控制](https://www.cnblogs.com/tinywan/p/6402007.html)
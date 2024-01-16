---
layout: post
title: webrtc msid
date: 2024-01-08 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---


## 1. msid的ABNF语法

```cpp
msid-value = msid-id [ SP msid-appdata ]
msid-id = 1*64token-char ; see RFC 4566
msid-appdata = 1*64token-char  ; see RFC 4566
```



### 1.1 举例

```less
m=audio 61003 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126
a=msid:5jbA0s68fQKkMHiM7ZDVxoVOox1mXyMvC7bR 22ceddc2-d518-4ed4-b9ad-708c87720561
a=ssrc:1293363954 cname:A9rtwcbhBbK7g2ma
a=ssrc:1293363954 msid:5jbA0s68fQKkMHiM7ZDVxoVOox1mXyMvC7bR 22ceddc2-d518-4ed4-b9ad-708c87720561
a=ssrc:1293363954 mslabel:5jbA0s68fQKkMHiM7ZDVxoVOox1mXyMvC7bR
a=ssrc:1293363954 label:22ceddc2-d518-4ed4-b9ad-708c87720561
```

上面这个m=audio的媒体中一个ssrc为`1293363954`的源属于`5jbA0s68fQKkMHiM7ZDVxoVOox1mXyMvC7bR`这个MediaStream的。

1. `5jbA0s68fQKkMHiM7ZDVxoVOox1mXyMvC7bR`就是一个msid-id；就是mslabel；
2. `22ceddc2-d518-4ed4-b9ad-708c87720561`就是msid-appdata；就是label，就是track-id；



## 2. msid作用

- msid就是用于标识不同的MediaStream。所以能简单的推理出来，一个sdp中可以有多个MediaStream;
- sdp中添加msid相关的属性还有一个作用，用于标识MediaStreamTrack和MediaStream的关系的，指定哪些MediaStreamTrack是属于MediaStream。

> **为什么`msid` 将 `MediaStream ID` 和 `MediaStreamTrack ID` 组合在一起？？？**
>
> 在Plan-B中，同类型的MediaStream只能有个一个mline（比如audio，video），同类型的多个MediaStream之间通过msid区分。什么情况下出现多个MediaStream，譬如 发两路视频，两路视频的`MediaStreamTrack ID` 是不同的。
>
> 而在Unifed Plan 是每个MediaStream 都有一个mline，因此两路流就有两个video mline。
>
> msid感觉，更像是历史遗留的设计。



## 3. mslabel lable

Chrome m103 webRTC 将 SDP ssrc 里的 mslabel 和 label 两个属性移除了。

```less
`label` 对应 `MediaStreamTrack ID`，
`mslabel` 对应 `MediaStream ID`，
`msid` 将 `MediaStream ID` 和 `MediaStreamTrack ID` 组合在一起。
```

> - **一个 a=ssrc 代表一个 RTP stream** ；
> - **一个 MediaStreamTrack 通常包含一个或多个 RTP stream**，例如一个视频 MediaStreamTrack 中通常包含两个 RTP stream，一个用于常规传输，一个用于 nack 重传；
> - **一个 MediaStream 通常包含一个或多个 MediaStreamTrack** ，例如 simulcast 场景下，一个 MediaStream 通常会包含三个不同编码质量的 MediaStreamTrack ；或者audio/video 添加到同一个MediaStream；



### 3.1chrome m102 及以前版本

```less
a=ssrc:1269806375 cname:POIsoqWs3fb2wRHA
a=ssrc:1269806375 msid:a5fabc49-5488-4909-90b3-f25c09dd3f38 6120f11b-62a9-4982-985a-fe3a20c7dfae
a=ssrc:1269806375 mslabel:a5fabc49-5488-4909-90b3-f25c09dd3f38
a=ssrc:1269806375 label:6120f11b-62a9-4982-985a-fe3a20c7dfae
```



### 3.2 chrome m103

```less
a=ssrc:1269806375 cname:POIsoqWs3fb2wRHA
a=ssrc:1269806375 msid:a5fabc49-5488-4909-90b3-f25c09dd3f38 6120f11b-62a9-4982-985a-fe3a20c7dfae
```



## 4. 关于MediaStreamTrack 和 MediaStream

- MediaStreamTrack 代表一个媒体源，例如音频，视频。一个MediaStreamTrack可以被包含在0个，1个中或者多个MediaStream中。它并不是和一个ssrc划上等号，它还会由rid源创建。
- MediaStream表示由一组MediaStreamTrack组成，并且包含的MediaStreamTrack在渲染的时候需要做时间同步。**在Unified Plan的设计中，一个MediaStream 对应一个mline。 而 一个MediaStream包含多个MediaStreamTrack，更多是在Plan-B中出现。**
  

## 5. planB sdp

```less
a=group:BUNDLE audio video data
a=msid-semantic: WMS h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 126
a=mid:audio
a=ssrc:18509423 msid:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C 15598a91-caf9-4fff-a28f-3082310b2b7a
m=video 9 UDP/TLS/RTP/SAVPF 100 101 107 116 117 96 97 99 98
a=mid:video
a=ssrc:3463951252 msid:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C ead4b4e9-b650-4ed5-86f8-6f5f5806346d
m=application 9 DTLS/SCTP 5000
a=mid:data
```

1. `a=msid-semantic: WMS h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C` , 是一个全局性的描述，告知了这个会话中存在几个逻辑上的媒体流，WMS表示WebRTC Media Stream。此处只有一个，**流id为**h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C。
2. **mLine的a=msid=xxx的属性表示了该mline所对应地媒体轨道逻辑上所属的流，可以同时属于多个流。示例上，音频轨和视频轨均属于流h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C。其后跟随的是轨道的id。**

## 参考

1. https://yoshihisaonoue.wordpress.com/2019/10/13/msid-attribute-in-sdp-in-webrtc/
2. [mslable, label](https://xie.infoq.cn/article/c80861245a30ec1e72f24f789)
3. [webrtc的sdp中msid介绍](https://blog.csdn.net/lqq_419/article/details/118358598)
4. https://datatracker.ietf.org/doc/html/draft-ietf-mmusic-msid-17#section-2




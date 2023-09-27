---
layout: post
title: webrtc sdp--ssrc
date: 2023-09-27 01:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

## 1. ssrc

SSRC（Synchronization Source）。SSRC是媒体源的唯一标识，每一路媒体流都有一个唯一的SSRC来标识它。早期的 Chrome 产生的 SDP 中每个 SSRC 通常有 4 行，例如：

```js
a=ssrc:3245185839 cname:Cx4i/VTR51etgjT7
a=ssrc:3245185839 msid:cb373bff-0fea-4edb-bc39-e49bb8e8e3b9 0cf7e597-36a2-4480-9796-69bf0955eef5
a=ssrc:3245185839 mslabel:cb373bff-0fea-4edb-bc39-e49bb8e8e3b9
a=ssrc:3245185839 label:0cf7e597-36a2-4480-9796-69bf0955eef5
```

> **为什么ssrc 这么复杂**
>
> 这么繁杂的信息是有历史原因的，cname 是必须的，msid/mslabel/label 这三个属性都是 WebRTC 自创的，或者说 Google 自创的，可以参考 https://tools.ietf.org/html/draft-ietf-mmusic-msid-17，理解它们三者的关系需要先了解三个概念：RTP stream / MediaStreamTrack / MediaStream 。
>
> ------------------------------------------
>
> `label` 对应 `MediaStreamTrack ID`，
> `mslabel` 对应 `MediaStream ID`，
> `msid` 将 `MediaStream ID` 和 `MediaStreamTrack ID` 组合在一起。
>
> ------------------------------------------
>
> - **一个 a=ssrc 代表一个 RTP stream** ；
> - **一个 MediaStreamTrack 通常包含一个或多个 RTP stream**，例如一个视频 MediaStreamTrack 中通常包含两个 RTP stream，一个用于常规传输，一个用于 nack 重传；
> - **一个 MediaStream 通常包含一个或多个 MediaStreamTrack** ，例如 simulcast 场景下，一个 MediaStream 通常会包含三个不同编码质量的 MediaStreamTrack ；
>
> FireFox 其实就只有第一行。

> **cname（canonical name）**
>
> 通常称为别名，可以用在域名解析中。当想为某个域名起一个别名的时候，就可以使用它。如在直播推/拉流中，想将某个云厂商的推流地址push.xxx.yun.xxx.com换成你自己的地址push.advancedu.com，就可以使用cname。cname在SDP中的作用与域名解析中的作用是一致的，就是为媒体流起了一个别名。在同一个视频媒体描述中的两个SSRC后面都跟着同样的cname，说明这两个SSRC属于同一个媒体流。
>
> https://blog.csdn.net/weixin_39766005/article/details/132294422
>
> RTCP 为每个 RTP 用户提供了一个全局唯一的规范名称标识符 CNAME (Canonical Name) ，接收者使用它来追踪一个 RTP stream。当发现冲突或程序重新启动时，RTP 中的 SSRC (同步源标识符) 可能会发生改变，接收者可利用 CNAME 来索引 RTP stream。同时，接收者也需要利用 CNAME 在相关 RTP 连接中多个 RTP stream 之间建立联系。当 RTP 需要进行音视频同步的时候，接受者就需要使用 CNAME 来使得同一发送者的音视频数据相关联，然后根据 RTCP 包中的 NTP (Network Time Protocol) 来实现音频和视频的同步。
>
> https://blog.csdn.net/aggresss/article/details/109850434



### 1.2 如何判断两个不同ssrc，属于同一个Stream

比如一个Stream，可以有一个ssrc 标识原始流，还有一个ssrc 重传包；
两个的ssrc是不一样的，但是如何知道他们是同一个stream的呢？通过cname

```cpp
...
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126
...
a=rtpmap :111 opus /48000/2
a=rtcp -fb:111 transport -cc
a=fmtp :111 minptime =10; useinbandfec =1
a=rtpmap :103 ISAC /16000
a=rtpmap :104 ISAC /32000
a=rtpmap :9 G722 /8000
...
a=ssrc-group:FID 1531262201 2412323032
// 视频流的SSRC
a=ssrc :1531262201 cname:Hmks0 +2 NwywExB+s
…
// 丢包重传流的SSRC
a=ssrc :2412323032 cname:Hmks0 +2 NwywExB+s
```

- SSRC为1531262201的cname名字与SSRC为2412323032的cname名字是一模一样的，说明这两个SSRC属于同一个媒体流。

- ssrc-group指明两个SSRC是有关联关系的，后一个SSRC（2412323032）是前一个SSRC（1531262201）的重传流。也就是说，1531262201是真正代表视频流的SSRC，而2412323032是视频流（1531262201）丢包时使用的SSRC，这就是为什么在同一个媒体描述中会有两个SSRC。所以，虽然在同一个视频描述中有两个SSRC，但对于该视频来说，仍然只有唯一的SSRC用来标识它。



### 1.3 多个视频源，放入同一个Stream中

多个视频源，放入同一个Stream中，也是会不同的ssrc，相同的cname

```js
// 1626168465 trackId
// 1073938433 streamId

// ssrc 2992955434, 第一路视频流
// ssrc 2949075267, rtx
a=ssrc-group:FID 2992955434 2949075267
a=ssrc:2992955434 cname:JKnlVv1bAzdF0Wjh
a=ssrc:2992955434 msid:1073938433 1626168465 ----》streamId+trackId
a=ssrc:2992955434 mslabel:1073938433  ----------》 streamId
a=ssrc:2992955434 label:1626168465   -----------》 trackId

a=ssrc:2949075267 cname:JKnlVv1bAzdF0Wjh
a=ssrc:2949075267 msid:1073938433 1626168465
a=ssrc:2949075267 mslabel:1073938433
a=ssrc:2949075267 label:1626168465

// 913807816 trackId
// 1073938433 streamId

// ssrc 276575604, 第二路视频流
// ssrc 525075894, rtx
a=ssrc-group:FID 276575604 525075894
a=ssrc:276575604 cname:JKnlVv1bAzdF0Wjh
a=ssrc:276575604 msid:1073938433 913807816
a=ssrc:276575604 mslabel:1073938433
a=ssrc:276575604 label:913807816

a=ssrc:525075894 cname:JKnlVv1bAzdF0Wjh
a=ssrc:525075894 msid:1073938433 913807816
a=ssrc:525075894 mslabel:1073938433
a=ssrc:525075894 label:913807816


// 2992955434，2949075267， 276575604，525075894 都是同一个cname = JKnlVv1bAzdF0Wjh
```

两路视频都属于同一个Stream。有四个ssrc，同属于一个Stream

> **注意**
>
> - 在PlanB规格中，只有两个媒体描述，即音频媒体描述（m=audio……）和视频媒体描述（m=video……）。如果要传输多路视频，则它们在视频媒体描述中需要通过SSRC来区分。
>
> - 在UnifiedPlan中可以有多个媒体描述，因此，对于上面多路视频的情况，将其拆成多个视频媒体描述（多行“m=video……”）即可。

### 1.3 simulcast  

```js
// SIM simulcast 使用
a=ssrc-group:SIM 432309628 1414321037------> simulcast 两道流
// FID fec 使用
a=ssrc-group:FID 432309628 4002080514------> 432309628 主流， 4002080514 重传流
a=ssrc-group:FID 1414321037 2646291283------> 1414321037 主流， 2646291283 重传流

// simulcast的第一道流
a=ssrc:432309628 cname:q8Drj/0cwfbEANzz
a=ssrc:432309628 msid:1075707905 video-default ------》mediastreamId + trackId
a=ssrc:432309628 mslabel:1075707905    ----》mediastreamId
a=ssrc:432309628 label:video-default	----》 trackId

// simulcast的第二道流
a=ssrc:1414321037 cname:q8Drj/0cwfbEANzz
a=ssrc:1414321037 msid:1075707905 video-default
a=ssrc:1414321037 mslabel:1075707905
a=ssrc:1414321037 label:video-default

// 第一道流的重传流
a=ssrc:4002080514 cname:q8Drj/0cwfbEANzz
a=ssrc:4002080514 msid:1075707905 video-default
a=ssrc:4002080514 mslabel:1075707905
a=ssrc:4002080514 label:video-default

// 第二道流的重传流
a=ssrc:2646291283 cname:q8Drj/0cwfbEANzz
a=ssrc:2646291283 msid:1075707905 video-default
a=ssrc:2646291283 mslabel:1075707905
a=ssrc:2646291283 label:video-default

// 432309628，1414321037，4002080514，2646291283 的cname 都一样q8Drj/0cwfbEANzz
```

有四个ssrc，同属于一个Stream



## 2. ssrc-group

a=ssrc-group 定义参考 RFC5576 ，用于描述多个 ssrc 之间的关联，常见的有两种：

- a=ssrc-group:FID 2430709021 3715850271 FID (Flow Identification) 最初用在 FEC（前向纠错） 的关联中，WebRTC 中通常用于关联一组常规 RTP stream 和 重传 RTP stream 。

- a=ssrc-group:SIM 360918977 360918978 360918980 在 Chrome 独有的 SDP munging 风格的 simulcast 中使用，将三组编码质量由低到高的 MediaStreamTrack 关联在一起。

  https://blog.csdn.net/aggresss/article/details/109850434





## 3. 各种id

### 3.1 TrackId

track 的id

### 3.2 MediaStreamId

addtrack的时候，传入的stream_id

### 3.3 msid

TrackId+MediaStreamId



## 参考

[烫青菜-WebRTC | SDP详解](https://blog.csdn.net/weixin_39766005/article/details/132294422)

[WebRTC 中 SDP 信息解析](https://blog.csdn.net/aggresss/article/details/109850434)
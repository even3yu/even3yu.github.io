---
layout: post
title: rtcp sr 和 rr 的区别
date: 2023-11-01 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtp rtcp
---


* content
{:toc}

---

## 1. SR和RR的区别

从报文内容上看SR比RR多出来了sender info这部分数据段，这部分包含了发送报告方作为发送者相关的信息。

SR表面译为作为发送者报告，RR表面译为接收者报告
实际上按字面上理解容易造成误解，以为SR就是发送端的报告，而RR就是接收端的报告。其实不是完全这样的。

**SR并不只是发送者发送多少包告诉对方，而是还有将发送者这端接收了多少包，接收过程中丢了多少包**。
所以SR包含两个信息，

- 将发送者自己发送的信息告诉对端，
- 同时将自己接收的信息也告诉对端。

**RR是当报文发送方只作为接收者，而不发送媒体数据时，发给对端自己作为接收方接收到的数据的统计信息**

> 这是从另一篇文章中提到的相应的内容，可以帮助理解
> RTP 报文的接收者可以利用两种类型的 RTCP 报告报文(SR 或 RR)来提供有关数据接收质量的统计信息,具体选用 SR 报文还是 RR 报文要看该接收者是否同时是一个 RTP 报文的发送者,明确地讲,如果一个会话参加者自最后一次 发送 RTCP报文后,发送了新的 RTP 数据报文,那么该参加者需要传送 SR 报文, 否则传送 RR 报文。SR 报文和RR 报文的主要区别在于前者包含了20字节有关发送者的信息

当然你可以问自己一个问题，为什么有SR了还需要RR这个类型呢？如果这个问题能回答上来，你对这个概念就算理解了。
答案很简单，本质区别在于发送RTCP报文的这个家伙只是媒体数据的接收者还是说它既是接收者又是发送者。



## 2. 抓包

### 2.1 SR的抓包

![img]({{ site.url }}{{ site.baseurl }}/images/rtp-sr-and-rr.assets/sr.png)

### 2.2 RR的抓包

![img]({{ site.url }}{{ site.baseurl }}/images/rtp-sr-and-rr.assets/rr.png)

### 2.3 可以看到SR和RR的区别（sender info）

![img]({{ site.url }}{{ site.baseurl }}/images/rtp-sr-and-rr.assets/sr-mark.png)


从以下抓包可以看出，通话方A（192.168.12.139）和B（192.168.12.245）在正常建立通话时，A发出的是SR类型报文，当A按下hold（挂掉）之后，不再发送媒体数据，发出的是RR类型的报文。

![img]({{ site.url }}{{ site.baseurl }}/images/rtp-sr-and-rr.assets/sr-2-rr.png)



## 参考

[RTCP中SR和RR的简介与区别](https://blog.csdn.net/csdn_zmf/article/details/105575968)




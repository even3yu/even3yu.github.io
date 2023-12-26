---
layout: post
title: 如何通过wireshark 抓rtp包
date: 2023-12-16 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: rtp wireshark
---


* content
{:toc}

---


## RTP报文抓包分析

### 1. rtp包

拿到抓包文件后使用wireshark打开，并使用合适的过滤条件进行过滤，然后选中其中一个数据包，右键选择[解码为…(Decode As…)]

![image-20231108114015096]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/00.png)



![image-20231108114103212]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/01.png)



![image-20231108114130992]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/02.png)



![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/1.png)

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/2.png)

根据提供的帧信息，这是一个 RTP(实时传输协议) 包，

- 在以太网 II 上传输，
- 长度为 1147 字节 (9176 位)。
- 该帧的源地址为 Cisco_ba:5a:44(00:d\e:fb:ba:5a:44),
- 目标地址为 fa:16:3e:d0:ca:a7(fa:16:3e:d0:ca:a7)。
- 该帧使用 IP(互联网协议) 版本 4，
- 源地址为 112.17.79.157，
- 目标地址为 172.21.145.154。
- 该帧使用 UDP(用户数据报协议),
- 源端口为 59037，
- 目标端口为 20309。



### 2. rtp 包详细信息

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/3.png)

这是一个 RTP(实时传输协议) 包，

- 它被 DTLS-SRTP(数据加密协议 - 安全 rtp) 协议初始化。
- 该包的序列号(Sequence number)为 9632，
- 扩展序列号([Extended sequence number)为 75168，
- 时间戳(Timestamp)为 4038466672。
- 该包包含一个未知的同步源标识符，其定义未知。
- 该包包含三个 RFC 5285(现代 RTP 协议) 头扩展。
- 在 RTP 包中，数据部分是一个动态 RTP 类型 (DynamicRTP-Type-96),其序列号为 96。该序列号是动态分配的，用于标识当前数据传输。



## RTCP报文抓包分析(以NACK为例)

### 1. rtcp - NACK

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/4.png)

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/5.png)

- RTCP 帧序列号 416 是一个用于在实时通信中传输控制信息的 RTCP(Real-Time Control Protocol) 帧。
- RTCP 帧长度为 72 字节 (576 位),包括帧头和数据部分的长度。RTCP 帧头包含发送者和接收者的源和目标地址，以及传输协议。
- 通用 RTP 反馈消息是用于报告数据传输错误的 RTP 反馈消息类型。在这种情况下，NACK 消息表明在传输过程中丢失了 8 个帧
- RTCP 帧的源端口号为 20309，这是发送者的应用程序端口号。目标端口号为 59037，这是接收者的应用程序端口号。
  总的来说，捕获结果表明接收者接收到了一个带有通用 RTP 反馈消息的 RTCP 帧，这表明在传输过程中丢失了 8 个帧。
- **PID(消息类型标识符) 用于标识传输反馈消息的类型**。RTCP Transport Feedback NACK PID:19163 表示一个 NACK(否定确认) 消息被发送，以通知接收方有一定数量的帧丢失。NACK 消息通常由发送方发送，以请求接收方重新传输丢失的帧。
  PID:19163 是一个用于传输反馈消息的 PID，它标识了这种消息的类型。在 RTCP 传输中，有许多不同的 PID 用于不同类型的传输反馈消息。

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-wireshark.assets/6.png)

- RTCP Transport Feedback NACK PID 包用于指示 RTP 包的丢失或错误，而 RTCP Transport Feedback NACK BLP 包用于指示 RTP 包的源地址或目的地址不正确。

  RTCP Transport Feedback NACK BLP: Cx027d (Frames 19164 19166 19167 19168 19169 19170 19173 lost)
  Frame 19164 also lost
  Frame 19166 also lost
  Frame 19167 also lost
  Frame 19168 also lost
  Frame 19169 also lost
  Frame 19170 also lost
  Frame 19173 also lost

## 参考

https://blog.csdn.net/Magic_o/article/details/130409187

[使用wireshark解析RTP包中的音频流](https://blog.csdn.net/halazi100/article/details/106550470)

https://www.pianshen.com/article/78351144335/
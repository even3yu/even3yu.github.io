---
layout: post
title: rtcp 开篇
date: 2024-03-26 22:52:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtcp
---


* content
{:toc}

---


## 1. 什么是RTCP

在RFC3550中，除了定义了⽤来进⾏实时数据传输的 RTP 协议外，还定义了 RTCP 协议，**⽤来反馈会话传输质量、⽤户源识别、控制 RTCP 传输间隔**。在 Webrtc 中，通过 RTCP 我们可以实现发送数据/接收数据的反馈，传输控制如丢包重传、关键帧请求，⽹络指标 RTT、丢包率、抖动的计算及反馈，拥塞控制相关的带宽 反馈，以及⽤户体验相关的⾳视频同步等等。为了让开发者获取以上数据指标，Webrtc 提供了统⼀的接⼝调用,如在GoogleChrome中,可以通过 RTCPeerConnection.getStats()或者chrome://webrtc-internals/查看以上指标。



## 2. RTCP 基于UDP发送

RTCP也是用UDP来传送的，但RTCP封装的仅仅是一些控制信息，因而分组很短，所以可以将多个RTCP分组封装在一个UDP包中。类似于RTP信息包，每个RTCP信息包以固定部分开始，紧接着的是可变长结构单元，最后以一个32位边界结束。



## 3. RTCP的作用

在使用 RTP 包传输数据时，难免会发生丢包、乱序、抖动等问题，下面我们来看一下使用的网络一般都会在什么情况下出现问题：

网络线路质量问题引起丢包率高；
传输的数据超过了带宽的负载引起的丢包问题；
信号干扰（信号弱）引起的丢包问题；
跨运营商引入的丢包问题 ;

RTCP 的作用，收集当前网络质量状态

1、为应用程序提供会话质量或者广播性能质量的信息　

RTCP的主要功能是为应用程序提供会话质量或者广播性能质量的信息。每个RTCP信息包不封装声音数据或者电视数据，而是封装发送端（和 / 或者）接收端的统计报表。这些信息包括发送的信息包数目、丢失的信息包数目和信息包的抖动等情况，这些反馈信息反映了当前的网络状况，对发送端、接收端或者网络管理员都非常有用。RTCP规格没有指定应用程序应该使用这些反馈信息做什么，这完全取决于应用程序开发人员。例如，发送端可以根据反馈信息来调整传输速率，接收端可以根据反馈信息判断问题是本地的、区域性的还是全球性的，网络管理员也可以使用RTCP信息包中的信息来评估网络用于多目标广播的性能。

2、确定 RTP用户源　

RTCP为每个RTP用户提供了一个全局唯一的规范名称 (Canonical Name)标志符 CNAME，接收者使用它来追踪一个RTP进程的参加者。当发现冲突或程序重新启动时，RTP中的同步源标识符SSRC可能发生改变，接收者可利用CNAME来跟踪参加者。同时，接收者也需要利用CNAME在相关RTP连接中的几个数据流之间建立联系。当 RTP需要进行音视频同步的时候，接受者就需要使用 CNAME来使得同一发送者的音视频数据相关联，然后根据RTCP包中的计时信息(Network time protocol)来实现音频和视频的同步。

3、控制 RTCP传输间隔

由于每个对话成员定期发送RTCP信息包，随着参加者不断增加，RTCP信息包频繁发送将占用过多的网络资源，为了防止拥塞，必须限制RTCP信息包的流量，控制信息所占带宽一般不超过可用带宽的 5%，因此就需要调整 RTCP包的发送速率。由于任意两个RTP终端之间都互发 RTCP包，因此终端的总数很容易估计出来，应用程序根据参加者总数就可以调整RTCP包的发送速率。

4、传输最小进程控制信息　

这项功能对于参加者可以任意进入和离开的松散会话进程十分有用，参加者可以自由进入或离开，没有成员控制或参数协调。




## 4. RTCP报文类型

⽬前 RTCP 主要定义了以下8种类型的报⽂，其中业务场景中主要⽤到 SR/RR/RTPFB/PSFB，接下来我们也将重点介绍这四种报⽂。未来如果有新类型的话，会继续从208-223中分配， 0/255⽬前禁⽌使⽤。

| 类型 | 缩写表示                         | 用途                                                         |
| ---- | -------------------------------- | ------------------------------------------------------------ |
| 200  | SR（Sender Report）              | 发送者报告，向接收端发送，包含发送端的统计信息，如发送的数据包数量、字节数、丢包数量等。 |
| 201  | RR（Receiver Report）            | 接收者报告。 接收端可以使用RTCP的RR报文向发送端发送接收报告，报告中记录着从上一次报告到本次报告之间丢失了多少包、丢包率是多少、延时是多少等一系列信息。 |
| 202  | SDES（Source Description Items） | 源描述报文，包含有关参与会话的参与者的信息，如CNAME（参与者的标识符）、名称、电话号码等。 |
| 203  | BYE                              | 结束会话报文，用于说明哪些（音视频）媒体源现在不可用了。当WebRTC收到该报文后，应该将SSRC所对应的通道删除。 |
| 204  | APP                              | 实验性的功能和开发。给应用预留的RTCP报文，应用可以根据自己的需要自定义一些应用层可以解析的报文。 |
| 205  | RTPFB                            | 传输层反馈 RTPFeedback rfc4585/5104。RTP的反馈报文，是指RTP传输层面的报文。该报文可以装入不同类型的子报文。该报文中可以包含多个子报文，其中WebRTC使用到的报文只有4项。 |
| 206  | PSFB                             | 特定载荷类型反馈 PayloadspecifcFeedback rfc4585/5104。RTP中与负载相关的反馈报文。同样，该报文也可以装入不同类型的子报文。 |
| 207  | XR                               | XR，RTCP扩展，rfc3611                                        |





## 5. RTCP的报文格式

每个 RTCP 包都有⼀个和 RTP 类似的固定格式的头，⻓度为8，后⾯跟着⻓度不定的结构化数据，在不同 RTCP 类型时，这些结构化数据各不⼀样，但是它们必须都要 32-bit 对⻬。RTCP 的头部是定⻓的，⽽且在头部有⼀个字段来描述这个 RTCP 数据的⻓度，因此 RTCP 可以被复合成⼀组⼀同发送。



### 5.1 通用header

```
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
header |V=2|P|    RC   |   PT=RR=201   |             length            |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                     SSRC of packet sender                     |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_1 (SSRC of first source)                 |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  1    | fraction lost |       cumulative number of packets lost       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           extended highest sequence number received           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      interarrival jitter                      |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         last SR (LSR)                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                   delay since last SR (DLSR)                  |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_2 (SSRC of second source)                |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  2    :                               ...                             :
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
       |                  profile-specific extensions                  |
```

1. V，版本号（2bit）: 版本号为2；

2. P，填充（1bit）: 填充标志，同RTP的填充标志；

3. FMT，接收报告数量（5bit）：这是RR包的定义，接收报告块数量，可以是0个（只发送流），也可以是多个。

4. PT，荷载类型（8bit）: 荷载类型SR、RR、SDES、BYE、APP 等，后续介绍QOS策略还会介绍其他的协议类型；

   | abbrev | name                | value |
   | :----- | :------------------ | :---- |
   | SR     | sender report       | 200   |
   | RR     | receiver report     | 201   |
   | SDES   | source description  | 202   |
   | BYE    | goodbye             | 203   |
   | APP    | application-defined | 204   |
   | …      |                     |       |

5. length，长度（16bit）: 32位的数量，长度代表整个数据包的大小（协议头+荷载+填充）

6. SSRC of sender， 同步源（32bit）：同步源标识符，RR数据包的发送者的识别码



### 5.2 SR: Sender Report RTCP Packet

发送rtp报文端向接受者发送SR报文，**主要目的是方便接收方做好音视频同步工作**。

https://even3yu.github.io/2023/11/01/rtcp-sr/

### 5.3 RR: Receiver Report RTCP Packet

https://even3yu.github.io/2023/11/01/rtcp-rr/

### 5.4 --SDES（Source Description）

### 5.5 --BYE

### 5.6 --APP

### 5.7 RTPFB

https://even3yu.github.io/2023/11/02/rtcp-feedback/

RTP的反馈报文，是指RTP传输层面的报文。该报文可以装入不同类型的子报文。该报文中可以包含多个子报文，其中WebRTC使用到的报文只有4项。RTP-FEEDBACK 主要⽤来在传输层进⾏反馈，实现数据包的丢包重传，码率控制。

作为WebRTC RTCP消息中的一种，RTP Feedback包含的内容很多，所以这里单独介绍。 在RTCP Header中，这类消息的负载类型Payload Type=205，反馈消息类型FMT如下：

| FMT  | Name      | Long Name                                            | Description              | Reference                                                    |
| :--- | :-------- | :--------------------------------------------------- | :----------------------- | :----------------------------------------------------------- |
| 1    | NACK      | Generic negative acknowledgement                     | 丢包重传请求             | [RFC4585](https://tools.ietf.org/html/rfc4585)               |
| 3    | ~~TMMBR~~ | Temporary Maximum Media Stream Bit Rate Request      | 临时最大媒体流比特率请求 | [RFC5104](https://tools.ietf.org/html/rfc5104)               |
| 4    | ~~TMMBN~~ | Temporary Maximum Media Stream Bit Rate Notification | 临时最大媒体流比特率通知 | [RFC5104](https://tools.ietf.org/html/rfc5104)               |
| 7    | TLLEI     | Transport-Layer Third-Party Loss Early Indication    | 传输层第三方丢失早期指示 | [RFC6642](https://tools.ietf.org/html/rfc6642)               |
| 8    | ECN       | Explicit Congestion Notification                     | 显式拥塞通知             | [RFC6679](https://tools.ietf.org/html/rfc6679)               |
| 15   | TWCC      | Transport Wide Congestion Control                    | 传输宽拥塞控制           | [draft-holmer-rmcat-transport-wide-cc-extensions-01](https://tools.ietf.org/html/draft-holmer-rmcat-transport-wide-cc-extensions-01) |

#### 5.7.1 NACK

https://even3yu.github.io/2023/11/16/rtcp-fb-nack-1/

https://even3yu.github.io/2023/11/17/rtcp-fb-nack-2/

https://even3yu.github.io/2023/11/18/rtcp-fb-nack-3/

NACK，接收端用于通知发送方在上次包发送周期内有哪些包丢失了。在NACK报文中包含两个字段：PID和BLP。PID（Package ID）字段用于标识从哪个包开始统计丢包；而BLP（16位）字段表示从PID包开始，接下来的16个RTP包的丢失情况。

在 RTP-FEEDBACK 中，最重要的当属NACK，区别于 TCP 中的 ACK，在 RTCP 中 NACK 代表否定应答，当接收⽅监测到数据包丢失时，发送⼀个 NACK 到发送⽅，表明⾃⼰没有收到某个报⽂。⽬前 NACK 可以认为是对抗弱⽹最主要的⼿段，报⽂格式如下:

- 4个字节(2字节 PID 代表第⼀个开始丢的包、2字节BLP代表紧随其后的16个包的丢包信息、bit1代表丢包);
- 可以同时携带多个nack block.

#### ~~TMMBR和TMMBN~~

~~TMMBR和TMMBN是一对报文，TMMBR表示临时最大码流请求报文，TMMBN是对临时最大码流请求的应答报文。这两个报文虽然在WebRTC中实现了，但已被WebRTC废弃，其功能由TFB和REMB报文所代替。~~

#### 5.7.2 TWCC

https://even3yu.github.io/2023/11/06/rtcp-twcc/

TWCC是WebRTC中TCC算法的反馈报文，该报文会记录包的延迟情况，然后交由发送端的TCC算法计算下行带宽。

- Transport-cc 是⽬前 Webrtc 中最新的拥塞控制算法，替代旧的 GCC 算法;
- Transport-CC 需要在 RTP 中增加扩展，接收端记录 RTP 包的到达时间、间隔并反馈给发送端，这⾥不做详细介绍，后期可以和 GCC ⼀起分享。

### 5.8 PSFB

https://even3yu.github.io/2023/11/08/rtcp-psfb/

RTC 中，主要传输的是⾳频、视频，由于两种媒体有不同的特点，⾳频⼩包，前后帧⽆参考；视频帧具有前后参考关系，关键帧⼜是 GOP 中后续帧的主要参考对象，可以说是重中之重，需要 RTCP 针对具体的载荷类型进⾏更精细化的信息反馈。

- 当前的实现中，主要的反馈实现是PLI， 当⽹络出现丢包时，接收⽅反馈帧丢失，请求发送⽅重新编码关键帧发送；
- FIR 也是请求关键帧，主要⽤在新⽤户加⼊的场景中；
- ApplicationLayerFeeedbackMessage 中主要是 REMB 经常使⽤。

  RTP中与负载相关的反馈报文。同样，该报文也可以装入不同类型的子报文。

| Value | Name      | Long Name                                                    | Reference                                                  |
| :---- | :-------- | :----------------------------------------------------------- | :--------------------------------------------------------- |
| 1     | PLI       | Picture Loss Indication                                      | [RFC4585](https://tools.ietf.org/html/rfc4585)             |
| 2     | SLI       | Slice Loss Indication                                        | [RFC4585](https://tools.ietf.org/html/rfc4585)             |
| 3     | RPSI      | Reference Picture Selection Indication                       | [RFC4585](https://tools.ietf.org/html/rfc4585)             |
| 4     | FIR       | Full Intra Request Command                                   | [RFC5104](https://tools.ietf.org/html/rfc5104)             |
| 5     | TSTR      | Temporal-Spatial Trade-off Request                           | [RFC5104](https://tools.ietf.org/html/rfc5104)             |
| 6     | TSTN      | Temporal-Spatial Trade-off Notification                      | [RFC5104](https://tools.ietf.org/html/rfc5104)             |
| 7     | VBCM      | Video Back Channel Message                                   | [RFC5104](https://tools.ietf.org/html/rfc5104)             |
| 8     | PSLEI     | Payload-Specific Third-Party Loss Early Indication           | [RFC6642](https://tools.ietf.org/html/rfc6642)             |
| 9     | ROI       | Video region-of-interest (ROI)                               | [[3GPP TS 26.114 v16.3.0][Ozgur_Oyman]][Ozgur_Oyman]       |
| 10    | LRR       | Layer Refresh Request Command                                | RFC-ietf-avtext-lrr-07                                     |
| 15    | AFB(REMB) | Application Layer Feedback (Receiver Estimated Maximum Bitrate) | https://tools.ietf.org/html/draft-alvestrand-rmcat-remb-03 |
| 31    | EXT       | Reserved for future extensions                               | [RFC4585](https://tools.ietf.org/html/rfc4585)             |
|       |           |                                                              |                                                            |

#### 5.8.1 PLI与FIR

https://even3yu.github.io/2023/11/26/rtcp-psfb-pli/

PLI报文与FIR报文很类似，当发送端收到这两个报文时，都会触发生成关键帧（IDR帧），但两者还是有一些区别的。PLI报文是在接收端解码器无法解码时发送的报文。FIR报文主要应用于多方通信时后加入房间的参与者向已加入房间的共享者申请关键帧。通过这种方式，可以保障后加入房间的参与者不会因收到的第一帧不是关键帧而引起花屏或黑屏的问题。

#### 5.8.2 REMB

REMB报文是WebRTC增加的反馈报文，用于将接收端评估出的带宽值发给发送端。不过，由于最新的WebRTC已全面启用基于发送端的带宽估算方法，即TCC，因此目前REMB仅用于向后兼容，不再做进一步更新。

REMB 即接收端最⼤接收码率估测，需要配合rtp扩展abs-send-time⼀起使⽤，是接收端估算的本地最⼤带宽能⼒，发送端根据 REMB、丢包等进⾏拥塞控制。 

REMB 估算的码率代表的是⼀个传输通道内所有 SSRC 的码率之和, ⽽不是针对于某⼀个特定的 SSRC。



### 5.9 ??? XR

https://even3yu.github.io/2023/11/03/rtcp-xr/

RTCP 在很早之前就定义了扩展报告，主要是在 SR/RR 基础之上携带补充信息，开发者可以基于其中更加详细的指标做更深层次的拥塞控制上的优化。包含以下七个报告块：
...



## 6. ??? RTCP传输间隔

由于RTP设计成允许应用自动扩展，可从几个人的小规模系统扩展成上千人的大规模系统。而每个会话参与者周期性地向所有其他参与者发送RTCP控制信息包，如每个参与者以固定速率发送接收报告，控制流量将随参与者数量线性增长。由于网络资源有限，相应的数据包就要减少，直接影响用户关心的数据传输。为了限制控制信息的流量，RTCP控制信息包速率必须按比例下降。

**一旦确认加入到RTP会话中，即使后来被标记成非活动站，地址的状态仍会被保留，地址应继续计入共享RTCP带宽地址的总数中，时间要保证能扫描典型网络分区，建议为30分钟。注意，这仍大于RTCP报告间隔最大值的五倍。**


## 7. ??? RTP/ RTCP的不足之处

RTP与RTCP相结合虽然保证了实时数据的传输，但也有自己的缺点。最显著的是当有许多用户一起加入会话进程的时候，由于每个参与者都周期发送RTCP信息包，导致RTCP包泛滥(flooding)。



## 8. SR和RR的区别

https://even3yu.github.io/2023/11/01/rtcp-sr-and-rr/



## 9. ??? RTT 计算

https://even3yu.github.io/2023/10/31/rtt/

https://even3yu.github.io/2023/11/03/rtcp-xr/





## 参考

[RTCP协议详解](https://blog.csdn.net/bytxl/article/details/50400987)

[WebRTC | 网络传输协议RTP与RTCP](https://blog.csdn.net/weixin_39766005/article/details/132301075)

[技术解码丨Webrtc中RTCP使用及相关指标计算](https://mp.weixin.qq.com/s/c9AYDyj9-oeZvmCgP1cAUQ)
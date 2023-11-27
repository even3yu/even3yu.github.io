---
layout: post
title: rtp header extension
date: 2023-11-25 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtp
---


* content
{:toc}

---


## 0. 前言

RTP协议中，RTP Header（报头）包括**固定报头**（Fixed Header）与**报头扩展**（Header extension，可选）。

RTP Fixed Header结构如下，其中前12字节内容必须包含的。

```less
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|            contributing source (CSRC) identifiers             |
|                             ....                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

但是这Fixed Header携带的信息满足不了更复杂的需求。所以引入了RTP Header Extension，可以携带更多的信息，同时方便各种扩展。



## 1. WebRTC RTP Header Extension 格式说明

在 RTP协议 rfc3550 section 5.3.1 中定义 RTP header extension 结构如下图所示：

```less
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |      defined by profile       |           length              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        header extension                       |
   |                             ....                              |
```

如果 RTP 标准头部 X 位为1，就表示CSRC后面还有一些额外的 RTP 扩展头，上面的定义中允许使用 16-bit 长度作为 identifier，16-bit 长度作为 header extension 长度说明。
这种定义方式存在两个缺点：① 一个 RTP packet 只能携带一个 header extension； ② 其次，没有给出如何分配 16-bit header extension identifier 以避免冲突。
基于以上两个缺点，rfc5285 对 header extension 做了拓展，支持两种类型的拓展头 One-byte Header 和 Two-byte Header



### 1.1 One-byte Header

```less
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |       0xBE    |    0xDE       |           length =3           |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  ID   | L=0   |     data      |  ID   |  L=1  |   data...
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        ...data   |    0 (pad)    |    0 (pad)    |  ID   | L=3   |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                          data                                 |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- 2个字节 为固定为 0XBEDE 标志，意味着这是一个 one-byte 扩展，

- length = 3 说明 header extension 的总长度为 3 * 32bit = 96bit = 12byte

  ```less
  第一个 一个字节的 header extension头 + 1个字节的data----》 	2个字节
  第二个 一个字节的 header extension头 + 2个字节的data----》	3字节
  为了4字节对齐，加入了pad														----》	2字节			
  第三个 一个字节的 header extension头 + 4个字节的data----》	5字节
  ```

- 每个扩展头首先以一个 byte 开始，前 4 位是这个扩展头的 ID， 后四位是 data 的长度 -1，譬如说 L=0 意味着后面有1个 byte 的 data，

  ```less
  0 1 2 3 4 5 6 7
  +-+-+-+-+-+-+-+-+
  |  ID   |  len  |
  +-+-+-+-+-+-+-+-+
  ```

  - **ID**:4-bit长度的ID表示本地标识符
  - **len**:表示extension data长度，范围：0~15，为0表示长度为1字节，15表示16字节

- 同理第二个扩展头的 L=1 说明后面还有 2 个 byte 的 data，

- **但是注意，其后没有紧跟第三个扩展头，而是添加了 2 个byte大小的全 0 的 data**，这是为了作填充对齐，因为扩展头是以为 32bit 作填充对齐的



#### 1.1.1 抓包

![img]({{ site.url }}{{ site.baseurl }}/images/rtp-header-extesion-1.assets/one-byte-wireshark.png)

- `Defined by profile`字段为0xBEDE，表示One-Byte Header，
- `Extension length`为1，表示Header Extension长度为`1x4`字节，
- 对于Header Extension：ID为3，Lengh为2（data为三个字节）。
  一个字节的 header extension头 + data 为三个字节。

![image-20231127144529653](image-20231127144529653.png)

- 构造相关代码位于`RtpPacket::AllocateRawExtension`中，分配扩展
- 解析相关代码位于`RtpPacket::ParseBuffer`中。解析扩展



#### 1.1.2 !!! 四字节对齐

先算头，再算pad

### 1.2 Two-bytes Header

扩展头为two-byte的情况下， RTP头后的第一个16 bit 如下所示， 一个0x100 + appbits， appbits 可以用来填充应用层级别的数据

```less
   0                   1
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |         0x100         |appbits|
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```




```less
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |       0x10    |    0x00       |           length=3            |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |      ID       |     L=0       |     ID        |     L=1       |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |       data    |    0 (pad)    |       ID      |      L=4      |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                          data                                 |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- 可以看到开头为 0x100 + 0x0， 

- length=3 表示接下来有 3 个 32bit 长度，

- 扩展头和数据，扩展头除了 ID 和 L 相对于 one-byte header **从 4bits 变成了 8bits** ，参考 rfc8285 4.3 节 two-byte 中 **L 表示了真实的长度，不同于 one-byte 中需要进行 +1 计算**。

  ```less
  0                   1
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |       ID      |     length    |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  ```

  - **ID**：本地标识符
  - **length**:表示extension data长度，范围1~255



>  注意：
>
>  rfc8285 中同一个 RTPStream 中允许 one-byte header extension 和 two-byte header extension 同时出现，需要 sdp中声明 ‘a=extmap-allow-mixed’
>  **一个 rtp packet 中 只能使用 one-byte header extension 或 two-byte header extension 其中一种；**
>
>  
>
>  由于WebRTC中默认都是One-Byte Header，所以就不抓包分析了，具体构造解析代码跟One-Byte Header位于同一地方。



#### 1.2.1 extmap-allow-mixed

使用两字节，必须sdp 允许 ExtmapAllowMixed



#### 1.2.2 抓包

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-header-extesion-1.assets/tow-bytes-wireshark.png)



![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtp-header-extesion-1.assets/tow-bytes-wireshark-2.png)

- 扩展为 9*4 = 36 字节
- 第一个扩展， 2个字节的扩展头 + 2字节数据
- 第二个扩展， 2个字节的扩展头 + 27字节数据
- 第三个扩展， 2个字节的扩展头 + 1字节数据



### 1.3 !!! 两者区别

| 区别点           | one-byte                    | two-bytes                         |
| ---------------- | --------------------------- | --------------------------------- |
| 固定头           | 0xBE    0xDE                | 0x100    0x0X                     |
| 每个扩展头的长度 | 4 + 4 = 8，ID和L 分别为4    | 8 + 8 = 16，ID和L 分别从4 升级为8 |
| L所表示的意思    | L + 1（L的取值范围为0～15） | 实际长度                          |
|                  |                             |                                   |



### 1.4 !!! 同一个rtp，只能有一种格式



## 2. 常见RTP Header Extension

```cpp
constexpr ExtensionInfo kExtensions[] = {
    CreateExtensionInfo<TransmissionOffset>(),
    CreateExtensionInfo<AudioLevel>(),
    CreateExtensionInfo<CsrcAudioLevel>(),
    CreateExtensionInfo<AbsoluteSendTime>(),
    CreateExtensionInfo<AbsoluteCaptureTimeExtension>(),
    CreateExtensionInfo<VideoOrientation>(),
    CreateExtensionInfo<TransportSequenceNumber>(),
    CreateExtensionInfo<TransportSequenceNumberV2>(),
    CreateExtensionInfo<PlayoutDelayLimits>(),
    CreateExtensionInfo<VideoContentTypeExtension>(),
    CreateExtensionInfo<RtpVideoLayersAllocationExtension>(),
    CreateExtensionInfo<VideoTimingExtension>(),
    CreateExtensionInfo<RtpStreamId>(),
    CreateExtensionInfo<RepairedRtpStreamId>(),
    CreateExtensionInfo<RtpMid>(),
    CreateExtensionInfo<RtpGenericFrameDescriptorExtension00>(),
    CreateExtensionInfo<RtpDependencyDescriptorExtension>(),
    CreateExtensionInfo<ColorSpaceExtension>(),
    CreateExtensionInfo<InbandComfortNoiseExtension>(),
    CreateExtensionInfo<VideoFrameTrackingIdExtension>(),
};
```



![img]({{ site.url }}{{ site.baseurl }}/images/rtp-header-extesion-1.assets/webrtc-extension.webp)



## 3. RTP header extension SDP 信息说明

在 SDP 信息中 ‘a=extmap’ 用来描述 RTP header extension。格式如下

```less
a=extmap:<value>["/"<direction>] <URI> <extensionattributes>
```

其中：

为 header extension 的 ID 信息，在 one-byte header 中，0保留作为 padding 数据，15保留作为停止标识；
有效类型为 “sendonly”, “recvonly”, “sendrecv”, “inactive”，表示传输方向
为 header extension 的描述 URI
在 IANA 注册的 URI 格式为 urn:ietf:params:rtp-hdrext:avt-example-metadata
没有在 IANA 注册的 URI 格式为 http://example.com/082005/ext.htm#example-metadata
示例：

```less
a=extmap:1 http://example.com/082005/ext.htm#ttime
a=extmap:2/sendrecv http://example.com/082005/ext.htm#xmeta
```



### 3.1 示例

```less
m=audio 9 UDP/TLS/RTP/SAVPF 111 63 103 104 9 102 0 8 106 105 13 110 112 113 126
a=mid:0
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 127 119 125 118 124 107 108 109 117 116 123
a=mid:1
a=extmap:14 urn:ietf:params:rtp-hdrext:toffset
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:13 urn:3gpp:video-orientation
a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:5 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
```



### 3.2 抓包

![img]({{ site.url }}{{ site.baseurl }}/images/rtp-header-extesion-1.assets/head-extension-wireshark.png)

该RTP包共有4个Header extension，这四个Header extension ID分别为：2，3，4，10。根据示例SDP，可知分别是：abs-send-time，transport-wide-cc-extensions，mid，rtp-stream-id扩展。



## 4. ??? WebRTC 支持的 RTP Header Extension 说明

下面列举了 WebRTC 中常见的几种 header extension 的 URI：

|      | URI                                                          | 链接                                                         | extension id |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------ |
| 1    | urn:ietf:params:rtp-hdrext:sdes:mid                          | https://tools.ietf.org/html/rfc7941                          | 4            |
| 2    | urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id                | https://tools.ietf.org/html/draft-ietf-avtext-rid-09         | 10           |
| 3    | urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id       | https://tools.ietf.org/html/draft-ietf-avtext-rid-09         | 11           |
| 4    | http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time   | https://webrtc.googlesource.com/src/+/refs/heads/master/docs/native-code/rtp-hdrext/abs-send-time | 2            |
| 5    | http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01 | https://tools.ietf.org/html/draft-holmer-rmcat-transport-wide-cc-extensions-01 | 3            |
| 6    | urn:ietf:params:rtp-hdrext:framemarking                      | https://tools.ietf.org/html/draft-ietf-avtext-framemarking-07 |              |
| 7    | urn:ietf:params:rtp-hdrext:ssrc-audio-level                  | https://tools.ietf.org/html/rfc6464                          | 1            |
| 8    | urn:3gpp:video-orientation                                   | http://www.3gpp.org/ftp/Specs/html-info/26114.htm            | 13           |
| 9    | urn:ietf:params:rtp-hdrext:toffset                           | https://tools.ietf.org/html/rfc5450                          | 14           |
|      |                                                              |                                                              |              |
| 10   | http://www.webrtc.org/experiments/rtp-hdrext/playout-delay   |                                                              | 5            |
| 11   | http://www.webrtc.org/experiments/rtp-hdrext/video-content-type |                                                              | 6            |
| 12   | http://www.webrtc.org/experiments/rtp-hdrext/video-timing    |                                                              | 7            |
| 13   | http://www.webrtc.org/experiments/rtp-hdrext/color-space     |                                                              | 8            |



### 4.1 urn:ietf:params:rtp-hdrext:sdes:mid

MID: This is a media description identifier that matches the value of the SDP [RFC4566] a=mid attribute [RFC5888], to associate RTP streams multiplexed on the same transport with their respective SDP media description.
在 unified SDP 描述中 ‘a=mid’ 是每个 audio/video line 的必要元素，这个 header extension 将 SDP 中 ‘a=mid’ 后信息保存，用于标识一个 RTP packet 的 media 信息，可以作为一个 media 的唯一标识。

扩展用于在RTP的SDES（Session Description Protocol Security Descriptions）中传递媒体流标识符（MID）。它提供了一个唯一标识符，用于标识RTP报文所属的特定媒体流，有助于会话中的媒体流识别和管理。



### 4.2 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id

在 RFC7656 中定义了两个概念 ‘Media Source’ 和 ‘RTP Stream’ 。其中：

Media Source 可以等同于 WebRTC 中 Track 的概念，在 SDP 描述中可以使用 mid 作为唯一标识；
RTP Stream 是 RTP 流传输的最小流单位，例如在 Simulcast 或 SVC 场景中，一个 Media Source 中包含多个 RTP Stream，这时的SDP中 使用 ‘a=rid’ 来描述每个 RTP Stream
https://tools.ietf.org/html/draft-ietf-avtext-rid-09 草案中声明了两种新的 RTCP 类型的 SDES 信息，同时基于 RFC7941 的方法，可以把这个信息放入 RTP header extension 中。

### 4.3 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id

同 4.2，用于声明重传时使用的 rid 标识。

### 4.4 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time

Wire format: 1-byte extension, 3 bytes of data. total 4 bytes extra per packet (plus shared 4 bytes for all extensions present: 2 byte magic word 0xBEDE, 2 byte # of extensions). Will in practice replace the “toffset” extension so we should see no long term increase in traffic as a result.
Encoding: Timestamp is in seconds, 24 bit 6.18 fixed point, yielding 64s wraparound and 3.8us resolution (one increment for each 477 bytes going out on a 1Gbps interface).
Relation to NTP timestamps: abs_send_time_24 = (ntp_timestamp_64 >> 14) & 0x00ffffff ; NTP timestamp is 32 bits for whole seconds, 32 bits fraction of second.
abs-send-time 为 一个 3 bytes 的时间数据。
mediasoup 中 将 毫秒转换为 abs-send-time 的方法为：

```cpp
static uint32_t TimeMsToAbsSendTime(uint64_t ms)
{
    return static_cast<uint32_t>(((ms << 18) + 500) / 1000) & 0x00FFFFFF;
}
```

GCC模块的 REMB 计算需要 RTP 报文扩展头部 abs-send-time 的支持，用以记录 RTP 数据包在发送端的绝对发送时间

#### kRtpExtensionAbsoluteSendTime

一个包的绝对发送时间

```cpp
# rtp_sender.cc +572
padding_packet.SetExtension<AbsoluteSendTime>(AbsoluteSendTime::MsTo24Bits(now_ms));
```

该扩展用于在RTP报文中添加绝对发送[时间戳](https://so.csdn.net/so/search?q=时间戳&spm=1001.2101.3001.7020)。它提供了一个时间戳，表示该报文离开系统发送时的时间（或尽可能接近此时间）。这对于计算网络延迟和时钟同步等操作非常有用。



### 4.5 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01

IETF成立了RMCAT(RTP Media Congestion Avoidance Techniques)工作组，制定在RTP协议之上的拥塞控制机制。
下图为 holmer-rmcat-transport-wide-cc-extensions 的数据组成：

```less
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |       0xBE    |    0xDE       |           length=1            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  ID   | L=1   |transport-wide sequence number | zero padding  |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

可以看出，有效数据只有 16 bit，它记录了一个 sequence number，称为 transport-wide sequence number。
发送端在发送RTP数据包时，在RTP头部扩展中设置传输层序列号TransportSequenceNumber；数据包到达接收端后记录该序列号和包到达时间，然后接收端基于此构造TransportCC报文返回到发送端；发送端解析该报文，并执行Sendside-BWE算法，计算得到基于延迟的码率；最终Ar和基于丢包率的码率进行比较得到最终目标码率。

#### kRtpExtensionTransportSequenceNumber

扩展序号，不管是第一次发送还是重发的包，此需要都会递增

```cpp
# rtp_sender.cc +1066
*packet_id = transport_sequence_number_allocator_->AllocateSequenceNumber();
 if (!packet->SetExtension<TransportSequenceNumber>(*packet_id))
```

这是一个与传输宽带拥塞控制相关的扩展。具体而言，它是与传输层拥塞控制（Transport Wide Congestion Control）相关的扩展，用于在RTP报文中传递拥塞控制信息，以帮助网络适应和调整传输的数据量。



### 4.6 urn:ietf:params:rtp-hdrext:framemarking

WebRTC 中 RTP payload 部分通过 SRTP 进行加密，这样导致 RTP packet 在经过交换节点或转发节点时，有些场景下需要知道当前 RTP packet 的编码信息，例如传输优先级的优化，这时 urn:ietf:params:rtp-hdrext:framemarking 可以帮助转发或交换节点解决这个问题，因为 RTP header 部分在 SRTP 中不被加密。
不同的编码通常定义不同的标识信息，以 H.264 为例，数据结构定义为：

```less
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  ID=2 |  L=2  |S|E|I|D|B| TID |0|0|0|0|0|0|0|0|    TL0PICIDX  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The following information are extracted from the media payload and sent in the Frame Marking RTP header extension.
S: Start of Frame (1 bit) - MUST be 1 in the first packet in a frame within a layer; otherwise MUST be 0.
E: End of Frame (1 bit) - MUST be 1 in the last packet in a frame within a layer; otherwise MUST be 0.
I: Independent Frame (1 bit) - MUST be 1 for frames that can be decoded independent of temporally prior frames, e.g. intra-frame, VPX keyframe, H.264 IDR [RFC6184], H.265 IDR/CRA/BLA/RAP [RFC7798]; otherwise MUST be 0. Note that this bit only signals temporal independence, so it can be 1 in spatial or quality enhancement layers that depend on temporally co-located layers but not temporally prior frames.
D: Discardable Frame (1 bit) - MUST be 1 for frames that can be discarded, and still provide a decodable media stream; otherwise MUST be 0.
B: Base Layer Sync (1 bit) - MUST be 1 if this frame only depends on the base layer; otherwise MUST be 0. If no scalability is used, this MUST be 0.
TID: Temporal ID (3 bits) - The base temporal layer starts with 0, and increases with 1 for each higher temporal layer/sub-layer. If no scalability is used, this MUST be 0.
LID: Layer ID (8 bits) - Identifies the spatial and quality layer encoded. If no scalability is used, this MUST be 0 or omitted. When omitted, TL0PICIDX MUST also be omitted.
TL0PICIDX: Temporal Layer 0 Picture Index (8 bits) - Running index of base temporal layer 0 frames when TID is 0. When TID is not 0, this indicates a dependency on the given index. If no scalability is used, this MUST be 0 or omitted. When omitted, LID MUST also be omitted.

### 4.7 urn:ietf:params:rtp-hdrext:ssrc-audio-level

结构比较简单，使用RFC6464定义的针对audio的扩展首部，用来调节音量，比如在大型会议中有多个音频流就可以用它来调整音频混流的策略

```less
0                   1
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  ID   | len=0 |V| level       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### kRtpExtensionAudioLevel

一帧音频数据的分贝值

```cpp
# rtp_sender_audio.cc +226
packet->SetExtension<AudioLevel>(frame_type == kAudioFrameSpeech, audio_level_dbov);
```

该扩展用于在RTP报文中传递音频级别信息。它可以提供有关音频流的音量级别或音频能量的指示，有助于接收端进行音频处理和调整。

### 4.8 urn:3gpp:video-orientation

Coordination of video orientation (CVO)
3GPP TS 26.114 section 7.4.5
定义文件参考 26114-g52.doc

```less
0                   1
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  ID   | len=0 |R0R1F C        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

With the following definitions:
C = Camera: indicates the direction of the camera used for this video stream. It can be used by the MTSI client in receiver to e.g. display the received video differently depending on the source camera.
0: Front-facing camera, facing the user. If camera direction is unknown by the sending MTSI client in the terminal then this is the default value used.
1: Back-facing camera, facing away from the user.
F = Flip: indicates a horizontal (left-right flip) mirror operation on the video as sent on the link.
0: No flip operation. If the sending MTSI client in terminal does not know if a horizontal mirror operation is necessary, then this is the default value used.
1: Horizontal flip operation
R1, R0 = Rotation: indicates the rotation of the video as transmitted on the link. The receiver should rotate the video to compensate that rotation. E.g. a 90° Counter Clockwise rotation should be compensated by the receiver with a 90° Clockwise rotation prior to displaying.



#### kRtpExtensionVideoRotation

帧视频帧的方向

```cpp
# rtp_sender_video.cc +331
last_packet->SetExtension<VideoOrientation>(current_rotation);
```

此头扩展用于提供时间戳偏移信息，用于校准接收方的时钟。



### 4.9 urn:ietf:params:rtp-hdrext:toffset

传输时间偏移 (Transmission Time Offset)，offset 为 RTP packet 中 timestamp 与 实际发送时间的偏移。一些前向编码 packet 所携带的 timestamp 和实际的发送时间有一定偏移，加入偏移可以更精确的计算传输耗时。

```less
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  ID   | len=2 |              transmission offset              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### kRtpExtensionTransmissionTimeOffset

一个包相对于采集时间的偏移的滴答数

```cpp
# rtp_sender.cc +569
if (capture_time_ms > 0) {
  padding_packet.SetExtension<TransmissionOffset>(
      (now_ms - capture_time_ms) * kTimestampTicksPerMs);
}
```



## 5. 参考

1. https://www.cnblogs.com/ishen/p/12050077.html
2. https://www.jianshu.com/p/ab32a8a3552f
3. https://blog.csdn.net/linux_vae/article/details/100558804
4. [WebRTC RTP Header Extension 分析](https://blog.csdn.net/aggresss/article/details/106436703)
5. [WebRTC研究：RTP报头扩展](https://blog.jianchihu.net/webrtc-research-rtp-header-extension.html)
6. [SDP中的RTP头扩展 说明](https://blog.csdn.net/zrjliming/article/details/132337587)
7. [【RTP】两字节扩展](https://zhangbin.blog.csdn.net/article/details/129399799)
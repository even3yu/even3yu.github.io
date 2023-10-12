---
layout: post
title: webrtc sdp-sample
date: 2023-10-04 10:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---



## 1
```less
//---------------------- Session Metadata -------------------//
// v = 0  “v =”字段给出SDP的版本,默认为0。
v=0

// o = <username> <sess-id> <sess-version> <nettype> <addrtype> <unicast-address> 
// <username> 是用户在始发主机上的登录名
// <sess-id> <sess-id>是一个数字字符串，使得<username><sess-id>nettype><addrtype>和<unicast-address>的元组形成会话的全局唯一标识符。
// <sess-version>是此会话描述的版本号 
// <nettype>是一个给出网络类型的文本字符串。 “IN”被定义为具有“Internet”的含义
// <addrtype>是一个文本字符串，给出了后面的地址类型 定义了“IP4”和“IP6”
// <unicast-address>是创建会话的计算机的地址。   
o=mozilla...THIS_IS_SDPARTA-67.0.4 7082731800936198286 0 IN IP4 0.0.0.0

// s = <会话名称>  “s =”字段是文本会话名称.每个会话描述必须有一个且只有一个“s =”字段。“s =”字段不能为空
s=-

//t =<start-time> <stop-time>“t =”行指定会话的开始和停止时间。如果<stop-time>设置为零，则会话不受限制，但在<start-time>之后才会生效。如果<start-time>也为零，则会话被视为永久会话。
t=0 0
//********************* Session Metadata ********************//


//---------------------- Security Descriptions-------------------//
//a = <attribute>：<value> 
//a = <fingerprint> SRTP 所需的DTLS指纹信息
a=fingerprint:sha-256 16:9A:59:1B:25:91:DD:C3:4D:B6:22:AB:36:0E:74:1A:F0:C1:62:0B:7F:7D:D3:08:93:13:83:B0:A9:4D:94:E7

//Negotiating Media Multiplexing Using the Session Description Protocol
//表示需要共用一个传输通道传输的媒体，通过ssrc进行区分不同的流。如果没有这一行，音视频数据就会分别用单独udp端口来发送.
a=group:BUNDLE 0 1 2

//Trickle ICE:Incremental Provisioning of Candidates for the Interactive Connectivity Establishment (ICE) Protocol
//ICE建立候选时 采用增量设置的方式
a=ice-options:trickle

//WebRTC MediaStream Identification in the Session Description Protocol
//标识SDP中包含MediaStream的标识
a=msid-semantic:WMS *
//********************* Security Descriptions ********************//


//---------------------- Stream Description -------------------//
// m=<media> <port> <proto> <fmt> ...
//<media>是媒体类型 当前定义的媒体 "audio","video", "text", "application", and "message"
//<port>是传输媒体流的传输端口。在相关的“c =”字段中指定，以及在媒体字段的<proto>子字段中定义的传输协议。
//<proto>是传输协议。传输协议的含义取决于相关“c =”字段中的地址类型字段。
//<fmt>是媒体格式描述(编码类型)。第四个和任何后续 如果<proto>子字段是“RTP / AVP”或“RTP / SAVP”，则<fmt> 子字段包含RTP有效载荷类型号。
m=audio 9 UDP/TLS/RTP/SAVPF 109 9 0 8 101

// c=<nettype> <addrtype> <connection-address>
// 会话描述必须包含 每个媒体描述中的至少一个“c =”字段或会话级别的单个“c =”字段
// <nettype>   “IN”被定义为具有“Internet”的含义
// <addrtype> 为IP4和IP6时
c=IN IP4 0.0.0.0
//a = sendrecv 这指定应以发送和接收模式启动工具。对于具有默认为仅接收模式的工具的交互式会议，这是必需的。
a=sendrecv

//   The URI for declaring this header extension in an extmap attribute is "urn:ietf:params:rtp-hdrext:ssrc-audio-level".
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:2/recvonly urn:ietf:params:rtp-hdrext:csrc-audio-level
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:mid

//a=fmtp:<format> <format specific parameters> 用于传递特殊参数
// 此属性允许以SDP不必理解的方式传达特定于特定格式的参数。格式必须是为媒体指定的格式之一。格式特定参数可以是SDP要求传达的任何参数集，
// maxplaybackrate-采样率，stereo-声道，useinbandfec-fec
a=fmtp:109 maxplaybackrate=48000;stereo=1;useinbandfec=1
a=fmtp:101 0-15

//---------------------- Security Descriptions-------------------//
a=ice-pwd:141cd92cd90d3f5c446004c3404b13c7
a=ice-ufrag:33748035
a=mid:0
a=msid:{15d330d9-3078-4727-b51e-cdc8a2554dd1} {435405e5-84b9-49bb-85a3-9d06b16970f7}
//********************* Security Descriptions ********************//


//"a = rtcp-mux"属性以指示需要RTP和RTCP多路复用
a=rtcp-mux
//rtpmap：<payload type> <encoding name> / <clock rate> [/ <encoding parameters>]
//<payload type> 此属性从RTP有效内容类型编号（在 “m =”行中使用）映射到表示要使用的有效载荷格式,
// 					编码类型 采样率 声道
a=rtpmap:109 opus/48000/2
a=rtpmap:9 G722/8000/1
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:101 telephone-event/8000/1

//DTLS 握手方式 "active" / "passive" / "actpass"/ "holdconn"   'actpass'： 连接或启动传出连接。
a=setup:actpass

//同步源（SSRC）标识符识别SDP，将属性与这些来源相关联
//新的SDP媒体级属性“ssrc”，用于标识特定的属性RTP会话中的同步源并充当元数据属性将源级属性信息映射到这些源
// cname stream的别名，ssrc是否属于同一个stream
a=ssrc:2040182694 cname:{eee1aa1e-411a-47d5-98ad-b26dc639bc67}


m=video 9 UDP/TLS/RTP/SAVPF 120 121 126 97
c=IN IP4 0.0.0.0
a=sendrecv

a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:4 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:5 urn:ietf:params:rtp-hdrext:toffset

a=fmtp:126 profile-level-id=42e01f;level-asymmetry-allowed=1;packetization-mode=1
a=fmtp:97 profile-level-id=42e01f;level-asymmetry-allowed=1
a=fmtp:120 max-fs=12288;max-fr=60
a=fmtp:121 max-fs=12288;max-fr=60

//---------------------- Security Descriptions-------------------//
a=ice-pwd:141cd92cd90d3f5c446004c3404b13c7
a=ice-ufrag:33748035

a=mid:1
a=msid:{15d330d9-3078-4727-b51e-cdc8a2554dd1} {47004068-3cac-4ee8-87df-2cdf02ae0921}
//---------------------- Security Descriptions-------------------//



//a = rtcp-fb： RTCP-FB-PT SP RTCP-FB-VAL CRLF
//rtcp-fb-pt是有效负载类型
///rtcp-fb-val定义反馈消息的类型 ack，nack，trr-int和rtcp-fb-id
a=rtcp-fb:120 nack
a=rtcp-fb:120 nack pli
a=rtcp-fb:120 ccm fir
a=rtcp-fb:120 goog-remb
a=rtcp-fb:121 nack
a=rtcp-fb:121 nack pli
a=rtcp-fb:121 ccm fir
a=rtcp-fb:121 goog-remb
a=rtcp-fb:126 nack
a=rtcp-fb:126 nack pli
a=rtcp-fb:126 ccm fir
a=rtcp-fb:126 goog-remb
a=rtcp-fb:97 nack
a=rtcp-fb:97 nack pli
a=rtcp-fb:97 ccm fir
a=rtcp-fb:97 goog-remb


a=rtcp-mux
//rtpmap：<payload type> <encoding name> / <clock rate> [/ <encoding parameters>]
//<payload type> 此属性从RTP有效内容类型编号（在 “m =”行中使用）映射到表示要使用的有效载荷格式,编码类型 采样率 编码参数
//而以下格式的编码的格式要去相应编码格式标准文档中查看 比如H264 就是RFC3984 https://tools.ietf.org/html/rfc3984
// 频率 90k
a=rtpmap:120 VP8/90000
a=rtpmap:121 VP9/90000
a=rtpmap:126 H264/90000
a=rtpmap:97 H264/90000

a=setup:actpass
a=ssrc:1381656291 cname:{eee1aa1e-411a-47d5-98ad-b26dc639bc67}
//********************* Stream Description ********************//

m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=sendrecv
a=ice-pwd:141cd92cd90d3f5c446004c3404b13c7
a=ice-ufrag:33748035
a=mid:2
a=setup:actpass
a=sctp-port:5000
a=max-message-size:1073741823
```









```less
v=0
o=- 4513006962530722834 5 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE audio video
a=msid-semantic: WMS
m=audio 51821 RTP/SAVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
c=IN IP4 192.168.1.18
a=rtcp:9 IN IP4 0.0.0.0
a=candidate:918459911 1 udp 2122260223 192.168.1.18 51821 typ host generation 0 network-id 1
a=candidate:3500871610 1 udp 2122194687 192.168.31.231 51822 typ host generation 0 network-id 4 network-cost 10
a=ice-ufrag:6qy7
a=ice-pwd:STyALQEpY1K0/m9cv8w+K4rL
a=ice-options:trickle
a=mid:audio
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=recvonly
a=rtcp-mux
a=crypto:0 AES_CM_128_HMAC_SHA1_80 inline:z3Jk6YCdDrh41VkHJdllZbj4k59XDs0Ig8JbJjeq
a=crypto:1 SM4_CTR_HMAC_SM3_128 inline:Xc8d2rG6unEJt8HODrSb49GGZG4PL6LxfAxCuI8c
a=rtpmap:111 opus/48000/2
a=rtcp-fb:111 transport-cc
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:104 ISAC/32000
a=rtpmap:9 G722/8000
a=rtpmap:102 ILBC/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:106 CN/32000
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
a=rtpmap:110 telephone-event/48000
a=rtpmap:112 telephone-event/32000
a=rtpmap:113 telephone-event/16000
a=rtpmap:126 telephone-event/8000
m=video 9 RTP/SAVPF 100 96 98 127 125 97 99 101 124
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:6qy7
a=ice-pwd:STyALQEpY1K0/m9cv8w+K4rL
a=ice-options:trickle
a=mid:video
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 urn:3gpp:video-orientation
a=extmap:5 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=crypto:0 AES_CM_128_HMAC_SHA1_80 inline:z3Jk6YCdDrh41VkHJdllZbj4k59XDs0Ig8JbJjeq
a=crypto:1 SM4_CTR_HMAC_SM3_128 inline:Xc8d2rG6unEJt8HODrSb49GGZG4PL6LxfAxCuI8c
a=rtpmap:100 H264/90000
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=rtcp-fb:100 transport-cc
a=fmtp:100 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 transport-cc
a=rtpmap:98 VP9/90000
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=rtcp-fb:98 transport-cc
a=rtpmap:127 red/90000
a=rtpmap:125 ulpfec/90000
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=100
a=rtpmap:124 rtx/90000
a=fmtp:124 apt=127
```





## 2. offer sap

```less
v=0
o=- 630428897091081157 2 IN IP4 127.0.0.1
s=-
t=0 0


a=group:BUNDLE 0 1 2
a=extmap-allow-mixed
a=msid-semantic: WMS 28c3fbd4-e027-42ac-aefb-0db49e653d08


m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102 125 104 124 106 107 108 109 127
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:FKLe
a=ice-pwd:AeHntE7WZTKbsvb5NwwzxNqQ
a=ice-options:trickle
a=fingerprint:sha-256 82:76:79:C3:05:F5:02:51:4A:B3:65:BF:58:65:C7:D6:84:BB:BE:3F:A6:50:7D:39:5C:F4:EE:08:20:4F:3E:FB
a=setup:actpass
a=mid:0
a=extmap:1 urn:ietf:params:rtp-hdrext:toffset
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 urn:3gpp:video-orientation
a=extmap:4 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:5 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space
a=extmap:9 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendrecv
a=msid:28c3fbd4-e027-42ac-aefb-0db49e653d08 0ef12a68-9f01-45ac-8b61-27736713eb50
a=rtcp-mux
a=rtcp-rsize

//
a=rtpmap:96 H264/90000
a=rtcp-fb:96 goog-remb
a=rtcp-fb:96 transport-cc
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=fmtp:96 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=640c1f
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

//
a=rtpmap:98 H264/90000
a=rtcp-fb:98 goog-remb
a=rtcp-fb:98 transport-cc
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=fmtp:98 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98

//
a=rtpmap:100 H264/90000
a=rtcp-fb:100 goog-remb
a=rtcp-fb:100 transport-cc
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=fmtp:100 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=640c1f
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=100

//
a=rtpmap:102 H264/90000
a=rtcp-fb:102 goog-remb
a=rtcp-fb:102 transport-cc
a=rtcp-fb:102 ccm fir
a=rtcp-fb:102 nack
a=rtcp-fb:102 nack pli
a=fmtp:102 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42e01f
// rtx 重传流
a=rtpmap:125 rtx/90000
// 125 是 102 的重传流
a=fmtp:125 apt=102

// 
a=rtpmap:104 VP8/90000
a=rtcp-fb:104 goog-remb
a=rtcp-fb:104 transport-cc
a=rtcp-fb:104 ccm fir
a=rtcp-fb:104 nack
a=rtcp-fb:104 nack pli
a=rtpmap:124 rtx/90000
a=fmtp:124 apt=104

a=rtpmap:106 VP9/90000
a=rtcp-fb:106 goog-remb
a=rtcp-fb:106 transport-cc
a=rtcp-fb:106 ccm fir
a=rtcp-fb:106 nack
a=rtcp-fb:106 nack pli
a=fmtp:106 profile-id=0

//
a=rtpmap:107 rtx/90000
a=fmtp:107 apt=106

// red
a=rtpmap:108 red/90000
//  rtx 重传
a=rtpmap:109 rtx/90000
a=fmtp:109 apt=108

a=rtpmap:127 ulpfec/90000
// fec， 1221012940主流， 259630068重传流
a=ssrc-group:FID 1221012940 259630068
// cname 相同，表示属于同一个stream
a=ssrc:1221012940 cname:W/+RaDQx1drLlyks
a=ssrc:1221012940 msid:28c3fbd4-e027-42ac-aefb-0db49e653d08 0ef12a68-9f01-45ac-8b61-27736713eb50
a=ssrc:1221012940 mslabel:28c3fbd4-e027-42ac-aefb-0db49e653d08
a=ssrc:1221012940 label:0ef12a68-9f01-45ac-8b61-27736713eb50
a=ssrc:259630068 cname:W/+RaDQx1drLlyks
a=ssrc:259630068 msid:28c3fbd4-e027-42ac-aefb-0db49e653d08 0ef12a68-9f01-45ac-8b61-27736713eb50
a=ssrc:259630068 mslabel:28c3fbd4-e027-42ac-aefb-0db49e653d08
a=ssrc:259630068 label:0ef12a68-9f01-45ac-8b61-27736713eb50


// 端口 在webrtc都不用
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 9 0 8 105 13 110 113 126
// ip 在webrtc都不用
c=IN IP4 0.0.0.0
// 端口和ip 在webrtc都不用
a=rtcp:9 IN IP4 0.0.0.0

// ice
a=ice-ufrag:FKLe
a=ice-pwd:AeHntE7WZTKbsvb5NwwzxNqQ
// trickle
// 使用trickle方式，sdp里面描述媒体信息和ice后选项的信息可以分开传输，
// 先发送sdp过去，在收集地址信息，目的是为了同时进行，而不是等待收集地址信息完成后才开始。
a=ice-options:trickle

// dtls
a=fingerprint:sha-256 82:76:79:C3:05:F5:02:51:4A:B3:65:BF:58:65:C7:D6:84:BB:BE:3F:A6:50:7D:39:5C:F4:EE:08:20:4F:3E:FB
a=setup:actpass

a=mid:1
// rtp扩展
a=extmap:14 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:9 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
// 方向
a=sendrecv
a=msid:28c3fbd4-e027-42ac-aefb-0db49e653d08 cf0b2bd2-d3e5-469d-a75c-4d4b851a1369
// rtp，rtcp端口复用
a=rtcp-mux

// opus， 采样率=48000，声道=2
a=rtpmap:111 opus/48000/2
// rtcp-fb = transport-cc
a=rtcp-fb:111 transport-cc
// minptime 最小帧=10ms， fec
a=fmtp:111 minptime=10;useinbandfec=1

a=rtpmap:103 ISAC/16000
a=rtpmap:9 G722/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
// 舒适噪音
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
//
a=rtpmap:110 telephone-event/48000
a=rtpmap:113 telephone-event/16000
a=rtpmap:126 telephone-event/8000
// cname 一样，代表是同一个stream
a=ssrc:1584357457 cname:W/+RaDQx1drLlyks
// msid = mslabel + label
a=ssrc:1584357457 msid:28c3fbd4-e027-42ac-aefb-0db49e653d08 cf0b2bd2-d3e5-469d-a75c-4d4b851a1369
// mslabel = MediaStream ID
a=ssrc:1584357457 mslabel:28c3fbd4-e027-42ac-aefb-0db49e653d08
// label = MediaStreamTrack ID
a=ssrc:1584357457 label:cf0b2bd2-d3e5-469d-a75c-4d4b851a1369


m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=ice-ufrag:FKLe
a=ice-pwd:AeHntE7WZTKbsvb5NwwzxNqQ
a=ice-options:trickle
a=fingerprint:sha-256 82:76:79:C3:05:F5:02:51:4A:B3:65:BF:58:65:C7:D6:84:BB:BE:3F:A6:50:7D:39:5C:F4:EE:08:20:4F:3E:FB
a=setup:actpass
a=mid:2
a=sctp-port:5000
a=max-message-size:262144
```



## 3. answer sdp

```less
v=0
o=- 2520977872537921153 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1 2
a=extmap-allow-mixed
a=msid-semantic: WMS
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102 125 104 124 106 107 108 109 127
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:fdS+
a=ice-pwd:1xH4YrDUzm6GCpDc73OxpaJS
a=ice-options:trickle
a=fingerprint:sha-256 8D:E1:9F:5E:E8:CD:0D:DD:51:8B:FB:BE:EA:82:68:17:94:38:B2:0A:C1:98:EC:A3:DA:F6:CE:08:33:06:8F:89
a=setup:active
a=mid:0
a=extmap:1 urn:ietf:params:rtp-hdrext:toffset
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 urn:3gpp:video-orientation
a=extmap:4 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:5 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space
a=extmap:9 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 H264/90000
a=rtcp-fb:96 goog-remb
a=rtcp-fb:96 transport-cc
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=fmtp:96 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=640c1f
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=rtpmap:98 H264/90000
a=rtcp-fb:98 goog-remb
a=rtcp-fb:98 transport-cc
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=fmtp:98 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98
a=rtpmap:100 H264/90000
a=rtcp-fb:100 goog-remb
a=rtcp-fb:100 transport-cc
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=fmtp:100 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=640c1f
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=100
a=rtpmap:102 H264/90000
a=rtcp-fb:102 goog-remb
a=rtcp-fb:102 transport-cc
a=rtcp-fb:102 ccm fir
a=rtcp-fb:102 nack
a=rtcp-fb:102 nack pli
a=fmtp:102 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42e01f
a=rtpmap:125 rtx/90000
a=fmtp:125 apt=102
a=rtpmap:104 VP8/90000
a=rtcp-fb:104 goog-remb
a=rtcp-fb:104 transport-cc
a=rtcp-fb:104 ccm fir
a=rtcp-fb:104 nack
a=rtcp-fb:104 nack pli
a=rtpmap:124 rtx/90000
a=fmtp:124 apt=104
a=rtpmap:106 VP9/90000
a=rtcp-fb:106 goog-remb
a=rtcp-fb:106 transport-cc
a=rtcp-fb:106 ccm fir
a=rtcp-fb:106 nack
a=rtcp-fb:106 nack pli
a=fmtp:106 profile-id=0
a=rtpmap:107 rtx/90000
a=fmtp:107 apt=106
a=rtpmap:108 red/90000
a=rtpmap:109 rtx/90000
a=fmtp:109 apt=108
a=rtpmap:127 ulpfec/90000
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 9 0 8 105 13 110 113 126
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:fdS+
a=ice-pwd:1xH4YrDUzm6GCpDc73OxpaJS
a=ice-options:trickle
a=fingerprint:sha-256 8D:E1:9F:5E:E8:CD:0D:DD:51:8B:FB:BE:EA:82:68:17:94:38:B2:0A:C1:98:EC:A3:DA:F6:CE:08:33:06:8F:89
a=setup:active
a=mid:1
a=extmap:14 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:9 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=rtcp-fb:111 transport-cc
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:9 G722/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
a=rtpmap:110 telephone-event/48000
a=rtpmap:113 telephone-event/16000
a=rtpmap:126 telephone-event/8000
m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=ice-ufrag:fdS+
a=ice-pwd:1xH4YrDUzm6GCpDc73OxpaJS
a=ice-options:trickle
a=fingerprint:sha-256 8D:E1:9F:5E:E8:CD:0D:DD:51:8B:FB:BE:EA:82:68:17:94:38:B2:0A:C1:98:EC:A3:DA:F6:CE:08:33:06:8F:89
a=setup:active
a=mid:2
a=sctp-port:5000
a=max-message-size:262144
```


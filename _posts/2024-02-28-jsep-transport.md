---
layout: post
title: webrtc JsepTransport
date: 2024-02-28 02:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc transport
---


* content
{:toc}

---

## 0. 继承关系

![img](https://even3yu.github.io/images/set-local-description.assets/transport.jpg)

|                         |      |      |
| ----------------------- | ---- | ---- |
| sigslot::has_slots<>    |      |      |
| PacketTransportInternal |      |      |
| DtlsTransportInternal   |      |      |
| DtlsTransport           |      |      |



|                      |                             |                             |
| -------------------- | --------------------------- | --------------------------- |
| sigslot::has_slots<> |                             |                             |
| RtpTransportInternal | pc/rtp_transport_internal.h |                             |
| RtpTransport         | pc/rtp_transport.h          | 传入DtlsTransport，rtp/rtcp |
| SrtpTransport        | pc/srtp_transport.h         | 传入DtlsTransport，rtp/rtcp |
| DtlsSrtpTransport    | pc/dtls_srtp_transport.h    | 传入DtlsTransport，rtp/rtcp |



## 说明

1. RtpTransport 继承关系

   | Transport            | 说明                                                         |
   | :------------------- | :----------------------------------------------------------- |
   | RtpTransportInternal | lpc/rtp_transport_internal.h；基础类，虚基类；               |
   | RtpTransport         | 非加密的 Transport 类，rtp 是不加密的                        |
   | SrtpTransport        | RTP 进行 protecting/unprotecting，底层用的是 SrtpSession，而 SrtpSession 是对 libsrtp 的封装 |
   | DtlsSrtpTransport    | 维护了两个 DtlsTransport 对象，分别用作 rtp 和 rtcp 会话的建立。当 DtlsTransport 完成协商后，获取加密套件，最终是通过 OpenSSLStreamAdapter::GetDtlsSrtpCryptoSuite 方法实现的 |

2. DtlsTransport， 是DTLS 会话协商过程的控制逻辑。

3. P2PTransportChannel 是连接底层 UDP socket 层和 Transport 的纽带。是 Candidate-flex Address 收集的入口，是 P2P 穿越的入口，穿越成功后，会建立连接，管理着 Connection 对象。

4. JsepTransport 是一个辅助、包装类，实现 Jsep 接口，同时持有 RtpTransport、SrtpTransport、DtlsSrtpTransport 等。

5. JsepTransportController，遵循 JSEP 规范的，实现 Jsep 接口；负载管理各种类型 Transport 对象的**创建、获取、销毁**；是 Peer Connection 到 transport 层的入口，设置远端 candidate，启动 p2p 连接穿越过程。

6. OpenSSLStreamAdapter 是对 openssl 的包装，主要是实现 dtls 协议的应用

7. DtlsSrtpTransport ， DtlsTransport 不一样，注意注意！！！



## 1. JsepTransport

```cpp
  const std::string mid_;
  // needs-ice-restart bit as described in JSEP.
  bool needs_ice_restart_ RTC_GUARDED_BY(accessor_lock_) = false;
  rtc::scoped_refptr<rtc::RTCCertificate> local_certificate_
      RTC_GUARDED_BY(network_thread_);
  std::unique_ptr<JsepTransportDescription> local_description_
      RTC_GUARDED_BY(network_thread_);
  std::unique_ptr<JsepTransportDescription> remote_description_
      RTC_GUARDED_BY(network_thread_);

  // Ice transport which may be used by any of upper-layer transports (below).
  // Owned by JsepTransport and guaranteed to outlive the transports below.
  const rtc::scoped_refptr<webrtc::IceTransportInterface> ice_transport_;
  const rtc::scoped_refptr<webrtc::IceTransportInterface> rtcp_ice_transport_;

  // To avoid downcasting and make it type safe, keep three unique pointers for
  // different SRTP mode and only one of these is non-nullptr.
  std::unique_ptr<webrtc::RtpTransport> unencrypted_rtp_transport_
      RTC_GUARDED_BY(accessor_lock_);
  std::unique_ptr<webrtc::SrtpTransport> sdes_transport_
      RTC_GUARDED_BY(accessor_lock_);
  std::unique_ptr<webrtc::DtlsSrtpTransport> dtls_srtp_transport_
      RTC_GUARDED_BY(accessor_lock_);

  // If multiple RTP transports are in use, |composite_rtp_transport_| will be
  // passed to callers.  This is only valid for offer-only, receive-only
  // scenarios, as it is not possible for the composite to correctly choose
  // which transport to use for sending.
  std::unique_ptr<webrtc::CompositeRtpTransport> composite_rtp_transport_
      RTC_GUARDED_BY(accessor_lock_);

  rtc::scoped_refptr<webrtc::DtlsTransport> rtp_dtls_transport_
      RTC_GUARDED_BY(accessor_lock_);
  rtc::scoped_refptr<webrtc::DtlsTransport> rtcp_dtls_transport_
      RTC_GUARDED_BY(accessor_lock_);
  rtc::scoped_refptr<webrtc::DtlsTransport> datagram_dtls_transport_
      RTC_GUARDED_BY(accessor_lock_);

  std::unique_ptr<webrtc::DataChannelTransportInterface>
      sctp_data_channel_transport_ RTC_GUARDED_BY(accessor_lock_);
  rtc::scoped_refptr<webrtc::SctpTransport> sctp_transport_
      RTC_GUARDED_BY(accessor_lock_);

  SrtpFilter sdes_negotiator_ RTC_GUARDED_BY(network_thread_);
  RtcpMuxFilter rtcp_mux_negotiator_ RTC_GUARDED_BY(network_thread_);

  // Cache the encrypted header extension IDs for SDES negoitation.
  absl::optional<std::vector<int>> send_extension_ids_
      RTC_GUARDED_BY(network_thread_);
  absl::optional<std::vector<int>> recv_extension_ids_
      RTC_GUARDED_BY(network_thread_);

  std::unique_ptr<webrtc::RtpTransportInternal> datagram_rtp_transport_
      RTC_GUARDED_BY(accessor_lock_);
```



### 1.1 JsepTransport创建时机

- SetLocalDescrption的 `SdpOfferAnswerHandler::PushdownTransportDescription`， 
  根据`cricket::SessionDescription::contents()`  这里会创建JsepTransport。

- `JsepTransportController::MaybeCreateJsepTransport`
  CreateIceTransport
  CreateDtlsTransport
  CreateUnencryptedRtpTransport
  CreateSdesTransport
  CreateDtlsSrtpTransport

  ----------->  以上全部放到 JsepTransport

  （mid， JsepTransport ）以这样的键值对存放，
  其中，如果Bundle 多个m Line， 那么以第一个mid 为key，公用JsepTransport；
  如果没有Bundle，那么每个mLine 都会生成JsepTransport；





## 2. JsepTransportController





### 2.1 JsepTransportController创建时机

在PeerConnection::Initialize 创建的

提供了 根据mid，name 查找JsepTransport。



## 3. JsepTransportDescription



### JsepTransportDescription 创建时机

`JsepTransportController::MaybeCreateJsepTransport` 创建了 JsepTransportDescription



## 4. DtlsTransport

## 5. RtpTransport

## 6. SrtpTransport

## 7. DtlsSrtpTransport



---
layout: post
title: webrtc::DtlsTransport(DtlsTransportInterface)， RtpSenderInternal 绑定
date: 2024-03-06 16:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq
---


* content
{:toc}

---

## 前言

在`SdpOfferAnswerHandler::ApplyLocalDescription   SdpOfferAnswerHandler::ApplyRemoteDescription`的时候，遍历所有的Transceiver，根据Transceiver的mid，从`JsepTransportController` 通过 `LookupDtlsTransportByMid` 找到对应的`JsepTransport`，从而从`JsepTransport`获取得到webrtc::`DtlsTransport`。

把webrtc::`DtlsTransport` 绑定到`RtpSenderInternal`和`RtpReceiverInternal`中。

虽然绑定了webrtc::DtlsTransport，`RtpSenderInternal` 也提供了接口，但是目前没发现调用的地方，有知道的吗？？？

> 1. 绑定的是`webrtc::DtlsTransport`，不是`cricket::DtlsTransport`
> 2. `cricket::DtlsTransport`和`webrtc::DtlsTransport` 关系，`cricket::DtlsTransport`作为`webrtc::DtlsTransport `属性

## 0. 类的关系

```less
rtc::PacketTransportInternal
DtlsTransportInternal
cricket::DtlsTransport
```



```less
DtlsTransportInterface
webrtc::DtlsTransport
```



## !!! webrtc::DtlsTransport 和 cricket::DtlsTransport 区别

pc/dtls_transport.h

```cpp
namespace webrtc {

  class DtlsTransport : public DtlsTransportInterface

  ..
  }
}
```



p2p/base/dtls_transport.h

```cpp
namespace rtc {
  namespace cricket {
    class DtlsTransport : public DtlsTransportInternal {
    ..
    }
  }
}
```



## 1. SdpOfferAnswerHandler::ApplyLocalDescription

```cpp
RTCError SdpOfferAnswerHandler::ApplyLocalDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {
 ...
  if (IsUnifiedPlan()) {
   	...
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>> remove_list;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> removed_streams;
    for (const auto& transceiver : transceivers()->List()) {
      if (transceiver->stopped()) {
        continue;
      }

			// !!!!!!!!!!!!!!!
			// !!!!!!!!!!!!!!!
			// !!!!!!!!!!!!!!!
      // 2.2.7.1.1.(6-9): Set sender and receiver's transport slots.
      // Note that code paths that don't set MID won't be able to use
      // information about DTLS transports.
      if (transceiver->mid()) {
        // webrtc::DtlsTransport, dtls_transport
        auto dtls_transport = transport_controller()->LookupDtlsTransportByMid(
            *transceiver->mid());
        // send和 receiver 共用一个DtlsTransport
        transceiver->internal()->sender_internal()->set_transport(
            dtls_transport);
        transceiver->internal()->receiver_internal()->set_transport(
            dtls_transport);
      }
			// !!!!!!!!!!!!!!!
			// !!!!!!!!!!!!!!!
			// !!!!!!!!!!!!!!!
      ...
    }
    ...
  } else {
    ...
  }

  ...

  return RTCError::OK();
}
```



## 2. SdpOfferAnswerHandler::ApplyRemoteDescription

```cpp
RTCError SdpOfferAnswerHandler::ApplyRemoteDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {

...
  if (IsUnifiedPlan()) {
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>>
        now_receiving_transceivers;
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>> remove_list;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> added_streams;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> removed_streams;
    for (const auto& transceiver : transceivers()->List()) {
      const ContentInfo* content =
          FindMediaSectionForTransceiver(transceiver, remote_description());
      if (!content) {
        continue;
      }
      const MediaContentDescription* media_desc = content->media_description();
     	
      ...
        
      // 2.2.8.1.11: If description is of type "answer" or "pranswer", then run
      // the following steps:
      if (type == SdpType::kPrAnswer || type == SdpType::kAnswer) {
        // 2.2.8.1.11.1: Set transceiver's [[CurrentDirection]] slot to
        // direction.
        transceiver->internal()->set_current_direction(local_direction);
        // 2.2.8.1.11.[3-6]: Set the transport internal slots.
        //!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!
        if (transceiver->mid()) {
          // webrtc::DtlsTransport
          auto dtls_transport =
              transport_controller()->LookupDtlsTransportByMid(
                  *transceiver->mid());
          transceiver->internal()->sender_internal()->set_transport(
              dtls_transport);
          transceiver->internal()->receiver_internal()->set_transport(
              dtls_transport);
        }
        //!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!
      }
      ...
    }
   	...
  }

  ...

  return RTCError::OK();
}
```



## 3. webrtc::DtlsTransport 从哪里获取

### 3.1 JsepTransportController::LookupDtlsTransportByMid

根据mid 找到webrtc::DtlsTransport。

pc/jsep_transport_controller.cc

```cpp
// 返回 webrtc::DtlsTransport
rtc::scoped_refptr<webrtc::DtlsTransport>
JsepTransportController::LookupDtlsTransportByMid(const std::string& mid) {
  // JsepTransport
  auto jsep_transport = GetJsepTransportForMid(mid);
  if (!jsep_transport) {
    return nullptr;
  }
  return jsep_transport->RtpDtlsTransport();
}
```



### 3.2 JsepTransport::RtpDtlsTransport

pc/jsep_transport.h

返回的是webrtc::DtlsTransport， 其中有个属性是cricket::DtlsTransportInternal。

```cpp
rtc::scoped_refptr<webrtc::DtlsTransport> rtp_dtls_transport_
      RTC_GUARDED_BY(accessor_lock_);
////////////
rtc::scoped_refptr<webrtc::DtlsTransport> RtpDtlsTransport()
      RTC_LOCKS_EXCLUDED(accessor_lock_) {
    rtc::CritScope scope(&accessor_lock_);
    return rtp_dtls_transport_;
  }
```



## 4. DtlsTransport创建时机

```less
SdpOfferAnswerHandler::ApplyLocalDescription   SdpOfferAnswerHandler::ApplyRemoteDescription
SdpOfferAnswerHandler::PushdownTransportDescription
JsepTransportController::SetRemoteDescription   JsepTransportController::SetLocalDescription
JsepTransportController::ApplyDescription_n
JsepTransportController::MaybeCreateJsepTransport
JsepTransportController::CreateDtlsTransport
```

在`SdpOfferAnswerHandler::PushdownTransportDescription` 调用`JsepTransportController` 创建JsepTransport，在创建JsepTransport的时机，需要把DtlsTransport 作为构造的参数，保存到JsepTransport中。



### 4.1 JsepTransportController::MaybeCreateJsepTransport

创建了cricket::DtlsTransportInternal， 创建了JsepTransport。

pc/jsep_transport_controller.cc

```cpp
RTCError JsepTransportController::MaybeCreateJsepTransport(
    bool local,
    const cricket::ContentInfo& content_info,
    const cricket::SessionDescription& description) {
  ...
    
	//!!!!!!!!!!
	//!!!!!!!!!!
	//!!!!!!!!!!
  // 创建 DtlsTransportInternal
  std::unique_ptr<cricket::DtlsTransportInternal> rtp_dtls_transport =
      CreateDtlsTransport(content_info, ice->internal());
	//!!!!!!!!!!
	//!!!!!!!!!!
	//!!!!!!!!!!

  std::unique_ptr<cricket::DtlsTransportInternal> rtcp_dtls_transport;
  std::unique_ptr<RtpTransport> unencrypted_rtp_transport;
  std::unique_ptr<SrtpTransport> sdes_transport;
  std::unique_ptr<DtlsSrtpTransport> dtls_srtp_transport;
  std::unique_ptr<RtpTransportInternal> datagram_rtp_transport;

  rtc::scoped_refptr<webrtc::IceTransportInterface> rtcp_ice;
  if (config_.rtcp_mux_policy !=
          PeerConnectionInterface::kRtcpMuxPolicyRequire &&
      content_info.type == cricket::MediaProtocolType::kRtp) {
    rtcp_ice = CreateIceTransport(content_info.name, /*rtcp=*/true);
    rtcp_dtls_transport =
        CreateDtlsTransport(content_info, rtcp_ice->internal());
  }

  if (config_.disable_encryption) {
    RTC_LOG(LS_INFO)
        << "Creating UnencryptedRtpTransport, becayse encryption is disabled.";
    unencrypted_rtp_transport = CreateUnencryptedRtpTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
  } else if (!content_desc->cryptos().empty()) {
    sdes_transport = CreateSdesTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
    RTC_LOG(LS_INFO) << "Creating SdesTransport.";
  } else {
    RTC_LOG(LS_INFO) << "Creating DtlsSrtpTransport.";
    dtls_srtp_transport = CreateDtlsSrtpTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
  }

  std::unique_ptr<cricket::SctpTransportInternal> sctp_transport;
  if (config_.sctp_factory) {
    sctp_transport =
        config_.sctp_factory->CreateSctpTransport(rtp_dtls_transport.get());
  }

	//!!!!!!!!!!
	//!!!!!!!!!!
	//!!!!!!!!!!
  // 创建JsepTransport
  std::unique_ptr<cricket::JsepTransport> jsep_transport =
      std::make_unique<cricket::JsepTransport>(
          content_info.name, certificate_, std::move(ice), std::move(rtcp_ice),
          std::move(unencrypted_rtp_transport), std::move(sdes_transport),
          std::move(dtls_srtp_transport), std::move(datagram_rtp_transport),
    			//!!!!!!!!!!!
   				// 把DtlsTransportInternal 保存到JsepTransport
          std::move(rtp_dtls_transport),
    			//!!!!!!!!!!!
    			std::move(rtcp_dtls_transport),
          std::move(sctp_transport));
	//!!!!!!!!!!
	//!!!!!!!!!!
	//!!!!!!!!!!

  jsep_transport->rtp_transport()->SignalRtcpPacketReceived.connect(
      this, &JsepTransportController::OnRtcpPacketReceived_n);

  jsep_transport->SignalRtcpMuxActive.connect(
      this, &JsepTransportController::UpdateAggregateStates_n);
  SetTransportForMid(content_info.name, jsep_transport.get());

  jsep_transports_by_name_[content_info.name] = std::move(jsep_transport);
  UpdateAggregateStates_n();
  return RTCError::OK();
}
```



### !!! 4.2 JsepTransport::JsepTransport

pc/jsep_transport.h

把cricket::DtlsTransportInternal 就是cricket::DtlsTransport 做为参数传入到JsepTransport， 在创建了webrtc::DtlsTransport。
所以webrtc::DtlsTransport 封装了cricket::DtlsTransportInternal。

```cpp
JsepTransport::JsepTransport(
    const std::string& mid,
    const rtc::scoped_refptr<rtc::RTCCertificate>& local_certificate,
    rtc::scoped_refptr<webrtc::IceTransportInterface> ice_transport,
    rtc::scoped_refptr<webrtc::IceTransportInterface> rtcp_ice_transport,
    std::unique_ptr<webrtc::RtpTransport> unencrypted_rtp_transport,
    std::unique_ptr<webrtc::SrtpTransport> sdes_transport,
    std::unique_ptr<webrtc::DtlsSrtpTransport> dtls_srtp_transport,
    std::unique_ptr<webrtc::RtpTransportInternal> datagram_rtp_transport,
    std::unique_ptr<DtlsTransportInternal> rtp_dtls_transport,
    std::unique_ptr<DtlsTransportInternal> rtcp_dtls_transport,
    std::unique_ptr<SctpTransportInternal> sctp_transport)
    : 
		// 创建了webrtc::DtlsTransport，把cricket::DtlsTransportInternal作为构造参数
		// 注意 cricket::DtlsTransportInternal和 webrtc::DtlsTransport 是不同的。
      rtp_dtls_transport_(
          rtp_dtls_transport ? new rtc::RefCountedObject<webrtc::DtlsTransport>(
                                   std::move(rtp_dtls_transport))
                             : nullptr),
                             ...
                             {
                             ...
                             }
```



### 4.3 JsepTransportController::CreateDtlsTransport

pc/jsep_transport_controller.cc

创建了cricket::DtlsTransport， 父类就是 cricket::DtlsTransportInternal

```cpp
// 返回cricket::DtlsTransportInternal
std::unique_ptr<cricket::DtlsTransportInternal>
JsepTransportController::CreateDtlsTransport(
    const cricket::ContentInfo& content_info,
    cricket::IceTransportInternal* ice) {
  RTC_DCHECK(network_thread_->IsCurrent());

  std::unique_ptr<cricket::DtlsTransportInternal> dtls;
	// config_.dtls_transport_factory 是空值，没有赋值
  if (config_.dtls_transport_factory) {
    ...
  } else {
    // 创建cricket::DtlsTransport
    dtls = std::make_unique<cricket::DtlsTransport>(ice, config_.crypto_options,
                                                    config_.event_log);
  }

  ...
  return dtls;
}
```



## 
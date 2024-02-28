---
layout: post
title: Transceiver和BaseChannel 绑定
date: 2024-02-27 06:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq
---


* content
{:toc}

---

## Transceiver和BaseChannel 绑定

在SetLocalDescription的流程中， 根据createoffer 回调的 `SessionDescriptionInterface` （实现类是JsepSessionDescription） ， 根据`JsepSessionDescription::ContentInfos`, `ContentInfo::MediaContentDescrption`先更新好transceiver的mid和mline_index， 然后为每个Transceiver 对应创建一个BaseChannel，`RtpTransceiver::SetChannel` 把BaseChannel绑定到RtpTransceiver。解除绑定也是通过这个接口。

### 1. SdpOfferAnswerHandler::UpdateTransceiverChannel

pc/sdp_offer_answer.cc

```cpp
RTCError SdpOfferAnswerHandler::UpdateTransceiverChannel(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const cricket::ContentInfo& content,
    const cricket::ContentGroup* bundle_group) {
  RTC_DCHECK(IsUnifiedPlan());
  RTC_DCHECK(transceiver);
  cricket::ChannelInterface* channel = transceiver->internal()->channel();
  if (content.rejected) {
    if (channel) {
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      transceiver->internal()->SetChannel(nullptr);
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      DestroyChannelInterface(channel);
    }
  } else {
    if (!channel) {
      if (transceiver->media_type() == cricket::MEDIA_TYPE_AUDIO) {
        channel = CreateVoiceChannel(content.name);
      } else {
        RTC_DCHECK_EQ(cricket::MEDIA_TYPE_VIDEO, transceiver->media_type());
        channel = CreateVideoChannel(content.name);
      }
      if (!channel) {
        LOG_AND_RETURN_ERROR(
            RTCErrorType::INTERNAL_ERROR,
            "Failed to create channel for mid=" + content.name);
      }
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      transceiver->internal()->SetChannel(channel);
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
    }
  }
  return RTCError::OK();
}
```





### 2. RtpTransceiver::SetChannel

pc/rtp_transceiver.cc

```cpp
void RtpTransceiver::SetChannel(cricket::ChannelInterface* channel) {
  // Cannot set a non-null channel on a stopped transceiver.
  if (stopped_ && channel) {
    return;
  }

  if (channel) {
    RTC_DCHECK_EQ(media_type(), channel->media_type());
  }

  if (channel_) {
    channel_->SignalFirstPacketReceived().disconnect(this);
  }

  // BaseChannel
  channel_ = channel;

  if (channel_) {
    channel_->SignalFirstPacketReceived().connect(
        this, &RtpTransceiver::OnFirstPacketReceived);
  }

  // std::vector<rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>> senders_;
  for (const auto& sender : senders_) {
    sender->internal()->SetMediaChannel(channel_ ? channel_->media_channel()
                                                 : nullptr);
  }

  // std::vector<rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>> receivers_;
  for (const auto& receiver : receivers_) {
    if (!channel_) {
      receiver->internal()->Stop();
    }

    receiver->internal()->SetMediaChannel(channel_ ? channel_->media_channel()
                                                   : nullptr);
  }
}
```




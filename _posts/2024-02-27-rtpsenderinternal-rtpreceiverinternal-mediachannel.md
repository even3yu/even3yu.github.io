---
layout: post
title: RtpSenderInternal，RtpReceiverInternal 和MediaChannel绑定
date: 2024-02-27 02:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq
---


* content
{:toc}

---


## RtpSenderInternal，RtpReceiverInternal 和MediaChannel绑定

在SetLocalDescription的流程中， 根据createoffer 回调的 `SessionDescriptionInterface` （实现类是JsepSessionDescription） ， 根据`JsepSessionDescription::ContentInfos`, `ContentInfo::MediaContentDescrption`先更新好transceiver的mid和mline_index， 然后为每个Transceiver 对应创建一个BaseChannel，`RtpTransceiver::SetChannel` 把BaseChannel绑定到RtpTransceiver。

### 1. RtpTransceiver::SetChannel

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





### 2. RtpSenderBase::SetMediaChannel

```less
RtpSenderBase
AudioRtpSender  VideoRtpSender
```

pc/rtp_sender.cc

```cpp
void RtpSenderBase::SetMediaChannel(cricket::MediaChannel* media_channel) {
  RTC_DCHECK(media_channel == nullptr ||
             media_channel->media_type() == media_type());
  media_channel_ = media_channel;
}
```



### ------



### 3. AudioRtpReceiver::SetMediaChannel

```less
RtpReceiverInterface
RtpReceiverInternal
AudioRtpReceiver
```

pc/audio_rtp_receiver.cc

```cpp
void AudioRtpReceiver::SetMediaChannel(cricket::MediaChannel* media_channel) {
  RTC_DCHECK(media_channel == nullptr ||
             media_channel->media_type() == media_type());
  media_channel_ = static_cast<cricket::VoiceMediaChannel*>(media_channel);
}
```



### 4. VideoRtpReceiver::SetMediaChannel

pc/video_rtp_receiver.cc

```cpp
void VideoRtpReceiver::SetMediaChannel(cricket::MediaChannel* media_channel) {
  RTC_DCHECK(media_channel == nullptr ||
             media_channel->media_type() == media_type());
  worker_thread_->Invoke<void>(RTC_FROM_HERE, [&] {
    RTC_DCHECK_RUN_ON(worker_thread_);
    bool encoded_sink_enabled = saved_encoded_sink_enabled_;
    if (encoded_sink_enabled && media_channel_) {
      // Turn off the old sink, if any.
      SetEncodedSinkEnabled(false);
    }

    media_channel_ = static_cast<cricket::VideoMediaChannel*>(media_channel);

    if (media_channel_) {
      if (saved_generate_keyframe_) {
        // TODO(bugs.webrtc.org/8694): Stop using 0 to mean unsignalled SSRC
        media_channel_->GenerateKeyFrame(ssrc_.value_or(0));
        saved_generate_keyframe_ = false;
      }
      if (encoded_sink_enabled) {
        SetEncodedSinkEnabled(true);
      }
      if (frame_transformer_) {
        media_channel_->SetDepacketizerToDecoderFrameTransformer(
            ssrc_.value_or(0), frame_transformer_);
      }
    }
  });
}
```




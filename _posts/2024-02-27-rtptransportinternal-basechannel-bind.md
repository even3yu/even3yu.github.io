---
layout: post
title: RtpTransportInternal和 BaseChannel 绑定
date: 2024-02-27 05:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq
---


* content
{:toc}

---

## RtpTransportInternal和 BaseChannel 绑定

`RtpTransportInternal`  是在SetLocalDescription，调用`SdpOfferAnswerHandler::PushdownTransportDescription`，`JsepTransportController::SetLocalDescription`，根据配置会创建`RtpTransport`,`SrtpTransport`,`DtlsSrtpTransport` 。

`BaseChannel::SetRtpTransport`， 有两个地方调用

- BaseChannel::Init_w
- SdpOfferAnswerHandler::CreateVoiceChannel
- SdpOfferAnswerHandler::CreateVideoChannel

### 1. SdpOfferAnswerHandler::CreateVideoChannel

pc/sdp_offer_answer.cc

```cpp
cricket::VideoChannel* SdpOfferAnswerHandler::CreateVideoChannel(
    const std::string& mid) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  RtpTransportInternal* rtp_transport = pc_->GetRtpTransport(mid);
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!

  // TODO(bugs.webrtc.org/11992): CreateVideoChannel internally switches to the
  // worker thread. We shouldn't be using the |call_ptr_| hack here but simply
  // be on the worker thread and use |call_| (update upstream code).
  cricket::VideoChannel* video_channel;
  {
    RTC_DCHECK_RUN_ON(pc_->signaling_thread());
      //!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!
    video_channel = channel_manager()->CreateVideoChannel(
        pc_->call_ptr(), pc_->configuration()->media_config, rtp_transport,
        signaling_thread(), mid, pc_->SrtpRequired(), pc_->GetCryptoOptions(),
        &ssrc_generator_, video_options(),
        video_bitrate_allocator_factory_.get());
      //!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!
  }
  if (!video_channel) {
    return nullptr;
  }
  video_channel->SignalSentPacket().connect(pc_,
                                            &PeerConnection::OnSentPacket_w);
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  // BaseChannel
  video_channel->SetRtpTransport(rtp_transport);
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!

  return video_channel;
}
```





#### 1.1 ChannelManager::CreateVideoChannel

pc/channel_manager.cc

```cpp
VideoChannel* ChannelManager::CreateVideoChannel(
    webrtc::Call* call,
    const cricket::MediaConfig& media_config,
    webrtc::RtpTransportInternal* rtp_transport,
    rtc::Thread* signaling_thread,
    const std::string& content_name,
    bool srtp_required,
    const webrtc::CryptoOptions& crypto_options,
    rtc::UniqueRandomIdGenerator* ssrc_generator,
    const VideoOptions& options,
    webrtc::VideoBitrateAllocatorFactory* video_bitrate_allocator_factory) {
  // TODO(bugs.webrtc.org/11992): Remove this workaround after updates in
  // PeerConnection and add the expectation that we're already on the right
  // thread.
  if (!worker_thread_->IsCurrent()) {
    return worker_thread_->Invoke<VideoChannel*>(RTC_FROM_HERE, [&] {
      return CreateVideoChannel(call, media_config, rtp_transport,
                                signaling_thread, content_name, srtp_required,
                                crypto_options, ssrc_generator, options,
                                video_bitrate_allocator_factory);
    });
  }

  RTC_DCHECK_RUN_ON(worker_thread_);
  RTC_DCHECK(initialized_);
  RTC_DCHECK(call);
  if (!media_engine_) {
    return nullptr;
  }

  VideoMediaChannel* media_channel = media_engine_->video().CreateMediaChannel(
      call, media_config, options, crypto_options,
      video_bitrate_allocator_factory);
  if (!media_channel) {
    return nullptr;
  }

  auto video_channel = std::make_unique<VideoChannel>(
      worker_thread_, network_thread_, signaling_thread,
      absl::WrapUnique(media_channel), content_name, srtp_required,
      crypto_options, ssrc_generator);

  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  // VideoChannel video_channel
  video_channel->Init_w(rtp_transport);
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

  VideoChannel* video_channel_ptr = video_channel.get();
  video_channels_.push_back(std::move(video_channel));
  return video_channel_ptr;
}
```



#### 1.2 BaseChannel::Init_w

pc/channel.cc

```cpp
void BaseChannel::Init_w(webrtc::RtpTransportInternal* rtp_transport) {
  RTC_DCHECK_RUN_ON(worker_thread());

  network_thread_->Invoke<void>(
      RTC_FROM_HERE, [this, rtp_transport] { SetRtpTransport(rtp_transport); });

  // Both RTP and RTCP channels should be set, we can call SetInterface on
  // the media channel and it can set network options.
  media_channel_->SetInterface(this);
}
```



#### 1.3 BaseChannel::SetRtpTransport

pc/channel.cc

```cpp
bool BaseChannel::SetRtpTransport(webrtc::RtpTransportInternal* rtp_transport) {
  if (!network_thread_->IsCurrent()) {
    return network_thread_->Invoke<bool>(RTC_FROM_HERE, [this, rtp_transport] {
      return SetRtpTransport(rtp_transport);
    });
  }
  RTC_DCHECK_RUN_ON(network_thread());
  if (rtp_transport == rtp_transport_) {
    return true;
  }

  if (rtp_transport_) {
    DisconnectFromRtpTransport();
  }
  
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  rtp_transport_ = rtp_transport;
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  if (rtp_transport_) {
    transport_name_ = rtp_transport_->transport_name();

    if (!ConnectToRtpTransport()) {
      RTC_LOG(LS_ERROR) << "Failed to connect to the new RtpTransport for "
                        << ToString() << ".";
      return false;
    }
    OnTransportReadyToSend(rtp_transport_->IsReadyToSend());
    UpdateWritableState_n();

    // Set the cached socket options.
    for (const auto& pair : socket_options_) {
      rtp_transport_->SetRtpOption(pair.first, pair.second);
    }
    if (!rtp_transport_->rtcp_mux_enabled()) {
      for (const auto& pair : rtcp_socket_options_) {
        rtp_transport_->SetRtcpOption(pair.first, pair.second);
      }
    }
  }
  return true;
}
```





### 2. SdpOfferAnswerHandler::CreateVoiceChannel

```cpp
cricket::VoiceChannel* SdpOfferAnswerHandler::CreateVoiceChannel(
    const std::string& mid) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  RtpTransportInternal* rtp_transport = pc_->GetRtpTransport(mid);
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

  // TODO(bugs.webrtc.org/11992): CreateVoiceChannel internally switches to the
  // worker thread. We shouldn't be using the |call_ptr_| hack here but simply
  // be on the worker thread and use |call_| (update upstream code).
  cricket::VoiceChannel* voice_channel;
  {
    RTC_DCHECK_RUN_ON(pc_->signaling_thread());
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    voice_channel = channel_manager()->CreateVoiceChannel(
        pc_->call_ptr(), pc_->configuration()->media_config, rtp_transport,
        signaling_thread(), mid, pc_->SrtpRequired(), pc_->GetCryptoOptions(),
        &ssrc_generator_, audio_options());
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  }
  if (!voice_channel) {
    return nullptr;
  }
  voice_channel->SignalSentPacket().connect(pc_,
                                            &PeerConnection::OnSentPacket_w);
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  voice_channel->SetRtpTransport(rtp_transport);
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

  return voice_channel;
}
```



#### ChannelManager::CreateVoiceChannel

pc/channel_manager.cc

```cpp
VoiceChannel* ChannelManager::CreateVoiceChannel(
    webrtc::Call* call,
    const cricket::MediaConfig& media_config,
    webrtc::RtpTransportInternal* rtp_transport,
    rtc::Thread* signaling_thread,
    const std::string& content_name,
    bool srtp_required,
    const webrtc::CryptoOptions& crypto_options,
    rtc::UniqueRandomIdGenerator* ssrc_generator,
    const AudioOptions& options) {
  // TODO(bugs.webrtc.org/11992): Remove this workaround after updates in
  // PeerConnection and add the expectation that we're already on the right
  // thread.
  if (!worker_thread_->IsCurrent()) {
    return worker_thread_->Invoke<VoiceChannel*>(RTC_FROM_HERE, [&] {
      return CreateVoiceChannel(call, media_config, rtp_transport,
                                signaling_thread, content_name, srtp_required,
                                crypto_options, ssrc_generator, options);
    });
  }

  RTC_DCHECK_RUN_ON(worker_thread_);
  RTC_DCHECK(initialized_);
  RTC_DCHECK(call);
  if (!media_engine_) {
    return nullptr;
  }

  VoiceMediaChannel* media_channel = media_engine_->voice().CreateMediaChannel(
      call, media_config, options, crypto_options);
  if (!media_channel) {
    return nullptr;
  }

  auto voice_channel = std::make_unique<VoiceChannel>(
      worker_thread_, network_thread_, signaling_thread,
      absl::WrapUnique(media_channel), content_name, srtp_required,
      crypto_options, ssrc_generator);

  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  voice_channel->Init_w(rtp_transport);
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

  VoiceChannel* voice_channel_ptr = voice_channel.get();
  voice_channels_.push_back(std::move(voice_channel));
  return voice_channel_ptr;
}
```




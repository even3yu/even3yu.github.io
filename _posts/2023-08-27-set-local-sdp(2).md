---
layout: post
title: webrtc setLocalDescription(2)
date: 2023-08-27 23:36:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


![]({{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/view.png)



## 8. SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels(
    cricket::ContentSource source, // 
    const SessionDescriptionInterface& new_session, // new sdp
    const SessionDescriptionInterface* old_local_description, // old sdp
    const SessionDescriptionInterface* old_remote_description) { // remote sdp 为空
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(IsUnifiedPlan());

  const cricket::ContentGroup* bundle_group = nullptr;
  // 这里type 就是 offer
  if (new_session.GetType() == SdpType::kOffer) {
    // 是什么,a=group:BUNDLE audio video
    auto bundle_group_or_error =
        GetEarlyBundleGroup(*new_session.description());
    if (!bundle_group_or_error.ok()) {
      return bundle_group_or_error.MoveError();
    }
    bundle_group = bundle_group_or_error.MoveValue();
  }
  
  // ContentInfo 是什么？？？就是 a=mid:audio 相关的mline信息保存，ContentInfo 对应一个mline
  const ContentInfos& new_contents = new_session.description()->contents();
  for (size_t i = 0; i < new_contents.size(); ++i) {
    const cricket::ContentInfo& new_content = new_contents[i];
    cricket::MediaType media_type = new_content.media_description()->type();
    mid_generator_.AddKnownId(new_content.name);
    if (media_type == cricket::MEDIA_TYPE_AUDIO ||
        media_type == cricket::MEDIA_TYPE_VIDEO) {
      const cricket::ContentInfo* old_local_content = nullptr;
      if (old_local_description &&
          i < old_local_description->description()->contents().size()) {
        old_local_content =
            &old_local_description->description()->contents()[i];
      }
      const cricket::ContentInfo* old_remote_content = nullptr;
      if (old_remote_description &&
          i < old_remote_description->description()->contents().size()) {
        old_remote_content =
            &old_remote_description->description()->contents()[i];
      }
      // 找到transceiver，设置set_mid，set_mline_index
      auto transceiver_or_error =
          AssociateTransceiver(source, new_session.GetType(), i, new_content,
                               old_local_content, old_remote_content);
      if (!transceiver_or_error.ok()) {
        // In the case where a transceiver is rejected locally, we don't
        // expect to find a transceiver, but might find it in the case
        // where state is still "stopping", not "stopped".
        if (new_content.rejected) {
          continue;
        }
        return transceiver_or_error.MoveError();
      }
      auto transceiver = transceiver_or_error.MoveValue();
      // 设置Channel， 如果原先的没有，创建VoiceChannel 或者 VideoChannel【章节2.3】
      // 参数理解下？？？
      RTCError error =
          UpdateTransceiverChannel(transceiver, new_content, bundle_group);
      if (!error.ok()) {
        return error;
      }
    } else if (media_type == cricket::MEDIA_TYPE_DATA) {
      ...
    } else if (media_type == cricket::MEDIA_TYPE_UNSUPPORTED) {
      RTC_LOG(LS_INFO) << "Ignoring unsupported media type";
    } else {
      LOG_AND_RETURN_ERROR(RTCErrorType::INTERNAL_ERROR,
                           "Unknown section type.");
    }
  }

  return RTCError::OK();
}
```

1. 参数说明

   | 参数                                                      | 说明                                      |
   | --------------------------------------------------------- | ----------------------------------------- |
   | cricket::ContentSource source                             | 本地，还是远端， 这里是cricket::CS_LOCAL, |
   | const SessionDescriptionInterface& new_session            | createOffer的 回调的session               |
   | const SessionDescriptionInterface* old_local_description  | 上次createOffer的local_description        |
   | const SessionDescriptionInterface* old_remote_description | 这里是null，因为这里setLocalDescription   |

2. `ContentInfos& new_contents = new_session.description()->contents();`, 对所有的ContentInfo进行遍历；
   通过`AssociateTransceiver`找到对应的Transceiver，设置Channel， 如果原先的没有，创建VoiceChannel 或者 VideoChannel；

3. UpdateTransceiverChannel()方法中检查PC中的每个RtpTranceiver是存在MediaChannel，不存在的会调用WebRtcVideoEngine::CreateMediaChannel()创建WebRtcVideoChannel对象，并赋值给RtpTranceiver的RtpSender和RtpReceiver，这儿解决了VideoRtpSender的media_channel_成员为空的问题；

3. AssociateTransceiver， transceiver  是什么添加到列表的？？？RtpTransmissionManager::CreateAndAddTransceiver 创建和添加。其中就是AddTrack的时候添加。



### 8.1 SdpOfferAnswerHandler::AssociateTransceiver

pc/sdp_offer_answer.cc

```c++
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
SdpOfferAnswerHandler::AssociateTransceiver(
    cricket::ContentSource source,
    SdpType type,
    size_t mline_index,
    const ContentInfo& content,
    const ContentInfo* old_local_content,
    const ContentInfo* old_remote_content) {
  ...

  const MediaContentDescription* media_desc = content.media_description();
  // 根据 mid 找到对应的Transceiver
  auto transceiver = transceivers()->FindByMid(content.name);
  if (source == cricket::CS_LOCAL) {
    // Find the RtpTransceiver that corresponds to this m= section, using the
    // mapping between transceivers and m= section indices established when
    // creating the offer.
    if (!transceiver) {
      ????
      // transceiver 的mline_index 是在createoffer的时候赋值的，具体参考creatoffer
      transceiver = transceivers()->FindByMLineIndex(mline_index);
    }
    if (!transceiver) {
      // This may happen normally when media sections are rejected.
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "Transceiver not found based on m-line index");
    }
  } 

  ...
    // Associate the found or created RtpTransceiver with the m= section by
  // setting the value of the RtpTransceiver's mid property to the MID of the m=
  // section, and establish a mapping between the transceiver and the index of
  // the m= section.
  // 只有在这里会绑定 mid；
  
  // transceiver 是包了线程安全 RtpTransceiverProxy；
  // internal 就是 RtpTransceiver
  transceiver->internal()->set_mid(content.name);
  transceiver->internal()->set_mline_index(mline_index);
  return std::move(transceiver);
}
```

- 根据mid找transceiver。
- transceiver 设置 mid和 mline_index。
- transceiver 绑定mid；
- transceiver 的 mline_index 在createoffer 会进行赋值。比如是上一个sdpOffer的。



#### !!! RtpTransceiver::set_mid

#### RtpTransceiver::set_mline_index



### 8.2 !!! SdpOfferAnswerHandler::UpdateTransceiverChannel

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::UpdateTransceiverChannel(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const cricket::ContentInfo& content,
    const cricket::ContentGroup* bundle_group) {

  // transceiver 是包了线程安全 RtpTransceiverProxy；
  // internal 就是 RtpTransceiver
  cricket::ChannelInterface* channel = transceiver->internal()->channel();
  if (content.rejected) {
    if (channel) {
      // 与RtpTrasceiver解除绑定
      transceiver->internal()->SetChannel(nullptr);
      // 销毁channel
      DestroyChannelInterface(channel);
    }
  } else {
    if (!channel) {
      if (transceiver->media_type() == cricket::MEDIA_TYPE_AUDIO) {
        // 创建voicechannel
        channel = CreateVoiceChannel(content.name);
      } else {
        RTC_DCHECK_EQ(cricket::MEDIA_TYPE_VIDEO, transceiver->media_type());
        // 创建videochannel 【章节2.3.1】
        // content.name 是啥，就是a=mid:0,这里就是0
        channel = CreateVideoChannel(content.name);
      }
      if (!channel) {
        LOG_AND_RETURN_ERROR(
            RTCErrorType::INTERNAL_ERROR,
            "Failed to create channel for mid=" + content.name);
      }
      // VideoChannel 通知给RtpTransceiver， Transceiver 得到了VideoChannel
      // 【章节2.3.3】
      transceiver->internal()->SetChannel(channel);
    }
  }
  return RTCError::OK();
}
```

1. RtpTransceiver::channel, 返回的是BaseChannel；
   pc/rtp_transceiver.h

   ```cpp
     cricket::ChannelInterface* channel() const { return channel_; }
   ```

2. 如果ContentInfo.rejected = true， VideoChannel 和 RtpTransceiver绑定。DestroyChannelInterface

3. 创建voice/video channel

3.  **把voice/video channel 设置给 transceiver（RtpTransceiver），RtpTransceiver和BaseChannel就关联起来了**



#### 8.2.1 --SdpOfferAnswerHandler::CreateVideoChannel

pc/sdp_offer_answer.cc
创建 VideoChannel，chanel 和transport 绑定。参考章节9.

#### 8.2.2 SdpOfferAnswerHandler::CreateVoiceChannel

pc/sdp_offer_answer.cc

 

## 9. SdpOfferAnswerHandler::CreateVideoChannel

pc/sdp_offer_answer.cc

```c++
cricket::VideoChannel* SdpOfferAnswerHandler::CreateVideoChannel(
    const std::string& mid) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // 1.根据mid 获取RtpTransportInternal,  三种类型的Transport，
	//  RtpTransport，SrtpTransport，DtlsSrtpTransport
  RtpTransportInternal* rtp_transport = pc_->GetRtpTransport(mid);

  // TODO(bugs.webrtc.org/11992): CreateVideoChannel internally switches to the
  // worker thread. We shouldn't be using the |call_ptr_| hack here but simply
  // be on the worker thread and use |call_| (update upstream code).
  cricket::VideoChannel* video_channel;
  {
    RTC_DCHECK_RUN_ON(pc_->signaling_thread());
    // 2. 通过Channel Manager 创建VideoChannel
    // RtpTransportInternal *rtp_transport， 
    video_channel = channel_manager()->CreateVideoChannel(
        pc_->call_ptr(), pc_->configuration()->media_config, rtp_transport,
        signaling_thread(), mid, pc_->SrtpRequired(), pc_->GetCryptoOptions(),
        &ssrc_generator_, video_options(),
        video_bitrate_allocator_factory_.get());
  }
  if (!video_channel) {
    return nullptr;
  }
  video_channel->SignalSentPacket().connect(pc_,
                                            &PeerConnection::OnSentPacket_w);
  // chanel 和transport 绑定
  video_channel->SetRtpTransport(rtp_transport);

  return video_channel;
}
```

1. 通过mid 找到`JsepTransportController.MaybeCreateJsepTransport` 创建的 RtpTransportInternal；
2. 创建 VideoChannel，
3. chanel 和transport 绑定
4. 



### 9.1 RtpTransportInternal::GetRtpTransport

pc/peer_connection.h

```c++
RtpTransportInternal* GetRtpTransport(const std::string& mid) {
    RTC_DCHECK_RUN_ON(signaling_thread());
  	// JsepTransportController transport_controller_
    auto rtp_transport = transport_controller_->GetRtpTransport(mid);
    RTC_DCHECK(rtp_transport);
    return rtp_transport;
  }
```

 JsepTransportController.MaybeCreateJsepTransport` 创建的transport，`JsepTransportController.SetTransportForMid` 放到向量中，以 mid为key，JsepTransport 为value。



#### JsepTransportController::GetRtpTransport

pc/jsep_transport_controller.cc

```c++
RtpTransportInternal* JsepTransportController::GetRtpTransport(
    const std::string& mid) const {

  // 返回JsepTransport
  auto jsep_transport = GetJsepTransportForMid(mid);
  if (!jsep_transport) {
    return nullptr;
  }
  // 返回RtpTransportInternal
  return jsep_transport->rtp_transport();
}
```



#### RtpTransportInternal.rtp_transport

pc/jsep_transport.h

```c++
webrtc::RtpTransportInternal* rtp_transport() const
      RTC_LOCKS_EXCLUDED(accessor_lock_) {
    rtc::CritScope scope(&accessor_lock_);
  
    if (composite_rtp_transport_) {
      return composite_rtp_transport_.get();
    } else if (datagram_rtp_transport_) {
      return datagram_rtp_transport_.get();
    } else {
      // 走了这里
      return default_rtp_transport();
    }
  }

// Returns the default (non-datagram) rtp transport, if any.
  webrtc::RtpTransportInternal* default_rtp_transport() const
      RTC_EXCLUSIVE_LOCKS_REQUIRED(accessor_lock_) {
    if (dtls_srtp_transport_) {
      return dtls_srtp_transport_.get();
    } else if (sdes_transport_) {
      return sdes_transport_.get();
    } else if (unencrypted_rtp_transport_) {
      return unencrypted_rtp_transport_.get();
    } else {
      return nullptr;
    }
  }
```



### 9.2 ChannelManager.CreateVideoChannel

pc/channel_manager.cc
创建VideoChannel。



### 9.3 !!! BaseChannel::SetRtpTransport

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

  rtp_transport_ = rtp_transport;
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



### 9.3 RtpTransceiver.setChannel

VideoChannel 带着WebRtcVideoChannel 给到RtpTransceiver

在【章节2.3.0】updateTransceiverChannel的时候 setChannel

```c++
// channel 就是VideoChannel
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

  // RtpTransceiver保存了 VideoChannel
  channel_ = channel;

  // 接收到第一rtp 包
  if (channel_) {
    channel_->SignalFirstPacketReceived().connect(
        this, &RtpTransceiver::OnFirstPacketReceived);
  }

  // unify plan,应该是只有一个sender，VideoRtpSender
  for (const auto& sender : senders_) {
    // 把WebRtcVoiceChannel/WebRtcVideoChannel 给 RtpSender
    // sender 是 RtpSenderProxyWithInternal<RtpSenderInternal>
    // internal 就是VideoRtpSender
    sender->internal()->SetMediaChannel(channel_ ? channel_->media_channel()
                                                 : nullptr);
  }

  for (const auto& receiver : receivers_) {
    if (!channel_) {
      receiver->internal()->Stop();
    }

    receiver->internal()->SetMediaChannel(channel_ ? channel_->media_channel()
                                                   : nullptr);
  }
}
```

1. sender_

   ```c++
   std::vector<rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>>
         senders_;
   ```



## 10. ChannelManager.CreateVideoChannel

pc/channel_manager.cc

```c++
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
  
  // 切换到worker_thread_
  if (!worker_thread_->IsCurrent()) {
    return worker_thread_->Invoke<VideoChannel*>(RTC_FROM_HERE, [&] {
      return CreateVideoChannel(call, media_config, rtp_transport,
                                signaling_thread, content_name, srtp_required,
                                crypto_options, ssrc_generator, options,
                                video_bitrate_allocator_factory);
    });
  }

   ...

  // 1. 通过CompositeMediaEngine（media_engine_） 获取 videoengine（WebRtcVideoEngine），
	// 创建CreateMediaChannel
  // WebRtcVideoChannel
  VideoMediaChannel* media_channel = media_engine_->video().CreateMediaChannel(
      call, media_config, options, crypto_options,
      video_bitrate_allocator_factory);
  if (!media_channel) {
    return nullptr;
  }

  // 2. 创建VideoChannel，把VideoMediaChannel传入了
  auto video_channel = std::make_unique<VideoChannel>(
      worker_thread_, network_thread_, signaling_thread,
      absl::WrapUnique(media_channel), content_name, srtp_required,
      crypto_options, ssrc_generator);

  // 3. ？？？？？
  video_channel->Init_w(rtp_transport);

  VideoChannel* video_channel_ptr = video_channel.get();
  // 将VideoChannel 保存在video_channels_ 列表中
  video_channels_.push_back(std::move(video_channel));
  return video_channel_ptr;
}
```

1. 主要是创建MediaChannel，接着 创建VideoChannel；
   
1. std::vector<std::unique_ptr<VoiceChannel>> voice_channels_;
   
   std::vector<std::unique_ptr<VideoChannel>> video_channels_;

2. media_engine_ 就是 CompositeMediaEngine, 内部可有videoengine，voiceengine。

3. media_engine_->video() 返回的是 WebRtcVideoEngine
   
   ```c++
   VoiceEngineInterface& CompositeMediaEngine::voice() {
     return *voice_engine_.get();
   }
   
   VideoEngineInterface& CompositeMediaEngine::video() {
     return *video_engine_.get();
   }
   ```



### 10.1 WebRtcVideoEngine.CreateMediaChannel

media/engine/webrtc_video_engine.h

```c++
VideoMediaChannel* WebRtcVideoEngine::CreateMediaChannel(
    webrtc::Call* call,
    const MediaConfig& config,
    const VideoOptions& options,
    const webrtc::CryptoOptions& crypto_options,
    webrtc::VideoBitrateAllocatorFactory* video_bitrate_allocator_factory) {
  RTC_LOG(LS_INFO) << "CreateMediaChannel. Options: " << options.ToString();
  return new WebRtcVideoChannel(call, config, options, crypto_options,
                                encoder_factory_.get(), decoder_factory_.get(),
                                video_bitrate_allocator_factory);
}
```

创建WebRtcVideoChannel，就是MediaChannel。

```
MediaChannel
VideoMediaChannel				webrtc::Transport
WebRtcVideoChannel
```



#### 10.1.1 WebRtcVideoChannel::WebRtcVideoChannel

media/engine/webrtc_video_engine.h

```c++
WebRtcVideoChannel::WebRtcVideoChannel(
    webrtc::Call* call,
    const MediaConfig& config,
    const VideoOptions& options,
    const webrtc::CryptoOptions& crypto_options,
    webrtc::VideoEncoderFactory* encoder_factory,
    webrtc::VideoDecoderFactory* decoder_factory,
    webrtc::VideoBitrateAllocatorFactory* bitrate_allocator_factory)
    : VideoMediaChannel(config),
      worker_thread_(rtc::Thread::Current()),
      call_(call),
      unsignalled_ssrc_handler_(&default_unsignalled_ssrc_handler_),
      video_config_(config.video),
      encoder_factory_(encoder_factory),
      decoder_factory_(decoder_factory),
      bitrate_allocator_factory_(bitrate_allocator_factory),
      default_send_options_(options),
      last_stats_log_ms_(-1),
      discard_unknown_ssrc_packets_(
          IsEnabled(call_->trials(),
                    "WebRTC-Video-DiscardPacketsWithUnknownSsrc")),
      crypto_options_(crypto_options),
      unknown_ssrc_packet_buffer_(
          IsEnabled(call_->trials(),
                    "WebRTC-Video-BufferPacketsWithUnknownSsrc")
              ? new UnhandledPacketsBuffer()
              : nullptr) {///////////////////////
  RTC_DCHECK(thread_checker_.IsCurrent());

  rtcp_receiver_report_ssrc_ = kDefaultRtcpReceiverReportSsrc;
  sending_ = false;
  recv_codecs_ = MapCodecs(GetPayloadTypesAndDefaultCodecs(
      decoder_factory_, /*is_decoder_factory=*/true, call_->trials()));
  recv_flexfec_payload_type_ =
      recv_codecs_.empty() ? 0 : recv_codecs_.front().flexfec_payload_type;
}
```



### 10.2 VideoChannel::VideoChannel

pc/channel_manager.h

```c++
VideoChannel::VideoChannel(rtc::Thread* worker_thread,
                           rtc::Thread* network_thread,
                           rtc::Thread* signaling_thread,
                           std::unique_ptr<VideoMediaChannel> media_channel,
                           const std::string& content_name,
                           bool srtp_required,
                           webrtc::CryptoOptions crypto_options,
                           UniqueRandomIdGenerator* ssrc_generator)
    : BaseChannel(worker_thread,
                  network_thread,
                  signaling_thread,
                  std::move(media_channel),
                  content_name,
                  srtp_required,
                  crypto_options,
                  ssrc_generator) {}
```

1. 拥有VideoMediaChannel， 就是MediaChannel；
2. BaseChannel
   ----》 VideoChannel/VoiceChannel；



### 10.3 VideoChannel::Init_w

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



#### 10.3.1 ???MediaChannel::SetRtpTransport

media/base/media_channel.cc



#### 10.3.2 !!! MediaChannel::SetInterface

media/base/media_channel.cc

```cpp
void MediaChannel::SetInterface(NetworkInterface* iface) {
  webrtc::MutexLock lock(&network_interface_mutex_);
  network_interface_ = iface;
  UpdateDscp();
}
```


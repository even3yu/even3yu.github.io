---
layout: post
title: rtp 扩展头 sdp 协商
date: 2023-11-27 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtp sdp rtpxh
---


* content
{:toc}

---


## 0. 前言

1. addtrack的时候，根据audio、video 添加rtp header extension
2. createoffer 的时候，过滤了rtp header extension， 包括direct，以及unified plan等
3. setLocalDescription/setRemoteDescription 注册rtp header extension



## 1. RTPExtensionType

modules/rtp_rtcp/include/rtp_rtcp_defines.h

```cpp
enum RTPExtensionType : int {
  kRtpExtensionNone,
  kRtpExtensionTransmissionTimeOffset,
  kRtpExtensionAudioLevel,
  kRtpExtensionInbandComfortNoise,
  kRtpExtensionAbsoluteSendTime,
  kRtpExtensionAbsoluteCaptureTime,
  kRtpExtensionVideoRotation,
  kRtpExtensionTransportSequenceNumber,
  kRtpExtensionTransportSequenceNumber02,
  kRtpExtensionPlayoutDelay,
  kRtpExtensionVideoContentType,
  kRtpExtensionVideoLayersAllocation,
  kRtpExtensionVideoTiming,
  kRtpExtensionRtpStreamId,
  kRtpExtensionRepairedRtpStreamId,
  kRtpExtensionMid,
  kRtpExtensionGenericFrameDescriptor00,
  kRtpExtensionGenericFrameDescriptor = kRtpExtensionGenericFrameDescriptor00,
  kRtpExtensionGenericFrameDescriptor02,
  kRtpExtensionColorSpace,
  kRtpExtensionNumberOfExtensions  // Must be the last entity in the enum.
};
```





## 2. 获取默认的extension扩展头配置

```less
PeerConnection::AddTrack
RtpTransmissionManager::AddTrack
RtpTransmissionManager::AddTrackUnifiedPlan
RtpTransmissionManager::CreateAndAddTransceiver
	ChannelManager::GetSupportedAudioRtpHeaderExtensions
	RtpTransceiver::RtpTransceiver
```





### RtpTransmissionManager::CreateAndAddTransceiver

pc/peer_connection.cc

```cpp
rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
RtpTransmissionManager::CreateAndAddTransceiver(
    rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>> sender,
    rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>
        receiver) {
	...
  auto transceiver = RtpTransceiverProxyWithInternal<RtpTransceiver>::Create(
      signaling_thread(),
      new RtpTransceiver(
          sender, receiver, channel_manager(),
          sender->media_type() == cricket::MEDIA_TYPE_AUDIO
              ? channel_manager()->GetSupportedAudioRtpHeaderExtensions()
              : channel_manager()->GetSupportedVideoRtpHeaderExtensions(),
          [this_weak_ptr = weak_ptr_factory_.GetWeakPtr()]() {
            if (this_weak_ptr) {
              this_weak_ptr->OnNegotiationNeeded();
            }
          }));
  transceivers()->Add(transceiver);
  return transceiver;
}
```



#### ChannelManager::GetSupportedAudioRtpHeaderExtensions

pc/channel_manager.cc

```cpp
std::vector<webrtc::RtpHeaderExtensionCapability>
ChannelManager::GetSupportedAudioRtpHeaderExtensions() const {
  if (!media_engine_)
    return {};
  return media_engine_->voice().GetRtpHeaderExtensions();
}
```



##### !!! WebRtcVoiceEngine::GetRtpHeaderExtensions

media/engine/webrtc_voice_engine.cc

```cpp
std::vector<webrtc::RtpHeaderExtensionCapability>
WebRtcVoiceEngine::GetRtpHeaderExtensions() const {
  RTC_DCHECK(signal_thread_checker_.IsCurrent());
  std::vector<webrtc::RtpHeaderExtensionCapability> result;
  int id = 1;
  for (const auto& uri :
       {webrtc::RtpExtension::kAudioLevelUri,
        webrtc::RtpExtension::kAbsSendTimeUri,
        webrtc::RtpExtension::kTransportSequenceNumberUri,
        webrtc::RtpExtension::kMidUri, webrtc::RtpExtension::kRidUri,
        webrtc::RtpExtension::kRepairedRidUri}) {
    result.emplace_back(uri, id++, webrtc::RtpTransceiverDirection::kSendRecv);
  }
  return result;
}
```

获取音频默认扩展头



#### ChannelManager::GetSupportedVideoRtpHeaderExtensions

pc/channel_manager.cc

```cpp
std::vector<webrtc::RtpHeaderExtensionCapability>
ChannelManager::GetSupportedVideoRtpHeaderExtensions() const {
  if (!media_engine_)
    return {};
  return media_engine_->video().GetRtpHeaderExtensions();
}
```

##### !!! WebRtcVideoEngine::GetRtpHeaderExtensions

media/engine/webrtc_video_engine.cc

```cpp
std::vector<webrtc::RtpHeaderExtensionCapability>
WebRtcVideoEngine::GetRtpHeaderExtensions() const {
  std::vector<webrtc::RtpHeaderExtensionCapability> result;
  int id = 1;
  for (const auto& uri :
       {webrtc::RtpExtension::kTimestampOffsetUri,
        webrtc::RtpExtension::kAbsSendTimeUri,
        webrtc::RtpExtension::kVideoRotationUri,
        webrtc::RtpExtension::kTransportSequenceNumberUri,
        webrtc::RtpExtension::kPlayoutDelayUri,
        webrtc::RtpExtension::kVideoContentTypeUri,
        webrtc::RtpExtension::kVideoTimingUri,
        webrtc::RtpExtension::kColorSpaceUri, webrtc::RtpExtension::kMidUri,
        webrtc::RtpExtension::kRidUri, webrtc::RtpExtension::kRepairedRidUri}) {
    result.emplace_back(uri, id++, webrtc::RtpTransceiverDirection::kSendRecv);
  }
  result.emplace_back(webrtc::RtpExtension::kGenericFrameDescriptorUri00, id++,
                      IsEnabled(trials_, "WebRTC-GenericDescriptorAdvertised")
                          ? webrtc::RtpTransceiverDirection::kSendRecv
                          : webrtc::RtpTransceiverDirection::kStopped);
  result.emplace_back(
      webrtc::RtpExtension::kDependencyDescriptorUri, id++,
      IsEnabled(trials_, "WebRTC-DependencyDescriptorAdvertised")
          ? webrtc::RtpTransceiverDirection::kSendRecv
          : webrtc::RtpTransceiverDirection::kStopped);

  result.emplace_back(
      webrtc::RtpExtension::kVideoLayersAllocationUri, id++,
      IsEnabled(trials_, "WebRTC-VideoLayersAllocationAdvertised")
          ? webrtc::RtpTransceiverDirection::kSendRecv
          : webrtc::RtpTransceiverDirection::kStopped);

  return result;
}
```

获取视频默认扩展头

![img_v3_025l_c81db9d0-d8d6-4a03-a283-6e9e41e07e9g]({{ site.url }}{{ site.baseurl }}/images/rtp-header-extesion-2.assets/extension-default.png)

### RtpTransceiver::RtpTransceiver

pc/rtp_transceiver.cc

```cpp
RtpTransceiver::RtpTransceiver(
    rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>> sender,
    rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>
        receiver,
    cricket::ChannelManager* channel_manager,
////////////////
    std::vector<RtpHeaderExtensionCapability> header_extensions_offered,
////////////////
    std::function<void()> on_negotiation_needed)
    : thread_(GetCurrentTaskQueueOrThread()),
      unified_plan_(true),
      media_type_(sender->media_type()),
      channel_manager_(channel_manager),
////////////////
      header_extensions_to_offer_(std::move(header_extensions_offered)),
////////////////
      on_negotiation_needed_(std::move(on_negotiation_needed)) {
  RTC_DCHECK(media_type_ == cricket::MEDIA_TYPE_AUDIO ||
             media_type_ == cricket::MEDIA_TYPE_VIDEO);
  RTC_DCHECK_EQ(sender->media_type(), receiver->media_type());
  senders_.push_back(sender);
  receivers_.push_back(receiver);
}
```



## 3. 将extension扩展头封装到sdp中

```less
PeerConnection::CreateOffer
SdpOfferAnswerHandler::CreateOffer
SdpOfferAnswerHandler::GetOptionsForOffer
SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer
MediaDescriptionOptions::GetMediaDescriptionOptionsForTransceiver
RtpTransceiver::HeaderExtensionsToOffer
```



### MediaSessionDescriptionFactory::CreateOffer

pc/media_session.cc

```cpp
std::unique_ptr<SessionDescription> MediaSessionDescriptionFactory::CreateOffer(
    const MediaSessionOptions& session_options,
    const SessionDescription* current_description) const {
	...

  AudioVideoRtpHeaderExtensions extensions_with_ids =
      GetOfferedRtpHeaderExtensionsWithIds(
          current_active_contents, session_options.offer_extmap_allow_mixed,
          session_options.media_description_options);
          ...
}
```

这里会把direct 是stop过滤掉，还有sdp如果是planB，则会把unified的过掉。

#### MediaSessionDescriptionFactory::GetOfferedRtpHeaderExtensionsWithIds

pc/media_session.cc

```cpp
MediaSessionDescriptionFactory::AudioVideoRtpHeaderExtensions
MediaSessionDescriptionFactory::GetOfferedRtpHeaderExtensionsWithIds(
    const std::vector<const ContentInfo*>& current_active_contents,
    bool extmap_allow_mixed,
    const std::vector<MediaDescriptionOptions>& media_description_options)
    const {
  // All header extensions allocated from the same range to avoid potential
  // issues when using BUNDLE.

  // Strictly speaking the SDP attribute extmap_allow_mixed signals that the
  // receiver supports an RTP stream where one- and two-byte RTP header
  // extensions are mixed. For backwards compatibility reasons it's used in
  // WebRTC to signal that two-byte RTP header extensions are supported.
  UsedRtpHeaderExtensionIds used_ids(
      extmap_allow_mixed ? UsedRtpHeaderExtensionIds::IdDomain::kTwoByteAllowed
                         : UsedRtpHeaderExtensionIds::IdDomain::kOneByteOnly);
  RtpHeaderExtensions all_regular_extensions;
  RtpHeaderExtensions all_encrypted_extensions;

  AudioVideoRtpHeaderExtensions offered_extensions;
  // First - get all extensions from the current description if the media type
  // is used.
  // Add them to |used_ids| so the local ids are not reused if a new media
  // type is added.
  for (const ContentInfo* content : current_active_contents) {
    if (IsMediaContentOfType(content, MEDIA_TYPE_AUDIO)) {
      const AudioContentDescription* audio =
          content->media_description()->as_audio();
      MergeRtpHdrExts(audio->rtp_header_extensions(), &offered_extensions.audio,
                      &all_regular_extensions, &all_encrypted_extensions,
                      &used_ids);
    } else if (IsMediaContentOfType(content, MEDIA_TYPE_VIDEO)) {
      const VideoContentDescription* video =
          content->media_description()->as_video();
      MergeRtpHdrExts(video->rtp_header_extensions(), &offered_extensions.video,
                      &all_regular_extensions, &all_encrypted_extensions,
                      &used_ids);
    }
  }

  // Add all encountered header extensions in the media description options that
  // are not in the current description.

  for (const auto& entry : media_description_options) {
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    RtpHeaderExtensions filtered_extensions =
        filtered_rtp_header_extensions(UnstoppedOrPresentRtpHeaderExtensions(
            entry.header_extensions, all_regular_extensions,
            all_encrypted_extensions));
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    if (entry.type == MEDIA_TYPE_AUDIO)
      MergeRtpHdrExts(filtered_extensions, &offered_extensions.audio,
                      &all_regular_extensions, &all_encrypted_extensions,
                      &used_ids);
    else if (entry.type == MEDIA_TYPE_VIDEO)
      MergeRtpHdrExts(filtered_extensions, &offered_extensions.video,
                      &all_regular_extensions, &all_encrypted_extensions,
                      &used_ids);
  }
  // TODO(jbauch): Support adding encrypted header extensions to existing
  // sessions.
  if (enable_encrypted_rtp_header_extensions_ &&
      current_active_contents.empty()) {
    AddEncryptedVersionsOfHdrExts(&offered_extensions.audio,
                                  &all_encrypted_extensions, &used_ids);
    AddEncryptedVersionsOfHdrExts(&offered_extensions.video,
                                  &all_encrypted_extensions, &used_ids);
  }
  return offered_extensions;
}
```





#### UnstoppedOrPresentRtpHeaderExtensions

```cpp
cricket::RtpHeaderExtensions UnstoppedOrPresentRtpHeaderExtensions(
    const std::vector<webrtc::RtpHeaderExtensionCapability>& capabilities,
    const cricket::RtpHeaderExtensions& unencrypted,
    const cricket::RtpHeaderExtensions& encrypted) {
  cricket::RtpHeaderExtensions extensions;
  for (const auto& capability : capabilities) {
      // 根据direct 过滤
    if (capability.direction != RtpTransceiverDirection::kStopped ||
        IsCapabilityPresent(capability, unencrypted) ||
        IsCapabilityPresent(capability, encrypted)) {
      extensions.push_back(RtpExtensionFromCapability(capability));
    }
  }
  return extensions;
}
```

- 有三个extension，根据开放的功能，来确定direct，参考`WebRtcVideoEngine::GetRtpHeaderExtensions`

```less
webrtc::RtpExtension::kGenericFrameDescriptorUri00
webrtc::RtpExtension::kDependencyDescriptorUri,
webrtc::RtpExtension::kVideoLayersAllocationUri,
```





#### MediaSessionDescriptionFactory::filtered_rtp_header_extensions

```cpp
RtpHeaderExtensions
MediaSessionDescriptionFactory::filtered_rtp_header_extensions(
    RtpHeaderExtensions extensions) const {
  // 如果是planb 去掉unified 相关的
  if (!is_unified_plan_) {
    RemoveUnifiedPlanExtensions(&extensions);
  }
  return extensions;
}
```



##### RemoveUnifiedPlanExtensions

```cpp
static void RemoveUnifiedPlanExtensions(RtpHeaderExtensions* extensions) {
  RTC_DCHECK(extensions);

  extensions->erase(
      std::remove_if(extensions->begin(), extensions->end(),
                     [](auto extension) {
                       return extension.uri == webrtc::RtpExtension::kMidUri ||
                              extension.uri == webrtc::RtpExtension::kRidUri ||
                              extension.uri ==
                                  webrtc::RtpExtension::kRepairedRidUri;
                     }),
      extensions->end());
}
```

```less
webrtc::RtpExtension::kMidUri
webrtc::RtpExtension::kRidUri
webrtc::RtpExtension::kRepairedRidUri
```







### SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer

```cpp
void SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer(
    const RTCOfferAnswerOptions& offer_answer_options,
    cricket::MediaSessionOptions* session_options) {
  ...

  // Next, look for transceivers that are newly added (that is, are not stopped
  // and not associated). Reuse media sections marked as recyclable first,
  // otherwise append to the end of the offer. New media sections should be
  // added in the order they were added to the PeerConnection.
  for (const auto& transceiver : transceivers()->List()) {
    if (transceiver->mid() || transceiver->stopping()) {
      continue;
    }
    size_t mline_index;
    if (!recycleable_mline_indices.empty()) {
      mline_index = recycleable_mline_indices.front();
      recycleable_mline_indices.pop();
      session_options->media_description_options[mline_index] =
          GetMediaDescriptionOptionsForTransceiver(
              transceiver, mid_generator_.GenerateString(),
              /*is_create_offer=*/true);
    } else {
      mline_index = session_options->media_description_options.size();
      session_options->media_description_options.push_back(
          GetMediaDescriptionOptionsForTransceiver(
              transceiver, mid_generator_.GenerateString(),
              /*is_create_offer=*/true));
    }
    // See comment above for why CreateOffer changes the transceiver's state.
    transceiver->internal()->set_mline_index(mline_index);
  }

}
```



### GetMediaDescriptionOptionsForTransceiver

pc/sdp_offer_answer.cc

```cpp
static cricket::MediaDescriptionOptions
GetMediaDescriptionOptionsForTransceiver(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const std::string& mid,
    bool is_create_offer) {
 ...
  media_description_options.header_extensions =
      transceiver->HeaderExtensionsToOffer();
  ...

  return media_description_options;
}
```



### RtpTransceiver::HeaderExtensionsToOffer

pc/rtp_transceiver.cc

```cpp
std::vector<RtpHeaderExtensionCapability>
RtpTransceiver::HeaderExtensionsToOffer() const {
  return header_extensions_to_offer_;
}
```



## 4. 解析sdp的extension，并注册ExtensionMap

```less
SdpOfferAnswerHandler::SetLocalDescription    
SdpOfferAnswerHandler::DoSetLocalDescription
SdpOfferAnswerHandler::ApplyLocalDescription   
SdpOfferAnswerHandler::UpdateSessionState  
SdpOfferAnswerHandler::PushdownMediaDescription
BaseChannel::SetRemoteContent
VideoChannel::SetRemoteContent_w / VoiceChannel::SetRemoteContent_w
```



### 音频

```less
VoiceChannel::SetRemoteContent_w
WebRtcVoiceMediaChannel::SetSendParameters
AudioSendStream::ConfigureStream
ModuleRtpRtcpImpl2::RegisterRtpHeaderExtension
RTPSender::RegisterRtpHeaderExtension
RtpHeaderExtensionMap::RegisterByUri
RtpHeaderExtensionMap::Register
```



#### AudioSendStream::ConfigureStream

audio/audio_send_stream.cc

```cpp
void AudioSendStream::ConfigureStream(
    const webrtc::AudioSendStream::Config& new_config,
    bool first_time) {
  RTC_LOG(LS_INFO) << "AudioSendStream::ConfigureStream: "
                   << new_config.ToString();
  UpdateEventLogStreamConfig(event_log_, new_config,
                             first_time ? nullptr : &config_);

  const auto& old_config = config_;

  // Configuration parameters which cannot be changed.
  RTC_DCHECK(first_time ||
             old_config.send_transport == new_config.send_transport);
  RTC_DCHECK(first_time || old_config.rtp.ssrc == new_config.rtp.ssrc);
  if (suspended_rtp_state_ && first_time) {
    rtp_rtcp_module_->SetRtpState(*suspended_rtp_state_);
  }
  if (first_time || old_config.rtp.c_name != new_config.rtp.c_name) {
    channel_send_->SetRTCP_CNAME(new_config.rtp.c_name);
  }

  // Enable the frame encryptor if a new frame encryptor has been provided.
  if (first_time || new_config.frame_encryptor != old_config.frame_encryptor) {
    channel_send_->SetFrameEncryptor(new_config.frame_encryptor);
  }

  if (first_time ||
      new_config.frame_transformer != old_config.frame_transformer) {
    channel_send_->SetEncoderToPacketizerFrameTransformer(
        new_config.frame_transformer);
  }

  if (first_time ||
      new_config.rtp.extmap_allow_mixed != old_config.rtp.extmap_allow_mixed) {
    rtp_rtcp_module_->SetExtmapAllowMixed(new_config.rtp.extmap_allow_mixed);
  }

  const ExtensionIds old_ids = FindExtensionIds(old_config.rtp.extensions);
  const ExtensionIds new_ids = FindExtensionIds(new_config.rtp.extensions);

  // Audio level indication
  if (first_time || new_ids.audio_level != old_ids.audio_level) {
    channel_send_->SetSendAudioLevelIndicationStatus(new_ids.audio_level != 0,
                                                     new_ids.audio_level);
  }

  /////////1 AbsoluteSendTime::kUri
  if (first_time || new_ids.abs_send_time != old_ids.abs_send_time) {
    rtp_rtcp_module_->DeregisterSendRtpHeaderExtension(
        kRtpExtensionAbsoluteSendTime);
    if (new_ids.abs_send_time) {
      //!!!!!!!!!!
      rtp_rtcp_module_->RegisterRtpHeaderExtension(AbsoluteSendTime::kUri,
                                                   new_ids.abs_send_time);
    }
  }

  /////////2 TransportSequenceNumber::kUri
  bool transport_seq_num_id_changed =
      new_ids.transport_sequence_number != old_ids.transport_sequence_number;
  if (first_time ||
      (transport_seq_num_id_changed && !allocate_audio_without_feedback_)) {
    if (!first_time) {
      channel_send_->ResetSenderCongestionControlObjects();
    }

    RtcpBandwidthObserver* bandwidth_observer = nullptr;

    if (!allocate_audio_without_feedback_ &&
        new_ids.transport_sequence_number != 0) {
      //!!!!!!!!!!
      rtp_rtcp_module_->RegisterRtpHeaderExtension(
          TransportSequenceNumber::kUri, new_ids.transport_sequence_number);
      // Probing in application limited region is only used in combination with
      // send side congestion control, wich depends on feedback packets which
      // requires transport sequence numbers to be enabled.
      // Optionally request ALR probing but do not override any existing
      // request from other streams.
      if (enable_audio_alr_probing_) {
        rtp_transport_->EnablePeriodicAlrProbing(true);
      }
      bandwidth_observer = rtp_transport_->GetBandwidthObserver();
    }
    channel_send_->RegisterSenderCongestionControlObjects(rtp_transport_,
                                                          bandwidth_observer);
  }
  /////////3 RtpMid::kUri
  // MID RTP header extension.
  if ((first_time || new_ids.mid != old_ids.mid ||
       new_config.rtp.mid != old_config.rtp.mid) &&
      new_ids.mid != 0 && !new_config.rtp.mid.empty()) {
      //!!!!!!!!!!
    rtp_rtcp_module_->RegisterRtpHeaderExtension(RtpMid::kUri, new_ids.mid);
    rtp_rtcp_module_->SetMid(new_config.rtp.mid);
  }

  /////////4 RtpStreamId::kUri
  // RID RTP header extension
  if ((first_time || new_ids.rid != old_ids.rid ||
       new_ids.repaired_rid != old_ids.repaired_rid ||
       new_config.rtp.rid != old_config.rtp.rid)) {
    if (new_ids.rid != 0 || new_ids.repaired_rid != 0) {
      if (new_config.rtp.rid.empty()) {
        rtp_rtcp_module_->DeregisterSendRtpHeaderExtension(RtpStreamId::kUri);
      } else if (new_ids.repaired_rid != 0) {
      //!!!!!!!!!!
        rtp_rtcp_module_->RegisterRtpHeaderExtension(RtpStreamId::kUri,
                                                     new_ids.repaired_rid);
      } else {
      //!!!!!!!!!!
        rtp_rtcp_module_->RegisterRtpHeaderExtension(RtpStreamId::kUri,
                                                     new_ids.rid);
      }
    }
    rtp_rtcp_module_->SetRid(new_config.rtp.rid);
  }

  /////////5 AbsoluteCaptureTimeExtension::kUri
  if (first_time || new_ids.abs_capture_time != old_ids.abs_capture_time) {
    rtp_rtcp_module_->DeregisterSendRtpHeaderExtension(
        kRtpExtensionAbsoluteCaptureTime);
    if (new_ids.abs_capture_time) {
      //!!!!!!!!!!
      rtp_rtcp_module_->RegisterRtpHeaderExtension(
          AbsoluteCaptureTimeExtension::kUri, new_ids.abs_capture_time);
    }
  }

  if (!ReconfigureSendCodec(new_config)) {
    RTC_LOG(LS_ERROR) << "Failed to set up send codec state.";
  }

  // Set currently known overhead (used in ANA, opus only).
  {
    MutexLock lock(&overhead_per_packet_lock_);
    UpdateOverheadForEncoder();
  }

  channel_send_->CallEncoder([this](AudioEncoder* encoder) {
    if (!encoder) {
      return;
    }
    worker_queue_->PostTask(
        [this, length_range = encoder->GetFrameLengthRange()] {
          RTC_DCHECK_RUN_ON(worker_queue_);
          frame_length_range_ = length_range;
        });
  });

  if (sending_) {
    ReconfigureBitrateObserver(new_config);
  }
  config_ = new_config;
}
```





### 视频分支

```less
VideoChannel::SetRemoteContent_w
WebRtcVideoChannel::SetSendParameters
WebRtcVideoChannel::ApplyChangedParams
WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec
WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream
internal::Call::CreateVideoSendStream
VideoSendStream::VideoSendStream
VideoSendStreamImpl::VideoSendStreamImpl
RtpTransportControllerSend::CreateRtpVideoSender
RtpVideoSender::RtpVideoSender
ModuleRtpRtcpImpl2::RegisterRtpHeaderExtension
RTPSender::RegisterRtpHeaderExtension
RtpHeaderExtensionMap::Register
```



#### VideoChannel::SetRemoteContent_w

pc/channel.cc

```cpp
bool VideoChannel::SetRemoteContent_w(const MediaContentDescription* content,
                                      SdpType type,
                                      std::string* error_desc) {
    

 
  const VideoContentDescription* video = content->as_video();

  // 从 MediaContentDescription* content 获取到 rtp header extension
  // 这里的数据是在addTrack的时候，默认配置了extension
  if (type == SdpType::kAnswer)
    SetNegotiatedHeaderExtensions_w(video->rtp_header_extensions());

  RtpHeaderExtensions rtp_header_extensions =
      GetFilteredRtpHeaderExtensions(video->rtp_header_extensions());

  // 把 rtp_header_extensions 存放到  send_params
  VideoSendParameters send_params = last_send_params_;
  RtpSendParametersFromMediaDescription(
      video, rtp_header_extensions,
      webrtc::RtpTransceiverDirectionHasRecv(video->direction()), &send_params);

  ...

  // 传递到WebRtcVideoChannel
  if (!media_channel()->SetSendParameters(send_params)) {
    SafeSetError(
        "Failed to set remote video description send parameters for m-section "
        "with mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }
  
  ...
  
  return true;
}
```



##### BaseChannel::GetFilteredRtpHeaderExtensions

pc/channel.cc

```
RtpHeaderExtensions BaseChannel::GetFilteredRtpHeaderExtensions(
    const RtpHeaderExtensions& extensions) {
  if (crypto_options_.srtp.enable_encrypted_rtp_header_extensions) {
    RtpHeaderExtensions filtered;
    absl::c_copy_if(extensions, std::back_inserter(filtered),
                    [](const webrtc::RtpExtension& extension) {
                      return !extension.encrypt;
                    });
    return filtered;
  }

  return webrtc::RtpExtension::FilterDuplicateNonEncrypted(extensions);
}
```





#### WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream() {
...
  webrtc::VideoSendStream::Config config = parameters_.config.Copy();
  if (!config.rtp.rtx.ssrcs.empty() && config.rtp.rtx.payload_type == -1) {
    RTC_LOG(LS_WARNING) << "RTX SSRCs configured but there's no configured RTX "
                           "payload type the set codec. Ignoring RTX.";
    config.rtp.rtx.ssrcs.clear();
  }
  if (parameters_.encoder_config.number_of_streams == 1) {
    // SVC is used instead of simulcast. Remove unnecessary SSRCs.
    if (config.rtp.ssrcs.size() > 1) {
      config.rtp.ssrcs.resize(1);
      if (config.rtp.rtx.ssrcs.size() > 1) {
        config.rtp.rtx.ssrcs.resize(1);
      }
    }
  }
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());

...
}
```



#### RtpVideoSender::RtpVideoSender

call/rtp_video_sender.cc

```cpp
RtpVideoSender::RtpVideoSender(
    Clock* clock,
    std::map<uint32_t, RtpState> suspended_ssrcs,
    const std::map<uint32_t, RtpPayloadState>& states,
    const RtpConfig& rtp_config,
    int rtcp_report_interval_ms,
    Transport* send_transport,
    const RtpSenderObservers& observers,
    RtpTransportControllerSendInterface* transport,
    RtcEventLog* event_log,
    RateLimiter* retransmission_limiter,
    std::unique_ptr<FecController> fec_controller,
    FrameEncryptorInterface* frame_encryptor,
    const CryptoOptions& crypto_options,
    rtc::scoped_refptr<FrameTransformerInterface> frame_transformer)
	...
  for (size_t i = 0; i < rtp_config_.extensions.size(); ++i) {
    const std::string& extension = rtp_config_.extensions[i].uri;
    int id = rtp_config_.extensions[i].id;
    RTC_DCHECK(RtpExtension::IsSupportedForVideo(extension));
    for (const RtpStreamSender& stream : rtp_streams_) {
      stream.rtp_rtcp->RegisterRtpHeaderExtension(extension, id);
    }
  }
  ...
}
```



### ModuleRtpRtcpImpl2::RegisterRtpHeaderExtension

modules/rtp_rtcp/source/rtp_rtcp_impl2.cc

```cpp
void ModuleRtpRtcpImpl2::RegisterRtpHeaderExtension(absl::string_view uri,
                                                    int id) {
  //	std::unique_ptr<RtpSenderContext> rtp_sender_;
  //  RTPSender packet_generator;
  bool registered =
      rtp_sender_->packet_generator.RegisterRtpHeaderExtension(uri, id);
  RTC_CHECK(registered);
}
```





### RTPSender::RegisterRtpHeaderExtension

modules/rtp_rtcp/source/rtp_sender.cc

```cpp
bool RTPSender::RegisterRtpHeaderExtension(absl::string_view uri, int id) {
  MutexLock lock(&send_mutex_);
  bool registered = rtp_header_extension_map_.RegisterByUri(id, uri);
  supports_bwe_extension_ = HasBweExtension(rtp_header_extension_map_);
  UpdateHeaderSizes();
  return registered;
}
```



### RtpHeaderExtensionMap::RegisterByUri

modules/rtp_rtcp/source/rtp_header_extension_map.cc

```cpp
bool RtpHeaderExtensionMap::RegisterByUri(int id, absl::string_view uri) {
  for (const ExtensionInfo& extension : kExtensions)
    if (uri == extension.uri)
      return Register(id, extension.type, extension.uri);
  RTC_LOG(LS_WARNING) << "Unknown extension uri:'" << uri << "', id: " << id
                      << '.';
  return false;
}
```



### RtpHeaderExtensionMap::Register

modules/rtp_rtcp/source/rtp_header_extension_map.cc

```cpp
bool RtpHeaderExtensionMap::Register(int id,
                                     RTPExtensionType type,
                                     const char* uri) {
  RTC_DCHECK_GT(type, kRtpExtensionNone);
  RTC_DCHECK_LT(type, kRtpExtensionNumberOfExtensions);

  if (id < RtpExtension::kMinId || id > RtpExtension::kMaxId) {
    RTC_LOG(LS_WARNING) << "Failed to register extension uri:'" << uri
                        << "' with invalid id:" << id << ".";
    return false;
  }

  RTPExtensionType registered_type = GetType(id);
  if (registered_type == type) {  // Same type/id pair already registered.
    RTC_LOG(LS_VERBOSE) << "Reregistering extension uri:'" << uri
                        << "', id:" << id;
    return true;
  }

  if (registered_type !=
      kInvalidType) {  // |id| used by another extension type.
    RTC_LOG(LS_WARNING) << "Failed to register extension uri:'" << uri
                        << "', id:" << id
                        << ". Id already in use by extension type "
                        << static_cast<int>(registered_type);
    return false;
  }
  RTC_DCHECK(!IsRegistered(type));

  // There is a run-time check above id fits into uint8_t.
  ids_[type] = static_cast<uint8_t>(id);
  return true;
}
```





## 参考

[webrtc代码走读二十：extension扩展头协商及初始化流程](https://blog.csdn.net/CrystalShaw/article/details/120182012)
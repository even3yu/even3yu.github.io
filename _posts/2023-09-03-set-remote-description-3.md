---
layout: post
title: webrtc setRemoteDescription-3
date: 2023-09-03 23:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---


![setRemoteSDP]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/setRemoteSDP-6649445.svg)



![videoencodesdp]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/videoencodesdp.svg)

![videotrack]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/videotrack.svg)

![setRemoteSDP]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/setRemoteSDP-6663142.svg)



## 9. SdpOfferAnswerHandler.UpdateSessionState

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::UpdateSessionState(
    SdpType type,
    cricket::ContentSource source,
    const cricket::SessionDescription* description) {
  RTC_DCHECK_RUN_ON(signaling_thread());

  // If there's already a pending error then no state transition should happen.
  // But all call-sites should be verifying this before calling us!
  RTC_DCHECK(session_error() == SessionError::kNone);

  // Update the signaling state according to the specified state machine (see
  // https://w3c.github.io/webrtc-pc/#rtcsignalingstate-enum).
  if (type == SdpType::kOffer) {
    ...
  } else if (type == SdpType::kPrAnswer) {
    ...
  } else {
    RTC_DCHECK(type == SdpType::kAnswer);
    ChangeSignalingState(PeerConnectionInterface::kStable);
    transceivers()->DiscardStableStates();
    have_pending_rtp_data_channel_ = false;
  }

  // Update internal objects according to the session description's media
  // descriptions.
  RTCError error = PushdownMediaDescription(type, source);
  if (!error.ok()) {
    return error;
  }

  return RTCError::OK();
}
```



### 9.1 SdpOfferAnswerHandler.PushdownMediaDescription

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::PushdownMediaDescription(
    SdpType type,
    cricket::ContentSource source) { // source = cricket::CS_REMOTE
  const SessionDescriptionInterface* sdesc =
      (source == cricket::CS_LOCAL ? local_description()
                                   : remote_description());
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(sdesc);

  // Gather lists of updates to be made on cricket channels on the signaling
  // thread, before performing them all at once on the worker thread. Necessary
  // due to threading restrictions.
   auto payload_type_demuxing_updates = GetPayloadTypeDemuxingUpdates(source);
  std::vector<ContentUpdate> content_updates;

  // Collect updates for each audio/video transceiver.
  for (const auto& transceiver : transceivers()->List()) {
    //  SessionDescriptionInterface* sdesc，
		// 通过transceiver 从SessionDescription找到ContentInfo
    const ContentInfo* content_info =
        FindMediaSectionForTransceiver(transceiver, sdesc);
    // channel 就是 VideoChannel
    cricket::ChannelInterface* channel = transceiver->internal()->channel();
    if (!channel || !content_info || content_info->rejected) {
      continue;
    }
    // ContentInfo 得到 ContentInfo::MediaContentDescription 
    const MediaContentDescription* content_desc =
        content_info->media_description();
    if (content_desc) {
      // 类似push_back，减少copy, 插入struct ContentUpdate对象
      content_updates.emplace_back(channel, content_desc);
    }
  }

	...

  RTCError error = pc_->worker_thread()->Invoke<RTCError>(
      RTC_FROM_HERE,
      rtc::Bind(&SdpOfferAnswerHandler::ApplyChannelUpdates, this, type, source,
                std::move(payload_type_demuxing_updates),
                std::move(content_updates)));
  if (!error.ok()) {
    return error;
  }
  
  ...

  return RTCError::OK();
}
```

1. SdpType type= kAnswer
2. cricket::ContentSource source = CS_REMOTE
3. 通过transceivers()->List() 获取到 `ContentUpdate content_updates`；
4. 





#### SdpOfferAnswerHandler::GetPayloadTypeDemuxingUpdates

pc/sdp_offer_answer.cc

```cpp
std::vector<SdpOfferAnswerHandler::PayloadTypeDemuxingUpdate>
SdpOfferAnswerHandler::GetPayloadTypeDemuxingUpdates(
    cricket::ContentSource source) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // We may need to delete any created default streams and disable creation of
  // new ones on the basis of payload type. This is needed to avoid SSRC
  // collisions in Call's RtpDemuxer, in the case that a transceiver has
  // created a default stream, and then some other channel gets the SSRC
  // signaled in the corresponding Unified Plan "m=" section. Specifically, we
  // need to disable payload type based demuxing when two bundled "m=" sections
  // are using the same payload type(s). For more context
  // see https://bugs.chromium.org/p/webrtc/issues/detail?id=11477
  const SessionDescriptionInterface* sdesc =
      (source == cricket::CS_LOCAL ? local_description()
                                   : remote_description());
  const cricket::ContentGroup* bundle_group =
      sdesc->description()->GetGroupByName(cricket::GROUP_TYPE_BUNDLE);
  
  // 存放了codec id
  std::set<int> audio_payload_types;
  std::set<int> video_payload_types;
  bool pt_demuxing_enabled_audio = true;
  bool pt_demuxing_enabled_video = true;
  // ContentInfo content_info
  for (auto& content_info : sdesc->description()->contents()) {
   ...
    
    // MediaContentInfo media_description()
    switch (content_info.media_description()->type()) {
      case cricket::MediaType::MEDIA_TYPE_AUDIO: {
        const cricket::AudioContentDescription* audio_desc =
            content_info.media_description()->as_audio();
        // codec id是否重复，如果重复 pt_demuxing_enabled_audio = false
        for (const cricket::AudioCodec& audio : audio_desc->codecs()) {
          if (audio_payload_types.count(audio.id)) {
            // Two m= sections are using the same payload type, thus demuxing
            // by payload type is not possible.
            pt_demuxing_enabled_audio = false;
          }
          audio_payload_types.insert(audio.id);
        }
        break;
      }
      case cricket::MediaType::MEDIA_TYPE_VIDEO: {
        const cricket::VideoContentDescription* video_desc =
            content_info.media_description()->as_video();
        for (const cricket::VideoCodec& video : video_desc->codecs()) {
          if (video_payload_types.count(video.id)) {
            // Two m= sections are using the same payload type, thus demuxing
            // by payload type is not possible.
            pt_demuxing_enabled_video = false;
          }
          video_payload_types.insert(video.id);
        }
        break;
      }
      default:
        // Ignore data channels.
        continue;
    }
  }

  // Gather all updates ahead of time so that all channels can be updated in a
  // single Invoke; necessary due to thread guards.
  std::vector<PayloadTypeDemuxingUpdate> channel_updates;
  for (const auto& transceiver : transceivers()->List()) {
    cricket::ChannelInterface* channel = transceiver->internal()->channel();
    const ContentInfo* content =
        FindMediaSectionForTransceiver(transceiver, sdesc);
    if (!channel || !content) {
      continue;
    }
    RtpTransceiverDirection local_direction =
        content->media_description()->direction();
    if (source == cricket::CS_REMOTE) {
      local_direction = RtpTransceiverDirectionReversed(local_direction);
    }
    cricket::MediaType media_type = channel->media_type();
    bool in_bundle_group =
        (bundle_group && bundle_group->HasContentName(channel->content_name()));
    bool payload_type_demuxing_enabled = false;
    if (media_type == cricket::MediaType::MEDIA_TYPE_AUDIO) {
      payload_type_demuxing_enabled =
          (!in_bundle_group || pt_demuxing_enabled_audio) &&
          RtpTransceiverDirectionHasRecv(local_direction);
    } else if (media_type == cricket::MediaType::MEDIA_TYPE_VIDEO) {
      payload_type_demuxing_enabled =
          (!in_bundle_group || pt_demuxing_enabled_video) &&
          RtpTransceiverDirectionHasRecv(local_direction);
    }
    channel_updates.emplace_back(channel, payload_type_demuxing_enabled);
  }
  return channel_updates;
}
```

- 从codec查找是否有重复的codec.id， 如果重复 `pt_demuxing_enabled_video = false，pt_demuxing_enabled_audio = false` ，相同 payload type, 不能复用；
- PayloadTypeDemuxingUpdate



#### PayloadTypeDemuxingUpdate

pc/sdp_offer_answer.h

```cpp
struct PayloadTypeDemuxingUpdate {
    cricket::ChannelInterface* channel;
    bool enabled;
  };
```



#### MediaContentDescription

#### AudioContentDescription

#### AudioCodec

media/base/codec.h



#### ContentUpdate

pc/sdp_offer_answer.h

```cpp
  struct ContentUpdate {
    ...
    cricket::ChannelInterface* channel;
    const cricket::MediaContentDescription* content_description;
  };
```



### 9.2 --SdpOfferAnswerHandler.ApplyChannelUpdates

pc/sdp_offer_answer.cc

【章节10】。



## 10. SdpOfferAnswerHandler.ApplyChannelUpdates

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::ApplyChannelUpdates(
    SdpType type,
    cricket::ContentSource source, // cricket::CS_REMOTE
    std::vector<PayloadTypeDemuxingUpdate> payload_type_demuxing_updates,
    std::vector<ContentUpdate> content_updates) {
  RTC_DCHECK_RUN_ON(pc_->worker_thread());
  // If this is answer-ish we're ready to let media flow.
  bool enable_sending = type == SdpType::kPrAnswer || type == SdpType::kAnswer;

  // modified_channels 是集合，所以这里两个循环都会添加channel
  std::set<cricket::ChannelInterface*> modified_channels;
  for (const auto& update : payload_type_demuxing_updates) {
    modified_channels.insert(update.channel);
    update.channel->SetPayloadTypeDemuxingEnabled(update.enabled);
  }
  // 对所有的channel 进行处理
  for (const auto& update : content_updates) {
    modified_channels.insert(update.channel);
    std::string error;
    bool success = (source == cricket::CS_LOCAL)
                       ? update.channel->SetLocalContent(
                             update.content_description, type, &error)
                        // channel 就是VideoChannel
                        // update.content_description 就是MediaContentDescription
                       : update.channel->SetRemoteContent(
                             update.content_description, type, &error);
    if (!success) {
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER, error);
    }
    if (enable_sending && !update.channel->enabled()) {
      update.channel->Enable(true);
    }
  }
  // The above calls may have modified properties of the channel (header
  // extension mappings, demuxer criteria) which still need to be applied to the
  // RtpTransport.
  return pc_->network_thread()->Invoke<RTCError>(
      RTC_FROM_HERE, [modified_channels] {
        for (auto channel : modified_channels) {
          std::string error;
          if (!channel->UpdateRtpTransport(&error)) {
            LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER, error);
          }
        }
        return RTCError::OK();
      });
}
```

- 通过transceivers()->List() 获取到content_updates； 这是在上一步处理的，对content_updates 循环，处理数据。

- 参数说明

  | 参数                                                         |                                                              |
  | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | SdpType type                                                 |                                                              |
  | cricket::ContentSource source                                | cricket::CS_LOCAL                                            |
  | std::vector<PayloadTypeDemuxingUpdate> payload_type_demuxing_updates | 存放cricket::ChannelInterface* channel 和    bool enabled; enabled 是能不能复用； |
  | std::vector<ContentUpdate> content_updates                   | 存放的是BaseChannel，MediaContentDescription                 |

- 



### 10.1 BaseChannel.SetRemoteContent

pc/channel.cc

```c++
bool BaseChannel::SetRemoteContent(const MediaContentDescription* content,
                                   SdpType type,
                                   std::string* error_desc) {
  TRACE_EVENT0("webrtc", "BaseChannel::SetRemoteContent");
  return InvokeOnWorker<bool>(
      RTC_FROM_HERE,
      Bind(&BaseChannel::SetRemoteContent_w, this, content, type, error_desc));
}
```



### 10.2 VideoChannel.SetRemoteContent_w

pc/channel.cc



## 11. VideoChannel.SetRemoteContent_w

pc/channel.cc

```c++
bool VideoChannel::SetRemoteContent_w(const MediaContentDescription* content,
                                      SdpType type,
                                      std::string* error_desc) {
	...
  const VideoContentDescription* video = content->as_video();

  if (type == SdpType::kAnswer)
    SetNegotiatedHeaderExtensions_w(video->rtp_header_extensions());

  RtpHeaderExtensions rtp_header_extensions =
      GetFilteredRtpHeaderExtensions(video->rtp_header_extensions());

  
  // 1. 得到  VideoSendParameters send_params
  VideoSendParameters send_params = last_send_params_;
  RtpSendParametersFromMediaDescription(
      video, rtp_header_extensions,
      webrtc::RtpTransceiverDirectionHasRecv(video->direction()), &send_params);
  if (video->conference_mode()) {
    send_params.conference_mode = true;
  }
  send_params.mid = content_name();

  // 2. setLocalContent_w 保存的last_recv_params_
  VideoRecvParameters recv_params = last_recv_params_;

  bool needs_recv_params_update = false;
  // 3. codec 匹配，找到最佳codec
  if (type == SdpType::kAnswer || type == SdpType::kPrAnswer) {
    for (auto& recv_codec : recv_params.codecs) {
      auto* send_codec = FindMatchingCodec(send_params.codecs, recv_codec);
      if (send_codec) {
        if (!send_codec->packetization && recv_codec.packetization) {
          recv_codec.packetization.reset();
          needs_recv_params_update = true;
        } else if (send_codec->packetization != recv_codec.packetization) {
          SafeSetError(
              "Failed to set remote answer due to invalid codec packetization "
              "specifid in m-section with mid='" +
                  content_name() + "'.",
              error_desc);
          return false;
        }
      }
    }
  }

  // 4. 
  if (!media_channel()->SetSendParameters(send_params)) {
    SafeSetError(
        "Failed to set remote video description send parameters for m-section "
        "with mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }
  
  last_send_params_ = send_params;

  if (needs_recv_params_update) {
    if (!media_channel()->SetRecvParameters(recv_params)) {
      SafeSetError("Failed to set recv parameters for m-section with mid='" +
                       content_name() + "'.",
                   error_desc);
      return false;
    }
    last_recv_params_ = recv_params;
  }

  if (!webrtc::RtpTransceiverDirectionHasSend(content->direction())) {
    RTC_DLOG(LS_VERBOSE) << "SetRemoteContent_w: remote side will not send - "
                            "disable payload type demuxing for "
                         << ToString();
    ClearHandledPayloadTypes();
  }

  // TODO(pthatcher): Move remote streams into VideoRecvParameters,
  // and only give it to the media channel once we have a local
  // description too (without a local description, we won't be able to
  // recv them anyway).
  if (!UpdateRemoteStreams_w(video->streams(), type, error_desc)) {
    SafeSetError(
        "Failed to set remote video description streams for m-section with "
        "mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }
  set_remote_content_direction(content->direction());
  UpdateMediaSendRecvState_w();
  return true;
}
```



### 11.1 RtpParametersFromMediaDescription

pc/channel.cc

```cpp
template <class Codec>
void RtpParametersFromMediaDescription(
    const MediaContentDescriptionImpl<Codec>* desc,
    const RtpHeaderExtensions& extensions,
    bool is_stream_active,
    RtpParameters<Codec>* params) {
  params->is_stream_active = is_stream_active;
  params->codecs = desc->codecs();
  // TODO(bugs.webrtc.org/11513): See if we really need
  // rtp_header_extensions_set() and remove it if we don't.
  if (desc->rtp_header_extensions_set()) {
    params->extensions = extensions;
  }
  params->rtcp.reduced_size = desc->rtcp_reduced_size();
  params->rtcp.remote_estimate = desc->remote_estimate();
}
```

![RtpParameters-video](set-local-description.assets/RtpParameters-video.jpg)



###  WebRtcVideoChannel.SetSendParameters

media/engine/webrtc_video_engine.cc



## 12. WebRtcVideoChannel.SetSendParameters

media/engine/webrtc_video_engine.cc

```c++
bool WebRtcVideoChannel::SetSendParameters(const VideoSendParameters& params) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  TRACE_EVENT0("webrtc", "WebRtcVideoChannel::SetSendParameters");
  RTC_LOG(LS_INFO) << "SetSendParameters: " << params.ToString();
  ChangedSendParameters changed_params;
  if (!GetChangedSendParameters(params, &changed_params)) {
    return false;
  }

  if (changed_params.negotiated_codecs) {
    for (const auto& send_codec : *changed_params.negotiated_codecs)
      RTC_LOG(LS_INFO) << "Negotiated codec: " << send_codec.codec.ToString();
  }

  send_params_ = params;
  return ApplyChangedParams(changed_params);
}
```



### WebRtcVideoChannel.ApplyChangedParams

media/engine/webrtc_video_engine.cc

```c++
bool WebRtcVideoChannel::ApplyChangedParams(
    const ChangedSendParameters& changed_params) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  if (changed_params.negotiated_codecs)
    negotiated_codecs_ = *changed_params.negotiated_codecs;

  if (changed_params.send_codec)
    send_codec_ = changed_params.send_codec;

  if (changed_params.extmap_allow_mixed) {
    SetExtmapAllowMixed(*changed_params.extmap_allow_mixed);
  }
  if (changed_params.rtp_header_extensions) {
    send_rtp_extensions_ = changed_params.rtp_header_extensions;
  }

  if (changed_params.send_codec || changed_params.max_bandwidth_bps) {
    if (send_params_.max_bandwidth_bps == -1) {
      // Unset the global max bitrate (max_bitrate_bps) if max_bandwidth_bps is
      // -1, which corresponds to no "b=AS" attribute in SDP. Note that the
      // global max bitrate may be set below in GetBitrateConfigForCodec, from
      // the codec max bitrate.
      // TODO(pbos): This should be reconsidered (codec max bitrate should
      // probably not affect global call max bitrate).
      bitrate_config_.max_bitrate_bps = -1;
    }

    if (send_codec_) {
      // TODO(holmer): Changing the codec parameters shouldn't necessarily mean
      // that we change the min/max of bandwidth estimation. Reevaluate this.
      bitrate_config_ = GetBitrateConfigForCodec(send_codec_->codec);
      if (!changed_params.send_codec) {
        // If the codec isn't changing, set the start bitrate to -1 which means
        // "unchanged" so that BWE isn't affected.
        bitrate_config_.start_bitrate_bps = -1;
      }
    }

    if (send_params_.max_bandwidth_bps >= 0) {
      // Note that max_bandwidth_bps intentionally takes priority over the
      // bitrate config for the codec. This allows FEC to be applied above the
      // codec target bitrate.
      // TODO(pbos): Figure out whether b=AS means max bitrate for this
      // WebRtcVideoChannel (in which case we're good), or per sender (SSRC),
      // in which case this should not set a BitrateConstraints but rather
      // reconfigure all senders.
      bitrate_config_.max_bitrate_bps = send_params_.max_bandwidth_bps == 0
                                            ? -1
                                            : send_params_.max_bandwidth_bps;
    }

    call_->GetTransportControllerSend()->SetSdpBitrateParameters(
        bitrate_config_);
  }

  for (auto& kv : send_streams_) {
    kv.second->SetSendParameters(changed_params);
  }
  if (changed_params.send_codec || changed_params.rtcp_mode) {
    // Update receive feedback parameters from new codec or RTCP mode.
    RTC_LOG(LS_INFO)
        << "SetFeedbackOptions on all the receive streams because the send "
           "codec or RTCP mode has changed.";
    for (auto& kv : receive_streams_) {
      RTC_DCHECK(kv.second != nullptr);
      kv.second->SetFeedbackParameters(
          HasLntf(send_codec_->codec), HasNack(send_codec_->codec),
          HasTransportCc(send_codec_->codec),
          send_params_.rtcp.reduced_size ? webrtc::RtcpMode::kReducedSize
                                         : webrtc::RtcpMode::kCompound);
    }
  }
  return true;
}
```



## 13. !!! WebRtcVideoChannel.WebRtcVideoSendStream.SetSendParameters

media/engine/webrtc_video_engine.cc

```c++
void WebRtcVideoChannel::WebRtcVideoSendStream::SetSendParameters(
    const ChangedSendParameters& params) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  // |recreate_stream| means construction-time parameters have changed and the
  // sending stream needs to be reset with the new config.
  bool recreate_stream = false;
  if (params.rtcp_mode) {
    parameters_.config.rtp.rtcp_mode = *params.rtcp_mode;
    rtp_parameters_.rtcp.reduced_size =
        parameters_.config.rtp.rtcp_mode == webrtc::RtcpMode::kReducedSize;
    recreate_stream = true;
  }
  if (params.extmap_allow_mixed) {
    parameters_.config.rtp.extmap_allow_mixed = *params.extmap_allow_mixed;
    recreate_stream = true;
  }
  if (params.rtp_header_extensions) {
    parameters_.config.rtp.extensions = *params.rtp_header_extensions;
    rtp_parameters_.header_extensions = *params.rtp_header_extensions;
    recreate_stream = true;
  }
  if (params.mid) {
    parameters_.config.rtp.mid = *params.mid;
    recreate_stream = true;
  }
  if (params.max_bandwidth_bps) {
    parameters_.max_bitrate_bps = *params.max_bandwidth_bps;
    ReconfigureEncoder();
  }
  if (params.conference_mode) {
    parameters_.conference_mode = *params.conference_mode;
  }

  // Set codecs and options.
  if (params.send_codec) {
    // 设置codec
    SetCodec(*params.send_codec);
    recreate_stream = false;  // SetCodec has already recreated the stream.
  } else if (params.conference_mode && parameters_.codec_settings) {
    SetCodec(*parameters_.codec_settings);
    recreate_stream = false;  // SetCodec has already recreated the stream.
  }
  //  这里没有走
  if (recreate_stream) {
    RTC_LOG(LS_INFO)
        << "RecreateWebRtcStream (send) because of SetSendParameters";
    RecreateWebRtcStream();
  }
}
```



## 14. !!! WebRtcVideoChannel.WebRtcVideoSendStream.SetCodec

media/engine/webrtc_video_engine.cc

```c++
void WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec(
    const VideoCodecSettings& codec_settings) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  parameters_.encoder_config = CreateVideoEncoderConfig(codec_settings.codec);
  RTC_DCHECK_GT(parameters_.encoder_config.number_of_streams, 0);

  parameters_.config.rtp.payload_name = codec_settings.codec.name;
  parameters_.config.rtp.payload_type = codec_settings.codec.id;
  parameters_.config.rtp.raw_payload =
      codec_settings.codec.packetization == kPacketizationParamRaw;
  parameters_.config.rtp.ulpfec = codec_settings.ulpfec;
  parameters_.config.rtp.flexfec.payload_type =
      codec_settings.flexfec_payload_type;

  // Set RTX payload type if RTX is enabled.
  if (!parameters_.config.rtp.rtx.ssrcs.empty()) {
    if (codec_settings.rtx_payload_type == -1) {
      RTC_LOG(LS_WARNING)
          << "RTX SSRCs configured but there's no configured RTX "
             "payload type. Ignoring.";
      parameters_.config.rtp.rtx.ssrcs.clear();
    } else {
      parameters_.config.rtp.rtx.payload_type = codec_settings.rtx_payload_type;
    }
  }

  const bool has_lntf = HasLntf(codec_settings.codec);
  parameters_.config.rtp.lntf.enabled = has_lntf;
  parameters_.config.encoder_settings.capabilities.loss_notification = has_lntf;

  parameters_.config.rtp.nack.rtp_history_ms =
      HasNack(codec_settings.codec) ? kNackHistoryMs : 0;

  parameters_.codec_settings = codec_settings;

  // TODO(nisse): Avoid recreation, it should be enough to call
  // ReconfigureEncoder.
  RTC_LOG(LS_INFO) << "RecreateWebRtcStream (send) because of SetCodec.";
  RecreateWebRtcStream();
}
```



## 15. !!! WebRtcVideoChannel.WebRtcVideoSendStream.RecreateWebRtcStream

media/engine/webrtc_video_engine.cc

```c++
void WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream() {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  if (stream_ != NULL) {
    call_->DestroyVideoSendStream(stream_);
  }

  RTC_CHECK(parameters_.codec_settings);
  RTC_DCHECK_EQ((parameters_.encoder_config.content_type ==
                 webrtc::VideoEncoderConfig::ContentType::kScreen),
                parameters_.options.is_screencast.value_or(false))
      << "encoder content type inconsistent with screencast option";
  parameters_.encoder_config.encoder_specific_settings =
      ConfigureVideoEncoderSettings(parameters_.codec_settings->codec);

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
  // create VideoSendStream
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());

  parameters_.encoder_config.encoder_specific_settings = NULL;

  if (source_) {
    stream_->SetSource(source_, GetDegradationPreference());
  }

  // Call stream_->Start() if necessary conditions are met.
  UpdateSendState();
}
```




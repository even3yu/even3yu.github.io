---
layout: post
title: webrtc setLocalDescription-3
date: 2023-08-27 23:40:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

![caller-set-local-description]({{ site.url }}{{ site.baseurl }}/images/set-local-description.assets/caller-set-local-description.jpg)


![jsep-session-description]({{ site.url }}{{ site.baseurl }}/images/set-local-description.assets/jsep-session-description.png)


![img]({{ site.url }}{{ site.baseurl }}/images/set-local-description.assets/sessiondescription.png)

## 11. SdpOfferAnswerHandler::UpdateSessionState

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::UpdateSessionState(
    SdpType type,// kOffer
    cricket::ContentSource source,//cricket::CS_LOCAL
    const cricket::SessionDescription* description) {
  ...

  // 通知PeerConnection的PeerConnectionObserver  的 SignalingState 变化
  if (type == SdpType::kOffer) {
    // 走的是这里
    ChangeSignalingState(source == cricket::CS_LOCAL
                             ? PeerConnectionInterface::kHaveLocalOffer
                             : PeerConnectionInterface::kHaveRemoteOffer);
  } else if (type == SdpType::kPrAnswer) {
    ...
  } else {
    ...
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

创建WebRtcVideoSendStream。



### 11.1 SdpOfferAnswerHandler.PushdownMediaDescription

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::PushdownMediaDescription(
    SdpType type,
    cricket::ContentSource source) { // source = cricket::CS_LOCAL
  const SessionDescriptionInterface* sdesc =
      (source == cricket::CS_LOCAL ? local_description()
                                   : remote_description());
  ...

  // Gather lists of updates to be made on cricket channels on the signaling
  // thread, before performing them all at once on the worker thread. Necessary
  // due to threading restrictions.
  // ？？？
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

  // If using the RtpDataChannel, add it to the list of updates.
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

1. 通过transceivers()->List() 获取到 `ContentUpdate content_updates`；
2. 



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



### 11.2 --SdpOfferAnswerHandler.ApplyChannelUpdates

pc/sdp_offer_answer.cc

【章节12】。

## 12. SdpOfferAnswerHandler.ApplyChannelUpdates

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::ApplyChannelUpdates(
    SdpType type,
    cricket::ContentSource source, // cricket::CS_LOCAL
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
                                      // channel 就是VideoChannel
                                      // update.content_description 就是MediaContentDescription
                       ? update.channel->SetLocalContent(
                             update.content_description, type, &error)
                                      // 
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



### 12.1 BaseChannel.SetLocalContent

pc/channel.cc

```c++
bool BaseChannel::SetLocalContent(const MediaContentDescription* content,
                                  SdpType type,
                                  std::string* error_desc) {
  TRACE_EVENT0("webrtc", "BaseChannel::SetLocalContent");
  return InvokeOnWorker<bool>(
      RTC_FROM_HERE,
      Bind(&BaseChannel::SetLocalContent_w, this, content, type, error_desc));
}
```



### 12.2 VideoChannel::SetLocalContent_w

pc/channel.cc



## 13. VideoChannel::SetLocalContent_w

pc/channel.cc

```cpp
bool VideoChannel::SetLocalContent_w(const MediaContentDescription* content,
                                     SdpType type,
                                     std::string* error_desc) {
	...

  const VideoContentDescription* video = content->as_video();

  ...

  RtpHeaderExtensions rtp_header_extensions =
      GetFilteredRtpHeaderExtensions(video->rtp_header_extensions());
  SetReceiveExtensions(rtp_header_extensions);
  media_channel()->SetExtmapAllowMixed(video->extmap_allow_mixed());

  // 1. 得到  VideoRecvParameters recv_params
  VideoRecvParameters recv_params = last_recv_params_;
  RtpParametersFromMediaDescription(
      video, rtp_header_extensions,
      webrtc::RtpTransceiverDirectionHasRecv(video->direction()), &recv_params);

  VideoSendParameters send_params = last_send_params_;

  bool needs_send_params_update = false;
  if (type == SdpType::kAnswer || type == SdpType::kPrAnswer) {
    ...
  }

  // 2. 设置RecvParameters
  if (!media_channel()->SetRecvParameters(recv_params)) {
    SafeSetError(
        "Failed to set local video description recv parameters for m-section "
        "with mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }

  if (webrtc::RtpTransceiverDirectionHasRecv(video->direction())) {
    for (const VideoCodec& codec : video->codecs()) {
      MaybeAddHandledPayloadType(codec.id);
    }
  }

  last_recv_params_ = recv_params;

  if (needs_send_params_update) {
    if (!media_channel()->SetSendParameters(send_params)) {
      SafeSetError("Failed to set send parameters for m-section with mid='" +
                       content_name() + "'.",
                   error_desc);
      return false;
    }
    last_send_params_ = send_params;
  }

  // TODO(pthatcher): Move local streams into VideoSendParameters, and
  // only give it to the media channel once we have a remote
  // description too (without a remote description, we won't be able
  // to send them anyway).
  // 3. VideoContentDescription video;
	// StreamParamsVec streams(); 
  if (!UpdateLocalStreams_w(video->streams(), type, error_desc)) {
    SafeSetError(
        "Failed to set local video description streams for m-section with "
        "mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }

  set_local_content_direction(content->direction());
  UpdateMediaSendRecvState_w();
  return true;
}
```



### 13.1 RtpParametersFromMediaDescription

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



### 13.2 WebRtcVideoChannel::SetRecvParameters

media/engine/webrtc_video_engine.cc

```cpp
bool WebRtcVideoChannel::SetRecvParameters(const VideoRecvParameters& params) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  TRACE_EVENT0("webrtc", "WebRtcVideoChannel::SetRecvParameters");
  RTC_LOG(LS_INFO) << "SetRecvParameters: " << params.ToString();
  ChangedRecvParameters changed_params;
  if (!GetChangedRecvParameters(params, &changed_params)) {
    return false;
  }
  if (changed_params.flexfec_payload_type) {
    RTC_LOG(LS_INFO) << "Changing FlexFEC payload type (recv) from "
                     << recv_flexfec_payload_type_ << " to "
                     << *changed_params.flexfec_payload_type;
    recv_flexfec_payload_type_ = *changed_params.flexfec_payload_type;
  }
  if (changed_params.rtp_header_extensions) {
    recv_rtp_extensions_ = *changed_params.rtp_header_extensions;
  }
  if (changed_params.codec_settings) {
    RTC_LOG(LS_INFO) << "Changing recv codecs from "
                     << CodecSettingsVectorToString(recv_codecs_) << " to "
                     << CodecSettingsVectorToString(
                            *changed_params.codec_settings);
    recv_codecs_ = *changed_params.codec_settings;
  }

  for (auto& kv : receive_streams_) {
    kv.second->SetRecvParameters(changed_params);
  }
  recv_params_ = params;
  return true;
}
```



### 13.3 !!! BaseChannel.UpdateLocalStreams_w

pc/channel.cc



## 14. !!! BaseChannel.UpdateLocalStreams_w

pc/channel.cc

```c++
bool BaseChannel::UpdateLocalStreams_w(const std::vector<StreamParams>& streams,
                                       SdpType type,
                                       std::string* error_desc) {
  // In the case of RIDs (where SSRCs are not negotiated), this method will
  // generate an SSRC for each layer in StreamParams. That representation will
  // be stored internally in |local_streams_|.
  // In subsequent offers, the same stream can appear in |streams| again
  // (without the SSRCs), so it should be looked up using RIDs (if available)
  // and then by primary SSRC.
  // In both scenarios, it is safe to assume that the media channel will be
  // created with a StreamParams object with SSRCs. However, it is not safe to
  // assume that |local_streams_| will always have SSRCs as there are scenarios
  // in which niether SSRCs or RIDs are negotiated.

  // Check for streams that have been removed.
  bool ret = true;
  for (const StreamParams& old_stream : local_streams_) {
    if (!old_stream.has_ssrcs() ||
        GetStream(streams, StreamFinder(&old_stream))) {
      continue;
    }
    if (!media_channel()->RemoveSendStream(old_stream.first_ssrc())) {
      rtc::StringBuilder desc;
      desc << "Failed to remove send stream with ssrc "
           << old_stream.first_ssrc() << " from m-section with mid='"
           << content_name() << "'.";
      SafeSetError(desc.str(), error_desc);
      ret = false;
    }
  }
  // Check for new streams.
  std::vector<StreamParams> all_streams;
  for (const StreamParams& stream : streams) {
    StreamParams* existing = GetStream(local_streams_, StreamFinder(&stream));
    if (existing) {
      // Parameters cannot change for an existing stream.
      all_streams.push_back(*existing);
      continue;
    }

    all_streams.push_back(stream);
    StreamParams& new_stream = all_streams.back();

    if (!new_stream.has_ssrcs() && !new_stream.has_rids()) {
      continue;
    }

    RTC_DCHECK(new_stream.has_ssrcs() || new_stream.has_rids());
    if (new_stream.has_ssrcs() && new_stream.has_rids()) {
      rtc::StringBuilder desc;
      desc << "Failed to add send stream: " << new_stream.first_ssrc()
           << " into m-section with mid='" << content_name()
           << "'. Stream has both SSRCs and RIDs.";
      SafeSetError(desc.str(), error_desc);
      ret = false;
      continue;
    }

    // At this point we use the legacy simulcast group in StreamParams to
    // indicate that we want multiple layers to the media channel.
    if (!new_stream.has_ssrcs()) {
      // TODO(bugs.webrtc.org/10250): Indicate if flex is desired here.
      new_stream.GenerateSsrcs(new_stream.rids().size(), /* rtx = */ true,
                               /* flex_fec = */ false, ssrc_generator_);
    }

    // WebRtcVideoChanel/WebRtcVoiceChanel
    if (media_channel()->AddSendStream(new_stream)) {
      RTC_LOG(LS_INFO) << "Add send stream ssrc: " << new_stream.ssrcs[0]
                       << " into " << ToString();
    } else {
      rtc::StringBuilder desc;
      desc << "Failed to add send stream ssrc: " << new_stream.first_ssrc()
           << " into m-section with mid='" << content_name() << "'";
      SafeSetError(desc.str(), error_desc);
      ret = false;
    }
  }
  local_streams_ = all_streams;
  return ret;
}
```

- StreamParams （media/base/stream_params.h），主要的ssrc， 表示rtp流的id
- for循环，streams， 从local_streams_查找，如果粗在则忽略，不存在则MediaChannel::AddSendStream
-  更新local_streams_ = all_streams; 



### 14.1 !!! WebRtcVideoChannel.AddSendStream

media/engine/webrtc_video_engine.cc

```c++
bool WebRtcVideoChannel::AddSendStream(const StreamParams& sp) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  RTC_LOG(LS_INFO) << "AddSendStream: " << sp.ToString();
  if (!ValidateStreamParams(sp))
    return false;

  if (!ValidateSendSsrcAvailability(sp))
    return false;

  for (uint32_t used_ssrc : sp.ssrcs)
    send_ssrcs_.insert(used_ssrc);

  webrtc::VideoSendStream::Config config(this);

  for (const RidDescription& rid : sp.rids()) {
    config.rtp.rids.push_back(rid.rid);
  }

  config.suspend_below_min_bitrate = video_config_.suspend_below_min_bitrate;
  config.periodic_alr_bandwidth_probing =
      video_config_.periodic_alr_bandwidth_probing;
  config.encoder_settings.experiment_cpu_load_estimator =
      video_config_.experiment_cpu_load_estimator;
  config.encoder_settings.encoder_factory = encoder_factory_;
  config.encoder_settings.bitrate_allocator_factory =
      bitrate_allocator_factory_;
  config.encoder_settings.encoder_switch_request_callback = this;
  config.crypto_options = crypto_options_;
  config.rtp.extmap_allow_mixed = ExtmapAllowMixed();
  config.rtcp_report_interval_ms = video_config_.rtcp_report_interval_ms;

  // 创建WebRtcVideoSendStream
  WebRtcVideoSendStream* stream = new WebRtcVideoSendStream(
      call_, sp, std::move(config), default_send_options_,
      video_config_.enable_cpu_adaptation, bitrate_config_.max_bitrate_bps,
      send_codec_, send_rtp_extensions_, send_params_);

  uint32_t ssrc = sp.first_ssrc();
  RTC_DCHECK(ssrc != 0);
  // 保存stream，key 是ssrc
  send_streams_[ssrc] = stream;

  if (rtcp_receiver_report_ssrc_ == kDefaultRtcpReceiverReportSsrc) {
    rtcp_receiver_report_ssrc_ = ssrc;
    RTC_LOG(LS_INFO)
        << "SetLocalSsrc on all the receive streams because we added "
           "a send stream.";
    for (auto& kv : receive_streams_)
      kv.second->SetLocalSsrc(ssrc);
  }
  if (sending_) {
    stream->SetSend(true);
  }

  return true;
}
```

1. send_streams_

   ```c++
   // Using primary-ssrc (first ssrc) as key.
   std::map<uint32_t, WebRtcVideoSendStream*> send_streams_  RTC_GUARDED_BY(thread_checker_);
   ```

2. 创建并保存WebRtcVideoSendStream



### 14.2 WebRtcVideoSendStream.WebRtcVideoSendStream

media/engine/webrtc_video_engine.cc

```c++
WebRtcVideoChannel::WebRtcVideoSendStream::WebRtcVideoSendStream(
    webrtc::Call* call,
    const StreamParams& sp,
    webrtc::VideoSendStream::Config config,
    const VideoOptions& options,
    bool enable_cpu_overuse_detection,
    int max_bitrate_bps,
    const absl::optional<VideoCodecSettings>& codec_settings,
    const absl::optional<std::vector<webrtc::RtpExtension>>& rtp_extensions,
    // TODO(deadbeef): Don't duplicate information between send_params,
    // rtp_extensions, options, etc.
    const VideoSendParameters& send_params)
    : worker_thread_(rtc::Thread::Current()),
      ssrcs_(sp.ssrcs),
      ssrc_groups_(sp.ssrc_groups),
      call_(call),
      enable_cpu_overuse_detection_(enable_cpu_overuse_detection),
      source_(nullptr),
      stream_(nullptr),
      parameters_(std::move(config), options, max_bitrate_bps, codec_settings),
      rtp_parameters_(CreateRtpParametersWithEncodings(sp)),
      sending_(false),
      disable_automatic_resize_(
          IsEnabled(call->trials(), "WebRTC-Video-DisableAutomaticResize")) {
  // Maximum packet size may come in RtpConfig from external transport, for
  // example from QuicTransportInterface implementation, so do not exceed
  // given max_packet_size.
  parameters_.config.rtp.max_packet_size =
      std::min<size_t>(parameters_.config.rtp.max_packet_size, kVideoMtu);
  parameters_.conference_mode = send_params.conference_mode;

  sp.GetPrimarySsrcs(&parameters_.config.rtp.ssrcs);

  // ValidateStreamParams should prevent this from happening.
  RTC_CHECK(!parameters_.config.rtp.ssrcs.empty());
  rtp_parameters_.encodings[0].ssrc = parameters_.config.rtp.ssrcs[0];

  // RTX.
  sp.GetFidSsrcs(parameters_.config.rtp.ssrcs,
                 &parameters_.config.rtp.rtx.ssrcs);

  // FlexFEC SSRCs.
  // TODO(brandtr): This code needs to be generalized when we add support for
  // multistream protection.
  if (IsEnabled(call_->trials(), "WebRTC-FlexFEC-03")) {
    uint32_t flexfec_ssrc;
    bool flexfec_enabled = false;
    for (uint32_t primary_ssrc : parameters_.config.rtp.ssrcs) {
      if (sp.GetFecFrSsrc(primary_ssrc, &flexfec_ssrc)) {
        if (flexfec_enabled) {
          RTC_LOG(LS_INFO)
              << "Multiple FlexFEC streams in local SDP, but "
                 "our implementation only supports a single FlexFEC "
                 "stream. Will not enable FlexFEC for proposed "
                 "stream with SSRC: "
              << flexfec_ssrc << ".";
          continue;
        }

        flexfec_enabled = true;
        parameters_.config.rtp.flexfec.ssrc = flexfec_ssrc;
        parameters_.config.rtp.flexfec.protected_media_ssrcs = {primary_ssrc};
      }
    }
  }

  parameters_.config.rtp.c_name = sp.cname;
  if (rtp_extensions) {
    parameters_.config.rtp.extensions = *rtp_extensions;
    rtp_parameters_.header_extensions = *rtp_extensions;
  }
  parameters_.config.rtp.rtcp_mode = send_params.rtcp.reduced_size
                                         ? webrtc::RtcpMode::kReducedSize
                                         : webrtc::RtcpMode::kCompound;
  parameters_.config.rtp.mid = send_params.mid;
  rtp_parameters_.rtcp.reduced_size = send_params.rtcp.reduced_size;

  if (codec_settings) {
    // 创建的时候一般情况下codec_settings为空，
    SetCodec(*codec_settings);
  }
}
```


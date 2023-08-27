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

![]({{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/view.png)

流水线上对象的创建：**

1） PeerConnection::UpdateTransceiverChannel()方法中检查PC中的每个RtpTranceiver是存在MediaChannel，不存在的会调用WebRtcVideoEngine::CreateMediaChannel()创建WebRtcVideoChannel对象，并赋值给RtpTranceiver的RtpSender和RtpReceiver，这儿解决了VideoRtpSender的media_channel_成员为空的问题；

2） PeerConnection::UpdateSessionState()方法中，将SDP中的信息应用到上一步创建的视频媒体通道对象WebRtcVideoChannel上，调用WebRtcVideoChannel::AddSendStream()方法为通道创建WebRtcVideoSendStream，如果有多个视频Track，会有多个WebRtcVideoSendStream分别与之对应。WebRtcVideoSendStream对象存入WebRtcVideoChannel的std::map<uint32_t, WebRtcVideoSendStream*> send_streams_成员，以ssrc为key。创建WebRtcVideoSendStream，其构造函数中会进一步创建VideoSendStream，VideoSendStream的构造中会进一步创建

VideoStreamEncoder对象。 到此，所有有关的对象都已经创建完成。

**流水线的建立：**

之前就分析过VideoRtpSender->SetSsrc()方法非常重要，该方法在PeerConnection::ApplyLocalDescription()中最后被调用。会触发Track被传递，从VideoRtpSender传递到WebRtcVideoChannel，再传递到WebRtcVideoSendStream，成为WebRtcVideoSendStream的成员source_。 从而实现了逻辑上VideoRtpSender->WebRtcVideoChannel->WebRtcVideoSendStream流水线的建立；

WebRtcVideoSendStream::SetVideoSend()方法紧接着又触发调用VideoSendStream的SetSource()方法，以WebRtcVideoSendStream为视频源参数（看之前的类图，WebRtcVideoSendStream实现了VideoSourceInterface接口）一路传递给VideoStreamEncoder的成员VideoSourceProxy。在这个VideoSourceProxy::SetSource方法中，反向调用WebRtcVideoSendStream::AddOrUpdateSink()方法将VideoStreamEncoder作为VideoSink（看之前的类图，VideoStreamEncoder实现了VideoSinkInterface接口）添加到了WebRtcVideoSendStream。注意，在WebRtcVideoSendStream::AddOrUpdateSink()中会调用source_->AddOrUpdateSink()进一步将VideoStreamEncoder添加到了VideoTrack（如之前的描述VideoTrack已经被传递到WebRtcVideoSendStream成为WebRtcVideoSendStream的成员source_）。在逻辑上实现了视频流从WebRtcVideoSendStream->VideoSendStream->VideoStreamEncoder这段流水线。

至此，以发送端角度来看，从采集到编码器的整个流水线都已建立完毕。

- PeerConnection::SetLocalDescription
  ：把 track 交给 channel，但 channel 把 track 当 source 用；
- `PeerConnection::setLocalDescription` ->
  - `PeerConnection::ApplyLocalDescription` 的 `transceiver->internal()->sender_internal()->SetSsrc` ->
  - `RtpSenderBase::SetSsrc` ->
  - `VideoRtpSender::SetSend` ->
  - `WebRtcVideoChannel::SetVideoSend` ->
  - `WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend`，channel 把 source 存了起来，但此时 channel 的 stream（`WebRtcVideoSendStream`）还没有创建，需要等 `SetRemoteDescription` 才会创建；



## 11. SdpOfferAnswerHandler::UpdateSessionState

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::UpdateSessionState(
    SdpType type,// kOffer
    cricket::ContentSource source,//cricket::CS_LOCAL
    const cricket::SessionDescription* description) {
  RTC_DCHECK_RUN_ON(signaling_thread());

  // If there's already a pending error then no state transition should happen.
  // But all call-sites should be verifying this before calling us!
  RTC_DCHECK(session_error() == SessionError::kNone);

  // Update the signaling state according to the specified state machine (see
  // https://w3c.github.io/webrtc-pc/#rtcsignalingstate-enum).
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

1. SignalingState
   
   ```c++
    enum SignalingState {
       kStable,
       kHaveLocalOffer,
       kHaveLocalPrAnswer,
       kHaveRemoteOffer,
       kHaveRemotePrAnswer,
       kClosed,
     };
   ```
   
   ```c++
   void SdpOfferAnswerHandler::ChangeSignalingState(
       PeerConnectionInterface::SignalingState signaling_state) {
     RTC_DCHECK_RUN_ON(signaling_thread());
     if (signaling_state_ == signaling_state) {
       return;
     }
     RTC_LOG(LS_INFO) << "Session: " << pc_->session_id() << " Old state: "
                      << GetSignalingStateString(signaling_state_)
                      << " New state: "
                      << GetSignalingStateString(signaling_state);
     signaling_state_ = signaling_state;
     pc_->Observer()->OnSignalingChange(signaling_state_);
   }
   ```



### 11.1 SdpOfferAnswerHandler.PushdownMediaDescription

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::PushdownMediaDescription(
    SdpType type,
    cricket::ContentSource source) {
  const SessionDescriptionInterface* sdesc =
      (source == cricket::CS_LOCAL ? local_description()
                                   : remote_description());
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(sdesc);

  // Gather lists of updates to be made on cricket channels on the signaling
  // thread, before performing them all at once on the worker thread. Necessary
  // due to threading restrictions.
  // ？？？
  auto payload_type_demuxing_updates = GetPayloadTypeDemuxingUpdates(source);
  std::vector<ContentUpdate> content_updates;

  // Collect updates for each audio/video transceiver.
  // 更新 tranceivers
  for (const auto& transceiver : transceivers()->List()) {
    //  SessionDescriptionInterface* sdesc
    const ContentInfo* content_info =
        FindMediaSectionForTransceiver(transceiver, sdesc);
    // channel 就是 VideoChannel
    cricket::ChannelInterface* channel = transceiver->internal()->channel();
    if (!channel || !content_info || content_info->rejected) {
      continue;
    }
    // 需要看JsepTransport，SessionDescription？？？？
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

1. ContentUpdate ？？？？，MediaContentDescription
   
   ```c++
   struct ContentUpdate {
       ContentUpdate(cricket::ChannelInterface* channel,
                     const cricket::MediaContentDescription* content_description)
           : channel(channel), content_description(content_description) {}
       cricket::ChannelInterface* channel;
       const cricket::MediaContentDescription* content_description;
     };
   ```



### 11.2 SdpOfferAnswerHandler.ApplyChannelUpdates

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::ApplyChannelUpdates(
    SdpType type,
    cricket::ContentSource source,
    std::vector<PayloadTypeDemuxingUpdate> payload_type_demuxing_updates,
    std::vector<ContentUpdate> content_updates) {
  RTC_DCHECK_RUN_ON(pc_->worker_thread());
  // If this is answer-ish we're ready to let media flow.
  bool enable_sending = type == SdpType::kPrAnswer || type == SdpType::kAnswer;
  //????
  std::set<cricket::ChannelInterface*> modified_channels;
  //????
  for (const auto& update : payload_type_demuxing_updates) {
    modified_channels.insert(update.channel);
    update.channel->SetPayloadTypeDemuxingEnabled(update.enabled);
  }
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



### 2.4.3 BaseChannel.SetLocalContent

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



### 2.4.4 VideoChannel.SetLocalContent_w

```c++
bool VideoChannel::SetLocalContent_w(const MediaContentDescription* content,
                                     SdpType type,
                                     std::string* error_desc) {
  TRACE_EVENT0("webrtc", "VideoChannel::SetLocalContent_w");
  RTC_DCHECK_RUN_ON(worker_thread());
  RTC_LOG(LS_INFO) << "Setting local video description for " << ToString();

  RTC_DCHECK(content);
  if (!content) {
    SafeSetError("Can't find video content in local description.", error_desc);
    return false;
  }

  const VideoContentDescription* video = content->as_video();

  if (type == SdpType::kAnswer)
    SetNegotiatedHeaderExtensions_w(video->rtp_header_extensions());

  RtpHeaderExtensions rtp_header_extensions =
      GetFilteredRtpHeaderExtensions(video->rtp_header_extensions());
  SetReceiveExtensions(rtp_header_extensions);
  media_channel()->SetExtmapAllowMixed(video->extmap_allow_mixed());

  VideoRecvParameters recv_params = last_recv_params_;
  RtpParametersFromMediaDescription(
      video, rtp_header_extensions,
      webrtc::RtpTransceiverDirectionHasRecv(video->direction()), &recv_params);

  VideoSendParameters send_params = last_send_params_;

  bool needs_send_params_update = false;
  if (type == SdpType::kAnswer || type == SdpType::kPrAnswer) {
    for (auto& send_codec : send_params.codecs) {
      auto* recv_codec = FindMatchingCodec(recv_params.codecs, send_codec);
      if (recv_codec) {
        if (!recv_codec->packetization && send_codec.packetization) {
          send_codec.packetization.reset();
          needs_send_params_update = true;
        } else if (recv_codec->packetization != send_codec.packetization) {
          SafeSetError(
              "Failed to set local answer due to invalid codec packetization "
              "specified in m-section with mid='" +
                  content_name() + "'.",
              error_desc);
          return false;
        }
      }
    }
  }

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



### 2.4.5 BaseChannel.UpdateLocalStreams_w

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



### 2.4.6 WebRtcVideoChannel.AddSendStream——创建并保存WebRtcVideoSendStream

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

  // 创建WebRtcVideoSendStream 【章节2.4.7】
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

### 2.4.7 WebRtcVideoSendStream.WebRtcVideoSendStream

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

## 2.5 RtpSenderBase.SetSsrc—— videosoure 和encoder绑定

这是从【章节2】中`SdpOfferAnswerHandler.ApplyLocalDescription`

```c++
 transceiver->internal()->sender_internal()->SetSsrc(
            streams[0].first_ssrc());
```

设置ssrc的。

```c++
void RtpSenderBase::SetSsrc(uint32_t ssrc) {
  TRACE_EVENT0("webrtc", "RtpSenderBase::SetSsrc");
  if (stopped_ || ssrc == ssrc_) {
    return;
  }
  // If we are already sending with a particular SSRC, stop sending.
  if (can_send_track()) {
    // ssrc，videotrack，WebVideotrack 解除绑定
    ClearSend();
    // video 这是空实现
    RemoveTrackFromStats();
  }
  // 保存了ssrc
  ssrc_ = ssrc;
  if (can_send_track()) {
    // WebRtcVideoChannel 是在【章节2.4.6】addsendstream创建的
    // videotrack 设置给WebRtcVideoChannel【章节2.5.1】
    SetSend();
    // video 这是空实现
    AddTrackToStats();
  }
  if (!init_parameters_.encodings.empty()) {
    worker_thread_->Invoke<void>(RTC_FROM_HERE, [&] {
      RTC_DCHECK(media_channel_);
      // Get the current parameters, which are constructed from the SDP.
      // The number of layers in the SDP is currently authoritative to support
      // SDP munging for Plan-B simulcast with "a=ssrc-group:SIM <ssrc-id>..."
      // lines as described in RFC 5576.
      // All fields should be default constructed and the SSRC field set, which
      // we need to copy.
      RtpParameters current_parameters =
          media_channel_->GetRtpSendParameters(ssrc_);
      RTC_DCHECK_GE(current_parameters.encodings.size(),
                    init_parameters_.encodings.size());
      for (size_t i = 0; i < init_parameters_.encodings.size(); ++i) {
        init_parameters_.encodings[i].ssrc =
            current_parameters.encodings[i].ssrc;
        init_parameters_.encodings[i].rid = current_parameters.encodings[i].rid;
        current_parameters.encodings[i] = init_parameters_.encodings[i];
      }
      current_parameters.degradation_preference =
          init_parameters_.degradation_preference;
      media_channel_->SetRtpSendParameters(ssrc_, current_parameters);
      init_parameters_.encodings.clear();
    });
  }
  // Attempt to attach the frame decryptor to the current media channel.
  if (frame_encryptor_) {
    SetFrameEncryptor(frame_encryptor_);
  }
  if (frame_transformer_) {
    SetEncoderToPacketizerFrameTransformer(frame_transformer_);
  }
}
```

1. can_send_track
   
   ```c++
       // TODO(nisse): Since SSRC == 0 is technically valid, figure out
     // some other way to test if we have a valid SSRC.
     bool can_send_track() const { return track_ && ssrc_; }
   ```

#### 2.5.1  VideoRtpSender.SetSend()—— 把ssrc和track设置给WebRtcVideoChannel

```c++
void VideoRtpSender::SetSend() {
  RTC_DCHECK(!stopped_);
  RTC_DCHECK(can_send_track());
  if (!media_channel_) {
    RTC_LOG(LS_ERROR) << "SetVideoSend: No video channel exists.";
    return;
  }
  cricket::VideoOptions options;
  // VideoTrackSource
  VideoTrackSourceInterface* source = video_track()->GetSource();
  if (source) {
    options.is_screencast = source->is_screencast();
    options.video_noise_reduction = source->needs_denoising();
  }
  options.content_hint = cached_track_content_hint_;
  switch (cached_track_content_hint_) {
    case VideoTrackInterface::ContentHint::kNone:
      break;
    case VideoTrackInterface::ContentHint::kFluid:
      options.is_screencast = false;
      break;
    case VideoTrackInterface::ContentHint::kDetailed:
    case VideoTrackInterface::ContentHint::kText:
      options.is_screencast = true;
      break;
  }
  bool success = worker_thread_->Invoke<bool>(RTC_FROM_HERE, [&] {
    // video_media_channel() 返回的就是WebRtcVideoChannel，把ssrc和videotrack 丢入
    return video_media_channel()->SetVideoSend(ssrc_, &options, video_track());
  });
  RTC_DCHECK(success);
}
```

1. video_track,  就是在addvideotrack 一文中说的，VideoRtpSender.setTrack 的时候，设置的videoTrack
   
   ```c++
     rtc::scoped_refptr<VideoTrackInterface> video_track() const {
       return rtc::scoped_refptr<VideoTrackInterface>(
           static_cast<VideoTrackInterface*>(track_.get()));
     }
   ```

2. Video_media_channel
   
   ```c++
    cricket::VideoMediaChannel* video_media_channel() {
       return static_cast<cricket::VideoMediaChannel*>(media_channel_);
     }
   
   void RtpSenderBase::SetMediaChannel(cricket::MediaChannel* media_channel) {
     RTC_DCHECK(media_channel == nullptr ||
                media_channel->media_type() == media_type());
     media_channel_ = media_channel;
   }
   ```

#### 2.5.2 WebRtcVideoChannel.SetVideoSend——WebRtcVideoSendStream 和VideoSource 绑定

```c++
bool WebRtcVideoChannel::SetVideoSend(
    uint32_t ssrc,
    const VideoOptions* options,
    rtc::VideoSourceInterface<webrtc::VideoFrame>* source) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  TRACE_EVENT0("webrtc", "SetVideoSend");
  RTC_DCHECK(ssrc != 0);
  RTC_LOG(LS_INFO) << "SetVideoSend (ssrc= " << ssrc << ", options: "
                   << (options ? options->ToString() : "nullptr")
                   << ", source = " << (source ? "(source)" : "nullptr") << ")";
    // 通过ssrc 找到 WebRtcVideoSendStream
  // WebRtcVideoSendStream 是在【章节2.4.6】创建并加入send_streams_中
  const auto& kv = send_streams_.find(ssrc);
  if (kv == send_streams_.end()) {
    // Allow unknown ssrc only if source is null.
    RTC_CHECK(source == nullptr);
    RTC_LOG(LS_ERROR) << "No sending stream on ssrc " << ssrc;
    return false;
  }
    // WebRtcVideoSendStream 和VideoSource 绑定
  return kv->second->SetVideoSend(options, source);
}
```

1. Send_streams_, 
   
   ```c++
     // Using primary-ssrc (first ssrc) as key.
     std::map<uint32_t, WebRtcVideoSendStream*> send_streams_
         RTC_GUARDED_BY(thread_checker_);
   ```

#### 2.5.3 WebRtcVideoChannel.WebRtcVideoSendStream.SetVideoSend——VideoSendStream和VideoSource，并保存了videosource，但是没有设置给VideoSendStream

```c++
bool WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend(
 const VideoOptions* options,
 rtc::VideoSourceInterface<webrtc::VideoFrame>* source) {
  TRACE_EVENT0("webrtc", "WebRtcVideoSendStream::SetVideoSend");
  RTC_DCHECK_RUN_ON(&thread_checker_);

  if (options) {
 VideoOptions old_options = parameters_.options;
 parameters_.options.SetAll(*options);
 if (parameters_.options.is_screencast.value_or(false) !=
         old_options.is_screencast.value_or(false) &&
     parameters_.codec_settings) {
   // If screen content settings change, we may need to recreate the codec
   // instance so that the correct type is used.

   SetCodec(*parameters_.codec_settings);
   // Mark screenshare parameter as being updated, then test for any other
   // changes that may require codec reconfiguration.
   old_options.is_screencast = options->is_screencast;
 }
 if (parameters_.options != old_options) {
   // 
   ReconfigureEncoder();
 }
  }

  if (source_ && stream_) {
 // VideoSendStream stream_
 stream_->SetSource(nullptr, webrtc::DegradationPreference::DISABLED);
  }
  // 保存了videosource
  // Switch to the new source.
  source_ = source;
  // stream_ 是空的
  // channel 把 source 存了起来，但此时 channel 的 stream（`WebRtcVideoSendStream`）还没有创建，需要等 `SetRemoteDescription` 才会创建；
  // RecreateWebRtcStream 时候创建VideoSendStream
  if (source && stream_) {
    // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    // 这段语句是把源与编码器对象关联到一起!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
 stream_->SetSource(source_, GetDegradationPreference());
  }
  return true;
}
```

1. Stream_
   
   ```c++
   webrtc::VideoSendStream* stream_ RTC_GUARDED_BY(&thread_checker_);
   ```
   
   VideoSendStream 是在WebRtcVideoSendStream.SetCodec 调用生成的【章节3】
   
   SetLocalSsrc

## WebRtcVideoChannel.WebRtcVideoSendStream.ReconfigureEncoder

```c++
void WebRtcVideoChannel::WebRtcVideoSendStream::ReconfigureEncoder() {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  if (!stream_) {
    // The webrtc::VideoSendStream |stream_| has not yet been created but other
    // parameters has changed.
    return;
  }

  RTC_DCHECK_GT(parameters_.encoder_config.number_of_streams, 0);

  RTC_CHECK(parameters_.codec_settings);
  // VideoSendStreamParameters parameters_
  VideoCodecSettings codec_settings = *parameters_.codec_settings;

  // videoencode config 【章节】
  webrtc::VideoEncoderConfig encoder_config =
      CreateVideoEncoderConfig(codec_settings.codec);

  encoder_config.encoder_specific_settings =
      ConfigureVideoEncoderSettings(codec_settings.codec);

  // webrtc::VideoSendStream stream_
  stream_->ReconfigureVideoEncoder(encoder_config.Copy());

  encoder_config.encoder_specific_settings = NULL;

  parameters_.encoder_config = std::move(encoder_config);
}
```

1. parameters 

```c++
// Contains settings that are the same for all streams in the MediaChannel,
    // such as codecs, header extensions, and the global bitrate limit for the
    // entire channel.
    VideoSendStreamParameters parameters_ RTC_GUARDED_BY(&thread_checker_);
```

### WebRtcVideoChannel.WebRtcVideoSendStream.CreateVideoEncoderConfig

```c++
webrtc::VideoEncoderConfig
WebRtcVideoChannel::WebRtcVideoSendStream::CreateVideoEncoderConfig(
    const VideoCodec& codec) const {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  webrtc::VideoEncoderConfig encoder_config;
  encoder_config.codec_type = webrtc::PayloadStringToCodecType(codec.name);
  encoder_config.video_format =
      webrtc::SdpVideoFormat(codec.name, codec.params);

  bool is_screencast = parameters_.options.is_screencast.value_or(false);
  if (is_screencast) {
    encoder_config.min_transmit_bitrate_bps =
        1000 * parameters_.options.screencast_min_bitrate_kbps.value_or(0);
    encoder_config.content_type =
        webrtc::VideoEncoderConfig::ContentType::kScreen;
  } else {
    encoder_config.min_transmit_bitrate_bps = 0;
    encoder_config.content_type =
        webrtc::VideoEncoderConfig::ContentType::kRealtimeVideo;
  }

  // By default, the stream count for the codec configuration should match the
  // number of negotiated ssrcs. But if the codec is disabled for simulcast
  // or a screencast (and not in simulcast screenshare experiment), only
  // configure a single stream.
  encoder_config.number_of_streams = parameters_.config.rtp.ssrcs.size();
  if (IsCodecDisabledForSimulcast(codec.name, call_->trials())) {
    encoder_config.number_of_streams = 1;
  }

  // parameters_.max_bitrate comes from the max bitrate set at the SDP
  // (m-section) level with the attribute "b=AS." Note that we override this
  // value below if the RtpParameters max bitrate set with
  // RtpSender::SetParameters has a lower value.
  int stream_max_bitrate = parameters_.max_bitrate_bps;
  // When simulcast is enabled (when there are multiple encodings),
  // encodings[i].max_bitrate_bps will be enforced by
  // encoder_config.simulcast_layers[i].max_bitrate_bps. Otherwise, it's
  // enforced by stream_max_bitrate, taking the minimum of the two maximums
  // (one coming from SDP, the other coming from RtpParameters).
  if (rtp_parameters_.encodings[0].max_bitrate_bps &&
      rtp_parameters_.encodings.size() == 1) {
    stream_max_bitrate =
        MinPositive(*(rtp_parameters_.encodings[0].max_bitrate_bps),
                    parameters_.max_bitrate_bps);
  }

  // The codec max bitrate comes from the "x-google-max-bitrate" parameter
  // attribute set in the SDP for a specific codec. As done in
  // WebRtcVideoChannel::SetSendParameters, this value does not override the
  // stream max_bitrate set above.
  int codec_max_bitrate_kbps;
  if (codec.GetParam(kCodecParamMaxBitrate, &codec_max_bitrate_kbps) &&
      stream_max_bitrate == -1) {
    stream_max_bitrate = codec_max_bitrate_kbps * 1000;
  }
  encoder_config.max_bitrate_bps = stream_max_bitrate;

  // The encoder config's default bitrate priority is set to 1.0,
  // unless it is set through the sender's encoding parameters.
  // The bitrate priority, which is used in the bitrate allocation, is done
  // on a per sender basis, so we use the first encoding's value.
  encoder_config.bitrate_priority =
      rtp_parameters_.encodings[0].bitrate_priority;

  // Application-controlled state is held in the encoder_config's
  // simulcast_layers. Currently this is used to control which simulcast layers
  // are active and for configuring the min/max bitrate and max framerate.
  // The encoder_config's simulcast_layers is also used for non-simulcast (when
  // there is a single layer).
  RTC_DCHECK_GE(rtp_parameters_.encodings.size(),
                encoder_config.number_of_streams);
  RTC_DCHECK_GT(encoder_config.number_of_streams, 0);

  // Copy all provided constraints.
  encoder_config.simulcast_layers.resize(rtp_parameters_.encodings.size());
  for (size_t i = 0; i < encoder_config.simulcast_layers.size(); ++i) {
    encoder_config.simulcast_layers[i].active =
        rtp_parameters_.encodings[i].active;
    encoder_config.simulcast_layers[i].scalability_mode =
        rtp_parameters_.encodings[i].scalability_mode;
    if (rtp_parameters_.encodings[i].min_bitrate_bps) {
      encoder_config.simulcast_layers[i].min_bitrate_bps =
          *rtp_parameters_.encodings[i].min_bitrate_bps;
    }
    if (rtp_parameters_.encodings[i].max_bitrate_bps) {
      encoder_config.simulcast_layers[i].max_bitrate_bps =
          *rtp_parameters_.encodings[i].max_bitrate_bps;
    }
    if (rtp_parameters_.encodings[i].max_framerate) {
      encoder_config.simulcast_layers[i].max_framerate =
          *rtp_parameters_.encodings[i].max_framerate;
    }
    if (rtp_parameters_.encodings[i].scale_resolution_down_by) {
      encoder_config.simulcast_layers[i].scale_resolution_down_by =
          *rtp_parameters_.encodings[i].scale_resolution_down_by;
    }
    if (rtp_parameters_.encodings[i].num_temporal_layers) {
      encoder_config.simulcast_layers[i].num_temporal_layers =
          *rtp_parameters_.encodings[i].num_temporal_layers;
    }
  }

  encoder_config.legacy_conference_mode = parameters_.conference_mode;

  int max_qp = kDefaultQpMax;
  codec.GetParam(kCodecParamMaxQuantization, &max_qp);
  encoder_config.video_stream_factory =
      new rtc::RefCountedObject<EncoderStreamFactory>(
          codec.name, max_qp, is_screencast, parameters_.conference_mode);
  return encoder_config;
}
```

### 流程

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ljZV9seTAwMA==,size_16,color_FFFFFF,t_70.png)

- 绿色为两个起点和一个终点：启动Transport创建的两个顶层API，给DataChanel设置Transport的终点站。

- 红色的部分为创建DataChannel可使用的Transport实体类的地方；

- 蓝色部分主要体现在方法JsepTransportController::MaybeCreateJsepTransport中，该方法创建了WebRTC中Transport的所有类别，每个蓝色和红色部分都在创建不同层次的一个Transport。

### QA

- PushdownTransportDescription

- ContentInfo 是什么
  
  > Unified Plan，一个m行用一个ContentInfo存，会建立一个transceiver(mid-mline_index)，一个transceiver只有一个sender/receiver(本端的SDP建sender，远端的SDP建receiver)。
  > 一个ContentInfo有一个StreamParams，StreamParams的id就是track-id，也就是sender-id/receiver-id，一个StreamParams存有多个SSRC和SsrcGroup，建立WebRtcAudioSendStream时使用first_ssrc，RTP报文里携带的。
  
  ```c++
  // Represents a session description section. Most information about the section
  // is stored in the description, which is a subclass of MediaContentDescription.
  // Owns the description.
  class RTC_EXPORT ContentInfo {
   public:
    explicit ContentInfo(MediaProtocolType type) : type(type) {}
    ~ContentInfo();
    // Copy
    ContentInfo(const ContentInfo& o);
    ContentInfo& operator=(const ContentInfo& o);
    ContentInfo(ContentInfo&& o) = default;
    ContentInfo& operator=(ContentInfo&& o) = default;
  
    // Alias for |name|.
    std::string mid() const { return name; }
    void set_mid(const std::string& mid) { this->name = mid; }
  
    // Alias for |description|.
    MediaContentDescription* media_description();
    const MediaContentDescription* media_description() const;
  
    void set_media_description(std::unique_ptr<MediaContentDescription> desc) {
      description_ = std::move(desc);
    }
  
    // TODO(bugs.webrtc.org/8620): Rename this to mid.
    // a=mid:0, 0 就是name
    std::string name;
    // MediaProtocolType，a/v 用rtp；datachannel 用 sctp
    MediaProtocolType type;
    bool rejected = false;
    // 是否公用一个传输通道
    bool bundle_only = false;
  
   private:
    friend class SessionDescription;
    // 存储了AudioContentDescription/VideoContentDescription/DataContentDescription 相关属性
    std::unique_ptr<MediaContentDescription> description_;
  };
  
  // Protocol used for encoding media. This is the "top level" protocol that may
  // be wrapped by zero or many transport protocols (UDP, ICE, etc.).
  enum class MediaProtocolType {
    kRtp,   // Section will use the RTP protocol (e.g., for audio or video).
            // https://tools.ietf.org/html/rfc3550
    kSctp,  // Section will use the SCTP protocol (e.g., for a data channel).
            // https://tools.ietf.org/html/rfc4960
    kOther  // Section will use another top protocol which is not
            // explicitly supported.
  };
  ```
  
  [webrtc的sdp相关结构1](https://blog.csdn.net/myiloveuuu/article/details/78998168)
  
  https://blog.csdn.net/xyphf/article/details/107749806?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-1&spm=1001.2101.3001.4242
  
  https://andyli.blog.csdn.net/article/details/96355737?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-3.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-3.control
  
  https://www.wangzhumo.com/2019/12/23/session-description-protocol/
  
  https://segmentfault.com/a/1190000038272539
  
  https://www.codenong.com/cs109714286/

- AssociateTransceiver() 通过mid找到 Transceiver

- BaseChannel.UpdateLocalStreams_w 需要分析？？？

- RtpTransportInternal* rtp_transport = pc_->GetRtpTransport(mid);???
  
  【章节2.3.1】

- VideoChannel和WebRTCVideoChannel 的参数

- GetPayloadTypeDemuxingUpdates

- MediaContentDescription

- sctp_transport->Start

- ssrc 的作用是什么？？？
  是rtcp/rtp时候用到，识别源的唯一识别码

- transpot 怎么transceiver 如何关联 
  【章节2】中有备注

- chanel 和transport 是什么时候关联
  【章节2.3.1】在VideoChannel 构造函数，把Transport设置给VideoChannel 

- JsepTransport, JsepTransportDescription, DtlsTranSpot,DtlsSrtpTransport 都是什么
  JsepTransport 是在【章节2.2.3】创建的，他拥有各种tanspot
  JsepTransportDescription 包含了SessionDesscription

- webrtc 中怎么根据 SDP 创建或关联底层的 socket 对象？
  https://blog.csdn.net/freeabc/article/details/110141938

- 其实也就是 JsepSessionDescription 对象，有关 SDP 的具体数据就是 SessionDescription
  
  ![img]({{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZyZWVhYmM=,size_16,color_FFFFFF,t_70-20210423180447192.png)
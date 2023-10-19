---
layout: post
title: AddSendStream, internal::VideoSendStream, WebRtcVideoSendStream
date: 2023-10-06 08:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


## 0. 前言

1. SenderOptions--> StreamParams

2. 根据 StreamParams 在`WebRtcVideoChannel`创建 WebRtcVideoSendStream， 存放在`std::map<uint32_t, WebRtcVideoSendStream*> send_streams_ `, 以 `uint32_t ssrc = sp.first_ssrc();` 为key；

3. `RtpSenderBase.SetSsrc` 设置ssrc的值，最终会在`WebRtcVideoSendStream` 中通过Call创建了`internal::VideoSendStream`；

   ```cpp
   RtpSenderBase.SetSsrc
   	VideoRtpSender.SetSend
   		WebRtcVideoChannel.SetVideoSend // MediaChannel
   			WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend
   				WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec
   					WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream
   						Call::CreateVideoSendStream
   							internal::VideoSendStream::VideoSendStream
   				WebRtcVideoChannel.WebRtcVideoSendStream.ReconfigureEncoder
   					WebRtcVideoChannel.WebRtcVideoSendStream.CreateVideoEncoderConfig
   				webrtc::VideoSendStream::SetSource
   ```

   



## 1. SenderOptions--> StreamParams

media/base/stream_params.cc

```cpp
StreamParams{
	...
	// A unique identifier of the StreamParams object. When the SDP is created,
  // this comes from the track ID of the sender that the StreamParams object
  // is associated with.
  // track id
  std::string id;
  // There may be no SSRCs stored in unsignaled case when stream_ids are
  // signaled with a=msid lines.
  std::vector<uint32_t> ssrcs;         // All SSRCs for this source
  ...
}
```

- trackId，

- ssrcs 集合，包括 nack + 原始等； simulcast ，多层的时候

  - 根据num_layers 生成 ssrcs，有几层就有几个
  - nack，AddFidSsrc， 第一条件有几个对应生成几个
  - fec，AddFecFrSsrc， 第一条件有几个对应生成几个

  ```cpp
  void StreamParams::GenerateSsrcs(int num_layers,
                                   bool generate_fid,
                                   bool generate_fec_fr,
                                   rtc::UniqueRandomIdGenerator* ssrc_generator) {
    RTC_DCHECK_GE(num_layers, 0);
    RTC_DCHECK(ssrc_generator);
    std::vector<uint32_t> primary_ssrcs;
    for (int i = 0; i < num_layers; ++i) {
      uint32_t ssrc = ssrc_generator->GenerateId();
      primary_ssrcs.push_back(ssrc);
      //!!!!!!!!!!!!!!!! 
      //   void add_ssrc(uint32_t ssrc) { ssrcs.push_back(ssrc); }
      add_ssrc(ssrc);
    }
  
    if (num_layers > 1) {
      SsrcGroup simulcast(kSimSsrcGroupSemantics, primary_ssrcs);
      ssrc_groups.push_back(simulcast);
    }
  
    if (generate_fid) {
      for (uint32_t ssrc : primary_ssrcs) {
        AddFidSsrc(ssrc, ssrc_generator->GenerateId());
      }
    }
  
    if (generate_fec_fr) {
      for (uint32_t ssrc : primary_ssrcs) {
        AddFecFrSsrc(ssrc, ssrc_generator->GenerateId());
      }
    }
  }
  
  
  bool StreamParams::AddSecondarySsrc(const std::string& semantics,
                                      uint32_t primary_ssrc,
                                      uint32_t secondary_ssrc) {
    if (!has_ssrc(primary_ssrc)) {
      return false;
    }
  
    // !!!!!!!!!!!!!!!
    ssrcs.push_back(secondary_ssrc);
    ssrc_groups.push_back(SsrcGroup(semantics, {primary_ssrc, secondary_ssrc}));
    return true;
  }
  ```

  



## 2. 创建WebRtcVideoSendStream——BaseChannel.UpdateLocalStreams_w

pc/channel.cc

```cpp
SdpOfferAnswerHandler::UpdateSessionState
  SdpOfferAnswerHandler::PushdownMediaDescription
	SdpOfferAnswerHandler.ApplyChannelUpdates
    BaseChannel::SetLocalContent
    VideoChannel::SetLocalContent_w
			WebRtcVideoChannel::SetRecvParameters
			BaseChannel.UpdateLocalStreams_w
				WebRtcVideoChannel::AddSendStream
					WebRtcVideoChannel::WebRtcVideoSendStream::WebRtcVideoSendStream
```

这里流程具体参考【set-local-description】。
根据StreamParmsVec ，调用`WebRtcVideoChannel::AddSendStream` ， 根据StreamParms 创建 WebRtcVideoSendStream 。

> - 就是说，有多少StreamParms， 就对应多少个WebRtcVideoSendStream。
> - SenderOptions---》StreamParms， 1:1
> - 在Unified plan的情况下，一个Track 对应 一个SenderOptions，一个transceiver

```cpp
// Using primary-ssrc (first ssrc) as key.
std::map<uint32_t, WebRtcVideoSendStream*> send_streams_  RTC_GUARDED_BY(thread_checker_);
```

>**注意**
>
>这里的key是一primary-ssrc (first ssrc) as key，就是`StreamParams.first_ssrc()`。



### 2.1 创建WebRtcVideoSendStream——WebRtcVideoChannel::AddSendStream

media/engine/webrtc_video_engine.cc

```cpp
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

// !!!!!!!!!!!!!!!!!
// !!!!!!!!!!!!!!!!!
  // 创建WebRtcVideoSendStream
  WebRtcVideoSendStream* stream = new WebRtcVideoSendStream(
      call_, sp, std::move(config), default_send_options_,
      video_config_.enable_cpu_adaptation, bitrate_config_.max_bitrate_bps,
      send_codec_, send_rtp_extensions_, send_params_);
// !!!!!!!!!!!!!!!!!
// !!!!!!!!!!!!!!!!!

// !!!!!!!!!!!!!!!!!
// std::map<uint32_t, WebRtcVideoSendStream*> send_streams_ 
// 存放了是一个StreamParams的第一个元素作为key
// !!!!!!!!!!!!!!!!!
  uint32_t ssrc = sp.first_ssrc();
  RTC_DCHECK(ssrc != 0);
  // 保存stream，key 是ssrc
  send_streams_[ssrc] = stream;
// !!!!!!!!!!!!!!!!!
// !!!!!!!!!!!!!!!!!
// !!!!!!!!!!!!!!!!!

  if (rtcp_receiver_report_ssrc_ == kDefaultRtcpReceiverReportSsrc) {
    rtcp_receiver_report_ssrc_ = ssrc;
    RTC_LOG(LS_INFO)
        << "SetLocalSsrc on all the receive streams because we added "
           "a send stream.";
    for (auto& kv : receive_streams_)
      kv.second->SetLocalSsrc(ssrc);
  }
  // 在WebRtcVideoChannel::SetSend 设置sending_
  if (sending_) {
    // WebRtcVideoSendStream stream
    stream->SetSend(true);
  }

  return true;
}
```

1. 创建WebRtcVideoSendStream

2. `std::map<uint32_t, WebRtcVideoSendStream*> send_streams_ `,  以`StreamParams.first_ssrc()` 为key，WebRtcVideoSendStream为value；

3. 发送数据，`sending_` 是在` WebRtcVideoChannel::SetSend` 设置值
   media/engine/webrtc_video_engine.cc

   ```cpp
   bool WebRtcVideoChannel::SetSend(bool send) {
     RTC_DCHECK_RUN_ON(&thread_checker_);
     TRACE_EVENT0("webrtc", "WebRtcVideoChannel::SetSend");
     RTC_LOG(LS_VERBOSE) << "SetSend: " << (send ? "true" : "false");
     if (send && !send_codec_) {
       RTC_DLOG(LS_ERROR) << "SetSend(true) called before setting codec.";
       return false;
     }
     // WebRtcVideoSendStream kv;
     for (const auto& kv : send_streams_) {
       kv.second->SetSend(send);
     }
     sending_ = send;
     return true;
   }
   ```

   

#### -- WebRtcVideoSendStream::WebRtcVideoSendStream



#### WebRtcVideoSendStream::SetSend

media/engine/webrtc_video_engine.cc

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::SetSend(bool send) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  sending_ = send;
  UpdateSendState();
}
```





## 3. Track和Encoder的数据链接——RtpSenderBase.SetSsrc

pc/rtp_sender.h

```cpp
RtpSenderBase.SetSsrc
	VideoRtpSender.SetSend
		WebRtcVideoChannel.SetVideoSend
			WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend
				WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec
					WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream
						Call::CreateVideoSendStream
							webrtc::VideoSendStream::VideoSendStream
				WebRtcVideoChannel.WebRtcVideoSendStream.ReconfigureEncoder
					WebRtcVideoChannel.WebRtcVideoSendStream.CreateVideoEncoderConfig
				webrtc::VideoSendStream::SetSource
```

在`pc/sdp_offer_answer.cc` 的`SdpOfferAnswerHandler::ApplyLocalDescription`。

设置ssrc的时候，**只取ssrc的vector的第一个元素。**

```cpp
RTCError SdpOfferAnswerHandler::ApplyLocalDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {
  ...

  if (IsUnifiedPlan()) {
    for (const auto& transceiver : transceivers()->List()) {
      if (transceiver->stopped()) {
        continue;
      }
      const ContentInfo* content =
          FindMediaSectionForTransceiver(transceiver, local_description());
      if (!content) {
        continue;
      }
      cricket::ChannelInterface* channel = transceiver->internal()->channel();
      if (content->rejected || !channel || channel->local_streams().empty()) {
        // 0 is a special value meaning "this sender has no associated send
        // stream". Need to call this so the sender won't attempt to configure
        // a no longer existing stream and run into DCHECKs in the lower
        // layers.
        transceiver->internal()->sender_internal()->SetSsrc(0);
      } else {
        // Get the StreamParams from the channel which could generate SSRCs.
        const std::vector<StreamParams>& streams = channel->local_streams();
        transceiver->internal()->sender_internal()->set_stream_ids(
            streams[0].stream_ids());
        //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!! SetSsrc, 设置的streams[0].first_ssrc() !!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
        transceiver->internal()->sender_internal()->SetSsrc(
            streams[0].first_ssrc());
      }
    }
  }
  ...

}
```

- 针对每个transceiver的VideoRtpSender 进行设置 SetSsrc

- 设置ssrc是取ssrcs中的第一个元素
  `media/base/stream_params.h`

  ```cpp
  StreamParams {
  ....
  	std::vector<uint32_t> ssrcs;         // All SSRCs for this source
  
   	uint32_t first_ssrc() const {
      if (ssrcs.empty()) {
        return 0;
      }
  
      return ssrcs[0];
    }
    ...
  }
  ```

- 设置ssrc， 这样把Track和encoder链接起来，Track提供数据，encoder 接收数据



### 3.1 VideoRtpSender.SetSend

pc/rtp_sender.h

```cpp
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
  // 在弱网情况下，设置视频丢帧，降清晰度的策略
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
    // ssrc_就是 setSsrc 设置的值，ssrc[0]
    return video_media_channel()->SetVideoSend(ssrc_, &options, video_track());
  });
  RTC_DCHECK(success);
}
```

- cricket::VideoOptions::is_screencast，根据VideoTrackInterface::ContentHint， 进行设置；



#### 3.1.1 VideoOptions

media/base/media_channel.h

```cpp
struct VideoOptions {

  std::string ToString() const {
    rtc::StringBuilder ost;
    ost << "VideoOptions {";
    ost << ToStringIfSet("noise reduction", video_noise_reduction);
    ost << ToStringIfSet("screencast min bitrate kbps",
                         screencast_min_bitrate_kbps);
    ost << ToStringIfSet("is_screencast ", is_screencast);
    ost << "}";
    return ost.Release();
  }
  
  // 覆盖值
  void SetAll(const VideoOptions& change) {
    SetFrom(&video_noise_reduction, change.video_noise_reduction);
    SetFrom(&screencast_min_bitrate_kbps, change.screencast_min_bitrate_kbps);
    SetFrom(&is_screencast, change.is_screencast);
  }

  // Enable denoising? This flag comes from the getUserMedia
  // constraint 'googNoiseReduction', and WebRtcVideoEngine passes it
  // on to the codec options. Disabled by default.
  absl::optional<bool> video_noise_reduction;
  // Force screencast to use a minimum bitrate. This flag comes from
  // the PeerConnection constraint 'googScreencastMinBitrate'. It is
  // copied to the encoder config by WebRtcVideoChannel.
  absl::optional<int> screencast_min_bitrate_kbps;
  // Set by screencast sources. Implies selection of encoding settings
  // suitable for screencast. Most likely not the right way to do
  // things, e.g., screencast of a text document and screencast of a
  // youtube video have different needs.
  absl::optional<bool> is_screencast;
  webrtc::VideoTrackInterface::ContentHint content_hint;
};
```

##### absl::optional<bool> video_noise_reduction;

##### absl::optional<int> screencast_min_bitrate_kbps;

##### absl::optional<bool> is_screencast;

对应的编码器，更加适合屏幕共享。

##### webrtc::VideoTrackInterface::ContentHint content_hint;



#### 3.1.2 WebRtcVideoChannel::SetVideoSend

media/engine/webrtc_video_engine.cc

```cpp
// MediaChannel
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

  const auto& kv = send_streams_.find(ssrc);
  if (kv == send_streams_.end()) {
    // Allow unknown ssrc only if source is null.
    RTC_CHECK(source == nullptr);
    RTC_LOG(LS_ERROR) << "No sending stream on ssrc " << ssrc;
    return false;
  }
	// WebRtcVideoSendStream
  return kv->second->SetVideoSend(options, source);
}
```





##### --WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend

media/engine/webrtc_video_engine.cc



### 3.2 !!! WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend

media/engine/webrtc_video_engine.cc

```cpp
bool WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend(
 const VideoOptions* options,
 	rtc::VideoSourceInterface<webrtc::VideoFrame>* source) {
	...

  if (options) {
    // VideoSendStreamParameters parameters_
     VideoOptions old_options = parameters_.options;
    // 新的options 覆盖旧的 options
     parameters_.options.SetAll(*options);
    // 默认情况下，parameters_.options.is_screencast 是null
    //
    // is_screencast 是在 VideoRtpSender::SetSend 赋值的
     if (parameters_.options.is_screencast.value_or(false) !=
             old_options.is_screencast.value_or(false) &&
         parameters_.codec_settings) {
       // If screen content settings change, we may need to recreate the codec
       // instance so that the correct type is used.

       // 1. 创建 webrtc::VideoSendStream
       SetCodec(*parameters_.codec_settings);
       // Mark screenshare parameter as being updated, then test for any other
       // changes that may require codec reconfiguration.
       old_options.is_screencast = options->is_screencast;
     }
    
     if (parameters_.options != old_options) {
       // 2. 创建编码器
       ReconfigureEncoder();
     }
  }

  if (source_ && stream_) {
     // 3. webrtc::VideoSendStream stream_， 解绑上一个source
     stream_->SetSource(nullptr, webrtc::DegradationPreference::DISABLED);
  }
  // 4. 保存了videosource
  // Switch to the new source.
  source_ = source;
  // stream_ 是空的
  // channel 把 source 存了起来，但此时 channel 的 stream（`WebRtcVideoSendStream`）还没有创建，需要等 		`SetRemoteDescription` 才会创建；
  // RecreateWebRtcStream 时候创建VideoSendStream
  if (source && stream_) {
    // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    // 5. 把源与编码器对象关联到一起!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
 		stream_->SetSource(source_, GetDegradationPreference());
  }
  return true;
}
```

1. VideoOptions.is_screencast 如果值发生变化，则调用SetCodec
1. VideoOptions 变更，则调用ReconfigureEncoder， 创建编码器或者重启；
2. webrtc::VideoSendStream 和Source绑定；



#### 3.2.1 WebRtcVideoSendStream::VideoSendStreamParameters

media/engine/webrtc_video_engine.h

```cpp
struct VideoSendStreamParameters {
  VideoSendStreamParameters(
    webrtc::VideoSendStream::Config config,
    const VideoOptions& options,
    int max_bitrate_bps,
    const absl::optional<VideoCodecSettings>& codec_settings);

  webrtc::VideoSendStream::Config config;
  VideoOptions options;
  int max_bitrate_bps;
  bool conference_mode;
  absl::optional<VideoCodecSettings> codec_settings;
  // Sent resolutions + bitrates etc. by the underlying VideoSendStream,
  // typically changes when setting a new resolution or reconfiguring
  // bitrates.
  webrtc::VideoEncoderConfig encoder_config;
};
```

##### webrtc::VideoSendStream::Config config

##### VideoOptions options

##### absl::optional< VideoCodecSettings > codec_settings

##### webrtc::VideoEncoderConfig encoder_config





#### 3.2.2 -- WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec

调用 RecreateWebRtcStream



#### 3.2.3 -- WebRtcVideoChannel.WebRtcVideoSendStream.ReconfigureEncoder



#### 3.2.4 -- VideoSendStream::SetSource

VideoSendStream和rtc::VideoSourceInterface 绑定，同时提供策略



##### DegradationPreference

api/rtp_parameters.h

```cpp
// Based on the spec in
// https://w3c.github.io/webrtc-pc/#idl-def-rtcdegradationpreference.
// These options are enforced on a best-effort basis. For instance, all of
// these options may suffer some frame drops in order to avoid queuing.
// TODO(sprang): Look into possibility of more strictly enforcing the
// maintain-framerate option.
// TODO(deadbeef): Default to "balanced", as the spec indicates?
enum class DegradationPreference {
  // Don't take any actions based on over-utilization signals. Not part of the
  // web API.
  DISABLED,
  // On over-use, request lower resolution, possibly causing down-scaling.
  MAINTAIN_FRAMERATE,
  // On over-use, request lower frame rate, possibly causing frame drops.
  MAINTAIN_RESOLUTION,
  // Try to strike a "pleasing" balance between frame rate or resolution.
  BALANCED,
};
```



### 3.3 WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec——RecreateWebRtcStream

media/engine/webrtc_video_engine.cc

 调用 RecreateWebRtcStream

```cpp
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



#### VideoCodecSettings

```cpp
  struct VideoCodecSettings {
    VideoCodecSettings();

    // Checks if all members of |*this| are equal to the corresponding members
    // of |other|.
    bool operator==(const VideoCodecSettings& other) const;
    bool operator!=(const VideoCodecSettings& other) const;

    // Checks if all members of |a|, except |flexfec_payload_type|, are equal
    // to the corresponding members of |b|.
    static bool EqualsDisregardingFlexfec(const VideoCodecSettings& a,
                                          const VideoCodecSettings& b);

    VideoCodec codec;
    webrtc::UlpfecConfig ulpfec;
    int flexfec_payload_type;  // -1 if absent.
    int rtx_payload_type;      // -1 if absent.
  };
```





#### WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream

media/engine/webrtc_video_engine.cc

```cpp
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



##### --Call::CreateVideoSendStream



### 3.4 !!! 创建internal::VideoSendStream —— Call::CreateVideoSendStream

call/call.cc

```cpp
// webrtc::VideoSendStream::Config config 是从 parameters_获取到的； 
// WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream 该函数
//
// This method can be used for Call tests with external fec controller factory.
webrtc::VideoSendStream* Call::CreateVideoSendStream(
    webrtc::VideoSendStream::Config config,
    VideoEncoderConfig encoder_config,
    std::unique_ptr<FecController> fec_controller) {
  TRACE_EVENT0("webrtc", "Call::CreateVideoSendStream");
  RTC_DCHECK_RUN_ON(worker_thread_);

  EnsureStarted();

  video_send_delay_stats_->AddSsrcs(config);
  for (size_t ssrc_index = 0; ssrc_index < config.rtp.ssrcs.size();
       ++ssrc_index) {
    event_log_->Log(std::make_unique<RtcEventVideoSendStreamConfig>(
        CreateRtcLogStreamConfig(config, ssrc_index)));
  }

  // TODO(mflodman): Base the start bitrate on a current bandwidth estimate, if
  // the call has already started.
  // Copy ssrcs from |config| since |config| is moved.
  std::vector<uint32_t> ssrcs = config.rtp.ssrcs;

  // 创建 internal::VideoSendStream
  VideoSendStream* send_stream = new VideoSendStream(
      clock_, num_cpu_cores_, module_process_thread_->process_thread(),
      task_queue_factory_, call_stats_->AsRtcpRttStats(), transport_send_ptr_,
      bitrate_allocator_.get(), video_send_delay_stats_.get(), event_log_,
      std::move(config), std::move(encoder_config), suspended_video_send_ssrcs_,
      suspended_video_payload_states_, std::move(fec_controller));

  // webrtc::VideoSendStream::Config
  // ssrcs = config.rtp.ssrcs;
  for (uint32_t ssrc : ssrcs) {
    RTC_DCHECK(video_send_ssrcs_.find(ssrc) == video_send_ssrcs_.end());
    //！！！！！！！！！！！！！！！！！！
    //！！！！！！！！！！！！！！！！！！
    //！！！！！！！！！！！！！！！！！！
    // std::map<uint32_t, VideoSendStream*> video_send_ssrcs_
    // 所有的ssrc 共用VideoSendStream 对象
    //！！！！！！！！！！！！！！！！！！
    //！！！！！！！！！！！！！！！！！！
    //！！！！！！！！！！！！！！！！！！
    video_send_ssrcs_[ssrc] = send_stream;
  }
  // std::set<VideoSendStream*> video_send_streams
  video_send_streams_.insert(send_stream);
  // Forward resources that were previously added to the call to the new stream.
  for (const auto& resource_forwarder : adaptation_resource_forwarders_) {
    resource_forwarder->OnCreateVideoSendStream(send_stream);
  }

  UpdateAggregateNetworkState();

  return send_stream;
}
```



```cpp

  std::map<uint32_t, AudioSendStream*> audio_send_ssrcs_
      RTC_GUARDED_BY(worker_thread_);
// 这个没发现有什么用，除了删除对象
  std::map<uint32_t, VideoSendStream*> video_send_ssrcs_
      RTC_GUARDED_BY(worker_thread_);
  std::set<VideoSendStream*> video_send_streams_ RTC_GUARDED_BY(worker_thread_);

```



#### --internal::VideoSendStream::VideoSendStream

libs/mediaengine/third_party/WebRTC/src/video/video_send_stream.h



#### webrtc::VideoSendStream::Config

call/video_send_stream.h

```cpp
struct Config {
   public:
    Config() = delete;
    Config(Config&&);
    explicit Config(Transport* send_transport);

    Config& operator=(Config&&);
    Config& operator=(const Config&) = delete;

    ~Config();

    // Mostly used by tests.  Avoid creating copies if you can.
    Config Copy() const { return Config(*this); }

    std::string ToString() const;

    RtpConfig rtp;

    VideoStreamEncoderSettings encoder_settings;

    // Time interval between RTCP report for video
    int rtcp_report_interval_ms = 1000;

    // Transport for outgoing packets.
    Transport* send_transport = nullptr;

    // Expected delay needed by the renderer, i.e. the frame will be delivered
    // this many milliseconds, if possible, earlier than expected render time.
    // Only valid if |local_renderer| is set.
    int render_delay_ms = 0;

    // Target delay in milliseconds. A positive value indicates this stream is
    // used for streaming instead of a real-time call.
    int target_delay_ms = 0;

    // True if the stream should be suspended when the available bitrate fall
    // below the minimum configured bitrate. If this variable is false, the
    // stream may send at a rate higher than the estimated available bitrate.
    bool suspend_below_min_bitrate = false;

    // Enables periodic bandwidth probing in application-limited region.
    bool periodic_alr_bandwidth_probing = false;

    // An optional custom frame encryptor that allows the entire frame to be
    // encrypted in whatever way the caller chooses. This is not required by
    // default.
    rtc::scoped_refptr<webrtc::FrameEncryptorInterface> frame_encryptor;

    // Per PeerConnection cryptography options.
    CryptoOptions crypto_options;

    rtc::scoped_refptr<webrtc::FrameTransformerInterface> frame_transformer;

   private:
    // Access to the copy constructor is private to force use of the Copy()
    // method for those exceptional cases where we do use it.
    Config(const Config&);
  };
```





#### config.rtp.ssrcs

参数传入

```
 webrtc::VideoSendStream::Config
```





### 3.5 填充 webrtc::VideoSendStream::Config





#### 3.5.1 创建webrtc::VideoSendStream::Config —— WebRtcVideoChannel::AddSendStream(const StreamParams& sp)

```cpp
bool WebRtcVideoChannel::AddSendStream(const StreamParams& sp) {
	...

  for (uint32_t used_ssrc : sp.ssrcs)
    send_ssrcs_.insert(used_ssrc);

  //////////!!!!!!!
  //////////!!!!!!!
  //////////!!!!!!!
  webrtc::VideoSendStream::Config config(this);
  //////////!!!!!!!
  //////////!!!!!!!
  //////////!!!!!!!

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
    	// ！！！！！！！！！！！！！！！
    	// ！！！！！！！！！！！！！！！
      call_, sp, std::move(config), default_send_options_,
    	// ！！！！！！！！！！！！！！！
    	// ！！！！！！！！！！！！！！！
      video_config_.enable_cpu_adaptation, bitrate_config_.max_bitrate_bps,
      send_codec_, send_rtp_extensions_, send_params_);

 ...

  return true;
}
```





#### 3.5.2 填充了ssrcs属性——WebRtcVideoSendStream.WebRtcVideoSendStream

```cpp
WebRtcVideoChannel::WebRtcVideoSendStream::WebRtcVideoSendStream(
    webrtc::Call* call,
    const StreamParams& sp,
    //////////!!!!!!!
    webrtc::VideoSendStream::Config config,
    //////////!!!!!!!
    const VideoOptions& options,
    bool enable_cpu_overuse_detection,
    int max_bitrate_bps,
    const absl::optional<VideoCodecSettings>& codec_settings,
    const absl::optional<std::vector<webrtc::RtpExtension>>& rtp_extensions,
    // TODO(deadbeef): Don't duplicate information between send_params,
    // rtp_extensions, options, etc.
    const VideoSendParameters& send_params):
    ...
       parameters_(std::move(config),
    ....{
  
    	...
  		sp.GetPrimarySsrcs(&parameters_.config.rtp.ssrcs);
      ...
    }
```

1. 将`webrtc::VideoSendStream::Config config`存放到`parameters_`。
2. 填充了ssrcs





##### StreamParams::GetPrimarySsrcs

```cpp
void StreamParams::GetPrimarySsrcs(std::vector<uint32_t>* ssrcs) const {
  const SsrcGroup* sim_group = get_ssrc_group(kSimSsrcGroupSemantics);
  if (sim_group == NULL) {
    // 无simulcast
    ssrcs->push_back(first_ssrc());
  } else {
    // simulcast
    ssrcs->insert(ssrcs->end(), sim_group->ssrcs.begin(),
                  sim_group->ssrcs.end());
  }
}
```





#### 3.5.3 根据VideoCodecSettings填充了codec，playload相关属性——WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec

```cpp
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





#### 3.5.4 WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream() {
 ...
   //////!!!!!!!!!!!!!!!!!!!!!!
   //////!!!!!!!!!!!!!!!!!!!!!!
   //////!!!!!!!!!!!!!!!!!!!!!!
   //////!!!!!!!!!!!!!!!!!!!!!!
  webrtc::VideoSendStream::Config config = parameters_.config.Copy();
   //////!!!!!!!!!!!!!!!!!!!!!!
   //////!!!!!!!!!!!!!!!!!!!!!!
   //////!!!!!!!!!!!!!!!!!!!!!!
   //////!!!!!!!!!!!!!!!!!!!!!!
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
  ///////////！！！！！////////
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());
  ///////////！！！！！////////

 ...
}
```





## 4. internal::VideoSendStream::VideoSendStream

video/video_send_stream.h

```cpp
namespace internal {

  class VideoSendStreamImpl;

  // VideoSendStream implements webrtc::VideoSendStream.
  // Internally, it delegates all public methods to VideoSendStreamImpl and / or
  // VideoStreamEncoder. VideoSendStreamInternal is created and deleted on
  // |worker_queue|.
  class VideoSendStream : public webrtc::VideoSendStream {
  ...
    SendStatisticsProxy stats_proxy_;
    const VideoSendStream::Config config_;
    const VideoEncoderConfig::ContentType content_type_;
    std::unique_ptr<VideoSendStreamImpl> send_stream_;
    std::unique_ptr<VideoStreamEncoderInterface> video_stream_encoder_;
  };
}
```





```cpp
VideoSendStream::VideoSendStream(
    Clock* clock,
    int num_cpu_cores,
    ProcessThread* module_process_thread,
    TaskQueueFactory* task_queue_factory,
    RtcpRttStats* call_stats,
    RtpTransportControllerSendInterface* transport,
    BitrateAllocatorInterface* bitrate_allocator,
    SendDelayStats* send_delay_stats,
    RtcEventLog* event_log,
    VideoSendStream::Config config,
    VideoEncoderConfig encoder_config,
    const std::map<uint32_t, RtpState>& suspended_ssrcs,
    const std::map<uint32_t, RtpPayloadState>& suspended_payload_states,
    std::unique_ptr<FecController> fec_controller)
    : worker_queue_(transport->GetWorkerQueue()),
      stats_proxy_(clock, config, encoder_config.content_type),
      config_(std::move(config)),
      content_type_(encoder_config.content_type) {
  RTC_DCHECK(config_.encoder_settings.encoder_factory);
  RTC_DCHECK(config_.encoder_settings.bitrate_allocator_factory);

  video_stream_encoder_ = std::make_unique<VideoStreamEncoder>(
      clock, num_cpu_cores, &stats_proxy_, config_.encoder_settings,
      std::make_unique<OveruseFrameDetector>(&stats_proxy_), task_queue_factory,
      GetBitrateAllocationCallbackType(config_));

  // TODO(srte): Initialization should not be done posted on a task queue.
  // Note that the posted task must not outlive this scope since the closure
  // references local variables.
  worker_queue_->PostTask(ToQueuedTask(
      [this, clock, call_stats, transport, bitrate_allocator, send_delay_stats,
       event_log, &suspended_ssrcs, &encoder_config, &suspended_payload_states,
       &fec_controller]() {
        send_stream_.reset(new VideoSendStreamImpl(
            clock, &stats_proxy_, worker_queue_, call_stats, transport,
            bitrate_allocator, send_delay_stats, video_stream_encoder_.get(),
            event_log, &config_, encoder_config.max_bitrate_bps,
            encoder_config.bitrate_priority, suspended_ssrcs,
            suspended_payload_states, encoder_config.content_type,
            std::move(fec_controller)));
      },
      [this]() { thread_sync_event_.Set(); }));

  // Wait for ConstructionTask to complete so that |send_stream_| can be used.
  // |module_process_thread| must be registered and deregistered on the thread
  // it was created on.
  thread_sync_event_.Wait(rtc::Event::kForever);
  send_stream_->RegisterProcessThread(module_process_thread);
  ReconfigureVideoEncoder(std::move(encoder_config));
}
```



### VideoStreamEncoder

### VideoSendStreamImpl

---
layout: post
title: webrtc setLocalDescription-4
date: 2023-08-30 23:32:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

![]({{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/view.png)


## 15. !!! RtpSenderBase.SetSsrc

pc/rtp_sender.h

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
    // WebRtcVideoChannel 是在addsendstream创建的
    // videotrack 设置给WebRtcVideoChannel
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
   
1. 这是从`SdpOfferAnswerHandler.ApplyLocalDescription`
   
   ```c++
    transceiver->internal()->sender_internal()->SetSsrc(
               streams[0].first_ssrc());
   ```
   
   设置ssrc的值。



### 15.0 继续关系

```
ObserverInterface
RtpSenderInternal----抽象类
RtpSenderBase
AudioRtpSender  VideoRtpSender
```



### 15.1  VideoRtpSender.SetSend()

pc/rtp_sender.h

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

1. **把ssrc和track设置给WebRtcVideoChannel**
   
1. video_track,  就是在addvideotrack 一文中说的，VideoRtpSender.setTrack 的时候，设置的videoTrack
   
   ```c++
     rtc::scoped_refptr<VideoTrackInterface> video_track() const {
       return rtc::scoped_refptr<VideoTrackInterface>(
           static_cast<VideoTrackInterface*>(track_.get()));
     }
   ```

2. video_media_channel
   
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



### 15.2 !!! WebRtcVideoChannel.SetVideoSend

media/engine/webrtc_video_engine.cc

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
  // WebRtcVideoSendStream 是在上面章节创建并加入send_streams_中
  const auto& kv = send_streams_.find(ssrc);
  if (kv == send_streams_.end()) {
    // Allow unknown ssrc only if source is null.
    RTC_CHECK(source == nullptr);
    RTC_LOG(LS_ERROR) << "No sending stream on ssrc " << ssrc;
    return false;
  }
  ///// !!!!!!!
  ///WebRtcVideoSendStream 和VideoSource 绑定
  /////
  return kv->second->SetVideoSend(options, source);
}
```

1. send_streams_, 
   
   ```c++
     // Using primary-ssrc (first ssrc) as key.
     std::map<uint32_t, WebRtcVideoSendStream*> send_streams_
         RTC_GUARDED_BY(thread_checker_);
   ```

2. WebRtcVideoSendStream 和VideoSource 绑定

### 15.3 WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend

media/engine/webrtc_video_engine.cc



## 16. !!! WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend

media/engine/webrtc_video_engine.cc

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
     // webrtc::VideoSendStream stream_
     stream_->SetSource(nullptr, webrtc::DegradationPreference::DISABLED);
  }
  // 保存了videosource
  // Switch to the new source.
  source_ = source;
  // stream_ 是空的
  // channel 把 source 存了起来，但此时 channel 的 stream（`WebRtcVideoSendStream`）还没有创建，需要等 		`SetRemoteDescription` 才会创建；
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

1. stream_
   
   ```c++
   webrtc::VideoSendStream* stream_ RTC_GUARDED_BY(&thread_checker_);
   ```
   
   webrtc::VideoSendStream 是在WebRtcVideoSendStream.SetCodec 调用生成的【章节3】
   
   SetLocalSsrc
   
2. VideoSendStream和VideoSource，并保存了videosource，但是没有设置给VideoSendStream



### WebRtcVideoChannel.WebRtcVideoSendStream.SetCodec

media/engine/webrtc_video_engine.cc



### WebRtcVideoChannel.WebRtcVideoSendStream.ReconfigureEncoder

media/engine/webrtc_video_engine.cc



### webrtc::VideoSendStream::setSource



## 17. WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec

media/engine/webrtc_video_engine.cc

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

- 更新配置，parameters_
- RecreateWebRtcStream， 创建 webrtc::VideoSendStream



### WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream

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
  ///////////！！！！！////////
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());
  ///////////！！！！！////////

  parameters_.encoder_config.encoder_specific_settings = NULL;

  if (source_) {
    stream_->SetSource(source_, GetDegradationPreference());
  }

  // Call stream_->Start() if necessary conditions are met.
  UpdateSendState();
}
```



### Call::CreateVideoSendStream

call/call.cc

```cpp
webrtc::VideoSendStream* Call::CreateVideoSendStream(
    webrtc::VideoSendStream::Config config,
    VideoEncoderConfig encoder_config) {
  if (config_.fec_controller_factory) {
    RTC_LOG(LS_INFO) << "External FEC Controller will be used.";
  }
  std::unique_ptr<FecController> fec_controller =
      config_.fec_controller_factory
          ? config_.fec_controller_factory->CreateFecController()
          : std::make_unique<FecControllerDefault>(clock_);
  return CreateVideoSendStream(std::move(config), std::move(encoder_config),
                               std::move(fec_controller));
}

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

  VideoSendStream* send_stream = new VideoSendStream(
      clock_, num_cpu_cores_, module_process_thread_->process_thread(),
      task_queue_factory_, call_stats_->AsRtcpRttStats(), transport_send_ptr_,
      bitrate_allocator_.get(), video_send_delay_stats_.get(), event_log_,
      std::move(config), std::move(encoder_config), suspended_video_send_ssrcs_,
      suspended_video_payload_states_, std::move(fec_controller));

  for (uint32_t ssrc : ssrcs) {
    RTC_DCHECK(video_send_ssrcs_.find(ssrc) == video_send_ssrcs_.end());
    video_send_ssrcs_[ssrc] = send_stream;
  }
  video_send_streams_.insert(send_stream);
  // Forward resources that were previously added to the call to the new stream.
  for (const auto& resource_forwarder : adaptation_resource_forwarders_) {
    resource_forwarder->OnCreateVideoSendStream(send_stream);
  }

  UpdateAggregateNetworkState();

  return send_stream;
}
```



### webrtc::VideoSendStream





## 18. WebRtcVideoChannel::WebRtcVideoSendStream::ReconfigureEncoder

media/engine/webrtc_video_engine.cc

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



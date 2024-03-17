---
layout: post
title: webrtc::VideoEncoderConfig 说明和创建时机
date: 2024-03-07 20:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc stream video
---


* content
{:toc}

---


## 1. 前言

webrtc::VideoEncoderConfig 的属性 WebRtcVideoSendStream::VideoSendStreamParameters 。

```cpp
media/engine/webrtc_video_engine.h

 // Parameters needed to reconstruct the underlying stream.
    // webrtc::VideoSendStream doesn't support setting a lot of options on the
    // fly, so when those need to be changed we tear down and reconstruct with
    // similar parameters depending on which options changed etc.
    struct VideoSendStreamParameters {
      VideoSendStreamParameters(
          webrtc::VideoSendStream::Config config,
          const VideoOptions& options,
          int max_bitrate_bps,
          const absl::optional<VideoCodecSettings>& codec_settings);
      //!!!!!!!!!!!!!
      //!!!!!!!!!!!!!
      //!!!!!!!!!!!!!
      webrtc::VideoSendStream::Config config;
      VideoOptions options;
      int max_bitrate_bps;
      bool conference_mode;
      
      absl::optional<VideoCodecSettings> codec_settings;
      //!!!!!!!!!!!!!
      //!!!!!!!!!!!!!
      //!!!!!!!!!!!!!
      // Sent resolutions + bitrates etc. by the underlying VideoSendStream,
      // typically changes when setting a new resolution or reconfiguring
      // bitrates.
      webrtc::VideoEncoderConfig encoder_config;
    };
```



## 2. webrtc::VideoEncoderConfig

api/video_codecs/video_encoder_config.h

```cpp
// TODO(nisse): Consolidate on one of these.
  VideoCodecType codec_type;
  SdpVideoFormat video_format;

  rtc::scoped_refptr<VideoStreamFactoryInterface> video_stream_factory;
  std::vector<SpatialLayer> spatial_layers;
  ContentType content_type;
	// WebRtcVideoSendStream::RecreateWebRtcStream 赋值
  rtc::scoped_refptr<const EncoderSpecificSettings> encoder_specific_settings;

  // Padding will be used up to this bitrate regardless of the bitrate produced
  // by the encoder. Padding above what's actually produced by the encoder helps
  // maintaining a higher bitrate estimate. Padding will however not be sent
  // unless the estimated bandwidth indicates that the link can handle it.
  int min_transmit_bitrate_bps;
  int max_bitrate_bps;
  // The bitrate priority used for all VideoStreams.
  double bitrate_priority;

  // The simulcast layer's configurations set by the application for this video
  // sender. These are modified by the video_stream_factory before being passed
  // down to lower layers for the video encoding.
  // |simulcast_layers| is also used for configuring non-simulcast (when there
  // is a single VideoStream).
  std::vector<VideoStream> simulcast_layers;

	//!!!!!!!!!!!!!!
	//!!!!!!!!!!!!!!
	//!!!!!!!!!!!!!!
  // Max number of encoded VideoStreams to produce.
  size_t number_of_streams;

  // Legacy Google conference mode flag for simulcast screenshare
  bool legacy_conference_mode;
```





## 3. 创建赋值时机

### 3.1 WebRtcVideoSendStream::SetCodec

media/engine/webrtc_video_engine.cc

1. 创建 VideoEncoderConfig对象；

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec(
    const VideoCodecSettings& codec_settings) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  //!!!!!!!!!
  //!!!!!!!!!
  //!!!!!!!!!
  // WebRtcVideoSendStream::VideoSendStreamParameters parameters_
  // webrtc::VideoEncoderConfig encoder_config
  parameters_.encoder_config = CreateVideoEncoderConfig(codec_settings.codec);
  //!!!!!!!!!
  //!!!!!!!!!
  //!!!!!!!!!
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



#### 3.1.1 --WebRtcVideoSendStream::CreateVideoEncoderConfig

media/engine/webrtc_video_engine.cc

### 3.2 WebRtcVideoSendStream::ReconfigureEncoder

media/engine/webrtc_video_engine.cc

1. 对webrtc::VideoEncoderConfig的copy；

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::ReconfigureEncoder() {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  if (!stream_) {
    // The webrtc::VideoSendStream |stream_| has not yet been created but other
    // parameters has changed.
    return;
  }

  RTC_DCHECK_GT(parameters_.encoder_config.number_of_streams, 0);

  RTC_CHECK(parameters_.codec_settings);
  VideoCodecSettings codec_settings = *parameters_.codec_settings;

  webrtc::VideoEncoderConfig encoder_config =
      CreateVideoEncoderConfig(codec_settings.codec);

  encoder_config.encoder_specific_settings =
      ConfigureVideoEncoderSettings(codec_settings.codec);

  stream_->ReconfigureVideoEncoder(encoder_config.Copy());

  encoder_config.encoder_specific_settings = NULL;

  // 重新赋值webrtc::VideoEncoderConfig
  parameters_.encoder_config = std::move(encoder_config);
}
```



### 3.3 WebRtcVideoSendStream::RecreateWebRtcStream

1. 对`rtc::scoped_refptr<const EncoderSpecificSettings> encoder_specific_settings` 属性的赋值。

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
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  parameters_.encoder_config.encoder_specific_settings =
      ConfigureVideoEncoderSettings(parameters_.codec_settings->codec);
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!

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
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  parameters_.encoder_config.encoder_specific_settings = NULL;
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!

  if (source_) {
    stream_->SetSource(source_, GetDegradationPreference());
  }

  // Call stream_->Start() if necessary conditions are met.
  UpdateSendState();
}
```



#### 3.3.1 --WebRtcVideoSendStream::ConfigureVideoEncoderSettings

media/engine/webrtc_video_engine.cc



## 4. !!! WebRtcVideoSendStream::CreateVideoEncoderConfig

media/engine/webrtc_video_engine.cc

```cpp
webrtc::VideoEncoderConfig
WebRtcVideoChannel::WebRtcVideoSendStream::CreateVideoEncoderConfig(
    const VideoCodec& codec) const {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  webrtc::VideoEncoderConfig encoder_config;
  // 1. codec type, h264, vp8 等
  encoder_config.codec_type = webrtc::PayloadStringToCodecType(codec.name);
  // 2. codec name 和 codec param组成
  encoder_config.video_format =
      webrtc::SdpVideoFormat(codec.name, codec.params);

  // 3. WebRtcVideoSendStream::VideoSendStreamParameters parameters_
	// VideoOptions options 
  // 屏幕共享参数的特化处理
  bool is_screencast = parameters_.options.is_screencast.value_or(false);
  if (is_screencast) {
    //？？？？？？？？
    encoder_config.min_transmit_bitrate_bps =
        1000 * parameters_.options.screencast_min_bitrate_kbps.value_or(0);
    encoder_config.content_type =
        webrtc::VideoEncoderConfig::ContentType::kScreen;
  } else {
    encoder_config.min_transmit_bitrate_bps = 0;
    encoder_config.content_type =
        webrtc::VideoEncoderConfig::ContentType::kRealtimeVideo;
  }

  // 4. number_of_streams， 是RtpConfig ssrcs的个数，就是simulcast的层数
  // By default, the stream count for the codec configuration should match the
  // number of negotiated ssrcs. But if the codec is disabled for simulcast
  // or a screencast (and not in simulcast screenshare experiment), only
  // configure a single stream.
  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
  encoder_config.number_of_streams = parameters_.config.rtp.ssrcs.size();
  // codec不支持simulcast，则为 number_of_streams = 1
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
  // 5. 最大码流设置
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

  // 6. std::vector<VideoStream> simulcast_layers; 赋值
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
	// 7. 
  encoder_config.legacy_conference_mode = parameters_.conference_mode;

  int max_qp = kDefaultQpMax;
  codec.GetParam(kCodecParamMaxQuantization, &max_qp);
  // 8. 创建 EncoderStreamFactory
  encoder_config.video_stream_factory =
      new rtc::RefCountedObject<EncoderStreamFactory>(
          codec.name, max_qp, is_screencast, parameters_.conference_mode);
  return encoder_config;
}
```



### PayloadStringToCodecType

api/video_codecs/video_codec.cc

```cpp
VideoCodecType PayloadStringToCodecType(const std::string& name) {
  if (absl::EqualsIgnoreCase(name, kPayloadNameVp8))
    return kVideoCodecVP8;
  if (absl::EqualsIgnoreCase(name, kPayloadNameVp9))
    return kVideoCodecVP9;
  if (absl::EqualsIgnoreCase(name, kPayloadNameAv1))
    return kVideoCodecAV1;
  if (absl::EqualsIgnoreCase(name, kPayloadNameH264))
    return kVideoCodecH264;
  if (absl::EqualsIgnoreCase(name, kPayloadNameMultiplex))
    return kVideoCodecMultiplex;
  return kVideoCodecGeneric;
}
```



### 4.1 IsCodecDisabledForSimulcast

```cpp
// Returns true if the given codec is disallowed from doing simulcast.
bool IsCodecDisabledForSimulcast(const std::string& codec_name,
                                 const webrtc::WebRtcKeyValueConfig& trials) {
  return !absl::StartsWith(trials.Lookup("WebRTC-H264Simulcast"), "Disabled")
             ? absl::EqualsIgnoreCase(codec_name, kVp9CodecName)
             : absl::EqualsIgnoreCase(codec_name, kH264CodecName) ||
                   absl::EqualsIgnoreCase(codec_name, kVp9CodecName);
}
```



### webrtc::VideoStream

api/video_codecs/video_encoder_config.h

```cpp
// The |VideoStream| struct describes a simulcast layer, or "stream".
struct VideoStream {
  VideoStream();
  ~VideoStream();
  VideoStream(const VideoStream& other);
  std::string ToString() const;

  // Width in pixels.
  size_t width;

  // Height in pixels.
  size_t height;

  // Frame rate in fps.
  int max_framerate;

  // Bitrate, in bps, for the stream.
  int min_bitrate_bps;
  int target_bitrate_bps;
  int max_bitrate_bps;

  // Scaling factor applied to the stream size.
  // |width| and |height| values are already scaled down.
  double scale_resolution_down_by;

  // Maximum Quantization Parameter to use when encoding the stream.
  int max_qp;

  // Determines the number of temporal layers that the stream should be
  // encoded with. This value should be greater than zero.
  // TODO(brandtr): This class is used both for configuring the encoder
  // (meaning that this field _must_ be set), and for signaling the app-level
  // encoder settings (meaning that the field _may_ be set). We should separate
  // this and remove this optional instead.
  absl::optional<size_t> num_temporal_layers;

  // The priority of this stream, to be used when allocating resources
  // between multiple streams.
  absl::optional<double> bitrate_priority;

  absl::optional<std::string> scalability_mode;

  // If this stream is enabled by the user, or not.
  bool active;
};
```



### ？？？ EncoderStreamFactory

media/engine/webrtc_video_engine.h



## 5. !!! ??? WebRtcVideoSendStream::ConfigureVideoEncoderSettings

media/engine/webrtc_video_engine.cc

```cpp
rtc::scoped_refptr<webrtc::VideoEncoderConfig::EncoderSpecificSettings>
WebRtcVideoChannel::WebRtcVideoSendStream::ConfigureVideoEncoderSettings(
    const VideoCodec& codec) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  bool is_screencast = parameters_.options.is_screencast.value_or(false);
  // No automatic resizing when using simulcast or screencast, or when
  // disabled by field trial flag.
  bool automatic_resize = !disable_automatic_resize_ && !is_screencast &&
                          (parameters_.config.rtp.ssrcs.size() == 1 ||
                           NumActiveStreams(rtp_parameters_) == 1);

  bool frame_dropping = !is_screencast;
  bool denoising;
  bool codec_default_denoising = false;
  if (is_screencast) {
    denoising = false;
  } else {
    // Use codec default if video_noise_reduction is unset.
    codec_default_denoising = !parameters_.options.video_noise_reduction;
    denoising = parameters_.options.video_noise_reduction.value_or(false);
  }

  if (absl::EqualsIgnoreCase(codec.name, kH264CodecName)) {
    webrtc::VideoCodecH264 h264_settings =
        webrtc::VideoEncoder::GetDefaultH264Settings();
    h264_settings.frameDroppingOn = frame_dropping;
    return new rtc::RefCountedObject<
        webrtc::VideoEncoderConfig::H264EncoderSpecificSettings>(h264_settings);
  }
  if (absl::EqualsIgnoreCase(codec.name, kVp8CodecName)) {
    webrtc::VideoCodecVP8 vp8_settings =
        webrtc::VideoEncoder::GetDefaultVp8Settings();
    vp8_settings.automaticResizeOn = automatic_resize;
    // VP8 denoising is enabled by default.
    vp8_settings.denoisingOn = codec_default_denoising ? true : denoising;
    vp8_settings.frameDroppingOn = frame_dropping;
    return new rtc::RefCountedObject<
        webrtc::VideoEncoderConfig::Vp8EncoderSpecificSettings>(vp8_settings);
  }
  if (absl::EqualsIgnoreCase(codec.name, kVp9CodecName)) {
    webrtc::VideoCodecVP9 vp9_settings =
        webrtc::VideoEncoder::GetDefaultVp9Settings();
    const size_t default_num_spatial_layers =
        parameters_.config.rtp.ssrcs.size();
    const size_t num_spatial_layers =
        GetVp9SpatialLayersFromFieldTrial(call_->trials())
            .value_or(default_num_spatial_layers);

    const size_t default_num_temporal_layers =
        num_spatial_layers > 1 ? kConferenceDefaultNumTemporalLayers : 1;
    const size_t num_temporal_layers =
        GetVp9TemporalLayersFromFieldTrial(call_->trials())
            .value_or(default_num_temporal_layers);

    vp9_settings.numberOfSpatialLayers = std::min<unsigned char>(
        num_spatial_layers, kConferenceMaxNumSpatialLayers);
    vp9_settings.numberOfTemporalLayers = std::min<unsigned char>(
        num_temporal_layers, kConferenceMaxNumTemporalLayers);

    // VP9 denoising is disabled by default.
    vp9_settings.denoisingOn = codec_default_denoising ? true : denoising;
    vp9_settings.automaticResizeOn = automatic_resize;
    // Ensure frame dropping is always enabled.
    RTC_DCHECK(vp9_settings.frameDroppingOn);
    if (!is_screencast) {
      webrtc::FieldTrialFlag interlayer_pred_experiment_enabled =
          webrtc::FieldTrialFlag("Enabled");
      webrtc::FieldTrialEnum<webrtc::InterLayerPredMode> inter_layer_pred_mode(
          "inter_layer_pred_mode", webrtc::InterLayerPredMode::kOnKeyPic,
          ...);
      webrtc::ParseFieldTrial(
          {&interlayer_pred_experiment_enabled, &inter_layer_pred_mode},
          call_->trials().Lookup("WebRTC-Vp9InterLayerPred"));
      if (interlayer_pred_experiment_enabled) {
        vp9_settings.interLayerPred = inter_layer_pred_mode;
      } else {
        // Limit inter-layer prediction to key pictures by default.
        vp9_settings.interLayerPred = webrtc::InterLayerPredMode::kOnKeyPic;
      }
    } else {
      // Multiple spatial layers vp9 screenshare needs flexible mode.
      vp9_settings.flexibleMode = vp9_settings.numberOfSpatialLayers > 1;
      vp9_settings.interLayerPred = webrtc::InterLayerPredMode::kOn;
    }
    return new rtc::RefCountedObject<
        webrtc::VideoEncoderConfig::Vp9EncoderSpecificSettings>(vp9_settings);
  }
  return nullptr;
}
```





## 6. webrtc::VideoEncoderConfig 使用？？？
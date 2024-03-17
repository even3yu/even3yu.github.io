---
layout: post
title: webrtc::VideoSendStream::Config 说明和创建时机
date: 2024-03-07 19:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc stream video
---


* content
{:toc}

---


![videoo_send_stream_config]({{ site.url }}{{ site.baseurl }}/images/videosendstream-config.assets/video_send_stream_config.png)

## 1. webrtc::VideoSendStream::Config

call/video_send_stream.h

```cpp
struct Config {
  public:
  explicit Config(Transport* send_transport);

  RtpConfig rtp;

  // WebRtcVideoChannel::AddSendStream赋值
  VideoStreamEncoderSettings encoder_settings;

  // WebRtcVideoChannel::AddSendStream赋值
  // Time interval between RTCP report for video
  int rtcp_report_interval_ms = 1000;

  // WebRtcVideoChannel::AddSendStream，Config(Transport* send_transport) 赋值
  // Transport for outgoing packets.
  Transport* send_transport = nullptr;

  // Expected delay needed by the renderer, i.e. the frame will be delivered
  // this many milliseconds, if possible, earlier than expected render time.
  // Only valid if |local_renderer| is set.
  int render_delay_ms = 0;

  // Target delay in milliseconds. A positive value indicates this stream is
  // used for streaming instead of a real-time call.
  int target_delay_ms = 0;

  // WebRtcVideoChannel::AddSendStream赋值
  // True if the stream should be suspended when the available bitrate fall
  // below the minimum configured bitrate. If this variable is false, the
  // stream may send at a rate higher than the estimated available bitrate.
  bool suspend_below_min_bitrate = false;

  // WebRtcVideoChannel::AddSendStream 赋值
  // Enables periodic bandwidth probing in application-limited region.
  bool periodic_alr_bandwidth_probing = false;

  // WebRtcVideoChannel::WebRtcVideoSendStream::SetFrameEncryptor 赋值
  // An optional custom frame encryptor that allows the entire frame to be
  // encrypted in whatever way the caller chooses. This is not required by
  // default.
  rtc::scoped_refptr<webrtc::FrameEncryptorInterface> frame_encryptor;

  // Per PeerConnection cryptography options.
  // 在WebRtcVideoChannel::AddSendStream 赋值
  CryptoOptions crypto_options;

  // WebRtcVideoSendStream::SetEncoderToPacketizerFrameTransformer 赋值
  rtc::scoped_refptr<webrtc::FrameTransformerInterface> frame_transformer;
};
```

 

### 1.0 VideoSendStream::Config::Config

```cpp
 VideoSendStream::Config::Config(Transport* send_transport)
      : rtp(),
        encoder_settings(VideoEncoder::Capabilities(rtp.lntf.enabled)),
        send_transport(send_transport) {}
```



### 1.1 --RtpConfig

call/rtp_config.h

### 1.2 -- VideoStreamEncoderSettings

api/video/video_stream_encoder_settings.h

### 1.3 --webrtc::Transport



### 1.4 -- webrtc::FrameEncryptorInterface



### 1.5 -- CryptoOptions



### 1.6 -- webrtc::FrameTransformerInterface

## 2. RtpConfig

call/rtp_config.h

```cpp
static const size_t kDefaultMaxPacketSize = 1500 - 40;  // TCP over IPv4.
struct RtpConfig {
 ...

  // WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
	// 包括simulcast ssrc
  std::vector<uint32_t> ssrcs;

  // WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
  // The Rtp Stream Ids (aka RIDs) to send in the RID RTP header extension
  // if the extension is included in the list of extensions.
  // If rids are specified, they should correspond to the |ssrcs| vector.
  // This means that:
  // 1. rids.size() == 0 || rids.size() == ssrcs.size().
  // 2. If rids is not empty, then |rids[i]| should use |ssrcs[i]|.
  std::vector<std::string> rids;

  // WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
  // WebRtcVideoSendStream::SetSendParameters 赋值
  // The value to send in the MID RTP header extension if the extension is
  // included in the list of extensions.
  std::string mid;

  // WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
  // WebRtcVideoSendStream::SetSendParameters 赋值
  // See RtcpMode for description.
  RtcpMode rtcp_mode = RtcpMode::kCompound;

  // WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
  // Max RTP packet size delivered to send transport from VideoEngine.
  size_t max_packet_size = kDefaultMaxPacketSize;
  
  // WebRtcVideoChannel::AddSendStream赋值
  // WebRtcVideoSendStream::SetSendParameters 赋值
  // Corresponds to the SDP attribute extmap-allow-mixed.
  bool extmap_allow_mixed = false;

  // WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
  // WebRtcVideoSendStream::SetSendParameters 赋值
  // RTP header extensions to use for this send stream.
  std::vector<RtpExtension> extensions;

  // WebRtcVideoSendStream::SetCodec 赋值
  // TODO(nisse): For now, these are fixed, but we'd like to support
  // changing codec without recreating the VideoSendStream. Then these
  // fields must be removed, and association between payload type and codec
  // must move above the per-stream level. Ownership could be with
  // RtpTransportControllerSend, with a reference from PayloadRouter, where
  // the latter would be responsible for mapping the codec type of encoded
  // images to the right payload type.
  std::string payload_name;
  // WebRtcVideoSendStream::SetCodec 赋值
  int payload_type = -1;
  // WebRtcVideoSendStream::SetCodec 赋值
  // Payload should be packetized using raw packetizer (payload header will
  // not be added, additional meta data is expected to be present in generic
  // frame descriptor RTP header extension).
  bool raw_payload = false;

  // WebRtcVideoSendStream::SetCodec 赋值
  // See LntfConfig for description.
  LntfConfig lntf;

  // WebRtcVideoSendStream::SetCodec 赋值
  // See NackConfig for description.
  NackConfig nack;

  // WebRtcVideoSendStream::SetCodec 赋值
  // See UlpfecConfig for description.
  UlpfecConfig ulpfec;

  struct Flexfec {
    Flexfec();
    Flexfec(const Flexfec&);
    ~Flexfec();
    // WebRtcVideoSendStream::SetCodec 赋值
    // Payload type of FlexFEC. Set to -1 to disable sending FlexFEC.
    int payload_type = -1;

  	// WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
    // SSRC of FlexFEC stream.
    uint32_t ssrc = 0;

  	// WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
    // Vector containing a single element, corresponding to the SSRC of the
    // media stream being protected by this FlexFEC stream.
    // The vector MUST have size 1.
    //
    // TODO(brandtr): Update comment above when we support
    // multistream protection.
    std::vector<uint32_t> protected_media_ssrcs;
  } flexfec;

  // Settings for RTP retransmission payload format, see RFC 4588 for
  // details.
  struct Rtx {
    Rtx();
    Rtx(const Rtx&);
    ~Rtx();
    std::string ToString() const;
  	// WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
    // SSRCs to use for the RTX streams.
    std::vector<uint32_t> ssrcs;
    
		// WebRtcVideoSendStream::SetCodec 赋值
    // Payload type to use for the RTX stream.
    int payload_type = -1;
  } rtx;

  // WebRtcVideoSendStream::WebRtcVideoSendStream 赋值
  // RTCP CNAME, see RFC 3550.
  std::string c_name;

  bool IsMediaSsrc(uint32_t ssrc) const;
  bool IsRtxSsrc(uint32_t ssrc) const;
  bool IsFlexfecSsrc(uint32_t ssrc) const;
  absl::optional<uint32_t> GetRtxSsrcAssociatedWithMediaSsrc(
      uint32_t media_ssrc) const;
  uint32_t GetMediaSsrcAssociatedWithRtxSsrc(uint32_t rtx_ssrc) const;
  uint32_t GetMediaSsrcAssociatedWithFlexfecSsrc(uint32_t flexfec_ssrc) const;
  absl::optional<std::string> GetRidForSsrc(uint32_t ssrc) const;
};
```



### 2.1 LntfConfig

call/rtp_config.h

```cpp
// Settings for LNTF (LossNotification). Still highly experimental.
struct LntfConfig {
  std::string ToString() const;

  bool enabled{false};
};
```



### 2.2 NackConfig

call/rtp_config.h

```cpp
// Settings for NACK, see RFC 4585 for details.
struct NackConfig {
  NackConfig() : rtp_history_ms(0) {}
  std::string ToString() const;
  // WebRtcVideoSendStream::SetCodec 赋值
  // Send side: the time RTP packets are stored for retransmissions.
  // Receive side: the time the receiver is prepared to wait for
  // retransmissions.
  // Set to '0' to disable.
  int rtp_history_ms;
};
```



### 2.3 UlpfecConfig

call/rtp_config.h

```cpp
// Settings for ULPFEC forward error correction.
// Set the payload types to '-1' to disable.
struct UlpfecConfig {
  UlpfecConfig()
      : ulpfec_payload_type(-1),
        red_payload_type(-1),
        red_rtx_payload_type(-1) {}
  std::string ToString() const;
  bool operator==(const UlpfecConfig& other) const;

  // Payload type used for ULPFEC packets.
  int ulpfec_payload_type;

  // Payload type used for RED packets.
  int red_payload_type;

  // RTX payload type for RED payload.
  int red_rtx_payload_type;
};
```



### 2.4 RtpConfig::Flexfec

call/rtp_config.h



### 2.5 RtpConfig::Rtx

call/rtp_config.h



## 3. VideoStreamEncoderSettings

api/video/video_stream_encoder_settings.h

```cpp
struct VideoStreamEncoderSettings {
  explicit VideoStreamEncoderSettings(
      const VideoEncoder::Capabilities& capabilities)
      : capabilities(capabilities) {}

  // WebRtcVideoChannel::AddSendStream 赋值
  // Enables the new method to estimate the cpu load from encoding, used for
  // cpu adaptation.
  bool experiment_cpu_load_estimator = false;

  // WebRtcVideoChannel::AddSendStream 赋值
  // Ownership stays with WebrtcVideoEngine (delegated from PeerConnection).
  VideoEncoderFactory* encoder_factory = nullptr;

  // WebRtcVideoChannel::AddSendStream 赋值
  // Requests the WebRtcVideoChannel to perform a codec switch.
  EncoderSwitchRequestCallback* encoder_switch_request_callback = nullptr;

  // WebRtcVideoChannel::AddSendStream 赋值
  // Ownership stays with WebrtcVideoEngine (delegated from PeerConnection).
  VideoBitrateAllocatorFactory* bitrate_allocator_factory = nullptr;

  // WebRtcVideoChannel::AddSendStream，Config(Transport* send_transport) 赋值
  // Negotiated capabilities which the VideoEncoder may expect the other
  // side to use.
  VideoEncoder::Capabilities capabilities;
};
```



## 4. ??? webrtc::Transport





## 5. ??? webrtc::FrameEncryptorInterface



## 6. ??? CryptoOptions



## 7. ??? webrtc::FrameTransformerInterface



## 8. !!! VideoSendStream::Config 创建时机和赋值时机

```
WebRtcVideoChannel::AddSendStream
	WebRtcVideoChannel::WebRtcVideoSendStream::WebRtcVideoSendStream
		WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream()
```



### 8.0 !!! VideoSendStream::Config修改时机

| 修改时机                                                     | 对config的属性的修改                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| WebRtcVideoChannel::AddSendStream                            | 创建VideoSendStream::Config，部分属性赋值                    |
| WebRtcVideoSendStream::WebRtcVideoSendStream                 | 对VideoSendStream::Config，主要对ssrc进行了赋值，包括simulcast，rtx，flex-fec |
| WebRtcVideoSendStream::SetCodec                              | 对VideoSendStream::Config ，和codec 相关的属性赋值           |
| WebRtcVideoSendStream::SetSendParameters                     | 动态修改VideoSendStream::Config 的部分属性                   |
| WebRtcVideoSendStream::SetFrameEncryptor                     | 支持端到端加密，对frame_encryptor的修改                      |
| WebRtcVideoSendStream::SetEncoderToPacketizerFrameTransformer | 操作编码后的数据以及解码前数据，对frame_transformer修改      |



### 8.1 !!! WebRtcVideoChannel::AddSendStream

media/engine/webrtc_video_engine.cc

1. 构建webrtc::VideoSendStream::Config， 传入参数webrtc::Transport， 这里的Transport 就是WebRtcVideoChannel，

   ```cpp
   media/engine/webrtc_video_engine.h
   class WebRtcVideoChannel : public VideoMediaChannel,
                              public webrtc::Transport,
                              public webrtc::EncoderSwitchRequestCallback {
                              ...
                              }
   
2. 对webrtc::VideoSendStream::Config 的一些属性，以及webrtc::VideoSendStream::Config::encoder_settings 进行赋值；

2. 创建WebRtcVideoSendStream对象；

2. 存放WebRtcVideoSendStream 到`std::map<uint32_t, WebRtcVideoSendStream*> send_streams_`;

5. `stream->SetSend

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

  //!!!!!!!!!!!!!
  //!!!!!!!!!!!!!
  //!!!!!!!!!!!!!
  // 1. 创建 webrtc::VideoSendStream::Config，
	// 注意，this，就是webrtc::Tranpsort
  webrtc::VideoSendStream::Config config(this);

  // 2. webrtc::VideoSendStream::Config 赋值
  for (const RidDescription& rid : sp.rids()) {
    config.rtp.rids.push_back(rid.rid);
  }

  config.suspend_below_min_bitrate = video_config_.suspend_below_min_bitrate;
  config.periodic_alr_bandwidth_probing =
      video_config_.periodic_alr_bandwidth_probing;
  // cpu的负载
  config.encoder_settings.experiment_cpu_load_estimator =
      video_config_.experiment_cpu_load_estimator;
  // 编码器 std::unique_ptr<webrtc::VideoEncoderFactory>
  // PeerConnectFactory里创建
  config.encoder_settings.encoder_factory = encoder_factory_;
  
  // 码流分配 VideoBitrateAllocatorFactory* bitrate_allocator_factory
  // PeerConnectFactory里创建，如果没有创建，就用默认的CreateBuiltinVideoBitrateAllocatorFactory
	// SdpOfferAnswerHandler::Initialize的是负值，BuiltinVideoBitrateAllocatorFactory
  config.encoder_settings.bitrate_allocator_factory =
      bitrate_allocator_factory_;
  // 编码器切换 回调
  config.encoder_settings.encoder_switch_request_callback = this;
  // 加密策略
  config.crypto_options = crypto_options_;
  // rtpheader extension 是否支持两字节
  config.rtp.extmap_allow_mixed = ExtmapAllowMixed();
  // rtcp report的时间间隔
  config.rtcp_report_interval_ms = video_config_.rtcp_report_interval_ms;
  //!!!!!!!!!!!!!
  //!!!!!!!!!!!!!
  //!!!!!!!!!!!!!

  // 3. 创建WebRtcVideoSendStream
  WebRtcVideoSendStream* stream = new WebRtcVideoSendStream(
      call_, sp, std::move(config), default_send_options_,
      video_config_.enable_cpu_adaptation, bitrate_config_.max_bitrate_bps,
    	// const absl::optional<VideoCodecSettings>& send_codec_
      send_codec_, 
    	send_rtp_extensions_, send_params_);

  // 4. 主流的ssrc
  uint32_t ssrc = sp.first_ssrc();
  RTC_DCHECK(ssrc != 0);
  // std::map<uint32_t, WebRtcVideoSendStream*> send_streams_
  send_streams_[ssrc] = stream;

  if (rtcp_receiver_report_ssrc_ == kDefaultRtcpReceiverReportSsrc) {
    rtcp_receiver_report_ssrc_ = ssrc;
    RTC_LOG(LS_INFO)
        << "SetLocalSsrc on all the receive streams because we added "
           "a send stream.";
    for (auto& kv : receive_streams_)
      kv.second->SetLocalSsrc(ssrc);
  }
  // 5. 发送
  if (sending_) {
    stream->SetSend(true);
  }

  return true;
}
```



#### --WebRtcVideoSendStream::WebRtcVideoSendStream



### 8.2 WebRtcVideoSendStream::WebRtcVideoSendStream

media/engine/webrtc_video_engine.cc

1. rtp 包大小赋值
2. 从StreamParams 得到ssrc， 获取每一层的ssrc，就是simulcast 每一层的ssrc，RtpConfig的ssrc中；
3. 从StreamParams 得到ssrc， 是重传流rtx的ssrc，存放在 RtpConfig::Rtx的ssrcs中；
4. 从StreamParams 得到ssrc， 是flex fec的ssrc，存放在 RtpConfig::FlexFec的ssrc中；
5. rtp.rtcp_mode ？？？ 做什么用
6. rtp.mid ???
7. 注意构造传入的参数，`const absl::optional<VideoCodecSettings>& codec_settings,` 赋值时机
   SetCodec(*codec_settings) 是否调用？？？？

```cpp
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
  
  // 1. rtp 包大小赋值
  // Maximum packet size may come in RtpConfig from external transport, for
  // example from QuicTransportInterface implementation, so do not exceed
  // given max_packet_size.
  parameters_.config.rtp.max_packet_size =
      std::min<size_t>(parameters_.config.rtp.max_packet_size, kVideoMtu);
  parameters_.conference_mode = send_params.conference_mode;

  // 2. 从StreamParams 得到ssrc， 获取每一层的ssrc，就是simulcast 每一层的ssrc
  sp.GetPrimarySsrcs(&parameters_.config.rtp.ssrcs);

  // ValidateStreamParams should prevent this from happening.
  RTC_CHECK(!parameters_.config.rtp.ssrcs.empty());
  rtp_parameters_.encodings[0].ssrc = parameters_.config.rtp.ssrcs[0];

  // 3. RTX. RtpConfig::Rtx::ssrcs 赋值
  sp.GetFidSsrcs(parameters_.config.rtp.ssrcs,
                 &parameters_.config.rtp.rtx.ssrcs);

  // 4. FlexFEC SSRCs.
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
        // RtpConfig::FlexFec::ssrcs
        parameters_.config.rtp.flexfec.ssrc = flexfec_ssrc;
        parameters_.config.rtp.flexfec.protected_media_ssrcs = {primary_ssrc};
      }
    }
  }

  // 5. cname
  parameters_.config.rtp.c_name = sp.cname;
  // 6. rtp 扩展头
  if (rtp_extensions) {
    parameters_.config.rtp.extensions = *rtp_extensions;
    rtp_parameters_.header_extensions = *rtp_extensions;
  }
  // 7. rtcpmode？？？
  parameters_.config.rtp.rtcp_mode = send_params.rtcp.reduced_size
                                         ? webrtc::RtcpMode::kReducedSize
                                         : webrtc::RtcpMode::kCompound;
  // rtp.mid ???
  parameters_.config.rtp.mid = send_params.mid;
  rtp_parameters_.rtcp.reduced_size = send_params.rtcp.reduced_size;

  // ？？？？？？？
  if (codec_settings) {
    SetCodec(*codec_settings);
  }
}
```



#### 8.2.1 StreamParams::GetPrimarySsrcs

media/base/stream_params.cc

```cpp
void StreamParams::GetPrimarySsrcs(std::vector<uint32_t>* ssrcs) const {
  // "SIM"
  // 找打simulcast ssrcgroup
  const SsrcGroup* sim_group = get_ssrc_group(kSimSsrcGroupSemantics);
  // 没有开启simulcast
  if (sim_group == NULL) {
    // first_ssrc() 就是主流的ssrc
    ssrcs->push_back(first_ssrc());
  } else {
    // simulcast 每一层的ssrc
    ssrcs->insert(ssrcs->end(), sim_group->ssrcs.begin(),
                  sim_group->ssrcs.end());
  }
}
```



```cpp
const SsrcGroup* get_ssrc_group(const std::string& semantics) const {
    for (const SsrcGroup& ssrc_group : ssrc_groups) {
      if (ssrc_group.has_semantics(semantics)) {
        return &ssrc_group;
      }
    }
    return NULL;
  }
```



```cpp
struct SsrcGroup {
...
 	bool has_semantics(const std::string& semantics) const;

  std::string ToString() const;

  std::string semantics;        // e.g FIX, FEC, SIM.
  std::vector<uint32_t> ssrcs;  // SSRCs of this type.
}
```



#### 8.2.2 StreamParams::GetFidSsrcs

media/base/stream_params.cc

```cpp
void StreamParams::GetFidSsrcs(const std::vector<uint32_t>& primary_ssrcs,
                               std::vector<uint32_t>* fid_ssrcs) const {
  for (uint32_t primary_ssrc : primary_ssrcs) {
    uint32_t fid_ssrc;
    if (GetFidSsrc(primary_ssrc, &fid_ssrc)) {
      fid_ssrcs->push_back(fid_ssrc);
    }
  }
}
```



```cpp
  bool GetFidSsrc(uint32_t primary_ssrc, uint32_t* fid_ssrc) const {
    // "FID"
    return GetSecondarySsrc(kFidSsrcGroupSemantics, primary_ssrc, fid_ssrc);
  }
```



```cpp
bool StreamParams::GetSecondarySsrc(const std::string& semantics,
                                    uint32_t primary_ssrc,
                                    uint32_t* secondary_ssrc) const {
  for (const SsrcGroup& ssrc_group : ssrc_groups) {
    if (ssrc_group.has_semantics(semantics) && ssrc_group.ssrcs.size() >= 2 &&
        ssrc_group.ssrcs[0] == primary_ssrc) {
      *secondary_ssrc = ssrc_group.ssrcs[1];
      return true;
    }
  }
  return false;
}
```



#### 8.2.3 --WebRtcVideoSendStream::SetCodec

media/engine/webrtc_video_engine.cc



### 8.3 WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec

media/engine/webrtc_video_engine.cc

1. 创建webrtc::VideoEncoderConfig
2. RtpConfig codec相关属性的赋值；包括payload_name， payload_type，raw_payload，以及 ulpfec，flexfec赋值
3. RtpConfig.Rtx  codec相关属性的赋值, 包括payload_type
4. LlntfConfig 的属性赋值；
5. NackConfig 的属性赋值；
6. 重新创建 webrtc::internal::VideoSendStream对象

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec(
    const VideoCodecSettings& codec_settings) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  // 1. 创建 webrtc::VideoEncoderConfig
  parameters_.encoder_config = CreateVideoEncoderConfig(codec_settings.codec);
  RTC_DCHECK_GT(parameters_.encoder_config.number_of_streams, 0);

  // 2. rtp codec 相关属性赋值
  parameters_.config.rtp.payload_name = codec_settings.codec.name;
  parameters_.config.rtp.payload_type = codec_settings.codec.id;
  parameters_.config.rtp.raw_payload =
      codec_settings.codec.packetization == kPacketizationParamRaw;
  parameters_.config.rtp.ulpfec = codec_settings.ulpfec;
  parameters_.config.rtp.flexfec.payload_type =
      codec_settings.flexfec_payload_type;

  // 3. rtx 开启
  // Set RTX payload type if RTX is enabled.
  if (!parameters_.config.rtp.rtx.ssrcs.empty()) {
    if (codec_settings.rtx_payload_type == -1) {
      RTC_LOG(LS_WARNING)
          << "RTX SSRCs configured but there's no configured RTX "
             "payload type. Ignoring.";
      parameters_.config.rtp.rtx.ssrcs.clear();
    } else {
      // 赋值 rtx.payload_type
      parameters_.config.rtp.rtx.payload_type = codec_settings.rtx_payload_type;
    }
  }

  // 4. Llntf
  const bool has_lntf = HasLntf(codec_settings.codec);
  parameters_.config.rtp.lntf.enabled = has_lntf;
  parameters_.config.encoder_settings.capabilities.loss_notification = has_lntf;

  // 5. nack
  parameters_.config.rtp.nack.rtp_history_ms =
      HasNack(codec_settings.codec) ? kNackHistoryMs : 0;

  parameters_.codec_settings = codec_settings;

  // TODO(nisse): Avoid recreation, it should be enough to call
  // ReconfigureEncoder.
  RTC_LOG(LS_INFO) << "RecreateWebRtcStream (send) because of SetCodec.";
  // 重新创建 webrtc::internal::VideoSendStream对象
  RecreateWebRtcStream();
}
```



#### HasLntf

media/base/codec.h

```cpp
bool HasLntf(const Codec& codec) {
  return codec.HasFeedbackParam(
    	// const char kRtcpFbParamLntf[] = "goog-lntf";
      FeedbackParam(kRtcpFbParamLntf, kParamValueEmpty));
}
```



#### HasNack

media/base/codec.h

```cpp
bool HasNack(const Codec& codec) {
  return codec.HasFeedbackParam(
    	// const char kRtcpFbParamNack[] = "nack";
    	//FeedbackParam 就是k-v
      FeedbackParam(kRtcpFbParamNack, kParamValueEmpty));
}
```



#### -- WebRtcVideoSendStream::RecreateWebRtcStream()



### 8.4 WebRtcVideoSendStream::SetSendParameters

media/engine/webrtc_video_engine.cc

1. 修改webrtc::VideoSendStream::Config 的部分参数，需要重新创建webrtc::internal::VideoSendStream;

```cpp
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
    SetCodec(*params.send_codec);
    recreate_stream = false;  // SetCodec has already recreated the stream.
  } else if (params.conference_mode && parameters_.codec_settings) {
    SetCodec(*parameters_.codec_settings);
    recreate_stream = false;  // SetCodec has already recreated the stream.
  }
  if (recreate_stream) {
    RTC_LOG(LS_INFO)
        << "RecreateWebRtcStream (send) because of SetSendParameters";
    RecreateWebRtcStream();
  }
}
```



### 8.5 WebRtcVideoSendStream::SetFrameEncryptor

media/engine/webrtc_video_engine.cc

1. webrtc添加端到端加密函数，https://blog.csdn.net/danieljinbiao/article/details/103668703
2. webrtc::FrameEncryptorInterface，api/crypto/frame_encryptor_interface.h

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::SetFrameEncryptor(
    rtc::scoped_refptr<webrtc::FrameEncryptorInterface> frame_encryptor) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  parameters_.config.frame_encryptor = frame_encryptor;
  if (stream_) {
    RTC_LOG(LS_INFO)
        << "RecreateWebRtcStream (send) because of SetFrameEncryptor, ssrc="
        << parameters_.config.rtp.ssrcs[0];
    RecreateWebRtcStream();
  }
}
```



### 8.6 WebRtcVideoSendStream::SetEncoderToPacketizerFrameTransformer

media/engine/webrtc_video_engine.cc

1. **Encoded Transform**：操作编码后的数据以及解码前数据（Post-encoding/Pre-decoding）
2. https://blog.jianchihu.net/webrtc-research-encoded-transform.html
3. webrtc::FrameTransformerInterface，api/frame_transformer_interface.h

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::
    SetEncoderToPacketizerFrameTransformer(
        rtc::scoped_refptr<webrtc::FrameTransformerInterface>
            frame_transformer) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  parameters_.config.frame_transformer = std::move(frame_transformer);
  if (stream_)
    RecreateWebRtcStream();
}
```



### 8.7 WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream()

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
  
  // VideoSendStreamParameters parameters_
  parameters_.encoder_config.encoder_specific_settings =
      ConfigureVideoEncoderSettings(parameters_.codec_settings->codec);

  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
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
  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!
  // 创建webrtc::internal::VideoSendStream的实例，父类是webrtc::VideoSendStream
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());
  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!

  parameters_.encoder_config.encoder_specific_settings = NULL;

  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!
  if (source_) {
    stream_->SetSource(source_, GetDegradationPreference());
  }
  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!

  // Call stream_->Start() if necessary conditions are met.
  UpdateSendState();
}
```


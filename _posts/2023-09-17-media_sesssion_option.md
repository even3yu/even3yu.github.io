---
layout: post
title: webrtc MediaSesssionOption
date: 2023-09-17 23:30:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


## 1. MediaSessionOption

pc/media_session.h

![media_session_option](CreateOffer.assets/media_session_option.png)



```cpp
struct MediaSessionOptions {
  MediaSessionOptions() {}

  bool has_audio() const { return HasMediaDescription(MEDIA_TYPE_AUDIO); }
  bool has_video() const { return HasMediaDescription(MEDIA_TYPE_VIDEO); }
  bool has_data() const { return HasMediaDescription(MEDIA_TYPE_DATA); }

  bool MediaSessionOptions::HasMediaDescription(MediaType type) const {
    return absl::c_any_of(
        media_description_options,
        [type](const MediaDescriptionOptions& t) { return t.type == type; });
	}

  DataChannelType data_channel_type = DCT_NONE;
  bool vad_enabled = true;  // When disabled, removes all CN codecs from SDP.
  bool rtcp_mux_enabled = true;
  bool bundle_enabled = false;
  bool offer_extmap_allow_mixed = false;
  bool raw_packetization_for_video = false;
  std::string rtcp_cname = kDefaultRtcpCname;
  webrtc::CryptoOptions crypto_options;
  // List of media description options in the same order that the media
  // descriptions will be generated.
  std::vector<MediaDescriptionOptions> media_description_options;
  std::vector<IceParameters> pooled_ice_credentials;

  // Use the draft-ietf-mmusic-sctp-sdp-03 obsolete syntax for SCTP
  // datachannels.
  // Default is true for backwards compatibility with clients that use
  // this internal interface.
  bool use_obsolete_sctp_sdp = true;
};
```



### 1.1 默认值

![](assets/2023-09-15-13-24-42-img_v2_182e02a8-9b09-40c9-975e-98e847dafa1g.jpg)



### 1.2 bundle_enabled

对应sdp`a=group:BUNDLE audio video`;
如果是true，就是共用一个传输通道；

pc\sdp_offer_answer.cc

```cpp
void ExtractSharedMediaSessionOptions(
    const PeerConnectionInterface::RTCOfferAnswerOptions& rtc_options,
    cricket::MediaSessionOptions* session_options) {
  session_options->vad_enabled = rtc_options.voice_activity_detection;
  session_options->bundle_enabled = rtc_options.use_rtp_mux;
  session_options->raw_packetization_for_video =
      rtc_options.raw_packetization_for_video;
}
```

![](assets/2023-09-15-13-26-07-c85a30a7-13b1-42c4-9226-cf201c5645ae.jpeg)



### 1.3 vad_enabled

舒适噪音，当不可以用的时候，去除codec中的`CN` codec；



### 1.4 rtcp_mux_enabled

rtp和rtcp 共用端口



### 1.5 ??? offer_extmap_allow_mixed

https://blog.csdn.net/commshare/article/details/129052265



### 1.6 ???raw_packetization_for_video



### 1.7 -- CryptoOptions

Srtp的加密套件；SFrame？？



### 1.8 -- MediaDescriptionOptions



### 1.9 -- IceParameters pooled_ice_credentials



### 1.10 MediaSessionOptions::HasMediaDescription

```cpp
  bool MediaSessionOptions::HasMediaDescription(MediaType type) const {
    return absl::c_any_of(
        media_description_options,
        [type](const MediaDescriptionOptions& t) { return t.type == type; });
	}
```

是否包含某个媒体类型的数据；

```cpp
enum MediaType {
  MEDIA_TYPE_AUDIO,
  MEDIA_TYPE_VIDEO,
  MEDIA_TYPE_DATA,
  MEDIA_TYPE_UNSUPPORTED
};
```



## 2. CryptoOptions

api/crypto/crypto_options.h

```cpp
struct RTC_EXPORT CryptoOptions {
  std::vector<int> GetSupportedDtlsSrtpCryptoSuites() const;

  // SRTP的加密套件开关，默认开启aes128_sha1_80
  struct Srtp {
    bool enable_gcm_crypto_suites = false;
    bool enable_aes128_sha1_32_crypto_cipher = false;
    bool enable_aes128_sha1_80_crypto_cipher = true;
    bool enable_encrypted_rtp_header_extensions = false;
  } srtp;

  // Options to be used when the FrameEncryptor / FrameDecryptor APIs are used.
  struct SFrame {
    // If set all RtpSenders must have an FrameEncryptor attached to them before
    // they are allowed to send packets. All RtpReceivers must have a
    // FrameDecryptor attached to them before they are able to receive packets.
    bool require_frame_encryption = false;
  } sframe;
};
```

- SRTP的加密套件开关，默认开启aes128_sha1_80
- SFrame？？



### 2.1 CryptoOptions::GetSupportedDtlsSrtpCryptoSuites

```cpp
std::vector<int> CryptoOptions::GetSupportedDtlsSrtpCryptoSuites() const {
  std::vector<int> crypto_suites;
  // Note: SRTP_AES128_CM_SHA1_80 is what is required to be supported (by
  // draft-ietf-rtcweb-security-arch), but SRTP_AES128_CM_SHA1_32 is allowed as
  // well, and saves a few bytes per packet if it ends up selected.
  // As the cipher suite is potentially insecure, it will only be used if
  // enabled by both peers.
  if (srtp.enable_aes128_sha1_32_crypto_cipher) {
    crypto_suites.push_back(rtc::SRTP_AES128_CM_SHA1_32);
  }
  if (srtp.enable_aes128_sha1_80_crypto_cipher) {
    crypto_suites.push_back(rtc::SRTP_AES128_CM_SHA1_80);
  }

  // Note: GCM cipher suites are not the top choice since they increase the
  // packet size. In order to negotiate them the other side must not support
  // SRTP_AES128_CM_SHA1_80.
  if (srtp.enable_gcm_crypto_suites) {
    crypto_suites.push_back(rtc::SRTP_AEAD_AES_256_GCM);
    crypto_suites.push_back(rtc::SRTP_AEAD_AES_128_GCM);
  }
  RTC_CHECK(!crypto_suites.empty());
  return crypto_suites;
}
```

根据Srtp 的配置，获取SRTP的加密套件；主要用于媒体数据加密的时候使用。
目前支持下面4种加密套件。
rtc_base/ssl_stream_adapter.h

```cpp
const int SRTP_AES128_CM_SHA1_80 = 0x0001;
const int SRTP_AES128_CM_SHA1_32 = 0x0002;
const int SRTP_AEAD_AES_128_GCM = 0x0007;
const int SRTP_AEAD_AES_256_GCM = 0x0008;
```



### 2.2 CryptoOptions::Srtp

支持的Srtp的加密套件的支持，默认支持`enable_aes128_sha1_80_crypto_cipher`。



### 2.3 ??? CryptoOptions::SFrame



## 3. MediaDescriptionOptions

pc/media_session.h

```cpp
// Options for an individual media description/"m=" section.
struct MediaDescriptionOptions {
  ...
  // TODO(deadbeef): When we don't support Plan B, there will only be one
  // sender per media description and this can be simplified.
  void AddAudioSender(const std::string& track_id,
                      const std::vector<std::string>& stream_ids);
  void AddVideoSender(const std::string& track_id,
                      const std::vector<std::string>& stream_ids,
                      const std::vector<RidDescription>& rids,
                      const SimulcastLayerList& simulcast_layers,
                      int num_sim_layers);
	...

  MediaType type;
  std::string mid;
  webrtc::RtpTransceiverDirection direction;
  bool stopped;
  TransportOptions transport_options;
  // Note: There's no equivalent "RtpReceiverOptions" because only send
  // stream information goes in the local descriptions.
  std::vector<SenderOptions> sender_options;
  std::vector<webrtc::RtpCodecCapability> codec_preferences;
  std::vector<webrtc::RtpHeaderExtensionCapability> header_extensions;
};
```

对应sdp中的mLine。有几个mLine就有几个MediaDescriptionOptions。

![](assets/2023-09-15-13-34-44-img_v2_a616c41c-bf61-42ac-b12e-b9552ab8d89g.jpg)



### 3.1 MediaType type

api/media_types.h

```cpp
enum MediaType {
  MEDIA_TYPE_AUDIO,
  MEDIA_TYPE_VIDEO,
  MEDIA_TYPE_DATA,
  MEDIA_TYPE_UNSUPPORTED
};
```

媒体的类型

```js
a=group:BUNDLE audio video
...
m=audio 9 RTP/AVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
...
a=mid:audio
...
a=recvonly
...
//-------------------------------------------------------
m=video 9 RTP/AVPF 100 96 98 127 125 97 99 101 124
...
a=mid:video
...
a=sendrecv
...
```

一个是`m=audio`, 一个是`m=video`。


api/media_types.cc

```cpp
const char kMediaTypeVideo[] = "video";
const char kMediaTypeAudio[] = "audio";
const char kMediaTypeData[] = "data";
```



### 3.2 std::string mid

m line 的唯一id

```js
a=group:BUNDLE audio video
...
m=audio 9 RTP/AVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
...
a=mid:audio
...
a=recvonly
...
//-------------------------------------------------------
m=video 9 RTP/AVPF 100 96 98 127 125 97 99 101 124
...
a=mid:video
...
a=sendrecv
...
```

一个是`a=mid:audio`, 一个是`a=mid:video`。



### 3.3 !!! webrtc::RtpTransceiverDirection direction

媒体数据的方向

```js
a=group:BUNDLE audio video
...
m=audio 9 RTP/AVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
...
a=mid:audio
...
a=recvonly
...
//-------------------------------------------------------
m=video 9 RTP/AVPF 100 96 98 127 125 97 99 101 124
...
a=mid:video
...
a=sendrecv
...
```

如上，有两个mLine，一个是audio，一个video；

- `a=mid:audio`的方向，recvonly
- `a=mid:video`的方向，sendrecv

#### planB

pc/sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::GetOptionsForPlanBOffer(
    const PeerConnectionInterface::RTCOfferAnswerOptions& offer_answer_options,
    cricket::MediaSessionOptions* session_options) {
  ...
	// Figure out transceiver directional preferences.
  bool send_audio =
      !rtp_manager()->GetAudioTransceiver()->internal()->senders().empty();
  bool send_video =
      !rtp_manager()->GetVideoTransceiver()->internal()->senders().empty();

  // By default, generate sendrecv/recvonly m= sections.
  bool recv_audio = true;
  bool recv_video = true;

  // By default, only offer a new m= section if we have media to send with it.
  bool offer_new_audio_description = send_audio;
  bool offer_new_video_description = send_video;
  bool offer_new_data_description =
      data_channel_controller()->HasDataChannels();

  // The "offer_to_receive_X" options allow those defaults to be overridden.
  if (offer_answer_options.offer_to_receive_audio !=
      PeerConnectionInterface::RTCOfferAnswerOptions::kUndefined) {
    recv_audio = (offer_answer_options.offer_to_receive_audio > 0);
    offer_new_audio_description =
        offer_new_audio_description ||
        (offer_answer_options.offer_to_receive_audio > 0);
  }
  if (offer_answer_options.offer_to_receive_video !=
      RTCOfferAnswerOptions::kUndefined) {
    recv_video = (offer_answer_options.offer_to_receive_video > 0);
    offer_new_video_description =
        offer_new_video_description ||
        (offer_answer_options.offer_to_receive_video > 0);
  }
  ...
}
```

- send 方向是由`rtp_manager()->GetAudioTransceiver()->internal()->senders()` 个书决定
- recv方向，默认是true的， 根据`PeerConnectionInterface::RTCOfferAnswerOptions` 进行配置；



#### unify plan

##### PeerConnection::AddTransceiver

pc/peer_connection.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverInterface>>
PeerConnection::AddTransceiver(
    cricket::MediaType media_type,
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const RtpTransceiverInit& init,
    bool update_negotiation_needed) {
  ...
  transceiver->internal()->set_direction(init.direction);
  ...
    
}
```



##### RtpTransceiver::set_direction

pc/rtp_transceiver.h

```cpp
RtpTransceiverDirection direction_ = RtpTransceiverDirection::kInactive;
```

```cpp
  // Sets the intended direction for this transceiver. Intended to be used
  // internally over SetDirection since this does not trigger a negotiation
  // needed callback.
  void set_direction(RtpTransceiverDirection direction) {
    direction_ = direction;
  }

```



```cpp
RtpTransceiverDirection RtpTransceiver::direction() const {
  if (unified_plan_ && stopping())
    return webrtc::RtpTransceiverDirection::kStopped;

  return direction_;
}
```



```cpp
RTCError RtpTransceiver::SetDirectionWithError(
    RtpTransceiverDirection new_direction) {
  if (unified_plan_ && stopping()) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_STATE,
                         "Cannot set direction on a stopping transceiver.");
  }
  if (new_direction == direction_)
    return RTCError::OK();

  if (new_direction == RtpTransceiverDirection::kStopped) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                         "The set direction 'stopped' is invalid.");
  }

  direction_ = new_direction;
  on_negotiation_needed_();

  return RTCError::OK();
}
```







### 3.4 ???bool stopped



### 3.5 ??? TransportOptions transport_options

p2p/base/transport_description_factory.h

```cpp
struct TransportOptions {
  bool ice_restart = false;
  bool prefer_passive_role = false;
  // If true, ICE renomination is supported and will be used if it is also
  // supported by the remote side.
  bool enable_ice_renomination = false;
};
```

![](assets/2023-09-15-13-49-45-img_v2_ef84c54c-eda6-43c6-9ac5-f534f00dc37g.jpg)

![](assets/2023-09-15-13-50-08-img_v2_59a7cf34-ff2f-4e43-a50e-2bc81928128g.jpg)

### 3.6 ice_restart

### 3.7 prefer_passive_role

### 3.8 enable_ice_renomination

 

### 3.9 --std::vector\<SenderOptions> sender_options

 // Note: There's no equivalent "RtpReceiverOptions" because only send
  // stream information goes in the local descriptions.

【参考章节5】

### 3.10 --std::vector\<webrtc::RtpCodecCapability> codec_preferences

【参考章节6】

### 3.11 --std::vector\<webrtc::RtpHeaderExtensionCapability> header_extensions

【参考章节7】



## 4. IceParameters

p2p/base/transport_description.h

```cpp
struct IceParameters {
 	...
  // TODO(honghaiz): Include ICE mode in this structure to match the ORTC
  // struct:
  // http://ortc.org/wp-content/uploads/2016/03/ortc.html#idl-def-RTCIceParameters
  std::string ufrag;
  std::string pwd;
  bool renomination = false;
  ...
};
```



```js
...
a=group:BUNDLE audio video
m=audio 9 RTP/AVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
...
a=ice-ufrag:kT8P
a=ice-pwd:JbuD5zv2R9HY+9sMnBC1Ni26
a=ice-options:trickle
...
//-------------------------------
m=video 9 RTP/AVPF 100 96 98 127 125 97 99 101 124
...
a=ice-ufrag:kT8P
a=ice-pwd:JbuD5zv2R9HY+9sMnBC1Ni26
a=ice-options:trickle
...
```

- a=ice-ufrag (ICE Username Fragment)，描述当前 ICE 连接临时凭证的用户名部分。
- a=ice-pwd (ICE Password)，描述当前 ICE 连接临时凭证的密码部分。
- a=ice-options 用于描述 ICE 连接的属性信息，ice-options 的定义有很多种，WebRTC 中常见的有：
- a=ice-options:trickle client 一边收集 candidate 一边发送给对端并开始连通性检查，可以缩短 ICE 建立连接的时间。
- a=ice-options:renomination 允许 ICE controlling 一方动态重新提名新的 candidate ，默认情况 Offer 一方为controlling 角色，answer 一方为 controlled 角色；同时 Lite 一方只能为 controlled 角色。

### renomination

允许 ICE controlling 一方动态重新提名新的 candidate ，默认情况 Offer 一方为controlling 角色，answer 一方为 controlled 角色；同时 Lite 一方只能为 controlled 角色。

https://zhuanlan.zhihu.com/p/603456718



## 5. !!! SenderOptions

pc/media_session.h

多个SenderOptions，存放在vector中，多个是针对planB。

```cpp
struct SenderOptions {
  std::string track_id;
  std::vector<std::string> stream_ids;
  // Use RIDs and Simulcast Layers to indicate spec-compliant Simulcast.
  std::vector<RidDescription> rids;
  SimulcastLayerList simulcast_layers;
  // Use |num_sim_layers| to indicate legacy simulcast.
  int num_sim_layers;
};
```

- track_id 就是 Track的唯一识别码；
- 根据RtpSenderInternal 对应生成 SenderOptions，和RtpReceiverInternal没关系；
  SenderOptions和RtpSenderInternal，一一对应；
- Unify plan 一个Transceiver 只会 有一个SenderOptions ，planB可以有多个；



### 5.1 创建时机

#### planB

根据RtpSenderInternal 的向量来创建SenderOptions

```cpp
void MediaDescriptionOptions::AddSenderInternal(
    const std::string& track_id,
    const std::vector<std::string>& stream_ids,
    const std::vector<RidDescription>& rids,
    const SimulcastLayerList& simulcast_layers,
    int num_sim_layers) {
  // TODO(steveanton): Support any number of stream ids.
  RTC_CHECK(stream_ids.size() == 1U);
  SenderOptions options;
  options.track_id = track_id;
  options.stream_ids = stream_ids;
  options.simulcast_layers = simulcast_layers;
  options.rids = rids;
  options.num_sim_layers = num_sim_layers;
  sender_options.push_back(options);
}
```

```cpp
void MediaDescriptionOptions::AddAudioSender(
    const std::string& track_id,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK(type == MEDIA_TYPE_AUDIO);
  AddSenderInternal(track_id, stream_ids, {}, SimulcastLayerList(), 1);
}
```

```cpp
void AddPlanBRtpSenderOptions(
    const std::vector<rtc::scoped_refptr<
        RtpSenderProxyWithInternal<RtpSenderInternal>>>& senders,
    cricket::MediaDescriptionOptions* audio_media_description_options,
    cricket::MediaDescriptionOptions* video_media_description_options,
    int num_sim_layers) {
  for (const auto& sender : senders) {
    if (sender->media_type() == cricket::MEDIA_TYPE_AUDIO) {
      if (audio_media_description_options) {
        audio_media_description_options->AddAudioSender(
            sender->id(), sender->internal()->stream_ids());
      }
    } else {
      RTC_DCHECK(sender->media_type() == cricket::MEDIA_TYPE_VIDEO);
      if (video_media_description_options) {
        video_media_description_options->AddVideoSender(
            sender->id(), sender->internal()->stream_ids(), {},
            SimulcastLayerList(), num_sim_layers);
      }
    }
  }
}
```

![](assets/2023-09-15-13-55-05-img_v2_2100f37a-aa26-48e6-ab50-ee14578cdbfg.jpg)

![](assets/2023-09-15-13-55-10-img_v2_2c4655f5-79b1-443c-afb5-bce2e2e2e36g.jpg)

![](assets/2023-09-15-13-55-16-img_v2_34627f39-cbd7-402e-92f9-89fd7e6d3b4g.jpg)



#### Unify plan

pc/sdp_offer_answer.cc

##### GetMediaDescriptionOptionsForTransceiver

```cpp
static cricket::MediaDescriptionOptions
GetMediaDescriptionOptionsForTransceiver(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const std::string& mid,
    bool is_create_offer) {
  
  ...
  

  cricket::SenderOptions sender_options;
  sender_options.track_id = transceiver->sender()->id();
  sender_options.stream_ids = transceiver->sender()->stream_ids();

  // The following sets up RIDs and Simulcast.
  // RIDs are included if Simulcast is requested or if any RID was specified.
  RtpParameters send_parameters =
      transceiver->internal()->sender_internal()->GetParametersInternal();
  bool has_rids = std::any_of(send_parameters.encodings.begin(),
                              send_parameters.encodings.end(),
                              [](const RtpEncodingParameters& encoding) {
                                return !encoding.rid.empty();
                              });

  std::vector<RidDescription> send_rids;
  SimulcastLayerList send_layers;
  for (const RtpEncodingParameters& encoding : send_parameters.encodings) {
    if (encoding.rid.empty()) {
      continue;
    }
    send_rids.push_back(RidDescription(encoding.rid, RidDirection::kSend));
    send_layers.AddLayer(SimulcastLayer(encoding.rid, !encoding.active));
  }

  if (has_rids) {
    sender_options.rids = send_rids;
  }

  sender_options.simulcast_layers = send_layers;
  // When RIDs are configured, we must set num_sim_layers to 0 to.
  // Otherwise, num_sim_layers must be 1 because either there is no
  // simulcast, or simulcast is acheived by munging the SDP.
  sender_options.num_sim_layers = has_rids ? 0 : 1;
  ////////////////////
  media_description_options.sender_options.push_back(sender_options);
  ////////////////////

  }
  
```



##### RtpParameters？？？

##### GetParametersInternal？？？

### 5.2 track_id

AudioTrack，VideoTrack的id



### 5.3 ？？？stream_ids

- RtpSenderInternal的唯一识别码，唯一识别码，用户id，或者随机生成的uuid
- 多个stream_ids_， 不过在unify plan中应该不会存在，因为一个transceiver 只有一个rtpsende。所以这个vector，一般是针对planB的。
- `PeerConnection::AddTransceiver`  ，  或者`PeerConnection::AddTrack` 参数传入， 由应用层创建；
- 作用是什么？？？



#### PeerConnection::AddTransceiver

pc/peer_connection.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverInterface>>
PeerConnection::AddTransceiver(
    cricket::MediaType media_type,
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const RtpTransceiverInit& init,
    bool update_negotiation_needed) {
    ...
    
  // Set the sender ID equal to the track ID if the track is specified unless
  // that sender ID is already in use.
  std::string sender_id = (track && !rtp_manager()->FindSenderById(track->id())
                               ? track->id()
                               : rtc::CreateRandomUuid());
  auto sender = rtp_manager()->CreateSender(
      media_type, sender_id, track, init.stream_ids, parameters.encodings);
  auto receiver =
      rtp_manager()->CreateReceiver(media_type, rtc::CreateRandomUuid());
  auto transceiver = rtp_manager()->CreateAndAddTransceiver(sender, receiver);
 	 ...
 }

```

- CreateSender，创建RtpSender， 通过`PeerConnection::AddTransceiver` 参数传入，应用层来给个唯一识别码，比如用户的id；
- CreateReceiver，创建RtpReceiver， stream_ids 随机生成；



#### RtpTransceiverInit

api/rtp_transceiver_interface.h

```cpp
struct RTC_EXPORT RtpTransceiverInit final {
  RtpTransceiverInit();
  RtpTransceiverInit(const RtpTransceiverInit&);
  ~RtpTransceiverInit();
  // Direction of the RtpTransceiver. See RtpTransceiverInterface::direction().
  RtpTransceiverDirection direction = RtpTransceiverDirection::kSendRecv;

  // The added RtpTransceiver will be added to these streams.
  std::vector<std::string> stream_ids;

  // TODO(bugs.webrtc.org/7600): Not implemented.
  std::vector<RtpEncodingParameters> send_encodings;
};
```

- RtpTransceiverDirection direction, 通过RtpTransceiver::set_direction 接口进行配置方向
- stream_ids
-  std::vector<RtpEncodingParameters> send_encodings



#### ----------------

#### PeerConnection::AddTrack

pc/peer_connection.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>> PeerConnection::AddTrack(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  ...
  auto sender_or_error = rtp_manager()->AddTrack(track, stream_ids);
  ...
}
```

stream_ids 是通过接口设置的，



#### RtpTransmissionManager::AddTrack

pc/rtp_transmission_manager.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrack(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK_RUN_ON(signaling_thread());

  return (IsUnifiedPlan() ? AddTrackUnifiedPlan(track, stream_ids)
                          : AddTrackPlanB(track, stream_ids));
}
```



#### RtpTransmissionManager::AddTrackPlanB

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackPlanB(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
	...
  std::vector<std::string> adjusted_stream_ids = stream_ids;
  if (adjusted_stream_ids.empty()) {
    adjusted_stream_ids.push_back(rtc::CreateRandomUuid());
  }
  cricket::MediaType media_type =
      (track->kind() == MediaStreamTrackInterface::kAudioKind
           ? cricket::MEDIA_TYPE_AUDIO
           : cricket::MEDIA_TYPE_VIDEO);
  //////////////////////
  auto new_sender =
      CreateSender(media_type, track->id(), track, adjusted_stream_ids, {});
  //////////////////////
  ...
}
```



#### RtpTransmissionManager::AddTrackUnifiedPlan

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackUnifiedPlan(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  auto transceiver = FindFirstTransceiverForAddedTrack(track);
  if (transceiver) {
		...
    if (transceiver->direction() == RtpTransceiverDirection::kRecvOnly) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendRecv);
    } else if (transceiver->direction() == RtpTransceiverDirection::kInactive) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendOnly);
    }
    transceiver->sender()->SetTrack(track);
    //////////////
    transceiver->internal()->sender_internal()->set_stream_ids(stream_ids);
    ///////////////
    transceiver->internal()->set_reused_for_addtrack(true);
  } else {
		...
    std::string sender_id = track->id();
    // Avoid creating a sender with an existing ID by generating a random ID.
    // This can happen if this is the second time AddTrack has created a sender
    // for this track.
    if (FindSenderById(sender_id)) {
      sender_id = rtc::CreateRandomUuid();
    }
    /////////////////
    auto sender = CreateSender(media_type, sender_id, track, stream_ids, {});
    ////////////////
		...
  }
  return transceiver->sender();
}
```

#### -------公共部分------

#### PeerConnection::CreateSender

pc/peer_connection.cc

```cpp
rtc::scoped_refptr<RtpSenderInterface> PeerConnection::CreateSender(
    const std::string& kind,
    const std::string& stream_id) {
  ...
      // Internally we need to have one stream with Plan B semantics, so we
  // generate a random stream ID if not specified.
  std::vector<std::string> stream_ids;
  if (stream_id.empty()) {
    stream_ids.push_back(rtc::CreateRandomUuid());
    RTC_LOG(LS_INFO)
        << "No stream_id specified for sender. Generated stream ID: "
        << stream_ids[0];
  } else {
    stream_ids.push_back(stream_id);
  }
  ...
  new_sender->internal()->set_stream_ids(stream_ids);
}
```



#### RtpSenderBase

rc/pc/rtp_sender.h

```cpp
class RtpSenderBase : public RtpSenderInternal, public ObserverInterface {

  ...
  std::vector<std::string> stream_ids() const override { return stream_ids_; }
  void set_stream_ids(const std::vector<std::string>& stream_ids) override {
    stream_ids_ = stream_ids;
  }
  ...
    
  std::vector<std::string> stream_ids_;
  
}
```

多个stream_ids_， 不过在unify plan中应该不会存在，因为一个transceiver 只有一个rtpsende。所以这个vector，一般是针对planB的。



### 5.4 ？？？RidDescription rids



### 5.5 ？？？SimulcastLayerList simulcast_layers



### 5.6 ？？？num_sim_layers



## 6. RtpCodecCapability

api/rtp_parameters.h

多个，vector。 

```cpp
struct RTC_EXPORT RtpCodecCapability {
  // Build MIME "type/subtype" string from |name| and |kind|.
  std::string mime_type() const { return MediaTypeToString(kind) + "/" + name; }

  // Used to identify the codec. Equivalent to MIME subtype.
  std::string name;

  // The media type of this codec. Equivalent to MIME top-level type.
  cricket::MediaType kind = cricket::MEDIA_TYPE_AUDIO;

  // Clock rate in Hertz. If unset, the codec is applicable to any clock rate.
  absl::optional<int> clock_rate;

  // Default payload type for this codec. Mainly needed for codecs that use
  // that have statically assigned payload types.
  absl::optional<int> preferred_payload_type;

  // Maximum packetization time supported by an RtpReceiver for this codec.
  // TODO(deadbeef): Not implemented.
  absl::optional<int> max_ptime;

  // Preferred packetization time for an RtpReceiver or RtpSender of this codec.
  // TODO(deadbeef): Not implemented.
  absl::optional<int> ptime;

  // The number of audio channels supported. Unused for video codecs.
  absl::optional<int> num_channels;

  // Feedback mechanisms supported for this codec.
  std::vector<RtcpFeedback> rtcp_feedback;

  // Codec-specific parameters that must be signaled to the remote party.
  //
  // Corresponds to "a=fmtp" parameters in SDP.
  //
  // Contrary to ORTC, these parameters are named using all lowercase strings.
  // This helps make the mapping to SDP simpler, if an application is using SDP.
  // Boolean values are represented by the string "1".
  std::map<std::string, std::string> parameters;

  // Codec-specific parameters that may optionally be signaled to the remote
  // party.
  // TODO(deadbeef): Not implemented.
  std::map<std::string, std::string> options;

  // Maximum number of temporal layer extensions supported by this codec.
  // For example, a value of 1 indicates that 2 total layers are supported.
  // TODO(deadbeef): Not implemented.
  int max_temporal_layer_extensions = 0;

  // Maximum number of spatial layer extensions supported by this codec.
  // For example, a value of 1 indicates that 2 total layers are supported.
  // TODO(deadbeef): Not implemented.
  int max_spatial_layer_extensions = 0;

  // Whether the implementation can send/receive SVC layers with distinct SSRCs.
  // Always false for audio codecs. True for video codecs that support scalable
  // video coding with MRST.
  // TODO(deadbeef): Not implemented.
  bool svc_multi_stream_support = false;
};

```



```js
...
a=group:BUNDLE audio video
m=audio 9 RTP/AVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
...
a=mid:audio
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=recvonly
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=rtcp-fb:111 transport-cc
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:104 ISAC/32000
a=rtpmap:9 G722/8000
a=rtpmap:102 ILBC/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:106 CN/32000
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
a=rtpmap:110 telephone-event/48000
a=rtpmap:112 telephone-event/32000
a=rtpmap:113 telephone-event/16000
a=rtpmap:126 telephone-event/8000
//-------------------------------
m=video 9 RTP/AVPF 100 96 98 127 125 97 99 101 124
...
a=mid:video
...
a=sendrecv
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:100 H264/90000
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=rtcp-fb:100 transport-cc
a=fmtp:100 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 transport-cc
a=rtpmap:98 VP9/90000
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=rtcp-fb:98 transport-cc
a=rtpmap:127 red/90000
a=rtpmap:125 ulpfec/90000
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=100
a=rtpmap:124 rtx/90000
a=fmtp:124 apt=127
```





### 6.1 赋值时机

pc/rtp_transceiver.cc

```cpp
RTCError RtpTransceiver::SetCodecPreferences(
    rtc::ArrayView<RtpCodecCapability> codec_capabilities) {
  RTC_DCHECK(unified_plan_);

  // 3. If codecs is an empty list, set transceiver's [[PreferredCodecs]] slot
  // to codecs and abort these steps.
  if (codec_capabilities.empty()) {
    codec_preferences_.clear();
    return RTCError::OK();
  }

  // 4. Remove any duplicate values in codecs.
  std::vector<RtpCodecCapability> codecs;
  absl::c_remove_copy_if(codec_capabilities, std::back_inserter(codecs),
                         [&codecs](const RtpCodecCapability& codec) {
                           return absl::c_linear_search(codecs, codec);
                         });

  // 6. to 8.
  RTCError result;
  if (media_type_ == cricket::MEDIA_TYPE_AUDIO) {
    std::vector<cricket::AudioCodec> recv_codecs, send_codecs;
    channel_manager_->GetSupportedAudioReceiveCodecs(&recv_codecs);
    channel_manager_->GetSupportedAudioSendCodecs(&send_codecs);

    result = VerifyCodecPreferences(codecs, send_codecs, recv_codecs);
  } else if (media_type_ == cricket::MEDIA_TYPE_VIDEO) {
    std::vector<cricket::VideoCodec> recv_codecs, send_codecs;
    channel_manager_->GetSupportedVideoReceiveCodecs(&recv_codecs);
    channel_manager_->GetSupportedVideoSendCodecs(&send_codecs);

    result = VerifyCodecPreferences(codecs, send_codecs, recv_codecs);
  }

  if (result.ok()) {
    codec_preferences_ = codecs;
  }

  return result;
}
```





#### GetMediaDescriptionOptionsForTransceiver

```cpp

static cricket::MediaDescriptionOptions
GetMediaDescriptionOptionsForTransceiver(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const std::string& mid,
    bool is_create_offer) {
  // NOTE: a stopping transceiver should be treated as a stopped one in
  // createOffer as specified in
  // https://w3c.github.io/webrtc-pc/#dom-rtcpeerconnection-createoffer.
  bool stopped =
      is_create_offer ? transceiver->stopping() : transceiver->stopped();
  cricket::MediaDescriptionOptions media_description_options(
      transceiver->media_type(), mid, transceiver->direction(), stopped);
  ////////////////////////////////
  media_description_options.codec_preferences =
      transceiver->codec_preferences();
  ////////////////////////////////
  media_description_options.header_extensions =
      transceiver->HeaderExtensionsToOffer();
  // This behavior is specified in JSEP. The gist is that:
  // 1. The MSID is included if the RtpTransceiver's direction is sendonly or
  //    sendrecv.
  // 2. If the MSID is included, then it must be included in any subsequent
  //    offer/answer exactly the same until the RtpTransceiver is stopped.
  if (stopped || (!RtpTransceiverDirectionHasSend(transceiver->direction()) &&
                  !transceiver->internal()->has_ever_been_used_to_send())) {
    return media_description_options;
  }

  cricket::SenderOptions sender_options;
  sender_options.track_id = transceiver->sender()->id();
  sender_options.stream_ids = transceiver->sender()->stream_ids();

  // The following sets up RIDs and Simulcast.
  // RIDs are included if Simulcast is requested or if any RID was specified.
  RtpParameters send_parameters =
      transceiver->internal()->sender_internal()->GetParametersInternal();
  bool has_rids = std::any_of(send_parameters.encodings.begin(),
                              send_parameters.encodings.end(),
                              [](const RtpEncodingParameters& encoding) {
                                return !encoding.rid.empty();
                              });

  std::vector<RidDescription> send_rids;
  SimulcastLayerList send_layers;
  for (const RtpEncodingParameters& encoding : send_parameters.encodings) {
    if (encoding.rid.empty()) {
      continue;
    }
    send_rids.push_back(RidDescription(encoding.rid, RidDirection::kSend));
    send_layers.AddLayer(SimulcastLayer(encoding.rid, !encoding.active));
  }

  if (has_rids) {
    sender_options.rids = send_rids;
  }

  sender_options.simulcast_layers = send_layers;
  // When RIDs are configured, we must set num_sim_layers to 0 to.
  // Otherwise, num_sim_layers must be 1 because either there is no
  // simulcast, or simulcast is acheived by munging the SDP.
  sender_options.num_sim_layers = has_rids ? 0 : 1;
  media_description_options.sender_options.push_back(sender_options);

  return media_description_options;
}
```



### 6.2 RtcpFeedback

```cpp
struct RTC_EXPORT RtcpFeedback {
  RtcpFeedbackType type = RtcpFeedbackType::CCM;

  // Equivalent to ORTC "parameter" field with slight differences:
  // 1. It's an enum instead of a string.
  // 2. Generic NACK feedback is represented by a GENERIC_NACK message type,
  //    rather than an unset "parameter" value.
  absl::optional<RtcpFeedbackMessageType> message_type;
	...
};
```



```cpp
// Used in RtcpFeedback struct when type is NACK or CCM.
enum class RtcpFeedbackMessageType {
  // Equivalent to {type: "nack", parameter: undefined} in ORTC.
  GENERIC_NACK,
  PLI,  // Usable with NACK.
  FIR,  // Usable with CCM.
};
```



```cpp
// Used in RtcpFeedback struct.
enum class RtcpFeedbackType {
  CCM,
  LNTF,  // "goog-lntf"
  NACK,
  REMB,  // "goog-remb"
  TRANSPORT_CC,
};

```





## 7. RtpHeaderExtensionCapability

api/rtp_parameters.h

多个，vector。

```cpp
struct RTC_EXPORT RtpHeaderExtensionCapability {
  // URI of this extension, as defined in RFC8285.
  std::string uri;

  // Preferred value of ID that goes in the packet.
  absl::optional<int> preferred_id;

  // If true, it's preferred that the value in the header is encrypted.
  // TODO(deadbeef): Not implemented.
  bool preferred_encrypt = false;

  // The direction of the extension. The kStopped value is only used with
  // RtpTransceiverInterface::HeaderExtensionsToOffer() and
  // SetOfferedRtpHeaderExtensions().
  RtpTransceiverDirection direction = RtpTransceiverDirection::kSendRecv;
};
```



```js
...
a=group:BUNDLE audio video
...
m=audio 9 RTP/AVPF 111 103 104 9 102 0 8 106 105 13 110 112 113 126
a=mid:audio
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=recvonly
...
//-----------------------
m=video 9 RTP/AVPF 100 96 98 127 125 97 99 101 124
...
a=mid:video
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 urn:3gpp:video-orientation
a=extmap:5 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=sendrecv
...
```

![](assets/2023-09-15-13-27-28-9693013c-9dd7-4128-b4fb-8fecae0563cc.jpeg)

![](assets/2023-09-15-13-27-45-img_v2_38adf991-6bb8-4237-9894-be2df651307g.jpg)

![](assets/2023-09-15-13-38-00-img_v2_51226699-9573-49af-9826-e80d647db06g.jpg)



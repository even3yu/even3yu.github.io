---
layout: post
title: webrtc MediaSesssionOption && JsepSessionDescription
date: 2023-09-12 23:11:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


## MediaSessionOption

pc/media_session.h



![media_session_option]({{ site.url }}{{ site.baseurl }}/images/CreateOffer.assets/media_session_option.png)



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



### bundle_enabled

对应sdp`a=group:BUNDLE audio video`;
如果是true，就是共用一个传输通道；



### vad_enabled

舒适噪音，当不可以用的时候，去除codec中的`CN` codec；



### rtcp_mux_enabled

rtp和rtcp 共用端口



### ??? offer_extmap_allow_mixed

https://blog.csdn.net/commshare/article/details/129052265



### ???raw_packetization_for_video



### -- CryptoOptions

Srtp的加密套件；SFrame？？



### -- MediaDescriptionOptions



### -- IceParameters pooled_ice_credentials



### MediaSessionOptions::HasMediaDescription

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





## CryptoOptions

api/crypto/crypto_options.h

```cpp
struct RTC_EXPORT CryptoOptions {
  std::vector<int> GetSupportedDtlsSrtpCryptoSuites() const;

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



### CryptoOptions::GetSupportedDtlsSrtpCryptoSuites

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



### Srtp

支持的Srtp的加密套件的支持，默认支持`enable_aes128_sha1_80_crypto_cipher`。



### ??? SFrame



## MediaDescriptionOptions

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



### MediaType type

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



### std::string mid

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



### webrtc::RtpTransceiverDirection direction

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



### ???bool stopped



### --TransportOptions transport_options

 

### --std::vector\<SenderOptions> sender_options

 // Note: There's no equivalent "RtpReceiverOptions" because only send
  // stream information goes in the local descriptions.

### --std::vector\<webrtc::RtpCodecCapability> codec_preferences



### --std::vector\<webrtc::RtpHeaderExtensionCapability> header_extensions



## IceParameters

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





## ???TransportOptions

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



### ice_restart

### prefer_passive_role

### enable_ice_renomination



## ???SenderOptions

pc/media_session.h

多个，vector。

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

### track_id



### stream_ids



### rids



### simulcast_layers



### num_sim_layers



## RtpCodecCapability

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



### RtcpFeedback

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





## RtpHeaderExtensionCapability

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





## JsepSessionDescription

![jsep-session-description]({{ site.url }}{{ site.baseurl }}/images/CreateOffer.assets/jsep-session-description.png)
---
layout: post
title: MediaContentDescription
date: 2024-03-11 12:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---

## MediaContentDescription 以及相关子类

```less
MediaContentDescription
MediaContentDescriptionImpl
AudioContentDescription VideoContentDescription RtpDataContentDescription
```



### MediaContentDescription

pc/session_description.h

```cpp
// Describes a session description media section. There are subclasses for each
// media type (audio, video, data) that will have additional information.
class MediaContentDescription {
 public:
  MediaContentDescription() = default;
  virtual ~MediaContentDescription() = default;

  virtual MediaType type() const = 0;

  // Try to cast this media description to an AudioContentDescription. Returns
  // nullptr if the cast fails.
  virtual AudioContentDescription* as_audio() { return nullptr; }
  virtual const AudioContentDescription* as_audio() const { return nullptr; }

  // Try to cast this media description to a VideoContentDescription. Returns
  // nullptr if the cast fails.
  virtual VideoContentDescription* as_video() { return nullptr; }
  virtual const VideoContentDescription* as_video() const { return nullptr; }

  virtual RtpDataContentDescription* as_rtp_data() { return nullptr; }
  virtual const RtpDataContentDescription* as_rtp_data() const {
    return nullptr;
  }

  virtual SctpDataContentDescription* as_sctp() { return nullptr; }
  virtual const SctpDataContentDescription* as_sctp() const { return nullptr; }

  virtual UnsupportedContentDescription* as_unsupported() { return nullptr; }
  virtual const UnsupportedContentDescription* as_unsupported() const {
    return nullptr;
  }

  virtual bool has_codecs() const = 0;

  // Copy operator that returns an unique_ptr.
  // Not a virtual function.
  // If a type-specific variant of Clone() is desired, override it, or
  // simply use std::make_unique<typename>(*this) instead of Clone().
  std::unique_ptr<MediaContentDescription> Clone() const {
    return absl::WrapUnique(CloneInternal());
  }

  // |protocol| is the expected media transport protocol, such as RTP/AVPF,
  // RTP/SAVPF or SCTP/DTLS.
  virtual std::string protocol() const { return protocol_; }
  virtual void set_protocol(const std::string& protocol) {
    protocol_ = protocol;
  }

  virtual webrtc::RtpTransceiverDirection direction() const {
    return direction_;
  }
  virtual void set_direction(webrtc::RtpTransceiverDirection direction) {
    direction_ = direction;
  }

  virtual bool rtcp_mux() const { return rtcp_mux_; }
  virtual void set_rtcp_mux(bool mux) { rtcp_mux_ = mux; }

  virtual bool rtcp_reduced_size() const { return rtcp_reduced_size_; }
  virtual void set_rtcp_reduced_size(bool reduced_size) {
    rtcp_reduced_size_ = reduced_size;
  }

  // Indicates support for the remote network estimate packet type. This
  // functionality is experimental and subject to change without notice.
  virtual bool remote_estimate() const { return remote_estimate_; }
  virtual void set_remote_estimate(bool remote_estimate) {
    remote_estimate_ = remote_estimate;
  }

  virtual int bandwidth() const { return bandwidth_; }
  virtual void set_bandwidth(int bandwidth) { bandwidth_ = bandwidth; }
  virtual std::string bandwidth_type() const { return bandwidth_type_; }
  virtual void set_bandwidth_type(std::string bandwidth_type) {
    bandwidth_type_ = bandwidth_type;
  }

  virtual const std::vector<CryptoParams>& cryptos() const { return cryptos_; }
  virtual void AddCrypto(const CryptoParams& params) {
    cryptos_.push_back(params);
  }
  virtual void set_cryptos(const std::vector<CryptoParams>& cryptos) {
    cryptos_ = cryptos;
  }

  virtual const RtpHeaderExtensions& rtp_header_extensions() const {
    return rtp_header_extensions_;
  }
  virtual void set_rtp_header_extensions(
      const RtpHeaderExtensions& extensions) {
    rtp_header_extensions_ = extensions;
    rtp_header_extensions_set_ = true;
  }
  virtual void AddRtpHeaderExtension(const webrtc::RtpExtension& ext) {
    rtp_header_extensions_.push_back(ext);
    rtp_header_extensions_set_ = true;
  }
  virtual void ClearRtpHeaderExtensions() {
    rtp_header_extensions_.clear();
    rtp_header_extensions_set_ = true;
  }
  // We can't always tell if an empty list of header extensions is
  // because the other side doesn't support them, or just isn't hooked up to
  // signal them. For now we assume an empty list means no signaling, but
  // provide the ClearRtpHeaderExtensions method to allow "no support" to be
  // clearly indicated (i.e. when derived from other information).
  virtual bool rtp_header_extensions_set() const {
    return rtp_header_extensions_set_;
  }
  virtual const StreamParamsVec& streams() const { return send_streams_; }
  // TODO(pthatcher): Remove this by giving mediamessage.cc access
  // to MediaContentDescription
  virtual StreamParamsVec& mutable_streams() { return send_streams_; }
  virtual void AddStream(const StreamParams& stream) {
    send_streams_.push_back(stream);
  }
  // Legacy streams have an ssrc, but nothing else.
  void AddLegacyStream(uint32_t ssrc) {
    AddStream(StreamParams::CreateLegacy(ssrc));
  }
  void AddLegacyStream(uint32_t ssrc, uint32_t fid_ssrc) {
    StreamParams sp = StreamParams::CreateLegacy(ssrc);
    sp.AddFidSsrc(ssrc, fid_ssrc);
    AddStream(sp);
  }

  // Sets the CNAME of all StreamParams if it have not been set.
  virtual void SetCnameIfEmpty(const std::string& cname) {
    for (cricket::StreamParamsVec::iterator it = send_streams_.begin();
         it != send_streams_.end(); ++it) {
      if (it->cname.empty())
        it->cname = cname;
    }
  }
  virtual uint32_t first_ssrc() const {
    if (send_streams_.empty()) {
      return 0;
    }
    return send_streams_[0].first_ssrc();
  }
  virtual bool has_ssrcs() const {
    if (send_streams_.empty()) {
      return false;
    }
    return send_streams_[0].has_ssrcs();
  }

  virtual void set_conference_mode(bool enable) { conference_mode_ = enable; }
  virtual bool conference_mode() const { return conference_mode_; }

  // https://tools.ietf.org/html/rfc4566#section-5.7
  // May be present at the media or session level of SDP. If present at both
  // levels, the media-level attribute overwrites the session-level one.
  virtual void set_connection_address(const rtc::SocketAddress& address) {
    connection_address_ = address;
  }
  virtual const rtc::SocketAddress& connection_address() const {
    return connection_address_;
  }

  // Determines if it's allowed to mix one- and two-byte rtp header extensions
  // within the same rtp stream.
  enum ExtmapAllowMixed { kNo, kSession, kMedia };
  virtual void set_extmap_allow_mixed_enum(
      ExtmapAllowMixed new_extmap_allow_mixed) {
    if (new_extmap_allow_mixed == kMedia &&
        extmap_allow_mixed_enum_ == kSession) {
      // Do not downgrade from session level to media level.
      return;
    }
    extmap_allow_mixed_enum_ = new_extmap_allow_mixed;
  }
  virtual ExtmapAllowMixed extmap_allow_mixed_enum() const {
    return extmap_allow_mixed_enum_;
  }
  virtual bool extmap_allow_mixed() const {
    return extmap_allow_mixed_enum_ != kNo;
  }

  // Simulcast functionality.
  virtual bool HasSimulcast() const { return !simulcast_.empty(); }
  virtual SimulcastDescription& simulcast_description() { return simulcast_; }
  virtual const SimulcastDescription& simulcast_description() const {
    return simulcast_;
  }
  virtual void set_simulcast_description(
      const SimulcastDescription& simulcast) {
    simulcast_ = simulcast;
  }
  virtual const std::vector<RidDescription>& receive_rids() const {
    return receive_rids_;
  }
  virtual void set_receive_rids(const std::vector<RidDescription>& rids) {
    receive_rids_ = rids;
  }

 protected:
  bool rtcp_mux_ = false;
  bool rtcp_reduced_size_ = false;
  bool remote_estimate_ = false;
  int bandwidth_ = kAutoBandwidth;
  std::string bandwidth_type_ = kApplicationSpecificBandwidth;
  std::string protocol_;
  std::vector<CryptoParams> cryptos_;
  std::vector<webrtc::RtpExtension> rtp_header_extensions_;
  bool rtp_header_extensions_set_ = false;
  StreamParamsVec send_streams_;
  bool conference_mode_ = false;
  webrtc::RtpTransceiverDirection direction_ =
      webrtc::RtpTransceiverDirection::kSendRecv;
  rtc::SocketAddress connection_address_;
  // Mixed one- and two-byte header not included in offer on media level or
  // session level, but we will respond that we support it. The plan is to add
  // it to our offer on session level. See todo in SessionDescription.
  ExtmapAllowMixed extmap_allow_mixed_enum_ = kNo;

  SimulcastDescription simulcast_;
  std::vector<RidDescription> receive_rids_;

 private:
  // Copy function that returns a raw pointer. Caller will assert ownership.
  // Should only be called by the Clone() function. Must be implemented
  // by each final subclass.
  virtual MediaContentDescription* CloneInternal() const = 0;
};
```



### MediaContentDescriptionImpl

pc/session_description.h

主要负责codec的管理。

```cpp
template <class C>
class MediaContentDescriptionImpl : public MediaContentDescription {
 public:
  void set_protocol(const std::string& protocol) override {
    RTC_DCHECK(IsRtpProtocol(protocol));
    protocol_ = protocol;
  }

  typedef C CodecType;

  // Codecs should be in preference order (most preferred codec first).
  virtual const std::vector<C>& codecs() const { return codecs_; }
  virtual void set_codecs(const std::vector<C>& codecs) { codecs_ = codecs; }
  bool has_codecs() const override { return !codecs_.empty(); }
  virtual bool HasCodec(int id) {
    bool found = false;
    for (typename std::vector<C>::iterator iter = codecs_.begin();
         iter != codecs_.end(); ++iter) {
      if (iter->id == id) {
        found = true;
        break;
      }
    }
    return found;
  }
  virtual void AddCodec(const C& codec) { codecs_.push_back(codec); }
  virtual void AddOrReplaceCodec(const C& codec) {
    for (typename std::vector<C>::iterator iter = codecs_.begin();
         iter != codecs_.end(); ++iter) {
      if (iter->id == codec.id) {
        *iter = codec;
        return;
      }
    }
    AddCodec(codec);
  }
  virtual void AddCodecs(const std::vector<C>& codecs) {
    typename std::vector<C>::const_iterator codec;
    for (codec = codecs.begin(); codec != codecs.end(); ++codec) {
      AddCodec(*codec);
    }
  }

 private:
  std::vector<C> codecs_;
};

```





### AudioContentDescription

pc/session_description.h

```cpp
class AudioContentDescription : public MediaContentDescriptionImpl<AudioCodec> {
 public:
  AudioContentDescription() {}

  virtual MediaType type() const { return MEDIA_TYPE_AUDIO; }
  virtual AudioContentDescription* as_audio() { return this; }
  virtual const AudioContentDescription* as_audio() const { return this; }

 private:
  virtual AudioContentDescription* CloneInternal() const {
    return new AudioContentDescription(*this);
  }
};
```



### VideoContentDescription

pc/session_description.h

```cpp
class VideoContentDescription : public MediaContentDescriptionImpl<VideoCodec> {
 public:
  virtual MediaType type() const { return MEDIA_TYPE_VIDEO; }
  virtual VideoContentDescription* as_video() { return this; }
  virtual const VideoContentDescription* as_video() const { return this; }

 private:
  virtual VideoContentDescription* CloneInternal() const {
    return new VideoContentDescription(*this);
  }
};
```
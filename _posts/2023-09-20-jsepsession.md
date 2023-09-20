---
layout: post
title: webrtc JsepSession
date: 2023-09-20 19:17:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


## 1. JsepSessionDescription

![jsep-session-description]({{ site.url }}{{ site.baseurl }}/images/CreateOffer.assets/jsep-session-description.png)

```cpp
class JsepSessionDescription : public SessionDescriptionInterface {
...
  std::unique_ptr<cricket::SessionDescription> description_;
  std::string session_id_;
  std::string session_version_;
  SdpType type_;
  std::vector<JsepCandidateCollection> candidate_collection_;
...
};
```

### 创建时机

`WebRtcSessionDescriptionFactory::InternalCreateOffer` 创建了 JsepSessionDescription 对象。



### -- SessionDescription



### --JsepCandidateCollection







## 2. SessionDescription

```cpp
class SessionDescription {
  ...
   // Content mutators.
  // Adds a content to this description. Takes ownership of ContentDescription*.
  void AddContent(const std::string& name,
                  MediaProtocolType type,
                  std::unique_ptr<MediaContentDescription> description);
  void AddContent(const std::string& name,
                  MediaProtocolType type,
                  bool rejected,
                  std::unique_ptr<MediaContentDescription> description);
  void AddContent(const std::string& name,
                  MediaProtocolType type,
                  bool rejected,
                  bool bundle_only,
                  std::unique_ptr<MediaContentDescription> description);
  void AddContent(ContentInfo&& content);

  bool RemoveContentByName(const std::string& name);
  
  //-------------------------------
  
  ...
	ContentInfos contents_;
  TransportInfos transport_infos_;
  ContentGroups content_groups_;
  bool msid_supported_ = true;
  // Default to what Plan B would do.
  // TODO(bugs.webrtc.org/8530): Change default to kMsidSignalingMediaSection.
  int msid_signaling_ = kMsidSignalingSsrcAttribute;
  // TODO(webrtc:9985): Activate mixed one- and two-byte header extension in
  // offer at session level. It's currently not included in offer by default
  // because clients prior to https://bugs.webrtc.org/9712 cannot parse this
  // correctly. If it's included in offer to us we will respond that we support
  // it.
  bool extmap_allow_mixed_ = false;
}
```



### SessionDescription::AddContent

添加ContentInfo，name 是从MediaDescriptionOptions::mid()。

```cpp

void SessionDescription::AddContent(
    const std::string& name,
    MediaProtocolType type,
    std::unique_ptr<MediaContentDescription> description) {
  ContentInfo content(type);
  content.name = name;
  content.set_media_description(std::move(description));
  AddContent(std::move(content));
}

void SessionDescription::AddContent(
    const std::string& name,
    MediaProtocolType type,
    bool rejected,
    std::unique_ptr<MediaContentDescription> description) {
  ContentInfo content(type);
  content.name = name;
  content.rejected = rejected;
  content.set_media_description(std::move(description));
  AddContent(std::move(content));
}

void SessionDescription::AddContent(
    const std::string& name,
    MediaProtocolType type,
    bool rejected,
    bool bundle_only,
    std::unique_ptr<MediaContentDescription> description) {
  ContentInfo content(type);
  content.name = name;
  content.rejected = rejected;
  content.bundle_only = bundle_only;
  content.set_media_description(std::move(description));
  AddContent(std::move(content));
}

void SessionDescription::AddContent(ContentInfo&& content) {
  if (extmap_allow_mixed()) {
    // Mixed support on session level overrides setting on media level.
    content.media_description()->set_extmap_allow_mixed_enum(
        MediaContentDescription::kSession);
  }
  contents_.push_back(std::move(content));
}
```



### 创建时机

`MediaSessionDescriptionFactory::CreateOffer` 创建了 SessionDescription 对象。MediaSessionOptions.media_description_options 转换为ContentInfos。

#### MediaSessionDescriptionFactory::CreateOffer



#### MediaSessionDescriptionFactory::CreateAnswer

```cpp
std::unique_ptr<SessionDescription>
MediaSessionDescriptionFactory::CreateAnswer(
    const SessionDescription* offer,
    const MediaSessionOptions& session_options,
    const SessionDescription* current_description) const {
  if (!offer) {
    return nullptr;
  }

  // Must have options for exactly as many sections as in the offer.
  RTC_DCHECK_EQ(offer->contents().size(),
                session_options.media_description_options.size());

  IceCredentialsIterator ice_credentials(
      session_options.pooled_ice_credentials);

  std::vector<const ContentInfo*> current_active_contents;
  if (current_description) {
    current_active_contents =
        GetActiveContents(*current_description, session_options);
  }

  StreamParamsVec current_streams =
      GetCurrentStreamParams(current_active_contents);

  // Get list of all possible codecs that respects existing payload type
  // mappings and uses a single payload type space.
  //
  // Note that these lists may be further filtered for each m= section; this
  // step is done just to establish the payload type mappings shared by all
  // sections.
  AudioCodecs answer_audio_codecs;
  VideoCodecs answer_video_codecs;
  RtpDataCodecs answer_rtp_data_codecs;
  GetCodecsForAnswer(current_active_contents, *offer, &answer_audio_codecs,
                     &answer_video_codecs, &answer_rtp_data_codecs);

  if (!session_options.vad_enabled) {
    // If application doesn't want CN codecs in answer.
    StripCNCodecs(&answer_audio_codecs);
  }

  auto answer = std::make_unique<SessionDescription>();

  // If the offer supports BUNDLE, and we want to use it too, create a BUNDLE
  // group in the answer with the appropriate content names.
  const ContentGroup* offer_bundle = offer->GetGroupByName(GROUP_TYPE_BUNDLE);
  ContentGroup answer_bundle(GROUP_TYPE_BUNDLE);
  // Transport info shared by the bundle group.
  std::unique_ptr<TransportInfo> bundle_transport;

  answer->set_extmap_allow_mixed(offer->extmap_allow_mixed());

  // Iterate through the media description options, matching with existing
  // media descriptions in |current_description|.
  size_t msection_index = 0;
  for (const MediaDescriptionOptions& media_description_options :
       session_options.media_description_options) {
    const ContentInfo* offer_content = &offer->contents()[msection_index];
    // Media types and MIDs must match between the remote offer and the
    // MediaDescriptionOptions.
    RTC_DCHECK(
        IsMediaContentOfType(offer_content, media_description_options.type));
    RTC_DCHECK(media_description_options.mid == offer_content->name);
    const ContentInfo* current_content = nullptr;
    if (current_description &&
        msection_index < current_description->contents().size()) {
      current_content = &current_description->contents()[msection_index];
    }
    RtpHeaderExtensions header_extensions = RtpHeaderExtensionsFromCapabilities(
        UnstoppedRtpHeaderExtensionCapabilities(
            media_description_options.header_extensions));
    switch (media_description_options.type) {
      case MEDIA_TYPE_AUDIO:
        if (!AddAudioContentForAnswer(
                media_description_options, session_options, offer_content,
                offer, current_content, current_description,
                bundle_transport.get(), answer_audio_codecs, header_extensions,
                &current_streams, answer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_VIDEO:
        if (!AddVideoContentForAnswer(
                media_description_options, session_options, offer_content,
                offer, current_content, current_description,
                bundle_transport.get(), answer_video_codecs, header_extensions,
                &current_streams, answer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
        ...
      default:
        RTC_NOTREACHED();
    }
    ++msection_index;
    // See if we can add the newly generated m= section to the BUNDLE group in
    // the answer.
    ContentInfo& added = answer->contents().back();
    if (!added.rejected && session_options.bundle_enabled && offer_bundle &&
        offer_bundle->HasContentName(added.name)) {
      answer_bundle.AddContentName(added.name);
      bundle_transport.reset(
          new TransportInfo(*answer->GetTransportInfoByName(added.name)));
    }
  }

  // If a BUNDLE group was offered, put a BUNDLE group in the answer even if
  // it's empty. RFC5888 says:
  //
  //   A SIP entity that receives an offer that contains an "a=group" line
  //   with semantics that are understood MUST return an answer that
  //   contains an "a=group" line with the same semantics.
  if (offer_bundle) {
    answer->AddGroup(answer_bundle);
  }

  if (answer_bundle.FirstContentName()) {
    // Share the same ICE credentials and crypto params across all contents,
    // as BUNDLE requires.
    if (!UpdateTransportInfoForBundle(answer_bundle, answer.get())) {
      RTC_LOG(LS_ERROR)
          << "CreateAnswer failed to UpdateTransportInfoForBundle.";
      return NULL;
    }

    if (!UpdateCryptoParamsForBundle(answer_bundle, answer.get())) {
      RTC_LOG(LS_ERROR)
          << "CreateAnswer failed to UpdateCryptoParamsForBundle.";
      return NULL;
    }
  }

  // The following determines how to signal MSIDs to ensure compatibility with
  // older endpoints (in particular, older Plan B endpoints).
  if (is_unified_plan_) {
    // Unified Plan needs to look at what the offer included to find the most
    // compatible answer.
    if (offer->msid_signaling() == 0) {
      // We end up here in one of three cases:
      // 1. An empty offer. We'll reply with an empty answer so it doesn't
      //    matter what we pick here.
      // 2. A data channel only offer. We won't add any MSIDs to the answer so
      //    it also doesn't matter what we pick here.
      // 3. Media that's either sendonly or inactive from the remote endpoint.
      //    We don't have any information to say whether the endpoint is Plan B
      //    or Unified Plan, so be conservative and send both.
      answer->set_msid_signaling(cricket::kMsidSignalingMediaSection |
                                 cricket::kMsidSignalingSsrcAttribute);
    } else if (offer->msid_signaling() ==
               (cricket::kMsidSignalingMediaSection |
                cricket::kMsidSignalingSsrcAttribute)) {
      // If both a=msid and a=ssrc MSID signaling methods were used, we're
      // probably talking to a Unified Plan endpoint so respond with just
      // a=msid.
      answer->set_msid_signaling(cricket::kMsidSignalingMediaSection);
    } else {
      // Otherwise, it's clear which method the offerer is using so repeat that
      // back to them.
      answer->set_msid_signaling(offer->msid_signaling());
    }
  } else {
    // Plan B always signals MSID using a=ssrc lines.
    answer->set_msid_signaling(cricket::kMsidSignalingSsrcAttribute);
  }

  return answer;
}
```



### --ContentInfos



### --TransportInfos



### --ContentGroups



## 3. ContentInfos

pc/session_description.h

```cpp
typedef std::vector<ContentInfo> ContentInfos;
```



```cpp
class RTC_EXPORT ContentInfo {
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
  std::string name;
  MediaProtocolType type;
  bool rejected = false;
  bool bundle_only = false;
private:
  std::unique_ptr<MediaContentDescription> description_;
}
```



### ContentInfo创建时机

pc/media_session.cc

```cpp
MediaSessionDescriptionFactory::CreateOffer
MediaSessionDescriptionFactory::CreateAnswer
  
对所有的MediaDescriptionOption 数组 进行处理
====>
MediaSessionDescriptionFactory::AddAudioContentForOffer
MediaSessionDescriptionFactory::AddVideoContentForOffer
MediaSessionDescriptionFactory::AddAudioContentForAnswer
MediaSessionDescriptionFactory::AddVideoContentForAnswer
====>
SessionDescription::AddContent
```



### MediaSessionDescriptionFactory::AddAudioContentForOffer

```cpp
bool MediaSessionDescriptionFactory::AddAudioContentForOffer(
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const ContentInfo* current_content,
    const SessionDescription* current_description,
    const RtpHeaderExtensions& audio_rtp_extensions,
    const AudioCodecs& audio_codecs,
    StreamParamsVec* current_streams,
    SessionDescription* desc,
    IceCredentialsIterator* ice_credentials) const {
  // Filter audio_codecs (which includes all codecs, with correctly remapped
  // payload types) based on transceiver direction.
  const AudioCodecs& supported_audio_codecs =
      GetAudioCodecsForOffer(media_description_options.direction);

  AudioCodecs filtered_codecs;

  if (!media_description_options.codec_preferences.empty()) {
    // Add the codecs from the current transceiver's codec preferences.
    // They override any existing codecs from previous negotiations.
    filtered_codecs = MatchCodecPreference(
        media_description_options.codec_preferences, supported_audio_codecs);
  } else {
    // Add the codecs from current content if it exists and is not rejected nor
    // recycled.
    if (current_content && !current_content->rejected &&
        current_content->name == media_description_options.mid) {
      RTC_CHECK(IsMediaContentOfType(current_content, MEDIA_TYPE_AUDIO));
      const AudioContentDescription* acd =
          current_content->media_description()->as_audio();
      for (const AudioCodec& codec : acd->codecs()) {
        if (FindMatchingCodec<AudioCodec>(acd->codecs(), audio_codecs, codec,
                                          nullptr)) {
          filtered_codecs.push_back(codec);
        }
      }
    }
    // Add other supported audio codecs.
    AudioCodec found_codec;
    for (const AudioCodec& codec : supported_audio_codecs) {
      if (FindMatchingCodec<AudioCodec>(supported_audio_codecs, audio_codecs,
                                        codec, &found_codec) &&
          !FindMatchingCodec<AudioCodec>(supported_audio_codecs,
                                         filtered_codecs, codec, nullptr)) {
        // Use the |found_codec| from |audio_codecs| because it has the
        // correctly mapped payload type.
        filtered_codecs.push_back(found_codec);
      }
    }
  }

  cricket::SecurePolicy sdes_policy =
      IsDtlsActive(current_content, current_description) ? cricket::SEC_DISABLED
                                                         : secure();

  auto audio = std::make_unique<AudioContentDescription>();
  std::vector<std::string> crypto_suites;
  GetSupportedAudioSdesCryptoSuiteNames(session_options.crypto_options,
                                        &crypto_suites);
  if (!CreateMediaContentOffer(media_description_options, session_options,
                               filtered_codecs, sdes_policy,
                               GetCryptos(current_content), crypto_suites,
                               audio_rtp_extensions, ssrc_generator_,
                               current_streams, audio.get())) {
    return false;
  }

  bool secure_transport = (transport_desc_factory_->secure() != SEC_DISABLED);
  SetMediaProtocol(secure_transport, audio.get());

  audio->set_direction(media_description_options.direction);
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  desc->AddContent(media_description_options.mid, MediaProtocolType::kRtp,
                   media_description_options.stopped, std::move(audio));
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  if (!AddTransportOffer(media_description_options.mid,
                         media_description_options.transport_options,
                         current_description, desc, ice_credentials)) {
    return false;
  }

  return true;
}
```



### MediaSessionDescriptionFactory::AddVideoContentForOffer

### MediaSessionDescriptionFactory::AddAudioContentForAnswer

### MediaSessionDescriptionFactory::AddVideoContentForAnswer



## 4. TransportInfos

p2p\base\transport_info.h

```cpp
typedef std::vector<TransportInfo> TransportInfos;
```

```cpp
struct TransportInfo {
  std::string content_name;
  TransportDescription description;
};
```



### 创建时机



#### MediaSessionDescriptionFactory::CreateAnswer

```cpp
std::unique_ptr<SessionDescription>
MediaSessionDescriptionFactory::CreateAnswer(
    const SessionDescription* offer,
    const MediaSessionOptions& session_options,
    const SessionDescription* current_description) const {
  ...
    // Transport info shared by the bundle group.
  std::unique_ptr<TransportInfo> bundle_transport;
	...

  // Iterate through the media description options, matching with existing
  // media descriptions in |current_description|.
  size_t msection_index = 0;
  for (const MediaDescriptionOptions& media_description_options :
       session_options.media_description_options) {
    const ContentInfo* offer_content = &offer->contents()[msection_index];
    // Media types and MIDs must match between the remote offer and the
    // MediaDescriptionOptions.
    ...
    const ContentInfo* current_content = nullptr;
    if (current_description &&
        msection_index < current_description->contents().size()) {
      current_content = &current_description->contents()[msection_index];
    }
    RtpHeaderExtensions header_extensions = RtpHeaderExtensionsFromCapabilities(
        UnstoppedRtpHeaderExtensionCapabilities(
            media_description_options.header_extensions));
    switch (media_description_options.type) {
      case MEDIA_TYPE_AUDIO:
        if (!AddAudioContentForAnswer(
                media_description_options, session_options, offer_content,
                offer, current_content, current_description,
                bundle_transport.get(), answer_audio_codecs, header_extensions,
                &current_streams, answer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_VIDEO:
        if (!AddVideoContentForAnswer(
                media_description_options, session_options, offer_content,
                offer, current_content, current_description,
                bundle_transport.get(), answer_video_codecs, header_extensions,
                &current_streams, answer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_DATA:
        if (!AddDataContentForAnswer(
                media_description_options, session_options, offer_content,
                offer, current_content, current_description,
                bundle_transport.get(), answer_rtp_data_codecs,
                &current_streams, answer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_UNSUPPORTED:
        if (!AddUnsupportedContentForAnswer(
                media_description_options, session_options, offer_content,
                offer, current_content, current_description,
                bundle_transport.get(), answer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      default:
        RTC_NOTREACHED();
    }
    ++msection_index;
    // See if we can add the newly generated m= section to the BUNDLE group in
    // the answer.
    ContentInfo& added = answer->contents().back();
    if (!added.rejected && session_options.bundle_enabled && offer_bundle &&
        offer_bundle->HasContentName(added.name)) {
      answer_bundle.AddContentName(added.name);
      bundle_transport.reset(
          new TransportInfo(*answer->GetTransportInfoByName(added.name)));
    }
  }
  ...
}
```



#### MediaSessionDescriptionFactory::AddTransportOffer

```cpp
MediaSessionDescriptionFactory::AddAudioContentForOffer
MediaSessionDescriptionFactory::AddVideoContentForOffer
```

```cpp
bool MediaSessionDescriptionFactory::AddTransportOffer(
    const std::string& content_name,
    const TransportOptions& transport_options,
    const SessionDescription* current_desc,
    SessionDescription* offer_desc,
    IceCredentialsIterator* ice_credentials) const {
  if (!transport_desc_factory_)
    return false;
  const TransportDescription* current_tdesc =
      GetTransportDescription(content_name, current_desc);
  std::unique_ptr<TransportDescription> new_tdesc(
      transport_desc_factory_->CreateOffer(transport_options, current_tdesc,
                                           ice_credentials));
  if (!new_tdesc) {
    RTC_LOG(LS_ERROR) << "Failed to AddTransportOffer, content name="
                      << content_name;
  }
  offer_desc->AddTransportInfo(TransportInfo(content_name, *new_tdesc));
  return true;
}
```



#### MediaSessionDescriptionFactory::AddTransportAnswer

```cpp
MediaSessionDescriptionFactory::AddAudioContentForAnswer
MediaSessionDescriptionFactory::AddVideoContentForAnswer
```



```cpp
bool MediaSessionDescriptionFactory::AddTransportAnswer(
    const std::string& content_name,
    const TransportDescription& transport_desc,
    SessionDescription* answer_desc) const {
  answer_desc->AddTransportInfo(TransportInfo(content_name, transport_desc));
  return true;
}
```



### TransportDescription

p2p\base\transport_description.h

```cpp
struct TransportDescription {
   ...
  // TODO(deadbeef): Rename to HasIceOption, etc.
  bool HasOption(const std::string& option) const {
    return absl::c_linear_search(transport_options, option);
  }
  void AddOption(const std::string& option) {
    transport_options.push_back(option);
  }
  bool secure() const { return identity_fingerprint != nullptr; }

  IceParameters GetIceParameters() const {
    return IceParameters(ice_ufrag, ice_pwd,
                         HasOption(ICE_OPTION_RENOMINATION));
  }

  static rtc::SSLFingerprint* CopyFingerprint(const rtc::SSLFingerprint* from) {
    if (!from)
      return NULL;

    return new rtc::SSLFingerprint(*from);
  }

  // These are actually ICE options (appearing in the ice-options attribute in
  // SDP).
  // TODO(deadbeef): Rename to ice_options.
  std::vector<std::string> transport_options;
  std::string ice_ufrag;
  std::string ice_pwd;
  IceMode ice_mode;
  ConnectionRole connection_role;

  std::unique_ptr<rtc::SSLFingerprint> identity_fingerprint;
};
```



## 5. ContentGroups

pc/session_description.h

```cpp
typedef std::vector<ContentGroup> ContentGroups;
typedef std::vector<std::string> ContentNames;
```



```cpp
class ContentGroup {
  
  const std::string& semantics() const { return semantics_; }
  const ContentNames& content_names() const { return content_names_; }

  const std::string* FirstContentName() const;
  bool HasContentName(const std::string& content_name) const;
  void AddContentName(const std::string& content_name);
  bool RemoveContentName(const std::string& content_name);

 private:
  std::string semantics_;
  ContentNames content_names_;
};
```



### semantics_



### ContentNames content_names_



### ContentGroup::FirstContentName

```cpp
const std::string* ContentGroup::FirstContentName() const {
  return (!content_names_.empty()) ? &(*content_names_.begin()) : NULL;
}
```



### 创建时机

```cpp
std::unique_ptr<SessionDescription> MediaSessionDescriptionFactory::CreateOffer(
    const MediaSessionOptions& session_options,
    const SessionDescription* current_description) const {
  
  //创建添加了contentInfo
  ....

	if (session_options.bundle_enabled) {
    ContentGroup offer_bundle(GROUP_TYPE_BUNDLE);
    for (const ContentInfo& content : offer->contents()) {
      if (content.rejected) {
        continue;
      }
      // TODO(deadbeef): There are conditions that make bundling two media
      // descriptions together illegal. For example, they use the same payload
      // type to represent different codecs, or same IDs for different header
      // extensions. We need to detect this and not try to bundle those media
      // descriptions together.
      //！！！！！！！！！！！！！！！！！！！！！！！！！
      offer_bundle.AddContentName(content.name);
      //！！！！！！！！！！！！！！！！！！！！！！！！！
    }
    if (!offer_bundle.content_names().empty()) {
      offer->AddGroup(offer_bundle);
      if (!UpdateTransportInfoForBundle(offer_bundle, offer.get())) {
        RTC_LOG(LS_ERROR)
            << "CreateOffer failed to UpdateTransportInfoForBundle.";
        return nullptr;
      }
      if (!UpdateCryptoParamsForBundle(offer_bundle, offer.get())) {
        RTC_LOG(LS_ERROR)
            << "CreateOffer failed to UpdateCryptoParamsForBundle.";
        return nullptr;
      }
    }
  }
  ...
}
```



## 6. AudioContentDescription



### MediaContentDescriptionImpl

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



#### 赋值时机

pc\media_session.cc

```cpp
template <class C>
static bool CreateMediaContentOffer(
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const std::vector<C>& codecs,
    const SecurePolicy& secure_policy,
    const CryptoParamsVec* current_cryptos,
    const std::vector<std::string>& crypto_suites,
    const RtpHeaderExtensions& rtp_extensions,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescriptionImpl<C>* offer) {
  offer->AddCodecs(codecs);
  if (!AddStreamParams(media_description_options.sender_options,
                       session_options.rtcp_cname, ssrc_generator,
                       current_streams, offer)) {
    return false;
  }

  return CreateContentOffer(media_description_options, session_options,
                            secure_policy, current_cryptos, crypto_suites,
                            rtp_extensions, ssrc_generator, current_streams,
                            offer);
}
```

![](assets/2023-09-15-18-20-22-image.png)







### MediaContentDescription

```cpp
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
```





#### StreamParamsVec send_streams_



#### StreamParams

```cpp
static StreamParams CreateStreamParamsForNewSenderWithSsrcs(
    const SenderOptions& sender,
    const std::string& rtcp_cname,
    bool include_rtx_streams,
    bool include_flexfec_stream,
    UniqueRandomIdGenerator* ssrc_generator) {
  StreamParams result;
  result.id = sender.track_id;

  // TODO(brandtr): Update when we support multistream protection.
  if (include_flexfec_stream && sender.num_sim_layers > 1) {
    include_flexfec_stream = false;
    RTC_LOG(LS_WARNING)
        << "Our FlexFEC implementation only supports protecting "
           "a single media streams. This session has multiple "
           "media streams however, so no FlexFEC SSRC will be generated.";
  }
  if (include_flexfec_stream &&
      !webrtc::field_trial::IsEnabled("WebRTC-FlexFEC-03")) {
    include_flexfec_stream = false;
    RTC_LOG(LS_WARNING)
        << "WebRTC-FlexFEC trial is not enabled, not sending FlexFEC";
  }

  result.GenerateSsrcs(sender.num_sim_layers, include_rtx_streams,
                       include_flexfec_stream, ssrc_generator);

  result.cname = rtcp_cname;
  result.set_stream_ids(sender.stream_ids);

  return result;
}
```

![](assets/2023-09-15-18-45-15-image.png)

![](assets/2023-09-15-18-50-18-image.png)
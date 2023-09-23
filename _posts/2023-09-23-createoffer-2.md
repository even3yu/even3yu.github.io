---
layout: post
title: webrtc create offer-2
date: 2023-09-20 01:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


## 7. MediaSessionDescriptionFactory.CreateOffer

pc/media_session.cc

根据MediaSessionOptions创建SessionDescription,为每个m-line创建对应的新的ContentInfo结构体

```cpp
// - session_options 是SdpOfferAnswerHandler.GetOptionsForOffer准备好的
// - current_description 是当前的会话描述内容，如果是第一次 CreateOffer ，这个值为 nullptr，
// 如果中途因为某些原因需要再次协商会话描述信息，这个值就是有意义的。
 std::unique_ptr<SessionDescription> MediaSessionDescriptionFactory::CreateOffer(
    const MediaSessionOptions& session_options,
    const SessionDescription* current_description) const {
   // 1. 从已被应用的offer 和 当前MediaSessionOptions中抽取一些信息，
  //    以便后续为每个m-line创建对应的新的ContentInfo结构体
  // 1.1 当前已被应用的offer sdp中的mlinege个数必须比    
  //    MediaSessionOptions.media_description_options要少或者等于。
  //    实际上回顾GetOptionsForUnifiedPlanOffer方法搜集MediaSessionOptions
  //    中的media_description_options过程，就保证了这点。

   ...

  // 1.2 获取ice的凭证：ice credential即是ice parameter，包含
  //    ufrag，pwd，renomination三个参数
  IceCredentialsIterator ice_credentials(
      session_options.pooled_ice_credentials);


  // 1.3 从已被应用的当前offer中(就是上次setLocalDescription的offer)，获取活动的ContentInfo
  //    判断是否是活动的ContentInfo，必须是ContentInfo.rejected=fasle
  //    并且对应的session_options.media_options的stopped=false
  //
  // 第一次进来，current_description 是null，所以这个流程不走
  //！！！！！！！！
  // 获取到当前current_description在正常使用的m-line
  // ！！！！！！！
  std::vector<const ContentInfo*> current_active_contents;
  if (current_description) {
    current_active_contents =
        GetActiveContents(*current_description, session_options);
  }

  // 1.4 从激活(不是stop或者reject)的ContentInfo获取m-line的StreamParams，
  //    注意一个m-line对应一个ContentInfo，一个ContentInfo可能含有多个StreamParams
  //  	typedef std::vector<StreamParams> StreamParamsVec;
  StreamParamsVec current_streams =
      GetCurrentStreamParams(current_active_contents);

  // 1.5 从活动的ContentInfo中获取媒体编码器信息
  // 1.5.1 获取编码器信息，得到engine 本地支持的所有codec
  AudioCodecs offer_audio_codecs;
  VideoCodecs offer_video_codecs;
  RtpDataCodecs offer_rtp_data_codecs;
  GetCodecsForOffer(
      current_active_contents, &offer_audio_codecs, &offer_video_codecs,
      session_options.data_channel_type == DataChannelType::DCT_SCTP
          ? nullptr
          : &offer_rtp_data_codecs);
  // 1.5.2 根据session_options的信息对编码器进行过滤处理
  //			 去除codec， const char kComfortNoiseCodecName[] = "CN";
  //			 舒适噪音
  if (!session_options.vad_enabled) {
    // If application doesn't want CN codecs in offer.
    StripCNCodecs(&offer_audio_codecs);
  }
  // 1.6 获取Rtp扩展头信息
  AudioVideoRtpHeaderExtensions extensions_with_ids =
      GetOfferedRtpHeaderExtensionsWithIds(
          current_active_contents, session_options.offer_extmap_allow_mixed,
          session_options.media_description_options);

  // --------------------------------
  // --------------------------------
  // --------------------------------
  // 2. 为每个mline创建对应的ContentInfo，添加到SessionDescription
  // 2.1 创建SessionDescription对象
  auto offer = std::make_unique<SessionDescription>();

  // 2.2 迭代MediaSessionOptions中的每个MediaDescriptionOptions，创建Conteninfo，并添加到
  //     新建SessionDescription对象
  // 2.2.1 循环迭代， msection_index 不是 mid， 一直累
  // Iterate through the media description options, matching with existing media
  // descriptions in |current_description|.
  size_t msection_index = 0;
  for (const MediaDescriptionOptions& media_description_options :
       session_options.media_description_options) {
    // 2.2.2 获取当前ContentInfo
    //       要么存在于当前的offer sdp中，则从当前的offer sdp中获取即可
    //       要么是新加入的媒体，还没有ContentInfo，因此为空
    //
    // 			 第一次 current_description 为空
    const ContentInfo* current_content = nullptr;
    if (current_description &&
        msection_index < current_description->contents().size()) {
      // 从上次的offer中获取到current_content
      current_content = &current_description->contents()[msection_index];
      // Media type must match unless this media section is being recycled.
      RTC_DCHECK(current_content->name != media_description_options.mid ||
                 IsMediaContentOfType(current_content,
                                      media_description_options.type));
    }
    // 2.2.3 根据媒体类别，分别调用不同的方法创建ContentInfo，并添加到SessionDescription
    switch (media_description_options.type) {
      case MEDIA_TYPE_AUDIO:
        if (!AddAudioContentForOffer(media_description_options, session_options,
                                     current_content, current_description,
                                     extensions_with_ids.audio,
                                     offer_audio_codecs, &current_streams,
                                     offer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_VIDEO:
        if (!AddVideoContentForOffer(media_description_options, session_options,
                                     current_content, current_description,
                                     extensions_with_ids.video,
                                     offer_video_codecs, &current_streams,
                                     offer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_DATA:
        ...
        break;
      case MEDIA_TYPE_UNSUPPORTED:
        ...
        break;
      default:
        RTC_NOTREACHED();
    }
    ++msection_index;
  }

  // 3. 处理Bundle，如果session_options.bundle_enabled为真（默认为真），则需要将所有的
  //    ContentInfo全都进入一个ContentGroup，同一个ContentGroup是复用同一个底层传输的
  // Bundle the contents together, if we've been asked to do so, and update any
  // parameters that need to be tweaked for BUNDLE.
  if (session_options.bundle_enabled) {
    // 3.1 创建ContentGroup，并将每个有效的(活动的)ContentInfo添加到ContentGroup
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
      offer_bundle.AddContentName(content.name);
    }
   // 3.2 添加bundle到offer并更新bundle的传输通道信息、加密参数信息
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
 // 4. 设置一些其他信息
  // 4.1 设置msid信息
  // The following determines how to signal MSIDs to ensure compatibility with
  // older endpoints (in particular, older Plan B endpoints).
  if (is_unified_plan_) {
    // Be conservative and signal using both a=msid and a=ssrc lines. Unified
    // Plan answerers will look at a=msid and Plan B answerers will look at the
    // a=ssrc MSID line.
    offer->set_msid_signaling(cricket::kMsidSignalingMediaSection |
                              cricket::kMsidSignalingSsrcAttribute);
  } else {
    // Plan B always signals MSID using a=ssrc lines.
    offer->set_msid_signaling(cricket::kMsidSignalingSsrcAttribute);
  }

  // 4.2 
  offer->set_extmap_allow_mixed(session_options.offer_extmap_allow_mixed);

  return offer;
}
```

1. GetActiveContents， current_description就是上一次setLocalDescription 保存；
   从current_description 获取到获取到在正常使用的m-line（content.rejected = fasle&& media_options.stopped = false）；

2. 从1的中过滤出来ContentInfos，根据ContentInfo获取m-line的StreamParams，GetCurrentStreamParams；

   > 注意一个m-line对应一个ContentInfo，一个ContentInfo包含一个StreamParams（这是针对Unified Plan），Plan B可能含有多个StreamParams 。

3. GetCodecsForOffer， 根据 本地从engine获取到的codec（MediaSessionDescriptionFactory 构造函数）；

4. 创建SessionDescription，利用上面步骤提供的信息 && MediaSessionOptions提供的信息为每个mline创建对应的ContentInfo，添加到SessionDescription。`AddAudioContentForOffer`， `AddVideoContentForOffer`；

5. 如果`session_options.bundle_enabled = true`，且offer_bundle.name 不为空， `offer->AddGroup(offer_bundle);`



### ---这部分是根据上次协商的offer 提取的相关信息----

### 7.1 MediaSessionDescriptionFactory.GetActiveContents

pc/media_session.cc

```cpp
static std::vector<const ContentInfo*> GetActiveContents(
    const SessionDescription& description,
    const MediaSessionOptions& session_options) {
  std::vector<const ContentInfo*> active_contents;
  for (size_t i = 0; i < description.contents().size(); ++i) {
    RTC_DCHECK_LT(i, session_options.media_description_options.size());
    const ContentInfo& content = description.contents()[i];
    const MediaDescriptionOptions& media_options =
        session_options.media_description_options[i];
    // 正常使用
    if (!content.rejected && !media_options.stopped &&
        content.name == media_options.mid) {
      active_contents.push_back(&content);
    }
  }
  return active_contents;
}
```

从上次 协商的sdp信息中，和当前的MediaSessionOptions，获取到在正常使用的m-line。



### 7.2 MediaSessionDescriptionFactory.GetCurrentStreamParams

pc/media_session.cc

```cpp
// Finds all StreamParams of all media types and attach them to stream_params.
static StreamParamsVec GetCurrentStreamParams(
    const std::vector<const ContentInfo*>& active_local_contents) {
  StreamParamsVec stream_params;
  for (const ContentInfo* content : active_local_contents) {
    // media_description() 就说是返回 MediaContentDescription
    for (const StreamParams& params : content->media_description()->streams()) {
      stream_params.push_back(params);
    }
  }
  return stream_params;
}
```

从上一步中得到的active_local_contents， 来得到StreamParamsVec。



#### 7.2.1 ???StreamParams

media/base/stream_params.h

```cpp
struct StreamParams {
  // Resource of the MUC jid of the participant of with this stream.
  // For 1:1 calls, should be left empty (which means remote streams
  // and local streams should not be mixed together). This is not used
  // internally and should be deprecated.
  std::string groupid;
  // A unique identifier of the StreamParams object. When the SDP is created,
  // this comes from the track ID of the sender that the StreamParams object
  // is associated with.
  std::string id;
  // There may be no SSRCs stored in unsignaled case when stream_ids are
  // signaled with a=msid lines.
  std::vector<uint32_t> ssrcs;         // All SSRCs for this source
  std::vector<SsrcGroup> ssrc_groups;  // e.g. FID, FEC, SIM
  std::string cname;                   // RTCP CNAME

private:
  // The stream IDs of the sender that the StreamParams object is associated
  // with. In Plan B this should always be size of 1, while in Unified Plan this
  // could be none or multiple stream IDs.
  std::vector<std::string> stream_ids_;

  std::vector<RidDescription> rids_;
}
```

一个ContentInfo有一个StreamParams，StreamParams的id就是track-id，也就是sender-id/receiver-id，一个StreamParams存有多个SSRC和SsrcGroup，建立WebRtcAudioSendStream时使用first_ssrc，RTP报文里携带的。



### 7.3 MediaSessionDescriptionFactory.GetCodecsForOffer

获取音视频数据的所支持的编码

```cpp
 AudioCodecs offer_audio_codecs; // 存放的是最终的codec
 VideoCodecs offer_video_codecs;
 DataCodecs offer_data_codecs;
```

```cpp
// Getting codecs for an offer involves these steps:
//
// 1. Construct payload type -> codec mappings for current description.
// 2. Add any reference codecs that weren't already present
// 3. For each individual media description (m= section), filter codecs based
//    on the directional attribute (happens in another method).

// current_description 是空的
// audio_codecs，video_codecs，data_codecs 就是存放结果
void MediaSessionDescriptionFactory::GetCodecsForOffer(
    const SessionDescription* current_description,
    AudioCodecs* audio_codecs,
    VideoCodecs* video_codecs,
    DataCodecs* data_codecs) const {
  UsedPayloadTypes used_pltypes;
  audio_codecs->clear();
  video_codecs->clear();
  data_codecs->clear();

  // First - get all codecs from the current description if the media type
  // is used. Add them to |used_pltypes| so the payload type is not reused if a
  // new media type is added.
  // 第一次createOffer，current_description是空的
  if (current_description) {
    MergeCodecsFromDescription(current_description, audio_codecs, video_codecs,
                               data_codecs, &used_pltypes);
  }

  // Add our codecs that are not in |current_description|.
  // audio_codecs 就是合并存放的结果，
  // all_audio_codecs_就是从VoiceMediaEngine 获取的codec的能力
  MergeCodecs<AudioCodec>(all_audio_codecs_, audio_codecs, &used_pltypes);
  MergeCodecs<VideoCodec>(all_video_codecs_, video_codecs, &used_pltypes);
  MergeCodecs<DataCodec>(rtp_data_codecs_, data_codecs, &used_pltypes);
}
```

- 执行 clear() 的动作，避免指针指向了无效数据；
- 如果 current_description 不为空，也就是不是第一次执行 CreateOffer ，那么执行 MergeCodecsFromDescription ，将current_description 中记录的编码信息存入 offer_xxx_codecs；
- 执行 MergeCodecs，将本地支持的编码格式存入 offer_xxx_codecs。`all_audio_codecs_` 就是从VoiceMediaEngine 获取的codec的能力；



#### MediaSessionDescriptionFactory::MediaSessionDescriptionFactory

```cpp
MediaSessionDescriptionFactory::MediaSessionDescriptionFactory(
    ChannelManager* channel_manager,
    const TransportDescriptionFactory* transport_desc_factory,
    rtc::UniqueRandomIdGenerator* ssrc_generator)
    : MediaSessionDescriptionFactory(transport_desc_factory, ssrc_generator) {
  channel_manager->GetSupportedAudioSendCodecs(&audio_send_codecs_);
  channel_manager->GetSupportedAudioReceiveCodecs(&audio_recv_codecs_);
  channel_manager->GetSupportedVideoSendCodecs(&video_send_codecs_);
  channel_manager->GetSupportedVideoReceiveCodecs(&video_recv_codecs_);
  channel_manager->GetSupportedDataCodecs(&rtp_data_codecs_);
   
  // 得到 all_audio_codecs_
  ComputeAudioCodecsIntersectionAndUnion();
  // 得到 all_video_codecs_
  ComputeVideoCodecsIntersectionAndUnion();
}
```



#### MediaSessionDescriptionFactory::ComputeAudioCodecsIntersectionAndUnion

```cpp
void MediaSessionDescriptionFactory::ComputeAudioCodecsIntersectionAndUnion() {
  audio_sendrecv_codecs_.clear();
  all_audio_codecs_.clear();
  // Compute the audio codecs union.
  for (const AudioCodec& send : audio_send_codecs_) {
    all_audio_codecs_.push_back(send);
    if (!FindMatchingCodec<AudioCodec>(audio_send_codecs_, audio_recv_codecs_,
                                       send, nullptr)) {
      // It doesn't make sense to have an RTX codec we support sending but not
      // receiving.
      RTC_DCHECK(!IsRtxCodec(send));
    }
  }
  for (const AudioCodec& recv : audio_recv_codecs_) {
    if (!FindMatchingCodec<AudioCodec>(audio_recv_codecs_, audio_send_codecs_,
                                       recv, nullptr)) {
      all_audio_codecs_.push_back(recv);
    }
  }
  // Use NegotiateCodecs to merge our codec lists, since the operation is
  // essentially the same. Put send_codecs as the offered_codecs, which is the
  // order we'd like to follow. The reasoning is that encoding is usually more
  // expensive than decoding, and prioritizing a codec in the send list probably
  // means it's a codec we can handle efficiently.
  NegotiateCodecs(audio_recv_codecs_, audio_send_codecs_,
                  &audio_sendrecv_codecs_, true);
}
```

#### 7.3.1 MergeCodecsFromDescription

pc/media_session.cc

```cpp
MergeCodecsFromDescription(
    const std::vector<const ContentInfo*>& current_active_contents,
    AudioCodecs* audio_codecs,
    VideoCodecs* video_codecs,
    RtpDataCodecs* rtp_data_codecs,
    UsedPayloadTypes* used_pltypes) {
  for (const ContentInfo* content : current_active_contents) {
    if (IsMediaContentOfType(content, MEDIA_TYPE_AUDIO)) {
      const AudioContentDescription* audio =
          content->media_description()->as_audio();
      MergeCodecs<AudioCodec>(audio->codecs(), audio_codecs, used_pltypes);
    } else if (IsMediaContentOfType(content, MEDIA_TYPE_VIDEO)) {
      const VideoContentDescription* video =
          content->media_description()->as_video();
      MergeCodecs<VideoCodec>(video->codecs(), video_codecs, used_pltypes);
    } else if (IsMediaContentOfType(content, MEDIA_TYPE_DATA)) {
      const RtpDataContentDescription* data =
          content->media_description()->as_rtp_data();
      if (data) {
        // Only relevant for RTP datachannels
        MergeCodecs<RtpDataCodec>(data->codecs(), rtp_data_codecs,
                                  used_pltypes);
      }
    }
  }
}
```

#### 7.3.2 FindMatchingCodec

pc/media_session.cc

```cpp
// Finds a codec in |codecs2| that matches |codec_to_match|, which is
// a member of |codecs1|. If |codec_to_match| is an RTX codec, both
// the codecs themselves and their associated codecs must match.
template <class C>
static bool FindMatchingCodec(const std::vector<C>& codecs1,
                              const std::vector<C>& codecs2,
                              const C& codec_to_match,
                              C* found_codec) {
  // |codec_to_match| should be a member of |codecs1|, in order to look up RTX
  // codecs' associated codecs correctly. If not, that's a programming error.
  RTC_DCHECK(absl::c_any_of(codecs1, [&codec_to_match](const C& codec) {
    return &codec == &codec_to_match;
  }));
  for (const C& potential_match : codecs2) {
    if (potential_match.Matches(codec_to_match)) {
      if (IsRtxCodec(codec_to_match)) {
        int apt_value_1 = 0;
        int apt_value_2 = 0;
        if (!codec_to_match.GetParam(kCodecParamAssociatedPayloadType,
                                     &apt_value_1) ||
            !potential_match.GetParam(kCodecParamAssociatedPayloadType,
                                      &apt_value_2)) {
          RTC_LOG(LS_WARNING) << "RTX missing associated payload type.";
          continue;
        }
        if (!ReferencedCodecsMatch(codecs1, apt_value_1, codecs2,
                                   apt_value_2)) {
          continue;
        }
      }
      if (found_codec) {
        *found_codec = potential_match;
      }
      return true;
    }
  }
  return false;
}
```

从 codecs2 中 找到 codec_to_match， 找到return true；
codecs1 是用于rtx codec；

#### 7.3.3 MergeCodecs

pc/media_session.cc

```cpp
// Adds all codecs from |reference_codecs| to |offered_codecs| that don't
// already exist in |offered_codecs| and ensure the payload types don't
// collide.
template <class C>
static void MergeCodecs(const std::vector<C>& reference_codecs,
                        std::vector<C>* offered_codecs,
                        UsedPayloadTypes* used_pltypes) {
  // Add all new codecs that are not RTX codecs.
  // 添加新的codec
  for (const C& reference_codec : reference_codecs) {
    if (!IsRtxCodec(reference_codec) &&
        !FindMatchingCodec<C>(reference_codecs, *offered_codecs,
                              reference_codec, nullptr)) {
      C codec = reference_codec;
      used_pltypes->FindAndSetIdUsed(&codec);
      offered_codecs->push_back(codec);
    }
  }

  // Add all new RTX codecs.
  // 添加新的rtx codec
  for (const C& reference_codec : reference_codecs) {
    if (IsRtxCodec(reference_codec) &&
        !FindMatchingCodec<C>(reference_codecs, *offered_codecs,
                              reference_codec, nullptr)) {
      C rtx_codec = reference_codec;
      const C* associated_codec =
          GetAssociatedCodec(reference_codecs, rtx_codec);
      if (!associated_codec) {
        continue;
      }
      // Find a codec in the offered list that matches the reference codec.
      // Its payload type may be different than the reference codec.
      C matching_codec;
      if (!FindMatchingCodec<C>(reference_codecs, *offered_codecs,
                                *associated_codec, &matching_codec)) {
        RTC_LOG(LS_WARNING)
            << "Couldn't find matching " << associated_codec->name << " codec.";
        continue;
      }

      rtx_codec.params[kCodecParamAssociatedPayloadType] =
          rtc::ToString(matching_codec.id);
      used_pltypes->FindAndSetIdUsed(&rtx_codec);
      offered_codecs->push_back(rtx_codec);
    }
  }
}
```

1. reference_codecs，的元素在 offered_codecs 匹配，如果在offered_codecs 没找到，并且不是rtx，则 加入到offered_codecs

2. reference_codecs，的元素在 offered_codecs 匹配，如果在offered_codecs 没找到，并且是rtx，则 加入到offered_codecs

### 7.4 MediaSessionDescriptionFactory.GetOfferedRtpHeaderExtensionsWithIds



### ----end------



### 7.5 SessionDescription::SessionDescription

pc/session_description.h



### 7.6 !!!  ---MediaSessionDescriptionFactory::AddVideoContentForOffer

pc/media_session.cc

【详见8】。创建了VideoContentDescription，存入了ContentInfo， 并加入到SessionDescription。



### 7.8 ???MediaSessionDescriptionFactory.UpdateTransportInfoForBundle

pc/media_session.cc

```cpp
// Updates the transport infos of the |sdesc| according to the given
// |bundle_group|. The transport infos of the content names within the
// |bundle_group| should be updated to use the ufrag, pwd and DTLS role of the
// first content within the |bundle_group|.
static bool UpdateTransportInfoForBundle(const ContentGroup& bundle_group,
                                         SessionDescription* sdesc) {
  // The bundle should not be empty.
  if (!sdesc || !bundle_group.FirstContentName()) {
    return false;
  }

  // We should definitely have a transport for the first content.
  const std::string& selected_content_name = *bundle_group.FirstContentName();
  const TransportInfo* selected_transport_info =
      sdesc->GetTransportInfoByName(selected_content_name);
  if (!selected_transport_info) {
    return false;
  }

  // Set the other contents to use the same ICE credentials.
  const std::string& selected_ufrag =
      selected_transport_info->description.ice_ufrag;
  const std::string& selected_pwd =
      selected_transport_info->description.ice_pwd;
  ConnectionRole selected_connection_role =
      selected_transport_info->description.connection_role;
  for (TransportInfo& transport_info : sdesc->transport_infos()) {
    if (bundle_group.HasContentName(transport_info.content_name) &&
        transport_info.content_name != selected_content_name) {
      transport_info.description.ice_ufrag = selected_ufrag;
      transport_info.description.ice_pwd = selected_pwd;
      transport_info.description.connection_role = selected_connection_role;
    }
  }
  return true;
}
```



### 7.9 ???MediaSessionDescriptionFactory.UpdateCryptoParamsForBundle

pc/media_session.cc

```cpp
// Updates the crypto parameters of the |sdesc| according to the given
// |bundle_group|. The crypto parameters of all the contents within the
// |bundle_group| should be updated to use the common subset of the
// available cryptos.
static bool UpdateCryptoParamsForBundle(const ContentGroup& bundle_group,
                                        SessionDescription* sdesc) {
  // The bundle should not be empty.
  if (!sdesc || !bundle_group.FirstContentName()) {
    return false;
  }

  bool common_cryptos_needed = false;
  // Get the common cryptos.
  const ContentNames& content_names = bundle_group.content_names();
  CryptoParamsVec common_cryptos;
  bool first = true;
  for (const std::string& content_name : content_names) {
    if (!IsRtpContent(sdesc, content_name)) {
      continue;
    }
    // The common cryptos are needed if any of the content does not have DTLS
    // enabled.
    if (!sdesc->GetTransportInfoByName(content_name)->description.secure()) {
      common_cryptos_needed = true;
    }
    if (first) {
      first = false;
      // Initial the common_cryptos with the first content in the bundle group.
      if (!GetCryptosByName(sdesc, content_name, &common_cryptos)) {
        return false;
      }
      if (common_cryptos.empty()) {
        // If there's no crypto params, we should just return.
        return true;
      }
    } else {
      CryptoParamsVec cryptos;
      if (!GetCryptosByName(sdesc, content_name, &cryptos)) {
        return false;
      }
      PruneCryptos(cryptos, &common_cryptos);
    }
  }

  if (common_cryptos.empty() && common_cryptos_needed) {
    return false;
  }

  // Update to use the common cryptos.
  for (const std::string& content_name : content_names) {
    if (!IsRtpContent(sdesc, content_name)) {
      continue;
    }
    ContentInfo* content = sdesc->GetContentByName(content_name);
    if (IsMediaContent(content)) {
      MediaContentDescription* media_desc = content->media_description();
      if (!media_desc) {
        return false;
      }
      media_desc->set_cryptos(common_cryptos);
    }
  }
  return true;
}
```



## 8. !!! MediaSessionDescriptionFactory::AddVideoContentForOffer

pc/media_session.cc

创建了VideoContentDescription，存入了ContentInfo， 并加入到SessionDescription

```cpp
// TODO(kron): This function is very similar to AddAudioContentForOffer.
// Refactor to reuse shared code.
bool MediaSessionDescriptionFactory::AddVideoContentForOffer(
    const MediaDescriptionOptions& media_description_options, // local，对应m-line
    const MediaSessionOptions& session_options, // local
    const ContentInfo* current_content, // 上一次offer的ContentInfo
    const SessionDescription* current_description, // 上一次offer的SessionDescription 
    const RtpHeaderExtensions& video_rtp_extensions,
    const VideoCodecs& video_codecs, // 本地engine的所有codec
    StreamParamsVec* current_streams, // 上一次offer的 stream parsms
    SessionDescription* desc, // desc SessionDescription， 就是要填充的对象
    IceCredentialsIterator* ice_credentials) const {
  // Filter video_codecs (which includes all codecs, with correctly remapped
  // payload types) based on transceiver direction.
  // 1. 根据RtpTransceiverDirection， 来获取codec
  const VideoCodecs& supported_video_codecs =
      GetVideoCodecsForOffer(media_description_options.direction);

  VideoCodecs filtered_codecs;

  if (!media_description_options.codec_preferences.empty()) {
    // Add the codecs from the current transceiver's codec preferences.
    // They override any existing codecs from previous negotiations.
    // 2.1 根据codec_preferences过滤， 是从transceiver's codec preferences 设置过来的
    filtered_codecs = MatchCodecPreference(
        media_description_options.codec_preferences, supported_video_codecs);
  } else {
    // Add the codecs from current content if it exists and is not rejected nor
    // recycled.
    // 2.2 根据上一次offer的 ContentInfo，过滤出codec
    if (current_content && !current_content->rejected &&
        current_content->name == media_description_options.mid) {
      RTC_CHECK(IsMediaContentOfType(current_content, MEDIA_TYPE_VIDEO));
      // 2.3 根据上一次offer的 ContentInfo 得到VideoContentDescription；
      // 从 video_codecs 找到 codec， 如果找到，则filtered_codecs.push_back(codec);
      const VideoContentDescription* vcd =
          current_content->media_description()->as_video();
      for (const VideoCodec& codec : vcd->codecs()) {
        if (FindMatchingCodec<VideoCodec>(vcd->codecs(), video_codecs, codec,
                                          nullptr)) {
          filtered_codecs.push_back(codec);
        }
      }
    }
    // Add other supported video codecs.
    // 2.4 supported_video_codecs 是从本地的codec 过滤出来的
    VideoCodec found_codec;
    for (const VideoCodec& codec : supported_video_codecs) {
      if (FindMatchingCodec<VideoCodec>(supported_video_codecs, video_codecs,
                                        codec, &found_codec) &&
          !FindMatchingCodec<VideoCodec>(supported_video_codecs,
                                         filtered_codecs, codec, nullptr)) {
        // Use the |found_codec| from |video_codecs| because it has the
        // correctly mapped payload type.
        filtered_codecs.push_back(found_codec);
      }
    }
  }

  if (session_options.raw_packetization_for_video) {
    for (VideoCodec& codec : filtered_codecs) {
      if (codec.GetCodecType() == VideoCodec::CODEC_VIDEO) {
        codec.packetization = kPacketizationParamRaw;
      }
    }
  }

  // 3. 媒体数据加密方式，不加密，sdes，dtls
  // SecurePolicy secure() const { return secure_; }
  // 默认是SEC_DISABLED，
  // 安全策略
  cricket::SecurePolicy sdes_policy =
      IsDtlsActive(current_content, current_description) ? cricket::SEC_DISABLED
                                                      : secure();
  /////////////////
  //!!!!!!!! VideoContentDescription !!!!!!!!!!!!!!!!
  /////////////////
  auto video = std::make_unique<VideoContentDescription>();
  // 4. 加密套件
  std::vector<std::string> crypto_suites;
  GetSupportedVideoSdesCryptoSuiteNames(session_options.crypto_options,
                                        &crypto_suites);
  // 5. 填充 VideoContentDescription video
  if (!CreateMediaContentOffer(media_description_options, // // local，对应m-line
                               session_options, // local
                               filtered_codecs, // 过滤出来的codec
                               sdes_policy, // 安全策略
   														 // 根据上一次offer的 ContentInfo current_content，得到Cryptos
                               GetCryptos(current_content), 
                               crypto_suites, // local的加密套件
                               video_rtp_extensions, 
                               ssrc_generator_, // sscr 生成器
                               current_streams, // 上一次offer的 stream parsms
                               video.get())) { // VideoContentDescription
    return false;
  }

  video->set_bandwidth(kAutoBandwidth);

  // 6. 填充MediaContentDescription video 协议填充
  bool secure_transport = (transport_desc_factory_->secure() != SEC_DISABLED);
  SetMediaProtocol(secure_transport, video.get());

  video->set_direction(media_description_options.direction);

  // 7. SessionDescription* desc,向SessionDescription 中添加Content
  // 添加ContentInfo和VideoContentDescription
  desc->AddContent(media_description_options.mid, MediaProtocolType::kRtp,
                   media_description_options.stopped, std::move(video));
  // 8. 添加
  if (!AddTransportOffer(media_description_options.mid,
                         media_description_options.transport_options,
                         current_description, desc, ice_credentials)) {
    return false;
  }

  return true;
}
```

- 根据RtpTransceiverDirection，GetVideoCodecsForOffer 来获取codec
- **过滤 codec， GetVideoCodecsForAnswer 来获取codec + 7.3 计算得到codec 取交集；具体参考【11章节】**
- 创建VideoContentDescription
- 安全套件 `session_options.crypto_options`
- 设置协议
- 



### !!! ContentInfo::rejected

pc/session_description.h
这个值是 根据`media_description_options.stopped` 来确定的。



### -------codec 协商----

### 8.2 MediaSessionDescriptionFactory.NegotiateRtpTransceiverDirection

pc/media_session.cc

```cpp
static RtpTransceiverDirection NegotiateRtpTransceiverDirection(
    RtpTransceiverDirection offer,
    RtpTransceiverDirection wants) {
  // 远端支持发送
  bool offer_send = webrtc::RtpTransceiverDirectionHasSend(offer);
  // 远端支持接收
  bool offer_recv = webrtc::RtpTransceiverDirectionHasRecv(offer);
  // 本地想支持发送
  bool wants_send = webrtc::RtpTransceiverDirectionHasSend(wants);
  // 本地想支持接收
  bool wants_recv = webrtc::RtpTransceiverDirectionHasRecv(wants);
  // ！！！！！！！！！！！！！！！！
  // 取与，
	// offer_recv && wants_send， 远端支持接收，本地支持发送，本地才支持发送
  // offer_send && wants_recv， 远端支持接收，本地支持发送，本地才支持接收
  return webrtc::RtpTransceiverDirectionFromSendRecv(offer_recv && wants_send,
                                                     offer_send && wants_recv);
}

RtpTransceiverDirection RtpTransceiverDirectionFromSendRecv(bool send,
                                                            bool recv) {
  if (send && recv) {
    return RtpTransceiverDirection::kSendRecv;
  } else if (send && !recv) {
    return RtpTransceiverDirection::kSendOnly;
  } else if (!send && recv) {
    return RtpTransceiverDirection::kRecvOnly;
  } else {
    return RtpTransceiverDirection::kInactive;
  }
}
```

**本端接收的数据的，所以 direct 就是Recv；**
**远端接收发送数据，所以 direct 是send recv；**
**这样远端和本地进行协商，得到本端的方向就是 就是recv；**

api/rtp_transceiver_direction.h

```cpp
enum class RtpTransceiverDirection {
  kSendRecv,
  kSendOnly,
  kRecvOnly,
  kInactive,
  kStopped,
};
```

![engine-codc-of-direct](create-offer-2.assets/engine-codec-of-direct.jpg)



```
视频打开
remote sdp direct = SendRecv, 这是为什么？？？ 
local sdp direct =  Recv
=========》
Recv

视频关闭的时候
remote sdp direct = Recv, 这是为什么？？？ 
local sdp direct =  Recv
=========》
Init
```



### 8.3 MediaSessionDescriptionFactory::GetVideoCodecsForOffer

pc/media_session.cc

根据RtpTransceiverDirection 获取 engine 所支持的发送的codec，接收的codec，还是发送+接收的codec。
这是本地engine的能力。

```cpp
  AudioCodecs audio_send_codecs_;
  AudioCodecs audio_recv_codecs_;
  // Intersection of send and recv.
  AudioCodecs audio_sendrecv_codecs_;
  // Union of send and recv.
  AudioCodecs all_audio_codecs_;



MediaSessionDescriptionFactory::MediaSessionDescriptionFactory(
    ChannelManager* channel_manager,
    const TransportDescriptionFactory* transport_desc_factory,
    rtc::UniqueRandomIdGenerator* ssrc_generator)
    : MediaSessionDescriptionFactory(transport_desc_factory, ssrc_generator) {
  channel_manager->GetSupportedAudioSendCodecs(&audio_send_codecs_);
  channel_manager->GetSupportedAudioReceiveCodecs(&audio_recv_codecs_);
  channel_manager->GetSupportedVideoSendCodecs(&video_send_codecs_);
  channel_manager->GetSupportedVideoReceiveCodecs(&video_recv_codecs_);
  channel_manager->GetSupportedDataCodecs(&rtp_data_codecs_);
  ComputeAudioCodecsIntersectionAndUnion();
  ComputeVideoCodecsIntersectionAndUnion();
}
```



```cpp
const VideoCodecs& MediaSessionDescriptionFactory::GetVideoCodecsForOffer(
    const RtpTransceiverDirection& direction) const {
  switch (direction) {
    // If stream is inactive - generate list as if sendrecv.
    case RtpTransceiverDirection::kSendRecv:
    case RtpTransceiverDirection::kStopped:
    case RtpTransceiverDirection::kInactive:
      return video_sendrecv_codecs_;
    case RtpTransceiverDirection::kSendOnly:
      return video_send_codecs_;
    case RtpTransceiverDirection::kRecvOnly:
      return video_recv_codecs_;
  }
  RTC_CHECK_NOTREACHED();
}
```

- answer 就是 本地的方向，是接收，发送，还是接收+发送，或者即不发送也不接收；
  offer 是remote 的方向；

- direct 是 kSendOnly， kRecvOnly 好理解，对应`video_send_codecs_， video_recv_codecs_`；
- direct 是 kSendRecv，kInactive，kStopped，**根据远端的方向取反**（
  如果远端是 kSendRecv/kInactive/kStopped 取反保持不变， 返回的就是video_sendrecv_codecs_；
  kSendOnly 取反 就是 kRecvOnly， 返回 `video_recv_codecs_`；
  kRecvOnly 取反 就是 kSendOnly， 返回 `video_send_codecs_`）;



![negotiate-direct](create-offer-2.assets/negotiate-direct.jpg)

#### RtpTransceiverDirectionReversed

pc/rtp_media_utils.cc

```cpp
RtpTransceiverDirection RtpTransceiverDirectionReversed(
    RtpTransceiverDirection direction) {
  switch (direction) {
    case RtpTransceiverDirection::kSendRecv:
    case RtpTransceiverDirection::kInactive:
    case RtpTransceiverDirection::kStopped:
      return direction; // 原路返回
    case RtpTransceiverDirection::kSendOnly:
      return RtpTransceiverDirection::kRecvOnly;
    case RtpTransceiverDirection::kRecvOnly:
      return RtpTransceiverDirection::kSendOnly;
    default:
      RTC_NOTREACHED();
      return direction;
  }
}
```



### 8.4 MatchCodecPreference

pc/media_session.cc

### -------end codec 协商--------



### 8.5 VideoContentDescription

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



```cpp
MediaContentDescription
MediaContentDescriptionImpl<VideoCodec>
VideoContentDescription
```



### 8.6 !!!MediaSessionDescriptionFactory.GetSupportedVideoSdesCryptoSuiteNames

pc/media_session.cc

```cpp
void GetSupportedVideoSdesCryptoSuiteNames(
    const webrtc::CryptoOptions& crypto_options,
    std::vector<std::string>* crypto_suite_names) {
  GetSupportedSdesCryptoSuiteNames(GetSupportedVideoSdesCryptoSuites,
                                   crypto_options, crypto_suite_names);
}
```

获取srtp 加密套件。



#### GetSupportedVideoSdesCryptoSuites

```cpp
void GetSupportedVideoSdesCryptoSuites(
    const webrtc::CryptoOptions& crypto_options,
    std::vector<int>* crypto_suites) {
  crypto_suites->push_back(rtc::SRTP_AES128_CM_SHA1_80);
  if (crypto_options.srtp.enable_gcm_crypto_suites) {
    crypto_suites->push_back(rtc::SRTP_AEAD_AES_256_GCM);
    crypto_suites->push_back(rtc::SRTP_AEAD_AES_128_GCM);
  }
}
```

根据CryptoOptions 得到`std::vector<int>* crypto_suites`。



#### GetSupportedSdesCryptoSuiteNames

```cpp
void GetSupportedSdesCryptoSuiteNames(
    void (*func)(const webrtc::CryptoOptions&, std::vector<int>*),
    const webrtc::CryptoOptions& crypto_options,
    std::vector<std::string>* names) {
  std::vector<int> crypto_suites;
  // func = GetSupportedVideoSdesCryptoSuites
  func(crypto_options, &crypto_suites);
  for (const auto crypto : crypto_suites) {
    names->push_back(rtc::SrtpCryptoSuiteToName(crypto));
  }
}
```

获取加密套件的名字。



rtc_base/ssl_stream_adapter.cc

```cpp
std::string SrtpCryptoSuiteToName(int crypto_suite) {
  switch (crypto_suite) {
    case SRTP_AES128_CM_SHA1_32:
      return CS_AES_CM_128_HMAC_SHA1_32;
    case SRTP_AES128_CM_SHA1_80:
      return CS_AES_CM_128_HMAC_SHA1_80;
    case SRTP_AEAD_AES_128_GCM:
      return CS_AEAD_AES_128_GCM;
    case SRTP_AEAD_AES_256_GCM:
      return CS_AEAD_AES_256_GCM;
    default:
      return std::string();
  }
}


const char CS_AES_CM_128_HMAC_SHA1_80[] = "AES_CM_128_HMAC_SHA1_80";
const char CS_AES_CM_128_HMAC_SHA1_32[] = "AES_CM_128_HMAC_SHA1_32";
const char CS_AEAD_AES_128_GCM[] = "AEAD_AES_128_GCM";
const char CS_AEAD_AES_256_GCM[] = "AEAD_AES_256_GCM";
```







### GetCryptos

pc/media_session.cc

```cpp
const CryptoParamsVec* GetCryptos(const ContentInfo* content) {
  if (!content || !content->media_description()) {
    return nullptr;
  }
  return &content->media_description()->cryptos();
}
```



### 8.7 --MediaSessionDescriptionFactory.CreateMediaContentOffer

pc/media_session.cc

【章节10】



### !!! MediaSessionDescriptionFactory.SetMediaProtocol

pc/media_session.cc

```cpp
static void SetMediaProtocol(bool secure_transport,
                             MediaContentDescription* desc) {
  if (!desc->cryptos().empty())
    desc->set_protocol(kMediaProtocolSavpf); 
  	// const char kMediaProtocolSavpf[] = "RTP/SAVPF";
  else if (secure_transport)
    desc->set_protocol(kMediaProtocolDtlsSavpf); 
  	// const char kMediaProtocolDtlsSavpf[] = "UDP/TLS/RTP/SAVPF";
  else
    desc->set_protocol(kMediaProtocolAvpf);
  	// const char kMediaProtocolAvpf[] = "RTP/AVPF";
}
```



### 8.8 ???MediaSessionDescriptionFactory::AddTransportOffer

pc/media_session.cc

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





## 9. ??? MediaSessionDescriptionFactory::CreateTransportAnswer

pc/media_session.cc

```cpp
std::unique_ptr<TransportDescription>
MediaSessionDescriptionFactory::CreateTransportAnswer(
    const std::string& content_name,
    const SessionDescription* offer_desc,
    const TransportOptions& transport_options,
    const SessionDescription* current_desc,
    bool require_transport_attributes,
    IceCredentialsIterator* ice_credentials) const {
  if (!transport_desc_factory_)
    return NULL;
  const TransportDescription* offer_tdesc =
      GetTransportDescription(content_name, offer_desc);
  const TransportDescription* current_tdesc =
      GetTransportDescription(content_name, current_desc);
  return transport_desc_factory_->CreateAnswer(offer_tdesc, transport_options,
                                               require_transport_attributes,
                                               current_tdesc, ice_credentials);
}
 
```



### TransportDescriptionFactory::CreateAnswer

p2p/base/transport_description_factory.cc

```cpp
std::unique_ptr<TransportDescription> TransportDescriptionFactory::CreateAnswer(
    const TransportDescription* offer,
    const TransportOptions& options,
    bool require_transport_attributes,
    const TransportDescription* current_description,
    IceCredentialsIterator* ice_credentials) const {
  // TODO(juberti): Figure out why we get NULL offers, and fix this upstream.
  if (!offer) {
    RTC_LOG(LS_WARNING) << "Failed to create TransportDescription answer "
                           "because offer is NULL";
    return NULL;
  }

  auto desc = std::make_unique<TransportDescription>();
  // Generate the ICE credentials if we don't already have them or ice is
  // being restarted.
  if (!current_description || options.ice_restart) {
    IceParameters credentials = ice_credentials->GetIceCredentials();
    desc->ice_ufrag = credentials.ufrag;
    desc->ice_pwd = credentials.pwd;
  } else {
    desc->ice_ufrag = current_description->ice_ufrag;
    desc->ice_pwd = current_description->ice_pwd;
  }
  desc->AddOption(ICE_OPTION_TRICKLE);
  if (options.enable_ice_renomination) {
    desc->AddOption(ICE_OPTION_RENOMINATION);
  }

  // Negotiate security params.
  if (offer && offer->identity_fingerprint.get()) {
    // The offer supports DTLS, so answer with DTLS, as long as we support it.
    if (secure_ == SEC_ENABLED || secure_ == SEC_REQUIRED) {
      // Fail if we can't create the fingerprint.
      // Setting DTLS role to active.
      ConnectionRole role = (options.prefer_passive_role)
                                ? CONNECTIONROLE_PASSIVE
                                : CONNECTIONROLE_ACTIVE;

      if (!SetSecurityInfo(desc.get(), role)) {
        return NULL;
      }
    }
  } else if (require_transport_attributes && secure_ == SEC_REQUIRED) {
    // We require DTLS, but the other side didn't offer it. Fail.
    RTC_LOG(LS_WARNING) << "Failed to create TransportDescription answer "
                           "because of incompatible security settings";
    return NULL;
  }

  return desc;
}
```



## 10. ??? MediaSessionDescriptionFactory.CreateMediaContentOffer

创建VideoContentDescription

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



### MediaSessionDescriptionFactory.AddStreamParams

创建StreamParams

```cpp
template <class C>
static bool AddStreamParams(
    const std::vector<SenderOptions>& sender_options,
    const std::string& rtcp_cname,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescriptionImpl<C>* content_description) {
  // SCTP streams are not negotiated using SDP/ContentDescriptions.
  if (IsSctpProtocol(content_description->protocol())) {
    return true;
  }

  const bool include_rtx_streams =
      ContainsRtxCodec(content_description->codecs());

  const bool include_flexfec_stream =
      ContainsFlexfecCodec(content_description->codecs());

  for (const SenderOptions& sender : sender_options) {
    // groupid is empty for StreamParams generated using
    // MediaSessionDescriptionFactory.
    StreamParams* param =
        GetStreamByIds(*current_streams, "" /*group_id*/, sender.track_id);
    if (!param) {
      // This is a new sender.
      StreamParams stream_param =
          sender.rids.empty()
              ?
              // Signal SSRCs and legacy simulcast (if requested).
              CreateStreamParamsForNewSenderWithSsrcs(
                  sender, rtcp_cname, include_rtx_streams,
                  include_flexfec_stream, ssrc_generator)
              :
              // Signal RIDs and spec-compliant simulcast (if requested).
              CreateStreamParamsForNewSenderWithRids(sender, rtcp_cname);

      content_description->AddStream(stream_param);

      // Store the new StreamParams in current_streams.
      // This is necessary so that we can use the CNAME for other media types.
      current_streams->push_back(stream_param);
    } else {
      // Use existing generated SSRCs/groups, but update the sync_label if
      // necessary. This may be needed if a MediaStreamTrack was moved from one
      // MediaStream to another.
      param->set_stream_ids(sender.stream_ids);
      content_description->AddStream(*param);
    }
  }
  return true;
}
```



#### MediaSessionDescriptionFactory.CreateStreamParamsForNewSenderWithSsrcs

![ssrc1](create-offer-2.assets/ssrc1.png)

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



#### StreamParams::GenerateSsrcs——生成ssrc

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
```





### CreateContentOffer

```cpp
// Create a media content to be offered for the given |sender_options|,
// according to the given options.rtcp_mux, session_options.is_muc, codecs,
// secure_transport, crypto, and current_streams. If we don't currently have
// crypto (in current_cryptos) and it is enabled (in secure_policy), crypto is
// created (according to crypto_suites). The created content is added to the
// offer.
static bool CreateContentOffer(
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const SecurePolicy& secure_policy,
    const CryptoParamsVec* current_cryptos,
    const std::vector<std::string>& crypto_suites,
    const RtpHeaderExtensions& rtp_extensions,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescription* offer) {
  offer->set_rtcp_mux(session_options.rtcp_mux_enabled);
  if (offer->type() == cricket::MEDIA_TYPE_VIDEO) {
    offer->set_rtcp_reduced_size(true);
  }

  // Build the vector of header extensions with directions for this
  // media_description's options.
  RtpHeaderExtensions extensions;
  for (auto extension_with_id : rtp_extensions) {
    for (const auto& extension : media_description_options.header_extensions) {
      if (extension_with_id.uri == extension.uri) {
        // TODO(crbug.com/1051821): Configure the extension direction from
        // the information in the media_description_options extension
        // capability.
        extensions.push_back(extension_with_id);
      }
    }
  }
  offer->set_rtp_header_extensions(extensions);

  AddSimulcastToMediaDescription(media_description_options, offer);

  if (secure_policy != SEC_DISABLED) {
    if (current_cryptos) {
      AddMediaCryptos(*current_cryptos, offer);
    }
    if (offer->cryptos().empty()) {
      if (!CreateMediaCryptos(crypto_suites, offer)) {
        return false;
      }
    }
  }

  if (secure_policy == SEC_REQUIRED && offer->cryptos().empty()) {
    return false;
  }
  return true;
}

```





## 11. !!!关于codec 的协商

- 7.3 MediaSessionDescriptionFactory::GetCodecsForOffer
  这里是根据 本地`engine all_video_codecs_`(发送+接收);

- 8 MediaSessionDescriptionFactory::AddVideoContentForOffer
  根据 RtpTransceiverDirection 计算得到 engine 的codec能力，返回的就以下几种情况，

  

  | 本地方向direct               | 返回的codecs           | 远端方向                     |
  | ---------------------------- | ---------------------- | ---------------------------- |
  | kSendOnly                    | video_send_codecs_     | kRecvOnly/kSendRecv          |
  | kRecvOnly                    | video_recv_codecs_     | kSendOnly/kSendRecv          |
  | kSendRecv/kInactive/kStopped | video_sendrecv_codecs_ | kSendRecv/kInactive/kStopped |
  | kSendRecv/kInactive/kStopped | video_recv_codecs_     | kSendOnly                    |
  | kSendRecv/kInactive/kStopped | video_send_codecs_     | kRecvOnly                    |

  

  接着，video_codecs + supported_video_codecs 取交集；
  supported_video_codecs 就是根据方向计算出来的codec能力；
  video_codecs 就是第一条中计算得到的codec；



## 12. 关于RtpTransceiverDirection

- 两端的 direct 方向要相反，对方才能收到数据；
  远端发送 kSend，本端kRecv，则本端就能收到数据；

  ```js
  	// offer_recv && wants_send， 远端支持接收，本地支持发送，本地才支持发送
    // offer_send && wants_recv， 远端支持接收，本地支持发送，本地才支持接收
  ```



## 参考

[webrtc 的 CreateOffer 过程分析](https://blog.csdn.net/zhuiyuanqingya/article/details/84314487)

[WebRTC源码分析-呼叫建立过程之五(创建Offer，CreateOffer，上篇)](https://blog.csdn.net/ice_ly000/article/details/105763753)



## QA

1. 这是需要说明下OfferToReceiveAudio，OfferToReceiveVideo？？？

2. raw_packetization_for_video？？？？

   bundle_enabled？？？ 作用 audio/video/data 打包一起发送

3. ContentInfos，ContentInfo是什么
   【章节SessionDescription.AddContent】生成了ContentInfo，并加入向量ContentInfos中
   一个mline 对应一个ContentInfo，其中存了MediaContentDescription

4. current_local_description(),local_contents= local_description() 区别???

5. IceParameters（用于ICE过程的ufrag、pwd等信息）、StreamParams（每个媒体源的参数，包括id(即track id)、ssrcs、ssrc_groups、cname等）、音视频数据的编码器信息（编码器的id、name、时钟clockrate、编码参数表params、反馈参数feedback_params）、Rtp扩展头信息（uri、id、encrypt）

6. Ssrc 是什么时候产生的
   参考【章节3.6.1.6】

7. m-line 和MediaSessionDecroption 是什么关系
   m-line 对应 一个MediaSessionDescrptions

   一个Track对应一个RtpTransceiver，实质上在SDP中一个track就会对应到一个m-line
   参考【章节3.4.3】SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer()中

8. WebRtcSessionDescriptionFactory 是什么创建的
   在 PeerConnection::Initialize()  时候创建了WebRtcSessionDescriptionFactory
   【参考】3.6.1.2 构造函数的说明

9. MediaSessionDescriptionFactory是什么时候创建的
   是在WebRtcSessionDescriptionFactory 构造函数的时候创建

10. 什么时候创建了sdp
    【章节3.6.1】

11. 本地的codecs 是怎么获取到的
    是在MediaSessionDescriptionFactory.MediaSessionDescriptionFactory构造函数从ChanelManager中获取到的
    可以参考【章节3.6.1.2】

12. Transceiver 什么是设置set_mline_index
    【章节3.4.3】SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer()中

13. transceiver setDirect 是做什么用的

14. raw_packetization_for_video 什么作用

15. MediaSessionOptions 关系 MediaDescriptionOptions
    【章节3.4】【章节3.4.1】

16. MediaSessionOptions，MediaDescriptionOptions，SessionDescription，MediaContentDescription，ContentInfo，VideoContentDescription，JsepSessionDescription

    MediaContentDescription 是VideoContentDescription 的父类

    ContentInfo 包含了VideoContentDescription，mid等

    MediaSessionOptions 包含了MediaDescriptionOptions（vector）

    SessionDescription 包含了ContentInfo（vector）

    JsepSessionDescription 包含了 SessionDescription


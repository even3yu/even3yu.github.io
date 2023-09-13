---
layout: post
title: webrtc create answer-2
date: 2023-09-12 23:11:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


## 7. MediaSessionDescriptionFactory::CreateAnswer

pc/media_session.cc

根据MediaSessionOptions创建SessionDescription,为每个mLine创建对应的新的ContentInfo结构体。

```cpp
// - session_options 是 4.SdpOfferAnswerHandler.GetOptionsForAnswer准备好的
// - offer 是 remote sdp， setRemoteDescription
// - current_description 是当前的会话描述内容，如果是第一次 CreateAnswer ，这个值为 nullptr，
// 如果中途因为某些原因需要再次协商会话描述信息，这个值就是有意义的。
std::unique_ptr<SessionDescription>
MediaSessionDescriptionFactory::CreateAnswer(
    const SessionDescription* offer,
    const MediaSessionOptions& session_options,
    const SessionDescription* current_description) const {
   // 1. 从已被应用的offer 和 当前MediaSessionOptions中抽取一些信息，
  //    以便后续为每个mLine创建对应的新的ContentInfo结构体
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
  // 获取到当前current_description在正常使用的mLine
  // ！！！！！！！
  std::vector<const ContentInfo*> current_active_contents;
  if (current_description) {
    current_active_contents =
        GetActiveContents(*current_description, session_options);
  }

  // 1.4 从激活(不是stop或者reject)的ContentInfo获取mLine的StreamParams，
  //    注意一个mLine对应一个ContentInfo，一个ContentInfo可能含有多个StreamParams
  //  	typedef std::vector<StreamParams> StreamParamsVec;
  StreamParamsVec current_streams =
      GetCurrentStreamParams(current_active_contents);

  // 1.5 从激活(不是stop或者reject)的ContentInfo中获取媒体编码器信息
  // 1.5.1 current_active_contents 第一的时候，是null，
	// 			 从本地的engine中获取到所有支持的codec；
	// 			 从远端SessionDescription* offer 中获取到远端的codec；
	// 			 然后两者取交集；
  AudioCodecs answer_audio_codecs;
  VideoCodecs answer_video_codecs;
  RtpDataCodecs answer_rtp_data_codecs;
  GetCodecsForAnswer(current_active_contents, *offer, &answer_audio_codecs,
                     &answer_video_codecs, &answer_rtp_data_codecs);

  // 1.5.2 根据session_options的信息对编码器进行过滤处理
  //			 去除codec， const char kComfortNoiseCodecName[] = "CN";
  //			 舒适噪音
  if (!session_options.vad_enabled) {
    // If application doesn't want CN codecs in answer.
    StripCNCodecs(&answer_audio_codecs);
  }

  // --------------------------------
  // --------------------------------
  // --------------------------------
  // 2. 为每个mline创建对应的ContentInfo，添加到SessionDescription
  // 2.1 创建SessionDescription对象
  auto answer = std::make_unique<SessionDescription>();

  // 2.2 如果remote sdp 支持 bundle， 后面会把answer_bundle 添加 addGroup
  const ContentGroup* offer_bundle = offer->GetGroupByName(GROUP_TYPE_BUNDLE);
  // 2.3 创建 answer bundle
  ContentGroup answer_bundle(GROUP_TYPE_BUNDLE);
  // Transport info shared by the bundle group.
  // 2.4 默认bundle_transport是空的，第一个bundle为空，如果>1个，则会创建TransportInfo
  std::unique_ptr<TransportInfo> bundle_transport;

  answer->set_extmap_allow_mixed(offer->extmap_allow_mixed());

  // 2.5 迭代MediaSessionOptions中的每个MediaDescriptionOptions，创建Conteninfo，并添加到
  //     新建SessionDescription对象
  // 2.5.1 循环迭代， msection_index 就是 mid， 一直累加
  size_t msection_index = 0;
  for (const MediaDescriptionOptions& media_description_options :
       session_options.media_description_options) {
    // 下面AddXXXContentForAnswer 用到
    const ContentInfo* offer_content = &offer->contents()[msection_index];
    ...

    // 2.5.2 获取当前ContentInfo
    //       要么存在于当前的offer sdp中，则从当前的offer sdp中获取即可
    //       要么是新加入的媒体，还没有ContentInfo，因此为空
    const ContentInfo* current_content = nullptr;
    if (current_description &&
        msection_index < current_description->contents().size()) {
      // 从上次的offer中获取到current_content
      current_content = &current_description->contents()[msection_index];
    }
    RtpHeaderExtensions header_extensions = RtpHeaderExtensionsFromCapabilities(
        UnstoppedRtpHeaderExtensionCapabilities(
            media_description_options.header_extensions));

    // 2.5.3 根据媒体类别，分别调用不同的方法创建ContentInfo，并添加到SessionDescription
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
        ...
        break;
      case MEDIA_TYPE_UNSUPPORTED:
        ...
        break;
      default:
        RTC_NOTREACHED();
    }
    ++msection_index;
    // See if we can add the newly generated m= section to the BUNDLE group in
    // the answer.
    // AddXXXContentForAnswer的时候，answer中添加了ContentInfo
    // 取出新增的ContentInfo
    ContentInfo& added = answer->contents().back();
    // 
    // added.rejected， ContentInfo没有rejected
    // session_options.bundle_enabled， local 绑定
    // offer_bundle，远端 ContentGroup bundle
    // offer_bundle->HasContentName(added.name)，
    //
    // added.name 其实就是mid， mline index
    if (!added.rejected && session_options.bundle_enabled && offer_bundle &&
        offer_bundle->HasContentName(added.name)) {
      answer_bundle.AddContentName(added.name);
      // 创建 TransportInfo
      bundle_transport.reset(
          new TransportInfo(*answer->GetTransportInfoByName(added.name)));
    }
  }// !end for

  // 如果remote sdp 有bundle，则answer 也需要add group
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

  ...

  return answer;
}
```

1. GetActiveContents， current_description就是上一次setLocalDescription 保存；
   从current_description 获取到获取到在正常使用的mLine（content.rejected = fasle&& media_options.stopped = false）；

2. 从1的中过滤出来ContentInfos，根据ContentInfo获取mLine的StreamParams，GetCurrentStreamParams；
   
   > 注意一个mLine对应一个ContentInfo，一个ContentInfo可能含有多个StreamParams。

3. GetCodecsForAnswer， 根据 本地从engine获取到的codec（MediaSessionDescriptionFactory 构造函数）和 从remote description获取到的 codec，取交集；

4. 创建SessionDescription，利用上面步骤提供的信息 && MediaSessionOptions提供的信息为每个mline创建对应的ContentInfo，添加到SessionDescription。`AddAudioContentForAnswer`， `AddVideoContentForAnswer`；
   为每个m line 创建新的TransportInfo；TransportInfo？？？？？为什么初始化的是空，后面就会每个mline都要重新创建TransportInfo；

5. 如果remote offer 有bundle，则 answer 也需要bundle， `answer->AddGroup(answer_bundle);`



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

从上次 协商的sdp信息中，和当前的MediaSessionOptions，获取到在正常使用的mLine。



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



### 7.3 MediaSessionDescriptionFactory::GetCodecsForAnswer

获取音视频数据的所支持的编码

```cpp
 AudioCodecs offer_audio_codecs; // 存放的是最终的codec
 VideoCodecs offer_video_codecs;
 DataCodecs offer_data_codecs;
```

```cpp
// Getting codecs for an answer involves these steps:
//
// 1. Construct payload type -> codec mappings for current description.
// 2. Add any codecs from the offer that weren't already present.
// 3. Add any remaining codecs that weren't already present.
// 4. For each individual media description (m= section), filter codecs based
//    on the directional attribute (happens in another method).

// current_active_contents 是空的
// remote_offer 就是setRemoteDescription 设置
void MediaSessionDescriptionFactory::GetCodecsForAnswer(
    const std::vector<const ContentInfo*>& current_active_contents,
    const SessionDescription& remote_offer,
    AudioCodecs* audio_codecs,
    VideoCodecs* video_codecs,
    RtpDataCodecs* rtp_data_codecs) const {
  // First - get all codecs from the current description if the media type
  // is used. Add them to |used_pltypes| so the payload type is not reused if a
  // new media type is added.
  // 这里没值, current_active_contents 是
  UsedPayloadTypes used_pltypes;
  MergeCodecsFromDescription(current_active_contents, audio_codecs,
                             video_codecs, rtp_data_codecs, &used_pltypes);

  // Second - filter out codecs that we don't support at all and should ignore.
  AudioCodecs filtered_offered_audio_codecs;
  VideoCodecs filtered_offered_video_codecs;
  RtpDataCodecs filtered_offered_rtp_data_codecs;
    // remote_offer.contents() 有两个ContentInfo
    // 一个是audio，一个是video
  for (const ContentInfo& content : remote_offer.contents()) {

    // 从本地的engine中获取到所有支持的codec
    // 从远端remote sdp 中获取到远端的codec
    // 然后两者取交集， 结果存放在filtered_offered_video_codecs
    if (IsMediaContentOfType(&content, MEDIA_TYPE_AUDIO)) {
      const AudioContentDescription* audio =
          content.media_description()->as_audio();
      for (const AudioCodec& offered_audio_codec : audio->codecs()) {
        // 1. 以 filtered_offered_audio_codecs 为主；
				// filtered_offered_audio_codecs就是当前循环过滤出来的 codec
        // 从remote_offer.contents()->codecs()找到offered_audio_codec
				// ！！！！主要作用是防止重复！！！！！
        // 
        // 2. 以 all_audio_codecs_（是在MediaSessionDescriptionFactory
        // 构造的时候从channelManager计算得到） 为主；
        // 从remote_offer.contents()->codecs()找到offered_audio_codec
        // 
        // 最后，（1）找不到offered_audio_codec
        // （2）找到offered_audio_codec
        // 则添加到filtered_offered_audio_codecs
        if (!FindMatchingCodec<AudioCodec>(audio->codecs(),
                                           filtered_offered_audio_codecs,
                                           offered_audio_codec, nullptr) &&
            FindMatchingCodec<AudioCodec>(audio->codecs(), all_audio_codecs_,
                                          offered_audio_codec, nullptr)) {
          filtered_offered_audio_codecs.push_back(offered_audio_codec);
        }
      }
    } else if (IsMediaContentOfType(&content, MEDIA_TYPE_VIDEO)) {
      const VideoContentDescription* video =
          content.media_description()->as_video();
      for (const VideoCodec& offered_video_codec : video->codecs()) {
        if (!FindMatchingCodec<VideoCodec>(video->codecs(),
                                           filtered_offered_video_codecs,
                                           offered_video_codec, nullptr) &&
            FindMatchingCodec<VideoCodec>(video->codecs(), all_video_codecs_,
                                          offered_video_codec, nullptr)) {
          filtered_offered_video_codecs.push_back(offered_video_codec);
        }
      }
    } else if (IsMediaContentOfType(&content, MEDIA_TYPE_DATA)) {
      ...
    }
  }

  // Add codecs that are not in the current description but were in
  // |remote_offer|.
  // filtered_offered_audio_codecs 远端和本地计算交集后的结果
  // audio_codecs 就是合并存放的结果
  MergeCodecs<AudioCodec>(filtered_offered_audio_codecs, audio_codecs,
                          &used_pltypes);
  MergeCodecs<VideoCodec>(filtered_offered_video_codecs, video_codecs,
                          &used_pltypes);
  MergeCodecs<DataCodec>(filtered_offered_rtp_data_codecs, rtp_data_codecs,
                         &used_pltypes);
}
```

1. 如果 current_active_contents 不为空，也就是不是第一次执行 CreateAnswer ，那么执行 MergeCodecsFromDescription ，将current_active_contents 中记录的编码信息存入 xxx_codecs (audio_codec)；
2. 从本地的engine中获取到所有支持的codec， 就是`all_video_codecs_或者all_audio_codecs_`;
   从远端remote sdp 中获取到远端的codec；
   然后两者取交集， 结果存放在filtered_offered_audio_codecs，filtered_offered_xxx_codecs；
3. 执行 MergeCodecs，将2 中的 filtered_offered_audio_codecs 和 1 中的audio_codec 合并存放到 audio_codec；

![all_engine_codec]({{ site.url }}{{ site.baseurl }}/images/create-answer-2.assets/remote & all_engine_codec.jpg)

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

### 7.7 TransportInfo::TransportInfo

p2p/base/transport_info.h

```cpp
// A TransportInfo is NOT a transport-info message.  It is comparable
// to a "ContentInfo". A transport-infos message is basically just a
// collection of TransportInfos.
struct TransportInfo {
  TransportInfo() {}

  TransportInfo(const std::string& content_name,
                const TransportDescription& description)
      : content_name(content_name), description(description) {}

  std::string content_name;
  TransportDescription description;
};

//-------------------------
typedef std::vector<TransportInfo> TransportInfos;
```



#### TransportDescription

p2p/base/transport_description.h

```cpp
struct TransportDescription {
  ...
  std::vector<std::string> transport_options;
  std::string ice_ufrag;
  std::string ice_pwd;
  IceMode ice_mode;
  ConnectionRole connection_role;

  std::unique_ptr<rtc::SSLFingerprint> identity_fingerprint;
}
```



### 7.8 MediaSessionDescriptionFactory.UpdateTransportInfoForBundle

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



### 7.9 MediaSessionDescriptionFactory.UpdateCryptoParamsForBundle

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
bool MediaSessionDescriptionFactory::AddVideoContentForAnswer(
    const MediaDescriptionOptions& media_description_options, // local，对应m line
    const MediaSessionOptions& session_options, // local, 在getOptionForAnswer收集的数据
    const ContentInfo* offer_content, // remote ContentInfo
    const SessionDescription* offer_description,// remote SessionDescription
    const ContentInfo* current_content, // 上一次answer的ContentInfo
    const SessionDescription* current_description, // 上一次answer的SessionDescription
    const TransportInfo* bundle_transport, // 第一次的为null，有了一个后会每次新建一个
    const VideoCodecs& video_codecs, // 远端和本地codec 取交集生成
    const RtpHeaderExtensions& default_video_rtp_header_extensions,
    StreamParamsVec* current_streams, // 上一次answer的 stream parsms
    SessionDescription* answer, // answer SessionDescription， 就是要填充的对象
    IceCredentialsIterator* ice_credentials) const {
  ...
  // 获取remote sdp 的 VideoContentDescription
  const VideoContentDescription* offer_video_description =
      offer_content->media_description()->as_video();

  // 1. 创建 TransportDescription
  std::unique_ptr<TransportDescription> video_transport = CreateTransportAnswer(
      media_description_options.mid, offer_description,
      media_description_options.transport_options, current_description,
      bundle_transport != nullptr, ice_credentials);
  if (!video_transport) {
    return false;
  }

  // local 的offer_rtd和remote的wants_rtd，---》answer_rtd
  auto wants_rtd = media_description_options.direction;
  auto offer_rtd = offer_video_description->direction();
  // 协商RtpTransceiverDirection
  auto answer_rtd = NegotiateRtpTransceiverDirection(offer_rtd, wants_rtd);
  // 根据remot的offer_rtd和answer_rtd 得到codec
  VideoCodecs supported_video_codecs =
      GetVideoCodecsForAnswer(offer_rtd, answer_rtd);

  VideoCodecs filtered_codecs;

  if (!media_description_options.codec_preferences.empty()) {
    // 根据codec_preferences过滤
    filtered_codecs = MatchCodecPreference(
        media_description_options.codec_preferences, supported_video_codecs);
  } else {
    // 根据上一次answer的 ContentInfo，过滤出codec
    if (current_content && !current_content->rejected &&
        current_content->name == media_description_options.mid) {

      // 根据上一次answer的 ContentInfo 得到VideoContentDescription；
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
    // supported_video_codecs 是从本地的codec 过滤出来的
    for (const VideoCodec& codec : supported_video_codecs) {
      if (FindMatchingCodec<VideoCodec>(supported_video_codecs, video_codecs,
                                        codec, nullptr) &&
          !FindMatchingCodec<VideoCodec>(supported_video_codecs,
                                         filtered_codecs, codec, nullptr)) {
        // We should use the local codec with local parameters and the codec id
        // would be correctly mapped in |NegotiateCodecs|.
        filtered_codecs.push_back(codec);
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

  // bundle mid
  bool bundle_enabled = offer_description->HasGroup(GROUP_TYPE_BUNDLE) &&
                        session_options.bundle_enabled;

  /////////////////
  //!!!!!!!! VideoContentDescription !!!!!!!!!!!!!!!!
  /////////////////
  auto video_answer = std::make_unique<VideoContentDescription>();
  
  // 媒体数据加密方式，不加密，sdes，dtls
  // bool secure() const { return identity_fingerprint != nullptr; }
  // 安全策略
  cricket::SecurePolicy sdes_policy =
      video_transport->secure() ? cricket::SEC_DISABLED : secure();
  if (!SetCodecsInAnswer(offer_video_description, filtered_codecs,
                         media_description_options, session_options,
                         ssrc_generator_, current_streams,
                         video_answer.get())) {
    return false;
  }
  if (!CreateMediaContentAnswer(
          offer_video_description, media_description_options, session_options,
          sdes_policy, GetCryptos(current_content),
          filtered_rtp_header_extensions(default_video_rtp_header_extensions),
          ssrc_generator_, enable_encrypted_rtp_header_extensions_,
          current_streams, bundle_enabled, video_answer.get())) {
    return false;  // Failed the sessin setup.
  }
  bool secure = bundle_transport ? bundle_transport->description.secure()
                                 : video_transport->secure();
  bool rejected = media_description_options.stopped ||
                  offer_content->rejected ||
                  !IsMediaProtocolSupported(MEDIA_TYPE_VIDEO,
                                            video_answer->protocol(), secure);
  if (!AddTransportAnswer(media_description_options.mid,
                          *(video_transport.get()), answer)) {
    return false;
  }

  if (!rejected) {
    video_answer->set_bandwidth(kAutoBandwidth);
  } else {
    RTC_LOG(LS_INFO) << "Video m= section '" << media_description_options.mid
                     << "' being rejected in answer.";
  }
  answer->AddContent(media_description_options.mid, offer_content->type,
                     rejected, std::move(video_answer));
  return true;
}
```

- 创建TransportDescription，
- 根据RtpTransceiverDirection，GetVideoCodecsForAnswer 来获取codec
- **过滤 codec， GetVideoCodecsForAnswer 来获取codec + 7.3 计算得到codec 取交集；具体参考【11章节】**
- 创建VideoContentDescription
- 安全套件 `session_options.crypto_options`
- **添加codec，SetCodecsInAnswer， 这里又协商了codec，具体参考【11章节】**
- 添加stream,StreamParams

### !!! ContentInfo::rejected

pc/session_description.h
这个值是 根据`media_description_options.stopped` 来确定的。

### 8.1 --MediaSessionDescriptionFactory::CreateTransportAnswer

pc/media_session.cc



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

![engine-codc-of-direct]({{ site.url }}{{ site.baseurl }}/images/create-answer-2.assets/engine-codec-of-direct.jpg)



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



### 8.3 MediaSessionDescriptionFactory::GetVideoCodecsForAnswer

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
```



```cpp
const VideoCodecs& MediaSessionDescriptionFactory::GetVideoCodecsForAnswer(
    const RtpTransceiverDirection& offer,
    const RtpTransceiverDirection& answer) const {
  switch (answer) {
    // For inactive and sendrecv answers, generate lists as if we were to accept
    // the offer's direction. See RFC 3264 Section 6.1.
    case RtpTransceiverDirection::kSendRecv:
    case RtpTransceiverDirection::kStopped:
    case RtpTransceiverDirection::kInactive:
      return GetVideoCodecsForOffer(
          webrtc::RtpTransceiverDirectionReversed(offer));
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



![negotiate-direct]({{ site.url }}{{ site.baseurl }}/images/create-answer-2.assets/negotiate-direct.jpg)

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



#### MediaSessionDescriptionFactory::GetVideoCodecsForOffer

pc/media_session.cc

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



### 8.6 !!!MediaSessionDescriptionFactory.SetCodecsInAnswer

pc/media_session.cc

```cpp
template <class C>
static bool SetCodecsInAnswer(
    const MediaContentDescriptionImpl<C>* offer,
    const std::vector<C>& local_codecs, // 8.3 计算出来的codec
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescriptionImpl<C>* answer) {
  
  // 添加codec 到MediaContentDescription
  std::vector<C> negotiated_codecs;
  // local_codecs 8.3 计算出来的codec
  // offer->codecs() 就是remote codec
  NegotiateCodecs(local_codecs, offer->codecs(), &negotiated_codecs,
                  media_description_options.codec_preferences.empty());
  answer->AddCodecs(negotiated_codecs);
  // protocal
  // |protocol| is the expected media transport protocol, such as RTP/AVPF,
  // RTP/SAVPF or SCTP/DTLS.
  // SAVPF（SRTP AUDIO VIDEO Profile ）
  answer->set_protocol(offer->protocol());
  
  if (!AddStreamParams(media_description_options.sender_options,
                       session_options.rtcp_cname, ssrc_generator,
                       current_streams, answer)) {
    return false;  // Something went seriously wrong.
  }
  return true;
}
```



#### 8.6.1 ???MediaSessionDescriptionFactory.AddStreamParams

```cpp
// Adds a StreamParams for each SenderOptions in |sender_options| to
// content_description.
// |current_params| - All currently known StreamParams of any media type.
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



### 8.7 --MediaSessionDescriptionFactory.CreateMediaContentAnswer

pc/media_session.cc

【章节10】

### 8.8 MediaSessionDescriptionFactory::AddTransportAnswer

pc/media_session.cc

```cpp
bool MediaSessionDescriptionFactory::AddTransportAnswer(
    const std::string& content_name,
    const TransportDescription& transport_desc,
    SessionDescription* answer_desc) const {
  answer_desc->AddTransportInfo(TransportInfo(content_name, transport_desc));
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
static bool CreateMediaContentAnswer(
    const MediaContentDescription* offer,
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const SecurePolicy& sdes_policy,
    const CryptoParamsVec* current_cryptos,
    const RtpHeaderExtensions& local_rtp_extensions,
    UniqueRandomIdGenerator* ssrc_generator,
    bool enable_encrypted_rtp_header_extensions,
    StreamParamsVec* current_streams,
    bool bundle_enabled,
    MediaContentDescription* answer) {
  answer->set_extmap_allow_mixed_enum(offer->extmap_allow_mixed_enum());
  RtpHeaderExtensions negotiated_rtp_extensions;
  NegotiateRtpHeaderExtensions(
      local_rtp_extensions, offer->rtp_header_extensions(),
      enable_encrypted_rtp_header_extensions, &negotiated_rtp_extensions);
  answer->set_rtp_header_extensions(negotiated_rtp_extensions);

  answer->set_rtcp_mux(session_options.rtcp_mux_enabled && offer->rtcp_mux());
  if (answer->type() == cricket::MEDIA_TYPE_VIDEO) {
    answer->set_rtcp_reduced_size(offer->rtcp_reduced_size());
  }

  answer->set_remote_estimate(offer->remote_estimate());

  if (sdes_policy != SEC_DISABLED) {
    CryptoParams crypto;
    if (SelectCrypto(offer, bundle_enabled, session_options.crypto_options,
                     &crypto)) {
      if (current_cryptos) {
        FindMatchingCrypto(*current_cryptos, crypto, &crypto);
      }
      answer->AddCrypto(crypto);
    }
  }

  if (answer->cryptos().empty() && sdes_policy == SEC_REQUIRED) {
    return false;
  }

  AddSimulcastToMediaDescription(media_description_options, answer);

  answer->set_direction(NegotiateRtpTransceiverDirection(
      offer->direction(), media_description_options.direction));

  return true;
}
```



### SelectCrypto







## 11. !!!关于codec 的协商

- 7.3 MediaSessionDescriptionFactory::GetCodecsForAnswer
  这里是根据 remote codec 和 本地`engine all_video_codecs_`(发送+接收)，计算交集，得到codec;

- 8 MediaSessionDescriptionFactory::AddVideoContentForOffer

  8.3 MediaSessionDescriptionFactory::GetVideoCodecsForAnswer
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

- 8.6 !!!MediaSessionDescriptionFactory.SetCodecsInAnswer

  第二条计算的结果  和 remote codec 取交集 得到codec



## 12. 关于RtpTransceiverDirection

- 两端的 direct 方向要相反，对方才能收到数据；
  远端发送 kSend，本端kRecv，则本端就能收到数据；

  ```js
  	// offer_recv && wants_send， 远端支持接收，本地支持发送，本地才支持发送
    // offer_send && wants_recv， 远端支持接收，本地支持发送，本地才支持接收
  ```

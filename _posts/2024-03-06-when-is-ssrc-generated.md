---
layout: post
title: ssrc 是什么时候生成
date: 2024-03-06 19:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq
---


* content
{:toc}

---


ssrc是在SenderOptions 转 StreamParams的时候。

生成ssrc是在`createOffer`的时候，就已经生了，但是只是一个随机数字，真正起作用的是在`SetLocalDescription` 才会起作用。

## 1. 调用堆栈

pc/media_session.cc

```less
SdpOfferAnswerHandler::DoCreateOffer
WebRtcSessionDescriptionFactory::CreateOffer
WebRtcSessionDescriptionFactory::InternalCreateOffer
MediaSessionDescriptionFactory::CreateOffer

MediaSessionDescriptionFactory::AddAudioContentForOffer
/MediaSessionDescriptionFactory::AddVideoContentForOffer

MediaSessionDescriptionFactory::CreateMediaContentOffer
MediaSessionDescriptionFactory::AddStreamParams

MediaSessionDescriptionFactory::CreateStreamParamsForNewSenderWithSsrcs
/MediaSessionDescriptionFactory::CreateStreamParamsForNewSenderWithRids
```



```less
SdpOfferAnswerHandler::DoCreateAnswer
WebRtcSessionDescriptionFactory::CreateAnswer
WebRtcSessionDescriptionFactory::InternalCreateAnswer
MediaSessionDescriptionFactory::CreateAnswer

MediaSessionDescriptionFactory::AddAudioContentForAnswer
/MediaSessionDescriptionFactory::AddVideoContentForAnswer

MediaSessionDescriptionFactory::SetCodecsInAnswer
MediaSessionDescriptionFactory::AddStreamParams

MediaSessionDescriptionFactory::CreateStreamParamsForNewSenderWithSsrcs
/MediaSessionDescriptionFactory::CreateStreamParamsForNewSenderWithRids
```





## 2. !!! 转换时机 MediaSessionDescriptionFactory.AddStreamParams

pc/media_session.cc

把`std::vector<SenderOptions>`转换为`StreamParamsVec`。
每个Transceiver 对应一个`SenderOptions`，对一个个`StreamParams`。这时候会生成ssrc。

```cpp
//  std::vector<SenderOptions>& sender_options 在 unified plan 中只有一个元素
template <class C>
static bool AddStreamParams(
    const std::vector<SenderOptions>& sender_options, // SenderOptions
    const std::string& rtcp_cname,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams, // StreamParamsVec
    MediaContentDescriptionImpl<C>* content_description) {
  // SCTP streams are not negotiated using SDP/ContentDescriptions.
  if (IsSctpProtocol(content_description->protocol())) {
    return true;
  }

  // 0. 根据sdp 中codecs 来判断 是否有 rtx 流
  const bool include_rtx_streams =
      ContainsRtxCodec(content_description->codecs());

  // 根据sdp 中codecs 来判断 是否有 flex fec 流
  const bool include_flexfec_stream =
      ContainsFlexfecCodec(content_description->codecs());

  // 1. 遍历sender_options
  for (const SenderOptions& sender : sender_options) {
    // groupid is empty for StreamParams generated using
    // MediaSessionDescriptionFactory.
    // 根据sender.track_id 找到对应的 StreamParams
    StreamParams* param =
      	//  StreamParamsVec* current_streams,
        GetStreamByIds(*current_streams, "" /*group_id*/, sender.track_id);
    if (!param) {
      // 2. 创建先的StreamParams
      // (1)sender.rids为空，就是rtpId， CreateStreamParamsForNewSenderWithSsrcs
      //		第一次rids为空，这是在createOffer的时候会赋值；第二次协商offer的时候，会赋值
      // (2)sender.rids 不为空， CreateStreamParamsForNewSenderWithRids
      // This is a new sender.
      //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
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
      //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

      // 3. MediaContentDescriptionImpl::AddStream， StreamParams添加到 MediaContentDescriptionImpl
      content_description->AddStream(stream_param);

      // Store the new StreamParams in current_streams.
      // This is necessary so that we can use the CNAME for other media types.
      //  StreamParamsVec* current_streams,
      current_streams->push_back(stream_param);
    } else {
      // Use existing generated SSRCs/groups, but update the sync_label if
      // necessary. This may be needed if a MediaStreamTrack was moved from one
      // MediaStream to another.
      // 如果在current_streams，就是上一次协商的offer中找到StreamParams，
      // 则直接添加到MediaContentDescriptionImpl
      param->set_stream_ids(sender.stream_ids);
      content_description->AddStream(*param);
    }
  }
  return true;
}
```

0. 根据sdp 中codecs 来判断 是否有 rtx 流，flex fec 流（red，ulpfec等的fec是不是单独流发送的，所有没有的ssrc的）;

1. 遍历sender_options ， SenderOptions数组，生成对应的 StreamParams

2. 创建先的StreamParams

   - sender.rids为空，就是rtpId， CreateStreamParamsForNewSenderWithSsrcs 第一次rids为空，这是在createOffer的时候会赋值；第二次协商offer的时候，会赋值

   - sender.rids 不为空， CreateStreamParamsForNewSenderWithRids

3. MediaContentDescriptionImpl::AddStream， StreamParams添加到 MediaContentDescriptionImpl



### 2.1 ContainsRtxCodec

codec 中是否包含“rtx”。

pc/media_session.cc

```cpp
template <class C>
static bool ContainsRtxCodec(const std::vector<C>& codecs) {
  for (const auto& codec : codecs) {
    if (IsRtxCodec(codec)) {
      return true;
    }
  }
  return false;
}
```



```cpp
// media/base/media_constants.cc
const char kRtxCodecName[] = "rtx";
//-----------------------------
template <class C>
static bool IsRtxCodec(const C& codec) {
  return absl::EqualsIgnoreCase(codec.name, kRtxCodecName);
}
```



### 2.2 ContainsFlexfecCodec

codec 中是否包含“flexfec-03”。

pc/media_session.cc

```cpp
template <class C>
static bool ContainsFlexfecCodec(const std::vector<C>& codecs) {
  for (const auto& codec : codecs) {
    if (IsFlexfecCodec(codec)) {
      return true;
    }
  }
  return false;
}
```



```cpp
// TODO(brandtr): Change this to 'flexfec' when we are confident that the
// header format is not changing anymore.
// media/base/media_constants.cc
const char kFlexfecCodecName[] = "flexfec-03";
//--------------

template <class C>
static bool IsFlexfecCodec(const C& codec) {
  return absl::EqualsIgnoreCase(codec.name, kFlexfecCodecName);
}
```



## 3. CreateStreamParamsForNewSenderWithRids

pc/media_session.cc

```cpp
static StreamParams CreateStreamParamsForNewSenderWithRids(
    const SenderOptions& sender,
    const std::string& rtcp_cname) {
  RTC_DCHECK(!sender.rids.empty());
  RTC_DCHECK_EQ(sender.num_sim_layers, 0)
      << "RIDs are the compliant way to indicate simulcast.";
  RTC_DCHECK(ValidateSimulcastLayers(sender.rids, sender.simulcast_layers));
  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
  StreamParams result;
  result.id = sender.track_id; // track id
  result.cname = rtcp_cname; // cname
  result.set_stream_ids(sender.stream_ids); // mediastream id

  // More than one rid should be signaled.
  if (sender.rids.size() > 1) { // simulcast 层级
    result.set_rids(sender.rids);
  }

  return result;
}
```

生成 StreamParams对象

## 4. CreateStreamParamsForNewSenderWithSsrcs

pc/media_session.cc

```cpp
static StreamParams CreateStreamParamsForNewSenderWithSsrcs(
    const SenderOptions& sender,
    const std::string& rtcp_cname,
    bool include_rtx_streams,
    bool include_flexfec_stream,
    UniqueRandomIdGenerator* ssrc_generator) {
   
  
  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!
  StreamParams result;
  result.id = sender.track_id; // track id

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

  // 
  result.GenerateSsrcs(sender.num_sim_layers, include_rtx_streams,
                       include_flexfec_stream, ssrc_generator);

  result.cname = rtcp_cname; // cname
  result.set_stream_ids(sender.stream_ids); // mediastream id，

  return result;
}
```

生成 StreamParams对象



### 4.1 --StreamParams::GenerateSsrcs

pc/media_session.cc



## !!! 5. StreamParams::GenerateSsrcs

1. 根据num_layers，为每个层级生成一个ssrc
2. num_layers> 1，表示是simulcast， 则生成”SIM“  ssrc_group;
3. 如果有rtx 重传流，则根据步骤1，有多少层级，就会对应生成多少道重传流，同时每道主流和重传流生成 "FID" ssrc_group；
4. 如果有flex fec FEC流，则根据步骤1，有多少层级，就会对应生成多少道FEC流，同时每道主流和FEC流生成 "FEC-FR" ssrc_group；
5. ssrc的个数 = num_layers（每个层级生成一个ssrc） + num_layers（ rtx 重传流的个数） +  num_layers（flex fec FEC流）
6. ssrc group的个数 = num_layers -1 (num_layers> 1 生成”SIM“ ) + num_layers（ rtx 重传流 都会生成一个group） +  num_layers（flex fec FEC流 都会生成一个group）

pc/media_session.cc

```cpp
void StreamParams::GenerateSsrcs(int num_layers,
                                 bool generate_fid, // rtx的codec
                                 bool generate_fec_fr, // 包含flex fec codec
                                 rtc::UniqueRandomIdGenerator* ssrc_generator) {
  RTC_DCHECK_GE(num_layers, 0);
  RTC_DCHECK(ssrc_generator);
  //  primary ssrc
  std::vector<uint32_t> primary_ssrcs;
  // 1. 根据simulcast 的层级，来确定primary_ssrc的个数
  // 同时为每个层级生成一个ssrc，相当于说每个层级都是一道独立的流
  for (int i = 0; i < num_layers; ++i) {
    uint32_t ssrc = ssrc_generator->GenerateId();
    primary_ssrcs.push_back(ssrc);
    // void add_ssrc(uint32_t ssrc) { ssrcs.push_back(ssrc); }
    add_ssrc(ssrc);
  }

  // simulcast
  // 2. 为 simulcat 生成 SsrcGroup
  if (num_layers > 1) {
    // const char kSimSsrcGroupSemantics[] = "SIM";
   	// 注意： 这里传入的是primary_ssrcs
    SsrcGroup simulcast(kSimSsrcGroupSemantics, primary_ssrcs);
    ssrc_groups.push_back(simulcast);
  }

  // 3. rtx， 包含rtx的codec，就是需要生成nack重传流
  if (generate_fid) {
    // 为每一个 primary ssrcs 生成一个配套ssrc， rtx
    for (uint32_t ssrc : primary_ssrcs) {
      AddFidSsrc(ssrc, ssrc_generator->GenerateId());
    }
  }

  // 4. fec_fr  包含rtx的codec，
  if (generate_fec_fr) {
    for (uint32_t ssrc : primary_ssrcs) {
      AddFecFrSsrc(ssrc, ssrc_generator->GenerateId());
    }
  }
}
```



### 5.1 StreamParams::AddFidSsrc

```cpp
 // Convenience function to add an FID ssrc for a primary_ssrc
  // that's already been added.
  bool AddFidSsrc(uint32_t primary_ssrc, uint32_t fid_ssrc) {
    // const char kFidSsrcGroupSemantics[] = "FID";
    return AddSecondarySsrc(kFidSsrcGroupSemantics, primary_ssrc, fid_ssrc);
  }
```



### 5.2 StreamParams::AddSecondarySsrc

对应primary_ssrc，生成secondary_ssrc， 绑定一起存放在SsrcGroup中。比如， FID

```cpp
SsrcGroup(semantics, {primary_ssrc, secondary_ssrc})
```

primary_ssrc + rtx 的secondary_ssrc；

```cpp
bool StreamParams::AddSecondarySsrc(const std::string& semantics,
                                    uint32_t primary_ssrc,
                                    uint32_t secondary_ssrc) {
  if (!has_ssrc(primary_ssrc)) {
    return false;
  }

  ssrcs.push_back(secondary_ssrc);
  // 生成SsrcGroup
  ssrc_groups.push_back(SsrcGroup(semantics, {primary_ssrc, secondary_ssrc}));
  return true;
}
```

- 根据primary_ssrc， 生成对应的第二个secondary_ssrc；
- 根据primary_ssrc，secondary_ssrc，semantics 生成SsrcGroup；



### 5.3 StreamParams::AddFecFrSsrc

```cpp
bool AddFecFrSsrc(uint32_t primary_ssrc, uint32_t fecfr_ssrc) {
	// const char kFecFrSsrcGroupSemantics[] = "FEC-FR";
  return AddSecondarySsrc(kFecFrSsrcGroupSemantics, primary_ssrc, fecfr_ssrc);
}
```


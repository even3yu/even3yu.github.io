---
layout: post
title: webrtc SenderOptions and StreamParams
date: 2023-09-29 01:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---

## 1. SenderOptions

pc/media_session.h

**多个SenderOptions，存放在vector中，多个是针对planB。**

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
- **根据RtpSenderInternal 对应生成 SenderOptions，和RtpReceiverInternal没关系；**
  **SenderOptions和RtpSenderInternal，一一对应；**
- Unified Plan 一个Transceiver 只会 有一个SenderOptions ，planB可以有多个；



### 1.1 track_id

track_id 就是 Track的唯一识别码；Unified Plan下，一个track 对应一个Transceiver；



### 1.2 ??? stream_ids

-  Plan B，只能有一个stream IDs；
- Unified Plan，0个或者多个stream IDs, **一个track 被多个stream所拥有？？？**

----------------

- RtpSenderInternal的唯一识别码，唯一识别码，用户id，或者随机生成的uuid

- PeerConnection::AddTransceiver`  ，  或者`PeerConnection::AddTrack` 参数传入， 由应用层创建；
- 作用是什么？？？
- 什么情况下有多个？？？



### 1.3 rids

rtpId



### 1.4 simulcast_layers



### 1.5 num_sim_layers



### 1.6 创建时机

#### 1.6.1 ~~planB，MediaDescriptionOptions::AddSenderInternal~~

pc/media_session.cc

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
  //！！！！！！！！！！！！！
  options.stream_ids = stream_ids;
  //！！！！！！！！！！！！！
  options.simulcast_layers = simulcast_layers;
  options.rids = rids;
  options.num_sim_layers = num_sim_layers;
  sender_options.push_back(options);
}

void MediaDescriptionOptions::AddAudioSender(
    const std::string& track_id,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK(type == MEDIA_TYPE_AUDIO);
  AddSenderInternal(track_id, stream_ids, {}, SimulcastLayerList(), 1);
}
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



#### !!! 1.6.2 Unify plan，GetMediaDescriptionOptionsForTransceiver

pc/sdp_offer_answer.cc

- CreateOffer
- CreateAnswer

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
  //!!!!!!!!!!!!!!!!
  // stream_ids 是从addTrack/addTransceiver 传入的参数
  sender_options.stream_ids = transceiver->sender()->stream_ids();
  //!!!!!!!!!!!!!!!!

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



### 1.7 !!! SenderOptions 什么情况下会有多个

- **根据RtpSenderInternal 对应生成 SenderOptions，和RtpReceiverInternal没关系；**
  **SenderOptions和RtpSenderInternal，一一对应；**
- Unified Plan 一个Transceiver 只会 有一个SenderOptions ，planB可以有多个；
- planB 的时候，会有多个情况
- Unified Plan 只有一个



## 2. StreamParams

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
  // track id
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



```cpp
struct StreamParams {
  ...
	uint32_t first_ssrc() const {
    if (ssrcs.empty()) {
      return 0;
    }

    return ssrcs[0];
  }
  bool has_ssrcs() const { return !ssrcs.empty(); }
  bool has_ssrc(uint32_t ssrc) const {
    return absl::c_linear_search(ssrcs, ssrc);
  }
  void add_ssrc(uint32_t ssrc) { ssrcs.push_back(ssrc); }
  bool has_ssrc_groups() const { return !ssrc_groups.empty(); }
  bool has_ssrc_group(const std::string& semantics) const {
    return (get_ssrc_group(semantics) != NULL);
  }
  const SsrcGroup* get_ssrc_group(const std::string& semantics) const {
    for (const SsrcGroup& ssrc_group : ssrc_groups) {
      if (ssrc_group.has_semantics(semantics)) {
        return &ssrc_group;
      }
    }
    return NULL;
  }

  // Convenience function to add an FID ssrc for a primary_ssrc
  // that's already been added.
  bool AddFidSsrc(uint32_t primary_ssrc, uint32_t fid_ssrc) {
    return AddSecondarySsrc(kFidSsrcGroupSemantics, primary_ssrc, fid_ssrc);
  }

  // Convenience function to lookup the FID ssrc for a primary_ssrc.
  // Returns false if primary_ssrc not found or FID not defined for it.
  bool GetFidSsrc(uint32_t primary_ssrc, uint32_t* fid_ssrc) const {
    return GetSecondarySsrc(kFidSsrcGroupSemantics, primary_ssrc, fid_ssrc);
  }

  // Convenience function to add an FEC-FR ssrc for a primary_ssrc
  // that's already been added.
  bool AddFecFrSsrc(uint32_t primary_ssrc, uint32_t fecfr_ssrc) {
    return AddSecondarySsrc(kFecFrSsrcGroupSemantics, primary_ssrc, fecfr_ssrc);
  }

  // Convenience function to lookup the FEC-FR ssrc for a primary_ssrc.
  // Returns false if primary_ssrc not found or FEC-FR not defined for it.
  bool GetFecFrSsrc(uint32_t primary_ssrc, uint32_t* fecfr_ssrc) const {
    return GetSecondarySsrc(kFecFrSsrcGroupSemantics, primary_ssrc, fecfr_ssrc);
  }

  // Convenience function to populate the StreamParams with the requested number
  // of SSRCs along with accompanying FID and FEC-FR ssrcs if requested.
  // SSRCs are generated using the given generator.
  void GenerateSsrcs(int num_layers,
                     bool generate_fid, // sscr
                     bool generate_fec_fr,
                     rtc::UniqueRandomIdGenerator* ssrc_generator);

  // Convenience to get all the SIM SSRCs if there are SIM ssrcs, or
  // the first SSRC otherwise.
  void GetPrimarySsrcs(std::vector<uint32_t>* ssrcs) const;

  // Convenience to get all the FID SSRCs for the given primary ssrcs.
  // If a given primary SSRC does not have a FID SSRC, the list of FID
  // SSRCS will be smaller than the list of primary SSRCs.
  void GetFidSsrcs(const std::vector<uint32_t>& primary_ssrcs,
                   std::vector<uint32_t>* fid_ssrcs) const;

  // Stream ids serialized to SDP.
  std::vector<std::string> stream_ids() const;
  void set_stream_ids(const std::vector<std::string>& stream_ids);

  // Returns the first stream id or "" if none exist. This method exists only
  // as temporary backwards compatibility with the old sync_label.
  std::string first_stream_id() const;

  std::string ToString() const;
  
  ...

  // RID functionality according to
  // https://tools.ietf.org/html/draft-ietf-mmusic-rid-15
  // Each layer can be represented by a RID identifier and can also have
  // restrictions (such as max-width, max-height, etc.)
  // If the track has multiple layers (ex. Simulcast), each layer will be
  // represented by a RID.
  bool has_rids() const { return !rids_.empty(); }
  const std::vector<RidDescription>& rids() const { return rids_; }
  void set_rids(const std::vector<RidDescription>& rids) { rids_ = rids; }
  ...
}
```



### 2.1 ~~std::string groupid~~



### 2.2 std::string id

Track ID。

### 2.3 !!! std::vector<uint32_t> ssrcs

SSRC（Synchronization Source）。SSRC是媒体源的唯一标识，每一路媒体流都有一个唯一的SSRC来标识它。

ssrc有多个情况，

- 开启Simulcast，多到流；
- nack 重传流，主流+重传流
- 同时发送两道流；这个是针对PlanB，unified plan 是一个track 对应一个transceiver，一个SenderOptions；

具体看【sdp-about-ssrc】这文章，有解释和举例相关sdp。

参考【章节2.8】。

### 2.4 std::vector\<SsrcGroup> ssrc_groups

在`pc/media_session.cc`的`CreateStreamParamsForNewSenderWithSsrcs` ， 调用`StreamParams::GenerateSsrcs` 填充ssrc和ssrc_groups。具体看【sdp-about-ssrc】这文章，有解释和举例相关sdp。
参考【章节2.8】。



### 2.5 std::string cname

rtcp_name。
通常称为别名，可以用在域名解析中。当想为某个域名起一个别名的时候，就可以使用它。如在直播推/拉流中，想将某个云厂商的推流地址push.xxx.yun.xxx.com换成你自己的地址push.advancedu.com，就可以使用cname。cname在SDP中的作用与域名解析中的作用是一致的，就是为媒体流起了一个别名。**在同一个视频媒体描述中的两个SSRC后面都跟着同样的cname，说明这两个SSRC属于同一个媒体流。**
具体看【sdp-about-ssrc】这文章，有解释和举例相关sdp。

https://blog.csdn.net/weixin_39766005/article/details/132294422



### 2.6 std::vector\<std::string> stream_ids_



### 2.7 ??? std::vector\<RidDescription> rids_



### 2.8 创建StreamParams， MediaSessionDescriptionFactory.AddStreamParams

pc/media_session.cc

把`std::vector<SenderOptions>`转换为`StreamParamsVec`。

```js
MediaSessionDescriptionFactory::AddVideoContentForOffer
MediaSessionDescriptionFactory.CreateMediaContentOffer
MediaSessionDescriptionFactory.AddStreamParams
```



```js
MediaSessionDescriptionFactory::AddVideoContentForAnswer
MediaSessionDescriptionFactory.SetCodecsInAnswer
MediaSessionDescriptionFactory.AddStreamParams
```



```cpp
//  std::vector<SenderOptions>& sender_options 在 unified plan 中只有一个元素
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

  // rtx 流
  const bool include_rtx_streams =
      ContainsRtxCodec(content_description->codecs());

  // flex fec 流
  const bool include_flexfec_stream =
      ContainsFlexfecCodec(content_description->codecs());

  // 遍历sender_options
  for (const SenderOptions& sender : sender_options) {
    // groupid is empty for StreamParams generated using
    // MediaSessionDescriptionFactory.
    // 根据sender.track_id 找到对应的 StreamParams
    StreamParams* param =
        GetStreamByIds(*current_streams, "" /*group_id*/, sender.track_id);
    if (!param) {
      // 创建先的StreamParams
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

      // MediaContentDescriptionImpl::AddStream， StreamParams添加到 MediaContentDescriptionImpl
      content_description->AddStream(stream_param);

      // Store the new StreamParams in current_streams.
      // This is necessary so that we can use the CNAME for other media types.
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



#### CreateStreamParamsForNewSenderWithSsrcs

pc/media_session.cc

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

  // 填充ssrc和ssrcgroup
  result.GenerateSsrcs(sender.num_sim_layers, include_rtx_streams,
                       include_flexfec_stream, ssrc_generator);

  result.cname = rtcp_cname;
  result.set_stream_ids(sender.stream_ids);

  return result;
}
```



#### --StreamParams::GenerateSsrcs

media/base/stream_params.cc



### 2.9 !!! StreamParams::GenerateSsrcs

media/base/stream_params.cc

```cpp
void StreamParams::GenerateSsrcs(int num_layers,
                                 bool generate_fid,
                                 bool generate_fec_fr,
                                 rtc::UniqueRandomIdGenerator* ssrc_generator) {
  RTC_DCHECK_GE(num_layers, 0);
  RTC_DCHECK(ssrc_generator);
  //  primary ssrc
  std::vector<uint32_t> primary_ssrcs;
  for (int i = 0; i < num_layers; ++i) {
    uint32_t ssrc = ssrc_generator->GenerateId();
    primary_ssrcs.push_back(ssrc);
    // void add_ssrc(uint32_t ssrc) { ssrcs.push_back(ssrc); }
    add_ssrc(ssrc);
  }

  // simulcast
  if (num_layers > 1) {
    // const char kSimSsrcGroupSemantics[] = "SIM";
   	// 注意： 这里传入的是primary_ssrcs
    SsrcGroup simulcast(kSimSsrcGroupSemantics, primary_ssrcs);
    ssrc_groups.push_back(simulcast);
  }

  // rtx 
  if (generate_fid) {
    // 为每一个 primary ssrcs 生成一个配套ssrc， rtx
    for (uint32_t ssrc : primary_ssrcs) {
      AddFidSsrc(ssrc, ssrc_generator->GenerateId());
    }
  }

  // fec_fr
  if (generate_fec_fr) {
    for (uint32_t ssrc : primary_ssrcs) {
      AddFecFrSsrc(ssrc, ssrc_generator->GenerateId());
    }
  }
}
```



#### SsrcGroup

```cpp
struct SsrcGroup {
	...
  bool has_semantics(const std::string& semantics) const;

  std::string ToString() const;

  std::string semantics;        // e.g FIX, FEC, SIM.
  std::vector<uint32_t> ssrcs;  // SSRCs of this type.
};
```



#### StreamParams::AddFidSsrc

```cpp
 // Convenience function to add an FID ssrc for a primary_ssrc
  // that's already been added.
  bool AddFidSsrc(uint32_t primary_ssrc, uint32_t fid_ssrc) {
    // const char kFidSsrcGroupSemantics[] = "FID";
    return AddSecondarySsrc(kFidSsrcGroupSemantics, primary_ssrc, fid_ssrc);
  }
```

- 生成`FID`的sscr group



#### StreamParams::AddSecondarySsrc

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



#### !!! SIM 和  FID 生成 SsrcGroup 区别

- SIM

  ```
  SsrcGroup simulcast(kSimSsrcGroupSemantics, primary_ssrcs);
  ```

  一个流，不同的分辨率的两个ssrc；

- FID

  ```cpp
  SsrcGroup(semantics, {primary_ssrc, secondary_ssrc})
  ```

  primary_ssrc + rtx 的secondary_ssrc；

- primary_ssrc 有多个
  planB，有两路视频源放到同一个视频源中



## 3. SenderOptions and StreamParams 关系

通过上文知道，StreamParams 根据SenderOptions 转化过来的。

- 有多少个SenderOptions，就对应有多少个StreamParams
- ssrcs 是属于SenderOptions， 可以有多个，比如主流+ rtx 重传流，主流+另外一道主流，simulcast的2道流；
- ssrc_group 是说明多到ssrcs 之间的关系， 比如主流和rtx重传流
- Unified Plan 一个Transceiver 只会 有一个SenderOptions ，planB可以有多个；
- track_id 就是 Track的唯一识别码；Unified Plan下，一个track 对应一个Transceiver；
- stream_ids的作用是什么？？？
---
layout: post
title: SenderOptions相关内容
date: 2024-02-23 00:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---


## 1. 定义

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

- **根据RtpSenderInternal（就是RtpSender，注意不是RTPSender） 对应生成 SenderOptions，和RtpReceiverInternal没关系；** **SenderOptions和RtpSenderInternal，一一对应；**

- **Unified Plan 一个Transceiver 只会 有一个SenderOptions ，planB可以有多个；**

  

### 1.1 track_id

track_id 就是 Track的唯一识别码；Unified Plan下，一个track 对应一个Transceiver；

### 1.2 ??? stream_ids

- Plan B，只能有一个stream IDs；
- Unified Plan，0个或者多个stream IDs, **一个track 被多个stream所拥有**

------

- RtpSenderInternal的唯一识别码，唯一识别码，用户id，或者随机生成的uuid
- PeerConnection::AddTransceiver` ， 或者`PeerConnection::AddTrack` 参数传入， 由应用层创建；
- 作用是什么？？？
- 什么情况下有多个？？？

### 1.3 rids

rtpId

### 1.4 simulcast_layers

### 1.5 num_sim_layers



## 2. 创建时机

### 2.1 unify plan, GetMediaDescriptionOptionsForTransceiver

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

  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!! 1. 一个transceiver 对应生成一个SenderOptions
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
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
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  media_description_options.sender_options.push_back(sender_options);

  return media_description_options;
}
```



1. 调用堆栈

```less
CreateAnswer/SetLocalDescription
DoCreateAnswer
GetOptionsForAnswer
GetOptionsForUnifiedPlanAnswer
GetMediaDescriptionOptionsForTransceiver

CreateOffer/SetLocalDescription
DoCreateOffer
GetOptionsForOffer
GetOptionsForUnifiedPlanOffer
GetMediaDescriptionOptionsForTransceiver
```

2. 一个transceiver 对应生成一个SenderOptions。

   ```cpp
     cricket::SenderOptions sender_options;
     sender_options.track_id = transceiver->sender()->id();
     sender_options.stream_ids = transceiver->sender()->stream_ids();
   ```

3. SenderOptions 添加到 MediaDescriptionOptions.sender_options 中



### ~~2.2 planB，AddPlanBRtpSenderOptions~~

根据RtpSenderInternal 的向量来创建SenderOptions

#### !!! AddPlanBRtpSenderOptions

pc/sdp_offer_answer.cc

调用堆栈

```less
CreateAnswer/SetLocalDescription
DoCreateAnswer
GetOptionsForAnswer
SdpOfferAnswerHandler::GetOptionsForPlanBOffer
AddPlanBRtpSenderOptions


CreateOffer/SetLocalDescription
DoCreateOffer
GetOptionsForOffer
SdpOfferAnswerHandler::GetOptionsForPlanBAnswer
AddPlanBRtpSenderOptions
```



```cpp
void AddPlanBRtpSenderOptions(
    const std::vector<rtc::scoped_refptr<
        RtpSenderProxyWithInternal<RtpSenderInternal>>>& senders,
    cricket::MediaDescriptionOptions* audio_media_description_options,
    cricket::MediaDescriptionOptions* video_media_description_options,
    int num_sim_layers) {
   // 多个RtpSender 转换为 SenderOptions
  for (const auto& sender : senders) {
     // 根据 mediatype，添加到对应的MediaDescriptionOptions
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

**多个RtpSender， 就会创建多个SenderOptions。planB是可以由多个RtpSender的， 在AddTrack的时候，每个track，对应生成一个RtpSender。**



#### MediaDescriptionOptions::AddAudioSender

pc/media_session.cc

```cpp
MediaDescriptionOptions::AddVideoSender

void MediaDescriptionOptions::AddAudioSender(
    const std::string& track_id,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK(type == MEDIA_TYPE_AUDIO);
  AddSenderInternal(track_id, stream_ids, {}, SimulcastLayerList(), 1);
}
```



#### MediaDescriptionOptions::AddSenderInternal

pc/media_session.cc

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
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  sender_options.push_back(options);
}
```



## 3. SenderOptions 转 StreamParams



### 3.1 调用堆栈

pc/media_session.cc

```less
CreateOffer
AddAudioContentForOffer/AddVideoContentForOffer
CreateMediaContentOffer
AddStreamParams
CreateStreamParamsForNewSenderWithSsrcs/CreateStreamParamsForNewSenderWithRids
```



```less
CreateAnswer
AddAudioContentForAnswer/AddAudioContentForAnswer
SetCodecsInAnswer
AddStreamParams
CreateStreamParamsForNewSenderWithSsrcs/CreateStreamParamsForNewSenderWithRids
```



### 3.2 !!! 转换时机 MediaSessionDescriptionFactory.AddStreamParams

pc/media_session.cc

把`std::vector<SenderOptions>`转换为`StreamParamsVec`。

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

  // rtx 流
  const bool include_rtx_streams =
      ContainsRtxCodec(content_description->codecs());

  // flex fec 流
  const bool include_flexfec_stream =
      ContainsFlexfecCodec(content_description->codecs());

  // 1. 遍历sender_options
  for (const SenderOptions& sender : sender_options) {
    // groupid is empty for StreamParams generated using
    // MediaSessionDescriptionFactory.
    // 根据sender.track_id 找到对应的 StreamParams
    StreamParams* param =
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

1. 遍历sender_options ， SenderOptions数组，生成对应的 StreamParams

2. 创建先的StreamParams

   - sender.rids为空，就是rtpId， CreateStreamParamsForNewSenderWithSsrcs
     第一次rids为空，这是在createOffer的时候会赋值；第二次协商offer的时候，会赋值
   - sender.rids 不为空， CreateStreamParamsForNewSenderWithRids

3. MediaContentDescriptionImpl::AddStream， StreamParams添加到 MediaContentDescriptionImpl

   

### 3.3 CreateStreamParamsForNewSenderWithRids

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
  result.id = sender.track_id;
  result.cname = rtcp_cname;
  result.set_stream_ids(sender.stream_ids);

  // More than one rid should be signaled.
  if (sender.rids.size() > 1) {
    result.set_rids(sender.rids);
  }

  return result;
}
```

生成 StreamParams对象



### 3.4 CreateStreamParamsForNewSenderWithSsrcs

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

生成 StreamParams对象



## 4. ??? stream_id 赋值时机

```
PeerConnection::AddTransceiver
PeerConnection::CreateSender
```



pc/rtp_sender.h

```cpp

// Internal interface used by PeerConnection.
class RtpSenderInternal : public RtpSenderInterface {
    ...

  // Returns a list of media stream ids associated with this sender's track.
  // These are signalled in the SDP so that the remote side can associate
  // tracks.
  virtual std::vector<std::string> stream_ids() const = 0;

  // Sets the IDs of the media streams associated with this sender's track.
  // These are signalled in the SDP so that the remote side can associate
  // tracks.
  virtual void SetStreams(const std::vector<std::string>& stream_ids) {}
}


// Shared implementation for RtpSenderInternal interface.
class RtpSenderBase : public RtpSenderInternal, public ObserverInterface {
...

  std::vector<std::string> stream_ids() const override { return stream_ids_; }
  void set_stream_ids(const std::vector<std::string>& stream_ids) override {
    stream_ids_ = stream_ids;
  }
  void SetStreams(const std::vector<std::string>& stream_ids) override;
}
```





### 4.1

![1]({{ site.url }}{{ site.baseurl }}/images/senderoptions.assets/1.png)

```cpp
rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>
RtpTransmissionManager::CreateSender(
    cricket::MediaType media_type,
    const std::string& id,
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids,
    const std::vector<RtpEncodingParameters>& send_encodings) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>> sender;
  if (media_type == cricket::MEDIA_TYPE_AUDIO) {
    RTC_DCHECK(!track ||
               (track->kind() == MediaStreamTrackInterface::kAudioKind));
    sender = RtpSenderProxyWithInternal<RtpSenderInternal>::Create(
        signaling_thread(),
        AudioRtpSender::Create(worker_thread(), id, stats_, this));
    NoteUsageEvent(UsageEvent::AUDIO_ADDED);
  } else {
    RTC_DCHECK_EQ(media_type, cricket::MEDIA_TYPE_VIDEO);
    RTC_DCHECK(!track ||
               (track->kind() == MediaStreamTrackInterface::kVideoKind));
    sender = RtpSenderProxyWithInternal<RtpSenderInternal>::Create(
        signaling_thread(), VideoRtpSender::Create(worker_thread(), id, this));
    NoteUsageEvent(UsageEvent::VIDEO_ADDED);
  }
  bool set_track_succeeded = sender->SetTrack(track);
  RTC_DCHECK(set_track_succeeded);
  sender->internal()->set_stream_ids(stream_ids);
  sender->internal()->set_init_send_encodings(send_encodings);
  return sender;
}
```





### 4.2

![2]({{ site.url }}{{ site.baseurl }}/images/senderoptions.assets/2.png)














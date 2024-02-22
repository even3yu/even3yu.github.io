---
layout: post
title: RtpTransceiver，RtpSender，RtpReceiver创建时机
date: 2024-02-22 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---



### 1. 在unify plan 和plan b 中的区别

其实RtpTransceiver 是不支持 plan b，只是代码里面做了兼容。

| 不同点                 | plan b                                                       | unify plan                                                   |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| addTrack               | 每个track，对应就会创建一个RtpSender，多个track就会有多个RtpSender。所以一个Transceiver 就可以由多个RtpSender | 每个track，对应就会创建一个RtpSender，RtpTransceiver。多个track就会有多个RtpTransceiver。所以一个Transceiver 就对应只有一个RtpSender |
| addTrack               | stream_id 只能添加一个                                       | stream_id 可以添加多个，就是说同一个track 可以添加到多个MediaStream中 |
| RtpTransceiver         | 一个RtpTransceiver对应一个mline                              | 一个RtpTransceiver对应一个mline                              |
| RtpTransceiver创建时机 | 在`PeerConnection::Initialize` 默认创建好一个audio，一个video的RtpTransceiver | 动态创建，一个track，对应创建一个transceiver                 |



### 2. addTrack不同

#### RtpTransmissionManager::AddTrackPlanB

pc/rtp_transmission_manager.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackPlanB(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // 1. 只能添加一个stream_id
  if (stream_ids.size() > 1u) {
    LOG_AND_RETURN_ERROR(RTCErrorType::UNSUPPORTED_OPERATION,
                         "AddTrack with more than one stream is not "
                         "supported with Plan B semantics.");
  }
  ...
  // 2. 每个track，对应就会创建一个RtpSender，多个track就会有多个RtpSender
  // 所以一个Transceiver 就可以由多个RtpSender
  //////////////////
  // 创建了CreateSender
  //////////////////
  auto new_sender =
      CreateSender(media_type, track->id(), track, adjusted_stream_ids, {});
  if (track->kind() == MediaStreamTrackInterface::kAudioKind) {
    new_sender->internal()->SetMediaChannel(voice_media_channel());
    // 3. 把RtpSender 添加到对应kind=audio的Transceiver中
    GetAudioTransceiver()->internal()->AddSender(new_sender);
    const RtpSenderInfo* sender_info =
        FindSenderInfo(local_audio_sender_infos_,
                       new_sender->internal()->stream_ids()[0], track->id());
    if (sender_info) {
      new_sender->internal()->SetSsrc(sender_info->first_ssrc);
    }
  } else {
    RTC_DCHECK_EQ(MediaStreamTrackInterface::kVideoKind, track->kind());
    new_sender->internal()->SetMediaChannel(video_media_channel());
    // 3. 把RtpSender 添加到对应kind=video的Transceiver中
    GetVideoTransceiver()->internal()->AddSender(new_sender);
    const RtpSenderInfo* sender_info =
        FindSenderInfo(local_video_sender_infos_,
                       new_sender->internal()->stream_ids()[0], track->id());
    if (sender_info) {
      new_sender->internal()->SetSsrc(sender_info->first_ssrc);
    }
  }
  return rtc::scoped_refptr<RtpSenderInterface>(new_sender);
}
```

1. plan b, 只能添加一个stream_id，为什么？？？
2. 每个track，对应就会创建一个RtpSender，多个track就会有多个RtpSender。所以一个Transceiver 就可以由多个RtpSender
3. 把RtpSender 添加到对应kind=audio/video的Transceiver中， 这里的`GetAudioTransceiver`,`GetVideoTransceiver` 是的Transceiver 是在`PeerConnection::Initialize` 中默认创建好了，对应audio或者video，Transceiver就只有一个。
4. 



#### RtpTransmissionManager::AddTrackUnifiedPlan

pc/rtp_transmission_manager.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackUnifiedPlan(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  // 1.找到一个已经用过但是空闲的Transceiver，
  auto transceiver = FindFirstTransceiverForAddedTrack(track);
   // 2.复用Transceiver
  if (transceiver) {
    RTC_LOG(LS_INFO) << "Reusing an existing "
                     << cricket::MediaTypeToString(transceiver->media_type())
                     << " transceiver for AddTrack.";
    if (transceiver->stopping()) {
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "The existing transceiver is stopping.");
    }

    // 设置transceiver direct， 这是sender，所以至少有kSend
    if (transceiver->direction() == RtpTransceiverDirection::kRecvOnly) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendRecv);
    } else if (transceiver->direction() == RtpTransceiverDirection::kInactive) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendOnly);
    }
    // 把track 关联到RtpSender
    transceiver->sender()->SetTrack(track);    
    transceiver->internal()->sender_internal()->set_stream_ids(stream_ids);
    transceiver->internal()->set_reused_for_addtrack(true);
  } else {
    //  3. 创建新的Transceiver， 创建 RtpSender，RtpReciver
    cricket::MediaType media_type =
        (track->kind() == MediaStreamTrackInterface::kAudioKind
             ? cricket::MEDIA_TYPE_AUDIO
             : cricket::MEDIA_TYPE_VIDEO);
    RTC_LOG(LS_INFO) << "Adding " << cricket::MediaTypeToString(media_type)
                     << " transceiver in response to a call to AddTrack.";
    std::string sender_id = track->id();
    // Avoid creating a sender with an existing ID by generating a random ID.
    // This can happen if this is the second time AddTrack has created a sender
    // for this track.
    if (FindSenderById(sender_id)) {
      sender_id = rtc::CreateRandomUuid();
    }
    auto sender = CreateSender(media_type, sender_id, track, stream_ids, {});
    auto receiver = CreateReceiver(media_type, rtc::CreateRandomUuid());
    transceiver = CreateAndAddTransceiver(sender, receiver);
    transceiver->internal()->set_created_by_addtrack(true);
    transceiver->internal()->set_direction(RtpTransceiverDirection::kSendRecv);
  }
  return transceiver->sender();
}
```

1. 找到一个已经用过但是空闲的Transceiver
2. 如果1 中找到一个Transceiver，则复用
3. 如果1 中未找到，则创建新的Transceiver， 创建 RtpSender，RtpReciver。
4. 每个track，对应就会创建一个RtpSender，RtpTransceiver。多个track就会有多个RtpTransceiver。所以一个Transceiver 就对应只有一个RtpSender。



##### RtpTransmissionManager::FindFirstTransceiverForAddedTrack

pc/rtp_transmission_manager.cc

```cpp
rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
RtpTransmissionManager::FindFirstTransceiverForAddedTrack(
    rtc::scoped_refptr<MediaStreamTrackInterface> track) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(track);
  for (auto transceiver : transceivers()->List()) {
    
    if (!transceiver->sender()->track() && // RtpSender 没有绑定track， 
											// 可以在RtpSenderInterface::SetTrack 解绑 
        cricket::MediaTypeToString(transceiver->media_type()) ==
            track->kind() && // mediatype 一致
        !transceiver->internal()->has_ever_been_used_to_send() && // 这个条件不是很理解？？？
        !transceiver->stopped()) { // transceiver 没有停止
      return transceiver;
    }
  }
  return nullptr;
}
```

满足以下四个条件的，从列表找到满足的一个就返回，拿去复用。

1. `!transceiver->sender()->track() `，RtpSender 没有绑定track，如果绑定了，可以在RtpSenderInterface::SetTrack 解绑 。
2. `cricket::MediaTypeToString(transceiver->media_type()) ==  track->kind() `， mediatype 一致
3. `!transceiver->internal()->has_ever_been_used_to_send()` 这个不是很理解???
4. `!transceiver->stopped()`， transceiver 没有停止



##### ??? has_ever_been_used_to_send

pc/rtp_transceiver.h

```cpp
// Returns true if this transceiver has ever had the current direction set to
  // sendonly or sendrecv.
  bool has_ever_been_used_to_send() const {
    return has_ever_been_used_to_send_;
  }
```



```cpp
void RtpTransceiver::set_current_direction(RtpTransceiverDirection direction) {
  RTC_LOG(LS_INFO) << "Changing transceiver (MID=" << mid_.value_or("<not set>")
                   << ") current direction from "
                   << (current_direction_ ? RtpTransceiverDirectionToString(
                                                *current_direction_)
                                          : "<not set>")
                   << " to " << RtpTransceiverDirectionToString(direction)
                   << ".";
  current_direction_ = direction;
  if (RtpTransceiverDirectionHasSend(*current_direction_)) {
    has_ever_been_used_to_send_ = true;
  }
}
```

1. `SdpOfferAnswerHandler::ApplyLocalDescription` 和 `SdpOfferAnswerHandler::ApplyRemoteDescription` 中调用， 且是在Answer和kPrAnswer 才会调用



### 3. Transceiver 创建时机和个数

#### plan b

```cpp
RTCError PeerConnection::Initialize(
    const PeerConnectionInterface::RTCConfiguration& configuration,
    PeerConnectionDependencies dependencies) {
    ...
// Add default audio/video transceivers for Plan B SDP.
  if (!IsUnifiedPlan()) {
    rtp_manager()->transceivers()->Add(
        RtpTransceiverProxyWithInternal<RtpTransceiver>::Create(
            signaling_thread(), new RtpTransceiver(cricket::MEDIA_TYPE_AUDIO)));
    rtp_manager()->transceivers()->Add(
        RtpTransceiverProxyWithInternal<RtpTransceiver>::Create(
            signaling_thread(), new RtpTransceiver(cricket::MEDIA_TYPE_VIDEO)));
  }
  ...
}
```

默认创建好，audio，video各一个



#### unify plan

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackUnifiedPlan(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  auto transceiver = FindFirstTransceiverForAddedTrack(track);
  if (transceiver) {
    RTC_LOG(LS_INFO) << "Reusing an existing "
                     << cricket::MediaTypeToString(transceiver->media_type())
                     << " transceiver for AddTrack.";
    if (transceiver->stopping()) {
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "The existing transceiver is stopping.");
    }

    // 设置transceiver direct， 这是sender，所以至少有kSend
    if (transceiver->direction() == RtpTransceiverDirection::kRecvOnly) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendRecv);
    } else if (transceiver->direction() == RtpTransceiverDirection::kInactive) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendOnly);
    }
    // 把track 关联到RtpSender
    transceiver->sender()->SetTrack(track);    
    transceiver->internal()->sender_internal()->set_stream_ids(stream_ids);
    transceiver->internal()->set_reused_for_addtrack(true);
  } else {
    cricket::MediaType media_type =
        (track->kind() == MediaStreamTrackInterface::kAudioKind
             ? cricket::MEDIA_TYPE_AUDIO
             : cricket::MEDIA_TYPE_VIDEO);
    RTC_LOG(LS_INFO) << "Adding " << cricket::MediaTypeToString(media_type)
                     << " transceiver in response to a call to AddTrack.";
    std::string sender_id = track->id();
    // Avoid creating a sender with an existing ID by generating a random ID.
    // This can happen if this is the second time AddTrack has created a sender
    // for this track.
    if (FindSenderById(sender_id)) {
      sender_id = rtc::CreateRandomUuid();
    }
    
    //!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!
    auto sender = CreateSender(media_type, sender_id, track, stream_ids, {});
    auto receiver = CreateReceiver(media_type, rtc::CreateRandomUuid());
    //!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!
    // 创建transceiver
    //!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!
    transceiver = CreateAndAddTransceiver(sender, receiver);
    transceiver->internal()->set_created_by_addtrack(true);
    transceiver->internal()->set_direction(RtpTransceiverDirection::kSendRecv);
  }
  return transceiver->sender();
}
```

动态创建，一个track，对应创建一个transceiver。





### 4. RtpSender和RtpReceiver 创建时机

`RtpTransmissionManager::AddTrackUnifiedPlan` 创建, 是在创建Transceiver的时候一起创建的。
在unify plan中 一个track 就对应一个Transceiver，也就是对应一个RtpSender。



#### plan b

pc/rtp_transmission_manager.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackPlanB(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // 1. 只能添加一个stream_id
  if (stream_ids.size() > 1u) {
    LOG_AND_RETURN_ERROR(RTCErrorType::UNSUPPORTED_OPERATION,
                         "AddTrack with more than one stream is not "
                         "supported with Plan B semantics.");
  }
  ...
  // 2. 每个track，对应就会创建一个RtpSender，多个track就会有多个RtpSender
  // 所以一个Transceiver 就可以由多个RtpSender
  //////////////////
  // 创建了CreateSender
  //////////////////
  auto new_sender =
      CreateSender(media_type, track->id(), track, adjusted_stream_ids, {});
   //!!!!!!!!!!!!!!!!!!!!!!!1
  
  if (track->kind() == MediaStreamTrackInterface::kAudioKind) {
    new_sender->internal()->SetMediaChannel(voice_media_channel());
    // 3. 把RtpSender 添加到对应kind=audio的Transceiver中
    GetAudioTransceiver()->internal()->AddSender(new_sender);
    const RtpSenderInfo* sender_info =
        FindSenderInfo(local_audio_sender_infos_,
                       new_sender->internal()->stream_ids()[0], track->id());
    if (sender_info) {
      new_sender->internal()->SetSsrc(sender_info->first_ssrc);
    }
  } else {
    RTC_DCHECK_EQ(MediaStreamTrackInterface::kVideoKind, track->kind());
    new_sender->internal()->SetMediaChannel(video_media_channel());
    // 3. 把RtpSender 添加到对应kind=video的Transceiver中
    GetVideoTransceiver()->internal()->AddSender(new_sender);
    const RtpSenderInfo* sender_info =
        FindSenderInfo(local_video_sender_infos_,
                       new_sender->internal()->stream_ids()[0], track->id());
    if (sender_info) {
      new_sender->internal()->SetSsrc(sender_info->first_ssrc);
    }
  }
  return rtc::scoped_refptr<RtpSenderInterface>(new_sender);
}
```

1. 每个track，对应就会创建一个RtpSender，多个track就会有多个RtpSender。所以一个Transceiver 就可以由多个RtpSender
2. 把RtpSender 添加到对应kind=audio/video的Transceiver中， 这里的`GetAudioTransceiver`,`GetVideoTransceiver` 是的Transceiver 是在`PeerConnection::Initialize` 中默认创建好了，对应audio或者video，Transceiver就只有一个。

#### unify plan

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackUnifiedPlan(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  auto transceiver = FindFirstTransceiverForAddedTrack(track);
  if (transceiver) {
    RTC_LOG(LS_INFO) << "Reusing an existing "
                     << cricket::MediaTypeToString(transceiver->media_type())
                     << " transceiver for AddTrack.";
    if (transceiver->stopping()) {
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "The existing transceiver is stopping.");
    }

    // 设置transceiver direct， 这是sender，所以至少有kSend
    if (transceiver->direction() == RtpTransceiverDirection::kRecvOnly) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendRecv);
    } else if (transceiver->direction() == RtpTransceiverDirection::kInactive) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendOnly);
    }
    // 把track 关联到RtpSender
    transceiver->sender()->SetTrack(track);    
    transceiver->internal()->sender_internal()->set_stream_ids(stream_ids);
    transceiver->internal()->set_reused_for_addtrack(true);
  } else {
    cricket::MediaType media_type =
        (track->kind() == MediaStreamTrackInterface::kAudioKind
             ? cricket::MEDIA_TYPE_AUDIO
             : cricket::MEDIA_TYPE_VIDEO);
    RTC_LOG(LS_INFO) << "Adding " << cricket::MediaTypeToString(media_type)
                     << " transceiver in response to a call to AddTrack.";
    std::string sender_id = track->id();
    // Avoid creating a sender with an existing ID by generating a random ID.
    // This can happen if this is the second time AddTrack has created a sender
    // for this track.
    if (FindSenderById(sender_id)) {
      sender_id = rtc::CreateRandomUuid();
    }
    
    //!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!
    auto sender = CreateSender(media_type, sender_id, track, stream_ids, {});
    auto receiver = CreateReceiver(media_type, rtc::CreateRandomUuid());
    //!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!
    // 创建transceiver
    //!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!
    transceiver = CreateAndAddTransceiver(sender, receiver);
    transceiver->internal()->set_created_by_addtrack(true);
    transceiver->internal()->set_direction(RtpTransceiverDirection::kSendRecv);
  }
  return transceiver->sender();
}
```

动态创建，一个track，对应创建一个transceiver。


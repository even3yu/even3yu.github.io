---
layout: post
title: webrtc Transceiver Direction
date: 2024-02-27 02:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---

## 1. RtpTransceiverDirection 定义

api/rtp_transceiver_direction.h

```cpp
// https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiverdirection
enum class RtpTransceiverDirection {
  kSendRecv, // 发送接收
  kSendOnly, // 只发送
  kRecvOnly, // 只接收
  kInactive, // 默认值
  kStopped,  // 停止
};
```



| Enum value | Description                                                  |
| :--------- | :----------------------------------------------------------- |
| `sendrecv` | The [`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpSender`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsender) sender will offer to send RTP, and will send RTP if the remote peer accepts and sender.[`getParameters`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsender-getparameters)`()`.[`encodings`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsendparameters-encodings)[i].[`active`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpencodingparameters-active) is `true` for any value of i. The [`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpReceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpreceiver) will offer to receive RTP, and will receive RTP if the remote peer accepts. |
| `sendonly` | The [`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpSender`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsender) sender will offer to send RTP, and will send RTP if the remote peer accepts and sender.[`getParameters`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsender-getparameters)`()`.[`encodings`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsendparameters-encodings)[i].[`active`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpencodingparameters-active) is `true` for any value of i. The [`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpReceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpreceiver) will not offer to receive RTP, and will not receive RTP. |
| `recvonly` | The [`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpSender`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsender) will not offer to send RTP, and will not send RTP. The[`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpReceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpreceiver) will offer to receive RTP, and will receive RTP if the remote peer accepts. |
| `inactive` | The [`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpSender`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsender) will not offer to send RTP, and will not send RTP. The[`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver)'s [`RTCRtpReceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpreceiver) will not offer to receive RTP, and will not receive RTP. |
| `stopped`  | The [`RTCRtpTransceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtptransceiver) will neither send nor receive RTP. It will generate a zero port in the offer. In answers, its [`RTCRtpSender`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpsender) will not offer to send RTP, and its [`RTCRtpReceiver`](https://w3c.github.io/webrtc-pc/#dom-rtcrtpreceiver) will not offer to receive RTP. This is a terminal state. |



### 1.1 属性

```cpp
 // transceiver direct 主要是用这个
  RtpTransceiverDirection direction_ = RtpTransceiverDirection::kInactive;
 // 这个目前没发现用法，
  absl::optional<RtpTransceiverDirection> current_direction_;
// 主要用于记录，本次和上一次direct的不同，从而去除掉remote的 stream 和track
  absl::optional<RtpTransceiverDirection> fired_direction_;
```





## 2. !!! RtpTransceiver::set_direction

pc/rtp_transceiver.h

```cpp
// Sets the intended direction for this transceiver. Intended to be used
  // internally over SetDirection since this does not trigger a negotiation
  // needed callback.
  void set_direction(RtpTransceiverDirection direction) {
    direction_ = direction;
  }
```

默认值为`  RtpTransceiverDirection direction_ = RtpTransceiverDirection::kInactive。



### 2.1 RtpTransmissionManager::AddTrackUnifiedPlan

- 创建新的Transceiver，则默认设置 direct = kSendRecv
- 复用Transceiver，则direct根据老的direct，
  kRecvOnly--》kSendRecv
  kInactive--》kSendOnly

pc/rtp_transmission_manager.cc

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

  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
    if (transceiver->direction() == RtpTransceiverDirection::kRecvOnly) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendRecv);
    } else if (transceiver->direction() == RtpTransceiverDirection::kInactive) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendOnly);
    }
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
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
    auto sender = CreateSender(media_type, sender_id, track, stream_ids, {});
    auto receiver = CreateReceiver(media_type, rtc::CreateRandomUuid());
    transceiver = CreateAndAddTransceiver(sender, receiver);
    transceiver->internal()->set_created_by_addtrack(true);
    
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!  
    transceiver->internal()->set_direction(RtpTransceiverDirection::kSendRecv);
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  }
  return transceiver->sender();
}
```





### 2.2 PeerConnection::RemoveTrack

去除track的时候，修改Transceiver的direct。

kSendRecv-》kRecvOnly
kSendOnly-》kInactive

pc/peer_connection.cc

```cpp
bool PeerConnection::RemoveTrack(RtpSenderInterface* sender) {
  TRACE_EVENT0("webrtc", "PeerConnection::RemoveTrack");
  return RemoveTrackNew(sender).ok();
}
```



```cpp
RTCError PeerConnection::RemoveTrackNew(
    rtc::scoped_refptr<RtpSenderInterface> sender) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  if (!sender) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER, "Sender is null.");
  }
  if (IsClosed()) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_STATE,
                         "PeerConnection is closed.");
  }
  if (IsUnifiedPlan()) {
    auto transceiver = FindTransceiverBySender(sender);
    if (!transceiver || !sender->track()) {
      return RTCError::OK();
    }
    //!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!
    sender->SetTrack(nullptr);
    if (transceiver->direction() == RtpTransceiverDirection::kSendRecv) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kRecvOnly);
    } else if (transceiver->direction() == RtpTransceiverDirection::kSendOnly) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kInactive);
    }
    //!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!
  } else {
    ...
  }
  sdp_handler_->UpdateNegotiationNeeded();
  return RTCError::OK();
}
```





### 2.3 PeerConnection::AddTransceiver

根据`PeerConnection::AddTransceiver` 传入 的参数`RtpTransceiverInit::direction`   来进行配置。
应用层自己配置direct。

pc/peer_connection.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverInterface>>
PeerConnection::AddTransceiver(
    cricket::MediaType media_type,
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const RtpTransceiverInit& init,
    bool update_negotiation_needed) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK((media_type == cricket::MEDIA_TYPE_AUDIO ||
              media_type == cricket::MEDIA_TYPE_VIDEO));
  if (track) {
    RTC_DCHECK_EQ(media_type,
                  (track->kind() == MediaStreamTrackInterface::kAudioKind
                       ? cricket::MEDIA_TYPE_AUDIO
                       : cricket::MEDIA_TYPE_VIDEO));
  }

  RTC_HISTOGRAM_COUNTS_LINEAR(kSimulcastNumberOfEncodings,
                              init.send_encodings.size(), 0, 7, 8);

  size_t num_rids = absl::c_count_if(init.send_encodings,
                                     [](const RtpEncodingParameters& encoding) {
                                       return !encoding.rid.empty();
                                     });
  if (num_rids > 0 && num_rids != init.send_encodings.size()) {
    LOG_AND_RETURN_ERROR(
        RTCErrorType::INVALID_PARAMETER,
        "RIDs must be provided for either all or none of the send encodings.");
  }

  if (num_rids > 0 && absl::c_any_of(init.send_encodings,
                                     [](const RtpEncodingParameters& encoding) {
                                       return !IsLegalRsidName(encoding.rid);
                                     })) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                         "Invalid RID value provided.");
  }

  if (absl::c_any_of(init.send_encodings,
                     [](const RtpEncodingParameters& encoding) {
                       return encoding.ssrc.has_value();
                     })) {
    LOG_AND_RETURN_ERROR(
        RTCErrorType::UNSUPPORTED_PARAMETER,
        "Attempted to set an unimplemented parameter of RtpParameters.");
  }

  RtpParameters parameters;
  parameters.encodings = init.send_encodings;

  // Encodings are dropped from the tail if too many are provided.
  if (parameters.encodings.size() > kMaxSimulcastStreams) {
    parameters.encodings.erase(
        parameters.encodings.begin() + kMaxSimulcastStreams,
        parameters.encodings.end());
  }

  // Single RID should be removed.
  if (parameters.encodings.size() == 1 &&
      !parameters.encodings[0].rid.empty()) {
    RTC_LOG(LS_INFO) << "Removing RID: " << parameters.encodings[0].rid << ".";
    parameters.encodings[0].rid.clear();
  }

  // If RIDs were not provided, they are generated for simulcast scenario.
  if (parameters.encodings.size() > 1 && num_rids == 0) {
    rtc::UniqueStringGenerator rid_generator;
    for (RtpEncodingParameters& encoding : parameters.encodings) {
      encoding.rid = rid_generator();
    }
  }

  if (UnimplementedRtpParameterHasValue(parameters)) {
    LOG_AND_RETURN_ERROR(
        RTCErrorType::UNSUPPORTED_PARAMETER,
        "Attempted to set an unimplemented parameter of RtpParameters.");
  }

  auto result = cricket::CheckRtpParametersValues(parameters);
  if (!result.ok()) {
    LOG_AND_RETURN_ERROR(result.type(), result.message());
  }

  RTC_LOG(LS_INFO) << "Adding " << cricket::MediaTypeToString(media_type)
                   << " transceiver in response to a call to AddTransceiver.";
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
  //!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!
  transceiver->internal()->set_direction(init.direction);
  //!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!

  if (update_negotiation_needed) {
    sdp_handler_->UpdateNegotiationNeeded();
  }

  return rtc::scoped_refptr<RtpTransceiverInterface>(transceiver);
}
```





### 2.4 SdpOfferAnswerHandler::AssociateTransceiver

```
ApplyRemoteDescription
ApplyLocalDescription
```

最终调用`SdpOfferAnswerHandler::AssociateTransceiver`。

根据sdp的数据，在收到answer的sdp的时候，`ApplyRemoteDescription`,   `source = cricket::CS_REMOTE`, mid对应的Transceiver没有创建，则创建新的Transciver，设置默认值为`RtpTransceiverDirection::kRecvOnly`。因为作为remote端，如果没有transceiver，新创建的，说明没有数据需要发送。

pc/sdp_offer_answer.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
SdpOfferAnswerHandler::AssociateTransceiver(
    cricket::ContentSource source,
    SdpType type,
    size_t mline_index,
    const ContentInfo& content,
    const ContentInfo* old_local_content,
    const ContentInfo* old_remote_content) {
  RTC_DCHECK(IsUnifiedPlan());
    
  ...

  const MediaContentDescription* media_desc = content.media_description();
  auto transceiver = transceivers()->FindByMid(content.name);
  if (source == cricket::CS_LOCAL) {
    // Find the RtpTransceiver that corresponds to this m= section, using the
    // mapping between transceivers and m= section indices established when
    // creating the offer.
    if (!transceiver) {
      transceiver = transceivers()->FindByMLineIndex(mline_index);
    }
    if (!transceiver) {
      // This may happen normally when media sections are rejected.
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "Transceiver not found based on m-line index");
    }
  } else {
    RTC_DCHECK_EQ(source, cricket::CS_REMOTE);
    // If the m= section is sendrecv or recvonly, and there are RtpTransceivers
    // of the same type...
    // When simulcast is requested, a transceiver cannot be associated because
    // AddTrack cannot be called to initialize it.
    if (!transceiver &&
        RtpTransceiverDirectionHasRecv(media_desc->direction()) &&
        !media_desc->HasSimulcast()) {
      transceiver = FindAvailableTransceiverToReceive(media_desc->type());
    }
    // If no RtpTransceiver was found in the previous step, create one with a
    // recvonly direction.
    if (!transceiver) {
      RTC_LOG(LS_INFO) << "Adding "
                       << cricket::MediaTypeToString(media_desc->type())
                       << " transceiver for MID=" << content.name
                       << " at i=" << mline_index
                       << " in response to the remote description.";
      std::string sender_id = rtc::CreateRandomUuid();
      std::vector<RtpEncodingParameters> send_encodings =
          GetSendEncodingsFromRemoteDescription(*media_desc);
      auto sender = rtp_manager()->CreateSender(media_desc->type(), sender_id,
                                                nullptr, {}, send_encodings);
      std::string receiver_id;
      if (!media_desc->streams().empty()) {
        receiver_id = media_desc->streams()[0].id;
      } else {
        receiver_id = rtc::CreateRandomUuid();
      }
      auto receiver =
          rtp_manager()->CreateReceiver(media_desc->type(), receiver_id);
      transceiver = rtp_manager()->CreateAndAddTransceiver(sender, receiver);
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kRecvOnly);
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!
      if (type == SdpType::kOffer) {
        transceivers()->StableState(transceiver)->set_newly_created();
      }
    }

    RTC_DCHECK(transceiver);

    // Check if the offer indicated simulcast but the answer rejected it.
    // This can happen when simulcast is not supported on the remote party.
    if (SimulcastIsRejected(old_local_content, *media_desc)) {
      RTC_HISTOGRAM_BOOLEAN(kSimulcastDisabled, true);
      RTCError error =
          DisableSimulcastInSender(transceiver->internal()->sender_internal());
      if (!error.ok()) {
        RTC_LOG(LS_ERROR) << "Failed to remove rejected simulcast.";
        return std::move(error);
      }
    }
  }
    
...
    
  // Associate the found or created RtpTransceiver with the m= section by
  // setting the value of the RtpTransceiver's mid property to the MID of the m=
  // section, and establish a mapping between the transceiver and the index of
  // the m= section.
  transceiver->internal()->set_mid(content.name);
  transceiver->internal()->set_mline_index(mline_index);
  return std::move(transceiver);
}
```



## 3. RtpTransceiver::set_current_direction

这个值本身，在代码没看到用途。函数最重要的作用就是修改`has_ever_been_used_to_send_` 这个状态值。

pc/rtp_transceiver.h

```cpp
// Sets the current direction for this transceiver as negotiated in an offer/
  // answer exchange. The current direction is null before an answer with this
  // transceiver has been set.
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
    // 这个状态很重要，一旦为true，那么Transceiver 就没发复用了
		// 这个可以查看下AddTransceiver，活着addTrack的时机，查找可复用的Transciver
    has_ever_been_used_to_send_ = true;
  }
}
```

这个值是在offer和answer 互换后，才会设置的。
以下两种情况会调用`set_current_direction`：

1. - A，createOffer，type = offer;
   - A，setLocalDescription，type = offer， 这时候**不会调用**`set_current_direction`;
   - 收到B 发过来的 answer sdp，type = answer/pranswer， setRemoteDescription， 这时候**调用**`set_current_direction`;

   

2. - A，setLocalDescription，type = offer， 

   - 收到B 发过来的 offer sdp， type = offer，setRemoteDescription，这时候**不会调用**`set_current_direction`;
   - B，createAnswer，type = answer/pranswer;
   - 调用 setLocalDescription， 这时候**调用**`set_current_direction`;

3. 默认初始化的时候，和Transceiver 设置stop的时候， `current_direction = absl::nullopt`;



### !!! has_ever_been_used_to_send_

pc/rtp_transceiver.h

```cpp
  // Returns true if this transceiver has ever had the current direction set to
  // sendonly or sendrecv.
  bool has_ever_been_used_to_send() const {
    return has_ever_been_used_to_send_;
  }
```





### 3.1 SdpOfferAnswerHandler::ApplyLocalDescription

pc/sdp_offer_answer.cc

```cpp
RTCError SdpOfferAnswerHandler::ApplyLocalDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(desc);

	...

  if (IsUnifiedPlan()) {
    RTCError error = UpdateTransceiversAndDataChannels(
        cricket::CS_LOCAL, *local_description(), old_local_description,
        remote_description());
    if (!error.ok()) {
      return error;
    }
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>> remove_list;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> removed_streams;
    for (const auto& transceiver : transceivers()->List()) {
			...
      const MediaContentDescription* media_desc = content->media_description();
      // 2.2.7.1.6: If description is of type "answer" or "pranswer", then run
      // the following steps:
      
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
      // 注意这里type, 远端收到offer后，createanswer，这时候type=kPrAnswer/kAnswer
			// 然后设置SetLocalDescription
      if (type == SdpType::kPrAnswer || type == SdpType::kAnswer) {
        // 2.2.7.1.6.1: If direction is "sendonly" or "inactive", and
        // transceiver's [[FiredDirection]] slot is either "sendrecv" or
        // "recvonly", process the removal of a remote track for the media
        // description, given transceiver, removeList, and muteTracks.
        if (!RtpTransceiverDirectionHasRecv(media_desc->direction()) &&
            (transceiver->internal()->fired_direction() &&
             RtpTransceiverDirectionHasRecv(
                 *transceiver->internal()->fired_direction()))) {
          ProcessRemovalOfRemoteTrack(transceiver, &remove_list,
                                      &removed_streams);
        }
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        // 2.2.7.1.6.2: Set transceiver's [[CurrentDirection]] and
        // [[FiredDirection]] slots to direction.
        transceiver->internal()->set_current_direction(media_desc->direction());
        transceiver->internal()->set_fired_direction(media_desc->direction());
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      }
    }
    auto observer = pc_->Observer();
    for (const auto& transceiver : remove_list) {
      observer->OnRemoveTrack(transceiver->receiver());
    }
    for (const auto& stream : removed_streams) {
      observer->OnRemoveStream(stream);
    }
  } else {
   ...
  }
	...

  return RTCError::OK();
}
```



### 3.2 SdpOfferAnswerHandler::ApplyRemoteDescription

```cpp
RTCError SdpOfferAnswerHandler::ApplyRemoteDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {
  ...


  // If setting the description decided our SSL role, allocate any necessary
  // SCTP sids.
  rtc::SSLRole role;
  if (IsSctpLike(pc_->data_channel_type()) && pc_->GetSctpSslRole(&role)) {
    data_channel_controller()->AllocateSctpSids(role);
  }

  if (IsUnifiedPlan()) {
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>>
        now_receiving_transceivers;
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>> remove_list;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> added_streams;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> removed_streams;
    for (const auto& transceiver : transceivers()->List()) {
      const ContentInfo* content =
          FindMediaSectionForTransceiver(transceiver, remote_description());
      if (!content) {
        continue;
      }
      const MediaContentDescription* media_desc = content->media_description();
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      RtpTransceiverDirection local_direction =
          RtpTransceiverDirectionReversed(media_desc->direction());
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      // Roughly the same as steps 2.2.8.6 of section 4.4.1.6 "Set the
      // RTCSessionDescription: Set the associated remote streams given
      // transceiver.[[Receiver]], msids, addList, and removeList".
      // https://w3c.github.io/webrtc-pc/#set-the-rtcsessiondescription
      if (RtpTransceiverDirectionHasRecv(local_direction)) {
        std::vector<std::string> stream_ids;
        if (!media_desc->streams().empty()) {
          // The remote description has signaled the stream IDs.
          stream_ids = media_desc->streams()[0].stream_ids();
        }
        transceivers()
            ->StableState(transceiver)
            ->SetRemoteStreamIdsIfUnset(transceiver->receiver()->stream_ids());

        RTC_LOG(LS_INFO) << "Processing the MSIDs for MID=" << content->name
                         << " (" << GetStreamIdsString(stream_ids) << ").";
        SetAssociatedRemoteStreams(transceiver->internal()->receiver_internal(),
                                   stream_ids, &added_streams,
                                   &removed_streams);
        // From the WebRTC specification, steps 2.2.8.5/6 of section 4.4.1.6
        // "Set the RTCSessionDescription: If direction is sendrecv or recvonly,
        // and transceiver's current direction is neither sendrecv nor recvonly,
        // process the addition of a remote track for the media description.
        if (!transceiver->fired_direction() ||
            !RtpTransceiverDirectionHasRecv(*transceiver->fired_direction())) {
          RTC_LOG(LS_INFO)
              << "Processing the addition of a remote track for MID="
              << content->name << ".";
          now_receiving_transceivers.push_back(transceiver);
        }
      }
      // 2.2.8.1.9: If direction is "sendonly" or "inactive", and transceiver's
      // [[FiredDirection]] slot is either "sendrecv" or "recvonly", process the
      // removal of a remote track for the media description, given transceiver,
      // removeList, and muteTracks.
      if (!RtpTransceiverDirectionHasRecv(local_direction) &&
          (transceiver->fired_direction() &&
           RtpTransceiverDirectionHasRecv(*transceiver->fired_direction()))) {
        ProcessRemovalOfRemoteTrack(transceiver, &remove_list,
                                    &removed_streams);
      }
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      // 2.2.8.1.10: Set transceiver's [[FiredDirection]] slot to direction.
      transceiver->internal()->set_fired_direction(local_direction);
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      
      // 2.2.8.1.11: If description is of type "answer" or "pranswer", then run
      // the following steps:
      
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      // 注意这里type, 本端收到answer后，这时候type=kPrAnswer/kAnswer
			// 然后设置SetRemoteDescription
      if (type == SdpType::kPrAnswer || type == SdpType::kAnswer) {
        // 2.2.8.1.11.1: Set transceiver's [[CurrentDirection]] slot to
        // direction.
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        // local_direction = RtpTransceiverDirectionReversed(media_desc->direction());
        transceiver->internal()->set_current_direction(local_direction);
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        // 2.2.8.1.11.[3-6]: Set the transport internal slots.
        if (transceiver->mid()) {
          auto dtls_transport =
              transport_controller()->LookupDtlsTransportByMid(
                  *transceiver->mid());
          transceiver->internal()->sender_internal()->set_transport(
              dtls_transport);
          transceiver->internal()->receiver_internal()->set_transport(
              dtls_transport);
        }
      }
      ...
    }
   ...
  }
  
	...

  return RTCError::OK();
}
```



### RtpTransceiver::StopTransceiverProcedure

pc/rtp_transceiver.cc

```cpp
void RtpTransceiver::StopTransceiverProcedure() {
  RTC_DCHECK_RUN_ON(thread_);
  // As specified in the "Stop the RTCRtpTransceiver" procedure
  // 1. If transceiver.[[Stopping]] is false, stop sending and receiving given
  // transceiver.
  if (!stopping_)
    StopSendingAndReceiving();

  // 2. Set transceiver.[[Stopped]] to true.
  stopped_ = true;

  // Signal the updated change to the senders.
  for (const auto& sender : senders_)
    sender->internal()->SetTransceiverAsStopped();

        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
  // 3. Set transceiver.[[Receptive]] to false.
  // 4. Set transceiver.[[CurrentDirection]] to null.
  current_direction_ = absl::nullopt;
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
}
```





## 4. !!! RtpTransceiver::set_fired_direction

`fired_direction_` 可以认为是上一次协商的direct。主要用于比对，如果direct发生改变，去除接收端的track和stream。

```cpp
void RtpTransceiver::set_fired_direction(RtpTransceiverDirection direction) {
  fired_direction_ = direction;
}
```



```cpp
absl::optional<RtpTransceiverDirection> RtpTransceiver::fired_direction()
    const {
  return fired_direction_;
}
```





### 4.1 SdpOfferAnswerHandler::ApplyLocalDescription

pc/sdp_offer_answer.cc

```cpp
RTCError SdpOfferAnswerHandler::ApplyLocalDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(desc);

	...

  if (IsUnifiedPlan()) {
    ...
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>> remove_list;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> removed_streams;
    for (const auto& transceiver : transceivers()->List()) {
			...
      const MediaContentDescription* media_desc = content->media_description();
      // 2.2.7.1.6: If description is of type "answer" or "pranswer", then run
      // the following steps:
      
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
      // 注意这里type, 远端收到offer后，createanswer，这时候type=kPrAnswer/kAnswer
			// 然后设置SetLocalDescription
      if (type == SdpType::kPrAnswer || type == SdpType::kAnswer) {
        // 2.2.7.1.6.1: If direction is "sendonly" or "inactive", and
        // transceiver's [[FiredDirection]] slot is either "sendrecv" or
        // "recvonly", process the removal of a remote track for the media
        // description, given transceiver, removeList, and muteTracks.
        
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
        // 1. answer的 direct 没有recv的方向
        // 2. transceiver->internal()->fired_direction()  不为空，说明已经收到answer/pranswer
				// 		且有recv方向
        // 3. 则删除remoteStream
        if (!RtpTransceiverDirectionHasRecv(media_desc->direction()) &&
            (transceiver->internal()->fired_direction() &&
             RtpTransceiverDirectionHasRecv(
                 *transceiver->internal()->fired_direction()))) {
          ProcessRemovalOfRemoteTrack(transceiver, &remove_list,
                                      &removed_streams);
        }
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        // 2.2.7.1.6.2: Set transceiver's [[CurrentDirection]] and
        // [[FiredDirection]] slots to direction.
        transceiver->internal()->set_current_direction(media_desc->direction());
        transceiver->internal()->set_fired_direction(media_desc->direction());
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      }
    }
    // 通知应用层，track和stream删除
    auto observer = pc_->Observer();
    for (const auto& transceiver : remove_list) {
      observer->OnRemoveTrack(transceiver->receiver());
    }
    for (const auto& stream : removed_streams) {
      observer->OnRemoveStream(stream);
    }
  } else {
   ...
  }
	...

  return RTCError::OK();
}
```



#### 4.1.1 -- SdpOfferAnswerHandler::ProcessRemovalOfRemoteTrack

参考 4.3



### 4.2 SdpOfferAnswerHandler::ApplyRemoteDescription

```cpp
RTCError SdpOfferAnswerHandler::ApplyRemoteDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {
  ...


  if (IsUnifiedPlan()) {
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>>
        now_receiving_transceivers;
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>> remove_list;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> added_streams;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> removed_streams;
    for (const auto& transceiver : transceivers()->List()) {
      const ContentInfo* content =
          FindMediaSectionForTransceiver(transceiver, remote_description());
      if (!content) {
        continue;
      }
      const MediaContentDescription* media_desc = content->media_description();
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      RtpTransceiverDirection local_direction =
          RtpTransceiverDirectionReversed(media_desc->direction());
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      // Roughly the same as steps 2.2.8.6 of section 4.4.1.6 "Set the
      // RTCSessionDescription: Set the associated remote streams given
      // transceiver.[[Receiver]], msids, addList, and removeList".
      // https://w3c.github.io/webrtc-pc/#set-the-rtcsessiondescription
      if (RtpTransceiverDirectionHasRecv(local_direction)) {
        std::vector<std::string> stream_ids;
        if (!media_desc->streams().empty()) {
          // The remote description has signaled the stream IDs.
          stream_ids = media_desc->streams()[0].stream_ids();
        }
        transceivers()
            ->StableState(transceiver)
            ->SetRemoteStreamIdsIfUnset(transceiver->receiver()->stream_ids());

        RTC_LOG(LS_INFO) << "Processing the MSIDs for MID=" << content->name
                         << " (" << GetStreamIdsString(stream_ids) << ").";
        SetAssociatedRemoteStreams(transceiver->internal()->receiver_internal(),
                                   stream_ids, &added_streams,
                                   &removed_streams);
        // From the WebRTC specification, steps 2.2.8.5/6 of section 4.4.1.6
        // "Set the RTCSessionDescription: If direction is sendrecv or recvonly,
        // and transceiver's current direction is neither sendrecv nor recvonly,
        // process the addition of a remote track for the media description.
        if (!transceiver->fired_direction() ||
            !RtpTransceiverDirectionHasRecv(*transceiver->fired_direction())) {
          RTC_LOG(LS_INFO)
              << "Processing the addition of a remote track for MID="
              << content->name << ".";
          now_receiving_transceivers.push_back(transceiver);
        }
      }
      // 2.2.8.1.9: If direction is "sendonly" or "inactive", and transceiver's
      // [[FiredDirection]] slot is either "sendrecv" or "recvonly", process the
      // removal of a remote track for the media description, given transceiver,
      // removeList, and muteTracks.
        
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!
        // 1. direct 没有recv的方向
        // 2. transceiver->internal()->fired_direction()  不为空，说明已经收到answer/pranswer
				// 		且有recv方向
        // 3. 则删除remoteStream
      if (!RtpTransceiverDirectionHasRecv(local_direction) &&
          (transceiver->fired_direction() &&
           RtpTransceiverDirectionHasRecv(*transceiver->fired_direction()))) {
        ProcessRemovalOfRemoteTrack(transceiver, &remove_list,
                                    &removed_streams);
      }
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      // 2.2.8.1.10: Set transceiver's [[FiredDirection]] slot to direction.
      transceiver->internal()->set_fired_direction(local_direction);
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      
      // 2.2.8.1.11: If description is of type "answer" or "pranswer", then run
      // the following steps:
      
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
      // 注意这里type, 本端收到answer后，这时候type=kPrAnswer/kAnswer
			// 然后设置SetRemoteDescription
      if (type == SdpType::kPrAnswer || type == SdpType::kAnswer) {
        // 2.2.8.1.11.1: Set transceiver's [[CurrentDirection]] slot to
        // direction.
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        // local_direction = RtpTransceiverDirectionReversed(media_desc->direction());
        transceiver->internal()->set_current_direction(local_direction);
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        //!!!!!!!!!!!!!!!!!!
        // 2.2.8.1.11.[3-6]: Set the transport internal slots.
        if (transceiver->mid()) {
          auto dtls_transport =
              transport_controller()->LookupDtlsTransportByMid(
                  *transceiver->mid());
          transceiver->internal()->sender_internal()->set_transport(
              dtls_transport);
          transceiver->internal()->receiver_internal()->set_transport(
              dtls_transport);
        }
      }
      ...
    }
   ...
  }
  
	...

  return RTCError::OK();
}
```



#### 4.2.1 -- SdpOfferAnswerHandler::ProcessRemovalOfRemoteTrack

参考 4.3

### 4.3 SdpOfferAnswerHandler::ProcessRemovalOfRemoteTrack

```cpp
void SdpOfferAnswerHandler::ProcessRemovalOfRemoteTrack(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>>* remove_list,
    std::vector<rtc::scoped_refptr<MediaStreamInterface>>* removed_streams) {
  RTC_DCHECK(transceiver->mid());
  RTC_LOG(LS_INFO) << "Processing the removal of a track for MID="
                   << *transceiver->mid();
  std::vector<rtc::scoped_refptr<MediaStreamInterface>> previous_streams =
      transceiver->internal()->receiver_internal()->streams();
  // This will remove the remote track from the streams.
  transceiver->internal()->receiver_internal()->set_stream_ids({});
  remove_list->push_back(transceiver);
  RemoveRemoteStreamsIfEmpty(previous_streams, removed_streams);
}
```



#### 4.3.1 SdpOfferAnswerHandler::RemoveRemoteStreamsIfEmpty

```cpp
void SdpOfferAnswerHandler::RemoveRemoteStreamsIfEmpty(
    const std::vector<rtc::scoped_refptr<MediaStreamInterface>>& remote_streams,
    std::vector<rtc::scoped_refptr<MediaStreamInterface>>* removed_streams) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // TODO(https://crbug.com/webrtc/9480): When we use stream IDs instead of
  // streams, see if the stream was removed by checking if this was the last
  // receiver with that stream ID.
  // 删除remote_streams_中对应的removed_streams
  for (const auto& remote_stream : remote_streams) {
    if (remote_stream->GetAudioTracks().empty() &&
        remote_stream->GetVideoTracks().empty()) {
      remote_streams_->RemoveStream(remote_stream);
      removed_streams->push_back(remote_stream);
    }
  }
}
```


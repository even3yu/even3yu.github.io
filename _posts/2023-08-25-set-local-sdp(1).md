---
layout: post
title: webrtc setLocalDescription-1
date: 2023-08-23 23:55:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

## 0. 前言

、<img src="{{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/view.png" style="zoom: 67%;" />

1. SdpOfferAnswerHandler::PushdownTransportDescription，创建了JsepTransport，DtlsTransport
   RtpTransport，SrtpTransport，DtlsSrtpTransport
   JsepTransportController，JsepTransportDescription。
   （mid， JsepTransport ）以这样的键值对存放，其中，如果Bundle 多个m Line， 那么以第一个mid 为key，公用JsepTransport；如果没有Bundle，那么每个mLine 都会生成JsepTransport；

2. SdpOfferAnswerHandler::UpdateTransceiverChannel，检查PC中的每个RtpTranceiver是存在MediaChannel，不存在的会调用WebRtcVideoEngine::CreateMediaChannel()创建WebRtcVideoChannel对象，并赋值给RtpTranceiver的RtpSender和RtpReceiver，这儿解决了VideoRtpSender的media_channel_成员为空的问题；

3. SdpOfferAnswerHandler::UpdateSessionState，将SDP中的信息应用到上一步创建的视频媒体通道对象WebRtcVideoChannel上，调用WebRtcVideoChannel::AddSendStream()方法为通道创建WebRtcVideoSendStream，如果有多个视频Track，会有多个WebRtcVideoSendStream分别与之对应。WebRtcVideoSendStream对象存入WebRtcVideoChannel的std::map<uint32_t, WebRtcVideoSendStream*> send_streams_成员，以ssrc为key。创建WebRtcVideoSendStream，其构造函数中会进一步创建VideoSendStream，VideoSendStream的构造中会进一步创建



## 1. PeerConnection::SetLocalDescription

pc/peer_connection.cc

```cpp
void PeerConnection::SetLocalDescription(
    SetSessionDescriptionObserver* observer,
    SessionDescriptionInterface* desc_ptr) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  sdp_handler_->SetLocalDescription(observer, desc_ptr);
}
```



## 2. SdpOfferAnswerHandler::SetLocalDescription

pc/sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::SetLocalDescription(
    SetSessionDescriptionObserver* observer,
    SessionDescriptionInterface* desc_ptr) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // Chain this operation. If asynchronous operations are pending on the chain,
  // this operation will be queued to be invoked, otherwise the contents of the
  // lambda will execute immediately.
  operations_chain_->ChainOperation(
      [this_weak_ptr = weak_ptr_factory_.GetWeakPtr(),
       observer_refptr =
           rtc::scoped_refptr<SetSessionDescriptionObserver>(observer),
       desc = std::unique_ptr<SessionDescriptionInterface>(desc_ptr)](
          std::function<void()> operations_chain_callback) mutable {

        ...

        // SetSessionDescriptionObserverAdapter takes care of making sure the
        // |observer_refptr| is invoked in a posted message.
        this_weak_ptr->DoSetLocalDescription(
            std::move(desc),
            rtc::scoped_refptr<SetLocalDescriptionObserverInterface>(
                new rtc::RefCountedObject<SetSessionDescriptionObserverAdapter>(
                    this_weak_ptr, observer_refptr)));
        // For backwards-compatability reasons, we declare the operation as
        // completed here (rather than in a post), so that the operation chain
        // is not blocked by this operation when the observer is invoked. This
        // allows the observer to trigger subsequent offer/answer operations
        // synchronously if the operation chain is now empty.
        operations_chain_callback();
      });
}
```

这里用了链式操作，之前文章有说过，是为了同步执行，同一时间只能有一个operation（createOffer， setLocalDescription, setRemoteDescription, createAnswer）在运行，直到结束。


### 2.1 SetLocalDescriptionObserverInterface

api/set_local_description_observer_interface.h

```cpp
class SetLocalDescriptionObserverInterface : public rtc::RefCountInterface {
 public:
  // On success, |error.ok()| is true.
  virtual void OnSetLocalDescriptionComplete(RTCError error) = 0;
};
```



### 2.2 SetSessionDescriptionObserver

api/jsep.h

```cpp
// SetLocalDescription and SetRemoteDescription callback interface.
class RTC_EXPORT SetSessionDescriptionObserver : public rtc::RefCountInterface {
 public:
  virtual void OnSuccess() = 0;
  // See description in CreateSessionDescriptionObserver for OnFailure.
  virtual void OnFailure(RTCError error) = 0;

 protected:
  ~SetSessionDescriptionObserver() override = default;
};
```



### 2.3 SetSessionDescriptionObserverAdapter

pc/sdp_offer_answer.cc

```cpp
class SdpOfferAnswerHandler::SetSessionDescriptionObserverAdapter
    : public SetLocalDescriptionObserverInterface,
      public SetRemoteDescriptionObserverInterface {
 public:
  SetSessionDescriptionObserverAdapter(
      rtc::WeakPtr<SdpOfferAnswerHandler> handler,
      rtc::scoped_refptr<SetSessionDescriptionObserver> inner_observer)
      : handler_(std::move(handler)),
        inner_observer_(std::move(inner_observer)) {}

        ...

  rtc::scoped_refptr<SetSessionDescriptionObserver> inner_observer_;
 }
```



## 3. SdpOfferAnswerHandler::DoSetLocalDescription

pc/sdp_offer_answer.cc

```C++
void SdpOfferAnswerHandler::DoSetLocalDescription(
    std::unique_ptr<SessionDescriptionInterface> desc,
    rtc::scoped_refptr<SetLocalDescriptionObserverInterface> observer) {

    ...

  // Grab the description type before moving ownership to ApplyLocalDescription,
  // which may destroy it before returning.
  const SdpType type = desc->GetType();

  error = ApplyLocalDescription(std::move(desc));
  // |desc| may be destroyed at this point.

  if (!error.ok()) {
    // If ApplyLocalDescription fails, the PeerConnection could be in an
    // inconsistent state, so act conservatively here and set the session error
    // so that future calls to SetLocalDescription/SetRemoteDescription fail.
    SetSessionError(SessionError::kContent, error.message());
    std::string error_message =
        GetSetDescriptionErrorMessage(cricket::CS_LOCAL, type, error);
    RTC_LOG(LS_ERROR) << error_message;
    observer->OnSetLocalDescriptionComplete(
        RTCError(RTCErrorType::INTERNAL_ERROR, std::move(error_message)));
    return;
  }

  if (local_description()->GetType() == SdpType::kAnswer) {
        ...
  }

  observer->OnSetLocalDescriptionComplete(RTCError::OK());
  pc_->NoteUsageEvent(UsageEvent::SET_LOCAL_DESCRIPTION_SUCCEEDED);

  // Check if negotiation is needed. We must do this after informing the
  // observer that SetLocalDescription() has completed to ensure negotiation is
  // not needed prior to the promise resolving.
  if (IsUnifiedPlan()) {
    bool was_negotiation_needed = is_negotiation_needed_;
    UpdateNegotiationNeeded();
    if (signaling_state() == PeerConnectionInterface::kStable &&
        was_negotiation_needed && is_negotiation_needed_) {
      // Legacy version.
      pc_->Observer()->OnRenegotiationNeeded();
      // Spec-compliant version; the event may get invalidated before firing.
      GenerateNegotiationNeededEvent();
    }
  }

  // MaybeStartGathering needs to be called after informing the observer so that
  // we don't signal any candidates before signaling that SetLocalDescription
  // completed.
  transport_controller()->MaybeStartGathering();
}
```

1. ApplyLocalDescription， 应用SessionDescription，创建Transport， 创建BaseChannel等
2. MaybeStartGathering， 主要是 ice 信息收集；



### 3.1 --SdpOfferAnswerHandler.ApplyLocalDescription

见章节4。



## 4. !!! SdpOfferAnswerHandler.ApplyLocalDescription

pc/sdp_offer_answer.cc

```c++
// desc 就是 JsepSessionDescription对象
RTCError SdpOfferAnswerHandler::ApplyLocalDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {

  ...

  // 上一次的sdp
  // local_description() {
  // pending_local_description_ ? pending_local_description_.get()
  // : current_local_description_.get(); }
  const SessionDescriptionInterface* old_local_description =
      local_description();
  // 新的sdp
  std::unique_ptr<SessionDescriptionInterface> replaced_local_description;
  SdpType type = desc->GetType();
  if (type == SdpType::kAnswer) {
    ...
  } else {
    // 第一的时候，pending_local_description_ 是空的
    replaced_local_description = std::move(pending_local_description_);
    // 1. pending_local_description_ 就是参数传入的desc
    pending_local_description_ = std::move(desc);
  }
  ...

  // is_caller_ 是空的话，进行赋值
  if (!is_caller_) {
    if (remote_description()) {
      // Remote description was applied first, so this PC is the callee.
      is_caller_ = false;
    } else {
      // Local description is applied first, so this PC is the caller.
      // 是主叫，因为remote_descroption
      is_caller_ = true;
    }
  }
  
  // 2. 应用SDP到底层传输，根据mid以及相关描述信息创建底层传输Transport
  // Transport层，用来创建对应于上层结构的传输层对象，
  RTCError error = PushdownTransportDescription(cricket::CS_LOCAL, type);
  if (!error.ok()) {
    return error;
  }

  if (IsUnifiedPlan()) {
    // 3. 创建BaseChanel，就是VideoChannel，VoiceChannel
    // enum ContentSource { CS_LOCAL, CS_REMOTE };
    RTCError error = UpdateTransceiversAndDataChannels(
        cricket::CS_LOCAL, *local_description(), old_local_description,
        remote_description());
    if (!error.ok()) {
      return error;
    }
    std::vector<rtc::scoped_refptr<RtpTransceiverInterface>> remove_list;
    std::vector<rtc::scoped_refptr<MediaStreamInterface>> removed_streams;
    for (const auto& transceiver : transceivers()->List()) {
      if (transceiver->stopped()) {
        continue;
      }

      // 2.2.7.1.1.(6-9): Set sender and receiver's transport slots.
      // Note that code paths that don't set MID won't be able to use
      // information about DTLS transports.
      if (transceiver->mid()) {
        // transport_controller就是在PeerConnection中创建的对象，主要管理transport
                //  transport 会通过下面的接口绑定到每个 RtpTransceiver 里面
        auto dtls_transport = transport_controller()->LookupDtlsTransportByMid(
            *transceiver->mid());
        transceiver->internal()->sender_internal()->set_transport(
            dtls_transport);
        transceiver->internal()->receiver_internal()->set_transport(
            dtls_transport);
      }

      const ContentInfo* content =
          FindMediaSectionForTransceiver(transceiver, local_description());
      if (!content) {
        continue;
      }
      const MediaContentDescription* media_desc = content->media_description();
      // 2.2.7.1.6: If description is of type "answer" or "pranswer", then run
      // the following steps:
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
        // 2.2.7.1.6.2: Set transceiver's [[CurrentDirection]] and
        // [[FiredDirection]] slots to direction.
        transceiver->internal()->set_current_direction(media_desc->direction());
        transceiver->internal()->set_fired_direction(media_desc->direction());
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
    // Media channels will be created only when offer is set. These may use new
    // transports just created by PushdownTransportDescription.
    if (type == SdpType::kOffer) {
      // TODO(bugs.webrtc.org/4676) - Handle CreateChannel failure, as new local
      // description is applied. Restore back to old description.
      RTCError error = CreateChannels(*local_description()->description());
      if (!error.ok()) {
        return error;
      }
    }
    // Remove unused channels if MediaContentDescription is rejected.
    RemoveUnusedChannels(local_description()->description());
  }

  // 创建WebRtcVideoSendStream 【章节2.4】
  // type= offer
  error = UpdateSessionState(type, cricket::CS_LOCAL,
                             local_description()->description());
  if (!error.ok()) {
    return error;
  }

  if (remote_description()) {
    // Now that we have a local description, we can push down remote candidates.
    UseCandidatesInSessionDescription(remote_description());
  }

  pending_ice_restarts_.clear();
  if (session_error() != SessionError::kNone) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INTERNAL_ERROR, GetSessionErrorMsg());
  }

  // If setting the description decided our SSL role, allocate any necessary
  // SCTP sids.
  rtc::SSLRole role;
  if (IsSctpLike(pc_->data_channel_type()) && pc_->GetSctpSslRole(&role)) {
    data_channel_controller()->AllocateSctpSids(role);
  }

  if (IsUnifiedPlan()) {
    for (const auto& transceiver : transceivers()->List()) {
      if (transceiver->stopped()) {
        continue;
      }
      const ContentInfo* content =
          FindMediaSectionForTransceiver(transceiver, local_description());
      if (!content) {
        continue;
      }
      cricket::ChannelInterface* channel = transceiver->internal()->channel();
      if (content->rejected || !channel || channel->local_streams().empty()) {
        // 0 is a special value meaning "this sender has no associated send
        // stream". Need to call this so the sender won't attempt to configure
        // a no longer existing stream and run into DCHECKs in the lower
        // layers.
        transceiver->internal()->sender_internal()->SetSsrc(0);
      } else {
        // Get the StreamParams from the channel which could generate SSRCs.
        const std::vector<StreamParams>& streams = channel->local_streams();
        transceiver->internal()->sender_internal()->set_stream_ids(
            streams[0].stream_ids());
        // 给 RtpSenderInternal设置SetSsrc 【章节2.5】，videosource和encoder 绑定
        transceiver->internal()->sender_internal()->SetSsrc(
            streams[0].first_ssrc());
      }
    }
  } else {
    // Plan B semantics.

    // Update state and SSRC of local MediaStreams and DataChannels based on the
    // local session description.
    const cricket::ContentInfo* audio_content =
        GetFirstAudioContent(local_description()->description());
    if (audio_content) {
      if (audio_content->rejected) {
        RemoveSenders(cricket::MEDIA_TYPE_AUDIO);
      } else {
        const cricket::AudioContentDescription* audio_desc =
            audio_content->media_description()->as_audio();
        UpdateLocalSenders(audio_desc->streams(), audio_desc->type());
      }
    }

    const cricket::ContentInfo* video_content =
        GetFirstVideoContent(local_description()->description());
    if (video_content) {
      if (video_content->rejected) {
        RemoveSenders(cricket::MEDIA_TYPE_VIDEO);
      } else {
        const cricket::VideoContentDescription* video_desc =
            video_content->media_description()->as_video();
        UpdateLocalSenders(video_desc->streams(), video_desc->type());
      }
    }
  }

  const cricket::ContentInfo* data_content =
      GetFirstDataContent(local_description()->description());
  if (data_content) {
    const cricket::RtpDataContentDescription* rtp_data_desc =
        data_content->media_description()->as_rtp_data();
    // rtp_data_desc will be null if this is an SCTP description.
    if (rtp_data_desc) {
      data_channel_controller()->UpdateLocalRtpDataChannels(
          rtp_data_desc->streams());
    }
  }

  if (type == SdpType::kAnswer &&
      local_ice_credentials_to_replace_->SatisfiesIceRestart(
          *current_local_description_)) {
    local_ice_credentials_to_replace_->ClearIceCredentials();
  }

  return RTCError::OK();
}
```

1. pending_local_description_， 保存了 sessionDescription；
   
1. localDescription
   
   ```c++
   const SessionDescriptionInterface* SdpOfferAnswerHandler::local_description()
       const {
     RTC_DCHECK_RUN_ON(signaling_thread());
     return pending_local_description_ ? pending_local_description_.get()
                                       : current_local_description_.get();
   }
   ```

2. desc 就是 JsepSessionDescription对象，是在CreateOffer 中的，WebRtcSessionDescriptionFactory::InternalCreateOffer中创建的

2. 根据mid以及相关描述信息ContentInfo创建JsepTransport。

3. transpot 怎么transceiver 如何关联
   
   ```c++
   void set_transport(
         rtc::scoped_refptr<DtlsTransportInterface> dtls_transport) override {
       dtls_transport_ = dtls_transport;
     }
   ```
   
   
   
6.  创建BaseChanel，就是VideoChannel，VoiceChannel



### 4.1 --SdpOfferAnswerHandler::PushdownTransportDescription

pc/sdp_offer_answer.cc
创建了JsepTransport，DtlsTransport
RtpTransport，SrtpTransport，DtlsSrtpTransport
JsepTransportController，JsepTransportDescription。
（mid， JsepTransport ）以这样的键值对存放，其中，如果Bundle 多个m Line， 那么以第一个mid 为key，公用JsepTransport；如果没有Bundle，那么每个mLine 都会生成JsepTransport；

### 4.2 --SdpOfferAnswerHandler.UpdateTransceiversAndDataChannels

pc/sdp_offer_answer.cc
创建BaseChannel。

### 4.3 --UpdateSessionState

pc/sdp_offer_answer.cc

## 5. SdpOfferAnswerHandler::PushdownTransportDescription

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::PushdownTransportDescription(
    cricket::ContentSource source,
    SdpType type) {
  ...

  if (source == cricket::CS_LOCAL) {
    const SessionDescriptionInterface* sdesc = local_description();
    RTC_DCHECK(sdesc);
    // transport_controller() 其实返回的就是peerConnection 中的 
    // JsepTransportController transport_controller_； 是在peerConnection
    // 的Initialize 中创建
    return transport_controller()->SetLocalDescription(type,
                                                       sdesc->description());
  } else {
    ...
  }
}
```

- transport_controller()， 其实返回的就是peerConnection::Initialize 中的 transport_controller_创建。



### 5.1 JsepTransportController::SetLocalDescription

pc/jsep_transport_controller.cc

```c++
RTCError JsepTransportController::SetLocalDescription(
    SdpType type,
    const cricket::SessionDescription* description) {
  // 切换到 network thread
  if (!network_thread_->IsCurrent()) {
    return network_thread_->Invoke<RTCError>(
        RTC_FROM_HERE, [=] { return SetLocalDescription(type, description); });
  }

    // 设置ICE控制角色，本端是协商的主导者，还是对端是主导者
  if (!initial_offerer_.has_value()) {
    initial_offerer_.emplace(type == SdpType::kOffer);
    if (*initial_offerer_) {
      SetIceRole_n(cricket::ICEROLE_CONTROLLING);
    } else {
      SetIceRole_n(cricket::ICEROLE_CONTROLLED);
    }
  }
  return ApplyDescription_n(/*local=*/true, type, description);
}
```

- 切换到network thread
- 设置 ice 的角色， 这里设置的是ICEROLE_CONTROLLING



### 5.2 !!!--JsepTransportController.ApplyDescription_n

pc/jsep_transport_controller.cc
详见小节6。



## 6. !!! JsepTransportController.ApplyDescription_n

pc/jsep_transport_controller.cc

```c++
RTCError JsepTransportController::ApplyDescription_n(
    bool local,
    SdpType type,    
    const cricket::SessionDescription* description) {
    ...

  if (local) {
    local_desc_ = description;
  } else {
    remote_desc_ = description;
  }

  RTCError error;
  // 0. 赋值ContentGroup bundle_group_，并校验有效性
  error = ValidateAndMaybeUpdateBundleGroup(local, type, description);
  if (!error.ok()) {
    return error;
  }

  std::vector<int> merged_encrypted_extension_ids;
  if (bundle_group_) {
    merged_encrypted_extension_ids =
        MergeEncryptedHeaderExtensionIdsForBundle(description);
  }
    // 1. 遍历sdp中所有ContentInfo,也即m Section的表征，mline
    // 针对每个 ContentInfo,(即每个sdp) 内容创建一个对应的 JsepTransport
  for (const cricket::ContentInfo& content_info : description->contents()) {
    // Don't create transports for rejected m-lines and bundled m-lines."
     // a. 被拒绝的content是无效的，因此，我们不应该为其创建Transport；
    //  b. 如果Content属于一个bundle，却又不是该bundle的第一个content，那么我们应该也要
    // 忽略该content，“因为属于一个bundle的content共享一个Transport进行传输，
    // 在遍历该bundle的第一个content时会去创建这个共享的Transport。
    if (content_info.rejected ||
        (IsBundled(content_info.name) && content_info.name != *bundled_mid())) {
      continue;
    }
    // 为每一个bundle 创建 共享JsepTransport
    // 没有bundle，则每个contentInfo创建 JsepTransport
    error = MaybeCreateJsepTransport(local, content_info, *description);
    if (!error.ok()) {
      return error;
    }
  }

  RTC_DCHECK(description->contents().size() ==
             description->transport_infos().size());
  for (size_t i = 0; i < description->contents().size(); ++i) {
    const cricket::ContentInfo& content_info = description->contents()[i];
    const cricket::TransportInfo& transport_info =
        description->transport_infos()[i];
    if (content_info.rejected) {
      //处理 rejected 的 contentinfo
      HandleRejectedContent(content_info, description);
      continue;
    }

    // 如果Content属于一个bundle，却又不是该bundle的第一个content
    if (IsBundled(content_info.name) && content_info.name != *bundled_mid()) {
      if (!HandleBundledContent(content_info)) {
        return RTCError(RTCErrorType::INVALID_PARAMETER,
                        "Failed to process the bundled m= section with mid='" +
                            content_info.name + "'.");
      }
      continue;
    }

    error = ValidateContent(content_info);
    if (!error.ok()) {
      return error;
    }

    std::vector<int> extension_ids;
    if (bundled_mid() && content_info.name == *bundled_mid()) {
      extension_ids = merged_encrypted_extension_ids;
    } else {
      extension_ids = GetEncryptedHeaderExtensionIds(content_info);
    }

    int rtp_abs_sendtime_extn_id =
        GetRtpAbsSendTimeHeaderExtensionId(content_info);

    // 2. 根据 content_info.name 获取JsepTransport 
    cricket::JsepTransport* transport =
        GetJsepTransportForMid(content_info.name);
    RTC_DCHECK(transport);

    SetIceRole_n(DetermineIceRole(transport, transport_info, type, local));

    // 3. 创建 JsepTransportDescription
    cricket::JsepTransportDescription jsep_description =
        CreateJsepTransportDescription(content_info, transport_info,
                                       extension_ids, rtp_abs_sendtime_extn_id);
    // 4. JsepTransportDescription 存到 JsepTransport
    if (local) {
      error =
          transport->SetLocalJsepTransportDescription(jsep_description, type);
    } else {
      error =
          transport->SetRemoteJsepTransportDescription(jsep_description, type);
    }

    if (!error.ok()) {
      LOG_AND_RETURN_ERROR(
          RTCErrorType::INVALID_PARAMETER,
          "Failed to apply the description for m= section with mid='" +
              content_info.name + "': " + error.message());
    }
  }
  if (type == SdpType::kAnswer) {
    pending_mids_.clear();
  }
  return RTCError::OK();
}
```

1. 赋值JsepTransportController.bundle_group_ 。
   由JsepTransportController.bundle_group_ 成员可知，实际应用中，SDP中一般只有一个bundle group。绝大多数情况下都是采用bundle形式进行传输，此时，这样可以减少底层Transport的数量，因此，也能减少需要分配的端口数。如下SDP的示例，4个mline都属于一个group，这个group的名字为默认的BUNDLE，其中0 1 2 3为4个mid值。这4个m section所代表的媒体将采用同一个JsepTransport

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/20200517222108730.png)

2. 遍历sdp中所有ContentInfo,也即m Section的表征，mline，针对每个 ContentInfo,(即每个sdp) 内容创建一个对应的 JsepTransport。那些情况需要创建，哪些不需要创建Transport：
   **a. 被拒绝的content是无效的，因此，不应该为其创建Transport；**
   **b. 如果Content属于一个bundle，却又不是该bundle的第一个content，那么我们应该也要忽略该content**，“因为属于一个bundle的content共享一个Transport进行传输，在遍历该bundle的第一个content时会去创建这个共享的Transport。
   `MaybeCreateJsepTransport` 这里就创建 JsepTransport。
3. 其实每个 content 就是 SDP 里面的一个 m 媒体段，具体参考博客 https://blog.csdn.net/freeabc/article/details/109784860
4. 创建JsepTransportDescription 对象，并对该对象进行填充 通过SetLocalJsepTransportDescription函数, 并JsepTransportDescription 存到 JsepTransport。



### 6.1 JsepTransportController::ValidateAndMaybeUpdateBundleGroup

pc/jsep_transport_controller.cc

```cpp
RTCError JsepTransportController::ValidateAndMaybeUpdateBundleGroup(
    bool local,
    SdpType type,
    const cricket::SessionDescription* description) {

  // 1. 从 SessionDescription 获取 ContentGroup, 包含 mline name
  const cricket::ContentGroup* new_bundle_group =
      description->GetGroupByName(cricket::GROUP_TYPE_BUNDLE);

    ...

  if (type == SdpType::kAnswer) {
    ...
  }
    ...

    // 2. 赋值 bundle_group_
  if (ShouldUpdateBundleGroup(type, description)) {
    bundle_group_ = *new_bundle_group;
  }
    ...

  return RTCError::OK();
}
```

- bundle_group_ 赋值



#### ContentGroup

pc/session_description.h

```cpp
class ContentGroup {
  const std::string* FirstContentName() const;

  const std::string* ContentGroup::FirstContentName() const {
      return (!content_names_.empty()) ? &(*content_names_.begin()) : NULL;
    }

  std::string semantics_;

    typedef std::vector<std::string> ContentNames;
  ContentNames content_names_;
}
```



### 6.2 !!!--JsepTransportController.MaybeCreateJsepTransport

pc/jsep_transport_controller.cc

为bundle，和没有bundle的 contentInfo 创建 JsepTransport，详细见【章节7】



### 6.3 --JsepTransportController::HandleRejectedContent

pc/jsep_transport_controller.cc

```cpp
void JsepTransportController::HandleRejectedContent(
    const cricket::ContentInfo& content_info,
    const cricket::SessionDescription* description) {
  
  // 从mid_to_transport_ map中 删除JsepTransport
  RemoveTransportForMid(content_info.name);
  // 如果是first mid，则 删除该bundle所有的mid的JsepTransport
  if (content_info.name == bundled_mid()) {
    for (const auto& content_name : bundle_group_->content_names()) {
      RemoveTransportForMid(content_name);
    }
    bundle_group_.reset();
  } else if (IsBundled(content_info.name)) {
    // Remove the rejected content from the |bundle_group_|.
    bundle_group_->RemoveContentName(content_info.name);
    // Reset the bundle group if nothing left.
    // 如果bundle没有存在mline，则重置bundle_group_
    if (!bundle_group_->FirstContentName()) {
      bundle_group_.reset();
    }
  }
  // 销毁JsepTransport
  MaybeDestroyJsepTransport(content_info.name);
}
```

被reject的 m line，对其做处理

- `std::map<std::string, cricket::JsepTransport*> mid_to_transport_;` 从列表中去处；
- 如果是bundle 的first mid == rejectd的 mid，则 整个bundle中的mid都需要从`mid_to_transport_`列表中去处；重置bundle_group_；
- 如果是bundle 的first mid != rejectd的 mid, 但是 bundle 包含了该mid， 则从bundle_group_ 去除该mid；同时如果`bundle_group_`没有mid了，则重置`bundle_group_`；
- 只有在没有存在共用的bundle， 才会销毁JsepTransport,；

> **bundle 首个mid被reject，也是删除所有该bundle的mid的；**



### 6.4 ??? --JsepTransportController::HandleBundledContent

pc/jsep_transport_controller.cc

```cpp
bool JsepTransportController::HandleBundledContent(
    const cricket::ContentInfo& content_info) {
  auto jsep_transport = GetJsepTransportByName(*bundled_mid());
  RTC_DCHECK(jsep_transport);
  // If the content is bundled, let the
  // BaseChannel/SctpTransport change the RtpTransport/DtlsTransport first,
  // then destroy the cricket::JsepTransport.
  if (SetTransportForMid(content_info.name, jsep_transport)) {
    // TODO(bugs.webrtc.org/9719) For media transport this is far from ideal,
    // because it means that we first create media transport and start
    // connecting it, and then we destroy it. We will need to address it before
    // video path is enabled.
    MaybeDestroyJsepTransport(content_info.name);
    return true;
  }
  return false;
}
```

- 如果Content属于一个bundle，却又不是该bundle的第一个content
- 
  

### 6.5 JsepTransportController::GetJsepTransportForMid

pc/jsep_transport_controller.cc

```c++
const cricket::JsepTransport* JsepTransportController::GetJsepTransportForMid(
    const std::string& mid) const {
  auto it = mid_to_transport_.find(mid);
  return it == mid_to_transport_.end() ? nullptr : it->second;
}

cricket::JsepTransport* JsepTransportController::GetJsepTransportForMid(
    const std::string& mid) {
  auto it = mid_to_transport_.find(mid);
  return it == mid_to_transport_.end() ? nullptr : it->second;
}
```

通过mid 找到JsepTransport。


### 6.6 JsepTransportController::CreateJsepTransportDescription

pc/jsep_transport_controller.cc

```c++
cricket::JsepTransportDescription
JsepTransportController::CreateJsepTransportDescription(
    const cricket::ContentInfo& content_info,
    const cricket::TransportInfo& transport_info,
    const std::vector<int>& encrypted_extension_ids,
    int rtp_abs_sendtime_extn_id) {
  const cricket::MediaContentDescription* content_desc =
      content_info.media_description();
  RTC_DCHECK(content_desc);
  bool rtcp_mux_enabled = content_info.type == cricket::MediaProtocolType::kSctp
                              ? true
                              : content_desc->rtcp_mux();

  return cricket::JsepTransportDescription(
      rtcp_mux_enabled, content_desc->cryptos(), encrypted_extension_ids,
      rtp_abs_sendtime_extn_id, transport_info.description);
}
```

创建CreateJsepTransportDescription。



### 6.7JsepTransportDescription::JsepTransportDescription

pc\jsep_transport.cc



### 6.8 JsepTransport.SetLocalJsepTransportDescription

pc\jsep_transport.cc

填充JsepTransportDescription



## 7. !!! JsepTransportController.MaybeCreateJsepTransport

pc/jsep_transport_controller.cc

```c++
RTCError JsepTransportController::MaybeCreateJsepTransport(
    bool local,
    const cricket::ContentInfo& content_info,
    const cricket::SessionDescription& description) {
  RTC_DCHECK(network_thread_->IsCurrent());
  //     如果对应的JsepTransport已经存在则返回就好了
  cricket::JsepTransport* transport = GetJsepTransportByName(content_info.name);
  if (transport) {
    return RTCError::OK();
  }
  // 判断Content中的媒体描述中是否存在加密参数，这些参数是给SDES使用的
  //    而JsepTransportController.certificate_是给DTLS-SRTP使用的
  //    二者不可同时存在，因此，需要做下判断。
  const cricket::MediaContentDescription* content_desc =
      content_info.media_description();
  if (certificate_ && !content_desc->cryptos().empty()) {
    return RTCError(RTCErrorType::INVALID_PARAMETER,
                    "SDES and DTLS-SRTP cannot be enabled at the same time.");
  }

  // 1. 重要, 这里产生的 ice 对象，其实就是 P2PTransportChannel 对象。
    // 创建ice层的传输对象，负责管理Candidates，联通性检测，收发数据包。
  // 注意，使用的是共享智能指针保存的。
  rtc::scoped_refptr<webrtc::IceTransportInterface> ice =
      CreateIceTransport(content_info.name, /*rtcp=*/false);
  RTC_DCHECK(ice);

  // 2. 创建dtls层的传输对象——>提供Dtls握手逻辑，密钥交换。
  // 注意其dtls内置了ice层的传输对象，其层次与DatagramTransport是平行关系
  std::unique_ptr<cricket::DtlsTransportInternal> rtp_dtls_transport =
      CreateDtlsTransport(content_info, ice->internal());

  std::unique_ptr<cricket::DtlsTransportInternal> rtcp_dtls_transport;
  std::unique_ptr<RtpTransport> unencrypted_rtp_transport;
  std::unique_ptr<SrtpTransport> sdes_transport;
  std::unique_ptr<DtlsSrtpTransport> dtls_srtp_transport;
  std::unique_ptr<RtpTransportInternal> datagram_rtp_transport;

  // 如果RTCP与RTP不复用，并且媒体是使用RTP协议传输的，则需要创建属于传输RTCP的
  //    ice层的传输对象，以及dtls层的传输对象
  rtc::scoped_refptr<webrtc::IceTransportInterface> rtcp_ice;
  if (config_.rtcp_mux_policy !=
          PeerConnectionInterface::kRtcpMuxPolicyRequire &&
      content_info.type == cricket::MediaProtocolType::kRtp) {
    rtcp_ice = CreateIceTransport(content_info.name, /*rtcp=*/true);
    rtcp_dtls_transport =
        CreateDtlsTransport(content_info, rtcp_ice->internal());
  }

  // 根据是否使用加密以及加密手段，来创建RTP层不同的传输对象
  // 不使用加密，则创建不使用加密手段的rtp层传输对象——>RtpTransport
  //  注意：仍然传递了dtls层的传输对象，但该对象可以不进行加密，直接将上层的
  //         包传递给ice层传输对象。
  if (config_.disable_encryption) {
    RTC_LOG(LS_INFO)
        << "Creating UnencryptedRtpTransport, becayse encryption is disabled.";
    unencrypted_rtp_transport = CreateUnencryptedRtpTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
  } else if (!content_desc->cryptos().empty()) {
    // 使用SDES加密，因为sdp中包含了加密参数
    // 创建使用SDES加密的rtp层传输对象——>SrtpTransport
    //  注意：传递了dtls层的传输对象，rtp层传输对象是其上层，但不使用dtls层的传输对象
    //  的握手，可以提高媒体建立链路的速度。
    sdes_transport = CreateSdesTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
    RTC_LOG(LS_INFO) << "Creating SdesTransport.";
  } else {
    RTC_LOG(LS_INFO) << "Creating DtlsSrtpTransport.";
    // 3. 使用dtls加密
    //     创建使用dtls加密手段的rtp层传输对象——>DtlsSrtpTransport
    //     注意：传递了dtls层的传输对象，rtp层传输对象是其上层，使用dtls层的传输对象提供的
    //  握手和加密，建立安全通信速度慢。
    dtls_srtp_transport = CreateDtlsSrtpTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
  }
  // 创建SCTP通道用于传输应用数据——>SctpTransport
  //     注意，其底层dtls层传输对象，使用dtls加密传输
  std::unique_ptr<cricket::SctpTransportInternal> sctp_transport;
  if (config_.sctp_factory) {
    sctp_transport =
        config_.sctp_factory->CreateSctpTransport(rtp_dtls_transport.get());
  }

  //     4. 创建JsepTransport对象，来容纳之前创建的各层对象
  //     ice层：两个传输对象ice、rtcp_ice；
  //     dtls层/datagram层：rtp_dtls_transport、rtcp_dtls_transport/datagram_transport
  //     基于dtls层的rtp层：unencrypted_rtp_transport、sdes_transport、dtls_srtp_transport
  //     基于datagram层的rtp层：datagram_rtp_transport
  //     基于dtls层的应用数据传输层：sctp_transport
  //     基于datagram层的应用数据传输层：data_channel_transport
  std::unique_ptr<cricket::JsepTransport> jsep_transport =
      std::make_unique<cricket::JsepTransport>(
          content_info.name, certificate_, std::move(ice), std::move(rtcp_ice),
          std::move(unencrypted_rtp_transport), std::move(sdes_transport),
          std::move(dtls_srtp_transport), std::move(datagram_rtp_transport),
          std::move(rtp_dtls_transport), std::move(rtcp_dtls_transport),
          std::move(sctp_transport));

  // 绑定JsepTransport信号-JsepTransportController槽
  jsep_transport->rtp_transport()->SignalRtcpPacketReceived.connect(
      this, &JsepTransportController::OnRtcpPacketReceived_n);

  jsep_transport->SignalRtcpMuxActive.connect(
      this, &JsepTransportController::UpdateAggregateStates_n);
  // 将创建的JsepTransport添加到JsepTransportController的成员上
  // 添加到mid_to_transport_
  // 参考【章节2.2.3】
  SetTransportForMid(content_info.name, jsep_transport.get());
    //添加到jsep_transports_by_name_
  // 后面的根据名称获取的 jsep_transport 就是这里产生的
  jsep_transports_by_name_[content_info.name] = std::move(jsep_transport);
  // 更新状态，进行ICE
  UpdateAggregateStates_n();
  return RTCError::OK();
}
```

1.   这里产生的 ice 对象，其实就是 P2PTransportChannel 对象。创建ice层的传输对象，负责管理Candidates，联通性检测，收发数据包。

2.   创建dtls层的传输对象，提供Dtls握手逻辑，密钥交换。注意其dtls内置了ice层的传输对象，其层次与DatagramTransport是平行关系。

3.   根据是否使用加密以及加密手段，来创建RTP层不同的传输对象

   - 不使用加密，则创建不使用加密手段的rtp层传输对象——>RtpTransport

     > 注意：仍然传递了dtls层的传输对象，但该对象可以不进行加密，直接将上层的包传递给ice层传输对象。

   - 使用SDES加密，因为sdp中包含了密钥。创建使用SDES加密的rtp层传输对象——>SrtpTransport

     > 注意：传递了dtls层的传输对象，rtp层传输对象是其上层，但不使用dtls层的传输对象的握手，可以提高媒体建立链路的速度。

   - 使用dtls加密。创建使用dtls加密手段的rtp层传输对象——>DtlsSrtpTransport

     > 注意：传递了dtls层的传输对象，rtp层传输对象是其上层，使用dtls层的传输对象提供的握手和加密，建立安全通信速度慢。

4. 创建 JsepTransport 对象。来容纳之前创建的各层对象
   ice层：两个传输对象ice、rtcp_ice；
   dtls层/datagram层：rtp_dtls_transport、rtcp_dtls_transport/datagram_transport
   基于dtls层的rtp层：unencrypted_rtp_transport、sdes_transport、dtls_srtp_transport
   基于datagram层的rtp层：datagram_rtp_transport
   基于dtls层的应用数据传输层：sctp_transport
   基于datagram层的应用数据传输层：data_channel_transport

5. `SetTransportForMid`, 将创建的JsepTransport添加到JsepTransportController的成员上添加到mid_to_transport_。

6. 更新状态，进行ICE

7. 这个函数根据 SDP 的内容创建各种各样的 transport ，这里我们看到熟悉的接口 CreateDtlsSrtpTransport ，这个创建了 webrtc::DtlsSrtpTransport 对象，webrtc::DtlsSrtpTransport 对象下面的两个成员对象非常关键。



### transport关系图

<img src={{ site.url }}{{ site.baseurl }}/images/3.setLocalSDP.assets/transport.jpg />

1. RtpTransport 继承关系

   | Transport            | 说明                                                         |
   | -------------------- | ------------------------------------------------------------ |
   | RtpTransportInternal | lpc/rtp_transport_internal.h；基础类，虚基类；               |
   | RtpTransport         | 非加密的 Transport 类，rtp 是不加密的                        |
   | SrtpTransport        | RTP 进行 protecting/unprotecting，底层用的是 SrtpSession，而 SrtpSession 是对 libsrtp 的封装 |
   | DtlsSrtpTransport    | 维护了两个 DtlsTransport 对象，分别用作 rtp 和 rtcp 会话的建立。当 DtlsTransport 完成协商后，获取加密套件，最终是通过 OpenSSLStreamAdapter::GetDtlsSrtpCryptoSuite 方法实现的 |

2. DtlsTransport， 是DTLS 会话协商过程的控制逻辑。

3. P2PTransportChannel 是连接底层 UDP socket 层和 Transport 的纽带。是 Candidate-flex Address 收集的入口，是 P2P 穿越的入口，穿越成功后，会建立连接，管理着 Connection 对象。

4. JsepTransport 是一个辅助、包装类，实现 Jsep 接口，同时持有 RtpTransport、SrtpTransport、DtlsSrtpTransport 等。

5. JsepTransportController，遵循 JSEP 规范的，实现 Jsep 接口；负载管理各种类型 Transport 对象的**创建、获取、销毁**；是 Peer Connection 到 transport 层的入口，设置远端 candidate，启动 p2p 连接穿越过程。

6. OpenSSLStreamAdapter 是对 openssl 的包装，主要是实现 dtls 协议的应用

7. DtlsSrtpTransport ， DtlsTransport 不一样，注意注意！！！



### !!! jsep_transports_by_name_ && mid_to_transport_

pc/jsep_transport_controller.cc

注意不同时间的用法。



### 7.1 JsepTransportController::GetJsepTransportByName

pc/jsep_transport_controller.cc

```c++
const cricket::JsepTransport* JsepTransportController::GetJsepTransportByName(
    const std::string& transport_name) const {
  auto it = jsep_transports_by_name_.find(transport_name);
  return (it == jsep_transports_by_name_.end()) ? nullptr : it->second.get();
}

cricket::JsepTransport* JsepTransportController::GetJsepTransportByName(
    const std::string& transport_name) {
  auto it = jsep_transports_by_name_.find(transport_name);
  return (it == jsep_transports_by_name_.end()) ? nullptr : it->second.get();
}
```

通过name( 这个 就是 mid) 获取 JsepTransport；



### 7.2 JsepTransportController::CreateIceTransport

pc/jsep_transport_controller.cc

```c++
rtc::scoped_refptr<webrtc::IceTransportInterface>
JsepTransportController::CreateIceTransport(const std::string& transport_name,
                                            bool rtcp) {
  int component = rtcp ? cricket::ICE_CANDIDATE_COMPONENT_RTCP
                       : cricket::ICE_CANDIDATE_COMPONENT_RTP;

  IceTransportInit init;
  init.set_port_allocator(port_allocator_);
  init.set_async_resolver_factory(async_resolver_factory_);
  init.set_event_log(config_.event_log);
  //  ice_transport_factory 参见源码中的 std::make_unique<DefaultIceTransportFactory>();
  return config_.ice_transport_factory->CreateIceTransport(
      transport_name, component, std::move(init));
}
```



#### DefaultIceTransportFactory.CreateIceTransport

创建P2PTransportChannel。

```c++
rtc::scoped_refptr<IceTransportInterface>
DefaultIceTransportFactory::CreateIceTransport(
    const std::string& transport_name,
    int component,
    IceTransportInit init) {
  BasicIceControllerFactory factory;
  return new rtc::RefCountedObject<DefaultIceTransport>(
      std::make_unique<cricket::P2PTransportChannel>(
          transport_name, component, init.port_allocator(),
          init.async_resolver_factory(), init.event_log(), &factory));
}
```



### 7.3 JsepTransportController::CreateDtlsTransport

pc/jsep_transport_controller.cc

```c++
std::unique_ptr<cricket::DtlsTransportInternal>
JsepTransportController::CreateDtlsTransport(
    const cricket::ContentInfo& content_info,
    cricket::IceTransportInternal* ice) {
  RTC_DCHECK(network_thread_->IsCurrent());

  std::unique_ptr<cricket::DtlsTransportInternal> dtls;

  if (config_.dtls_transport_factory) {
    dtls = config_.dtls_transport_factory->CreateDtlsTransport(
        ice, config_.crypto_options);
  } else {
    // 创建dtls
    dtls = std::make_unique<cricket::DtlsTransport>(ice, config_.crypto_options,
                                                    config_.event_log);
  }

  RTC_DCHECK(dtls);
  dtls->SetSslMaxProtocolVersion(config_.ssl_max_version);
  dtls->ice_transport()->SetIceRole(ice_role_);
  dtls->ice_transport()->SetIceTiebreaker(ice_tiebreaker_);
  dtls->ice_transport()->SetIceConfig(ice_config_);
  if (certificate_) {
    bool set_cert_success = dtls->SetLocalCertificate(certificate_);
    RTC_DCHECK(set_cert_success);
  }

  // Connect to signals offered by the DTLS and ICE transport.
  dtls->SignalWritableState.connect(
      this, &JsepTransportController::OnTransportWritableState_n);
  dtls->SignalReceivingState.connect(
      this, &JsepTransportController::OnTransportReceivingState_n);
  dtls->SignalDtlsHandshakeError.connect(
      this, &JsepTransportController::OnDtlsHandshakeError);
  dtls->ice_transport()->SignalGatheringState.connect(
      this, &JsepTransportController::OnTransportGatheringState_n);
  dtls->ice_transport()->SignalCandidateGathered.connect(
      this, &JsepTransportController::OnTransportCandidateGathered_n);
  dtls->ice_transport()->SignalCandidateError.connect(
      this, &JsepTransportController::OnTransportCandidateError_n);
  dtls->ice_transport()->SignalCandidatesRemoved.connect(
      this, &JsepTransportController::OnTransportCandidatesRemoved_n);
  dtls->ice_transport()->SignalRoleConflict.connect(
      this, &JsepTransportController::OnTransportRoleConflict_n);
  dtls->ice_transport()->SignalStateChanged.connect(
      this, &JsepTransportController::OnTransportStateChanged_n);
  dtls->ice_transport()->SignalIceTransportStateChanged.connect(
      this, &JsepTransportController::OnTransportStateChanged_n);
  dtls->ice_transport()->SignalCandidatePairChanged.connect(
      this, &JsepTransportController::OnTransportCandidatePairChanged_n);
  return dtls;
}
```

创建 DtlsTransport；查看关系图。



### 7.4 JsepTransportController::CreateUnencryptedRtpTransport

pc/jsep_transport_controller.cc

```cpp
std::unique_ptr<webrtc::RtpTransport>
JsepTransportController::CreateUnencryptedRtpTransport(
    const std::string& transport_name,
    rtc::PacketTransportInternal* rtp_packet_transport, // DtlsTransport
    rtc::PacketTransportInternal* rtcp_packet_transport) {
  RTC_DCHECK(network_thread_->IsCurrent());
  auto unencrypted_rtp_transport =
      std::make_unique<RtpTransport>(rtcp_packet_transport == nullptr);
  unencrypted_rtp_transport->SetRtpPacketTransport(rtp_packet_transport);
  if (rtcp_packet_transport) {
    unencrypted_rtp_transport->SetRtcpPacketTransport(rtcp_packet_transport);
  }
  return unencrypted_rtp_transport;
}
```

创建RtpTransport。


### 7.5 JsepTransportController::CreateSdesTransport

pc/jsep_transport_controller.cc

```cpp
std::unique_ptr<webrtc::SrtpTransport>
JsepTransportController::CreateSdesTransport(
    const std::string& transport_name,
    cricket::DtlsTransportInternal* rtp_dtls_transport, // DtlsTransport
    cricket::DtlsTransportInternal* rtcp_dtls_transport) {
  RTC_DCHECK(network_thread_->IsCurrent());
  auto srtp_transport =
      std::make_unique<webrtc::SrtpTransport>(rtcp_dtls_transport == nullptr);
  RTC_DCHECK(rtp_dtls_transport);
  srtp_transport->SetRtpPacketTransport(rtp_dtls_transport);
  if (rtcp_dtls_transport) {
    srtp_transport->SetRtcpPacketTransport(rtcp_dtls_transport);
  }
  if (config_.enable_external_auth) {
    srtp_transport->EnableExternalAuth();
  }
  return srtp_transport;
}
```

创建SrtpTransport。


### 7.6 JsepTransportController::CreateDtlsSrtpTransport

```c++
std::unique_ptr<webrtc::DtlsSrtpTransport>
JsepTransportController::CreateDtlsSrtpTransport(
    const std::string& transport_name,
    cricket::DtlsTransportInternal* rtp_dtls_transport, // DtlsTransport
    cricket::DtlsTransportInternal* rtcp_dtls_transport) {
  RTC_DCHECK(network_thread_->IsCurrent());
  auto dtls_srtp_transport = std::make_unique<webrtc::DtlsSrtpTransport>(
      rtcp_dtls_transport == nullptr);
  if (config_.enable_external_auth) {
    dtls_srtp_transport->EnableExternalAuth();
  }

  dtls_srtp_transport->SetDtlsTransports(rtp_dtls_transport,
                                         rtcp_dtls_transport);
  dtls_srtp_transport->SetActiveResetSrtpParams(
      config_.active_reset_srtp_params);
  dtls_srtp_transport->SignalDtlsStateChange.connect(
      this, &JsepTransportController::UpdateAggregateStates_n);
  return dtls_srtp_transport;
}
```

DTLS-SRTP， 是 DTLS(Datagram Transport Layer Security)即数据包传输层安全性*协议*。[TLS](https://baike.baidu.com/item/TLS/2979545)不能用来保证[UDP](https://baike.baidu.com/item/UDP/571511)上传输的数据的安全，因此Datagram TLS试图在现存的[TLS协议](https://baike.baidu.com/item/TLS%E5%8D%8F%E8%AE%AE/7129331)架构上提出扩展，使之支持UDP，即成为TLS的一个支持数据包传输的版本。



### 7.7 JsepTransport.JsepTransport

```c++
JsepTransport::JsepTransport(
    const std::string& mid,
    const rtc::scoped_refptr<rtc::RTCCertificate>& local_certificate,
    rtc::scoped_refptr<webrtc::IceTransportInterface> ice_transport,
    rtc::scoped_refptr<webrtc::IceTransportInterface> rtcp_ice_transport,
    std::unique_ptr<webrtc::RtpTransport> unencrypted_rtp_transport,
    std::unique_ptr<webrtc::SrtpTransport> sdes_transport,
    std::unique_ptr<webrtc::DtlsSrtpTransport> dtls_srtp_transport,
    std::unique_ptr<webrtc::RtpTransportInternal> datagram_rtp_transport,
    std::unique_ptr<DtlsTransportInternal> rtp_dtls_transport,
    std::unique_ptr<DtlsTransportInternal> rtcp_dtls_transport,
    std::unique_ptr<SctpTransportInternal> sctp_transport)
    : network_thread_(rtc::Thread::Current()),
      mid_(mid),
      local_certificate_(local_certificate),
      ice_transport_(std::move(ice_transport)),
      rtcp_ice_transport_(std::move(rtcp_ice_transport)),
      unencrypted_rtp_transport_(std::move(unencrypted_rtp_transport)),
      sdes_transport_(std::move(sdes_transport)),
      dtls_srtp_transport_(std::move(dtls_srtp_transport)),
      rtp_dtls_transport_(
          rtp_dtls_transport ? new rtc::RefCountedObject<webrtc::DtlsTransport>(
                                   std::move(rtp_dtls_transport))
                             : nullptr),
     
			...
}
```

保存了所有的Transport。



```c++
// Ice transport which may be used by any of upper-layer transports (below).
  // Owned by JsepTransport and guaranteed to outlive the transports below.
  const rtc::scoped_refptr<webrtc::IceTransportInterface> ice_transport_;
  const rtc::scoped_refptr<webrtc::IceTransportInterface> rtcp_ice_transport_;

  // To avoid downcasting and make it type safe, keep three unique pointers for
  // different SRTP mode and only one of these is non-nullptr.
  std::unique_ptr<webrtc::RtpTransport> unencrypted_rtp_transport_
      RTC_GUARDED_BY(accessor_lock_);
  std::unique_ptr<webrtc::SrtpTransport> sdes_transport_
      RTC_GUARDED_BY(accessor_lock_);
  std::unique_ptr<webrtc::DtlsSrtpTransport> dtls_srtp_transport_
      RTC_GUARDED_BY(accessor_lock_);


  rtc::scoped_refptr<webrtc::DtlsTransport> rtp_dtls_transport_
      RTC_GUARDED_BY(accessor_lock_);
  rtc::scoped_refptr<webrtc::DtlsTransport> rtcp_dtls_transport_
      RTC_GUARDED_BY(accessor_lock_);
  rtc::scoped_refptr<webrtc::DtlsTransport> datagram_dtls_transport_
      RTC_GUARDED_BY(accessor_lock_);
```



### 7.8 !!! JsepTransportController.SetTransportForMid

pc/jsep_transport_controller.cc

```c++
  // This keeps track of the mapping between media section
  // (BaseChannel/SctpTransport) and the JsepTransport underneath.
std::map<std::string, cricket::JsepTransport*> mid_to_transport_;


bool JsepTransportController::SetTransportForMid(
    const std::string& mid,
    cricket::JsepTransport* jsep_transport) {
  RTC_DCHECK(jsep_transport);
  if (mid_to_transport_[mid] == jsep_transport) {
    return true;
  }
  RTC_DCHECK_RUN_ON(network_thread_);
  pending_mids_.push_back(mid);
  mid_to_transport_[mid] = jsep_transport;
  return config_.transport_observer->OnTransportChanged(
      mid, jsep_transport->rtp_transport(), jsep_transport->RtpDtlsTransport(),
      jsep_transport->data_channel_transport());
}
```

存JsepTransport到 `std::map<std::string, cricket::JsepTransport*> mid_to_transport_;`。
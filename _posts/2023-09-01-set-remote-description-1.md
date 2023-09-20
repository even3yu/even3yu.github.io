---
layout: post
title: webrtc setRemoteDescription-1
date: 2023-09-01 23:32:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


![setRemoteSDP]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/setRemoteSDP-6649445.svg)



![videoencodesdp]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/videoencodesdp.svg)

![videotrack]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/videotrack.svg)

![setRemoteSDP]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/setRemoteSDP-6663142.svg)


sdp是从callee 端接收到，所以SessionDescriptionInterface 是 answer。



## 1. PeerConnection::SetRemoteDescription

pc/peer_connection.cc

```c++
void PeerConnection::SetRemoteDescription(
    SetSessionDescriptionObserver* observer,
    SessionDescriptionInterface* desc_ptr) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  sdp_handler_->SetRemoteDescription(observer, desc_ptr);
}
```

1. SetSessionDescriptionObserver* observer 就是 SetSessionDescriptionObserverAdapter对象

2. SessionDescriptionInterface* desc_ptr 就是 JsepSessionDescription 对象

   ![image-20210325135009539](set-remote-description.assets/image-20210325135009539.png)



## 2. SdpOfferAnswerHandler::SetRemoteDescription

pc/sdp_offer_answer.cc

```c++
void SdpOfferAnswerHandler::SetRemoteDescription(
    SetSessionDescriptionObserver* observer,
    SessionDescriptionInterface* desc_ptr) {
	...
  operations_chain_->ChainOperation(
      [this_weak_ptr = weak_ptr_factory_.GetWeakPtr(),
       observer_refptr =
           rtc::scoped_refptr<SetSessionDescriptionObserver>(observer),
       desc = std::unique_ptr<SessionDescriptionInterface>(desc_ptr)](
          std::function<void()> operations_chain_callback) mutable {
        ...
          
        this_weak_ptr->DoSetRemoteDescription(
            std::move(desc),
            rtc::scoped_refptr<SetRemoteDescriptionObserverInterface>(
                new rtc::RefCountedObject<SetSessionDescriptionObserverAdapter>(
                    this_weak_ptr, observer_refptr)));
        ...
      });
}
```



### 2.1 SdpOfferAnswerHandler::SetSessionDescriptionObserverAdapter 

pc/sdp_offer_answer.cc


   ```c++
   // Wrapper for SetSessionDescriptionObserver that invokes the success or failure
   // callback in a posted message handled by the peer connection. This introduces
   // a delay that prevents recursive API calls by the observer, but this also
   // means that the PeerConnection can be modified before the observer sees the
   // result of the operation. This is ill-advised for synchronizing states.
   //
   // Implements both the SetLocalDescriptionObserverInterface and the
   // SetRemoteDescriptionObserverInterface.
   class SdpOfferAnswerHandler::SetSessionDescriptionObserverAdapter
       : public SetLocalDescriptionObserverInterface,
         public SetRemoteDescriptionObserverInterface {
    public:
     SetSessionDescriptionObserverAdapter(
         rtc::WeakPtr<SdpOfferAnswerHandler> handler,
         rtc::scoped_refptr<SetSessionDescriptionObserver> inner_observer)
         : handler_(std::move(handler)),
           inner_observer_(std::move(inner_observer)) {}
   
     // SetLocalDescriptionObserverInterface implementation.
     void OnSetLocalDescriptionComplete(RTCError error) override {
       OnSetDescriptionComplete(std::move(error));
     }
     // SetRemoteDescriptionObserverInterface implementation.
     void OnSetRemoteDescriptionComplete(RTCError error) override {
       OnSetDescriptionComplete(std::move(error));
     }
   
    private:
     void OnSetDescriptionComplete(RTCError error) {
       if (!handler_)
         return;
       if (error.ok()) {
         handler_->pc_->message_handler()->PostSetSessionDescriptionSuccess(
             inner_observer_);
       } else {
         handler_->pc_->message_handler()->PostSetSessionDescriptionFailure(
             inner_observer_, std::move(error));
       }
     }
   
     rtc::WeakPtr<SdpOfferAnswerHandler> handler_;
     rtc::scoped_refptr<SetSessionDescriptionObserver> inner_observer_;
   };
   ```



### 2.2 --SdpOfferAnswerHandler::DoSetRemoteDescription

pc/sdp_offer_answer.cc

参考【小结3】



## 3. SdpOfferAnswerHandler.DoSetRemoteDescription

pc/sdp_offer_answer.cc

```c++
void SdpOfferAnswerHandler::DoSetRemoteDescription(
    std::unique_ptr<SessionDescriptionInterface> desc,
    rtc::scoped_refptr<SetRemoteDescriptionObserverInterface> observer) {
 	...

  // Handle remote descriptions missing a=mid lines for interop with legacy end
  // points.
  FillInMissingRemoteMids(desc->description());

	...
    
  // Grab the description type before moving ownership to
  // ApplyRemoteDescription, which may destroy it before returning.
  const SdpType type = desc->GetType();

  error = ApplyRemoteDescription(std::move(desc));
  // |desc| may be destroyed at this point.

  if (!error.ok()) {
    ...
    return;
  }

  if (type == SdpType::kAnswer) {
    RemoveStoppedTransceivers();
    // TODO(deadbeef): We already had to hop to the network thread for
    // MaybeStartGathering...
    pc_->network_thread()->Invoke<void>(
        RTC_FROM_HERE, rtc::Bind(&cricket::PortAllocator::DiscardCandidatePool,
                                 port_allocator()));
    // Make UMA notes about what was agreed to.
    ReportNegotiatedSdpSemantics(*remote_description());
  }

  observer->OnSetRemoteDescriptionComplete(RTCError::OK());
  pc_->NoteUsageEvent(UsageEvent::SET_REMOTE_DESCRIPTION_SUCCEEDED);

  // Check if negotiation is needed. We must do this after informing the
  // observer that SetRemoteDescription() has completed to ensure negotiation is
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
}
```

1. SdpType type = desc->GetType(); 这里的 answer offer。
1. `rtc::scoped_refptr<SetRemoteDescriptionObserverInterface> observer` 是SdpOfferAnswerHandler::SetSessionDescriptionObserverAdapter对象。



### ??? SdpOfferAnswerHandler::FillInMissingRemoteMids

兼容性， mid丢失???



### --SdpOfferAnswerHandler::ApplyRemoteDescription

pc/sdp_offer_answer.cc



## 4. SdpOfferAnswerHandler::ApplyRemoteDescription

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::ApplyRemoteDescription(
    std::unique_ptr<SessionDescriptionInterface> desc) {
	...

  // 1. null
  const SessionDescriptionInterface* old_remote_description =
      remote_description();
  std::unique_ptr<SessionDescriptionInterface> replaced_remote_description;
  SdpType type = desc->GetType();
  if (type == SdpType::kAnswer) {
    // 2. pending_remote_description_ = null
		// 		current_remote_description_ = null
    replaced_remote_description = pending_remote_description_
                                      ? std::move(pending_remote_description_)
                                      : std::move(current_remote_description_);
    current_remote_description_ = std::move(desc);
    // 3. 清空了pending_remote_description_
    pending_remote_description_ = nullptr;
    // 4. 这里才是真正的赋值current_local_description_，pending_local_description_转正
    current_local_description_ = std::move(pending_local_description_);
  } else {
    ...
  }
  ...

  // Report statistics about any use of simulcast.
  ReportSimulcastApiVersion(kSimulcastVersionApplyRemoteDescription,
                            *remote_description()->description());

  // 3. 创建transport
  RTCError error = PushdownTransportDescription(cricket::CS_REMOTE, type);
  if (!error.ok()) {
    return error;
  }
  // Transport and Media channels will be created only when offer is set.
  if (IsUnifiedPlan()) {
    // 4. 
    RTCError error = UpdateTransceiversAndDataChannels(
        cricket::CS_REMOTE, *remote_description(), local_description(),
        old_remote_description);
    if (!error.ok()) {
      return error;
    }
  } else {
    ...
  }

  // 5. 
  // NOTE: Candidates allocation will be initiated only when
  // SetLocalDescription is called.
  error = UpdateSessionState(type, cricket::CS_REMOTE,
                             remote_description()->description());
  if (!error.ok()) {
    return error;
  }

  if (local_description() &&
      !UseCandidatesInSessionDescription(remote_description())) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER, kInvalidCandidates);
  }

  if (old_remote_description) {
    for (const cricket::ContentInfo& content :
         old_remote_description->description()->contents()) {
      // Check if this new SessionDescription contains new ICE ufrag and
      // password that indicates the remote peer requests an ICE restart.
      // TODO(deadbeef): When we start storing both the current and pending
      // remote description, this should reset pending_ice_restarts and compare
      // against the current description.
      if (CheckForRemoteIceRestart(old_remote_description, remote_description(),
                                   content.name)) {
        if (type == SdpType::kOffer) {
          pending_ice_restarts_.insert(content.name);
        }
      } else {
        // We retain all received candidates only if ICE is not restarted.
        // When ICE is restarted, all previous candidates belong to an old
        // generation and should not be kept.
        // TODO(deadbeef): This goes against the W3C spec which says the remote
        // description should only contain candidates from the last set remote
        // description plus any candidates added since then. We should remove
        // this once we're sure it won't break anything.
        WebRtcSessionDescriptionFactory::CopyCandidatesFromSessionDescription(
            old_remote_description, content.name, mutable_remote_description());
      }
    }
  }

  if (session_error() != SessionError::kNone) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INTERNAL_ERROR, GetSessionErrorMsg());
  }

  // Set the the ICE connection state to connecting since the connection may
  // become writable with peer reflexive candidates before any remote candidate
  // is signaled.
  // TODO(pthatcher): This is a short-term solution for crbug/446908. A real fix
  // is to have a new signal the indicates a change in checking state from the
  // transport and expose a new checking() member from transport that can be
  // read to determine the current checking state. The existing SignalConnecting
  // actually means "gathering candidates", so cannot be be used here.
  if (remote_description()->GetType() != SdpType::kOffer &&
      remote_description()->number_of_mediasections() > 0u &&
      pc_->ice_connection_state() ==
          PeerConnectionInterface::kIceConnectionNew) {
    pc_->SetIceConnectionState(PeerConnectionInterface::kIceConnectionChecking);
  }

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
      RtpTransceiverDirection local_direction =
          RtpTransceiverDirectionReversed(media_desc->direction());
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
      // 2.2.8.1.10: Set transceiver's [[FiredDirection]] slot to direction.
      transceiver->internal()->set_fired_direction(local_direction);
      // 2.2.8.1.11: If description is of type "answer" or "pranswer", then run
      // the following steps:
      if (type == SdpType::kPrAnswer || type == SdpType::kAnswer) {
        // 2.2.8.1.11.1: Set transceiver's [[CurrentDirection]] slot to
        // direction.
        transceiver->internal()->set_current_direction(local_direction);
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
      // 2.2.8.1.12: If the media description is rejected, and transceiver is
      // not already stopped, stop the RTCRtpTransceiver transceiver.
      if (content->rejected && !transceiver->stopped()) {
        RTC_LOG(LS_INFO) << "Stopping transceiver for MID=" << content->name
                         << " since the media section was rejected.";
        transceiver->internal()->StopTransceiverProcedure();
      }
      if (!content->rejected &&
          RtpTransceiverDirectionHasRecv(local_direction)) {
        if (!media_desc->streams().empty() &&
            media_desc->streams()[0].has_ssrcs()) {
          uint32_t ssrc = media_desc->streams()[0].first_ssrc();
          transceiver->internal()->receiver_internal()->SetupMediaChannel(ssrc);
        } else {
          transceiver->internal()
              ->receiver_internal()
              ->SetupUnsignaledMediaChannel();
        }
      }
    }
    // Once all processing has finished, fire off callbacks.
    auto observer = pc_->Observer();
    for (const auto& transceiver : now_receiving_transceivers) {
      pc_->stats()->AddTrack(transceiver->receiver()->track());
      observer->OnTrack(transceiver);
      observer->OnAddTrack(transceiver->receiver(),
                           transceiver->receiver()->streams());
    }
    for (const auto& stream : added_streams) {
      observer->OnAddStream(stream);
    }
    for (const auto& transceiver : remove_list) {
      observer->OnRemoveTrack(transceiver->receiver());
    }
    for (const auto& stream : removed_streams) {
      observer->OnRemoveStream(stream);
    }
  }

  const cricket::ContentInfo* audio_content =
      GetFirstAudioContent(remote_description()->description());
  const cricket::ContentInfo* video_content =
      GetFirstVideoContent(remote_description()->description());
  const cricket::AudioContentDescription* audio_desc =
      GetFirstAudioContentDescription(remote_description()->description());
  const cricket::VideoContentDescription* video_desc =
      GetFirstVideoContentDescription(remote_description()->description());
  const cricket::RtpDataContentDescription* rtp_data_desc =
      GetFirstRtpDataContentDescription(remote_description()->description());

  // Check if the descriptions include streams, just in case the peer supports
  // MSID, but doesn't indicate so with "a=msid-semantic".
  if (remote_description()->description()->msid_supported() ||
      (audio_desc && !audio_desc->streams().empty()) ||
      (video_desc && !video_desc->streams().empty())) {
    remote_peer_supports_msid_ = true;
  }

  // We wait to signal new streams until we finish processing the description,
  // since only at that point will new streams have all their tracks.
  rtc::scoped_refptr<StreamCollection> new_streams(StreamCollection::Create());

  if (!IsUnifiedPlan()) {
    // TODO(steveanton): When removing RTP senders/receivers in response to a
    // rejected media section, there is some cleanup logic that expects the
    // voice/ video channel to still be set. But in this method the voice/video
    // channel would have been destroyed by the SetRemoteDescription caller
    // above so the cleanup that relies on them fails to run. The RemoveSenders
    // calls should be moved to right before the DestroyChannel calls to fix
    // this.

    // Find all audio rtp streams and create corresponding remote AudioTracks
    // and MediaStreams.
    if (audio_content) {
      if (audio_content->rejected) {
        RemoveSenders(cricket::MEDIA_TYPE_AUDIO);
      } else {
        bool default_audio_track_needed =
            !remote_peer_supports_msid_ &&
            RtpTransceiverDirectionHasSend(audio_desc->direction());
        UpdateRemoteSendersList(GetActiveStreams(audio_desc),
                                default_audio_track_needed, audio_desc->type(),
                                new_streams);
      }
    }

    // Find all video rtp streams and create corresponding remote VideoTracks
    // and MediaStreams.
    if (video_content) {
      if (video_content->rejected) {
        RemoveSenders(cricket::MEDIA_TYPE_VIDEO);
      } else {
        bool default_video_track_needed =
            !remote_peer_supports_msid_ &&
            RtpTransceiverDirectionHasSend(video_desc->direction());
        UpdateRemoteSendersList(GetActiveStreams(video_desc),
                                default_video_track_needed, video_desc->type(),
                                new_streams);
      }
    }

    // If this is an RTP data transport, update the DataChannels with the
    // information from the remote peer.
    if (rtp_data_desc) {
      data_channel_controller()->UpdateRemoteRtpDataChannels(
          GetActiveStreams(rtp_data_desc));
    }

    // Iterate new_streams and notify the observer about new MediaStreams.
    auto observer = pc_->Observer();
    for (size_t i = 0; i < new_streams->count(); ++i) {
      MediaStreamInterface* new_stream = new_streams->at(i);
      pc_->stats()->AddStream(new_stream);
      observer->OnAddStream(
          rtc::scoped_refptr<MediaStreamInterface>(new_stream));
    }

    UpdateEndedRemoteMediaStreams();
  }

  if (type == SdpType::kAnswer &&
      local_ice_credentials_to_replace_->SatisfiesIceRestart(
          *current_local_description_)) {
    local_ice_credentials_to_replace_->ClearIceCredentials();
  }

  return RTCError::OK();
}
```



### 4.0 current_local_description_ pending_local_description_ current_remote_description_ pending_remote_description_ 赋值时机

| descrption                  | setLocalDescription（caller）                  | setRemoteDescription（caller）                               | setLocalDescription（callee）                                | setRemoteDescription（callee）                      |
| --------------------------- | ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------- |
| current_local_description_  |                                                | 4.1 收到对方的anwersdp， ApplyRemoteDescription赋值          | 3.1 创建ansersdp，ApplyLocalDescription赋值                  |                                                     |
| pending_local_description_  | 1. 本端createOffer，ApplyLocalDescription 赋值 | 4.3 收到对方的anwersdp，ApplyLocalDescription，pending_local_description_转正 | 3.2 创建ansersdp，ApplyLocalDescription 清空                 |                                                     |
| current_remote_description_ |                                                | 4.2 收到对方的anwersdp， ApplyRemoteDescription清空          | 3.3 创建ansersdp，ApplyLocalDescription，pending_remote_description_转正 |                                                     |
| pending_remote_description_ |                                                |                                                              |                                                              | 2. 远端收到createOffer，ApplyRemoteDescription 赋值 |



### 4.1 --SdpOfferAnswerHandler::PushdownTransportDescription

pc/sdp_offer_answer.cc
创建了JsepTransport，DtlsTransport
RtpTransport，SrtpTransport，DtlsSrtpTransport
JsepTransportController，JsepTransportDescription。
（mid， JsepTransport ）以这样的键值对存放，其中，如果Bundle 多个m Line， 那么以第一个mid 为key，共用JsepTransport；如果没有Bundle，那么每个mLine 都会生成JsepTransport；

### 4.2 --SdpOfferAnswerHandler.UpdateTransceiversAndDataChannels

pc/sdp_offer_answer.cc
创建BaseChannel。

### 4.3 --SdpOfferAnswerHandler::UpdateSessionState

pc/sdp_offer_answer.cc



## 5. SdpOfferAnswerHandler::PushdownTransportDescription

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::PushdownTransportDescription(
    cricket::ContentSource source,
    SdpType type) {
  ...

  if (source == cricket::CS_LOCAL) {
		...
  } else {
    // 就是DoSetRemoteDescription传入的参数，赋值了 current_remote_description_；
    const SessionDescriptionInterface* sdesc = remote_description();
    ...
    // transport_controller() 其实返回的就是peerConnection 中的 
    // JsepTransportController transport_controller_； 是在peerConnection
    // 的Initialize 中创建
    return transport_controller()->SetRemoteDescription(type,
                                                        sdesc->description());
  }
}
```

- remote_description(), pending_remote_description_ 是null，current_remote_description_ 就是就是DoSetRemoteDescription传入的参数，赋值的；
- transport_controller()， 其实返回的就是peerConnection::Initialize 中的 transport_controller_创建。



### 5.1 JsepTransportController::SetRemoteDescription

pc/jsep_transport_controller.cc

```c++
RTCError JsepTransportController::SetRemoteDescription(
    SdpType type,
    const cricket::SessionDescription* description) {
  // 切换到 network thread
  if (!network_thread_->IsCurrent()) {
    return network_thread_->Invoke<RTCError>(
        RTC_FROM_HERE, [=] { return SetRemoteDescription(type, description); });
  }

  return ApplyDescription_n(/*local=*/false, type, description);
}
```

- 切换到network thread



### 5.2 !!!--JsepTransportController.ApplyDescription_n

pc/jsep_transport_controller.cc
详见小节6。



## 6. !!! JsepTransportController.ApplyDescription_n

pc/jsep_transport_controller.cc

```c++
// description = current_remote_description_
RTCError JsepTransportController::ApplyDescription_n(
    bool local,
    SdpType type,    
    const cricket::SessionDescription* description) {
    ...

  if (local) {
    ...
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
    //
		// 注意，这里callee SetLocalDescription， 创建了JsepTransport， 
		// 所以这里 ContentInfo， 对应的JsepTransport已经创建，这里就直接用这个对象，不需要创建
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

![在这里插入图片描述](set-remote-description.assets/20200517222108730.png)

2. 遍历sdp中所有ContentInfo,也即m Section的表征，mline，针对每个 ContentInfo,(即每个sdp) 内容创建一个对应的 JsepTransport。那些情况需要创建，哪些不需要创建Transport：
   **a. 被拒绝的content是无效的，因此，不应该为其创建Transport；**
   **b. 如果Content属于一个bundle，却又不是该bundle的第一个content，那么我们应该也要忽略该content**，“因为属于一个bundle的content共享一个Transport进行传输，在遍历该bundle的第一个content时会去创建这个共享的Transport。
   `MaybeCreateJsepTransport` 这里就创建 JsepTransport。**由于callee SetLocalDescription， 创建了JsepTransport。所以这里 ContentInfo， 直接用这个对象，不需要创建。**
3. 其实每个 content 就是 SDP 里面的一个 m 媒体段，具体参考博客 https://blog.csdn.net/freeabc/article/details/109784860
4. 创建JsepTransportDescription 对象，并对该对象进行填充 通过SetRemoteJsepTransportDescription函数, 并JsepTransportDescription 存到 JsepTransport。



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



### 6.8 JsepTransport.SetRemoteJsepTransportDescription

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
  ...
  return RTCError::OK();
}
```

1. 这里产生的 ice 对象，其实就是 P2PTransportChannel 对象。创建ice层的传输对象，负责管理Candidates，联通性检测，收发数据包。

2. 创建dtls层的传输对象，提供Dtls握手逻辑，密钥交换。注意其dtls内置了ice层的传输对象，其层次与DatagramTransport是平行关系。

3. 根据是否使用加密以及加密手段，来创建RTP层不同的传输对象

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

<img src="set-remote-description.assets/transport.jpg" />

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



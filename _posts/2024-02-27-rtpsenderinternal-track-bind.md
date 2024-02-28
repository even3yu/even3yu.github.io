---
layout: post
title: RtpSenderInternal 和Track 绑定
date: 2024-02-27 04:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq
---


* content
{:toc}

---

## RtpSenderInternal 和Track关联

AddTrack的时候， 如果没有空余的Transceiver，则创建新的Transceiver，然后创建RtpSenderInternal，然后通过`RtpSenderInternal::SetTrack` 绑定数据，解绑也是通过这个接口；
如果有空余的Transceiver，则通过`RtpSenderInternal::SetTrack` 绑定数据，替换旧的track

### 1. RtpTransmissionManager::AddTrackUnifiedPlan

pc/rtp_transmission_manager.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackUnifiedPlan(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  auto transceiver = FindFirstTransceiverForAddedTrack(track);
   //////////////////////
  if (transceiver) {
    RTC_LOG(LS_INFO) << "Reusing an existing "
                     << cricket::MediaTypeToString(transceiver->media_type())
                     << " transceiver for AddTrack.";
    if (transceiver->stopping()) {
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "The existing transceiver is stopping.");
    }

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
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
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
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
    auto sender = CreateSender(media_type, sender_id, track, stream_ids, {});
    auto receiver = CreateReceiver(media_type, rtc::CreateRandomUuid());
    transceiver = CreateAndAddTransceiver(sender, receiver);
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
    transceiver->internal()->set_created_by_addtrack(true);
    transceiver->internal()->set_direction(RtpTransceiverDirection::kSendRecv);
  }
  return transceiver->sender();
}
```



### 2. RtpTransmissionManager::CreateSender

pc/rtp_transmission_manager.cc

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
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  bool set_track_succeeded = sender->SetTrack(track);
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!
  RTC_DCHECK(set_track_succeeded);
  sender->internal()->set_stream_ids(stream_ids);
  sender->internal()->set_init_send_encodings(send_encodings);
  return sender;
}
```


---
layout: post
title: webrtc setRemoteDescription-2
date: 2023-09-02 23:32:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

![]({{ site.url }}{{ site.baseurl }}/images/set-local-description.assets/view.png)



## 8. SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels(
    cricket::ContentSource source, // 
    const SessionDescriptionInterface& new_session, // new sdp
    const SessionDescriptionInterface* old_local_description, // old sdp
    const SessionDescriptionInterface* old_remote_description) { // remote sdp 
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(IsUnifiedPlan());

  const cricket::ContentGroup* bundle_group = nullptr;
  // 这里type 就是 answer
  if (new_session.GetType() == SdpType::kOffer) {
    ...
  }
  
  // ContentInfo 是什么？？？就是 a=mid:audio 相关的mline信息保存，ContentInfo 对应一个mline
  const ContentInfos& new_contents = new_session.description()->contents();
  for (size_t i = 0; i < new_contents.size(); ++i) {
    const cricket::ContentInfo& new_content = new_contents[i];
    cricket::MediaType media_type = new_content.media_description()->type();
    mid_generator_.AddKnownId(new_content.name);
    if (media_type == cricket::MEDIA_TYPE_AUDIO ||
        media_type == cricket::MEDIA_TYPE_VIDEO) {
      const cricket::ContentInfo* old_local_content = nullptr;
      if (old_local_description &&
          i < old_local_description->description()->contents().size()) {
        old_local_content =
            &old_local_description->description()->contents()[i];
      }
      const cricket::ContentInfo* old_remote_content = nullptr;
      if (old_remote_description &&
          i < old_remote_description->description()->contents().size()) {
        old_remote_content =
            &old_remote_description->description()->contents()[i];
      }
      // 找到transceiver，设置set_mid，set_mline_index
      auto transceiver_or_error =
          AssociateTransceiver(source, new_session.GetType(), i, new_content,
                               old_local_content, old_remote_content);
      if (!transceiver_or_error.ok()) {
        // In the case where a transceiver is rejected locally, we don't
        // expect to find a transceiver, but might find it in the case
        // where state is still "stopping", not "stopped".
        if (new_content.rejected) {
          continue;
        }
        return transceiver_or_error.MoveError();
      }
      auto transceiver = transceiver_or_error.MoveValue();
      // 设置Channel， 如果原先的没有，创建VoiceChannel 或者 VideoChannel,就是BaseChannel
      RTCError error =
          UpdateTransceiverChannel(transceiver, new_content, bundle_group);
      if (!error.ok()) {
        return error;
      }
    } else if (media_type == cricket::MEDIA_TYPE_DATA) {
      ...
    } else if (media_type == cricket::MEDIA_TYPE_UNSUPPORTED) {
      RTC_LOG(LS_INFO) << "Ignoring unsupported media type";
    } else {
      LOG_AND_RETURN_ERROR(RTCErrorType::INTERNAL_ERROR,
                           "Unknown section type.");
    }
  }

  return RTCError::OK();
}
```

1. 参数说明

   | 参数                                                      | 说明                                       |
   | --------------------------------------------------------- | ------------------------------------------ |
   | cricket::ContentSource source                             | 本地，还是远端， 这里是cricket::CS_REMOTE, |
   | const SessionDescriptionInterface& new_session            | createOffer的 回调的session                |
   | const SessionDescriptionInterface* old_local_description  | 上次createOffer的local_description         |
   | const SessionDescriptionInterface* old_remote_description | setRemoteDescription                       |

2. `ContentInfos& new_contents = new_session.description()->contents();`, 对所有的ContentInfo进行遍历；
   通过`AssociateTransceiver`找到对应的Transceiver，设置Channel， 如果原先的没有，创建VoiceChannel 或者 VideoChannel；

3. UpdateTransceiverChannel()方法中检查PC中的每个RtpTranceiver是存在MediaChannel，不存在的会调用WebRtcVideoEngine::CreateMediaChannel()创建WebRtcVideoChannel对象，并赋值给RtpTranceiver的RtpSender和RtpReceiver，这儿解决了VideoRtpSender的media_channel_成员为空的问题；

3. AssociateTransceiver， transceiver  是什么添加到列表的？？？RtpTransmissionManager::CreateAndAddTransceiver 创建和添加。其中就是AddTrack的时候添加。



### 8.1 ??? SdpOfferAnswerHandler::AssociateTransceiver

pc/sdp_offer_answer.cc

```c++
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
SdpOfferAnswerHandler::AssociateTransceiver(
    cricket::ContentSource source,
    SdpType type,
    size_t mline_index,
    const ContentInfo& content,
    const ContentInfo* old_local_content,
    const ContentInfo* old_remote_content) {
  ...

  const MediaContentDescription* media_desc = content.media_description();
  // 根据 mid 找到对应的Transceiver
  auto transceiver = transceivers()->FindByMid(content.name);
  if (source == cricket::CS_LOCAL) {
    ...
  }  else {
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
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kRecvOnly);
      if (type == SdpType::kOffer) {
        transceivers()->StableState(transceiver)->set_newly_created();
      }
    }

    ...
  }

  ...
    // Associate the found or created RtpTransceiver with the m= section by
  // setting the value of the RtpTransceiver's mid property to the MID of the m=
  // section, and establish a mapping between the transceiver and the index of
  // the m= section.
  // 只有在这里会绑定 mid；
  
  // transceiver 是包了线程安全 RtpTransceiverProxy；
  // internal 就是 RtpTransceiver
  transceiver->internal()->set_mid(content.name);
  transceiver->internal()->set_mline_index(mline_index);
  return std::move(transceiver);
}
```

- 根据mid找transceiver。
- transceiver 设置 mid和 mline_index。
- transceiver 绑定mid；
- transceiver 的 mline_index 在createoffer 会进行赋值。比如是上一个sdpOffer的。



#### !!! RtpTransceiver::set_mid

#### RtpTransceiver::set_mline_index



### 8.2 !!! SdpOfferAnswerHandler::UpdateTransceiverChannel

pc/sdp_offer_answer.cc

```c++
RTCError SdpOfferAnswerHandler::UpdateTransceiverChannel(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const cricket::ContentInfo& content,
    const cricket::ContentGroup* bundle_group) {

  // transceiver 是包了线程安全 RtpTransceiverProxy；
  // internal 就是 RtpTransceiver
  cricket::ChannelInterface* channel = transceiver->internal()->channel();
  if (content.rejected) {
    if (channel) {
      // 与RtpTrasceiver解除绑定
      transceiver->internal()->SetChannel(nullptr);
      // 销毁channel
      DestroyChannelInterface(channel);
    }
  } else {
    // 这里channel不为空，因为在SetLocalDescription的时候，已经创建，存放在transceiver
    if (!channel) {
      ...
    }
  }
  return RTCError::OK();
}
```

1. RtpTransceiver::channel, 返回的是BaseChannel；
   pc/rtp_transceiver.h

   ```cpp
     cricket::ChannelInterface* channel() const { return channel_; }
   ```

2. 如果ContentInfo.rejected = true， VideoChannel 和 RtpTransceiver绑定。DestroyChannelInterface

3. 这里channel不为空，因为在SetLocalDescription的时候，已经创建，存放在transceiver



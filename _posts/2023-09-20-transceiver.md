---
layout: post
title: webrtc transceiver
date: 2023-09-20 11:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


![sdp-transceiver]({{ site.url }}{{ site.baseurl }}/images/transceiver.assets/sdp-transceiver.jpg)

# 1. RtpTransceiver

```js
pc/rtp_transceiver.cc
pc/rtp_transceiver.h
api/rtp_transceiver_interface.h
```

- RtpTransceiver 是  RtpSender和RtpReceiver 的组合， 共享一个mid。

- 在Jsep中，如果RtpTransceiver的mid不为空，则会对应到MediaDescription。

- 只支持Unified Plan。

- 由于需要向后兼容Plan B SDP，PeerConnection内部使用RtpTransceiver进行跟踪
  RtpSenders、RtpReceivers、BaseChannels 和 PeerConnection。

- 使用 Plan B SDP，RtpTransceiver 可以有任意数量的发送方，并且映射到 m= 部分中的 a=ssrc 行的接收者。
  使用 Unified Plan SDP，RtpTransceiver 将只有一个发送者和一个由 m= 部分封装的接收者。

- RtpTransceiver 的媒体类型，是在构造函数的时候传入的MediaType 决定 是 Audio RtpTransceivers 还是  Video RtpTransceivers。

  Audio RtpTransceivers 有AudioRtpSenders, AudioRtpReceivers, 和 VoiceChannel；
  Video RtpTransceivers 有VideoRtpSenders, VideoRtpReceivers, 和 VideoChannel；

  

## 1.1 类图

```
RtpTransceiverInterface
RtpTransceiver
```



## 1.2 属性

```cpp
  const bool unified_plan_;
  
  const cricket::MediaType media_type_;
  std::vector<rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>>
      senders_;
  std::vector<
      rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>>
      receivers_;

  bool stopped_ = false;
  bool stopping_ RTC_GUARDED_BY(thread_) = false;

  bool is_pc_closed_ = false;
  
  RtpTransceiverDirection direction_ = RtpTransceiverDirection::kInactive;
  absl::optional<RtpTransceiverDirection> current_direction_;
  absl::optional<RtpTransceiverDirection> fired_direction_;
  
  absl::optional<std::string> mid_;
  absl::optional<size_t> mline_index_;
  
  bool created_by_addtrack_ = false;
  bool reused_for_addtrack_ = false;
  bool has_ever_been_used_to_send_ = false;

  cricket::ChannelInterface* channel_ = nullptr;
  cricket::ChannelManager* channel_manager_ = nullptr;
  
  std::vector<RtpCodecCapability> codec_preferences_;
  std::vector<RtpHeaderExtensionCapability> header_extensions_to_offer_;
  
  const std::function<void()> on_negotiation_needed_;
```



## 1.3 RtpTransceiver 构造函数

### ~~RtpTransceiver::RtpTransceiver~~

```cpp
  // Construct a Plan B-style RtpTransceiver with no senders, receivers, or
  // channel set.
  // |media_type| specifies the type of RtpTransceiver (and, by transitivity,
  // the type of senders, receivers, and channel). Can either by audio or video.
RtpTransceiver::RtpTransceiver(cricket::MediaType media_type)
    : thread_(GetCurrentTaskQueueOrThread()),
      unified_plan_(false),
      media_type_(media_type) {
  RTC_DCHECK(media_type == cricket::MEDIA_TYPE_AUDIO ||
             media_type == cricket::MEDIA_TYPE_VIDEO);
}
```

- unified_plan_ = false



### RtpTransceiver::RtpTransceiver

```cpp
  // Construct a Unified Plan-style RtpTransceiver with the given sender and
  // receiver. The media type will be derived from the media types of the sender
  // and receiver. The sender and receiver should have the same media type.
  // |HeaderExtensionsToOffer| is used for initializing the return value of
  // HeaderExtensionsToOffer().

RtpTransceiver::RtpTransceiver(
    rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>> sender,
    rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>
        receiver,
    cricket::ChannelManager* channel_manager,
    std::vector<RtpHeaderExtensionCapability> header_extensions_offered,
    std::function<void()> on_negotiation_needed)
    : thread_(GetCurrentTaskQueueOrThread()),
      unified_plan_(true),
      media_type_(sender->media_type()),
      channel_manager_(channel_manager),
      header_extensions_to_offer_(std::move(header_extensions_offered)),
      on_negotiation_needed_(std::move(on_negotiation_needed)) {
  RTC_DCHECK(media_type_ == cricket::MEDIA_TYPE_AUDIO ||
             media_type_ == cricket::MEDIA_TYPE_VIDEO);
  RTC_DCHECK_EQ(sender->media_type(), receiver->media_type());
  senders_.push_back(sender);
  receivers_.push_back(receiver);
}
```

- `media_type_(sender->media_type())` 

- unified_plan_ = true

- 由peerConnection 中`PeerConnection::AddTransceiver`创建RtpTransceiver的时候，同时创建了rtpsender和rtpreceiver



## 1.4 !!! RtpTransceiver::SetChannel

```cpp
  // Returns the Voice/VideoChannel set for this transceiver. May be null if
  // the transceiver is not in the currently set local/remote description.
  cricket::ChannelInterface* channel() const { return channel_; }
```



```cpp
void RtpTransceiver::SetChannel(cricket::ChannelInterface* channel) {
	...
  channel_ = channel;

  if (channel_) {
    channel_->SignalFirstPacketReceived().connect(
        this, &RtpTransceiver::OnFirstPacketReceived);
  }

  //////////////////////////////////////
  // 在unified plan 中  senders_ 和 receivers_ 分别只有一个
  //////////////////////////////////////
  for (const auto& sender : senders_) {
    sender->internal()->SetMediaChannel(channel_ ? channel_->media_channel()
                                                 : nullptr);
  }

  for (const auto& receiver : receivers_) {
    if (!channel_) {
      receiver->internal()->Stop();
    }

    receiver->internal()->SetMediaChannel(channel_ ? channel_->media_channel()
                                                   : nullptr);
  }
  //////////////////////////////////////
}
```

- 这里存放channel_ 是BaseChannel；

- 为所有的 RtpSender 和 RtpReceiver  绑定MediaChannel, 不过unified plan 分别只有一个；



### 调用时机

### !!! SdpOfferAnswerHandler::UpdateTransceiverChannel

```cpp
SetLocalDescription / SetRemoteDescription
SdpOfferAnswerHandler::ApplyLocalDescription / SdpOfferAnswerHandler::ApplyRemoteDescription
SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels
SdpOfferAnswerHandler::UpdateTransceiverChannel
```



```cpp
RTCError SdpOfferAnswerHandler::UpdateTransceiverChannel(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const cricket::ContentInfo& content,
    const cricket::ContentGroup* bundle_group) {
  ...
  cricket::ChannelInterface* channel = transceiver->internal()->channel();
  if (content.rejected) {
    if (channel) {
      transceiver->internal()->SetChannel(nullptr);
      DestroyChannelInterface(channel);
    }
  } else {
    if (!channel) {
      if (transceiver->media_type() == cricket::MEDIA_TYPE_AUDIO) {
        channel = CreateVoiceChannel(content.name);
      } else {
        channel = CreateVideoChannel(content.name);
      }
      ...
      transceiver->internal()->SetChannel(channel);
    }
  }
}
```



### ~~SdpOfferAnswerHandler::CreateChannels~~

Plan B 中调用。



### SdpOfferAnswerHandler::DestroyTransceiverChannel

- ~~SdpOfferAnswerHandler::RemoveUnusedChannels~~ //-- 弃用

- PeerConnection::~PeerConnection()/ PeerConnection::Close()
  DestroyAllChannels

```cpp
void SdpOfferAnswerHandler::DestroyTransceiverChannel(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver) {
	...
  cricket::ChannelInterface* channel = transceiver->internal()->channel();
  if (channel) {
    transceiver->internal()->SetChannel(nullptr);
    DestroyChannelInterface(channel);
  }
}
```



## 1.5 Sender

### ~~RtpTransceiver::AddSender~~

```cpp
void RtpTransceiver::AddSender(
    rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>> sender) {
  senders_.push_back(sender);
}
```



### ~~RtpTransceiver::RemoveSender~~

```cpp
bool RtpTransceiver::RemoveSender(RtpSenderInterface* sender) {
  RTC_DCHECK(!unified_plan_);
  if (sender) {
    RTC_DCHECK_EQ(media_type(), sender->media_type());
  }
  auto it = absl::c_find(senders_, sender);
  if (it == senders_.end()) {
    return false;
  }
  (*it)->internal()->Stop();
  senders_.erase(it);
  return true;
}
```



### RtpTransceiver::senders

```cpp
// Returns a vector of the senders owned by this transceiver.
std::vector<rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>>
  senders() const {
  return senders_;
}
```



### RtpTransceiver::sender_internal

```cpp
// Returns the backing object for the transceiver's Unified Plan sender.
rtc::scoped_refptr<RtpSenderInternal> RtpTransceiver::sender_internal() const {
  return senders_[0]->internal();
}
```



## 1.6 Receiver

### ~~RtpTransceiver::AddReceiver~~

```cpp
void RtpTransceiver::AddReceiver(
    rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>
        receiver) {
  receivers_.push_back(receiver);
}
```



### ~~RtpTransceiver::RemoveReceiver~~

```cpp
bool RtpTransceiver::RemoveReceiver(RtpReceiverInterface* receiver) {
	...
  auto it = absl::c_find(receivers_, receiver);
  if (it == receivers_.end()) {
    return false;
  }
  (*it)->internal()->Stop();
  // After the receiver has been removed, there's no guarantee that the
  // contained media channel isn't deleted shortly after this. To make sure that
  // the receiver doesn't spontaneously try to use it's (potentially stale)
  // media channel reference, we clear it out.
  (*it)->internal()->SetMediaChannel(nullptr);
  receivers_.erase(it);
  return true;
}
```



### RtpTransceiver::receivers

```cpp
// Returns a vector of the receivers owned by this transceiver.
std::vector<
  rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>>
  receivers() const {
  return receivers_;
}
```



### RtpTransceiver::receiver_internal

```cpp
// Returns the backing object for the transceiver's Unified Plan receiver.
rtc::scoped_refptr<RtpReceiverInternal> RtpTransceiver::receiver_internal()
    const {
  return receivers_[0]->internal();
}
```





## 1.7 !!! set_mline_index

```cpp
// RtpTransceivers are not associated until they have a corresponding media
// section set in SetLocalDescription or SetRemoteDescription. Therefore,
// when setting a local offer we need a way to remember which transceiver was
// used to create which media section in the offer. Storing the m-line index
// in CreateOffer is specified in JSEP to allow us to do that.
absl::optional<size_t> mline_index() const { return mline_index_; }
void set_mline_index(absl::optional<size_t> mline_index) {
  mline_index_ = mline_index;
}

```



```cpp
absl::optional<size_t> mline_index() const { return mline_index_; }
```

在`SetLocalDescription` 或者 `SetRemoteDescription` , RtpTransceiver 才会和media section 关联起来；但是在设置local offer的时候，需要记住RtpTransceiver和media section 关联， 所以mline_index_的作用就是这个。



### 调用时机

### SdpOfferAnswerHandler::AssociateTransceiver

```cpp
SdpOfferAnswerHandler::ApplyLocalDescription/ SdpOfferAnswerHandler::ApplyRemoteDescription
SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels
SdpOfferAnswerHandler::AssociateTransceiver
```



```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
SdpOfferAnswerHandler::AssociateTransceiver(
    cricket::ContentSource source,
    SdpType type,
    size_t mline_index,
    const ContentInfo& content,
    const ContentInfo* old_local_content,
    const ContentInfo* old_remote_content) {
...

  // Associate the found or created RtpTransceiver with the m= section by
  // setting the value of the RtpTransceiver's mid property to the MID of the m=
  // section, and establish a mapping between the transceiver and the index of
  // the m= section.
  //！！！！！！！！！！！！！！
  transceiver->internal()->set_mid(content.name);
  transceiver->internal()->set_mline_index(mline_index);
  //！！！！！！！！！！！！！！
  return std::move(transceiver);
}
```

这个mline_index 是从SessionDescriptionInterface 中的ContentInfos，就数组的下标。

### SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer

```cpp
SdpOfferAnswerHandler::CreateOffer
SdpOfferAnswerHandler::DoCreateOffer
SdpOfferAnswerHandler::GetOptionsForOffer
```



```cpp
void SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer(
    const RTCOfferAnswerOptions& offer_answer_options,
    cricket::MediaSessionOptions* session_options) {
  // Rules for generating an offer are dictated by JSEP sections 5.2.1 (Initial
  // Offers) and 5.2.2 (Subsequent Offers).
  RTC_DCHECK_EQ(session_options->media_description_options.size(), 0);
  const ContentInfos no_infos;
  // local_contents, remote_contents 为空
  const ContentInfos& local_contents =
      (local_description() ? local_description()->description()->contents()
                           : no_infos);
  const ContentInfos& remote_contents =
      (remote_description() ? remote_description()->description()->contents()
                            : no_infos);
  // The m-line indices that can be recycled. New transceivers should reuse these
  // slots first.
  std::queue<size_t> recycleable_mline_indices;

  // 这里不会执行
  for (size_t i = 0;
       i < std::max(local_contents.size(), remote_contents.size()); ++i) {
  	...
  }

	// 现有的transceivers()， 就是addTack 或者addTransceiver 中添加的Transceiver
  for (const auto& transceiver : transceivers()->List()) {
    if (transceiver->mid() || transceiver->stopping()) {
      continue;
    }
    size_t mline_index;
    if (!recycleable_mline_indices.empty()) {
      mline_index = recycleable_mline_indices.front();
      recycleable_mline_indices.pop();
      session_options->media_description_options[mline_index] =
          GetMediaDescriptionOptionsForTransceiver(
              transceiver, mid_generator_.GenerateString(),
              /*is_create_offer=*/true);
    } else {
      mline_index = session_options->media_description_options.size();
      session_options->media_description_options.push_back(
          GetMediaDescriptionOptionsForTransceiver(
              transceiver, mid_generator_.GenerateString(),
              /*is_create_offer=*/true));
    }
    //！！！！！！！！！！！！！！
    // See comment above for why CreateOffer changes the transceiver's state.
    transceiver->internal()->set_mline_index(mline_index);
    //！！！！！！！！！！！！！！
  }

  ...
}
```

- 如果transceiver 没有mid，也没有stopping，绑定mline_index；mline_index一直叠加，如果mline_index回收了，则可以复用；

- mline_index 是可以回收的，

   （1）从上一次的localDescription中，`auto transceiver = transceivers()->FindByMid(mid);`  没找到，则回收

   （2）`had_been_rejected && transceiver->stopping()`, 可以回收



### SdpOfferAnswerHandler::RemoveStoppedTransceivers()

```cpp
void SdpOfferAnswerHandler::RemoveStoppedTransceivers() {
  if (!IsUnifiedPlan())
    return;
  // Traverse a copy of the transceiver list.
  auto transceiver_list = transceivers()->List();
  for (auto transceiver : transceiver_list) {
    // 3.2.10.1.1: If transceiver is stopped, associated with an m= section
    //             and the associated m= section is rejected in
    //             connection.[[CurrentLocalDescription]] or
    //             connection.[[CurrentRemoteDescription]], remove the
    //             transceiver from the connection's set of transceivers.
    if (!transceiver->stopped()) {
      continue;
    }
    const ContentInfo* local_content =
        FindMediaSectionForTransceiver(transceiver, local_description());
    const ContentInfo* remote_content =
        FindMediaSectionForTransceiver(transceiver, remote_description());
    if ((local_content && local_content->rejected) ||
        (remote_content && remote_content->rejected)) {
      RTC_LOG(LS_INFO) << "Dissociating transceiver"
                       << " since the media section is being recycled.";
      //!!!!!!!!!!!!!!!!!!!!!!!!!
      transceiver->internal()->set_mid(absl::nullopt);
      transceiver->internal()->set_mline_index(absl::nullopt);
      //!!!!!!!!!!!!!!!!!!!!!!!!!
      transceivers()->Remove(transceiver);
      continue;
    }
    if (!local_content && !remote_content) {
      // TODO(bugs.webrtc.org/11973): Consider if this should be removed already
      // See https://github.com/w3c/webrtc-pc/issues/2576
      RTC_LOG(LS_INFO)
          << "Dropping stopped transceiver that was never associated";
      transceivers()->Remove(transceiver);
      continue;
    }
  }
}
```



## 1.8 !!! set_mid

```cpp
  // Sets the MID for this transceiver. If the MID is not null, then the
  // transceiver is considered "associated" with the media section that has the
  // same MID.
  void set_mid(const absl::optional<std::string>& mid) { mid_ = mid; }
```



```cpp
absl::optional<std::string> RtpTransceiver::mid() const {
  return mid_;
}
```

如果transceiver的 mid 不为空，标识transceiver和media section 关联起来了。



### 调用时机

### SdpOfferAnswerHandler::AssociateTransceiver

```cpp
SdpOfferAnswerHandler::ApplyLocalDescription/ SdpOfferAnswerHandler::ApplyRemoteDescription
SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels
SdpOfferAnswerHandler::AssociateTransceiver
```



```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
SdpOfferAnswerHandler::AssociateTransceiver(
    cricket::ContentSource source,
    SdpType type,
    size_t mline_index,
    const ContentInfo& content,
    const ContentInfo* old_local_content,
    const ContentInfo* old_remote_content) {
...

  // Associate the found or created RtpTransceiver with the m= section by
  // setting the value of the RtpTransceiver's mid property to the MID of the m=
  // section, and establish a mapping between the transceiver and the index of
  // the m= section.
  //!!!!!!!!!!!!!!!!!!!!!!!!!
  transceiver->internal()->set_mid(content.name);
  transceiver->internal()->set_mline_index(mline_index);
 	//!!!!!!!!!!!!!!!!!!!!!!!!!
  return std::move(transceiver);
}
```

绑定mid。
mid 就是ContentInfo.name。 

### SdpOfferAnswerHandler::RemoveStoppedTransceivers

```cpp
void SdpOfferAnswerHandler::RemoveStoppedTransceivers() {

  if (!IsUnifiedPlan())
    return;
  // Traverse a copy of the transceiver list.
  auto transceiver_list = transceivers()->List();
  for (auto transceiver : transceiver_list) {
    // 3.2.10.1.1: If transceiver is stopped, associated with an m= section
    //             and the associated m= section is rejected in
    //             connection.[[CurrentLocalDescription]] or
    //             connection.[[CurrentRemoteDescription]], remove the
    //             transceiver from the connection's set of transceivers.
    if (!transceiver->stopped()) {
      continue;
    }
    const ContentInfo* local_content =
        FindMediaSectionForTransceiver(transceiver, local_description());
    const ContentInfo* remote_content =
        FindMediaSectionForTransceiver(transceiver, remote_description());
    if ((local_content && local_content->rejected) ||
        (remote_content && remote_content->rejected)) {
      RTC_LOG(LS_INFO) << "Dissociating transceiver"
                       << " since the media section is being recycled.";
      
      //!!!!!!!!!!!!!!!!!!!!!!!!!
      transceiver->internal()->set_mid(absl::nullopt);
      transceiver->internal()->set_mline_index(absl::nullopt);  
      //!!!!!!!!!!!!!!!!!!!!!!!!!
      transceivers()->Remove(transceiver);
      continue;
    }
    if (!local_content && !remote_content) {
      // TODO(bugs.webrtc.org/11973): Consider if this should be removed already
      // See https://github.com/w3c/webrtc-pc/issues/2576
      RTC_LOG(LS_INFO)
          << "Dropping stopped transceiver that was never associated";
      transceivers()->Remove(transceiver);
      continue;
    }
  }
}
```

解绑mid



## 1.9 direction

### set_direction

```cpp
  // Sets the intended direction for this transceiver. Intended to be used
  // internally over SetDirection since this does not trigger a negotiation
  // needed callback.
  void set_direction(RtpTransceiverDirection direction) {
    direction_ = direction;
  } 
```

设置预期的媒体方向；需要要协商。

```cpp
RtpTransceiverDirection RtpTransceiver::direction() const {
  if (unified_plan_ && stopping())
    return webrtc::RtpTransceiverDirection::kStopped;

  return direction_;
}
```

### ??? set_current_direction



### ??? set_fired_direction



### SetDirectionWithError

```cpp
RTCError RtpTransceiver::SetDirectionWithError(
    RtpTransceiverDirection new_direction) {
  if (unified_plan_ && stopping()) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_STATE,
                         "Cannot set direction on a stopping transceiver.");
  }
  if (new_direction == direction_)
    return RTCError::OK();

  if (new_direction == RtpTransceiverDirection::kStopped) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                         "The set direction 'stopped' is invalid.");
  }

  direction_ = new_direction;
  on_negotiation_needed_();

  return RTCError::OK();
}
```

需要协商，



## 1.10 ???stop

### RtpTransceiver::stopped/stopping

```cpp
bool RtpTransceiver::stopped() const {
  return stopped_;
}

bool RtpTransceiver::stopping() const {
  RTC_DCHECK_RUN_ON(thread_);
  return stopping_;
}
```

- createOffer的时候，stopping就需要把它认为是stopped



#### RtpTransceiver::StopStandard

```cpp
RtpTransceiver::StopStandard
RtpTransceiver::StopSendingAndReceiving
```

```cpp
RTCError RtpTransceiver::StopStandard() {
  RTC_DCHECK_RUN_ON(thread_);
  // If we're on Plan B, do what Stop() used to do there.
  if (!unified_plan_) {
    StopInternal();
    return RTCError::OK();
  }
  // 1. Let transceiver be the RTCRtpTransceiver object on which the method is
  // invoked.
  //
  // 2. Let connection be the RTCPeerConnection object associated with
  // transceiver.
  //
  // 3. If connection.[[IsClosed]] is true, throw an InvalidStateError.
  if (is_pc_closed_) {
    LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_STATE,
                         "PeerConnection is closed.");
  }

  // 4. If transceiver.[[Stopping]] is true, abort these steps.
  if (stopping_)
    return RTCError::OK();

  // 5. Stop sending and receiving given transceiver, and update the
  // negotiation-needed flag for connection.
  StopSendingAndReceiving();
  on_negotiation_needed_();

  return RTCError::OK();
}
```

- 该函数stop，需要等待stopped 返回true，才真正的结束。`RtpTransceiver::StopInternal` 这里才设置stopped = true。
- 对外接口，内部没地方调用



##### !!! RtpTransceiver::StopSendingAndReceiving

```cpp
void RtpTransceiver::StopSendingAndReceiving() {
  // 1. Let sender be transceiver.[[Sender]].
  // 2. Let receiver be transceiver.[[Receiver]].
  //
  // 3. Stop sending media with sender.
  //
  // 4. Send an RTCP BYE for each RTP stream that was being sent by sender, as
  // specified in [RFC3550].
  RTC_DCHECK_RUN_ON(thread_);
  for (const auto& sender : senders_)
    sender->internal()->Stop();

  // 5. Stop receiving media with receiver.
  for (const auto& receiver : receivers_)
    receiver->internal()->StopAndEndTrack();

  //!!!!!!!!!!!!!!!!!!
  stopping_ = true;
  //!!!!!!!!!!!!!!!!!!
  direction_ = webrtc::RtpTransceiverDirection::kInactive;
}
```



#### !!! RtpTransceiver::StopInternal

```cpp
PeerConnection::~PeerConnection()
//-----------
PeerConnection::Close()
//----------
RtpTransceiver::~RtpTransceiver()
//----------
RtpTransceiver::StopStandard() // plan B
//-----------  
RtpTransceiverInterface::Stop(), //弃用
```



```cpp
void RtpTransceiver::StopInternal() {
  StopTransceiverProcedure();
}

```

- 立即停止，不需要等待，这个是和StopStandard 区别。

##### !!! RtpTransceiver::StopTransceiverProcedure

```cpp
SdpOfferAnswerHandler::ApplyRemoteDescription
RtpTransceiver::StopTransceiverProcedure
```



```cpp
void RtpTransceiver::StopTransceiverProcedure() {
  RTC_DCHECK_RUN_ON(thread_);
  // As specified in the "Stop the RTCRtpTransceiver" procedure
  // 1. If transceiver.[[Stopping]] is false, stop sending and receiving given
  // transceiver.
  if (!stopping_)
    StopSendingAndReceiving();

  //!!!!!!!!!!!!
  // 2. Set transceiver.[[Stopped]] to true.
  stopped_ = true;
  //!!!!!!!!!!!!

  // Signal the updated change to the senders.
  for (const auto& sender : senders_)
    sender->internal()->SetTransceiverAsStopped();

  // 3. Set transceiver.[[Receptive]] to false.
  // 4. Set transceiver.[[CurrentDirection]] to null.
  current_direction_ = absl::nullopt;
}

```

立即停止。

#### ~~RtpTransceiver::Stop~~

api/rtp_transceiver_interface.cc

```cpp
void RtpTransceiverInterface::Stop() {
  StopInternal();
}

```



 

## 1.11 CodecPreferences

### RtpTransceiver::SetCodecPreferences

```cpp
RTCError RtpTransceiver::SetCodecPreferences(
    rtc::ArrayView<RtpCodecCapability> codec_capabilities) {
  RTC_DCHECK(unified_plan_);

  // 3. If codecs is an empty list, set transceiver's [[PreferredCodecs]] slot
  // to codecs and abort these steps.
  if (codec_capabilities.empty()) {
    codec_preferences_.clear();
    return RTCError::OK();
  }

  // 4. Remove any duplicate values in codecs.
  std::vector<RtpCodecCapability> codecs;
  absl::c_remove_copy_if(codec_capabilities, std::back_inserter(codecs),
                         [&codecs](const RtpCodecCapability& codec) {
                           return absl::c_linear_search(codecs, codec);
                         });

  // 6. to 8.
  RTCError result;
  if (media_type_ == cricket::MEDIA_TYPE_AUDIO) {
    std::vector<cricket::AudioCodec> recv_codecs, send_codecs;
    channel_manager_->GetSupportedAudioReceiveCodecs(&recv_codecs);
    channel_manager_->GetSupportedAudioSendCodecs(&send_codecs);

    result = VerifyCodecPreferences(codecs, send_codecs, recv_codecs);
  } else if (media_type_ == cricket::MEDIA_TYPE_VIDEO) {
    std::vector<cricket::VideoCodec> recv_codecs, send_codecs;
    channel_manager_->GetSupportedVideoReceiveCodecs(&recv_codecs);
    channel_manager_->GetSupportedVideoSendCodecs(&send_codecs);

    result = VerifyCodecPreferences(codecs, send_codecs, recv_codecs);
  }

  if (result.ok()) {
    codec_preferences_ = codecs;
  }

  return result;
}
```





```cpp
  std::vector<RtpCodecCapability> codec_preferences() const override {
    return codec_preferences_;
  }
```





## 1.12 HeaderExtension

```cpp
std::vector<RtpHeaderExtensionCapability>
RtpTransceiver::HeaderExtensionsToOffer() const {
  return header_extensions_to_offer_;
}

std::vector<RtpHeaderExtensionCapability>
RtpTransceiver::HeaderExtensionsNegotiated() const {
  if (!channel_)
    return {};
  std::vector<RtpHeaderExtensionCapability> result;
  for (const auto& ext : channel_->GetNegotiatedRtpHeaderExtensions()) {
    result.emplace_back(ext.uri, ext.id, RtpTransceiverDirection::kSendRecv);
  }
  return result;
}

RTCError RtpTransceiver::SetOfferedRtpHeaderExtensions(
    rtc::ArrayView<const RtpHeaderExtensionCapability>
        header_extensions_to_offer) {
  for (const auto& entry : header_extensions_to_offer) {
    // Handle unsupported requests for mandatory extensions as per
    // https://w3c.github.io/webrtc-extensions/#rtcrtptransceiver-interface.
    // Note:
    // - We do not handle setOfferedRtpHeaderExtensions algorithm step 2.1,
    //   this has to be checked on a higher level. We naturally error out
    //   in the handling of Step 2.2 if an unset URI is encountered.

    // Step 2.2.
    // Handle unknown extensions.
    auto it = std::find_if(
        header_extensions_to_offer_.begin(), header_extensions_to_offer_.end(),
        [&entry](const auto& offered) { return entry.uri == offered.uri; });
    if (it == header_extensions_to_offer_.end()) {
      return RTCError(RTCErrorType::UNSUPPORTED_PARAMETER,
                      "Attempted to modify an unoffered extension.");
    }

    // Step 2.4-2.5.
    // - Use of the transceiver interface indicates unified plan is in effect,
    //   hence the MID extension needs to be enabled.
    // - Also handle the mandatory video orientation extensions.
    if ((entry.uri == RtpExtension::kMidUri ||
         entry.uri == RtpExtension::kVideoRotationUri) &&
        entry.direction != RtpTransceiverDirection::kSendRecv) {
      return RTCError(RTCErrorType::INVALID_MODIFICATION,
                      "Attempted to stop a mandatory extension.");
    }
  }

  // Apply mutation after error checking.
  for (const auto& entry : header_extensions_to_offer) {
    auto it = std::find_if(
        header_extensions_to_offer_.begin(), header_extensions_to_offer_.end(),
        [&entry](const auto& offered) { return entry.uri == offered.uri; });
    it->direction = entry.direction;
  }

  return RTCError::OK();
}
```





# 2. RtpTransmissionManager

## 2.1 CreateAndAddTransceiver

```cpp
rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
RtpTransmissionManager::CreateAndAddTransceiver(
    rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>> sender,
    rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>
        receiver) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // Ensure that the new sender does not have an ID that is already in use by
  // another sender.
  // Allow receiver IDs to conflict since those come from remote SDP (which
  // could be invalid, but should not cause a crash).
  RTC_DCHECK(!FindSenderById(sender->id()));
  auto transceiver = RtpTransceiverProxyWithInternal<RtpTransceiver>::Create(
      signaling_thread(),
      new RtpTransceiver(
          sender, receiver, channel_manager(),
          sender->media_type() == cricket::MEDIA_TYPE_AUDIO
              ? channel_manager()->GetSupportedAudioRtpHeaderExtensions()
              : channel_manager()->GetSupportedVideoRtpHeaderExtensions(),
          [this_weak_ptr = weak_ptr_factory_.GetWeakPtr()]() {
            if (this_weak_ptr) {
              this_weak_ptr->OnNegotiationNeeded();
            }
          }));
  transceivers()->Add(transceiver);
  return transceiver;
}
```



### Transceiver 创建时机

- PeerConnection::AddTransceiver

- PeerConnection::AddTrack

#### PeerConnection::AddTransceiver

pc/peer_connection.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverInterface>>
PeerConnection::AddTransceiver(
    cricket::MediaType media_type,
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const RtpTransceiverInit& init,
    bool update_negotiation_needed) {
...
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

 ...


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
  transceiver->internal()->set_direction(init.direction);

  if (update_negotiation_needed) {
    sdp_handler_->UpdateNegotiationNeeded();
  }

  return rtc::scoped_refptr<RtpTransceiverInterface>(transceiver);
}
```



#### PeerConnection::Initialize

pc/peer_connection.cc

Plan B

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





## 2.2 CreateSender

```cpp
  // Create a new RTP sender. Does not associate with a transceiver.
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



## 2.3 CreateReceiver

```cpp
rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>
RtpTransmissionManager::CreateReceiver(cricket::MediaType media_type,
                                       const std::string& receiver_id) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  rtc::scoped_refptr<RtpReceiverProxyWithInternal<RtpReceiverInternal>>
      receiver;
  if (media_type == cricket::MEDIA_TYPE_AUDIO) {
    receiver = RtpReceiverProxyWithInternal<RtpReceiverInternal>::Create(
        signaling_thread(), new AudioRtpReceiver(worker_thread(), receiver_id,
                                                 std::vector<std::string>({})));
    NoteUsageEvent(UsageEvent::AUDIO_ADDED);
  } else {
    RTC_DCHECK_EQ(media_type, cricket::MEDIA_TYPE_VIDEO);
    receiver = RtpReceiverProxyWithInternal<RtpReceiverInternal>::Create(
        signaling_thread(), new VideoRtpReceiver(worker_thread(), receiver_id,
                                                 std::vector<std::string>({})));
    NoteUsageEvent(UsageEvent::VIDEO_ADDED);
  }
  return receiver;
}
```



## 2.4 AddTrack

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrack(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK_RUN_ON(signaling_thread());

  return (IsUnifiedPlan() ? AddTrackUnifiedPlan(track, stream_ids)
                          : AddTrackPlanB(track, stream_ids));
}

```





### AddTrackPlanB

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackPlanB(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  if (stream_ids.size() > 1u) {
    LOG_AND_RETURN_ERROR(RTCErrorType::UNSUPPORTED_OPERATION,
                         "AddTrack with more than one stream is not "
                         "supported with Plan B semantics.");
  }
  std::vector<std::string> adjusted_stream_ids = stream_ids;
  if (adjusted_stream_ids.empty()) {
    adjusted_stream_ids.push_back(rtc::CreateRandomUuid());
  }
  cricket::MediaType media_type =
      (track->kind() == MediaStreamTrackInterface::kAudioKind
           ? cricket::MEDIA_TYPE_AUDIO
           : cricket::MEDIA_TYPE_VIDEO);
  //////////////////
  // 创建了CreateSender
  //////////////////
  auto new_sender =
      CreateSender(media_type, track->id(), track, adjusted_stream_ids, {});
  if (track->kind() == MediaStreamTrackInterface::kAudioKind) {
    new_sender->internal()->SetMediaChannel(voice_media_channel());
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



### RtpTransmissionManager::AddTrackUnifiedPlan

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
    // 把track 关联到tpSender
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
    transceiver->internal()->set_direction(RtpTransceiverDirection::kSendRecv);
  }
  return transceiver->sender();
}
```



### ~~AddAudioTrack~~

addStream 弃用

### ~~AddVideoTrack~~







## 2.5 Find

### RtpTransmissionManager::FindSenderForTrack

```cpp
rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>
RtpTransmissionManager::FindSenderForTrack(
    MediaStreamTrackInterface* track) const {
  RTC_DCHECK_RUN_ON(signaling_thread());
  for (const auto& transceiver : transceivers_.List()) {
    for (auto sender : transceiver->internal()->senders()) {
      if (sender->track() == track) {
        return sender;
      }
    }
  }
  return nullptr;
}
```





### RtpTransmissionManager::FindSenderById

```cpp
// sender_id 就是track_id
rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>
RtpTransmissionManager::FindSenderById(const std::string& sender_id) const {
  RTC_DCHECK_RUN_ON(signaling_thread());
  for (const auto& transceiver : transceivers_.List()) {
    for (auto sender : transceiver->internal()->senders()) {
      if (sender->id() == sender_id) {
        return sender;
      }
    }
  }
  return nullptr;
}

```



# 3. TransceiverList



## 3.1 Add/Remove

```cpp
std::vector<RtpTransceiverProxyRefPtr> transceivers_;

std::vector<RtpTransceiverProxyRefPtr> List() const { return transceivers_; }

void Add(RtpTransceiverProxyRefPtr transceiver) {
  transceivers_.push_back(transceiver);
}
void Remove(RtpTransceiverProxyRefPtr transceiver) {
  transceivers_.erase(
    std::remove(transceivers_.begin(), transceivers_.end(), transceiver),
    transceivers_.end());
}
```

- `SdpOfferAnswerHandler::RemoveStoppedTransceivers()` 调用了remove
- `RtpTransmissionManager::CreateAndAddTransceiver` ，`PeerConnection::Initialize`调用工 add

## 3.2 Find

### TransceiverList::FindBySender

```cpp
RtpTransceiverProxyRefPtr TransceiverList::FindBySender(
    rtc::scoped_refptr<RtpSenderInterface> sender) const {
  for (auto transceiver : transceivers_) {
    if (transceiver->sender() == sender) {
      return transceiver;
    }
  }
  return nullptr;
}
```



### TransceiverList::FindByMid

```cpp
RtpTransceiverProxyRefPtr TransceiverList::FindByMid(
    const std::string& mid) const {
  for (auto transceiver : transceivers_) {
    if (transceiver->mid() == mid) {
      return transceiver;
    }
  }
  return nullptr;
}
```



#### 使用时机

#### SdpOfferAnswerHandler::AssociateTransceiver

```cpp
SdpOfferAnswerHandler::ApplyLocalDescription/ SdpOfferAnswerHandler::ApplyRemoteDescription
SdpOfferAnswerHandler::UpdateTransceiversAndDataChannels
SdpOfferAnswerHandler::AssociateTransceiver
```



```cpp
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
  auto transceiver = transceivers()->FindByMid(content.name);
  if (source == cricket::CS_LOCAL) {
    // Find the RtpTransceiver that corresponds to this m= section, using the
    // mapping between transceivers and m= section indices established when
    // creating the offer.
    if (!transceiver) {
      transceiver = transceivers()->FindByMLineIndex(mline_index);
    }
 		...
  }
   ...
}
```



### TransceiverList::FindByMLineIndex

```cpp
RtpTransceiverProxyRefPtr TransceiverList::FindByMLineIndex(
    size_t mline_index) const {
  for (auto transceiver : transceivers_) {
    if (transceiver->internal()->mline_index() == mline_index) {
      return transceiver;
    }
  }
  return nullptr;
}
```


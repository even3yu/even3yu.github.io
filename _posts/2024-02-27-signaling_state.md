---
layout: post
title: SignalingState
date: 2024-02-27 01:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---

## 1. SignalingState定义

api/peer_connection_interface.h

```cpp
class RTC_EXPORT PeerConnectionInterface : public rtc::RefCountInterface {
 public:
  // See https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate
  enum SignalingState {
    kStable,
    kHaveLocalOffer,
    kHaveLocalPrAnswer,
    kHaveRemoteOffer,
    kHaveRemotePrAnswer,
    kClosed,
  };
  ...
}
```



### 1.1 转换为字符串GetSignalingStateString

```cpp
// Map internal signaling state name to spec name:
//  https://w3c.github.io/webrtc-pc/#rtcsignalingstate-enum
std::string GetSignalingStateString(
    PeerConnectionInterface::SignalingState state) {
  switch (state) {
    case PeerConnectionInterface::kStable:
      return "stable";
    case PeerConnectionInterface::kHaveLocalOffer:
      return "have-local-offer";
    case PeerConnectionInterface::kHaveLocalPrAnswer:
      return "have-local-pranswer";
    case PeerConnectionInterface::kHaveRemoteOffer:
      return "have-remote-offer";
    case PeerConnectionInterface::kHaveRemotePrAnswer:
      return "have-remote-pranswer";
    case PeerConnectionInterface::kClosed:
      return "closed";
  }
  RTC_NOTREACHED();
  return "";
}
```



### 1.2 状态说明

| Enum value             | Description                                                  |
| :--------------------- | :----------------------------------------------------------- |
| `stable`               | There is no offer/answer exchange in progress. This is also the initial state, in which case the local and remote descriptions are empty. |
| `have-local-offer`     | A local description, of type "[`offer`](https://w3c.github.io/webrtc-pc/#dom-rtcsdptype-offer)", has been successfully applied. |
| `have-remote-offer`    | A remote description, of type "[`offer`](https://w3c.github.io/webrtc-pc/#dom-rtcsdptype-offer)", has been successfully applied. |
| `have-local-pranswer`  | A remote description of type "[`offer`](https://w3c.github.io/webrtc-pc/#dom-rtcsdptype-offer)" has been successfully applied and a local description of type "[`pranswer`](https://w3c.github.io/webrtc-pc/#dom-rtcsdptype-pranswer)" has been successfully applied. |
| `have-remote-pranswer` | A local description of type "[`offer`](https://w3c.github.io/webrtc-pc/#dom-rtcsdptype-offer)" has been successfully applied and a remote description of type "[`pranswer`](https://w3c.github.io/webrtc-pc/#dom-rtcsdptype-pranswer)" has been successfully applied. |
| `closed`               | The [`RTCPeerConnection`](https://w3c.github.io/webrtc-pc/#dom-rtcpeerconnection) has been closed; its [`[[IsClosed\]]`](https://w3c.github.io/webrtc-pc/#dfn-isclosed) slot is `true`. |



## 2. 状态转换

![signaling state transition diagram](https://w3c.github.io/webrtc-pc/images/peerstates.svg)

An example set of transitions might be:

- Caller transition:
  - new RTCPeerConnection(): "[`stable`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-stable)"
  - setLocalDescription(offer): "[`have-local-offer`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-have-local-offer)"
  - setRemoteDescription(pranswer): "[`have-remote-pranswer`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-have-remote-pranswer)"
  - setRemoteDescription(answer): "[`stable`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-stable)"
- Callee transition:
  - new RTCPeerConnection(): "[`stable`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-stable)"
  - setRemoteDescription(offer): "[`have-remote-offer`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-have-remote-offer)"
  - setLocalDescription(pranswer): "[`have-local-pranswer`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-have-local-pranswer)"
  - setLocalDescription(answer): "[`stable`](https://w3c.github.io/webrtc-pc/#dom-rtcsignalingstate-stable)"



### 2.1 SdpOfferAnswerHandler::UpdateSessionState

```js
SetLocalDescription
SetRemoteDescription
```

在设置`SetLocalDescription`，或者`SetRemoteDescription` 会发生SessionState 状态变化。

pc/sdp_offer_answer.cc

```cpp
RTCError SdpOfferAnswerHandler::UpdateSessionState(
    SdpType type,
    cricket::ContentSource source,
    const cricket::SessionDescription* description) {
  RTC_DCHECK_RUN_ON(signaling_thread());

  // If there's already a pending error then no state transition should happen.
  // But all call-sites should be verifying this before calling us!
  RTC_DCHECK(session_error() == SessionError::kNone);

  // Update the signaling state according to the specified state machine (see
  // https://w3c.github.io/webrtc-pc/#rtcsignalingstate-enum).
  if (type == SdpType::kOffer) {
    ChangeSignalingState(source == cricket::CS_LOCAL
                             ? PeerConnectionInterface::kHaveLocalOffer
                             : PeerConnectionInterface::kHaveRemoteOffer);
  } else if (type == SdpType::kPrAnswer) {
    ChangeSignalingState(source == cricket::CS_LOCAL
                             ? PeerConnectionInterface::kHaveLocalPrAnswer
                             : PeerConnectionInterface::kHaveRemotePrAnswer);
  } else {
    RTC_DCHECK(type == SdpType::kAnswer);
    ChangeSignalingState(PeerConnectionInterface::kStable);
    transceivers()->DiscardStableStates();
    have_pending_rtp_data_channel_ = false;
  }

  // Update internal objects according to the session description's media
  // descriptions.
  RTCError error = PushdownMediaDescription(type, source);
  if (!error.ok()) {
    return error;
  }

  return RTCError::OK();
}
```

1. SdpType, 可以参考对应章节

   ```cpp
   enum class SdpType {
     kOffer,     // Description must be treated as an SDP offer.
     kPrAnswer,  // Description must be treated as an SDP answer, but not a final
                 // answer.
     kAnswer,    // Description must be treated as an SDP final answer, and the
                 // offer-answer exchange must be considered complete after
                 // receiving this.
     kRollback   // Resets any pending offers and sets signaling state back to
                 // stable.
   };
   ```

2. `SdpType::kPrAnswer`,  是个中间状态，A发送了Offer，B收到了对方的offer，但还没准备好，就给A发送`pranswer`的状态，等到B准备好了，再发送完整的answer给A。



### 2.2 SdpOfferAnswerHandler::ChangeSignalingState

pc/sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::ChangeSignalingState(
    PeerConnectionInterface::SignalingState signaling_state) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  if (signaling_state_ == signaling_state) {
    return;
  }
  RTC_LOG(LS_INFO) << "Session: " << pc_->session_id() << " Old state: "
                   << GetSignalingStateString(signaling_state_)
                   << " New state: "
                   << GetSignalingStateString(signaling_state);
  signaling_state_ = signaling_state;
  // PeerConnectionObserver::OnSignalingChange
  pc_->Observer()->OnSignalingChange(signaling_state_);
}
```

通知应用层状态变化`PeerConnectionObserver::OnSignalingChange`。



## 参考

https://w3c.github.io/webrtc-pc/#rtcsignalingstate-enum

[[RFC8829\] JavaScript Session Establishment Protocol (JSEP)-CSDN博客](https://blog.csdn.net/aggresss/article/details/123516956)
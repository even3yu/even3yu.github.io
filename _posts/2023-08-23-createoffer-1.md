---
layout: post
title: webrtc create offer-1
date: 2023-08-23 14:56:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---

## 0. 前言

- CreateOffer是一个不断搜集信息、然后形成offer、通告结果的过程。

- 搜集信息：实际上是形成结构体MediaSessionOptions，并不断填充该结构体的过程。这些信息来源于PeerConnection::CreateOffer的入参RTCOfferAnswerOptions、当前的已被应用的Offer(主要是ContentInfos)、PeerConnection.transceivers_成员。主要集中在PeerConnection::GetOptionsForOffer实现填充过程。

- 形成Offer：实际上是根据搜集的信息MediaSessionOptions，经过一系列的函数调用来构建Offer对象的过程。Offer SDP实质上是JsepSessionDescription对象，不过该对象中重要的成员SessionDescription承载了绝大多数信息。

- 通告结果：不论Offer创建成功，还是失败，最终需要做两件事。一件是通告用户侧Offer创建成功还是失败；一件是触发操作链的下一个操作。这个是通过CreateSessionDescriptionObserverOperationWrapper对象封装创建Offer回调接口、封装操作链操作完成回调，并在CreateOffer过程中一直往下传递，直到创建失败或者成功的地方被触发，来实现的。

- 此外：不论是搜集信息，还是形成Offer都需要参考当前已被应用的Offer中的信息，以便复用部分信息，并使得两次Offer中同样的m-line处于同样的位置。

![createoffer-1]({{ site.url }}{{ site.baseurl }}/images/create-offer-1.assets/createoffer-1.png)

![createoffer-1]({{ site.url }}{{ site.baseurl }}/images/create-offer-1.assets/create-offer.jpg)
SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer()会遍历PC中所有的RtpTransceiver(RtpTransceiver是在addTrack的时候生成的)，**为每个RtpTransceiver创建一个媒体描述信息对象MediaDescriptionOptions**，在最终的生成的SDP对象中，**一个MediaDescriptionOptions就是一个m-line**。 根据由于之前的分析，一个Track对应一个RtpTransceiver，实质上在SDP中一个track就会对应到一个m-line。上述遍历形成所有媒体描述信息MediaDescriptionOptions会存入到MediaSessionOptions对象中，该对象在后续过程中一路传递，最终**在MediaSessionDescriptionFactory::CreateOffer()方法中被用来完成SDP创建**。

另外MediaSessionDescriptionFactory::CreateOffer() 创建SDP过程中，会为每个媒体对象，即每个track：audio、video、data创建对应的MediaContent。上图右边展示了为视频track创建VideoContent过程，标黄的静态方法CreateStreamParamsForNewSenderWithSsrcs()会为每个RtpSender生成唯一的ssrc值。ssrc是个关键信息，正如之前分析，但需要说明的一点是此处并不会调用RtpSender->SetSsrc()方法，ssrc当前只存在于SDP信息中，等待SetLocalDescription()的解析。



## 1.  CreateSessionDescriptionObserver——sdp创建的监听

api/jsep.h

```cpp
// CreateOffer and CreateAnswer callback interface.
class RTC_EXPORT CreateSessionDescriptionObserver
    : public rtc::RefCountInterface {
 public:
  // This callback transfers the ownership of the |desc|.
  // TODO(deadbeef): Make this take an std::unique_ptr<> to avoid confusion
  // around ownership.
  virtual void OnSuccess(SessionDescriptionInterface* desc) = 0;
  // The OnFailure callback takes an RTCError, which consists of an
  // error code and a string.
  // RTCError is non-copyable, so it must be passed using std::move.
  // Earlier versions of the API used a string argument. This version
  // is removed; its functionality was the same as passing
  // error.message.
  virtual void OnFailure(RTCError error) = 0;

 protected:
  ~CreateSessionDescriptionObserver() override = default;
};
```

examples/peerconnection/client/conductor.cc

```cpp
class Conductor : ...
                  public webrtc::CreateSessionDescriptionObserver,
                  ... {

    void Conductor::OnSuccess(webrtc::SessionDescriptionInterface* desc) {
        peer_connection_->SetLocalDescription(
            DummySetSessionDescriptionObserver::Create(), desc);
           ... 
    }
}
```

## 2. RTCOfferAnswerOptions

api/peer_connection_interface.h

```cpp
 // See: https://www.w3.org/TR/webrtc/#idl-def-rtcofferansweroptions
  struct RTCOfferAnswerOptions {
    static const int kUndefined = -1;
    static const int kMaxOfferToReceiveMedia = 1;

    // The default value for constraint offerToReceiveX:true.
    static const int kOfferToReceiveMediaTrue = 1;

    // These options are left as backwards compatibility for clients who need
    // "Plan B" semantics. Clients who have switched to "Unified Plan" semantics
    // should use the RtpTransceiver API (AddTransceiver) instead.
    //
    // offer_to_receive_X set to 1 will cause a media description to be
    // generated in the offer, even if no tracks of that type have been added.
    // Values greater than 1 are treated the same.
    //
    // If set to 0, the generated directional attribute will not include the
    // "recv" direction (meaning it will be "sendonly" or "inactive".
    int offer_to_receive_video = kUndefined;
    int offer_to_receive_audio = kUndefined;

    bool voice_activity_detection = true;
    bool ice_restart = false;

    // If true, will offer to BUNDLE audio/video/data together. Not to be
    // confused with RTCP mux (multiplexing RTP and RTCP together).
    bool use_rtp_mux = true;

    // If true, "a=packetization:<payload_type> raw" attribute will be offered
    // in the SDP for all video payload and accepted in the answer if offered.
    bool raw_packetization_for_video = false;

    // This will apply to all video tracks with a Plan B SDP offer/answer.
    int num_simulcast_layers = 1;

    // If true: Use SDP format from draft-ietf-mmusic-scdp-sdp-03
    // If false: Use SDP format from draft-ietf-mmusic-sdp-sdp-26 or later
    bool use_obsolete_sctp_sdp = false;

		...
  };
```



## 3. PeerConnection::CreateOffer

pc/peer_connection.cc

```cpp
void PeerConnection::CreateOffer(CreateSessionDescriptionObserver* observer,
                                 const RTCOfferAnswerOptions& options) {
...
  // 排队执行
  sdp_handler_->CreateOffer(observer, options);
}
```

1. sdp_handler_ 就是`SdpOfferAnswerHandler`，是在PeerConnnection中创建的对象（【章节4.1.2.3】）

2. 参数 CreateSessionDescriptionObserver* observer 就是【章节2】中创建的
   
   注意：CreateSessionDescriptionObserver只是一个接口，没有具体实现。一般用户层需要继承，并实现CreateSessionDescriptionObserver的方法，以便用户侧感知CreateOffer状态。
   
   另外，WebRTC内部提供了两个实现了CreateSessionDescriptionObserver接口的类，ImplicitCreateSessionDescriptionObserver && CreateSessionDescriptionObserverOperationWrapper。在后续分析过程中再来聊聊这两个实现所起的作用。

   > `CreateSessionDescriptionObserver::OnSuccess(webrtc::SessionDescriptionInterface* desc) `
   > 回调通知 create offer 创建成功，通知给了sdp `webrtc::SessionDescriptionInterface* desc`（），
   > 而 webrtc::SessionDescriptionInterface 的真实对象就是 JsepSessionDescription。



### 3.1 SdpOfferAnswerHandler::CreateOffer

pc/sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::CreateOffer(
    CreateSessionDescriptionObserver* observer,
    const PeerConnectionInterface::RTCOfferAnswerOptions& options) {
  // Chain this operation. If asynchronous operations are pending on the chain,
  // this operation will be queued to be invoked, otherwise the contents of the
  // lambda will execute immediately.
  // 1.排队执行【具体参考OperationChain】
  operations_chain_->ChainOperation(
      [this_weak_ptr = weak_ptr_factory_.GetWeakPtr(), // 2.
       observer_refptr =
           rtc::scoped_refptr<CreateSessionDescriptionObserver>(observer), // 3.
       options](std::function<void()> operations_chain_callback) { // start!!!
        // Abort early if |this_weak_ptr| is no longer valid.
        // 4.如果this_weak_ptr为空，意味着当前PC已经不存在，会话被关闭
        if (!this_weak_ptr) {
          // 4.1 通知用户侧CreateOffer失败及失败原因
          observer_refptr->OnFailure(
              RTCError(RTCErrorType::INTERNAL_ERROR,
                       "CreateOffer failed because the session was shut down"));
          // 4.2 执行操作结束的回调，通知执行下一个Operation
          operations_chain_callback();
          return;
        }
        // The operation completes asynchronously when the wrapper is invoked.
        // 5.执行真正的DoCreateOffer
        // 5.1 创建入参Observer的一个Wrapper对象，该对象还封装了操作回调函数的指针，
        // 使得CreatOffer结束后，能够调用回调函数，通知执行下一个Operation，同时能够通知
        // 用户侧本次CreatOffer的结果。
        rtc::scoped_refptr<CreateSessionDescriptionObserverOperationWrapper>
            observer_wrapper(new rtc::RefCountedObject<
                             CreateSessionDescriptionObserverOperationWrapper>(
                std::move(observer_refptr),
                std::move(operations_chain_callback)));
        // 6. 调用DoCreateOffer进一步去创建Offer
        this_weak_ptr->DoCreateOffer(options, observer_wrapper);
      });// end!!!
}
```

- WebRTC中将CreateOffer、CreateAnswer、SetLocalDescription、SetRemoteDescription、AddIceCandidate这5个与SDP会话相关的API认为是一个Operation，**这些Operation必须是挨个执行，不能乱序，不能同时有两个交互执行**。因此，设计了一套操作链的接口，由OperationsChain类提供此功能。当链入一个操作时，如果队列中没有其他操作，那么该操作会被立马执行；若是操作链中存在操作，那么本操作就入队操作链，等待上一个操作执行完成之后，以回调的形式（即上述代码中的operations_chain_callback回调方法）来告知执行下一步操作。~~具体实现可见文章：WebRTC源码分析——操作链实现OperationsChain~~

- CreateSessionDescriptionObserverOperationWrapper相当于一个封装了 "Offer操作结果回调 + 操作链操作完成回调"的一个对象，一直沿着CreateOffer调用链往下传，直到能够判断是否能成功创建Offer的地方，创建Offer这个操作完成的地方，然后去触发其承载的回调函数，以便告知上层操作结果，然后触发下一个操作。

- `rtc::WeakPtrFactory<PeerConnection> weak_ptr_factory_`：在构造PeerConnection时，传入了this指针。当从`weak_ptr_factory_`获取弱指针this_weak_ptr不存在时，意味着PC已经不存在了，也即当前会话已被关闭。这样的功能是由rtc::WeakPtrFactory && WeakPtr带来的，~~详见 WebRTC源码分析——弱指针WeakPtrFactory && WeakPtr~~。要注意的是weak_ptr_factory_必须声明在PC的最后，这样是为了：
  
  ```
    // |weak_ptr_factory_| must be declared last to make sure all WeakPtr's are
   // invalidated before any other members are destroyed.
  ```



### 3.2 SdpOfferAnswerHandler::DoCreateOffer

pc/sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::DoCreateOffer(
    const PeerConnectionInterface::RTCOfferAnswerOptions& options,
    rtc::scoped_refptr<CreateSessionDescriptionObserver> observer) {
  // 1. 状态判断
    ...

  // 1.5 验证options的合法性
  //     实际就是判断offer_to_receive_audio && offer_to_receive_video
  //     这两个参数是否合法(取值在kUndefined~kMaxOfferToReceiveMedia之间)
  //     默认二者皆为kUndefined。
  if (!ValidateOfferAnswerOptions(options)) {
    std::string error = "CreateOffer called with invalid options.";
    RTC_LOG(LS_ERROR) << error;
    pc_->message_handler()->PostCreateSessionDescriptionFailure(
        observer, RTCError(RTCErrorType::INVALID_PARAMETER, std::move(error)));
    return;
  }

  // Legacy handling for offer_to_receive_audio and offer_to_receive_video.
  // Specified in WebRTC section 4.4.3.2 "Legacy configuration extensions".
  // 1.6 如果是Unified Plan，处理options中遗留的字段
  if (IsUnifiedPlan()) {
    // 【章节3.3】
    RTCError error = HandleLegacyOfferOptions(options);
    if (!error.ok()) {
      pc_->message_handler()->PostCreateSessionDescriptionFailure(
          observer, std::move(error));
      return;
    }
  }

  // 2 获取MediaSessionOptions信息，为创建Offer提供信息
  //   MediaSessionOptions包含了创建Offer时对每个mline都适用的公共规则，
  //   并且为每个m-line都准备了一个MediaDescriptionOptions 【章节3.4】
  cricket::MediaSessionOptions session_options;
  GetOptionsForOffer(options, &session_options);
  // 3 执行WebRtcSessionDescriptionFactory::CreateOffer来创建Offer 【章节3.5】
  webrtc_session_desc_factory_->CreateOffer(observer, options, session_options);
}
```

- 对入参和当前状态的一些判断（如源码所示共6点），若这些条件和状态不对，则PostCreateSessionDescriptionFailure方法将错误信息post出去，并且不再继续创建Offer的后续动作；

- 获取MediaSessionOptions信息，然后调用WebRtcSessionDescriptionFactory::CreateOffer来实际创建Offer.
  
  ![media_session_option_value]({{ site.url }}{{ site.baseurl }}/images/create-offer-1.assets/media_session_option_value.png)



#### 3.2.1 SdpOfferAnswerHandler::HandleLegacyOfferOptions

pc/sdp_offer_answer.cc

```cpp
RTCError SdpOfferAnswerHandler::HandleLegacyOfferOptions(
    const PeerConnectionInterface::RTCOfferAnswerOptions& options) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(IsUnifiedPlan());
    // 1. 处理音频（offer_to_receive_audio）
 // 1.1 为0，不接受音频流，遍历移除
  if (options.offer_to_receive_audio == 0) {
    RemoveRecvDirectionFromReceivingTransceiversOfType(
        cricket::MEDIA_TYPE_AUDIO);
    // 1.2  为1，接受音频流，遍历添加
  } else if (options.offer_to_receive_audio == 1) {
    AddUpToOneReceivingTransceiverOfType(cricket::MEDIA_TYPE_AUDIO);
    // 1.3  >1，参数错误
  } else if (options.offer_to_receive_audio > 1) {
    LOG_AND_RETURN_ERROR(RTCErrorType::UNSUPPORTED_PARAMETER,
                         "offer_to_receive_audio > 1 is not supported.");
  }

  // 2. 处理视频（offer_to_receive_video）
  // 2.1 为0，不接受视频流，遍历移除
  if (options.offer_to_receive_video == 0) {
    RemoveRecvDirectionFromReceivingTransceiversOfType(
        cricket::MEDIA_TYPE_VIDEO);
 // 2.2 为1，接受视频流，遍历添加
  } else if (options.offer_to_receive_video == 1) {
    AddUpToOneReceivingTransceiverOfType(cricket::MEDIA_TYPE_VIDEO);
    // 2.3 >1，参数错误
  } else if (options.offer_to_receive_video > 1) {
    LOG_AND_RETURN_ERROR(RTCErrorType::UNSUPPORTED_PARAMETER,
                         "offer_to_receive_video > 1 is not supported.");
  }

  return RTCError::OK();
}
```

当采用Unified Plan时，需要针对options的offer_to_receive_audio和offer_to_receive_audio进行处理，当offer_to_receive_xxx为0表示本端不接收对应的流，offer_to_receive_xxx为1表示接收。需要对PC所持有的transceivers_进行遍历处理。

1）当不接收流时RemoveRecvDirectionFromReceivingTransceiversOfType进行处理：

```cpp
void SdpOfferAnswerHandler::RemoveRecvDirectionFromReceivingTransceiversOfType(
    cricket::MediaType media_type) {
  // 通过GetReceivingTransceiversOfType遍历transceivers_，获取所有对应媒体类型的
  // 传输方向包含recv的的Transceivers。然后再遍历这些符合条件的Transceivers。
  for (const auto& transceiver : GetReceivingTransceiversOfType(media_type)) {
    // 通过RtpTransceiverDirectionWithRecvSet方法获取新方向，新的方向中应保留
    // 旧方向中的Send（旧方向中若存在的话）。
    RtpTransceiverDirection new_direction =
        RtpTransceiverDirectionWithRecvSet(transceiver->direction(), false);
    // 若新方向与旧方向是不一致的，因此，有改变，调用transceiver的set_direction
    // 设置为新方向。
    if (new_direction != transceiver->direction()) {
      RTC_LOG(LS_INFO) << "Changing " << cricket::MediaTypeToString(media_type)
                       << " transceiver (MID="
                       << transceiver->mid().value_or("<not set>") << ") from "
                       << RtpTransceiverDirectionToString(
                              transceiver->direction())
                       << " to "
                       << RtpTransceiverDirectionToString(new_direction)
                       << " since CreateOffer specified offer_to_receive=0";
      // 更改方向Transceiver方向
      transceiver->internal()->set_direction(new_direction);
    }
  }
}
```

注意：GetReceivingTransceiversOfType返回的是`std::vector<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>`，而不是直接承载RtpTransceiver的vector。这样做的好处是使得对RtpTransceiver的操作都能通过RtpTransceiverProxyWithInternal被代理到对应的线程上去执行。最终，简单的只是修改了RtpTransceiver的direction_属性。

2）当接收流时AddUpToOneReceivingTransceiverOfType进行处理：

```cpp
void SdpOfferAnswerHandler::AddUpToOneReceivingTransceiverOfType(
    cricket::MediaType media_type) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  // 遍历PC::transceivers_，若所有的该媒体类型的transceiver都不接收流
  // 则创建一个新的transceiver，该transceiver的方向为kRecvOnly
  if (GetReceivingTransceiversOfType(media_type).empty()) {
    RTC_LOG(LS_INFO)
        << "Adding one recvonly " << cricket::MediaTypeToString(media_type)
        << " transceiver since CreateOffer specified offer_to_receive=1";
    RtpTransceiverInit init;
    init.direction = RtpTransceiverDirection::kRecvOnly;
    pc_->AddTransceiver(media_type, nullptr, init,
                        /*update_negotiation_needed=*/false);
  }
}
```

注意：与上面的处理并不对称，并不会去修改已存在的Transceiver的方向。

下面函数主要是根据mediaType 获取 tranceiver，过滤tranceiver

```cpp
std::vector<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
SdpOfferAnswerHandler::GetReceivingTransceiversOfType(
    cricket::MediaType media_type) {
  std::vector<
      rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
      receiving_transceivers;
  for (const auto& transceiver : transceivers()->List()) {
    if (!transceiver->stopped() && transceiver->media_type() == media_type &&
        RtpTransceiverDirectionHasRecv(transceiver->direction())) {
      receiving_transceivers.push_back(transceiver);
    }
  }
  return receiving_transceivers;
}
```

#### 3.2.2 SdpOfferAnswerHandler::GetOptionsForOffer

通过RTCOfferAnswerOptions 创建 MediaSessionOptions。
MediaSessionOptions 除了一些公共部的一些属性， 还存放了每个 m-line 特有的属性，多个mline以数组形式存放。
参考【章节4】。

#### 3.2.3 --WebRtcSessionDescriptionFactory::CreateOffer

根据MediaSessionOptions 创建 JsepSessionDescription， 当然JsepSessionDescription 主要的属性 SessionDescription。
参考【章节5】。

## 4. !!! SdpOfferAnswerHandler::GetOptionsForOffer

pc/sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::GetOptionsForOffer(
    const PeerConnectionInterface::RTCOfferAnswerOptions& offer_answer_options,
    cricket::MediaSessionOptions* session_options) {

   // 1. 从offer_answer_options抽取构建SDP时，所有mline共享的信息，放到session_options
  //    的公共字段，此方法从offer_answer_options拷贝的公共字段有：
  //      vad_enabled：是否使用静音检测
  //      bundle_enabled: 是否所有媒体数据都成为一个Bundle Gruop，从而复用一个底层传输通道
  //      raw_packetization_for_video：对sdp中所有video负载将产生
  //                    "a=packetization:<payload_type> raw"这样的属性描述。
  //【章节3.4.2】
  ExtractSharedMediaSessionOptions(offer_answer_options, session_options);

  // 2. 为每个mline，创建MediaDescriptionOptions存入MediaSessionOptions
  //【章节3.4.3】
  if (IsUnifiedPlan()) {
    GetOptionsForUnifiedPlanOffer(offer_answer_options, session_options);
  } else {
    ...
  }

  ...

  // 4. 复制ICE restart标识，
  //    并将ice restart标识和renomination标识赋值到每个mline对应的MediaDescript
  bool ice_restart = offer_answer_options.ice_restart || HasNewIceCredentials();
  for (auto& options : session_options->media_description_options) {
    options.transport_options.ice_restart = ice_restart;
    options.transport_options.enable_ice_renomination =
        pc_->configuration()->enable_ice_renomination;
  }

  // 5. 复制cname，加密算法选项，加密证书，extmap-allow-mixed属性
  session_options->rtcp_cname = rtcp_cname_;
  session_options->crypto_options = pc_->GetCryptoOptions();
  session_options->pooled_ice_credentials =
      pc_->network_thread()->Invoke<std::vector<cricket::IceParameters>>(
          RTC_FROM_HERE,
          rtc::Bind(&cricket::PortAllocator::GetPooledIceCredentials,
                    port_allocator()));
  session_options->offer_extmap_allow_mixed =
      pc_->configuration()->offer_extmap_allow_mixed;

  // Allow fallback for using obsolete SCTP syntax.
  // Note that the default in |session_options| is true, while
  // the default in |options| is false.
   // 是否允许回退到使用过时的sctp sdp
  session_options->use_obsolete_sctp_sdp =
      offer_answer_options.use_obsolete_sctp_sdp;
}
```





### 4.0 MediaSessionOptions和MediaDescriptionOptions类关系图

![media_session_option]({{ site.url }}{{ site.baseurl }}/images/create-offer-1.assets/media_session_option.png)

MediaSessionOptions提供了一个应该如何生成m-line的机制。一方面，MediaSessionOptions提供了适用于所有m-line的参数；另一方面，MediaSessionOptions对于每个具体的m-line，有差异性的参数使用 `std::vector<MediaDescriptionOptions> MediaSessionOptions::media_description_options`中的对应的那个MediaDescriptionOptions所提供的规则，注意`MediaSessionOptions::media_description_options`的下标和m-line在sdp中的顺序是一致的。

### 4.1 MediaSessionOptions

pc/media_session.h

```cpp
struct MediaSessionOptions {
  MediaSessionOptions() {}

  bool has_audio() const { return HasMediaDescription(MEDIA_TYPE_AUDIO); }
  bool has_video() const { return HasMediaDescription(MEDIA_TYPE_VIDEO); }
  bool has_data() const { return HasMediaDescription(MEDIA_TYPE_DATA); }

  bool HasMediaDescription(MediaType type) const;
  /////////////
  bool MediaSessionOptions::HasMediaDescription(MediaType type) const {
  return absl::c_any_of(
      media_description_options,
      [type](const MediaDescriptionOptions& t) { return t.type == type; });
    }
  ///////////


  DataChannelType data_channel_type = DCT_NONE;
  bool vad_enabled = true;  // When disabled, removes all CN codecs from SDP.
  bool rtcp_mux_enabled = true;
  bool bundle_enabled = false;
  bool offer_extmap_allow_mixed = false;
  bool raw_packetization_for_video = false;
  std::string rtcp_cname = kDefaultRtcpCname;
  webrtc::CryptoOptions crypto_options;
  // List of media description options in the same order that the media
  // descriptions will be generated.
  // 存放了每一个mline的相关属性
  std::vector<MediaDescriptionOptions> media_description_options;
  std::vector<IceParameters> pooled_ice_credentials;

  // Use the draft-ietf-mmusic-sctp-sdp-03 obsolete syntax for SCTP
  // datachannels.
  // Default is true for backwards compatibility with clients that use
  // this internal interface.
  bool use_obsolete_sctp_sdp = true;
};
```

MediaSessionOptions 属性分为两部分，

- 公共的属性：vad_enabled，bundle_enabled，raw_packetization_for_video；
  `ExtractSharedMediaSessionOptions` 就是来处理共享属性的【章节3.4.2】。
- m-line特有属性：就是每个mline特有的，分别放在各自创建MediaDescriptionOptions【章节3.4.3】。
  `std::vector<MediaDescriptionOptions> media_description_options` 中元素MediaDescriptionOptions，**注意media_description_options的下标和m-line在sdp中的顺序是一致的**。



#### raw_packetization_for_video ???



### 4.2 MediaDescriptionOptions

pc/media_session.h

```cpp
// Options for an individual media description/"m=" section.
struct MediaDescriptionOptions {
  MediaDescriptionOptions(MediaType type,
                          const std::string& mid,
                          webrtc::RtpTransceiverDirection direction,
                          bool stopped)
      : type(type), mid(mid), direction(direction), stopped(stopped) {}

  // TODO(deadbeef): When we don't support Plan B, there will only be one
  // sender per media description and this can be simplified.
  void AddAudioSender(const std::string& track_id,
                      const std::vector<std::string>& stream_ids);
  void AddVideoSender(const std::string& track_id,
                      const std::vector<std::string>& stream_ids,
                      const std::vector<RidDescription>& rids,
                      const SimulcastLayerList& simulcast_layers,
                      int num_sim_layers);

  // Internally just uses sender_options.
  void AddRtpDataChannel(const std::string& track_id,
                         const std::string& stream_id);

  MediaType type;
  std::string mid;
  webrtc::RtpTransceiverDirection direction;
  bool stopped;
  TransportOptions transport_options;
  // Note: There's no equivalent "RtpReceiverOptions" because only send
  // stream information goes in the local descriptions.
  std::vector<SenderOptions> sender_options;
  std::vector<webrtc::RtpCodecCapability> codec_preferences;
  std::vector<webrtc::RtpHeaderExtensionCapability> header_extensions;

 private:
  // Doesn't DCHECK on |type|.
  void AddSenderInternal(const std::string& track_id,
                         const std::vector<std::string>& stream_ids,
                         const std::vector<RidDescription>& rids,
                         const SimulcastLayerList& simulcast_layers,
                         int num_sim_layers);
};
```

- 区分MediaDescriptionOptions 就是 `std::string mid;`

- webrtc::RtpTransceiverDirection direction;
  src/api/rtp_transceiver_direction.h

  ```cpp
  enum class RtpTransceiverDirection {
    kSendRecv,
    kSendOnly,
    kRecvOnly,
    kInactive,
    kStopped,
  };
  ```



### 4.3 ExtractSharedMediaSessionOptions

pc/sdp_offer_answer.cc

```cpp
// From |rtc_options|, fill parts of |session_options| shared by all generated
// m= sectionss (in other words, nothing that involves a map/array).
void ExtractSharedMediaSessionOptions(
    const PeerConnectionInterface::RTCOfferAnswerOptions& rtc_options,
    cricket::MediaSessionOptions* session_options) {
  // 语音静音检测
  session_options->vad_enabled = rtc_options.voice_activity_detection;
  // auido，video等是否bundle 一起
  session_options->bundle_enabled = rtc_options.use_rtp_mux;
  // 对sdp中所有video负载将产生 "a=packetization:<payload_type> raw"这样的属性描述。
  session_options->raw_packetization_for_video =
      rtc_options.raw_packetization_for_video;
}
```

对MediaSessionOptions共享属性进行赋值。


### 4.4 !!! SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer

pc/sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer(
    const RTCOfferAnswerOptions& offer_answer_options,
    cricket::MediaSessionOptions* session_options) {
  // Rules for generating an offer are dictated by JSEP sections 5.2.1 (Initial
  // Offers) and 5.2.2 (Subsequent Offers).
  RTC_DCHECK_EQ(session_options->media_description_options.size(), 0);

  // 1.获取 local_contents和 remote_contents 的ContentInfos
  // ContentInfos 是 存放了所有的mline对应的数据；
  // ContentInfo存放了一个mline，ContentInfo和mline一一对应
  
  // typedef std::vector<ContentInfo> ContentInfos; 向量
  const ContentInfos no_infos;
  const ContentInfos& local_contents =
      (local_description() ? local_description()->description()->contents()
                           : no_infos);
  const ContentInfos& remote_contents =
      (remote_description() ? remote_description()->description()->contents()
                            : no_infos);
  
  // 1. 重用transceiver，先回收回收，就是重置一些状态，
	// MediaDescriptionOptions::stop_ = true
	// 把回收的mline的下标index记录在recycleable_mline_indices， 
	// 
	// (1)根据mid 找到对应的transceiver， 如果没找到，
	// 则认为这个mid 对应的 MediaDescriptionOptions 无效，可以复用， 
	// (2)ContentInfo.rejected && transceiver->stopping() , 则 也是可以复用
	//
  std::queue<size_t> recycleable_mline_indices;
  // First, go through each media section that exists in either the local or
  // remote description and generate a media section in this offer for the
  // associated transceiver. If a media section can be recycled, generate a
  // default, rejected media section here that can be later overwritten.
  for (size_t i = 0;
       i < std::max(local_contents.size(), remote_contents.size()); ++i) {
    // Either |local_content| or |remote_content| is non-null.
    const ContentInfo* local_content =
        (i < local_contents.size() ? &local_contents[i] : nullptr);
    // current_local_description(),local_contents= local_description() 区别???
    const ContentInfo* current_local_content =
        GetContentByIndex(current_local_description(), i);
    const ContentInfo* remote_content =
        (i < remote_contents.size() ? &remote_contents[i] : nullptr);
    const ContentInfo* current_remote_content =
        GetContentByIndex(current_remote_description(), i);

    bool had_been_rejected =
        (current_local_content && current_local_content->rejected) ||
        (current_remote_content && current_remote_content->rejected);
    const std::string& mid =
        (local_content ? local_content->name : remote_content->name);
    cricket::MediaType media_type =
        (local_content ? local_content->media_description()->type()
                       : remote_content->media_description()->type());
    if (media_type == cricket::MEDIA_TYPE_AUDIO ||
        media_type == cricket::MEDIA_TYPE_VIDEO) {
      // A media section is considered eligible for recycling if it is marked as
      // rejected in either the current local or current remote description.
      // 通过mid 找到对应的transceiver
      auto transceiver = transceivers()->FindByMid(mid);
      if (!transceiver) {
        // 没有对应的transceiver， 回收该下标
        recycleable_mline_indices.push(i);
        session_options->media_description_options.push_back(
            cricket::MediaDescriptionOptions(media_type, mid,
                                             RtpTransceiverDirection::kInactive,
                                             /*stopped=*/true));
      } else {
        // NOTE: a stopping transceiver should be treated as a stopped one in
        // createOffer as specified in
        // https://w3c.github.io/webrtc-pc/#dom-rtcpeerconnection-createoffer.
        // 有对应的tranceiver，且ContentInfo::rejection = true，同时tranceiver是停止的
        // 可回收复用的（即对应的ContentInfo被标记为rejected，transceiver标记为stopped）
        // 回收该下标
        //
        if (had_been_rejected && transceiver->stopping()) {
          session_options->media_description_options.push_back(
              cricket::MediaDescriptionOptions(
                  transceiver->media_type(), mid,
                  RtpTransceiverDirection::kInactive,
                  /*stopped=*/true));
          // 回收该下标
          recycleable_mline_indices.push(i);
        } else {
          // GetMediaDescriptionOptionsForTransceiver【章节3.4.4】
          // 有效的，则使用GetMediaDescriptionOptionsForTransceiver
          // 根据transceiver和之前的mid来构造MediaDescriptionOptions，占用下标
          session_options->media_description_options.push_back(
              GetMediaDescriptionOptionsForTransceiver(
                  transceiver, mid,
                  /*is_create_offer=*/true));
          
          // 给transceiver设置mline的index
          transceiver->internal()->set_mline_index(i);
        }
      }
    } else if (media_type == cricket::MEDIA_TYPE_UNSUPPORTED) {
      ...
    } else {
      ...
    }
  }

  // 2.通过找tranceivers中，没有绑定mid或者没有停止的的tranceiver，
  // 修改MediaDescriptionOptions
  for (const auto& transceiver : transceivers()->List()) {
    // 已经绑定mid或者tranceiver的状态为stop的
    if (transceiver->mid() || transceiver->stopping()) {
      continue;
    }
    size_t mline_index;
    if (!recycleable_mline_indices.empty()) {
      // 复用上面回收的下标，就是mline_index，m-line
      // 给mline_index对应的mline重新新建MediaDescriptionOptions
      // mid 是重新生成的
      mline_index = recycleable_mline_indices.front();
      recycleable_mline_indices.pop();
      session_options->media_description_options[mline_index] =
          GetMediaDescriptionOptionsForTransceiver(
              transceiver, mid_generator_.GenerateString(),
              /*is_create_offer=*/true);
    } else {
      // 没有可重用的，直接就新增mline_index
      mline_index = session_options->media_description_options.size();
      session_options->media_description_options.push_back(
          GetMediaDescriptionOptionsForTransceiver(
              transceiver, mid_generator_.GenerateString(),
              /*is_create_offer=*/true));
    }
    // 设置 m-line index，下标
    // See comment above for why CreateOffer changes the transceiver's state.
    transceiver->internal()->set_mline_index(mline_index);
  }
  
	...
}
```

- 获取本地`local_description()->description()->contents()`，远端 `remote_description()->description()->contents()`
  存放在 ContentInfos中，就是ContentInfo的向量；ContentInfo 就是mline的相关信息。一开始local_description()和remote_description()是空，经过setLocalDescription和setRemoteDescription后才会有值。

-  for 循环ContentInfos， 找到可以重用的mline的下标；把回收的mline的下标index记录在recycleable_mline_indices，
  有两种情况：
  (1) 根据mid 找到对应的transceiver， 如果没找到，则认为这个mid 对应的 MediaDescriptionOptions 无效，可以复用， 
  (2) ContentInfo.rejected && transceiver->stopping() , 则 也是可以复用

  > 刚开始createoffer的时候，
  > local_contents 和 remote_contents 都是空的；
  > transceivers 有2个, audio/video；

  这里会根据ContentInfos的数据，转换为 MediaDescriptionOptions 向量， 存放在session_options->media_description_options中；

- 最终的目的，把 transceivers 转换为 MediaDescriptionOptions 向量， 存放在session_options->media_description_options中；
  首先使用上面找到记录在recycleable_mline_indices 的可重用的下标，如果没有了再新增下标；

> **关于 m-line index 下标 复用问题说明：**
>
> 获取每个mline独享的参数MediaDescriptionOptions。本质上，每个mline的MediaDescriptionOptions信息可以从 transceiver 和 为其分配的mid 二者得来（上面的两个for循环），调用一个GetMediaDescriptionOptionsForTransceiver方法即可搞定。但为啥本方法会如此复杂呢？因为要考虑复用，之前可能已经进行过协商，但是没有达成一致，此时，就需要考虑这么样的情况：比方说，之前offer中包含3路流（1、2、3），协商时，2被自己或者对方拒绝。一方面，本地或者远端的SessionDescription对象中2所对应的内容被标记为rejected，另一方面transcervers_中的第二个transcerver会变成stopped，此时2处于可复用的状态。若不添加新流的情况下，再次协商，则只有1、3两路流是有效的，为了保持与前面的协商顺序一致，即之前的1、3仍位于1、3的位置，2会设置为inactive。若添加了新的轨道，再次协商时，之前的1、3仍位于1、3，2则会被新的轨道所在的transcerver复用。

> **关于set_mline_index**
>
> （1）根据transceiver，和上一次的协商好的localDescription，remoteDescription 。如果能够根据ContentInfo::mid，如果找到对应的transceiver， 且 `ContentInfo::rejection  && transceiver->stopping()` 这条件不满足，则会设置set_mline_index， **注意设置的值，是累加的**。
>
> （2）如果（1）中的有不可用的transceiver，则认为mline_index是可以复用的，遍历所有的transceiver，满足`transceiver->mid() || transceiver->stopping()` 则不需要绑定，因为在（1）中已经绑定。其他重新创建`MediaDescriptionOptions` 并设置 set_mline_index。



#### mid_generator_.GenerateString()

这里生成了mid。



#### SessionDescription

pc\session_description.h
SessionDescription 主要是管理了ContentInfo， 存了了一个向量，ContentInfos来表示。

#### ContentInfo

pc\session_description.h
ContentInfo 主要管理 mline对应的相关信息， 存放在MediaContentDescription；

#### 关系图

![jsep-session-description]({{ site.url }}{{ site.baseurl }}/images/create-offer-1.assets/jsep-session-description.png)

#### !!! 重要关系

| option                  | 转换 | Transceiver |
| ----------------------- | ---- | ----------- |
| MediaDescriptionOptions | <--> | Transceiver |
|                         |      |             |
|                         |      |             |

- MediaDescriptionOptions 的对象个数 是以 TransceiverList 中的个数为主；有多少个Transceiver，就有多少个mline， 有多少个MediaDescriptionOptions；
- MediaDescriptionOptions::stopped， 这个属性的值 是根据`  bool stopped = is_create_offer ? transceiver->stopping() : transceiver->stopped();`

GetMediaDescriptionOptionsForTransceiver， 根据is_create_offer ? transceiver->stopping() : transceiver->stopped()
GetOptionsForPlanBOffer， GenerateMediaDescriptionOptions， 新增的，stopped = false
GetOptionsForPlanBAnswer， GenerateMediaDescriptionOptions ， bool stopped = (video_direction == RtpTransceiverDirection::kInactive);
GetOptionsForUnifiedPlanOffer， 对于transceiver不存在的的，stopped = true；
GetOptionsForUnifiedPlanAnswer， 对于transceiver不存在的的，stopped = true；



### 4.5 GetMediaDescriptionOptionsForTransceiver

src\pc\sdp_offer_answer.cc

```cpp
static cricket::MediaDescriptionOptions
GetMediaDescriptionOptionsForTransceiver(
    rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>
        transceiver,
    const std::string& mid,
    bool is_create_offer) { // is_create_offer ,createoffer =true, createanswer = false
  // NOTE: a stopping transceiver should be treated as a stopped one in
  // createOffer as specified in
  // https://w3c.github.io/webrtc-pc/#dom-rtcpeerconnection-createoffer.
  // !!! 这里很重要，
  bool stopped =
      is_create_offer ? transceiver->stopping() : transceiver->stopped();
  // 创建 MediaDescriptionOptions
  cricket::MediaDescriptionOptions media_description_options(
      transceiver->media_type(), mid, transceiver->direction(), stopped);
  media_description_options.codec_preferences =
      transceiver->codec_preferences();
  media_description_options.header_extensions =
      transceiver->HeaderExtensionsToOffer();
  // This behavior is specified in JSEP. The gist is that:
  // 1. The MSID is included if the RtpTransceiver's direction is sendonly or
  //    sendrecv.
  // 2. If the MSID is included, then it must be included in any subsequent
  //    offer/answer exactly the same until the RtpTransceiver is stopped.
  if (stopped || (!RtpTransceiverDirectionHasSend(transceiver->direction()) &&
                  !transceiver->internal()->has_ever_been_used_to_send())) {
    return media_description_options;
  }

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
  media_description_options.sender_options.push_back(sender_options);

  return media_description_options;
}
```

- 对于没存在可以复用的mline 的 index 下标，则新创建MediaDescriptionOptions。
- 创建cricket::MediaDescriptionOptions
- stopped， `bool stopped = is_create_offer ? transceiver->stopping() : transceiver->stopped();`
- 其他相关属性赋值



## 5. WebRtcSessionDescriptionFactory::CreateOffer

pc\webrtc_session_description_factory.cc
【章节3.2】doCreateOffer中调用的

```cpp
void WebRtcSessionDescriptionFactory::CreateOffer(
    CreateSessionDescriptionObserver* observer,
    const PeerConnectionInterface::RTCOfferAnswerOptions& options,
    const cricket::MediaSessionOptions& session_options) {
  std::string error = "CreateOffer";
     ...

  // 3. 构造创建Offer的请求，根据情况排队执行，或者直接执行
  // 3.1 构造创建Offer的请求
  CreateSessionDescriptionRequest request(
      CreateSessionDescriptionRequest::kOffer, observer, session_options);
  // 3.2 若证书请求状态是CERTIFICATE_WAITING，则请求入队，等待执行
  if (certificate_request_state_ == CERTIFICATE_WAITING) {
    create_session_description_requests_.push(request);
   // 3.2 若证书请求状态是CERTIFICATE_SUCCEEDED已经成功状态或者CERTIFICATE_NOT_NEEDED
  //     不需要证书状态 ，则直接调用InternalCreateOffer来处理生成Offer的请求
  } else {
    RTC_DCHECK(certificate_request_state_ == CERTIFICATE_SUCCEEDED ||
               certificate_request_state_ == CERTIFICATE_NOT_NEEDED);
    InternalCreateOffer(request);
  }
}
```

- certificate_request_state_ = CERTIFICATE_NOT_NEEDED， 不需要证书状态 ，则直接调用InternalCreateOffer来处理生成Offer的请求；
- dtls 是需要证书的，所以certificate_request_state_ = CERTIFICATE_WAITING， 最终生成成功，certificate_request_state_ = CERTIFICATE_SUCCEEDED， 调用InternalCreateOffer来处理生成Offer的请求；
- 可以参考【章节5.4】

### 5.1 CertificateRequestState

WebRtcSessionDescriptionFactory::certificate_request_state_ 成员的取值影响了整个流程处理。那么certificate_request_state_ 取值是如何变化的呢？

```cpp
 enum CertificateRequestState {
    CERTIFICATE_NOT_NEEDED, // 不需要生成证书的，目前就dtls需要证书
    CERTIFICATE_WAITING,
    CERTIFICATE_SUCCEEDED,
    CERTIFICATE_FAILED,
  };
```

### 5.2 CreateSessionDescriptionRequest

pc\webrtc_session_description_factory.h

```cpp
struct CreateSessionDescriptionRequest {
  enum Type {
    kOffer,
    kAnswer,
  };

  CreateSessionDescriptionRequest(Type type,
                                  CreateSessionDescriptionObserver* observer,
                                  const cricket::MediaSessionOptions& options)
      : type(type), observer(observer), options(options) {}

  Type type;
  rtc::scoped_refptr<CreateSessionDescriptionObserver> observer;
  cricket::MediaSessionOptions options;
};
```

### 5.3 WebRtcSessionDescriptionFactory.InternalCreateOffer

根据MediaSessionOptions 创建 JsepSessionDescription，SessionDescription，参考【章节6】。

### 5.4 certificate_request_state_ 状态变更

pc\webrtc_session_description_factory.cc

- 在WebRtcSessionDescriptionFactory构造时，certificate_request_state_默认初始化为CERTIFICATE_NOT_NEEDED。

- 若构造函数中外部传入了certificate（若追根究底，这个certificate是在创建PC时由其配置参数带入的，并且在PC的初始化函数中构建了WebRtcSessionDescriptionFactory，并将该certificate传递进来），那么certificate_request_state_会被设置为CERTIFICATE_WAITING状态，并在信令线程Post一个包含有该certificate的消息（为什么要采用异步方式？是为了让PC能够绑定WebRtcSessionDescriptionFactory信号SignalCertificateReady，从而在后续该异步操作结束时能响应该信号）。
  **WebRtcSessionDescriptionFactory 是在 SdpOfferAnswerHandler  构造的时候创建的**
  
  ```cpp
  WebRtcSessionDescriptionFactory::WebRtcSessionDescriptionFactory(
      rtc::Thread* signaling_thread,
      cricket::ChannelManager* channel_manager,
      const SdpStateProvider* sdp_info,
      const std::string& session_id,
      bool dtls_enabled,
      std::unique_ptr<rtc::RTCCertificateGeneratorInterface> cert_generator,
      const rtc::scoped_refptr<rtc::RTCCertificate>& certificate,
      UniqueRandomIdGenerator* ssrc_generator,
      std::function<void(const rtc::scoped_refptr<rtc::RTCCertificate>&)>
          on_certificate_ready)
      : signaling_thread_(signaling_thread),
        session_desc_factory_(channel_manager,
                              &transport_desc_factory_,
                              ssrc_generator),
        // RFC 4566 suggested a Network Time Protocol (NTP) format timestamp
        // as the session id and session version. To simplify, it should be fine
        // to just use a random number as session id and start version from
        // |kInitSessionVersion|.
        session_version_(kInitSessionVersion),
        cert_generator_(dtls_enabled ? std::move(cert_generator) : nullptr),
        sdp_info_(sdp_info),
        session_id_(session_id),
        certificate_request_state_(CERTIFICATE_NOT_NEEDED), // 1.
        on_certificate_ready_(on_certificate_ready) {
    RTC_DCHECK(signaling_thread_);
  
    ...
  
    if (certificate) {
      // Use |certificate|.
      certificate_request_state_ = CERTIFICATE_WAITING; // 2.
  
      RTC_LOG(LS_VERBOSE) << "DTLS-SRTP enabled; has certificate parameter.";
      // We already have a certificate but we wait to do |SetIdentity|; if we do
      // it in the constructor then the caller has not had a chance to connect to
      // |SignalCertificateReady|.
      signaling_thread_->Post(
          RTC_FROM_HERE, this, MSG_USE_CONSTRUCTOR_CERTIFICATE,
          new rtc::ScopedRefMessageData<rtc::RTCCertificate>(certificate));
    } else {
     ...
    }
  }
  ```

因此，会在WebRtcSessionDescriptionFactory的OnMesaage方法中得到异步处理。最终是在SetCertificate完成证书的设置，状态更新为CERTIFICATE_SUCCEEDED，并发送SignalCertificateReady信号，由于CERTIFICATE_WAITING状态下，创建Offer的请求会排队，在SetCertificate中还会将排队的请求pop出来，调用InternalCreateOffer进行处理。

```cpp
  void WebRtcSessionDescriptionFactory::OnMessage(rtc::Message* msg) {
    switch (msg->message_id) {
      ...
      case MSG_USE_CONSTRUCTOR_CERTIFICATE: {
        rtc::ScopedRefMessageData<rtc::RTCCertificate>* param =
            static_cast<rtc::ScopedRefMessageData<rtc::RTCCertificate>*>(
                msg->pdata);
        RTC_LOG(LS_INFO) << "Using certificate supplied to the constructor.";
        SetCertificate(param->data());
        delete param;
        break;
      }
      default:
        RTC_NOTREACHED();
        break;
    }
  }
```

```cpp
void WebRtcSessionDescriptionFactory::SetCertificate(
    const rtc::scoped_refptr<rtc::RTCCertificate>& certificate) {
  RTC_DCHECK(certificate);
  RTC_LOG(LS_VERBOSE) << "Setting new certificate.";

  // 3.
  certificate_request_state_ = CERTIFICATE_SUCCEEDED;

  on_certificate_ready_(certificate);

  transport_desc_factory_.set_certificate(certificate);
  transport_desc_factory_.set_secure(cricket::SEC_ENABLED);

  while (!create_session_description_requests_.empty()) {
    if (create_session_description_requests_.front().type ==
        CreateSessionDescriptionRequest::kOffer) {
      InternalCreateOffer(create_session_description_requests_.front());
    } else {
      InternalCreateAnswer(create_session_description_requests_.front());
    }
    create_session_description_requests_.pop();
  }
}
```

- 若构造函数中没有传入外部的certificate，**则通过证书生成器来异步产生证书**，并以信号-槽方式来通知WebRtcSessionDescriptionFactory证书生成情况，
  （1）若成功则触发SetCertificate来完成证书设置；
  （2）若失败则触发OnCertificateRequestFailed，将certificate_request_state_更新为CERTIFICATE_FAILED。
  
  ```cpp
  WebRtcSessionDescriptionFactory::WebRtcSessionDescriptionFactory(
      rtc::Thread* signaling_thread,
      cricket::ChannelManager* channel_manager,
      const SdpStateProvider* sdp_info,
      const std::string& session_id,
      bool dtls_enabled,
      std::unique_ptr<rtc::RTCCertificateGeneratorInterface> cert_generator,
      const rtc::scoped_refptr<rtc::RTCCertificate>& certificate,
      UniqueRandomIdGenerator* ssrc_generator,
      std::function<void(const rtc::scoped_refptr<rtc::RTCCertificate>&)>
          on_certificate_ready)
      : signaling_thread_(signaling_thread),
        session_desc_factory_(channel_manager,
                              &transport_desc_factory_,
                              ssrc_generator),
        // RFC 4566 suggested a Network Time Protocol (NTP) format timestamp
        // as the session id and session version. To simplify, it should be fine
        // to just use a random number as session id and start version from
        // |kInitSessionVersion|.
        session_version_(kInitSessionVersion),
        cert_generator_(dtls_enabled ? std::move(cert_generator) : nullptr),
        sdp_info_(sdp_info),
        session_id_(session_id),
        certificate_request_state_(CERTIFICATE_NOT_NEEDED),
        on_certificate_ready_(on_certificate_ready) {
  
   ...
  
    if (certificate) {
     ...
    } else {
      // Generate certificate.
      certificate_request_state_ = CERTIFICATE_WAITING;
  
      rtc::scoped_refptr<WebRtcCertificateGeneratorCallback> callback(
          new rtc::RefCountedObject<WebRtcCertificateGeneratorCallback>());
      callback->SignalRequestFailed.connect(
          this, &WebRtcSessionDescriptionFactory::OnCertificateRequestFailed);
      callback->SignalCertificateReady.connect(
          this, &WebRtcSessionDescriptionFactory::SetCertificate);
  
      rtc::KeyParams key_params = rtc::KeyParams();
      RTC_LOG(LS_VERBOSE)
          << "DTLS-SRTP enabled; sending DTLS identity request (key type: "
          << key_params.type() << ").";
  
      // Request certificate. This happens asynchronously, so that the caller gets
      // a chance to connect to |SignalCertificateReady|.
      cert_generator_->GenerateCertificateAsync(key_params, absl::nullopt,
                                                callback);
    }
  }
  ```
  
  ```cpp
  void WebRtcSessionDescriptionFactory::OnCertificateRequestFailed() {
    RTC_DCHECK(signaling_thread_->IsCurrent());
  
    RTC_LOG(LS_ERROR) << "Asynchronous certificate generation request failed.";
    certificate_request_state_ = CERTIFICATE_FAILED;
  
    FailPendingRequests(kFailedDueToIdentityFailed);
  }
  ```

​    

## 6. WebRtcSessionDescriptionFactory.InternalCreateOffer

pc\webrtc_session_description_factory.cc

创建SessionDescription，以及JsepSessionDescription。SessionDescription 是JsepSessionDescription的属性之一。

```cpp
void WebRtcSessionDescriptionFactory::InternalCreateOffer(
    CreateSessionDescriptionRequest request) {
  ...
    // 2. 创建SessionDescription对象
  // 2.1 使用MediaSessionDescriptionFactory::CreateOffer来创建 
  // request.options 就是前面章节创建的 MediaSessionOptions
  //  const SdpStateProvider* sdp_info_; 就是SdpOfferAnswerHandler
  std::unique_ptr<cricket::SessionDescription> desc =
      session_desc_factory_.CreateOffer(
          request.options, sdp_info_->local_description()
                               ? sdp_info_->local_description()->description()
                               : nullptr);
  ...


  // 3. 构造最终的Offer SDP对象JsepSessionDescription
  // 3.1 每次创建Offer，会话版本session_version_需要自增1。必须确保
  //     session_version_ 自增后比之前大，即不发生数据溢出，session_version_
  //     被定义为uint64_t【章节3.6.2】
  RTC_DCHECK(session_version_ + 1 > session_version_);
  auto offer = std::make_unique<JsepSessionDescription>(
      SdpType::kOffer, std::move(desc), session_id_,
      rtc::ToString(session_version_++));
  // 3.2 根据每个mline是否需要重启ICE过程，若不需要重启，那么必须得拷贝
  //     之前得ICE过程收集的候选项到新的Offer中
  if (sdp_info_->local_description()) {
    for (const cricket::MediaDescriptionOptions& options :
         request.options.media_description_options) {
      if (!options.transport_options.ice_restart) {
        CopyCandidatesFromSessionDescription(sdp_info_->local_description(),
                                             options.mid, offer.get());
      }
    }
  }
  // 3.3 创建成功的最终处理【章节3.6.3】offer 就是JsepSessionDescription
  // 通过observer，创建sdp成功，同时把sdp 给下一步使用
  PostCreateSessionDescriptionSucceeded(request.observer, std::move(offer));
}
```

- MediaSessionDescriptionFactory::CreateOffer来创建SessionDescription对象，它是JsepSessionDescription的一部分。
- 接着创建了最终的Offer SDP对象，JsepSessionDescription
- 通过PostCreateSessionDescriptionSucceeded方法触发了用户侧回调 以及 操作链进入下一步操作。



### 6.0 关系图

![jsep-session-description]({{ site.url }}{{ site.baseurl }}/images/create-offer-1.assets/jsep-session-description.png)



### !!! 6.1 MediaSessionDescriptionFactory.CreateOffer

根据MediaSessionOptions创建SessionDescription,为每个m-line创建对应的新的ContentInfo结构体。参考【章节7】

### !!! 6.2 JsepSessionDescription.JsepSessionDescription

api\jsep_session_description.h

```cpp
JsepSessionDescription::JsepSessionDescription(
    SdpType type,
    std::unique_ptr<cricket::SessionDescription> description,
    absl::string_view session_id,
    absl::string_view session_version)
    : description_(std::move(description)),
      session_id_(session_id),
      session_version_(session_version),
      type_(type) {
  RTC_DCHECK(description_);
  candidate_collection_.resize(number_of_mediasections());
}
```



### 6.3 WebRtcSessionDescriptionFactory.PostCreateSessionDescriptionSucceeded——SDP 创建成功，发送消息

```cpp
void WebRtcSessionDescriptionFactory::PostCreateSessionDescriptionSucceeded(
    CreateSessionDescriptionObserver* observer,
    std::unique_ptr<SessionDescriptionInterface> description) {
  CreateSessionDescriptionMsg* msg =
      new CreateSessionDescriptionMsg(observer, RTCError::OK());
  msg->description = std::move(description);
  // this 就是MessageHandler， 就是WebRtcSessionDescriptionFactory， 实现了MessageHandler
  signaling_thread_->Post(RTC_FROM_HERE, this,
                          MSG_CREATE_SESSIONDESCRIPTION_SUCCESS, msg);
}
```

- offer 是 JsepSessionDescription对象， 通过消息队列TaskQueue，发送MSG_CREATE_SESSIONDESCRIPTION_SUCCESS。
  在`WebRtcSessionDescriptionFactory::OnMessage` 接收到消息。

  rtc_base/message_handler.h

  ```cpp
  class RTC_EXPORT MessageHandler {
   public:
    virtual ~MessageHandler() {}
    virtual void OnMessage(Message* msg) = 0;
  };
  ```

  

### 6.4 WebRtcSessionDescriptionFactory.OnMessage

```cpp
void WebRtcSessionDescriptionFactory::OnMessage(rtc::Message* msg) {
  switch (msg->message_id) {
    case MSG_CREATE_SESSIONDESCRIPTION_SUCCESS: {
      CreateSessionDescriptionMsg* param =
          static_cast<CreateSessionDescriptionMsg*>(msg->pdata);
      param->observer->OnSuccess(param->description.release());
      delete param;
      break;
    }
    case MSG_CREATE_SESSIONDESCRIPTION_FAILED: {
      CreateSessionDescriptionMsg* param =
          static_cast<CreateSessionDescriptionMsg*>(msg->pdata);
      param->observer->OnFailure(std::move(param->error));
      delete param;
      break;
    }
    case MSG_USE_CONSTRUCTOR_CERTIFICATE: {
      rtc::ScopedRefMessageData<rtc::RTCCertificate>* param =
          static_cast<rtc::ScopedRefMessageData<rtc::RTCCertificate>*>(
              msg->pdata);
      RTC_LOG(LS_INFO) << "Using certificate supplied to the constructor.";
      SetCertificate(param->data());
      delete param;
      break;
    }
    default:
      RTC_NOTREACHED();
      break;
  }
}
```



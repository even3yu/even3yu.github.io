---
layout: post
title: webrtc create offer(1)
date: 2023-08-23 14:56:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

## 0. 前言

- CreateOffer是一个不断搜集信息、然后形成offer、通告结果的过程。

- 搜集信息：实际上是形成结构体MediaSessionOptions，并不断填充该结构体的过程。这些信息来源于PeerConnection::CreateOffer的入参RTCOfferAnswerOptions、当前的已被应用的Offer(主要是ContentInfos)、PeerConnection.transceivers_成员。主要集中在PeerConnection::GetOptionsForOffer实现填充过程。

- 形成Offer：实际上是根据搜集的信息MediaSessionOptions，经过一系列的函数调用来构建Offer对象的过程。Offer SDP实质上是JsepSessionDescription对象，不过该对象中重要的成员SessionDescription承载了绝大多数信息。

- 通告结果：不论Offer创建成功，还是失败，最终需要做两件事。一件是通告用户侧Offer创建成功还是失败；一件是触发操作链的下一个操作。这个是通过CreateSessionDescriptionObserverOperationWrapper对象封装创建Offer回调接口、封装操作链操作完成回调，并在CreateOffer过程中一直往下传递，直到创建失败或者成功的地方被触发，来实现的。

- 此外：不论是搜集信息，还是形成Offer都需要参考当前已被应用的Offer中的信息，以便复用部分信息，并使得两次Offer中同样的mLine处于同样的位置。

![createoffer-1]({{ site.url }}{{ site.baseurl }}/images/create-offer.assets/createoffer-1.png)SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer()会遍历PC中所有的RtpTransceiver(RtpTransceiver是在addTrack的时候生成的)，**为每个RtpTransceiver创建一个媒体描述信息对象MediaDescriptionOptions**，在最终的生成的SDP对象中，**一个MediaDescriptionOptions就是一个m-line**。 根据由于之前的分析，一个Track对应一个RtpTransceiver，实质上在SDP中一个track就会对应到一个m-line。上述遍历形成所有媒体描述信息MediaDescriptionOptions会存入到MediaSessionOptions对象中，该对象在后续过程中一路传递，最终**在MediaSessionDescriptionFactory::CreateOffer()方法中被用来完成SDP创建**。

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



## 3. PeerConnection.CreateOffer

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



### 3.1 SdpOfferAnswerHandler.CreateOffer

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



### 3.2 SdpOfferAnswerHandler.DoCreateOffer

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
  
  ![media_session_option_value]({{ site.url }}{{ site.baseurl }}/images/create-offer.assets/media_session_option_value.png)



#### 3.2.1 SdpOfferAnswerHandler.HandleLegacyOfferOptions

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

#### 3.2.2 SdpOfferAnswerHandler.GetOptionsForOffer

通过RTCOfferAnswerOptions 创建 MediaSessionOptions。
MediaSessionOptions 除了一些公共部的一些属性， 还存放了每个 m-line 特有的属性，多个mline以数组形式存放。
参考【章节4】。

#### 3.2.3 WebRtcSessionDescriptionFactory.CreateOffer

根据MediaSessionOptions 创建 JsepSessionDescription， 当然JsepSessionDescription 主要的属性 SessionDescription。
参考【章节5】。

## 4. !!! SdpOfferAnswerHandler.GetOptionsForOffer

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

![media_session_option]({{ site.url }}{{ site.baseurl }}/images/create-offer.assets/media_session_option.png)

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



### 4.3 SdpOfferAnswerHandler.ExtractSharedMediaSessionOptions

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


### 4.4 !!! SdpOfferAnswerHandler.GetOptionsForUnifiedPlanOffer

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



#### \mid_generator_.GenerateString()

这里生成了mid。



#### SessionDescription

pc\session_description.h
SessionDescription 主要是管理了ContentInfo， 存了了一个向量，ContentInfos来表示。

#### ContentInfo

pc\session_description.h
ContentInfo 主要管理 mline对应的相关信息， 存放在MediaContentDescription；

#### 关系图

![jsep-session-description]({{ site.url }}{{ site.baseurl }}/images/create-offer.assets/jsep-session-description.png)

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



### 4.4 GetMediaDescriptionOptionsForTransceiver

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



## 5. WebRtcSessionDescriptionFactory.CreateOffer

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

![jsep-session-description]({{ site.url }}{{ site.baseurl }}/images/create-offer.assets/jsep-session-description.png)



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



## 7. ??? MediaSessionDescriptionFactory.CreateOffer

pc/media_session.cc

根据MediaSessionOptions创建SessionDescription,为每个m-line创建对应的新的ContentInfo结构体

```cpp
// - session_options 是SdpOfferAnswerHandler.GetOptionsForOffer准备好的
// - current_description 是当前的会话描述内容，如果是第一次 CreateOffer ，这个值为 nullptr，
// 如果中途因为某些原因需要再次协商会话描述信息，这个值就是有意义的。
 std::unique_ptr<SessionDescription> MediaSessionDescriptionFactory::CreateOffer(
    const MediaSessionOptions& session_options,
    const SessionDescription* current_description) const {
   // 1. 从已被应用的offer 和 当前MediaSessionOptions中抽取一些信息，
  //    以便后续为每个m-line创建对应的新的ContentInfo结构体
  // 1.1 当前已被应用的offer sdp中的mlinege个数必须比    
  //    MediaSessionOptions.media_description_options要少或者等于。
  //    实际上回顾GetOptionsForUnifiedPlanOffer方法搜集MediaSessionOptions
  //    中的media_description_options过程，就保证了这点。

   ...

  // 1.2 获取ice的凭证：ice credential即是ice parameter，包含
  //    ufrag，pwd，renomination三个参数
  IceCredentialsIterator ice_credentials(
      session_options.pooled_ice_credentials);

   // 1.3 从已被应用的当前offer中(就是上次setLocalDescription的offer)，获取活动的ContentInfo
  //    判断是否是活动的ContentInfo，必须是ContentInfo.rejected=fasle
  //    并且对应的session_options.media_options的stopped=false
   // 第一次进来，current_description 是null，所以这个流程不走
  std::vector<const ContentInfo*> current_active_contents;
  if (current_description) {
    current_active_contents =
        GetActiveContents(*current_description, session_options);
  }

  // 1.4 从活动的ContentInfo获取m-line的StreamParams，
  //    注意一个m-line对应一个ContentInfo，一个ContentInfo可能含有多个StreamParams
  //  typedef std::vector<StreamParams> StreamParamsVec;
  StreamParamsVec current_streams =
      GetCurrentStreamParams(current_active_contents);

  // 1.5 从活动的ContentInfo中获取媒体编码器信息
  // 1.5.1 获取编码器信息【章节3.6.1.3】
  AudioCodecs offer_audio_codecs;
  VideoCodecs offer_video_codecs;
  RtpDataCodecs offer_rtp_data_codecs;
  GetCodecsForOffer(
      current_active_contents, &offer_audio_codecs, &offer_video_codecs,
      session_options.data_channel_type == DataChannelType::DCT_SCTP
          ? nullptr
          : &offer_rtp_data_codecs);
  // 1.5.2 根据session_options的信息对编码器进行过滤处理
  if (!session_options.vad_enabled) {
    // If application doesn't want CN codecs in offer.
    StripCNCodecs(&offer_audio_codecs);
  }
  // 1.6 获取Rtp扩展头信息【章节3.6.1.】
  AudioVideoRtpHeaderExtensions extensions_with_ids =
      GetOfferedRtpHeaderExtensionsWithIds(
          current_active_contents, session_options.offer_extmap_allow_mixed,
          session_options.media_description_options);

  // --------------------------------
  // --------------------------------
  // --------------------------------
  // 2. 为每个mline创建对应的ContentInfo，添加到SessionDescription
  // 2.1 创建SessionDescription对象
  auto offer = std::make_unique<SessionDescription>();

  // 2.2 迭代MediaSessionOptions中的每个MediaDescriptionOptions，创建Conteninfo，并添加到
  //     新建SessionDescription对象
  // 2.2.1 循环迭代
  // Iterate through the media description options, matching with existing media
  // descriptions in |current_description|.
  size_t msection_index = 0;
  for (const MediaDescriptionOptions& media_description_options :
       session_options.media_description_options) {
    // 2.2.2 获取当前ContentInfo
    //       要么存在于当前的offer sdp中，则从当前的offer sdp中获取即可
    //       要么是新加入的媒体，还没有ContentInfo，因此为空
    const ContentInfo* current_content = nullptr;
    if (current_description &&
        msection_index < current_description->contents().size()) {
      // 从上次的offer中获取到current_content
      current_content = &current_description->contents()[msection_index];
      // Media type must match unless this media section is being recycled.
      RTC_DCHECK(current_content->name != media_description_options.mid ||
                 IsMediaContentOfType(current_content,
                                      media_description_options.type));
    }
    // 2.2.3 根据媒体类别，分别调用不同的方法创建ContentInfo，并添加到SessionDescription
    switch (media_description_options.type) {
      case MEDIA_TYPE_AUDIO:
        if (!AddAudioContentForOffer(media_description_options, session_options,
                                     current_content, current_description,
                                     extensions_with_ids.audio,
                                     offer_audio_codecs, &current_streams,
                                     offer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_VIDEO:
        // 【章节3.6.1.6】
        if (!AddVideoContentForOffer(media_description_options, session_options,
                                     current_content, current_description,
                                     extensions_with_ids.video,
                                     offer_video_codecs, &current_streams,
                                     offer.get(), &ice_credentials)) {
          return nullptr;
        }
        break;
      case MEDIA_TYPE_DATA:
        ...
        break;
      case MEDIA_TYPE_UNSUPPORTED:
        ...
        break;
      default:
        RTC_NOTREACHED();
    }
    ++msection_index;
  }

    // 3. 处理Bundle，如果session_options.bundle_enabled为真（默认为真），则需要将所有的
  //    ContentInfo全都进入一个ContentGroup，同一个ContentGroup是复用同一个底层传输的
  // Bundle the contents together, if we've been asked to do so, and update any
  // parameters that need to be tweaked for BUNDLE.
  if (session_options.bundle_enabled) {
    // 3.1 创建ContentGroup，并将每个有效的(活动的)ContentInfo添加到ContentGroup
    ContentGroup offer_bundle(GROUP_TYPE_BUNDLE);
    for (const ContentInfo& content : offer->contents()) {
      if (content.rejected) {
        continue;
      }
      // TODO(deadbeef): There are conditions that make bundling two media
      // descriptions together illegal. For example, they use the same payload
      // type to represent different codecs, or same IDs for different header
      // extensions. We need to detect this and not try to bundle those media
      // descriptions together.
      offer_bundle.AddContentName(content.name);
    }
   // 3.2 添加bundle到offer并更新bundle的传输通道信息、加密参数信息
     if (!offer_bundle.content_names().empty()) {
      offer->AddGroup(offer_bundle);
      if (!UpdateTransportInfoForBundle(offer_bundle, offer.get())) {
        RTC_LOG(LS_ERROR)
            << "CreateOffer failed to UpdateTransportInfoForBundle.";
        return nullptr;
      }
      if (!UpdateCryptoParamsForBundle(offer_bundle, offer.get())) {
        RTC_LOG(LS_ERROR)
            << "CreateOffer failed to UpdateCryptoParamsForBundle.";
        return nullptr;
      }
    }
  }
 // 4. 设置一些其他信息
  // 4.1 设置msid信息
  // The following determines how to signal MSIDs to ensure compatibility with
  // older endpoints (in particular, older Plan B endpoints).
  if (is_unified_plan_) {
    // Be conservative and signal using both a=msid and a=ssrc lines. Unified
    // Plan answerers will look at a=msid and Plan B answerers will look at the
    // a=ssrc MSID line.
    offer->set_msid_signaling(cricket::kMsidSignalingMediaSection |
                              cricket::kMsidSignalingSsrcAttribute);
  } else {
    // Plan B always signals MSID using a=ssrc lines.
    offer->set_msid_signaling(cricket::kMsidSignalingSsrcAttribute);
  }

  // 4.2 
  offer->set_extmap_allow_mixed(session_options.offer_extmap_allow_mixed);

  return offer;
}
```

- 从已被应用的offer 和 当前MediaSessionOptions中抽取一些信息，以便后续为每个m-line创建对应的新的ContentInfo结构体。这些信息包括：IceParameters（用于ICE过程的ufrag、pwd等信息）、StreamParams（每个媒体源的参数，包括id(即track id)、ssrcs、ssrc_groups、cname等）、音视频数据的编码器信息（编码器的id、name、时钟clockrate、编码参数表params、反馈参数feedback_params）、Rtp扩展头信息（uri、id、encrypt）等。

- 创建SessionDescription，利用上面步骤提供的信息 && MediaSessionOptions提供的信息为每个mline创建对应的ContentInfo，添加到SessionDescription。

- 处理所有ContentInfo的bundle关系，Bundle the contents together。创建一个BUNDLE，将所有ContentInfo加入bundle并更新bundle的底层传输信息、加密信息。

- 更新offer的其他信息：msid、extmap_allow_mixed等（行文至此，目前还不清楚这两个起什么作用，后续清楚了，再来更新）。

### ---这部分是根据上次协商的offer 提取的相关信息----

### 7.1 MediaSessionDescriptionFactory.GetActiveContents

pc/media_session.cc

```cpp
static std::vector<const ContentInfo*> GetActiveContents(
    const SessionDescription& description,
    const MediaSessionOptions& session_options) {
  std::vector<const ContentInfo*> active_contents;
  for (size_t i = 0; i < description.contents().size(); ++i) {
    RTC_DCHECK_LT(i, session_options.media_description_options.size());
    const ContentInfo& content = description.contents()[i];
    const MediaDescriptionOptions& media_options =
        session_options.media_description_options[i];
    // 正常使用
    if (!content.rejected && !media_options.stopped &&
        content.name == media_options.mid) {
      active_contents.push_back(&content);
    }
  }
  return active_contents;
}
```

从上次 协商的sdp信息中，和当前的MediaSessionOptions，获取到在正常使用的m-line。



### 7.2 MediaSessionDescriptionFactory.GetCurrentStreamParams

pc/media_session.cc

```cpp
// Finds all StreamParams of all media types and attach them to stream_params.
static StreamParamsVec GetCurrentStreamParams(
    const std::vector<const ContentInfo*>& active_local_contents) {
  StreamParamsVec stream_params;
  for (const ContentInfo* content : active_local_contents) {
    // media_description() 就说是返回 MediaContentDescription
    for (const StreamParams& params : content->media_description()->streams()) {
      stream_params.push_back(params);
    }
  }
  return stream_params;
}
```

从上一步中得到的active_local_contents， 来得到StreamParamsVec。



### 7.3 MediaSessionDescriptionFactory.GetCodecsForOffer

获取音视频数据的所支持的编码

```cpp
 AudioCodecs offer_audio_codecs;
 VideoCodecs offer_video_codecs;
 DataCodecs offer_data_codecs;
```

```cpp
void MediaSessionDescriptionFactory::GetCodecsForOffer(
    const SessionDescription* current_description,
    AudioCodecs* audio_codecs,
    VideoCodecs* video_codecs,
    DataCodecs* data_codecs) const {
  UsedPayloadTypes used_pltypes;
  audio_codecs->clear();
  video_codecs->clear();
  data_codecs->clear();

  // First - get all codecs from the current description if the media type
  // is used. Add them to |used_pltypes| so the payload type is not reused if a
  // new media type is added.
  if (current_description) {
    MergeCodecsFromDescription(current_description, audio_codecs, video_codecs,
                               data_codecs, &used_pltypes);
  }

  // Add our codecs that are not in |current_description|.
  MergeCodecs<AudioCodec>(all_audio_codecs_, audio_codecs, &used_pltypes);
  MergeCodecs<VideoCodec>(video_codecs_, video_codecs, &used_pltypes);
  MergeCodecs<DataCodec>(data_codecs_, data_codecs, &used_pltypes);
}
```

- 执行 clear() 的动作，避免指针指向了无效数据；
- 如果 current_description 不为空，也就是不是第一次执行 CreateOffer ，那么执行 MergeCodecsFromDescription ，将current_description 中记录的编码信息存入 offer_xxx_codecs；
- 执行 MergeCodecs，将本地支持的编码格式存入 offer_xxx_codecs。



### 7.4 MediaSessionDescriptionFactory.GetOfferedRtpHeaderExtensionsWithIds

### -------



### 7.5 SessionDescription::SessionDescription

pc/session_description.h


### 7.6 !!! MediaSessionDescriptionFactory.AddVideoContentForOffer

创建了VideoContentDescription，存入了ContentInfo， 并加入到SessionDescription

```cpp
// TODO(kron): This function is very similar to AddAudioContentForOffer.
// Refactor to reuse shared code.
bool MediaSessionDescriptionFactory::AddVideoContentForOffer(
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const ContentInfo* current_content,
    const SessionDescription* current_description,
    const RtpHeaderExtensions& video_rtp_extensions,
    const VideoCodecs& video_codecs,
    StreamParamsVec* current_streams,
    SessionDescription* desc,
    IceCredentialsIterator* ice_credentials) const {
  // Filter video_codecs (which includes all codecs, with correctly remapped
  // payload types) based on transceiver direction.
  // 根据RtpTransceiverDirection， 来获取codec
  const VideoCodecs& supported_video_codecs =
      GetVideoCodecsForOffer(media_description_options.direction);

  VideoCodecs filtered_codecs;

  
  if (!media_description_options.codec_preferences.empty()) {
    // Add the codecs from the current transceiver's codec preferences.
    // They override any existing codecs from previous negotiations.
    filtered_codecs = MatchCodecPreference(
        media_description_options.codec_preferences, supported_video_codecs);
  } else {
    // Add the codecs from current content if it exists and is not rejected nor
    // recycled.
    if (current_content && !current_content->rejected &&
        current_content->name == media_description_options.mid) {
      RTC_CHECK(IsMediaContentOfType(current_content, MEDIA_TYPE_VIDEO));
      const VideoContentDescription* vcd =
          current_content->media_description()->as_video();
      for (const VideoCodec& codec : vcd->codecs()) {
        if (FindMatchingCodec<VideoCodec>(vcd->codecs(), video_codecs, codec,
                                          nullptr)) {
          filtered_codecs.push_back(codec);
        }
      }
    }
    // Add other supported video codecs.
    VideoCodec found_codec;
    for (const VideoCodec& codec : supported_video_codecs) {
      if (FindMatchingCodec<VideoCodec>(supported_video_codecs, video_codecs,
                                        codec, &found_codec) &&
          !FindMatchingCodec<VideoCodec>(supported_video_codecs,
                                         filtered_codecs, codec, nullptr)) {
        // Use the |found_codec| from |video_codecs| because it has the
        // correctly mapped payload type.
        filtered_codecs.push_back(found_codec);
      }
    }
  }

  if (session_options.raw_packetization_for_video) {
    for (VideoCodec& codec : filtered_codecs) {
      if (codec.GetCodecType() == VideoCodec::CODEC_VIDEO) {
        codec.packetization = kPacketizationParamRaw;
      }
    }
  }

  cricket::SecurePolicy sdes_policy =
      IsDtlsActive(current_content, current_description) ? cricket::SEC_DISABLED
                                                      : secure();
  // 创建VideoContentDescription，
  auto video = std::make_unique<VideoContentDescription>();
  // 加密套件
  std::vector<std::string> crypto_suites;
  GetSupportedVideoSdesCryptoSuiteNames(session_options.crypto_options,
                                        &crypto_suites);
  // 【章节3.6.1.7】
  if (!CreateMediaContentOffer(media_description_options, session_options,
                               filtered_codecs, sdes_policy,
                               GetCryptos(current_content), crypto_suites,
                               video_rtp_extensions, ssrc_generator_,
                               current_streams, video.get())) {
    return false;
  }

  video->set_bandwidth(kAutoBandwidth);

  bool secure_transport = (transport_desc_factory_->secure() != SEC_DISABLED);
  SetMediaProtocol(secure_transport, video.get());

  video->set_direction(media_description_options.direction);

  // SessionDescription* desc,向SessionDescription 中添加Content
 // 添加ContentInfo和VideoContentDescription【章节3.6.1.8】
  desc->AddContent(media_description_options.mid, MediaProtocolType::kRtp,
                   media_description_options.stopped, std::move(video));
  if (!AddTransportOffer(media_description_options.mid,
                         media_description_options.transport_options,
                         current_description, desc, ice_credentials)) {
    return false;
  }

  return true;
}
```

- 根据RtpTransceiverDirection， 来获取codec
- 过滤 codec
- 创建VideoContentDescription
- 安全套件 `session_options.crypto_options`
-  添加codec
- 添加stream
- 

#### !!! ContentInfo::rejected

pc/session_description.h
这个值是 根据`media_description_options.stopped` 来确定的。



#### MediaContentDescription

pc/session_description.h

#### MediaContentDescriptionImpl

pc/session_description.h

#### AudioContentDescription

pc/session_description.h

#### VideoContentDescription

pc/session_description.h

#### MediaSessionDescriptionFactory::GetVideoCodecsForOffer

pc/media_session.cc

```cpp
const VideoCodecs& MediaSessionDescriptionFactory::GetVideoCodecsForOffer(
    const RtpTransceiverDirection& direction) const {
  switch (direction) {
    // If stream is inactive - generate list as if sendrecv.
    case RtpTransceiverDirection::kSendRecv:
    case RtpTransceiverDirection::kStopped:
    case RtpTransceiverDirection::kInactive:
      return video_sendrecv_codecs_;
    case RtpTransceiverDirection::kSendOnly:
      return video_send_codecs_;
    case RtpTransceiverDirection::kRecvOnly:
      return video_recv_codecs_;
  }
  RTC_CHECK_NOTREACHED();
}
```



```cpp
MediaSessionDescriptionFactory::MediaSessionDescriptionFactory(
    ChannelManager* channel_manager,
    const TransportDescriptionFactory* transport_desc_factory,
    rtc::UniqueRandomIdGenerator* ssrc_generator)
    : MediaSessionDescriptionFactory(transport_desc_factory, ssrc_generator) {
  channel_manager->GetSupportedAudioSendCodecs(&audio_send_codecs_);
  channel_manager->GetSupportedAudioReceiveCodecs(&audio_recv_codecs_);
  channel_manager->GetSupportedVideoSendCodecs(&video_send_codecs_);
  channel_manager->GetSupportedVideoReceiveCodecs(&video_recv_codecs_);
  channel_manager->GetSupportedDataCodecs(&rtp_data_codecs_);
  ComputeAudioCodecsIntersectionAndUnion();
  ComputeVideoCodecsIntersectionAndUnion();
}
```



#### GetSupportedVideoSdesCryptoSuiteNames



#### CreateMediaContentOffer



#### SetMediaProtocol



#### AddTransportOffer



### 7.7 MediaSessionDescriptionFactory.CreateMediaContentOffer

创建VideoContentDescription

```cpp
template <class C>
static bool CreateMediaContentOffer(
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const std::vector<C>& codecs,
    const SecurePolicy& secure_policy,
    const CryptoParamsVec* current_cryptos,
    const std::vector<std::string>& crypto_suites,
    const RtpHeaderExtensions& rtp_extensions,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescriptionImpl<C>* offer) {
  offer->AddCodecs(codecs);
  // 参考【章节3.6.1.9】
  if (!AddStreamParams(media_description_options.sender_options,
                       session_options.rtcp_cname, ssrc_generator,
                       current_streams, offer)) {
    return false;
  }

  return CreateContentOffer(media_description_options, session_options,
                            secure_policy, current_cryptos, crypto_suites,
                            rtp_extensions, ssrc_generator, current_streams,
                            offer);
}
```



#### CreateContentOffer

```cpp
// Create a media content to be offered for the given |sender_options|,
// according to the given options.rtcp_mux, session_options.is_muc, codecs,
// secure_transport, crypto, and current_streams. If we don't currently have
// crypto (in current_cryptos) and it is enabled (in secure_policy), crypto is
// created (according to crypto_suites). The created content is added to the
// offer.
static bool CreateContentOffer(
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const SecurePolicy& secure_policy,
    const CryptoParamsVec* current_cryptos,
    const std::vector<std::string>& crypto_suites,
    const RtpHeaderExtensions& rtp_extensions,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescription* offer) {
  offer->set_rtcp_mux(session_options.rtcp_mux_enabled);
  if (offer->type() == cricket::MEDIA_TYPE_VIDEO) {
    offer->set_rtcp_reduced_size(true);
  }

  // Build the vector of header extensions with directions for this
  // media_description's options.
  RtpHeaderExtensions extensions;
  for (auto extension_with_id : rtp_extensions) {
    for (const auto& extension : media_description_options.header_extensions) {
      if (extension_with_id.uri == extension.uri) {
        // TODO(crbug.com/1051821): Configure the extension direction from
        // the information in the media_description_options extension
        // capability.
        extensions.push_back(extension_with_id);
      }
    }
  }
  offer->set_rtp_header_extensions(extensions);

  AddSimulcastToMediaDescription(media_description_options, offer);

  if (secure_policy != SEC_DISABLED) {
    if (current_cryptos) {
      AddMediaCryptos(*current_cryptos, offer);
    }
    if (offer->cryptos().empty()) {
      if (!CreateMediaCryptos(crypto_suites, offer)) {
        return false;
      }
    }
  }

  if (secure_policy == SEC_REQUIRED && offer->cryptos().empty()) {
    return false;
  }
  return true;
}

```

#### SessionDescription.AddContent

创建ContentInfo，并加入SessionDescription 的ContentInfos

```cpp
typedef std::vector<ContentInfo> ContentInfos;
ContentInfos contents_;

void SessionDescription::AddContent(
    const std::string& name,
    MediaProtocolType type,
    bool rejected,
    bool bundle_only,
    std::unique_ptr<MediaContentDescription> description) {
  ContentInfo content(type);
  content.name = name;
  content.rejected = rejected;
  content.bundle_only = bundle_only;
  content.set_media_description(std::move(description));
  AddContent(std::move(content));
}

void SessionDescription::AddContent(ContentInfo&& content) {
  if (extmap_allow_mixed()) {
    // Mixed support on session level overrides setting on media level.
    content.media_description()->set_extmap_allow_mixed_enum(
        MediaContentDescription::kSession);
  }
  contents_.push_back(std::move(content));
}
```

1. MediaContentDescription
   
   ```cpp
   // Describes a session description media section. There are subclasses for each
   // media type (audio, video, data) that will have additional information.
   class MediaContentDescription {
   public:
   MediaContentDescription() = default;
   virtual ~MediaContentDescription() = default;
   ...
   }
   ```

2. AudioContentDescription
   
   ```cpp
   class AudioContentDescription : public MediaContentDescriptionImpl<AudioCodec> {
   ...
   }
   ```

3. VideoContentDescription
   
   ```cpp
   class VideoContentDescription : public MediaContentDescriptionImpl<VideoCodec> {
   ...
   }
   ```

#### MediaSessionDescriptionFactory.AddStreamParams

创建StreamParams

在章节【3.6.1.9】CreateMediaContentOffer中 调用

```cpp
template <class C>
static bool AddStreamParams(
    const std::vector<SenderOptions>& sender_options,
    const std::string& rtcp_cname,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescriptionImpl<C>* content_description) {
  // SCTP streams are not negotiated using SDP/ContentDescriptions.
  if (IsSctpProtocol(content_description->protocol())) {
    return true;
  }

  const bool include_rtx_streams =
      ContainsRtxCodec(content_description->codecs());

  const bool include_flexfec_stream =
      ContainsFlexfecCodec(content_description->codecs());

  for (const SenderOptions& sender : sender_options) {
    // groupid is empty for StreamParams generated using
    // MediaSessionDescriptionFactory.
    StreamParams* param =
        GetStreamByIds(*current_streams, "" /*group_id*/, sender.track_id);
    if (!param) {
      // This is a new sender.
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

      content_description->AddStream(stream_param);

      // Store the new StreamParams in current_streams.
      // This is necessary so that we can use the CNAME for other media types.
      current_streams->push_back(stream_param);
    } else {
      // Use existing generated SSRCs/groups, but update the sync_label if
      // necessary. This may be needed if a MediaStreamTrack was moved from one
      // MediaStream to another.
      param->set_stream_ids(sender.stream_ids);
      content_description->AddStream(*param);
    }
  }
  return true;
}
```

#### MediaSessionDescriptionFactory.CreateStreamParamsForNewSenderWithSsrcs

![ssrc1]({{ site.url }}{{ site.baseurl }}/images/create-offer.assets/ssrc1.png)

```cpp
static StreamParams CreateStreamParamsForNewSenderWithSsrcs(
    const SenderOptions& sender,
    const std::string& rtcp_cname,
    bool include_rtx_streams,
    bool include_flexfec_stream,
    UniqueRandomIdGenerator* ssrc_generator) {
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

#### StreamParams::GenerateSsrcs——生成ssrc

```cpp
void StreamParams::GenerateSsrcs(int num_layers,
                                 bool generate_fid,
                                 bool generate_fec_fr,
                                 rtc::UniqueRandomIdGenerator* ssrc_generator) {
  RTC_DCHECK_GE(num_layers, 0);
  RTC_DCHECK(ssrc_generator);
  std::vector<uint32_t> primary_ssrcs;
  for (int i = 0; i < num_layers; ++i) {
    uint32_t ssrc = ssrc_generator->GenerateId();
    primary_ssrcs.push_back(ssrc);
    add_ssrc(ssrc);
  }

  if (num_layers > 1) {
    SsrcGroup simulcast(kSimSsrcGroupSemantics, primary_ssrcs);
    ssrc_groups.push_back(simulcast);
  }

  if (generate_fid) {
    for (uint32_t ssrc : primary_ssrcs) {
      AddFidSsrc(ssrc, ssrc_generator->GenerateId());
    }
  }

  if (generate_fec_fr) {
    for (uint32_t ssrc : primary_ssrcs) {
      AddFecFrSsrc(ssrc, ssrc_generator->GenerateId());
    }
  }
}
```



## 参考

[webrtc 的 CreateOffer 过程分析](https://blog.csdn.net/zhuiyuanqingya/article/details/84314487)

[WebRTC源码分析-呼叫建立过程之五(创建Offer，CreateOffer，上篇)](https://blog.csdn.net/ice_ly000/article/details/105763753)



## QA

1. 这是需要说明下OfferToReceiveAudio，OfferToReceiveVideo？？？

2. raw_packetization_for_video？？？？
   
   bundle_enabled？？？ 作用 audio/video/data 打包一起发送

3. ContentInfos，ContentInfo是什么
   【章节SessionDescription.AddContent】生成了ContentInfo，并加入向量ContentInfos中
   一个mline 对应一个ContentInfo，其中存了MediaContentDescription

4. current_local_description(),local_contents= local_description() 区别???

5. IceParameters（用于ICE过程的ufrag、pwd等信息）、StreamParams（每个媒体源的参数，包括id(即track id)、ssrcs、ssrc_groups、cname等）、音视频数据的编码器信息（编码器的id、name、时钟clockrate、编码参数表params、反馈参数feedback_params）、Rtp扩展头信息（uri、id、encrypt）

6. Ssrc 是什么时候产生的
   参考【章节3.6.1.6】

7. m-line 和MediaSessionDecroption 是什么关系
   m-line 对应 一个MediaSessionDescrptions
   
   一个Track对应一个RtpTransceiver，实质上在SDP中一个track就会对应到一个m-line
   参考【章节3.4.3】SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer()中

8. WebRtcSessionDescriptionFactory 是什么创建的
   在 PeerConnection::Initialize()  时候创建了WebRtcSessionDescriptionFactory
   【参考】3.6.1.2 构造函数的说明

9. MediaSessionDescriptionFactory是什么时候创建的
   是在WebRtcSessionDescriptionFactory 构造函数的时候创建

10. 什么时候创建了sdp
    【章节3.6.1】

11. 本地的codecs 是怎么获取到的
    是在MediaSessionDescriptionFactory.MediaSessionDescriptionFactory构造函数从ChanelManager中获取到的
    可以参考【章节3.6.1.2】

12. Transceiver 什么是设置set_mline_index
    【章节3.4.3】SdpOfferAnswerHandler::GetOptionsForUnifiedPlanOffer()中

13. transceiver setDirect 是做什么用的

14. raw_packetization_for_video 什么作用

15. MediaSessionOptions 关系 MediaDescriptionOptions
    【章节3.4】【章节3.4.1】

16. MediaSessionOptions，MediaDescriptionOptions，SessionDescription，MediaContentDescription，ContentInfo，VideoContentDescription，JsepSessionDescription
    
    MediaContentDescription 是VideoContentDescription 的父类
    
    ContentInfo 包含了VideoContentDescription，mid等
    
    MediaSessionOptions 包含了MediaDescriptionOptions（vector）
    
    SessionDescription 包含了ContentInfo（vector）
    
    JsepSessionDescription 包含了 SessionDescription

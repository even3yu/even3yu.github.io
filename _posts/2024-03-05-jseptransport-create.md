---
layout: post
title: webrtc JsepTransport 创建流程 以及 相关各种Transport
date: 2024-03-05 00:05:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc transport
---


* content
{:toc}

---


## 0. 属性

![jseptransport]({{ site.url }}{{ site.baseurl }}/images/jseptransport-create.assets/jseptransport.png)



### 0.1 !!! jsep_transports_by_name_ && mid_to_transport_

pc/jsep_transport_controller.cc

注意不同的用法。

## 1. Transport关系图和说明

![img]({{ site.url }}{{ site.baseurl }}/images/jseptransport-create.assets/transport.jpg)

1. RtpTransport 继承关系

   | Transport            | 说明                                                         |
   | :------------------- | :----------------------------------------------------------- |
   | RtpTransportInternal | pc/rtp_transport_internal.h；基础类，虚基类；                |
   | RtpTransport         | pc/rtp_transport.h，非加密的 Transport 类，rtp 是不加密的    |
   | SrtpTransport        | pc/srtp_transport.h，RTP 进行 protecting/unprotecting，底层用的是 SrtpSession，而 SrtpSession 是对 libsrtp 的封装 |
   | DtlsSrtpTransport    | pc/dtls_srtp_transport.h，维护了两个 DtlsTransport 对象，分别用作 rtp 和 rtcp 会话的建立。当 DtlsTransport 完成协商后，获取加密套件，最终是通过 OpenSSLStreamAdapter::GetDtlsSrtpCryptoSuite 方法实现的 |


1. DtlsTransport， 是DTLS 会话协商过程的控制逻辑。

2. P2PTransportChannel 是连接底层 UDP socket 层和 Transport 的纽带。是 Candidate-flex Address 收集的入口，是 P2P 穿越的入口，穿越成功后，会建立连接，管理着 Connection 对象。

3. JsepTransport 是一个辅助、包装类，实现 Jsep 接口，同时持有 RtpTransport、SrtpTransport、DtlsSrtpTransport 等。

4. JsepTransportController，遵循 JSEP 规范的，实现 Jsep 接口；负载管理各种类型 Transport 对象的**创建、获取、销毁**；是 Peer Connection 到 transport 层的入口，设置远端 candidate，启动 p2p 连接穿越过程。

5. OpenSSLStreamAdapter 是对 openssl 的包装，主要是实现 dtls 协议的应用

6. DtlsSrtpTransport ， DtlsTransport 不一样，注意注意！！！

7. DtlsTransport继承关系

	 | 类                      |      |      |
   | ----------------------- | ---- | ---- |
   | sigslot::has_slots<>    |      |      |
   | PacketTransportInternal |      |      |
   | DtlsTransportInternal   |      |      |
   | DtlsTransport           |      |      |




### 1.1 各种 transport 类的关系

- PacketTransportInternal, PacketTransportInterface: packet base transport 的基类；
- P2PTransportChannel, IceTransportInternal: 继承自 PacketTransportInternal，负责 ICE 相关的功能，包括：收集本地 candidate、和远端 candidate 做连通性检查、数据传输；
- DtlsTransport, DtlsTransportInternal: 继承自 PacketTransportInternal，负责 DTLS 相关的功能，包括：DTLS 握手、DTLS 加解密；内含一个 IceTransportInternal (P2PTransportChannel)，收发数据通过它实现；
  - 不加密时，写入 DtlsTransport 的数据，直接交给 P2PTransportChannel；
  - 加密时，数据会交给 SSLStreamAdapter -> StreamInterfaceChannel -> P2PTransportChannel，这是为了桥接加解密的流式接口与 DtlsTransport 的包式接口；
- RtpTransport, RtpTransportInternal, SrtpTransportInterface, RtpTransportInterface: 提供了收发 RTP, RTCP 包的接口，其内部包了两个实际收发 RTP 和 RTCP 数据的 PacketTransportInternal (DtlsTransport)；
- DtlsSrtpTransport, SrtpTransport: DTLS-SRTP, SRTP 的实现类，继承自 RtpTransport； _有了 DTLS，为何还要 SRTP？_
- JsepTransport: JsepTransportController 管理 transport 的辅助类，sdp 里每个 m line 都对应于一个数据流（音频、视频、应用数据），每个数据流都需要一个 transport，但可以通过 bundle 技术复用同一个 transport，m line 里的 attribute 描述了 transport 的属性；
  - 根据 transport 的加密属性，构造它时会准备无加密的 RtpTransport，或 SDES 加密的 SrtpTransport，或 DTLS 加密的 DtlsSrtpTransport；这一逻辑在 JsepTransportController::MaybeCreateJsepTransport 函数里；
  - transport 对象将在设置 sdp 时创建，一个 transport 对象将会对应于一个最终的 P2P 网络连接（socket）；

### 1.2 关键类的数量关系

1. 一个 PeerConnection - 一个 JsepTransportController - 一个 JsepTransport（启用了 bundle） - 一个 DtlsSrtpTransport - 一个 DtlsTransport - 一个 P2PTransportChannel。
2. 一个 JsepTransportController - 一个 BasicPortAllocator - 多个 BasicPortAllocatorSession，但一次分配过程只会有一个 session。
3. 一个 BasicPortAllocatorSession - 多个 AllocationSequence。
4. 一个 AllocationSequence - 多个 port。
5. 一个 P2PTransportChannel - 多个 Connection，但最终会选出一个 Connection 使用。

 [参考 参考 参考 ](https://blog.csdn.net/paradox_1_0/article/details/109206079)



## 2. !!!MaybeCreateJsepTransport调用时机

在 **a=group:BUNDLE 0 1** 中，如果是非 **BUNDLE** 模式，那么会针对每一个 **m=** 行，都会创建一个 JsepTransport 对象；如果是 **BUNDLE** 模式，那么会创建一个公用的 JsepTransport。

1. 赋值JsepTransportController.bundle_group_ 。 由JsepTransportController.bundle_group_ 成员可知，实际应用中，SDP中一般只有一个bundle group。绝大多数情况下都是采用bundle形式进行传输，此时，这样可以减少底层Transport的数量，因此，也能减少需要分配的端口数。如下SDP的示例，4个mline都属于一个group，这个group的名字为默认的BUNDLE，其中0 1 2 3为4个mid值。这4个m section所代表的媒体将采用同一个JsepTransport.![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/jseptransport-create.assets/20200517222108730.png)
2. 遍历sdp中所有ContentInfo,也即m Section的表征，m-line，针对每个 ContentInfo,(即每个sdp) 内容创建一个对应的 JsepTransport。那些情况需要创建，哪些不需要创建Transport： **a. 被拒绝的content是无效的，因此，不应该为其创建Transport；** **b. 如果Content属于一个bundle，却又不是该bundle的第一个content，那么我们应该也要忽略该content**，“因为属于一个bundle的content共享一个Transport进行传输，在遍历该bundle的第一个content时会去创建这个共享的Transport。 `MaybeCreateJsepTransport` 这里就创建 JsepTransport。
3. 其实每个 content 就是 SDP 里面的一个 m 媒体段，具体参考博客 https://blog.csdn.net/freeabc/article/details/109784860
4. 创建JsepTransportDescription 对象，并对该对象进行填充 通过SetLocalJsepTransportDescription函数, 并JsepTransportDescription 存到 JsepTransport。



###  调用堆栈

- SetLocalDescrption的 `SdpOfferAnswerHandler::PushdownTransportDescription`， 
  根据`cricket::SessionDescription::contents()`  这里会创建JsepTransport。

- `JsepTransportController::MaybeCreateJsepTransport`
  CreateIceTransport
  CreateDtlsTransport
  CreateUnencryptedRtpTransport
  CreateSdesTransport
  CreateDtlsSrtpTransport

  ----------->  以上全部放到 JsepTransport

  （mid， JsepTransport ）以这样的键值对存放，
  其中，如果Bundle 多个m Line， 那么以第一个mid 为key，公用JsepTransport；
  如果没有Bundle，那么每个mLine 都会生成JsepTransport；



## 3. 创建时机 JsepTransportController::MaybeCreateJsepTransport

pc/jsep_transport_controller.cc

```cpp
RTCError JsepTransportController::MaybeCreateJsepTransport(
    bool local,
    const cricket::ContentInfo& content_info,
    const cricket::SessionDescription& description) {
  RTC_DCHECK(network_thread_->IsCurrent());
  // 如果对应的JsepTransport已经存在则返回就好了
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

  // 1. 重要, 这里产生的 ice 对象，其实就是 DefaultIceTransport 对象。
  //  通过internal() 返回的是 P2PTransportChannel。
  // 创建ice层的传输对象，负责管理Candidates，联通性检测，收发数据包。
  // 注意，使用的是共享智能指针保存的。
  // 
  // 创建IceTransport for rtp
  rtc::scoped_refptr<webrtc::IceTransportInterface> ice =
      CreateIceTransport(content_info.name, /*rtcp=*/false);
  RTC_DCHECK(ice);

  
  // 2. 创建dtls层的传输对象——>提供Dtls握手逻辑，密钥交换。
  // 注意其dtls内置了ice层的传输对象，其层次与DatagramTransport是平行关系
  // 
  // 创建DtlsTransportInternal for rtp
  std::unique_ptr<cricket::DtlsTransportInternal> rtp_dtls_transport =
      CreateDtlsTransport(content_info, ice->internal());

  std::unique_ptr<cricket::DtlsTransportInternal> rtcp_dtls_transport;
  std::unique_ptr<RtpTransport> unencrypted_rtp_transport;
  std::unique_ptr<SrtpTransport> sdes_transport;
  std::unique_ptr<DtlsSrtpTransport> dtls_srtp_transport;
  std::unique_ptr<RtpTransportInternal> datagram_rtp_transport;

  // 4. 创建IceTransport for rtcp
  rtc::scoped_refptr<webrtc::IceTransportInterface> rtcp_ice;
  // 5. 默认是kRtcpMuxPolicyRequire， 这里流程不走, 是在PeerConnectionFactory配置
  // 如果RTCP与RTP不复用，并且媒体是使用RTP协议传输的，则需要创建属于传输RTCP的
  //    ice层的传输对象，以及dtls层的传输对象
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
  // 6. 不加密 transport
  if (config_.disable_encryption) {
    RTC_LOG(LS_INFO)
        << "Creating UnencryptedRtpTransport, becayse encryption is disabled.";
    unencrypted_rtp_transport = CreateUnencryptedRtpTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
  // 7. crypto 不为空，则sdes transport
  // 使用SDES加密，因为sdp中包含了加密参数
  // 创建使用SDES加密的rtp层传输对象——>SrtpTransport
  //  注意：传递了dtls层的传输对象，rtp层传输对象是其上层，但不使用dtls层的传输对象
  //  的握手，可以提高媒体建立链路的速度。
  } else if (!content_desc->cryptos().empty()) {
    sdes_transport = CreateSdesTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
    RTC_LOG(LS_INFO) << "Creating SdesTransport.";
  } else {
    // 8. dtls transport
    // 使用dtls加密
    //     创建使用dtls加密手段的rtp层传输对象——>DtlsSrtpTransport
    //     注意：传递了dtls层的传输对象，rtp层传输对象是其上层，使用dtls层的传输对象提供的
    //  握手和加密，建立安全通信速度慢。
    RTC_LOG(LS_INFO) << "Creating DtlsSrtpTransport.";
    dtls_srtp_transport = CreateDtlsSrtpTransport(
        content_info.name, rtp_dtls_transport.get(), rtcp_dtls_transport.get());
  }

  // 9. sctp
  // 创建SCTP通道用于传输应用数据——>SctpTransport
  //     注意，其底层dtls层传输对象，使用dtls加密传输
  std::unique_ptr<cricket::SctpTransportInternal> sctp_transport;
  if (config_.sctp_factory) {
    sctp_transport =
        config_.sctp_factory->CreateSctpTransport(rtp_dtls_transport.get());
  }

  // 10. 创建JsepTransport
  //     创建JsepTransport对象，来容纳之前创建的各层对象
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

  // 把JsepTransport存放在map中
  //添加到jsep_transports_by_name_
  // 后面的根据名称获取的 jsep_transport 就是这里产生的
  jsep_transports_by_name_[content_info.name] = std::move(jsep_transport);
  // 更新状态，进行ICE
  UpdateAggregateStates_n();
  return RTCError::OK();
}
```

1. 这里产生的 ice 对象，其实就是 DefaultIceTransport 对象。通过internal() 返回的是 P2PTransportChannel。

   创建ice层的传输对象，负责管理Candidates，联通性检测，收发数据包。

2. 创建dtls层的传输对象，提供Dtls握手逻辑，密钥交换。注意其dtls内置了ice层的传输对象，其层次与DatagramTransport是平行关系。

3. 根据是否使用加密以及加密手段，来创建RTP层不同的传输对象

   - 不使用加密，则创建不使用加密手段的rtp层传输对象——>RtpTransport

     > 注意：仍然传递了dtls层的传输对象，但该对象可以不进行加密，直接将上层的包传递给ice层传输对象。

   - 使用SDES加密，因为sdp中包含了密钥。创建使用SDES加密的rtp层传输对象——>SrtpTransport

     > 注意：传递了dtls层的传输对象，rtp层传输对象是其上层，但不使用dtls层的传输对象的握手，可以提高媒体建立链路的速度。

   - 使用dtls加密。创建使用dtls加密手段的rtp层传输对象——>DtlsSrtpTransport

     > 注意：传递了dtls层的传输对象，rtp层传输对象是其上层，使用dtls层的传输对象提供的握手和加密，建立安全通信速度慢。

4. 创建 JsepTransport 对象。来容纳之前创建的各层对象 ice层：两个传输对象ice、rtcp_ice； dtls层/datagram层：rtp_dtls_transport、rtcp_dtls_transport/datagram_transport 基于dtls层的rtp层：unencrypted_rtp_transport、sdes_transport、dtls_srtp_transport 基于datagram层的rtp层：datagram_rtp_transport 基于dtls层的应用数据传输层：sctp_transport 基于datagram层的应用数据传输层：data_channel_transport
5. `SetTransportForMid`, 将创建的JsepTransport添加到JsepTransportController的成员上添加到mid_to_transport_。
6. 更新状态，进行ICE
7. 这个函数根据 SDP 的内容创建各种各样的 transport ，这里我们看到熟悉的接口 CreateDtlsSrtpTransport ，这个创建了 webrtc::DtlsSrtpTransport 对象，webrtc::DtlsSrtpTransport 对象下面的两个成员对象非常关键。



### 3.1 --JsepTransportController::CreateIceTransport



### 3.2 -- JsepTransportController::CreateDtlsTransport

### 3.3 -- JsepTransportController::CreateUnencryptedRtpTransport

### 3.4 -- JsepTransportController::CreateSdesTransport

### 3.5 -- JsepTransportController::CreateDtlsSrtpTransport

### 3.6 --JsepTransport::JsepTransport



## 4. JsepTransportController::CreateIceTransport

pc/jsep_transport_controller.cc

```cpp
// 返回的是 DefaultIceTransport
rtc::scoped_refptr<webrtc::IceTransportInterface>
JsepTransportController::CreateIceTransport(const std::string& transport_name,
                                            bool rtcp) {
  int component = rtcp ? cricket::ICE_CANDIDATE_COMPONENT_RTCP
                       : cricket::ICE_CANDIDATE_COMPONENT_RTP;

  IceTransportInit init;
  // cricket::PortAllocator* port_allocator_, 是从构造传入，
  // 是在PeerConnection的时候创建,
  // PeerConnectionFactory::CreatePeerConnectionOrError
  init.set_port_allocator(port_allocator_);
  init.set_async_resolver_factory(async_resolver_factory_);
  init.set_event_log(config_.event_log);
  // JsepTransportController::Config config_
  // config 是从构造函数传入
  // 是在PeerConnection::Init 创建
  //
  // DefaultIceTransportFactory ice_transport_factory
  // 是在PeerConnection的时候创建,
  // PeerConnectionFactory::CreatePeerConnectionOrError
  return config_.ice_transport_factory->CreateIceTransport(
      transport_name, component, std::move(init));
}
```

1. 通过`DefaultIceTransportFactory` 创建了 `DefaultIceTransport`。
2. 创建`DefaultIceTransport` 需要`P2PTransportChannel`;   `P2PTransportChannel` 存放在 `DefaultIceTransport`，通过`DefaultIceTransport::internal()`获取得到。
3. 返回的是 `DefaultIceTransport`,  就是` IceTransportInterface`



### 4.1 DefaultIceTransportFactory::CreateIceTransport

p2p/base/default_ice_transport_factory.cc

```cpp
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





#### 4.1.1 DefaultIceTransport

##### 继承关系

```
IceTransportInterface
DefaultIceTransport
```



p2p/base/default_ice_transport_factory.h

```cpp
// The default ICE transport wraps the implementation of IceTransportInternal
// provided by P2PTransportChannel. This default transport is not thread safe
// and must be constructed, used and destroyed on the same network thread on
// which the internal P2PTransportChannel lives.
class DefaultIceTransport : public IceTransportInterface {
 public:
  explicit DefaultIceTransport(
      std::unique_ptr<cricket::P2PTransportChannel> internal);
  ~DefaultIceTransport();

  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  cricket::IceTransportInternal* internal() override {
    RTC_DCHECK_RUN_ON(&thread_checker_);
    return internal_.get();
  }

 private:
  const rtc::ThreadChecker thread_checker_{};
  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  std::unique_ptr<cricket::P2PTransportChannel> internal_
      RTC_GUARDED_BY(thread_checker_);
};
```





#### 4.1.2 DefaultIceTransportFactory

```less
IceTransportFactory
DefaultIceTransportFactory
```

p2p/base/default_ice_transport_factory.h

```cpp
class DefaultIceTransportFactory : public IceTransportFactory {
 public:
  DefaultIceTransportFactory() = default;
  ~DefaultIceTransportFactory() = default;

  // Must be called on the network thread and returns a DefaultIceTransport.
  rtc::scoped_refptr<IceTransportInterface> CreateIceTransport(
      const std::string& transport_name,
      int component,
      IceTransportInit init) override;
};
```





#### 4.1.3 ??? P2PTransportChannel

##### 继承关系

```less
rtc::PacketTransportInternal
IceTransportInternal
P2PTransportChannel
```



## 5. JsepTransportController::CreateDtlsTransport

pc/jsep_transport_controller.cc

```cpp
std::unique_ptr<cricket::DtlsTransportInternal>
JsepTransportController::CreateDtlsTransport(
    const cricket::ContentInfo& content_info,
    cricket::IceTransportInternal* ice) {
  RTC_DCHECK(network_thread_->IsCurrent());

  std::unique_ptr<cricket::DtlsTransportInternal> dtls;

  // 这个是空置
  if (config_.dtls_transport_factory) {
      // for test
    dtls = config_.dtls_transport_factory->CreateDtlsTransport(
        ice, config_.crypto_options);
  } else {
     // 流程走这里
     // 传入的ice就是 IceTransportInterface，就是DefaultIceTransport
     // CryptoOptions config_.crypto_options， 加密策略，比如用什么加密算法
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

1. 创建DtlsTransport，传入的ice就是 `IceTransportInterface`，就是`DefaultIceTransport`，和`CryptoOptions config_.crypto_options`， 加密策略，比如用什么加密算法

2. 继承关系

   ```less
   rtc::PacketTransportInternal
   DtlsTransportInternal
   DtlsTransport
   ```

3. 返回 DtlsTransportInternal， 就是 DtlsTransport



### 5.1 DtlsTransport::DtlsTransport

p2p/base/dtls_transport.cc

```cpp
DtlsTransport::DtlsTransport(IceTransportInternal* ice_transport,
                             const webrtc::CryptoOptions& crypto_options,
                             webrtc::RtcEventLog* event_log)
    : transport_name_(ice_transport->transport_name()),
      component_(ice_transport->component()),
      ice_transport_(ice_transport),
      downward_(NULL),
      srtp_ciphers_(crypto_options.GetSupportedDtlsSrtpCryptoSuites()),
      ssl_max_version_(rtc::SSL_PROTOCOL_DTLS_12),
      crypto_options_(crypto_options),
      event_log_(event_log) {
  RTC_DCHECK(ice_transport_);
  ConnectToIceTransport();
}
```



```cpp
void DtlsTransport::ConnectToIceTransport() {
  RTC_DCHECK(ice_transport_);
  ice_transport_->SignalWritableState.connect(this,
                                              &DtlsTransport::OnWritableState);
  ice_transport_->SignalReadPacket.connect(this, &DtlsTransport::OnReadPacket);
  ice_transport_->SignalSentPacket.connect(this, &DtlsTransport::OnSentPacket);
  ice_transport_->SignalReadyToSend.connect(this,
                                            &DtlsTransport::OnReadyToSend);
  ice_transport_->SignalReceivingState.connect(
      this, &DtlsTransport::OnReceivingState);
  ice_transport_->SignalNetworkRouteChanged.connect(
      this, &DtlsTransport::OnNetworkRouteChanged);
}
```



### 5.2 ??? DtlsTransport

#### 5.2.1 继承关系

```
rtc::PacketTransportInternal
DtlsTransportInternal
DtlsTransport
```



p2p/base/dtls_transport.h

```cpp
class DtlsTransport : public DtlsTransportInternal {
...

  // Underlying ice_transport, not owned by this class.
  IceTransportInternal* ice_transport_;
  std::unique_ptr<rtc::SSLStreamAdapter> dtls_;  // The DTLS stream
}
```



## 6. JsepTransportController::CreateUnencryptedRtpTransport

```cpp
std::unique_ptr<webrtc::RtpTransport>
JsepTransportController::CreateUnencryptedRtpTransport(
    const std::string& transport_name,
    rtc::PacketTransportInternal* rtp_packet_transport,
    rtc::PacketTransportInternal* rtcp_packet_transport) {
  RTC_DCHECK(network_thread_->IsCurrent());
  auto unencrypted_rtp_transport =
      std::make_unique<RtpTransport>(rtcp_packet_transport == nullptr);
  // DtlsTransportInternal rtp
  // DtlsTransportInternal 父类 PacketTransportInternal
  unencrypted_rtp_transport->SetRtpPacketTransport(rtp_packet_transport);
  if (rtcp_packet_transport) {
    // DtlsTransportInternal rtcp 
    unencrypted_rtp_transport->SetRtcpPacketTransport(rtcp_packet_transport);
  }
  return unencrypted_rtp_transport;
}
```



### 6.1 继承关系

```less
RtpTransportInternal
webrtc::RtpTransport
```



## 7. JsepTransportController::CreateSdesTransport

```cpp
std::unique_ptr<webrtc::SrtpTransport>
JsepTransportController::CreateSdesTransport(
    const std::string& transport_name,
    cricket::DtlsTransportInternal* rtp_dtls_transport,
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



### 7.1 继承关系

```less
RtpTransportInternal
webrtc::RtpTransport
webrtc::SrtpTransport
```



## 8. JsepTransportController::CreateDtlsSrtpTransport

```cpp
std::unique_ptr<webrtc::DtlsSrtpTransport>
JsepTransportController::CreateDtlsSrtpTransport(
    const std::string& transport_name,
    cricket::DtlsTransportInternal* rtp_dtls_transport,
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

DTLS-SRTP， 是 DTLS(Datagram Transport Layer Security)即数据包传输层安全性*协议*。[TLS](https://baike.baidu.com/item/TLS/2979545)不能用来保证[UDP](https://baike.baidu.com/item/UDP/571511)上传输的数据的安全，因此Datagram TLS试图在现存的[TLS协议](https://baike.baidu.com/item/TLS协议/7129331)架构上提出扩展，使之支持UDP，即成为TLS的一个支持数据包传输的版本。

### 8.1 继承关系

```less
RtpTransportInternal
webrtc::RtpTransport
webrtc::SrtpTransport
webrtc::DtlsSrtpTransport
```



## 9. JsepTransport::JsepTransport

pc/jsep_transport.cc

```cpp
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
      rtcp_dtls_transport_(
          rtcp_dtls_transport
              ? new rtc::RefCountedObject<webrtc::DtlsTransport>(
                    std::move(rtcp_dtls_transport))
              : nullptr),
	  ... {
  	...
}
```



### 9.1 JsepTransport属性

```cpp
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
```





## 参考

[[WebRTC 架构分析] WebRTC 的 Transport 层分析](https://zhuanlan.zhihu.com/p/473821266)

[WebRTC Native源码分析——P2P连接过程详解](https://blog.csdn.net/paradox_1_0/article/details/109206079)




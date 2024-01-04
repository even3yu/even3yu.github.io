---
layout: post
title: BaseChannel
date: 2024-01-04 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio channel webrtc
---


* content
{:toc}

---

## 1. 继承关系

```less
ChannelInterface  MediaChannel::NetworkInterface  webrtc::RtpPacketSinkInterface
BaseChannel
```



## 2. ChannelInterface

pc/channel_interface.h

```cpp
// ChannelInterface contains methods common to voice, video and data channels.
// As more methods are added to BaseChannel, they should be included in the
// interface as well.
class ChannelInterface {
 public:
  virtual cricket::MediaType media_type() const = 0;

  virtual MediaChannel* media_channel() const = 0;

  // TODO(deadbeef): This is redundant; remove this.
  virtual const std::string& transport_name() const = 0;

  virtual const std::string& content_name() const = 0;

  virtual bool enabled() const = 0;

  // Enables or disables this channel
  virtual bool Enable(bool enable) = 0;

  // Used for latency measurements.
  virtual sigslot::signal1<ChannelInterface*>& SignalFirstPacketReceived() = 0;

  // Channel control
  virtual bool SetLocalContent(const MediaContentDescription* content,
                               webrtc::SdpType type,
                               std::string* error_desc) = 0;
  virtual bool SetRemoteContent(const MediaContentDescription* content,
                                webrtc::SdpType type,
                                std::string* error_desc) = 0;
  virtual void SetPayloadTypeDemuxingEnabled(bool enabled) = 0;
  virtual bool UpdateRtpTransport(std::string* error_desc) = 0;

  // Access to the local and remote streams that were set on the channel.
  virtual const std::vector<StreamParams>& local_streams() const = 0;
  virtual const std::vector<StreamParams>& remote_streams() const = 0;

  // Set an RTP level transport.
  // Some examples:
  //   * An RtpTransport without encryption.
  //   * An SrtpTransport for SDES.
  //   * A DtlsSrtpTransport for DTLS-SRTP.
  virtual bool SetRtpTransport(webrtc::RtpTransportInternal* rtp_transport) = 0;

  // Returns the last negotiated header extensions.
  virtual RtpHeaderExtensions GetNegotiatedRtpHeaderExtensions() const = 0;

 protected:
  virtual ~ChannelInterface() = default;
};
```

BaseChannel  对外接口 ChannelInterface。

## 3. !!! MediaChannel::NetworkInterface

media/base/media_channel.h

```cpp
  class NetworkInterface {
   public:
    enum SocketType { ST_RTP, ST_RTCP };
    virtual bool SendPacket(rtc::CopyOnWriteBuffer* packet,
                            const rtc::PacketOptions& options) = 0;
    virtual bool SendRtcp(rtc::CopyOnWriteBuffer* packet,
                          const rtc::PacketOptions& options) = 0;
    virtual int SetOption(SocketType type,
                          rtc::Socket::Option opt,
                          int option) = 0;
    virtual ~NetworkInterface() {}
  };
```

发送rtp，rtcp数据。

## 4. !!! webrtc::RtpPacketSinkInterface

```cpp
// This class represents a receiver of already parsed RTP packets.
class RtpPacketSinkInterface {
 public:
  virtual ~RtpPacketSinkInterface() = default;
  virtual void OnRtpPacket(const RtpPacketReceived& packet) = 0;
};
```

接收到rtp数据包。

## 5. BaseChannel

pc/channel.cc

```cpp
// BaseChannel contains logic common to voice and video, including enable,
// marshaling calls to a worker and network threads, and connection and media
// monitors.
//
// BaseChannel assumes signaling and other threads are allowed to make
// synchronous calls to the worker thread, the worker thread makes synchronous
// calls only to the network thread, and the network thread can't be blocked by
// other threads.
// All methods with _n suffix must be called on network thread,
//     methods with _w suffix on worker thread
// and methods with _s suffix on signaling thread.
// Network and worker threads may be the same thread.
//
// WARNING! SUBCLASSES MUST CALL Deinit() IN THEIR DESTRUCTORS!
// This is required to avoid a data race between the destructor modifying the
// vtable, and the media channel's thread using BaseChannel as the
// NetworkInterface.

class BaseChannel : public ChannelInterface,
                    public rtc::MessageHandlerAutoCleanup,
                    public sigslot::has_slots<>,
                    public MediaChannel::NetworkInterface,
                    public webrtc::RtpPacketSinkInterface {
 public:
  // If |srtp_required| is true, the channel will not send or receive any
  // RTP/RTCP packets without using SRTP (either using SDES or DTLS-SRTP).
  // The BaseChannel does not own the UniqueRandomIdGenerator so it is the
  // responsibility of the user to ensure it outlives this object.
  // TODO(zhihuang:) Create a BaseChannel::Config struct for the parameter lists
  // which will make it easier to change the constructor.
  BaseChannel(rtc::Thread* worker_thread,
              rtc::Thread* network_thread,
              rtc::Thread* signaling_thread,
              std::unique_ptr<MediaChannel> media_channel,
              const std::string& content_name,
              bool srtp_required,
              webrtc::CryptoOptions crypto_options,
              rtc::UniqueRandomIdGenerator* ssrc_generator);
  virtual ~BaseChannel();
  virtual void Init_w(webrtc::RtpTransportInternal* rtp_transport);

  // Deinit may be called multiple times and is simply ignored if it's already
  // done.
  void Deinit();

  rtc::Thread* worker_thread() const { return worker_thread_; }
  rtc::Thread* network_thread() const { return network_thread_; }
  const std::string& content_name() const override { return content_name_; }
  // TODO(deadbeef): This is redundant; remove this.
  const std::string& transport_name() const override { return transport_name_; }
  bool enabled() const override { return enabled_; }

  // This function returns true if using SRTP (DTLS-based keying or SDES).
  bool srtp_active() const {
    RTC_DCHECK_RUN_ON(network_thread());
    return rtp_transport_ && rtp_transport_->IsSrtpActive();
  }

  // Version of the above that can be called from any thread.
  bool SrtpActiveForTesting() const {
    if (!network_thread_->IsCurrent()) {
      return network_thread_->Invoke<bool>(RTC_FROM_HERE,
                                           [this] { return srtp_active(); });
    }
    RTC_DCHECK_RUN_ON(network_thread());
    return srtp_active();
  }
  // Set an RTP level transport which could be an RtpTransport without
  // encryption, an SrtpTransport for SDES or a DtlsSrtpTransport for DTLS-SRTP.
  // This can be called from any thread and it hops to the network thread
  // internally. It would replace the |SetTransports| and its variants.
  bool SetRtpTransport(webrtc::RtpTransportInternal* rtp_transport) override;

  webrtc::RtpTransportInternal* rtp_transport() const {
    RTC_DCHECK_RUN_ON(network_thread());
    return rtp_transport_;
  }

  // Version of the above that can be called from any thread.
  webrtc::RtpTransportInternal* RtpTransportForTesting() const {
    if (!network_thread_->IsCurrent()) {
      return network_thread_->Invoke<webrtc::RtpTransportInternal*>(
          RTC_FROM_HERE, [this] { return rtp_transport(); });
    }
    RTC_DCHECK_RUN_ON(network_thread());
    return rtp_transport();
  }

  // Channel control. Must call UpdateRtpTransport afterwards to apply any
  // changes to the RtpTransport on the network thread.
  bool SetLocalContent(const MediaContentDescription* content,
                       webrtc::SdpType type,
                       std::string* error_desc) override;
  bool SetRemoteContent(const MediaContentDescription* content,
                        webrtc::SdpType type,
                        std::string* error_desc) override;
  // Controls whether this channel will receive packets on the basis of
  // matching payload type alone. This is needed for legacy endpoints that
  // don't signal SSRCs or use MID/RID, but doesn't make sense if there is
  // more than channel of specific media type, As that creates an ambiguity.
  //
  // This method will also remove any existing streams that were bound to this
  // channel on the basis of payload type, since one of these streams might
  // actually belong to a new channel. See: crbug.com/webrtc/11477
  //
  // As with SetLocalContent/SetRemoteContent, must call UpdateRtpTransport
  // afterwards to apply changes to the RtpTransport on the network thread.
  void SetPayloadTypeDemuxingEnabled(bool enabled) override;
  bool UpdateRtpTransport(std::string* error_desc) override;

  bool Enable(bool enable) override;

  const std::vector<StreamParams>& local_streams() const override {
    return local_streams_;
  }
  const std::vector<StreamParams>& remote_streams() const override {
    return remote_streams_;
  }

  // Used for latency measurements.
  sigslot::signal1<ChannelInterface*>& SignalFirstPacketReceived() override;

  // Forward SignalSentPacket to worker thread.
  sigslot::signal1<const rtc::SentPacket&>& SignalSentPacket();

  // From RtpTransport - public for testing only
  void OnTransportReadyToSend(bool ready);

  // Only public for unit tests.  Otherwise, consider protected.
  int SetOption(SocketType type, rtc::Socket::Option o, int val) override;
  int SetOption_n(SocketType type, rtc::Socket::Option o, int val)
      RTC_RUN_ON(network_thread());

  // RtpPacketSinkInterface overrides.
  void OnRtpPacket(const webrtc::RtpPacketReceived& packet) override;

  // Used by the RTCStatsCollector tests to set the transport name without
  // creating RtpTransports.
  void set_transport_name_for_testing(const std::string& transport_name) {
    transport_name_ = transport_name;
  }

  MediaChannel* media_channel() const override {
    return media_channel_.get();
  }

 protected:
  bool was_ever_writable() const {
    RTC_DCHECK_RUN_ON(worker_thread());
    return was_ever_writable_;
  }
  void set_local_content_direction(webrtc::RtpTransceiverDirection direction) {
    RTC_DCHECK_RUN_ON(worker_thread());
    local_content_direction_ = direction;
  }
  void set_remote_content_direction(webrtc::RtpTransceiverDirection direction) {
    RTC_DCHECK_RUN_ON(worker_thread());
    remote_content_direction_ = direction;
  }
  // These methods verify that:
  // * The required content description directions have been set.
  // * The channel is enabled.
  // * And for sending:
  //   - The SRTP filter is active if it's needed.
  //   - The transport has been writable before, meaning it should be at least
  //     possible to succeed in sending a packet.
  //
  // When any of these properties change, UpdateMediaSendRecvState_w should be
  // called.
  bool IsReadyToReceiveMedia_w() const RTC_RUN_ON(worker_thread());
  bool IsReadyToSendMedia_w() const RTC_RUN_ON(worker_thread());
  rtc::Thread* signaling_thread() const { return signaling_thread_; }

  void FlushRtcpMessages_n() RTC_RUN_ON(network_thread());

  // NetworkInterface implementation, called by MediaEngine
  bool SendPacket(rtc::CopyOnWriteBuffer* packet,
                  const rtc::PacketOptions& options) override;
  bool SendRtcp(rtc::CopyOnWriteBuffer* packet,
                const rtc::PacketOptions& options) override;

  // From RtpTransportInternal
  void OnWritableState(bool writable);

  void OnNetworkRouteChanged(absl::optional<rtc::NetworkRoute> network_route);

  bool PacketIsRtcp(const rtc::PacketTransportInternal* transport,
                    const char* data,
                    size_t len);
  bool SendPacket(bool rtcp,
                  rtc::CopyOnWriteBuffer* packet,
                  const rtc::PacketOptions& options);

  void EnableMedia_w() RTC_RUN_ON(worker_thread());
  void DisableMedia_w() RTC_RUN_ON(worker_thread());

  // Performs actions if the RTP/RTCP writable state changed. This should
  // be called whenever a channel's writable state changes or when RTCP muxing
  // becomes active/inactive.
  void UpdateWritableState_n() RTC_RUN_ON(network_thread());
  void ChannelWritable_n() RTC_RUN_ON(network_thread());
  void ChannelNotWritable_n() RTC_RUN_ON(network_thread());

  bool AddRecvStream_w(const StreamParams& sp) RTC_RUN_ON(worker_thread());
  bool RemoveRecvStream_w(uint32_t ssrc) RTC_RUN_ON(worker_thread());
  void ResetUnsignaledRecvStream_w() RTC_RUN_ON(worker_thread());
  void SetPayloadTypeDemuxingEnabled_w(bool enabled)
      RTC_RUN_ON(worker_thread());
  bool AddSendStream_w(const StreamParams& sp) RTC_RUN_ON(worker_thread());
  bool RemoveSendStream_w(uint32_t ssrc) RTC_RUN_ON(worker_thread());

  // Should be called whenever the conditions for
  // IsReadyToReceiveMedia/IsReadyToSendMedia are satisfied (or unsatisfied).
  // Updates the send/recv state of the media channel.
  virtual void UpdateMediaSendRecvState_w() = 0;

  bool UpdateLocalStreams_w(const std::vector<StreamParams>& streams,
                            webrtc::SdpType type,
                            std::string* error_desc)
      RTC_RUN_ON(worker_thread());
  bool UpdateRemoteStreams_w(const std::vector<StreamParams>& streams,
                             webrtc::SdpType type,
                             std::string* error_desc)
      RTC_RUN_ON(worker_thread());
  virtual bool SetLocalContent_w(const MediaContentDescription* content,
                                 webrtc::SdpType type,
                                 std::string* error_desc) = 0;
  virtual bool SetRemoteContent_w(const MediaContentDescription* content,
                                  webrtc::SdpType type,
                                  std::string* error_desc) = 0;
  // Return a list of RTP header extensions with the non-encrypted extensions
  // removed depending on the current crypto_options_ and only if both the
  // non-encrypted and encrypted extension is present for the same URI.
  RtpHeaderExtensions GetFilteredRtpHeaderExtensions(
      const RtpHeaderExtensions& extensions);
  // Set a list of RTP extensions we should prepare to receive on the next
  // UpdateRtpTransport call.
  void SetReceiveExtensions(const RtpHeaderExtensions& extensions);

  // From MessageHandler
  void OnMessage(rtc::Message* pmsg) override;

  // Helper function template for invoking methods on the worker thread.
  template <class T>
  T InvokeOnWorker(const rtc::Location& posted_from,
                   rtc::FunctionView<T()> functor) {
    return worker_thread_->Invoke<T>(posted_from, functor);
  }

  // Add |payload_type| to |demuxer_criteria_| if payload type demuxing is
  // enabled.
  void MaybeAddHandledPayloadType(int payload_type) RTC_RUN_ON(worker_thread());

  void ClearHandledPayloadTypes() RTC_RUN_ON(worker_thread());
  // Return description of media channel to facilitate logging
  std::string ToString() const;

  void SetNegotiatedHeaderExtensions_w(const RtpHeaderExtensions& extensions);

  // ChannelInterface overrides
  RtpHeaderExtensions GetNegotiatedRtpHeaderExtensions() const override;

  bool has_received_packet_ = false;

 private:
  bool ConnectToRtpTransport();
  void DisconnectFromRtpTransport();
  void SignalSentPacket_n(const rtc::SentPacket& sent_packet)
      RTC_RUN_ON(network_thread());

  rtc::Thread* const worker_thread_;
  rtc::Thread* const network_thread_;
  rtc::Thread* const signaling_thread_;
  rtc::AsyncInvoker invoker_;
  sigslot::signal1<ChannelInterface*> SignalFirstPacketReceived_
      RTC_GUARDED_BY(signaling_thread_);
  sigslot::signal1<const rtc::SentPacket&> SignalSentPacket_
      RTC_GUARDED_BY(worker_thread_);

  const std::string content_name_;

  // Won't be set when using raw packet transports. SDP-specific thing.
  // TODO(bugs.webrtc.org/12230): Written on network thread, read on
  // worker thread (at least).
  std::string transport_name_;

  webrtc::RtpTransportInternal* rtp_transport_
      RTC_GUARDED_BY(network_thread()) = nullptr;

  std::vector<std::pair<rtc::Socket::Option, int> > socket_options_
      RTC_GUARDED_BY(network_thread());
  std::vector<std::pair<rtc::Socket::Option, int> > rtcp_socket_options_
      RTC_GUARDED_BY(network_thread());
  bool writable_ RTC_GUARDED_BY(network_thread()) = false;
  bool was_ever_writable_n_ RTC_GUARDED_BY(network_thread()) = false;
  bool was_ever_writable_ RTC_GUARDED_BY(worker_thread()) = false;
  const bool srtp_required_ = true;
  const webrtc::CryptoOptions crypto_options_;

  // MediaChannel related members that should be accessed from the worker
  // thread.
  const std::unique_ptr<MediaChannel> media_channel_;
  // Currently the |enabled_| flag is accessed from the signaling thread as
  // well, but it can be changed only when signaling thread does a synchronous
  // call to the worker thread, so it should be safe.
  bool enabled_ = false;
  bool payload_type_demuxing_enabled_ RTC_GUARDED_BY(worker_thread()) = true;
  std::vector<StreamParams> local_streams_ RTC_GUARDED_BY(worker_thread());
  std::vector<StreamParams> remote_streams_ RTC_GUARDED_BY(worker_thread());
  // TODO(bugs.webrtc.org/12230): local_content_direction and
  // remote_content_direction are set on the worker thread, but accessed on the
  // network thread.
  webrtc::RtpTransceiverDirection local_content_direction_ =
      webrtc::RtpTransceiverDirection::kInactive;
  webrtc::RtpTransceiverDirection remote_content_direction_ =
      webrtc::RtpTransceiverDirection::kInactive;

  // Cached list of payload types, used if payload type demuxing is re-enabled.
  std::set<uint8_t> payload_types_ RTC_GUARDED_BY(worker_thread());
  // TODO(bugs.webrtc.org/12239): These two variables are modified on the worker
  // thread, accessed on the network thread in UpdateRtpTransport.
  webrtc::RtpDemuxerCriteria demuxer_criteria_;
  RtpHeaderExtensions receive_rtp_header_extensions_;
  // This generator is used to generate SSRCs for local streams.
  // This is needed in cases where SSRCs are not negotiated or set explicitly
  // like in Simulcast.
  // This object is not owned by the channel so it must outlive it.
  rtc::UniqueRandomIdGenerator* const ssrc_generator_;

  // |negotiated_header_extensions_| is read on the signaling thread, but
  // written on the worker thread while being sync-invoked from the signal
  // thread in SdpOfferAnswerHandler::PushdownMediaDescription(). Hence the lock
  // isn't strictly needed, but it's anyway placed here for future safeness.
  mutable webrtc::Mutex negotiated_header_extensions_lock_;
  RtpHeaderExtensions negotiated_header_extensions_
      RTC_GUARDED_BY(negotiated_header_extensions_lock_);
};
```



### 5.1 属性

```cpp
  // MediaChannel related members that should be accessed from the worker
  // thread.
  const std::unique_ptr<MediaChannel> media_channel_;

//--------------------
  const std::string content_name_;

//--------------------
  // Won't be set when using raw packet transports. SDP-specific thing.
  // TODO(bugs.webrtc.org/12230): Written on network thread, read on
  // worker thread (at least).
  std::string transport_name_;
  webrtc::RtpTransportInternal* rtp_transport_
      RTC_GUARDED_BY(network_thread()) = nullptr;


//--------------------
  std::vector<StreamParams> local_streams_ RTC_GUARDED_BY(worker_thread());
  std::vector<StreamParams> remote_streams_ RTC_GUARDED_BY(worker_thread());
```



### 5.2 BaseChannel::Init_w

pc/channel.cc

```cpp
void BaseChannel::Init_w(webrtc::RtpTransportInternal* rtp_transport) {
  RTC_DCHECK_RUN_ON(worker_thread());

  // 保存webrtc::RtpTransportInternal
  network_thread_->Invoke<void>(
      RTC_FROM_HERE, [this, rtp_transport] { SetRtpTransport(rtp_transport); });

  // Both RTP and RTCP channels should be set, we can call SetInterface on
  // the media channel and it can set network options.
  // MediaChannel，数据会从MediaChannel 转发到BaseChannel
  media_channel_->SetInterface(this);
}
```

1. 保存webrtc::RtpTransportInternal；
2. media_channel_ 就是MediaChannel， 数据会从MediaChannel 转发到BaseChannel。

### ------

### 5.3 BaseChannel::SetRtpTransport

```cpp
// Set an RTP level transport which could be an RtpTransport without
  // encryption, an SrtpTransport for SDES or a DtlsSrtpTransport for DTLS-SRTP.
  // This can be called from any thread and it hops to the network thread
  // internally. It would replace the |SetTransports| and its variants.
bool BaseChannel::SetRtpTransport(webrtc::RtpTransportInternal* rtp_transport) {
  if (!network_thread_->IsCurrent()) {
    return network_thread_->Invoke<bool>(RTC_FROM_HERE, [this, rtp_transport] {
      return SetRtpTransport(rtp_transport);
    });
  }
  RTC_DCHECK_RUN_ON(network_thread());
  if (rtp_transport == rtp_transport_) {
    return true;
  }

  if (rtp_transport_) {
    DisconnectFromRtpTransport();
  }

  rtp_transport_ = rtp_transport;
  if (rtp_transport_) {
    transport_name_ = rtp_transport_->transport_name();

    if (!ConnectToRtpTransport()) {
      RTC_LOG(LS_ERROR) << "Failed to connect to the new RtpTransport for "
                        << ToString() << ".";
      return false;
    }
    OnTransportReadyToSend(rtp_transport_->IsReadyToSend());
    UpdateWritableState_n();

    // Set the cached socket options.
    for (const auto& pair : socket_options_) {
      rtp_transport_->SetRtpOption(pair.first, pair.second);
    }
    if (!rtp_transport_->rtcp_mux_enabled()) {
      for (const auto& pair : rtcp_socket_options_) {
        rtp_transport_->SetRtcpOption(pair.first, pair.second);
      }
    }
  }
  return true;
}
```



### 5.4 BaseChannel::rtp_transport

```cpp
  webrtc::RtpTransportInternal* rtp_transport() const {
    RTC_DCHECK_RUN_ON(network_thread());
    return rtp_transport_;
  }
```

### ??? BaseChannel::UpdateRtpTransport

什么时候调用。

```less
SdpOfferAnswerHandler::DoSetLocalDescription SdpOfferAnswerHandler::DoSetRemoteDescription
SdpOfferAnswerHandler::ApplyLocalDescription SdpOfferAnswerHandler::ApplyRemoteDescription
SdpOfferAnswerHandler::UpdateSessionState
SdpOfferAnswerHandler::PushdownMediaDescription
SdpOfferAnswerHandler::ApplyChannelUpdates
BaseChannel::UpdateRtpTransport
```



```cpp
bool BaseChannel::UpdateRtpTransport(std::string* error_desc) {
  return network_thread_->Invoke<bool>(RTC_FROM_HERE, [this, error_desc] {
    RTC_DCHECK_RUN_ON(network_thread());
    RTC_DCHECK(rtp_transport_);
    // TODO(bugs.webrtc.org/12230): This accesses demuxer_criteria_ on the
    // networking thread.
    if (!rtp_transport_->RegisterRtpDemuxerSink(demuxer_criteria_, this)) {
      RTC_LOG(LS_ERROR) << "Failed to set up demuxing for " << ToString();
      rtc::StringBuilder desc;
      desc << "Failed to set up demuxing for m-section with mid='"
           << content_name() << "'.";
      SafeSetError(desc.str(), error_desc);
      return false;
    }
    // NOTE: This doesn't take the BUNDLE case in account meaning the RTP header
    // extension maps are not merged when BUNDLE is enabled. This is fine
    // because the ID for MID should be consistent among all the RTP transports,
    // and that's all RtpTransport uses this map for.
    //
    // TODO(deadbeef): Move this call to JsepTransport, there is no reason
    // BaseChannel needs to be involved here.
    if (media_type() != cricket::MEDIA_TYPE_DATA) {
      rtp_transport_->UpdateRtpHeaderExtensionMap(
          receive_rtp_header_extensions_);
    }
    return true;
  });
}
```



### ------

### 5.5 !!! BaseChannel::SetLocalContent

```cpp
bool BaseChannel::SetLocalContent(const MediaContentDescription* content,
                                  SdpType type,
                                  std::string* error_desc) {
  TRACE_EVENT0("webrtc", "BaseChannel::SetLocalContent");
  return InvokeOnWorker<bool>(
      RTC_FROM_HERE,
      Bind(&BaseChannel::SetLocalContent_w, this, content, type, error_desc));
}
```



#### 5.5.1 BaseChannel::SetLocalContent_w

```cpp
virtual bool SetLocalContent_w(const MediaContentDescription* content,
                                 webrtc::SdpType type,
                                 std::string* error_desc) = 0;
```

### 5.6 !!! BaseChannel::SetRemoteContent

```cpp
bool BaseChannel::SetRemoteContent(const MediaContentDescription* content,
                                   SdpType type,
                                   std::string* error_desc) {
  TRACE_EVENT0("webrtc", "BaseChannel::SetRemoteContent");
  return InvokeOnWorker<bool>(
      RTC_FROM_HERE,
      Bind(&BaseChannel::SetRemoteContent_w, this, content, type, error_desc));
}
```



#### 5.6.1 BaseChannel::SetRemoteContent_w

```cpp
virtual bool SetRemoteContent_w(const MediaContentDescription* content,
                                  webrtc::SdpType type,
                                  std::string* error_desc) = 0;
```

### ------

### 5.7 BaseChannel::local_streams

```cpp
  const std::vector<StreamParams>& local_streams() const override {
    return local_streams_;
  }
```



### 5.8 BaseChannel::remote_streams

```cpp

  const std::vector<StreamParams>& remote_streams() const override {
    return remote_streams_;
  }
```

### ------

### 5.9 !!! BaseChannel::media_channel

```cpp
  MediaChannel* media_channel() const override {
    return media_channel_.get();
  }
```

### ------

### 5.10 BaseChannel::Enable

```cpp
bool BaseChannel::Enable(bool enable) {
  worker_thread_->Invoke<void>(
      RTC_FROM_HERE,
      Bind(enable ? &BaseChannel::EnableMedia_w : &BaseChannel::DisableMedia_w,
           this));
  return true;
}
```

开始发送或者接收数据，停止发送或者接收数据。

#### 5.10.1 bool enabled_

```cpp

  // Currently the |enabled_| flag is accessed from the signaling thread as
  // well, but it can be changed only when signaling thread does a synchronous
  // call to the worker thread, so it should be safe.
  bool enabled_ = false;
```



#### 5.10.2 BaseChannel::EnableMedia_w

```cpp
void BaseChannel::EnableMedia_w() {
  RTC_DCHECK(worker_thread_ == rtc::Thread::Current());
  if (enabled_)
    return;

  RTC_LOG(LS_INFO) << "Channel enabled: " << ToString();
  enabled_ = true;
  UpdateMediaSendRecvState_w();
}
```



#### 5.10.3 BaseChannel::DisableMedia_w

```cpp
void BaseChannel::DisableMedia_w() {
  RTC_DCHECK(worker_thread_ == rtc::Thread::Current());
  if (!enabled_)
    return;

  RTC_LOG(LS_INFO) << "Channel disabled: " << ToString();
  enabled_ = false;
  UpdateMediaSendRecvState_w();
}
```



#### 5.10.4 ??? BaseChannel::UpdateMediaSendRecvState_w

```cpp
  // Should be called whenever the conditions for
  // IsReadyToReceiveMedia/IsReadyToSendMedia are satisfied (or unsatisfied).
  // Updates the send/recv state of the media channel.
  virtual void UpdateMediaSendRecvState_w() = 0;
```

数据发送和播发。控制`webrtc::AudioSendStream`, `webrtc::AudioReceiveStream` , 通过start/stop。



### ------

### 5.11 BaseChannel::SendPacket

```cpp
bool BaseChannel::SendPacket(rtc::CopyOnWriteBuffer* packet,
                             const rtc::PacketOptions& options) {
  return SendPacket(false, packet, options);
}
```

MediaChannel::NetworkInterface 实现的方法。

### 5.12 BaseChannel::SendRtcp

```cpp
bool BaseChannel::SendRtcp(rtc::CopyOnWriteBuffer* packet,
                           const rtc::PacketOptions& options) {
  return SendPacket(true, packet, options);
}
```

MediaChannel::NetworkInterface 实现的方法。

### 5.13 !!! BaseChannel::SendPacket

```cpp
bool BaseChannel::SendPacket(bool rtcp,
                             rtc::CopyOnWriteBuffer* packet,
                             const rtc::PacketOptions& options) {
  // Until all the code is migrated to use RtpPacketType instead of bool.
  RtpPacketType packet_type = rtcp ? RtpPacketType::kRtcp : RtpPacketType::kRtp;
  // SendPacket gets called from MediaEngine, on a pacer or an encoder thread.
  // If the thread is not our network thread, we will post to our network
  // so that the real work happens on our network. This avoids us having to
  // synchronize access to all the pieces of the send path, including
  // SRTP and the inner workings of the transport channels.
  // The only downside is that we can't return a proper failure code if
  // needed. Since UDP is unreliable anyway, this should be a non-issue.
  if (!network_thread_->IsCurrent()) {
    // Avoid a copy by transferring the ownership of the packet data.
    int message_id = rtcp ? MSG_SEND_RTCP_PACKET : MSG_SEND_RTP_PACKET;
    SendPacketMessageData* data = new SendPacketMessageData;
    data->packet = std::move(*packet);
    data->options = options;
    network_thread_->Post(RTC_FROM_HERE, this, message_id, data);
    return true;
  }
  RTC_DCHECK_RUN_ON(network_thread());

  TRACE_EVENT0("webrtc", "BaseChannel::SendPacket");

  // Now that we are on the correct thread, ensure we have a place to send this
  // packet before doing anything. (We might get RTCP packets that we don't
  // intend to send.) If we've negotiated RTCP mux, send RTCP over the RTP
  // transport.
  if (!rtp_transport_ || !rtp_transport_->IsWritable(rtcp)) {
    return false;
  }

  // Protect ourselves against crazy data.
  if (!IsValidRtpPacketSize(packet_type, packet->size())) {
    RTC_LOG(LS_ERROR) << "Dropping outgoing " << ToString() << " "
                      << RtpPacketTypeToString(packet_type)
                      << " packet: wrong size=" << packet->size();
    return false;
  }

  if (!srtp_active()) {
    if (srtp_required_) {
      // The audio/video engines may attempt to send RTCP packets as soon as the
      // streams are created, so don't treat this as an error for RTCP.
      // See: https://bugs.chromium.org/p/webrtc/issues/detail?id=6809
      if (rtcp) {
        return false;
      }
      // However, there shouldn't be any RTP packets sent before SRTP is set up
      // (and SetSend(true) is called).
      RTC_LOG(LS_ERROR) << "Can't send outgoing RTP packet for " << ToString()
                        << " when SRTP is inactive and crypto is required";
      RTC_NOTREACHED();
      return false;
    }

#if 0
    std::string packet_type = rtcp ? "RTCP" : "RTP";
    RTC_DLOG(LS_WARNING) << "Sending an " << packet_type
                         << " packet without encryption for " << ToString()
                         << ".";
#endif
  }

  // Bon voyage.
  return rtcp ? rtp_transport_->SendRtcpPacket(packet, options, PF_SRTP_BYPASS)
              : rtp_transport_->SendRtpPacket(packet, options, PF_SRTP_BYPASS);
}
```



### 5.14 BaseChannel::OnRtpPacket

```cpp
void BaseChannel::OnRtpPacket(const webrtc::RtpPacketReceived& parsed_packet) {
  // Take packet time from the |parsed_packet|.
  // RtpPacketReceived.arrival_time_ms = (timestamp_us + 500) / 1000;
  int64_t packet_time_us = -1;
  if (parsed_packet.arrival_time_ms() > 0) {
    packet_time_us = parsed_packet.arrival_time_ms() * 1000;
  }

  if (!has_received_packet_) {
    has_received_packet_ = true;
    signaling_thread()->Post(RTC_FROM_HERE, this, MSG_FIRSTPACKETRECEIVED);
  }

  if (!srtp_active() && srtp_required_) {
    // Our session description indicates that SRTP is required, but we got a
    // packet before our SRTP filter is active. This means either that
    // a) we got SRTP packets before we received the SDES keys, in which case
    //    we can't decrypt it anyway, or
    // b) we got SRTP packets before DTLS completed on both the RTP and RTCP
    //    transports, so we haven't yet extracted keys, even if DTLS did
    //    complete on the transport that the packets are being sent on. It's
    //    really good practice to wait for both RTP and RTCP to be good to go
    //    before sending  media, to prevent weird failure modes, so it's fine
    //    for us to just eat packets here. This is all sidestepped if RTCP mux
    //    is used anyway.
    RTC_LOG(LS_WARNING) << "Can't process incoming RTP packet when "
                           "SRTP is inactive and crypto is required "
                        << ToString();
    return;
  }

  auto packet_buffer = parsed_packet.Buffer();

  invoker_.AsyncInvoke<void>(
      RTC_FROM_HERE, worker_thread_, [this, packet_buffer, packet_time_us] {
        RTC_DCHECK_RUN_ON(worker_thread());
        media_channel_->OnPacketReceived(packet_buffer, packet_time_us);
      });
}
```

webrtc::RtpPacketSinkInterface 实现的方法。

在` BaseChannel::ConnectToRtpTransport` 把 BaseChannle 注册给` webrtc::RtpTransportInternal`。数据会从webrtc::RtpTransportInternal 流向BaseChannel， 最终数据流转到MediaChannel。



## 6. VoiceChannel

pc/channel.h

### 6.1 继承关系

```less
ChannelInterface MediaChannel::NetworkInterface webrtc::RtpPacketSinkInterface
BaseChannel
VoiceChannel
```



### 6.2 属性

```cpp
const std::unique_ptr<MediaChannel> media_channel_; // 其实是在BaseChannel中

  // Last AudioSendParameters sent down to the media_channel() via
  // SetSendParameters.
  AudioSendParameters last_send_params_;
  // Last AudioRecvParameters sent down to the media_channel() via
  // SetRecvParameters.
  AudioRecvParameters last_recv_params_;·
```



### 6.3 ???SetLocalContent_w

```cpp
bool VoiceChannel::SetLocalContent_w(const MediaContentDescription* content,
                                     SdpType type,
                                     std::string* error_desc) {
  TRACE_EVENT0("webrtc", "VoiceChannel::SetLocalContent_w");
  RTC_DCHECK_RUN_ON(worker_thread());
  RTC_LOG(LS_INFO) << "Setting local voice description for " << ToString();

  RTC_DCHECK(content);
  if (!content) {
    SafeSetError("Can't find audio content in local description.", error_desc);
    return false;
  }

  const AudioContentDescription* audio = content->as_audio();

  if (type == SdpType::kAnswer)
    SetNegotiatedHeaderExtensions_w(audio->rtp_header_extensions());

  RtpHeaderExtensions rtp_header_extensions =
      GetFilteredRtpHeaderExtensions(audio->rtp_header_extensions());
  SetReceiveExtensions(rtp_header_extensions);
  media_channel()->SetExtmapAllowMixed(audio->extmap_allow_mixed());

  AudioRecvParameters recv_params = last_recv_params_;
  RtpParametersFromMediaDescription(
      audio, rtp_header_extensions,
      webrtc::RtpTransceiverDirectionHasRecv(audio->direction()), &recv_params);
  if (!media_channel()->SetRecvParameters(recv_params)) {
    SafeSetError(
        "Failed to set local audio description recv parameters for m-section "
        "with mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }

  if (webrtc::RtpTransceiverDirectionHasRecv(audio->direction())) {
    for (const AudioCodec& codec : audio->codecs()) {
      MaybeAddHandledPayloadType(codec.id);
    }
  }

  last_recv_params_ = recv_params;

  // TODO(pthatcher): Move local streams into AudioSendParameters, and
  // only give it to the media channel once we have a remote
  // description too (without a remote description, we won't be able
  // to send them anyway).
  if (!UpdateLocalStreams_w(audio->streams(), type, error_desc)) {
    SafeSetError(
        "Failed to set local audio description streams for m-section with "
        "mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }

  set_local_content_direction(content->direction());
  UpdateMediaSendRecvState_w();
  return true;
}
```

1. 从BaseChannel::SetLocalContent 调用 SetLocalContent_w。
2. 

### 6.4 ???SetRemoteContent_w

```cpp
bool VoiceChannel::SetRemoteContent_w(const MediaContentDescription* content,
                                      SdpType type,
                                      std::string* error_desc) {
  TRACE_EVENT0("webrtc", "VoiceChannel::SetRemoteContent_w");
  RTC_DCHECK_RUN_ON(worker_thread());
  RTC_LOG(LS_INFO) << "Setting remote voice description for " << ToString();

  RTC_DCHECK(content);
  if (!content) {
    SafeSetError("Can't find audio content in remote description.", error_desc);
    return false;
  }

  const AudioContentDescription* audio = content->as_audio();

  if (type == SdpType::kAnswer)
    SetNegotiatedHeaderExtensions_w(audio->rtp_header_extensions());

  RtpHeaderExtensions rtp_header_extensions =
      GetFilteredRtpHeaderExtensions(audio->rtp_header_extensions());

  AudioSendParameters send_params = last_send_params_;
  RtpSendParametersFromMediaDescription(
      audio, rtp_header_extensions,
      webrtc::RtpTransceiverDirectionHasRecv(audio->direction()), &send_params);
  send_params.mid = content_name();

  bool parameters_applied = media_channel()->SetSendParameters(send_params);
  if (!parameters_applied) {
    SafeSetError(
        "Failed to set remote audio description send parameters for m-section "
        "with mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }
  last_send_params_ = send_params;

  if (!webrtc::RtpTransceiverDirectionHasSend(content->direction())) {
    RTC_DLOG(LS_VERBOSE) << "SetRemoteContent_w: remote side will not send - "
                            "disable payload type demuxing for "
                         << ToString();
    ClearHandledPayloadTypes();
  }

  // TODO(pthatcher): Move remote streams into AudioRecvParameters,
  // and only give it to the media channel once we have a local
  // description too (without a local description, we won't be able to
  // recv them anyway).
  if (!UpdateRemoteStreams_w(audio->streams(), type, error_desc)) {
    SafeSetError(
        "Failed to set remote audio description streams for m-section with "
        "mid='" +
            content_name() + "'.",
        error_desc);
    return false;
  }

  set_remote_content_direction(content->direction());
  UpdateMediaSendRecvState_w();
  return true;
}
```

1. 从BaseChannel::SetRemoteContent 调用 SetRemoteContent_w。
2. 

### 6.5 ???UpdateMediaSendRecvState_w

```cpp
void VoiceChannel::UpdateMediaSendRecvState_w() {
  // Render incoming data if we're the active call, and we have the local
  // content. We receive data on the default channel and multiplexed streams.
  RTC_DCHECK_RUN_ON(worker_thread());
  bool recv = IsReadyToReceiveMedia_w();
  media_channel()->SetPlayout(recv);

  // Send outgoing data if we're the active call, we have the remote content,
  // and we have had some form of connectivity.
  bool send = IsReadyToSendMedia_w();
  media_channel()->SetSend(send);

  RTC_LOG(LS_INFO) << "Changing voice state, recv=" << recv << " send=" << send
                   << " for " << ToString();
}
```



#### 6.5.1 ??? BaseChannel::IsReadyToReceiveMedia_w

```cpp
bool BaseChannel::IsReadyToReceiveMedia_w() const {
  // Receive data if we are enabled and have local content,
  // local_content_direction_ 赋值？？？
  return enabled() &&
         webrtc::RtpTransceiverDirectionHasRecv(local_content_direction_);
}
```



#### 6.5.2 ??? BaseChannel::IsReadyToSendMedia_w

```cpp
bool BaseChannel::IsReadyToSendMedia_w() const {
  // Send outgoing data if we are enabled, have local and remote content,
  // and we have had some form of connectivity.
  // remote_content_direction_ 赋值？？？
  return enabled() &&
         webrtc::RtpTransceiverDirectionHasRecv(remote_content_direction_) &&
         webrtc::RtpTransceiverDirectionHasSend(local_content_direction_) &&
         was_ever_writable();
}
```



## 7. ???VideoChannel
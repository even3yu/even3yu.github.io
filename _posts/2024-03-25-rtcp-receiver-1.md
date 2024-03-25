---
layout: post
title: rtcp 数据包接收流程
date: 2024-03-25 22:52:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtp rtcp
---


* content
{:toc}

---



## 前言




## 1. RTCP接收数据的流程的堆栈信息

### 1.1 网络io 线程读取数据

```less
PhysicalSocketServer::WaitSelect


AllocationSequence::OnReadPacket(rtc::AsyncPacketSocket* socket,  const char* data, size_t size, const rtc::SocketAddress& remote_addr, const int64_t& packet_time_us)
UDPPort::HandleIncomingPacket(rtc::AsyncPacketSocket* socket, const char* data, size_t size, const rtc::SocketAddress& remote_addr, int64_t packet_time_us)
UDPPort::OnReadPacket(rtc::AsyncPacketSocket* socket, const char* data, size_t size, const rtc::SocketAddress& remote_addr, const int64_t& packet_time_us)
Connection::OnReadPacket(const char* data, size_t size, int64_t packet_time_us)
P2PTransportChannel::OnReadPacket(Connection* connection, const char* data, size_t len, int64_t packet_time_us)
DtlsTransport::OnReadPacket(rtc::PacketTransportInternal* transport, const char* data, size_t size, const int64_t& packet_time_us, int flags)
RtpTransport::OnReadPacket(rtc::PacketTransportInternal* transport, const char* data, size_t len, const int64_t& packet_time_us, int flags)
SrtpTransport::OnRtpPacketReceived(rtc::CopyOnWriteBuffer packet, int64_t packet_time_us)
RtpTransport::DemuxPacket(rtc::CopyOnWriteBuffer packet, int64_t packet_time_us)										
RtpDemuxer::OnRtpPacket(const RtpPacketReceived& packet)
BaseChannel::OnRtpPacket(const webrtc::RtpPacketReceived& parsed_packet)
```



### 1.2 线程切换的代码

```cpp
invoker_.AsyncInvoke<void>(
      RTC_FROM_HERE, worker_thread_,
      Bind(&WebRtcVideoChannel::OnPacketReceived, this, rtcp, packet, packet_time_us));
]

WebRtcVideoChannel::OnPacketReceived
Call::DeliverPacket
Call::DeliverRtcp
VideoSendStream::DeliverRtcp 
VideoSendStreamImpl::DeliverRtcp 
RtpVideoSender::DeliverRtcp 
ModuleRtpRtcpImpl::IncomingRtcpPacket(const uint8_t* rtcp_packet, const size_t length)
RTCPReceiver::IncomingPacket(const uint8_t* packet, size_t packet_size)  --> RTCPReceiver模块接受数据并读取数据格式
RTCPReceiver::TriggerCallbacksFromRtcpPacket(const PacketInformation& packet_information)
RtpTransportControllerSend::OnTransportFeedback(const rtcp::TransportFeedback& feedback)
```



### 1.3 线程切换 gcc

```less
task_queue_.PostTask([this, feedback, feedback_time]() 
  
GoogCcNetworkController::OnTransportPacketsFeedback(TransportPacketsFeedback report)
```



## 2. PhysicalSocketServer::WaitSelect

rtc_base/physical_socket_server.cc

```cpp
bool PhysicalSocketServer::WaitSelect(int cmsWait, bool process_io) {
  // Calculate timing information

  struct timeval* ptvWait = nullptr;
  struct timeval tvWait;
  int64_t stop_us;
  if (cmsWait != kForever) {
    // Calculate wait timeval
    tvWait.tv_sec = cmsWait / 1000;
    tvWait.tv_usec = (cmsWait % 1000) * 1000;
    ptvWait = &tvWait;

    // Calculate when to return
    stop_us = rtc::TimeMicros() + cmsWait * 1000;
  }


  fd_set fdsRead;
  fd_set fdsWrite;
// Explicitly unpoison these FDs on MemorySanitizer which doesn't handle the
// inline assembly in FD_ZERO.
// http://crbug.com/344505
#ifdef MEMORY_SANITIZER
  __msan_unpoison(&fdsRead, sizeof(fdsRead));
  __msan_unpoison(&fdsWrite, sizeof(fdsWrite));
#endif

  fWait_ = true;

  while (fWait_) {
    // Zero all fd_sets. Although select() zeros the descriptors not signaled,
    // we may need to do this for dispatchers that were deleted while
    // iterating.
    FD_ZERO(&fdsRead);
    FD_ZERO(&fdsWrite);
    int fdmax = -1;
    {
      CritScope cr(&crit_);
      current_dispatcher_keys_.clear();
      for (auto const& kv : dispatcher_by_key_) {
        uint64_t key = kv.first;
        Dispatcher* pdispatcher = kv.second;
        // Query dispatchers for read and write wait state
        if (!process_io && (pdispatcher != signal_wakeup_))
          continue;
        current_dispatcher_keys_.push_back(key);
        int fd = pdispatcher->GetDescriptor();
        // "select"ing a file descriptor that is equal to or larger than
        // FD_SETSIZE will result in undefined behavior.
        RTC_DCHECK_LT(fd, FD_SETSIZE);
        if (fd > fdmax)
          fdmax = fd;

        uint32_t ff = pdispatcher->GetRequestedEvents();
        if (ff & (DE_READ | DE_ACCEPT))
          FD_SET(fd, &fdsRead);
        if (ff & (DE_WRITE | DE_CONNECT))
          FD_SET(fd, &fdsWrite);
      }
    }

    // Wait then call handlers as appropriate
    // < 0 means error
    // 0 means timeout
    // > 0 means count of descriptors ready
    int n = select(fdmax + 1, &fdsRead, &fdsWrite, nullptr, ptvWait);

    // If error, return error.
    if (n < 0) {
      if (errno != EINTR) {
        RTC_LOG_E(LS_ERROR, EN, errno) << "select";
        return false;
      }
      // Else ignore the error and keep going. If this EINTR was for one of the
      // signals managed by this PhysicalSocketServer, the
      // PosixSignalDeliveryDispatcher will be in the signaled state in the next
      // iteration.
    } else if (n == 0) {
      // If timeout, return success
      return true;
    } else {
      // We have signaled descriptors
      CritScope cr(&crit_);
      // Iterate only on the dispatchers whose sockets were passed into
      // WSAEventSelect; this avoids the ABA problem (a socket being
      // destroyed and a new one created with the same file descriptor).
      for (uint64_t key : current_dispatcher_keys_) {
        if (!dispatcher_by_key_.count(key))
          continue;
        Dispatcher* pdispatcher = dispatcher_by_key_.at(key);

        int fd = pdispatcher->GetDescriptor();

        bool readable = FD_ISSET(fd, &fdsRead);
        if (readable) {
          FD_CLR(fd, &fdsRead);
        }

        bool writable = FD_ISSET(fd, &fdsWrite);
        if (writable) {
          FD_CLR(fd, &fdsWrite);
        }

        //！！！！！！！！！！
        //！！！！！！！！！！
        //！！！！！！！！！！
        // The error code can be signaled through reads or writes.
        ProcessEvents(pdispatcher, readable, writable, readable || writable);
      }
    }

    // Recalc the time remaining to wait. Doing it here means it doesn't get
    // calced twice the first time through the loop
    if (ptvWait) {
      ptvWait->tv_sec = 0;
      ptvWait->tv_usec = 0;
      int64_t time_left_us = stop_us - rtc::TimeMicros();
      if (time_left_us > 0) {
        ptvWait->tv_sec = time_left_us / rtc::kNumMicrosecsPerSec;
        ptvWait->tv_usec = time_left_us % rtc::kNumMicrosecsPerSec;
      }
    }
  }

  return true;
}
```



### 2.1 ProcessEvents

rtc_base/physical_socket_server.cc

```cpp
static void ProcessEvents(Dispatcher* dispatcher,
                          bool readable,
                          bool writable,
                          bool check_error) {
  int errcode = 0;
  // TODO(pthatcher): Should we set errcode if getsockopt fails?
  if (check_error) {
    socklen_t len = sizeof(errcode);
    ::getsockopt(dispatcher->GetDescriptor(), SOL_SOCKET, SO_ERROR, &errcode,
                 &len);
  }

  // Most often the socket is writable or readable or both, so make a single
  // virtual call to get requested events
  const uint32_t requested_events = dispatcher->GetRequestedEvents();
  uint32_t ff = 0;

  // Check readable descriptors. If we're waiting on an accept, signal
  // that. Otherwise we're waiting for data, check to see if we're
  // readable or really closed.
  // TODO(pthatcher): Only peek at TCP descriptors.
  if (readable) {
    if (requested_events & DE_ACCEPT) {
      ff |= DE_ACCEPT;
    } else if (errcode || dispatcher->IsDescriptorClosed()) {
      ff |= DE_CLOSE;
    } else {
      ff |= DE_READ;
    }
  }

  // Check writable descriptors. If we're waiting on a connect, detect
  // success versus failure by the reaped error code.
  if (writable) {
    if (requested_events & DE_CONNECT) {
      if (!errcode) {
        ff |= DE_CONNECT;
      } else {
        ff |= DE_CLOSE;
      }
    } else {
      ff |= DE_WRITE;
    }
  }

  // Tell the descriptor about the event.
  if (ff != 0) {
    dispatcher->OnPreEvent(ff);
    //！！！！！！！！！！
    //！！！！！！！！！！
    //！！！！！！！！！！
    // SocketDispatcher dispatcher
    dispatcher->OnEvent(ff, errcode);
  }
}
```



## 3. SocketDispatcher::OnEvent

rtc_base/physical_socket_server.cc

```less
Socket
AsyncSocket
PhysicalSocket
SocketDispatcher
```

通过 BasicPacketSocketFactory::CreateUdpSocket 创建了SocketDispatcher

```cpp
#if defined(WEBRTC_WIN)

void SocketDispatcher::OnEvent(uint32_t ff, int err) {
  int cache_id = id_;
  // Make sure we deliver connect/accept first. Otherwise, consumers may see
  // something like a READ followed by a CONNECT, which would be odd.
  if (((ff & DE_CONNECT) != 0) && (id_ == cache_id)) {
    if (ff != DE_CONNECT)
      RTC_LOG(LS_VERBOSE) << "Signalled with DE_CONNECT: " << ff;
    DisableEvents(DE_CONNECT);
#if !defined(NDEBUG)
    dbg_addr_ = "Connected @ ";
    dbg_addr_.append(GetRemoteAddress().ToString());
#endif
    SignalConnectEvent(this);
  }
  if (((ff & DE_ACCEPT) != 0) && (id_ == cache_id)) {
    DisableEvents(DE_ACCEPT);
    SignalReadEvent(this);
  }
  if ((ff & DE_READ) != 0) {
    DisableEvents(DE_READ);
    SignalReadEvent(this);
  }
  if (((ff & DE_WRITE) != 0) && (id_ == cache_id)) {
    DisableEvents(DE_WRITE);
    SignalWriteEvent(this);
  }
  if (((ff & DE_CLOSE) != 0) && (id_ == cache_id)) {
    signal_close_ = true;
    signal_err_ = err;
  }
}

#elif defined(WEBRTC_POSIX)

void SocketDispatcher::OnEvent(uint32_t ff, int err) {
#if defined(WEBRTC_USE_EPOLL)
  // Remember currently enabled events so we can combine multiple changes
  // into one update call later.
  // The signal handlers might re-enable events disabled here, so we can't
  // keep a list of events to disable at the end of the method. This list
  // would not be updated with the events enabled by the signal handlers.
  StartBatchedEventUpdates();
#endif
  // Make sure we deliver connect/accept first. Otherwise, consumers may see
  // something like a READ followed by a CONNECT, which would be odd.
  if ((ff & DE_CONNECT) != 0) {
    DisableEvents(DE_CONNECT);
    SignalConnectEvent(this);
  }
  if ((ff & DE_ACCEPT) != 0) {
    DisableEvents(DE_ACCEPT);
    // AsyncSocket::SignalReadEvent
    // 在UDPPort::Init 中注册，这样回调到 UDPPort
    SignalReadEvent(this);
  }
  if ((ff & DE_READ) != 0) {
    DisableEvents(DE_READ);
    SignalReadEvent(this);
  }
  if ((ff & DE_WRITE) != 0) {
    DisableEvents(DE_WRITE);
    SignalWriteEvent(this);
  }
  if ((ff & DE_CLOSE) != 0) {
    // The socket is now dead to us, so stop checking it.
    SetEnabledEvents(0);
    SignalCloseEvent(this, err);
  }
#if defined(WEBRTC_USE_EPOLL)
  FinishBatchedEventUpdates();
#endif
}

#endif  // WEBRTC_POSIX
```



### 3.1 AsyncSocket::SignalReadEvent

rtc_base/async_socket.h

消息发送到AsyncUDPSocket::AsyncUDPSocket。

```less
Socket
AsyncSocket
PhysicalSocket
SocketDispatcher
```



```cpp
class AsyncSocket : public Socket {
...
  // SignalReadEvent and SignalWriteEvent use multi_threaded_local to allow
  // access concurrently from different thread.
  // For example SignalReadEvent::connect will be called in AsyncUDPSocket ctor
  // but at the same time the SocketDispatcher maybe signaling the read event.
  // ready to read
  sigslot::signal1<AsyncSocket*, sigslot::multi_threaded_local> SignalReadEvent;
  // ready to write
  sigslot::signal1<AsyncSocket*, sigslot::multi_threaded_local>
      SignalWriteEvent;
  sigslot::signal1<AsyncSocket*> SignalConnectEvent;     // connected
  sigslot::signal2<AsyncSocket*, int> SignalCloseEvent;  // closed
}
```



## 4. AsyncUDPSocket::OnReadEvent

rtc_base/async_udp_socket.cc

```cpp
void AsyncUDPSocket::OnReadEvent(AsyncSocket* socket) {
  RTC_DCHECK(socket_.get() == socket);

  SocketAddress remote_addr;
  int64_t timestamp;
  int len = socket_->RecvFrom(buf_, size_, &remote_addr, &timestamp);
  if (len < 0) {
    // An error here typically means we got an ICMP error in response to our
    // send datagram, indicating the remote address was unreachable.
    // When doing ICE, this kind of thing will often happen.
    // TODO: Do something better like forwarding the error to the user.
    SocketAddress local_addr = socket_->GetLocalAddress();
    RTC_LOG(LS_INFO) << "AsyncUDPSocket[" << local_addr.ToSensitiveString()
                     << "] receive failed with error " << socket_->GetError();
    return;
  }

  //!!!!!!!!!
  //!!!!!!!!!
  //!!!!!!!!!
  // TODO: Make sure that we got all of the packet.
  // If we did not, then we should resize our buffer to be large enough.
  SignalReadPacket(this, buf_, static_cast<size_t>(len), remote_addr,
                   (timestamp > -1 ? timestamp : TimeMicros()));
}
```



### 4.1 AsyncPacketSocket::SignalReadPacket

rtc_base/async_packet_socket.h

```less
AsyncPacketSocket
AsyncUDPSocket
```



### 4.2 AsyncUDPSocket::AsyncUDPSocket 注册到SocketDispatcher--3.1 

rtc_base/async_udp_socket.cc

```
Socket
AsyncSocket
PhysicalSocket
SocketDispatcher
```

这里是用来接收 从 SocketDispatcher 发送到消息。

```cpp
AsyncUDPSocket::AsyncUDPSocket(AsyncSocket* socket) : socket_(socket) {
  size_ = BUF_SIZE;
  buf_ = new char[size_];

  // The socket should start out readable but not writable.
  // AsyncSocket socket_, 子类SocketDispatcher
  socket_->SignalReadEvent.connect(this, &AsyncUDPSocket::OnReadEvent);
  socket_->SignalWriteEvent.connect(this, &AsyncUDPSocket::OnWriteEvent);
}
```





## 5. AllocationSequence::OnReadPacket

p2p/client/basic_port_allocator.cc

```cpp
void AllocationSequence::OnReadPacket(rtc::AsyncPacketSocket* socket,
                                      const char* data,
                                      size_t size,
                                      const rtc::SocketAddress& remote_addr,
                                      const int64_t& packet_time_us) {
  RTC_DCHECK(socket == udp_socket_.get());

  bool turn_port_found = false;

  // Try to find the TurnPort that matches the remote address. Note that the
  // message could be a STUN binding response if the TURN server is also used as
  // a STUN server. We don't want to parse every message here to check if it is
  // a STUN binding response, so we pass the message to TurnPort regardless of
  // the message type. The TurnPort will just ignore the message since it will
  // not find any request by transaction ID.
  for (auto* port : relay_ports_) {
    if (port->CanHandleIncomingPacketsFrom(remote_addr)) {
      if (port->HandleIncomingPacket(socket, data, size, remote_addr,
                                     packet_time_us)) {
        return;
      }
      turn_port_found = true;
    }
  }

  if (udp_port_) {
    const ServerAddresses& stun_servers = udp_port_->server_addresses();

    // Pass the packet to the UdpPort if there is no matching TurnPort, or if
    // the TURN server is also a STUN server.
    if (!turn_port_found ||
        stun_servers.find(remote_addr) != stun_servers.end()) {
      RTC_DCHECK(udp_port_->SharedSocket());
      //!!!!!!!!!
      //!!!!!!!!!
      //!!!!!!!!!
      udp_port_->HandleIncomingPacket(socket, data, size, remote_addr,
                                      packet_time_us);
    }
  }
}
```



### 5.1 AllocationSequence::Init 注册到 AsyncUDPSocket--4.1 

p2p/client/basic_port_allocator.cc

这里是用来接收 从 AsyncUDPSocket 发送到消息。

```cpp
void AllocationSequence::Init() {
  if (IsFlagSet(PORTALLOCATOR_ENABLE_SHARED_SOCKET)) {
    // rtc::AsyncPacketSocket， 子类 AsyncUDPSocket
    udp_socket_.reset(session_->socket_factory()->CreateUdpSocket(
        rtc::SocketAddress(network_->GetBestIP(), 0),
        session_->allocator()->min_port(), session_->allocator()->max_port(),
        session_->allocator()->proxy()));
    if (udp_socket_) {
      udp_socket_->SignalReadPacket.connect(this,
                                            &AllocationSequence::OnReadPacket);
    }
    // Continuing if |udp_socket_| is NULL, as local TCP and RelayPort using TCP
    // are next available options to setup a communication channel.
  }
}
```



## 6. UDPPort::HandleIncomingPacket

p2p/base/stun_port.cc

```cpp
bool UDPPort::HandleIncomingPacket(rtc::AsyncPacketSocket* socket,
                                   const char* data,
                                   size_t size,
                                   const rtc::SocketAddress& remote_addr,
                                   int64_t packet_time_us) {
  // All packets given to UDP port will be consumed.
  //!!!!!!!!!
  //!!!!!!!!!
  //!!!!!!!!!
  OnReadPacket(socket, data, size, remote_addr, packet_time_us);
  return true;
}
```



## 7. UDPPort::OnReadPacket

p2p/base/stun_port.cc

```cpp
void UDPPort::OnReadPacket(rtc::AsyncPacketSocket* socket,
                           const char* data,
                           size_t size,
                           const rtc::SocketAddress& remote_addr,
                           const int64_t& packet_time_us) {
  RTC_DCHECK(socket == socket_);
  RTC_DCHECK(!remote_addr.IsUnresolvedIP());

  // Look for a response from the STUN server.
  // Even if the response doesn't match one of our outstanding requests, we
  // will eat it because it might be a response to a retransmitted packet, and
  // we already cleared the request when we got the first response.
  if (server_addresses_.find(remote_addr) != server_addresses_.end()) {
    requests_.CheckResponse(data, size);
    return;
  }

  if (Connection* conn = GetConnection(remote_addr)) {
      //!!!!!!!!!
      //!!!!!!!!!
      //!!!!!!!!!
    // Connection
    conn->OnReadPacket(data, size, packet_time_us);
  } else {
    Port::OnReadPacket(data, size, remote_addr, PROTO_UDP);
  }
}
```



### 7.1 UDPPort::Init() 注册

p2p/base/stun_port.cc

```
Socket
AsyncSocket
PhysicalSocket
SocketDispatcher
```



````cpp
bool UDPPort::Init() {
  stun_keepalive_lifetime_ = GetStunKeepaliveLifetime();
  if (!SharedSocket()) {
    RTC_DCHECK(socket_ == nullptr);
    socket_ = socket_factory()->CreateUdpSocket(
        rtc::SocketAddress(Network()->GetBestIP(), 0), min_port(), max_port());
    if (!socket_) {
      RTC_LOG(LS_WARNING) << ToString() << ": UDP socket creation failed";
      return false;
    }
    socket_->SignalReadPacket.connect(this, &UDPPort::OnReadPacket);
  }
  // 
  socket_->SignalSentPacket.connect(this, &UDPPort::OnSentPacket);
  socket_->SignalReadyToSend.connect(this, &UDPPort::OnReadyToSend);
  socket_->SignalAddressReady.connect(this, &UDPPort::OnLocalAddressReady);
  requests_.SignalSendPacket.connect(this, &UDPPort::OnSendPacket);
  return true;
}
````



## 8. Connection::OnReadPacket

p2p/base/connection.cc

```cpp
void Connection::OnReadPacket(const char* data,
                              size_t size,
                              int64_t packet_time_us) {
  std::unique_ptr<IceMessage> msg;
  std::string remote_ufrag;
  const rtc::SocketAddress& addr(remote_candidate_.address());
  if (!port_->GetStunMessage(data, size, addr, &msg, &remote_ufrag)) {
    // The packet did not parse as a valid STUN message
    // This is a data packet, pass it along.
    last_data_received_ = rtc::TimeMillis();
    UpdateReceiving(last_data_received_);
    recv_rate_tracker_.AddSamples(size);
    stats_.packets_received++;
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
    SignalReadPacket(this, data, size, packet_time_us);

    // If timed out sending writability checks, start up again
    if (!pruned_ && (write_state_ == STATE_WRITE_TIMEOUT)) {
      RTC_LOG(LS_WARNING)
          << "Received a data packet on a timed-out Connection. "
             "Resetting state to STATE_WRITE_INIT.";
      set_write_state(STATE_WRITE_INIT);
    }
  } else if (!msg) {
    // The packet was STUN, but failed a check and was handled internally.
  } else {
    // The packet is STUN and passed the Port checks.
    // Perform our own checks to ensure this packet is valid.
    // If this is a STUN request, then update the receiving bit and respond.
    // If this is a STUN response, then update the writable bit.
    // Log at LS_INFO if we receive a ping on an unwritable connection.
    rtc::LoggingSeverity sev = (!writable() ? rtc::LS_INFO : rtc::LS_VERBOSE);
    switch (msg->type()) {
      case STUN_BINDING_REQUEST:
        RTC_LOG_V(sev) << ToString() << ": Received "
                       << StunMethodToString(msg->type())
                       << ", id=" << rtc::hex_encode(msg->transaction_id());
        if (remote_ufrag == remote_candidate_.username()) {
          HandleStunBindingOrGoogPingRequest(msg.get());
        } else {
          // The packet had the right local username, but the remote username
          // was not the right one for the remote address.
          RTC_LOG(LS_ERROR)
              << ToString()
              << ": Received STUN request with bad remote username "
              << remote_ufrag;
          port_->SendBindingErrorResponse(msg.get(), addr,
                                          STUN_ERROR_UNAUTHORIZED,
                                          STUN_ERROR_REASON_UNAUTHORIZED);
        }
        break;

      // Response from remote peer. Does it match request sent?
      // This doesn't just check, it makes callbacks if transaction
      // id's match.
      case STUN_BINDING_RESPONSE:
      case STUN_BINDING_ERROR_RESPONSE:
        if (msg->ValidateMessageIntegrity(data, size,
                                          remote_candidate().password())) {
          requests_.CheckResponse(msg.get());
        }
        // Otherwise silently discard the response message.
        break;

      // Remote end point sent an STUN indication instead of regular binding
      // request. In this case |last_ping_received_| will be updated but no
      // response will be sent.
      case STUN_BINDING_INDICATION:
        ReceivedPing(msg->transaction_id());
        break;
      case GOOG_PING_REQUEST:
        HandleStunBindingOrGoogPingRequest(msg.get());
        break;
      case GOOG_PING_RESPONSE:
      case GOOG_PING_ERROR_RESPONSE:
        if (msg->ValidateMessageIntegrity32(data, size,
                                            remote_candidate().password())) {
          requests_.CheckResponse(msg.get());
        }
        break;
      default:
        RTC_NOTREACHED();
        break;
    }
  }
}
```



### 8.1 Connection::SignalReadPacket

p2p/base/connection.h

```
  sigslot::signal4<Connection*, const char*, size_t, int64_t> SignalReadPacket;

```



## 9. P2PTransportChannel::OnReadPacket

p2p/base/p2p_transport_channel.cc

```less
PacketTransportInternal
IceTransportInternal
P2PTransportChannel
```



```cpp
// We data is available, let listeners know
void P2PTransportChannel::OnReadPacket(Connection* connection,
                                       const char* data,
                                       size_t len,
                                       int64_t packet_time_us) {
  RTC_DCHECK_RUN_ON(network_thread_);

  if (connection == selected_connection_) {
    // Let the client know of an incoming packet
    RTC_DCHECK(connection->last_data_received() >= last_data_received_ms_);
    last_data_received_ms_ =
        std::max(last_data_received_ms_, connection->last_data_received());
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
    SignalReadPacket(this, data, len, packet_time_us, 0);
    return;
  }

  // Do not deliver, if packet doesn't belong to the correct transport channel.
  if (!FindConnection(connection))
    return;

  RTC_DCHECK(connection->last_data_received() >= last_data_received_ms_);
  last_data_received_ms_ =
      std::max(last_data_received_ms_, connection->last_data_received());

  //!!!!!!!!!
  //!!!!!!!!!
  //!!!!!!!!!
  // Let the client know of an incoming packet
  SignalReadPacket(this, data, len, packet_time_us, 0);

  // May need to switch the sending connection based on the receiving media path
  // if this is the controlled side.
  if (ice_role_ == ICEROLE_CONTROLLED) {
    MaybeSwitchSelectedConnection(connection,
                                  IceControllerEvent::DATA_RECEIVED);
  }
}
```



### 9.1 PacketTransportInternal.SignalReadPacket

p2p/base/packet_transport_internal.h

```cpp
class RTC_EXPORT PacketTransportInternal : public sigslot::has_slots<> {
...  
// Signalled each time a packet is received on this channel.
  sigslot::signal5<PacketTransportInternal*,
                   const char*,
                   size_t,
                   // TODO(bugs.webrtc.org/9584): Change to passing the int64_t
                   // timestamp by value.
                   const int64_t&,
                   int>
      SignalReadPacket;
  ...
}
```





### 9.2 P2PTransportChannel::AddConnection 注册到Connection --8.1

p2p/base/p2p_transport_channel.cc

注册了connection->SignalReadPacket。

```cpp
void P2PTransportChannel::AddConnection(Connection* connection) {
  RTC_DCHECK_RUN_ON(network_thread_);
  connection->set_remote_ice_mode(remote_ice_mode_);
  connection->set_receiving_timeout(config_.receiving_timeout);
  connection->set_unwritable_timeout(config_.ice_unwritable_timeout);
  connection->set_unwritable_min_checks(config_.ice_unwritable_min_checks);
  connection->set_inactive_timeout(config_.ice_inactive_timeout);
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  connection->SignalReadPacket.connect(this,
                                       &P2PTransportChannel::OnReadPacket);
  connection->SignalReadyToSend.connect(this,
                                        &P2PTransportChannel::OnReadyToSend);
  connection->SignalStateChange.connect(
      this, &P2PTransportChannel::OnConnectionStateChange);
  connection->SignalDestroyed.connect(
      this, &P2PTransportChannel::OnConnectionDestroyed);
  connection->SignalNominated.connect(this, &P2PTransportChannel::OnNominated);

  had_connection_ = true;

  connection->set_ice_event_log(&ice_event_log_);
  connection->SetIceFieldTrials(&field_trials_);
  LogCandidatePairConfig(connection,
                         webrtc::IceCandidatePairConfigType::kAdded);

  ice_controller_->AddConnection(connection);
}
```



## 10. DtlsTransport::OnReadPacket

p2p/base/dtls_transport.cc

```less
rtc::PacketTransportInternal
DtlsTransportInternal
DtlsTransport
```

这消息是从9 发过来。

```cpp
void DtlsTransport::OnReadPacket(rtc::PacketTransportInternal* transport,
                                 const char* data,
                                 size_t size,
                                 const int64_t& packet_time_us,
                                 int flags) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  RTC_DCHECK(transport == ice_transport_);
  RTC_DCHECK(flags == 0);

  if (!dtls_active_) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
    // Not doing DTLS.
    SignalReadPacket(this, data, size, packet_time_us, 0);
    return;
  }

  switch (dtls_state()) {
    case DTLS_TRANSPORT_NEW:
      if (dtls_) {
        RTC_LOG(LS_INFO) << ToString()
                         << ": Packet received before DTLS started.";
      } else {
        RTC_LOG(LS_WARNING) << ToString()
                            << ": Packet received before we know if we are "
                               "doing DTLS or not.";
      }
      // Cache a client hello packet received before DTLS has actually started.
      if (IsDtlsClientHelloPacket(data, size)) {
        RTC_LOG(LS_INFO) << ToString()
                         << ": Caching DTLS ClientHello packet until DTLS is "
                            "started.";
        cached_client_hello_.SetData(data, size);
        // If we haven't started setting up DTLS yet (because we don't have a
        // remote fingerprint/role), we can use the client hello as a clue that
        // the peer has chosen the client role, and proceed with the handshake.
        // The fingerprint will be verified when it's set.
        if (!dtls_ && local_certificate_) {
          SetDtlsRole(rtc::SSL_SERVER);
          SetupDtls();
        }
      } else {
        RTC_LOG(LS_INFO) << ToString()
                         << ": Not a DTLS ClientHello packet; dropping.";
      }
      break;

    case DTLS_TRANSPORT_CONNECTING:
    case DTLS_TRANSPORT_CONNECTED:
      // We should only get DTLS or SRTP packets; STUN's already been demuxed.
      // Is this potentially a DTLS packet?
      if (IsDtlsPacket(data, size)) {
        if (!HandleDtlsPacket(data, size)) {
          RTC_LOG(LS_ERROR) << ToString() << ": Failed to handle DTLS packet.";
          return;
        }
      } else {
        // Not a DTLS packet; our handshake should be complete by now.
        if (dtls_state() != DTLS_TRANSPORT_CONNECTED) {
          RTC_LOG(LS_ERROR) << ToString()
                            << ": Received non-DTLS packet before DTLS "
                               "complete.";
          return;
        }

        // And it had better be a SRTP packet.
        if (!IsRtpPacket(data, size)) {
          RTC_LOG(LS_ERROR)
              << ToString() << ": Received unexpected non-DTLS packet.";
          return;
        }

        // Sanity check.
        RTC_DCHECK(!srtp_ciphers_.empty());

        //!!!!!!!!!
        //!!!!!!!!!
        //!!!!!!!!!
        // Signal this upwards as a bypass packet.
        SignalReadPacket(this, data, size, packet_time_us, PF_SRTP_BYPASS);
      }
      break;
    case DTLS_TRANSPORT_FAILED:
    case DTLS_TRANSPORT_CLOSED:
      // This shouldn't be happening. Drop the packet.
      break;
  }
}
```



### 10.1 DtlsTransport::SignalReadPacket

p2p/base/packet_transport_internal.h

```less
rtc::PacketTransportInternal
DtlsTransportInternal
DtlsTransport
```



```cpp
  // Signalled each time a packet is received on this channel.
  sigslot::signal5<PacketTransportInternal*,
                   const char*,
                   size_t,
                   // TODO(bugs.webrtc.org/9584): Change to passing the int64_t
                   // timestamp by value.
                   const int64_t&,
                   int>
      SignalReadPacket;
```



### 10.2 DtlsTransport::ConnectToIceTransport 注册到 P2PTransportChannel

p2p/base/dtls_transport.cc

````cpp
void DtlsTransport::ConnectToIceTransport() {
  RTC_DCHECK(ice_transport_);
  ice_transport_->SignalWritableState.connect(this,
                                              &DtlsTransport::OnWritableState);
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  ice_transport_->SignalReadPacket.connect(this, &DtlsTransport::OnReadPacket);
  ice_transport_->SignalSentPacket.connect(this, &DtlsTransport::OnSentPacket);
  ice_transport_->SignalReadyToSend.connect(this,
                                            &DtlsTransport::OnReadyToSend);
  ice_transport_->SignalReceivingState.connect(
      this, &DtlsTransport::OnReceivingState);
  ice_transport_->SignalNetworkRouteChanged.connect(
      this, &DtlsTransport::OnNetworkRouteChanged);
}
````





## 11. RtpTransport::OnReadPacket

pc/rtp_transport.cc

```cpp
void RtpTransport::OnReadPacket(rtc::PacketTransportInternal* transport,
                                const char* data,
                                size_t len,
                                const int64_t& packet_time_us,
                                int flags) {
  TRACE_EVENT0("webrtc", "RtpTransport::OnReadPacket");

  // When using RTCP multiplexing we might get RTCP packets on the RTP
  // transport. We check the RTP payload type to determine if it is RTCP.
  auto array_view = rtc::MakeArrayView(data, len);
  cricket::RtpPacketType packet_type = cricket::InferRtpPacketType(array_view);
  // Filter out the packet that is neither RTP nor RTCP.
  if (packet_type == cricket::RtpPacketType::kUnknown) {
    return;
  }

  // Protect ourselves against crazy data.
  if (!cricket::IsValidRtpPacketSize(packet_type, len)) {
    RTC_LOG(LS_ERROR) << "Dropping incoming "
                      << cricket::RtpPacketTypeToString(packet_type)
                      << " packet: wrong size=" << len;
    return;
  }

  rtc::CopyOnWriteBuffer packet(data, len);
  if (packet_type == cricket::RtpPacketType::kRtcp) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
    OnRtcpPacketReceived(std::move(packet), packet_time_us);
  } else {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
    OnRtpPacketReceived(std::move(packet), packet_time_us);
  }
}
```



### 11.1 RtpTransport::SetRtpPacketTransport--注册到DtlsTransport

pc/rtp_transport.cc

```less
rtc::PacketTransportInternal
DtlsTransportInternal
DtlsTransport
```



```cpp
void RtpTransport::SetRtpPacketTransport(
    rtc::PacketTransportInternal* new_packet_transport) {
  if (new_packet_transport == rtp_packet_transport_) {
    return;
  }
  if (rtp_packet_transport_) {
    rtp_packet_transport_->SignalReadyToSend.disconnect(this);
    rtp_packet_transport_->SignalReadPacket.disconnect(this);
    rtp_packet_transport_->SignalNetworkRouteChanged.disconnect(this);
    rtp_packet_transport_->SignalWritableState.disconnect(this);
    rtp_packet_transport_->SignalSentPacket.disconnect(this);
    // Reset the network route of the old transport.
    SignalNetworkRouteChanged(absl::optional<rtc::NetworkRoute>());
  }
  if (new_packet_transport) {
    new_packet_transport->SignalReadyToSend.connect(
        this, &RtpTransport::OnReadyToSend);
    //!!!!!!!!!!!!
    //!!!!!!!!!!!!
    //!!!!!!!!!!!!
    // DtlsTransport new_packet_transport， 父类rtc::PacketTransportInternal, DtlsTransportInternal
    new_packet_transport->SignalReadPacket.connect(this,
                                                   &RtpTransport::OnReadPacket);
    new_packet_transport->SignalNetworkRouteChanged.connect(
        this, &RtpTransport::OnNetworkRouteChanged);
    new_packet_transport->SignalWritableState.connect(
        this, &RtpTransport::OnWritableState);
    new_packet_transport->SignalSentPacket.connect(this,
                                                   &RtpTransport::OnSentPacket);
    // Set the network route for the new transport.
    SignalNetworkRouteChanged(new_packet_transport->network_route());
  }

  rtp_packet_transport_ = new_packet_transport;
  // Assumes the transport is ready to send if it is writable. If we are wrong,
  // ready to send will be updated the next time we try to send.
  SetReadyToSend(false,
                 rtp_packet_transport_ && rtp_packet_transport_->writable());
}
```





## 12. SrtpTransport::OnRtpPacketReceived

pc/srtp_transport.cc

```cpp
void SrtpTransport::OnRtpPacketReceived(rtc::CopyOnWriteBuffer packet,
                                        int64_t packet_time_us) {
  if (!IsSrtpActive()) {
    RTC_LOG(LS_WARNING)
        << "Inactive SRTP transport received an RTP packet. Drop it.";
    return;
  }
  TRACE_EVENT0("webrtc", "SRTP Decode");
  char* data = packet.MutableData<char>();
  int len = rtc::checked_cast<int>(packet.size());
  if (!UnprotectRtp(data, len, &len)) {
    int seq_num = -1;
    uint32_t ssrc = 0;
    cricket::GetRtpSeqNum(data, len, &seq_num);
    cricket::GetRtpSsrc(data, len, &ssrc);

    // Limit the error logging to avoid excessive logs when there are lots of
    // bad packets.
    const int kFailureLogThrottleCount = 100;
    if (decryption_failure_count_ % kFailureLogThrottleCount == 0) {
      RTC_LOG(LS_ERROR) << "Failed to unprotect RTP packet: size=" << len
                        << ", seqnum=" << seq_num << ", SSRC=" << ssrc
                        << ", previous failure count: "
                        << decryption_failure_count_;
    }
    ++decryption_failure_count_;
    return;
  }
  packet.SetSize(len);
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  DemuxPacket(std::move(packet), packet_time_us);
}
```



### 12.1 RtpTransport::DemuxPacket

pc/rtp_transport.cc

```cpp
void RtpTransport::DemuxPacket(rtc::CopyOnWriteBuffer packet,
                               int64_t packet_time_us) {
  webrtc::RtpPacketReceived parsed_packet(&header_extension_map_);
  if (!parsed_packet.Parse(std::move(packet))) {
    RTC_LOG(LS_ERROR)
        << "Failed to parse the incoming RTP packet before demuxing. Drop it.";
    return;
  }

  if (packet_time_us != -1) {
    parsed_packet.set_arrival_time_ms((packet_time_us + 500) / 1000);
  }
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
	// RtpDemuxer rtp_demuxer_
  if (!rtp_demuxer_.OnRtpPacket(parsed_packet)) {
    RTC_LOG(LS_WARNING) << "Failed to demux RTP packet: "
                        << RtpDemuxer::DescribePacket(parsed_packet);
  }
}
```



## 13. RtpDemuxer::OnRtpPacket

```cpp
bool RtpDemuxer::OnRtpPacket(const RtpPacketReceived& packet) {
  RtpPacketSinkInterface* sink = ResolveSink(packet);
  if (sink != nullptr) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
    // BaseChannel
    sink->OnRtpPacket(packet);
    return true;
  }
  return false;
}
```



## 14. BaseChannel::OnRtpPacket

pc/channel.cc

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
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
        media_channel_->OnPacketReceived(packet_buffer, packet_time_us);
      });
}
```



## ------

## 15. WebRtcVideoChannel::OnPacketReceived

media/engine/webrtc_video_engine.cc

```cpp
void WebRtcVideoChannel::OnPacketReceived(rtc::CopyOnWriteBuffer packet,
                                          int64_t packet_time_us) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  const webrtc::PacketReceiver::DeliveryStatus delivery_result =
      call_->Receiver()->DeliverPacket(webrtc::MediaType::VIDEO, packet,
                                       packet_time_us);
  switch (delivery_result) {
    case webrtc::PacketReceiver::DELIVERY_OK:
      return;
    case webrtc::PacketReceiver::DELIVERY_PACKET_ERROR:
      return;
    case webrtc::PacketReceiver::DELIVERY_UNKNOWN_SSRC:
      break;
  }

  uint32_t ssrc = 0;
  if (!GetRtpSsrc(packet.cdata(), packet.size(), &ssrc)) {
    return;
  }

  if (unknown_ssrc_packet_buffer_) {
    unknown_ssrc_packet_buffer_->AddPacket(ssrc, packet_time_us, packet);
    return;
  }

  if (discard_unknown_ssrc_packets_) {
    return;
  }

  int payload_type = 0;
  if (!GetRtpPayloadType(packet.cdata(), packet.size(), &payload_type)) {
    return;
  }

  // See if this payload_type is registered as one that usually gets its own
  // SSRC (RTX) or at least is safe to drop either way (FEC). If it is, and
  // it wasn't handled above by DeliverPacket, that means we don't know what
  // stream it associates with, and we shouldn't ever create an implicit channel
  // for these.
  for (auto& codec : recv_codecs_) {
    if (payload_type == codec.rtx_payload_type ||
        payload_type == codec.ulpfec.red_rtx_payload_type ||
        payload_type == codec.ulpfec.ulpfec_payload_type) {
      return;
    }
  }
  if (payload_type == recv_flexfec_payload_type_) {
    return;
  }

  switch (unsignalled_ssrc_handler_->OnUnsignalledSsrc(this, ssrc)) {
    case UnsignalledSsrcHandler::kDropPacket:
      return;
    case UnsignalledSsrcHandler::kDeliverPacket:
      break;
  }

    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  // Call call_
  // call_->Receiver() 返回的就是Call
  if (call_->Receiver()->DeliverPacket(webrtc::MediaType::VIDEO, packet,
                                       packet_time_us) !=
      webrtc::PacketReceiver::DELIVERY_OK) {
    RTC_LOG(LS_WARNING) << "Failed to deliver RTP packet on re-delivery.";
    return;
  }
}
```



## 16. Call::DeliverPacket

call/call.cc

```cpp
PacketReceiver::DeliveryStatus Call::DeliverPacket(
    MediaType media_type,
    rtc::CopyOnWriteBuffer packet,
    int64_t packet_time_us) {
  RTC_DCHECK_RUN_ON(worker_thread_);

    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  if (IsRtcp(packet.cdata(), packet.size()))
    return DeliverRtcp(media_type, packet.cdata(), packet.size());

  return DeliverRtp(media_type, std::move(packet), packet_time_us);
}
```



## 17. Call::DeliverRtcp

call/call.cc

```cpp
PacketReceiver::DeliveryStatus Call::DeliverRtcp(MediaType media_type,
                                                 const uint8_t* packet,
                                                 size_t length) {
  TRACE_EVENT0("webrtc", "Call::DeliverRtcp");
  // TODO(pbos): Make sure it's a valid packet.
  //             Return DELIVERY_UNKNOWN_SSRC if it can be determined that
  //             there's no receiver of the packet.
  if (received_bytes_per_second_counter_.HasSample()) {
    // First RTP packet has been received.
    received_bytes_per_second_counter_.Add(static_cast<int>(length));
    received_rtcp_bytes_per_second_counter_.Add(static_cast<int>(length));
  }
  bool rtcp_delivered = false;
  if (media_type == MediaType::ANY || media_type == MediaType::VIDEO) {
    for (VideoReceiveStream2* stream : video_receive_streams_) {
      if (stream->DeliverRtcp(packet, length))
        rtcp_delivered = true;
    }
  }
  if (media_type == MediaType::ANY || media_type == MediaType::AUDIO) {
    for (AudioReceiveStream* stream : audio_receive_streams_) {
      stream->DeliverRtcp(packet, length);
      rtcp_delivered = true;
    }
  }
  if (media_type == MediaType::ANY || media_type == MediaType::VIDEO) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
    for (VideoSendStream* stream : video_send_streams_) {
      stream->DeliverRtcp(packet, length);
      rtcp_delivered = true;
    }
  }
  if (media_type == MediaType::ANY || media_type == MediaType::AUDIO) {
    for (auto& kv : audio_send_ssrcs_) {
      kv.second->DeliverRtcp(packet, length);
      rtcp_delivered = true;
    }
  }

  if (rtcp_delivered) {
    event_log_->Log(std::make_unique<RtcEventRtcpPacketIncoming>(
        rtc::MakeArrayView(packet, length)));
  }

  return rtcp_delivered ? DELIVERY_OK : DELIVERY_PACKET_ERROR;
}
```



## 18. VideoSendStream::DeliverRtcp

video/video_send_stream.cc

```cpp
void VideoSendStream::DeliverRtcp(const uint8_t* packet, size_t length) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  // Called on a network thread.
  send_stream_->DeliverRtcp(packet, length);
}
```



## 19. VideoSendStreamImpl::DeliverRtcp

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::DeliverRtcp(const uint8_t* packet, size_t length) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  // Runs on a network thread.
  RTC_DCHECK(!worker_queue_->IsCurrent());
  rtp_video_sender_->DeliverRtcp(packet, length);
}
```



## 20. RtpVideoSender::DeliverRtcp

call/rtp_video_sender.cc

```cpp
void RtpVideoSender::DeliverRtcp(const uint8_t* packet, size_t length) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  // Runs on a network thread.
  for (const RtpStreamSender& stream : rtp_streams_)
    stream.rtp_rtcp->IncomingRtcpPacket(packet, length);
}
```



## 21. ModuleRtpRtcpImpl2::IncomingRtcpPacket

modules/rtp_rtcp/source/rtp_rtcp_impl2.cc

```cpp
void ModuleRtpRtcpImpl2::IncomingRtcpPacket(const uint8_t* rtcp_packet,
                                            const size_t length) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  rtcp_receiver_.IncomingPacket(rtcp_packet, length);
}
```



## 22. RTCPReceiver::IncomingPacket

modules/rtp_rtcp/source/rtcp_receiver.cc

```cpp
void RTCPReceiver::IncomingPacket(rtc::ArrayView<const uint8_t> packet) {
  if (packet.empty()) {
    RTC_LOG(LS_WARNING) << "Incoming empty RTCP packet";
    return;
  }

  PacketInformation packet_information;
  if (!ParseCompoundPacket(packet, &packet_information))
    return;
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
  TriggerCallbacksFromRtcpPacket(packet_information);
}
```



## 23. RTCPReceiver::TriggerCallbacksFromRtcpPacket

modules/rtp_rtcp/source/rtcp_receiver.cc

```cpp
// Holding no Critical section.
void RTCPReceiver::TriggerCallbacksFromRtcpPacket(
    const PacketInformation& packet_information) {
  // Process TMMBR and REMB first to avoid multiple callbacks
  // to OnNetworkChanged.
  if (packet_information.packet_type_flags & kRtcpTmmbr) {
    // Might trigger a OnReceivedBandwidthEstimateUpdate.
    NotifyTmmbrUpdated();
  }
  uint32_t local_ssrc;
  std::set<uint32_t> registered_ssrcs;
  {
    // We don't want to hold this critsect when triggering the callbacks below.
    MutexLock lock(&rtcp_receiver_lock_);
    local_ssrc = main_ssrc_;
    registered_ssrcs = registered_ssrcs_;
  }
  if (!receiver_only_ && (packet_information.packet_type_flags & kRtcpSrReq)) {
    rtp_rtcp_->OnRequestSendReport();
  }
  if (!receiver_only_ && (packet_information.packet_type_flags & kRtcpNack)) {
    if (!packet_information.nack_sequence_numbers.empty()) {
      RTC_LOG(LS_VERBOSE) << "Incoming NACK length: "
                          << packet_information.nack_sequence_numbers.size();
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
      rtp_rtcp_->OnReceivedNack(packet_information.nack_sequence_numbers);
    }
  }

  // We need feedback that we have received a report block(s) so that we
  // can generate a new packet in a conference relay scenario, one received
  // report can generate several RTCP packets, based on number relayed/mixed
  // a send report block should go out to all receivers.
  if (rtcp_intra_frame_observer_) {
    RTC_DCHECK(!receiver_only_);
    if ((packet_information.packet_type_flags & kRtcpPli) ||
        (packet_information.packet_type_flags & kRtcpFir)) {
      if (packet_information.packet_type_flags & kRtcpPli) {
        RTC_LOG(LS_VERBOSE)
            << "Incoming PLI from SSRC " << packet_information.remote_ssrc;
      } else {
        RTC_LOG(LS_VERBOSE)
            << "Incoming FIR from SSRC " << packet_information.remote_ssrc;
      }
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
      rtcp_intra_frame_observer_->OnReceivedIntraFrameRequest(local_ssrc);
    }
  }
  if (rtcp_loss_notification_observer_ &&
      (packet_information.packet_type_flags & kRtcpLossNotification)) {
    rtcp::LossNotification* loss_notification =
        packet_information.loss_notification.get();
    RTC_DCHECK(loss_notification);
    if (loss_notification->media_ssrc() == local_ssrc) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
      rtcp_loss_notification_observer_->OnReceivedLossNotification(
          loss_notification->media_ssrc(), loss_notification->last_decoded(),
          loss_notification->last_received(),
          loss_notification->decodability_flag());
    }
  }
  if (rtcp_bandwidth_observer_) {
    RTC_DCHECK(!receiver_only_);
    if (packet_information.packet_type_flags & kRtcpRemb) {
      RTC_LOG(LS_VERBOSE)
          << "Incoming REMB: "
          << packet_information.receiver_estimated_max_bitrate_bps;
      rtcp_bandwidth_observer_->OnReceivedEstimatedBitrate(
          packet_information.receiver_estimated_max_bitrate_bps);
    }
    if ((packet_information.packet_type_flags & kRtcpSr) ||
        (packet_information.packet_type_flags & kRtcpRr)) {
      int64_t now_ms = clock_->TimeInMilliseconds();
      rtcp_bandwidth_observer_->OnReceivedRtcpReceiverReport(
          packet_information.report_blocks, packet_information.rtt_ms, now_ms);
    }
  }
  if ((packet_information.packet_type_flags & kRtcpSr) ||
      (packet_information.packet_type_flags & kRtcpRr)) {
    rtp_rtcp_->OnReceivedRtcpReportBlocks(packet_information.report_blocks);
  }

  if (transport_feedback_observer_ &&
      (packet_information.packet_type_flags & kRtcpTransportFeedback)) {
    uint32_t media_source_ssrc =
        packet_information.transport_feedback->media_ssrc();
    if (media_source_ssrc == local_ssrc ||
        registered_ssrcs.find(media_source_ssrc) != registered_ssrcs.end()) {
    //!!!!!!!!!
    //!!!!!!!!!
    //!!!!!!!!!
      transport_feedback_observer_->OnTransportFeedback(
          *packet_information.transport_feedback);
    }
  }

  if (network_state_estimate_observer_ &&
      packet_information.network_state_estimate) {
    network_state_estimate_observer_->OnRemoteNetworkEstimate(
        *packet_information.network_state_estimate);
  }

  if (bitrate_allocation_observer_ &&
      packet_information.target_bitrate_allocation) {
    bitrate_allocation_observer_->OnBitrateAllocationUpdated(
        *packet_information.target_bitrate_allocation);
  }

  if (!receiver_only_) {
    if (stats_callback_) {
      for (const auto& report_block : packet_information.report_blocks) {
        RtcpStatistics stats;
        stats.packets_lost = report_block.packets_lost;
        stats.extended_highest_sequence_number =
            report_block.extended_highest_sequence_number;
        stats.fraction_lost = report_block.fraction_lost;
        stats.jitter = report_block.jitter;

        stats_callback_->StatisticsUpdated(stats, report_block.source_ssrc);
      }
    }
    if (report_block_data_observer_) {
      for (const auto& report_block_data :
           packet_information.report_block_datas) {
        report_block_data_observer_->OnReportBlockDataUpdated(
            report_block_data);
      }
    }
  }
}
```



## 24. RtpTransportControllerSend::OnTransportFeedback

call/rtp_transport_controller_send.cc

```cpp
void RtpTransportControllerSend::OnTransportFeedback(
    const rtcp::TransportFeedback& feedback) {
  feedback_demuxer_.OnTransportFeedback(feedback);
  auto feedback_time = Timestamp::Millis(clock_->TimeInMilliseconds());
  task_queue_.PostTask([this, feedback, feedback_time]() {
    RTC_DCHECK_RUN_ON(&task_queue_);
    absl::optional<TransportPacketsFeedback> feedback_msg =
        transport_feedback_adapter_.ProcessTransportFeedback(feedback,
                                                             feedback_time);
    if (feedback_msg && controller_) {
      PostUpdates(controller_->OnTransportPacketsFeedback(*feedback_msg));
    }
    pacer()->UpdateOutstandingData(
        transport_feedback_adapter_.GetOutstandingData());
  });
}
```

## ------

## 25. GoogCcNetworkController::OnTransportPacketsFeedback

modules/congestion_controller/goog_cc/goog_cc_network_control.cc

```cpp
NetworkControlUpdate GoogCcNetworkController::OnTransportPacketsFeedback(
    TransportPacketsFeedback report) {
  if (report.packet_feedbacks.empty()) {
    // TODO(bugs.webrtc.org/10125): Design a better mechanism to safe-guard
    // against building very large network queues.
    return NetworkControlUpdate();
  }

  if (congestion_window_pushback_controller_) {
    congestion_window_pushback_controller_->UpdateOutstandingData(
        report.data_in_flight.bytes());
  }
  TimeDelta max_feedback_rtt = TimeDelta::MinusInfinity();
  TimeDelta min_propagation_rtt = TimeDelta::PlusInfinity();
  Timestamp max_recv_time = Timestamp::MinusInfinity();

  std::vector<PacketResult> feedbacks = report.ReceivedWithSendInfo();
  for (const auto& feedback : feedbacks)
    max_recv_time = std::max(max_recv_time, feedback.receive_time);

  for (const auto& feedback : feedbacks) {
    TimeDelta feedback_rtt =
        report.feedback_time - feedback.sent_packet.send_time;
    TimeDelta min_pending_time = feedback.receive_time - max_recv_time;
    TimeDelta propagation_rtt = feedback_rtt - min_pending_time;
    max_feedback_rtt = std::max(max_feedback_rtt, feedback_rtt);
    min_propagation_rtt = std::min(min_propagation_rtt, propagation_rtt);
  }

  if (max_feedback_rtt.IsFinite()) {
    feedback_max_rtts_.push_back(max_feedback_rtt.ms());
    const size_t kMaxFeedbackRttWindow = 32;
    if (feedback_max_rtts_.size() > kMaxFeedbackRttWindow)
      feedback_max_rtts_.pop_front();
    // TODO(srte): Use time since last unacknowledged packet.
    bandwidth_estimation_->UpdatePropagationRtt(report.feedback_time,
                                                min_propagation_rtt);
  }
  if (packet_feedback_only_) {
    if (!feedback_max_rtts_.empty()) {
      int64_t sum_rtt_ms = std::accumulate(feedback_max_rtts_.begin(),
                                           feedback_max_rtts_.end(), 0);
      int64_t mean_rtt_ms = sum_rtt_ms / feedback_max_rtts_.size();
      if (delay_based_bwe_)
        delay_based_bwe_->OnRttUpdate(TimeDelta::Millis(mean_rtt_ms));
    }

    TimeDelta feedback_min_rtt = TimeDelta::PlusInfinity();
    for (const auto& packet_feedback : feedbacks) {
      TimeDelta pending_time = packet_feedback.receive_time - max_recv_time;
      TimeDelta rtt = report.feedback_time -
                      packet_feedback.sent_packet.send_time - pending_time;
      // Value used for predicting NACK round trip time in FEC controller.
      feedback_min_rtt = std::min(rtt, feedback_min_rtt);
    }
    if (feedback_min_rtt.IsFinite()) {
      bandwidth_estimation_->UpdateRtt(feedback_min_rtt, report.feedback_time);
    }

    expected_packets_since_last_loss_update_ +=
        report.PacketsWithFeedback().size();
    for (const auto& packet_feedback : report.PacketsWithFeedback()) {
      if (packet_feedback.receive_time.IsInfinite())
        lost_packets_since_last_loss_update_ += 1;
    }
    if (report.feedback_time > next_loss_update_) {
      next_loss_update_ = report.feedback_time + kLossUpdateInterval;
      bandwidth_estimation_->UpdatePacketsLost(
          lost_packets_since_last_loss_update_,
          expected_packets_since_last_loss_update_, report.feedback_time);
      expected_packets_since_last_loss_update_ = 0;
      lost_packets_since_last_loss_update_ = 0;
    }
  }
  absl::optional<int64_t> alr_start_time =
      alr_detector_->GetApplicationLimitedRegionStartTime();

  if (previously_in_alr_ && !alr_start_time.has_value()) {
    int64_t now_ms = report.feedback_time.ms();
    acknowledged_bitrate_estimator_->SetAlrEndedTime(report.feedback_time);
    probe_controller_->SetAlrEndedTimeMs(now_ms);
  }
  previously_in_alr_ = alr_start_time.has_value();
  acknowledged_bitrate_estimator_->IncomingPacketFeedbackVector(
      report.SortedByReceiveTime());
  auto acknowledged_bitrate = acknowledged_bitrate_estimator_->bitrate();
  bandwidth_estimation_->SetAcknowledgedRate(acknowledged_bitrate,
                                             report.feedback_time);
  bandwidth_estimation_->IncomingPacketFeedbackVector(report);
  for (const auto& feedback : report.SortedByReceiveTime()) {
    if (feedback.sent_packet.pacing_info.probe_cluster_id !=
        PacedPacketInfo::kNotAProbe) {
      probe_bitrate_estimator_->HandleProbeAndEstimateBitrate(feedback);
    }
  }

  if (network_estimator_) {
    network_estimator_->OnTransportPacketsFeedback(report);
    auto prev_estimate = estimate_;
    estimate_ = network_estimator_->GetCurrentEstimate();
    // TODO(srte): Make OnTransportPacketsFeedback signal whether the state
    // changed to avoid the need for this check.
    if (estimate_ && (!prev_estimate || estimate_->last_feed_time !=
                                            prev_estimate->last_feed_time)) {
      event_log_->Log(std::make_unique<RtcEventRemoteEstimate>(
          estimate_->link_capacity_lower, estimate_->link_capacity_upper));
    }
  }
  absl::optional<DataRate> probe_bitrate =
      probe_bitrate_estimator_->FetchAndResetLastEstimatedBitrate();
  if (ignore_probes_lower_than_network_estimate_ && probe_bitrate &&
      estimate_ && *probe_bitrate < delay_based_bwe_->last_estimate() &&
      *probe_bitrate < estimate_->link_capacity_lower) {
    probe_bitrate.reset();
  }
  if (limit_probes_lower_than_throughput_estimate_ && probe_bitrate &&
      acknowledged_bitrate) {
    // Limit the backoff to something slightly below the acknowledged
    // bitrate. ("Slightly below" because we want to drain the queues
    // if we are actually overusing.)
    // The acknowledged bitrate shouldn't normally be higher than the delay
    // based estimate, but it could happen e.g. due to packet bursts or
    // encoder overshoot. We use std::min to ensure that a probe result
    // below the current BWE never causes an increase.
    DataRate limit =
        std::min(delay_based_bwe_->last_estimate(),
                 *acknowledged_bitrate * kProbeDropThroughputFraction);
    probe_bitrate = std::max(*probe_bitrate, limit);
  }

  NetworkControlUpdate update;
  bool recovered_from_overuse = false;
  bool backoff_in_alr = false;

  DelayBasedBwe::Result result;
  result = delay_based_bwe_->IncomingPacketFeedbackVector(
      report, acknowledged_bitrate, probe_bitrate, estimate_,
      alr_start_time.has_value());

  if (result.updated) {
    if (result.probe) {
      bandwidth_estimation_->SetSendBitrate(result.target_bitrate,
                                            report.feedback_time);
    }
    // Since SetSendBitrate now resets the delay-based estimate, we have to
    // call UpdateDelayBasedEstimate after SetSendBitrate.
    bandwidth_estimation_->UpdateDelayBasedEstimate(report.feedback_time,
                                                    result.target_bitrate);
    // Update the estimate in the ProbeController, in case we want to probe.
    MaybeTriggerOnNetworkChanged(&update, report.feedback_time);
  }
  recovered_from_overuse = result.recovered_from_overuse;
  backoff_in_alr = result.backoff_in_alr;

  if (recovered_from_overuse) {
    probe_controller_->SetAlrStartTimeMs(alr_start_time);
    auto probes = probe_controller_->RequestProbe(report.feedback_time.ms());
    update.probe_cluster_configs.insert(update.probe_cluster_configs.end(),
                                        probes.begin(), probes.end());
  } else if (backoff_in_alr) {
    // If we just backed off during ALR, request a new probe.
    auto probes = probe_controller_->RequestProbe(report.feedback_time.ms());
    update.probe_cluster_configs.insert(update.probe_cluster_configs.end(),
                                        probes.begin(), probes.end());
  }

  // No valid RTT could be because send-side BWE isn't used, in which case
  // we don't try to limit the outstanding packets.
  if (rate_control_settings_.UseCongestionWindow() &&
      max_feedback_rtt.IsFinite()) {
    UpdateCongestionWindowSize();
  }
  if (congestion_window_pushback_controller_ && current_data_window_) {
    congestion_window_pushback_controller_->SetDataWindow(
        *current_data_window_);
  } else {
    update.congestion_window = current_data_window_;
  }

  return update;
}
```





## 参考

WebRTC源码分析地址：https://github.com/chensongpoixs/cwebrtc
https://blog.csdn.net/Poisx/article/details/128439310
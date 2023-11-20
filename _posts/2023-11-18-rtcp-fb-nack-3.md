
---
layout: post
title: rtcp NACK 流程及代码解读
date: 2023-11-18 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtp nack
---


* content
{:toc}

---

## 1. 前言

## 1. 视频数据发送缓冲流程

### 1.1 视频发送数据流

![在这里插入图片描述](rtcp-fb-nack-3.assets/video-send.png)

### 1.2 调用堆栈

```less
 H264EncoderImpl::Encode
 VideoStreamEncoder::OnEncodedImage
 VideoSendStreamImpl::OnEncodedImage
 RtpVideoSender::OnEncodedImage
 RTPSenderVideo::SendEncodedImage
 RTPSenderVideo::SendVideo
 RTPSenderVideo::LogAndSendToNetwork
 RTPSender::EnqueuePackets
 pacer //
 RtpSenderEgress::SendPacket
 //放入队列
 RtpPacketHistory::PutRtpPacket
```



Pacer 发送

```cpp
ProcessThreadImpl::Process
->PacedSender::Process    
->PacingController::ProcessPackets
->PacketRouter::SendPacket
->ModuleRtpRtcpImpl2::TrySendPacket
->RtpSenderEgress::SendPacket
```





### 1.3 RtpSenderEgress::SendPacket

modules/rtp_rtcp/source/rtp_sender_egress.cc

```cpp
void RtpSenderEgress::SendPacket(RtpPacketToSend* packet,
                                 const PacedPacketInfo& pacing_info) {
  ...

  const bool send_success = SendPacketToNetwork(*packet, options, pacing_info);

  ///////////////////!!!
  // Put packet in retransmission history or update pending status even if
  // actual sending fails.
  if (is_media && packet->allow_retransmission()) {
    packet_history_->PutRtpPacket(std::make_unique<RtpPacketToSend>(*packet),
                                  now_ms);
  } else if (packet->retransmitted_sequence_number()) {
    packet_history_->MarkPacketAsSent(*packet->retransmitted_sequence_number());
  }
   ///////////////////!!!

 	...
}
```

每次pacer发送报文的时候，都会把媒体报文储存在packet_history_队列。



### 1.4 ???RtpPacketHistory

RtpPacketHistory 负责缓存历史数据，以SequenceNumber为索引。



#### 1.4.1 std::deque\<StoredPacket\> packet_history_

发送RTP报文，实时存储报文到`packet_history_ `。



#### 1.4.2 RtpPacketHistory::SetStorePacketsStatus

```cpp
void RtpPacketHistory::SetStorePacketsStatus(StorageMode mode,
                                             size_t number_to_store) {
  ...
  Reset();
  // 是否开启缓存功能
  mode_ = mode;
  // 设置队列的元素
  number_to_store_ = std::min(kMaxCapacity, number_to_store);
}
```

1. StorageMode mode_;

```cpp
  enum class StorageMode {
    kDisabled,     // Don't store any packets.
    kStoreAndCull  // Store up to |number_to_store| packets, but try to remove
                   // packets as they time out or as signaled as received.
  };
```

2. number_to_store_， 队列存放的元素个数，设置最小值

`number_to_store_ = std::min(kMaxCapacity, number_to_store);`

```cpp
// modules/rtp_rtcp/source/rtp_packet_history.cc  
// Maximum number of packets we ever allow in the history.
  static constexpr size_t kMaxCapacity = 9600;
```

3. 音频设置长度
   audio/channel_send.cc

   ```cpp
   void ChannelSend::RegisterSenderCongestionControlObjects(
       RtpTransportControllerSendInterface* transport,
       RtcpBandwidthObserver* bandwidth_observer) {
     RTC_DCHECK_RUN_ON(&worker_thread_checker_);
     RtpPacketSender* rtp_packet_pacer = transport->packet_sender();
     PacketRouter* packet_router = transport->packet_router();
   
     RTC_DCHECK(rtp_packet_pacer);
     RTC_DCHECK(packet_router);
     RTC_DCHECK(!packet_router_);
     rtcp_observer_->SetBandwidthObserver(bandwidth_observer);
     rtp_packet_pacer_proxy_->SetPacketPacer(rtp_packet_pacer);
     rtp_rtcp_->SetStorePacketsStatus(true, 600);
     constexpr bool remb_candidate = false;
     packet_router->AddSendRtpModule(rtp_rtcp_.get(), remb_candidate);
     packet_router_ = packet_router;
   }
   ```

4. 视频设置长度
   call/rtp_video_sender.cc

   ```cpp
   std::vector<RtpStreamSender> CreateRtpStreamSenders(
      ...
      std::unique_ptr<ModuleRtpRtcpImpl2> rtp_rtcp(
           ModuleRtpRtcpImpl2::Create(configuration));
       rtp_rtcp->SetSendingStatus(false);
       rtp_rtcp->SetSendingMediaStatus(false);
       rtp_rtcp->SetRTCPStatus(RtcpMode::kCompound);
       // Set NACK.
       rtp_rtcp->SetStorePacketsStatus(true, kMinSendSidePacketHistorySize);
   ...
   }
   ```

5. ModuleRtpRtcpImpl2::SetStorePacketsStatus

   modules/rtp_rtcp/source/rtp_rtcp_impl2.cc

   ```cpp
   // Store the sent packets, needed to answer to Negative acknowledgment requests.
   // number_to_store = 600;
   void ModuleRtpRtcpImpl2::SetStorePacketsStatus(const bool enable,
                                                  const uint16_t number_to_store) {
     rtp_sender_->packet_history.SetStorePacketsStatus(
         enable ? RtpPacketHistory::StorageMode::kStoreAndCull
                : RtpPacketHistory::StorageMode::kDisabled,
         number_to_store);
   }
   ```



#### 1.4.3 RtpPacketHistory::PutRtpPacket

modules/rtp_rtcp/source/rtp_packet_history.cc

```cpp
void RtpPacketHistory::PutRtpPacket(std::unique_ptr<RtpPacketToSend> packet,
                                    absl::optional<int64_t> send_time_ms) {
  RTC_DCHECK(packet);
  MutexLock lock(&lock_);
  int64_t now_ms = clock_->TimeInMilliseconds();
  if (mode_ == StorageMode::kDisabled) {
    return;
  }

  RTC_DCHECK(packet->allow_retransmission());
  // 删除过期的包或者超出容量的包
  CullOldPackets(now_ms);

  // Store packet.
  const uint16_t rtp_seq_no = packet->SequenceNumber();
  int packet_index = GetPacketIndex(rtp_seq_no);
  if (packet_index >= 0u &&
      static_cast<size_t>(packet_index) < packet_history_.size() &&
      packet_history_[packet_index].packet_ != nullptr) {
    RTC_LOG(LS_WARNING) << "Duplicate packet inserted: " << rtp_seq_no;
    // Remove previous packet to avoid inconsistent state.
    RemovePacket(packet_index);
    packet_index = GetPacketIndex(rtp_seq_no);
  }

  // Packet to be inserted ahead of first packet, expand front.
  for (; packet_index < 0; ++packet_index) {
    packet_history_.emplace_front(nullptr, absl::nullopt, 0);
  }
  // Packet to be inserted behind last packet, expand back.
  while (static_cast<int>(packet_history_.size()) <= packet_index) {
    packet_history_.emplace_back(nullptr, absl::nullopt, 0);
  }

  RTC_DCHECK_GE(packet_index, 0);
  RTC_DCHECK_LT(packet_index, packet_history_.size());
  RTC_DCHECK(packet_history_[packet_index].packet_ == nullptr);

  packet_history_[packet_index] =
      StoredPacket(std::move(packet), send_time_ms, packets_inserted_++);

  if (enable_padding_prio_) {
    if (padding_priority_.size() >= kMaxPaddingtHistory - 1) {
      padding_priority_.erase(std::prev(padding_priority_.end()));
    }
    auto prio_it = padding_priority_.insert(&packet_history_[packet_index]);
    RTC_DCHECK(prio_it.second) << "Failed to insert packet into prio set.";
  }
}
```

每次向`packet_history_`里放入rtp包的时候，都会检查`packet_history_`的容量，如注释所示，如果要放的包的位置里还有包，那么扩容，不过这样会破坏缓存的连续性，在查找的时候，需要遍历了。



#### 1.4.4 RtpPacketHistory::CullOldPackets

```cpp
void RtpPacketHistory::CullOldPackets(int64_t now_ms) {
  
  // static constexpr int kMinPacketDurationRtt = 3;
  // Don't remove packets within max(1000ms, 3x RTT).
  // static constexpr int64_t kMinPacketDurationMs = 1000;
  int64_t packet_duration_ms =
      std::max(kMinPacketDurationRtt * rtt_ms_, kMinPacketDurationMs);
  
  
  while (!packet_history_.empty()) {
    // 超过packet_history_最大存储 kMaxCapacity = 9600 ，则清除
    if (packet_history_.size() >= kMaxCapacity) {
      // We have reached the absolute max capacity, remove one packet
      // unconditionally.
      RemovePacket(0);
      continue;
    }

    // 正在发送
    const StoredPacket& stored_packet = packet_history_.front();
    if (stored_packet.pending_transmission_) {
      // Don't remove packets in the pacer queue, pending tranmission.
      return;
    }

    if (*stored_packet.send_time_ms_ + packet_duration_ms > now_ms) {
      // Don't cull packets too early to avoid failed retransmission requests.
      return;
    }

    // 1. 包数量过多； 2. 包过期，
    //  With kStoreAndCull, always remove packets after 3x max(1000ms, 3x rtt).
  	//	static constexpr int kPacketCullingDelayFactor = 3;
    if (packet_history_.size() >= number_to_store_ ||
        *stored_packet.send_time_ms_ +
                (packet_duration_ms * kPacketCullingDelayFactor) <=
            now_ms) {
      // Too many packets in history, or this packet has timed out. Remove it
      // and continue.
      RemovePacket(0);
    } else {
      // No more packets can be removed right now.
      return;
    }
  }
}
```

1. 包的数量`packet_history_.size() >= number_to_store_ ` 超过了，则就删除
2. 包的生命时长`*stored_packet.send_time_ms_ +   (packet_duration_ms * kPacketCullingDelayFactor) <=  now_ms`
   发送时间 + 包存在的最大时长（3 * rtt * 3 ）。





## 2. 视频Nack请求的发送过程

### 2.0 NackMoudle的创建时机

在RtpVideoStreamReceiver中构造函数中创建NackMoudle类，RtpVideoStreamReceiver是在VideoReceiveStream类的构造函数创建。



### 2.1 rtp接收数据流程

![在这里插入图片描述](rtcp-fb-nack-3.assets/nack.png)



### 2.2 nack对报序号的管理

接收到rtp数据包后，会调用nack模块对报序号进行记录，同时检测是否需要nack请求

#### NackModule2::OnReceivedPacket

modules/video_coding/nack_module2.cc

```cpp
int NackModule2::OnReceivedPacket(uint16_t seq_num,
                                    bool is_keyframe,
                                    bool is_recovered) {
  RTC_DCHECK_RUN_ON(worker_thread_);
  bool is_retransmitted = true;

  if (!initialized_) {
    newest_seq_num_ = seq_num;
    if (is_keyframe)
      keyframe_list_.insert(seq_num);
    initialized_ = true;
    return 0;
  }

  // Since the `newest_seq_num_` is a packet we have actually received we know
  // that packet has never been Nacked.
  if (seq_num == newest_seq_num_)
    return 0;
  //如果接收的报序号小于之前接收到的，可能是乱序的包，可能是重传包
  //如果nack列表有，则清除
  if (AheadOf(newest_seq_num_, seq_num)) {
    // An out of order packet has been received.
    auto nack_list_it = nack_list_.find(seq_num);
    int nacks_sent_for_packet = 0;
    if (nack_list_it != nack_list_.end()) {
      nacks_sent_for_packet = nack_list_it->second.retries;
      nack_list_.erase(nack_list_it);
    }
    if (!is_retransmitted)
      UpdateReorderingStatistics(seq_num);
    return nacks_sent_for_packet;
  }

  // Keep track of new keyframes.
  //保留最新的关键帧包
  if (is_keyframe)
    keyframe_list_.insert(seq_num);

  // And remove old ones so we don't accumulate keyframes.
  auto it = keyframe_list_.lower_bound(seq_num - kMaxPacketAge);
  if (it != keyframe_list_.begin())
    keyframe_list_.erase(keyframe_list_.begin(), it);

  if (is_recovered) {
    recovered_list_.insert(seq_num);

    // Remove old ones so we don't accumulate recovered packets.
    auto it = recovered_list_.lower_bound(seq_num - kMaxPacketAge);
    if (it != recovered_list_.begin())
      recovered_list_.erase(recovered_list_.begin(), it);

    // Do not send nack for packets recovered by FEC or RTX.
    return 0;
  }

  AddPacketsToNack(newest_seq_num_ + 1, seq_num);
  newest_seq_num_ = seq_num;

  // Are there any nacks that are waiting for this seq_num.
  //获取nack序号，如果有则触发nack
  std::vector<uint16_t> nack_batch = GetNackBatch(kSeqNumOnly);
  if (!nack_batch.empty()) {
    // This batch of NACKs is triggered externally; the initiator can
    // batch them with other feedback messages.
    nack_sender_->SendNack(nack_batch, /*buffering_allowed=*/true);
  }

  return 0;
}
```

这部分逻辑主要是收到包，查一下是不是乱序的，可能是网络造成乱序，也可能是重发过来的，收到了就把nack list里面的记录删掉



#### NackModule2::AddPacketsToNack

modules/video_coding/nack_module2.cc

```cpp
void NackModule2::AddPacketsToNack(uint16_t seq_num_start,
                                     uint16_t seq_num_end) {
  // Called on worker_thread_.
  // Remove old packets.
  //kMaxPacketAge=1000,删除超出队列数量，删除最老的
  auto it = nack_list_.lower_bound(seq_num_end - kMaxPacketAge);
  nack_list_.erase(nack_list_.begin(), it);

  // If the nack list is too large, remove packets from the nack list until
  // the latest first packet of a keyframe. If the list is still too large,
  // clear it and request a keyframe.
  //nack_list 的最大容量为 kMaxNackPackets = 1000, 
  //如果满了会删除最后一个 KeyFrame 之前的所有nacked 序号, 
  //如果删除之后还是满的那么清空 nack_list 并请求KeyFrame
  uint16_t num_new_nacks = ForwardDiff(seq_num_start, seq_num_end);
  if (nack_list_.size() + num_new_nacks > kMaxNackPackets) {
    while (RemovePacketsUntilKeyFrame() &&
           nack_list_.size() + num_new_nacks > kMaxNackPackets) {
    }

    if (nack_list_.size() + num_new_nacks > kMaxNackPackets) {
      nack_list_.clear();
      RTC_LOG(LS_WARNING) << "NACK list full, clearing NACK"
                             " list and requesting keyframe.";
      keyframe_request_sender_->RequestKeyFrame();
      return;
    }
  }

  for (uint16_t seq_num = seq_num_start; seq_num != seq_num_end; ++seq_num) {
    // Do not send nack for packets that are already recovered by FEC or RTX
    if (recovered_list_.find(seq_num) != recovered_list_.end())
      continue;
    NackInfo nack_info(seq_num, seq_num + WaitNumberOfPackets(0.5),
                       clock_->TimeInMilliseconds());
    RTC_DCHECK(nack_list_.find(seq_num) == nack_list_.end());
    nack_list_[seq_num] = nack_info;
  }
}
```

我们可以看到AddPacketsToNack()函数主要实现了：
nack_list 的最大容量为 kMaxNackPackets = 1000, 如果满了会删除最后一个 KeyFrame 之前的所有nacked 序号, 如果删除之后还是满的那么清空 nack_list 并请求KeyFrame。

#### NackModule2::GetNackBatch

modules/video_coding/nack_module2.cc

获取需要nack的

```cpp
std::vector<uint16_t> NackModule2::GetNackBatch(NackFilterOptions options) {
  // Called on worker_thread_.
  //只考虑根据序列号获取nacklist
  bool consider_seq_num = options != kTimeOnly;
  //只考虑根据时间戳获取nacklist
  bool consider_timestamp = options != kSeqNumOnly;
  //当前时间
  Timestamp now = clock_->CurrentTime();
  std::vector<uint16_t> nack_batch;
  auto it = nack_list_.begin();
  //遍历nack
  while (it != nack_list_.end()) {
    //初始化rtt为重发延时间隔
    TimeDelta resend_delay = TimeDelta::Millis(rtt_ms_);
    //如果使用了nack的配置
    if (backoff_settings_) {
      //设置最大的重发延时间隔
      resend_delay =
          std::max(resend_delay, backoff_settings_->min_retry_interval);
     // 如果重试次数超过了1次，则重新计算幂指后的重发延迟间隔（避免重试频繁）
     //每次延时增大25%，1.25的n次幂
      if (it->second.retries > 1) {
        TimeDelta exponential_backoff =
            std::min(TimeDelta::Millis(rtt_ms_), backoff_settings_->max_rtt) *
            std::pow(backoff_settings_->base, it->second.retries - 1);
        resend_delay = std::max(resend_delay, exponential_backoff);
      }
    }
   // 判断当前包seq_num是否该发送了（即超过了最大发送延迟时间）
    bool delay_timed_out =
        now.ms() - it->second.created_at_time >= send_nack_delay_ms_;
    // 判断基于rtt延迟时间时是否该发送了（即超过了重发延迟时间）
    bool nack_on_rtt_passed =
        now.ms() - it->second.sent_at_time >= resend_delay.ms();
    // 判断基于序列号时是否该发送了（即超过了重发延迟时间）
    bool nack_on_seq_num_passed =
         // 初次发送时有效，避免重复重发
        it->second.sent_at_time == -1 &&
        // 当前包序列号比较老
        AheadOrAt(newest_seq_num_, it->second.send_at_seq_num);
    // 如果该发送了
    if (delay_timed_out && ((consider_seq_num && nack_on_seq_num_passed) ||
                            (consider_timestamp && nack_on_rtt_passed))) {
       // 当前包seq_num加入到nack list
      nack_batch.emplace_back(it->second.seq_num);
      ++it->second.retries; // 累积重试次数
        // 设置发送时间
      it->second.sent_at_time = now.ms();
      // 超过最大重试次数了则从nack_list_移除
      if (it->second.retries >= kMaxNackRetries) {
        RTC_LOG(LS_WARNING) << "Sequence number " << it->second.seq_num
                            << " removed from NACK list due to max retries.";
        it = nack_list_.erase(it);
      } else {
        ++it;
      }
      continue;
    }
    ++it;
  }
  return nack_batch;
}
```

1、delay_timed_out ：加入nacklist的时间大于要发送nack的延时
2、nack_on_rtt_passed ：该序号上次发送NACK的时间到当前时间要超过前面计算出来的延时。
3：nack_on_seq_num_passed ：确定有最新的包序号比这个大，是被丢失的



#### nackmodule 说明

https://zhuanlan.zhihu.com/p/338409893

https://www.jianshu.com/p/6413cf2e8aca

### 2.3 nack请求的发送时机

![在这里插入图片描述](rtcp-fb-nack-3.assets/nack-request.png)

有两个地方触发nack，用红方框框出来了

#### 当收到rtp数据，nack模块会记录包序号，包类型

#### 线程定期检测是否存在丢包，需要nack请求



## 3. nack响应

### 3.1 rtcp数据流

![在这里插入图片描述](rtcp-fb-nack-3.assets/nack-response.png)

```less
RTCPReceiver::IncomingPacket  modules/rtp_rtcp/source/rtcp_receiver.cc
	RTCPReceiver::ParseCompoundPacket
RTCPReceiver::TriggerCallbacksFromRtcpPacket
ModuleRtpRtcpImpl2::OnReceivedNack  modules/rtp_rtcp/source/rtp_rtcp_impl2.cc
RTPSender::OnReceivedNack
RTPSender::ReSendPacket
```

- 首先是正常的 RTCP 处理流程: RTCPReceiver 中解析处理RTCP 详细, 在 `TriggerCallbacksFromRtcpPacket`处理不同的RTCP消息.
- 如果是 kRtcpNack 消息则触发 RtpRtcp 对象的 `ModuleRtpRtcpImpl::OnReceivedNack` 处理流程:
  将丢包的序号 记录到`PacketLossStats`, 获取RTT后进入 RTPSedner.`OnReceivedNack`.
- **RTPSender**中完成在`PacketHistory`中查找需要发送的RTP seq, 并决定重发时间. 重发也需要经过重发的比特率限制的检查. RTPSedner 初始化话时可以配置是否使用(PacedSend, 均匀发送), 最后检查重发格式(RtxStatus() 可以获取是否使用 RTX 封装)后使用 `RTPSedner::PrepareAndSendPacket`进行立即重发. 如果是使用 PacedSend, 则使用 `PacedSender::InsertPacket` 先加入发送列表中, 它的process会定时处理发送任务.



### 3.2 ModuleRtpRtcpImpl2::OnReceivedNack

modules/rtp_rtcp/source/rtp_rtcp_impl2.cc

```cpp
void ModuleRtpRtcpImpl2::OnReceivedNack(
    const std::vector<uint16_t>& nack_sequence_numbers) {
  if (!rtp_sender_)
    return;

  if (!StorePackets() || nack_sequence_numbers.empty()) {
    return;
  }
  // Use RTT from RtcpRttStats class if provided.
  int64_t rtt = rtt_ms();
  if (rtt == 0) {
    rtcp_receiver_.RTT(rtcp_receiver_.RemoteSSRC(), NULL, &rtt, NULL, NULL);
  }
  //取得rtt，把请求和rtt时间调用rtp补包
  rtp_sender_->packet_generator.OnReceivedNack(nack_sequence_numbers, rtt);
}
```



### 3.3 RTPSender::OnReceivedNack

modules/rtp_rtcp/source/rtp_sender.cc

```cpp
void RTPSender::OnReceivedNack(
    const std::vector<uint16_t>& nack_sequence_numbers,
    int64_t avg_rtt) {
  //设置历史队列rtt，取包时根据rtt计算
  packet_history_->SetRtt(5 + avg_rtt);
  for (uint16_t seq_no : nack_sequence_numbers) {
    const int32_t bytes_sent = ReSendPacket(seq_no);
    if (bytes_sent < 0) {
      // Failed to send one Sequence number. Give up the rest in this nack.
      RTC_LOG(LS_WARNING) << "Failed resending RTP packet " << seq_no
                          << ", Discard rest of packets.";
      break;
    }
  }
}


```



### 3.4 RTPSender::ReSendPacket

modules/rtp_rtcp/source/rtp_sender.cc

```cpp
int32_t RTPSender::ReSendPacket(uint16_t packet_id) {
  // Try to find packet in RTP packet history. Also verify RTT here, so that we
  // don't retransmit too often.
  absl::optional<RtpPacketHistory::PacketState> stored_packet =
      packet_history_->GetPacketState(packet_id);
  if (!stored_packet || stored_packet->pending_transmission) {
    // Packet not found or already queued for retransmission, ignore.
    return 0;
  }

  const int32_t packet_size = static_cast<int32_t>(stored_packet->packet_size);
  const bool rtx = (RtxStatus() & kRtxRetransmitted) > 0;

  std::unique_ptr<RtpPacketToSend> packet =
    // 两次发送之间的时长小于RTT，则不发送
      packet_history_->GetPacketAndMarkAsPending(
          packet_id, [&](const RtpPacketToSend& stored_packet) {
            // Check if we're overusing retransmission bitrate.
            // TODO(sprang): Add histograms for nack success or failure
            // reasons.
            std::unique_ptr<RtpPacketToSend> retransmit_packet;
            if (retransmission_rate_limiter_ &&
                !retransmission_rate_limiter_->TryUseRate(packet_size)) {
              return retransmit_packet;
            }
            // 携带FID的ssrc参数，走单独的rtx通道
            if (rtx) {
              retransmit_packet = BuildRtxPacket(stored_packet);
            } else { // 和正常的媒体数据一起发送
              retransmit_packet =
                  std::make_unique<RtpPacketToSend>(stored_packet);
            }
            if (retransmit_packet) {
              retransmit_packet->set_retransmitted_sequence_number(
                  stored_packet.SequenceNumber());
            }
            return retransmit_packet;
          });
  if (!packet) {
    return -1;
  }
  // 设置包的优先级
  packet->set_packet_type(RtpPacketMediaType::kRetransmission);
  packet->set_fec_protect_packet(false);
  std::vector<std::unique_ptr<RtpPacketToSend>> packets;
  packets.emplace_back(std::move(packet));
  paced_sender_->EnqueuePackets(std::move(packets));

  return packet_size;
}

```

1. GetPacketAndMarkAsPending会判断上次重传报文时间和当前时间差是否大于RTT，若小于则不重传。`RtpPacketHistory::VerifyRtt `。说白了就是两次发送之间的时长小于RTT，则不发送

2. NACK重新发送媒体数据有两种方式：单独RTX通道发送、与媒体数据混在一起发送。**两种形式对单纯的NACK抗性影响不太大，但是与媒体数据混在一起发送模式，接收端无法区分是NACK重传报文，还是正常媒体数据，会导致接收端反馈的丢包率低于实际值，影响gcc探测码率，及发送端FEC冗余度配置。所以建议还是以RTX通道单独发送。**
   RTX通道单独发送重传报文，需要配置参数???, [参考](https://blog.csdn.net/CrystalShaw/article/details/119176101)

3. RTPSender::ReSendPacket在将重传数据加入pacer队列，会设置报文优先级，为了保证实时性，NACK重传报文需要按照高优先级重传。优先级配置在set_packet_type，发送报文时，会根据kRetransmission获取发送优先级。在` PacingController::EnqueuePacket` 根据优先级发送；
   modules/pacing/pacing_controller.cc

   ```cpp
   int GetPriorityForType(RtpPacketMediaType type) {
     // Lower number takes priority over higher.
     switch (type) {
       case RtpPacketMediaType::kAudio:
         // Audio is always prioritized over other packet types.
         return kFirstPriority + 1;
       case RtpPacketMediaType::kRetransmission:
         // Send retransmissions before new media.
         return kFirstPriority + 2;
       case RtpPacketMediaType::kVideo:
       case RtpPacketMediaType::kForwardErrorCorrection:
         // Video has "normal" priority, in the old speak.
         // Send redundancy concurrently to video. If it is delayed it might have a
         // lower chance of being useful.
         return kFirstPriority + 3;
       case RtpPacketMediaType::kPadding:
         // Packets that are in themselves likely useless, only sent to keep the
         // BWE high.
         return kFirstPriority + 4;
     }
     RTC_CHECK_NOTREACHED();
   }
   ```

   



#### 3.4.1 RtpPacketHistory::GetPacketAndMarkAsPending

modules/rtp_rtcp/source/rtp_packet_history.cc

```cpp
std::unique_ptr<RtpPacketToSend> RtpPacketHistory::GetPacketAndMarkAsPending(
    uint16_t sequence_number,
    rtc::FunctionView<std::unique_ptr<RtpPacketToSend>(const RtpPacketToSend&)>
        encapsulate) {
  MutexLock lock(&lock_);
  if (mode_ == StorageMode::kDisabled) {
    return nullptr;
  }

  StoredPacket* packet = GetStoredPacket(sequence_number);
  if (packet == nullptr) {
    return nullptr;
  }

  if (packet->pending_transmission_) {
    // Packet already in pacer queue, ignore this request.
    return nullptr;
  }

  // 校验rtt
  if (!VerifyRtt(*packet, clock_->TimeInMilliseconds())) {
    // Packet already resent within too short a time window, ignore.
    return nullptr;
  }

  // Copy and/or encapsulate packet.
  std::unique_ptr<RtpPacketToSend> encapsulated_packet =
      encapsulate(*packet->packet_);
  if (encapsulated_packet) {
    packet->pending_transmission_ = true;
  }

  return encapsulated_packet;
}
```






```cpp
GetStoredPacket //按照序列号拿到packet
VerifyRtt //距离上次发送，超过rtt时间才能再次发送

```

```cpp
bool RtpPacketHistory::VerifyRtt(const RtpPacketHistory::StoredPacket& packet,
                                 int64_t now_ms) const {
  if (packet.send_time_ms_) {
    // Send-time already set, this check must be for a retransmission.
    if (packet.times_retransmitted() > 0 &&
        now_ms < *packet.send_time_ms_ + rtt_ms_) {
      // This packet has already been retransmitted once, and the time since
      // that even is lower than on RTT. Ignore request as this packet is
      // likely already in the network pipe.
      return false;
    }
  }

  return true;
}
```





## 4. 注意

nack请求次数限制，kMaxNackRetries ，不会一直请求，超过10次就不在请求
nack间隔越来越大， 如果重试次数超过了1次，则重新计算幂指后的重发延迟间隔（避免重试频繁），每次延时增大25%，1.25的n次幂
nack最大数量是1000，大于的不会重传 kMaxPacketAge
nack线程会间隔20ms检测一次， kProcessIntervalMs 默认为20ms，
发送端收到nack请求后，检测距离上次时间超过rtt才能再次发送



## 参考

[WebRtc Video Receiver(三)-NACK丢包重传原理](https://www.jianshu.com/p/6413cf2e8aca)

[webrtc 代码学习（二十五）video nack 模块](https://blog.csdn.net/baozi520cc/article/details/103023435)

[webrtc QOS方法一.3(发送端NACK实现)](https://blog.csdn.net/CrystalShaw/article/details/119176101)

[webrtc在发送端缓存RTP包的方法 RTPPacketHistory](https://blog.csdn.net/dong_beijing/article/details/81143074)

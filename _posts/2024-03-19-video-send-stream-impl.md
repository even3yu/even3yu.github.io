---
layout: post
title: Video Send Stream 介绍
date: 2024-03-19 23:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc video stream
---


* content
{:toc}

---


## 1. VideoSendStreamImpl

video/video_send_stream_impl.h

```cpp
class VideoSendStreamImpl : public webrtc::BitrateAllocatorObserver,
                            public VideoStreamEncoderInterface::EncoderSink {
                              
         ...
           
}
```



## 2. 属性

### 2.1 RepeatingTaskHandle check_encoder_activity_task_

检测编码的状态，每隔2s检测是否正常。如果2s内有编码回调则认为正常。

### 2.2 RtpTransportControllerSendInterface* const transport_

构造的时候传入，是在创建Call的时候创建，传输控制模块。

### 2.3 BitrateAllocatorInterface* const bitrate_allocator_

构造的时候传入, 是在Call 的构造创建。

```cpp
Call::Call(Clock* clock,
           const Call::Config& config,
           std::unique_ptr<RtpTransportControllerSendInterface> transport_send,
           rtc::scoped_refptr<SharedModuleThread> module_process_thread,
           TaskQueueFactory* task_queue_factory)
    : ...
      bitrate_allocator_(new BitrateAllocator(this))
    	...{
      ...
    }
```



### 2.4 VideoStreamEncoderInterface* const video_stream_encoder_

构造的时候传入，是在`VideoSendStream::VideoSendStream`创建的对象。

```cpp
VideoSendStream::VideoSendStream(...){
...
video_stream_encoder_ = std::make_unique<VideoStreamEncoder>(
      clock, num_cpu_cores, &stats_proxy_, config_.encoder_settings,
      std::make_unique<OveruseFrameDetector>(&stats_proxy_), task_queue_factory,
      GetBitrateAllocationCallbackType(config_));
  
  ...
}
```



### 2.5 RtpVideoSenderInterface* const rtp_video_sender_

在构造里创建的。其实是通过`RtpTransportControllerSendInterface::CreateRtpVideoSender ` 创建。

```cpp
rtp_video_sender_(
          transport_->CreateRtpVideoSender(suspended_ssrcs,
                                           suspended_payload_states,
                                           config_->rtp,
                                           config_->rtcp_report_interval_ms,
                                           config_->send_transport,
                                           CreateObservers(call_stats,
                                                           &encoder_feedback_,
                                                           stats_proxy_,
                                                           send_delay_stats),
                                           event_log,
                                           std::move(fec_controller),
                                           CreateFrameEncryptionConfig(config_),
                                           config->frame_transformer)),
```



## -----

## 3. VideoSendStreamImpl::RegisterProcessThread

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::RegisterProcessThread(
    ProcessThread* module_process_thread) {
  // Called on libjingle's worker thread (not worker_queue_), as part of the
  // initialization steps. That's also the correct thread/queue for setting the
  // state for |video_stream_encoder_|.

  // Only request rotation at the source when we positively know that the remote
  // side doesn't support the rotation extension. This allows us to prepare the
  // encoder in the expectation that rotation is supported - which is the common
  // case.
  //
  // rtp extension 是否包含了 rotation
  bool rotation_applied = absl::c_none_of(
      config_->rtp.extensions, [](const RtpExtension& extension) {
        return extension.uri == RtpExtension::kVideoRotationUri;
      });

  // VideoSendStreamImpl， 实现 了VideoStreamEncoderInterface::EncoderSink 
	// 这样编码后的数据就会回调VideoSendStreamImpl::OnEncodedImage
  video_stream_encoder_->SetSink(this, rotation_applied);

  // ProcessThread* module_process_thread 给 RtpVideoSender
  rtp_video_sender_->RegisterProcessThread(module_process_thread);
}
```



### 3.1 ProcessThread* module_process_thread 创建时机

call/call_factory.cc

```cpp
Call* CallFactory::CreateCall(const Call::Config& config) {
  RTC_DCHECK_RUN_ON(&call_thread_);
  absl::optional<webrtc::BuiltInNetworkBehaviorConfig> send_degradation_config =
      ParseDegradationConfig(true);
  absl::optional<webrtc::BuiltInNetworkBehaviorConfig>
      receive_degradation_config = ParseDegradationConfig(false);

  if (send_degradation_config || receive_degradation_config) {
    return new DegradedCall(std::unique_ptr<Call>(Call::Create(config)),
                            send_degradation_config, receive_degradation_config,
                            config.task_queue_factory);
  }

  //!!!!!!!!
  //!!!!!!!!
  //!!!!!!!!
  // 创建了 ProcessThread， 统一call 共用这个
  if (!module_thread_) {
    module_thread_ = SharedModuleThread::Create(
        ProcessThread::Create("SharedModThread"), [this]() {
          RTC_DCHECK_RUN_ON(&call_thread_);
          module_thread_ = nullptr;
        });
  }

  // module_thread_ 会传给每个VideoSendStream，VideoStreamImpl，RtpVideoSender
  return Call::Create(config, module_thread_);
  //!!!!!!!!
  //!!!!!!!!
  //!!!!!!!!
}
```



### 3.2 调用堆栈

```less
VideoSendStream::VideoSendStream
VideoSendStreamImpl::DeRegisterProcessThread
RtpVideoSender::RegisterProcessThread
ProcessThread::RegisterModule			//RtpStreamSender::ModuleRtpRtcpImpl2
```

就是把RtpStreamSender::ModuleRtpRtcpImpl2 注册到ProcessThread， 定时器定时处理，回调到`ModuleRtpRtcpImpl2::Process` 进行处理。具体做什么的？？？主要是定时发送rtcp包，每隔5ms发送一次。



#### 3.2.1 ModuleRtpRtcpImpl2::Process

modules/rtp_rtcp/source/rtp_rtcp_impl2.cc

```cpp
void ModuleRtpRtcpImpl2::Process() {
  RTC_DCHECK_RUN_ON(&process_thread_checker_);

  const Timestamp now = clock_->CurrentTime();

  // TODO(bugs.webrtc.org/11581): Figure out why we need to call Process() 200
  // times a second.
  next_process_time_ = now.ms() + kRtpRtcpMaxIdleTimeProcessMs;

  // TODO(bugs.webrtc.org/11581): once we don't use Process() to trigger
  // calls to SendRTCP(), the only remaining timer will require remote_bitrate_
  // to be not null. In that case, we can disable the timer when it is null.
  if (remote_bitrate_ && rtcp_sender_.Sending() && rtcp_sender_.TMMBR()) {
    unsigned int target_bitrate = 0;
    std::vector<unsigned int> ssrcs;
    if (remote_bitrate_->LatestEstimate(&ssrcs, &target_bitrate)) {
      if (!ssrcs.empty()) {
        target_bitrate = target_bitrate / ssrcs.size();
      }
      rtcp_sender_.SetTargetBitrate(target_bitrate);
    }
  }

  // TODO(bugs.webrtc.org/11581): Run this on a separate set of delayed tasks
  // based off of next_time_to_send_rtcp_ in RTCPSender.
  if (rtcp_sender_.TimeToSendRTCPReport())
    rtcp_sender_.SendRTCP(GetFeedbackState(), kRtcpReport);
}
```



## 4. VideoSendStreamImpl::DeRegisterProcessThread

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::DeRegisterProcessThread() {
  rtp_video_sender_->DeRegisterProcessThread();
}
```



## -----

## 5. ??? VideoSendStreamImpl::DeliverRtcp

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::DeliverRtcp(const uint8_t* packet, size_t length) {
  // Runs on a network thread.
  RTC_DCHECK(!worker_queue_->IsCurrent());
  rtp_video_sender_->DeliverRtcp(packet, length);
}
```

处理收到的rtcp包，最终全部转发到RTCPReceiver 进行处理，具体看后文。



### 5.1 调用堆栈

```less
...
Call::DeliverRtcp
VideoSendStream::DeliverRtcp
VideoSendStreamImpl::DeliverRtcp
RtpVideoSender::DeliverRtcp
ModuleRtpRtcpImpl2::IncomingRtcpPacket
RTCPReceiver::IncomingPacket
```





## 6. !!! VideoSendStreamImpl::Start

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::Start() {
  RTC_DCHECK_RUN_ON(worker_queue_);
  RTC_LOG(LS_INFO) << "VideoSendStream::Start";
  if (rtp_video_sender_->IsActive())
    return;
  TRACE_EVENT_INSTANT0("webrtc", "VideoSendStream::Start");
  // 修改状态，开始准备发送数据
  //RtpVideoSender rtp_video_sender_
  rtp_video_sender_->SetActive(true);
  // 检测编码的是否正常，以及请求关键帧
  StartupVideoSendStream();
}
```

开始发送数据。

- 修改 RtpVideoSender的状态，最终调用`RTPSender::SetSendingMediaStatus` 发送rtp包；调用`RTCPSender::SetSendingStatus` 发送rtcp包；具体的后文说明；
- 发送关键帧，通过VideoStreamEncoder进行处理。
- **在创建VideoSendStream的时候，就调用了start；**



### 6.1 VideoSendStreamImpl::StartupVideoSendStream

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::StartupVideoSendStream() {
  RTC_DCHECK_RUN_ON(worker_queue_);
  bitrate_allocator_->AddObserver(this, GetAllocationConfig());
  // Start monitoring encoder activity.
  {
    RTC_DCHECK(!check_encoder_activity_task_.Running());

    activity_ = false;
    timed_out_ = false;
    
    // constexpr TimeDelta kEncoderTimeOut = TimeDelta::Seconds(2);
    check_encoder_activity_task_ = RepeatingTaskHandle::DelayedStart(
        worker_queue_->Get(), kEncoderTimeOut, [this] {
          RTC_DCHECK_RUN_ON(worker_queue_);
          // VideoSendStreamImpl::OnEncodedImage 和 VideoSendStreamImpl::OnDroppedFrame
					// 会把 activity_ = true；
          // 
          // 延迟后，activity_ 还是false，就是没有数据编码出来
          // 则表示编码超时
          if (!activity_) {
            if (!timed_out_) {
              SignalEncoderTimedOut();
            }
            timed_out_ = true;
            disable_padding_ = true;
          } else if (timed_out_) {
            SignalEncoderActive();
            timed_out_ = false;
          }
          activity_ = false;
          return kEncoderTimeOut;
        });
  }

  // 发送关键帧
  video_stream_encoder_->SendKeyFrame();
}
```

1. 检测 编码超时，每隔2s执行一次。关于[RepeatingTaskHandle](https://blog.csdn.net/commshare/article/details/131619827);
2. 发送关键帧



### 6.2 activity_

```cpp
EncodedImageCallback::Result VideoSendStreamImpl::OnEncodedImage(
    const EncodedImage& encoded_image,
    const CodecSpecificInfo* codec_specific_info) {
  // Encoded is called on whatever thread the real encoder implementation run
  // on. In the case of hardware encoders, there might be several encoders
  // running in parallel on different threads.

  // Indicate that there still is activity going on.
  activity_ = true;
  ...
  }
```



```cpp
void VideoSendStreamImpl::OnDroppedFrame(
    EncodedImageCallback::DropReason reason) {
  activity_ = true;
}
```



### 6.3 timed_out_



### 6.4 VideoSendStreamImpl::SignalEncoderTimedOut

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::SignalEncoderTimedOut() {
  RTC_DCHECK_RUN_ON(worker_queue_);
  // If the encoder has not produced anything the last kEncoderTimeOut and it
  // is supposed to, deregister as BitrateAllocatorObserver. This can happen
  // if a camera stops producing frames.
  if (encoder_target_rate_bps_ > 0) {
    RTC_LOG(LS_INFO) << "SignalEncoderTimedOut, Encoder timed out.";
    // BitrateAllocatorInterface
    bitrate_allocator_->RemoveObserver(this);
  }
}
```



### 6.5 VideoSendStreamImpl::SignalEncoderActive

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::SignalEncoderActive() {
  RTC_DCHECK_RUN_ON(worker_queue_);
  if (rtp_video_sender_->IsActive()) {
    RTC_LOG(LS_INFO) << "SignalEncoderActive, Encoder is active.";
    // BitrateAllocatorInterface
    bitrate_allocator_->AddObserver(this, GetAllocationConfig());
  }
}
```



### 6.6 !!! VideoSendStreamImpl::GetAllocationConfig

video/video_send_stream_impl.cc

```cpp
MediaStreamAllocationConfig VideoSendStreamImpl::GetAllocationConfig() const {
  return MediaStreamAllocationConfig{
      static_cast<uint32_t>(encoder_min_bitrate_bps_),
      encoder_max_bitrate_bps_,
      static_cast<uint32_t>(disable_padding_ ? 0 : max_padding_bitrate_),
      /* priority_bitrate */ 0,
      !config_->suspend_below_min_bitrate,
      encoder_bitrate_priority_};
}
```



### 6.7 调用堆栈

media/engine/webrtc_video_engine.cc

```less
WebRtcVideoChannel::WebRtcVideoReceiveStream::SetLocalSsrc
///
WebRtcVideoChannel::WebRtcVideoReceiveStream::RecreateWebRtcVideoStream
VideoSendStream::Start
VideoSendStreamImpl::Start	
```





## 7. VideoSendStreamImpl::Stop

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::Stop() {
  RTC_DCHECK_RUN_ON(worker_queue_);
  RTC_LOG(LS_INFO) << "VideoSendStreamImpl::Stop";
  if (!rtp_video_sender_->IsActive())
    return;
  TRACE_EVENT_INSTANT0("webrtc", "VideoSendStream::Stop");
  // 修改发送的状态，修改为停止
  rtp_video_sender_->SetActive(false);
  // 清除一些数据
  StopVideoSendStream();
}
```

修改发送的状态，修改为停止，清除一些数据。

### 7.1 VideoSendStreamImpl::StopVideoSendStream

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::StopVideoSendStream() {
  bitrate_allocator_->RemoveObserver(this);
  check_encoder_activity_task_.Stop();
  // BitrateAllocatorInterface
  video_stream_encoder_->OnBitrateUpdated(DataRate::Zero(), DataRate::Zero(),
                                          DataRate::Zero(), 0, 0, 0);
  stats_proxy_->OnSetEncoderTargetRate(0);
}
```



## 8. VideoSendStreamImpl::UpdateActiveSimulcastLayers

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::UpdateActiveSimulcastLayers(
    const std::vector<bool> active_layers) {
  RTC_DCHECK_RUN_ON(worker_queue_);
  bool previously_active = rtp_video_sender_->IsActive();
  rtp_video_sender_->SetActiveModules(active_layers);
  if (!rtp_video_sender_->IsActive() && previously_active) {
    // Payload router switched from active to inactive.
    StopVideoSendStream();
  } else if (rtp_video_sender_->IsActive() && !previously_active) {
    // Payload router switched from inactive to active.
    StartupVideoSendStream();
  }
}
```

更新simulcast的层的状态。



## ------

## 9. !!! webrtc::BitrateAllocatorObserver

call/bitrate_allocator.h

```cpp
// Used by all send streams with adaptive bitrate, to get the currently
// allocated bitrate for the send stream. The current network properties are
// given at the same time, to let the send stream decide about possible loss
// protection.
class BitrateAllocatorObserver {
 public:
  // Returns the amount of protection used by the BitrateAllocatorObserver
  // implementation, as bitrate in bps.
  virtual uint32_t OnBitrateUpdated(BitrateAllocationUpdate update) = 0;

 protected:
  virtual ~BitrateAllocatorObserver() {}
};
```

VideoSendStreamImpl实现了BitrateAllocatorObserver。回调了码流大小，设置给编码器。



### 9.1 VideoSendStreamImpl::OnBitrateUpdated

video/video_send_stream_impl.cc

```cpp
uint32_t VideoSendStreamImpl::OnBitrateUpdated(BitrateAllocationUpdate update) {
  RTC_DCHECK_RUN_ON(worker_queue_);
  RTC_DCHECK(rtp_video_sender_->IsActive())
      << "VideoSendStream::Start has not been called.";

  // When the BWE algorithm doesn't pass a stable estimate, we'll use the
  // unstable one instead.
  if (update.stable_target_bitrate.IsZero()) {
    update.stable_target_bitrate = update.target_bitrate;
  }

  rtp_video_sender_->OnBitrateUpdated(update, stats_proxy_->GetSendFrameRate());
  encoder_target_rate_bps_ = rtp_video_sender_->GetPayloadBitrateBps();
  const uint32_t protection_bitrate_bps =
      rtp_video_sender_->GetProtectionBitrateBps();
  DataRate link_allocation = DataRate::Zero();
  if (encoder_target_rate_bps_ > protection_bitrate_bps) {
    link_allocation =
        DataRate::BitsPerSec(encoder_target_rate_bps_ - protection_bitrate_bps);
  }
  DataRate overhead =
      update.target_bitrate - DataRate::BitsPerSec(encoder_target_rate_bps_);
  DataRate encoder_stable_target_rate = update.stable_target_bitrate;
  if (encoder_stable_target_rate > overhead) {
    encoder_stable_target_rate = encoder_stable_target_rate - overhead;
  } else {
    encoder_stable_target_rate = DataRate::BitsPerSec(encoder_target_rate_bps_);
  }

  encoder_target_rate_bps_ =
      std::min(encoder_max_bitrate_bps_, encoder_target_rate_bps_);

  encoder_stable_target_rate =
      std::min(DataRate::BitsPerSec(encoder_max_bitrate_bps_),
               encoder_stable_target_rate);

  DataRate encoder_target_rate = DataRate::BitsPerSec(encoder_target_rate_bps_);
  link_allocation = std::max(encoder_target_rate, link_allocation);　
  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  video_stream_encoder_->OnBitrateUpdated(
      encoder_target_rate, encoder_stable_target_rate, link_allocation,
      rtc::dchecked_cast<uint8_t>(update.packet_loss_ratio * 256),
      update.round_trip_time.ms(), update.cwnd_reduce_ratio);
  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  //!!!!!!!!!!!!
  stats_proxy_->OnSetEncoderTargetRate(encoder_target_rate_bps_);
  return protection_bitrate_bps;
}
```



## ------

## 10. VideoStreamEncoderInterface::EncoderSink

api/video/video_stream_encoder_interface.h

```cpp
  // Interface for receiving encoded video frames and notifications about
  // configuration changes.
  class EncoderSink : public EncodedImageCallback {
   public:
    virtual void OnEncoderConfigurationChanged(
        std::vector<VideoStream> streams,
        bool is_svc,
        VideoEncoderConfig::ContentType content_type,
        int min_transmit_bitrate_bps) = 0;

    virtual void OnBitrateAllocationUpdated(
        const VideoBitrateAllocation& allocation) = 0;

    virtual void OnVideoLayersAllocationUpdated(
        VideoLayersAllocation allocation) = 0;
  };
```





### 10.1 VideoSendStreamImpl::OnEncoderConfigurationChanged

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::OnEncoderConfigurationChanged(
    std::vector<VideoStream> streams,
    bool is_svc,
    VideoEncoderConfig::ContentType content_type,
    int min_transmit_bitrate_bps) {
  if (!worker_queue_->IsCurrent()) {
    rtc::WeakPtr<VideoSendStreamImpl> send_stream = weak_ptr_;
    worker_queue_->PostTask([send_stream, streams, is_svc, content_type,
                             min_transmit_bitrate_bps]() mutable {
      if (send_stream) {
        send_stream->OnEncoderConfigurationChanged(
            std::move(streams), is_svc, content_type, min_transmit_bitrate_bps);
      }
    });
    return;
  }

  RTC_DCHECK_GE(config_->rtp.ssrcs.size(), streams.size());
  TRACE_EVENT0("webrtc", "VideoSendStream::OnEncoderConfigurationChanged");
  RTC_DCHECK_RUN_ON(worker_queue_);

  const VideoCodecType codec_type =
      PayloadStringToCodecType(config_->rtp.payload_name);

  const absl::optional<DataRate> experimental_min_bitrate =
      GetExperimentalMinVideoBitrate(codec_type);
  encoder_min_bitrate_bps_ =
      experimental_min_bitrate
          ? experimental_min_bitrate->bps()
          : std::max(streams[0].min_bitrate_bps, kDefaultMinVideoBitrateBps);

  encoder_max_bitrate_bps_ = 0;
  double stream_bitrate_priority_sum = 0;
  for (const auto& stream : streams) {
    // We don't want to allocate more bitrate than needed to inactive streams.
    encoder_max_bitrate_bps_ += stream.active ? stream.max_bitrate_bps : 0;
    if (stream.bitrate_priority) {
      RTC_DCHECK_GT(*stream.bitrate_priority, 0);
      stream_bitrate_priority_sum += *stream.bitrate_priority;
    }
  }
  RTC_DCHECK_GT(stream_bitrate_priority_sum, 0);
  encoder_bitrate_priority_ = stream_bitrate_priority_sum;
  encoder_max_bitrate_bps_ =
      std::max(static_cast<uint32_t>(encoder_min_bitrate_bps_),
               encoder_max_bitrate_bps_);

  // TODO(bugs.webrtc.org/10266): Query the VideoBitrateAllocator instead.
  max_padding_bitrate_ = CalculateMaxPadBitrateBps(
      streams, is_svc, content_type, min_transmit_bitrate_bps,
      config_->suspend_below_min_bitrate, has_alr_probing_);

  // Clear stats for disabled layers.
  for (size_t i = streams.size(); i < config_->rtp.ssrcs.size(); ++i) {
    stats_proxy_->OnInactiveSsrc(config_->rtp.ssrcs[i]);
  }

  const size_t num_temporal_layers =
      streams.back().num_temporal_layers.value_or(1);

  rtp_video_sender_->SetEncodingData(streams[0].width, streams[0].height,
                                     num_temporal_layers);

  if (rtp_video_sender_->IsActive()) {
    // The send stream is started already. Update the allocator with new bitrate
    // limits.
    bitrate_allocator_->AddObserver(this, GetAllocationConfig());
  }
}

```



### 10.2 VideoSendStreamImpl::OnBitrateAllocationUpdated

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::OnBitrateAllocationUpdated(
    const VideoBitrateAllocation& allocation) {
  if (!worker_queue_->IsCurrent()) {
    auto ptr = weak_ptr_;
    worker_queue_->PostTask([=] {
      if (!ptr.get())
        return;
      ptr->OnBitrateAllocationUpdated(allocation);
    });
    return;
  }

  RTC_DCHECK_RUN_ON(worker_queue_);

  int64_t now_ms = clock_->TimeInMilliseconds();
  if (encoder_target_rate_bps_ != 0) {
    if (video_bitrate_allocation_context_) {
      // If new allocation is within kMaxVbaSizeDifferencePercent larger than
      // the previously sent allocation and the same streams are still enabled,
      // it is considered "similar". We do not want send similar allocations
      // more once per kMaxVbaThrottleTimeMs.
      const VideoBitrateAllocation& last =
          video_bitrate_allocation_context_->last_sent_allocation;
      const bool is_similar =
          allocation.get_sum_bps() >= last.get_sum_bps() &&
          allocation.get_sum_bps() <
              (last.get_sum_bps() * (100 + kMaxVbaSizeDifferencePercent)) /
                  100 &&
          SameStreamsEnabled(allocation, last);
      if (is_similar &&
          (now_ms - video_bitrate_allocation_context_->last_send_time_ms) <
              kMaxVbaThrottleTimeMs) {
        // This allocation is too similar, cache it and return.
        video_bitrate_allocation_context_->throttled_allocation = allocation;
        return;
      }
    } else {
      video_bitrate_allocation_context_.emplace();
    }

    video_bitrate_allocation_context_->last_sent_allocation = allocation;
    video_bitrate_allocation_context_->throttled_allocation.reset();
    video_bitrate_allocation_context_->last_send_time_ms = now_ms;

    // Send bitrate allocation metadata only if encoder is not paused.
    rtp_video_sender_->OnBitrateAllocationUpdated(allocation);
  }
}
```



### 10.3 VideoSendStreamImpl::OnVideoLayersAllocationUpdated

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::OnVideoLayersAllocationUpdated(
    VideoLayersAllocation allocation) {
  // OnVideoLayersAllocationUpdated is handled on the encoder task queue in
  // order to not race with OnEncodedImage callbacks.
  rtp_video_sender_->OnVideoLayersAllocationUpdated(allocation);
}
```





## 11. EncodedImageCallback

api/video_codecs/video_encoder.h

```cpp
class RTC_EXPORT EncodedImageCallback {
 public:
  virtual ~EncodedImageCallback() {}

  struct Result {
    enum Error {
      OK,

      // Failed to send the packet.
      ERROR_SEND_FAILED,
    };

    explicit Result(Error error) : error(error) {}
    Result(Error error, uint32_t frame_id) : error(error), frame_id(frame_id) {}

    Error error;

    // Frame ID assigned to the frame. The frame ID should be the same as the ID
    // seen by the receiver for this frame. RTP timestamp of the frame is used
    // as frame ID when RTP is used to send video. Must be used only when
    // error=OK.
    uint32_t frame_id = 0;

    // Tells the encoder that the next frame is should be dropped.
    bool drop_next_frame = false;
  };

  // Used to signal the encoder about reason a frame is dropped.
  // kDroppedByMediaOptimizations - dropped by MediaOptimizations (for rate
  // limiting purposes).
  // kDroppedByEncoder - dropped by encoder's internal rate limiter.
  enum class DropReason : uint8_t {
    kDroppedByMediaOptimizations,
    kDroppedByEncoder
  };

  // Callback function which is called when an image has been encoded.
  virtual Result OnEncodedImage(
      const EncodedImage& encoded_image,
      const CodecSpecificInfo* codec_specific_info) = 0;

  virtual void OnDroppedFrame(DropReason reason) {}
};
```



### 11.1 VideoSendStreamImpl::OnEncodedImage

video/video_send_stream_impl.cc

```cpp
EncodedImageCallback::Result VideoSendStreamImpl::OnEncodedImage(
    const EncodedImage& encoded_image,
    const CodecSpecificInfo* codec_specific_info) {
  // Encoded is called on whatever thread the real encoder implementation run
  // on. In the case of hardware encoders, there might be several encoders
  // running in parallel on different threads.

  // Indicate that there still is activity going on.
  activity_ = true;

  auto enable_padding_task = [this]() {
    if (disable_padding_) {
      RTC_DCHECK_RUN_ON(worker_queue_);
      disable_padding_ = false;
      // To ensure that padding bitrate is propagated to the bitrate allocator.
      SignalEncoderActive();
    }
  };
  if (!worker_queue_->IsCurrent()) {
    worker_queue_->PostTask(enable_padding_task);
  } else {
    enable_padding_task();
  }

  EncodedImageCallback::Result result(EncodedImageCallback::Result::OK);
  result =
      rtp_video_sender_->OnEncodedImage(encoded_image, codec_specific_info);
  // Check if there's a throttled VideoBitrateAllocation that we should try
  // sending.
  rtc::WeakPtr<VideoSendStreamImpl> send_stream = weak_ptr_;
  auto update_task = [send_stream]() {
    if (send_stream) {
      RTC_DCHECK_RUN_ON(send_stream->worker_queue_);
      auto& context = send_stream->video_bitrate_allocation_context_;
      if (context && context->throttled_allocation) {
        send_stream->OnBitrateAllocationUpdated(*context->throttled_allocation);
      }
    }
  };
  if (!worker_queue_->IsCurrent()) {
    worker_queue_->PostTask(update_task);
  } else {
    update_task();
  }

  return result;
}
```



### 11.2 VideoSendStreamImpl::OnDroppedFrame

video/video_send_stream_impl.cc

```cpp
void VideoSendStreamImpl::OnDroppedFrame(
    EncodedImageCallback::DropReason reason) {
  activity_ = true;
}
```


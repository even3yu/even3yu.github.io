---
layout: post
title: 从Call::CreateVideoSendStream 到 RTPSender创建流程
date: 2024-03-11 11:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc stream video
---


* content
{:toc}

---


## 0. 前言

1. 有多少个mline，就生成多少个webrtc::VideoSendStream，子类就是webrtc::internal::VideoSendStream; 在Call中以第一个ssrc为key，webrtc::VideoSendStream为value，存在map中；**注意**， 只有主流的ssrc 对应生成webrtc::VideoSendStream，其他比如simulcast，rtx，felxfec都是不会生成webrtc::VideoSendStream。
2. 一个Call 对应会生成一个RtpTransportControllerSend，call所管理的流共用一个码流分配管理；
3. 一个webrtc::VideoSendStream 对应拥有一个VideoSendStreamImpl；
4. 通过RtpTransportControllerSend 创建了RtpVideoSender，存放在VideoSendStreamImpl中
5. webrtc_internal_rtp_video_sender::RtpStreamSender 是在 RtpVideoSender创建；webrtc_internal_rtp_video_sender::RtpStreamSender是和RtpConfig::ssrcs的个数相关，就是和simulcast的层级对应，有多少层sumlcast layer，就有多少个webrtc_internal_rtp_video_sender::RtpStreamSender；
6. 一个 webrtc_internal_rtp_video_sender::RtpStreamSender， 对应一个RTPSenderVideo和ModuleRtpRtcpModule2
7. 

## 1. 类关系图

![img]({{ site.url }}{{ site.baseurl }}/images/RTPSender-video-create-stack.assets/class.png)



![img]({{ site.url }}{{ site.baseurl }}/images/RTPSender-video-create-stack.assets/class2.png)

## 2. 调用堆栈

```less
WebRtcVideoChannel::WebRtcVideoSendStream::SetFrameEncryptor 
/ WebRtcVideoChannel::WebRtcVideoSendStream::SetCodec
/ WebRtcVideoChannel::WebRtcVideoSendStream::SetSendParameters
/ WebRtcVideoChannel::WebRtcVideoSendStream::SetEncoderToPacketizerFrameTransformer

WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream
```



```less
WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream

Call::CreateVideoSendStream
VideoSendStream::VideoSendStream
VideoSendStreamImpl::VideoSendStreamImpl
RtpTransportControllerSend::CreateRtpVideoSender
RtpVideoSender::RtpVideoSender
CreateRtpStreamSenders  // 创建了RTPSenderVideo，主要和RtpVideoSender 区别
	ModuleRtpRtcpImpl2::ModuleRtpRtcpImpl2
		ModuleRtpRtcpImpl2::RtpSenderContext::RtpSenderContext
			RTPSender::RTPSender
	RtpStreamSender::RtpStreamSender
  RTPSenderVideo::RTPSenderVideo
```



### 2.1 --WebRtcVideoSendStream::RecreateWebRtcStream



## 3. A. Call::CreateVideoSendStream

call/call.cc

```cpp
// This method can be used for Call tests with external fec controller factory.
webrtc::VideoSendStream* Call::CreateVideoSendStream(
    webrtc::VideoSendStream::Config config,
    VideoEncoderConfig encoder_config,
    std::unique_ptr<FecController> fec_controller) {
  TRACE_EVENT0("webrtc", "Call::CreateVideoSendStream");
  RTC_DCHECK_RUN_ON(worker_thread_);

  EnsureStarted();

  video_send_delay_stats_->AddSsrcs(config);
  for (size_t ssrc_index = 0; ssrc_index < config.rtp.ssrcs.size();
       ++ssrc_index) {
    event_log_->Log(std::make_unique<RtcEventVideoSendStreamConfig>(
        CreateRtcLogStreamConfig(config, ssrc_index)));
  }

  // TODO(mflodman): Base the start bitrate on a current bandwidth estimate, if
  // the call has already started.
  // Copy ssrcs from |config| since |config| is moved.
  std::vector<uint32_t> ssrcs = config.rtp.ssrcs;

            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
  	    // webrtc::internal::VideoSendStream
  VideoSendStream* send_stream = new VideoSendStream(
      clock_, num_cpu_cores_, module_process_thread_->process_thread(),
      task_queue_factory_, call_stats_->AsRtcpRttStats(),
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
      transport_send_ptr_,
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
      bitrate_allocator_.get(), video_send_delay_stats_.get(), event_log_,
      std::move(config), std::move(encoder_config), suspended_video_send_ssrcs_,
      suspended_video_payload_states_, std::move(fec_controller));
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!

  for (uint32_t ssrc : ssrcs) {
    RTC_DCHECK(video_send_ssrcs_.find(ssrc) == video_send_ssrcs_.end());
    video_send_ssrcs_[ssrc] = send_stream;
  }
  video_send_streams_.insert(send_stream);
  // Forward resources that were previously added to the call to the new stream.
  for (const auto& resource_forwarder : adaptation_resource_forwarders_) {
    resource_forwarder->OnCreateVideoSendStream(send_stream);
  }

  UpdateAggregateNetworkState();

  return send_stream;
}
```





### 3.1 Call::Create ——创建RtpTransportControllerSend

call\call.cc

```cpp
Call* Call::Create(const Call::Config& config,
                   Clock* clock,
                   rtc::scoped_refptr<SharedModuleThread> call_thread,
                   std::unique_ptr<ProcessThread> pacer_thread) {
  RTC_DCHECK(config.task_queue_factory);
  return new internal::Call(
      clock, config,
      
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
      std::make_unique<RtpTransportControllerSend>(
          clock, config.event_log, config.network_state_predictor_factory,
          config.network_controller_factory, config.bitrate_config,
          std::move(pacer_thread), config.task_queue_factory, config.trials),
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
      std::move(call_thread), config.task_queue_factory);
}
```



```cpp
// Caches transport_send_.get(), to avoid racing with destructor.
  // Note that this is declared before transport_send_ to ensure that it is not
  // invalidated until no more tasks can be running on the transport_send_ task
  // queue.
  RtpTransportControllerSendInterface* const transport_send_ptr_;
  // Declared last since it will issue callbacks from a task queue. Declaring it
  // last ensures that it is destroyed first and any running tasks are finished.
  std::unique_ptr<RtpTransportControllerSendInterface> transport_send_;
```



### -- 3.2 VideoSendStream::VideoSendStream



## 4. B. VideoSendStream::VideoSendStream

video/video_send_stream.cc

```cpp
VideoSendStream::VideoSendStream(
    Clock* clock,
    int num_cpu_cores,
    ProcessThread* module_process_thread,
    TaskQueueFactory* task_queue_factory,
    RtcpRttStats* call_stats,
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
    RtpTransportControllerSendInterface* transport,
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
    BitrateAllocatorInterface* bitrate_allocator,
    SendDelayStats* send_delay_stats,
    RtcEventLog* event_log,
    VideoSendStream::Config config,
    VideoEncoderConfig encoder_config,
    const std::map<uint32_t, RtpState>& suspended_ssrcs,
    const std::map<uint32_t, RtpPayloadState>& suspended_payload_states,
    std::unique_ptr<FecController> fec_controller)
    : worker_queue_(transport->GetWorkerQueue()),
      stats_proxy_(clock, config, encoder_config.content_type),
      config_(std::move(config)),
      content_type_(encoder_config.content_type) {
  RTC_DCHECK(config_.encoder_settings.encoder_factory);
  RTC_DCHECK(config_.encoder_settings.bitrate_allocator_factory);

  video_stream_encoder_ = std::make_unique<VideoStreamEncoder>(
      clock, num_cpu_cores, &stats_proxy_, config_.encoder_settings,
      std::make_unique<OveruseFrameDetector>(&stats_proxy_), task_queue_factory,
      GetBitrateAllocationCallbackType(config_));

  // TODO(srte): Initialization should not be done posted on a task queue.
  // Note that the posted task must not outlive this scope since the closure
  // references local variables.
  worker_queue_->PostTask(ToQueuedTask(
      [this, clock, call_stats, transport, bitrate_allocator, send_delay_stats,
       event_log, &suspended_ssrcs, &encoder_config, &suspended_payload_states,
       &fec_controller]() {
        send_stream_.reset(new VideoSendStreamImpl(
            clock, &stats_proxy_, worker_queue_, call_stats, 
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            transport,
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            //!!!!!!!!!!!!!!!!!!!!!
            bitrate_allocator, send_delay_stats, video_stream_encoder_.get(),
            event_log, &config_, encoder_config.max_bitrate_bps,
            encoder_config.bitrate_priority, suspended_ssrcs,
            suspended_payload_states, encoder_config.content_type,
            std::move(fec_controller)));
      },
      [this]() { thread_sync_event_.Set(); }));

  // Wait for ConstructionTask to complete so that |send_stream_| can be used.
  // |module_process_thread| must be registered and deregistered on the thread
  // it was created on.
  thread_sync_event_.Wait(rtc::Event::kForever);
  send_stream_->RegisterProcessThread(module_process_thread);
  ReconfigureVideoEncoder(std::move(encoder_config));
}
```



### -- 4.1 VideoSendStreamImpl::VideoSendStreamImpl



## 5. VideoSendStreamImpl::VideoSendStreamImpl

video/video_send_stream_impl.cc

```cpp
VideoSendStreamImpl::VideoSendStreamImpl(...){
// RtpVideoSenderInterface* const rtp_video_sender_;
// 就是RtpVideoSender
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
}
```



### -- 5.1 RtpTransportControllerSend::CreateRtpVideoSender



## 6. C. RtpTransportControllerSend::CreateRtpVideoSender

call/rtp_transport_controller_send.cc

```cpp
RtpVideoSenderInterface* RtpTransportControllerSend::CreateRtpVideoSender(
    std::map<uint32_t, RtpState> suspended_ssrcs,
    const std::map<uint32_t, RtpPayloadState>& states,
    const RtpConfig& rtp_config,
    int rtcp_report_interval_ms,
    Transport* send_transport,
    const RtpSenderObservers& observers,
    RtcEventLog* event_log,
    std::unique_ptr<FecController> fec_controller,
    const RtpSenderFrameEncryptionConfig& frame_encryption_config,
    rtc::scoped_refptr<FrameTransformerInterface> frame_transformer) {
  video_rtp_senders_.push_back(std::make_unique<RtpVideoSender>(
      clock_, suspended_ssrcs, states, rtp_config, rtcp_report_interval_ms,
      send_transport, observers,
      // TODO(holmer): Remove this circular dependency by injecting
      // the parts of RtpTransportControllerSendInterface that are really used.
      this, event_log, &retransmission_rate_limiter_, std::move(fec_controller),
      frame_encryption_config.frame_encryptor,
      frame_encryption_config.crypto_options, std::move(frame_transformer)));
  return video_rtp_senders_.back().get();
}
```



### -- 6.1 RtpVideoSender::RtpVideoSender



## 7. RtpVideoSender::RtpVideoSender

call/rtp_video_sender.cc

> 注意区别：RtpVideoSender，RTPSenderVideo
>
> 

```cpp
RtpVideoSender::RtpVideoSender(
    Clock* clock,
    std::map<uint32_t, RtpState> suspended_ssrcs,
    const std::map<uint32_t, RtpPayloadState>& states,
    const RtpConfig& rtp_config,
    int rtcp_report_interval_ms,
    Transport* send_transport,
    const RtpSenderObservers& observers,
    RtpTransportControllerSendInterface* transport,
    RtcEventLog* event_log,
    RateLimiter* retransmission_limiter,
    std::unique_ptr<FecController> fec_controller,
    FrameEncryptorInterface* frame_encryptor,
    const CryptoOptions& crypto_options,
    rtc::scoped_refptr<FrameTransformerInterface> frame_transformer)
    : send_side_bwe_with_overhead_(!absl::StartsWith(
          field_trials_.Lookup("WebRTC-SendSideBwe-WithOverhead"),
          "Disabled")),
      use_frame_rate_for_overhead_(absl::StartsWith(
          field_trials_.Lookup("WebRTC-Video-UseFrameRateForOverhead"),
          "Enabled")),
      has_packet_feedback_(TransportSeqNumExtensionConfigured(rtp_config)),
      active_(false),
      module_process_thread_(nullptr),
      suspended_ssrcs_(std::move(suspended_ssrcs)),
      fec_controller_(std::move(fec_controller)),
      fec_allowed_(true),
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
// Rtp modules are assumed to be sorted in simulcast index order.
//  const std::vector<webrtc_internal_rtp_video_sender::RtpStreamSender>
//      rtp_streams_;
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
// 注意区别：RtpVideoSender，RTPSenderVideo
      rtp_streams_(CreateRtpStreamSenders(clock,
                                          rtp_config,
                                          observers,
                                          rtcp_report_interval_ms,
                                          send_transport,
                                          transport->GetBandwidthObserver(),
                                          transport,
                                          suspended_ssrcs_,
                                          event_log,
                                          retransmission_limiter,
                                          frame_encryptor,
                                          crypto_options,
                                          std::move(frame_transformer),
                                          field_trials_)),
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
//!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
      rtp_config_(rtp_config),
      codec_type_(GetVideoCodecType(rtp_config)),
      transport_(transport),
      transport_overhead_bytes_per_packet_(0),
      encoder_target_rate_bps_(0),
      frame_counts_(rtp_config.ssrcs.size()),
      frame_count_observer_(observers.frame_count_observer) {
  RTC_DCHECK_EQ(rtp_config_.ssrcs.size(), rtp_streams_.size());
  if (send_side_bwe_with_overhead_ && has_packet_feedback_)
    transport_->IncludeOverheadInPacedSender();
  module_process_thread_checker_.Detach();
  // SSRCs are assumed to be sorted in the same order as |rtp_modules|.
  for (uint32_t ssrc : rtp_config_.ssrcs) {
    // Restore state if it previously existed.
    const RtpPayloadState* state = nullptr;
    auto it = states.find(ssrc);
    if (it != states.end()) {
      state = &it->second;
      shared_frame_id_ = std::max(shared_frame_id_, state->shared_frame_id);
    }
    params_.push_back(RtpPayloadParams(ssrc, state, field_trials_));
  }

  // RTP/RTCP initialization.

  // We add the highest spatial layer first to ensure it'll be prioritized
  // when sending padding, with the hope that the packet rate will be smaller,
  // and that it's more important to protect than the lower layers.

  // TODO(nisse): Consider moving registration with PacketRouter last, after the
  // modules are fully configured.
  for (const RtpStreamSender& stream : rtp_streams_) {
    constexpr bool remb_candidate = true;
    transport->packet_router()->AddSendRtpModule(stream.rtp_rtcp.get(),
                                                 remb_candidate);
  }

  for (size_t i = 0; i < rtp_config_.extensions.size(); ++i) {
    const std::string& extension = rtp_config_.extensions[i].uri;
    int id = rtp_config_.extensions[i].id;
    RTC_DCHECK(RtpExtension::IsSupportedForVideo(extension));
    for (const RtpStreamSender& stream : rtp_streams_) {
      stream.rtp_rtcp->RegisterRtpHeaderExtension(extension, id);
    }
  }

  ConfigureSsrcs();
  ConfigureRids();

  if (!rtp_config_.mid.empty()) {
    for (const RtpStreamSender& stream : rtp_streams_) {
      stream.rtp_rtcp->SetMid(rtp_config_.mid);
    }
  }

  bool fec_enabled = false;
  for (const RtpStreamSender& stream : rtp_streams_) {
    // Simulcast has one module for each layer. Set the CNAME on all modules.
    stream.rtp_rtcp->SetCNAME(rtp_config_.c_name.c_str());
    stream.rtp_rtcp->SetMaxRtpPacketSize(rtp_config_.max_packet_size);
    stream.rtp_rtcp->RegisterSendPayloadFrequency(rtp_config_.payload_type,
                                                  kVideoPayloadTypeFrequency);
    if (stream.fec_generator != nullptr) {
      fec_enabled = true;
    }
  }
  // Currently, both ULPFEC and FlexFEC use the same FEC rate calculation logic,
  // so enable that logic if either of those FEC schemes are enabled.
  fec_controller_->SetProtectionMethod(fec_enabled, NackEnabled());

  fec_controller_->SetProtectionCallback(this);
  // Signal congestion controller this object is ready for OnPacket* callbacks.
  transport_->GetStreamFeedbackProvider()->RegisterStreamFeedbackObserver(
      rtp_config_.ssrcs, this);
}
```



### -- 7.1 CreateRtpStreamSenders



## 8. !!!D. CreateRtpStreamSenders

call/rtp_video_sender.cc

```cpp
std::vector<RtpStreamSender> CreateRtpStreamSenders(
    Clock* clock,
    const RtpConfig& rtp_config,
    const RtpSenderObservers& observers,
    int rtcp_report_interval_ms,
    Transport* send_transport,
    RtcpBandwidthObserver* bandwidth_callback,
    RtpTransportControllerSendInterface* transport,
    const std::map<uint32_t, RtpState>& suspended_ssrcs,
    RtcEventLog* event_log,
    RateLimiter* retransmission_rate_limiter,
    FrameEncryptorInterface* frame_encryptor,
    const CryptoOptions& crypto_options,
    rtc::scoped_refptr<FrameTransformerInterface> frame_transformer,
    const WebRtcKeyValueConfig& trials) {
  RTC_DCHECK_GT(rtp_config.ssrcs.size(), 0);

  RtpRtcpInterface::Configuration configuration;
  configuration.clock = clock;
  configuration.audio = false;
  configuration.receiver_only = false;
  configuration.outgoing_transport = send_transport;
  configuration.intra_frame_callback = observers.intra_frame_callback;
  configuration.rtcp_loss_notification_observer =
      observers.rtcp_loss_notification_observer;
  configuration.bandwidth_callback = bandwidth_callback;
  configuration.network_state_estimate_observer =
      transport->network_state_estimate_observer();
  configuration.transport_feedback_callback =
      transport->transport_feedback_observer();
  configuration.rtt_stats = observers.rtcp_rtt_stats;
  configuration.rtcp_packet_type_counter_observer =
      observers.rtcp_type_observer;
  configuration.rtcp_statistics_callback = observers.rtcp_stats;
  configuration.report_block_data_observer =
      observers.report_block_data_observer;
  configuration.paced_sender = transport->packet_sender();
  configuration.send_bitrate_observer = observers.bitrate_observer;
  configuration.send_side_delay_observer = observers.send_delay_observer;
  configuration.send_packet_observer = observers.send_packet_observer;
  configuration.event_log = event_log;
  configuration.retransmission_rate_limiter = retransmission_rate_limiter;
  configuration.rtp_stats_callback = observers.rtp_stats;
  configuration.frame_encryptor = frame_encryptor;
  configuration.require_frame_encryption =
      crypto_options.sframe.require_frame_encryption;
  configuration.extmap_allow_mixed = rtp_config.extmap_allow_mixed;
  configuration.rtcp_report_interval_ms = rtcp_report_interval_ms;
  configuration.field_trials = &trials;

  std::vector<RtpStreamSender> rtp_streams;

  RTC_DCHECK(rtp_config.rtx.ssrcs.empty() ||
             rtp_config.rtx.ssrcs.size() == rtp_config.ssrcs.size());
  //!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!
  // 每层simulcast创建一个ssrc，存放在rtp_config.ssrcs中
  for (size_t i = 0; i < rtp_config.ssrcs.size(); ++i) {
      
    RTPSenderVideo::Config video_config;
    configuration.local_media_ssrc = rtp_config.ssrcs[i];

    std::unique_ptr<VideoFecGenerator> fec_generator =
        MaybeCreateFecGenerator(clock, rtp_config, suspended_ssrcs, i, trials);
    configuration.fec_generator = fec_generator.get();

    configuration.rtx_send_ssrc =
        rtp_config.GetRtxSsrcAssociatedWithMediaSsrc(rtp_config.ssrcs[i]);
    RTC_DCHECK_EQ(configuration.rtx_send_ssrc.has_value(),
                  !rtp_config.rtx.ssrcs.empty());

    configuration.need_rtp_packet_infos = rtp_config.lntf.enabled;

    // 创建了 ModuleRtpRtcpImpl2
    std::unique_ptr<ModuleRtpRtcpImpl2> rtp_rtcp(
        ModuleRtpRtcpImpl2::Create(configuration));
    rtp_rtcp->SetSendingStatus(false);
    rtp_rtcp->SetSendingMediaStatus(false);
    rtp_rtcp->SetRTCPStatus(RtcpMode::kCompound);
    // Set NACK.
    rtp_rtcp->SetStorePacketsStatus(true, kMinSendSidePacketHistorySize);

    video_config.clock = configuration.clock;
    video_config.rtp_sender = rtp_rtcp->RtpSender();
    video_config.frame_encryptor = frame_encryptor;
    video_config.require_frame_encryption =
        crypto_options.sframe.require_frame_encryption;
    video_config.enable_retransmit_all_layers = false;
    video_config.field_trials = &trials;

    const bool using_flexfec =
        fec_generator &&
        fec_generator->GetFecType() == VideoFecGenerator::FecType::kFlexFec;
    const bool should_disable_red_and_ulpfec =
        ShouldDisableRedAndUlpfec(using_flexfec, rtp_config, trials);
    if (!should_disable_red_and_ulpfec &&
        rtp_config.ulpfec.red_payload_type != -1) {
      video_config.red_payload_type = rtp_config.ulpfec.red_payload_type;
    }
    if (fec_generator) {
      video_config.fec_type = fec_generator->GetFecType();
      video_config.fec_overhead_bytes = fec_generator->MaxPacketOverhead();
    }
    video_config.frame_transformer = frame_transformer;
    video_config.send_transport_queue = transport->GetWorkerQueue()->Get();
    //!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!
    // 创建 RTPSenderVideo
    auto sender_video = std::make_unique<RTPSenderVideo>(video_config);
    //!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!
    // 创建 RtpStreamSender
    rtp_streams.emplace_back(std::move(rtp_rtcp), std::move(sender_video),
                             std::move(fec_generator));
  }
  return rtp_streams;
}
```







### -- 8.1 ModuleRtpRtcpImpl2::ModuleRtpRtcpImpl2



### -- 8.2 RtpStreamSender::RtpStreamSender



### -- 8.3 RTPSenderVideo::RTPSenderVideo

## --------

## 9. E. ModuleRtpRtcpImpl2::ModuleRtpRtcpImpl2

modules/rtp_rtcp/source/rtp_rtcp_impl2.cc

```cpp
// static
std::unique_ptr<ModuleRtpRtcpImpl2> ModuleRtpRtcpImpl2::Create(
    const Configuration& configuration) {
  RTC_DCHECK(configuration.clock);
  RTC_DCHECK(TaskQueueBase::Current());
  return std::make_unique<ModuleRtpRtcpImpl2>(configuration);
}
```



```cpp
ModuleRtpRtcpImpl2::ModuleRtpRtcpImpl2(const Configuration& configuration)
    : worker_queue_(TaskQueueBase::Current()),
      rtcp_sender_(configuration),
      rtcp_receiver_(configuration, this),
      clock_(configuration.clock),
      last_rtt_process_time_(clock_->TimeInMilliseconds()),
      next_process_time_(clock_->TimeInMilliseconds() +
                         kRtpRtcpMaxIdleTimeProcessMs),
      packet_overhead_(28),  // IPV4 UDP.
      nack_last_time_sent_full_ms_(0),
      nack_last_seq_number_sent_(0),
      remote_bitrate_(configuration.remote_bitrate_estimator),
      rtt_stats_(configuration.rtt_stats),
      rtt_ms_(0) {
  RTC_DCHECK(worker_queue_);
  process_thread_checker_.Detach();
  if (!configuration.receiver_only) {
    //!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!
    // 创建 RtpSenderContext
    rtp_sender_ = std::make_unique<RtpSenderContext>(configuration);
    // Make sure rtcp sender use same timestamp offset as rtp sender.
    rtcp_sender_.SetTimestampOffset(
        rtp_sender_->packet_generator.TimestampOffset());
  }

  // Set default packet size limit.
  // TODO(nisse): Kind-of duplicates
  // webrtc::VideoSendStream::Config::Rtp::kDefaultMaxPacketSize.
  const size_t kTcpOverIpv4HeaderSize = 40;
  SetMaxRtpPacketSize(IP_PACKET_SIZE - kTcpOverIpv4HeaderSize);

  if (rtt_stats_) {
    rtt_update_task_ = RepeatingTaskHandle::DelayedStart(
        worker_queue_, kRttUpdateInterval, [this]() {
          PeriodicUpdate();
          return kRttUpdateInterval;
        });
  }
}
```



### -- 9.1 ModuleRtpRtcpImpl2::RtpSenderContext::RtpSenderContext



## 10. F. ModuleRtpRtcpImpl2::RtpSenderContext::RtpSenderContext

modules/rtp_rtcp/source/rtp_rtcp_impl2.cc

```cpp
ModuleRtpRtcpImpl2::RtpSenderContext::RtpSenderContext(
    const RtpRtcpInterface::Configuration& config)
    : packet_history(config.clock, config.enable_rtx_padding_prioritization),
      packet_sender(config, &packet_history),
      non_paced_sender(&packet_sender, this),
			// RTPSender
      packet_generator(
          config,
          &packet_history,
          config.paced_sender ? config.paced_sender : &non_paced_sender) {}
```



### RtpSenderContext

```cpp
 struct RtpSenderContext : public SequenceNumberAssigner {
    explicit RtpSenderContext(const RtpRtcpInterface::Configuration& config);
    void AssignSequenceNumber(RtpPacketToSend* packet) override;
    // Storage of packets, for retransmissions and padding, if applicable.
    RtpPacketHistory packet_history;
    // Handles final time timestamping/stats/etc and handover to Transport.
    RtpSenderEgress packet_sender;
    // If no paced sender configured, this class will be used to pass packets
    // from |packet_generator_| to |packet_sender_|.
    RtpSenderEgress::NonPacedPacketSender non_paced_sender;
    // Handles creation of RTP packets to be sent.
    RTPSender packet_generator;
  };
```



### --RTPSender::RTPSender



## 11. RTPSender::RTPSender

modules/rtp_rtcp/source/rtp_sender.cc

```cpp
RTPSender::RTPSender(const RtpRtcpInterface::Configuration& config,
                     RtpPacketHistory* packet_history,
                     RtpPacketSender* packet_sender)
    : clock_(config.clock),
      random_(clock_->TimeInMicroseconds()),
      audio_configured_(config.audio),
      ssrc_(config.local_media_ssrc),
      rtx_ssrc_(config.rtx_send_ssrc),
      flexfec_ssrc_(config.fec_generator ? config.fec_generator->FecSsrc()
                                         : absl::nullopt),
      max_padding_size_factor_(GetMaxPaddingSizeFactor(config.field_trials)),
      packet_history_(packet_history),
      paced_sender_(packet_sender),
      sending_media_(true),                   // Default to sending media.
      max_packet_size_(IP_PACKET_SIZE - 28),  // Default is IP-v4/UDP.
      last_payload_type_(-1),
      rtp_header_extension_map_(config.extmap_allow_mixed),
      max_media_packet_header_(kRtpHeaderSize),
      max_padding_fec_packet_header_(kRtpHeaderSize),
      // RTP variables
      sequence_number_forced_(false),
      always_send_mid_and_rid_(config.always_send_mid_and_rid),
      ssrc_has_acked_(false),
      rtx_ssrc_has_acked_(false),
      last_rtp_timestamp_(0),
      capture_time_ms_(0),
      last_timestamp_time_ms_(0),
      last_packet_marker_bit_(false),
      csrcs_(),
      rtx_(kRtxOff),
      supports_bwe_extension_(false),
      retransmission_rate_limiter_(config.retransmission_rate_limiter) {
  // This random initialization is not intended to be cryptographic strong.
  timestamp_offset_ = random_.Rand<uint32_t>();
  // Random start, 16 bits. Can't be 0.
  sequence_number_rtx_ = random_.Rand(1, kMaxInitRtpSeqNumber);
  sequence_number_ = random_.Rand(1, kMaxInitRtpSeqNumber);

  RTC_DCHECK(paced_sender_);
  RTC_DCHECK(packet_history_);
}
```



## --------



## 12. RtpStreamSender::RtpStreamSender

call/rtp_video_sender.h

```cpp
RtpStreamSender::RtpStreamSender(
    std::unique_ptr<ModuleRtpRtcpImpl2> rtp_rtcp,
    std::unique_ptr<RTPSenderVideo> sender_video,
    std::unique_ptr<VideoFecGenerator> fec_generator)
    : rtp_rtcp(std::move(rtp_rtcp)),
			// RTPSenderVideo
      sender_video(std::move(sender_video)),
      fec_generator(std::move(fec_generator)) {}
```



### 12.1 RtpStreamSender

```cpp
namespace webrtc_internal_rtp_video_sender {
// RTP state for a single simulcast stream. Internal to the implementation of
// RtpVideoSender.
struct RtpStreamSender {
  RtpStreamSender(std::unique_ptr<ModuleRtpRtcpImpl2> rtp_rtcp,
                  std::unique_ptr<RTPSenderVideo> sender_video,
                  std::unique_ptr<VideoFecGenerator> fec_generator);
  ~RtpStreamSender();

  RtpStreamSender(RtpStreamSender&&) = default;
  RtpStreamSender& operator=(RtpStreamSender&&) = default;

  // Note: Needs pointer stability.
  std::unique_ptr<ModuleRtpRtcpImpl2> rtp_rtcp;
  std::unique_ptr<RTPSenderVideo> sender_video;
  std::unique_ptr<VideoFecGenerator> fec_generator;
};

}  // namespace webrtc_internal_rtp_video_sender
```





## 13. RTPSenderVideo::RTPSenderVideo

modules/rtp_rtcp/source/rtp_sender_video.cc

```cpp
RTPSenderVideo::RTPSenderVideo(const Config& config)
  	// RTPSender rtp_sender_
    : rtp_sender_(config.rtp_sender),
      clock_(config.clock),
      retransmission_settings_(
          config.enable_retransmit_all_layers
              ? kRetransmitAllLayers
              : (kRetransmitBaseLayer | kConditionallyRetransmitHigherLayers)),
      last_rotation_(kVideoRotation_0),
      transmit_color_space_next_frame_(false),
      send_allocation_(SendVideoLayersAllocation::kDontSend),
      current_playout_delay_{-1, -1},
      playout_delay_pending_(false),
      forced_playout_delay_(LoadVideoPlayoutDelayOverride(config.field_trials)),
      red_payload_type_(config.red_payload_type),
      fec_type_(config.fec_type),
      fec_overhead_bytes_(config.fec_overhead_bytes),
      packetization_overhead_bitrate_(1000, RateStatistics::kBpsScale),
      frame_encryptor_(config.frame_encryptor),
      require_frame_encryption_(config.require_frame_encryption),
      generic_descriptor_auth_experiment_(!absl::StartsWith(
          config.field_trials->Lookup("WebRTC-GenericDescriptorAuth"),
          "Disabled")),
      absolute_capture_time_sender_(config.clock),
      frame_transformer_delegate_(
          config.frame_transformer
              ? new rtc::RefCountedObject<
                    RTPSenderVideoFrameTransformerDelegate>(
                    this,
                    config.frame_transformer,
                    rtp_sender_->SSRC(),
                    config.send_transport_queue)
              : nullptr),
      include_capture_clock_offset_(absl::StartsWith(
          config.field_trials->Lookup(kIncludeCaptureClockOffset),
          "Enabled")) {
  if (frame_transformer_delegate_)
    frame_transformer_delegate_->Init();
}
```


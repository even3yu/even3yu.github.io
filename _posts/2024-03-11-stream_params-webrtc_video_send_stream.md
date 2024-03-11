---
layout: post
title: StreamParams 和 WebRtcVideoSendStream，webrtc::VideoSendStream 关系
date: 2024-03-1 10:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc stream
---


* content
{:toc}

---

## 0. 前言

- StreamParams  是SenderOptions，他们是一一对应的。
- 而SenderOptions 和 RtpSenderInternal 一一对应，
- 所以WebRtcVideoSendStream  和 也是 一一对应StreamParams  。
- 一个RtpSenderInternal  就对应一个WebRtcVideoSendStream  。
- webrtc::VideoSendStream 和 WebRtcVideoSendStream  也是一一对应。
- webrtc::VideoSendStream是一个抽象类，实现类是webrtc::internal::VideoSendStream。
- WebRtcVideoSendStream中有一个map是用来保存webrtc::VideoSendStream，ley就是ssrc。



## 1. WebRtcVideoChannel::AddSendStream

创建了WebRtcVideoSendStream。

media/engine/webrtc_video_engine.cc

```cpp
bool WebRtcVideoChannel::AddSendStream(const StreamParams& sp) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  RTC_LOG(LS_INFO) << "AddSendStream: " << sp.ToString();
  if (!ValidateStreamParams(sp))
    return false;

  if (!ValidateSendSsrcAvailability(sp))
    return false;

  for (uint32_t used_ssrc : sp.ssrcs)
    send_ssrcs_.insert(used_ssrc);

  webrtc::VideoSendStream::Config config(this);

  for (const RidDescription& rid : sp.rids()) {
    config.rtp.rids.push_back(rid.rid);
  }

  config.suspend_below_min_bitrate = video_config_.suspend_below_min_bitrate;
  config.periodic_alr_bandwidth_probing =
      video_config_.periodic_alr_bandwidth_probing;
  config.encoder_settings.experiment_cpu_load_estimator =
      video_config_.experiment_cpu_load_estimator;
  config.encoder_settings.encoder_factory = encoder_factory_;
  config.encoder_settings.bitrate_allocator_factory =
      bitrate_allocator_factory_;
  config.encoder_settings.encoder_switch_request_callback = this;
  config.crypto_options = crypto_options_;
  config.rtp.extmap_allow_mixed = ExtmapAllowMixed();
  config.rtcp_report_interval_ms = video_config_.rtcp_report_interval_ms;

  //!!!!!!!!!!!!!!!!!!!!!!!
  WebRtcVideoSendStream* stream = new WebRtcVideoSendStream(
      call_, sp, std::move(config), default_send_options_,
      video_config_.enable_cpu_adaptation, bitrate_config_.max_bitrate_bps,
      send_codec_, send_rtp_extensions_, send_params_);

  uint32_t ssrc = sp.first_ssrc();
  RTC_DCHECK(ssrc != 0);
  // Using priry-ssrc (first ssrc) as key.
  // std::map<ui// nt32_t, WebRtcVideoSendStream*> send_streams_
  send_streams_[ssrc] = stream;
  //!!!!!!!!!!!!!!!!!!!!!!!

  if (rtcp_receiver_report_ssrc_ == kDefaultRtcpReceiverReportSsrc) {
    rtcp_receiver_report_ssrc_ = ssrc;
    RTC_LOG(LS_INFO)
        << "SetLocalSsrc on all the receive streams because we added "
           "a send stream.";
    for (auto& kv : receive_streams_)
      kv.second->SetLocalSsrc(ssrc);
  }
  if (sending_) {
    stream->SetSend(true);
  }

  return true;
}
```



## ------

## 2. WebRtcVideoSendStream::RecreateWebRtcStream

media/engine/webrtc_video_engine.cc

 webrtc::VideoSendStream* stream_ 是 WebRtcVideoSendStream的属性

```cpp
class WebRtcVideoSendStream {
 ...

    webrtc::VideoSendStream* stream_ RTC_GUARDED_BY(&thread_checker_);
 ...
 }
```



创建了webrtc::VideoSendStream* stream_

```cpp

void WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream() {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  if (stream_ != NULL) {
    call_->DestroyVideoSendStream(stream_);
  }

  RTC_CHECK(parameters_.codec_settings);
  RTC_DCHECK_EQ((parameters_.encoder_config.content_type ==
                 webrtc::VideoEncoderConfig::ContentType::kScreen),
                parameters_.options.is_screencast.value_or(false))
      << "encoder content type inconsistent with screencast option";
  parameters_.encoder_config.encoder_specific_settings =
      ConfigureVideoEncoderSettings(parameters_.codec_settings->codec);

  webrtc::VideoSendStream::Config config = parameters_.config.Copy();
  if (!config.rtp.rtx.ssrcs.empty() && config.rtp.rtx.payload_type == -1) {
    RTC_LOG(LS_WARNING) << "RTX SSRCs configured but there's no configured RTX "
                           "payload type the set codec. Ignoring RTX.";
    config.rtp.rtx.ssrcs.clear();
  }
  if (parameters_.encoder_config.number_of_streams == 1) {
    // SVC is used instead of simulcast. Remove unnecessary SSRCs.
    if (config.rtp.ssrcs.size() > 1) {
      config.rtp.ssrcs.resize(1);
      if (config.rtp.rtx.ssrcs.size() > 1) {
        config.rtp.rtx.ssrcs.resize(1);
      }
    }
  }
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //创建了webrtc::VideoSendStream* stream_
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!

  parameters_.encoder_config.encoder_specific_settings = NULL;

  if (source_) {
    stream_->SetSource(source_, GetDegradationPreference());
  }

  // Call stream_->Start() if necessary conditions are met.
  UpdateSendState();
}
```





### 2.1 Call::CreateVideoSendStream

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
    //  webrtc::VideoSendStream::Config config 
    //  webrtc::VideoSendStream::Config::Rtp.ssrcs
    // 	是simulcast所有层的ssrc，不包括rtx和flex fec的ssrc
  std::vector<uint32_t> ssrcs = config.rtp.ssrcs;

    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //创建了webrtc::VideoSendStream* stream_
  VideoSendStream* send_stream = new VideoSendStream(
      clock_, num_cpu_cores_, module_process_thread_->process_thread(),
      task_queue_factory_, call_stats_->AsRtcpRttStats(), transport_send_ptr_,
      bitrate_allocator_.get(), video_send_delay_stats_.get(), event_log_,
      std::move(config), std::move(encoder_config), suspended_video_send_ssrcs_,
      suspended_video_payload_states_, std::move(fec_controller));
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    //创建了webrtc::VideoSendStream* stream_

  for (uint32_t ssrc : ssrcs) {
    RTC_DCHECK(video_send_ssrcs_.find(ssrc) == video_send_ssrcs_.end());
    // std::map<uint32_t, VideoSendStream*> video_send_ssrcs_
      // Call::UpdateAggregateNetworkState 使用
    video_send_ssrcs_[ssrc] = send_stream;
  }
  // std::set<VideoSendStream*> video_send_streams_
  // 用于分发rtcp包, Call::DeliverRtcp
  video_send_streams_.insert(send_stream);
  // Forward resources that were previously added to the call to the new stream.
  for (const auto& resource_forwarder : adaptation_resource_forwarders_) {
    resource_forwarder->OnCreateVideoSendStream(send_stream);
  }

  UpdateAggregateNetworkState();

  return send_stream;
}
```





# webrtc::interanl::VideoSendStream ， VideoStreamEncoder 和 VideoSendStreamImpl关系



## 1. VideoSendStream::VideoSendStream

video/video_send_stream.cc

```cpp
VideoSendStream::VideoSendStream(
    Clock* clock,
    int num_cpu_cores,
    ProcessThread* module_process_thread,
    TaskQueueFactory* task_queue_factory,
    RtcpRttStats* call_stats,
    RtpTransportControllerSendInterface* transport,
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

  //!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!
  // 创建 VideoStreamEncoder
  video_stream_encoder_ = std::make_unique<VideoStreamEncoder>(
      clock, num_cpu_cores, &stats_proxy_, config_.encoder_settings,
      std::make_unique<OveruseFrameDetector>(&stats_proxy_), task_queue_factory,
      GetBitrateAllocationCallbackType(config_));
  //!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!
  //!!!!!!!!!!!!!!!!!!!!!

  // TODO(srte): Initialization should not be done posted on a task queue.
  // Note that the posted task must not outlive this scope since the closure
  // references local variables.
  worker_queue_->PostTask(ToQueuedTask(
      [this, clock, call_stats, transport, bitrate_allocator, send_delay_stats,
       event_log, &suspended_ssrcs, &encoder_config, &suspended_payload_states,
       &fec_controller]() {
      //!!!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!!!
      // 创建 VideoSendStreamImpl
        send_stream_.reset(new VideoSendStreamImpl(
            clock, &stats_proxy_, worker_queue_, call_stats, transport,
            bitrate_allocator, send_delay_stats, video_stream_encoder_.get(),
            event_log, &config_, encoder_config.max_bitrate_bps,
            encoder_config.bitrate_priority, suspended_ssrcs,
            suspended_payload_states, encoder_config.content_type,
            std::move(fec_controller)));
      //!!!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!!!!!
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


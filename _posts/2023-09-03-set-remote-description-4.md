---
layout: post
title: webrtc setRemoteDescription-4
date: 2023-09-03 23:11:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


![setRemoteSDP]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/setRemoteSDP-6649445.svg)



![videoencodesdp]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/videoencodesdp.svg)

![videotrack]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/videotrack.svg)

![setRemoteSDP]({{ site.url }}{{ site.baseurl }}/images/set-remote-description.assets/setRemoteSDP-6663142.svg)



## Call::CreateVideoSendStream

```c++
webrtc::VideoSendStream* Call::CreateVideoSendStream(
    webrtc::VideoSendStream::Config config,
    VideoEncoderConfig encoder_config) {
  if (config_.fec_controller_factory) {
    RTC_LOG(LS_INFO) << "External FEC Controller will be used.";
  }
  std::unique_ptr<FecController> fec_controller =
      config_.fec_controller_factory
          ? config_.fec_controller_factory->CreateFecController()
          : std::make_unique<FecControllerDefault>(clock_);
  return CreateVideoSendStream(std::move(config), std::move(encoder_config),
                               std::move(fec_controller));
}
```



## Call::CreateVideoSendStream

```c++

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

  VideoSendStream* send_stream = new VideoSendStream(
      clock_, num_cpu_cores_, module_process_thread_->process_thread(),
      task_queue_factory_, call_stats_->AsRtcpRttStats(), transport_send_ptr_,
      bitrate_allocator_.get(), video_send_delay_stats_.get(), event_log_,
      std::move(config), std::move(encoder_config), suspended_video_send_ssrcs_,
      suspended_video_payload_states_, std::move(fec_controller));

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



## Call.EnsureStarted

```c++
void Call::EnsureStarted() {
  if (is_started_) {
    return;
  }
  is_started_ = true;

  // This call seems to kick off a number of things, so probably better left
  // off being kicked off on request rather than in the ctor.
  // TargetTransferRateObserver
  transport_send_ptr_->RegisterTargetTransferRateObserver(this);

  module_process_thread_->EnsureStarted();
  transport_send_ptr_->EnsureStarted();
}
```

```c++
// Implements TargetTransferRateObserver,
  void OnTargetTransferRate(TargetTransferRate msg) override;
  void OnStartRateUpdate(DataRate start_rate) override;
```



## RtpTransportControllerSend.RegisterTargetTransferRateObserver

```c++
void RtpTransportControllerSend::RegisterTargetTransferRateObserver(
    TargetTransferRateObserver* observer) {
  task_queue_.PostTask([this, observer] {
    RTC_DCHECK_RUN_ON(&task_queue_);
    RTC_DCHECK(observer_ == nullptr);
    observer_ = observer;
    observer_->OnStartRateUpdate(*initial_config_.constraints.starting_rate);
    MaybeCreateControllers();
  });
}
```



## Call.OnStartRateUpdate

```c++
void Call::OnStartRateUpdate(DataRate start_rate) {
  RTC_DCHECK_RUN_ON(send_transport_queue());
  bitrate_allocator_->UpdateStartRate(start_rate.bps<uint32_t>());
}
```





## VideoSendStream.SetSource

```c++
void VideoSendStream::SetSource(
    rtc::VideoSourceInterface<webrtc::VideoFrame>* source,
    const DegradationPreference& degradation_preference) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  video_stream_encoder_->SetSource(source, degradation_preference);
}
```



## VideoStreamEncoder::SetSource

```c++
void VideoStreamEncoder::SetSource(
    rtc::VideoSourceInterface<VideoFrame>* source,
    const DegradationPreference& degradation_preference) {
  RTC_DCHECK_RUN_ON(main_queue_);
  video_source_sink_controller_.SetSource(source);
  input_state_provider_.OnHasInputChanged(source);

  // This may trigger reconfiguring the QualityScaler on the encoder queue.
  encoder_queue_.PostTask([this, degradation_preference] {
    RTC_DCHECK_RUN_ON(&encoder_queue_);
    degradation_preference_manager_->SetDegradationPreference(
        degradation_preference);
    stream_resource_manager_.SetDegradationPreferences(degradation_preference);
    if (encoder_) {
      stream_resource_manager_.ConfigureQualityScaler(
          encoder_->GetEncoderInfo());
    }
  });
}
```





## VideoSourceSinkController::SetSource

```c++
void VideoSourceSinkController::SetSource(
    rtc::VideoSourceInterface<VideoFrame>* source) {
  RTC_DCHECK_RUN_ON(&sequence_checker_);

  rtc::VideoSourceInterface<VideoFrame>* old_source = source_;
  source_ = source;

  if (old_source != source && old_source)
    old_source->RemoveSink(sink_);

  if (!source)
    return;

  source->AddOrUpdateSink(sink_, CurrentSettingsToSinkWants());
}
```



## VideoTrackProxyWithInternal.AddOrUpdateSink



## VideoTrack::AddOrUpdateSink

```c++
// AddOrUpdateSink and RemoveSink should be called on the worker
// thread.
void VideoTrack::AddOrUpdateSink(rtc::VideoSinkInterface<VideoFrame>* sink,
                                 const rtc::VideoSinkWants& wants) {
  RTC_DCHECK(worker_thread_->IsCurrent());
  VideoSourceBase::AddOrUpdateSink(sink, wants);
  rtc::VideoSinkWants modified_wants = wants;
  modified_wants.black_frames = !enabled();
  video_source_->AddOrUpdateSink(sink, modified_wants);
}
```



## VideoTrackSourceProxyWithInternal.AddOrUpdateSink



## AdaptedVideoTrackSource.AddOrUpdateSink

```c++
void AdaptedVideoTrackSource::AddOrUpdateSink(
    rtc::VideoSinkInterface<webrtc::VideoFrame>* sink,
    const rtc::VideoSinkWants& wants) {
  broadcaster_.AddOrUpdateSink(sink, wants);
  OnSinkWantsChanged(broadcaster_.wants());
}
```



## VideoBroadcaster.AddOrUpdateSink


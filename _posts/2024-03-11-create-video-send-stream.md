---
layout: post
title: webrtc::VideoSendStream创建时机
date: 2024-03-11 12:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc stream
---


* content
{:toc}

---

# webrtc::VideoSendStream创建时机

```less
WebRtcVideoSendStream::SetCodec
WebRtcVideoSendStream::SetSendParameters
WebRtcVideoSendStream::SetFrameEncryptor
WebRtcVideoSendStream::SetEncoderToPacketizerFrameTransforme
```

这几个地方都会调用`RecreateWebRtcStream`。 stream 指的是 webrtc::VideoSendStream* stream_

```less
webrtc::VideoSendStream
webrtc::internal::VideoSendStream
```



## 1. WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream

media/engine/webrtc_video_engine.cc


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
  //!!!!!!!!!
  //!!!!!!!!!
  //!!!!!!!!!
  // webrtc::VideoSendStream* stream_
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());
  //!!!!!!!!!
  //!!!!!!!!!
  //!!!!!!!!!

  parameters_.encoder_config.encoder_specific_settings = NULL;

  if (source_) {
    stream_->SetSource(source_, GetDegradationPreference());
  }

  // Call stream_->Start() if necessary conditions are met.
  UpdateSendState();
}
```





## 2. Call::CreateVideoSendStream

创建了internal::VideoSendStream。

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
x
  VideoSendStream* send_stream = new VideoSendStream(
      clock_, num_cpu_cores_, module_process_thread_->process_thread(),
      task_queue_factory_, call_stats_->AsRtcpRttStats(), transport_send_ptr_,
      bitrate_allocator_.get(), video_send_delay_stats_.get(), event_log_,
      std::move(config), std::move(encoder_config), suspended_video_send_ssrcs_,
      suspended_video_payload_states_, std::move(fec_controller));
  //!!!!!!!!!!!
  //!!!!!!!!!!!
  //!!!!!!!!!!!

  // simulast每一层一个ssrc，不包括rtx和flexfec的， 所有的 ssrc 共用一个VideoSendStream
  for (uint32_t ssrc : ssrcs) {
    RTC_DCHECK(video_send_ssrcs_.find(ssrc) == video_send_ssrcs_.end());
    // std::map<uint32_t, VideoSendStream*> video_send_ssrcs_
    video_send_ssrcs_[ssrc] = send_stream;
  }
  //  std::set<VideoSendStream*> video_send_streams_
  video_send_streams_.insert(send_stream);
  // Forward resources that were previously added to the call to the new stream.
  for (const auto& resource_forwarder : adaptation_resource_forwarders_) {
    resource_forwarder->OnCreateVideoSendStream(send_stream);
  }

  UpdateAggregateNetworkState();

  return send_stream;
}
```

- std::map<uint32_t, VideoSendStream*> video_send_ssrcs_, `Call::UpdateAggregateNetworkState` 会用到。
- std::set<VideoSendStream*> video_send_streams_，

### !!! 2.1 std::vector<uint32_t> ssrcs = config.rtp.ssrcs;

webrtc::VideoSendStream::Config::Config::RtpConfig::ssrcs;

包括simulcast ssrc，不包括rtx和flexfec的ssrc。



### 2.2 --VideoSendStream::VideoSendStream

### 2.3 -- Call::UpdateAggregateNetworkState

## 3. VideoSendStream::VideoSendStream

video/video_send_stream.h

```cpp
class VideoSendStream : public webrtc::VideoSendStream {
..
}
```



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

  //!!!!!!!!!!!
  //!!!!!!!!!!!
  //!!!!!!!!!!!
  // 创建 VideoStreamEncoder
  video_stream_encoder_ = std::make_unique<VideoStreamEncoder>(
      clock, num_cpu_cores, &stats_proxy_, config_.encoder_settings,
      std::make_unique<OveruseFrameDetector>(&stats_proxy_), task_queue_factory,
      GetBitrateAllocationCallbackType(config_));
  //!!!!!!!!!!!
  //!!!!!!!!!!!
  //!!!!!!!!!!!

  // TODO(srte): Initialization should not be done posted on a task queue.
  // Note that the posted task must not outlive this scope since the closure
  // references local variables.
   
  //!!!!!!!!!!!
  //!!!!!!!!!!!
  //!!!!!!!!!!!
  // 创建 VideoSendStreamImpl
  worker_queue_->PostTask(ToQueuedTask(
      [this, clock, call_stats, transport, bitrate_allocator, send_delay_stats,
       event_log, &suspended_ssrcs, &encoder_config, &suspended_payload_states,
       &fec_controller]() {
        send_stream_.reset(new VideoSendStreamImpl(
            clock, &stats_proxy_, worker_queue_, call_stats, transport,
            bitrate_allocator, send_delay_stats, video_stream_encoder_.get(),
            event_log, &config_, encoder_config.max_bitrate_bps,
            encoder_config.bitrate_priority, suspended_ssrcs,
            suspended_payload_states, encoder_config.content_type,
            std::move(fec_controller)));
      },
      [this]() { thread_sync_event_.Set(); }));
  //!!!!!!!!!!!
  //!!!!!!!!!!!
  //!!!!!!!!!!!

  // Wait for ConstructionTask to complete so that |send_stream_| can be used.
  // |module_process_thread| must be registered and deregistered on the thread
  // it was created on.
  thread_sync_event_.Wait(rtc::Event::kForever);
  
  // VideoSendStreamImpl  send_stream_   
  send_stream_->RegisterProcessThread(module_process_thread);
  ReconfigureVideoEncoder(std::move(encoder_config));
}
```



### 3.1 RtpTransportControllerSendInterface* transport

CallFactory，创建Call对象的时候，创建了RtpTransportControllerSend，一个Call对应一个RtpTransportControllerSend，所以码流控制同一个Call下，是统一分配处理的。

### 3.2 BitrateAllocatorInterface* bitrate_allocator

码流分配器，后续会介绍。

### 3.3 VideoSendStream::Config

参考[webrtc::VideoSendStream::Config 说明和创建时机](https://even3yu.github.io/2024/03/07/videosendstream-config/)

### 3.4 VideoEncoderConfig

参考[webrtc::VideoEncoderConfig 说明和创建时机](https://even3yu.github.io/2024/03/07/video-encoder-config/)

### 3.5 ??? SendDelayStats* send_delay_stats

### 3.6 ??? FecController

### 

### 3.7 --VideoStreamEncoder::VideoStreamEncoder

video/video_stream_encoder.h

```cpp
// VideoStreamEncoder represent a video encoder that accepts raw video frames as
// input and produces an encoded bit stream.
// Usage:
//  Instantiate.
//  Call SetSink.
//  Call SetSource.
//  Call ConfigureEncoder with the codec settings.
//  Call Stop() when done.
class VideoStreamEncoder : public VideoStreamEncoderInterface,
                           private EncodedImageCallback,
                           public VideoSourceRestrictionsListener {
                           ...
                           }
```



### 3.8 -- VideoSendStreamImpl::VideoSendStreamImpl

video/video_send_stream_impl.h

```cpp
VideoSendStreamImpl::VideoSendStreamImpl(
    Clock* clock,
    SendStatisticsProxy* stats_proxy,
    rtc::TaskQueue* worker_queue,
    RtcpRttStats* call_stats,
    RtpTransportControllerSendInterface* transport,
    BitrateAllocatorInterface* bitrate_allocator,
    SendDelayStats* send_delay_stats,
    VideoStreamEncoderInterface* video_stream_encoder,
    RtcEventLog* event_log,
    const VideoSendStream::Config* config,
    int initial_encoder_max_bitrate,
    double initial_encoder_bitrate_priority,
    std::map<uint32_t, RtpState> suspended_ssrcs,
    std::map<uint32_t, RtpPayloadState> suspended_payload_states,
    VideoEncoderConfig::ContentType content_type,
    std::unique_ptr<FecController> fec_controller){
    
    }
```



### 3.9 VideoSendStream::ReconfigureVideoEncoder

```cpp
void VideoSendStream::ReconfigureVideoEncoder(VideoEncoderConfig config) {
  // TODO(perkj): Some test cases in VideoSendStreamTest call
  // ReconfigureVideoEncoder from the network thread.
  // RTC_DCHECK_RUN_ON(&thread_checker_);
  RTC_DCHECK(content_type_ == config.content_type);
  video_stream_encoder_->ConfigureEncoder(
      std::move(config),
      config_.rtp.max_packet_size - CalculateMaxHeaderSize(config_.rtp));
}
```



#### 3.9.1 ???VideoStreamEncoder::ConfigureEncoder

video/video_stream_encoder.cc

```cpp
void VideoStreamEncoder::ConfigureEncoder(VideoEncoderConfig config,
                                          size_t max_data_payload_length) {
  encoder_queue_.PostTask(
      [this, config = std::move(config), max_data_payload_length]() mutable {
        RTC_DCHECK_RUN_ON(&encoder_queue_);
        RTC_DCHECK(sink_);
        RTC_LOG(LS_INFO) << "ConfigureEncoder requested.";

        pending_encoder_creation_ =
            (!encoder_ || encoder_config_.video_format != config.video_format ||
             max_data_payload_length_ != max_data_payload_length);
        encoder_config_ = std::move(config);
        max_data_payload_length_ = max_data_payload_length;
        pending_encoder_reconfiguration_ = true;

        // Reconfigure the encoder now if the encoder has an internal source or
        // if the frame resolution is known. Otherwise, the reconfiguration is
        // deferred until the next frame to minimize the number of
        // reconfigurations. The codec configuration depends on incoming video
        // frame size.
        if (last_frame_info_) {
          ReconfigureEncoder();
        } else {
          codec_info_ = settings_.encoder_factory->QueryVideoEncoder(
              encoder_config_.video_format);
          if (HasInternalSource()) {
            last_frame_info_ = VideoFrameInfo(kDefaultInputPixelsWidth,
                                              kDefaultInputPixelsHeight, false);
            ReconfigureEncoder();
          }
        }
      });
}
```



#### 3.9.2 -- VideoStreamEncoder::ReconfigureEncoder



## 3.10 ??? Call::UpdateAggregateNetworkState

call/call.cc

更新所有的网络状态。

```cpp
void Call::UpdateAggregateNetworkState() {
  RTC_DCHECK_RUN_ON(worker_thread_);

  // 同一type，比如audio，video，所有的ssrc，对应同一个SendStream，
  bool have_audio =
      !audio_send_ssrcs_.empty() || !audio_receive_streams_.empty();
  bool have_video =
      !video_send_ssrcs_.empty() || !video_receive_streams_.empty();

  bool aggregate_network_up =
      ((have_video && video_network_state_ == kNetworkUp) ||
       (have_audio && audio_network_state_ == kNetworkUp));

  if (aggregate_network_up != aggregate_network_up_) {
    RTC_LOG(LS_INFO)
        << "UpdateAggregateNetworkState: aggregate_state change to "
        << (aggregate_network_up ? "up" : "down");
  } else {
    RTC_LOG(LS_VERBOSE)
        << "UpdateAggregateNetworkState: aggregate_state remains at "
        << (aggregate_network_up ? "up" : "down");
  }
  aggregate_network_up_ = aggregate_network_up;

  // RtpTransportControllerSendInterface
  transport_send_ptr_->OnNetworkAvailability(aggregate_network_up);
}
```



### 3.10.1 NetworkState

libs/mediaengine/third_party/WebRTC/src/call/call.cc

```cpp
 NetworkState audio_network_state_;
 NetworkState video_network_state_;
```



api/rtp_headers.h

```
enum NetworkState {
  kNetworkUp,
  kNetworkDown,
};
```




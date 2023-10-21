---
layout: post
title: 视频发送数据流
date: 2023-10-21 10:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc video
---


* content
{:toc}

---

## 0. 前言

![img]({{ site.url }}{{ site.baseurl }}/images/video-data-flow.assets/flow-data-sequence.png)

1. 摄像头采集数据，对数据进行旋转，下采样；
2. 数据流转到VideoSteamEncoder，这里的数据是把VideoSteamEncoder 注册到 `webrtc::ObjCVideoTrackSource`, 这样数据从capturer---> source ---> VideoStreamEncoder，进行编码
3. 编码后的数据，发送到VideoStreamImpl
4. 

## 1. capture 数据到encoder

### 1.1 RTCCameraVideoCapturer.captureOutput:didOutputSampleBuffer:fromConnection:

sdk/objc/components/capturer/RTCCameraVideoCapturer.m

```objective-c
- (void)captureOutput:(AVCaptureOutput *)captureOutput
    didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
           fromConnection:(AVCaptureConnection *)connection {
  NSParameterAssert(captureOutput == _videoDataOutput);

  if (CMSampleBufferGetNumSamples(sampleBuffer) != 1 || !CMSampleBufferIsValid(sampleBuffer) ||
      !CMSampleBufferDataIsReady(sampleBuffer)) {
    return;
  }
	// 获取 CVPixelBufferRef
  CVPixelBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
  if (pixelBuffer == nil) {
    return;
  }

#if TARGET_OS_IPHONE
  // Default to portrait orientation on iPhone.
  BOOL usingFrontCamera = NO;
  // Check the image's EXIF for the camera the image came from as the image could have been
  // delayed as we set alwaysDiscardsLateVideoFrames to NO.
  AVCaptureDevicePosition cameraPosition =
      [AVCaptureSession devicePositionForSampleBuffer:sampleBuffer];
  if (cameraPosition != AVCaptureDevicePositionUnspecified) {
    usingFrontCamera = AVCaptureDevicePositionFront == cameraPosition;
  } else {
    AVCaptureDeviceInput *deviceInput =
        (AVCaptureDeviceInput *)((AVCaptureInputPort *)connection.inputPorts.firstObject).input;
    usingFrontCamera = AVCaptureDevicePositionFront == deviceInput.device.position;
  }
  switch (_orientation) {
    case UIDeviceOrientationPortrait:
      _rotation = RTCVideoRotation_90;
      break;
    case UIDeviceOrientationPortraitUpsideDown:
      _rotation = RTCVideoRotation_270;
      break;
    case UIDeviceOrientationLandscapeLeft:
      _rotation = usingFrontCamera ? RTCVideoRotation_180 : RTCVideoRotation_0;
      break;
    case UIDeviceOrientationLandscapeRight:
      _rotation = usingFrontCamera ? RTCVideoRotation_0 : RTCVideoRotation_180;
      break;
    case UIDeviceOrientationFaceUp:
    case UIDeviceOrientationFaceDown:
    case UIDeviceOrientationUnknown:
      // Ignore.
      break;
  }
#else
  // No rotation on Mac.
  _rotation = RTCVideoRotation_0;
#endif

  RTC_OBJC_TYPE(RTCCVPixelBuffer) *rtcPixelBuffer =
      [[RTC_OBJC_TYPE(RTCCVPixelBuffer) alloc] initWithPixelBuffer:pixelBuffer];
  int64_t timeStampNs = CMTimeGetSeconds(CMSampleBufferGetPresentationTimeStamp(sampleBuffer)) *
      kNanosecondsPerSecond;
  RTC_OBJC_TYPE(RTCVideoFrame) *videoFrame =
      [[RTC_OBJC_TYPE(RTCVideoFrame) alloc] initWithBuffer:rtcPixelBuffer
                                                  rotation:_rotation
                                               timeStampNs:timeStampNs];
  [self.delegate capturer:self didCaptureVideoFrame:videoFrame];
}
```

1. 判断前后置摄像头，从而来调整旋转角度

2. RTCCVPixelBuffer 封装 CVPixelBufferRef 为RTCVideoFrame；


```objective-c
   RTC_OBJC_EXPORT
   @interface RTC_OBJC_TYPE (RTCCVPixelBuffer) : NSObject <RTC_OBJC_TYPE(RTCVideoFrameBuffer)>
     
     ...
//------------------------
     
     
   - (instancetype)initWithPixelBuffer:(CVPixelBufferRef)pixelBuffer {
     return [self initWithPixelBuffer:pixelBuffer
                         adaptedWidth:CVPixelBufferGetWidth(pixelBuffer)
                        adaptedHeight:CVPixelBufferGetHeight(pixelBuffer)
                            cropWidth:CVPixelBufferGetWidth(pixelBuffer)
                           cropHeight:CVPixelBufferGetHeight(pixelBuffer)
                                cropX:0
                                cropY:0];
      }
   
   - (instancetype)initWithPixelBuffer:(CVPixelBufferRef)pixelBuffer
                          adaptedWidth:(int)adaptedWidth
                         adaptedHeight:(int)adaptedHeight
                             cropWidth:(int)cropWidth
                            cropHeight:(int)cropHeight
                                 cropX:(int)cropX
                                 cropY:(int)cropY {
     if (self = [super init]) {
       _width = adaptedWidth;
       _height = adaptedHeight;
       _pixelBuffer = pixelBuffer;
       _bufferWidth = CVPixelBufferGetWidth(_pixelBuffer);
       _bufferHeight = CVPixelBufferGetHeight(_pixelBuffer);
       _cropWidth = cropWidth;
       _cropHeight = cropHeight;
       // Can only crop at even pixels.
       _cropX = cropX & ~1;
       _cropY = cropY & ~1;
       // retain _pixelBuffer 
       CVBufferRetain(_pixelBuffer);
     }
   
     return self;
      }
   
   - (void)dealloc {
     // 释放 _pixelBuffer
     CVBufferRelease(_pixelBuffer);
      }
```



### 1.2 RTCVideoSource.capturer:didCaptureVideoFrame:

```objective-c
- (void)capturer:(RTC_OBJC_TYPE(RTCVideoCapturer) *)capturer
    didCaptureVideoFrame:(RTC_OBJC_TYPE(RTCVideoFrame) *)frame {
      // ObjCVideoTrackSource _nativeVideoSource
  getObjCVideoSource(_nativeVideoSource)->OnCapturedFrame(frame);
}
```



#### getObjCVideoSource

sdk/objc/api/peerconnection/RTCVideoSource.mm

```cpp
static webrtc::ObjCVideoTrackSource *getObjCVideoSource(
    const rtc::scoped_refptr<webrtc::VideoTrackSourceInterface> nativeSource) {
  webrtc::VideoTrackSourceProxy *proxy_source =
      static_cast<webrtc::VideoTrackSourceProxy *>(nativeSource.get());
  return static_cast<webrtc::ObjCVideoTrackSource *>(proxy_source->internal());
}
```



### 1.3 webrtc::ObjCVideoTrackSource.OnCapturedFrame

sdk/objc/native/src/objc_video_track_source.h

```c++
void ObjCVideoTrackSource::OnCapturedFrame(RTC_OBJC_TYPE(RTCVideoFrame) * frame) {
  const int64_t timestamp_us = frame.timeStampNs / rtc::kNumNanosecsPerMicrosec;
  const int64_t translated_timestamp_us =
      timestamp_aligner_.TranslateTimestamp(timestamp_us, rtc::TimeMicros());

  int adapted_width;
  int adapted_height;
  int crop_width;
  int crop_height;
  int crop_x;
  int crop_y;
  if (!AdaptFrame(frame.width,
                  frame.height,
                  timestamp_us,
                  &adapted_width,
                  &adapted_height,
                  &crop_width,
                  &crop_height,
                  &crop_x,
                  &crop_y)) {
    return;
  }

  rtc::scoped_refptr<VideoFrameBuffer> buffer;
  if (adapted_width == frame.width && adapted_height == frame.height) {
    // No adaption - optimized path.
    buffer = new rtc::RefCountedObject<ObjCFrameBuffer>(frame.buffer);
  } else if ([frame.buffer isKindOfClass:[RTC_OBJC_TYPE(RTCCVPixelBuffer) class]]) {
    // Adapted CVPixelBuffer frame.
    RTC_OBJC_TYPE(RTCCVPixelBuffer) *rtcPixelBuffer =
        (RTC_OBJC_TYPE(RTCCVPixelBuffer) *)frame.buffer;
    buffer = new rtc::RefCountedObject<ObjCFrameBuffer>([[RTC_OBJC_TYPE(RTCCVPixelBuffer) alloc]
        initWithPixelBuffer:rtcPixelBuffer.pixelBuffer
               adaptedWidth:adapted_width
              adaptedHeight:adapted_height
                  cropWidth:crop_width
                 cropHeight:crop_height
                      cropX:crop_x + rtcPixelBuffer.cropX
                      cropY:crop_y + rtcPixelBuffer.cropY]);
  } else {
    // Adapted I420 frame.
    // TODO(magjed): Optimize this I420 path.
    rtc::scoped_refptr<I420Buffer> i420_buffer = I420Buffer::Create(adapted_width, adapted_height);
    buffer = new rtc::RefCountedObject<ObjCFrameBuffer>(frame.buffer);
    i420_buffer->CropAndScaleFrom(*buffer->ToI420(), crop_x, crop_y, crop_width, crop_height);
    buffer = i420_buffer;
  }

  // Applying rotation is only supported for legacy reasons and performance is
  // not critical here.
  VideoRotation rotation = static_cast<VideoRotation>(frame.rotation);
  if (apply_rotation() && rotation != kVideoRotation_0) {
    buffer = I420Buffer::Rotate(*buffer->ToI420(), rotation);
    rotation = kVideoRotation_0;
  }

  OnFrame(VideoFrame::Builder()
              .set_video_frame_buffer(buffer)
              .set_rotation(rotation)
              .set_timestamp_us(translated_timestamp_us)
              .build());
}
```



####  ???timestamp_aligner_.TranslateTimestamp

#### VideoAdapter::AdaptFrameResolution

media/base/video_adapter.cc



### 1.4  rtc::AdaptedVideoTrackSource.OnFrame

```c++
void AdaptedVideoTrackSource::OnFrame(const webrtc::VideoFrame& frame) {
  rtc::scoped_refptr<webrtc::VideoFrameBuffer> buffer(
      frame.video_frame_buffer());
  /* Note that this is a "best effort" approach to
     wants.rotation_applied; apply_rotation_ can change from false to
     true between the check of apply_rotation() and the call to
     broadcaster_.OnFrame(), in which case we generate a frame with
     pending rotation despite some sink with wants.rotation_applied ==
     true was just added. The VideoBroadcaster enforces
     synchronization for us in this case, by not passing the frame on
     to sinks which don't want it. */
  if (apply_rotation() && frame.rotation() != webrtc::kVideoRotation_0 &&
      buffer->type() == webrtc::VideoFrameBuffer::Type::kI420) {
    /* Apply pending rotation. */
    webrtc::VideoFrame rotated_frame(frame);
    rotated_frame.set_video_frame_buffer(
        webrtc::I420Buffer::Rotate(*buffer->GetI420(), frame.rotation()));
    rotated_frame.set_rotation(webrtc::kVideoRotation_0);
    broadcaster_.OnFrame(rotated_frame);
  } else {
    broadcaster_.OnFrame(frame);
  }
}
```



### 1.5 VideoBroadcaster.OnFrame

![callstack]({{ site.url }}{{ site.baseurl }}/images/video-data-flow.assets/call-stack.png)

```c++
void VideoBroadcaster::OnFrame(const webrtc::VideoFrame& frame) {
  webrtc::MutexLock lock(&sinks_and_wants_lock_);
  bool current_frame_was_discarded = false;
  for (auto& sink_pair : sink_pairs()) {
    if (sink_pair.wants.rotation_applied &&
        frame.rotation() != webrtc::kVideoRotation_0) {
      // Calls to OnFrame are not synchronized with changes to the sink wants.
      // When rotation_applied is set to true, one or a few frames may get here
      // with rotation still pending. Protect sinks that don't expect any
      // pending rotation.
      RTC_LOG(LS_VERBOSE) << "Discarding frame with unexpected rotation.";
      sink_pair.sink->OnDiscardedFrame();
      current_frame_was_discarded = true;
      continue;
    }
    if (sink_pair.wants.black_frames) {
      webrtc::VideoFrame black_frame =
          webrtc::VideoFrame::Builder()
              .set_video_frame_buffer(
                  GetBlackFrameBuffer(frame.width(), frame.height()))
              .set_rotation(frame.rotation())
              .set_timestamp_us(frame.timestamp_us())
              .set_id(frame.id())
              .build();
      sink_pair.sink->OnFrame(black_frame);
    } else if (!previous_frame_sent_to_all_sinks_ && frame.has_update_rect()) {
      // Since last frame was not sent to some sinks, no reliable update
      // information is available, so we need to clear the update rect.
      webrtc::VideoFrame copy = frame;
      copy.clear_update_rect();
      sink_pair.sink->OnFrame(copy);
    } else {
      sink_pair.sink->OnFrame(frame);
    }
  }
  previous_frame_sent_to_all_sinks_ = !current_frame_was_discarded;
}
```



## 2. VideoStreamEncoder.OnFrame——编码

```c++
void VideoStreamEncoder::OnFrame(const VideoFrame& video_frame) {
  RTC_DCHECK_RUNS_SERIALIZED(&incoming_frame_race_checker_);
  VideoFrame incoming_frame = video_frame;

  // Local time in webrtc time base.
  Timestamp now = clock_->CurrentTime();

  // In some cases, e.g., when the frame from decoder is fed to encoder,
  // the timestamp may be set to the future. As the encoding pipeline assumes
  // capture time to be less than present time, we should reset the capture
  // timestamps here. Otherwise there may be issues with RTP send stream.
  if (incoming_frame.timestamp_us() > now.us())
    incoming_frame.set_timestamp_us(now.us());

  // Capture time may come from clock with an offset and drift from clock_.
  int64_t capture_ntp_time_ms;
  if (video_frame.ntp_time_ms() > 0) {
    capture_ntp_time_ms = video_frame.ntp_time_ms();
  } else if (video_frame.render_time_ms() != 0) {
    capture_ntp_time_ms = video_frame.render_time_ms() + delta_ntp_internal_ms_;
  } else {
    capture_ntp_time_ms = now.ms() + delta_ntp_internal_ms_;
  }
  incoming_frame.set_ntp_time_ms(capture_ntp_time_ms);

  // Convert NTP time, in ms, to RTP timestamp.
  const int kMsToRtpTimestamp = 90;
  incoming_frame.set_timestamp(
      kMsToRtpTimestamp * static_cast<uint32_t>(incoming_frame.ntp_time_ms()));

  if (incoming_frame.ntp_time_ms() <= last_captured_timestamp_) {
    // We don't allow the same capture time for two frames, drop this one.
    RTC_LOG(LS_WARNING) << "Same/old NTP timestamp ("
                        << incoming_frame.ntp_time_ms()
                        << " <= " << last_captured_timestamp_
                        << ") for incoming frame. Dropping.";
    encoder_queue_.PostTask([this, incoming_frame]() {
      RTC_DCHECK_RUN_ON(&encoder_queue_);
      accumulated_update_rect_.Union(incoming_frame.update_rect());
      accumulated_update_rect_is_valid_ &= incoming_frame.has_update_rect();
    });
    return;
  }

  bool log_stats = false;
  if (now.ms() - last_frame_log_ms_ > kFrameLogIntervalMs) {
    last_frame_log_ms_ = now.ms();
    log_stats = true;
  }

  last_captured_timestamp_ = incoming_frame.ntp_time_ms();

  int64_t post_time_us = clock_->CurrentTime().us();
  ++posted_frames_waiting_for_encode_;

  encoder_queue_.PostTask(
      [this, incoming_frame, post_time_us, log_stats]() {
        RTC_DCHECK_RUN_ON(&encoder_queue_);
        encoder_stats_observer_->OnIncomingFrame(incoming_frame.width(),
                                                 incoming_frame.height());
        ++captured_frame_count_;
        const int posted_frames_waiting_for_encode =
            posted_frames_waiting_for_encode_.fetch_sub(1);
        RTC_DCHECK_GT(posted_frames_waiting_for_encode, 0);
        CheckForAnimatedContent(incoming_frame, post_time_us);
        bool cwnd_frame_drop =
            cwnd_frame_drop_interval_ &&
            (cwnd_frame_counter_++ % cwnd_frame_drop_interval_.value() == 0);
        if (posted_frames_waiting_for_encode == 1 && !cwnd_frame_drop) {
          MaybeEncodeVideoFrame(incoming_frame, post_time_us);
        } else {
          if (cwnd_frame_drop) {
            // Frame drop by congestion window pusback. Do not encode this
            // frame.
            ++dropped_frame_cwnd_pushback_count_;
            encoder_stats_observer_->OnFrameDropped(
                VideoStreamEncoderObserver::DropReason::kCongestionWindow);
          } else {
            // There is a newer frame in flight. Do not encode this frame.
            RTC_LOG(LS_VERBOSE)
                << "Incoming frame dropped due to that the encoder is blocked.";
            ++dropped_frame_encoder_block_count_;
            encoder_stats_observer_->OnFrameDropped(
                VideoStreamEncoderObserver::DropReason::kEncoderQueue);
          }
          accumulated_update_rect_.Union(incoming_frame.update_rect());
          accumulated_update_rect_is_valid_ &= incoming_frame.has_update_rect();
        }
        if (log_stats) {
          RTC_LOG(LS_INFO) << "Number of frames: captured "
                           << captured_frame_count_
                           << ", dropped (due to congestion window pushback) "
                           << dropped_frame_cwnd_pushback_count_
                           << ", dropped (due to encoder blocked) "
                           << dropped_frame_encoder_block_count_
                           << ", interval_ms " << kFrameLogIntervalMs;
          captured_frame_count_ = 0;
          dropped_frame_cwnd_pushback_count_ = 0;
          dropped_frame_encoder_block_count_ = 0;
        }
      });
}
```



### ntp，rtp 时间戳？？？

ntp （ms）* （一毫米增长量 = 90000/1000，1ms 采样90次）
可以理解为采样次数



### 2.0 VideoStreamEncoder.MaybeEncodeVideoFrame

```c++
void VideoStreamEncoder::MaybeEncodeVideoFrame(const VideoFrame& video_frame,
                                               int64_t time_when_posted_us) {
  RTC_DCHECK_RUN_ON(&encoder_queue_);
  input_state_provider_.OnFrameSizeObserved(video_frame.size());

  if (!last_frame_info_ || video_frame.width() != last_frame_info_->width ||
      video_frame.height() != last_frame_info_->height ||
      video_frame.is_texture() != last_frame_info_->is_texture) {
    pending_encoder_reconfiguration_ = true;
    last_frame_info_ = VideoFrameInfo(video_frame.width(), video_frame.height(),
                                      video_frame.is_texture());
    RTC_LOG(LS_INFO) << "Video frame parameters changed: dimensions="
                     << last_frame_info_->width << "x"
                     << last_frame_info_->height
                     << ", texture=" << last_frame_info_->is_texture << ".";
    // Force full frame update, since resolution has changed.
    accumulated_update_rect_ =
        VideoFrame::UpdateRect{0, 0, video_frame.width(), video_frame.height()};
  }

  // We have to create then encoder before the frame drop logic,
  // because the latter depends on encoder_->GetScalingSettings.
  // According to the testcase
  // InitialFrameDropOffWhenEncoderDisabledScaling, the return value
  // from GetScalingSettings should enable or disable the frame drop.

  // Update input frame rate before we start using it. If we update it after
  // any potential frame drop we are going to artificially increase frame sizes.
  // Poll the rate before updating, otherwise we risk the rate being estimated
  // a little too high at the start of the call when then window is small.
  uint32_t framerate_fps = GetInputFramerateFps();
  input_framerate_.Update(1u, clock_->TimeInMilliseconds());

  int64_t now_ms = clock_->TimeInMilliseconds();
  if (pending_encoder_reconfiguration_) {
    ReconfigureEncoder();
    last_parameters_update_ms_.emplace(now_ms);
  } else if (!last_parameters_update_ms_ ||
             now_ms - *last_parameters_update_ms_ >=
                 kParameterUpdateIntervalMs) {
    if (last_encoder_rate_settings_) {
      // Clone rate settings before update, so that SetEncoderRates() will
      // actually detect the change between the input and
      // |last_encoder_rate_setings_|, triggering the call to SetRate() on the
      // encoder.
      EncoderRateSettings new_rate_settings = *last_encoder_rate_settings_;
      new_rate_settings.rate_control.framerate_fps =
          static_cast<double>(framerate_fps);
      SetEncoderRates(UpdateBitrateAllocation(new_rate_settings));
    }
    last_parameters_update_ms_.emplace(now_ms);
  }

  // Because pending frame will be dropped in any case, we need to
  // remember its updated region.
  if (pending_frame_) {
    encoder_stats_observer_->OnFrameDropped(
        VideoStreamEncoderObserver::DropReason::kEncoderQueue);
    accumulated_update_rect_.Union(pending_frame_->update_rect());
    accumulated_update_rect_is_valid_ &= pending_frame_->has_update_rect();
  }

  if (DropDueToSize(video_frame.size())) {
    RTC_LOG(LS_INFO) << "Dropping frame. Too large for target bitrate.";
    stream_resource_manager_.OnFrameDroppedDueToSize();
    // Storing references to a native buffer risks blocking frame capture.
    if (video_frame.video_frame_buffer()->type() !=
        VideoFrameBuffer::Type::kNative) {
      pending_frame_ = video_frame;
      pending_frame_post_time_us_ = time_when_posted_us;
    } else {
      // Ensure that any previously stored frame is dropped.
      pending_frame_.reset();
      accumulated_update_rect_.Union(video_frame.update_rect());
      accumulated_update_rect_is_valid_ &= video_frame.has_update_rect();
    }
    return;
  }
  stream_resource_manager_.OnMaybeEncodeFrame();

  if (EncoderPaused()) {
    // Storing references to a native buffer risks blocking frame capture.
    if (video_frame.video_frame_buffer()->type() !=
        VideoFrameBuffer::Type::kNative) {
      if (pending_frame_)
        TraceFrameDropStart();
      pending_frame_ = video_frame;
      pending_frame_post_time_us_ = time_when_posted_us;
    } else {
      // Ensure that any previously stored frame is dropped.
      pending_frame_.reset();
      TraceFrameDropStart();
      accumulated_update_rect_.Union(video_frame.update_rect());
      accumulated_update_rect_is_valid_ &= video_frame.has_update_rect();
    }
    return;
  }

  pending_frame_.reset();

  frame_dropper_.Leak(framerate_fps);
  // Frame dropping is enabled iff frame dropping is not force-disabled, and
  // rate controller is not trusted.
  const bool frame_dropping_enabled =
      !force_disable_frame_dropper_ &&
      !encoder_info_.has_trusted_rate_controller;
  frame_dropper_.Enable(frame_dropping_enabled);
  if (frame_dropping_enabled && frame_dropper_.DropFrame()) {
    RTC_LOG(LS_VERBOSE)
        << "Drop Frame: "
           "target bitrate "
        << (last_encoder_rate_settings_
                ? last_encoder_rate_settings_->encoder_target.bps()
                : 0)
        << ", input frame rate " << framerate_fps;
    OnDroppedFrame(
        EncodedImageCallback::DropReason::kDroppedByMediaOptimizations);
    accumulated_update_rect_.Union(video_frame.update_rect());
    accumulated_update_rect_is_valid_ &= video_frame.has_update_rect();
    return;
  }

  EncodeVideoFrame(video_frame, time_when_posted_us);
}
```



### 2.1 VideoStreamEncoder.EncodeVideoFrame

```c++
void VideoStreamEncoder::EncodeVideoFrame(const VideoFrame& video_frame,
                                          int64_t time_when_posted_us) {
  RTC_DCHECK_RUN_ON(&encoder_queue_);

  // If the encoder fail we can't continue to encode frames. When this happens
  // the WebrtcVideoSender is notified and the whole VideoSendStream is
  // recreated.
  if (encoder_failed_)
    return;

  TraceFrameDropEnd();

  // Encoder metadata needs to be updated before encode complete callback.
  VideoEncoder::EncoderInfo info = encoder_->GetEncoderInfo();
  if (info.implementation_name != encoder_info_.implementation_name) {
    encoder_stats_observer_->OnEncoderImplementationChanged(
        info.implementation_name);
    if (bitrate_adjuster_) {
      // Encoder implementation changed, reset overshoot detector states.
      bitrate_adjuster_->Reset();
    }
  }

  if (encoder_info_ != info) {
    OnEncoderSettingsChanged();
    RTC_LOG(LS_INFO) << "Encoder settings changed from "
                     << encoder_info_.ToString() << " to " << info.ToString();
  }

  if (bitrate_adjuster_) {
    for (size_t si = 0; si < kMaxSpatialLayers; ++si) {
      if (info.fps_allocation[si] != encoder_info_.fps_allocation[si]) {
        bitrate_adjuster_->OnEncoderInfo(info);
        break;
      }
    }
  }
  encoder_info_ = info;
  last_encode_info_ms_ = clock_->TimeInMilliseconds();

  VideoFrame out_frame(video_frame);
  if (out_frame.video_frame_buffer()->type() ==
          VideoFrameBuffer::Type::kNative &&
      !info.supports_native_handle) {
    // This module only supports software encoding.
    rtc::scoped_refptr<VideoFrameBuffer> buffer =
        out_frame.video_frame_buffer()->GetMappedFrameBuffer(
            info.preferred_pixel_formats);
    bool buffer_was_converted = false;
    if (!buffer) {
      buffer = out_frame.video_frame_buffer()->ToI420();
      // TODO(https://crbug.com/webrtc/12021): Once GetI420 is pure virtual,
      // this just true as an I420 buffer would return from
      // GetMappedFrameBuffer.
      buffer_was_converted =
          (out_frame.video_frame_buffer()->GetI420() == nullptr);
    }
    if (!buffer) {
      RTC_LOG(LS_ERROR) << "Frame conversion failed, dropping frame.";
      return;
    }

    VideoFrame::UpdateRect update_rect = out_frame.update_rect();
    if (!update_rect.IsEmpty() &&
        out_frame.video_frame_buffer()->GetI420() == nullptr) {
      // UpdatedRect is reset to full update if it's not empty, and buffer was
      // converted, therefore we can't guarantee that pixels outside of
      // UpdateRect didn't change comparing to the previous frame.
      update_rect =
          VideoFrame::UpdateRect{0, 0, out_frame.width(), out_frame.height()};
    }
    out_frame.set_video_frame_buffer(buffer);
    out_frame.set_update_rect(update_rect);
  }

  // Crop frame if needed.
  if ((crop_width_ > 0 || crop_height_ > 0) &&
      out_frame.video_frame_buffer()->type() !=
          VideoFrameBuffer::Type::kNative) {
    // If the frame can't be converted to I420, drop it.
    int cropped_width = video_frame.width() - crop_width_;
    int cropped_height = video_frame.height() - crop_height_;
    rtc::scoped_refptr<VideoFrameBuffer> cropped_buffer;
    // TODO(ilnik): Remove scaling if cropping is too big, as it should never
    // happen after SinkWants signaled correctly from ReconfigureEncoder.
    VideoFrame::UpdateRect update_rect = video_frame.update_rect();
    if (crop_width_ < 4 && crop_height_ < 4) {
      cropped_buffer = video_frame.video_frame_buffer()->CropAndScale(
          crop_width_ / 2, crop_height_ / 2, cropped_width, cropped_height,
          cropped_width, cropped_height);
      update_rect.offset_x -= crop_width_ / 2;
      update_rect.offset_y -= crop_height_ / 2;
      update_rect.Intersect(
          VideoFrame::UpdateRect{0, 0, cropped_width, cropped_height});

    } else {
      cropped_buffer = video_frame.video_frame_buffer()->Scale(cropped_width,
                                                               cropped_height);
      if (!update_rect.IsEmpty()) {
        // Since we can't reason about pixels after scaling, we invalidate whole
        // picture, if anything changed.
        update_rect =
            VideoFrame::UpdateRect{0, 0, cropped_width, cropped_height};
      }
    }
    if (!cropped_buffer) {
      RTC_LOG(LS_ERROR) << "Cropping and scaling frame failed, dropping frame.";
      return;
    }

    out_frame.set_video_frame_buffer(cropped_buffer);
    out_frame.set_update_rect(update_rect);
    out_frame.set_ntp_time_ms(video_frame.ntp_time_ms());
    // Since accumulated_update_rect_ is constructed before cropping,
    // we can't trust it. If any changes were pending, we invalidate whole
    // frame here.
    if (!accumulated_update_rect_.IsEmpty()) {
      accumulated_update_rect_ =
          VideoFrame::UpdateRect{0, 0, out_frame.width(), out_frame.height()};
      accumulated_update_rect_is_valid_ = false;
    }
  }

  if (!accumulated_update_rect_is_valid_) {
    out_frame.clear_update_rect();
  } else if (!accumulated_update_rect_.IsEmpty() &&
             out_frame.has_update_rect()) {
    accumulated_update_rect_.Union(out_frame.update_rect());
    accumulated_update_rect_.Intersect(
        VideoFrame::UpdateRect{0, 0, out_frame.width(), out_frame.height()});
    out_frame.set_update_rect(accumulated_update_rect_);
    accumulated_update_rect_.MakeEmptyUpdate();
  }
  accumulated_update_rect_is_valid_ = true;

  TRACE_EVENT_ASYNC_STEP0("webrtc", "Video", video_frame.render_time_ms(),
                          "Encode");

  stream_resource_manager_.OnEncodeStarted(out_frame, time_when_posted_us);

  RTC_DCHECK_LE(send_codec_.width, out_frame.width());
  RTC_DCHECK_LE(send_codec_.height, out_frame.height());
  // Native frames should be scaled by the client.
  // For internal encoders we scale everything in one place here.
  RTC_DCHECK((out_frame.video_frame_buffer()->type() ==
              VideoFrameBuffer::Type::kNative) ||
             (send_codec_.width == out_frame.width() &&
              send_codec_.height == out_frame.height()));

  TRACE_EVENT1("webrtc", "VCMGenericEncoder::Encode", "timestamp",
               out_frame.timestamp());

  frame_encode_metadata_writer_.OnEncodeStarted(out_frame);

  const int32_t encode_status = encoder_->Encode(out_frame, &next_frame_types_);
  was_encode_called_since_last_initialization_ = true;

  if (encode_status < 0) {
    if (encode_status == WEBRTC_VIDEO_CODEC_ENCODER_FAILURE) {
      RTC_LOG(LS_ERROR) << "Encoder failed, failing encoder format: "
                        << encoder_config_.video_format.ToString();

      if (settings_.encoder_switch_request_callback) {
        if (encoder_selector_) {
          if (auto encoder = encoder_selector_->OnEncoderBroken()) {
            QueueRequestEncoderSwitch(*encoder);
          }
        } else {
          encoder_failed_ = true;
          main_queue_->PostTask(ToQueuedTask(task_safety_, [this]() {
            RTC_DCHECK_RUN_ON(main_queue_);
            settings_.encoder_switch_request_callback->RequestEncoderFallback();
          }));
        }
      } else {
        RTC_LOG(LS_ERROR)
            << "Encoder failed but no encoder fallback callback is registered";
      }
    } else {
      RTC_LOG(LS_ERROR) << "Failed to encode frame. Error code: "
                        << encode_status;
    }

    return;
  }

  for (auto& it : next_frame_types_) {
    it = VideoFrameType::kVideoFrameDelta;
  }
}
```



### 2.2 ObjCVideoEncoder.Encode

```c++
int32_t Encode(const VideoFrame &frame,
                 const std::vector<VideoFrameType> *frame_types) override {
    NSMutableArray<NSNumber *> *rtcFrameTypes = [NSMutableArray array];
    for (size_t i = 0; i < frame_types->size(); ++i) {
      [rtcFrameTypes addObject:@(RTCFrameType(frame_types->at(i)))];
    }

    return [encoder_ encode:ToObjCVideoFrame(frame)
          codecSpecificInfo:nil
                 frameTypes:rtcFrameTypes];
  }
```



### 2.3 RTCVideoEncoderH264.encode:codecSpecificInfo:frameTypes:——硬件编码

```objective-c
- (NSInteger)encode:(RTC_OBJC_TYPE(RTCVideoFrame) *)frame
    codecSpecificInfo:(nullable id<RTC_OBJC_TYPE(RTCCodecSpecificInfo)>)codecSpecificInfo
           frameTypes:(NSArray<NSNumber *> *)frameTypes {
  RTC_DCHECK_EQ(frame.width, _width);
  RTC_DCHECK_EQ(frame.height, _height);
  if (!_callback || !_compressionSession) {
    return WEBRTC_VIDEO_CODEC_UNINITIALIZED;
  }
  BOOL isKeyframeRequired = NO;

  // Get a pixel buffer from the pool and copy frame data over.
  if ([self resetCompressionSessionIfNeededWithFrame:frame]) {
    isKeyframeRequired = YES;
  }

  CVPixelBufferRef pixelBuffer = nullptr;
  if ([frame.buffer isKindOfClass:[RTC_OBJC_TYPE(RTCCVPixelBuffer) class]]) {
    // Native frame buffer
    RTC_OBJC_TYPE(RTCCVPixelBuffer) *rtcPixelBuffer =
        (RTC_OBJC_TYPE(RTCCVPixelBuffer) *)frame.buffer;
    if (![rtcPixelBuffer requiresCropping]) {
      // This pixel buffer might have a higher resolution than what the
      // compression session is configured to. The compression session can
      // handle that and will output encoded frames in the configured
      // resolution regardless of the input pixel buffer resolution.
      pixelBuffer = rtcPixelBuffer.pixelBuffer;
      CVBufferRetain(pixelBuffer);
    } else {
      // Cropping required, we need to crop and scale to a new pixel buffer.
      pixelBuffer = CreatePixelBuffer(_pixelBufferPool);
      if (!pixelBuffer) {
        return WEBRTC_VIDEO_CODEC_ERROR;
      }
      int dstWidth = CVPixelBufferGetWidth(pixelBuffer);
      int dstHeight = CVPixelBufferGetHeight(pixelBuffer);
      if ([rtcPixelBuffer requiresScalingToWidth:dstWidth height:dstHeight]) {
        int size =
            [rtcPixelBuffer bufferSizeForCroppingAndScalingToWidth:dstWidth height:dstHeight];
        _frameScaleBuffer.resize(size);
      } else {
        _frameScaleBuffer.clear();
      }
      _frameScaleBuffer.shrink_to_fit();
      if (![rtcPixelBuffer cropAndScaleTo:pixelBuffer withTempBuffer:_frameScaleBuffer.data()]) {
        CVBufferRelease(pixelBuffer);
        return WEBRTC_VIDEO_CODEC_ERROR;
      }
    }
  }

  if (!pixelBuffer) {
    // We did not have a native frame buffer
    pixelBuffer = CreatePixelBuffer(_pixelBufferPool);
    if (!pixelBuffer) {
      return WEBRTC_VIDEO_CODEC_ERROR;
    }
    RTC_DCHECK(pixelBuffer);
    if (!CopyVideoFrameToNV12PixelBuffer([frame.buffer toI420], pixelBuffer)) {
      RTC_LOG(LS_ERROR) << "Failed to copy frame data.";
      CVBufferRelease(pixelBuffer);
      return WEBRTC_VIDEO_CODEC_ERROR;
    }
  }

  // Check if we need a keyframe.
  if (!isKeyframeRequired && frameTypes) {
    for (NSNumber *frameType in frameTypes) {
      if ((RTCFrameType)frameType.intValue == RTCFrameTypeVideoFrameKey) {
        isKeyframeRequired = YES;
        break;
      }
    }
  }

  CMTime presentationTimeStamp = CMTimeMake(frame.timeStampNs / rtc::kNumNanosecsPerMillisec, 1000);
  CFDictionaryRef frameProperties = nullptr;
  if (isKeyframeRequired) {
    CFTypeRef keys[] = {kVTEncodeFrameOptionKey_ForceKeyFrame};
    CFTypeRef values[] = {kCFBooleanTrue};
    frameProperties = CreateCFTypeDictionary(keys, values, 1);
  }

  std::unique_ptr<RTCFrameEncodeParams> encodeParams;
  encodeParams.reset(new RTCFrameEncodeParams(self,
                                              codecSpecificInfo,
                                              _width,
                                              _height,
                                              frame.timeStampNs / rtc::kNumNanosecsPerMillisec,
                                              frame.timeStamp,
                                              frame.rotation));
  encodeParams->codecSpecificInfo.packetizationMode = _packetizationMode;

  // Update the bitrate if needed.
  [self setBitrateBps:_bitrateAdjuster->GetAdjustedBitrateBps() frameRate:_encoderFrameRate];

  OSStatus status = VTCompressionSessionEncodeFrame(_compressionSession,
                                                    pixelBuffer,
                                                    presentationTimeStamp,
                                                    kCMTimeInvalid,
                                                    frameProperties,
                                                    encodeParams.release(),
                                                    nullptr);
  if (frameProperties) {
    CFRelease(frameProperties);
  }
  if (pixelBuffer) {
    CVBufferRelease(pixelBuffer);
  }

  if (status == kVTInvalidSessionErr) {
    // This error occurs when entering foreground after backgrounding the app.
    RTC_LOG(LS_ERROR) << "Invalid compression session, resetting.";
    [self resetCompressionSessionWithPixelFormat:[self pixelFormatOfFrame:frame]];

    return WEBRTC_VIDEO_CODEC_NO_OUTPUT;
  } else if (status == kVTVideoEncoderMalfunctionErr) {
    // Sometimes the encoder malfunctions and needs to be restarted.
    RTC_LOG(LS_ERROR)
        << "Encountered video encoder malfunction error. Resetting compression session.";
    [self resetCompressionSessionWithPixelFormat:[self pixelFormatOfFrame:frame]];

    return WEBRTC_VIDEO_CODEC_NO_OUTPUT;
  } else if (status != noErr) {
    RTC_LOG(LS_ERROR) << "Failed to encode frame with code: " << status;
    return WEBRTC_VIDEO_CODEC_ERROR;
  }
  return WEBRTC_VIDEO_CODEC_OK;
}
```







## 3. RTCVideoEncoderH264.compressionOutputCallback——编码回掉



```c++
// This is the callback function that VideoToolbox calls when encode is
// complete. From inspection this happens on its own queue.
void compressionOutputCallback(void *encoder,
                               void *params,
                               OSStatus status,
                               VTEncodeInfoFlags infoFlags,
                               CMSampleBufferRef sampleBuffer) {
  if (!params) {
    // If there are pending callbacks when the encoder is destroyed, this can happen.
    return;
  }
  std::unique_ptr<RTCFrameEncodeParams> encodeParams(
      reinterpret_cast<RTCFrameEncodeParams *>(params));
  [encodeParams->encoder frameWasEncoded:status
                                   flags:infoFlags
                            sampleBuffer:sampleBuffer
                       codecSpecificInfo:encodeParams->codecSpecificInfo
                                   width:encodeParams->width
                                  height:encodeParams->height
                            renderTimeMs:encodeParams->render_time_ms
                               timestamp:encodeParams->timestamp
                                rotation:encodeParams->rotation];
}
```





### 3.1 RTCVideoEncoderH264.frameWasEncoded:

```c++
- (void)frameWasEncoded:(OSStatus)status
                  flags:(VTEncodeInfoFlags)infoFlags
           sampleBuffer:(CMSampleBufferRef)sampleBuffer
      codecSpecificInfo:(id<RTC_OBJC_TYPE(RTCCodecSpecificInfo)>)codecSpecificInfo
                  width:(int32_t)width
                 height:(int32_t)height
           renderTimeMs:(int64_t)renderTimeMs
              timestamp:(uint32_t)timestamp
               rotation:(RTCVideoRotation)rotation {
  if (status != noErr) {
    RTC_LOG(LS_ERROR) << "H264 encode failed with code: " << status;
    return;
  }
  if (infoFlags & kVTEncodeInfo_FrameDropped) {
    RTC_LOG(LS_INFO) << "H264 encode dropped frame.";
    return;
  }

  BOOL isKeyframe = NO;
  CFArrayRef attachments = CMSampleBufferGetSampleAttachmentsArray(sampleBuffer, 0);
  if (attachments != nullptr && CFArrayGetCount(attachments)) {
    CFDictionaryRef attachment =
        static_cast<CFDictionaryRef>(CFArrayGetValueAtIndex(attachments, 0));
    isKeyframe = !CFDictionaryContainsKey(attachment, kCMSampleAttachmentKey_NotSync);
  }

  if (isKeyframe) {
    RTC_LOG(LS_INFO) << "Generated keyframe";
  }

  __block std::unique_ptr<rtc::Buffer> buffer = std::make_unique<rtc::Buffer>();
  if (!webrtc::H264CMSampleBufferToAnnexBBuffer(sampleBuffer, isKeyframe, buffer.get())) {
    return;
  }

  RTC_OBJC_TYPE(RTCEncodedImage) *frame = [[RTC_OBJC_TYPE(RTCEncodedImage) alloc] init];
  // This assumes ownership of `buffer` and is responsible for freeing it when done.
  frame.buffer = [[NSData alloc] initWithBytesNoCopy:buffer->data()
                                              length:buffer->size()
                                         deallocator:^(void *bytes, NSUInteger size) {
                                           buffer.reset();
                                         }];
  frame.encodedWidth = width;
  frame.encodedHeight = height;
  frame.frameType = isKeyframe ? RTCFrameTypeVideoFrameKey : RTCFrameTypeVideoFrameDelta;
  frame.captureTimeMs = renderTimeMs;
  frame.timeStamp = timestamp;
  frame.rotation = rotation;
  frame.contentType = (_mode == RTCVideoCodecModeScreensharing) ? RTCVideoContentTypeScreenshare :
                                                                  RTCVideoContentTypeUnspecified;
  frame.flags = webrtc::VideoSendTiming::kInvalid;

  _h264BitstreamParser.ParseBitstream(*buffer);
  frame.qp = @(_h264BitstreamParser.GetLastSliceQp().value_or(-1));

	// encode callback 
	// RTCVideoEncoderCallback _callback
  BOOL res = _callback(frame, codecSpecificInfo);
  if (!res) {
    RTC_LOG(LS_ERROR) << "Encode callback failed";
    return;
  }
  _bitrateAdjuster->Update(frame.buffer.length);
}
```



### 3.2 RTCVideoEncoderCallback

```objective-c
int32_t RegisterEncodeCompleteCallback(EncodedImageCallback *callback) override {
    [encoder_ setCallback:^BOOL(RTC_OBJC_TYPE(RTCEncodedImage) * _Nonnull frame,
                                id<RTC_OBJC_TYPE(RTCCodecSpecificInfo)> _Nonnull info) {
      EncodedImage encodedImage = [frame nativeEncodedImage];

      // Handle types that can be converted into one of CodecSpecificInfo's hard coded cases.
      CodecSpecificInfo codecSpecificInfo;
      if ([info isKindOfClass:[RTC_OBJC_TYPE(RTCCodecSpecificInfoH264) class]]) {
        codecSpecificInfo =
            [(RTC_OBJC_TYPE(RTCCodecSpecificInfoH264) *)info nativeCodecSpecificInfo];
      }
    
      EncodedImageCallback::Result res = callback->OnEncodedImage(encodedImage, &codecSpecificInfo);
      return res.error == EncodedImageCallback::Result::OK;
    }];
```



### 3.3 VideoStreamEncoder.OnEncodedImage

```c++
EncodedImageCallback::Result VideoStreamEncoder::OnEncodedImage(
    const EncodedImage& encoded_image,
    const CodecSpecificInfo* codec_specific_info) {
  TRACE_EVENT_INSTANT1("webrtc", "VCMEncodedFrameCallback::Encoded",
                       "timestamp", encoded_image.Timestamp());
  const size_t spatial_idx = encoded_image.SpatialIndex().value_or(0);
  EncodedImage image_copy(encoded_image);

  frame_encode_metadata_writer_.FillTimingInfo(spatial_idx, &image_copy);

  frame_encode_metadata_writer_.UpdateBitstream(codec_specific_info,
                                                &image_copy);

  // Piggyback ALR experiment group id and simulcast id into the content type.
  const uint8_t experiment_id =
      experiment_groups_[videocontenttypehelpers::IsScreenshare(
          image_copy.content_type_)];

  // TODO(ilnik): This will force content type extension to be present even
  // for realtime video. At the expense of miniscule overhead we will get
  // sliced receive statistics.
  RTC_CHECK(videocontenttypehelpers::SetExperimentId(&image_copy.content_type_,
                                                     experiment_id));
  // We count simulcast streams from 1 on the wire. That's why we set simulcast
  // id in content type to +1 of that is actual simulcast index. This is because
  // value 0 on the wire is reserved for 'no simulcast stream specified'.
  RTC_CHECK(videocontenttypehelpers::SetSimulcastId(
      &image_copy.content_type_, static_cast<uint8_t>(spatial_idx + 1)));

  // Currently internal quality scaler is used for VP9 instead of webrtc qp
  // scaler (in no-svc case or if only a single spatial layer is encoded).
  // It has to be explicitly detected and reported to adaptation metrics.
  // Post a task because |send_codec_| requires |encoder_queue_| lock.
  unsigned int image_width = image_copy._encodedWidth;
  unsigned int image_height = image_copy._encodedHeight;
  VideoCodecType codec = codec_specific_info
                             ? codec_specific_info->codecType
                             : VideoCodecType::kVideoCodecGeneric;
  encoder_queue_.PostTask([this, codec, image_width, image_height] {
    RTC_DCHECK_RUN_ON(&encoder_queue_);
    if (codec == VideoCodecType::kVideoCodecVP9 &&
        send_codec_.VP9()->automaticResizeOn) {
      unsigned int expected_width = send_codec_.width;
      unsigned int expected_height = send_codec_.height;
      int num_active_layers = 0;
      for (int i = 0; i < send_codec_.VP9()->numberOfSpatialLayers; ++i) {
        if (send_codec_.spatialLayers[i].active) {
          ++num_active_layers;
          expected_width = send_codec_.spatialLayers[i].width;
          expected_height = send_codec_.spatialLayers[i].height;
        }
      }
      RTC_DCHECK_LE(num_active_layers, 1)
          << "VP9 quality scaling is enabled for "
             "SVC with several active layers.";
      encoder_stats_observer_->OnEncoderInternalScalerUpdate(
          image_width < expected_width || image_height < expected_height);
    }
  });

  // Encoded is called on whatever thread the real encoder implementation run
  // on. In the case of hardware encoders, there might be several encoders
  // running in parallel on different threads.
  encoder_stats_observer_->OnSendEncodedImage(image_copy, codec_specific_info);

  // The simulcast id is signaled in the SpatialIndex. This makes it impossible
  // to do simulcast for codecs that actually support spatial layers since we
  // can't distinguish between an actual spatial layer and a simulcast stream.
  // TODO(bugs.webrtc.org/10520): Signal the simulcast id explicitly.
  int simulcast_id = 0;
  if (codec_specific_info &&
      (codec_specific_info->codecType == kVideoCodecVP8 ||
       codec_specific_info->codecType == kVideoCodecH264 ||
       codec_specific_info->codecType == kVideoCodecGeneric)) {
    simulcast_id = encoded_image.SpatialIndex().value_or(0);
  }

  EncodedImageCallback::Result result =
      sink_->OnEncodedImage(image_copy, codec_specific_info);

  // We are only interested in propagating the meta-data about the image, not
  // encoded data itself, to the post encode function. Since we cannot be sure
  // the pointer will still be valid when run on the task queue, set it to null.
  DataSize frame_size = DataSize::Bytes(image_copy.size());
  image_copy.ClearEncodedData();

  int temporal_index = 0;
  if (codec_specific_info) {
    if (codec_specific_info->codecType == kVideoCodecVP9) {
      temporal_index = codec_specific_info->codecSpecific.VP9.temporal_idx;
    } else if (codec_specific_info->codecType == kVideoCodecVP8) {
      temporal_index = codec_specific_info->codecSpecific.VP8.temporalIdx;
    }
  }
  if (temporal_index == kNoTemporalIdx) {
    temporal_index = 0;
  }

  RunPostEncode(image_copy, clock_->CurrentTime().us(), temporal_index,
                frame_size);

  if (result.error == Result::OK) {
    // In case of an internal encoder running on a separate thread, the
    // decision to drop a frame might be a frame late and signaled via
    // atomic flag. This is because we can't easily wait for the worker thread
    // without risking deadlocks, eg during shutdown when the worker thread
    // might be waiting for the internal encoder threads to stop.
    if (pending_frame_drops_.load() > 0) {
      int pending_drops = pending_frame_drops_.fetch_sub(1);
      RTC_DCHECK_GT(pending_drops, 0);
      result.drop_next_frame = true;
    }
  }

  return result;
}

```



## 4. VideoSendStreamImpl.OnEncodedImage

```c++
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



## 5. RtpVideoSender.OnEncodedImage

```c++
EncodedImageCallback::Result RtpVideoSender::OnEncodedImage(
    const EncodedImage& encoded_image,
    const CodecSpecificInfo* codec_specific_info) {
  fec_controller_->UpdateWithEncodedData(encoded_image.size(),
                                         encoded_image._frameType);
  MutexLock lock(&mutex_);
  RTC_DCHECK(!rtp_streams_.empty());
  if (!active_)
    return Result(Result::ERROR_SEND_FAILED);

  shared_frame_id_++;
  size_t stream_index = 0;
  if (codec_specific_info &&
      (codec_specific_info->codecType == kVideoCodecVP8 ||
       codec_specific_info->codecType == kVideoCodecH264 ||
       codec_specific_info->codecType == kVideoCodecGeneric)) {
    // Map spatial index to simulcast.
    stream_index = encoded_image.SpatialIndex().value_or(0);
  }
  RTC_DCHECK_LT(stream_index, rtp_streams_.size());

  uint32_t rtp_timestamp =
      encoded_image.Timestamp() +
      rtp_streams_[stream_index].rtp_rtcp->StartTimestamp();

  // RTCPSender has it's own copy of the timestamp offset, added in
  // RTCPSender::BuildSR, hence we must not add the in the offset for this call.
  // TODO(nisse): Delete RTCPSender:timestamp_offset_, and see if we can confine
  // knowledge of the offset to a single place.
  if (!rtp_streams_[stream_index].rtp_rtcp->OnSendingRtpFrame(
          encoded_image.Timestamp(), encoded_image.capture_time_ms_,
          rtp_config_.payload_type,
          encoded_image._frameType == VideoFrameType::kVideoFrameKey)) {
    // The payload router could be active but this module isn't sending.
    return Result(Result::ERROR_SEND_FAILED);
  }

  absl::optional<int64_t> expected_retransmission_time_ms;
  if (encoded_image.RetransmissionAllowed()) {
    expected_retransmission_time_ms =
        rtp_streams_[stream_index].rtp_rtcp->ExpectedRetransmissionTimeMs();
  }

  if (IsFirstFrameOfACodedVideoSequence(encoded_image, codec_specific_info)) {
    // If encoder adapter produce FrameDependencyStructure, pass it so that
    // dependency descriptor rtp header extension can be used.
    // If not supported, disable using dependency descriptor by passing nullptr.
    rtp_streams_[stream_index].sender_video->SetVideoStructure(
        (codec_specific_info && codec_specific_info->template_structure)
            ? &*codec_specific_info->template_structure
            : nullptr);
  }

  bool send_result = rtp_streams_[stream_index].sender_video->SendEncodedImage(
      rtp_config_.payload_type, codec_type_, rtp_timestamp, encoded_image,
      params_[stream_index].GetRtpVideoHeader(
          encoded_image, codec_specific_info, shared_frame_id_),
      expected_retransmission_time_ms);
  if (frame_count_observer_) {
    FrameCounts& counts = frame_counts_[stream_index];
    if (encoded_image._frameType == VideoFrameType::kVideoFrameKey) {
      ++counts.key_frames;
    } else if (encoded_image._frameType == VideoFrameType::kVideoFrameDelta) {
      ++counts.delta_frames;
    } else {
      RTC_DCHECK(encoded_image._frameType == VideoFrameType::kEmptyFrame);
    }
    frame_count_observer_->FrameCountUpdated(counts,
                                             rtp_config_.ssrcs[stream_index]);
  }
  if (!send_result)
    return Result(Result::ERROR_SEND_FAILED);

  return Result(Result::OK, rtp_timestamp);
}
```



### RtpTransportControllerSend::CreateRtpVideoSender 创建



## 6. RTPSenderVideo.SendEncodedImage

```c++
bool RTPSenderVideo::SendVideo(
    int payload_type,
    absl::optional<VideoCodecType> codec_type,
    uint32_t rtp_timestamp,
    int64_t capture_time_ms,
    rtc::ArrayView<const uint8_t> payload,
    RTPVideoHeader video_header,
    absl::optional<int64_t> expected_retransmission_time_ms,
    absl::optional<int64_t> estimated_capture_clock_offset_ms) {
#if RTC_TRACE_EVENTS_ENABLED
  TRACE_EVENT_ASYNC_STEP1("webrtc", "Video", capture_time_ms, "Send", "type",
                          FrameTypeToString(video_header.frame_type));
#endif
  RTC_CHECK_RUNS_SERIALIZED(&send_checker_);

  if (video_header.frame_type == VideoFrameType::kEmptyFrame)
    return true;

  if (payload.empty())
    return false;

  int32_t retransmission_settings = retransmission_settings_;
  if (codec_type == VideoCodecType::kVideoCodecH264) {
    // Backward compatibility for older receivers without temporal layer logic.
    retransmission_settings = kRetransmitBaseLayer | kRetransmitHigherLayers;
  }

  MaybeUpdateCurrentPlayoutDelay(video_header);
  if (video_header.frame_type == VideoFrameType::kVideoFrameKey) {
    if (!IsNoopDelay(current_playout_delay_)) {
      // Force playout delay on key-frames, if set.
      playout_delay_pending_ = true;
    }
    if (allocation_) {
      // Send the bitrate allocation on every key frame.
      send_allocation_ = SendVideoLayersAllocation::kSendWithResolution;
    }
  }

  if (video_structure_ != nullptr && video_header.generic) {
    active_decode_targets_tracker_.OnFrame(
        video_structure_->decode_target_protected_by_chain,
        video_header.generic->active_decode_targets,
        video_header.frame_type == VideoFrameType::kVideoFrameKey,
        video_header.generic->frame_id, video_header.generic->chain_diffs);
  }

  const uint8_t temporal_id = GetTemporalId(video_header);
  // No FEC protection for upper temporal layers, if used.
  const bool use_fec = fec_type_.has_value() &&
                       (temporal_id == 0 || temporal_id == kNoTemporalIdx);

  // Maximum size of packet including rtp headers.
  // Extra space left in case packet will be resent using fec or rtx.
  int packet_capacity = rtp_sender_->MaxRtpPacketSize() -
                        (use_fec ? FecPacketOverhead() : 0) -
                        (rtp_sender_->RtxStatus() ? kRtxHeaderSize : 0);

  std::unique_ptr<RtpPacketToSend> single_packet =
      rtp_sender_->AllocatePacket();
  RTC_DCHECK_LE(packet_capacity, single_packet->capacity());
  single_packet->SetPayloadType(payload_type);
  single_packet->SetTimestamp(rtp_timestamp);
  single_packet->set_capture_time_ms(capture_time_ms);

  const absl::optional<AbsoluteCaptureTime> absolute_capture_time =
      absolute_capture_time_sender_.OnSendPacket(
          AbsoluteCaptureTimeSender::GetSource(single_packet->Ssrc(),
                                               single_packet->Csrcs()),
          single_packet->Timestamp(), kVideoPayloadTypeFrequency,
          Int64MsToUQ32x32(single_packet->capture_time_ms() + NtpOffsetMs()),
          /*estimated_capture_clock_offset=*/
          include_capture_clock_offset_ ? estimated_capture_clock_offset_ms
                                        : absl::nullopt);

  auto first_packet = std::make_unique<RtpPacketToSend>(*single_packet);
  auto middle_packet = std::make_unique<RtpPacketToSend>(*single_packet);
  auto last_packet = std::make_unique<RtpPacketToSend>(*single_packet);
  // Simplest way to estimate how much extensions would occupy is to set them.
  AddRtpHeaderExtensions(video_header, absolute_capture_time,
                         /*first_packet=*/true, /*last_packet=*/true,
                         single_packet.get());
  AddRtpHeaderExtensions(video_header, absolute_capture_time,
                         /*first_packet=*/true, /*last_packet=*/false,
                         first_packet.get());
  AddRtpHeaderExtensions(video_header, absolute_capture_time,
                         /*first_packet=*/false, /*last_packet=*/false,
                         middle_packet.get());
  AddRtpHeaderExtensions(video_header, absolute_capture_time,
                         /*first_packet=*/false, /*last_packet=*/true,
                         last_packet.get());

  RTC_DCHECK_GT(packet_capacity, single_packet->headers_size());
  RTC_DCHECK_GT(packet_capacity, first_packet->headers_size());
  RTC_DCHECK_GT(packet_capacity, middle_packet->headers_size());
  RTC_DCHECK_GT(packet_capacity, last_packet->headers_size());
  RtpPacketizer::PayloadSizeLimits limits;
  limits.max_payload_len = packet_capacity - middle_packet->headers_size();

  RTC_DCHECK_GE(single_packet->headers_size(), middle_packet->headers_size());
  limits.single_packet_reduction_len =
      single_packet->headers_size() - middle_packet->headers_size();

  RTC_DCHECK_GE(first_packet->headers_size(), middle_packet->headers_size());
  limits.first_packet_reduction_len =
      first_packet->headers_size() - middle_packet->headers_size();

  RTC_DCHECK_GE(last_packet->headers_size(), middle_packet->headers_size());
  limits.last_packet_reduction_len =
      last_packet->headers_size() - middle_packet->headers_size();

  bool has_generic_descriptor =
      first_packet->HasExtension<RtpGenericFrameDescriptorExtension00>() ||
      first_packet->HasExtension<RtpDependencyDescriptorExtension>();

  // Minimization of the vp8 descriptor may erase temporal_id, so use
  // |temporal_id| rather than reference |video_header| beyond this point.
  if (has_generic_descriptor) {
    MinimizeDescriptor(&video_header);
  }

  // TODO(benwright@webrtc.org) - Allocate enough to always encrypt inline.
  rtc::Buffer encrypted_video_payload;
  if (frame_encryptor_ != nullptr) {
    if (!has_generic_descriptor) {
      return false;
    }

    const size_t max_ciphertext_size =
        frame_encryptor_->GetMaxCiphertextByteSize(cricket::MEDIA_TYPE_VIDEO,
                                                   payload.size());
    encrypted_video_payload.SetSize(max_ciphertext_size);

    size_t bytes_written = 0;

    // Enable header authentication if the field trial isn't disabled.
    std::vector<uint8_t> additional_data;
    if (generic_descriptor_auth_experiment_) {
      additional_data = RtpDescriptorAuthentication(video_header);
    }

    if (frame_encryptor_->Encrypt(
            cricket::MEDIA_TYPE_VIDEO, first_packet->Ssrc(), additional_data,
            payload, encrypted_video_payload, &bytes_written) != 0) {
      return false;
    }

    encrypted_video_payload.SetSize(bytes_written);
    payload = encrypted_video_payload;
  } else if (require_frame_encryption_) {
    RTC_LOG(LS_WARNING)
        << "No FrameEncryptor is attached to this video sending stream but "
           "one is required since require_frame_encryptor is set";
  }

  std::unique_ptr<RtpPacketizer> packetizer =
      RtpPacketizer::Create(codec_type, payload, limits, video_header);

  // TODO(bugs.webrtc.org/10714): retransmission_settings_ should generally be
  // replaced by expected_retransmission_time_ms.has_value(). For now, though,
  // only VP8 with an injected frame buffer controller actually controls it.
  const bool allow_retransmission =
      expected_retransmission_time_ms.has_value()
          ? AllowRetransmission(temporal_id, retransmission_settings,
                                expected_retransmission_time_ms.value())
          : false;
  const size_t num_packets = packetizer->NumPackets();

  if (num_packets == 0)
    return false;

  bool first_frame = first_frame_sent_();
  std::vector<std::unique_ptr<RtpPacketToSend>> rtp_packets;
  for (size_t i = 0; i < num_packets; ++i) {
    std::unique_ptr<RtpPacketToSend> packet;
    int expected_payload_capacity;
    // Choose right packet template:
    if (num_packets == 1) {
      packet = std::move(single_packet);
      expected_payload_capacity =
          limits.max_payload_len - limits.single_packet_reduction_len;
    } else if (i == 0) {
      packet = std::move(first_packet);
      expected_payload_capacity =
          limits.max_payload_len - limits.first_packet_reduction_len;
    } else if (i == num_packets - 1) {
      packet = std::move(last_packet);
      expected_payload_capacity =
          limits.max_payload_len - limits.last_packet_reduction_len;
    } else {
      packet = std::make_unique<RtpPacketToSend>(*middle_packet);
      expected_payload_capacity = limits.max_payload_len;
    }

    packet->set_first_packet_of_frame(i == 0);

    if (!packetizer->NextPacket(packet.get()))
      return false;
    RTC_DCHECK_LE(packet->payload_size(), expected_payload_capacity);
    if (!rtp_sender_->AssignSequenceNumber(packet.get()))
      return false;

    packet->set_allow_retransmission(allow_retransmission);
    packet->set_is_key_frame(video_header.frame_type ==
                             VideoFrameType::kVideoFrameKey);

    // Put packetization finish timestamp into extension.
    if (packet->HasExtension<VideoTimingExtension>()) {
      packet->set_packetization_finish_time_ms(clock_->TimeInMilliseconds());
    }

    packet->set_fec_protect_packet(use_fec);

    if (red_enabled()) {
      // TODO(sprang): Consider packetizing directly into packets with the RED
      // header already in place, to avoid this copy.
      std::unique_ptr<RtpPacketToSend> red_packet(new RtpPacketToSend(*packet));
      BuildRedPayload(*packet, red_packet.get());
      red_packet->SetPayloadType(*red_payload_type_);
      red_packet->set_is_red(true);

      // Send |red_packet| instead of |packet| for allocated sequence number.
      red_packet->set_packet_type(RtpPacketMediaType::kVideo);
      red_packet->set_allow_retransmission(packet->allow_retransmission());
      rtp_packets.emplace_back(std::move(red_packet));
    } else {
      packet->set_packet_type(RtpPacketMediaType::kVideo);
      rtp_packets.emplace_back(std::move(packet));
    }

    if (first_frame) {
      if (i == 0) {
        RTC_LOG(LS_INFO)
            << "Sent first RTP packet of the first video frame (pre-pacer)";
      }
      if (i == num_packets - 1) {
        RTC_LOG(LS_INFO)
            << "Sent last RTP packet of the first video frame (pre-pacer)";
      }
    }
  }

  LogAndSendToNetwork(std::move(rtp_packets), payload.size());

  // Update details about the last sent frame.
  last_rotation_ = video_header.rotation;

  if (video_header.color_space != last_color_space_) {
    last_color_space_ = video_header.color_space;
    transmit_color_space_next_frame_ = !IsBaseLayer(video_header);
  } else {
    transmit_color_space_next_frame_ =
        transmit_color_space_next_frame_ ? !IsBaseLayer(video_header) : false;
  }

  if (video_header.frame_type == VideoFrameType::kVideoFrameKey ||
      PacketWillLikelyBeRequestedForRestransmitionIfLost(video_header)) {
    // This frame will likely be delivered, no need to populate playout
    // delay extensions until it changes again.
    playout_delay_pending_ = false;
    send_allocation_ = SendVideoLayersAllocation::kDontSend;
  }

  TRACE_EVENT_ASYNC_END1("webrtc", "Video", capture_time_ms, "timestamp",
                         rtp_timestamp);
  return true;
}

bool RTPSenderVideo::SendEncodedImage(
    int payload_type,
    absl::optional<VideoCodecType> codec_type,
    uint32_t rtp_timestamp,
    const EncodedImage& encoded_image,
    RTPVideoHeader video_header,
    absl::optional<int64_t> expected_retransmission_time_ms) {
  if (frame_transformer_delegate_) {
    // The frame will be sent async once transformed.
    return frame_transformer_delegate_->TransformFrame(
        payload_type, codec_type, rtp_timestamp, encoded_image, video_header,
        expected_retransmission_time_ms);
  }
  return SendVideo(payload_type, codec_type, rtp_timestamp,
                   encoded_image.capture_time_ms_, encoded_image, video_header,
                   expected_retransmission_time_ms);
}
```




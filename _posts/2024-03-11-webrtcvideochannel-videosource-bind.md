---
layout: post
title: WebRtcVideoChannel 和 VideoSourceInterface 的绑定时机
date: 2024-03-11 14:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq
---


* content
{:toc}

---

## 前言

1. 从VideoTrack中获取到VideoTrackSource，绑定到WebRtcVideoChannel中。
   同时对视频进行一些属性配置，通过VideoOption，比如video_noise_reduction，is_screencast等；
2. 在WebRtcVideoChannel中 存放了WebRtcVideoSendStream。在 unify plan 中，一个mline 对应只有 就一个 WebRtcVideoSendStream。在WebRtcVideoChannel::AddSendStream 创建了 WebRtcVideoSendStream；
3. **VideoSourceInterface 和  webrtc::internal::VideoSendStream 进行了绑定；**WebRtcVideoSendStream 中 保存了 VideoSourceInterface * source； 对应一些策略；



### 1. VideoRtpSender.SetSend

pc/rtp_sender.h

1. 从VideoTrack中获取到VideoTrackSource，绑定到WebRtcVideoChannel中。
2. 同时对视频进行一些属性配置，通过VideoOption，比如video_noise_reduction，is_screencast等；

```cpp
void VideoRtpSender::SetSend() {
  RTC_DCHECK(!stopped_);
  RTC_DCHECK(can_send_track());
  if (!media_channel_) {
    RTC_LOG(LS_ERROR) << "SetVideoSend: No video channel exists.";
    return;
  }
  cricket::VideoOptions options;
  // VideoTrackSource
  VideoTrackSourceInterface* source = video_track()->GetSource();
  if (source) {
    options.is_screencast = source->is_screencast();
    options.video_noise_reduction = source->needs_denoising();
  }
  options.content_hint = cached_track_content_hint_;
  switch (cached_track_content_hint_) {
    case VideoTrackInterface::ContentHint::kNone:
      break;
    case VideoTrackInterface::ContentHint::kFluid:
      options.is_screencast = false;
      break;
    case VideoTrackInterface::ContentHint::kDetailed:
    case VideoTrackInterface::ContentHint::kText:
      options.is_screencast = true;
      break;
  }
  bool success = worker_thread_->Invoke<bool>(RTC_FROM_HERE, [&] {
    // video_media_channel() 返回的就是WebRtcVideoChannel，把ssrc和videotrack 丢入
    return video_media_channel()->SetVideoSend(ssrc_, &options, video_track());
  });
  RTC_DCHECK(success);
}
```



### 2. WebRtcVideoChannel.SetVideoSend

media/engine/webrtc_video_engine.cc

1. 通过ssrc 找到 WebRtcVideoSendStream，在unify plan 中，一个mline 对应只有 就一个 WebRtcVideoSendStream；
2. WebRtcVideoSendStream 是在上面章节创建并加入send_streams_中；
3. 在WebRtcVideoChannel::AddSendStream 创建了 WebRtcVideoSendStream；WebRtcVideoSendStream是根据StreamParams 创建。**SenderOptions 转换为 StreamParams，根据 StreamParams 生成 WebRtcVideoSendStream，这三者是一一对应关系；**

```cpp
bool WebRtcVideoChannel::SetVideoSend(
    uint32_t ssrc,
    const VideoOptions* options,
    rtc::VideoSourceInterface<webrtc::VideoFrame>* source) {
  ...
  RTC_LOG(LS_INFO) << "SetVideoSend (ssrc= " << ssrc << ", options: "
                   << (options ? options->ToString() : "nullptr")
                   << ", source = " << (source ? "(source)" : "nullptr") << ")";
    // 通过ssrc 找到 WebRtcVideoSendStream
  // WebRtcVideoSendStream 是在上面章节创建并加入send_streams_中
  // 是在WebRtcVideoChannel::AddSendStream 创建了 WebRtcVideoSendStream
  // unify plan 中，一个mline 对应只有 就一个 WebRtcVideoSendStream
  const auto& kv = send_streams_.find(ssrc);
  if (kv == send_streams_.end()) {
    // Allow unknown ssrc only if source is null.
    RTC_CHECK(source == nullptr);
    RTC_LOG(LS_ERROR) << "No sending stream on ssrc " << ssrc;
    return false;
  }
  ///// !!!!!!!
  ///WebRtcVideoSendStream 和VideoSource 绑定
  /////
  return kv->second->SetVideoSend(options, source);
}
```



### 3. WebRtcVideoSendStream::SetVideoSend

media/engine/webrtc_video_engine.cc

1. 如果VideoOptions的 is_screencast 属性发生改变，则会重新创建 codec；
2. **VideoSourceInterface 和  webrtc::internal::VideoSendStream 进行了绑定；**
3. WebRtcVideoSendStream 中 保存了 VideoSourceInterface * source；

```cpp
bool WebRtcVideoChannel::WebRtcVideoSendStream::SetVideoSend(
 const VideoOptions* options,
 	rtc::VideoSourceInterface<webrtc::VideoFrame>* source) {
  TRACE_EVENT0("webrtc", "WebRtcVideoSendStream::SetVideoSend");
  RTC_DCHECK_RUN_ON(&thread_checker_);

  if (options) {
     VideoOptions old_options = parameters_.options;
     parameters_.options.SetAll(*options);
     if (parameters_.options.is_screencast.value_or(false) !=
             old_options.is_screencast.value_or(false) &&
         parameters_.codec_settings) {
       // If screen content settings change, we may need to recreate the codec
       // instance so that the correct type is used.

       // 1. 创建 webrtc::VideoSendStream
       SetCodec(*parameters_.codec_settings);
       // Mark screenshare parameter as being updated, then test for any other
       // changes that may require codec reconfiguration.
       old_options.is_screencast = options->is_screencast;
     }
     if (parameters_.options != old_options) {
       // 2. 创建编码器
       ReconfigureEncoder();
     }
  }

  if (source_ && stream_) {
     // webrtc::internal::VideoSendStream stream_
     stream_->SetSource(nullptr, webrtc::DegradationPreference::DISABLED);
  }
  // 保存了videosource
  // Switch to the new source.
  source_ = source;
  // stream_ 是空的
  // channel 把 source 存了起来，但此时 channel 的 stream（`WebRtcVideoSendStream`）还没有创建，需要等 		`SetRemoteDescription` 才会创建；
  // RecreateWebRtcStream 时候创建VideoSendStream
  if (source && stream_) {
    // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    // 这段语句是把源与编码器对象关联到一起!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    // webrtc::internal::VideoSendStream stream_
    // 
    // GetDegradationPreference， 网络和cpu不足等的情况狭隘，视频降级策略，降视频帧率或者视频分辨率
 		stream_->SetSource(source_, GetDegradationPreference());
  }
  return true;
}
```



#### 3.1 WebRtcVideoSendStream::GetDegradationPreference

media/engine/webrtc_video_engine.cc

视频降级策略，视频流畅度优先，视频清晰度优先，或者balance；

```cpp
webrtc::DegradationPreference
WebRtcVideoChannel::WebRtcVideoSendStream::GetDegradationPreference() const {
  // Do not adapt resolution for screen content as this will likely
  // result in blurry and unreadable text.
  // |this| acts like a VideoSource to make sure SinkWants are handled on the
  // correct thread.
  if (!enable_cpu_overuse_detection_) {
    return webrtc::DegradationPreference::DISABLED;
  }

  webrtc::DegradationPreference degradation_preference;
  if (rtp_parameters_.degradation_preference.has_value()) {
    degradation_preference = *rtp_parameters_.degradation_preference;
  } else {
    if (parameters_.options.content_hint ==
        webrtc::VideoTrackInterface::ContentHint::kFluid) {
      degradation_preference =
          webrtc::DegradationPreference::MAINTAIN_FRAMERATE;
    } else if (parameters_.options.is_screencast.value_or(false) ||
               parameters_.options.content_hint ==
                   webrtc::VideoTrackInterface::ContentHint::kDetailed ||
               parameters_.options.content_hint ==
                   webrtc::VideoTrackInterface::ContentHint::kText) {
      degradation_preference =
          webrtc::DegradationPreference::MAINTAIN_RESOLUTION;
    } else if (IsEnabled(call_->trials(), "WebRTC-Video-BalancedDegradation")) {
      // Standard wants balanced by default, but it needs to be tuned first.
      degradation_preference = webrtc::DegradationPreference::BALANCED;
    } else {
      // Keep MAINTAIN_FRAMERATE by default until BALANCED has been tuned for
      // all codecs and launched.
      degradation_preference =
          webrtc::DegradationPreference::MAINTAIN_FRAMERATE;
    }
  }

  return degradation_preference;
}
```


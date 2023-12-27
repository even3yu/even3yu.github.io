---
layout: post
title: AudioTransportImpl
date: 2023-12-20 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio
---


* content
{:toc}

---

## 0. 前言

音频的数据流向。重点介绍一下数据流转中心 AudioTransportImpl 在整个环节中扮演的重要角色。

![audio]({{ site.url }}{{ site.baseurl }}/images/audio-transport-impl.assets/audio_transportimpl.jpeg)

数据流转中心 AudioTransportImpl 实现了采集数据处理接口 RecordDataIsAvailbale和播放数据处理接口 NeedMorePlayData。RecordDataIsAvailbale 负责采集音频数据的处理和将其分发到所有的发送 Streams。NeedMorePlayData 负责混音所有接收到的 Streams，然后输送给 APM 作为一路参考信号处理，最后将其重采样到请求输出的采样率。

`webrtc::AudioTransport` 是一个适配和胶水模块，它把 `AudioDeviceModule` 的音频数据采集和 `webrtc::AudioProcessing` 的音频数据处理及 `webrtc::AudioSender`/`webrtc::AudioSendStream` 的音频数据编码和发送控制粘起来，`webrtc::AudioTransport` 把采集的音频数据送给 `webrtc::AudioProcessing` 处理，之后再把处理后的数据给到 `webrtc::AudioSender`/`webrtc::AudioSendStream` 编码发送出去。
https://www.jianshu.com/p/64d40d7ca74a





## 1. ??? RecordedDataIsAvailable 内部主要流程

1. 由硬件采集过来的音频数据，直接重采样到发送采样率; 降低或者提高音频的通道数

2. 由音频前处理针对重采样之后的音频数据进行 3A 处理

3. VAD 处理???

4. 数字增益调整采集音量？？？

5. 音频数据回调外部进行外部前处理

6. 混音发送端所有需要发送的音频数据，包括采集的数据和伴音的数据？？？， 这个放这里吗？

7. 计算音频数据的能量值

8. 将其分发到所有的发送 Streams

   最后`SendAudioData`阶段，先遍历`sending_streams_`除了第一个`AudioSendStream`，新建`AudioFrame`拷贝`audio_frame`数据，这里必须要拷贝，因为每个`AudioSendStream`都独立编码处理音频帧，而第一个`AudioSendStream`不需要拷贝数据直接将`audio_frame`提交给其处理。

https://www.jianshu.com/p/3254fbbc381c



>  **为什么需要发送多份数据**
>
> `webrtc::AudioTransport` 支持把录制获得的同一份数据同时发送给多个 `webrtc::AudioSender`/`webrtc::AudioSendStream`，`webrtc::AudioSendStream` 用于管理音频数据的编码和编码数据的发送控制，这也就意味着，WebRTC 的音频数据处理管线，支持同时把录制获得的音频数据，以不同的编码方式和编码数据发送控制机制及策略发送到不同的网络，比如一路发送到基于 UDP 传输的 RTC 网络，另一路发送到基于 TCP 传输的 RTMP 网络。
>
> https://www.jianshu.com/p/64d40d7ca74a
> 



audio/audio_transport_impl.cc

```cpp
// Not used in Chromium. Process captured audio and distribute to all sending
// streams, and try to do this at the lowest possible sample rate.
int32_t AudioTransportImpl::RecordedDataIsAvailable(
    const void* audio_data,
    const size_t number_of_frames,
    const size_t bytes_per_sample,
    const size_t number_of_channels,
    const uint32_t sample_rate,
    const uint32_t audio_delay_milliseconds,
    const int32_t /*clock_drift*/,
    const uint32_t /*volume*/,
    const bool key_pressed,
    uint32_t& /*new_mic_volume*/) {  // NOLINT: to avoid changing APIs
  RTC_DCHECK(audio_data);
  RTC_DCHECK_GE(number_of_channels, 1);
  RTC_DCHECK_LE(number_of_channels, 2);
  RTC_DCHECK_EQ(2 * number_of_channels, bytes_per_sample);
  RTC_DCHECK_GE(sample_rate, AudioProcessing::NativeRate::kSampleRate8kHz);
  // 100 = 1 second / data duration (10 ms).
  RTC_DCHECK_EQ(number_of_frames * 100, sample_rate);
  RTC_DCHECK_LE(bytes_per_sample * number_of_frames * number_of_channels,
                AudioFrame::kMaxDataSizeBytes);

  int send_sample_rate_hz = 0;
  size_t send_num_channels = 0;
  bool swap_stereo_channels = false;
  {
    MutexLock lock(&capture_lock_);
    send_sample_rate_hz = send_sample_rate_hz_;
    send_num_channels = send_num_channels_;
    swap_stereo_channels = swap_stereo_channels_;
  }

  std::unique_ptr<AudioFrame> audio_frame(new AudioFrame());
  // 在不丢音频信息的情况下，选择最低的采样率和通道数，
	// 选择本机最小的设备输入和编码采样频率，最小的通道数配置AudioFrame
  InitializeCaptureFrame(sample_rate, send_sample_rate_hz, number_of_channels,
                         send_num_channels, audio_frame.get());
  // 重采样到发送的采样率
  // 重采样 降低或者提高音频的通道数
  voe::RemixAndResample(static_cast<const int16_t*>(audio_data),
                        number_of_frames, number_of_channels, sample_rate,
                        &capture_resampler_, audio_frame.get());
  // 由音频前处理针对重采样之后的音频数据进行 3A 处理
  ProcessCaptureFrame(audio_delay_milliseconds, key_pressed,
                      swap_stereo_channels, audio_processing_,
                      audio_frame.get());
  // VAD 处理
  // Typing detection (utilizes the APM/VAD decision). We let the VAD determine
  // if we're using this feature or not.
  // TODO(solenberg): GetConfig() takes a lock. Work around that.
  bool typing_detected = false;
  if (audio_processing_ &&
      audio_processing_->GetConfig().voice_detection.enabled) {
    if (audio_frame->vad_activity_ != AudioFrame::kVadUnknown) {
      bool vad_active = audio_frame->vad_activity_ == AudioFrame::kVadActive;
      typing_detected = typing_detection_.Process(key_pressed, vad_active);
    }
  }

  // Copy frame and push to each sending stream. The copy is required since an
  // encoding task will be posted internally to each stream.
  {
    MutexLock lock(&capture_lock_);
    typing_noise_detected_ = typing_detected;
  }

  if (media_mode_enable) {
    MixOrReplaceAudioWithMediaAudio(audio_frame);
    bMediaShareStarted = true;
  }

  if (!media_mode_enable && bMediaShareStarted) {
    audio_processing_->EmptyQueuedLoopbackAudio();
    bMediaShareStarted = false;
    mediaShareCount++;
  }

  // 音频数据回调外部进行外部前处理
  {
    MutexLock lock(&capture_lock_);
    if (raw_audio_sink_) {
      raw_audio_sink_->OnData(
          audio_frame->data(), sizeof(int16_t) * 8,
          audio_frame->sample_rate_hz_, audio_frame->num_channels_,
          audio_frame->samples_per_channel_, absl::nullopt);
    }
  }

  // 计算音频数据的能量值
  // 将其分发到所有的发送 Streams
  RTC_DCHECK_GT(audio_frame->samples_per_channel_, 0);
  if (async_audio_processing_)
    async_audio_processing_->Process(std::move(audio_frame));
  else
    SendProcessedData(std::move(audio_frame));

  return 0;
}
```



### 1.1 RemixAndResample

audio/remix_resample.cc

重采样 降低或者提高音频的通道数。



### 1.2 AudioSendStream::SendAudioData

audio/audio_send_stream.cc

```cpp
void AudioSendStream::SendAudioData(std::unique_ptr<AudioFrame> audio_frame) {
  RTC_CHECK_RUNS_SERIALIZED(&audio_capture_race_checker_);
  RTC_DCHECK_GT(audio_frame->sample_rate_hz_, 0);
  double duration = static_cast<double>(audio_frame->samples_per_channel_) /
                    audio_frame->sample_rate_hz_;
  {
    // Note: SendAudioData() passes the frame further down the pipeline and it
    // may eventually get sent. But this method is invoked even if we are not
    // connected, as long as we have an AudioSendStream (created as a result of
    // an O/A exchange). This means that we are calculating audio levels whether
    // or not we are sending samples.
    // TODO(https://crbug.com/webrtc/10771): All "media-source" related stats
    // should move from send-streams to the local audio sources or tracks; a
    // send-stream should not be required to read the microphone audio levels.
    MutexLock lock(&audio_level_lock_);
    // 计算能量值
    audio_level_.ComputeLevel(*audio_frame, duration);
  }
  channel_send_->ProcessAndEncodeAudio(std::move(audio_frame));
}
```



#### 1.2.1 AudioSendStream

```less
AudioSender
webrtc::AudioSendStream
internal::AudioSendStream
```



## 2. ??? NeedMorePlayData 内部主要流程

1. 混音所有接收到的 Streams 的音频数据
   1. 计算输出采样率 CalculateOutputFrequency()
   2. 从 Source 收集音频数据 GetAudioFromSources() ,选取没有mute，且能量最大的三路进行混音
   3. 执行混音操作 FrameCombiner::Combine()
2. 特定条件下，进行噪声注入，用于采集侧作为参考信号???
3. 对本地伴音进行混音操作
4. 数字增益调整播放音量???
5. 音频数据回调外部进行外部前处理
6. 计算音频数据的能量值???
7. 将音频重采样到请求输出的采样率
8. 将音频数据输送给 APM 作为一路参考信号处理??? 为什么在这里处理



audio/audio_transport_impl.cc

```cpp
// Mix all received streams, feed the result to the AudioProcessing module, then
// resample the result to the requested output rate.
int32_t AudioTransportImpl::NeedMorePlayData(const size_t nSamples,
                                             const size_t nBytesPerSample,
                                             const size_t nChannels,
                                             const uint32_t samplesPerSec,
                                             void* audioSamples,
                                             size_t& nSamplesOut,
                                             int64_t* elapsed_time_ms,
                                             int64_t* ntp_time_ms) {
  RTC_DCHECK_EQ(sizeof(int16_t) * nChannels, nBytesPerSample);
  RTC_DCHECK_GE(nChannels, 1);
  RTC_DCHECK_LE(nChannels, 2);
  RTC_DCHECK_GE(
      samplesPerSec,
      static_cast<uint32_t>(AudioProcessing::NativeRate::kSampleRate8kHz));

  // 100 = 1 second / data duration (10 ms).
  RTC_DCHECK_EQ(nSamples * 100, samplesPerSec);
  RTC_DCHECK_LE(nBytesPerSample * nSamples * nChannels,
                AudioFrame::kMaxDataSizeBytes);

  // 混音所有接收到的 Streams 的音频数据
  mixer_->Mix(nChannels, &mixed_frame_);
  *elapsed_time_ms = mixed_frame_.elapsed_time_ms_;
  *ntp_time_ms = mixed_frame_.ntp_time_ms_;

  // 将音频数据输送给 APM 作为一路参考信号处理
  if (audio_processing_) {
    const auto error =
        ProcessReverseAudioFrame(audio_processing_, &mixed_frame_);
    RTC_DCHECK_EQ(error, AudioProcessing::kNoError);
  }
  
  if (media_mode_enable) {
    ProcessReverseAudioFrame(media_apm_.get(), &mixed_frame_);
   }
  
  // 对本地伴音进行混音操作

  // 音频数据回调外部进行外部前处理
  if (audio_playout_sink_) {
    audio_playout_sink_->OnData(mixed_frame_.data(), 16,
      mixed_frame_.sample_rate_hz_, mixed_frame_.num_channels_,
      mixed_frame_.samples_per_channel_);
  }

  // 将音频重采样到请求输出的采样率
  nSamplesOut = Resample(mixed_frame_, samplesPerSec, &render_resampler_,
                         static_cast<int16_t*>(audioSamples));
  RTC_DCHECK_EQ(nSamplesOut, nChannels * nSamples);
  return 0;
}
```



## 3. 为什么需要FineAudioBuffer和AudioDeviceBuffer

由上图的数据流向发现，为什么需要FineAudioBuffer和AudioDeviceBuffer？因为WebRTC 的音频流水线只支持处理 10 ms 的数据，不同的操作系统平台提供了不同的采集和播放时长的音频数据，不同的采样率也会提供不同时长的数据。例如 iOS 上，16K 采样率会提供 8ms 的音频数据 128 帧；8K 采样率会提供 16ms 的音频数据 128帧；48K 采样率会提供 10.67ms的音频数据 512 帧。

AudioDeviceModule 播放和采集的数据，总会通过 AudioDeviceBuffer 拿进来或者送出去 10ms 的音频数据。对于不支持采集和播放 10ms 音频数据的平台，在平台的 AudioDeviceModule 和 AudioDeviceBuffer 还会插入一个 FineAudioBuffer，用于将平台的音频数据格式转换为 10ms 的 WebRTC 能处理的音频帧。在 AudioDeviceBuffer 中，还会 10s 定时统计一下当前硬件设备过来的音频数据对应的采样点个数和采样率，可以用于检测当前硬件的一个工作状态。

## 参考

https://worktile.com/kb/p/5925

https://www.jianshu.com/p/3254fbbc381c

https://www.jianshu.com/p/64d40d7ca74a
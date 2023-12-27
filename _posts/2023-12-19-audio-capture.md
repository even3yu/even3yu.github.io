---
layout: post
title: 音频采集
date: 2023-12-19 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio
---


* content
{:toc}

---



## 1. 调用堆栈



![img]({{ site.url }}{{ site.baseurl }}/images/audio-capture.assets/capture-call-stack.jpg)

1. AudioDeviceModule 提供了一个工厂方法用于创建 AudioDeviceModule 实例。
2. 调用 AudioDeviceModule::Init 方法初始化 AudioDevice，主要是根据不同平台的类型，创建相应平台的 AudioDevice。比如，在windows 上是创建 AudioDeviceWindowsCore。
3. 调用 AudioDeviceModule::RegisterAudioCallback 注册一个接收或者发送音频数据的类的实例，此类需要实现 AudioTransport 接口。
4. 调用 AudioDeviceModule::SetRecordingDevice 方法设置录音设备，参数是录音设备的索引。在 Windows 上可以传入参数。 webrtc::AudioDeviceModule::WindowsDeviceType::kDefaultCommunicationDevice。
5. 调用 AudioDeviceModule::InitRecording 方法初始化录音逻辑。
6. 通过 AudioDeviceModule::StartRecording 启动录音。
7. 通过调用 AudioDeviceModule::StopRecording 方法停止录音。
8. 如果停止音频设备，需要调用 AudioDeviceModule::Terminate 来释放资源。

通过以上步骤就可以采集到音频的 PCM 数据。



## 2. AudioDeviceModule::Create

modules/audio_device/audio_device_impl.cc

```cpp
// static
rtc::scoped_refptr<AudioDeviceModule> AudioDeviceModule::Create(
    AudioLayer audio_layer,
    TaskQueueFactory* task_queue_factory) {
  RTC_LOG(INFO) << __FUNCTION__;
  return AudioDeviceModule::CreateForTest(audio_layer, task_queue_factory);
}

// static
rtc::scoped_refptr<AudioDeviceModuleForTest> AudioDeviceModule::CreateForTest(
    AudioLayer audio_layer,
    TaskQueueFactory* task_queue_factory) {
  RTC_LOG(INFO) << __FUNCTION__;

  // The "AudioDeviceModule::kWindowsCoreAudio2" audio layer has its own
  // dedicated factory method which should be used instead.
  if (audio_layer == AudioDeviceModule::kWindowsCoreAudio2) {
    RTC_LOG(LS_ERROR) << "Use the CreateWindowsCoreAudioAudioDeviceModule() "
                         "factory method instead for this option.";
    return nullptr;
  }

  // Create the generic reference counted (platform independent) implementation.
  rtc::scoped_refptr<AudioDeviceModuleImpl> audioDevice(
      new rtc::RefCountedObject<AudioDeviceModuleImpl>(audio_layer,
                                                       task_queue_factory));

  // Ensure that the current platform is supported.
  if (audioDevice->CheckPlatform() == -1) {
    return nullptr;
  }

  // Create the platform-dependent implementation.
  if (audioDevice->CreatePlatformSpecificObjects() == -1) {
    return nullptr;
  }

  // Ensure that the generic audio buffer can communicate with the platform
  // specific parts.
  if (audioDevice->AttachAudioBuffer() == -1) {
    return nullptr;
  }

  return audioDevice;
}
```



## 3. AudioDeviceModuleImpl::AudioDeviceModuleImpl

modules/audio_device/audio_device_impl.cc



## 4. AudioDeviceModuleImpl::Init

modules/audio_device/audio_device_impl.cc

```cpp
int32_t AudioDeviceModuleImpl::Init() {
  RTC_LOG(INFO) << __FUNCTION__;
  if (initialized_)
    return 0;
  RTC_CHECK(audio_device_);
  AudioDeviceGeneric::InitStatus status = audio_device_->Init();
  RTC_HISTOGRAM_ENUMERATION(
      "WebRTC.Audio.InitializationResult", static_cast<int>(status),
      static_cast<int>(AudioDeviceGeneric::InitStatus::NUM_STATUSES));
  if (status != AudioDeviceGeneric::InitStatus::OK) {
    RTC_LOG(LS_ERROR) << "Audio device initialization failed.";
    return -1;
  }
  initialized_ = true;
  return 0;
}
```



## 5. !!! AudioDeviceModuleImpl::RegisterAudioCallback

modules/audio_device/audio_device_impl.cc

```cpp
int32_t AudioDeviceModuleImpl::RegisterAudioCallback(
    AudioTransport* audioCallback) {
  RTC_LOG(INFO) << __FUNCTION__;
  return audio_device_buffer_.RegisterAudioCallback(audioCallback);
}
```

数据从Device，转发到AudioDeviceModule，转达到AudioDeviceBuffer，最后转发到AudioTransportImpl。

### 5.1 AudioDeviceBuffer::RegisterAudioCallback

modules/audio_device/audio_device_buffer.cc

```cpp
int32_t AudioDeviceBuffer::RegisterAudioCallback(
    AudioTransport* audio_callback) {
  RTC_DCHECK_RUN_ON(&main_thread_checker_);
  RTC_LOG(INFO) << __FUNCTION__;
  if (playing_ || recording_) {
    RTC_LOG(LS_ERROR) << "Failed to set audio transport since media was active";
    return -1;
  }
  audio_transport_cb_ = audio_callback;
  return 0;
}
```



### 5.2 AudioDeviceBuffer::GetAudioTransport()

modules/audio_device/audio_device_buffer.cc

```cpp
AudioTransport* AudioDeviceBuffer::GetAudioTransport() const {
  return audio_transport_cb_;
}
```



### 5.3 使用AudioTransport

#### 5.3.0 AudioDeviceBuffer::SetRecordedBuffer

```cpp
int32_t AudioDeviceBuffer::SetRecordedBuffer(const void* audio_buffer,
                                             size_t samples_per_channel) {
  // Copy the complete input buffer to the local buffer.
  const size_t old_size = rec_buffer_.size();
  //  rtc::BufferT<int16_t> rec_buffer_;
  rec_buffer_.SetData(static_cast<const int16_t*>(audio_buffer),
                      rec_channels_ * samples_per_channel);
  // Keep track of the size of the recording buffer. Only updated when the
  // size changes, which is a rare event.
  if (old_size != rec_buffer_.size()) {
    RTC_LOG(LS_INFO) << "Size of recording buffer: " << rec_buffer_.size();
  }

  // Derive a new level value twice per second and check if it is non-zero.
  int16_t max_abs = 0;
  RTC_DCHECK_LT(rec_stat_count_, 50);
  if (++rec_stat_count_ >= 50) {
    // Returns the largest absolute value in a signed 16-bit vector.
    max_abs = WebRtcSpl_MaxAbsValueW16(rec_buffer_.data(), rec_buffer_.size());
    rec_stat_count_ = 0;
    // Set |only_silence_recorded_| to false as soon as at least one detection
    // of a non-zero audio packet is found. It can only be restored to true
    // again by restarting the call.
    if (max_abs > 0) {
      only_silence_recorded_ = false;
    }
  }
  // Update recording stats which is used as base for periodic logging of the
  // audio input state.
  UpdateRecStats(max_abs, samples_per_channel);
  return 0;
}
```

数据写入到 `rec_buffer_`中，然后DeliverRecordedData 从`rec_buffer_`中获取数据。 

#### rtc::BufferT<int16_t> rec_buffer_????



#### 5.3.1 采集的数据——AudioDeviceBuffer::DeliverRecordedData

modules/audio_device/audio_device_buffer.cc

```cpp
int32_t AudioDeviceBuffer::DeliverRecordedData() {
  if (!audio_transport_cb_) {
    RTC_LOG(LS_WARNING) << "Invalid audio transport";
    return 0;
  }
  const size_t frames = rec_buffer_.size() / rec_channels_;
  const size_t bytes_per_frame = rec_channels_ * sizeof(int16_t);
  uint32_t new_mic_level_dummy = 0;
  uint32_t total_delay_ms = play_delay_ms_ + rec_delay_ms_;
  int32_t res = audio_transport_cb_->RecordedDataIsAvailable(
      rec_buffer_.data(), frames, bytes_per_frame, rec_channels_,
      rec_sample_rate_, total_delay_ms, 0, 0, typing_status_,
      new_mic_level_dummy);
  if (res == -1) {
    RTC_LOG(LS_ERROR) << "RecordedDataIsAvailable() failed";
  }
  return 0;
}
```



#### 5.3.2 播放的数据——AudioDeviceBuffer::RequestPlayoutData

modules/audio_device/audio_device_buffer.cc

```cpp
int32_t AudioDeviceBuffer::RequestPlayoutData(size_t samples_per_channel) {
  // The consumer can change the requested size on the fly and we therefore
  // resize the buffer accordingly. Also takes place at the first call to this
  // method.
  const size_t total_samples = play_channels_ * samples_per_channel;
  if (play_buffer_.size() != total_samples) {
    play_buffer_.SetSize(total_samples);
    RTC_LOG(LS_INFO) << "Size of playout buffer: " << play_buffer_.size();
  }

  size_t num_samples_out(0);
  // It is currently supported to start playout without a valid audio
  // transport object. Leads to warning and silence.
  if (!audio_transport_cb_) {
    RTC_LOG(LS_WARNING) << "Invalid audio transport";
    return 0;
  }

  // Retrieve new 16-bit PCM audio data using the audio transport instance.
  int64_t elapsed_time_ms = -1;
  int64_t ntp_time_ms = -1;
  const size_t bytes_per_frame = play_channels_ * sizeof(int16_t);
  uint32_t res = audio_transport_cb_->NeedMorePlayData(
      samples_per_channel, bytes_per_frame, play_channels_, play_sample_rate_,
      play_buffer_.data(), num_samples_out, &elapsed_time_ms, &ntp_time_ms);
  if (res != 0) {
    RTC_LOG(LS_ERROR) << "NeedMorePlayData() failed";
  }

  // Derive a new level value twice per second.
  int16_t max_abs = 0;
  RTC_DCHECK_LT(play_stat_count_, 50);
  if (++play_stat_count_ >= 50) {
    // Returns the largest absolute value in a signed 16-bit vector.
    max_abs =
        WebRtcSpl_MaxAbsValueW16(play_buffer_.data(), play_buffer_.size());
    play_stat_count_ = 0;
  }
  // Update playout stats which is used as base for periodic logging of the
  // audio output state.
  UpdatePlayStats(max_abs, num_samples_out / play_channels_);
  return static_cast<int32_t>(num_samples_out / play_channels_);
}
```



## 6. AudioDeviceModuleImpl::SetRecordingDevice

modules/audio_device/audio_device_impl.cc

```cpp
int32_t AudioDeviceModuleImpl::SetRecordingDevice(WindowsDeviceType device) {
  RTC_LOG(INFO) << __FUNCTION__;
  CHECKinitialized_();
  // AudioDeviceGeneric audio_device_
  return audio_device_->SetRecordingDevice(device);
}
```

该方法`AudioDeviceGeneric::SetRecordingDevice`  只有`AudioDeviceWindowsCore` 才实现了，其他的设备和平台都是空的实现。



## 7. !!! AudioDeviceModuleImpl::InitRecording

modules/audio_device/audio_device_impl.cc

```cpp
int32_t AudioDeviceModuleImpl::InitRecording() {
  RTC_LOG(INFO) << __FUNCTION__;
  CHECKinitialized_();
  // 设备是否初始化
  if (RecordingIsInitialized()) {
    return 0;
  }
  // 初始化采集设备
  int32_t result = audio_device_->InitRecording();
  RTC_LOG(INFO) << "output: " << result;
  RTC_HISTOGRAM_BOOLEAN("WebRTC.Audio.InitRecordingSuccess",
                        static_cast<int>(result == 0));
  return result;
}
```



#### 7.1 AudioDeviceModuleImpl::RecordingIsInitialized

```cpp
bool AudioDeviceModuleImpl::RecordingIsInitialized() const {
  RTC_LOG(INFO) << __FUNCTION__;
  CHECKinitialized__BOOL();
  return audio_device_->RecordingIsInitialized();
}
```



## 8. !!! AudioDeviceModuleImpl::StartRecording

modules/audio_device/audio_device_impl.cc

```cpp
int32_t AudioDeviceModuleImpl::StartRecording() {
  RTC_LOG(INFO) << __FUNCTION__;
  CHECKinitialized_();
  if (Recording()) {
    return 0;
  }
  // AudioDeviceBuffer的状态更新
  audio_device_buffer_.StartRecording();
  // 开始采集
  int32_t result = audio_device_->StartRecording();
  RTC_LOG(INFO) << "output: " << result;
  RTC_HISTOGRAM_BOOLEAN("WebRTC.Audio.StartRecordingSuccess",
                        static_cast<int>(result == 0));
  return result;
}
```

注意，StartRecording 之前，一定要先调用InitRecording。



## 9. AudioDeviceModuleImpl::StopRecording

modules/audio_device/audio_device_impl.cc

```cpp
int32_t AudioDeviceModuleImpl::StopRecording() {
  RTC_LOG(INFO) << __FUNCTION__;
  CHECKinitialized_();
  int32_t result = audio_device_->StopRecording();
  audio_device_buffer_.StopRecording();
  RTC_LOG(INFO) << "output: " << result;
  RTC_HISTOGRAM_BOOLEAN("WebRTC.Audio.StopRecordingSuccess",
                        static_cast<int>(result == 0));
  return result;
}
```



## 10. AudioDeviceModuleImpl::Terminate

modules/audio_device/audio_device_impl.cc

```cpp
int32_t AudioDeviceModuleImpl::Terminate() {
  RTC_LOG(INFO) << __FUNCTION__;
  if (!initialized_)
    return 0;
  if (audio_device_->Terminate() == -1) {
    return -1;
  }
  initialized_ = false;
  return 0;
}
```



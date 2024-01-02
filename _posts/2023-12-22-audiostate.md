---
layout: post
title: AudioState
date: 2023-12-22 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio
---


* content
{:toc}

---


## 1. AudioState 的坐标

![Audio Pipeline]({{ site.url }}{{ site.baseurl }}/images/audiostate.assets/audio-state.png)



## 2. AudioState

WebRTC 音频的静态流水线节点，主要包括 `AudioDeviceModule`，`AudioProcessing`，和 `AudioMixer` 等，这些相关节点状态由 `AudioState` 维护和管理。`AudioState`既是 WebRTC 音频流水线节点的容器，也可用于对这些节点做一些管理和控制，以及将不同节点粘起来构成流水线。

```less
webrtc::internal::AudioState
    std::map<webrtc::AudioSendStream*, StreamProperties> sending_streams_;
    std::unordered_set<webrtc::AudioReceiveStream*> receiving_streams_;
    AudioTransportImpl audio_transport_;
    webrtc::AudioState::Config config_;
        rtc::scoped_refptr<AudioMixer> audio_mixer;
        rtc::scoped_refptr<webrtc::AudioProcessing> audio_processing;
        rtc::scoped_refptr<webrtc::AudioDeviceModule> audio_device_module;
        rtc::scoped_refptr<AsyncAudioProcessing::Factory> async_audio_processing_factory;
```

![image]({{ site.url }}{{ site.baseurl }}/images/audiostate.assets/audiostate.png)


## 3. AudioState 的创建时机

WebRTC 音频静态流水线在 `WebRtcVoiceEngine::Init() `中建立。动态流水线和静态流水线在 AudioState 和 AudioTransport 处交汇。

- 根据需要创建 AudioDeviceModule，
- 然后创建 webrtc::AudioState，webrtc::AudioState 用注入的 AudioMixer 和 AudioProcessing 创建 AudioTransport，
- 随后将 webrtc::AudioState 创建的 AudioTransport 作为 AudioCallback 注册给 AudioDeviceModule。

media/engine/webrtc_voice_engine.cc

```cpp
void WebRtcVoiceEngine::Init() {
  RTC_DCHECK(worker_thread_checker_.IsCurrent());
  RTC_LOG(LS_INFO) << "WebRtcVoiceEngine::Init";

  // create adm process thread and start
  if (!adm_process_thread_.get())
    adm_process_thread_ = webrtc::ProcessThread::Create("AudioDeviceModuleProcessThread");
  if (adm_process_thread_.get())
    adm_process_thread_.get()->Start();

  // TaskQueue expects to be created/destroyed on the same thread.
  low_priority_worker_queue_.reset(
      new rtc::TaskQueue(task_queue_factory_->CreateTaskQueue(
          "rtc-low-prio", webrtc::TaskQueueFactory::Priority::LOW)));

  // Load our audio codec lists.
  // std::vector<AudioCodec> send_codecs_;
  RTC_LOG(LS_VERBOSE) << "Supported send codecs in order of preference:";
  send_codecs_ = CollectCodecs(encoder_factory_->GetSupportedEncoders());
  for (const AudioCodec& codec : send_codecs_) {
    RTC_LOG(LS_VERBOSE) << ToString(codec);
  }

  // std::vector<AudioCodec> recv_codecs_;
  RTC_LOG(LS_VERBOSE) << "Supported recv codecs in order of preference:";
  recv_codecs_ = CollectCodecs(decoder_factory_->GetSupportedDecoders());
  for (const AudioCodec& codec : recv_codecs_) {
    RTC_LOG(LS_VERBOSE) << ToString(codec);
  }

#if defined(WEBRTC_INCLUDE_INTERNAL_AUDIO_DEVICE)
  // No ADM supplied? Create a default one.
  // adm 是可以由外部自定义传入的，在构造函数里
  if (!adm_) {
    adm_ = webrtc::AudioDeviceModule::Create(
        webrtc::AudioDeviceModule::kPlatformDefaultAudio, task_queue_factory_);
  }
#endif  // WEBRTC_INCLUDE_INTERNAL_AUDIO_DEVICE
  RTC_CHECK(adm());
  //register audio device module to adm process thread
  if (adm_process_thread_.get())
    adm_process_thread_.get()->RegisterModule(adm(), RTC_FROM_HERE);
  webrtc::adm_helpers::Init(adm());

  // 配置AudioState::Config
  // Set up AudioState.
  {
    webrtc::AudioState::Config config;
    if (audio_mixer_) {
      config.audio_mixer = audio_mixer_;
    } else {
      config.audio_mixer = webrtc::AudioMixerImpl::Create();
    }
    config.audio_processing = apm_;
    config.audio_device_module = adm_;
    if (audio_frame_processor_)
      config.async_audio_processing_factory =
          new rtc::RefCountedObject<webrtc::AsyncAudioProcessing::Factory>(
              *audio_frame_processor_, *task_queue_factory_);
    audio_state_ = webrtc::AudioState::Create(config);
  }

  // Connect the ADM to our audio path.
  adm()->RegisterAudioCallback(audio_state()->audio_transport());

  // Set default engine options.
  {
    AudioOptions options;
    options.echo_cancellation = true;
    options.auto_gain_control = true;
#if defined(WEBRTC_IOS)
    // On iOS, VPIO provides built-in NS.
    options.noise_suppression = false;
    options.typing_detection = false;
#else
    options.noise_suppression = true;
    options.typing_detection = true;
#endif
    options.experimental_ns = false;
    options.highpass_filter = true;
    options.stereo_swapping = false;
    options.audio_jitter_buffer_max_packets = 200;
    options.audio_jitter_buffer_fast_accelerate = false;
    options.audio_jitter_buffer_min_delay_ms = 0;
    options.audio_jitter_buffer_enable_rtx_handling = false;
    options.experimental_agc = false;
    options.residual_echo_detector = true;
    bool error = ApplyOptions(options);
    RTC_DCHECK(error);
  }
  initialized_ = true;
}
```



## 4. AudioState 的工作

1. 音频流水线的节点的 Getter
2. 管理音频静态流水线和动态流水线的连接，也就是添加和移除 AudioSendStream 和 AudioReceiveStream：把 AudioSendStream 注入给 AudioTransportImpl 或从中移除，把 AudioReceiveStream 添加到 AudioMixer 或从中移除。这些接口主要由 webrtc::Call 使用。
3. 控制播放和录制的启动和停止。



## 5. webrtc::AudioState

call/audio_state.h

```cpp

namespace webrtc {

class AudioTransport;
class AudioPlayoutTrackInterface;

// AudioState holds the state which must be shared between multiple instances of
// webrtc::Call for audio processing purposes.
class AudioState : public rtc::RefCountInterface {
 public:
  struct Config {
    Config();
    ~Config();

    // The audio mixer connected to active receive streams. One per
    // AudioState.
    rtc::scoped_refptr<AudioMixer> audio_mixer;

    // The audio processing module.
    rtc::scoped_refptr<webrtc::AudioProcessing> audio_processing;

    // TODO(solenberg): Temporary: audio device module.
    rtc::scoped_refptr<webrtc::AudioDeviceModule> audio_device_module;

    rtc::scoped_refptr<AsyncAudioProcessing::Factory>
        async_audio_processing_factory;
  };

  virtual AudioProcessing* audio_processing() = 0;
  virtual AudioTransport* audio_transport() = 0;

  // Enable/disable playout of the audio channels. Enabled by default.
  // This will stop playout of the underlying audio device but start a task
  // which will poll for audio data every 10ms to ensure that audio processing
  // happens and the audio stats are updated.
  virtual void SetPlayout(bool enabled) = 0;

  // Enable/disable recording of the audio channels. Enabled by default.
  // This will stop recording of the underlying audio device and no audio
  // packets will be encoded or transmitted.
  virtual void SetRecording(bool enabled) = 0;

  // Audio processing to swap the left and right channels.
  // 在AudioOptions 有配置项
  // api\audio_options.h
  virtual void SetStereoChannelSwapping(bool enable) = 0;
  
  virtual void SetPlayoutTrack(AudioPlayoutTrackInterface* track) = 0;

  static rtc::scoped_refptr<AudioState> Create(
      const AudioState::Config& config);

  ~AudioState() override {}
};
}  // namespace webrtc
```



### 5.1 !!! AudioState::Config

call/audio_state.h

```cpp
class AudioState : public rtc::RefCountInterface {
 public:
  struct Config {
    Config();
    ~Config();

    // The audio mixer connected to active receive streams. One per
    // AudioState.
    rtc::scoped_refptr<AudioMixer> audio_mixer;

    // The audio processing module.
    rtc::scoped_refptr<webrtc::AudioProcessing> audio_processing;

    // TODO(solenberg): Temporary: audio device module.
    rtc::scoped_refptr<webrtc::AudioDeviceModule> audio_device_module;

    rtc::scoped_refptr<AsyncAudioProcessing::Factory>
        async_audio_processing_factory;
  };
  ...
}
```



## 6. webrtc::internal::AudioState

WebRTC 中实际在用的 AudioState 对象为 webrtc::internal::AudioState 的对象。

```less
webrtc::AudioState
webrtc::internal::AudioState
```



### 6.1 AudioState::Create

audio/audio_state.cc

```cpp
rtc::scoped_refptr<AudioState> AudioState::Create(
    const AudioState::Config& config) {
  return new rtc::RefCountedObject<internal::AudioState>(config);
}
```

静态方法创建 `internal::AudioState`。


### 6.2 AudioState::AudioState

audio/audio_state.cc

```cpp
AudioState::AudioState(const AudioState::Config& config)
    : config_(config),
			// AudioTransportImpl audio_transport_;
      audio_transport_(config_.audio_mixer,
                       config_.audio_processing.get(),
                       config_.async_audio_processing_factory.get()) {
  process_thread_checker_.Detach();
  RTC_DCHECK(config_.audio_mixer);
  RTC_DCHECK(config_.audio_device_module);
}
```

创建 `AudioTransportImpl audio_transport_;`。



### 6.3 AudioState::audio_processing

audio/audio_state.cc

```cpp
AudioProcessing* AudioState::audio_processing() {
  return config_.audio_processing.get();
}
```



### 6.4 AudioState::audio_transport

audio/audio_state.cc

```cpp
AudioTransport* AudioState::audio_transport() {
  return &audio_transport_;
}
```



### 6.5 !!! AudioState::SetPlayout

audio/audio_state.cc

```cpp
void AudioState::SetPlayout(bool enabled) {
  RTC_LOG(INFO) << "SetPlayout(" << enabled << ")";
  RTC_DCHECK(thread_checker_.IsCurrent());
  if (playout_enabled_ != enabled) {
    playout_enabled_ = enabled;
    if (enabled) {
      UpdateNullAudioPollerState();
      if (!receiving_streams_.empty()) {
        config_.audio_device_module->StartPlayout();
      }
    } else {
      config_.audio_device_module->StopPlayout();
      UpdateNullAudioPollerState();
    }
  }
}
```



### 6.6 !!! AudioState::SetRecording

audio/audio_state.cc

```cpp
void AudioState::SetRecording(bool enabled) {
  RTC_LOG(INFO) << "SetRecording(" << enabled << ")";
  RTC_DCHECK(thread_checker_.IsCurrent());
  if (recording_enabled_ != enabled) {
    recording_enabled_ = enabled;
    if (enabled) {
      //   std::map<webrtc::AudioSendStream*, StreamProperties> sending_streams_;
      if (!sending_streams_.empty()) {
        config_.audio_device_module->StartRecording();
      }
    } else {
      config_.audio_device_module->StopRecording();
    }
  }
}
```



### 6.7 AudioState::audio_device_module

audio/audio_state.h

```cpp
  AudioDeviceModule* audio_device_module() {
    RTC_DCHECK(config_.audio_device_module);
    return config_.audio_device_module.get();
  }
```



### 6.8 !!! AudioState::AddReceivingStream

audio/audio_state.cc

```cpp
void AudioState::AddReceivingStream(webrtc::AudioReceiveStream* stream) {
  RTC_DCHECK(thread_checker_.IsCurrent());
  RTC_DCHECK_EQ(0, receiving_streams_.count(stream));
  receiving_streams_.insert(stream);
  // rtc::scoped_refptr<AudioMixer> audio_mixer;
  if (!config_.audio_mixer->AddSource(
          static_cast<internal::AudioReceiveStream*>(stream))) {
    RTC_DLOG(LS_ERROR) << "Failed to add source to mixer.";
  }

  // Make sure playback is initialized; start playing if enabled.
  UpdateNullAudioPollerState();
  // 开始播放
  auto* adm = config_.audio_device_module.get();
  if (!adm->Playing()) {
    if (adm->InitPlayout() == 0) {
      if (playout_enabled_) {
        adm->StartPlayout();
      }
    } else {
      RTC_DLOG_F(LS_ERROR) << "Failed to initialize playout.";
    }
  }
}
```



#### 6.8.1 AudioMixerImpl::AddSource

modules/audio_mixer/audio_mixer_impl.cc

```cpp
bool AudioMixerImpl::AddSource(Source* audio_source) {
  RTC_DCHECK(audio_source);
  MutexLock lock(&mutex_);
  RTC_DCHECK(FindSourceInList(audio_source, &audio_source_list_) ==
             audio_source_list_.end())
      << "Source already added to mixer";
  audio_source_list_.emplace_back(new SourceStatus(audio_source, false, 0));
  helper_containers_->resize(audio_source_list_.size());
  return true;
}
```



#### 6.8.2 AudioMixerImpl::HelperContainers::resize

modules/audio_mixer/audio_mixer_impl.cc

```cpp
struct AudioMixerImpl::HelperContainers {
  void resize(size_t size) {
    audio_to_mix.resize(size);
    audio_source_mixing_data_list.resize(size);
    ramp_list.resize(size);
    preferred_rates.resize(size);
  }

  std::vector<AudioFrame*> audio_to_mix;
  std::vector<SourceFrame> audio_source_mixing_data_list;
  std::vector<SourceFrame> ramp_list;
  std::vector<int> preferred_rates;
};
```



### 6.9 AudioState::RemoveReceivingStream

audio/audio_state.cc

```cpp
void AudioState::RemoveReceivingStream(webrtc::AudioReceiveStream* stream) {
  RTC_DCHECK(thread_checker_.IsCurrent());
  auto count = receiving_streams_.erase(stream);
  RTC_DCHECK_EQ(1, count);
  // rtc::scoped_refptr<AudioMixer> audio_mixer;
  // ??? 
  config_.audio_mixer->RemoveSource(
      static_cast<internal::AudioReceiveStream*>(stream));
  UpdateNullAudioPollerState();
  // 停止播放
   if (receiving_streams_.empty() && !playout_track_source_) {
    config_.audio_device_module->StopPlayout();
  }
}
```



#### 6.9.1 AudioMixerImpl::RemoveSource

modules/audio_mixer/audio_mixer_impl.cc

```cpp
void AudioMixerImpl::RemoveSource(Source* audio_source) {
  RTC_DCHECK(audio_source);
  MutexLock lock(&mutex_);
  const auto iter = FindSourceInList(audio_source, &audio_source_list_);
  RTC_DCHECK(iter != audio_source_list_.end()) << "Source not present in mixer";
  audio_source_list_.erase(iter);
}
```



#### 6.9.2 AudioState::UpdateNullAudioPollerState

audio/audio_state.cc

```cpp
void AudioState::UpdateNullAudioPollerState() {
  // Run NullAudioPoller when there are receiving streams and playout is
  // disabled.
  // std::unordered_set<webrtc::AudioReceiveStream*> receiving_streams_;
  // 停止播放，并且 没有stream，则 创建null_audio_poller_
  if (!receiving_streams_.empty() && !playout_enabled_) {
    if (!null_audio_poller_)
      null_audio_poller_ = std::make_unique<NullAudioPoller>(&audio_transport_);
  } else {
    null_audio_poller_.reset();
  }
}
```

1. `playout_enabled_` 是在 AudioState::SetPlayout 赋值；
2. `!receiving_streams_.empty() && !playout_enabled_`， 没有recevie stream，并且 playout_enabled_ = false，则就创建NullAudioPoller；



### 6.10 !!! AudioState::AddSendingStream

audio/audio_state.cc

```cpp
void AudioState::AddSendingStream(webrtc::AudioSendStream* stream,
                                  int sample_rate_hz,
                                  size_t num_channels) {
  RTC_DCHECK(thread_checker_.IsCurrent());
  //  std::map<webrtc::AudioSendStream*, StreamProperties> sending_streams_;
  // StreamProperties properties;
  auto& properties = sending_streams_[stream];
  properties.sample_rate_hz = sample_rate_hz;
  properties.num_channels = num_channels;
  UpdateAudioTransportWithSendingStreams();

  // Make sure recording is initialized; start recording if enabled.
  auto* adm = config_.audio_device_module.get();
  // 没有开启录制，则开启录制
  if (!adm->Recording()) {
    if (adm->InitRecording() == 0) {
      if (recording_enabled_) {
        adm->StartRecording();
      }
    } else {
      RTC_DLOG_F(LS_ERROR) << "Failed to initialize recording.";
    }
  }
}
```



#### 6.10.1 --AudioState::UpdateAudioTransportWithSendingStreams



#### 6.10.2 std::map<webrtc::AudioSendStream*, StreamProperties> sending_streams_



#### 6.10.3 StreamProperties

audio/audio_state.h

```cpp
  struct StreamProperties {
    int sample_rate_hz = 0;
    size_t num_channels = 0;
  };
```



#### 6.10.4 调用堆栈

```less
WebRtcVoiceMediaChannel::WebRtcAudioSendStream::WebRtcAudioSendStream
Call::CreateAudioSendStream
AudioSendStream::AudioSendStream



WebRtcVoiceMediaChannel::WebRtcAudioSendStream::SetSend
												SetSource
                                                ClearSource
                                                OnClose
                                                SetRtpParameters
                                                
WebRtcAudioSendStream::UpdateSendState
AudioSendStream::Start/Stop
AudioState::AddSendingStream/RemoveSendingStream
```





### 6.11 AudioState::RemoveSendingStream

audio/audio_state.cc

```cpp
void AudioState::RemoveSendingStream(webrtc::AudioSendStream* stream) {
  RTC_DCHECK(thread_checker_.IsCurrent());
  auto count = sending_streams_.erase(stream);
  RTC_DCHECK_EQ(1, count);
  UpdateAudioTransportWithSendingStreams();
 	// do not stop recording when stopAudio(), rtcengine needs to actively call stopRecording
  // if (sending_streams_.empty()) {
  //   config_.audio_device_module->StopRecording();
  // }
}
```



#### 6.11.1 --AudioState::UpdateAudioTransportWithSendingStreams

audio/audio_state.cc



### 6.12 AudioState::UpdateAudioTransportWithSendingStreams

audio/audio_state.cc

```cpp
void AudioState::UpdateAudioTransportWithSendingStreams() {
  RTC_DCHECK(thread_checker_.IsCurrent());
  // AudioSender, 子类webrtc::AudioSendStream
  std::vector<AudioSender*> audio_senders;
  int max_sample_rate_hz = 8000;
  size_t max_num_channels = 1;
  // std::map<webrtc::AudioSendStream*, StreamProperties> sending_streams_;
  // 计算所有的sending_streams_， 最大的max_sample_rate_hz，max_num_channels
  for (const auto& kv : sending_streams_) {
    audio_senders.push_back(kv.first);
    max_sample_rate_hz = std::max(max_sample_rate_hz, kv.second.sample_rate_hz);
    max_num_channels = std::max(max_num_channels, kv.second.num_channels);
  }
  audio_transport_.UpdateAudioSenders(std::move(audio_senders),
                                      max_sample_rate_hz, max_num_channels);
}
```



## `webrtc::Call` 使用 `AudioState` 将音频处理流水线的静态部分和动态部分连接起来。

## 参考

https://blog.csdn.net/tq08g2z/article/details/103684797
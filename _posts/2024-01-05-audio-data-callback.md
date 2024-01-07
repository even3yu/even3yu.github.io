---
layout: post
title: webrtc的采集和接收到混音后的音频数据回调到应用层做二次处理
date: 2024-01-05 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio webrtc
---


* content
{:toc}

---


## 0. 前言

如果要使用WebRTC封装SDK，就要把RTC的一些基础能力暴露出来，本地视频、远端音视频都可以在相应的track里添加Sink就能拿到，但本地音频回调就没那么容易了，需要修改RTC的源码才可以。 

在WebRTC的默认实现中会用AudioTransportImpl实例化AudioTransport 。如果想把设备控制权从WebRTC里拿出来可以自定义类继承至AudioDeviceModuleImpl，如果要把采集到的音频回调出来可以自定类继承至AudioTransportImpl。然后通过自定义的AudioDeviceModule 传递给应用层。



## 1. AudioState

如果想让它从子类中创建必须为类对象指针，所以将它改为`AudioTransportImpl`*类型，`AudioState`的构造， 创建`AudioTransportImpl`。AudioTransport 由外部来创建，就是继承了AudioDeviceModule创建。

audio/audio_state.h

```cpp
class {
...
  // Transports mixed audio from the mixer to the audio device and
  // recorded audio to the sending streams.
  //AudioTransportImpl audio_transport_;
  //==>
  AudioTransportImpl* audio_transport_;
  ...
}
```


audio/audio_state.cc

```cpp
AudioState::AudioState(const AudioState::Config& config)
    : config_(config),audio_transport_(nullptr) {
  process_thread_checker_.Detach();
  RTC_DCHECK(config_.audio_mixer);
  RTC_DCHECK(config_.audio_device_module);
  /////////////// 由外部来创建AudioTransportImpl， 默认不用内部创建的
  audio_transport_ = audio_device_module()->CreateAudioTransPort(
      config_.audio_mixer, config_.audio_processing.get());
  needRelease_ = audio_transport_ == nullptr;
  if (needRelease_) {
    audio_transport_ = new AudioTransportImpl(config_.audio_mixer,
                                              config_.audio_processing.get());
  }
   /////////////
  RTC_DCHECK(audio_transport_);
}


```



## 2. AudioDeviceModule

在`AudioDeviceModule`类中增加纯虚函数`CreateAudioTransPort`。
modules/audio_device/include/audio_device.h

```cpp
class AudioDeviceModule : public rtc::RefCountInterface, public Module {
 ...
 virtual AudioTransportImpl* CreateAudioTransPort(
      AudioMixer* mixer,
      AudioProcessing* audio_processing) = 0;
...
}
```


modules/audio_device/audio_device_impl.cc

```cpp
 AudioTransportImpl* AudioDeviceModuleImpl::CreateAudioTransPort(
      AudioMixer* mixer,
      AudioProcessing* audio_processing){
   		//在AudioDeviceModuleImpl的实现为返回nullptr，这样如果没有子类实现它，结果就和原来的没有区别。
      return nullptr;
 }
```



## 3. CustomizedAudioDeviceModule

```cpp
class CustomizedAudioDeviceModule : public webrtc::AudioDeviceModuleImpl{
...
   
  ///////////////////////////////////
   class CustomAudioTransportImpl : public AudioTransportImpl {
   public:
    CustomAudioTransportImpl(webrtc::AudioMixer* mixer, webrtc::AudioProcessing* audio_processing);
    virtual ~CustomAudioTransportImpl();
     
    int32_t RecordedDataIsAvailable(const void* audioSamples,
     const size_t nSamples,
     const size_t nBytesPerSample,
     const size_t nChannels,
     const uint32_t samplesPerSec,
     const uint32_t totalDelayMS,
     const int32_t clockDrift,
     const uint32_t currentMicLevel,
     const bool keyPressed,
     uint32_t& newMicLevel) override;
     
     int32_t NeedMorePlayData(const size_t nSamples,
                              const size_t nBytesPerSample,
                              const size_t nChannels,
                              const uint32_t samples_per_sec,
                              void* audioSamples,
                              size_t& nSamplesOut,
                              int64_t* elapsed_time_ms,
                              int64_t* ntp_time_ms) override;
     
    void RegisterObserver(RecordedDataObserver* dataObserver);
   private:
    RecordedDataObserver* observer_;
   };
  ///////////////////////////////////
   
   AudioTransportImpl* CreateAudioTransPort(
    webrtc::AudioMixer* mixer,
    webrtc::AudioProcessing* audio_processing) override;
   bool IsExternalAudioInput() const override;
   void SetExternalAudioMode(bool bExternal);
   void RegisterObserver(RecordedDataObserver* dataObserver);
  private:
   std::unique_ptr<CustomAudioTransportImpl> customAudioTransport_;
   bool externalAudio_;
   AudioDeviceDataObserver* dataObserver_;
};
```



```cpp
webrtc::AudioTransportImpl* base::CustomizedAudioDeviceModule::CreateAudioTransPort(
 webrtc::AudioMixer* mixer, 
 webrtc::AudioProcessing* audio_processing) {
  // 创建 CustomAudioTransportImpl
  customAudioTransport_ = std::make_unique<CustomAudioTransportImpl>(mixer, audio_processing);
   if (dataObserver_) {
    // 注册监听，record，play数据可以导出到应用层
    customAudioTransport_->RegisterObserver(dataObserver_);
   }
   return customAudioTransport_.get();
 }

bool base::CustomizedAudioDeviceModule::IsExternalAudioInput() const {
	return externalAudio_;
}
void base::CustomizedAudioDeviceModule::SetExternalAudioMode(bool bExternal) {
	externalAudio_ = bExternal;
}

void base::CustomizedAudioDeviceModule::RegisterObserver(AudioDeviceDataObserver* dataObserver) {
//要在CreatePeerConnectionFactory后调用
 RTC_CHECK(dataObserver);
 RTC_CHECK(customAudioTransport_);
 dataObserver_ = dataObserver;
 customAudioTransport_->RegisterObserver(dataObserver);
}

```



### 3.1 CustomizedAudioDeviceModule::CustomAudioTransportImpl

```cpp
//////////////////////////////////////////////////////
//////////////////CustomAudioTransportImpl ///////////
//////////////////////////////////////////////////////
base::CustomizedAudioDeviceModule::CustomAudioTransportImpl::CustomAudioTransportImpl(
 webrtc::AudioMixer* mixer,
 webrtc::AudioProcessing* audio_processing) : 
 AudioTransportImpl(mixer,audio_processing), 
 observer_(nullptr) {
 }

int32_t CustomizedAudioDeviceModule::CustomAudioTransportImpl::RecordedDataIsAvailable( const void* audioSamples, 
 const size_t nSamples, 
 const size_t nBytesPerSample, 
 const size_t nChannels, 
 const uint32_t samplesPerSec, 
 const uint32_t totalDelayMS, 
 const int32_t clockDrift, 
 const uint32_t currentMicLevel, 
 const bool keyPressed, 
 uint32_t& newMicLevel) {
		// 这里把 录制的数据回调到应用层，可以做语音数据的特效处理
   if (observer_) {
    observer_->onRecodedData(audioSamples, nSamples, nBytesPerSample, nChannels, samplesPerSec);
   }

   return AudioTransportImpl::RecordedDataIsAvailable(
    audioSamples, 
    nSamples, 
    nBytesPerSample, 
    nChannels, 
    samplesPerSec, 
    totalDelayMS, 
    clockDrift, 
    currentMicLevel, 
    keyPressed, 
    newMicLevel);
 }

int32_t CustomizedAudioDeviceModule::CustomAudioTransportImpl::NeedMorePlayData(const size_t nSamples,
                         const size_t nBytesPerSample,
                         const size_t nChannels,
                         const uint32_t samples_per_sec,
                         void* audioSamples,
                         size_t& nSamplesOut,
                         int64_t* elapsed_time_ms,
                         int64_t* ntp_time_ms) {

   int32_t ret = AudioTransportImpl::RecordedDataIsAvailable(
    nSamples, nBytesPerSample, nChannels, samples_per_sec, audioSamples,
          nSamplesOut, elapsed_time_ms, ntp_time_ms);
	 // 这里把 播放的数据回调到应用层
   if (observer_) {
    observer_->OnRenderData(audioSamples, nSamples, nBytesPerSample, nChannels, samples_per_sec);
   }
  
   return ret;
}

void CustomizedAudioDeviceModule::CustomAudioTransportImpl::RegisterObserver(AudioDeviceDataObserver* dataObserver) {
	observer_ = dataObserver;
}
```



## 

### 3.2 webrtc::AudioDeviceDataObserver 

modules/audio_device/include/audio_device_data_observer.h

```cpp
// This interface will capture the raw PCM data of both the local captured as
// well as the mixed/rendered remote audio.
class AudioDeviceDataObserver {
 public:
  virtual void OnCaptureData(const void* audio_samples,
                             const size_t num_samples,
                             const size_t bytes_per_sample,
                             const size_t num_channels,
                             const uint32_t samples_per_sec) = 0;

  virtual void OnRenderData(const void* audio_samples,
                            const size_t num_samples,
                            const size_t bytes_per_sample,
                            const size_t num_channels,
                            const uint32_t samples_per_sec) = 0;

  AudioDeviceDataObserver() = default;
  virtual ~AudioDeviceDataObserver() = default;
};

```



## 参考

[WebRTC Native 视频会议实例教程（三） 启动WebRTC引擎， 获取本地与远程音频数据](https://blog.csdn.net/xqk/article/details/118758074)

[WebRTC本地音频回调、选用音频采集设备及自定义输入音频_webrtcaudiorecord 混合-CSDN博客](https://blog.csdn.net/weixin_39343678/article/details/99948451)
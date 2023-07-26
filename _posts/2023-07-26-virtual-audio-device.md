---
layout: post
title: webrtc 支持外部语音输入（虚拟设备）
date: 2023-07-26 15:54:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---

## 1. 前言 

从 WebRTC ADM 的实现来看，WebRTC 只实现对应了各个平台具体的硬件设备，并没什么虚拟设备。但是在实际的项目，往往需要支持外部音频输入/输出，就是由业务上层 push/pull 音频数据（PCM ...），而不是直接启动平台硬件进行采集/播放。在这种情况下，虽然原生的 WebRTC 不支持，但是要改造也是非常的简单，由于虚拟设备与平台无关，所以可以直接在 AudioDeviceModuleImpl 中增加一个与真实设备 audio_device_ 对应的Virtual Device（变量名暂定为`virtual_device_`），`virtual_device_`  也跟 `audio_device_ `一样，实现 AudioDeviceGeneric 相关接口，然后参考 audio_device_ 的实现去实现数据的“采集”（push）与 “播放”（pull），无须对接具体平台的硬件设备，唯一需要处理的就是物理设备 audio_device_ 与虚拟设备 virtual_device_ 之间的切换或协同工作。



## 2. 实现思路

1. 虚拟一个设备，实现音频数据的采集（当然这里指的是有外部数据的输入）。实现方式继承AudioDeviceGeneric， 实现自己的语音设备CustomizedAudioCapturer。启动了一个采集线程，每隔10ms 从CustomAudioFrameGenerator 获取一次数据；

2. 创建AudioDeviceBuffer，用于保存采集到的音频数据。AudioDeviceBuffer 的实例是由 CustomizedAudioDeviceModule 持有。

3. 定义CustomizedAudioDeviceModule，继承了webrtc::AudioDeviceModule。其实CustomizedAudioDeviceModule 内部，创建了默认的AudioDeviceModule，

    `webrtc::AudioDeviceModule internal_adm_ = webrtc::AudioDeviceModuleImpl::Create(AudioDeviceModule::kPlatformDefaultAudio, task_queue_factory_.get());`。

   为什么这么设计？

   - 这样就可以很方便的进行设备的切换，虚拟设备和真实设备之间的切换，这样增加一个变量`use_custom_device_`, 就可以很方便的切换。
   - 因为虚拟设备AudioDeviceGeneric ，知识实现record，没有实现playout，那么CustomizedAudioDeviceModule 就只要调用internal_adm_的 实现就可以解决了。

4.  创建 CustomAudioFrameGenerator，提供音频数据；

![virtal-audio-device]({{ site.url }}{{ site.baseurl }}/images/custom-audio-devcie.png)

## 3. 音频采集器——CustomizedAudioCapturer

实现音频的数据的采集，这里对播放不做实现。

```cpp
using namespace webrtc;

// This is a customized audio device which retrieves audio from a
// AudioFrameGenerator implementation as its microphone.
// CustomizedAudioCapturer is not able to output audio.
class CustomizedAudioCapturer : public AudioDeviceGeneric {
 public:
  // Constructs a customized audio device with |frame_generator|. It will read
  // audio from |frame_generator|.
  CustomizedAudioCapturer(
      std::shared_ptr<AudioFrameGeneratorInterface> frame_generator);
  virtual ~CustomizedAudioCapturer();
  // Retrieve the currently utilized audio layer
  int32_t ActiveAudioLayer(
      AudioDeviceModule::AudioLayer& audioLayer) const override;
  // Main initializaton and termination
	...
  // Device enumeration
  int16_t RecordingDevices() override;
  int32_t RecordingDeviceName(uint16_t index,
                              char name[kAdmMaxDeviceNameSize],
                              char guid[kAdmMaxGuidSize]) override;
  // Device selection
  int32_t SetRecordingDevice(uint16_t index) override;
  // Audio transport initialization
  int32_t RecordingIsAvailable(bool& available) override;
  int32_t InitRecording() override;
  bool RecordingIsInitialized() const override;
  // Audio transport control
  int32_t StartRecording() override;
  int32_t StopRecording() override;
  bool Recording() const override;
  // Audio mixer initialization
 /// int32_t InitMicrophone() override;
 /// bool MicrophoneIsInitialized() const override;
  // Speaker volume controls
  ...
  // Microphone volume controls
  int32_t MicrophoneVolumeIsAvailable(bool& available) override;
  int32_t SetMicrophoneVolume(uint32_t volume) override;
  int32_t MicrophoneVolume(uint32_t& volume) const override;
  ///int32_t MaxMicrophoneVolume(uint32_t& maxVolume) const override;
  ///int32_t MinMicrophoneVolume(uint32_t& minVolume) const override;
  // Speaker mute control
  ...
  // Microphone mute control
  ///int32_t MicrophoneMuteIsAvailable(bool& available) override;
  ///int32_t SetMicrophoneMute(bool enable) override;
  ///int32_t MicrophoneMute(bool& enabled) const override;

  // Stereo support
  int32_t StereoRecordingIsAvailable(bool& available) override;
  int32_t SetStereoRecording(bool enable) override;
  int32_t StereoRecording(bool& enabled) const override;
  // Delay information and control
  void AttachAudioBuffer(AudioDeviceBuffer* audioBuffer) override;

  int32_t sendCustomAudioData(RTCAudioFormat format, char* data, uint32_t length);
 private:
  static void RecThreadFunc(void*);
  bool RecThreadProcess();
  std::shared_ptr<AudioFrameGeneratorInterface> frame_generator_;
  AudioDeviceBuffer* audio_buffer_;
  // 存放从外部获取的音频数据
  std::unique_ptr<uint8_t[], webrtc::AlignedFreeDeleter>
      recording_buffer_;  // Pointer to a useable memory for audio frames.
  std::mutex mutex_;
  size_t recording_frames_in_10ms_;
  int recording_sample_rate_;
  int recording_channel_number_;
  size_t recording_buffer_size_;
  std::unique_ptr<rtc::PlatformThread> thread_rec_;
  bool recording_;
  uint64_t last_call_record_millis_;
  uint64_t last_thread_rec_end_time_;
  Clock* clock_;
  int64_t need_sleep_ms_;
  int64_t real_sleep_ms_;
  std::atomic<uint32_t> recording_volume_;
};
```



### CustomizedAudioCapturer::ActiveAudioLayer

```cpp
int32_t CustomizedAudioCapturer::ActiveAudioLayer(
    AudioDeviceModule::AudioLayer& audioLayer) const {
  return -1;
}
```

audioLayer 是在 AudioDeviceModuleImpl(modules/audio_device/audio_device_impl.h)创建的时候使用。 

- AudioDeviceModuleImpl::PlatformAudioLayer 返回的就是 AudioDeviceModuleImpl 传入的值。

- AudioDeviceModuleImpl::CreatePlatformSpecificObjects， 各个平台如果有多套实现方式，就根据audioLayer值进行创建。
  比如android，就有如下的实现方式。

  ```
  kAndroidJavaAudio,
  kAndroidOpenSLESAudio,
  kAndroidJavaInputAndOpenSLESOutputAudio,
  kAndroidAAudioAudio,
  kAndroidJavaInputAndAAudioOutputAudio,
  ```

- 不过这里

#### AudioLayer

```cpp
  enum AudioLayer {
    kPlatformDefaultAudio = 0,
    kWindowsCoreAudio,
    kWindowsCoreAudio2,
    kLinuxAlsaAudio,
    kLinuxPulseAudio,
    kAndroidJavaAudio,
    kAndroidOpenSLESAudio,
    kAndroidJavaInputAndOpenSLESOutputAudio,
    kAndroidAAudioAudio,
    kAndroidJavaInputAndAAudioOutputAudio,
    kDummyAudio,
  };
```



### CustomizedAudioCapturer::RecordingDevices

````cpp
int16_t CustomizedAudioCapturer::RecordingDevices() {
  return 1;
}
````

目前就支持一个设备。



### CustomizedAudioCapturer::RecordingDeviceName

设备名称，设备的唯一识别码。

```cpp
int32_t CustomizedAudioCapturer::RecordingDeviceName(
    uint16_t index,
    char name[kAdmMaxDeviceNameSize],
    char guid[kAdmMaxGuidSize]) {
  const char* kName = "customized_audio_device";
  const char* kGuid = "customized_audio_device_unique_id";
  if (index < 1) {
    memset(name, 0, kAdmMaxDeviceNameSize);
    memset(guid, 0, kAdmMaxGuidSize);
    memcpy(name, kName, strlen(kName));
    memcpy(guid, kGuid, strlen(kGuid));
    return 0;
  }
  return -1;
}
```



### CustomizedAudioCapturer::SetRecordingDevice

```cpp
int32_t CustomizedAudioCapturer::SetRecordingDevice(uint16_t index) {
  return 0;
}
```

目前就支持一个设备，所以默认就选择了这个设备，这里就是空的实现。



### !!!CustomizedAudioCapturer::InitRecording

采集的初始化。

```cpp
int32_t CustomizedAudioCapturer::InitRecording() {
  std::lock_guard<std::mutex> lock(mutex_);
  if (recording_) {
    return -1;
  }
  recording_sample_rate_ = frame_generator_->GetSampleRate();
  recording_channel_number_ = frame_generator_->GetChannelNumber();
  // 10ms 的samplereate = samplereate / 100， 就是10ms的采集次数
  recording_frames_in_10ms_ = static_cast<size_t>(recording_sample_rate_ / 100);
  // 2 就是bytesPerSample
  recording_buffer_size_ =
      recording_frames_in_10ms_ * recording_channel_number_ * 2;
  recording_buffer_.reset(static_cast<uint8_t*>(webrtc::AlignedMalloc<uint8_t>(
      recording_buffer_size_ * sizeof(uint8_t), 16)));
  if (audio_buffer_) {
    audio_buffer_->SetRecordingChannels(frame_generator_->GetChannelNumber());
    audio_buffer_->SetRecordingSampleRate(frame_generator_->GetSampleRate());
  }
  return 0;
}
```

1. recording_sample_rate_  ，采样率， 16k，48k等， 1s的采集次数

2. recording_channel_number_ ，声道，单声道，双声道等；

3. recording_frames_in_10ms_ ，10ms 内采集次数

   ```
     // 10ms 的samplereate = samplereate / 100， 就是10ms的采集次数
     recording_frames_in_10ms_ = static_cast<size_t>(recording_sample_rate_ / 100);
   ```

4. recording_buffer_size_，就一次性读取录制数据大小

   ```cpp
   // 2 就是bytesPerSample
   recording_buffer_size_ =
         recording_frames_in_10ms_ * recording_channel_number_ * 2;
   ```

5. 

### CustomizedAudioCapturer::RecordingIsInitialized

采集状态的判断。通过`recording_frames_in_10ms_`来判断，当然可以增加个bool变量来判断也是可以的。`recording_frames_in_10ms_`是在`CustomizedAudioCapturer::InitRecording`赋值的。

```cpp
bool CustomizedAudioCapturer::RecordingIsInitialized() const {
  if (recording_frames_in_10ms_ != 0)
    return true;
  else
    return false;
}
```



### CustomizedAudioCapturer::StartRecording

```cpp
int32_t CustomizedAudioCapturer::StartRecording() {
  recording_ = true;
  const char* thread_name = "custom_audio_capture_thread";
  thread_rec_.reset(new rtc::PlatformThread(RecThreadFunc, this, thread_name, rtc::kRealtimePriority));
  thread_rec_->Start();
  return 0;
}
```

开始录制，启动custom_audio_capture_thread 线程。

- `std::unique_ptr<rtc::PlatformThread> thread_rec_` 采集线程，优先级`rtc::kRealtimePriority`。
- 启动线程



#### rtc::PlatformThread

src/rtc_base/platform_thread.cc
PlatformThread 其实是一个对象，不是继承于thread，内部封装了不同平台线程的实现。

```cpp
// Represents a simple worker thread.  The implementation must be assumed
// to be single threaded, meaning that all methods of the class, must be
// called from the same thread, including instantiation.
class PlatformThread {
 public:
  PlatformThread(ThreadRunFunction func,
                 void* obj,
                 absl::string_view thread_name,
                 ThreadPriority priority = kNormalPriority);
  virtual ~PlatformThread();

  const std::string& name() const { return name_; }

  // Spawns a thread and tries to set thread priority according to the priority
  // from when CreateThread was called.
  void Start();

  bool IsRunning() const;

  // Returns an identifier for the worker thread that can be used to do
  // thread checks.
  PlatformThreadRef GetThreadRef() const;

  // Stops (joins) the spawned thread.
  void Stop();

 protected:
#if defined(WEBRTC_WIN)
  // Exposed to derived classes to allow for special cases specific to Windows.
  bool QueueAPC(PAPCFUNC apc_function, ULONG_PTR data);
#endif

 private:
  void Run();
  bool SetPriority(ThreadPriority priority);

  ThreadRunFunction const run_function_ = nullptr;
  const ThreadPriority priority_ = kNormalPriority;
  void* const obj_;
  // TODO(pbos): Make sure call sites use string literals and update to a const
  // char* instead of a std::string.
  const std::string name_;
  rtc::ThreadChecker thread_checker_;
  rtc::ThreadChecker spawned_thread_checker_;
#if defined(WEBRTC_WIN)
  static DWORD WINAPI StartThread(void* param);

  HANDLE thread_ = nullptr;
  DWORD thread_id_ = 0;
#else
  static void* StartThread(void* param);

  pthread_t thread_ = 0;
#endif  // defined(WEBRTC_WIN)
  RTC_DISALLOW_COPY_AND_ASSIGN(PlatformThread);
};
```





#### !!! RecThreadFunc

```cpp
void CustomizedAudioCapturer::RecThreadFunc(void* pThis) {
  static_cast<CustomizedAudioCapturer*>(pThis)->RecThreadProcess();
}

bool CustomizedAudioCapturer::RecThreadProcess() {
  while (recording_) {
    uint64_t current_time = clock_->CurrentNtpInMilliseconds();
    // 循环消耗的时间 = 就是本次开始录制的时间-上次录制结束的时间 
    uint64_t loop_cost_ms = 0;
    mutex_.lock();
    if (last_thread_rec_end_time_ > 0) {
        loop_cost_ms = current_time - last_thread_rec_end_time_;
    }
    if (last_call_record_millis_ == 0 ||
        (int64_t)(current_time - last_call_record_millis_) >= need_sleep_ms_) {
      // 获取数据保存到recording_buffer_
			// 每次读取的数据recording_buffer_size_， 具体参考CustomizedAudioCapturer::InitRecording
      if (frame_generator_->GenerateFramesForNext10Ms(
              recording_buffer_.get(),
              static_cast<uint32_t>(recording_buffer_size_)) !=
          static_cast<uint32_t>(recording_buffer_size_)) {
          mutex_.unlock();
        // 读取数据长度不对
        SleepMs(1);
        LOG( "Get audio frames failed." );
        last_thread_rec_end_time_ = clock_->CurrentNtpInMilliseconds();
        continue;
      }
      if (0 == recording_volume_) {
          memset(recording_buffer_.get(), 0, recording_buffer_size_);
      }
      // Sample rate and channel number cannot be changed on the fly.
      // 把recording_buffer_数据设置到AudioBuffer，
      audio_buffer_->SetRecordedBuffer(
          recording_buffer_.get(), recording_frames_in_10ms_);  // Buffer copied here
      last_call_record_millis_ = current_time;
      mutex_.unlock();
      /// ？？？？
      audio_buffer_->DeliverRecordedData();
      mutex_.lock();
    }
    mutex_.unlock();
    // 本次采集消耗的时间
    int64_t cost_ms = clock_->CurrentNtpInMilliseconds() - current_time;
    // 第一次 
		// 上次need_sleep_ms_ = 0；上次real_sleep_ms_ = 0；上次loop_cost_ms = 0；
    // 第一次need_sleep_ms_ = 10 - 本次采集消耗的时间cost_ms 
    //											- (上次need_sleep_ms_ - 上次real_sleep_ms_) - 上次loop_cost_ms
    // 非第一次
		// 上次need_sleep_ms_；上次real_sleep_ms_；上次loop_cost_ms；
    // 第一次need_sleep_ms_ = 10 - 本次采集消耗的时间cost_ms 
    //											- (上次need_sleep_ms_ - 上次real_sleep_ms_) - 上次loop_cost_ms
    need_sleep_ms_ = 10 - cost_ms + need_sleep_ms_ - real_sleep_ms_ - loop_cost_ms;
    if (need_sleep_ms_ > 0) {
      current_time = clock_->CurrentNtpInMilliseconds();
      SleepMs(need_sleep_ms_);
      // 
      real_sleep_ms_ = clock_->CurrentNtpInMilliseconds() - current_time;
    } else {
      LOG( "Cost too much time to get audio frames. This may "
                         "leads to large latency" );
      real_sleep_ms_ = 0;
    }
    // 采集结束的ntp
    last_thread_rec_end_time_ = clock_->CurrentNtpInMilliseconds();
    continue;
  }
  return true;
}
```

1. 每次循环采集的时间间隔为10ms

2. 注意：下面计算的都是ntp时间

3. 计算本次采集所消耗的时间 cost_ms = clock_->CurrentNtpInMilliseconds() - current_time;
   其中current_time 是在本次开始采集的时候的时间

4. 计算需要休眠的时间need_sleep_ms_ = 10ms - 本次采集消耗的时间cost_ms  - 上次循环休眠误差时间 - 上次loop_cost_ms
   其中上次循环休眠误差时间 = 上次need_sleep_ms_ - 上次real_sleep_ms_；

5. 真实休眠时间 real_sleep_ms_ = clock_->CurrentNtpInMilliseconds() - current_time;
   其中current_time 是在开始进入休眠前的时间

6. 循环消耗的时间 loop_cost_ms = 本次current_time - last_thread_rec_end_time_; 

   last_thread_rec_end_time_上次整体循环结束的结束时间



### CustomizedAudioCapturer::StopRecording

```cpp
int32_t CustomizedAudioCapturer::StopRecording() {
  // 加锁，在采集线程进行状态判断
  {
    std::lock_guard<std::mutex> lock(mutex_);
    recording_ = false;
  }
  if (thread_rec_) {
    thread_rec_->Stop();
    thread_rec_.reset();
  }

  last_thread_rec_end_time_ = 0;
  need_sleep_ms_ = 0;
  real_sleep_ms_ = 0;
  last_call_record_millis_ = 0;
  return 0;
}
```

停止录制。停止线程，修改状态。



### CustomizedAudioCapturer::Recording

```cpp
bool CustomizedAudioCapturer::Recording() const {
  return recording_;
}
```

录制状态。



### 关于MicrophoneVolume

```cpp
int32_t CustomizedAudioCapturer::MicrophoneVolumeIsAvailable(bool& available) {
  available = true;
  return 0;
}
int32_t CustomizedAudioCapturer::SetMicrophoneVolume(uint32_t volume) {
  recording_volume_ = volume;
  return 0;
}
int32_t CustomizedAudioCapturer::MicrophoneVolume(uint32_t& volume) const {
  volume = recording_volume_;
  return 0;
}
```



### !!!CustomizedAudioCapturer::AttachAudioBuffer

```cpp
void CustomizedAudioCapturer::AttachAudioBuffer(
    AudioDeviceBuffer* audioBuffer) {
  std::lock_guard<std::mutex> lock(mutex_);
  audio_buffer_ = audioBuffer;
  // Inform the AudioBuffer about default settings for this implementation.
  // Set all values to zero here since the actual settings will be done by
  // InitPlayout and InitRecording later.
  audio_buffer_->SetRecordingSampleRate(0);
  audio_buffer_->SetPlayoutSampleRate(0);
  audio_buffer_->SetRecordingChannels(0);
  audio_buffer_->SetPlayoutChannels(0);
}
```

1. `AudioDeviceBuffer* audio_buffer_;`



## 4. ADM——CustomizedAudioDeviceModule

```cpp
/**
 @brief CustomizedADM is able to create customized audio device use customized
 audio input.
 @details CustomizedADM does not support audio output yet.
 */
class CustomizedAudioDeviceModule : public webrtc::AudioDeviceModule {
 public:
  CustomizedAudioDeviceModule();
  virtual ~CustomizedAudioDeviceModule();

  int32_t CreateCustomizedAudioDevice(
      std::shared_ptr<AudioFrameGeneratorInterface> frame_generator);

  int64_t TimeUntilNextProcess() override;
  void Process() override;

...

  // Factory methods (resource allocation/deallocation)
  static rtc::scoped_refptr<AudioDeviceModule> Create(
      std::shared_ptr<AudioFrameGeneratorInterface> frame_generator,
      const char name[kAdmMaxDeviceNameSize], const char guid[kAdmMaxGuidSize]);
  // Retrieve the currently utilized audio layer
  int32_t ActiveAudioLayer(AudioLayer* audioLayer) const override;
  // Full-duplex transportation of PCM audio
  int32_t RegisterAudioCallback(AudioTransport* audioCallback) override;
  AudioTransport* GetAudioTransport() const override;
  // Main initialization and termination
  int32_t Init() override;
  int32_t Terminate() override;
  bool Initialized() const override;
  // Device enumeration
  ...
  int16_t RecordingDevices() override;
  int32_t RecordingDeviceName(uint16_t index,
                              char name[kAdmMaxDeviceNameSize],
                              char guid[kAdmMaxGuidSize]) override;
  // Device selection
  ...
  int32_t SetRecordingDevice(uint16_t index) override;
  int32_t SetRecordingDevice(WindowsDeviceType device) override;
  
  bool SetLocalAudioShare(bool enable) override;
  // Audio transport initialization
  ...
  int32_t RecordingIsAvailable(bool* available) override;
  int32_t InitRecording() override;
  bool RecordingIsInitialized() const override;
  // Audio transport control
	...
  int32_t StartRecording() override;
  int32_t StopRecording() override;
  bool Recording() const override;
  // Audio mixer initialization
	...
  int32_t InitMicrophone() override;
  bool MicrophoneIsInitialized() const override;
  // Speaker volume controls
  ...
  // Microphone volume controls
  int32_t MicrophoneVolumeIsAvailable(bool* available) override;
  int32_t SetMicrophoneVolume(uint32_t volume) override;
  int32_t MicrophoneVolume(uint32_t* volume) const override;
  int32_t MaxMicrophoneVolume(uint32_t* maxVolume) const override;
  int32_t MinMicrophoneVolume(uint32_t* minVolume) const override;
  // Speaker mute control
	...
  // Microphone mute control
  int32_t MicrophoneMuteIsAvailable(bool* available) override;
  int32_t SetMicrophoneMute(bool enable) override;
  int32_t MicrophoneMute(bool* enabled) const override;
  // Stereo support
  ...
  int32_t StereoRecordingIsAvailable(bool* available) const override;
  int32_t SetStereoRecording(bool enable) override;
  int32_t StereoRecording(bool* enabled) const override;
  // Delay information and control
	...

  // Only supported on Android.
 	...
  // Enables the built-in audio effects. Only supported on Android.
  ...
  //int32_t sendCustomAudioData(RTCAudioFormat format, char* data, uint32_t length);
// Only supported on iOS.
#if defined(WEBRTC_IOS)
  ...
  int GetRecordAudioParameters(AudioParameters* params) const override;
#endif  // WEBRTC_IOS
 private:
  int32_t AttachAudioBuffer();
  void CreateDefaultAdm();
  bool CustomizedAudioAvailable();
  std::mutex mutex_;
  std::unique_ptr<webrtc::TaskQueueFactory> task_queue_factory_;
  std::unique_ptr<AudioDeviceGeneric> custom_audio_device_;
  AudioDeviceBuffer* _ptrAudioDeviceBuffer;
  int64_t _lastProcessTime;
  bool _initialized;
  // Default internal adm for playout.
  rtc::scoped_refptr<webrtc::AudioDeviceModule> internal_adm_;

  bool use_custom_device_;
};
```

1. 定义CustomizedAudioDeviceModule，继承了webrtc::AudioDeviceModule。其实CustomizedAudioDeviceModule 内部，创建了默认的AudioDeviceModule，

    `webrtc::AudioDeviceModule internal_adm_ = webrtc::AudioDeviceModuleImpl::Create(AudioDeviceModule::kPlatformDefaultAudio, task_queue_factory_.get());`。

2. 

### 4.1 Create——创建CustomizedAudioDeviceModule

```cpp
rtc::scoped_refptr<AudioDeviceModule> CustomizedAudioDeviceModule::Create(
    std::shared_ptr<AudioFrameGeneratorInterface> frame_generator) {
  // Create the generic ref counted implementation.
  rtc::scoped_refptr<CustomizedAudioDeviceModule> audioDevice(
      new rtc::RefCountedObject<CustomizedAudioDeviceModule>());
  // Create the customized implementation.
  if (audioDevice->CreateCustomizedAudioDevice(frame_generator) ==
      -1) {
    return nullptr;
  }
  //// Ensure that the generic audio buffer can communicate with the
  //// platform-specific parts.
  if (audioDevice->AttachAudioBuffer() == -1) {
    return nullptr;
  }
  return audioDevice;
}
```

1. 创建CustomizedAudioDeviceModule 对象，内部创建了AudioDeviceBuffer和默认AudioDeviceModuleImpl internal_adm_；
2. 创建虚拟设备，就是CustomizedAudioCapturer对象
3. AttachAudioBuffer



#### CustomizedAudioDeviceModule::CustomizedAudioDeviceModule

```cpp
CustomizedAudioDeviceModule::CustomizedAudioDeviceModule()
    : task_queue_factory_(webrtc::CreateDefaultTaskQueueFactory()),
      _ptrAudioDeviceBuffer(
          new webrtc::AudioDeviceBuffer(task_queue_factory_.get())),
      _lastProcessTime(rtc::TimeMillis()),
      _initialized(false){
  CreateDefaultAdm();
  use_custom_device_ = false;
}
```

1. 创建了_ptrAudioDeviceBuffer =  webrtc::AudioDeviceBuffer

2. CreateDefaultAdm

   ```cpp
   void CustomizedAudioDeviceModule::CreateDefaultAdm(){
     if (internal_adm_ == nullptr) {
   #if defined(WEBRTC_INCLUDE_INTERNAL_AUDIO_DEVICE)
       // audioLayer = AudioDeviceModule::kPlatformDefaultAudio
       internal_adm_ = webrtc::AudioDeviceModuleImpl::Create(
           AudioDeviceModule::kPlatformDefaultAudio, task_queue_factory_.get());
   #else
       internal_adm_ = new rtc::RefCountedObject<webrtc::FakeAudioDeviceModule>();
   #endif
     }
   }
   ```

3. use_custom_device_ 虚拟设备和真实设备之间切换。



#### CustomizedAudioDeviceModule::CreateCustomizedAudioDevice

```cpp
int32_t CustomizedAudioDeviceModule::CreateCustomizedAudioDevice(
    std::shared_ptr<AudioFrameGeneratorInterface> frame_generator) {
  // 创建 CustomizedAudioCapturer
  custom_audio_device_.reset(
      new CustomizedAudioCapturer(frame_generator));
  return 0;
}
```

1. 创建 CustomizedAudioCapturer



#### CustomizedAudioDeviceModule::AttachAudioBuffer

```cpp
//  AttachAudioBuffer
//
//  Install "bridge" between the platform implementation and the generic
//  implementation. The "child" shall set the native sampling rate and the
//  number of channels in this function call.
// ----------------------------------------------------------------------------
int32_t CustomizedAudioDeviceModule::AttachAudioBuffer() {
  // AudioDeviceBuffer _ptrAudioDeviceBuffer
  // audioDeviceBuffer 设置给虚拟设备
  if (custom_audio_device_)
    custom_audio_device_->AttachAudioBuffer(_ptrAudioDeviceBuffer);
  return 0;
}
```



### --------------

### 4.2 CustomizedAudioDeviceModule::Init

```cpp
int32_t CustomizedAudioDeviceModule::Init() {
  if (_initialized)
    return 0;
  if (custom_audio_device_  && custom_audio_device_->Init() !=
      AudioDeviceGeneric::InitStatus::OK) {
    return -1;
  }
  if (!internal_adm_ || internal_adm_->Init() == -1)
    return -1;
  _initialized = true;
  return 0;
}
```

1. 初始化 custom audio device， 其实内部什么也没做。
2. 初始化 internal adm。
3. 变量_initialized 用来判断是否初始化。

### 4.3 CustomizedAudioDeviceModule::Terminate

```cpp
int32_t CustomizedAudioDeviceModule::Terminate() {
  if (!_initialized)
    return 0;
  if (custom_audio_device_ && custom_audio_device_->Terminate() == -1) {
    return -1;
  }
  if (!internal_adm_ || internal_adm_->Terminate() == -1)
    return -1;
  _initialized = false;
  return 0;
}
```

1. 销毁  custom audio device。
2. 销毁 internal adm。



### 4.4 CustomizedAudioDeviceModule::Initialized

```cpp
bool CustomizedAudioDeviceModule::Initialized() const {
  return (_initialized && internal_adm_->Initialized());
}
```



### -----------

### 4.5 关于麦克风的方法

#### CustomizedAudioDeviceModule::RecordingDevices

```cpp
int16_t CustomizedAudioDeviceModule::RecordingDevices() {
  CHECK_INITIALIZED();
  auto count = internal_adm_->RecordingDevices();
  if (custom_audio_device_) {
    count += custom_audio_device_->RecordingDevices();
  }
  return count;
}
```

1. 创建了虚拟设备 custom_audio_device_ ，则录制设备的设备数目添加



#### CustomizedAudioDeviceModule::SetRecordingDevice

```cpp
int32_t CustomizedAudioDeviceModule::SetRecordingDevice(uint16_t index) {
  CHECK_INITIALIZED();
  RTC_DCHECK(!this->Recording());
  auto count = internal_adm_->RecordingDevices();
  if (index < count) {
    use_custom_device_ = false;
    return internal_adm_->SetRecordingDevice(index);
  }
  return -1;
  // 虚拟设备没创建，则不能选择
  if (!custom_audio_device_)
      return -1;
  // 启动虚拟设备
  use_custom_device_ = true;
  return custom_audio_device_->SetRecordingDevice(index - count);
}
```

1. 选择了虚拟设备

#### CustomizedAudioDeviceModule::RecordingDeviceName

```cpp
int32_t CustomizedAudioDeviceModule::RecordingDeviceName(
    uint16_t index,
    char name[kAdmMaxDeviceNameSize],
    char guid[kAdmMaxGuidSize]) {
  CHECK_INITIALIZED();
  if (name == NULL) {
    return -1;
  }
  auto count = internal_adm_->RecordingDevices();
  if (index < count) {
    return internal_adm_->RecordingDeviceName(index, name, guid);
  }
  return -1;
  if (!custom_audio_device_) {
    return -1;
  }
  
  return custom_audio_device_->RecordingDeviceName(index - count, name, guid);
}
```



#### !!!CustomizedAudioDeviceModule::StartRecording

```cpp
int32_t CustomizedAudioDeviceModule::StartRecording() {
  CHECK_INITIALIZED();
  if (use_custom_device_) {
    _ptrAudioDeviceBuffer->StartRecording();
    return custom_audio_device_->StartRecording();
  }
  if (!internal_adm_->RecordingIsInitialized()) {
    internal_adm_->InitRecording();
  }
  return internal_adm_->StartRecording();
}
```

1. 虚拟设备开始录制，则先调用AudioDeviceBuffer 的StartRecording，然后调用虚拟设备CustomizedAudioCapturer的StartRecording

#### !!!CustomizedAudioDeviceModule::StopRecording

```cpp
int32_t CustomizedAudioDeviceModule::StopRecording() {
  CHECK_INITIALIZED();
  if (use_custom_device_) {
    _ptrAudioDeviceBuffer->StopRecording();
    return custom_audio_device_->StopRecording();
  }

  return internal_adm_->StopRecording();
}
```



#### !!!CustomizedAudioDeviceModule::SetStereoRecording

支持立体声，双声道支持。

```cpp
int32_t CustomizedAudioDeviceModule::SetStereoRecording(bool enable) {
  CHECK_INITIALIZED();
  if (use_custom_device_) {
    if (custom_audio_device_->RecordingIsInitialized()) {
      return -1;
    }
    if (custom_audio_device_->SetStereoRecording(enable) == -1) {
      return -1;
    }
    int8_t nChannels(1);
    if (enable) {
      nChannels = 2;
    }
    return _ptrAudioDeviceBuffer->SetRecordingChannels(nChannels);
  }
  
  return internal_adm_->SetStereoRecording(enable);
}
```



#### 其他

- 先判断是否启用虚拟设备，use_custom_device_；
- 如果是调用虚拟设备的对应的方法
- 如果否调用internal adm对应的方法
- 举个例子：

```cpp
bool CustomizedAudioDeviceModule::MicrophoneIsInitialized() const {
  CHECK_INITIALIZED_BOOL();
  if (use_custom_device_) {
    return custom_audio_device_->MicrophoneIsInitialized();
  }

  return internal_adm_->MicrophoneIsInitialized();
}
```



### 4.6 关于扬声器的方法

直接调用internal adm 的方法。举个例子，

```cpp
int32_t CustomizedAudioDeviceModule::SpeakerVolumeIsAvailable(bool* available) {
  return internal_adm_->SpeakerVolumeIsAvailable(available);
}
```



## 5. CustomAudioFrameGenerator

从文件中读取pcm数据。

```cpp

class CustomAudioFrameGenerator : public AudioFrameGeneratorInterface {
public:
    explicit CustomAudioFrameGenerator(std::string path, int channelNumber, int sampleRate);
    virtual ~CustomAudioFrameGenerator();
    virtual uint32_t GenerateFramesForNext10Ms(uint8_t* frame_buffer, const uint32_t capacity);
    virtual int GetSampleRate() override;
    virtual int GetChannelNumber() override;
    std::string path_;

private:
    FILE * fd;
    int channelNumber_;
    int sampleRate_;
    int bytesPerSample_;
    int framesize_;
    int framesForNext10Ms_;
};


CustomAudioFrameGenerator::CustomAudioFrameGenerator(std::string path, int channelNumber, int sampleRate)
{
    path_ = path;
  	// 声道
    channelNumber_ = channelNumber;
  	// 采样率
    sampleRate_ = sampleRate;
  	// 采集的每个包的大小，16bit
    bytesPerSample_ = 2;
    framesize_ = channelNumber_*bytesPerSample_*sampleRate_;
    framesForNext10Ms_ = framesize_ * 10 / 1000;
  
    fopen_s(&fd, path_.c_str(), "rb");
    if (!fd) {
        std::cout << "Failed to open the " << path_.c_str() << std::endl;
    }
    else {
        std::cout << "Successfully open the " << path_.c_str() << std::endl;
    }
}

CustomAudioFrameGenerator::~CustomAudioFrameGenerator()
{
    fclose(fd);
}

int CustomAudioFrameGenerator::GetSampleRate() {
    return sampleRate_;
}

int CustomAudioFrameGenerator::GetChannelNumber() {
    return channelNumber_;
}

uint32_t CustomAudioFrameGenerator::GenerateFramesForNext10Ms(uint8_t* frame_buffer, const uint32_t capacity)
{
    if (frame_buffer == nullptr) {
        std::cout << "Invaild buffer for frame generator." << std::endl;
        return 0;
    }
    if (framesForNext10Ms_ <= static_cast<int>(capacity)) {
        if (fread(frame_buffer, 1, framesForNext10Ms_, fd) != framesForNext10Ms_)
            fseek(fd, 0, SEEK_SET);
        fread(frame_buffer, 1, framesForNext10Ms_, fd);
    }
    return framesForNext10Ms_;
}

```



#### AudioFrameGeneratorInterface

```cpp
/**
 @brief frame generator interface for audio
 @details Sample rate and channel numbers cannot be changed once the generator is
 created. Currently, only 16 bit little-endian PCM is supported.
*/
class AudioFrameGeneratorInterface {
 public:
  /**
   @brief Generate frames for next 10ms.
   @param buffer Points to the start address for frame data. The memory is
   allocated and owned by SDK. Implementations should fill frame data to the
   memory starts from |buffer|.
   @param capacity Buffer's capacity. It will be equal or greater to expected
   frame buffer size.
   @return The size of actually frame buffer size.
   */
  virtual uint32_t GenerateFramesForNext10Ms(uint8_t* buffer,
                                             const uint32_t capacity) = 0;
  /// Get sample rate for frames generated.
  virtual int GetSampleRate() = 0;
  /// Get numbers of channel for frames generated.
  virtual int GetChannelNumber() = 0;


  virtual ~AudioFrameGeneratorInterface(){}
};
```


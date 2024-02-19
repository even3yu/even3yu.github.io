---
layout: post
title: 音频总篇
date: 2024-02-18 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc audio
---


* content
{:toc}

---


## 1. 概览

![Audio Pipeline]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-state.png)

| 模块                                        | 说明                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| AudioDeviceModule                           | 用于控制各个操作系统平台的音频设备，主要用来做音频的采集和播放 |
| webrtc::AudioTransport                      | 是一个适配和胶水模块，它把 "AudioDeviceModule 的音频数据采集"和 "webrtc::AudioProcessing 的音频数据处理"及 "webrtc::AudioSender/webrtc::AudioSendStream 的音频数据编码和发送控制"粘起来，----------------------------------------------------------webrtc::AudioTransport 把采集的音频数据送给 webrtc::AudioProcessing 处理，之后再把处理后的数据给到 webrtc::AudioSender/webrtc::AudioSendStream 编码发送出去。 |
| webrtc::AudioProcessing                     | 用于做音频数据处理，如降噪、自动增益控制和回声消除等。       |
| webrtc::AudioSender/webrtc::AudioSendStream | 用于对音频数据做编码，比如 OPUS、AAC 等，RTP 打包和发送控制  |
| webrtc::Transport                           | 也是一个适配和胶水模块，它把 webrtc::AudioSender/webrtc::AudioSendStream 得到的 RTP 和 RTCP 包发送给后面的网络接口模块。 |
| cricket::MediaChannel::NetworkInterface     | 用于实现真正地把 RTP 和 RTCP 包通过底层的网络接口和协议发送，如 UDP 等，ICE 的作用即为创建一个工作良好的网络接口模块实例。 |



![img]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-send-stream)

这个图中的箭头表示数据流动的方向，数据在各个模块中处理的先后顺序为自左向右。

1. `webrtc::AudioSendStream` 的实现的数据处理流程中，输入数据为 PCM，来自于 `webrtc::AudioTransport`，输出则为 RTP 包，被送给 `webrtc::Transport` 发出去。

2. `webrtc::AudioSendStream` 的实现内部，数据首先会经过 **ACM 模块** 来做编码，随后经过编码的音频帧送进 **rtp_rtcp 模块** 打成 RTP 包，然后 RTP 包被送进 **pacing 模块** 做平滑发送控制，最后再在 **rtp_rtcp 模块** 中被给到`webrtc::Transport` 发出去。

站在 `webrtc::AudioSendStream` 的视角，基于抽象的模块及接口，搭建数据处理流水线是这个组件的机制设计，各个模块及接口的具体实现是机制下的策略。这里先来看一下，关于 `webrtc::AudioSendStream` 的实现的这个数据处理流水线的搭建过程。

| 模块                                        | 说明                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| webrtc::AudioTransport                      | 是一个适配和胶水模块，它把 "AudioDeviceModule 的音频数据采集"和 "webrtc::AudioProcessing 的音频数据处理"及 "webrtc::AudioSender/webrtc::AudioSendStream 的音频数据编码和发送控制"粘起来，----------------------------------------------------------webrtc::AudioTransport 把采集的音频数据送给 webrtc::AudioProcessing 处理，之后再把处理后的数据给到 webrtc::AudioSender/webrtc::AudioSendStream 编码发送出去。 |
| webrtc::internal::AudioSendStream           | 用于对音频数据做编码，比如 OPUS、AAC 等，RTP 打包和发送控制  |
| webrtc::voe::ChannelSendInterface           |                                                              |
| webrtc::AudioCodingModule                   | 语音编码模块                                                 |
| webrtc::AudioEncoder                        | 语音编码器                                                   |
| webrtc::RTPSenderAudio                      |                                                              |
| webrtc::RTPSender                           |                                                              |
| webrtc::RtpPacketSender                     |                                                              |
| webrtc::PacedSender                         |                                                              |
| webrtc::PacingController                    |                                                              |
| webrtc::PacketRouter                        |                                                              |
| webrtc::PacingController::PacketSender      |                                                              |
| webrtc::RtpTransportControllerSendInterface |                                                              |
| webrtc::RtpSenderEgress                     |                                                              |
| webrtc::RtpRtcpInterface                    |                                                              |
| webrtc::ModuleRtpRtcpImpl2                  |                                                              |
| webrtc::Transport                           | 也是一个适配和胶水模块，它把 webrtc::AudioSender/webrtc::AudioSendStream 得到的 RTP 和 RTCP 包发送给后面的网络接口模块。 |
|                                             |                                                              |



## 2. 各大模块的关系

```less
-----------------------------     --------------------------     ---------------------------
|                           |     |                        | ==> | webrtc::AudioProcessing |
| webrtc::AudioDeviceModule | ==> | webrtc::AudioTransport |     ---------------------------
|                           |     |                        |          ｜｜
-----------------------------     --------------------------          ｜｜
                                                                      ｜｜
                                          +=+=========================+=+
                                          | |
                                          \ /
                                           |
                    -----------------------------------------------     ---------------------
                    | webrtc::AudioSender/webrtc::AudioSendStream | ==> | webrtc::Transport |
                    -----------------------------------------------     ---------------------
                                                                                ｜｜
                                                                                \ /
                                                 -------------------------------------------
                                                 | cricket::MediaChannel::NetworkInterface |
                                                 -------------------------------------------
```

- 调用AudioDeviceModule::RegistAudioCallback， 进而调用 AudioDeviceBuffer::RegistAudioCallback， 这样，采集的音频数据从 AudioDeviceModule 到 AudioTransport。
- `webrtc::AudioSendStream` 的实现的数据处理流程中，输入数据为 PCM，来自于 `webrtc::AudioTransport`，输出则为 RTP 包，被送给 `webrtc::Transport` 发出去。



去掉两个粘合剂，数据流转的节点

```less
-----------------------------     -----------------------------------------------   
|                           |     |                        						| 
| webrtc::AudioDeviceModule | ==> | webrtc::AudioSender/webrtc::AudioSendStream |     
|                           |     |                        						|         
-----------------------------     -----------------------------------------------   
 													    ｜｜
                                                        \ /
                                   -------------------------------------------
                                   | cricket::MediaChannel::NetworkInterface |
                                   -------------------------------------------

```



## 3. 数据流向

[audio pipeline (even3yu.github.io)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://even3yu.github.io/2023/12/18/audio-pipeline/)

[WebRTC 的音频数据编码及发送控制管线 - 简书 (jianshu.com)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://www.jianshu.com/p/64d40d7ca74a)

![audio]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio_transportimpl.jpeg)





## 4. 发送端调用堆栈

[audio pipeline (even3yu.github.io)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://even3yu.github.io/2023/12/18/audio-pipeline/#1-音频采集)

![img]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-send-pipeline.png)

上图是音频数据发送的核心流程，主要是核心函数的调用及线程的切换。PCM 数据从硬件设备中被采集出来，在采集线程做些简单的数据封装会很快进入 APM 模块做相应的 3A 处理，从流程上看 APM 模块很靠近原始 PCM 数据，这一点对 APM 的处理效果有非常大的帮助，感兴趣的同学可以深入研究下 APM 相关的知识。之后数据就会被封装成一个 Task，投递到一个叫 rtp_send_controller 的线程中，到此采集线程的工作就完成了，采集线程也能尽快开始下一轮数据的读取，这样能最大限度的减小对采集的影响，尽快读取新的 PCM 数据，防止 PCM 数据丢失或带来不必要的延时。

接着数据就到了 rtp_send_controller 线程，rtp_send_controller 线程的在此的作用主要有三个，一是做 rtp 发送的拥塞控制，二是做 PCM 数据的编码，三是将编码后的数据打包成 RtpPacketToSend（RtpPacket）格式。最终的 RtpPacket 数据会被投递到一个叫 RoundRobinPacketQueue 的队列中，至此 rtp_send_controller 线程的工作完成。

后面的 RtpPacket 数据将会在 SendControllerThread 中被处理，SendControllerThread 主要用于发送状态及窗口拥塞的控制，最后数据通过消息的形式（type: MSG_SEND_RTP_PACKET）发送到 Webrtc 三大线程之一的网络线程（Network Thread），再往后就是发送给网络。到此整个发送过程结束。



## 5. 接收端调用堆栈

![img]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-recv-pipeline.png)

上图是音频数据接收及播放的核心流程。网络线程（Network Thread）负责从网络接收 RTP 数据，随后异步给工作线程（Work Thread）进行解包及分发。如果接收多路音频，那么就有多个 ChannelReceive，每个的处理流程都一样，最后未解码的音频数据存放在 NetEq 模块的 packet_buffer_ 中。与此同时播放设备线程不断的从当前所有音频 ChannelReceive 获取音频数据（10ms 长度），进而触发 NetEq 请求解码器进行音频解码。对于音频解码，WebRTC 提供了统一的接口，具体的解码器只需要实现相应的接口即可，比如 WebRTC 默认的音频解码器 opus 就是如此。当遍历并解码完所有 ChannelReceive 中的数据，后面就是通过 AudioMixer 混音，混完后交给 APM 模块处理，处理完最后是给设备播放。



## 6. 模块

### 6.1 ??? AudioDeviceBuffer,FineAudioBuffer



#### 6.1.1 对象结构

![audio-device-buffer]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-device-buffer.png)



### 6.2 AudioDeviceGeneric

[webrtc 支持外部语音输入（虚拟设备） (even3yu.github.io)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://even3yu.github.io/2023/07/26/virtual-audio-device/)

[音频采集 (even3yu.github.io)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://even3yu.github.io/2023/12/19/audio-capture/)

![virtal-audio-device]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/custom-audio-devcie.png)



#### 6.2.1音频设备的启动和停止

![img]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/capture-call-stack.jpg)



![img]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/start-stop-device.png)



### 6.3 AudioDeviceModule

[AudioDeviceModule (even3yu.github.io)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://even3yu.github.io/2023/07/18/audiodevice/)

WebRTC的音频设备类主要包含AudioDeviceGeneric和AudioDeviceModule。

- AudioDeviceGeneric代表一个输入输出设备，提供数据的真正采集和播放，调整音量等; AudioDeviceGeneric是个抽象的，具体的是由各个平台的实现，AudioDeviceDummy，AudioDeviceWindowsCore，AudioDeviceTemplate，AudioDeviceLinuxALSA，AudioDeviceLinuxPulse，AudioDeviceIOS，AudioDeviceMac。
- AudioDeviceModule是音频设备模块，它实际上内部会有一个AudioDeviceGeneric设备的引用（成员），然后把设备相关的调用转到设备对象上。它主要的目的是WebRTC音频的抽象接口定义，并完成不同平台的设备对象的创建。



#### 类图

![audio-AuidoDevieModule-AudioDevice.drawio]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-AuidoDevieModule-AudioDevice.drawio.png)





![img]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/5c85b0fa2e57831467de36d674e701bb.png)



#### 6.3.1 创建时机

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



#### 6.3.2 对象结构

![audio-device-module]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-device-module.png)



### ------



### 6.4 AudioTransport

[AudioTransportImpl (even3yu.github.io)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://even3yu.github.io/2023/12/20/audio-transport-impl/)

https://www.jianshu.com/p/64d40d7ca74a

数据流转中心 AudioTransportImpl 实现了采集数据处理接口 RecordDataIsAvailbale和播放数据处理接口 NeedMorePlayData。RecordDataIsAvailbale 负责采集音频数据的处理和将其分发到所有的发送 Streams。NeedMorePlayData 负责混音所有接收到的 Streams，然后输送给 APM 作为一路参考信号处理，最后将其重采样到请求输出的采样率。



- `webrtc::AudioTransport` 是一个适配和胶水模块，它把 `AudioDeviceModule` 的音频数据采集和 `webrtc::AudioProcessing` 的音频数据处理及 `webrtc::AudioSender`/`webrtc::AudioSendStream` 的音频数据编码和发送控制粘起来，`webrtc::AudioTransport` 把采集的音频数据送给 `webrtc::AudioProcessing` 处理，之后再把处理后的数据给到 `webrtc::AudioSender`/`webrtc::AudioSendStream` 编码发送出去。



#### 6.4.1 创建时机

audio/audio_state.cc

```cpp
AudioState::AudioState(const AudioState::Config& config)
    : config_(config),
      audio_transport_(config_.audio_mixer,
                       config_.audio_processing.get(),
                       config_.async_audio_processing_factory.get()) {
  process_thread_checker_.Detach();
  RTC_DCHECK(config_.audio_mixer);
  RTC_DCHECK(config_.audio_device_module);
}
```

audio/audio_state.h

```
  AudioTransportImpl audio_transport_;
```



#### 6.4.2 对象组成

![audio-transport]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-transport.png)



### 6.5 ??? AudioProcessing

`webrtc::AudioProcessing` 用于做音频数据处理，如降噪、自动增益控制和回声消除等。



### 6.6 ??? AudioMixer





### ------

### 6.7 AudioState

[AudioState (even3yu.github.io)]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/https://even3yu.github.io/2023/12/22/audiostate)

WebRTC 音频的静态流水线节点，主要包括 `AudioDeviceModule`，`AudioProcessing`，和 `AudioMixer` 等，这些相关节点状态由 `AudioState` 维护和管理。`AudioState`既是 WebRTC 音频流水线节点的容器，也可用于对这些节点做一些管理和控制，以及将不同节点粘起来构成流水线。



#### 6.7.1 对象组成

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



![image]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audiostate.png)



#### 6.7.2 创建时机

WebRTC 音频静态流水线在 `WebRtcVoiceEngine::Init() `中建立。动态流水线和静态流水线在 AudioState 和 AudioTransport 处交汇。

- 根据需要创建 AudioDeviceModule，
- 然后创建 webrtc::AudioState，webrtc::AudioState 用注入的 AudioMixer 和 AudioProcessing 创建 AudioTransport，
- 随后将 webrtc::AudioState 创建的 AudioTransport 作为 AudioCallback 注册给 AudioDeviceModule。

media/engine/webrtc_voice_engine.cc

### ------

### 6.8 ??? webrtc::Call



### 6.9 ??? AudioSender/AudioSendStream

`webrtc::AudioSender`/`webrtc::AudioSendStream` 用于对音频数据做编码，比如 OPUS、AAC 等，RTP 打包和发送控制。

<img src="audio-introduction.assets/audio-send-stream-class" alt="img" style="zoom:50%;" />



`webrtc::AudioSendStream` 的设计与实现主要可以从两个角度来看，一是配置和控制，二是数据流。

1. 对于配置和控制，可以对 `webrtc::AudioSendStream` 执行的配置和控制主要有如下这些：

   - 配置音频编码的编码器及编码参数，如最大码率、最小码率、payload type、FEC 等；

   - 配置用于把 RTP 发送到网络的 `webrtc::Transport` 、加密参数及静音控制等；

   - `webrtc::AudioSendStream` 的生命周期控制，如启动停止；

   - 设置网络传输上的额外开销大小，及动态更新的分配码率。

2. 对于数据流，一是采集处理的音频数据被送进 `webrtc::AudioSendStream` 以做编码和发送处理；二是网络传回的 RTCP 包，以对编码发送过程产生影响。

传回的 RTCP 包和 `webrtc::AudioSendStream` 的控制接口，共同构成音频数据编码及发送控制过程的信息来源。

`webrtc::AudioSendStream` 的实现中，最主要的数据处理流程 —— 音频数据编码、发送过程，及相关模块如下图：

![img]({{ SITE.URL }}{{ SITE.BASEURL }}/IMAGES/audio-introduction.assets/audio-send-stream)





#### 把 `webrtc::AudioSendStream` 的实现加进 `webrtc::AudioTransport`

`webrtc::AudioSendStream` 的音频 PCM 数据来源于 `webrtc::AudioTransport`。`webrtc::AudioSendStream` 的实现被作为 `webrtc::AudioTransport` 的一个 `webrtc::AudioSender` 加进 `webrtc::AudioTransport`，在 `webrtc::AudioSendStream` 的生命周期函数 `Start()` 被调用时执行，这个添加的过程大体如下：



```cpp
#0  webrtc::AudioTransportImpl::UpdateAudioSenders(std::vector<webrtc::AudioSender*, std::allocator<webrtc::AudioSender*> >, int, unsigned long) ()
    at webrtc/audio/audio_transport_impl.cc:262
#1  webrtc::internal::AudioState::UpdateAudioTransportWithSendingStreams() () at webrtc/audio/audio_state.cc:172
#2  webrtc::internal::AudioState::AddSendingStream(webrtc::AudioSendStream*, int, unsigned long) ()
    at webrtc/audio/audio_state.cc:100
#3  webrtc::internal::AudioSendStream::Start() () at webrtc/audio/audio_send_stream.cc:370
```

`webrtc::AudioSendStream` 把它自己加进 `webrtc::AudioState`，`webrtc::AudioState` 把新加的 `webrtc::AudioSendStream` 和之前已经添加的 `webrtc::AudioSendStream` 一起更新进 `webrtc::AudioTransport`。这个过程有几个值得关注的地方：

- `webrtc::AudioTransport` 支持把录制获得的同一份数据同时发送给多个 `webrtc::AudioSender`/`webrtc::AudioSendStream`，`webrtc::AudioSendStream` 用于管理音频数据的编码和编码数据的发送控制，这也就意味着，WebRTC 的音频数据处理管线，支持同时把录制获得的音频数据，以不同的编码方式和编码数据发送控制机制及策略发送到不同的网络，比如一路发送到基于 UDP 传输的 RTC 网络，另一路发送到基于 TCP 传输的 RTMP 网络。
- 如果添加的 `webrtc::AudioSendStream` 是第一个 `webrtc::AudioSendStream`，`webrtc::AudioState` 还会自动地初始化并启动录音。



#### 对象组成

![audio-send-stream-.png](audio-introduction.assets/audio-send-stream-.png)



### 6.10 ??? webrtc::AudioCodingModule



### 6.11 ??? webrtc::RTPSenderAudio







### ------

### 6.12 ??? webrtc::Transport

`webrtc::Transport` 也是一个适配和胶水模块，它把 `webrtc::AudioSender`/`webrtc::AudioSendStream` 得到的 RTP 和 RTCP 包发送给后面的网络接口模块。

### ------

### 6.13 ??? cricket::MediaChannel::NetworkInterface

- `cricket::MediaChannel::NetworkInterface` 用于实现真正地把 RTP 和 RTCP 包通过底层的网络接口和协议发送，如 UDP 等，ICE 的作用即为创建一个工作良好的网络接口模块实例。





## 参考

[WebRTC 的音频数据编码及发送控制管线 - 简书 (jianshu.com)](https://www.jianshu.com/p/64d40d7ca74a)
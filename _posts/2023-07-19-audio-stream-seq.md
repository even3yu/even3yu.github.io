---
layout: post
title: 音频数据流走向分析
date: 2023-07-19 23:35:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc audio
---


* content
{:toc}

---

## 1. 前言

webrtc 默认是对音频是是封装在内部，直接采集麦克风设备的语音数据作为数据源。可以看PeerConnectionFactoryInterface的接口中

- CreateAudioSource， 创建音频源；
- CreateAudioTrack，对sourc的封装
- AddTrack

接下去我们逐个分析下。



## 2. CreateAudioSource 流程分析

src/pc/peer_connection_factory.cc

```cpp
rtc::scoped_refptr<AudioSourceInterface>
PeerConnectionFactory::CreateAudioSource(const cricket::AudioOptions& options) {
  RTC_DCHECK(signaling_thread()->IsCurrent());
  rtc::scoped_refptr<LocalAudioSource> source(
      LocalAudioSource::Create(&options));
  return source;
}
```

主要作用是创建了AudioSource， 而LocalAudioSource是继承于AudioSourceInterface。

1. LocalAudioSource 存放了cricket::AudioOptions， 对3A等音频处理的开关控制；`src/pc/local_audio_source.h`

   ```cpp
   class LocalAudioSource : public Notifier<AudioSourceInterface> {
     ...
   }
   ```

2. 通过AudioSourceInterface的`void AddSink(AudioTrackSinkInterface* sink)` 就可以从source得到语音数据了。`src/api/media_stream_interface.h`

   ```cpp
   // AudioSourceInterface is a reference counted source used for AudioTracks.
   // The same source can be used by multiple AudioTracks.
   class RTC_EXPORT AudioSourceInterface : public MediaSourceInterface {
   	...
     // TODO(tommi): Make pure virtual.
     virtual void AddSink(AudioTrackSinkInterface* sink) {}
     virtual void RemoveSink(AudioTrackSinkInterface* sink) {}
     ...
   };
   ```

3. 这里有个疑问， `LocalAudioSource` 应该是一个可以自己获得或者生成音频 PCM 数据，并吐出去的组件。但是从代码上看，没有产生数据的代码，那么数据从何而来？其实Webrtc对于内置的采集的音频数据是不走这个流程。这个LocalAudioSource唯一作用是cricket::AudioOptions。

> 如果自定义音频数据，是不是只要自己继承下AudioTrackSinkInterface，实现数据的发送，就可以了。

### 2.1 LocalAudioSource::Create

src/pc/local_audio_source.cc

```cpp
rtc::scoped_refptr<LocalAudioSource> LocalAudioSource::Create(
    const cricket::AudioOptions* audio_options) {
  rtc::scoped_refptr<LocalAudioSource> source(
      new rtc::RefCountedObject<LocalAudioSource>());
  source->Initialize(audio_options);
  return source;
}
```

构建了LocalAudioSource对象。

### 2.2 LocalAudioSource::Initialize

src/pc/local_audio_source.cc

```cpp
void LocalAudioSource::Initialize(const cricket::AudioOptions* audio_options) {
  if (!audio_options)
    return;

  options_ = *audio_options;
}
```

保存了cricket::AudioOptions。

### 2.3 LocalAudioSource 继承关系

|                      | 方法                                                         |
| -------------------- | ------------------------------------------------------------ |
| NotifierInterface    | RegisterObserver                                             |
| MediaSourceInterface | remote/state                                                 |
| AudioSourceInterface | 1. 回调OnSetVolume, RegisterAudioObserver/UnregisterAudioObserver; 2. 回调数据onData AddSink/RemoveSink (AudioTrackSinkInterface) 3. options aec等相关的配置 4. SetVolume 设置音量 |
| Notifier             | 这是NotifierInterface的实现，存了数组                        |
| LocalAudioSource     |                                                              |



## 3. CreateAudioTrack 流程分析

src/pc/peer_connection_factory.cc

```cpp
rtc::scoped_refptr<AudioTrackInterface> PeerConnectionFactory::CreateAudioTrack(
    const std::string& id,
    AudioSourceInterface* source) {
  RTC_DCHECK(signaling_thread()->IsCurrent());
  rtc::scoped_refptr<AudioTrackInterface> track(AudioTrack::Create(id, source));
  return AudioTrackProxy::Create(signaling_thread(), track);
}
```

创建了AudioTrack。

1. 创建AudioTrack， 存储了AudioSource, 这就是上个步骤CreateAudioSource 创建的对象；

   ```cpp
   class AudioTrack : public MediaStreamTrack<AudioTrackInterface>,
                      public ObserverInterface {
    protected:
     // Protected ctor to force use of factory method.
     AudioTrack(const std::string& label,
                const rtc::scoped_refptr<AudioSourceInterface>& source);
   	...
    private:
     // AudioTrackInterface implementation.
     AudioSourceInterface* GetSource() const override;
   
    private:
     const rtc::scoped_refptr<AudioSourceInterface> audio_source_;
   	...
   };
   ```

   

2. 创建了AudioTrackProxy，其实这个对象的作用主要用来线程安全用的。在应用层调用AudioTrack的相关方法的时候，不需要关心线程安全问题。具体的实现可以参考 [webrtc proxy]()；



### 3.1 AudioTrack::Create

src/pc/audio_track.cc

```cpp
rtc::scoped_refptr<AudioTrack> AudioTrack::Create(
    const std::string& id,
    const rtc::scoped_refptr<AudioSourceInterface>& source) {
  return new rtc::RefCountedObject<AudioTrack>(id, source);
}
```

创建AudioTrack。

### 3.2 AudioTrack::AudioTrack

```cpp
AudioTrack::AudioTrack(const std::string& label,
                       const rtc::scoped_refptr<AudioSourceInterface>& source)
    : MediaStreamTrack<AudioTrackInterface>(label), audio_source_(source) {
  if (audio_source_) {
    audio_source_->RegisterObserver(this);
    OnChanged();
  }
}
```

1. 保存`rtc::scoped_refptr<AudioSourceInterface>& source`到audio_source_；
2. 注册RegisterObserver，收到OnChanged， 



### 3.3 AudioTrackProxy::Create

src/api/media_stream_track_proxy.h

src/api/proxy.h

```cpp
BEGIN_SIGNALING_PROXY_MAP(AudioTrack)
```

```cpp
#define BEGIN_SIGNALING_PROXY_MAP(c)                                         \
  PROXY_MAP_BOILERPLATE(c)                                                   \
  SIGNALING_PROXY_MAP_BOILERPLATE(c)                                         \
  REFCOUNTED_PROXY_MAP_BOILERPLATE(c)                                        \
 public:                                                                     \
  static rtc::scoped_refptr<c##ProxyWithInternal> Create(                    \
      rtc::Thread* signaling_thread, INTERNAL_CLASS* c) {                    \
    return new rtc::RefCountedObject<c##ProxyWithInternal>(signaling_thread, \
                                                           c);               \
  }
```

转换下

```cpp
template <class INTERNAL_CLASS>  class AudioTrackProxyWithInternal;  
typedef AudioTrackProxyWithInternal<AudioTrackInterface> AudioTrackProxy;    
template <class INTERNAL_CLASS>                         
class AudioTrackProxyWithInternal : public AudioTrackInterface {      
protected:                                             
    typedef AudioTrackInterface C;                                                                          
public:                                                
    const INTERNAL_CLASS* internal() const { return c_; } 
    INTERNAL_CLASS* internal() { return c_; }
protected:                                                              
  AudioTrackProxyWithInternal(rtc::Thread* signaling_thread, INTERNAL_CLASS* c) 
      : signaling_thread_(signaling_thread), c_(c) {}                    
private:                                                                
  mutable rtc::Thread* signaling_thread_;    
  
  ...
public:                                                                     
  static rtc::scoped_refptr<AudioTrackProxyWithInternal> Create(                    
      rtc::Thread* signaling_thread, INTERNAL_CLASS* c) {                  
    return new rtc::RefCountedObject<AudioTrackProxyWithInternal>(signaling_thread,  c);   
  }
  
  ...
};  
```

主要作用是调用AudioTrack方法的时候，线程安全。



## 4. AddTrack 流程分析

src/pc/peer_connection.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>> PeerConnection::AddTrack(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
 ...
  auto sender_or_error = rtp_manager()->AddTrack(track, stream_ids);
  if (sender_or_error.ok()) {
    sdp_handler_->UpdateNegotiationNeeded();
    stats_->AddTrack(track);
  }
  return sender_or_error;
}
```

把上一步骤创建的AudioTrack 加入到PeerConnectinon中。

### 4.1 RtpTransmissionManager::AddTrack

src/pc/rtp_transmission_manager.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrack(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {

  return (IsUnifiedPlan() ? AddTrackUnifiedPlan(track, stream_ids)
                          : AddTrackPlanB(track, stream_ids));
}
```

这里走AddTrackUnifiedPlan。



### 4.2 RtpTransmissionManager::AddTrackUnifiedPlan

src/pc/rtp_transmission_manager.cc

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpSenderInterface>>
RtpTransmissionManager::AddTrackUnifiedPlan(
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids) {
  auto transceiver = FindFirstTransceiverForAddedTrack(track);
  // transceiver 已经创建过了，这里先不分析
  if (transceiver) {
    RTC_LOG(LS_INFO) << "Reusing an existing "
                     << cricket::MediaTypeToString(transceiver->media_type())
                     << " transceiver for AddTrack.";
    if (transceiver->stopping()) {
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "The existing transceiver is stopping.");
    }

    if (transceiver->direction() == RtpTransceiverDirection::kRecvOnly) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendRecv);
    } else if (transceiver->direction() == RtpTransceiverDirection::kInactive) {
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kSendOnly);
    }
    transceiver->sender()->SetTrack(track);
    transceiver->internal()->sender_internal()->set_stream_ids(stream_ids);
    transceiver->internal()->set_reused_for_addtrack(true);
  } else {
    cricket::MediaType media_type =
        (track->kind() == MediaStreamTrackInterface::kAudioKind
             ? cricket::MEDIA_TYPE_AUDIO
             : cricket::MEDIA_TYPE_VIDEO);
    RTC_LOG(LS_INFO) << "Adding " << cricket::MediaTypeToString(media_type)
                     << " transceiver in response to a call to AddTrack.";
    std::string sender_id = track->id();
    // Avoid creating a sender with an existing ID by generating a random ID.
    // This can happen if this is the second time AddTrack has created a sender
    // for this track.
    if (FindSenderById(sender_id)) {
      sender_id = rtc::CreateRandomUuid();
    }
    
    // 创建sender
    auto sender = CreateSender(media_type, sender_id, track, stream_ids, {});
    auto receiver = CreateReceiver(media_type, rtc::CreateRandomUuid());
    // 创建Transceiver
    transceiver = CreateAndAddTransceiver(sender, receiver);
    transceiver->internal()->set_created_by_addtrack(true);
    transceiver->internal()->set_direction(RtpTransceiverDirection::kSendRecv);
  }
  return transceiver->sender();
}
```



### 4.3 RtpTransmissionManager::CreateSender

src/pc/rtp_transmission_manager.cc

```cpp
rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>>
RtpTransmissionManager::CreateSender(
    cricket::MediaType media_type,
    const std::string& id,
    rtc::scoped_refptr<MediaStreamTrackInterface> track,
    const std::vector<std::string>& stream_ids,
    const std::vector<RtpEncodingParameters>& send_encodings) {
	...
  rtc::scoped_refptr<RtpSenderProxyWithInternal<RtpSenderInternal>> sender;
  if (media_type == cricket::MEDIA_TYPE_AUDIO) {
		...
    sender = RtpSenderProxyWithInternal<RtpSenderInternal>::Create(
        signaling_thread(),
        AudioRtpSender::Create(worker_thread(), id, stats_, this));
    NoteUsageEvent(UsageEvent::AUDIO_ADDED);
  } else {
    ...
  }
  /////////////////////
  bool set_track_succeeded = sender->SetTrack(track);
  ///////////////////
  RTC_DCHECK(set_track_succeeded);
  sender->internal()->set_stream_ids(stream_ids);
  sender->internal()->set_init_send_encodings(send_encodings);
  return sender;
}
```



#### RtpSenderProxyWithInternal<RtpSenderInternal>::Create

INTERNAL_CLASS 是 RtpSenderInternal。

src/api/rtp_sender_interface.h

```cpp
BEGIN_SIGNALING_PROXY_MAP(RtpSender)
```

转换下

```cpp
template <class INTERNAL_CLASS>  class RtpSenderProxyWithInternal;  
typedef RtpSenderProxyWithInternal<RtpSenderInterface> RtpSenderProxy;    
template <class INTERNAL_CLASS>                         
class RtpSenderProxyWithInternal : public RtpSenderInterface {      
protected:                                             
    typedef RtpSenderInterface C;                                                                          
public:                                                
    const INTERNAL_CLASS* internal() const { return c_; } 
    INTERNAL_CLASS* internal() { return c_; }
protected:                                                              
  RtpSenderProxyWithInternal(rtc::Thread* signaling_thread, INTERNAL_CLASS* c) 
      : signaling_thread_(signaling_thread), c_(c) {}                    
private:                                                                
  mutable rtc::Thread* signaling_thread_;    
  
  ...
public:                                                                     
  static rtc::scoped_refptr<RtpSenderProxyWithInternal> Create(                    
      rtc::Thread* signaling_thread, INTERNAL_CLASS* c) {                  
    return new rtc::RefCountedObject<RtpSenderProxyWithInternal>(signaling_thread,  c);   
  }
  
  ...
};
```



#### AudioRtpSender::Create

```cpp
rtc::scoped_refptr<AudioRtpSender> AudioRtpSender::Create(
    rtc::Thread* worker_thread,
    const std::string& id,
    StatsCollectorInterface* stats,
    SetStreamsObserver* set_streams_observer) {
  return rtc::scoped_refptr<AudioRtpSender>(
      new rtc::RefCountedObject<AudioRtpSender>(worker_thread, id, stats,
                                                set_streams_observer));
}
```

作为`AudioRtpSender::Create`作为 `RtpSenderProxyWithInternal<RtpSenderInternal>::Create` 参数。

> 注意这里用的是RtpSenderInternal， 不是RtpSenderInterface。

#### AudioRtpSender,RtpSenderProxyWithInternal 类继承关系

```less
RtpSenderInterface
RtpSenderInternal
RtpSenderBase
AudioRtpSender
==============================
RtpSenderInterface
RtpSenderProxyWithInternal
```



#### RtpSenderProxyWithInternal::SetTrack

```cpp
PROXY_METHOD1(bool, SetTrack, MediaStreamTrackInterface*)
```

#### ------

#### RtpSenderBase::SetTrack

src/pc/rtp_sender.cc

```cpp
bool RtpSenderBase::SetTrack(MediaStreamTrackInterface* track) {
	...

  // Attach to new track.
  bool prev_can_send_track = can_send_track();
  // Keep a reference to the old track to keep it alive until we call SetSend.
  rtc::scoped_refptr<MediaStreamTrackInterface> old_track = track_;
  track_ = track;
  if (track_) {
    track_->RegisterObserver(this);
    AttachTrack();
  }

	...
  return true;
}
```

RtpSenderBase是AudioRtpSender 父类，把AudioTrack 绑定到AudioRtpSender。



#### AudioRtpSender::AttachTrack

src/pc/rtp_sender.cc

```cpp
void AudioRtpSender::AttachTrack() {
  RTC_DCHECK(track_);
  cached_track_enabled_ = track_->enabled();
  audio_track()->AddSink(sink_adapter_.get()); // LocalAudioSinkAdapter  sink_adapter_
}
```

1. audio_track() 返回的就是AudioTrackInterface

   ```cpp
    rtc::scoped_refptr<AudioTrackInterface> audio_track() const {
       return rtc::scoped_refptr<AudioTrackInterface>(
           static_cast<AudioTrackInterface*>(track_.get()));
    }
   ```

2. track_ 存的就是RtpSenderBase::SetTrack 的参数track。

3. sink_adapter_是 LocalAudioSinkAdapter。



#### AudioTrack::AddSink

src/pc/audio_track.cc

```cpp
void AudioTrack::AddSink(AudioTrackSinkInterface* sink) {
  RTC_DCHECK(thread_checker_.IsCurrent());
  if (audio_source_)
    audio_source_->AddSink(sink);
}
```

这样就打通，AudioSource--> AudioTrack --> LocalAudioSinkAdapter, 音频数据流向。
最终LocalAudioSinkAdapter拥有了音频数据。

#### ------



#### 第一轮音频数据的流向

```
AudioTrack::AddSink,放入的是LocalAudioSinkAdapter 的指针
LocalAudioSource::AddSink， 放入的是LocalAudioSinkAdapter 的指针
```

这样音频数据流的走向，如下：

```less
LocalAudioSource
-->AudioTrack
-->LocalAudioSinkAdapter
```



#### LocalAudioSinkAdapter

```less
cricket::AudioSource                SetSink，AudioSource::Sink,回调onData
-->AudioTrackSinkInterface					onData
-->LocalAudioSinkAdapter
```



```cpp
class LocalAudioSinkAdapter : public AudioTrackSinkInterface,
                              public cricket::AudioSource {
...
  private:
  // AudioSinkInterface implementation.
  void OnData(const void* audio_data,
              int bits_per_sample,
              int sample_rate,
              size_t number_of_channels,
              size_t number_of_frames,
              absl::optional<int64_t> absolute_capture_timestamp_ms) override;

  // AudioSinkInterface implementation.
  void OnData(const void* audio_data,
              int bits_per_sample,
              int sample_rate,
              size_t number_of_channels,
              size_t number_of_frames) override {
    OnData(audio_data, bits_per_sample, sample_rate, number_of_channels,
           number_of_frames,
           /*absolute_capture_timestamp_ms=*/absl::nullopt);
  }

  // AudioSinkInterface implementation.
  int NumPreferredChannels() const override { return num_preferred_channels_; }

  // cricket::AudioSource implementation.
  void SetSink(cricket::AudioSource::Sink* sink) override;

  cricket::AudioSource::Sink* sink_;
...  
}
```



## 5. 设置sdp的流程

这里不详细展开，只说setSsrc这部分。

### 5.1 SdpOfferAnswerHandler::DoSetLocalDescription

src/pc/sdp_offer_answer.cc

### 5.2 SdpOfferAnswerHandler::ApplyLocalDescription

src/pc/sdp_offer_answer.cc（1408行，1415行）



### ------

### 5.3 RtpSenderBase::SetSsrc

src/pc/rtp_sender.cc

```cpp
void RtpSenderBase::SetSsrc(uint32_t ssrc) {
  TRACE_EVENT0("webrtc", "RtpSenderBase::SetSsrc");
  if (stopped_ || ssrc == ssrc_) {
    return;
  }
  // If we are already sending with a particular SSRC, stop sending.
  if (can_send_track()) {
    ClearSend();
    RemoveTrackFromStats();
  }
  ssrc_ = ssrc;
  if (can_send_track()) {
    SetSend();
    AddTrackToStats();
  }
  ...
  
}
```



### 5.4 AudioRtpSender::SetSend()

src/pc/rtp_sender.cc

```cpp
void AudioRtpSender::SetSend() {
  ...
  cricket::AudioOptions options;
#if !defined(WEBRTC_CHROMIUM_BUILD) && !defined(WEBRTC_WEBKIT_BUILD)
  // TODO(tommi): Remove this hack when we move CreateAudioSource out of
  // PeerConnection.  This is a bit of a strange way to apply local audio
  // options since it is also applied to all streams/channels, local or remote.
  if (track_->enabled() && audio_track()->GetSource() &&
      !audio_track()->GetSource()->remote()) {
    options = audio_track()->GetSource()->options();
  }
#endif

  // |track_->enabled()| hops to the signaling thread, so call it before we hop
  // to the worker thread or else it will deadlock.
  bool track_enabled = track_->enabled();
  bool success = worker_thread_->Invoke<bool>(RTC_FROM_HERE, [&] {
    return voice_media_channel()->SetAudioSend(ssrc_, track_enabled, &options,
                                               sink_adapter_.get());
  });
  if (!success) {
    RTC_LOG(LS_ERROR) << "SetAudioSend: ssrc is incorrect: " << ssrc_;
  }
}
```

1. voice_media_channel()

   ```cpp
   cricket::VoiceMediaChannel* voice_media_channel() {
       return static_cast<cricket::VoiceMediaChannel*>(media_channel_);
     }

2. WebRtcVoiceMediaChannel 继承于 VoiceMediaChannel
   src/media/engine/webrtc_voice_engine.h

   ```cpp
   class WebRtcVoiceMediaChannel final : public VoiceMediaChannel,
                                         public webrtc::Transport {
                                           ...
                                         }
   ```

   

### 5.5 WebRtcVoiceMediaChannel::SetAudioSend

src/media/engine/webrtc_voice_engine.cc

```cpp
bool WebRtcVoiceMediaChannel::SetAudioSend(uint32_t ssrc,
                                           bool enable,
                                           const AudioOptions* options,
                                           AudioSource* source) {
  RTC_DCHECK(worker_thread_checker_.IsCurrent());
  // TODO(solenberg): The state change should be fully rolled back if any one of
  //                  these calls fail.
  if (!SetLocalSource(ssrc, source)) {
    return false;
  }
  if (!MuteStream(ssrc, !enable)) {
    return false;
  }
  if (enable && options) {
    return SetOptions(*options);
  }
  return true;
}
```



### 5.6 WebRtcVoiceMediaChannel::SetLocalSource

src/media/engine/webrtc_voice_engine.cc

```cpp
// AudioSource* source 就是 LocalAudioSinkAdapter
bool WebRtcVoiceMediaChannel::SetLocalSource(uint32_t ssrc,
                                             AudioSource* source) {
  auto it = send_streams_.find(ssrc);
  if (it == send_streams_.end()) {
    if (source) {
      // Return an error if trying to set a valid source with an invalid ssrc.
      RTC_LOG(LS_ERROR) << "SetLocalSource failed with ssrc " << ssrc;
      return false;
    }

    // The channel likely has gone away, do nothing.
    return true;
  }

  if (source) {
    it->second->SetSource(source);
  } else {
    it->second->ClearSource();
  }

  return true;
}
```



### 5.7 WebRtcVoiceMediaChannel::WebRtcAudioSendStream::SetSource

src/media/engine/webrtc_voice_engine.cc

```cpp
  // Starts the sending by setting ourselves as a sink to the AudioSource to
  // get data callbacks.
  // This method is called on the libjingle worker thread.
  // TODO(xians): Make sure Start() is called only once.
  void SetSource(AudioSource* source) {
    RTC_DCHECK(worker_thread_checker_.IsCurrent());
    RTC_DCHECK(source);
    if (source_) {
      RTC_DCHECK(source_ == source);
      return;
    }
    // AudioSource* source 就是 LocalAudioSinkAdapter
    source->SetSink(this);
    source_ = source;
    UpdateSendState();
  }
```

1. AudioSource* source, 就是 LocalAudioSinkAdapter

2. WebRtcVoiceMediaChannel::WebRtcAudioSendStream 继承于  AudioSource::Sink

   ```cpp
   class WebRtcVoiceMediaChannel::WebRtcAudioSendStream
       : public AudioSource::Sink {
   		...
       }
   ```

3. AudioSource
   src/media/base/audio_source.h

   ```cpp
   
   // Abstract interface for providing the audio data.
   // TODO(deadbeef): Rename this to AudioSourceInterface, and rename
   // webrtc::AudioSourceInterface to AudioTrackSourceInterface.
   class AudioSource {
    public:
     class Sink {
      public:
       // Callback to receive data from the AudioSource.
       virtual void OnData(
           const void* audio_data,
           int bits_per_sample,
           int sample_rate,
           size_t number_of_channels,
           size_t number_of_frames,
           absl::optional<int64_t> absolute_capture_timestamp_ms) = 0;
   
       // Called when the AudioSource is going away.
       virtual void OnClose() = 0;
   
       // Returns the number of channels encoded by the sink. This can be less than
       // the number_of_channels if down-mixing occur. A value of -1 means an
       // unknown number.
       virtual int NumPreferredChannels() const = 0;
   
      protected:
       virtual ~Sink() {}
     };
   
     // Sets a sink to the AudioSource. There can be only one sink connected
     // to the source at a time.
     virtual void SetSink(Sink* sink) = 0;
   
    protected:
     virtual ~AudioSource() {}
   };
   ```



### 5.8 LocalAudioSinkAdapter::SetSink

src/pc/rtp_sender.cc

```cpp
void LocalAudioSinkAdapter::SetSink(cricket::AudioSource::Sink* sink) {
  MutexLock lock(&lock_);
  RTC_DCHECK(!sink || !sink_);
  sink_ = sink;
}
```

1. cricket::AudioSource::Sink* sink_;



### 5.9 第二轮数据流走向

```
AudioTrack::AddSink,放入的是LocalAudioSinkAdapter 的指针
LocalAudioSource::AddSink， 放入的是LocalAudioSinkAdapter 的指针
LocalAudioSinkAdapter::SetSink, 放入的是WebRtcVoiceMediaChannel::WebRtcAudioSendStream 的指针
```

这样音频数据流的走向，如下：

```less
LocalAudioSource
-->AudioTrack
-->LocalAudioSinkAdapter
-->WebRtcVoiceMediaChannel::WebRtcAudioSendStream
```



### 5.10 LocalAudioSinkAdapter::OnData

src/pc/rtp_sender.cc

```cpp
void LocalAudioSinkAdapter::OnData(
    const void* audio_data,
    int bits_per_sample,
    int sample_rate,
    size_t number_of_channels,
    size_t number_of_frames,
    absl::optional<int64_t> absolute_capture_timestamp_ms) {
  MutexLock lock(&lock_);
  if (sink_) {
    sink_->OnData(audio_data, bits_per_sample, sample_rate, number_of_channels,
                  number_of_frames, absolute_capture_timestamp_ms);
    num_preferred_channels_ = sink_->NumPreferredChannels();
  }
}
```



### 5.11 AudioSource::Sink::onData

src/media/engine/webrtc_voice_engine.cc

```cpp
// AudioSource::Sink implementation.
  // This method is called on the audio thread.
  void OnData(const void* audio_data,
              int bits_per_sample,
              int sample_rate,
              size_t number_of_channels,
              size_t number_of_frames,
              absl::optional<int64_t> absolute_capture_timestamp_ms) override {
    RTC_DCHECK_EQ(16, bits_per_sample);
    RTC_CHECK_RUNS_SERIALIZED(&audio_capture_race_checker_);
    RTC_DCHECK(stream_);
    std::unique_ptr<webrtc::AudioFrame> audio_frame(new webrtc::AudioFrame());
    audio_frame->UpdateFrame(
        audio_frame->timestamp_, static_cast<const int16_t*>(audio_data),
        number_of_frames, sample_rate, audio_frame->speech_type_,
        audio_frame->vad_activity_, number_of_channels);
    // TODO(bugs.webrtc.org/10739): add dcheck that
    // |absolute_capture_timestamp_ms| always receives a value.
    if (absolute_capture_timestamp_ms) {
      audio_frame->set_absolute_capture_timestamp_ms(
          *absolute_capture_timestamp_ms);
    }
    stream_->SendAudioData(std::move(audio_frame));
  }
```



### 5.12 webrtc::AudioSendStream::SendAudioData

src/audio/audio_send_stream.cc

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
    audio_level_.ComputeLevel(*audio_frame, duration);
  }
  channel_send_->ProcessAndEncodeAudio(std::move(audio_frame));
}
```

### 5.13 第三轮数据流走向

```
AudioTrack::AddSink,放入的是LocalAudioSinkAdapter 的指针
LocalAudioSource::AddSink， 放入的是LocalAudioSinkAdapter 的指针
LocalAudioSinkAdapter::SetSink, 放入的是WebRtcVoiceMediaChannel::WebRtcAudioSendStream 的指针
```

这样音频数据流的走向，如下：

```less
LocalAudioSource
-->AudioTrack
-->LocalAudioSinkAdapter
-->WebRtcVoiceMediaChannel::WebRtcAudioSendStream
-->AudioSendStream
```


---
layout: post
title: WebRTC通话的同时播放本地音
date: 2024-01-01 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio
---


* content
{:toc}

---



## 0. 前言

WebRTC通话的同时播放本地音，有两种方式。

- 方式一（模拟一路接收流）
  把本地音乐当做一路对端过来的流来处理，这个方案的好处是不需要修改WebRTC的代码，调用WebRtcVoiceMediaChannel对象的AddRecvStream方法来添加一路流即可。通过WebRtcVoiceMediaChannel对象的OnPacketReceived方法传递数据，数据需要打包为RTP格式。
- 方式二（播放混音）
  把数据结构webrtc::AudioState的webrtc::AudioMixer对象导出来。
  通过webrtc::AudioMixer对象的AddSource来添加一路流，不需要的时候记得调用RemoveSource移除。
  这个方法的好处可以直接播放一路PCM数据的流，至于这路流是怎么来可以方便自己控制。



## 1. AudioPlayoutTrackInterface

api/media_stream_interface.h

```cpp
class AudioPlayoutTrackInterface {
 public:
  virtual bool GetAudioFrame(AudioFrame & audio_frame) = 0;
};
```

定义一个接口，外部传入数据`AudioFrame`。



## 2. PeerConnectionFactoryProxy

src/api/peer_connection_factory_proxy.h

```cpp
PROXY_METHOD1(void,
              SetAudioPlayoutTrack,
              AudioPlayoutTrackInterface*)
```



## 3. PeerConnectionFactory

src/api/peer_connection_interface.h

```cpp
virtual void SetAudioPlayoutTrack(AudioPlayoutTrackInterface* track) = 0;
```

pc/peer_connection_factory.h
pc/peer_connection_factory.cc

```cpp
void SetAudioPlayoutTrack(AudioPlayoutTrackInterface* track) override;

void PeerConnectionFactory::SetAudioPlayoutTrack(
  AudioPlayoutTrackInterface* track) {
  channel_manager()->SetAudioPlayoutTrack(track);
}
```



## 4. ChannelManager

pc/channel_manager.h

```cpp
void SetAudioPlayoutTrack(webrtc::AudioPlayoutTrackInterface* track);

void ChannelManager::SetAudioPlayoutTrack(
  webrtc::AudioPlayoutTrackInterface* track) {
  return worker_thread_->Invoke<void>(RTC_FROM_HERE, [&] {
    return media_engine_->voice().GetAudioState()->SetPlayoutTrack(track);
  });
}
```



## 5. WebRtcVoiceEngine

media/base/media_engine.h

```cpp
virtual void SetAudioPlayoutTrack(webrtc::AudioPlayoutTrackInterface* track) = 0;
```

media/engine/webrtc_voice_engine.cc
media/engine/webrtc_voice_engine.h

```cpp
void SetAudioPlayoutTrack(webrtc::AudioPlayoutTrackInterface* track) override;

void WebRtcVoiceEngine::SetAudioPlayoutTrack(webrtc::AudioPlayoutTrackInterface* track) {
  RTC_DCHECK(worker_thread_checker_.IsCurrent());
  audio_state()->SetPlayoutTrack(track);
}
```



## 6. webrtc::AudioState

call/audio_state.h

```cpp
  virtual void SetPlayoutTrack(AudioPlayoutTrackInterface* track) = 0;
```



## 7. webrtc::internal::AudioState

audio/audio_state.cc
audio/audio_state.h

```cpp
// 定义一个属性，就是
std::unique_ptr<PlayAudioMixerSource> playout_track_source_;


void AudioState::SetPlayoutTrack(webrtc::AudioPlayoutTrackInterface* track) {
  RTC_DCHECK(thread_checker_.IsCurrent());
  // 如果已经创建了playout_track_source_， 则先销毁 
  if (playout_track_source_ && playout_track_source_->track_ != track) {
    config_.audio_mixer->RemoveSource(playout_track_source_.get());
    playout_track_source_ = nullptr;
    UpdateNullAudioPollerState();
    if (receiving_streams_.empty()) {
      config_.audio_device_module->StopPlayout();
    }
  }

   // 创建 playout_track_source_
  if (track && !playout_track_source_) {
    playout_track_source_.reset(new AudioMixerSource(track));
     //给mixer 提供source
    config_.audio_mixer->AddSource(playout_track_source_.get());
    UpdateNullAudioPollerState();
     // 如果没有播放，则开始播放
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
}
```



```cpp
void AudioState::RemoveReceivingStream(webrtc::AudioReceiveStream* stream) {
  RTC_DCHECK(thread_checker_.IsCurrent());
  auto count = receiving_streams_.erase(stream);
  RTC_DCHECK_EQ(1, count);
  config_.audio_mixer->RemoveSource(
      static_cast<internal::AudioReceiveStream*>(stream));
  UpdateNullAudioPollerState();
  if (receiving_streams_.empty()) {
  
   if (receiving_streams_.empty()
   /////////////在去除最后一个RecieveStream的时候，如果playout_track_source_ 也是空，则停止播放//////////////////
   && !playout_track_source_
   ///////////////////////////////
   ) {
    config_.audio_device_module->StopPlayout();
  }
}
```



### 7.1 webrtc::AudioMixer::Source

```cpp
class PlayAudioMixerSource : public webrtc::AudioMixer::Source {
    public:
    PlayAudioMixerSource(webrtc::AudioPlayoutTrackInterface* track)
        : track_(track) {}
  
    AudioFrameInfo GetAudioFrameWithInfo(
        int sample_rate_hz, AudioFrame* audio_frame) override {
        audio_frame->sample_rate_hz_ = sample_rate_hz;
        if (track_->GetAudioFrame(*audio_frame)) {
            return AudioFrameInfo::kNormal;
        }
        return AudioFrameInfo::kError;
    }
  
    int Ssrc() const override { return  0; }
  
    int PreferredSampleRate() const override { return 48000; }
  
    webrtc::AudioPlayoutTrackInterface* track_;
}; 
```

实现PlayAudioMixerSource， 用于给AudioMixer。

api\audio\audio_mixer.h

```cpp
class Source {
   public:
    enum class AudioFrameInfo {
      kNormal,  // The samples in audio_frame are valid and should be used.
      kMuted,   // The samples in audio_frame should not be used, but
                // should be implicitly interpreted as zero. Other
                // fields in audio_frame may be read and should
                // contain meaningful values.
      kError,   // The audio_frame will not be used.
    };

    // Overwrites |audio_frame|. The data_ field is overwritten with
    // 10 ms of new audio (either 1 or 2 interleaved channels) at
    // |sample_rate_hz|. All fields in |audio_frame| must be updated.
    virtual AudioFrameInfo GetAudioFrameWithInfo(int sample_rate_hz,
                                                 AudioFrame* audio_frame) = 0;

    // A way for a mixer implementation to distinguish participants.
    virtual int Ssrc() const = 0;

    // A way for this source to say that GetAudioFrameWithInfo called
    // with this sample rate or higher will not cause quality loss.
    virtual int PreferredSampleRate() const = 0;

    virtual ~Source() {}
  };
```



## 参考

[WebRTC通话的同时播放本地音乐](https://blog.csdn.net/momo0853/article/details/122428934)
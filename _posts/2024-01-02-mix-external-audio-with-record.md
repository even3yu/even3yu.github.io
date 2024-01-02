---
layout: post
title: WebRTC支持伴音功能
date: 2024-01-02 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio
---


* content
{:toc}

---


## 0. 前言

在开启麦克风的同时，有些时候需要混入其他语音，比如伴音。



## 1. AudioRecordTrackInterface

api/media_stream_interface.h

```cpp
class AudioRecordTrackInterface {
 public:
  virtual bool GetAudioFrame(AudioFrame & audio_frame) = 0;
};
```

定义一个接口，外部传入数据`AudioFrame`。



## 2. PeerConnectionFactoryProxy

src/api/peer_connection_factory_proxy.h

```cpp
PROXY_METHOD1(void,
              SetExternalAudioMixedTrack,
              AudioPlayoutTrackInterface*)
```



## 3. PeerConnectionFactory

src/api/peer_connection_interface.h

```cpp
virtual void SetExternalAudioMixedTrack(AudioRecordTrackInterface* track) = 0;
```

pc/peer_connection_factory.h
pc/peer_connection_factory.cc

```cpp
void SetExternalAudioMixedTrack(AudioRecordTrackInterface* track) override;

void PeerConnectionFactory::SetExternalAudioMixedTrack(
  AudioRecordTrackInterface* track) {
  channel_manager()->SetExternalAudioMixedTrack(track);
}
```



## 4. ChannelManager

```cpp
void SetExternalAudioMixedTrack(webrtc::AudioRecordTrackInterface* track);

void ChannelManager::SetLocalAudioMixedTrack(
    webrtc::AudioRecordTrackInterface* track) {
  return worker_thread_->Invoke<void>(RTC_FROM_HERE, [&] {
    return media_engine_->voice().SetLocalAudioMixedTrack(track);
  });
}
```



## 5. WebRtcVoiceEngine

media/base/media_engine.h

```cpp
virtual void SetLocalAudioMixedTrack(webrtc::AudioRecordTrackInterface* track) = 0;
```

media/engine/webrtc_voice_engine.cc
media/engine/webrtc_voice_engine.h

```cpp
void SetLocalAudioMixedTrack(webrtc::AudioRecordTrackInterface* track) override;

void WebRtcVoiceEngine::SetLocalAudioMixedTrack(webrtc::AudioRecordTrackInterface* track) {
  RTC_DCHECK(worker_thread_checker_.IsCurrent());
  webrtc::AudioDeviceModule* pAdm = adm();
  if (pAdm) {
    webrtc::AudioTransport* pTransport = pAdm->GetAudioTransport();
    if (pTransport) {
      pTransport->SetLocalAudioMixedTrack(track);
    } 
  } 
}
```



## 6. AudioTransport

modules/audio_device/include/audio_device_defines.h

```cpp
virtual void SetLocalAudioMixedTrack(
      AudioRecordTrackInterface* track) {};
```



## 7. AudioTransportImpl

audio/audio_transport_impl.cc

```cpp
void AudioTransportImpl::SetLocalAudioMixedTrack(
    webrtc::AudioRecordTrackInterface* track) {
  rtc::CritScope lock(&capture_lock_);
  external_audio_track_ = track;
}
```



```cpp
AudioTransportImpl::RecordedDataIsAvailable(...)
{
  typing_noise_detected_ = typing_detected;
	...
  // mix
  if (external_audio_track_) {
      AudioFrame frame;
      frame.sample_rate_hz_ = audio_frame->sample_rate_hz_;
      frame.num_channels_ = audio_frame->num_channels_;
      frame.samples_per_channel_ = audio_frame->sample_rate_hz_ / 100;
      if (external_audio_track_->GetAudioFrame(frame)) {
        voe::MixWithSat(audio_frame->mutable_data(), audio_frame->num_channels_,
                        frame.data(), frame.num_channels_,
                        frame.samples_per_channel_ * frame.num_channels_);
      }
    }
    ..
}
```




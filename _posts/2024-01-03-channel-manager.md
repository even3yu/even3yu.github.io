---
layout: post
title: ChannelManager
date: 2024-01-03 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio channel webrtc
---


* content
{:toc}

---


## 1. 前言

WeRTC有很多Channel，那么谁来管理这些Channel呢？自然是ChannelManager。ChannelManager会创建MediaChannel，并创建BaseChannel，将MediaChannel挂在相应的BaseChannel上。

ChannelManager是怎样创建MediaChannel的呢？实际上是通过其成员`media_engine_`创建的。这个`media_engine_`就是MediaEngineInterface，也就是媒体引擎接口，这个接口的最终目的就是创建MediaChannel。可见MediaChannel是由MediaEngine创建的，准确来说是WebrtcVoiceEngine和WebrtcVideoEngine。



## 2. ChannelManager

```less
pc/channel_manager.h
pc/channel_manager.cc
```



## 3. 说明

ChannelManager， 提供了创建BaseChannel ( VoiceChannel，Video Channel)的接口， 存储在数组中。在创建BaseChannel的时候，同时也会创建 MediaChannel，存放在BaseChannel中。



## 4. 属性

```cpp
  std::vector<std::unique_ptr<VoiceChannel>> voice_channels_;
  std::vector<std::unique_ptr<VideoChannel>> video_channels_;

  std::unique_ptr<MediaEngineInterface> media_engine_;
```



## 5. ChannelManager::ChannelManager

```cpp
ChannelManager::ChannelManager(
    std::unique_ptr<MediaEngineInterface> media_engine,
    std::unique_ptr<DataEngineInterface> data_engine,
    rtc::Thread* worker_thread,
    rtc::Thread* network_thread)
    : media_engine_(std::move(media_engine)),
      data_engine_(std::move(data_engine)),
      main_thread_(rtc::Thread::Current()),
      worker_thread_(worker_thread),
      network_thread_(network_thread) {
...
}
```

MediaEngineInterface media_engine 是从外面传入的，是在创建PeerConnectionFactory的时候创建的media_engine。

### 5.1 ChannelManager 创建时机

> PeerConnectionFactory::Create
> ConnectionContext::Create
> ConnectionContext::ConnectionContext
> ChannelManager::ChannelManager



## 6. ChannelManager::media_engine

```cpp
MediaEngineInterface* media_engine() { return media_engine_.get(); }
```



## 7. !!! ChannelManager::CreateVoiceChannel

```cpp
VoiceChannel* ChannelManager::CreateVoiceChannel(
    webrtc::Call* call,
    const cricket::MediaConfig& media_config,
    webrtc::RtpTransportInternal* rtp_transport,
    rtc::Thread* signaling_thread,
    const std::string& content_name,
    bool srtp_required,
    const webrtc::CryptoOptions& crypto_options,
    rtc::UniqueRandomIdGenerator* ssrc_generator,
    const AudioOptions& options) {
  
	...
  
	// MediaEngineInterface media_engine_
  VoiceMediaChannel* media_channel = media_engine_->voice().CreateMediaChannel(
      call, media_config, options, crypto_options);
  if (!media_channel) {
    return nullptr;
  }

  auto voice_channel = std::make_unique<VoiceChannel>(
      worker_thread_, network_thread_, signaling_thread,
      absl::WrapUnique(media_channel), content_name, srtp_required,
      crypto_options, ssrc_generator);

  voice_channel->Init_w(rtp_transport);

  VoiceChannel* voice_channel_ptr = voice_channel.get();
  // std::vector<std::unique_ptr<VoiceChannel>> voice_channels_;
  voice_channels_.push_back(std::move(voice_channel));
  return voice_channel_ptr;
}
```

1. 通过WebRtcVoiceEngine 创建 VoiceMediaChannel；
2. 创建VoiceChannel， 把VoiceMediaChannel 作为参数传入；
3. ??? 同时把`webrtc::RtpTransportInternal* rtp_transport` 传递给 VoiceChannel； 这个细节后面做分析；

### 7.1 -- VoiceMediaChannel

```less
MediaChannel
VoiceMediaChannel
WebRtcVoiceMediaChannel
```



### 7.2 -- VoiceChannel

```less
BaseChannel
VoiceChannel
```



### 7.3 ??? webrtc::RtpTransportInternal

### 7.4 调用堆栈

```less
SdpOfferAnswerHandler::CreateVoiceChannel
  ChannelManager::CreateVoiceChannel
		WebRtcVoiceEngine::CreateMediaChannel
		VoiceChannel::VoiceChannel
```



### 7.5 !!! WebRtcVoiceEngine::CreateMediaChannel

media/engine/webrtc_voice_engine.cc

```cpp
VoiceMediaChannel* WebRtcVoiceEngine::CreateMediaChannel(
    webrtc::Call* call,
    const MediaConfig& config,
    const AudioOptions& options,
    const webrtc::CryptoOptions& crypto_options) {
  RTC_DCHECK(worker_thread_checker_.IsCurrent());
  return new WebRtcVoiceMediaChannel(this, config, options, crypto_options,
                                     call);
}
```

创建WebRtcVoiceMediaChannel。

```less
MediaChannel
VoiceMediaChannel
WebRtcVoiceMediaChannel
```



## 8. !!! ChannelManager::CreateVideoChannel

```cpp
VideoChannel* ChannelManager::CreateVideoChannel(
    webrtc::Call* call,
    const cricket::MediaConfig& media_config,
    webrtc::RtpTransportInternal* rtp_transport,
    rtc::Thread* signaling_thread,
    const std::string& content_name,
    bool srtp_required,
    const webrtc::CryptoOptions& crypto_options,
    rtc::UniqueRandomIdGenerator* ssrc_generator,
    const VideoOptions& options,
    webrtc::VideoBitrateAllocatorFactory* video_bitrate_allocator_factory) {
  
  ...

  VideoMediaChannel* media_channel = media_engine_->video().CreateMediaChannel(
      call, media_config, options, crypto_options,
      video_bitrate_allocator_factory);
  if (!media_channel) {
    return nullptr;
  }

  auto video_channel = std::make_unique<VideoChannel>(
      worker_thread_, network_thread_, signaling_thread,
      absl::WrapUnique(media_channel), content_name, srtp_required,
      crypto_options, ssrc_generator);

  video_channel->Init_w(rtp_transport);

  VideoChannel* video_channel_ptr = video_channel.get();
  video_channels_.push_back(std::move(video_channel));
  return video_channel_ptr;
}
```

1. 通过WebRtcVideoEngine 创建 VideoMediaChannel；
2. 创建VideoChannel， 把VideoMediaChannel 作为参数传入；
3. ??? 同时把`webrtc::RtpTransportInternal* rtp_transport` 传递给 VideoChannel； 这个细节后面做分析；

### 8.1 -- VideoMediaChannel

```less
MediaChannel
VideoMediaChannel
```



### 8.2 -- VideoChannel

```less
BaseChannel
VoiceChannel
```



### 8.3 ??? webrtc::RtpTransportInternal

### 8.4 调用堆栈

```less
SdpOfferAnswerHandler::CreateVideoChannel
  ChannelManager::CreateVideoChannel
		WebRtcVideoEngine::CreateMediaChannel
		VideoChannel::VideoChannel
```



### 8.5 !!! WebRtcVideoEngine::CreateMediaChannel

media/engine/webrtc_video_engine.cc

```cpp
VideoMediaChannel* WebRtcVideoEngine::CreateMediaChannel(
    webrtc::Call* call,
    const MediaConfig& config,
    const VideoOptions& options,
    const webrtc::CryptoOptions& crypto_options,
    webrtc::VideoBitrateAllocatorFactory* video_bitrate_allocator_factory) {
  RTC_LOG(LS_INFO) << "CreateMediaChannel. Options: " << options.ToString();
  return new WebRtcVideoChannel(call, config, options, crypto_options,
                                encoder_factory_.get(), decoder_factory_.get(),
                                video_bitrate_allocator_factory);
}
```

创建WebRtcVideoChannel。

```less
MediaChannel
VideoMediaChannel
WebRtcVideoChannel
```


---
layout: post
title: webrtc channel 创建过程
date: 2024-01-06 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: audio webrtc channel
---


* content
{:toc}

---


## Channel 创建过程



![img]({{ site.url }}{{ site.baseurl }}/images/channel-create.assets/channel-create.webp)



- PeerConnection::ApplyRemoteDescription 和 PeerConnection::ApplyLocalDescription 

- PeerConnection::UpdateTransceiversAndDataChannels 

  针对每一个 **m=** 行，都会创建一个 RtpTransceiver 对象，而每一个 RtpTransceiver 对象都会创建一个对应的 BaseChannel 对象。

- SdpOfferAnswerHandler::CreateVoiceChannel

- PeerConnection::GetTransport
  创建 BaseChannel 之前，会根据 **m=** 的值在 JsepTransportController 中获取一个 RtpTransport 对象，这个对象是提前创建好的。

- ChannelManager::CreateVoiceChannel

  创建好 VoiceChannel 及 对应的 WebRtcVoiceChannel。

  WebRtcVoiceEngine::CreateMediaChannel

- WebRtcVoiceChannel::WebRtcVoiceChannel

- VoiceChannel::VoiceChannel

- BaseChannel::init_w

- BaseChannel::SetRtpTransport
  将 VoiceChannel 注册到 RtpTransportInternal 对象，准备接收 Transport 分发的数据。

- RtpTransportInternal::RegistRtpDemuxerSink
  RtpDemuxer 会进行接收数据的分发。

- MediaChannel::SetInterface




![caller-set-local-description]({{ site.url }}{{ site.baseurl }}/images/channel-create.assets/caller-set-local-description.jpg)
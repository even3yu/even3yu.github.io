---
layout: post
title: Frame encryptor
date: 2024-03-11 13:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc stream encryptor video
---


* content
{:toc}

---



## RtpSenderInterface::SetFrameEncryptor

api/rtp_sender_interface.h

```cpp
  // Sets a user defined frame encryptor that will encrypt the entire frame
  // before it is sent across the network. This will encrypt the entire frame
  // using the user provided encryption mechanism regardless of whether SRTP is
  // enabled or not.
  virtual void SetFrameEncryptor(
      rtc::scoped_refptr<FrameEncryptorInterface> frame_encryptor);
```



### FrameEncryptorInterface

api/crypto/frame_encryptor_interface.h

```cpp
// FrameEncryptorInterface allows users to provide a custom encryption
// implementation to encrypt all outgoing audio and video frames. The user must
// also provide a FrameDecryptorInterface to be able to decrypt the frames on
// the receiving device. Note this is an additional layer of encryption in
// addition to the standard SRTP mechanism and is not intended to be used
// without it. Implementations of this interface will have the same lifetime as
// the RTPSenders it is attached to. Additional data may be null.
class FrameEncryptorInterface : public rtc::RefCountInterface {
 public:
  ~FrameEncryptorInterface() override {}

  // Attempts to encrypt the provided frame. You may assume the encrypted_frame
  // will match the size returned by GetMaxCiphertextByteSize for a give frame.
  // You may assume that the frames will arrive in order if SRTP is enabled.
  // The ssrc will simply identify which stream the frame is travelling on. You
  // must set bytes_written to the number of bytes you wrote in the
  // encrypted_frame. 0 must be returned if successful all other numbers can be
  // selected by the implementer to represent error codes.
  virtual int Encrypt(cricket::MediaType media_type,
                      uint32_t ssrc,
                      rtc::ArrayView<const uint8_t> additional_data,
                      rtc::ArrayView<const uint8_t> frame,
                      rtc::ArrayView<uint8_t> encrypted_frame,
                      size_t* bytes_written) = 0;

  // Returns the total required length in bytes for the output of the
  // encryption. This can be larger than the actual number of bytes you need but
  // must never be smaller as it informs the size of the encrypted_frame buffer.
  virtual size_t GetMaxCiphertextByteSize(cricket::MediaType media_type,
                                          size_t frame_size) = 0;
};
```



## RtpSenderBase::SetFrameEncryptor

RtpSenderBase 子类 VideoRtpSender，AudioRtpSender

pc/rtp_sender.cc

```cpp
void RtpSenderBase::SetFrameEncryptor(
    rtc::scoped_refptr<FrameEncryptorInterface> frame_encryptor) {
  frame_encryptor_ = std::move(frame_encryptor);
  // Special Case: Set the frame encryptor to any value on any existing channel.
  // MediaChannel 
  if (media_channel_ && ssrc_ && !stopped_) {
    worker_thread_->Invoke<void>(RTC_FROM_HERE, [&] {
      media_channel_->SetFrameEncryptor(ssrc_, frame_encryptor_);
    });
  }
}
```



## ---------

## WebRtcVoiceMediaChannel::SetFrameEncryptor

MediaChannel  子类  WebRtcVoiceMediaChannel，WebRtcVideoMediaChannel

media/engine/webrtc_voice_engine.cc

```cpp
 void WebRtcVoiceMediaChannel::SetFrameEncryptor(
    uint32_t ssrc,
    rtc::scoped_refptr<webrtc::FrameEncryptorInterface> frame_encryptor) {
  RTC_DCHECK(worker_thread_checker_.IsCurrent());
  auto matching_stream = send_streams_.find(ssrc);
  if (matching_stream != send_streams_.end()) {
    matching_stream->second->SetFrameEncryptor(frame_encryptor);
  }
}
```



## WebRtcVoiceMediaChannel::WebRtcAudioSendStream::SetFrameEncryptor

```cpp
WebRtcVoiceMediaChannel::WebRtcAudioSendStream
    : public AudioSource::Sink
    ...
```



media/engine/webrtc_voice_engine.cc

```cpp
  void SetFrameEncryptor(
      rtc::scoped_refptr<webrtc::FrameEncryptorInterface> frame_encryptor) {
    RTC_DCHECK(worker_thread_checker_.IsCurrent());
    config_.frame_encryptor = frame_encryptor;
    ReconfigureAudioSendStream();
  }
```



## WebRtcVoiceMediaChannel::WebRtcAudioSendStream::ReconfigureAudioSendStream

media/engine/webrtc_voice_engine.cc

```cpp
  void ReconfigureAudioSendStream() {
    RTC_DCHECK(worker_thread_checker_.IsCurrent());
    RTC_DCHECK(stream_);
    stream_->Reconfigure(config_);
  }
```

1. 这里的stream_ 就是` webrtc::AudioSendStream* stream_ = nullptr;`

   ```
   stream_ = call_->CreateAudioSendStream(config_);
   ```

   在WebRtcVoiceMediaChannel::WebRtcAudioSendStream 构造函数里创建的。



## AudioSendStream::Reconfigure

audio/audio_send_stream.cc

```cpp
void AudioSendStream::Reconfigure(
    const webrtc::AudioSendStream::Config& new_config) {
  RTC_DCHECK(worker_thread_checker_.IsCurrent());
  ConfigureStream(new_config, false);
}
```



## AudioSendStream::ConfigureStream

audio/audio_send_stream.cc

```cpp
void AudioSendStream::ConfigureStream(
    const webrtc::AudioSendStream::Config& new_config,
    bool first_time) {
  RTC_LOG(LS_INFO) << "AudioSendStream::ConfigureStream: "
                   << new_config.ToString();
  UpdateEventLogStreamConfig(event_log_, new_config,
                             first_time ? nullptr : &config_);

  const auto& old_config = config_;

  // Configuration parameters which cannot be changed.
  RTC_DCHECK(first_time ||
             old_config.send_transport == new_config.send_transport);
  RTC_DCHECK(first_time || old_config.rtp.ssrc == new_config.rtp.ssrc);
  if (suspended_rtp_state_ && first_time) {
    rtp_rtcp_module_->SetRtpState(*suspended_rtp_state_);
  }
  if (first_time || old_config.rtp.c_name != new_config.rtp.c_name) {
    channel_send_->SetRTCP_CNAME(new_config.rtp.c_name);
  }

  // Enable the frame encryptor if a new frame encryptor has been provided.
  if (first_time || new_config.frame_encryptor != old_config.frame_encryptor) {
    channel_send_->SetFrameEncryptor(new_config.frame_encryptor);
  }

  if (first_time ||
      new_config.frame_transformer != old_config.frame_transformer) {
    channel_send_->SetEncoderToPacketizerFrameTransformer(
        new_config.frame_transformer);
  }

  if (first_time ||
      new_config.rtp.extmap_allow_mixed != old_config.rtp.extmap_allow_mixed) {
    rtp_rtcp_module_->SetExtmapAllowMixed(new_config.rtp.extmap_allow_mixed);
  }

  const ExtensionIds old_ids = FindExtensionIds(old_config.rtp.extensions);
  const ExtensionIds new_ids = FindExtensionIds(new_config.rtp.extensions);

  // Audio level indication
  if (first_time || new_ids.audio_level != old_ids.audio_level) {
    channel_send_->SetSendAudioLevelIndicationStatus(new_ids.audio_level != 0,
                                                     new_ids.audio_level);
  }

  if (first_time || new_ids.abs_send_time != old_ids.abs_send_time) {
    rtp_rtcp_module_->DeregisterSendRtpHeaderExtension(
        kRtpExtensionAbsoluteSendTime);
    if (new_ids.abs_send_time) {
      rtp_rtcp_module_->RegisterRtpHeaderExtension(AbsoluteSendTime::kUri,
                                                   new_ids.abs_send_time);
    }
  }

  bool transport_seq_num_id_changed =
      new_ids.transport_sequence_number != old_ids.transport_sequence_number;
  if (first_time ||
      (transport_seq_num_id_changed && !allocate_audio_without_feedback_)) {
    if (!first_time) {
      channel_send_->ResetSenderCongestionControlObjects();
    }

    RtcpBandwidthObserver* bandwidth_observer = nullptr;

    if (!allocate_audio_without_feedback_ &&
        new_ids.transport_sequence_number != 0) {
      rtp_rtcp_module_->RegisterRtpHeaderExtension(
          TransportSequenceNumber::kUri, new_ids.transport_sequence_number);
      // Probing in application limited region is only used in combination with
      // send side congestion control, wich depends on feedback packets which
      // requires transport sequence numbers to be enabled.
      // Optionally request ALR probing but do not override any existing
      // request from other streams.
      if (enable_audio_alr_probing_) {
        rtp_transport_->EnablePeriodicAlrProbing(true);
      }
      bandwidth_observer = rtp_transport_->GetBandwidthObserver();
    }
    channel_send_->RegisterSenderCongestionControlObjects(rtp_transport_,
                                                          bandwidth_observer);
  }
  // MID RTP header extension.
  if ((first_time || new_ids.mid != old_ids.mid ||
       new_config.rtp.mid != old_config.rtp.mid) &&
      new_ids.mid != 0 && !new_config.rtp.mid.empty()) {
    rtp_rtcp_module_->RegisterRtpHeaderExtension(RtpMid::kUri, new_ids.mid);
    rtp_rtcp_module_->SetMid(new_config.rtp.mid);
  }

  // RID RTP header extension
  if ((first_time || new_ids.rid != old_ids.rid ||
       new_ids.repaired_rid != old_ids.repaired_rid ||
       new_config.rtp.rid != old_config.rtp.rid)) {
    if (new_ids.rid != 0 || new_ids.repaired_rid != 0) {
      if (new_config.rtp.rid.empty()) {
        rtp_rtcp_module_->DeregisterSendRtpHeaderExtension(RtpStreamId::kUri);
      } else if (new_ids.repaired_rid != 0) {
        rtp_rtcp_module_->RegisterRtpHeaderExtension(RtpStreamId::kUri,
                                                     new_ids.repaired_rid);
      } else {
        rtp_rtcp_module_->RegisterRtpHeaderExtension(RtpStreamId::kUri,
                                                     new_ids.rid);
      }
    }
    rtp_rtcp_module_->SetRid(new_config.rtp.rid);
  }

  if (first_time || new_ids.abs_capture_time != old_ids.abs_capture_time) {
    rtp_rtcp_module_->DeregisterSendRtpHeaderExtension(
        kRtpExtensionAbsoluteCaptureTime);
    if (new_ids.abs_capture_time) {
      rtp_rtcp_module_->RegisterRtpHeaderExtension(
          AbsoluteCaptureTimeExtension::kUri, new_ids.abs_capture_time);
    }
  }

  if (!ReconfigureSendCodec(new_config)) {
    RTC_LOG(LS_ERROR) << "Failed to set up send codec state.";
  }

  // Set currently known overhead (used in ANA, opus only).
  {
    MutexLock lock(&overhead_per_packet_lock_);
    UpdateOverheadForEncoder();
  }

  channel_send_->CallEncoder([this](AudioEncoder* encoder) {
    if (!encoder) {
      return;
    }
    worker_queue_->PostTask(
        [this, length_range = encoder->GetFrameLengthRange()] {
          RTC_DCHECK_RUN_ON(worker_queue_);
          frame_length_range_ = length_range;
        });
  });

  if (sending_) {
    ReconfigureBitrateObserver(new_config);
  }
  config_ = new_config;
}
```



## ChannelSend::SetFrameEncryptor

audio/channel_send.cc

```cpp
void ChannelSend::SetFrameEncryptor(
    rtc::scoped_refptr<FrameEncryptorInterface> frame_encryptor) {
  RTC_DCHECK_RUN_ON(&worker_thread_checker_);
  encoder_queue_.PostTask([this, frame_encryptor]() mutable {
    RTC_DCHECK_RUN_ON(&encoder_queue_);
    frame_encryptor_ = std::move(frame_encryptor);
  });
}
```



-----------

### ChannelSend::SendRtpAudio

audio/channel_send.cc

这里用到了frame_encryptor_。

```cpp
int32_t ChannelSend::SendRtpAudio(AudioFrameType frameType,
                                  uint8_t payloadType,
                                  uint32_t rtp_timestamp,
                                  rtc::ArrayView<const uint8_t> payload,
                                  int64_t absolute_capture_timestamp_ms) {
  if (_includeAudioLevelIndication) {
    // Store current audio level in the RTP sender.
    // The level will be used in combination with voice-activity state
    // (frameType) to add an RTP header extension
    rtp_sender_audio_->SetAudioLevel(rms_level_.Average());
  }

  // E2EE Custom Audio Frame Encryption (This is optional).
  // Keep this buffer around for the lifetime of the send call.
  rtc::Buffer encrypted_audio_payload;
  // We don't invoke encryptor if payload is empty, which means we are to send
  // DTMF, or the encoder entered DTX.
  // TODO(minyue): see whether DTMF packets should be encrypted or not. In
  // current implementation, they are not.
  if (!payload.empty()) {
    if (frame_encryptor_ != nullptr) {
      // TODO(benwright@webrtc.org) - Allocate enough to always encrypt inline.
      // Allocate a buffer to hold the maximum possible encrypted payload.
      size_t max_ciphertext_size = frame_encryptor_->GetMaxCiphertextByteSize(
          cricket::MEDIA_TYPE_AUDIO, payload.size());
      encrypted_audio_payload.SetSize(max_ciphertext_size);

      // Encrypt the audio payload into the buffer.
      size_t bytes_written = 0;
      int encrypt_status = frame_encryptor_->Encrypt(
          cricket::MEDIA_TYPE_AUDIO, rtp_rtcp_->SSRC(),
          /*additional_data=*/nullptr, payload, encrypted_audio_payload,
          &bytes_written);
      if (encrypt_status != 0) {
        RTC_DLOG(LS_ERROR)
            << "Channel::SendData() failed encrypt audio payload: "
            << encrypt_status;
        return -1;
      }
      // Resize the buffer to the exact number of bytes actually used.
      encrypted_audio_payload.SetSize(bytes_written);
      // Rewrite the payloadData and size to the new encrypted payload.
      payload = encrypted_audio_payload;
    } else if (crypto_options_.sframe.require_frame_encryption) {
      RTC_DLOG(LS_ERROR) << "Channel::SendData() failed sending audio payload: "
                            "A frame encryptor is required but one is not set.";
      return -1;
    }
  }

  // Push data from ACM to RTP/RTCP-module to deliver audio frame for
  // packetization.
  if (!rtp_rtcp_->OnSendingRtpFrame(rtp_timestamp,
                                    // Leaving the time when this frame was
                                    // received from the capture device as
                                    // undefined for voice for now.
                                    -1, payloadType,
                                    /*force_sender_report=*/false)) {
    return -1;
  }

  // RTCPSender has it's own copy of the timestamp offset, added in
  // RTCPSender::BuildSR, hence we must not add the in the offset for the above
  // call.
  // TODO(nisse): Delete RTCPSender:timestamp_offset_, and see if we can confine
  // knowledge of the offset to a single place.

  // This call will trigger Transport::SendPacket() from the RTP/RTCP module.
  if (!rtp_sender_audio_->SendAudio(
          frameType, payloadType, rtp_timestamp + rtp_rtcp_->StartTimestamp(),
          payload.data(), payload.size(), absolute_capture_timestamp_ms)) {
    RTC_DLOG(LS_ERROR)
        << "ChannelSend::SendData() failed to send data to RTP/RTCP module";
    return -1;
  }

  return 0;
}
```





## ----------

## WebRtcVideoChannel::SetFrameEncryptor

media/engine/webrtc_video_engine.cc

```cpp
void WebRtcVideoChannel::SetFrameEncryptor(
    uint32_t ssrc,
    rtc::scoped_refptr<webrtc::FrameEncryptorInterface> frame_encryptor) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  auto matching_stream = send_streams_.find(ssrc);
  if (matching_stream != send_streams_.end()) {
    matching_stream->second->SetFrameEncryptor(frame_encryptor);
  } else {
    RTC_LOG(LS_ERROR) << "No stream found to attach frame encryptor";
  }
}
```



## WebRtcVideoChannel::WebRtcVideoSendStream::SetFrameEncryptor

media/engine/webrtc_video_engine.cc

```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::SetFrameEncryptor(
    rtc::scoped_refptr<webrtc::FrameEncryptorInterface> frame_encryptor) {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  parameters_.config.frame_encryptor = frame_encryptor;
  if (stream_) {
    RTC_LOG(LS_INFO)
        << "RecreateWebRtcStream (send) because of SetFrameEncryptor, ssrc="
        << parameters_.config.rtp.ssrcs[0];
    RecreateWebRtcStream();
  }
}
```



## WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream

media/engine/webrtc_video_engine.cc

```
WebRtcVideoSendStream::SetCodec
WebRtcVideoSendStream::SetSendParameters
WebRtcVideoSendStream::SetFrameEncryptor
WebRtcVideoSendStream::SetEncoderToPacketizerFrameTransforme
```

这几个地方都会调用`RecreateWebRtcStream`。 stream 指的是 webrtc::VideoSendStream* stream_

```cpp
webrtc::VideoSendStream
internal::VideoSendStream
```





```cpp
void WebRtcVideoChannel::WebRtcVideoSendStream::RecreateWebRtcStream() {
  RTC_DCHECK_RUN_ON(&thread_checker_);
  if (stream_ != NULL) {
    call_->DestroyVideoSendStream(stream_);
  }

  RTC_CHECK(parameters_.codec_settings);
  RTC_DCHECK_EQ((parameters_.encoder_config.content_type ==
                 webrtc::VideoEncoderConfig::ContentType::kScreen),
                parameters_.options.is_screencast.value_or(false))
      << "encoder content type inconsistent with screencast option";
  parameters_.encoder_config.encoder_specific_settings =
      ConfigureVideoEncoderSettings(parameters_.codec_settings->codec);

  webrtc::VideoSendStream::Config config = parameters_.config.Copy();
  if (!config.rtp.rtx.ssrcs.empty() && config.rtp.rtx.payload_type == -1) {
    RTC_LOG(LS_WARNING) << "RTX SSRCs configured but there's no configured RTX "
                           "payload type the set codec. Ignoring RTX.";
    config.rtp.rtx.ssrcs.clear();
  }
  if (parameters_.encoder_config.number_of_streams == 1) {
    // SVC is used instead of simulcast. Remove unnecessary SSRCs.
    if (config.rtp.ssrcs.size() > 1) {
      config.rtp.ssrcs.resize(1);
      if (config.rtp.rtx.ssrcs.size() > 1) {
        config.rtp.rtx.ssrcs.resize(1);
      }
    }
  }
  // webrtc::VideoSendStream* stream_
  stream_ = call_->CreateVideoSendStream(std::move(config),
                                         parameters_.encoder_config.Copy());

  parameters_.encoder_config.encoder_specific_settings = NULL;

  if (source_) {
    stream_->SetSource(source_, GetDegradationPreference());
  }

  // Call stream_->Start() if necessary conditions are met.
  UpdateSendState();
}
```


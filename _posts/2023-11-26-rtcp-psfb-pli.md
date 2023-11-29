---
layout: post
title: rtcp psfb pli 关键帧请求
date: 2023-11-26 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtcp pli
---


* content
{:toc}

---

## 1. 前言

NACK是一种通知技术，只是触发通知的条件刚好的ACK相反，在未收到消息时，通知发送方“我未收到消息”，即通知未达。
在rfc4585协议中定义可重传未到达数据的类型有二种：

1. RTPFB：rtp报文丢失重传。

2. PSFB：指定净荷重传，指定净荷重传里面又分如下三种：
   - PLI    (Picture Loss Indication) 视频帧丢失重传。
   - SLI    (Slice Loss Indication)    slice丢失重转。
   - RPSI (Reference Picture Selection Indication)参考帧丢失重传。

关键帧也叫做即时刷新帧，简称IDR帧。对视频来说，IDR帧的解码无需参考之前的帧，因此在丢包严重时可以通过发送关键帧请求进行画面的恢复。关键帧的请求方式分为三种：RTCP FIR反馈（Full intra frame request）、RTCP PLI 反馈（Picture Loss Indictor）或SIP Info消息，具体使用哪种可通过协商确定。



## 2. sdp 

在创建视频连接的SDP协议里面，会协商以上述哪种类型进行NACK重转。以webrtc为例，会协商两种NACK，一个rtp报文丢包重传的nack（nack后面不带参数，默认RTPFB）、PLI 视频帧丢失重传的nack。

```js
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
...
a=rtpmap:97 rtx/90000
...

```



## 3. 报文

```less
// RFC 4585: Feedback format.
Common packet format:
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|   FMT=1 |       PT=206      |          length           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  SSRC of packet sender                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  SSRC of media source                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:            Feedback Control Information (FCI)                 :
:                                                               :
```

- 
  V： 2bit， 目前固定为2

- P： 1bi，t padding
- FMT：5bit， Feedback message type。`FMT=1`；
- PT:   8bit， Payload type。`PT=206`;
- length
- ssrc sender
- ssrc media source

对于PLI，由于只需要通知发送关键帧，无需携带其他消息，所以FCI部分为空。对于FMT规定为1，PT规定为PSFB。



## 4. 抓包

## 

## 5. 关键帧请求场景

### 5.1 H264解码无sps,pps信息

#### H264SpsPpsTracker.PacketAction

modules/video_coding/h264_sps_pps_tracker.h

```cpp
class H264SpsPpsTracker {
 public:
  enum PacketAction { kInsert, kDrop, kRequestKeyframe };
  ...
  }
```



#### RtpVideoStreamReceiver2::OnReceivedPayloadData

video/rtp_video_stream_receiver2.cc

```cpp
void RtpVideoStreamReceiver2::OnReceivedPayloadData(
    rtc::CopyOnWriteBuffer codec_payload,
    const RtpPacketReceived& rtp_packet,
    const RTPVideoHeader& video) {
    ...
    video_coding::H264SpsPpsTracker::FixedBitstream fixed =
        tracker_.CopyAndFixBitstream(
            rtc::MakeArrayView(codec_payload.cdata(), codec_payload.size()),
            &packet->video_header);

    switch (fixed.action) {
      case video_coding::H264SpsPpsTracker::kRequestKeyframe:
        rtcp_feedback_buffer_.RequestKeyFrame();
        rtcp_feedback_buffer_.SendBufferedRtcpFeedback();
        ABSL_FALLTHROUGH_INTENDED;
      case video_coding::H264SpsPpsTracker::kDrop:
        return;
        }
     ...
  }
```



#### H264SpsPpsTracker::CopyAndFixBitstream

modules/video_coding/h264_sps_pps_tracker.cc

```cpp
H264SpsPpsTracker::FixedBitstream H264SpsPpsTracker::CopyAndFixBitstream(
    rtc::ArrayView<const uint8_t> bitstream,
    RTPVideoHeader* video_header) {
  RTC_DCHECK(video_header);
  RTC_DCHECK(video_header->codec == kVideoCodecH264);
  RTC_DCHECK_GT(bitstream.size(), 0);

  auto& h264_header =
      absl::get<RTPVideoHeaderH264>(video_header->video_type_header);

  bool append_sps_pps = false;
  auto sps = sps_data_.end();
  auto pps = pps_data_.end();

  for (size_t i = 0; i < h264_header.nalus_length; ++i) {
    const NaluInfo& nalu = h264_header.nalus[i];
    switch (nalu.type) {
      ...
      case H264::NaluType::kIdr: {
        // If this is the first packet of an IDR, make sure we have the required
        // SPS/PPS and also calculate how much extra space we need in the buffer
        // to prepend the SPS/PPS to the bitstream with start codes.
        if (video_header->is_first_packet_in_frame) {
          if (nalu.pps_id == -1) {
            RTC_LOG(LS_WARNING) << "No PPS id in IDR nalu.";
            return {kRequestKeyframe};
          }

          pps = pps_data_.find(nalu.pps_id);
          if (pps == pps_data_.end()) {
            RTC_LOG(LS_WARNING)
                << "No PPS with id << " << nalu.pps_id << " received";
            return {kRequestKeyframe};
          }

          sps = sps_data_.find(pps->second.sps_id);
          if (sps == sps_data_.end()) {
            RTC_LOG(LS_WARNING)
                << "No SPS with id << " << pps->second.sps_id << " received";
            return {kRequestKeyframe};
          }

          ...
        }
        break;
      }
      default:
        break;
    }
  }

 ...
  return fixed;
}
```



### 5.2 丢失包过多，NACK

在非常高的丢包率情况下，丢失的包太多，若都一一重传，将造成延时增大（等帧数据完整了才会去解码渲染），此时新来的数据也只能一直缓存，所以jitterbuffer大小也会不断增大，此时不如直接请求发送一个关键帧来得实际，以前丢的那些包都不管了，由于关键帧可以单独解码，所以不会造成解码端花屏马赛克现象。但是由于前面那些视频帧都丢弃了，此时生成的关键帧会与之前播放的视频存在不连贯性，所以画面变化大时会有轻微卡顿现象，相当于跳帧了。

```less
NackModule2::AddPacketsToNack
RtpVideoStreamReceiver2::RtcpFeedbackBuffer::RequestKeyFrame
```



#### NackModule2::AddPacketsToNack

```cpp
void NackModule2::AddPacketsToNack(uint16_t seq_num_start,
                                   uint16_t seq_num_end) {
  // Called on worker_thread_.
  // Remove old packets.
  auto it = nack_list_.lower_bound(seq_num_end - kMaxPacketAge);
  nack_list_.erase(nack_list_.begin(), it);

  // If the nack list is too large, remove packets from the nack list until
  // the latest first packet of a keyframe. If the list is still too large,
  // clear it and request a keyframe.
  uint16_t num_new_nacks = ForwardDiff(seq_num_start, seq_num_end);
  if (nack_list_.size() + num_new_nacks > kMaxNackPackets) {
    while (RemovePacketsUntilKeyFrame() &&
           nack_list_.size() + num_new_nacks > kMaxNackPackets) {
    }

    ////////////////////
    ////////////////////
    ////////////////////
    if (nack_list_.size() + num_new_nacks > kMaxNackPackets) {
      nack_list_.clear();
      RTC_LOG(LS_WARNING) << "NACK list full, clearing NACK"
                             " list and requesting keyframe.";
      // video/rtp_video_stream_receiver2.h
      // RtcpFeedbackBuffer 对象
      keyframe_request_sender_->RequestKeyFrame();
      return;
    }
    ////////////////////
    ////////////////////
    ////////////////////
  }

  for (uint16_t seq_num = seq_num_start; seq_num != seq_num_end; ++seq_num) {
    // Do not send nack for packets that are already recovered by FEC or RTX
    if (recovered_list_.find(seq_num) != recovered_list_.end())
      continue;
    NackInfo nack_info(seq_num, seq_num + WaitNumberOfPackets(0.5),
                       clock_->TimeInMilliseconds());
    RTC_DCHECK(nack_list_.find(seq_num) == nack_list_.end());
    nack_list_[seq_num] = nack_info;
  }
}
```



#### KeyFrameRequestSender 如何传入到NackModule2 

```cpp
VideoReceiveStream2::VideoReceiveStream2(
    TaskQueueFactory* task_queue_factory,
    TaskQueueBase* current_queue,
    RtpStreamReceiverControllerInterface* receiver_controller,
    int num_cpu_cores,
    PacketRouter* packet_router,
    VideoReceiveStream::Config config,
    ProcessThread* process_thread,
    CallStats* call_stats,
    Clock* clock,
    VCMTiming* timing)
    : ...
      rtp_video_stream_receiver_(worker_thread_,
                                 clock_,
                                 &transport_adapter_,
                                 call_stats->AsRtcpRttStats(),
                                 packet_router,
                                 &config_,
                                 rtp_receive_statistics_.get(),
                                 &stats_proxy_,
                                 &stats_proxy_,
                                 process_thread,
                                 this,     // NackSender
                                 ////////////////
                                 nullptr,  // Use default KeyFrameRequestSender
                                 /////////////////
                                 this,     // OnCompleteFrameCallback
                                 config_.frame_decryptor,
                                 config_.frame_transformer),
		...
//=========================================》

RtpVideoStreamReceiver2::RtpVideoStreamReceiver2(
    TaskQueueBase* current_queue,
    Clock* clock,
    Transport* transport,
    RtcpRttStats* rtt_stats,
    PacketRouter* packet_router,
    const VideoReceiveStream::Config* config,
    ReceiveStatistics* rtp_receive_statistics,
    RtcpPacketTypeCounterObserver* rtcp_packet_type_counter_observer,
    RtcpCnameCallback* rtcp_cname_callback,
    ProcessThread* process_thread,
    NackSender* nack_sender,
     ///////////
    KeyFrameRequestSender* keyframe_request_sender,
      ////////////
    video_coding::OnCompleteFrameCallback* complete_frame_callback,
    rtc::scoped_refptr<FrameDecryptorInterface> frame_decryptor,
    rtc::scoped_refptr<FrameTransformerInterface> frame_transformer)
      ...
      keyframe_request_sender_(keyframe_request_sender),
			...
      nack_module_(MaybeConstructNackModule(current_queue,
                                            config_,
                                            clock_,
                                            &rtcp_feedback_buffer_,
                                            &rtcp_feedback_buffer_)), // KeyFrameRequestSender rtcp_feedback_buffer_
			...
  
//=========================================》
        
std::unique_ptr<NackModule2> MaybeConstructNackModule(
    TaskQueueBase* current_queue,
    const VideoReceiveStream::Config& config,
    Clock* clock,
    NackSender* nack_sender,
    KeyFrameRequestSender* keyframe_request_sender) {
  if (config.rtp.nack.rtp_history_ms == 0)
    return nullptr;

  return std::make_unique<NackModule2>(current_queue, clock, nack_sender,
                                       keyframe_request_sender);
}


NackModule2::NackModule2(TaskQueueBase* current_queue,
                         Clock* clock,
                         NackSender* nack_sender,
                         KeyFrameRequestSender* keyframe_request_sender,
                         TimeDelta update_interval /*= kUpdateInterval*/)
```





### 5.3 解码前获取帧数据超时

接收端根据时间控制I帧请求是在解码前要包里面实现的。在指定时间内没有要到包，就发送I帧请求。
要解码的帧数据存放在FrameBuffer中，解码器解码时如果超过规定时间一直无法从FrameBuffer中获取解码数据，此时就会请求关键帧。

```less
VideoReceiveStream2::StartNextDecode
VideoReceiveStream2::HandleFrameBufferTimeout
VideoReceiveStream2::RequestKeyFrame
RtpVideoStreamReceiver2::RequestKeyFrame
RtpVideoStreamReceiver2::RtcpFeedbackBuffer::RequestKeyFrame
```



#### VideoReceiveStream2::StartNextDecode

video/video_receive_stream2.cc

```cpp
void VideoReceiveStream2::StartNextDecode() {
  // Running on the decode thread.
  TRACE_EVENT0("webrtc", "VideoReceiveStream2::StartNextDecode");
  frame_buffer_->NextFrame(
      GetMaxWaitMs(), keyframe_required_, &decode_queue_,
      /* encoded frame handler */
      [this](std::unique_ptr<EncodedFrame> frame, ReturnReason res) {
        RTC_DCHECK_EQ(frame == nullptr, res == ReturnReason::kTimeout);
        RTC_DCHECK_EQ(frame != nullptr, res == ReturnReason::kFrameFound);
        RTC_DCHECK_RUN_ON(&decode_queue_);
        if (decoder_stopped_)
          return;
        if (frame) {
          HandleEncodedFrame(std::move(frame));
        } else {
          int64_t now_ms = clock_->TimeInMilliseconds();
          worker_thread_->PostTask(ToQueuedTask(
              task_safety_, [this, now_ms, wait_ms = GetMaxWaitMs()]() {
                RTC_DCHECK_RUN_ON(&worker_sequence_checker_);
                HandleFrameBufferTimeout(now_ms, wait_ms);
              }));
        }
        StartNextDecode();
      });
}
```



#### VideoReceiveStream2::HandleFrameBufferTimeout

video/video_receive_stream2.cc

```cpp
void VideoReceiveStream2::HandleFrameBufferTimeout(int64_t now_ms,
                                                   int64_t wait_ms) {
  // Running on |worker_sequence_checker_|.
  absl::optional<int64_t> last_packet_ms =
      rtp_video_stream_receiver_.LastReceivedPacketMs();

  // To avoid spamming keyframe requests for a stream that is not active we
  // check if we have received a packet within the last 5 seconds.
  const bool stream_is_active =
      last_packet_ms && now_ms - *last_packet_ms < 5000;
  if (!stream_is_active)
    stats_proxy_.OnStreamInactive();

  if (stream_is_active && !IsReceivingKeyFrame(now_ms) &&
      (!config_.crypto_options.sframe.require_frame_encryption ||
       rtp_video_stream_receiver_.IsDecryptable())) {
    RTC_LOG(LS_WARNING) << "No decodable frame in " << wait_ms
                        << " ms, requesting keyframe.";
    RequestKeyFrame(now_ms);
  }
}
```



### 5.4 解码出错

当解码器返回解码失败，或者解码器返回请求关键帧结果时，需要请求关键帧。

```less
VideoReceiveStream2::HandleEncodedFrame
VideoReceiveStream2::HandleKeyFrameGeneration
VideoReceiveStream2::RequestKeyFrame
RtpVideoStreamReceiver2::RequestKeyFrame
```



#### VideoReceiveStream2::HandleEncodedFrame

```cpp
void VideoReceiveStream2::HandleEncodedFrame(
    std::unique_ptr<EncodedFrame> frame) {
  // Running on |decode_queue_|.
  int64_t now_ms = clock_->TimeInMilliseconds();

  // Current OnPreDecode only cares about QP for VP8.
  int qp = -1;
  if (frame->CodecSpecific()->codecType == kVideoCodecVP8) {
    if (!vp8::GetQp(frame->data(), frame->size(), &qp)) {
      RTC_LOG(LS_WARNING) << "Failed to extract QP from VP8 video frame";
    }
  }
  stats_proxy_.OnPreDecode(frame->CodecSpecific()->codecType, qp);

  bool force_request_key_frame = false;
  int64_t decoded_frame_picture_id = -1;

  const bool keyframe_request_is_due =
      now_ms >= (last_keyframe_request_ms_ + max_wait_for_keyframe_ms_);

  int decode_result = video_receiver_.Decode(frame.get());
  if (decode_result == WEBRTC_VIDEO_CODEC_OK ||
      decode_result == WEBRTC_VIDEO_CODEC_OK_REQUEST_KEYFRAME) {
    keyframe_required_ = false;
    frame_decoded_ = true;

    decoded_frame_picture_id = frame->id.picture_id;

    if (decode_result == WEBRTC_VIDEO_CODEC_OK_REQUEST_KEYFRAME)
      force_request_key_frame = true;
  } else if (!frame_decoded_ || !keyframe_required_ ||
             keyframe_request_is_due) {
    keyframe_required_ = true;
    // TODO(philipel): Remove this keyframe request when downstream project
    //                 has been fixed.
    force_request_key_frame = true;
  }

  bool received_frame_is_keyframe =
      frame->FrameType() == VideoFrameType::kVideoFrameKey;

  worker_thread_->PostTask(ToQueuedTask(
      task_safety_,
      [this, now_ms, received_frame_is_keyframe, force_request_key_frame,
       decoded_frame_picture_id, keyframe_request_is_due]() {
        RTC_DCHECK_RUN_ON(&worker_sequence_checker_);

        if (decoded_frame_picture_id != -1)
          rtp_video_stream_receiver_.FrameDecoded(decoded_frame_picture_id);

        HandleKeyFrameGeneration(received_frame_is_keyframe, now_ms,
                                 force_request_key_frame,
                                 keyframe_request_is_due);
      }));

  if (encoded_frame_buffer_function_) {
    frame->Retain();
    encoded_frame_buffer_function_(WebRtcRecordableEncodedFrame(*frame));
  }
}
```



#### VideoReceiveStream2::HandleKeyFrameGeneration

```cpp
void VideoReceiveStream2::HandleKeyFrameGeneration(
    bool received_frame_is_keyframe,
    int64_t now_ms,
    bool always_request_key_frame,
    bool keyframe_request_is_due) {
  // Running on worker_sequence_checker_.

  bool request_key_frame = always_request_key_frame;

  // Repeat sending keyframe requests if we've requested a keyframe.
  if (keyframe_generation_requested_) {
    if (received_frame_is_keyframe) {
      keyframe_generation_requested_ = false;
    } else if (keyframe_request_is_due) {
      if (!IsReceivingKeyFrame(now_ms)) {
        request_key_frame = true;
      }
    } else {
      // It hasn't been long enough since the last keyframe request, do nothing.
    }
  }

  if (request_key_frame) {
    // HandleKeyFrameGeneration is initated from the decode thread -
    // RequestKeyFrame() triggers a call back to the decode thread.
    // Perhaps there's a way to avoid that.
    RequestKeyFrame(now_ms);
  }
}
```



### VideoReceiveStream2::RequestKeyFrame

video/video_receive_stream2.cc

```cpp
void VideoReceiveStream2::RequestKeyFrame(int64_t timestamp_ms) {
  // Running on worker_sequence_checker_.
  // Called from RtpVideoStreamReceiver (rtp_video_stream_receiver_ is
  // ultimately responsible).
  // RtpVideoStreamReceiver2 rtp_video_stream_receiver_
  rtp_video_stream_receiver_.RequestKeyFrame();
  decode_queue_.PostTask([this, timestamp_ms]() {
    RTC_DCHECK_RUN_ON(&decode_queue_);
    last_keyframe_request_ms_ = timestamp_ms;
  });
}
```



### RtpVideoStreamReceiver2::RequestKeyFrame

video/rtp_video_stream_receiver2.cc

```cpp
void RtpVideoStreamReceiver2::RequestKeyFrame() {
  RTC_DCHECK_RUN_ON(&worker_task_checker_);
  // TODO(bugs.webrtc.org/10336): Allow the sender to ignore key frame requests
  // issued by anything other than the LossNotificationController if it (the
  // sender) is relying on LNTF alone.
  if (keyframe_request_sender_) {
    keyframe_request_sender_->RequestKeyFrame();
  } else {
    rtp_rtcp_->SendPictureLossIndication();
  }
}
```



### RtpVideoStreamReceiver2::RtcpFeedbackBuffer::RequestKeyFrame

video/rtp_video_stream_receiver2.cc

```cpp
void RtpVideoStreamReceiver2::RtcpFeedbackBuffer::RequestKeyFrame() {
  RTC_DCHECK_RUN_ON(&worker_task_checker_);
  request_key_frame_ = true;
}
```





## 6. 代码导读

### 6.1 Feedback 添加

media/engine/webrtc_video_engine.cc

```cpp
void AddDefaultFeedbackParams(VideoCodec* codec,
                              const webrtc::WebRtcKeyValueConfig& trials) {
  // Don't add any feedback params for RED and ULPFEC.
  if (codec->name == kRedCodecName || codec->name == kUlpfecCodecName)
    return;
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamRemb, kParamValueEmpty));
  codec->AddFeedbackParam(
      FeedbackParam(kRtcpFbParamTransportCc, kParamValueEmpty));
  // Don't add any more feedback params for FLEXFEC.
  if (codec->name == kFlexfecCodecName)
    return;
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamCcm, kRtcpFbCcmParamFir));
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamNack, kParamValueEmpty));
  // pli feedback
  codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamNack, kRtcpFbNackParamPli));
  if (codec->name == kVp8CodecName &&
      IsEnabled(trials, "WebRTC-RtcpLossNotification")) {
    codec->AddFeedbackParam(FeedbackParam(kRtcpFbParamLntf, kParamValueEmpty));
  }
}
```



### 6.2 PLI FB 发送

参考上面【章节5】



### 6.3 ??? 处理 PLI 请求



## 参考

[RFC 4585: Extended RTP Profile for Real-time Transport Control Protocol (RTCP)-Based Feedback (RTP/AVPF)](https://tools.ietf.org/html/rfc4585)

https://blog.csdn.net/CrystalShaw/article/details/81218394

[webrtc QOS方法十二（接收端IDR帧请求）](https://blog.csdn.net/CrystalShaw/article/details/124614098)

[八 WebRTC 关键帧请求PLI与FIR](https://blog.csdn.net/zrjliming/article/details/120412631)
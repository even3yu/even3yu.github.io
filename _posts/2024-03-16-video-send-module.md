---
layout: post
title: Video 流程中的模块介绍
date: 2024-03-16 15:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc video stream
---


* content
{:toc}

---


![img]({{ site.url }}{{ site.baseurl }}/images/video-send-module.assets/flow-data-sequence.png)



![img]({{ site.url }}{{ site.baseurl }}/images/video-send-module.assets/class.png)



![img]({{ site.url }}{{ site.baseurl }}/images/video-send-module.assets/class2.png)

## 1. VideoSendStream

Video模块 (video/video_send_stream.h，video/video_send_stream.cc)

1. 创建VideoStreamEncoderInterface. 用于对source进行编码。       
2. 创建VideoSendStreamImpl，用于接收VideoStreamEncoderInterface编码后的视频数据。  
3. 初始化编码器相关参数，设置编码器输入，VideoSourceInterface提供需要编码的视频流原始数据。 
4. 启动，停止流控制，Start/Stop；
5. 处理接收端反馈的rtcp消息，（分发到VideoSendStreamImpl进行处理）VideoSendStream::DeliverRtcp



## 2. VideoSendStreamImpl

Video模块 (video_send_stream_impl.h,video_send_stream_impl.cc)

1. 创建RtpVideoSenderInterface( RtpVideoSender ) 用于编码后的视频帧处理，（MediaTransportInterface如果初始化的时候，使用此接口，则使用它进行数据的发送，这个不用使用webrtc内部的发送控制，反馈逻辑）。	
2. 注册接收VideoStreamEncoderInterface::EncoderSink的编码结果，OnEncodedImage。
3. 编码起码率控制逻辑，设置编码器开始码率，当前希望码率（接收码率分配策略的控制BitrateAllocatorInterface）。
4. 启动，停止流控制，进行编码器的控制。
5. 包含EncoderRtcpFeedback（继承于 RtcpIntraFrameObserver，RtcpLossNotificationObserver）成员，最终会传递到RTCPReceiver 这个类里，从这里转发到EncoderRtcpFeedback。 响应VideoStreamEncoderInterface的关键帧请求(EncoderRtcpFeedback::OnReceivedIntraFrameRequest)，丢包信息(EncoderRtcpFeedback::OnReceivedLossNotification)，分发到VideoStreamEncoderInterface中。   
6. 处理接收端反馈的rtcp消息，（分发到RtpVideoSenderInterface进行处理） VideoSendStreamImpl::DeliverRtcp。



## 3. RtpVideoSender: public RtpVideoSenderInterface

Call模块 (rtp_video_sender.h,rtp_video_sender.cc)

1. 创建n组。RTPSenderVideo，RtpRtcp，用内部RtpStreamSender包装到一起的，`std::vector<webrtc_internal_rtp_video_sender::RtpStreamSender> rtp_streams_`。
2. 通过构造参数RtpConfig配置RtpRtcp，RTPSenderVideo。CreateRtpStreamSenders 这里创建了这些对象。
3. 根据配置，决定是否开启fec, nack，使用FlexFec。  成员包含了FecController，用于生成fec的控制参数，
4. 接收编码器的码率分配通知，分发到Rtprtcp模块用于反馈给接收端(RtcpXr扩展消息)
5. 接收分配给这个流的码率，进行fec的计算，计算fec保护使用的码率，用于VideoSendStreamImpl码率计算
6. 接收Rtcp twcc的反馈，然后分发到RTPSender请求重发，或seq确认。RtpVideoSender::OnPacketFeedbackVecto。
7. 处理接收端反馈的rtcp消息，（分发到RtpRtcp模块）RtpVideoSender::DeliverRtcp。



## 4. RtpStreamSender

包括RTPSenderVideo,ModuleRtpRtcpImpl。



## 5. RTPSenderVideo

rtprtcp模块 (rtp_sender_video.h, rtp_sender_video.h).

 1. 视频帧拆RTP包（使用RtpPacketizer）  
 2. rtp部分扩展头添加
     PlayoutDelay ： 接收端播放时候使用的延迟控制，相当于jittbuffer控制。
     VideoRatation： 视频旋转角度控制。 
     ColorSapce ：   使用的颜色空间， 颜色还原公式（yuv 601， 709）
     Contenttype ：  使用图像类型，是否屏幕采集的
     FrameMark ：    区分svc编码不同层的信息，
     VideoTiming：   视频调试信息，视频的采集时间，编码时间，一系列时间
     Genericxxx：    保存的数据自定义加密等信息
 3. 视频数据加密，不同于srtp ,这个使用外部设置接口现对视频数据进行第一次加密
 4. 根据相关fec,nakc设置 ,是否保存发送的seq,用于外部请求是否可以进行重发等
 5. 根据设置，使用flexfec，还是 RED,ulp ,还是不开启fec进行发送，如果开启fec，则生成fec包，适当时机发送fec包。（RTPSender发送数据）

## 6. RTPSender

rtprtcp模块 (modules/rtp_rtcp/source/rtp_sender.cc).

 1. 使用PacerSender或meidiaTranpot进行发送rtp包逻辑
 2. rtp部分扩展头添加
    TransmissionOffset：   采集时间点到发送时间点的延迟值
    AbsoluteSendTime :     remb 码率估算的数据（接收端估算）
    TramsportSeqiemceMumber ： 发送端估算（Twcc），现在支持2个版本，接收端定时反馈，发送端请求反馈
    RtpMid：       对应sdp中mid，没看什么作用
    RtpStreamId：  对应RID，应该是simulate流进行区分不同码率的标志
    

  3.保存需要发送rtp包到列表，



## 7. PacedSender

pacing模块 (paced_sender.h, paced_sender.cc).

  1. 保存待发送数据包队列，所有流一个队列，根据当前设置的码率，网络状态定时进行发送。 （节拍器）
  2. 音频rtp包优先级高（不受网络阻塞控制，所以在真的阻塞情况下 音频丢包率回高），
  3. padding包控制，探测网络带宽回用到。 由外部启动的。



## 8. PacketRouter： public PacedSender::PacketSender

pacing模块 (packet_touter.h, packet_touter.cc).

   1.  保存多个流的RtpRtcp模块，用于控制它进行数据发送
   2.  PacedSender 触发TimeToSendPacket，然后派发给RtpRtcp模块可以进行对应seq Rtp包的发送了。
   3.  remb和twcc反馈的发送控制



## 9. ModuleRtpRtcpImpl: public RtpRtcp

rtp_rtcp模块 (rtp_rtcp_impl.h, rtp_rtcp_impl.cc).

1.  创建 RTPSender，RTCPSender ，RTCPReceiver
2.  设置 rtp相关参数
3.  TimeToSendPacket  控制RTPSender使用RTPSender.Transport进行实质的数据发送，从保存的列表中取出数据进行发送。


> 注意：
>
> RTPSender.SendToNetwork 保存 rtp包到列表中。
> RTPSender.TimeToSendPacket 从列表中查找到rtp包，添加部分扩展头，然后通过外部设置Transport发送到网络上
>
> 就是如果使用pacer数据先进入 RTPSender转一圈，到pacer,然后又回来进行真正的发送。



## 10. RtpSenderContext

包括RtpPacketHistory，RtpSenderEgress，RtpSenderEgress::NonPacedPacketSender，RTPSender。



## 参考

1. [webrtc(m76) video sender总结](https://blog.csdn.net/qq_16135205/article/details/101114575)
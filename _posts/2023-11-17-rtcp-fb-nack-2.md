---
layout: post
title: rtcp nack
date: 2023-11-17 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc nack rtcp
---


* content
{:toc}

---

## 1. 前言

NACK 全称为 `Negative Acknowledgment Packet`，是一种对 RTP 数据传输层进行反馈的 RTCP 包，包类型为 205，反馈类型为 1。相对于 TCP 的 ACK 接收确认，NACK 则是未接收确认。

丢包重传(NACK)是抵抗网络错误的重要手段。NACK在接收端检测到数据丢包后，发送NACK报文到发送端；发送端根据NACK报文中的序列号，在发送缓冲区找到对应的数据包，重新发送到接收端。NACK需要发送端，发送缓冲区的支持。

NACK 模块总体的发送策略为：对于每一个因为不连续而被判为丢失的包，首次都是基于序列号**立即发送** nack 包以请求重传，之后则都是基于时间序列，**周期性批量处理**nack_list，并根据距离上次发送的时间间隔是否已经超过一个 rtt 来决定是否发送 nack 包。



## 2. 工作流程

1. 发送端发送rtp，

2. 到达接收端时，发现丢包，接收端发送nack请求，

3. 发送端会从历史队列中取出数据重发。


![300]({{ site.url }}{{ site.baseurl }}/images/rtcp-fb-nack-2.assets/nack.png)



### 2.1 接收端 nack 三个缓存list

#### nack_list_

 用于记录已丢包的信息,seq 即为list key。

- insert :
  insert : AddPacketsToNack()会判断包的连续性,相应的丢包序列如果不在recover list里面就会插入
- erase ：
  1.序列号距离当前收到的序列号过旧的包kMaxPacketAge(10000)
  2.nack_list 大小 + 即将插入的nack 序列数量如果超过kMaxNackPackets(1000) 就会清理掉关键帧之前的nack,循环直至size 小于1000 或者 已经到了最新关键帧
  3.如果经步骤2 nack_list大小仍然超过了nackkMaxNackPackets(1000) 会全部清理掉,并重新请求关键帧
  4.收到乱序的包, 可能是抖动过来的 或者 后面恢复过来的包.
  5.发送超过10次仍然没有收到重传回来的包.

#### keyframe_list_

记录关键帧序列号,可用于后面清理比关键帧老的过旧nack。

- insert :
  只需要判断是否是关键帧或恢复过来的包即可插入
- erase:
  三个缓存都相同的删除点,清理序列号距离当前收到的序列号过旧的包kMaxPacketAge(10000) 例如[6,7,…,100007],6即被清理.

#### recovered_list_

 用于记录从RTX或FEC恢复过来的包，存储重传包(防止需要重传的包已经在重传包中了，重传包有可能优先过来)

- insert :
  只需要判断是否是关键帧或恢复过来的包即可插入
- erase:
  三个缓存都相同的删除点,清理序列号距离当前收到的序列号过旧的包kMaxPacketAge(10000) 例如[6,7,…,100007],6即被清理.



### 2.2 nack 重传请求两种发送方式

1. kSeqNumOnly : 开启nack模块后， nack会检查接收到packet的序列号，如果序列号连续性中断即认为是丢包了。 如下例子，上一次最新收到的包序列号为38，当前新收到的序列号为41，那么[39,40]就判定为是丢掉了，会立刻发送这组[39,40]的nack重传请求。

```less
newest_seq_num_:36 seq_num:37 is_keyframe:0 is_recovered: 0 
newest_seq_num_:37 seq_num:38 is_keyframe:0 is_recovered: 0 
newest_seq_num_:38 seq_num:41 is_keyframe:0 is_recovered: 0 
newest_seq_num_:41 seq_num:42 is_keyframe:0 is_recovered: 0 
newest_seq_num_:42 seq_num:43 is_keyframe:0 is_recovered: 0
```

2. kTimeOnly : nack 模块创建后会启动一个定时任务，默认周期kUpdateInterval(20ms), 这个周期任务会调用`GetNackBatch(kTimeOnly)` 从`nack_list`里面获取满足发送条件的seq，批量发送nack重传请求。

```cpp
repeating_task_ = RepeatingTaskHandle::DelayedStart(
    TaskQueueBase::Current(), kUpdateInterval,
    [this]() {
      std::vector<uint16_t> nack_batch = GetNackBatch(kTimeOnly);
      if (!nack_batch.empty()) {
        nack_sender_->SendNack(nack_batch, false);
      }
    });
```

3. **kSeqNumOnly模式是在接收packet的时候触发一次，并且只发送一次，即第一次，之后如果仍然没有收到重传回来的包就通过kTimeOnly定时任务方式继续请求重传。**



## 3. 报文格式

NACK 报文的定义在 [[rfc4585\]](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc4585) 文档中定义。

RTCP 的反馈报文包头定义如下，FMT 和 PT 决定了该报文的类型，FCI 则是该类型报文的具体负载：

![img]({{ site.url }}{{ site.baseurl }}/images/rtcp-fb-nack-2.assets/nack-reprort.webp)

协议规定的 NACK 反馈报文的 PT= 205，FMT=1，FCI 的格式如下（可以附带多个 FCI，通过 header 的 length 字段来标示其长度）：

```less
   0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |            PID                |             BLP               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

这里的设计比较巧妙，它可以一次性携带多个连续的数据包的丢包情况：

- PID（Packet identifier），即为丢失的 RTP 数据包的序列号
- BLP（Bitmao of Lost Packets），通过掩码的方式指出了接下来 16 个数据包的丢失情况



## 4. sdp

在创建视频连接的SDP协议里面，会协商以上述哪种类型进行NACK重转。以webrtc为例，会协商两种NACK，一个rtp报文丢包重传的nack（nack后面不带参数，默认RTPFB）、PLI 视频帧丢失重传的nack。

```js
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 nack goog-remb
a=rtcp-fb:96 nack transport-cc
a=rtpmap:97 rtx/90000
...

```



## 5. !!! 抓包

ack rtcp报文格式如上图所示，pt=205。Packet identifier(PID) 为丢包起始参考值，Bitmap of Lost Packets(BLP)为16位的bitmap，对应为1的为表示丢包数据，具体如下抓包分析：
![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/rtcp-fb-nack-2.assets/nack-wireshark.png)

- Packet identifier(PID)为176。
- Bitmap of Lost Packets(BLP)：0x6ae1。
- 解析的时候需要按照小模式解析，0x6ae1对应二进制：110101011100001倒过来看1000 0111 0101 0110。
  按照1bit是丢包，0bit是没有丢包解析，丢失报文序列号分别是：176 177 182 183 184 186 188 190 191与wireshark解析一致，当然pid和blp可以有多个。



## 参考

1. [webrtc源码分析 nack详解](https://blog.csdn.net/liuhongxiangm/article/details/123231033)
2. [抓个包，一起来看看NACK是怎么回事？](https://blog.csdn.net/epubcn/article/details/83827849?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-13-83827849-blog-123231033.235^v38^pc_relevant_sort_base2&spm=1001.2101.3001.4242.8&utm_relevant_index=16)
3. https://blog.csdn.net/qq_17308321/article/details/129948951
4. [webrtc QOS笔记四 Nack机制浅析](https://blog.csdn.net/qq_17308321/article/details/129948951?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-129948951-blog-125192641.235^v38^pc_relevant_sort_base2&spm=1001.2101.3001.4242.2&utm_relevant_index=4)




## ??? 关于rtcp的一些疑问

> nack是rtcp数据，属于控制信息。信令在整个传输中流量的占比应该严格控制，目前网络的做法，rtcp信令可能只占到整体流量的5%左右，xx公司则是会达到近10%的占比（包含了各类带宽估计的feedback），而某著名媒体服务商也达到了12%。每个nack数据包最大也只为网络传输的MTU大小（1500字节），如果按20ms的定时要nack，传输数据量每秒50个包，每个包可以携带约300 ~ 400个seq（xx公司是347个）。因此计算得到一秒最大可以要将近15000 ~ 20000个数据包。往往我们1.2m的码率每秒也只有500 - 600个包，因此我们在整个传输中间我们向上行索要seq号有部分是重复的。
>   经验归纳出来：
>   ■ 在带宽允许的情况下，要保证能补就补，尽量延迟超时机制；
>   ■ 在索要seq的数量上做一个最大的控制量，某著名媒体服务商在70%丢包的环境下会索要最高达27次。
>
> 原文链接：https://blog.csdn.net/qw225967/article/details/123206637




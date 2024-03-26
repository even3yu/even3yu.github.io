---
layout: post
title: rtcp 组包
date: 2023-11-04 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtcp
---


* content
{:toc}

---

## 组包（Packing）问题

**RTCP包不会单独发送，而是组包成一个复合数据包进行传输**。

- 生成复合RTCP包的参与者是活跃的数据**发送方，那么该复合包必须以RTCP  SR包开始。否则必须从RTCP  RR包开始**。即使还没有发送或接收数据，这也是正确的，在这种情况下，SR/RR包不会包含接收方的报告块（包头字段RC为0）。
- 另一方面，**如果从多个源接收数据，并且报告太多，导致无法放入一个SR/RR包，则复合后的数据应以一个SR/RR包开始，后面在跟着多个RR包**。跟在SR/RR包后面的是一个SDES包。这个包必须包含一个CNAME条目。它可能包含其他的 条目。包含其他（非CNAME）SDES条目的频度由使用中的RTP配置文件决定。Bye包必须做为最后一个数据包发送。要发送的其他RTCP包可以按 任何顺序。这些严格的排序规则，旨在使数据包的校验更容易，因为错误定向的数据包， 大概率不会满足这些约束。

在生成复合RTCP包时，一个潜在的问题就是如何处理大量活跃发送者的会话。如果存在超 过31个活跃的发送者，那么有必要在复合包中增加额外的RR包。这可以根据需要重复此过 程，直到达到MTU的上限。如果发送方太多，以致于接收方报告不能被MTU容纳，则必须 忽略某些发送方的接收报告。如果出现这种情况，那么被忽略的报告，应该在生成的下一 个复合包中被包含（要求接收方跟踪每个间隔中报告的源）。

有时候需要将一个复合RTCP包填充并超出其原始大小。在这种场景下，填充只是添加到复 合包中的最后一个RTCP包中，P位（P bit）在最后一个包中被设置。

![img](https://img-blog.csdnimg.cn/ef92ee18d3dc4b798ec01dd9baeff200.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oGL5LiK6LGG5rKZ5YyF,size_20,color_FFFFFF,t_70,g_se,x_16)



https://blog.csdn.net/Doubao93/article/details/121622858
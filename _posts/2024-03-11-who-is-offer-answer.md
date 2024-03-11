---
layout: post
title: sdp协商，哪一方作为offer端，哪一方作为answer端
date: 2024-03-11 15:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc faq sdp
---


* content
{:toc}

---


一般来说，推流方先发起 Offer，接收方给 Answer。比如客户端推流到 SFU，客户端发起 Offer 推流，SFU 给客户端 Answer，客户端将流推到 SFU，SFU 再转发给其他客户端。Licode 和 Janus 都是这种做法，这种方式下，如果客户端需要拉取其他的客户端的流，一般需要使用另外的 PeerConnection，接收 SFU 的 Offer，生成 Answer 后回应给 SFU。

不过，推流方发起 Offer 不是必须的，接收方也可以给 Offer，推流方给 Answer。比如 MediaSoup 这种 SFU，客户端先给一个 Offer 给 SFU，SFU 只是检查这个 Offer 中的媒体特性，然后 SFU 会生成 Offer（包含会议中的其他客户端的流，如果没有人则没有 SSRC）给客户端，客户端发送 Answer 给 SFU。这种方式的好处是其他客户端加入，以及流的变更（比如关闭视频打开视频时），都可以使用 Reoffer，也就是统一由 SFU 发起新的 Offer，客户端响应，SFU 和客户端的交互模式只有一种。



## 参考

https://blog.csdn.net/VideoCloudTech/article/details/110092546


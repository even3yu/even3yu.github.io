---
layout: post
title: webrtc sdp 协商流程
date: 2023-08-23 14:55:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---

![sdp-sequence]({{ site.url }}{{ site.baseurl }}/images/sdp-exchange.assets/sdp-sequence.png)

1. caller， createOffer， 这是个异步的，需要等待回调，CreateSessionDescriptionObserver::OnSuccess, 得到JsepSessionDescription ，就是描述sdp的对象
2. caller，setLocalDescription
3. caller，通过信令通道，发送sdp 到callee
4. callee，setRemoteDescription
5. callee，createAnswer
6. callee，setLocalDescription
7. callee，通过信令通道，发送sdp 到caller
8. caller，setRemoteDescription
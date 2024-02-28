---
layout: post
title: webrtc current_local_description_ pending_local_description_ current_remote_description_ pending_remote_description_ 赋值时机
date: 2024-02-28 03:00:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc sdp
---


* content
{:toc}

---

## current_local_description_ pending_local_description_ current_remote_description_ pending_remote_description_ 赋值时机

| descrption                  | setLocalDescription（caller）                  | setRemoteDescription（caller）                               | setLocalDescription（callee）                                | setRemoteDescription（callee）                      |
| --------------------------- | ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------- |
| current_local_description_  |                                                | 4.1 收到对方的anwersdp， ApplyRemoteDescription赋值          | 3.1 创建ansersdp，ApplyLocalDescription赋值                  |                                                     |
| pending_local_description_  | 1. 本端createOffer，ApplyLocalDescription 赋值 | 4.3 收到对方的anwersdp，ApplyLocalDescription，pending_local_description_转正 | 3.2 创建ansersdp，ApplyLocalDescription 清空                 |                                                     |
| current_remote_description_ |                                                | 4.2 收到对方的anwersdp， ApplyRemoteDescription清空          | 3.3 创建ansersdp，ApplyLocalDescription，pending_remote_description_转正 |                                                     |
| pending_remote_description_ |                                                |                                                              |                                                              | 2. 远端收到createOffer，ApplyRemoteDescription 赋值 |

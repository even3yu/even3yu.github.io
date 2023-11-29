---
layout: post
title: Windows的AudioDeviceModule的两种实现方式
date: 2023-07-19 23:35:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc audio
---


* content
{:toc}

---

## Windows的AudioDeviceModule的两种实现方式

目前Windows平台关于音频设备的有两种实现方式，分别是AudioDeviceWindowsCore （继承于AudioDeviceGeneric）和WindowsAudioDeviceModule（继承于AudioDeviceModule）。



### 1. AudioDeviceWindowsCore

特点是将音频输入与输入混合在一起，导致出现的一个显著缺陷是：只有当音频输入与输出设备都正常时，音频功能才能正常进行。如没有扬声器设备时，即使拥有麦克风设备也无法有效地输入音频数据。

如果程序没有可以在无某一音频设备的情况下正常运行的要求，当前可以选择AudioDeviceWindowsCore。

### 2. WindowsAudioDeviceModule

实际上Windows平台上有一个更新的实现，它直接实现了AudioDeviceModule类，没有AudioDeviceGeneric。它是WindowsAudioDeviceModule！我们目前使用这个对象。它重新设计了设备定义，将输入和输出分别进行了定义，分别为：AudioOutput 和 AudioInput 接口。

特点是将音频输入与输出分开进行，二者互不相干，弥补了前者的缺陷。但作为较新的实现，WindowsAudioDeviceModule中一些接口还并未被实现。如音量控制和设备端AEC、AGC、NSI可用性检查与设置。

在实际情况中，WindowsAudioDeviceModule并不稳定，它的实现不仅不完整，经过测试它还有一些致命bug（频繁插拔音频设备使WindowsAudioDeviceModule陷入重启中丧失功能，或者在初始化时崩溃）。但是使用WindowsAudioDeviceModule是未来趋势。

### 3. 优缺点

| 类型                     | 缺点                                                         | 优点                                             |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------ |
| AudioDeviceWindowsCore   | 将音频输入与输入混合在一起，导致出现的一个显著缺陷是：只有当音频输入与输出设备都正常时，音频功能才能正常进行。如没有扬声器设备时，即使拥有麦克风设备也无法有效地输入音频数据 | 这个是目前rtc 默认使用的，不用改代码（优点勉强） |
| WindowsAudioDeviceModule | 不稳定                                                       | 将音频输入与输出分开进行，二者互不相干           |

[音频管理模块AudioDeviceModule解读](https://zhuanlan.zhihu.com/p/296684134)
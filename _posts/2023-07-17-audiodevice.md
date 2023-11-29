---
layout: post
title: AudioDeviceModule
date: 2023-07-17 24:08:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc audio
---


* content
{:toc}

---

## AudioDeviceModule

WebRTC的音频设备类主要包含AudioDeviceGeneric和AudioDeviceModule。

- AudioDeviceGeneric代表一个输入输出设备，提供数据的真正采集和播放，调整音量等;  AudioDeviceGeneric是个抽象的，具体的是由各个平台的实现，AudioDeviceDummy，AudioDeviceWindowsCore，AudioDeviceTemplate，AudioDeviceLinuxALSA，AudioDeviceLinuxPulse，AudioDeviceIOS，AudioDeviceMac。
- AudioDeviceModule是音频设备模块，它实际上内部会有一个AudioDeviceGeneric设备的引用（成员），然后把设备相关的调用转到设备对象上。它主要的目的是WebRTC音频的抽象接口定义，并完成不同平台的设备对象的创建。



### 1. AudioDeviceModuleImpl 和 AudioDeviceGeneric关系图


![audio-AuidoDevieModule-AudioDevice.drawio]({{ site.url }}{{ site.baseurl }}/images/audio-AuidoDevieModule-AudioDevice.drawio.png)


1. AudioDeviceModuleImple，对外提供的主要实现类，主要是管理AudioDeviceGeneric，AudioDeviceBuffer。在初始化ADM对象时候的CreatePlatformSpecificObjects()方法时候创建不同的平台相关音频设备。提供采音放音接口，音量控制，静音控制等。

2. AudioDeviceGeneric，可以看到有各个平台的不同实现，继承了AudioDeviceGeneric。

   | 平台                          | 说明                                                         |
   | ----------------------------- | ------------------------------------------------------------ |
   | AudioDeviceGeneric            | 硬件接口类，采音和放音、音量控制等等， 被不同的系统实现集成。 |
   | AudioDeviceLinuxALSA          | 继承AudioDeviceGeneric类， 主要调用AudioMixerManagerLinuxALSA（linux下alsa声卡驱动封装类） |
   | AudioDeviceLinuxPulse         | 继承AudioDeviceGeneric类， 主要调用AudioMixerManagerLinuxPulse（linux下pulse声卡驱动封装类） |
   | AudioDeviceMac                | 继承AudioDeviceGeneric类， 主要调用AudioMixerManagerMac（max下声卡驱动封装类） |
   | AudioDeviceWindowsCore        | 继承AudioDeviceGeneric， windows下的两套实现类               |
   | AudioDeviceIOS                | 继承AudioDeviceGeneric类， iOS下的实现类                     |
   | OpenSlesInput, OpenSlesOutput | Android下的opensles的实现封装类                              |
   | AudioRecordJni, AudioTrackJni | android下的JNI实现类，放音和采集动作有JAVA层实现             |
   | AudioDeviceTemplate           | 模板类，继承AudioDeviceGeneric类，用于采集和放音分开的类，主要用于Android。 |

   

3. AudioDeviceBuffer， 保存和Device的交互的音频数据。

   

### 2. AudioDeviceModule 功能介绍

![img]({{ site.url }}{{ site.baseurl }}/images/5c85b0fa2e57831467de36d674e701bb.png)

从上面的架构图可以看出 AudioDeviceModule 定义了 ADM 相关的所有行为（上图只列出了部分核心,更详细的请参考源码中的完整定义）。从 AudioDeviceModule 的定义可以看出 AudioDeviceModule 的主要职责如下：

- 初始化音频播放/采集设备；
- 启动音频播放/采集设备；
- 停止音频播放/采集设备；
- 在音频播放/采集设备工作时，对其进行操作（例如：Mute , Adjust Volume）；
- 平台内置 3A 开关的调整（主要是针对 Android 平台）；
- 获取当前音频播放/采集设备各种与此相关的状态（类图中未完全体现，详情参考源码）

AudioDeviceModule 具体由 AudioDeviceModuleImpl 实现，二者之间还有一个 AudioDeviceModuleForTest，主要是添加了一些测试接口，对本文的分析无影响，可直接忽略。AudioDeviceModuleImpl 中有两个非常重要的成员变量，一个是 audio_device_ ，它的具体类型是 std::unique_ptr，另一个是 audio_device_buffer_ ，它的具体类型是 AudioDeviceBuffer。

### 3. AudioDeviceGeneric


其中 audio_device_ 是 AudioDeviceGeneric 类型，AudioDeviceGeneric 是各个平台具体音频采集和播放设备的一个抽象，由它承担 AudioDeviceModuleImpl 对具体设备的操作。涉及到具体设备的操作，AudioDeviceModuleImpl 除了做一些状态的判断具体的操作设备工作都由 AudioDeviceGeneric 来完成。AudioDeviceGeneric 的具体实现由各个平台自己实现，例如对于 iOS 平台具体实现是 AudioDeviceIOS，Android 平台具体实现是 AudioDeviceTemplate。至于各个平台的具体实现，有兴趣的可以单个分析。这里说一下最重要的共同点，从各个平台具体实现的定义中可以发现，他们都有一个 audio_device_buffer 成员变量，而这个变量与前面提到的 AudioDeviceModuleImpl 中的另一个重要成员变量 audio_device_buffer_ ，其实二者是同一个。AudioDeviceModuleImpl 通过 AttachAudioBuffer() 方法，将自己的 audio_device_buffer_ 对象传给具体的平台实现对象。

### 4. AudioDeviceBuffer

audio_device_buffer_ 的具体类型是 AudioDeviceBuffer，AudioDeviceBuffer 中的 `play_buffer_`、`rec_buffer_` 是 `int16_t`  类型的 buffer，前者做为向下获取播放 PCM 数据的 Buffer，后者做为向下传递采集 PCM 数据的 Buffer，具体的 PCM 数据流向在后面的数据流向章节具体分析，而另一个成员变量  `audio_transport_cb_` ，类型为 AudioTransport，从 AudioTransport 接口定义的中的两个核心方法不难看出他的作用，一是向下获取播放 PCM 数据存储在 `play_buffer_` ，另一个把采集存储在 `rec_buffer_` 中的 PCM 数据向下传递，后续具体流程参考数据流向章节。

### 5. AudioTransport

待补充。



### 6. 关于 ADM 扩展的思考 

从 WebRTC ADM 的实现来看，WebRTC 只实现对应了各个平台具体的硬件设备，并没什么虚拟设备。但是在实际的项目，往往需要支持外部音频输入/输出，就是由业务上层 push/pull 音频数据（PCM ...），而不是直接启动平台硬件进行采集/播放。在这种情况下，虽然原生的 WebRTC 不支持，但是要改造也是非常的简单，由于虚拟设备与平台无关，所以可以直接在 AudioDeviceModuleImpl 中增加一个与真实设备 `audio_device_` 对应的Virtual Device（变量名暂定为`virtual_device_`），`virtual_device_` 也跟 `audio_device_` 一样，实现 AudioDeviceGeneric 相关接口，然后参考 `audio_device_` 的实现去实现数据的“采集”（push）与 “播放”（pull），无须对接具体平台的硬件设备，唯一需要处理的就是物理设备 `audio_device_` 与虚拟设备 `virtual_device_` 之间的切换或协同工作。



## 参考

[WebRTC ADM 源码流程分析](https://blog.csdn.net/netease_im/article/details/123088666)


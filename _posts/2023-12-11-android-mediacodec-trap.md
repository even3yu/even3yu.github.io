---
layout: post
title: android 硬件编解码的坑
date: 2023-12-11 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android codec
---


* content
{:toc}

---


关于编解码，FFMpeg不香吗，为什么要吊死在Android的MediaCodec上？对于这个问题，我也很无奈，FFMpeg很香，但是因为包体积、效率等问题引发的工作业务的需要，使我不得不在Android MediaCodec的摧残下苟且偷生。MediaCodec的api比较简单，用来写demo毫无难度，让人痛不欲生的是它的兼容性问题。使用MediaCodec遇到的问题，往往都是和机型、版本、某类媒体文件相关的问题。

## 1. 编码器或者解码器Config时候崩溃，分辨率过大

这个问题对于MediaCodec的使用者来说应该是一个比较普遍的问题了，在MediaCodec进行config崩溃，往往会给出具体的崩溃信息，**比较常见的是格式不支持，最多的是宽高不支持或者编解码器数量限制**。

一个手机能够解码或者编码视频能力是有限的，在手机的/etc/media_codec.xml中是有说明的，比如使用一个15年左右的低端机型去解码1080p的视频，大概率会失败。当然，时代发展，限制的低端机型也大多支持1080p，那不妨解码4k视频试试。

另外，在现在的一些能够支持1080p解码的机型上，使用MediaCodec同时解码两路1080p视频也会出现config失败的问题，比如SM-J250f、SM-J700f等手机。这种情况，就只能避免这么使用了，引入软解，或者把1080p视频先转成小视频等。



### 如何获取当前手机支持的解码最大分辨率

每个手机下都有这样一个文件，/system/etc/media_codecs.xml (your path)。或者 /system/vendor/etc/media_codecs.xml这是一个xml文件，可以直接看到MediaCodecs–>Decoders节点下的各个视频格式的支持情况，以**华为荣耀7x Android 8.0 **为例

```xml
<Decoders>
    <MediaCodec name="OMX.hisi.video.decoder.avc" type="video/avc">
        <Quirk name="needs-flush-on-all-ports"/>
      <!-- min max res -->
        <Limit name="size" min="128x128" max="4096x2304" />
        <Limit name="alignment" value="2x2" />
        <Limit name="block-size" value="16x16" />
        <Limit name="blocks-per-second" min="1" max="972000" />
        <Limit name="bitrate" range="1-100000000" />
        <Feature name="adaptive-playback"/>
        <Quirk name="requires-allocate-on-input-ports"/>
        <Ouirk name="requires-allocate-on-output-ports" />
        <Limit name="concurrent-instances" max="16" />
    </MediaCodec>
    <MediaCodec name="OMX.hisi.video.decoder.avc.secure" type="video/avc">
        <Quirk name="needs-flush-on-all-ports"/>
        <Limit name="size" min="128x128" max="4096x2304" />
        <Limit name="alignment" value="2x2" />
        <Limit name="block-size" value="16x16" />
        <Limit name="blocks-per-second" min="1" max="972000" />
        <Limit name="bitrate" range="1-100000000" />
        <Feature name="adaptive-playback"/>
        <Feature name="secure-playback" required="true" />
        <Quirk name="requires-allocate-on-input-ports"/>
        <Quirk name="requires-allocate-on-output-ports"/>
        <Limit name="concurrent-instances" max="1" />
    </MediaCodec>
</Decoders>
```



### 获取解码视频的宽和高

```java
//获得音视频的配置器MediaFormat
private static MediaFormat getFormat(String path,boolean isVideo) {
	try {
	     MediaExtractor mediaExtractor = new MediaExtractor();
	     mediaExtractor.setDataSource(path);
	     int trackCount = mediaExtractor.getTrackCount();
	     for (int i = 0; i < trackCount; i++) {
	         MediaFormat trackFormat = mediaExtractor.getTrackFormat(i);
	         if (trackFormat.getString(MediaFormat.KEY_MIME).startsWith(isVideo ? "video/" :"audio/")) {
	             return mediaExtractor.getTrackFormat(i);
	         }
	     }
	 } catch (IOException e) {
	     e.printStackTrace();
	 }
	 return null;
}
//单独获取宽高
MediaFormat newFormat =  getFormat(path,true);
int videoWidth = newFormat.getInteger(MediaFormat.KEY_WIDTH);
int videoHeight = newFormat.getInteger(MediaFormat.KEY_HEIGHT);
//结合编码时获取宽高
MediaFormat newFormat = mMediaCodec.getOutputFormat();
int videoWidth = newFormat.getInteger(MediaFormat.KEY_WIDTH);
int videoHeight = newFormat.getInteger(MediaFormat.KEY_HEIGHT);
```



## 2. 解码视频，渲染出来的图像异常

这个一般是新手可能会遇到的问题，不是MediaCodec的问题，而是使用者的问题。有时候为了加速解码或者是其他的原因，使用者在进行解码的时候，会对一些帧进行丢弃，但是如果丢弃后，解码帧不是从关键帧开始的话，到下一个关键帧之前，解码出来的图像，都会是花屏的图像。所以需要做的就是要保证每次解码的流程开始时，塞入给解码的数据要从关键帧开始，并且不要随意丢输入帧。否则不会只有MediaCodec解码会花屏，FFMpeg同样无解。

在使用MediaCodec解码到ByteBuffer中，然后拿出来渲染，没有丢帧也有可能会遇到这个问题。这种情况下一般也不是解码器的问题，而是**渲染的问题，比如stride和width不匹配，在渲染的时候需要对数据进行处理。或者是颜色空间不正确，导致了渲染的色彩不正确等问题。**



## 3. 解码Mp4中的音频，seek到0后解码，解码出的音频数据不是从头开始的

遇到了使用MediaCodec进行Mp4音频解码，**MediaExtractor已经选中音频轨道，并且seek到0后，播放出来的音频并不是从头开始的问题**。并且同样的视频，在有的手机上是好的，有的手机上又有问题，不仅仅是使用MediaCodec，使用MediaPlayer直接来播放也会遇到同样的问题。调试后发现，解码出来的数据前面有很多帧给出来的pts是对的，但是数据的size一直为0，而塞入的数据又的确没啥问题。

在有问题的手机上，使用另外一个不存在问题的视频，和存在问题的视频对比了下，解码器解码时候的MediaFormat，发现有问题的视频带有"encoder-delay"标记，值为1，而正常的视频是没有的。google了下发现罪魁祸首果然是它，Android P此问题存在，Android Q版本进行了修复。**本地解决方案将"encoder-delay"标记值设置为0即可修复音频不是从头开始的问题**。



## 4. 音频解码无数据

三星J200G解码mp4中的音频，格式为mp4a-latm时遇到的偶现的问题。MediaCodec未返回错误，dequeueInputBuffer返回值正常，可以请求到buffer，但是dequeueOutputBuffer一直无法获得输出。log中有error信息：

```less
04-18 12:27:47.926 32443-2244/com.uc.vmate E/ACodec: OMXCodec::onEvent, OMX_ErrorStreamCorrupt
04-18 12:27:47.931 2175-2266/? E/SEC_AAC_DEC: saacd_decode() failed ret_val: -5, Indata 0x 21 1b 94 a5, length : 1326
04-18 12:27:47.931 2175-2266/? E/SEC_AAC_DEC: ASI 0x 11, 90 00 00
04-18 12:27:47.931 32443-2244/com.uc.vmate E/ACodec: OMXCodec::onEvent, OMX_ErrorStreamCorrupt
04-18 12:27:47.936 2175-2266/? E/SEC_AAC_DEC: saacd_decode() failed ret_val: -5, Indata 0x 21 1b 94 bd, length : 1314
```

最后发现问题不是出在解码器上，而是出在MediaExtractor上，MediaExtractor使用的时候，内部可能出错，导致MediaExtractor解封装的时候给的数据存在异常。



## 5. 视频解码时，获取输入输出Buffer始终为-1

在Oppo A37wf 5.1.1系统的手机上进行视频解码的时候，遇到了一个问题。**同时解码两路1080p视频，结果一路正常，一路解码出一帧后，dequeueInputBuffer和dequeueOutputBuffer始终返回-1，无报错**。而看grafika是可以同时解码的。对比了下我的使用和grafika中的示例，发现最终的原因是我在使用MediaCodec进行解码的时候，并不是dequeueOutputBuffer后就会releaseOutputBuffer，保证同时只会持有一帧数据。而是同时持有了两帧的outputBufer。

把代码改成只持有一个outputBuffer就正常了。回头再看了下Android cts中MediaCodec相关代码，并没有对同时多路持有多个Buffer进行测试，所以最终只能认命了，避免掉持有多帧不立即释放的情况。这里也就不得不提一个建议了，要想使用MediaCodec时，代码无bug，使用尽量不超出Android cts的范围，超出范围的使用就要做好面对一堆bug的准备了。



## 6. 借助SurfaceTexture解码，图像方向不正确

MediaCodec解码视频可以直接将视频图像解码到Surface上，而Surface又可以通过SurfaceTexture来进行构建，也就是说MediaCodec可以直接将图像解码到SurfaceTexture上，当我们需要对解码后的视频进行图像处理时，通常会选择这样的方式。

对于带有旋转或者裁边信息的视频，我们通常需要在渲染SurfaceTexture的纹理时，将其纹理矩阵信息通过getTransformMatrix拿到，然后去做渲染处理。但是在一些情况下，我们通过getTransformMatrix并不能拿到正确的矩阵。依旧是在Oppo A37wf这个手机上，使用两路1080p视频解码，会遇到一路视频正常，一路视频的SurfaceTexture无法通过getTransformMatrix获得正确的Matrix的情况。在grafika上也存在同样的问题。

进一步跟进，发现在出错的那一路解码器中，codec.getOutputFormat()得到的Format中有一个using-sw-renderer字段被设为1了，所以根本原因就是这个手机用Surface的方式同时硬解两路1080p视频，会有一路渲染使用的软件渲染的方式进行渲染，而这个垃圾手机软件渲染又有bug。 在MediaCodec.cpp可以找到设置和处理相关源码，但是找到可以强制改回使用硬件渲染的方式。

我使用的解决方案是在发现解码使用的是软件渲染的OutputFormat时，对获取的矩阵进行判断，如果矩阵信息和OutputFormat中的信息不匹配的话，重新计算一个渲染矩阵出来，然后渲染时候使用重新计算的矩阵。计算方式可以参考native层源码GLConsumer.cpp中的计算。直接copy过来就能用了。


## 7. 颜色格式问题

MediaCodec在初始化的时候，在`configure`的时候，需要传入一个MediaFormat对象，当作为编码器使用的时候，我们一般需要在MediaFormat中指定视频的宽高，帧率，码率，I帧间隔等基本信息，除此之外，还有一个重要的信息就是，指定编码器接受的YUV帧的颜色格式。这个是因为由于YUV根据其采样比例，UV分量的排列顺序有很多种不同的颜色格式，而对于Android的摄像头在`onPreviewFrame`输出的YUV帧格式，如果没有配置任何参数的情况下，基本上都是NV21格式，但Google对MediaCodec的API在设计和规范的时候，显得很不厚道，过于贴近Android的HAL层了，导致了NV21格式并不是所有机器的MediaCodec都支持这种格式作为编码器的输入格式！ 因此，在初始化MediaCodec的时候，我们需要通过`codecInfo.getCapabilitiesForType`来查询机器上的MediaCodec实现具体支持哪些YUV格式作为输入格式，一般来说，起码在4.4+的系统上，这两种格式在大部分机器都有支持：

```less
MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Planar
MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420SemiPlanar
```

两种格式分别是YUV420P和NV21，**如果机器上只支持YUV420P格式的情况下，则需要先将摄像头输出的NV21格式先转换成YUV420P，才能送入编码器进行编码，否则最终出来的视频就会花屏，或者颜色出现错乱**。

基本上用上了MediaCodec进行视频编码都会遇上这个问题。

### 解决方案

需要通过`codecInfo.getCapabilitiesForType`来查询机器上的MediaCodec实现具体支持哪些YUV格式作为输入格式。如果机器上只支持YUV420P格式的情况下，则需要先将摄像头输出的NV21格式先转换成YUV420P，才能送入编码器进行编码，否则最终出来的视频就会花屏，或者颜色出现错乱。



## 8. 编码器支持特性相当有限, level、profile设置

由于视频编码后显示的数据质量偏低，所以需要调整质量。这个时候需要在这个设置level、profile

Profile是对视频压缩特性的描述（CABAC呀、颜色采样数等等）。
Level是对视频本身特性的描述（码率、分辨率、fps）。
简单来说，Profile越高，就说明采用了越高级的压缩特性。
Level越高，视频的码率、分辨率、fps越高

```java
// 不支持设置Profile和Level，而应该采用默认设置
mediaFormat.setInteger(MediaFormat.KEY_PROFILE, MediaCodecInfo.CodecProfileLevel.AVCProfileHigh); 
mediaFormat.setInteger("level", MediaCodecInfo.CodecProfileLevel.AVCLevel41); // Level 4.1
```

如果使用MediaCodec来编码H264视频流，对于H264格式来说，会有一些针对压缩率以及码率相关的视频质量设置，典型的诸如Profile(baseline, main, high)，Profile Level, Bitrate mode(CBR, CQ, VBR)，合理配置这些参数可以让我们在同等的码率下，获得更高的压缩率，从而提升视频的质量，Android也提供了对应的API进行设置，可以设置到MediaFormat中这些设置项:

```less
MediaFormat.KEY_BITRATE_MODE
MediaFormat.KEY_PROFILE
MediaFormat.KEY_LEVEL
```

但问题是，对于Profile，Level, Bitrate mode这些设置，在大部分手机上都是不支持的，即使是设置了最终也不会生效，例如设置了Profile为high，最后出来的视频依然还会是Baseline。

这个问题，在7.0以下的机器几乎是必现的，其中一个可能的原因是，Android在源码层级[hardcode](http://androidxref.com/6.0.1_r10/xref/frameworks/av/media/libstagefright/ACodec.cpp)了profile的的设置：

```cobol
if (h264type.eProfile != OMX_VIDEO_AVCProfileBaseline) {
    ALOGW("Use baseline profile instead of %d for AVC recording",
            h264type.eProfile);
    h264type.eProfile = OMX_VIDEO_AVCProfileBaseline;
}
```

Android直到[7.0](http://androidxref.com/7.0.0_r1/xref/frameworks/av/media/libstagefright/ACodec.cpp)之后才取消了这段地方的Hardcode

```cobol
if (h264type.eProfile == OMX_VIDEO_AVCProfileBaseline) {
    ....
} else if (h264type.eProfile == OMX_VIDEO_AVCProfileMain ||
            h264type.eProfile == OMX_VIDEO_AVCProfileHigh) {
    .....
}
```

这个问题可以说间接导致了MediaCodec编码出来的视频质量偏低，同等码率下，难以获得跟软编码甚至iOS那样的视频质量。



## 9. 16位对齐要求

MediaCodec这个API在设计的时候，过于贴近HAL层，这在很多Soc的实现上，是直接把传入MediaCodec的buffer，在不经过任何前置处理的情况下就直接送入了Soc中。而在编码h264视频流的时候，**由于h264的编码块大小一般是16x16，于是乎在一开始设置视频的宽高的时候，如果设置了一个没有对齐16的大小，例如960x540，在某些cpu上，最终编码出来的视频就会直接花屏**！

**webrtc 中，开启cpu过载检测会自动调节分辨率，如果没有设置16位对齐，调整后的分辨率也会出现花屏。**



### 解决方法

- 动态调整分辨率的时候生成的分辨率默认是2位对齐的，如果设备是16位对齐的话需要修改Androidvideotracksource.cc下的对齐定义为16，如：`const int kRequiredResolutionAlignment = 16;`
- 很明显这还是因为厂商在实现这个API的时候，对传入的数据缺少校验以及前置处理导致的，目前来看，华为，三星的Soc出现这个问题会比较频繁，其他厂商的一些早期Soc也有这种问题，**一般来说解决方法还是在设置视频宽高的时候，统一设置成对齐16位之后的大小就好了**。



## 10. ??? BitMode

文档上写着三种支持三种码率模式，CQ的意思是尽量保证帧率，保证图片质量；CBR 则是尽量保证每一帧的码率，但在一些动作比较大的Video中，就很容易模糊；VBR 则是前两者之间一个中庸的方案。
但是，绝大多数机型都只支持 VRB 一种。

MediaCodec中的bitrate mode，设置之前想确认下CBR是否支持，那么会调用isBitrateModeSupported()判断

```java
// MediaCodecInfo.java
public boolean isBitrateModeSupported(int mode) {
    for (Feature feat: bitrates) {
        if (mode == feat.mValue) {
            return (mBitControl & (1 << mode)) != 0;
        }
    }
    return false;
}
```

**mode是否支持从bitrates判断**

```java
private static final Feature[] bitrates = new Feature[] {
    new Feature("VBR", BITRATE_MODE_VBR, true),
    new Feature("CBR", BITRATE_MODE_CBR, false),
    new Feature("CQ",  BITRATE_MODE_CQ,  false)
};
```

大部分是不支持的，可以通过xml查看，位置：vendor/etc/media_codecs.xml

另外，在默认情况下，如果上层没有主动设置bitrate_mode的话，返回的是VBR。 也就是默认采用VBR 关于VBR CQ CBR区别，可查看[Android原生编解码接口 MediaCodec 之——完全解析中的流控](https://blog.csdn.net/gb702250823/article/details/81627503)。


### 解决方法

网易有相关文章，没找到。通过`BitrateAdjuster` 解决。
[(Android-RTC-8)分析HardwareVideoEncoder—BitrateAdjuster](https://blog.csdn.net/a360940265a/article/details/122583855)



## 11.  分辨率过小导致硬件解码失败

客户反馈在直播过程中，画面突然卡住，声音正常。

分析：通过对播放日志以及视频流 Dump 分析我们发现问题发生时，播放端正在使用 Android 硬件解码，且拉到的视频流分辨率发生了变化（1080p->16p)，通过查阅资料我们锁定了问题的根因，即 Android 的 MediaCodec 编解码对分辨率的支持范围是因设备而异的。可以通过查看设备中/vendor/etc/media_codecs_xxx.xml 查看，如设备“M2006C3LC”的 media_codecs_mediatek_video.xml，关于 H264 的解码描述如下图，该“OMX.MTK.VIDEO.DECODER.AVC”所支持的分辨率范围为（64*64 ）- （2048*1088），故在解析分辨率为 16*16 的视频时可能会出现一些无法预知的问题。

![img]({{ site.url }}{{ site.baseurl }}/images/android-mediacodec-trap.assets/resoulation.png)



## 12. 关键帧

MediaCodec 有两种方式触发输出关键帧，

- 由配置时设置的 KEY_FRAME_RATE 和KEY_I_FRAME_INTERVAL参数自动触发;
- 运行过程中通过 setParameters 手动触发输出关键帧。

### 自动触发输出关键帧

在MediaCodec硬编码中设置Ｉ(关键帧)时间间隔，在 api 中是这么设置的

```java
mediaFormat.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, 1);  //关键帧间隔时间 单位s
```

**自动触发实际是按照帧数触发的**，例如设置帧率为 20 fps，关键帧间隔为 1s ，那就会每 20桢输出一个关键帧，**一旦实际帧率低于配置帧率，那就会导致关键帧间隔时间变长**。**由于 MediaCodec 启动后就不能修改配置帧率/关键帧间隔了，所以如果希望改变关键帧间隔帧数，就必须重启编码器**。

### 手动触发输出关键帧

```java
if (System.currentTimeMillis() - timeStamp >= 1000) {//1000毫秒后，设置参数
	timeStamp = System.currentTimeMillis();
	if (Build.VERSION.SDK_INT >= 23) {
		Bundle params = new Bundle();
		params.putInt(MediaCodec.PARAMETER_KEY_REQUEST_SYNC_FRAME, 0);
		mMediaCodec.setParameters(params);
	}
｝
```

### 关键帧踩坑

有时候你会发现自动触发关键帧方式失效了。

- 经排查发现真正的原因是在于视频的输入源，如果是通过Camera的PreviewCallback的方式来获取视频数据再喂给MediaCodec的方式是无法控制输出关键帧的数量的。

- 发现当选择支持颜色格式为`yuv420p`的编码器时，KEY_I_FRAME_INTERVAL 设置无效；
  选择支持`yuv420sp`的编码器时，KEY_I_FRAME_INTERVAL 设置有效；

- 想要控制输出输出关键帧数量就必须通过调用MediaCodec.createInputSurface()方法获取输入Surface，再通过Opengl渲染后喂给MediaCodec才能真正控制关键帧的数量。

```java
//判断输出数据是否为关键帧的方法：
boolean keyFrame = (bufferInfo.flags & MediaCodec.BUFFER_FLAG_KEY_FRAME) != 0;
```



## 13. 部分机型MediaCodec.dequeueOutputBuffer一直报IllegalStateException

部分机型会一直卡在`MediaCodec.INFO_TRY_AGAIN_LATER`中，有的原因也是因为这个
**该机型硬解码最大配置分辨率低于当前视频流的分辨率**



## 14. 部分机型输出的数据太短，或者为0

取出 output buffer 后，要手动设置 position 和 limit（api19以下必须设置），有些设备的编码器不会设置这两个值，导致无法正确取出数据；取出 input buffer 后，要手动调用 clear。参见 [bigflake FAQ #11](https://bigflake.com/mediacodec/#q11)



## 15. 关于BufferInfo中的presentationTimeUs设置

如果不正确设置presentationTimeUs，有的设备的编码器会丢掉输入桢，或者输出图像质量很差，参见[bigflake FAQ #8](https://bigflake.com/mediacodec/#q8);
MediaCodec 使用的是微秒，大多数java 使用毫秒和纳秒，单位要处理好



## 参考

[Android MediaCodec踩坑笔记](https://blog.csdn.net/junzia/article/details/106036509)

[(Android-RTC-8)分析HardwareVideoEncoder—BitrateAdjuster](https://blog.csdn.net/a360940265a/article/details/122583855)

[Android原生编解码接口 MediaCodec 之——踩坑](https://blog.csdn.net/gb702250823/article/details/81669684)
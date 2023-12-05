---
layout: post
title: android 视频硬件编码工厂
date: 2023-12-01 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android encode video
---


* content
{:toc}

---



## 1. 编码器工厂

| encoder factory                   | 说明                                                         |      |
| --------------------------------- | ------------------------------------------------------------ | ---- |
| VideoEncoderFactory               | 接口类                                                       |      |
| DefaultVideoEncoderFactory        | 是个容器，HardwareVideoEncoderFactory+SoftwareVideoEncoderFactory |      |
| HardwareVideoEncoderFactory       | 软件编码                                                     |      |
| SoftwareVideoEncoderFactory       | 硬件编码                                                     |      |
| CustomHardwareVideoEncoderFactory | 测试使用                                                     |      |





## 2. VideoEncoderFactory

sdk/android/api/org/webrtc/VideoEncoderFactory.java

视频编码器工厂接口类。

- VideoEncoder createEncoder(VideoCodecInfo info);

  根据 VideoCodecInfo 创建编码器。

- VideoCodecInfo[] getSupportedCodecs();

  支持的codec

-  VideoEncoderFactory.VideoEncoderSelector

```java
/** Factory for creating VideoEncoders. */
public interface VideoEncoderFactory {
  public interface VideoEncoderSelector {
    /** Called with the VideoCodecInfo of the currently used encoder. */
    @CalledByNative("VideoEncoderSelector") void onCurrentEncoder(VideoCodecInfo info);

    /**
     * Called with the current available bitrate. Returns null if the encoder selector prefers to
     * keep the current encoder or a VideoCodecInfo if a new encoder is preferred.
     */
    @Nullable @CalledByNative("VideoEncoderSelector") VideoCodecInfo onAvailableBitrate(int kbps);

    /**
     * Called when the currently used encoder signal itself as broken. Returns null if the encoder
     * selector prefers to keep the current encoder or a VideoCodecInfo if a new encoder is
     * preferred.
     */
    @Nullable @CalledByNative("VideoEncoderSelector") VideoCodecInfo onEncoderBroken();
  }

  /**
   * Returns a VideoEncoderSelector if implemented by the VideoEncoderFactory,
   * null otherwise.
   */
  @CalledByNative
  default VideoEncoderSelector getEncoderSelector() {
    return null;
  }
}
```



## 3. ??? SoftwareVideoEncoderFactory

sdk/android/api/org/webrtc/SoftwareVideoEncoderFactory.java

```java
public class SoftwareVideoEncoderFactory implements VideoEncoderFactory {
  // 根据VideoCodecInfo 创建对应的编码器
  @Nullable
  @Override
  public VideoEncoder createEncoder(VideoCodecInfo info) {
    if (info.name.equalsIgnoreCase("VP8")) {
      return new LibvpxVp8Encoder();
    }
    if (info.name.equalsIgnoreCase("VP9") && LibvpxVp9Encoder.nativeIsSupported()) {
      return new LibvpxVp9Encoder();
    }

    return null;
  }

  // 支持的codec
  @Override
  public VideoCodecInfo[] getSupportedCodecs() {
    return supportedCodecs();
  }

  static VideoCodecInfo[] supportedCodecs() {
    List<VideoCodecInfo> codecs = new ArrayList<VideoCodecInfo>();

    codecs.add(new VideoCodecInfo("VP8", new HashMap<>()));
    if (LibvpxVp9Encoder.nativeIsSupported()) {
      codecs.add(new VideoCodecInfo("VP9", new HashMap<>()));
    }

    return codecs.toArray(new VideoCodecInfo[codecs.size()]);
  }
}
```

是通过调用jni实现，还是调用了系统自带的软件编码？？？？



## 4. HardwareVideoEncoderFactory

sdk/android/api/org/webrtc/HardwareVideoEncoderFactory.java

```java
/** Factory for android hardware video encoders. */
@SuppressWarnings("deprecation") // API 16 requires the use of deprecated methods.
public class HardwareVideoEncoderFactory implements VideoEncoderFactory {
  private static final String TAG = "HardwareVideoEncoderFactory";

  // Forced key frame interval - used to reduce color distortions on Qualcomm platforms.
  private static final int QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_L_MS = 15000;
  private static final int QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_M_MS = 20000;
  private static final int QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_N_MS = 15000;

  // List of devices with poor H.264 encoder quality.
  // HW H.264 encoder on below devices has poor bitrate control - actual
  // bitrates deviates a lot from the target value.
  private static final List<String> H264_HW_EXCEPTION_MODELS =
      Arrays.asList("SAMSUNG-SGH-I337", "Nexus 7", "Nexus 4");

  @Nullable private final EglBase14.Context sharedContext;
  // 支持 vp8 encoder
  private final boolean enableIntelVp8Encoder;
  // 支持h264 hp
  private final boolean enableH264HighProfile;
  @Nullable private final Predicate<MediaCodecInfo> codecAllowedPredicate;


  /**
   * Creates a HardwareVideoEncoderFactory that supports surface texture encoding.
   *
   * @param sharedContext The textures generated will be accessible from this context. May be null,
   *                      this disables texture support.
   * @param enableIntelVp8Encoder true if Intel's VP8 encoder enabled.
   * @param enableH264HighProfile true if H264 High Profile enabled.
   * @param codecAllowedPredicate optional predicate to filter codecs. All codecs are allowed
   *                              when predicate is not provided.
   */
  public HardwareVideoEncoderFactory(EglBase.Context sharedContext, boolean enableIntelVp8Encoder,
      boolean enableH264HighProfile, @Nullable Predicate<MediaCodecInfo> codecAllowedPredicate) {
    // Texture mode requires EglBase14.
    if (sharedContext instanceof EglBase14.Context) {
      this.sharedContext = (EglBase14.Context) sharedContext;
    } else {
      Logging.w(TAG, "No shared EglBase.Context.  Encoders will not use texture mode.");
      this.sharedContext = null;
    }
    this.enableIntelVp8Encoder = enableIntelVp8Encoder;
    this.enableH264HighProfile = enableH264HighProfile;
    this.codecAllowedPredicate = codecAllowedPredicate;
  }

  private boolean isH264HighProfileSupported(MediaCodecInfo info) {
    return enableH264HighProfile && Build.VERSION.SDK_INT > Build.VERSION_CODES.M
        && info.getName().startsWith(EXYNOS_PREFIX);
  }
}

```



### 4.1 HardwareVideoEncoderFactory.createEncoder

```java
  // 创建编码器
  @Nullable
  @Override
  public VideoEncoder createEncoder(VideoCodecInfo input) {
    // HW encoding is not supported below Android Kitkat.
    // Android 4.4 
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
      return null;
    }

    // 根据VideoCodecInfo.name， 转换为VideoCodecMimeType 枚举, 就是编码格式
    VideoCodecMimeType type = VideoCodecMimeType.valueOf(input.name);
    // 根据编码格式，比如avc，hevc，vp8，查找支持的 android.media.MediaCodecInfo;
    MediaCodecInfo info = findCodecForType(type);

    if (info == null) {
      return null;
    }

    String codecName = info.getName();
    String mime = type.mimeType();
    // 纹理 色彩空间
    Integer surfaceColorFormat = MediaCodecUtils.selectColorFormat(
        MediaCodecUtils.TEXTURE_COLOR_FORMATS, info.getCapabilitiesForType(mime));
    // 编码 色彩空间
    Integer yuvColorFormat = MediaCodecUtils.selectColorFormat(
        MediaCodecUtils.ENCODER_COLOR_FORMATS, info.getCapabilitiesForType(mime));

    // h264 codec 处理
    if (type == VideoCodecMimeType.H264) {
      boolean isHighProfile = H264Utils.isSameH264Profile(
          input.params, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ true));
      boolean isBaselineProfile = H264Utils.isSameH264Profile(
          input.params, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ false));

      if (!isHighProfile && !isBaselineProfile) {
        return null;
      }
      if (isHighProfile && !isH264HighProfileSupported(info)) {
        return null;
      }
    }

    // 创建 HardwareVideoEncoder 编码器
    return new HardwareVideoEncoder(new MediaCodecWrapperFactoryImpl(), codecName, type,
        surfaceColorFormat, yuvColorFormat, input.params, getKeyFrameIntervalSec(type),
        getForcedKeyFrameIntervalMs(type, codecName), createBitrateAdjuster(type, codecName),
        sharedContext);
  }
```

主要作用创建`HardwareVideoEncoder`编码器。

- 低于Android 4.4 不支持硬件编码

- 根据`VideoCodecMimeType`编码格式，比如avc，hevc，vp8，查找支持的 `android.media.MediaCodecInfo`;

- 纹理色彩空间和编码色彩空间；通过`MediaCodecUtils.selectColorFormat` 找到合适的色彩空间；

  sdk/android/src/java/org/webrtc/MediaCodecUtils.java

  ```java
    // Color formats supported by hardware encoder - in order of preference.
    static final int[] ENCODER_COLOR_FORMATS = {
        MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Planar,
        MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420SemiPlanar,
        MediaCodecInfo.CodecCapabilities.COLOR_QCOM_FormatYUV420SemiPlanar,
        MediaCodecUtils.COLOR_QCOM_FORMATYUV420PackedSemiPlanar32m};
  ```

  sdk/android/src/java/org/webrtc/MediaCodecUtils.java

  ```java
    // Color formats supported by texture mode encoding - in order of preference.
    static final int[] TEXTURE_COLOR_FORMATS = getTextureColorFormats();
  
    private static int[] getTextureColorFormats() {
      if (Build.VERSION.SDK_INT >= 18) {
        return new int[] {MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface};
      } else {
        return new int[] {};
      }
    }
  ```

- 创建 `HardwareVideoEncoder`;



#### MediaCodecUtils.selectColorFormat

sdk/android/src/java/org/webrtc/MediaCodecUtils.java

```java
/**
supportedColorFormats 目前只的色彩空间，就是枚举
CodecCapabilities capabilities， 就是 MediaCodecInfo 所支持的色彩空间
通过两者匹配找到合适的色彩空间
*/
static @Nullable Integer selectColorFormat(
      int[] supportedColorFormats, CodecCapabilities capabilities) {
    for (int supportedColorFormat : supportedColorFormats) {
      for (int codecColorFormat : capabilities.colorFormats) {
        if (codecColorFormat == supportedColorFormat) {
          return codecColorFormat;
        }
      }
    }
    return null;
  }
```

通过两者匹配找到合适的色彩空间。



#### HardwareVideoEncoderFactory.getKeyFrameIntervalSec

```java
  private int getKeyFrameIntervalSec(VideoCodecMimeType type) {
    switch (type) {
      case VP8: // Fallthrough intended.
      case VP9:
        return 100;
      case H264:
        return 20;
    }
    throw new IllegalArgumentException("Unsupported VideoCodecMimeType " + type);
  }
```

关键帧的间隔。 返回秒。



#### ??? HardwareVideoEncoderFactory.getForcedKeyFrameIntervalMs

```java
  private int getForcedKeyFrameIntervalMs(VideoCodecMimeType type, String codecName) {
    if (type == VideoCodecMimeType.VP8 && codecName.startsWith(QCOM_PREFIX)) {
      if (Build.VERSION.SDK_INT == Build.VERSION_CODES.LOLLIPOP
          || Build.VERSION.SDK_INT == Build.VERSION_CODES.LOLLIPOP_MR1) {
        return QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_L_MS;
      } else if (Build.VERSION.SDK_INT == Build.VERSION_CODES.M) {
        return QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_M_MS;
      } else if (Build.VERSION.SDK_INT > Build.VERSION_CODES.M) {
        return QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_N_MS;
      }
    }
    // Other codecs don't need key frame forcing.
    return 0;
  }


  // Forced key frame interval - used to reduce color distortions on Qualcomm platforms.
  private static final int QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_L_MS = 15000;
  private static final int QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_M_MS = 20000;
  private static final int QCOM_VP8_KEY_FRAME_INTERVAL_ANDROID_N_MS = 15000;
```

关键帧的间隔。返回毫秒。



#### ??? HardwareVideoEncoderFactory.createBitrateAdjuster

```java
  private BitrateAdjuster createBitrateAdjuster(VideoCodecMimeType type, String codecName) {
    if (codecName.startsWith(EXYNOS_PREFIX)) {
      if (type == VideoCodecMimeType.VP8) {
        // Exynos VP8 encoders need dynamic bitrate adjustment.
        return new DynamicBitrateAdjuster();
      } else {
        // Exynos VP9 and H264 encoders need framerate-based bitrate adjustment.
        return new FramerateBitrateAdjuster();
      }
    }
    // Other codecs don't need bitrate adjustment.
    return new BaseBitrateAdjuster();
  }
```



### 4.2 ???VideoCodecMimeType

参考

### 4.3 ???VideoCodecInfo

参考



### 4.4 HardwareVideoEncoderFactory.getSupportedCodecs

```java
  @Override
  public VideoCodecInfo[] getSupportedCodecs() {
    // HW encoding is not supported below Android Kitkat.
    // Android 4.4
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
      return new VideoCodecInfo[0];
    }

    List<VideoCodecInfo> supportedCodecInfos = new ArrayList<VideoCodecInfo>();
    // Generate a list of supported codecs in order of preference:
    // VP8, VP9, H264 (high profile), and H264 (baseline profile).
    // 找到VP8，VP9，H264分别支持的codec，每个VideoCodecMimeType 找到第一个适配的；
    for (VideoCodecMimeType type : new VideoCodecMimeType[] {
             VideoCodecMimeType.VP8, VideoCodecMimeType.VP9, VideoCodecMimeType.H264}) {
      MediaCodecInfo codec = findCodecForType(type);
      if (codec != null) {
        String name = type.name();
        // TODO(sakal): Always add H264 HP once WebRTC correctly removes codecs that are not
        // supported by the decoder.
        // 判断是否 支持high profile， 则添加high
        if (type == VideoCodecMimeType.H264 && isH264HighProfileSupported(codec)) {
          supportedCodecInfos.add(new VideoCodecInfo(
              name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ true)));
        }
				// 正常添加codec info
        supportedCodecInfos.add(new VideoCodecInfo(
            name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ false)));
      }
    }

    return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
  }
```

1. 找到`VP8，VP9，H264`对应支持的codec，对于H264， 如果支持high profile，则添加high profile 和baseline profile；
2. `findCodecForType` **找到第一个匹配的`VideoCodecMimeType`; 如果有多个codec，就用匹配到的codec；**这里要注意；







### 4.5 !!! HardwareVideoEncoderFactory.findCodecForType

```java
  // 根据 VideoCodecMimeType，编码格式， 查看对应的MediaCodecInfo
  private @Nullable MediaCodecInfo findCodecForType(VideoCodecMimeType type) {
    // android.media.MediaCodecList;
    for (int i = 0; i < MediaCodecList.getCodecCount(); ++i) {
      MediaCodecInfo info = null;
      try {
        info = MediaCodecList.getCodecInfoAt(i);
      } catch (IllegalArgumentException e) {
        Logging.e(TAG, "Cannot retrieve encoder codec info", e);
      }

      // 是否是编码器
      if (info == null || !info.isEncoder()) {
        continue;
      }

      // 过滤codec, 这里的条件很多。
      if (isSupportedCodec(info, type)) {
        return info;
      }
    }
    return null; // No support for this type.
  }
```

根据 `VideoCodecMimeType type` , 从android 系统查找得到MediaCodecInfo。

> 注意，这里找到第一个匹配的codec，就返回了，即使有多个匹配也不会做什么处理



#### 4.5.1 HardwareVideoEncoderFactory.isSupportedCodec

```java
  // Returns true if the given MediaCodecInfo indicates a supported encoder for the given type.
  private boolean isSupportedCodec(MediaCodecInfo info, VideoCodecMimeType type) {
    // MediaCodecInfo是否支持type（这里是h264）
    if (!MediaCodecUtils.codecSupportsType(info, type)) {
      return false;
    }
    // 匹配编码色彩空间
    // Check for a supported color format.
    if (MediaCodecUtils.selectColorFormat(
            MediaCodecUtils.ENCODER_COLOR_FORMATS, info.getCapabilitiesForType(type.mimeType()))
        == null) {
      return false;
    }
    // isHardwareSupportedInCurrentSdk，是否支持codec，isMediaCodecAllowed 是否允许codec，
    return isHardwareSupportedInCurrentSdk(info, type) && isMediaCodecAllowed(info);
  }
```

1. MediaCodecInfo是否支持type（这里是h264）

2. MediaCodecInfo 的编码器的色彩空间

   ```java
     // Color formats supported by hardware encoder - in order of preference.
     static final int[] ENCODER_COLOR_FORMATS = {
         MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Planar,
         MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420SemiPlanar,
         MediaCodecInfo.CodecCapabilities.COLOR_QCOM_FormatYUV420SemiPlanar,
         MediaCodecUtils.COLOR_QCOM_FORMATYUV420PackedSemiPlanar32m};
   ```

3. `isHardwareSupportedInCurrentSdk`, 硬件编码支持的判断；

4. `isMediaCodecAllowed`, 规则过滤，这里可以定制自己的codec优先级；



#### 4.5.2 MediaCodecUtils.codecSupportsType

sdk/android/src/java/org/webrtc/MediaCodecUtils.java

```java
 /**
 * MediaCodecInfo 是否支持 VideoCodecMimeType type;
 * MediaCodecInfo info， 编码器支持的编码格式，VideoCodecMimeType
 * VideoCodecMimeType type， 编码格式
 */
 static boolean codecSupportsType(MediaCodecInfo info, VideoCodecMimeType type) {
    for (String mimeType : info.getSupportedTypes()) {
      if (type.mimeType().equals(mimeType)) {
        return true;
      }
    }
    return false;
  }
```





#### 4.5.3 !!! HardwareVideoEncoderFactory.isHardwareSupportedInCurrentSdk

```java
  // Returns true if the given MediaCodecInfo indicates a hardware module that is supported on the
  // current SDK.
  private boolean isHardwareSupportedInCurrentSdk(MediaCodecInfo info, VideoCodecMimeType type) {
    switch (type) {
      case VP8:
        return isHardwareSupportedInCurrentSdkVp8(info);
      case VP9:
        return isHardwareSupportedInCurrentSdkVp9(info);
      case H264:
        return isHardwareSupportedInCurrentSdkH264(info);
    }
    return false;
  }

	// vp8
  private boolean isHardwareSupportedInCurrentSdkVp8(MediaCodecInfo info) {
    String name = info.getName();
    // QCOM Vp8 encoder is supported in KITKAT or later.
    return (name.startsWith(QCOM_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT)
        // Exynos VP8 encoder is supported in M or later.
        || (name.startsWith(EXYNOS_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M)
        // Intel Vp8 encoder is supported in LOLLIPOP or later, with the intel encoder enabled.
        || (name.startsWith(INTEL_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP
               && enableIntelVp8Encoder);
  }

	// vp9
  private boolean isHardwareSupportedInCurrentSdkVp9(MediaCodecInfo info) {
    String name = info.getName();
    return (name.startsWith(QCOM_PREFIX) || name.startsWith(EXYNOS_PREFIX))
        // Both QCOM and Exynos VP9 encoders are supported in N or later.
        && Build.VERSION.SDK_INT >= Build.VERSION_CODES.N;
  }

	// h264
  private boolean isHardwareSupportedInCurrentSdkH264(MediaCodecInfo info) {
    // H264_HW_EXCEPTION_MODELS 这里设备，不支持硬件编码，性能不足
    // First, H264 hardware might perform poorly on this model.
    if (H264_HW_EXCEPTION_MODELS.contains(Build.MODEL)) {
      return false;
    }
    // 根据codec的 name的前缀匹配， 这里可以添加自己匹配支持的编码器
    String name = info.getName();
    // QCOM H264 encoder is supported in KITKAT or later.
    return (name.startsWith(QCOM_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT)
        // Exynos H264 encoder is supported in LOLLIPOP or later.
        || (name.startsWith(EXYNOS_PREFIX)
               && Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP)
        || (name.startsWith(MTK_PREFIX))
        || (name.startsWith(RK_PREFIX))
        || (name.startsWith(IMG_TOPAZ_PREFIX))
        || (name.startsWith(C2_RK_PREFIX))
        || (name.startsWith(AMLOGIC_PREFIX));
  }
```



#### !!! 4.5.4 HardwareVideoEncoderFactory.isMediaCodecAllowed

```java
  private boolean isMediaCodecAllowed(MediaCodecInfo info) {
    if (codecAllowedPredicate == null) {
      return true;
    }
    // 是否包含 info
    return codecAllowedPredicate.test(info);
  }
```



## 5. ??? MediaCodecWrapperFactory

sdk/android/src/java/org/webrtc/MediaCodecWrapperFactory.java

```java
interface MediaCodecWrapperFactory {
  /**
   * Creates a new {@link MediaCodecWrapper} by codec name.
   *
   * <p>For additional information see {@link android.media.MediaCodec#createByCodecName}.
   */
  MediaCodecWrapper createByCodecName(String name) throws IOException;
}

```



### 5.1 MediaCodecWrapperFactoryImpl

sdk/android/src/java/org/webrtc/MediaCodecWrapperFactoryImpl.java

```java
/**
 * Implementation of MediaCodecWrapperFactory that returns MediaCodecInterfaces wrapping
 * {@link android.media.MediaCodec} objects.
 */
class MediaCodecWrapperFactoryImpl implements MediaCodecWrapperFactory {
  private static class MediaCodecWrapperImpl implements MediaCodecWrapper {
    private final MediaCodec mediaCodec;

    public MediaCodecWrapperImpl(MediaCodec mediaCodec) {
      this.mediaCodec = mediaCodec;
    }

    @Override
    public void configure(MediaFormat format, Surface surface, MediaCrypto crypto, int flags) {
      mediaCodec.configure(format, surface, crypto, flags);
    }

    @Override
    public void start() {
      mediaCodec.start();
    }

    @Override
    public void flush() {
      mediaCodec.flush();
    }

    @Override
    public void stop() {
      mediaCodec.stop();
    }

    @Override
    public void release() {
      mediaCodec.release();
    }

    @Override
    public int dequeueInputBuffer(long timeoutUs) {
      return mediaCodec.dequeueInputBuffer(timeoutUs);
    }

    @Override
    public void queueInputBuffer(
        int index, int offset, int size, long presentationTimeUs, int flags) {
      mediaCodec.queueInputBuffer(index, offset, size, presentationTimeUs, flags);
    }

    @Override
    public int dequeueOutputBuffer(BufferInfo info, long timeoutUs) {
      return mediaCodec.dequeueOutputBuffer(info, timeoutUs);
    }

    @Override
    public void releaseOutputBuffer(int index, boolean render) {
      mediaCodec.releaseOutputBuffer(index, render);
    }

    @Override
    public MediaFormat getOutputFormat() {
      return mediaCodec.getOutputFormat();
    }

    @Override
    public ByteBuffer[] getInputBuffers() {
      return mediaCodec.getInputBuffers();
    }

    @Override
    public ByteBuffer[] getOutputBuffers() {
      return mediaCodec.getOutputBuffers();
    }

    @Override
    @TargetApi(18)
    public Surface createInputSurface() {
      return mediaCodec.createInputSurface();
    }

    @Override
    @TargetApi(19)
    public void setParameters(Bundle params) {
      mediaCodec.setParameters(params);
    }
  }

  @Override
  public MediaCodecWrapper createByCodecName(String name) throws IOException {
    return new MediaCodecWrapperImpl(MediaCodec.createByCodecName(name));
  }
}
```



## 6. DefaultVideoEncoderFactory

sdk/android/api/org/webrtc/DefaultVideoEncoderFactory.java

```java

/** Helper class that combines HW and SW encoders. */
public class DefaultVideoEncoderFactory implements VideoEncoderFactory {
  private final VideoEncoderFactory hardwareVideoEncoderFactory;
  private final VideoEncoderFactory softwareVideoEncoderFactory = new SoftwareVideoEncoderFactory();

  /** Create encoder factory using default hardware encoder factory. */
  public DefaultVideoEncoderFactory(
      EglBase.Context eglContext, boolean enableIntelVp8Encoder, boolean enableH264HighProfile) {
    this.hardwareVideoEncoderFactory =
        new HardwareVideoEncoderFactory(eglContext, enableIntelVp8Encoder, enableH264HighProfile);
  }

  /** Create encoder factory using explicit hardware encoder factory. */
  DefaultVideoEncoderFactory(VideoEncoderFactory hardwareVideoEncoderFactory) {
    this.hardwareVideoEncoderFactory = hardwareVideoEncoderFactory;
  }

  @Nullable
  @Override
  public VideoEncoder createEncoder(VideoCodecInfo info) {
    final VideoEncoder softwareEncoder = softwareVideoEncoderFactory.createEncoder(info);
    final VideoEncoder hardwareEncoder = hardwareVideoEncoderFactory.createEncoder(info);
    // 如果软硬件都支持的话，先用hardware，失败，则用software
    if (hardwareEncoder != null && softwareEncoder != null) {
      // Both hardware and software supported, wrap it in a software fallback
      return new VideoEncoderFallback(
          /* fallback= */ softwareEncoder, /* primary= */ hardwareEncoder);
    }
    return hardwareEncoder != null ? hardwareEncoder : softwareEncoder;
  }

  @Override
  public VideoCodecInfo[] getSupportedCodecs() {
    LinkedHashSet<VideoCodecInfo> supportedCodecInfos = new LinkedHashSet<VideoCodecInfo>();

    supportedCodecInfos.addAll(Arrays.asList(softwareVideoEncoderFactory.getSupportedCodecs()));
    supportedCodecInfos.addAll(Arrays.asList(hardwareVideoEncoderFactory.getSupportedCodecs()));

    return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
  }
}
```





## 7. VideoCodecMimeType

java/org/webrtc/VideoCodecMimeType.java

```java
enum VideoCodecMimeType {
  VP8("video/x-vnd.on2.vp8"),
  VP9("video/x-vnd.on2.vp9"),
  H264("video/avc");

  private final String mimeType;

  private VideoCodecMimeType(String mimeType) {
    this.mimeType = mimeType;
  }

  String mimeType() {
    return mimeType;
  }
}
```



## 8. VideoCodecInfo

```java
public class VideoCodecInfo {
  // Keys for H264 VideoCodecInfo properties.
  public static final String H264_FMTP_PROFILE_LEVEL_ID = "profile-level-id";
  public static final String H264_FMTP_LEVEL_ASYMMETRY_ALLOWED = "level-asymmetry-allowed";
  public static final String H264_FMTP_PACKETIZATION_MODE = "packetization-mode";

  // 支持I/P 帧, 42= base profile, 
  public static final String H264_PROFILE_CONSTRAINED_BASELINE = "42e0"; // baseline profile
  // ???,64 = high profile , 
  public static final String H264_PROFILE_CONSTRAINED_HIGH = "640c"; // high profile
  
  // 分辨率 720p
  public static final String H264_LEVEL_3_1 = "1f"; // 31 in hex.
  
  // 640c1f
  public static final String H264_CONSTRAINED_HIGH_3_1 =
      H264_PROFILE_CONSTRAINED_HIGH + H264_LEVEL_3_1;
  // 42e01f
  public static final String H264_CONSTRAINED_BASELINE_3_1 =
      H264_PROFILE_CONSTRAINED_BASELINE + H264_LEVEL_3_1;

  // codec name
  public final String name;
  // parms map
  public final Map<String, String> params;

  @CalledByNative
  public VideoCodecInfo(String name, Map<String, String> params) {
    this.name = name;
    this.params = params;
  }

  ...
}
```

存放编码器信息的，编码器名称，对应的一些参数配置。








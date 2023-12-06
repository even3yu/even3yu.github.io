---
layout: post
title: android 视频硬件解码工厂
date: 2023-12-02 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android decode video
---


* content
{:toc}

---


## 1. 继承关系

```less
VideoDecoderFactory
MediaCodecVideoDecoderFactory DefaultVideoDecoderFactory
HardwareVideoDecoderFactory  / PlatformSoftwareVideoDecoderFactory


sdk/android/api/org/webrtc/PlatformSoftwareVideoDecoderFactory.java
sdk/android/api/org/webrtc/HardwareVideoDecoderFactory.java
```

- VideoDecoderFactory 接口类
- MediaCodecVideoDecoderFactory ，实现android codec的解码器工厂；
- DefaultVideoDecoderFactory是个容器，真正的实现是 PlatformSoftwareVideoDecoderFactory 和 HardwareVideoDecoderFactory；
- HardwareVideoDecoderFactory子类，用MediaCodec来实现的硬解码
- PlatformSoftwareVideoDecoderFactory子类，用Android平台系统自带的来实现的软解码，不是用ffempg等；



## 2. VideoDecoderFactory

sdk/android/api/org/webrtc/VideoDecoderFactory.java

```java
/** Factory for creating VideoDecoders. */
public interface VideoDecoderFactory {
  /**
   * Creates a VideoDecoder for the given codec. Supports the same codecs supported by
   * VideoEncoderFactory.
   */
  @Deprecated
  @Nullable
  default VideoDecoder createDecoder(String codecType) {
    throw new UnsupportedOperationException("Deprecated and not implemented.");
  }
  
  /** Creates a decoder for the given video codec. */
  @Nullable
  @CalledByNative
  default VideoDecoder createDecoder(VideoCodecInfo info) {
    return createDecoder(info.getName());
  }

  /**
   * Enumerates the list of supported video codecs.
   */
  @CalledByNative
  default VideoCodecInfo[] getSupportedCodecs() {
    return new VideoCodecInfo[0];
  }
}
```



## 3. MediaCodecVideoDecoderFactory

sdk/android/src/java/org/webrtc/MediaCodecVideoDecoderFactory.java



### 3.1 !!! Predicate\<MediaCodecInfo\> codecAllowedPredicate

这个是由子类来实现。决定使用哪个codec来解码。

#### 3.1.1 HardwareVideoDecoderFactory

```java
  private final static Predicate<MediaCodecInfo> defaultAllowedPredicate =
      new Predicate<MediaCodecInfo>() {
        @Override
        public boolean test(MediaCodecInfo arg) {

          String name = arg.getName();
          if (name.equalsIgnoreCase(preferH264Codec)){
            Logging.d(TAG, "preferH264Codec=" + preferH264Codec);
            return true;
          }
          return MediaCodecUtils.isHardwareAccelerated(arg);
        }
      };
```

硬件的实现。



#### 3.1.2 PlatformSoftwareVideoDecoderFactory

```java
  private static final Predicate<MediaCodecInfo> defaultAllowedPredicate =
      new Predicate<MediaCodecInfo>() {
        @Override
        public boolean test(MediaCodecInfo arg) {
          return MediaCodecUtils.isSoftwareOnly(arg);
        }
      };
```

软件的实现。



### 3.2 createDecoder

```java
  @Nullable
  @Override
  public VideoDecoder createDecoder(VideoCodecInfo codecType) {
    // 根据codec name，转换为 VideoCodecMimeType
    VideoCodecMimeType type = VideoCodecMimeType.valueOf(codecType.getName());
    // type 找到对应的MediaCodecInfo
    MediaCodecInfo info = findCodecForType(type);

    if (info == null) {
      return null;
    }

    // 创建解码器
    CodecCapabilities capabilities = info.getCapabilitiesForType(type.mimeType());
    return new AndroidVideoDecoder(new MediaCodecWrapperFactoryImpl(), info.getName(), type,
        MediaCodecUtils.selectColorFormat(MediaCodecUtils.DECODER_COLOR_FORMATS, capabilities),
        sharedContext);
  }
```

1. 找到支持对应的MediaCodecInfo
2. 创建解码器



### 3.3 getSupportedCodecs

```java
 @Override
  public VideoCodecInfo[] getSupportedCodecs() {
    List<VideoCodecInfo> supportedCodecInfos = new ArrayList<VideoCodecInfo>();
    // Generate a list of supported codecs in order of preference:
    // VP8, VP9, H264 (high profile), and H264 (baseline profile).
    for (VideoCodecMimeType type : new VideoCodecMimeType[] {
             VideoCodecMimeType.VP8, VideoCodecMimeType.VP9, VideoCodecMimeType.H264}) {
      // 根据 VideoCodecMimeType 找到 MediaCodecInfo
      MediaCodecInfo codec = findCodecForType(type);
      if (codec != null) {
        String name = type.name();
        // h264 检查是否支持 high profile
        if (type == VideoCodecMimeType.H264 && isH264HighProfileSupported(codec)) {
          supportedCodecInfos.add(new VideoCodecInfo(
              name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ true)));
        }

       	// 支持的codec
        supportedCodecInfos.add(new VideoCodecInfo(
          				// 获取codec的相关参数
            name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ false)));
      }
    }

    // 转换为数组
    return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
  }
```





#### 3.3.1 isH264HighProfileSupported

```java
  private boolean isH264HighProfileSupported(MediaCodecInfo info) {
    String name = info.getName();
    // Android 5
    // Support H.264 HP decoding on QCOM chips for Android L and above.
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP && name.startsWith(QCOM_PREFIX)) {
      return true;
    }
    // Android 6
    // Support H.264 HP decoding on Exynos chips for Android M and above.
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && name.startsWith(EXYNOS_PREFIX)) {
      return true;
    }
    return false;
  }
```





#### 3.3.2 getCodecProperties

```java
  static Map<String, String> getCodecProperties(VideoCodecMimeType type, boolean highProfile) {
    switch (type) {
      case VP8:
      case VP9:
        return new HashMap<String, String>();
      case H264:
        return H264Utils.getDefaultH264Params(highProfile);
      default:
        throw new IllegalArgumentException("Unsupported codec: " + type);
    }
  }
```



```java
  public static Map<String, String> getDefaultH264Params(boolean isHighProfile) {
    final Map<String, String> params = new HashMap<>();
    params.put(VideoCodecInfo.H264_FMTP_LEVEL_ASYMMETRY_ALLOWED, "1");
    params.put(VideoCodecInfo.H264_FMTP_PACKETIZATION_MODE, "1");
    params.put(VideoCodecInfo.H264_FMTP_PROFILE_LEVEL_ID,
        isHighProfile ? VideoCodecInfo.H264_CONSTRAINED_HIGH_3_1
                      : VideoCodecInfo.H264_CONSTRAINED_BASELINE_3_1);
    return params;
  }
  
  public static final String H264_FMTP_PROFILE_LEVEL_ID = "profile-level-id";
  public static final String H264_FMTP_LEVEL_ASYMMETRY_ALLOWED = "level-asymmetry-allowed";
  public static final String H264_FMTP_PACKETIZATION_MODE = "packetization-mode";

  public static final String H264_PROFILE_CONSTRAINED_BASELINE = "42e0";
  public static final String H264_PROFILE_CONSTRAINED_HIGH = "640c";
  public static final String H264_LEVEL_3_1 = "1f"; // 31 in hex.
```





#### 3.3.3 findCodecForType

```java
private @Nullable MediaCodecInfo findCodecForType(VideoCodecMimeType type) {
  // HW decoding is not supported on builds before KITKAT.
  if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
    return null;
  }

  for (int i = 0; i < MediaCodecList.getCodecCount(); ++i) {
    MediaCodecInfo info = null;
    try {
      info = MediaCodecList.getCodecInfoAt(i);
    } catch (IllegalArgumentException e) {
      Logging.e(TAG, "Cannot retrieve decoder codec info", e);
    }

    // 如果不是解码器，则跳过
    if (info == null || info.isEncoder()) {
      continue;
    }
		// 判断支持的codec
    if (isSupportedCodec(info, type)) {
      return info;
    }
  }

  return null; // No support for this type.
}
```

通过以下的条件进行判断：

1. MediaCodecInfo 是解码器；
2. MediaCodecInfo 是否支持 `VideoCodecMimeType type`；`MediaCodecUtils.codecSupportsType`;
3. MediaCodecInfo 的解码色彩空间支持；`MediaCodecUtils.selectColorFormat`;
4. MediaCodecInfo 是否支持硬件加速，`isSoftwareOnly(), isHardwareAccelerated`;



### 3.4 !!! isSupportedCodec

```java
  private boolean isSupportedCodec(MediaCodecInfo info, VideoCodecMimeType type) {
    String name = info.getName();
    // MediaCodecInfo是否支持此VideoCodecMimeType type
    if (!MediaCodecUtils.codecSupportsType(info, type)) {
      return false;
    }
    // 色彩空间
    // Check for a supported color format.
    if (MediaCodecUtils.selectColorFormat(
            MediaCodecUtils.DECODER_COLOR_FORMATS, info.getCapabilitiesForType(type.mimeType()))
        == null) {
      return false;
    }
    return isCodecAllowed(info);
  }
```





#### 3.4.1 MediaCodecUtils.codecSupportsType

sdk/android/src/java/org/webrtc/MediaCodecUtils.java

```java
  static boolean codecSupportsType(MediaCodecInfo info, VideoCodecMimeType type) {
    for (String mimeType : info.getSupportedTypes()) {
      if (type.mimeType().equals(mimeType)) {
        return true;
      }
    }
    return false;
  }
```

判断 MediaCodecInfo 是否支持` VideoCodecMimeType type`。



#### 3.4.2 MediaCodecUtils.selectColorFormat

sdk/android/src/java/org/webrtc/MediaCodecUtils.java

```java
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

色彩空间。

#### 3.4.3 DECODER_COLOR_FORMATS

sdk/android/src/java/org/webrtc/MediaCodecUtils.java

```java
  // Color formats supported by hardware decoder - in order of preference.
  static final int[] DECODER_COLOR_FORMATS = new int[] {CodecCapabilities.COLOR_FormatYUV420Planar,
      CodecCapabilities.COLOR_FormatYUV420SemiPlanar,
      CodecCapabilities.COLOR_QCOM_FormatYUV420SemiPlanar,
      MediaCodecUtils.COLOR_QCOM_FORMATYVU420PackedSemiPlanar32m4ka,
      MediaCodecUtils.COLOR_QCOM_FORMATYVU420PackedSemiPlanar16m4ka,
      MediaCodecUtils.COLOR_QCOM_FORMATYVU420PackedSemiPlanar64x32Tile2m8ka,
      MediaCodecUtils.COLOR_QCOM_FORMATYUV420PackedSemiPlanar32m};
```





#### 3.4.4 isCodecAllowed

```java
  private boolean isCodecAllowed(MediaCodecInfo info) {
    if (codecAllowedPredicate == null) {
      return true;
    }
    // 这里是可以定制化的，就是codecAllowedPredicate， 可以由子类来决定，支持怎样的codec
    return codecAllowedPredicate.test(info);
  }

```



- 子类 HardwareVideoDecoderFactory 赋值了 defaultAllowedPredicate

```java
  private final @Nullable Predicate<MediaCodecInfo> codecAllowedPredicate;

  public MediaCodecVideoDecoderFactory(@Nullable EglBase.Context sharedContext,
      @Nullable Predicate<MediaCodecInfo> codecAllowedPredicate) {
    this.sharedContext = sharedContext;
    this.codecAllowedPredicate = codecAllowedPredicate;
  }


  public HardwareVideoDecoderFactory(@Nullable EglBase.Context sharedContext,
      @Nullable Predicate<MediaCodecInfo> codecAllowedPredicate) {
    super(sharedContext,
        (codecAllowedPredicate == null ? defaultAllowedPredicate
                                       : codecAllowedPredicate.and(defaultAllowedPredicate)));
  }

  private final static Predicate<MediaCodecInfo> defaultAllowedPredicate =
      new Predicate<MediaCodecInfo>() {
        @Override
        public boolean test(MediaCodecInfo arg) {
          return MediaCodecUtils.isHardwareAccelerated(arg);
        }
      };

```



##### isHardwareAccelerated

```java
  static boolean isHardwareAccelerated(MediaCodecInfo info) {
    // Android 10
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
      // 是否支持硬件加速
      return isHardwareAcceleratedQOrHigher(info);
    }
    return !isSoftwareOnly(info);
  }

  @TargetApi(29)
  private static boolean isHardwareAcceleratedQOrHigher(android.media.MediaCodecInfo codecInfo) {
    return codecInfo.isHardwareAccelerated();
  }

  static boolean isSoftwareOnly(android.media.MediaCodecInfo codecInfo) {
    // Android 10
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
      return isSoftwareOnlyQOrHigher(codecInfo);
    }
    String name = codecInfo.getName();
    //   static final String[] SOFTWARE_IMPLEMENTATION_PREFIXES = {
    //  "OMX.google.", "OMX.SEC.", "c2.android"};
    for (String prefix : SOFTWARE_IMPLEMENTATION_PREFIXES) {
      if (name.startsWith(prefix)) {
        return true;
      }
    }
    return false;
  }

  @TargetApi(29)
  private static boolean isSoftwareOnlyQOrHigher(android.media.MediaCodecInfo codecInfo) {
    return codecInfo.isSoftwareOnly();
  }
```








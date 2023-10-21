---
layout: post
title: video encoder factory
date: 2023-10-07 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc video
---


* content
{:toc}

---


```less
webrtc/api/video_codecs/video_encoder_factory.h
webrtc/api/video_codecs/builtin_video_encoder_factory.h
webrtc/api/video_codecs/builtin_video_encoder_factory.cc
webrtc/media/engine/internal_encoder_factory.h
webrtc/media/engine/internal_encoder_factory.cc


========================================
// oc-》c++的转换类
webrtc/sdk/objc/native/api/video_encoder_factory.h
webrtc/sdk/objc/native/api/video_encoder_factory.mm

// c++ 编码 factrory 封装
webrtc/sdk/objc/native/src/objc_video_encoder_factory.h
webrtc/sdk/objc/native/src/objc_video_encoder_factory.mm


// oc RTCDefaultVideoEncoderFactory
webrtc/sdk/objc/components/video_codec/RTCDefaultVideoEncoderFactory.h
webrtc/sdk/objc/components/video_codec/RTCDefaultVideoEncoderFactory.m


========================================
webrtc/sdk/android/native_api/codecs/android_default_video_encoder_factory.h
webrtc/sdk/android/native_api/codecs/android_default_video_encoder_factory.cpp

webrtc/sdk/android/native_api/codecs/android_hardware_codec_factory_helper.h
webrtc/sdk/android/native_api/codecs/android_hardware_codec_factory_helper.cpp

webrtc/api/video_codecs/video_encoder_software_fallback_wrapper.h
webrtc/api/video_codecs/video_encoder_software_fallback_wrapper.cpp

webrtc/sdk/android/api/org/webrtc/HardwareVideoEncoderFactory.java
```



## 0. 前言

1. 通过全局函数`CreateBuiltinVideoEncoderFactory` 创建 `VideoEncoderFactory`，就是BuiltinVideoEncoderFactory对象
2. BuiltinVideoEncoderFactory 继承于 VideoEncoderFactory
3. BuiltinVideoEncoderFactory内部调用的是InternalEncoderFactory， 真正实现是在InternalEncoderFactory
4. ObjCVideoEncoderFactory  继承于 VideoEncoderFactory， 用于iOS/Mac的硬件编码
4. AndroidDefaultVideoEncoderFactory 继承于 VideoEncoderFactory，用于android的硬件编码，硬件失败，则用软编码



## 1. 类图

```
VideoEncoderFactory
BuiltinVideoEncoderFactory \
InternalEncoderFactory \
ObjCVideoEncoderFactory \
AndroidDefaultVideoEncoderFactory
```



- VideoEncoderFactory是基类；
- BuiltinVideoEncoderFactory 子类，是软件编码的工厂类。其实内部调用的是InternalEncoderFactory；
- InternalEncoderFactory 也是子类，这是软件编码真正实现类；
- ObjCVideoEncoderFactory 子类，是硬件编码的工厂类，其实内部调用的是oc实现的factory，继承于RTCVideoEncoderFactory；
- AndroidDefaultVideoEncoderFactory 子类，是硬件编码的工厂类，其实内部调用的是java实现的factory，HardwareVideoEncoderFactory；



## 2. -------------BuiltinVideoEncoderFactory------------------

通用平台，软件编码factory

### 2.1 CreateBuiltinVideoEncoderFactory

api/video_codecs/builtin_video_encoder_factory.h
api/video_codecs/builtin_video_encoder_factory.cc

```c++
// Creates a new factory that can create the built-in types of video encoders.
// The factory has simulcast support for VP8.
RTC_EXPORT std::unique_ptr<VideoEncoderFactory>
CreateBuiltinVideoEncoderFactory();
```



```c++
std::unique_ptr<VideoEncoderFactory> CreateBuiltinVideoEncoderFactory() {
  return std::make_unique<BuiltinVideoEncoderFactory>();
}
```

创建BuiltinVideoEncoderFactory 实例。



### 2.2 VideoEncoderFactory

api/video_codecs/video_encoder_factory.h

VideoEncoderFactory是个基类，实现这个接口类。

```c++
// A factory that creates VideoEncoders.
// NOTE: This class is still under development and may change without notice.
class VideoEncoderFactory {
 public:

  // Returns a list of supported video formats in order of preference, to use
  // for signaling etc.
  // 得到所支持的编码器类型
  virtual std::vector<SdpVideoFormat> GetSupportedFormats() const = 0;

  // Returns a list of supported video formats in order of preference, that can
  // also be tagged with additional information to allow the VideoEncoderFactory
  // to separate between different implementations when CreateVideoEncoder is
  // called.
  virtual std::vector<SdpVideoFormat> GetImplementations() const {
    return GetSupportedFormats();
  }

  // Returns information about how this format will be encoded. The specified
  // format must be one of the supported formats by this factory.
  // TODO(magjed): Try to get rid of this method.
  // 查询编码器的状态
  virtual CodecInfo QueryVideoEncoder(const SdpVideoFormat& format) const = 0;

  // Creates a VideoEncoder for the specified format.
  // 创建编码器webrtc::VideoEncoder实例
  virtual std::unique_ptr<VideoEncoder> CreateVideoEncoder(
      const SdpVideoFormat& format) = 0;

  virtual std::unique_ptr<EncoderSelectorInterface> GetEncoderSelector() const {
    return nullptr;
  }

  virtual ~VideoEncoderFactory() {}
};
```



### 2.3 BuiltinVideoEncoderFactory

api/video_codecs/builtin_video_encoder_factory.cc

BuiltinVideoEncoderFactory 实现了 VideoEncoderFactory的
`QueryVideoEncoder
CreateVideoEncoder
GetSupportedFormats`。
真正的实现是在InternalEncoderFactory 这里实现，这里类似使用 桥接模式。

```c++
class BuiltinVideoEncoderFactory : public VideoEncoderFactory {
 public:
  BuiltinVideoEncoderFactory()
      : internal_encoder_factory_(new InternalEncoderFactory()) {}

  VideoEncoderFactory::CodecInfo QueryVideoEncoder(
      const SdpVideoFormat& format) const override {
    // Format must be one of the internal formats.
    RTC_DCHECK(IsFormatSupported(
        internal_encoder_factory_->GetSupportedFormats(), format));
    VideoEncoderFactory::CodecInfo info;
    return info;
  }

  std::unique_ptr<VideoEncoder> CreateVideoEncoder(
      const SdpVideoFormat& format) override {
    // Try creating internal encoder.
    std::unique_ptr<VideoEncoder> internal_encoder;
    if (IsFormatSupported(internal_encoder_factory_->GetSupportedFormats(),
                          format)) {
      internal_encoder = std::make_unique<EncoderSimulcastProxy>(
          internal_encoder_factory_.get(), format);
    }

    return internal_encoder;
  }

  std::vector<SdpVideoFormat> GetSupportedFormats() const override {
    return internal_encoder_factory_->GetSupportedFormats();
  }

 private:
  const std::unique_ptr<VideoEncoderFactory> internal_encoder_factory_;
};
```



#### IsFormatSupported

api/video_codecs/builtin_video_encoder_factory.cc

```cpp
bool IsFormatSupported(const std::vector<SdpVideoFormat>& supported_formats,
                       const SdpVideoFormat& format) {
  for (const SdpVideoFormat& supported_format : supported_formats) {
    if (cricket::IsSameCodec(format.name, format.parameters,
                             supported_format.name,
                             supported_format.parameters)) {
      return true;
    }
  }
  return false;
}

```



### 2.4 InternalEncoderFactory

media/engine/internal_encoder_factory.h

```c++
class RTC_EXPORT InternalEncoderFactory : public VideoEncoderFactory {
 public:
  static std::vector<SdpVideoFormat> SupportedFormats();
  std::vector<SdpVideoFormat> GetSupportedFormats() const override;

  CodecInfo QueryVideoEncoder(const SdpVideoFormat& format) const override;

  std::unique_ptr<VideoEncoder> CreateVideoEncoder(
      const SdpVideoFormat& format) override;
};
```

InternalEncoderFactory 是 EncoderFactory的真正实现类。



#### 2.4.1 SdpVideoFormat

api/video_codecs/sdp_video_format.cc

```cpp
struct RTC_EXPORT SdpVideoFormat {
  using Parameters = std::map<std::string, std::string>;

  std::string ToString() const {
    rtc::StringBuilder builder;
    builder << "Codec name: " << name << ", parameters: {";
    for (const auto& kv : parameters)
      builder << " " << kv.first << "=" << kv.second;
    builder << " }";

    return builder.str();
  }


  std::string name;
  Parameters parameters;
};
```

用来描述video codec。



#### 2.4.2 InternalEncoderFactory::CreateVideoEncoder

```c++
std::unique_ptr<VideoEncoder> InternalEncoderFactory::CreateVideoEncoder(
    const SdpVideoFormat& format) {
  if (absl::EqualsIgnoreCase(format.name, cricket::kVp8CodecName))
    return VP8Encoder::Create();
  if (absl::EqualsIgnoreCase(format.name, cricket::kVp9CodecName))
    return VP9Encoder::Create(cricket::VideoCodec(format));
  if (absl::EqualsIgnoreCase(format.name, cricket::kH264CodecName))
    return H264Encoder::Create(cricket::VideoCodec(format));
  if (kIsLibaomAv1EncoderSupported &&
      absl::EqualsIgnoreCase(format.name, cricket::kAv1CodecName))
    return CreateLibaomAv1Encoder();
  RTC_LOG(LS_ERROR) << "Trying to created encoder of unsupported format "
                    << format.name;
  return nullptr;
}
```

创建编码器。
从`InternalEncoderFactory::CreateVideoEncoder`的实现里我们可以看到，webrtc支持四种类型的视频编码器：

- VP8Encoder
- VP9Encoder
- H264Encoder
- LibaomAv1Encoder

> H264`默认是不支持的，需要在编译的时候加上特殊的编译参数`rtc_use_h264=true



#### 2.4.3 InternalEncoderFactory::GetSupportedFormats()

```c++
std::vector<SdpVideoFormat> InternalEncoderFactory::SupportedFormats() {
  std::vector<SdpVideoFormat> supported_codecs;
  supported_codecs.push_back(SdpVideoFormat(cricket::kVp8CodecName));
  for (const webrtc::SdpVideoFormat& format : webrtc::SupportedVP9Codecs())
    supported_codecs.push_back(format);
  for (const webrtc::SdpVideoFormat& format : webrtc::SupportedH264Codecs())
    supported_codecs.push_back(format); 
  if (kIsLibaomAv1EncoderSupported)
    supported_codecs.push_back(SdpVideoFormat(cricket::kAv1CodecName));
  return supported_codecs;
}

std::vector<SdpVideoFormat> InternalEncoderFactory::GetSupportedFormats()
    const {
  return SupportedFormats();
}
```

支持的codec列表。

##### 2.4.3.1 SupportedH264Codecs

modules/video_coding/codecs/h264/h264.cc

```cpp
std::vector<SdpVideoFormat> SupportedH264Codecs() {
  if (!IsH264CodecSupported())
    return std::vector<SdpVideoFormat>();
  // We only support encoding Constrained Baseline Profile (CBP), but the
  // decoder supports more profiles. We can list all profiles here that are
  // supported by the decoder and that are also supersets of CBP, i.e. the
  // decoder for that profile is required to be able to decode CBP. This means
  // we can encode and send CBP even though we negotiated a potentially
  // higher profile. See the H264 spec for more information.
  //
  // We support both packetization modes 0 (mandatory) and 1 (optional,
  // preferred).
  return {
      CreateH264Format(H264::kProfileBaseline, H264::kLevel3_1, "1"),
      CreateH264Format(H264::kProfileBaseline, H264::kLevel3_1, "0"),
      CreateH264Format(H264::kProfileConstrainedBaseline, H264::kLevel3_1, "1"),
      CreateH264Format(H264::kProfileConstrainedBaseline, H264::kLevel3_1,
                       "0")};
}
```



## 3. ------------ObjcVideoCodecFactory--------

1. ObjcVideoCodecFactory， 提供了静态方法，创建encoder factory
2. **ObjCVideoEncoderFactory（和1的挺像，但是不一样）**，对oc的封装，内部调用OWTDefaultVideoEncoderFactory的方法
3. RTCVideoEncoderFactory， oc的 encoder factory的协议
4. RTCVideoEncoderFactoryH264，oc的 encoder factory的 webrtc的实现，支持h264



### 3.1 CreateObjCEncoderFactory

modules/video_coding/codecs/test/objc_codec_factory_helper.h
modules/video_coding/codecs/test/objc_codec_factory_helper.mm

```c++
std::unique_ptr<VideoEncoderFactory> CreateObjCEncoderFactory();

std::unique_ptr<VideoEncoderFactory> CreateObjCEncoderFactory() {
  return ObjCToNativeVideoEncoderFactory([[RTC_OBJC_TYPE(RTCVideoEncoderFactoryH264) alloc] init]);
}
```

全局函数，创建ObjCVideoEncoderFactory对象,  传入真实的实现factory，这里传入的是RTCVideoEncoderFactoryH264。
ObjCVideoEncoderFactory调用的就是传入的factory。



#### --RTCVideoEncoderFactoryH264

【3.4】

#### 3.1.1 ObjCToNativeVideoEncoderFactory

sdk/objc/native/api/video_encoder_factory.h
sdk/objc/native/api/video_encoder_factory.mm

```c++
std::unique_ptr<VideoEncoderFactory> ObjCToNativeVideoEncoderFactory(
    id<RTCVideoEncoderFactory> objc_video_encoder_factory);

//------------------------
std::unique_ptr<VideoEncoderFactory> ObjCToNativeVideoEncoderFactory(
    id<RTCVideoEncoderFactory> objc_video_encoder_factory) {
  return std::make_unique<ObjCVideoEncoderFactory>(objc_video_encoder_factory);
}
```

创建ObjCVideoEncoderFactory对象。



### 3.2 ObjCVideoEncoderFactory——iOS/Mac 硬件编码，内部调用oc的factory

```c++
class ObjCVideoEncoderFactory : public VideoEncoderFactory {
 public:
  explicit ObjCVideoEncoderFactory(id<RTCVideoEncoderFactory>);
  ~ObjCVideoEncoderFactory() override;

  id<RTCVideoEncoderFactory> wrapped_encoder_factory() const;

  std::vector<SdpVideoFormat> GetSupportedFormats() const override;
  std::vector<SdpVideoFormat> GetImplementations() const override;
  std::unique_ptr<VideoEncoder> CreateVideoEncoder(
      const SdpVideoFormat& format) override;
  std::unique_ptr<EncoderSelectorInterface> GetEncoderSelector() const override;

 private:
  id<RTCVideoEncoderFactory> encoder_factory_;
};
```



#### 3.2.1 ObjCVideoEncoderFactory::wrapped_encoder_factory

```cpp
id<RTC_OBJC_TYPE(RTCVideoEncoderFactory)> ObjCVideoEncoderFactory::wrapped_encoder_factory() const {
  return encoder_factory_;
}
```

encoder_factory_ 就是创建ObjCVideoEncoderFactory传入的，是真实实现类。



#### 3.2.2 ObjCVideoEncoderFactory::GetSupportedFormats()

```c++
std::vector<SdpVideoFormat> ObjCVideoEncoderFactory::GetSupportedFormats() const {
  std::vector<SdpVideoFormat> supported_formats;
  for (RTCVideoCodecInfo *supportedCodec in [encoder_factory_ supportedCodecs]) {
    SdpVideoFormat format = [supportedCodec nativeSdpVideoFormat];
    supported_formats.push_back(format);
  }

  return supported_formats;
}
```

支持的codec列表。



#### 3.2.3 ObjCVideoEncoderFactory::GetImplementations()

```c++
std::vector<SdpVideoFormat> ObjCVideoEncoderFactory::GetImplementations() const {
  if ([encoder_factory_ respondsToSelector:SEL("implementations")]) {
    std::vector<SdpVideoFormat> supported_formats;
    for (RTCVideoCodecInfo *supportedCodec in [encoder_factory_ implementations]) {
      SdpVideoFormat format = [supportedCodec nativeSdpVideoFormat];
      supported_formats.push_back(format);
    }
    return supported_formats;
  }
  return GetSupportedFormats();
}
```



#### 3.2.4 ObjCVideoEncoderFactory::CreateVideoEncoder

```c++
std::unique_ptr<VideoEncoder> ObjCVideoEncoderFactory::CreateVideoEncoder(
    const SdpVideoFormat &format) {
  RTCVideoCodecInfo *info = [[RTCVideoCodecInfo alloc] initWithNativeSdpVideoFormat:format];
  id<RTCVideoEncoder> encoder = [encoder_factory_ createEncoder:info];
  // Because of symbol conflict, isKindOfClass doesn't work as expected.
  // See https://bugs.webkit.org/show_bug.cgi?id=198782.
  // if ([encoder isKindOfClass:[RTCWrappedNativeVideoEncoder class]]) {
  if ([info.name isEqual:@"VP8"] || [info.name isEqual:@"VP9"]) {
    return [(RTCWrappedNativeVideoEncoder *)encoder releaseWrappedEncoder];
  } else {
    return std::unique_ptr<ObjCVideoEncoder>(new ObjCVideoEncoder(encoder));
  }
}
```

创建编码器。



#### 3.2.5 ObjCVideoEncoderFactory::GetEncoderSelector

```c++
std::unique_ptr<VideoEncoderFactory::EncoderSelectorInterface>
    ObjCVideoEncoderFactory::GetEncoderSelector() const {
  if ([encoder_factory_ respondsToSelector:@selector(encoderSelector)]) {
    id<RTC_OBJC_TYPE(RTCVideoEncoderSelector)> selector = [encoder_factory_ encoderSelector];
    if (selector) {
      return absl::make_unique<ObjcVideoEncoderSelector>(selector);
    }
  }
  return nullptr;
}
```





### 3.3 RTCVideoEncoderFactory——protocal

sdk/objc/base/RTCVideoEncoderFactory.h

```objc
/** RTCVideoEncoderFactory is an Objective-C version of webrtc::VideoEncoderFactory.
 */
RTC_OBJC_EXPORT
@protocol RTC_OBJC_TYPE
(RTCVideoEncoderFactory)<NSObject>

- (nullable id<RTC_OBJC_TYPE(RTCVideoEncoder)>)createEncoder
    : (RTC_OBJC_TYPE(RTCVideoCodecInfo) *)info;
- (NSArray<RTC_OBJC_TYPE(RTCVideoCodecInfo) *> *)
    supportedCodecs;  // TODO(andersc): "supportedFormats" instead?

@optional
- (NSArray<RTC_OBJC_TYPE(RTCVideoCodecInfo) *> *)implementations;
- (nullable id<RTC_OBJC_TYPE(RTCVideoEncoderSelector)>)encoderSelector;

@end
```



### 3.4 RTCVideoEncoderFactoryH264

sdk/objc/components/video_codec/RTCVideoEncoderFactoryH264.h

```objc
RTC_OBJC_EXPORT
@interface RTC_OBJC_TYPE (RTCVideoEncoderFactoryH264) : NSObject <RTC_OBJC_TYPE(RTCVideoEncoderFactory)>
@end
```



```objc
@implementation RTC_OBJC_TYPE (RTCVideoEncoderFactoryH264)

- (NSArray<RTC_OBJC_TYPE(RTCVideoCodecInfo) *> *)supportedCodecs {
  NSMutableArray<RTC_OBJC_TYPE(RTCVideoCodecInfo) *> *codecs = [NSMutableArray array];
  NSString *codecName = kRTCVideoCodecH264Name;

  NSDictionary<NSString *, NSString *> *constrainedHighParams = @{
    @"profile-level-id" : kRTCMaxSupportedH264ProfileLevelConstrainedHigh,
    @"level-asymmetry-allowed" : @"1",
    @"packetization-mode" : @"1",
  };
  RTC_OBJC_TYPE(RTCVideoCodecInfo) *constrainedHighInfo =
      [[RTC_OBJC_TYPE(RTCVideoCodecInfo) alloc] initWithName:codecName
                                                  parameters:constrainedHighParams];
  [codecs addObject:constrainedHighInfo];

  NSDictionary<NSString *, NSString *> *constrainedBaselineParams = @{
    @"profile-level-id" : kRTCMaxSupportedH264ProfileLevelConstrainedBaseline,
    @"level-asymmetry-allowed" : @"1",
    @"packetization-mode" : @"1",
  };
  RTC_OBJC_TYPE(RTCVideoCodecInfo) *constrainedBaselineInfo =
      [[RTC_OBJC_TYPE(RTCVideoCodecInfo) alloc] initWithName:codecName
                                                  parameters:constrainedBaselineParams];
  [codecs addObject:constrainedBaselineInfo];

  return [codecs copy];
}

- (id<RTC_OBJC_TYPE(RTCVideoEncoder)>)createEncoder:(RTC_OBJC_TYPE(RTCVideoCodecInfo) *)info {
  return [[RTC_OBJC_TYPE(RTCVideoEncoderH264) alloc] initWithCodecInfo:info];
}

@end
```



#### profile-level-id

#### level-asymmetry-allowed

#### packetization-mode



### 3.5 关系总结

- ObjCVideoEncoderFactory 只是一个壳，具体的codec的factory，是用oc封装的，RTCVideoEncoderFactoryH264
  ObjCVideoEncoderFactory 是继承于VideoEncoderFactory；
- RTCVideoEncoderFactory 是protocol，RTCVideoEncoderFactoryH264 是具体的实现。ObjCVideoEncoderFactory 内部是调用RTCVideoEncoderFactoryH264；



### 3.6 ???Mac/iOS 如何支持软编码



## 4. ------------AndroidDefaultVideoEncoderFactory--------

1. 全局函数 CreateAndroidDefaultVideoEncoderFactory， 
1. AndroidDefaultVideoEncoderFactory，
1. AndroidDefaultVideoEncoderFactory 是c++封装的类，内部其实调用的是BuiltinVideoEncoderFactory和HardwareVideoEncoderFactory；这里为什么创建两个factory，**创建了硬件和软件的encoder，当硬件失败的时候，软件做为备用**；
1. HardwareVideoEncoderFactory 是java的，`org/webrtc/HardwareVideoEncoderFactory`, c++ 调用java创建的对象；



### 4.1 CreateAndroidDefaultVideoEncoderFactory

```c++
std::unique_ptr<webrtc::VideoEncoderFactory>
CreateAndroidDefaultVideoEncoderFactory();
```



```c++
std::unique_ptr<webrtc::VideoEncoderFactory> CreateAndroidDefaultVideoEncoderFactory() {
  return std::make_unique<AndroidDefaultVideoEncoderFactory>();
}
```



### 4.2 AndroidDefaultVideoEncoderFactory

```c++
class AndroidDefaultVideoEncoderFactory : public webrtc::VideoEncoderFactory {
 public:
  AndroidDefaultVideoEncoderFactory();
  std::vector<webrtc::SdpVideoFormat> GetSupportedFormats() const override;

  std::unique_ptr<webrtc::VideoEncoder> CreateVideoEncoder(
      const webrtc::SdpVideoFormat& format) override;


 private:
  std::unique_ptr<webrtc::VideoEncoderFactory> m_hardwareVideoEncoderFactory;
  std::unique_ptr<webrtc::VideoEncoderFactory> m_softwareVideoEncoderFactory;
};
```



#### 4.2.1 AndroidDefaultVideoEncoderFactory::AndroidDefaultVideoEncoderFactory

```c++
AndroidDefaultVideoEncoderFactory::AndroidDefaultVideoEncoderFactory()
{
  m_softwareVideoEncoderFactory = webrtc::CreateBuiltinVideoEncoderFactory();
  m_hardwareVideoEncoderFactory = CreateAndroidHardwareEncoderFactory();
}
```



##### --CreateAndroidHardwareEncoderFactory

【4.3】

#### 4.2.2 AndroidDefaultVideoEncoderFactory::GetSupportedFormats

```c++
std::vector<webrtc::SdpVideoFormat> AndroidDefaultVideoEncoderFactory::GetSupportedFormats()
    const {
    std::vector<webrtc::SdpVideoFormat> supported_codecs;
    std::vector<webrtc::SdpVideoFormat> software_supported_codecs 
      = m_softwareVideoEncoderFactory->GetSupportedFormats();
    std::vector<webrtc::SdpVideoFormat> hardware_supported_codecs 
      = m_hardwareVideoEncoderFactory->GetSupportedFormats();
    supported_codecs.insert(supported_codecs.end()
                            , software_supported_codecs.begin(),
                            software_supported_codecs.end());
    supported_codecs.insert(supported_codecs.end(),
                            hardware_supported_codecs.begin(),
                            hardware_supported_codecs.end());
    return supported_codecs;
}
```



#### 4.2.3 !!! AndroidDefaultVideoEncoderFactory::CreateVideoEncoder

这里注意，设置里主要编码器 是硬件编码器，如果失败了则用软件编码器

```c++
std::unique_ptr<webrtc::VideoEncoder> AndroidDefaultVideoEncoderFactory::CreateVideoEncoder(
    const webrtc::SdpVideoFormat& format) {

  std::unique_ptr<webrtc::VideoEncoder> fallback_encoder =
      m_softwareVideoEncoderFactory->CreateVideoEncoder(format);
  std::unique_ptr<webrtc::VideoEncoder> primary_encoder =
      m_hardwareVideoEncoderFactory->CreateVideoEncoder(format);

  if (primary_encoder && fallback_encoder)
      return webrtc::CreateVideoEncoderSoftwareFallbackWrapper(std::move(fallback_encoder),
                                                std::move(primary_encoder));
  else if (primary_encoder)
    return primary_encoder;
  else 
    return fallback_encoder;
}
```



##### !!! CreateVideoEncoderSoftwareFallbackWrapper

api/video_codecs/video_encoder_software_fallback_wrapper.h

```cpp
std::unique_ptr<VideoEncoder> CreateVideoEncoderSoftwareFallbackWrapper(
    std::unique_ptr<VideoEncoder> sw_fallback_encoder,
    std::unique_ptr<VideoEncoder> hw_encoder,
    bool prefer_temporal_support) {
  return std::make_unique<VideoEncoderSoftwareFallbackWrapper>(
      std::move(sw_fallback_encoder), std::move(hw_encoder),
      prefer_temporal_support);
}
```



##### VideoEncoderSoftwareFallbackWrapper

```
```



### 4.3 CreateAndroidHardwareEncoderFactory

创建硬件编码工厂，c++调用java

```c++
std::unique_ptr<webrtc::VideoEncoderFactory> CreateAndroidHardwareEncoderFactory();
```



```c++
std::unique_ptr<webrtc::VideoEncoderFactory> CreateAndroidHardwareEncoderFactory() {
  JVM* jvm = webrtc::JVM::GetInstance();
  // eglContext
  jobject eglContext = jvm->GetEglContext().obj();

  JNIEnv* env = webrtc::AttachCurrentThreadIfNeeded();
  webrtc::ScopedJavaLocalRef<jclass> factory_class =
      webrtc::GetClass(env, "org/webrtc/HardwareVideoEncoderFactory");
  jmethodID factory_constructor = env->GetMethodID(
      factory_class.obj(), "<init>", "(Lorg/webrtc/EglBase$Context;ZZ)V");
  webrtc::ScopedJavaLocalRef<jobject> factory_object(
      env, env->NewObject(factory_class.obj(), factory_constructor,
                          eglContext /* shared_context */,
                          false /* enable_intel_vp8_encoder */,
                          true /* enable_h264_high_profile */));
  return webrtc::JavaToNativeVideoEncoderFactory(env, factory_object.obj());
}
```



#### 4.3.1 HardwareVideoEncoderFactory

sdk/android/api/org/webrtc/HardwareVideoEncoderFactory.java

java的VideoEncoderFactory

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
  private final boolean enableIntelVp8Encoder;
  private final boolean enableH264HighProfile;
  private final String  extraMediaCodecFile = "sdcard/mediaCodec.xml";
  private final VideoCapabilityParser vcp = new VideoCapabilityParser();

  @Nullable private final Predicate<MediaCodecInfo> codecAllowedPredicate;

  /**
   * Creates a HardwareVideoEncoderFactory that supports surface texture encoding.
   *
   * @param sharedContext The textures generated will be accessible from this context. May be null,
   *                      this disables texture support.
   * @param enableIntelVp8Encoder true if Intel's VP8 encoder enabled.
   * @param enableH264HighProfile true if H264 High Profile enabled.
   */
  public HardwareVideoEncoderFactory(
      EglBase.Context sharedContext, boolean enableIntelVp8Encoder, boolean enableH264HighProfile) {
    this(sharedContext, enableIntelVp8Encoder, enableH264HighProfile,
        /* codecAllowedPredicate= */ null);
  }

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

  @Deprecated
  public HardwareVideoEncoderFactory(boolean enableIntelVp8Encoder, boolean enableH264HighProfile) {
    this(null, enableIntelVp8Encoder, enableH264HighProfile);
  }

	...
}
```



#### 4.3.2 HardwareVideoEncoderFactory.createEncoder

```java
  @Nullable
  @Override
  public VideoEncoder createEncoder(VideoCodecInfo input) {
    // HW encoding is not supported below Android Kitkat.
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
      return null;
    }

    VideoCodecMimeType type = VideoCodecMimeType.valueOf(input.name);
    MediaCodecInfo info = findCodecForType(type);

    if (info == null) {
      return null;
    }

    String codecName = info.getName();
    String mime = type.mimeType();
    Integer surfaceColorFormat = MediaCodecUtils.selectColorFormat(
        MediaCodecUtils.TEXTURE_COLOR_FORMATS, info.getCapabilitiesForType(mime));
    Integer yuvColorFormat = MediaCodecUtils.selectColorFormat(
        MediaCodecUtils.ENCODER_COLOR_FORMATS, info.getCapabilitiesForType(mime));

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

    return new HardwareVideoEncoder(new MediaCodecWrapperFactoryImpl(), codecName, type,
        surfaceColorFormat, yuvColorFormat, input.params, getKeyFrameIntervalSec(type),
        getForcedKeyFrameIntervalMs(type, codecName), createBitrateAdjuster(type, codecName),
        sharedContext);
  }
```



#### 4.3.3 HardwareVideoEncoderFactory.getSupportedCodecs

```java
 @Override
  public VideoCodecInfo[] getSupportedCodecs() {
    // HW encoding is not supported below Android Kitkat.
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
      return new VideoCodecInfo[0];
    }

    List<VideoCodecInfo> supportedCodecInfos = new ArrayList<VideoCodecInfo>();
    // Generate a list of supported codecs in order of preference:
    // VP8, VP9, H264 (high profile), and H264 (baseline profile).
    for (VideoCodecMimeType type : new VideoCodecMimeType[] {
             VideoCodecMimeType.VP8, VideoCodecMimeType.VP9, VideoCodecMimeType.H264}) {
      MediaCodecInfo codec = findCodecForType(type);
      if (codec != null) {
        String name = type.name();
        // TODO(sakal): Always add H264 HP once WebRTC correctly removes codecs that are not
        // supported by the decoder.
        if (type == VideoCodecMimeType.H264 && isH264HighProfileSupported(codec)) {
          supportedCodecInfos.add(new VideoCodecInfo(
              name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ true)));
        }

        supportedCodecInfos.add(new VideoCodecInfo(
            name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ false)));
      }
    }

    return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
  }
```





## 参考

1. [WebRTC Native开发实战之视频编码](https://www.cnblogs.com/xl2432/p/13864448.html)


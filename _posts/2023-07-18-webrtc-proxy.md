---
layout: post
title: webrtc proxy——线程安全
date: 2023-07-18 23:58:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc base
---


* content
{:toc}

---

## 1. 前言

```cpp
rtc::scoped_refptr<AudioTrackInterface> PeerConnectionFactory::CreateAudioTrack(
    const std::string& id,
    AudioSourceInterface* source) {
  RTC_DCHECK(signaling_thread()->IsCurrent());
  rtc::scoped_refptr<AudioTrackInterface> track(AudioTrack::Create(id, source));
  // 分析下这行代码
  return AudioTrackProxy::Create(signaling_thread(), track);
}
```

在源码中找了一圈，没有找到AudioTrackProxy，那么他是从哪里来的呢？？？后来发现`api/media_stream_track_proxy.h` 和 `api/proxy.h`  这两个文件。

```cpp
BEGIN_SIGNALING_PROXY_MAP(AudioTrack)
```

```cpp
#define BEGIN_SIGNALING_PROXY_MAP(c)                                         \
  PROXY_MAP_BOILERPLATE(c)                                                   \
  SIGNALING_PROXY_MAP_BOILERPLATE(c)                                         \
  REFCOUNTED_PROXY_MAP_BOILERPLATE(c)                                        \
 public:                                                                     \
  static rtc::scoped_refptr<c##ProxyWithInternal> Create(                    \
      rtc::Thread* signaling_thread, INTERNAL_CLASS* c) {                    \
    return new rtc::RefCountedObject<c##ProxyWithInternal>(signaling_thread, \
                                                           c);               \
  }
```

这个看起来蛮像的啊 ，接下去我们来分析下proxy吧。
其实在webrtc中，许多重要的对象实际上都是“代理对象”，如PeerConnection， PeerConnectionFactory等。



## 2. 宏定义分析

AudioTrackProxy是通过宏定义产生的，我们正好以AudioTrack代理的产生过程为入口点，一步步分析。
首先，在AudioTrack代理产生在`api/media_stream_track_proxy.h`文件中，该文件是专门为生成AudioTrack代理类而存在的。看上去很复杂，宏种类很多，简化来看，其实这里面的宏定义就4种而已：

```cpp
BEGIN_SIGNALING_PROXY_MAP：用来产生代理类的名称，形如"class AudioTrackProxy {"；
PROXY_SIGNALING_THREAD_DESTRUCTOR：用来产生析新类的构函数；
PROXY_METHOD1~PROXY_METHOD4：用来产生新类的方法，数字代表入参个数；
END_PROXY_MAP()：用来产生类声明的结尾，形如"};"
```



### 2.1 media_stream_track_proxy.h

```cpp
// This file includes proxy classes for tracks. The purpose is
// to make sure tracks are only accessed from the signaling thread.

#ifndef API_MEDIA_STREAM_TRACK_PROXY_H_
#define API_MEDIA_STREAM_TRACK_PROXY_H_

#include <string>

#include "api/media_stream_interface.h"
#include "api/proxy.h"

namespace webrtc {

// TODO(deadbeef): Move this to .cc file and out of api/. What threads methods
// are called on is an implementation detail.

BEGIN_SIGNALING_PROXY_MAP(AudioTrack)
PROXY_SIGNALING_THREAD_DESTRUCTOR()
BYPASS_PROXY_CONSTMETHOD0(std::string, kind)
BYPASS_PROXY_CONSTMETHOD0(std::string, id)
PROXY_CONSTMETHOD0(TrackState, state)
PROXY_CONSTMETHOD0(bool, enabled)
PROXY_CONSTMETHOD0(AudioSourceInterface*, GetSource)
PROXY_METHOD1(void, AddSink, AudioTrackSinkInterface*)
PROXY_METHOD1(void, RemoveSink, AudioTrackSinkInterface*)
PROXY_METHOD1(bool, GetSignalLevel, int*)
PROXY_METHOD0(rtc::scoped_refptr<AudioProcessorInterface>, GetAudioProcessor)
PROXY_METHOD1(bool, set_enabled, bool)
PROXY_METHOD1(void, RegisterObserver, ObserverInterface*)
PROXY_METHOD1(void, UnregisterObserver, ObserverInterface*)
END_PROXY_MAP()

}  // namespace webrtc

#endif  // API_MEDIA_STREAM_TRACK_PROXY_H_
```

这些宏定义都位于`api/proxy.h`的文件中，我们继续分析各个宏定义。



### 2.2 分析proxy.h

#### 2.2.1 BEGIN_SIGNALING_PROXY_MAP

```cpp
#define BEGIN_SIGNALING_PROXY_MAP(c)                                         \
  PROXY_MAP_BOILERPLATE(c)                                                   \
  SIGNALING_PROXY_MAP_BOILERPLATE(c)                                         \
  REFCOUNTED_PROXY_MAP_BOILERPLATE(c)                                        \
 public:                                                                     \
  static rtc::scoped_refptr<c##ProxyWithInternal> Create(                    \
      rtc::Thread* signaling_thread, INTERNAL_CLASS* c) {                    \
    return new rtc::RefCountedObject<c##ProxyWithInternal>(signaling_thread, \
                                                           c);               \
  }
```

该宏可以解释为另外3个宏+一个Create方法，如果将AudioTrack代入进去的话将是

```cpp
 public:                                                                     
  static rtc::scoped_refptr<AudioTrackProxyWithInternal> Create(                    
      rtc::Thread* signaling_thread, INTERNAL_CLASS* c) {                  
    return new rtc::RefCountedObject<AudioTrackProxyWithInternal>(signaling_thread,  c);   
  }
```

代理类名称是AudioTrackProxyWithInternal。

#### 2.2.2 PROXY_MAP_BOILERPLATE

```cpp
#define PROXY_MAP_BOILERPLATE(c)                          \
  template <class INTERNAL_CLASS>                         \
  class c##ProxyWithInternal;                             \
  typedef c##ProxyWithInternal<c##Interface> c##Proxy;    \
  template <class INTERNAL_CLASS>                         \
  class c##ProxyWithInternal : public c##Interface {      \
   protected:                                             \
    typedef c##Interface C;                               \
                                                          \
   public:                                                \
    const INTERNAL_CLASS* internal() const { return c_; } \
    INTERNAL_CLASS* internal() { return c_; }
```

> 小知识：
> ##在宏定义中，是连接符的意思，`c##ProxyWithInternal`。
> 举例：
>
> 1. c 就是 AudioTrack
> 2. `##`是连接符，去除掉
> 3. 最终转换为AudioTrackProxyWithInternal。

代入AudioTrack，变为：

```cpp
  template <class INTERNAL_CLASS>                         
  class AudioTrackProxyWithInternal;                             
  typedef AudioTrackProxyWithInternal<AudioTrackInterface> AudioTrackProxy;    
  template <class INTERNAL_CLASS>                         
  class AudioTrackProxyWithInternal : public AudioTrackInterface {      
   protected:                                             
    typedef AudioTrackInterface C;                                                                          
   public:                                                
    const INTERNAL_CLASS* internal() const { return c_; } 
    INTERNAL_CLASS* internal() { return c_; }
```

这里的`typedef c##Interface C; `  就是`typedef AudioTrackInterface C;`，在Method地方会用到， 参考小节 4.2。



#### 2.2.3 SIGNALING_PROXY_MAP_BOILERPLATE

```cpp
#define SIGNALING_PROXY_MAP_BOILERPLATE(c)                               \
 protected:                                                              \
  c##ProxyWithInternal(rtc::Thread* signaling_thread, INTERNAL_CLASS* c) \
      : signaling_thread_(signaling_thread), c_(c) {}                    \
                                                                         \
 private:                                                                \
  mutable rtc::Thread* signaling_thread_;
```

代入AudioTrack，变为：

```cpp
 protected:                                                              
  AudioTrackProxyWithInternal(rtc::Thread* signaling_thread, INTERNAL_CLASS* c) 
      : signaling_thread_(signaling_thread), c_(c) {}                    
                                                
 private:                                                                
  mutable rtc::Thread* signaling_thread_;
```

#### 2.2.4 REFCOUNTED_PROXY_MAP_BOILERPLATE

```cpp
#define REFCOUNTED_PROXY_MAP_BOILERPLATE(c)            \
 protected:                                            \
  ~c##ProxyWithInternal() {                            \
    MethodCall0<c##ProxyWithInternal, void> call(      \
        this, &c##ProxyWithInternal::DestroyInternal); \
    call.Marshal(RTC_FROM_HERE, destructor_thread());  \
  }                                                    \
                                                       \
 private:                                              \
  void DestroyInternal() { c_ = nullptr; }             \
  rtc::scoped_refptr<INTERNAL_CLASS> c_;
```

代入AudioTrack，变为：

```cpp
 protected:                                            
  ~AudioTrackProxyWithInternal() {                            
    MethodCall0<AudioTrackProxyWithInternal, void> call(      
        this, &AudioTrackProxyWithInternal::DestroyInternal); 
    call.Marshal(RTC_FROM_HERE, destructor_thread());  
  }                                                    
                                                       
 private:                                              
  void DestroyInternal() { c_ = nullptr; }             
  rtc::scoped_refptr<INTERNAL_CLASS> c_;
```

#### 2.2.5 PROXY_SIGNALING_THREAD_DESTRUCTOR

```cpp
#define PROXY_SIGNALING_THREAD_DESTRUCTOR()                            \
 private:                                                              \
  rtc::Thread* destructor_thread() const { return signaling_thread_; } \
                                                                       \
 public:  // NOLINTNEXTLINE
```

#### 2.2.6 PROXY_METHOD

```cpp
#define PROXY_METHOD1(r, method, t1)                           \
  r method(t1 a1) override {                                   \
    MethodCall1<C, r, t1> call(c_, &C::method, std::move(a1)); \
    return call.Marshal(RTC_FROM_HERE, signaling_thread_);     \
  }
```

关于PROXY_METHOD宏系列，举个示例，其他的都基本类似。
以AudioTrack的GetSignalLevel方法为例，宏定义

```cpp
PROXY_METHOD1(bool, GetSignalLevel, int*)
```

转化为：

```cpp
  bool GetSignalLevel(int* a1) override {                                   
    MethodCall1<AudioTrackInterface, bool , a1> call(c_, &AudioTrackInterface::GetSignalLevel, std::move(a1)); 
    return call.Marshal(RTC_FROM_HERE, signaling_thread_);     
  }
```

#### 2.2.7 END_PROXY_MAP()

```cpp
#define END_PROXY_MAP() \
  };
```



### 2.3 完整AudioTrackProxy

通过上面的分析，得到完整的AudioTrackProxy。

```cpp
template <class INTERNAL_CLASS>  class AudioTrackProxyWithInternal;  
typedef AudioTrackProxyWithInternal<AudioTrackInterface> AudioTrackProxy;    
template <class INTERNAL_CLASS>                         
class AudioTrackProxyWithInternal : public AudioTrackInterface {      
protected:                                             
    typedef AudioTrackInterface C;                                                                          
public:                                                
    const INTERNAL_CLASS* internal() const { return c_; } 
    INTERNAL_CLASS* internal() { return c_; }
protected:                                                              
  AudioTrackProxyWithInternal(rtc::Thread* signaling_thread, INTERNAL_CLASS* c) 
      : signaling_thread_(signaling_thread), c_(c) {}                    
private:                                                                
  mutable rtc::Thread* signaling_thread_;
protected:                                            
  ~AudioTrackProxyWithInternal() {                            
    MethodCall0<AudioTrackProxyWithInternal, void> call(      
        this, &AudioTrackProxyWithInternal::DestroyInternal); 
    call.Marshal(RTC_FROM_HERE, destructor_thread());  
  }                                                    
                                                       
 private:                                              
  void DestroyInternal() { c_ = nullptr; }             
  rtc::scoped_refptr<INTERNAL_CLASS> c_;
public:                                                                     
  static rtc::scoped_refptr<AudioTrackProxyWithInternal> Create(                    
      rtc::Thread* signaling_thread, INTERNAL_CLASS* c) {                  
    return new rtc::RefCountedObject<AudioTrackProxyWithInternal>(signaling_thread,  c);   
  }
private:                                                              
  rtc::Thread* destructor_thread() const { return signaling_thread_; } 
                                                                       
public:  // NOLINTNEXTLINE
  ...
    
  bool GetSignalLevel(int* a1) override {                                   
    MethodCall1<AudioTrackInterface, bool , a1> call(c_, &AudioTrackInterface::GetSignalLevel, std::move(a1)); 
    return call.Marshal(RTC_FROM_HERE, signaling_thread_);     
  }
  
  ...
};   
```

可以看出定义了一个template <class INTERNAL_CLASS> AudioTrackProxyWithInternal模板类，该类继承于 AudioTrackInterface；

- 定义了AudioTrackProxy为AudioTrackProxyWithInternal<AudioTrackInterface>是该模板的实例化。

- Create方法创建的是AudioTrackProxy对象，该对象内部持有AudioTrackInterface类别的成员（指向实体对象AudioTrack），以及目标线程对象signaling_thread_。_
- 执行AudioTrackProxy的其他公有方法，比如GetSignalLevel()，内部将会执行AudioTrack的GetSignalLevel()方法，并且该方法是在目标线程signaling_thread_运行的。



## 3. 为什么如此设计

为什么如此设计呢？我们知道WebRTC是个多线程的程序，最直观的感受就是肯定会有三个基础线程signaling_thread（信令），worker_thread（工作），network_thread（网络）。并且另外一个非常重要的点在于WebRTC内部很多对象的方法必须在指定的线程中执行，比如之前分析的PeerConnectionFactory所有成员方法必须在signaling_thread(信令)中执行，并且每个方法的开始位置都有一个断言，形如以下代码所示。如果方法没有在对应线程执行，程序走到这里就crash了。

```cpp
rtc::scoped_refptr<AudioSourceInterface>
PeerConnectionFactory::CreateAudioSource(const cricket::AudioOptions& options) {
  RTC_DCHECK(signaling_thread_->IsCurrent());
  rtc::scoped_refptr<LocalAudioSource> source(
      LocalAudioSource::Create(&options));
  return source;
}
```

站在用户的角度来讲，如果直接面向的是如此接口的话，我们势必需要了解每个接口的具体实现，以便知道使用这些方法时让其在哪个线程上去执行，否则就麻烦了。何况WebRTC的api层对外暴露的都是Interface，应用层哪知道实际持有的到底是哪个Implement类对象呢？更何况应用层还不一定知道所有线程的存在呢，毕竟正如`example/peerconnection_client`示例工程那样，应用层根本就不会去创建上面三个基础线程，都是WebRTC内部创建的。

所以，为了使得用户使用WebRTC时更方便，不用考虑线程，Proxy代理层就合理的，巧妙的诞生了。Proxy代理层使得用户想调用某个对方的某个方法时(实际上操作的是对应的Proxy对象)，将使得方法调用被代理到期待的对象的期待的方法上，并且是在恰当的线程上同步执行。



## 4. 其他

我们看到AudioTrackProxy的所有方法都是需要在信令线程中执行，因此相对来说还比较简单。但是我们会遇到这种情况：一个类的某些方法需要运行在信令线程上，另一些方法需要运行在工作线程上。在api/proxy类中，另外几个宏配合在一起提供了这个功能。

### 4.1 BEGIN_PROXY_MAP

确保了可以同时传入信令线程和工作线程。

```cpp
#define BEGIN_PROXY_MAP(c)                                                    \
  PROXY_MAP_BOILERPLATE(c)                                                    \
  WORKER_PROXY_MAP_BOILERPLATE(c)                                             \
  REFCOUNTED_PROXY_MAP_BOILERPLATE(c)                                         \
 public:                                                                      \
  static rtc::scoped_refptr<c##ProxyWithInternal> Create(                     \
      rtc::Thread* signaling_thread, rtc::Thread* worker_thread,              \
      INTERNAL_CLASS* c) {                                                    \
    return new rtc::RefCountedObject<c##ProxyWithInternal>(signaling_thread,  \
                                                           worker_thread, c); \
  }
```

### 4.2 PROXY_WORKER_METHOD0

确保了方法在工作者线程上执行。

```cpp
#define PROXY_WORKER_METHOD0(r, method)                 \
  r method() override {                                 \
    MethodCall0<C, r> call(c_, &C::method);             \
    return call.Marshal(RTC_FROM_HERE, worker_thread_); \
  }
```

### 4.3 OWNED_PROXY_MAP_BOILERPLATE

```cpp
#define OWNED_PROXY_MAP_BOILERPLATE(c)                 \
 public:                                               \
  ~c##ProxyWithInternal() {                            \
    MethodCall0<c##ProxyWithInternal, void> call(      \
        this, &c##ProxyWithInternal::DestroyInternal); \
    call.Marshal(RTC_FROM_HERE, destructor_thread());  \
  }                                                    \
                                                       \
 private:                                              \
  void DestroyInternal() { delete c_; }                \
  INTERNAL_CLASS* c_;
```

### 4.4 REFCOUNTED_PROXY_MAP_BOILERPLATE

```cpp
#define REFCOUNTED_PROXY_MAP_BOILERPLATE(c)            \
 protected:                                            \
  ~c##ProxyWithInternal() {                            \
    MethodCall0<c##ProxyWithInternal, void> call(      \
        this, &c##ProxyWithInternal::DestroyInternal); \
    call.Marshal(RTC_FROM_HERE, destructor_thread());  \
  }                                                    \
                                                       \
 private:                                              \
  void DestroyInternal() { c_ = nullptr; }             \
  rtc::scoped_refptr<INTERNAL_CLASS> c_;
```

### 4.5 REFCOUNTED_PROXY_MAP_BOILERPLATE 和 OWNED_PROXY_MAP_BOILERPLATE 区别

`REFCOUNTED_PROXY_MAP_BOILERPLATE` 和 `OWNED_PROXY_MAP_BOILERPLATE`两者的区别在于内部成员`c_`的类别，和销毁方式:

- \_owned方式，成员为`INTERNAL_CLASS* c_`，销毁的方式是`delete c_`；直接删除对象。
- refcounted方式，成员为`scoped_refptr<INTERNAL_CLASS> c_`，销毁方式是`c_ = nullptr`；解除引用。
  

## 5. 总结

为了线程安全，通过Proxy代理对象，将所有操作代理到对应的线程上执行对应的方法， 这也是proxy存在的价值。



## 参考

1. [WebRTC源码分析-线程安全之Proxy，防止线程乱入](https://blog.csdn.net/ice_ly000/article/details/103191668)

2. [webrtc proxy 分析](https://blog.csdn.net/liwenlong_only/article/details/80166016)


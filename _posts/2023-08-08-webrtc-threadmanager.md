---
layout: post
title: webrtc ThreadManager
date: 2023-08-08 20:59:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---

## 1. 前言

WebRTC实现了跨平台(Windows，MacOS，Linux，IOS，Android)的线程类rtc::Thread，WebRTC内部的network_thread，worker_thread，signaling_thread均是该类的实例。该类的源码位于rtc_base目录下的thread.h与thread.cc中。

rtc:: Thread及其相关类，ThreadManager、MessageQueue，Runnable等等一起提供了如下的基础功能：

- 线程的管理：通过ThreadManager单例对象，可以管理所有的Thread实例；
- 线程的常规基本功能：rtc:: Thread提供创建线程对象，设置线程名称，启动线程去执行用户代码；
- 消息循环，消息投递：rtc:: Thread通过继承MessageQueue类，提供了内部消息循环，并提供了线程间异步，同步投递消息的功能；
- 跨线程执行方法：提供了跨线程执行方法，并返回执行结果的功能。该功能非常强大，因为WebRTC在某些功能模块的使用上，有要求其必需在指定的线程中才能调用的基本要求，比如音频模块：ADM 的创建必须要在 WebRTC 的 worker thread 中进行；
  多路分离器：通过持有SocketServer对象，实现了多路分离器的功能，能处理网络IO；



## 2. ThreadManager

rtc_base/thread.cc

WebRTC中的线程管理是通过ThreadManager对象来实施的，该类起着牧羊者的作用，rtc::Thread类对象就是羊群。其通过什么样的技术来实现对rtc::Thread管理的？在不同的系统平台下如何实现？下文将进行阐述。

```cpp
class RTC_EXPORT ThreadManager {
 public:
  static const int kForever = -1;

  // Singleton, constructor and destructor are private.
  static ThreadManager* Instance();

  static void Add(Thread* message_queue);
  static void Remove(Thread* message_queue);
  static void Clear(MessageHandler* handler);

  // For testing purposes, for use with a simulated clock.
  // Ensures that all message queues have processed delayed messages
  // up until the current point in time.
  static void ProcessAllMessageQueuesForTesting();

  Thread* CurrentThread();
  void SetCurrentThread(Thread* thread);
  // Allows changing the current thread, this is intended for tests where we
  // want to simulate multiple threads running on a single physical thread.
  void ChangeCurrentThreadForTest(Thread* thread);

  // Returns a thread object with its thread_ ivar set
  // to whatever the OS uses to represent the thread.
  // If there already *is* a Thread object corresponding to this thread,
  // this method will return that.  Otherwise it creates a new Thread
  // object whose wrapped() method will return true, and whose
  // handle will, on Win32, be opened with only synchronization privileges -
  // if you need more privilegs, rather than changing this method, please
  // write additional code to adjust the privileges, or call a different
  // factory method of your own devising, because this one gets used in
  // unexpected contexts (like inside browser plugins) and it would be a
  // shame to break it.  It is also conceivable on Win32 that we won't even
  // be able to get synchronization privileges, in which case the result
  // will have a null handle.
  Thread* WrapCurrentThread();
  void UnwrapCurrentThread();

  bool IsMainThread();

#if RTC_DCHECK_IS_ON
  // Registers that a Send operation is to be performed between |source| and
  // |target|, while checking that this does not cause a send cycle that could
  // potentially cause a deadlock.
  void RegisterSendAndCheckForCycles(Thread* source, Thread* target);
#endif

 private:
  ThreadManager();
  ~ThreadManager();

  void SetCurrentThreadInternal(Thread* thread);
  void AddInternal(Thread* message_queue);
  void RemoveInternal(Thread* message_queue);
  void ClearInternal(MessageHandler* handler);
  void ProcessAllMessageQueuesInternal();
#if RTC_DCHECK_IS_ON
  void RemoveFromSendGraph(Thread* thread) RTC_EXCLUSIVE_LOCKS_REQUIRED(crit_);
#endif

  // This list contains all live Threads.
  std::vector<Thread*> message_queues_ RTC_GUARDED_BY(crit_);

  // Methods that don't modify the list of message queues may be called in a
  // re-entrant fashion. "processing_" keeps track of the depth of re-entrant
  // calls.
  RecursiveCriticalSection crit_;
  size_t processing_ RTC_GUARDED_BY(crit_) = 0;
#if RTC_DCHECK_IS_ON
  // Represents all thread seand actions by storing all send targets per thread.
  // This is used by RegisterSendAndCheckForCycles. This graph has no cycles
  // since we will trigger a CHECK failure if a cycle is introduced.
  std::map<Thread*, std::set<Thread*>> send_graph_ RTC_GUARDED_BY(crit_);
#endif

#if defined(WEBRTC_POSIX)
  pthread_key_t key_;
#endif

#if defined(WEBRTC_WIN)
  const DWORD key_;
#endif

  // The thread to potentially autowrap.
  const PlatformThreadRef main_thread_ref_;

};

```

### 2.1 属性

- std::vector<Thread*> message_queues_ ；存放管理的线程
- const PlatformThreadRef main_thread_ref_; 主线程Id



### 2.2 ThreadManager::Instance

ThreadManager实现为单例模式，通过静态方法Instance()来获取唯一的实例。其构造与 析构函数均声明为private。

```cpp
ThreadManager* ThreadManager::Instance() {
  static ThreadManager* const thread_manager = new ThreadManager();
  return thread_manager;
}
```

#### 2.2.1 如何保证线程安全

该方法很简单，但是注意这个方法不是线程安全的，那么在WebRTC的多线程环境下是如何保证ThreadManager对象被安全的构造？WebRTC通过一定机制确保了Instance()方法第一次的调用肯定是在单线程的环境下，也即在主线程中被调用，因此是线程安全的。如何实现这点？

1. WebRTC中启动新线程的标准方法是通过创建Thread对象，然后调用Thread.Start()方法来启用新的线程，而该方法的内部会直接调用一次Insance()，如下

```cpp
bool Thread::Start() {
  RTC_DCHECK(!IsRunning());

  if (IsRunning())
    return false;

  Restart();  // reset IsQuitting() if the thread is being restarted

  // Make sure that ThreadManager is created on the main thread before
  // we start a new thread.
  ThreadManager::Instance();
  ...
}
```

2. WebRTC启动新线程的非标准方法，即用户继承了Thread对象，并且不能通过Thread.Start()方法来启用新线程。此时，WebRTC中是如何保证这点的？如下，Thread的WrapCurrent()方法的说明以及其实现说明了此种情况：

```cpp
  // These functions are public to avoid injecting test hooks. Don't call them
  // outside of tests.
  // This method should be called when thread is created using non standard
  // method, like derived implementation of rtc::Thread and it can not be
  // started by calling Start(). This will set started flag to true and
  // owned to false. This must be called from the current thread.
  bool WrapCurrent();
```

继承Thread的类，如果不能通过Thread.Start()来启动线程时，应该在构造中调用WrapCurrent()方法，该方法如下图所示，首先就会调用ThreadManager::Instance()来获取ThreadManager的单例对象。

```cpp
bool Thread::WrapCurrent() {
  return WrapCurrentWithThreadManager(ThreadManager::Instance(), true);
}
```



### 2.3 ThreadManager::ThreadManager

rtc_base/thread.cc

```cpp
#if defined(WEBRTC_POSIX)
ThreadManager::ThreadManager() : main_thread_ref_(CurrentThreadRef()) {
#if defined(WEBRTC_MAC)
  InitCocoaMultiThreading();
#endif
  pthread_key_create(&key_, nullptr);
}

Thread* ThreadManager::CurrentThread() {
  return static_cast<Thread*>(pthread_getspecific(key_));
}

void ThreadManager::SetCurrentThreadInternal(Thread* thread) {
  pthread_setspecific(key_, thread);
}
#endif

#if defined(WEBRTC_WIN)
ThreadManager::ThreadManager()
    : key_(TlsAlloc()), main_thread_ref_(CurrentThreadRef()) {}

Thread* ThreadManager::CurrentThread() {
  return static_cast<Thread*>(TlsGetValue(key_));
}

void ThreadManager::SetCurrentThreadInternal(Thread* thread) {
  TlsSetValue(key_, thread);
}
#endif
```

1. Windows和类Unix系统中实现进行了区分，WEBRTC_POSIX宏表征该系统是类Unix系统，而WEBRTC_WIN宏表征是Windows系统。虽然实现稍微有些许不同，在Mac下还需要调用InitCocoaMultiThreading()方法来初始化多线程库。但是两个构造函数均初始化了成员key_ 与main_thread_ref_ 。其中key_ 是线程管理的关键。

2. key_ 的初始化：在Windows平台上，key_ 被声明为DWORD类型，赋值为TlsAlloc()函数的返回值，TlsAlloc()函数是Windows的系统API，Tls表示的是线程局部存储Thread Local Storage的缩写，其为每个可能的线程分配了一个线程局部变量的槽位，该槽位用来存储WebRTC的Thread线程对象指针。如果不了解相关概念，可以看微软的官方文档，或者TLS--线程局部存储这篇博客来了解。
   在类Unix系统上，key_ 被声明pthread_key_t类型，使用方法pthread_key_create(&key_, nullptr);赋值。实质是类Unix系统上的线程局部存储实现，隶属于线程库pthread，因此方法与变量均以pthread开头。总之，在ThreadManager的构造之初，WebRTC就为各个线程所对应的Thread对象制造了一个线程局部变量的槽位，成为多线程管理的关键。

   > set是把一个变量的地址告诉key，一般放在变量定义之后，get会把这个地址读出来，然后你自己转义成相应的类型再去操作，注意变量的有效期。
   > 只不过，**在不同的线程里可以操作同一个key，他们不会冲突，比如线程a,b,c set同样的key，分别get得到的地址会是之前各自传进去的值。**
   > 这样做的意义在于，可以写一份线程代码，通过key的方式多线程操作不同的数据。
   > [多线程私有数据pthread_key_create](https://blog.csdn.net/qixiang2013/article/details/126126112)

3. main_thread_ref_ 的初始化：该成员为PlatformThreadRef类型的对象，赋值为CurrentThreadRef()方法的返回值，如下源码所示：
   rtc_base/platform_thread_types.h

   ```cpp
   PlatformThreadRef CurrentThreadRef() {
   #if  defined(WEBRTC_WIN)
           return GetCurrentThreadId();
   #elif  defined(WEBRTC_FUCHSIA)
           return zx_thread_self();
   #elif  defined(WEBRTC_POSIX)
           return pthread_self();
   #endif
   }
   
   
   #if defined(WEBRTC_WIN)
   typedef DWORD PlatformThreadId;
   typedef DWORD PlatformThreadRef;
   #elif defined(WEBRTC_FUCHSIA)
   typedef zx_handle_t PlatformThreadId;
   typedef zx_handle_t PlatformThreadRef;
   #elif defined(WEBRTC_POSIX)
   typedef pid_t PlatformThreadId;
   typedef pthread_t PlatformThreadRef;
   #endif
   ```

   在Windows系统下，取值为WinAPI GetCurrentThreadId()返回的当前线程描述符，DWORD类型；
   在FUCHSIA系统下(该系统是Google新开发的操作系统，像Android还是基于Linux内核属于类Unix范畴，遵循POSIX规范，但FUCHSIA是基于新内核zircon开发的)，返回zx_thread_self()，zx_handle_t类型；
   在类Unix系统下，通过pthread库的pthread_self()返回，pthread_t类型。
   总之，如前文所述，这部分代码肯定是在主线程中所运行，因此，main_thread_ref_存储了主线程TID在不同平台下的不同表示。

   

### 2.4 ThreadManager::~ThreadManager

该函数应该不会被调用。

```cpp
ThreadManager::~ThreadManager() {
  // By above RTC_DEFINE_STATIC_LOCAL.
  RTC_NOTREACHED() << "ThreadManager should never be destructed.";
}
```



### 2.5 !!! ThreadManager::CurrentThread && ThreadManager::SetCurrentThread

```cpp
#if defined(WEBRTC_POSIX)
Thread* ThreadManager::CurrentThread() {
  return static_cast<Thread*>(pthread_getspecific(key_));
}

void ThreadManager::SetCurrentThreadInternal(Thread* thread) {
  pthread_setspecific(key_, thread);
}
#endif

#if defined(WEBRTC_WIN)
Thread* ThreadManager::CurrentThread() {
  return static_cast<Thread*>(TlsGetValue(key_));
}

void ThreadManager::SetCurrentThreadInternal(Thread* thread) {
  TlsSetValue(key_, thread);
}
#endif

void ThreadManager::SetCurrentThread(Thread* thread) {
	...
  if (thread) {
    thread->EnsureIsCurrentTaskQueue();
  } else {
    Thread* current = CurrentThread();
    if (current) {
      // The current thread is being cleared, e.g. as a result of
      // UnwrapCurrent() being called or when a thread is being stopped
      // (see PreRun()). This signals that the Thread instance is being detached
      // from the thread, which also means that TaskQueue::Current() must not
      // return a pointer to the Thread instance.
      current->ClearCurrentTaskQueue();
    }
  }

  SetCurrentThreadInternal(thread);
}
```

在ThreadManager的构造之初就为Thread指针分配了线程局部存储的槽位key_，通过不同平台的get，set方法就可以将当前线程所关联的Thread对象指针从该槽位取出或设置进去。但是，有这么几个点需要注意：

1. **Thread是用户层线程的表征，可以通过其来访问，操作该线程在内核中的数据结构。但用户层和内核层的线程表征，二者并非是共存关系（有系统的线程，比如pthread对象，未必有rtc::Thread 对象）**。以主线程来说，进程一运行起来其线程内核结构就存在，但是用户层主线程的表征Thread对象是不存在的，因此，在程序入口main()函数开头调用ThreadManager::CurrentThread()方法，得到的必然是空指针。
   **如果想要将主线程纳入管理，必然要先创建一个Thread对象，然后调用ThreadManager::SetCurrentThread(Thread* thread)设置到当前线程的线程局部存储的槽位中**。正如example目录下的peerconnection_client示例工程那样做的，其中Win32Thread就是Thread类的子类。

   ```cpp
   	rtc::WinsockInitializer winsock_init;
     rtc::Win32SocketServer w32_ss;
     rtc::Win32Thread w32_thread(&w32_ss);
     rtc::ThreadManager::Instance()->SetCurrentThread(&w32_thread);
   ```

2. **对于非主线程**，如何纳入管理？主线程外，WebRTC的其他线程以Thread.Start()来启动，新的线程中会执行Thread.PreRun()方法。该方法中就调用了ThreadManager::SetCurrentThread(Thread* thread)方法将新的线程纳入ThreadManager的管理，在线程结束后，调用ThreadManager::SetCurrentThread(nullptr)解除管理。

> - **对于已经存在的线程（比如住线程），不是通过rtc::Thread 创建，Thread.Start()启动的线程，则这时候是没有rtc::Thread 对象。这时候需要创建一个rtc::Thread 对象，ThreadManager::SetCurrentThread(Thread* thread)，把该已经存在的线程加入到TheadManager中管理；**
> - **对于新建的的线程，就是通过rtc::Thread 创建，Thread.Start()启动的线程，则会自动加入到ThreadManager中管理；**

#### 2.5.1 EnsureIsCurrentTaskQueue

```cpp
// Called by the ThreadManager when being set as the current thread.
void Thread::EnsureIsCurrentTaskQueue() {
  task_queue_registration_ =
      std::make_unique<TaskQueueBase::CurrentTaskQueueSetter>(this);
}
```



#### 2.5.2 Thread::ClearCurrentTaskQueue

```cpp
// Called by the ThreadManager when being set as the current thread.
void Thread::ClearCurrentTaskQueue() {
  task_queue_registration_.reset();
}
```



#### 2.5.3 std::unique_ptr\<TaskQueueBase::CurrentTaskQueueSetter> task_queue_registration_;

api/task_queue/task_queue_base.cc

```cpp
  class CurrentTaskQueueSetter {
   public:
    explicit CurrentTaskQueueSetter(TaskQueueBase* task_queue);
    CurrentTaskQueueSetter(const CurrentTaskQueueSetter&) = delete;
    CurrentTaskQueueSetter& operator=(const CurrentTaskQueueSetter&) = delete;
    ~CurrentTaskQueueSetter();

   private:
    TaskQueueBase* const previous_;
  };


TaskQueueBase::CurrentTaskQueueSetter::CurrentTaskQueueSetter(
    TaskQueueBase* task_queue)
    : previous_(TaskQueueBase::Current()) {
  pthread_setspecific(GetQueuePtrTls(), task_queue);
}

TaskQueueBase::CurrentTaskQueueSetter::~CurrentTaskQueueSetter() {
  pthread_setspecific(GetQueuePtrTls(), previous_);
}

```



### 2.6 ThreadManager::WrapCurrentThread

```cpp
Thread* ThreadManager::WrapCurrentThread() {
  Thread* result = CurrentThread();
  if (nullptr == result) {
    result = new Thread(SocketServer::CreateDefault());
    result->WrapCurrentWithThreadManager(this, true);
  }
  return result;
}
```

如果已经有Thread对象与当前线程关联，那么直接返回该对象。否则构造一个新的Thread对象，并通过该对象的WrapCurrentWithThreadManager()方法将新建的Thread对象纳入ThreadManager的管理之中。



#### 2.6.1 Thread::WrapCurrentWithThreadManager

```cpp
bool Thread::WrapCurrentWithThreadManager(ThreadManager* thread_manager,
                                          bool need_synchronize_access) {
#if defined(WEBRTC_WIN)
  if (need_synchronize_access) {
    // We explicitly ask for no rights other than synchronization.
    // This gives us the best chance of succeeding.
    thread_ = OpenThread(SYNCHRONIZE, FALSE, GetCurrentThreadId());
    if (!thread_) {
      RTC_LOG_GLE(LS_ERROR) << "Unable to get handle to thread.";
      return false;
    }
    thread_id_ = GetCurrentThreadId();
  }
#elif defined(WEBRTC_POSIX)
  thread_ = pthread_self();
#endif
  owned_ = false;
  thread_manager->SetCurrentThread(this);
  return true;
}

bool Thread::IsRunning() {
#if defined(WEBRTC_WIN)
  return thread_ != nullptr;
#elif defined(WEBRTC_POSIX)
  return thread_ != 0;
#endif
}
```

在Windows系统与类Unix系统下的差别一点在于Thread.thread_ 的赋值方式。Windows系统上，使用OpenThread()来打开当前已存在的线程，获取其句柄，此处注明只获取该线程的同步操作权限，也即在该线程进行Wait等操作，这样能提高该方法的成功率；而类Unix系统上，使用pthread库的pthread_self()方法来获取当前线程pthread_t对象。另外将Thread.owned_标志位置位false，表示该线程对象是通过wrap而来，而非调用Thread.Start的标准方式而来。最后使用 ThreadManager. SetCurrentThread方法将新创建的Thread对象纳入管理。


### 2.7 ThreadManager::IsMainThread

```cpp
bool ThreadManager::IsMainThread() {
  return IsThreadRefEqual(CurrentThreadRef(), main_thread_ref_);
}

bool IsThreadRefEqual(const PlatformThreadRef& a, const PlatformThreadRef& b) {
#if defined(WEBRTC_WIN) || defined(WEBRTC_FUCHSIA)
  return a == b;
#elif defined(WEBRTC_POSIX)
  return pthread_equal(a, b);
#endif
}

//----------------对main_thread_ref_ 进行赋值------------

#if defined(WEBRTC_POSIX)
ThreadManager::ThreadManager() : main_thread_ref_(CurrentThreadRef()) {
#if defined(WEBRTC_MAC)
  InitCocoaMultiThreading();
#endif
  pthread_key_create(&key_, nullptr);
}

#endif

#if defined(WEBRTC_WIN)
ThreadManager::ThreadManager()
    : key_(TlsAlloc()), main_thread_ref_(CurrentThreadRef()) {}
#end
```

main_thread_ref_ 就是ThreadManager::ThreadManager 创建的时候，也就是ThreadManager::Instance() 第一次调用的线程。



#### 2.7.1 CurrentThreadRef

rtc_base/platform_thread_types.cc

```cpp
PlatformThreadRef CurrentThreadRef() {
#if defined(WEBRTC_WIN)
  return GetCurrentThreadId();
#elif defined(WEBRTC_FUCHSIA)
  return zx_thread_self();
#elif defined(WEBRTC_POSIX)
  return pthread_self();
#endif
}
```



### 2.8 !!! ThreadManager::WrapCurrentThread 和 ThreadManager::SetCurrentThread区别

1. `ThreadManager::WrapCurrentThread` ，会先通过 ThreadManager::CurrentThread， 如果发现返回的rtc::Thread 是空的，则创建rtc::Thread 的对象；同时调用rtc::Thread::WrapCurrentWithThreadManager, 该函数会通过pthread_self，得到当前调用线程的线程，调用`ThreadManager::SetCurrentThread`, 把创建的rtc::Thread对象 存放到ThreadManager 进行管理；

   > 注意： 这里知识创建了rtc::Thread对象，当时没有创建真正的线程，而是使用了当前的线程

2. `ThreadManager::SetCurrentThread`,**如果想要将主线程纳入管理，必然要先创建一个Thread对象，然后调用ThreadManager::SetCurrentThread(Thread* thread)设置到当前线程的线程局部存储的槽位中**。正如example目录下的peerconnection_client示例工程那样做的，其中Win32Thread就是Thread类的子类。

   ```cpp
   	rtc::WinsockInitializer winsock_init;
     rtc::Win32SocketServer w32_ss;
     rtc::Win32Thread w32_thread(&w32_ss);
     rtc::ThreadManager::Instance()->SetCurrentThread(&w32_thread);
   ```

   > 注意： 这里其实也是没有创建线程，而只是创建了rtc::Thread对象，而是用了调用线程，这里是主线程。
   > 这里没有对`rtc::Thread::thread_` 进行赋值。

3. 共同点：都没有创建新的线程，而是使用了现已经存在的线程



## 参考

[多线程私有数据pthread_key_create](https://blog.csdn.net/qixiang2013/article/details/126126112)

[WebRTC源码分析-线程基础之线程管理](https://blog.csdn.net/ice_ly000/article/details/103178691)

[WebRTC源码分析-线程基础之MessageQueue](https://www.jianshu.com/p/b7372d01b819)
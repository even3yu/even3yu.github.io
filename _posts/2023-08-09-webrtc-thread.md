---
layout: post
title: webrtc Thread
date: 2023-08-09 19:45:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

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

[参考前文](https://even3yu.github.io/2023/08/08/webrtc-threadmanager/)



## 3. Thread

rtc::Thread类封装了WebRTC中线程的功能,

- 一般功能，设置线程名称，启动线程执行用户代码，线程的join，sleep，run，stop等方法
- 同时也提供了线程内部的消息循环，以及线程之间以同步、异步方式投递消息，同步方式在目标线程执行方法并返回结果等线程之间交互的方式；
- 每个线程均持有SocketServer类成员对象，该类实现了IO多路复用功能。



### 3.1 类图

![thread]({{ site.url }}{{ site.baseurl }}/images/thread.png)

### 3.2 属性

- pthread_t thread_， 表示线程；
- volatile int stop_; 线程的运行状况，是否停止；`Thread::Quit()`的时候设置为1；`Thread::IsQuitting` 判断是否为1； `Thread::Restart()`重置为0；
- bool owned_，线程对象Thread不是Wrap而来。 owned\_ 设置为true，表征Thread对象是常规的Start()方法启用的，而非Wrap而来。(Thread 创建的两种方式，一种Wrap，一种是Thread::Start)
- SocketServer* const ss_ ; std::unique_ptr<SocketServer> own_ss_ ; 存放了SocketServer；
- MessageList messages_，存放rtc::Message;
- PriorityQueue delayed_messages_，存放 延迟rtc::Message;



### 3.3 线程创建

#### 3.3.1 CreateWithSocketServer

```cpp
std::unique_ptr<Thread> Thread::CreateWithSocketServer() {
  return std::unique_ptr<Thread>(new Thread(SocketServer::CreateDefault()));
}
```



#### 3.3.2 Create

```cpp
std::unique_ptr<Thread> Thread::Create() {
  return std::unique_ptr<Thread>(
      new Thread(std::unique_ptr<SocketServer>(new NullSocketServer())));
}
```



#### 3.3.3 Thread::Thread

四个构造函数，最终都会调用`Thread(SocketServer* ss, bool do_init)`;
唯一区别`std::unique_ptr<SocketServer> ss` 保存到 `std::unique_ptr<SocketServer> own_ss_`;
`SocketServer* const ss_` 保存了指针　`SocketServer* const ss_`;

```cpp
Thread::Thread(SocketServer* ss)
Thread::Thread(std::unique_ptr<SocketServer> ss)
Thread::Thread(SocketServer* ss, bool do_init)
Thread::Thread(std::unique_ptr<SocketServer> ss, bool do_init)
```



```cpp
Thread::Thread(SocketServer* ss, bool do_init)
  ...
      ss_(ss) {
	...
	// 设置queue给SocketServer
  ss_->SetMessageQueue(this);
  SetName("Thread", this);  // default name
  if (do_init) {
    DoInit();
  }
}
```



```cpp
Thread::Thread(std::unique_ptr<SocketServer> ss, bool do_init)
    : Thread(ss.get(), do_init) {
  own_ss_ = std::move(ss);
}
```



#### 3.3.4 Thread::DoInit

```cpp
void Thread::DoInit() {
  if (fInitialized_) {
    return;
  }

  fInitialized_ = true;
  ThreadManager::Add(this);
}
```



#### 3.3.5 ThreadManager::Add

```cpp
void ThreadManager::Add(Thread* message_queue) {
  return Instance()->AddInternal(message_queue);
}
```

Static 方法



#### 3.3.6 ThreadManager::AddInternal

```cpp
void ThreadManager::AddInternal(Thread* message_queue) {
  CritScope cs(&crit_);
  message_queues_.push_back(message_queue);
}
```

- std::vector<Thread*> message_queues_，包Thread 保存到vector中



### 3.4 Thread::Start()

- 创建ThreadManager::Instance()，确保ThreadManager在主线程中构建。
- owned_，线程对象Thread不是Wrap而来。 owned\_ 设置为true，表征Thread对象是常规的Start()方法启用的，而非Wrap而来。(Thread 创建的两种方式，一种Wrap，一种是Thread::Start)
- 创建线程，回调函数PreRun。

```cpp
bool Thread::Start() {
  if (IsRunning())
    return false;
	// 复位消息循环stop标志位
  Restart();  // reset IsQuitting() if the thread is being restarted

  // Make sure that ThreadManager is created on the main thread before
  // we start a new thread.
  // 确保ThreadManager在主线程中构建
  ThreadManager::Instance();
	// 线程对象Thread不是Wrap而来
  owned_ = true;

#if defined(WEBRTC_WIN)
  thread_ = CreateThread(nullptr, 0, PreRun, this, 0, &thread_id_);
  if (!thread_) {
    return false;
  }
#elif defined(WEBRTC_POSIX)
  pthread_attr_t attr;
  pthread_attr_init(&attr);

  int error_code = pthread_create(&thread_, &attr, PreRun, this);
  if (0 != error_code) {
    RTC_LOG(LS_ERROR) << "Unable to create pthread, error " << error_code;
    thread_ = 0;
    return false;
  }
#endif
  return true;
}
```



#### 3.4.1 Thread::IsRunning

```cpp
bool Thread::IsRunning() {
#if defined(WEBRTC_WIN)
  return thread_ != nullptr;
#elif defined(WEBRTC_POSIX)
  return thread_ != 0;
#endif
}
```

通过Thread ::Start, 对thread_ 进行了赋值，创建了线程；



#### 3.4.2 Thread::Restart

是父类MQ的方法， 作用是原子操作将stop_置为0，表征该MQ未停止。

```cpp
void Thread::Restart() {
  AtomicOps::ReleaseStore(&stop_, 0);
}
```



#### 3.4.3 !!! Thread::PreRun——static 方法

```cpp
// static
#if defined(WEBRTC_WIN)
DWORD WINAPI Thread::PreRun(LPVOID pv) {
#else
void* Thread::PreRun(void* pv) {
#endif
  Thread* thread = static_cast<Thread*>(pv);
  // 1. 将新创建的Thread对象纳入管理，与当前线程进行绑定， 就是把thread 存放到thread 的tls中
  ThreadManager::Instance()->SetCurrentThread(thread);
  rtc::SetCurrentThreadName(thread->name_.c_str());
  // 2. 如果是iOS/Mac系统，通过pool对象的创建和析构来使用oc的自动释放池技术，进行内存回收。
#if defined(WEBRTC_MAC)
  ScopedAutoReleasePool pool;
#endif
  // 3. Run, while 循环
  thread->Run();

  // 4. 到此，线程主要的活儿已干完，以下做清理工作
  // 将线程对象与当前线程解绑
  ThreadManager::Instance()->SetCurrentThread(nullptr);
#ifdef WEBRTC_WIN
  return 0;
#else
  return nullptr;
#endif
}  // namespace rtc
```

1）新线程上PreRun()方法执行起来后，ThreadManager立马将当前线程与该Thread对象关联起来，纳入管理之中，当PreRun()方法要执行完毕了，又将当前线程与Thread对象解绑，毕竟该方法退出后，线程就会停止。
2）启动了一个消息循环不停地在此执行。



### 3.5 !!! Thread::Run && Thread::ProcessMessages ——消息循环

PreRun 最终调用 Thread::Run()，不断调用Get 函数获取执行的Message ，Get 方法，如果没有消息会阻塞，获取消息后，调用Dispatch （MessageQueue::Dispatch ）进行执行调用。

```cpp
// static const int kForever = -1;
void Thread::Run() {
  ProcessMessages(kForever);
}

bool Thread::ProcessMessages(int cmsLoop) {
  // Using ProcessMessages with a custom clock for testing and a time greater
  // than 0 doesn't work, since it's not guaranteed to advance the custom
  // clock's time, and may get stuck in an infinite loop.
  RTC_DCHECK(GetClockForTesting() == nullptr || cmsLoop == 0 ||
             cmsLoop == kForever);
  // cmsLoop = kForever， msEnd = 0； 一直获取消息
  // 
  int64_t msEnd = (kForever == cmsLoop) ? 0 : TimeAfter(cmsLoop);
  // 下次获取消息的时间
  int cmsNext = cmsLoop;

  while (true) {
#if defined(WEBRTC_MAC)
    ScopedAutoReleasePool pool;
#endif
    Message msg;
    // 1. Get Message
    if (!Get(&msg, cmsNext))
      return !IsQuitting();
    // 2. Dispatch Message
    Dispatch(&msg);

    if (cmsLoop != kForever) {
      cmsNext = static_cast<int>(TimeUntil(msEnd));
      if (cmsNext < 0)
        return true;
    }
  }
}
```



#### 3.5.1 static const int kForever = -1



#### 3.5.2 Thread::Get

获取消息，见后文 4.3 小节



#### 3.5.3 Thread::Dispatch

执行消息



#### 3.5.4 Thread::IsQuitting

```cpp
bool Thread::IsQuitting() {
  return AtomicOps::AcquireLoad(&stop_) != 0;
}
```



### 3.6 Thread::Stop

```cpp
void Thread::Stop() {
  Thread::Quit();
  Join();
}
```

停止一个线程，可以通过调用线程的Thread.Stop()方法来实施，**但千万不能在当前线程上调用该方法来终止自己**。MQ的Quit()方法会在介绍消息循环时来详细解释，此处作用就是停止线程消息循环。Join()方法在后面介绍，此处作用是阻塞地等待目标线程终止，因此，Stop函数一般会阻塞当前线程。



#### 3.6.1 Thread::Quit

```cpp
void Thread::Quit() {
  AtomicOps::ReleaseStore(&stop_, 1);
  WakeUpSocketServer();
}
```



#### 3.6.2 Thread::WakeUpSocketServer

```cpp
void Thread::WakeUpSocketServer() {
  ss_->WakeUp();
}
```



#### 3.6.3 Thread::Join

```cpp
void Thread::Join() {
  if (!IsRunning())
    return;

  if (Current() && !Current()->blocking_calls_allowed_) {
    RTC_LOG(LS_WARNING) << "Waiting for the thread to join, "
                           "but blocking calls have been disallowed";
  }

#if defined(WEBRTC_WIN)
  WaitForSingleObject(thread_, INFINITE);
  CloseHandle(thread_);
  thread_ = nullptr;
  thread_id_ = 0;
#elif defined(WEBRTC_POSIX)
  pthread_join(thread_, nullptr);
  thread_ = 0;
#endif
}
```



### 3.7 Thread::WrapCurrent

```cpp
bool Thread::WrapCurrent() {
  return WrapCurrentWithThreadManager(ThreadManager::Instance(), true);
}
```



```cpp
bool Thread::WrapCurrentWithThreadManager(ThreadManager* thread_manager,
                                          bool need_synchronize_access) {
  RTC_DCHECK(!IsRunning());

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
```



### 3.8 Thread::UnwrapCurrent()

```cpp
void Thread::UnwrapCurrent() {
  // Clears the platform-specific thread-specific storage.
  ThreadManager::Instance()->SetCurrentThread(nullptr);
#if defined(WEBRTC_WIN)
  if (thread_ != nullptr) {
    if (!CloseHandle(thread_)) {
      RTC_LOG_GLE(LS_ERROR)
          << "When unwrapping thread, failed to close handle.";
    }
    thread_ = nullptr;
    thread_id_ = 0;
  }
#elif defined(WEBRTC_POSIX)
  thread_ = 0;
#endif
}
```



### 3.9 Thread::SleepMs

```cpp
  // Sleeps the calling thread for the specified number of milliseconds, during
  // which time no processing is performed. Returns false if sleeping was
  // interrupted by a signal (POSIX only).
bool Thread::SleepMs(int milliseconds) {
  AssertBlockingIsAllowedOnCurrentThread();

#if defined(WEBRTC_WIN)
  ::Sleep(milliseconds);
  return true;
#else
  // POSIX has both a usleep() and a nanosleep(), but the former is deprecated,
  // so we use nanosleep() even though it has greater precision than necessary.
  struct timespec ts;
  ts.tv_sec = milliseconds / 1000;
  ts.tv_nsec = (milliseconds % 1000) * 1000000;
  int ret = nanosleep(&ts, nullptr);
  if (ret != 0) {
    RTC_LOG_ERR(LS_WARNING) << "nanosleep() returning early";
    return false;
  }
  return true;
#endif
}
```



## 调用堆栈

### Thread::CreateWithSocketServer

```cpp
Thread::CreateWithSocketServer
SocketServer::CreateDefault
	new rtc::NullSocketServer/rtc::PhysicalSocketServer
Thread::Thread
```



### Thread::Start

```cpp
  Thread::Start()
    Thread::IsRunning()
    Thread::Restart()
    ThreadManager::Instance() // 重点
    pthread_create

  Thread::PreRun
    ThreadManager::SetCurrentThread
      Thread::EnsureIsCurrentTaskQueue
      TaskQueueBase::CurrentTaskQueueSetter
      	pthread_setspecific
    	ThreadManager::SetCurrentThreadInternal
    		pthread_setspecific
    SetCurrentThreadName
    Thread::Run 
      Thread::ProcessMessages
        Thread::Get
        Thread::IsQuitting
        Thread::Dispatch
    ThreadManager::SetCurrentThread
    	Thread::ClearCurrentTaskQueue
    	TaskQueueBase::reset
    	ThreadManager::SetCurrentThreadInternal
    		pthread_setspecific
```



### Thread::Stop

```cpp
Thread::Stop
  Thread::Quit()
  	AtomicOps::ReleaseStore
  	Thread::WakeUpSocketServer
  	SocketServer::WakeUp
  Thread::Join()
    Thread::IsRunning
    Thread::IsCurrent
  	ThreadManager::CurrentThread()
    	TlsGetValue
  	pthread_join
```



### Thread::WrapCurrent

```cpp
Thread::WrapCurrent
Thread::WrapCurrentWithThreadManager
  pthread_self
  ThreadManager::SetCurrentThread
        Thread::EnsureIsCurrentTaskQueue
        TaskQueueBase::CurrentTaskQueueSetter
          pthread_setspecific
        ThreadManager::SetCurrentThreadInternal
          pthread_setspecific
  
```



### Thread::UnwrapCurrent

```cpp
Thread::UnwrapCurrent
ThreadManager::SetCurrentThread
  Thread::ClearCurrentTaskQueue
  TaskQueueBase::reset
  ThreadManager::SetCurrentThreadInternal
    pthread_setspecific
  
```



### Thread::Current

```cpp
ThreadManager::Instance
ThreadManager::CurrentThread
 	pthread_getspecific
```



### ThreadManager::WrapCurrentThread()

```cpp
ThreadManager::CurrentThread
Thread::Thread
Thread::WrapCurrentWithThreadManager
  pthread_self
  ThreadManager::SetCurrentThread
        Thread::EnsureIsCurrentTaskQueue
        TaskQueueBase::CurrentTaskQueueSetter
          pthread_setspecific
        ThreadManager::SetCurrentThreadInternal
          pthread_setspecific
  
```



### Thread::Post

```cpp
Thread::Post
Thread::IsQuitting
    std::list<Message> messages_.push_back
Thread::WakeUpSocketServer
  SocketServer::WakeUp
```



### Thread::ProcessMessages

```cpp
Thread::ProcessMessages
Thread::Get
```



## 参考

[多线程私有数据pthread_key_create](https://blog.csdn.net/qixiang2013/article/details/126126112)

[WebRTC源码分析-线程基础之线程管理](https://blog.csdn.net/ice_ly000/article/details/103178691)

[WebRTC源码分析-线程基础之MessageQueue](https://www.jianshu.com/p/b7372d01b819)



```cpp
class RTC_LOCKABLE RTC_EXPORT Thread : public webrtc::TaskQueueBase {
 public:
  static const int kForever = -1;

  explicit Thread(SocketServer* ss);
  explicit Thread(std::unique_ptr<SocketServer> ss);

  Thread(SocketServer* ss, bool do_init);
  Thread(std::unique_ptr<SocketServer> ss, bool do_init);

  ~Thread() override;

  static std::unique_ptr<Thread> CreateWithSocketServer();
  static std::unique_ptr<Thread> Create();
  static Thread* Current();

  
  class ScopedDisallowBlockingCalls {
   public:
    ScopedDisallowBlockingCalls();
    ScopedDisallowBlockingCalls(const ScopedDisallowBlockingCalls&) = delete;
    ScopedDisallowBlockingCalls& operator=(const ScopedDisallowBlockingCalls&) =
        delete;
    ~ScopedDisallowBlockingCalls();

   private:
    Thread* const thread_;
    const bool previous_state_;
  };

  SocketServer* socketserver();


  virtual void Quit();
  virtual bool IsQuitting();
  virtual void Restart();
  ...

  // Get() will process I/O until:
  //  1) A message is available (returns true)
  //  2) cmsWait seconds have elapsed (returns false)
  //  3) Stop() is called (returns false)
  virtual bool Get(Message* pmsg,
                   int cmsWait = kForever,
                   bool process_io = true);
  virtual bool Peek(Message* pmsg, int cmsWait = 0);
  // |time_sensitive| is deprecated and should always be false.
  virtual void Post(const Location& posted_from,
                    MessageHandler* phandler,
                    uint32_t id = 0,
                    MessageData* pdata = nullptr,
                    bool time_sensitive = false);
  virtual void PostDelayed(const Location& posted_from,
                           int delay_ms,
                           MessageHandler* phandler,
                           uint32_t id = 0,
                           MessageData* pdata = nullptr);
  virtual void PostAt(const Location& posted_from,
                      int64_t run_at_ms,
                      MessageHandler* phandler,
                      uint32_t id = 0,
                      MessageData* pdata = nullptr);
  virtual void Clear(MessageHandler* phandler,
                     uint32_t id = MQID_ANY,
                     MessageList* removed = nullptr);
  virtual void Dispatch(Message* pmsg);

  // Amount of time until the next message can be retrieved
  virtual int GetDelay();

  bool empty() const { return size() == 0u; }
  size_t size() const {
    CritScope cs(&crit_);
    return messages_.size() + delayed_messages_.size() + (fPeekKeep_ ? 1u : 0u);
  }

  // Internally posts a message which causes the doomed object to be deleted
  template <class T>
  void Dispose(T* doomed) {
    if (doomed) {
      Post(RTC_FROM_HERE, nullptr, MQID_DISPOSE, new DisposeData<T>(doomed));
    }
  }

  // When this signal is sent out, any references to this queue should
  // no longer be used.
  sigslot::signal0<> SignalQueueDestroyed;

  bool IsCurrent() const;

  // Sleeps the calling thread for the specified number of milliseconds, during
  // which time no processing is performed. Returns false if sleeping was
  // interrupted by a signal (POSIX only).
  static bool SleepMs(int millis);

  // Sets the thread's name, for debugging. Must be called before Start().
  // If |obj| is non-null, its value is appended to |name|.
  const std::string& name() const { return name_; }
  bool SetName(const std::string& name, const void* obj);

  // Starts the execution of the thread.
  bool Start();

  // Tells the thread to stop and waits until it is joined.
  // Never call Stop on the current thread.  Instead use the inherited Quit
  // function which will exit the base Thread without terminating the
  // underlying OS thread.
  virtual void Stop();

  // By default, Thread::Run() calls ProcessMessages(kForever).  To do other
  // work, override Run().  To receive and dispatch messages, call
  // ProcessMessages occasionally.
  virtual void Run();

  virtual void Send(const Location& posted_from,
                    MessageHandler* phandler,
                    uint32_t id = 0,
                    MessageData* pdata = nullptr);

  template <
      class ReturnT,
      typename = typename std::enable_if<!std::is_void<ReturnT>::value>::type>
  ReturnT Invoke(const Location& posted_from, FunctionView<ReturnT()> functor) {
    ReturnT result;
    InvokeInternal(posted_from, [functor, &result] { result = functor(); });
    return result;
  }

  template <
      class ReturnT,
      typename = typename std::enable_if<std::is_void<ReturnT>::value>::type>
  void Invoke(const Location& posted_from, FunctionView<void()> functor) {
    InvokeInternal(posted_from, functor);
  }

  // Allows invoke to specified |thread|. Thread never will be dereferenced and
  // will be used only for reference-based comparison, so instance can be safely
  // deleted. If NDEBUG is defined and DCHECK_ALWAYS_ON is undefined do nothing.
  void AllowInvokesToThread(Thread* thread);

  // If NDEBUG is defined and DCHECK_ALWAYS_ON is undefined do nothing.
  void DisallowAllInvokes();
  // Returns true if |target| was allowed by AllowInvokesToThread() or if no
  // calls were made to AllowInvokesToThread and DisallowAllInvokes. Otherwise
  // returns false.
  // If NDEBUG is defined and DCHECK_ALWAYS_ON is undefined always returns true.
  bool IsInvokeToThreadAllowed(rtc::Thread* target);

  template <class FunctorT>
  void PostTask(const Location& posted_from, FunctorT&& functor) {
    Post(posted_from, GetPostTaskMessageHandler(), /*id=*/0,
         new rtc_thread_internal::MessageWithFunctor<FunctorT>(
             std::forward<FunctorT>(functor)));
  }
  template <class FunctorT>
  void PostDelayedTask(const Location& posted_from,
                       FunctorT&& functor,
                       uint32_t milliseconds) {
    PostDelayed(posted_from, milliseconds, GetPostTaskMessageHandler(),
                /*id=*/0,
                new rtc_thread_internal::MessageWithFunctor<FunctorT>(
                    std::forward<FunctorT>(functor)));
  }

  // From TaskQueueBase
  void PostTask(std::unique_ptr<webrtc::QueuedTask> task) override;
  void PostDelayedTask(std::unique_ptr<webrtc::QueuedTask> task,
                       uint32_t milliseconds) override;
  void Delete() override;

  // ProcessMessages will process I/O and dispatch messages until:
  //  1) cms milliseconds have elapsed (returns true)
  //  2) Stop() is called (returns false)
  bool ProcessMessages(int cms);

  // Returns true if this is a thread that we created using the standard
  // constructor, false if it was created by a call to
  // ThreadManager::WrapCurrentThread().  The main thread of an application
  // is generally not owned, since the OS representation of the thread
  // obviously exists before we can get to it.
  // You cannot call Start on non-owned threads.
  bool IsOwned();

  // These functions are public to avoid injecting test hooks. Don't call them
  // outside of tests.
  // This method should be called when thread is created using non standard
  // method, like derived implementation of rtc::Thread and it can not be
  // started by calling Start(). This will set started flag to true and
  // owned to false. This must be called from the current thread.
  bool WrapCurrent();
  void UnwrapCurrent();

  // Sets the per-thread allow-blocking-calls flag to false; this is
  // irrevocable. Must be called on this thread.
  void DisallowBlockingCalls() { SetAllowBlockingCalls(false); }

 protected:
  class CurrentThreadSetter : CurrentTaskQueueSetter {
   public:
    explicit CurrentThreadSetter(Thread* thread)
        : CurrentTaskQueueSetter(thread),
          manager_(rtc::ThreadManager::Instance()),
          previous_(manager_->CurrentThread()) {
      manager_->ChangeCurrentThreadForTest(thread);
    }
    ~CurrentThreadSetter() { manager_->ChangeCurrentThreadForTest(previous_); }

   private:
    rtc::ThreadManager* const manager_;
    rtc::Thread* const previous_;
  };

  // DelayedMessage goes into a priority queue, sorted by trigger time. Messages
  // with the same trigger time are processed in num_ (FIFO) order.
  class DelayedMessage {
   public:
    DelayedMessage(int64_t delay,
                   int64_t run_time_ms,
                   uint32_t num,
                   const Message& msg)
        : delay_ms_(delay),
          run_time_ms_(run_time_ms),
          message_number_(num),
          msg_(msg) {}

    bool operator<(const DelayedMessage& dmsg) const {
      return (dmsg.run_time_ms_ < run_time_ms_) ||
             ((dmsg.run_time_ms_ == run_time_ms_) &&
              (dmsg.message_number_ < message_number_));
    }

    int64_t delay_ms_;  // for debugging
    int64_t run_time_ms_;
    // Monotonicaly incrementing number used for ordering of messages
    // targeted to execute at the same time.
    uint32_t message_number_;
    Message msg_;
  };

  class PriorityQueue : public std::priority_queue<DelayedMessage> {
   public:
    container_type& container() { return c; }
    void reheap() { make_heap(c.begin(), c.end(), comp); }
  };

  void DoDelayPost(const Location& posted_from,
                   int64_t cmsDelay,
                   int64_t tstamp,
                   MessageHandler* phandler,
                   uint32_t id,
                   MessageData* pdata);

  // Perform initialization, subclasses must call this from their constructor
  // if false was passed as init_queue to the Thread constructor.
  void DoInit();

  // Does not take any lock. Must be called either while holding crit_, or by
  // the destructor (by definition, the latter has exclusive access).
  void ClearInternal(MessageHandler* phandler,
                     uint32_t id,
                     MessageList* removed) RTC_EXCLUSIVE_LOCKS_REQUIRED(&crit_);

  // Perform cleanup; subclasses must call this from the destructor,
  // and are not expected to actually hold the lock.
  void DoDestroy() RTC_EXCLUSIVE_LOCKS_REQUIRED(&crit_);

  void WakeUpSocketServer();

  // Same as WrapCurrent except that it never fails as it does not try to
  // acquire the synchronization access of the thread. The caller should never
  // call Stop() or Join() on this thread.
  void SafeWrapCurrent();

  // Blocks the calling thread until this thread has terminated.
  void Join();

  static void AssertBlockingIsAllowedOnCurrentThread();

  friend class ScopedDisallowBlockingCalls;

  RecursiveCriticalSection* CritForTest() { return &crit_; }

 private:
  class QueuedTaskHandler final : public MessageHandler {
   public:
    QueuedTaskHandler() {}
    void OnMessage(Message* msg) override;
  };

  // Sets the per-thread allow-blocking-calls flag and returns the previous
  // value. Must be called on this thread.
  bool SetAllowBlockingCalls(bool allow);

#if defined(WEBRTC_WIN)
  static DWORD WINAPI PreRun(LPVOID context);
#else
  static void* PreRun(void* pv);
#endif

  // ThreadManager calls this instead WrapCurrent() because
  // ThreadManager::Instance() cannot be used while ThreadManager is
  // being created.
  // The method tries to get synchronization rights of the thread on Windows if
  // |need_synchronize_access| is true.
  bool WrapCurrentWithThreadManager(ThreadManager* thread_manager,
                                    bool need_synchronize_access);

  // Return true if the thread is currently running.
  bool IsRunning();

  void InvokeInternal(const Location& posted_from,
                      rtc::FunctionView<void()> functor);

  // Called by the ThreadManager when being set as the current thread.
  void EnsureIsCurrentTaskQueue();

  // Called by the ThreadManager when being unset as the current thread.
  void ClearCurrentTaskQueue();

  // Returns a static-lifetime MessageHandler which runs message with
  // MessageLikeTask payload data.
  static MessageHandler* GetPostTaskMessageHandler();

  bool fPeekKeep_;
  Message msgPeek_;
  MessageList messages_ RTC_GUARDED_BY(crit_);
  PriorityQueue delayed_messages_ RTC_GUARDED_BY(crit_);
  uint32_t delayed_next_num_ RTC_GUARDED_BY(crit_);
#if (!defined(NDEBUG) || defined(DCHECK_ALWAYS_ON))
  std::vector<Thread*> allowed_threads_ RTC_GUARDED_BY(this);
  bool invoke_policy_enabled_ RTC_GUARDED_BY(this) = false;
#endif
  RecursiveCriticalSection crit_;
  bool fInitialized_;
  bool fDestroyed_;

  volatile int stop_;

  // The SocketServer might not be owned by Thread.
  SocketServer* const ss_;
  // Used if SocketServer ownership lies with |this|.
  std::unique_ptr<SocketServer> own_ss_;

  std::string name_;

  // TODO(tommi): Add thread checks for proper use of control methods.
  // Ideally we should be able to just use PlatformThread.

#if defined(WEBRTC_POSIX)
  pthread_t thread_ = 0;
#endif

#if defined(WEBRTC_WIN)
  HANDLE thread_ = nullptr;
  DWORD thread_id_ = 0;
#endif

  // Indicates whether or not ownership of the worker thread lies with
  // this instance or not. (i.e. owned_ == !wrapped).
  // Must only be modified when the worker thread is not running.
  bool owned_ = true;

  // Only touched from the worker thread itself.
  bool blocking_calls_allowed_ = true;

  // Runs webrtc::QueuedTask posted to the Thread.
  QueuedTaskHandler queued_task_handler_;
  std::unique_ptr<TaskQueueBase::CurrentTaskQueueSetter>
      task_queue_registration_;

  friend class ThreadManager;

  RTC_DISALLOW_COPY_AND_ASSIGN(Thread);
};
```


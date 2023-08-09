---
layout: post
title: webrtc module
date: 2023-07-29 23:59:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

## 1. 前言

在webrtc中存在一些需要定时且重复的任务，比如

```
NackModule 视频nack处理模块 modules/video_coding/deprecated/nack_module.h
BitrateController 码率控制模块
VideoSender 视频发送模块
VideoReceiver 视频接收模块 modules/video_coding/video_coding_impl.h
VideoCodingModule 视频编解码模块 modules/video_coding/include/video_coding.h

PacedSender 平滑发送模块 modules/pacing/paced_sender.h

RtpRtcp rtprtcp模块 modules/rtp_rtcp/include/rtp_rtcp.h
ModuleRtpRtcpImpl2 rtprtcp模块 modules/rtp_rtcp/source/rtp_rtcp_impl2.h

ReceiveSideCongestionController GCC模块 modules/congestion_controller/include/receive_side_congestion_controller.h
RemoteBitrateEstimator  modules/remote_bitrate_estimator/include/remote_bitrate_estimator.h

CallStats src/video/call_stats.h

RtpStreamsSynchronizer src/video/rtp_streams_synchronizer.h
```

所以就抽象出了`Module`这个任务类和`ProcessThread`这个定时执行任务的线程封装类。

WebRTC采用模块机制，把数据流水线上功能相对独立的处理点定义为模块，每个模块专注于自己的任务，模块之间基于数据流进行通信。与此同时，专有线程收集和处理模块内部的运行状态信息，并把这些信息反馈到目标模块，实现模块运行状态监控和服务质量保证。



## 2. 类的关系

![module]({{ site.url }}{{ site.baseurl }}/images/module-processThread.png)

- ProcessThread， 是抽象类，实现类是ProcessModuleImpl；
- Event，类似锁，实现线程阻塞和唤醒；
- PlatformThread，是包含了不同平台的线程API的类，快平台的线程类，不过具体的实现是由各个平台对应实现；
- Module是个抽象类，三个方法，TimeUntilNextProcess（距离下个Task执行的时间ms）， Process（处理Task），ProcessThreadAttached（处理模块的线程，就是ProcessThread， 通过参数ProcessThread 传递给实现Module中去）；
- ProcessModuleImpl 包含了PlatformThread（工作线程），std::list<Module>（注册的所有模块）
- ProcessModuleImpl::Run 就是传递给了PlatformThread， 就是ThreadFunction run_funciton_; 最终传递个pthread；
  就是PlatformThread执行的函数；


## 3. Module

modules/include/module.h

```cpp
class ProcessThread;

class Module {
 public:
  virtual int64_t TimeUntilNextProcess() = 0;
  virtual void Process() = 0;
  virtual void ProcessThreadAttached(ProcessThread* process_thread) {}
}

```



### 3.1 TimeUntilNextProcess

返回下一次执行Process函数的时间，**单位是毫秒**。
此方法和Process在同一个工作线程上被调用。ProcessThread执行。
该方法是在`GetNextCallbackTime`，计算下次Process执行时间。

### 3.2 Process

任务执行函数。在ProcessThread执行。
该方法是在 `ProcessThreadImpl::Process` 调用；
两种情况下会调用改方法：

- 到执行时间了，时间是通过TimeUntilNextProcess，计算所得
- ProcessThread::WakeUp，设置`m.next_callback = kCallProcessImmediately;`,立即执行；



### 3.3 !!! ProcessThreadAttached

用来把模块挂载到模块处理线程，或者从模块处理线程分离出来。
该方法是在`ProcessThreadImpl::Start`, 或者`ProcessThreadImpl::RegisterModule` 调用的。
是为了把处理模块的线程，就是ProcessThread， 通过参数ProcessThread 传递给实现Module中去。

```js
// This method is called when the module is attached to a *running* process
  // thread or detached from one.  In the case of detaching, |process_thread|
  // will be nullptr.
  //
  // This method will be called in the following cases:
  //
  // * Non-null process_thread:
  //   * ProcessThread::RegisterModule() is called while the thread is running.
  //   * ProcessThread::Start() is called and RegisterModule has previously
  //     been called.  The thread will be started immediately after notifying
  //     all modules.
  //
  // * Null process_thread:
  //   * ProcessThread::DeRegisterModule() is called while the thread is
  //     running.
  //   * ProcessThread::Stop() was called and the thread has been stopped.
  //
  // NOTE: This method is not called from the worker thread itself, but from
  //       the thread that registers/deregisters the module or calls Start/Stop.
```



## 4. ProcessThread && ProcessThreadImpl

modules/utility/include/process_thread.h
modules/utility/sourcr/process_thread_impl.h

WebRTC模块处理线程ProcessThread是模块处理机制的驱动器，它的核心作用是对所有挂载在本线程下的模块，周期性调用其Process()处理函数处理模块内部事务，并处理异步任务。

线程启动以后会循环执行`ProcessThreadImpl::Process`，它会从模块列表中找到当前需要执行的模块，并找出最近一次需要执行的最小时间，把这个时间给定时器
如果某个模块需要立马被执行可以调用WakeUp函数，它会中断定时器，马上执行指定模块

```cpp
class Module;

// TODO(tommi): ProcessThread probably doesn't need to be a virtual
// interface.  There exists one override besides ProcessThreadImpl,
// MockProcessThread, but when looking at how it is used, it seems
// a nullptr might suffice (or simply an actual ProcessThread instance).
class ProcessThread : public TaskQueueBase {
 public:
  ~ProcessThread() override;

  static std::unique_ptr<ProcessThread> Create(const char* thread_name);

  // Starts the worker thread.  Must be called from the construction thread.
  virtual void Start() = 0;

  // Stops the worker thread.  Must be called from the construction thread.
  virtual void Stop() = 0;

  // Wakes the thread up to give a module a chance to do processing right
  // away.  This causes the worker thread to wake up and requery the specified
  // module for when it should be called back. (Typically the module should
  // return 0 from TimeUntilNextProcess on the worker thread at that point).
  // Can be called on any thread.
  virtual void WakeUp(Module* module) = 0;

  // Adds a module that will start to receive callbacks on the worker thread.
  // Can be called from any thread.
  virtual void RegisterModule(Module* module, const rtc::Location& from) = 0;

  // Removes a previously registered module.
  // Can be called from any thread.
  virtual void DeRegisterModule(Module* module) = 0;
};
```



### 4.1 ProcessThread::Create

```cpp
// static
std::unique_ptr<ProcessThread> ProcessThread::Create(const char* thread_name) {
  return std::unique_ptr<ProcessThread>(new ProcessThreadImpl(thread_name));
}
```



### 4.2 Start()

调用Start创建一个新的线程，并启动线程执行定时任务；必须从构造线程中调用。

```cpp
void ProcessThreadImpl::Start() {
  if (thread_.get())
    return;
  
  //1. start 之前，就调用RegisterModule
	// typedef std::list<ModuleCallback> ModuleList;
  // ModuleList modules_;
  for (ModuleCallback& m : modules_)
    m.module->ProcessThreadAttached(this); // this 是 ProcessThreadImpl

  // 2. 新建rtc::PlatformThread
  thread_.reset(
      new rtc::PlatformThread(&ProcessThreadImpl::Run, this, thread_name_));
  // 3. 启动线程
  thread_->Start();
}
```



### 4.3 Stop()

调用Stop停止线程，并销毁线程。

```cpp
void ProcessThreadImpl::Stop() {
  if (!thread_.get())
    return;

  {
    rtc::CritScope lock(&lock_);
    stop_ = true;
  }

  wake_up_.Set();

  // 停止线程
  thread_->Stop();
  stop_ = false;

  thread_.reset();
  for (ModuleCallback& m : modules_)
    m.module->ProcessThreadAttached(nullptr);
}
```



### 4.4 WakeUp()

用来唤醒挂载在本线程下的某个模块，使得该模块有机会马上执行其Process()处理函数；

```cpp
void ProcessThreadImpl::WakeUp(Module* module) {
  // Allowed to be called on any thread.
  {
    rtc::CritScope lock(&lock_);
    for (ModuleCallback& m : modules_) {
      // 下次执行时间next_callback = -1 = kCallProcessImmediately
      if (m.module == module)
        m.next_callback = kCallProcessImmediately;
    }
  }
  wake_up_.Set();
}
```



### 4.5 PostTask()

函数用来邮递一个任务给本线程，任务对象的所有权已转移到ProcessThread，如果该任务没有运行的机会（例如，在关闭或线程永不运行时），则该对象将在工作线程被删除；向模块处理线程投递任务



### 4.6 RegisterModule()

通过RegisterModule接口注册需要定时执行的模块，ProcessThread把模块加入到模块列表中(modules_)，并调用ProcessThreadAttached注册此线程到新加入模块。

```cpp
void ProcessThreadImpl::RegisterModule(Module* module,
                                       const rtc::Location& from) {
  // Now that we know the module isn't in the list, we'll call out to notify
  // the module that it's attached to the worker thread.  We don't hold
  // the lock while we make this call.
  if (thread_.get())
    module->ProcessThreadAttached(this);

  {
    rtc::CritScope lock(&lock_);
    modules_.push_back(ModuleCallback(module, from));
  }

  // Wake the thread calling ProcessThreadImpl::Process() to update the
  // waiting time. The waiting time for the just registered module may be
  // shorter than all other registered modules.
  // 唤醒
  wake_up_.Set();
}
```



### 4.7 DeRegisterModule()

通过DeRegisterModule接口移除不再需要定时执行的模块，ProcessThread把模块从模块列表中移除，并调用ProcessThreadAttached取消注册此线程到移除模块

```cpp
void ProcessThreadImpl::DeRegisterModule(Module* module) {

  {
    rtc::CritScope lock(&lock_);
    modules_.remove_if(
        [&module](const ModuleCallback& m) { return m.module == module; });
  }

  // Notify the module that it's been detached.
  module->ProcessThreadAttached(nullptr);
}
```



## 5. ProcessThreadImpl

modules/utility/sourcr/process_thread_impl.h

```cpp
namespace webrtc {
 
class ProcessThreadImpl : public ProcessThread {
 public:
  explicit ProcessThreadImpl(const char* thread_name);
  ~ProcessThreadImpl() override;
 
  void Start() override;
  void Stop() override;
 
  void WakeUp(Module* module) override;
  void PostTask(std::unique_ptr<QueuedTask> task) override;
 
  void RegisterModule(Module* module, const rtc::Location& from) override;
  void DeRegisterModule(Module* module) override;
 
 protected:
  static void Run(void* obj);
  bool Process();
 
 private:
 
  typedef std::list<ModuleCallback> ModuleList;
 
  // Warning: For some reason, if |lock_| comes immediately before |modules_|
  // with the current class layout, we will  start to have mysterious crashes
  // on Mac 10.9 debug.  I (Tommi) suspect we're hitting some obscure alignemnt
  // issues, but I haven't figured out what they are, if there are alignment
  // requirements for mutexes on Mac or if there's something else to it.
  // So be careful with changing the layout.
  rtc::CriticalSection lock_;  // Used to guard modules_, tasks_ and stop_.
 
  rtc::ThreadChecker thread_checker_;
  rtc::Event wake_up_;
  // TODO(pbos): Remove unique_ptr and stop recreating the thread.
  std::unique_ptr<rtc::PlatformThread> thread_;
 
  ModuleList modules_;
  std::queue<QueuedTask*> queue_;
  bool stop_;
  const char* thread_name_;
};
 
}  // namespace webrtc
```



### 5.1 属性

- ModuleList modules_， 注册的模块存储的list；（typedef std::list<ModuleCallback> ModuleList;）
- rtc::Event wake_up_，用于阻塞、唤醒模块处理线程
- std::unique_ptr\<rtc::PlatformThread> thread_， PlatformThread线程对象;
- rtc::ThreadChecker thread_checker_, 记录ProcessThreadImpl对象创建时的线程;
- bool stop_; 结束模块处理线程
- std::queue<QueuedTask*> queue_, 任务队列，保存投递的任务

### 5.2 ProcessThreadImpl::ModuleCallback

```cpp
struct ModuleCallback
{
    ModuleCallback() = delete;
    ModuleCallback(ModuleCallback&& cb) = default;
    ModuleCallback(const ModuleCallback& cb) = default;

    /*构造器*/
    ModuleCallback(Module* module, const rtc::Location& location)
        : module(module), location(location) {}

    bool operator==(const ModuleCallback& cb) const
    {
        return cb.module == module;
    }

    /*保存的模块*/
    Module* const module;
    
    /*模块下次执行时间，绝对时间。*/
    int64_t next_callback = 0;     
    
    /*模块注册时的位置*/
    const rtc::Location location;   
    
private:
    ModuleCallback& operator=(ModuleCallback&);
};


```



### 5.3 ProcessThreadImpl::Run

1. `ProcessThreadImpl::Start` 启动的时候，执行的Run函数。

```cpp
// static
void ProcessThreadImpl::Run(void* obj) {
  ProcessThreadImpl* impl = static_cast<ProcessThreadImpl*>(obj);
  CurrentTaskQueueSetter set_current(impl);
  // 死循环
  while (impl->Process()) {
  }
}
```



### 5.4 !!! ProcessThreadImpl::Process

Process()函数首先处理挂载在本线程下的模块，这也是模块处理线程的核心任务：针对每个模块，计算其下次调用模块Process()处理函数的时刻(调用该模块的TimeUntilNextProcess()函数)。如果时刻是当前时刻，则调用模块的Process()处理函数，并更新下次调用时刻。需要注意，不同模块的执行频率不一样，线程在本轮调用末尾的等待时间和本线程下所有模块的最近下次调用时刻相关。

接下来线程Process()函数处理ProcessTask队列中的异步任务，针对每个任务调用Run()函数，然后任务出队列并销毁。等模块调用和任务都处理完后，则把本次时间片的剩余时间等待完毕，然后返回。如果在等待期间其他线程向本线程Wakeup模块或者邮递一个任务，则线程被立即唤醒并返回，进行下一轮时间片的执行。

```cpp
bool ProcessThreadImpl::Process() {
  TRACE_EVENT1("webrtc", "ProcessThreadImpl", "name", thread_name_);
  int64_t now = rtc::TimeMillis();
  // 下次调用的时间，由于每个模块的时间不同，计算得到所有模块的最小值
  // 初始值是 60s
  int64_t next_checkpoint = now + (1000 * 60);   /*60秒后*/
 
  {
    rtc::CritScope lock(&lock_);
    // stop的时候，设置stop_ = true
    /*主线程标识模块处理线程退出时，结束执行，退出线程。*/
    if (stop_)
      return false;
    
    // 1. 处理挂载在本线程下的模块
	  // 遍历modules_，处理每个module,检查是否到了module的调用时间。
	  // 到了调用时间的模块，调用模块内的处理函数Process()。
    for (ModuleCallback& m : modules_) {
      // TODO(tommi): Would be good to measure the time TimeUntilNextProcess
      // takes and dcheck if it takes too long (e.g. >=10ms).  Ideally this
      // operation should not require taking a lock, so querying all modules
      // should run in a matter of nanoseconds.
      
      // 属性 int64_t next_callback = 0;  // Absolute timestamp.
      // 模块首次被调用, 初始化的时候，m.next_callback = 0
      if (m.next_callback == 0)
        m.next_callback = GetNextCallbackTime(m.module, now);
 
      // kCallProcessImmediately = -1
      // 1. m.next_callback <= now；模块的调用时间到了；
      // 2. m.next_callback = kCallProcessImmediately = -1；立即执行； 在ProcessThreadImpl::WakeUp 
      // 上面两种条件下立即 执行Module::Process
      if (m.next_callback <= now ||
          m.next_callback == kCallProcessImmediately) {
            {
              TRACE_EVENT2("webrtc", "ModuleProcess", "function",
                           m.location.function_name(), "file",
                           m.location.file_and_line());
              // 执行Module::Process
              m.module->Process();
            }
        
            // Use a new 'now' reference to calculate when the next callback
            // should occur.  We'll continue to use 'now' above for the baseline
            // of calculating how long we should wait, to reduce variance.
            int64_t new_now = rtc::TimeMillis();
        		// 模块处理结束后，更新模块下次调用的时间
            m.next_callback = GetNextCallbackTime(m.module, new_now);
      }
      
 			/*找到所模块中最小的下次调用时间*/
      if (m.next_callback < next_checkpoint)
        next_checkpoint = m.next_callback;
    }
 
    /*处理完模块，处理线程的异步任务。*/
    while (!queue_.empty()) {
      /*任务的处理方式按照FIFO*/
      QueuedTask* task = queue_.front();
      queue_.pop();
      lock_.Leave();
      /*处理任务*/
      task->Run();
      delete task;
      lock_.Enter();
    }
  }
  
  /*计算模块处理线程的阻塞时间*/
  int64_t time_to_wait = next_checkpoint - rtc::TimeMillis();
  /*若等待时间大于零，则将模块处理线程阻塞指定时间。*/
  if (time_to_wait > 0)
    wake_up_.Wait(static_cast<int>(time_to_wait));
 
  return true;
}
```

1. 先处理Module，就是存放在modules_ 的Module。遍历modules_，处理每个module,检查是否到了module的调用时间。
   到了调用时间的模块，调用模块内的处理函数Process()。

2. m.next_callback，绝对时间， 下次执行的时间。
   m.next_callback = 0，模块首次调用， 则从GetNextCallbackTime获取时间， 内部调用Module::TimeUntilNextProcess

3. 满足以下两种条件下立即 执行Module::Process， 更新m.next_callback的时间，通过GetNextCallbackTime；

   - m.next_callback <= now；就是到执行时间了
   - m.next_callback == kCallProcessImmediately  == -1； 在ProcessThreadImpl::WakeUp 会m.next_callback =-1赋值；
   
4. next_checkpoint 默认是60*1000 = 60s；next_checkpoint 是存放所有 Module的m.next_callback的最小值；即下次最近执行时间
   
4. 处理完模块，处理线程的异步任务
   
4. 计算休眠时间，time_to_wait，休眠 `wake_up_.Wait(static_cast<int>(time_to_wait));`
   
   
   
   
   
   
   
   
   



### 5.5 GetNextCallbackTime

```cpp
// We use this constant internally to signal that a module has requested
// a callback right away.  When this is set, no call to TimeUntilNextProcess
// should be made, but Process() should be called directly.
// 立即执行 Module::Process
const int64_t kCallProcessImmediately = -1;

int64_t GetNextCallbackTime(Module* module, int64_t time_now) {
  int64_t interval = module->TimeUntilNextProcess();
  if (interval < 0) {
    // Falling behind, we should call the callback now.
    return time_now;
  }
  return time_now + interval;
}
```



## 6. 调用堆栈

### 6.1 启动线程

```
ProcessThreadImpl::Start
Module::ProcessThreadAttached // ProcessThread* process_thread
PlatformThread::PlatformThread 
PlatformThread::Start
ProcessThreadImpl::Run // while 死循环
ProcessThreadImpl::Process
GetNextCallbackTime
Module::Process
rtc::Event::Wait
```

### 6.2 注册模块

```
ProcessThreadImpl::RegisterModule
Module::ProcessThreadAttached // ProcessThread* process_thread
rtc::Event::Set // 触发ProcessThreadImpl::Process
```



## 7. 示例

- 向模块处理线程注册了两个模块，其中一个模块一秒执行一次，另外一个模块两秒执行一次。
- 在模块的执行期间向模块处理线程投递了两个异步的任务。

```cpp
#include <iostream>
#include "api/task_queue/queued_task.h"
#include "modules/utility/source/process_thread_impl.h"
#include "modules/include/module.h"
#include "rtc_base/location.h"

using namespace std;
using namespace webrtc;

/*模块一*/
class my_module_one :public Module
{
public:
    /*每秒执行一次*/
    int64_t TimeUntilNextProcess() { return 1000; }
    
    void Process()
    {
        static int count = 0;
        cout << "process module_one  " << count++ << endl;
    }
};

/*模块二*/
class my_module_two : public Module 
{
public:
    /*每两秒执行一次*/
    int64_t TimeUntilNextProcess() { return 2000; }

    void Process() 
    {
        static int count = 0;
        cout << "process module_two  " << count++ << endl;
    }
};

/*任务*/
class asyn_task : public QueuedTask 
{
 public:
  asyn_task(string task_name)
      : task_name_(task_name) {}

  bool Run() override 
  {
    cout << "post a " << task_name_ <<endl;

    return true;
  }

 private:
  string task_name_;
};

int main()
{
    /*创建模块处理对象*/
    unique_ptr<ProcessThread> pt = ProcessThread::Create("process thread");

    /*定义任务*/
    unique_ptr<QueuedTask> task1(new asyn_task("task_one"));
    unique_ptr<QueuedTask> task2(new asyn_task("task_two"));
    
    /*定义模块*/
    my_module_one mmo;
    my_module_two mmt;

    /*开始处理任务*/
    pt->Start();
    
    /*模块注册*/
    pt->RegisterModule(&mmo, RTC_FROM_HERE);
    pt->RegisterModule(&mmt, RTC_FROM_HERE);

    /*3秒后投递一个即时任务*/
    Sleep(3000);
    pt->PostTask(std::move(task1));

    /*3秒后再投递一个即时任务*/
    Sleep(3000);
    pt->PostTask(std::move(task2));

    /*1秒后注销模块*/
    Sleep(1000);
    pt->DeRegisterModule(&mmo);
    pt->DeRegisterModule(&mmt);

    /*关闭模块处理线程，回收资源。*/
    pt->Stop();

    return 0;
}
```



```
process module_one  0
process module_one  1
process module_two  0 
process module_one  2
post a task_one
process module_two  1
process module_one  3
process module_one  4
process module_one  5
process module_two  2
post a task_two
process module_one  6
```



## 参考

[WebRTC之Module](https://blog.csdn.net/momo0853/article/details/87277472)

[WebRTC学习进阶之路 --- 十七、源码分析之WebRTC的数据流水线详解&模块机制核心ProcessThread与ProcessThreadImpl](https://blog.csdn.net/xiaomucgwlmx/article/details/103417862)

[WebRTC源码分析之模块的执行-Module](https://blog.csdn.net/qiuguolu1108/article/details/114796578)
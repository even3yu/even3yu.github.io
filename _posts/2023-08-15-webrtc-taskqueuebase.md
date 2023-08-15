---
layout: post
title: webrtc taskqueuebase and taskqueue
date: 2023-08-15 23:29:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

## 	1. 前言

webrtc 中的并发任务, 通常每个线程维护一个任务队列，不同的线程有不同的分工。其中，taskqueue 结构便是线程模型的基础。

> - webrtc的模型
>   <img src="{{ site.url }}{{ site.baseurl }}/images/webrtc-taskqueuebase.assets/taskqueuebase-threadmodle-rtc.png" alt="image-20230815220023182" style="zoom:25%;" />
> - 传统型的模型
>   <img src="{{ site.url }}{{ site.baseurl }}/images/webrtc-taskqueuebase.assets/taskqueuebase-threadmodle.png" alt="image-20230815215409251" style="zoom:25%;" />

在WebRTC中，会单独创建线程用于处理即时任务和延迟任务。其他线程可以将待做的事包装成任务投递到该线程去，由该线程执行。这个线程只做一件事，就是循环的处理任务。 

任务队列 `TaskQueue`的用途是将任务放到到另外一个线程执行，与`std::aysnc`功能相似，但是它不能取得执行结果。任务队列是多线程编程下的基础设施，是一种线程间交互手段。

一个`TaskQueue`对象就代表了一个任务执行线程，**在构造时产生线程，在析构时结束线程**。

一个TaskQueue对象产生一个Task执行线程，它本质上是一个异步任务队列，将任务从一个线程分发至任务执行队列，可以起到线程隔离的作用，而减少同步的需求，比如将一系列关联方法放入任务队列可以减少互斥的使用，通过isCurrent判断方法是否运行在任务执行线程 ，如果是则无需添加同步措施。

可以将TaskQueue作为特定用途的线程，比如用作编码线程，将编码任务放入线程中。在webrtc中的视频编码就是如此。

## 2. 跨线程通信

<img src="{{ site.url }}{{ site.baseurl }}/images/webrtc-taskqueuebase.assets/taskqueuebase-thread.png" alt="taskqueuebase-thread" style="zoom:50%;" />

在创建任务队列时，会创建一个线程，这个线程只用于任务的处理，其他线程可以向其投递任务。在主线程中又创建了一个子线程，并将任务队列传入，在子线程也可以向其投递任务。
这里有三个线程；

- 主线程，
- 子线程
- 工作线程（或者任务队列处理线程）

主线程，子线程可以把Task 投递给工作线程，这就实现了线程的通信。



## 3. TaskQueueBase，以及他的子类们

![taskqueuebase]({{ site.url }}{{ site.baseurl }}/images/webrtc-taskqueuebase.assets/taskqueuebase.png)



![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/webrtc-taskqueuebase.assets/taskqueuebase2.png)

- TaskQueueBase 是基类, TaskQueueFactory 是factory的基类
- 跨平台的子类

| TaskQueue         | 平台    | 对应的Factroy                     |
| ----------------- | ------- | --------------------------------- |
| TaskQueueGcd      | iOS/mac | TaskQueueGcdFactory               |
| TaskQueueLibevent | linux   | TaskQueueLibeventFactory          |
| TaskQueueStdlib   | 跨平台  | TaskQueuestdlibFactory            |
| TaskQueueWin      | Windows | TaskQueueWinFactory               |
| rtc::Thread       | -       | -                                 |
| ProcessThread     | -       | -                                 |
| TaskQueue         |         | 是个包装类，可以包装各种TaskQueue |



## 4. TaskQueueBase

src/api/task_queue/task_queue_base.h

```cpp
// Asynchronously executes tasks in a way that guarantees that they're executed
// in FIFO order and that tasks never overlap. Tasks may always execute on the
// same worker thread and they may not. To DCHECK that tasks are executing on a
// known task queue, use IsCurrent().
class RTC_LOCKABLE RTC_EXPORT TaskQueueBase {
 public:
  // Starts destruction of the task queue.
  // On return ensures no task are running and no new tasks are able to start
  // on the task queue.
  // Responsible for deallocation. Deallocation may happen syncrhoniously during
  // Delete or asynchronously after Delete returns.
  // Code not running on the TaskQueue should not make any assumption when
  // TaskQueue is deallocated and thus should not call any methods after Delete.
  // Code running on the TaskQueue should not call Delete, but can assume
  // TaskQueue still exists and may call other methods, e.g. PostTask.
  virtual void Delete() = 0;

  // Schedules a task to execute. Tasks are executed in FIFO order.
  // If |task->Run()| returns true, task is deleted on the task queue
  // before next QueuedTask starts executing.
  // When a TaskQueue is deleted, pending tasks will not be executed but they
  // will be deleted. The deletion of tasks may happen synchronously on the
  // TaskQueue or it may happen asynchronously after TaskQueue is deleted.
  // This may vary from one implementation to the next so assumptions about
  // lifetimes of pending tasks should not be made.
  virtual void PostTask(std::unique_ptr<QueuedTask> task) = 0;

  // Schedules a task to execute a specified number of milliseconds from when
  // the call is made. The precision should be considered as "best effort"
  // and in some cases, such as on Windows when all high precision timers have
  // been used up, can be off by as much as 15 millseconds.
  virtual void PostDelayedTask(std::unique_ptr<QueuedTask> task,
                               uint32_t milliseconds) = 0;

  // Returns the task queue that is running the current thread.
  // Returns nullptr if this thread is not associated with any task queue.
  static TaskQueueBase* Current();
  bool IsCurrent() const { return Current() == this; }

 protected:
  class CurrentTaskQueueSetter {
   public:
    explicit CurrentTaskQueueSetter(TaskQueueBase* task_queue);
    CurrentTaskQueueSetter(const CurrentTaskQueueSetter&) = delete;
    CurrentTaskQueueSetter& operator=(const CurrentTaskQueueSetter&) = delete;
    ~CurrentTaskQueueSetter();

   private:
    TaskQueueBase* const previous_;
  };

  // Users of the TaskQueue should call Delete instead of directly deleting
  // this object.
  virtual ~TaskQueueBase() = default;
};
```



### 4.1 TaskQueueBase::Delete

释放任务队列。**不能直接将TaskQueue删除，需要等待任务队列处理完正在处理的任务后，才能释放任务队列**。
**在任务队列上执行的任务不可以调用Delete**。
可以参考 小节`TaskQueueStdlib::Delete——销毁TaskQueue`。

### 4.2 TaskQueueBase::PostTask

投递即时任务。此函数用于安排一个即时任务的处理，这些任务按照先进先出的顺序执行（所以称为任务队列）。
如果`task->Run()`的返回值是`true`,代表任务成功执行，任务会在下一个`QueuedTask`开始执行之前从任务队列中被移除。



### 4.3 TaskQueueBase::PostDelayedTask

投递延迟任务。安排一个延迟任务的处理，处理会在调用`PostDelayedTask`函数后的`milliseconds`毫秒后执行。
关于延迟时间的精度可以称作“尽力而为”，在某些场景下，定时可能会有一些毫秒级的误差。



### 4.4 TaskQueueBase::Current()

返回当前线程保存的任务队列

```cpp
TaskQueueBase* TaskQueueBase::Current() {
  return static_cast<TaskQueueBase*>(pthread_getspecific(GetQueuePtrTls()));
}
```

从tls 读取当前线程的TaskQueue。



### 4.5 TaskQueueBase::IsCurrent

判断当前线程保存的任务队列是否是自己

```cpp
bool IsCurrent() const { return Current() == this; }
```



### 4.6 !!! TaskQueueBase::CurrentTaskQueueSetter

```cpp
class CurrentTaskQueueSetter 
{
 public:
  explicit CurrentTaskQueueSetter(TaskQueueBase* task_queue);
  CurrentTaskQueueSetter(const CurrentTaskQueueSetter&) = delete;
  CurrentTaskQueueSetter& operator=(const CurrentTaskQueueSetter&) = delete;
  ~CurrentTaskQueueSetter();

 private:
  TaskQueueBase* const previous_; /*保存的是上一个任务队列*/
};
```

把任务队列绑定到当前线程上。这里用thread 的tls技术。



#### 4.6.1 CurrentTaskQueueSetter::CurrentTaskQueueSetter

```cpp
TaskQueueBase::CurrentTaskQueueSetter::CurrentTaskQueueSetter(
    TaskQueueBase* task_queue)
    : previous_(TaskQueueBase::Current()) {
      // 任务队列保存在当前线程的TLS中
  pthread_setspecific(GetQueuePtrTls(), task_queue);
}
```



#### 4.6.2 CurrentTaskQueueSetter::~CurrentTaskQueueSetter

```cpp
CurrentTaskQueueSetter::~CurrentTaskQueueSetter() {
  // 将之前的任务队列保存到线程内
  pthread_setspecific(GetQueuePtrTls(), previous_);
}

```



#### 4.6.3 InitializeTls

```cpp
void InitializeTls() {
  // 创建TLS
  RTC_CHECK(pthread_key_create(&g_queue_ptr_tls, nullptr) == 0);
}
```



#### 4.6.4 GetQueuePtrTls

```cpp
pthread_key_t GetQueuePtrTls() {
  // 保证InitializeTls()只运行一次
  static pthread_once_t init_once = PTHREAD_ONCE_INIT;
  RTC_CHECK(pthread_once(&init_once, &InitializeTls) == 0);
  return g_queue_ptr_tls;
}
```



#### 4.6.5 ABSL_CONST_INIT pthread_key_t g_queue_ptr_tls = 0;

创建线程局部存储。



## 5. TaskQueueStdlib

TaskQueueBase的具体实现TaskQueueStdlib。

![taskqueue-stdlib]({{ site.url }}{{ site.baseurl }}/images/webrtc-taskqueuebase.assets/taskqueue-stdlib.png)

rtc_base/task_queue_stdlib.cc

```cpp
class TaskQueueStdlib final : public TaskQueueBase {
 public:
  TaskQueueStdlib(absl::string_view queue_name, rtc::ThreadPriority priority);
  ~TaskQueueStdlib() override = default;

  void Delete() override;
  void PostTask(std::unique_ptr<QueuedTask> task) override;
  void PostDelayedTask(std::unique_ptr<QueuedTask> task,
                       uint32_t milliseconds) override;

 private:
  using OrderId = uint64_t;

  struct DelayedEntryTimeout {
    int64_t next_fire_at_ms_{};
    OrderId order_{};

    bool operator<(const DelayedEntryTimeout& o) const {
      return std::tie(next_fire_at_ms_, order_) <
             std::tie(o.next_fire_at_ms_, o.order_);
    }
  };

  struct NextTask {
    bool final_task_{false};
    std::unique_ptr<QueuedTask> run_task_;
    int64_t sleep_time_ms_{};
  };

  NextTask GetNextTask();

  static void ThreadMain(void* context);

  void ProcessTasks();

  void NotifyWake();

  // Indicates if the thread has started.
  rtc::Event started_;

  // Indicates if the thread has stopped.
  rtc::Event stopped_;

  // Signaled whenever a new task is pending.
  rtc::Event flag_notify_;

  // Contains the active worker thread assigned to processing
  // tasks (including delayed tasks).
  rtc::PlatformThread thread_;

  Mutex pending_lock_;

  // Indicates if the worker thread needs to shutdown now.
  bool thread_should_quit_ RTC_GUARDED_BY(pending_lock_){false};

  // Holds the next order to use for the next task to be
  // put into one of the pending queues.
  OrderId thread_posting_order_ RTC_GUARDED_BY(pending_lock_){};

  // The list of all pending tasks that need to be processed in the
  // FIFO queue ordering on the worker thread.
  std::queue<std::pair<OrderId, std::unique_ptr<QueuedTask>>> pending_queue_
      RTC_GUARDED_BY(pending_lock_);

  // The list of all pending tasks that need to be processed at a future
  // time based upon a delay. On the off change the delayed task should
  // happen at exactly the same time interval as another task then the
  // task is processed based on FIFO ordering. std::priority_queue was
  // considered but rejected due to its inability to extract the
  // std::unique_ptr out of the queue without the presence of a hack.
  std::map<DelayedEntryTimeout, std::unique_ptr<QueuedTask>> delayed_queue_
      RTC_GUARDED_BY(pending_lock_);
};

```



### 5.1 属性

- rtc::Event started_， 阻塞主线程直到工作线程启动完成，才结束阻塞主线程；
- rtc::Event stopped_， 阻塞主线程，知道工作线程执行完所有task，退出wile循环，才结束阻塞主线程，停止工作线程，销毁TaskQueue；
- rtc::Event flag_notify_ ，每当有新的任务等待时(Post/PostDelay)，就会发出信号，唤醒工作线程；工作线程在没task的处于休眠状态；
- rtc::PlatformThread thread_，表示被分配用于处理任务的活动工作线程， 工作线程，处理task的；
- bool thread_should_quit_ ， 表示工作线程是否需要现在关闭；
- OrderId thread_posting_order_， 保存下一个任务的序号，用于将其放入一个挂起的队列中，in64_t 类型；
- std::queue<std::pair<OrderId, std::unique_ptr<QueuedTask>>> pending_queue_， 需要在工作线程中按照先进先出的队列顺序处理的待办任务列表；
- std::map<DelayedEntryTimeout, std::unique_ptr<QueuedTask>> delayed_queue_，需要在工作线程中延迟一段时间再处理的待办任务列表，如果两个任务在相同的时间间隔内，发生，那么将根据先后顺序进行处理。



### 5.2 TaskQueueStdlib::TaskQueueStdlib

```cpp
TaskQueueStdlib::TaskQueueStdlib(absl::string_view queue_name,
                                 rtc::ThreadPriority priority)
    : started_(/*manual_reset=*/false, /*initially_signaled=*/false),
      stopped_(/*manual_reset=*/false, /*initially_signaled=*/false),
      flag_notify_(/*manual_reset=*/false, /*initially_signaled=*/false),
      thread_(&TaskQueueStdlib::ThreadMain, this, queue_name, priority) {
  thread_.Start();
  // 主线程被挂起， 不是thead_
  started_.Wait(rtc::Event::kForever);
}
```

- 创建处理task的工作线程`rtc::PlatformThread thread_`，比启动

- `rtc::Event started_;` 这里阻塞了主线程（调用线程），一直到queue 的 thread_的启动完成；

  ```cpp
  void TaskQueueStdlib::ProcessTasks() {
    started_.Set();  // 这里就结束线程阻塞了，注意started_ 阻塞的不是thread_，而是主线程
    ...
    }
  ```



#### 5.2.1 TaskQueueStdlib::ThreadMain

```cpp
// static
void TaskQueueStdlib::ThreadMain(void* context) {
  TaskQueueStdlib* me = static_cast<TaskQueueStdlib*>(context);
  CurrentTaskQueueSetter set_current(me);
  me->ProcessTasks();
}
```

- thread_的 线程函数，静态方法；
-  创建 CurrentTaskQueueSetter， 用tls 保存了当前TaskQueueStdlib对象，可以通过`TaskQueueBase::Current` 得到TaskQueueStdlib对象；
- 调用 ProcessTasks，进入while 死循环，处理task；



#### 5.2.2 TaskQueueStdlib::ProcessTasks

见后面小节。



### 5.3 !!! TaskQueueStdlib::Delete——销毁TaskQueue

```cpp
void TaskQueueStdlib::Delete() {
  // 销毁操作不能在任务处理线程中进行， 不能在thread_中销毁
  RTC_DCHECK(!IsCurrent());

  {
    // 标记任务处理线程需要退出
    rtc::CritScope lock(&pending_lock_);
    thread_should_quit_ = true;
  }
	// 唤醒线程去执行任务
  NotifyWake();

  // 阻塞主线，等到工作线程的任务处理完，退出while循环，才可以销毁TaskQueueStdlib
  stopped_.Wait(rtc::Event::kForever);
  thread_.Stop();
  delete this;
}
```

- 首先判断当前线程是不是任务处理线程（就是`thread_`），因为销毁操作不可以在任务处理线程中进行

- 随后标记任务处理线程需要退出并唤醒线程执行相关任务

- 然后当前线程等待任务处理线程退出后唤醒主线程

- 主线程回收任务处理线程，最后释放任务队列对象

  ```cpp
  void TaskQueueStdlib::ProcessTasks() {
  	...
    while (true) {
      ...
    }
  	// 这里代表 tasekqueue的处理线程（工作线程）都处理完成了，通知可以释放TaskQueueStdlib对象
    stopped_.Set();
  }
  ```

  



### 5.4 TaskQueueStdlib::PostTask——发送即时任务

```cpp
void TaskQueueStdlib::PostTask(std::unique_ptr<QueuedTask> task) {
  {
    MutexLock lock(&pending_lock_);
    // 序号自增赋值
    OrderId order = thread_posting_order_++;
		// 加入到即时任务队列中
    pending_queue_.push(std::pair<OrderId, std::unique_ptr<QueuedTask>>(
        order, std::move(task)));
  }
	// 唤醒任务处理线程处理任务
  NotifyWake();
}
```



#### 5.4.1 QueuedTask

api\task_queue\queued_task.h

```cpp
class QueuedTask 
{
 public:
  virtual ~QueuedTask() = default;
  virtual bool Run() = 0;
};
```

子类实现QueuedTask，复写Run，而工作线程执行task的时候，最终调用Run，就可以执行不同的业务代码了。



### 5.5 TaskQueueStdlib::PostDelayedTask——发送延迟任务

```cpp
void TaskQueueStdlib::PostDelayedTask(std::unique_ptr<QueuedTask> task,
                                      uint32_t milliseconds) {
  // 计算延迟任务的执行时间（绝对时间），
	// 执行的时间 = fire_at = 当前时间 + 延迟的时间
  auto fire_at = rtc::TimeMillis() + milliseconds;

  DelayedEntryTimeout delay;
  // 设置延迟任务的执行时间
  delay.next_fire_at_ms_ = fire_at;

  {
    MutexLock lock(&pending_lock_);
    // 序号自增赋值
    delay.order_ = ++thread_posting_order_;
    delayed_queue_[delay] = std::move(task);
  }

  // 唤醒任务处理线程处理任务
  NotifyWake();
}
```



#### 5.5.1 TaskQueueStdlib::NotifyWake

唤醒工作线程，处理task。任务处理线程将所有任务处理完毕后，没有事做了，会进入睡眠状态。当有新的任务到达以后，需要唤醒线程去处理任务。

```cpp
void TaskQueueStdlib::NotifyWake() {
  // The queue holds pending tasks to complete. Either tasks are to be
  // executed immediately or tasks are to be run at some future delayed time.
  // For immediate tasks the task queue's thread is busy running the task and
  // the thread will not be waiting on the flag_notify_ event. If no immediate
  // tasks are available but a delayed task is pending then the thread will be
  // waiting on flag_notify_ with a delayed time-out of the nearest timed task
  // to run. If no immediate or pending tasks are available, the thread will
  // wait on flag_notify_ until signaled that a task has been added (or the
  // thread to be told to shutdown).

  // In all cases, when a new immediate task, delayed task, or request to
  // shutdown the thread is added the flag_notify_ is signaled after. If the
  // thread was waiting then the thread will wake up immediately and re-assess
  // what task needs to be run next (i.e. run a task now, wait for the nearest
  // timed delayed task, or shutdown the thread). If the thread was not waiting
  // then the thread will remained signaled to wake up the next time any
  // attempt to wait on the flag_notify_ event occurs.

  // Any immediate or delayed pending task (or request to shutdown the thread)
  // must always be added to the queue prior to signaling flag_notify_ to wake
  // up the possibly sleeping thread. This prevents a race condition where the
  // thread is notified to wake up but the task queue's thread finds nothing to
  // do so it waits once again to be signaled where such a signal may never
  // happen.
  flag_notify_.Set();
}
```



#### 5.5.2 DelayedEntryTimeout

```cpp
struct DelayedEntryTimeout 
{
  int64_t next_fire_at_ms_{}; 
  OrderId order_{};

  /*用于比较大小*/
  bool operator<(const DelayedEntryTimeout& o) const 
  {
    /*对两个参数依次比较*/
    return std::tie(next_fire_at_ms_, order_) <
           std::tie(o.next_fire_at_ms_, o.order_);
  }
};
```

在保存延迟任务的时候，需要记录延迟任务的执行时间和任务序号。延迟任务的执行顺序，首先按照时间顺序，时间早的先执行，若执行时间相同，则按照序号，序号小的先执行，也就是先到的先执行。



### 5.6 !!! TaskQueueStdlib::ProcessTasks

```cpp
void TaskQueueStdlib::ProcessTasks() 
{
  /*任务处理线程就绪，激活主线程。*/
  started_.Set();   

  while (true) 
  {
	/*从任务表（队列、map)中获取了一个待执行的任务。*/
    auto task = GetNextTask();

		// 首先判断，是否要结束本任务队列线程， 就是Delete
    // 要销毁任务处理线程了，跳出循环。
    if (task.final_task_)
      break;    

    if (task.run_task_) 
		{
      QueuedTask* release_ptr = task.run_task_.release();  
      
      // 执行任务
      if (release_ptr->Run())   
        delete release_ptr;    

      /*获取下一个任务，继续执行。*/
      continue;
    }

    if (0 == task.sleep_time_ms_) 
      /*说明没有延迟任务也没有即时任务，则一直睡眠，直到被激活。*/
      flag_notify_.Wait(rtc::Event::kForever);
    else
      /*有待执行的延迟任务，则线程睡眠指定的时间。*/
      flag_notify_.Wait(task.sleep_time_ms_);   
  }

  // 通知主线程，本线程准备退出了， 唤醒主线程
  stopped_.Set();
}
```

- NextTask.final_task_ = true, 则 退出循环，消息处理线程准备结束
- task.run_task_ 不为空， 执行任务，执行完，继续获取下个任务
- task.run_task_为空，如果`task.sleep_time_ms_ = 0`, 则就说明没有任何任务了，工作线程进入休眠状态；否则，说明有延迟任务，休眠指定的时间；



#### 5.6.1 NextTask

```cpp
struct NextTask 
{
  bool final_task_{false};                /*是否释放任务处理队列对象*/
  std::unique_ptr<QueuedTask>  run_task_; /*实际的任务*/
  int64_t sleep_time_ms_{};               /*任务处理线程休眠时间*/
};
```

| result.run_task | result.sleep_time_ms |                                                              |
| --------------- | -------------------- | ------------------------------------------------------------ |
| nullptr         | 0                    | 没有任务，延迟任务和即时任务都没有了，队列空了；工作线程进入休眠状态； |
| nullptr         | !0                   | 没有即时任务，延迟任务还没到时间，但是延迟队列里有task，需要休眠result.sleep_time_ms时间 |
| ! nullptr       | 0                    | 有即时任务但没有延迟任务，延迟任务的队列里也没有task了       |
| ! nullptr       | !0                   | 有即时任务，延迟任务还没到时间；                             |



#### 5.6.2 !!! TaskQueueStdlib::GetNextTask

```cpp
TaskQueueStdlib::NextTask TaskQueueStdlib::GetNextTask() 
{
  /*初始化为nullptr*/
  NextTask result{};    

  /*此刻的时间，用于检测延迟任务是否可以执行。*/
  auto tick = rtc::TimeMillis();

  rtc::CritScope lock(&pending_lock_);
  
  // 任务处理线程需要退出. 在Delete的时候
  if (thread_should_quit_) 
  {
    result.final_task_ = true;  
    
    // 直接返回，退出任务处理线程，退出wile循环
    return result;               
  }

  /*延迟任务列表中有任务*/
  if (delayed_queue_.size() > 0) 
  {
	/*获取延迟最小的任务*/
    auto delayed_entry = delayed_queue_.begin();

	/*获取任务的信息*/
    const auto& delay_info = delayed_entry->first;

	/*获取任务对象 queuedTask*/
    auto& delay_run = delayed_entry->second;

  	/*若到达了延迟任务的执行时间*/
    if (tick >= delay_info.next_fire_at_ms_)   
		{
      /*！！！
     	即时任务队列中也有任务待执行
			如果即时任务中的序号比延迟任务中的序号小，先执即时任务 */
      if (pending_queue_.size() > 0)    
	  	{
        auto& entry = pending_queue_.front();
        auto& entry_order = entry.first;
        auto& entry_run = entry.second;

        /*需要选择序号小的任务执行*/
        if (entry_order < delay_info.order_)    
				{
		  		/*即时任务的序号小*/
          result.run_task_ = std::move(entry_run);
          pending_queue_.pop();    
          /*执行即时任务*/
          return result;
        }
      } // !end if (pending_queue_.size() > 0)  

	  	/*延迟任务的序号小*/
      result.run_task_ = std::move(delay_run);
        
      /*从map中删除*/
      delayed_queue_.erase(delayed_entry);  
      /*执行延迟任务*/
      return result;
    } // !end if (tick >= delay_info.next_fire_at_ms_) 

    // 这里说明 延迟任务的时间 还未到时间， 返回下个任务的休眠时间
		// 注意：执行到这里时result.run_task是nullptr， 没有延迟任务可以执行
		/*延迟任务没有到执行时间，计算下次延迟任务的执行时间。*/
    result.sleep_time_ms_ = delay_info.next_fire_at_ms_ - tick;
  }

  /*如果没有延迟任务可执行，则在即时任务中取出一个任务。*/
  if (pending_queue_.size() > 0) 
  {
    auto& entry = pending_queue_.front();
    result.run_task_ = std::move(entry.second);
    pending_queue_.pop();
  }

  /*
   * 1. result.run_task为nullptr，result.sleep_time_ms_不为0时：
   *     没有即时任务，也没有要执行的延迟任务，延迟任务还没到时间，但是延迟队列里有task
   *
   * 2. result.run_task不为nullptr，result.sleep_time_ms_为0时：
   *     有即时任务但没有延迟任务，延迟任务的队列里也没有task了；
   *
   * 3. result.run_task不为nullptr，result.sleep_time_ms_不为0时：
   *     有即时任务但没有可执行的延迟任务
   *
   * 4. result.run_task为nullptr，result.sleep_time_ms_为0时：
   *     没有任务了
   */
  return result;
}
```

- thread_should_quit_ = ture， 则 就是调用了Delete，即将要销毁TaskQueue，那么这是就不再去task任务了，` NextTask.final_task_ = true;  `, `TaskQueueStdlib::ProcessTasks`   会跳出wile循环，接着就退出了工作线程；
- 处理延迟任务，从队列中获取队首（有序的队列，时间从小大排序），最早要执行的task，与当前时间戳做对比，如果已经到达执行时间了；这时候还不能直接放回该task；**还需要和即时消息队列的队首的元素进行比对序号， 序号小的先执行；在即时任务和延迟任务都可以执行时，需要按照任务的序号执行任务，保证任务总体上按照FIFO执行；**
- 没有延迟任务可以执行，则执行即时任务；



## 6. TaskQueue

```cpp
class RTC_LOCKABLE RTC_EXPORT TaskQueue {
 public:
  // TaskQueue priority levels. On some platforms these will map to thread
  // priorities, on others such as Mac and iOS, GCD queue priorities.
  using Priority = ::webrtc::TaskQueueFactory::Priority;

  explicit TaskQueue(std::unique_ptr<webrtc::TaskQueueBase,
                                     webrtc::TaskQueueDeleter> task_queue);
  ~TaskQueue();

  // Used for DCHECKing the current queue.
  bool IsCurrent() const;

  // Returns non-owning pointer to the task queue implementation.
  webrtc::TaskQueueBase* Get() { return impl_; }

  // TODO(tommi): For better debuggability, implement RTC_FROM_HERE.

  // Ownership of the task is passed to PostTask.
  void PostTask(std::unique_ptr<webrtc::QueuedTask> task);

  // Schedules a task to execute a specified number of milliseconds from when
  // the call is made. The precision should be considered as "best effort"
  // and in some cases, such as on Windows when all high precision timers have
  // been used up, can be off by as much as 15 millseconds (although 8 would be
  // more likely). This can be mitigated by limiting the use of delayed tasks.
  void PostDelayedTask(std::unique_ptr<webrtc::QueuedTask> task,
                       uint32_t milliseconds);

  // std::enable_if is used here to make sure that calls to PostTask() with
  // std::unique_ptr<SomeClassDerivedFromQueuedTask> would not end up being
  // caught by this template.
  template <class Closure,
            typename std::enable_if<!std::is_convertible<
                Closure,
                std::unique_ptr<webrtc::QueuedTask>>::value>::type* = nullptr>
  void PostTask(Closure&& closure) {
    PostTask(webrtc::ToQueuedTask(std::forward<Closure>(closure)));
  }

  // See documentation above for performance expectations.
  template <class Closure,
            typename std::enable_if<!std::is_convertible<
                Closure,
                std::unique_ptr<webrtc::QueuedTask>>::value>::type* = nullptr>
  void PostDelayedTask(Closure&& closure, uint32_t milliseconds) {
    PostDelayedTask(webrtc::ToQueuedTask(std::forward<Closure>(closure)),
                    milliseconds);
  }

 private:
  webrtc::TaskQueueBase* const impl_;

  RTC_DISALLOW_COPY_AND_ASSIGN(TaskQueue);
};
```



### 6.1 类图

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/webrtc-taskqueuebase.assets/tasekqueue.png)



### 6.2 属性

- webrtc::TaskQueueBase* const impl_， 保存着托管的任务队列对象



### 6.3 TaskQueue::TaskQueue

```cpp
TaskQueue::TaskQueue(std::unique_ptr<webrtc::TaskQueueBase, webrtc::TaskQueueDeleter> task_queue)
    : impl_(task_queue.release()) {}
```

把 TaskQueueBase 对象传入TaskQueue 中。



#### 6.3.1 TaskQueueDeleter

api\task_queue\task_queue_base.h

```cpp
struct TaskQueueDeleter   
{
  /*仿函数，用于释放TaskQueueBase。*/
  void operator()(TaskQueueBase* task_queue) const 
  { 
      task_queue->Delete(); 
  }
};
```



### 6.4 TaskQueue::~TaskQueue() 

```cpp
TaskQueue::~TaskQueue() 
{
  /*主动调用Delete()函数释放对象*/
  impl_->Delete();
}
```



### 6.5 TaskQueue::PostTask

```cpp
void TaskQueue::PostTask(std::unique_ptr<webrtc::QueuedTask> task) {
  return impl_->PostTask(std::move(task));
}
```



```cpp
  // std::enable_if is used here to make sure that calls to PostTask() with
  // std::unique_ptr<SomeClassDerivedFromQueuedTask> would not end up being
  // caught by this template.
  template <class Closure,
            typename std::enable_if<!std::is_convertible<
                Closure,
                std::unique_ptr<webrtc::QueuedTask>>::value>::type* = nullptr>
  void PostTask(Closure&& closure) {
    PostTask(webrtc::ToQueuedTask(std::forward<Closure>(closure)));
  }
```



### 6.6 TaskQueue::PostDelayedTask

```cpp
void TaskQueue::PostDelayedTask(std::unique_ptr<webrtc::QueuedTask> task,
                                uint32_t milliseconds) {
  return impl_->PostDelayedTask(std::move(task), milliseconds);
}
```



```cpp
  template <class Closure,
            typename std::enable_if<!std::is_convertible<
                Closure,
                std::unique_ptr<webrtc::QueuedTask>>::value>::type* = nullptr>
  void PostDelayedTask(Closure&& closure, uint32_t milliseconds) {
    PostDelayedTask(webrtc::ToQueuedTask(std::forward<Closure>(closure)),
                    milliseconds);
  }
```



## 7. TaskQueueFactory

api/task_queue/task_queue_factory.h

```cpp
// The implementation of this interface must be thread-safe.
class TaskQueueFactory {
 public:
  // TaskQueue priority levels. On some platforms these will map to thread
  // priorities, on others such as Mac and iOS, GCD queue priorities.
  enum class Priority { NORMAL = 0, HIGH, LOW };

  virtual ~TaskQueueFactory() = default;
  virtual std::unique_ptr<TaskQueueBase, TaskQueueDeleter> CreateTaskQueue(
      absl::string_view name,
      Priority priority) const = 0;
};
```

TaskQueueFactory的基类，TaskQueueStdlibFactory等都是继承于他。


## 8. TaskQueueStdlibFactory

rtc_base/task_queue_stdlib.cc

```cpp
class TaskQueueStdlibFactory final : public TaskQueueFactory {
 public:
  std::unique_ptr<TaskQueueBase, TaskQueueDeleter> CreateTaskQueue(
      absl::string_view name,
      Priority priority) const override {
    return std::unique_ptr<TaskQueueBase, TaskQueueDeleter>(
        new TaskQueueStdlib(name, TaskQueuePriorityToThreadPriority(priority)));
  }
};
```



### 8.1 CreateTaskQueueStdlibFactory

```cpp
std::unique_ptr<TaskQueueFactory> CreateTaskQueueStdlibFactory() {
  return std::make_unique<TaskQueueStdlibFactory>();
}
```



## 9. CreateDefaultTaskQueueFactory

这是个跨平台的函数，创建了TaskQueueFactory。

api/task_queue/default_task_queue_factory_stdlib.cc

```cpp
std::unique_ptr<TaskQueueFactory> CreateDefaultTaskQueueFactory() {
  return CreateTaskQueueStdlibFactory();
}
```

api/task_queue/default_task_queue_factory_gcd.cc

```cpp
std::unique_ptr<TaskQueueFactory> CreateDefaultTaskQueueFactory() {
  return CreateTaskQueueGcdFactory();
}
```

api/task_queue/default_task_queue_factory_win.cc

```cpp
std::unique_ptr<TaskQueueFactory> CreateDefaultTaskQueueFactory() {
  return CreateTaskQueueWinFactory();
}
```

api/task_queue/default_task_queue_factory_libevent.cc

```cpp
std::unique_ptr<TaskQueueFactory> CreateDefaultTaskQueueFactory() {
  return CreateTaskQueueLibeventFactory();
}
```



## 10. 示例

### 10.1 VideoStreamEncoder

```cpp
//创建一个TaskQueue对象
rtc::TaskQueue encoder_queue_(task_queue_factory->CreateTaskQueue(
    "EncoderQueue",
    TaskQueueFactory::Priority::NORMAL));


//PostTask传入了一个lambada表达式，执行SendKeyFrame方法
encoder_queue_.PostTask([this] { SendKeyFrame(); });
```



### 10.2

```cpp
 #include <iostream>
 #include <string>
 #include "rtc_base/task_queue.h"
 #include "rtc_base/task_queue_stdlib.h"
 #include "rtc_base/platform_thread.h"
 #include "api/task_queue/queued_task.h"
 #include "api/task_queue/task_queue_base.h"
 #include "api/task_queue/task_queue_factory.h"
 

 using namespace webrtc;
 using namespace rtc;
 
 /*定义即时任务*/
 class instant_task :public QueuedTask {
 public:
     instant_task(string thread_name)
         :thread_name_(thread_name)
     {}
 
     bool Run()  override {
         cout << thread_name_ << " post a task..." << endl;
         return true;
     }
 
 private:
     std::string thread_name_;
 };
 
 /*定义延迟任务*/
 class delay_task : public QueuedTask {
  public:
     delay_task(string thread_name)
       : thread_name_(thread_name)
     {}
 
   bool Run() override {
     cout << thread_name_ << " post a task..." << endl;
     return true;
   }
 
  private:
   std::string thread_name_;
 };
 
// 子线程的回调
 void other_thread(void* arg) {
     TaskQueue* tq = (TaskQueue*)arg;
 
     std::unique_ptr<QueuedTask> instantTask(new instant_task("child_thread"));
     std::unique_ptr<QueuedTask> delayedTask(new delay_task("child_thread"));
 
     /*延迟任务3秒后执行*/
     tq->PostDelayedTask(std::move(delayedTask), 3 * 1000);
 
     /*这个任务是最先执行*/
     tq->PostTask(std::move(instantTask));
 
     /*睡眠5秒等待任务被执行*/
     Sleep(5000);
 }
 
 int main() {
     /*定义一个工厂*/
     std::unique_ptr<TaskQueueFactory> tqFactory = CreateTaskQueueStdlibFactory();
 
     /*用工厂创建任务队列*/
     std::unique_ptr<TaskQueueBase, TaskQueueDeleter> tqb = tqFactory->CreateTaskQueue("task queue", TaskQueueFactory::Priority::NORMAL);
 
     /*托管任务队列*/
     TaskQueue tq(std::move(tqb));
 
     /*开启一个子线程*/
     PlatformThread other_th(other_thread, (void*)&tq, "child_thread");
     other_th.Start();
 
     /*睡眠1秒后投递任务*/
     Sleep(1000);
 
     /*投递即时任务*/
     std::unique_ptr<QueuedTask> task(new instant_task("main_thread"));
     tq.PostTask(std::move(task));
 
     /*回收线程*/
     other_th.Stop();
 
     return 0;
 }
```



```
 child_thread post a task...
 main_thread post a task...
 child_thread post a task...
```

1. 创建 Factory，`CreateTaskQueueStdlibFactory`, 不同平台对应都有方法来创建Factory
2. 创建TaskQueue，通过factory 来创建
3. 在任何线程都可以通过 TaskQueueBase::PostTask 投递任务，在TaskQueue中的工作线程里处理





## 参考

[WebRTC源码分析之任务队列-TaskQueue](https://blog.csdn.net/qiuguolu1108/article/details/114796457)

[WebRTC源码分析-TaskQueue（任务队列）-TaskQueueBase](https://blog.csdn.net/weixin_52764708/article/details/127816857)
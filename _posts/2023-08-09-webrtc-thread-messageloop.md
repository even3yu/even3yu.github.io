---
layout: post
title: webrtc message loop
date: 2023-08-09 22:29:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


# 消息队列

三类消息：

- 一个是Peek消息msgPeek_ ；
- 一个即时消息队列 messages_ ；
- 一个是延迟消息队列 delayed_messages_ ；
----
消息循环中的消息队列功能以及通过持有SocketServer对象带来的IO多路复用功能。两部分功能不是完全孤立的，而是相互配合在一起使用。

----
跨线程的消息通信，两种：同步和异步；

1. PostTask/PostDelayTask，注意区别，这组，其实有两组实现，主要输入的参数不同，不过最终都会调用到Post，后文详细说明；
2. Invoke，同步调用，会阻塞调用线程，等到执行任务的工作线程放回，才会释放；

----
消息循环，其实就是在Thread 的线程中，

- 准备了一个while死循环，
- 一个消息队列，存放Post进来的消息
- 通过Get获取消息，从消息队列获取
- Post输入消息（发送消息），
- Dispatch处理消息；

这样就形成了消息的循环。



## 1. 属性

- PriorityQueue delayed_messages_ ， 延迟消息
- Message msgPeek_;
- MessageList messages_， 即时消息



## 2. 消息获取

### 2.1 Thread::Peek

简而言之就是查看之前是否已经Peek过一个MSG到成员msgPeek_ 中，若已经Peek过一个则直接将该消息返回；若没有，则通过Get()方法从消息队列中取出一个消息，成功则将该消息交给msgPeek_ 成员，并将fPeekKeep_标志置为true。

```cpp
bool Thread::Peek(Message* pmsg, int cmsWait = 0) {
  // fPeekKeep_为真，表示已经Peek过一个MSG到msgPeek_
  // 直接将该MSG返回
  if (fPeekKeep_) {
    *pmsg = msgPeek_;
    return true;
  }
  // 若没有之前没有Peek过一个MSG
  if (!Get(pmsg, cmsWait))
    return false;
  //将Get到的消息放在msgPeek_中保存，并设置标志位
  msgPeek_ = *pmsg;
  fPeekKeep_ = true;
  return true;
}
```



### 2.2 !!! Thread::Get

```cpp
// Get() will process I/O until:
//  1) A message is available (returns true)
//  2) cmsWait seconds have elapsed (returns false)
//  3) Stop() is called (returns false)
// pmsg 获取到的消息
// cmsWait 消息等待时间，间隔时间
bool Thread::Get(Message* pmsg, int cmsWait = kForever, bool process_io = true) {
  // Return and clear peek if present
  // Always return the peek if it exists so there is Peek/Get symmetry
  // 1. 是否存在一个Peek过的消息没有被处理？
  // 优先处理该消息     
  if (fPeekKeep_) {
    *pmsg = msgPeek_;
    fPeekKeep_ = false;
    return true;
  }

  // Get w/wait + timer scan / dispatch + socket / event multiplexer dispatch

  int64_t cmsTotal = cmsWait;
  int64_t cmsElapsed = 0;
  int64_t msStart = TimeMillis();
  int64_t msCurrent = msStart;
  while (true) {
    // Check for posted events
    // 检查所有post消息(即时消息+延迟消息)
    
    // cmsDelayNext 就是下个消息执行还有多久
    int64_t cmsDelayNext = kForever;
    bool first_pass = true;
    while (true) {
      // All queue operations need to be locked, but nothing else in this loop
      // (specifically handling disposed message) can happen inside the crit.
      // Otherwise, disposed MessageHandlers will cause deadlocks.
       // 上锁进行消息队列的访问
      {
        CritScope cs(&crit_);
        // On the first pass, check for delayed messages that have been
        // triggered and calculate the next trigger time.
        // 2. 内部第一次循环，先检查延迟消息队列     
        if (first_pass) {
          first_pass = false;
          // 将延迟消息队列delayed_messages_中已经超过触发时间的消息全部取出放入到即时消息队列messages_中
          // 计算当前时间距离下一个最早将要到达触发时间的消息还有多长时间cmsDelayNext。
          while (!delayed_messages_.empty()) {
            // msCurrent 当前时间
            // delayed_messages_.top().run_time_ms_，队首元素的时间，delayed_messages_有序的队列
            if (msCurrent < delayed_messages_.top().run_time_ms_) {
              // cmsDelayNext = 计算当前时间距离下一个最早将要到达触发时间的消息还有多长时间
              cmsDelayNext =
                  TimeDiff(delayed_messages_.top().run_time_ms_, msCurrent);
              break;
            }
            // 放入消息队列
            messages_.push_back(delayed_messages_.top().msg_);
            // 从延迟队列删除
            delayed_messages_.pop();
          }
        } // ！if (first_pass)
        // Pull a message off the message queue, if available.
        // 3. 从即时消息队列msgq_队首取出第一个消息  
        if (messages_.empty()) {
          // 没有消息，退出第二个while循环， 直接到`if (IsQuitting())`
          break;
        } else {
          // 取出messages_的队首消息， 代表找到要执行的消息
          *pmsg = messages_.front();
          messages_.pop_front();
        }
      }  // crit_ is released here.

      // If this was a dispose message, delete it and skip it.
      // 如果消息对时间敏感，那么如果超过了最大忍耐时间kMaxMsgLatency才被处理
      // 则打印警告日志
      if (MQID_DISPOSE == pmsg->message_id) {
        RTC_DCHECK(nullptr == pmsg->phandler);
        delete pmsg->pdata;
        *pmsg = Message();
        continue;
      }
      // 代表找到要执行的消息
      return true;
    }// while (true) 2

    // 走到这，说明当前没有消息要处理，messages_是空的，很可能是处于Quit状态了，先判断一下
    if (IsQuitting())
      break;

    // Which is shorter, the delay wait or the asked wait?
		// 4. 计算留给IO处理的时间 
    int64_t cmsNext;
    //Get无限期，那么距离下个延迟消息的时间就作为本次IO处理时间
    if (cmsWait == kForever) {
      cmsNext = cmsDelayNext;
    } else {
      //Get有超时时间，计算本次IO处理时间
      //总体来说还剩多少时间
      cmsNext = std::max<int64_t>(0, cmsTotal - cmsElapsed);
      //总体剩余时间和下一个延迟消息触发时间谁先到达？取其短者
      if ((cmsDelayNext != kForever) && (cmsDelayNext < cmsNext))
        cmsNext = cmsDelayNext;
    }

    {
      // Wait and multiplex in the meantime
      // 5.阻塞处理IO多路复用  
      if (!ss_->Wait(static_cast<int>(cmsNext), process_io))
        return false;
    }

    // If the specified timeout expired, return
 		// 6.计算是否所有时间都已耗尽，是否进入下一个大循环 
    msCurrent = TimeMillis();
    // 大循环所耗用的时间长度
    cmsElapsed = TimeDiff(msCurrent, msStart);
    if (cmsWait != kForever) {
      // 大循环所耗用的时间长度 超过 预期的等待时间，则return false；
      if (cmsElapsed >= cmsWait)
        return false;
    }
  }// while (true) 1
  return false;
}
```

- cmsWait， 传入的参数，获取消息的间隔时间，` kForever=-1`就是一直获取；赋值给cmsTotal；
- cmsTotal = cmsWait，
- 
- cmsElapsed，每次大循环使用的时间（包括休眠等待时间）， 初始值是0；每次大循环都会计算一次，在休眠结束后，cmsElapsed = msCurrent-msStart；
- 
- cmsDelayNext， 从delayed_messages_，计算出下一个延迟消息的还有多少时间；每次大循环都要执行一次；
- cmsNext，真正距离下次循环的时间，就是等待休眠时间；
  如果cmsWait == kForever，则cmsNext = cmsDelayNext；
  否则，cmsNext = std::max<int64_t>(0, cmsTotal - cmsElapsed);  cmsTotal - cmsElapsed就是距离下个消息时间-整个循环所消耗的时间；这个消息还要和cmsDelayNext对比，如果cmsNext<cmsDelayNext， 则cmsNext = cmsDelayNext；
- 
- msCurrent， 当前时间戳，会在Get 开始的赋值，循环开始之前；没次循环休眠结束，计算cmsElapsed；
- msStart，会在Get 开始的赋值, 当前时间戳

大概可以分为6个步骤：

1. **检查Peek消息。**先检查之前是否已经Peek过一个消息到msgPeek_还未被处理，若有，当前就处理该消息吧，函数返回~ 若无，继续处理其他消息。
2. **处理Post消息中的延迟消息。**从延迟消息队列 delayed_messages_ 中取出所有已经到达触发时间点的延迟消息，并塞入即时消息队列 messages_ 的队尾。同时计算下一个延迟消息还过多久将被触发（如果延迟消息队列中还有未超时的消息），这个时间可能会被作为后续IO多路复用处理的超时时间。这点在redis，nginx上理念一致。
3. **处理即时消息。**取出即时消息队列msgq_的队首消息。若该消息是个要销毁的消息，那么销毁该消息，并取下一个即时消息；若取到一个非要销毁的即时消息，那么就先处理该即时消息吧，函数返回；若本步骤没有取到即时消息，表示当前没有消息要处理，那干点啥好呢~处理网络IO吧
4. **计算留给网络IO的时间。**消息处理才是迫切的，网络IO嘛，看我能给你分配多少时间吧~~ 分两种情形来对待：
   1）若外部Get方法无限期，那么下一个延迟消息触发时间到来之前我都可以用来处理IO；若是延迟队列中没有延迟消息呢？也就是消息循环队列中没有任何要处理的消息了，那当然我就可以无限期地，阻塞地将时间都用来处理IO了，直到有消息进入消息队列，将消息循环从IO处理中唤醒为止，继续处理消息。
   2）若外部Get方法是有超时时间的，那么我们有必要先计算下已经花费了多长时间，到此刻，我们总共最多还剩多长时间留给IO处理。将总剩余时间跟下一个延迟消息触发时间做个比较，哪个小取哪个作为IO处理的时间；若是延迟队列中没有延迟消息呢？那就将剩下的所有时间都交给IO处理咯，反正也没有消息要处理~
5. **IO多路复用处理。** 阻塞地花费上述计算好的时间进行IO处理。过程中要是处理出错，则函数返回；若是处理时间耗完或者时间没有耗完，但是有新消息进入循环了使得阻塞式的IO处理被唤醒，那么进入下个步骤。
6. **计算剩余时间。**既然消息已经被处理完过一次，IO也处理完了，先计算下是不是所有时间都已经耗尽？耗尽时间了，我还没找到可用的即时消息，sorry~函数返回false；没有耗尽的话，那么我们计算下剩余的时间，并将剩余的时间把2-7过程再来一遍吧：处理Send消息，检查延迟消息，检查并返回即时消息，再次计算IO处理时间，IO处理，再次计算剩余时间。什么？为什么没有重复步骤1检查Peek消息？同一个线程中我既然在执行Get，怎么可能Peek嘛，怎么可能又蹦跶出一个Peek消息呢？

[参考](https://www.jianshu.com/p/b7372d01b819)

## 3. 消息投递

### 3.1 !!! Thread::Post

```cpp
// |time_sensitive| is deprecated and should always be false.
  virtual void Post(const Location& posted_from,
                    MessageHandler* phandler,
                    uint32_t id = 0,
                    MessageData* pdata = nullptr,
                    bool time_sensitive = false);

void Thread::Post(const Location& posted_from,
                  MessageHandler* phandler,
                  uint32_t id,
                  MessageData* pdata,
                  bool time_sensitive) {
	...
  if (IsQuitting()) {
    delete pdata;
    return;
  }

  // Keep thread safe
  // Add the message to the end of the queue
  // Signal for the multiplexer to return

  {
    CritScope cs(&crit_);
    Message msg;
    msg.posted_from = posted_from;
    msg.phandler = phandler;
    msg.message_id = id;
    msg.pdata = pdata;
    messages_.push_back(msg);
  }
  // 线程激活
  WakeUpSocketServer();
}
```

- 如果消息循环已经处理停止状态，即stop_状态值不为0，那么消息循环拒绝消息入队，消息数据会被释放掉。此时，投递消息者是不知情的。
- 如果消息对时间敏感，即想知道该消息是否即时被处理了，最大延迟不超过kMaxMsgLatency 150ms；
- 为了线程安全，队列的入队操作是需要加锁的，CritScope cs(&crit_ )对象的构造和析构确保了这点；
- 消息入队后，由于处理消息是首要任务，因此，需要调用WakeUpSocketServer()使得IO多路复用的处理赶紧返回，即调用ss_->WakeUp()方法实现。由于这块儿是IO多路复用实现内容，后续会专门写文章分析，此处只要知道该方法能够使得阻塞式的IO多路复用能结束阻塞，回到消息处理上来即可。



### 3.2 Thread::PostDelayed

```cpp
void Thread::PostDelayed(const Location& posted_from,
                         int delay_ms,
                         MessageHandler* phandler,
                         uint32_t id,
                         MessageData* pdata) {
  return DoDelayPost(posted_from, delay_ms, TimeAfter(delay_ms), phandler, id,
                     pdata);
}
```



### 3.3 Thread::PostAt

```cpp
void Thread::PostAt(const Location& posted_from,
                    int64_t run_at_ms,
                    MessageHandler* phandler,
                    uint32_t id,
                    MessageData* pdata) {
  return DoDelayPost(posted_from, TimeUntil(run_at_ms), run_at_ms, phandler, id,
                     pdata);
}
```



#### 3.3.1 Thread::DoDelayPost

```cpp
void Thread::DoDelayPost(const Location& posted_from,
                         int64_t delay_ms,
                         int64_t run_at_ms,
                         MessageHandler* phandler,
                         uint32_t id,
                         MessageData* pdata) {
  if (IsQuitting()) {
    delete pdata;
    return;
  }

  // Keep thread safe
  // Add to the priority queue. Gets sorted soonest first.
  // Signal for the multiplexer to return.

  {
    CritScope cs(&crit_);
    Message msg;
    msg.posted_from = posted_from;
    msg.phandler = phandler;
    msg.message_id = id;
    msg.pdata = pdata;
    DelayedMessage delayed(delay_ms, run_at_ms, delayed_next_num_, msg);
    delayed_messages_.push(delayed);
    // If this message queue processes 1 message every millisecond for 50 days,
    // we will wrap this number.  Even then, only messages with identical times
    // will be misordered, and then only briefly.  This is probably ok.
    ++delayed_next_num_;
    RTC_DCHECK_NE(0, delayed_next_num_);
  }
  // 线程激活
  WakeUpSocketServer();
}
```



### 3.4 !!! Thead::Invoke——同步发送消息

Invoke方法提供了一个方便的方式：阻塞当前线程，在另外一个线程上同步执行方法，并且返回执行结果。
本质上就是将需要执行的方法封装到消息处理器FunctorMessageHandler中，然后向目标线程Send这个携带特殊消息处理器FunctorMessageHandler的消息，该消息被消费后，阻塞结束，FunctorMessageHandler对象携带了方法执行的结果，当前线程可以从中获取到执行结果。其实，这里的重点有二：

- FunctorMessageHandler类的封装，见[WebRTC源码分析-线程基础之Message && MessageData && MesaageHandler](https://www.jianshu.com/p/f0dca647c4fa)
- Invoke的本质是调用Send消息的方式来执行方法。

```cpp
// Convenience method to invoke a functor on another thread.  Caller must
  // provide the |ReturnT| template argument, which cannot (easily) be deduced.
  // Uses Send() internally, which blocks the current thread until execution
  // is complete.
  // Ex: bool result = thread.Invoke<bool>(RTC_FROM_HERE,
  // &MyFunctionReturningBool);
  // NOTE: This function can only be called when synchronous calls are allowed.
  // See ScopedDisallowBlockingCalls for details.
  // NOTE: Blocking invokes are DISCOURAGED, consider if what you're doing can
  // be achieved with PostTask() and callbacks instead.

// 返回值 ReturnT
  template <
      class ReturnT,
      typename = typename std::enable_if<!std::is_void<ReturnT>::value>::type>
  ReturnT Invoke(const Location& posted_from, FunctionView<ReturnT()> functor) {
    ReturnT result;
    InvokeInternal(posted_from, [functor, &result] { result = functor(); });
    return result;
  }

// 返回值 void
  template <
      class ReturnT,
      typename = typename std::enable_if<std::is_void<ReturnT>::value>::type>
  void Invoke(const Location& posted_from, FunctionView<void()> functor) {
    InvokeInternal(posted_from, functor);
  }
```



#### 3.4.1 Thread::InvokeInternal

```cpp
void Thread::InvokeInternal(const Location& posted_from,
                            rtc::FunctionView<void()> functor) {
  TRACE_EVENT2("webrtc", "Thread::Invoke", "src_file", posted_from.file_name(),
               "src_func", posted_from.function_name());

  class FunctorMessageHandler : public MessageHandler {
   public:
    explicit FunctorMessageHandler(rtc::FunctionView<void()> functor)
        : functor_(functor) {}
    void OnMessage(Message* msg) override { functor_(); }

   private:
    rtc::FunctionView<void()> functor_;
  } handler(functor);

  Send(posted_from, &handler);
}
```



#### 3.4.2 !!! Thread::Send

```cpp
void Thread::Send(const Location& posted_from,
                  MessageHandler* phandler,
                  uint32_t id,
                  MessageData* pdata) {
  RTC_DCHECK(!IsQuitting());
  if (IsQuitting())
    return;

  // Sent messages are sent to the MessageHandler directly, in the context
  // of "thread", like Win32 SendMessage. If in the right context,
  // call the handler directly.
  Message msg;
  msg.posted_from = posted_from;
  msg.phandler = phandler;
  msg.message_id = id;
  msg.pdata = pdata;
  // 1. 若目标线程就是自己，那么直接在此处处理完消息就ok
  if (IsCurrent()) {
    msg.phandler->OnMessage(&msg);
    return;
  }
  
 	// 2. 断言当前线程是否具有阻塞权限，无阻塞权限
  // 那么向别的线程Send消息就是个非法操作
  AssertBlockingIsAllowedOnCurrentThread();

  // 3. 确保当前线程有一个Thread对象与之绑定
  // 这是调用的线程，不是workthread，workthread是目标线程，是task执行的线程
  // 同步就是阻塞当前线程
  Thread* current_thread = Thread::Current();

#if RTC_DCHECK_IS_ON
  if (current_thread) {
    RTC_DCHECK(current_thread->IsInvokeToThreadAllowed(this));
    ThreadManager::Instance()->RegisterSendAndCheckForCycles(current_thread,
                                                             this);
  }
#endif

  // Perhaps down the line we can get rid of this workaround and always require
  // current_thread to be valid when Send() is called.
  // 如果当前线程没有绑定，current_thread == null
  std::unique_ptr<rtc::Event> done_event;
  if (!current_thread)
    done_event.reset(new rtc::Event());

  bool ready = false;
  // 4. 发送异步消息，创建了ClosureTaskWithCleanup
  PostTask(webrtc::ToQueuedTask(
      [&msg]() mutable { msg.phandler->OnMessage(&msg);  // 这里处理消息 
			},
      [this, &ready, current_thread, done = done_event.get()] {
        // 这里ClosureTaskWithCleanup 对象是否的时候，代表消息处理完成，任务结束，异步任务结束
        if (current_thread) {
          CritScope cs(&crit_);
          ready = true;
          current_thread->socketserver()->WakeUp();
        } else {
          done->Set();
        }
      }));

  if (current_thread) {
    // 当前线程阻塞，等待
    bool waited = false;
    crit_.Enter();
    while (!ready) {
      crit_.Leave();
      // 一直等待，到 postTask 任务执行完成，WakeUp， ready=true，跳出循环
      current_thread->socketserver()->Wait(kForever, false);
      waited = true;
      crit_.Enter();
    }
    crit_.Leave();

    // Our Wait loop above may have consumed some WakeUp events for this
    // Thread, that weren't relevant to this Send.  Losing these WakeUps can
    // cause problems for some SocketServers.
    //
    // Concrete example:
    // Win32SocketServer on thread A calls Send on thread B.  While processing
    // the message, thread B Posts a message to A.  We consume the wakeup for
    // that Post while waiting for the Send to complete, which means that when
    // we exit this loop, we need to issue another WakeUp, or else the Posted
    // message won't be processed in a timely manner.
    
		//  如果出现过waited，那么再唤醒一次当前线程去处理Post消息。
    if (waited) {
      current_thread->socketserver()->WakeUp();
    }
  } else { // !end if (current_thread)
    done_event->Wait(rtc::Event::kForever);
  }
}
```

要理解上述算法，需要搞清楚Send方法的代码是在当前线程执行的，而调用的是目标线程对象Thread的Send方法，即Send方法中的this，是目标线程线程对象Thread。

1. **判断目标线程是否就是当前线程：** 通过Thread.IsCurrent()可以判别这点，如果目标线程就是当前线程，那就是自己给自己Send消息了，直接在此处消费消息并返回。
2. **断言当前线程是否允许阻塞：** 注意，这儿不是断言目标线程。因为，向另外一个线程Send消息时，当前线程需要阻塞地等待目标线程处理完消息后才返回。如果，当前线程没有阻塞权限的话，那就是非法操作了。
3. **确保当前线程有一个关联的Thread对象：** 为什么？因为后续的阻塞唤醒操作都要通过Thread对象的方法来实现，如果当前线程没有关联Thread对象，那么这些操作就无法完成。怎么做？通过创建一个局部对象AutoThread thread来实现。源码如下，注意两点：
   1）只有当当前线程无Thread关联时，才会将AutoThread作为当前线程的关联Thread；
   2）由于AutoThread thread是局部对象，当Send函数结束时该对象生命周期走到尾声，可以利用其析构函数中需要恢复当前对象无Thread对象绑定的状态(当然，前提是之前就无Thread对象关联)。



#### 3.4.3 ToQueuedTask

rtc_base/task_utils/to_queued_task.h

```cpp
template <typename Closure,
          typename Cleanup,
          typename std::enable_if<!std::is_same<
              typename std::remove_const<
                  typename std::remove_reference<Closure>::type>::type,
              ScopedTaskSafety>::value>::type* = nullptr>
std::unique_ptr<QueuedTask> ToQueuedTask(Closure&& closure, Cleanup&& cleanup) {
  return std::make_unique<
      webrtc_new_closure_impl::ClosureTaskWithCleanup<Closure, Cleanup>>(
      std::forward<Closure>(closure), std::forward<Cleanup>(cleanup));
}

```



### 3.5 Thread::PostTask

```cpp
// Posts a task to invoke the functor on |this| thread asynchronously, i.e.
  // without blocking the thread that invoked PostTask(). Ownership of |functor|
  // is passed and (usually, see below) destroyed on |this| thread after it is
  // invoked.
  // Requirements of FunctorT:
  // - FunctorT is movable.
  // - FunctorT implements "T operator()()" or "T operator()() const" for some T
  //   (if T is not void, the return value is discarded on |this| thread).
  // - FunctorT has a public destructor that can be invoked from |this| thread
  //   after operation() has been invoked.
  // - The functor must not cause the thread to quit before PostTask() is done.
  //
  // Destruction of the functor/task mimics what TaskQueue::PostTask does: If
  // the task is run, it will be destroyed on |this| thread. However, if there
  // are pending tasks by the time the Thread is destroyed, or a task is posted
  // to a thread that is quitting, the task is destroyed immediately, on the
  // calling thread. Destroying the Thread only blocks for any currently running
  // task to complete. Note that TQ abstraction is even vaguer on how
  // destruction happens in these cases, allowing destruction to happen
  // asynchronously at a later time and on some arbitrary thread. So to ease
  // migration, don't depend on Thread::PostTask destroying un-run tasks
  // immediately.
  //
  // Example - Calling a class method:
  // class Foo {
  //  public:
  //   void DoTheThing();
  // };
  // Foo foo;
  // thread->PostTask(RTC_FROM_HERE, Bind(&Foo::DoTheThing, &foo));
  //
  // Example - Calling a lambda function:
  // thread->PostTask(RTC_FROM_HERE,
  //                  [&x, &y] { x.TrackComputations(y.Compute()); });
  template <class FunctorT>
  void PostTask(const Location& posted_from, FunctorT&& functor) {
    Post(posted_from, GetPostTaskMessageHandler(), /*id=*/0,
         new rtc_thread_internal::MessageWithFunctor<FunctorT>(
             std::forward<FunctorT>(functor)));
  }
```



### 3.6 Thread::PostDelayedTask

```cpp
  template <class FunctorT>
  void PostDelayedTask(const Location& posted_from,
                       FunctorT&& functor,
                       uint32_t milliseconds) {
    PostDelayed(posted_from, milliseconds, GetPostTaskMessageHandler(),
                /*id=*/0,
                new rtc_thread_internal::MessageWithFunctor<FunctorT>(
                    std::forward<FunctorT>(functor)));
  }
```



### 3.7 TaskQueueBase::PostTask

```cpp
void Thread::PostTask(std::unique_ptr<webrtc::QueuedTask> task) {
  // Though Post takes MessageData by raw pointer (last parameter), it still
  // takes it with ownership.
  Post(RTC_FROM_HERE, &queued_task_handler_,
       /*id=*/0, new ScopedMessageData<webrtc::QueuedTask>(std::move(task)));
}
```



### 3.8 TaskQueueBase::PostDelayedTask

```cpp
void Thread::PostDelayedTask(std::unique_ptr<webrtc::QueuedTask> task,
                             uint32_t milliseconds) {
  // Though PostDelayed takes MessageData by raw pointer (last parameter),
  // it still takes it with ownership.
  PostDelayed(RTC_FROM_HERE, milliseconds, &queued_task_handler_,
              /*id=*/0,
              new ScopedMessageData<webrtc::QueuedTask>(std::move(task)));
}
```

- QueuedTaskHandler queued_task_handler_; 参考9.3
- ScopedMessageData，参考8.2



### 3.9 TaskQueueBase::Delete

```cpp
void Thread::Delete() {
  Stop();
  delete this;
}
```



### 3.10 TaskQueueBase::PostTask 和 Thread::PostTask 区别

1. TaskQueueBase::PostTask 和 Thread::PostTask 都调用了 TaskQueueBase::Post

2. 输入参数不同

   ```cpp
   TaskQueueBase::PostTask(std::unique_ptr<webrtc::QueuedTask> task){
     Post(RTC_FROM_HERE, &queued_task_handler_,
          /*id=*/0, new ScopedMessageData<webrtc::QueuedTask>(std::move(task)));
   }
   
   Thread::PostTask(const Location& posted_from, FunctorT&& functor) {
     Post(posted_from, GetPostTaskMessageHandler(), /*id=*/0,
            new rtc_thread_internal::MessageWithFunctor<FunctorT>(
                std::forward<FunctorT>(functor)));
   }
   
   Thread::Post(const Location& posted_from,
                     MessageHandler* phandler,
                     uint32_t id,
                     MessageData* pdata,
                     bool time_sensitive)
   ```

   | 参数                        | TaskQueueBase::PostTask                | Thread::PostTask                        |
   | --------------------------- | -------------------------------------- | --------------------------------------- |
   | const Location& posted_from | RTC_FROM_HERE                          | 调用者来指定                            |
   | MessageHandler* phandler    | QueuedTaskHandler                      | MessageHandlerWithTask                  |
   | uint32_t id,                | 0                                      | 0                                       |
   | MessageData* pdata          | ScopedMessageData\<webrtc::QueuedTask> | rtc_thread_internal::MessageWithFunctor |
   | bool time_sensitive         | /                                      | /                                       |





## 4. 消息处理

### 4.1 !!! Thread::Dispatch

```cpp
void Thread::Dispatch(Message* pmsg) {
	...
  int64_t start_time = TimeMillis();
  // 处理消息
  pmsg->phandler->OnMessage(pmsg);
  
  int64_t end_time = TimeMillis();
  int64_t diff = TimeDiff(end_time, start_time);
  if (diff >= kSlowDispatchLoggingThreshold) {
    RTC_LOG(LS_INFO) << "Message took " << diff
                     << "ms to dispatch. Posted from: "
                     << pmsg->posted_from.ToString();
  }
}
```

- 记录消息处理的开始时间start_time；
- 调用消息的MessageHandler的OnMessage方法进行消息处理；
- 记录消息处理的结束时间end_time；
- 计算消息处理花费了多长时间diff，如果消息花费时间过程，超过kSlowDispatchLoggingThreshold（50ms），则打印一条警告日志，告知从哪儿构建的消息花费了多长时间才消费完。

#### const int kSlowDispatchLoggingThreshold = 50;  // 50 ms





## 5. 消息数目

### 1.8 Message 消息个数

```cpp
  bool empty() const { return size() == 0u; }

  size_t size() const {
    CritScope cs(&crit_);
    return messages_.size() + delayed_messages_.size() + (fPeekKeep_ ? 1u : 0u);
  }
```

由于MQ有3个地方存储了消息，一个是Peek消息msgPeek_ ，一个即时消息队列 messages_ ，一个是延迟消息队列 delayed_messages_。那么计算MQ中存储的消息个数时，这三个地方都得算上。



## 6. 消息清理

### 6.1 Thread::Clear

```cpp
const uint32_t MQID_ANY = static_cast<uint32_t>(-1);
virtual void Clear(MessageHandler* phandler,
                   uint32_t id = MQID_ANY,
                   MessageList* removed = nullptr);

void Thread::Clear(MessageHandler* phandler,
                   uint32_t id,
                   MessageList* removed) {
  CritScope cs(&crit_);
  ClearInternal(phandler, id, removed);
}

```



### 6.2 Thread::ClearInternal

```cpp
void Thread::ClearInternal(MessageHandler* phandler,
                           uint32_t id,
                           MessageList* removed) {
  // Remove messages with phandler

  if (fPeekKeep_ && msgPeek_.Match(phandler, id)) {
    if (removed) {
      removed->push_back(msgPeek_);
    } else {
      delete msgPeek_.pdata;
    }
    fPeekKeep_ = false;
  }

  // Remove from ordered message queue

  for (auto it = messages_.begin(); it != messages_.end();) {
    if (it->Match(phandler, id)) {
      if (removed) {
        removed->push_back(*it);
      } else {
        delete it->pdata;
      }
      it = messages_.erase(it);
    } else {
      ++it;
    }
  }

  // Remove from priority queue. Not directly iterable, so use this approach

  auto new_end = delayed_messages_.container().begin();
  for (auto it = new_end; it != delayed_messages_.container().end(); ++it) {
    if (it->msg_.Match(phandler, id)) {
      if (removed) {
        removed->push_back(it->msg_);
      } else {
        delete it->msg_.pdata;
      }
    } else {
      *new_end++ = *it;
    }
  }
  delayed_messages_.container().erase(new_end,
                                      delayed_messages_.container().end());
  delayed_messages_.reheap();
}
```

- MQ中消息可能存在的位置有3个：Peek消息msgPeek_ ，即时消息队列msgq_ ，延迟消息队列 dmsgq_ ；因此，需要从这3个地方去挨个查找能匹配的消息。
- 如果Clear()方法传入了一个MessageList* removed，匹配的消息都会进入该list；若是没有传入这样一个list，那么消息数据都将会立马销毁。



## 7. 销毁消息

### 7.1 Thread::Dispose

```cpp
const uint32_t MQID_DISPOSE = static_cast<uint32_t>(-2);

// Internally posts a message which causes the doomed object to be deleted
  template <class T>
  void Dispose(T* doomed) {
    if (doomed) {
      Post(RTC_FROM_HERE, nullptr, MQID_DISPOSE, new DisposeData<T>(doomed));
    }
  }
```





## 8. MessageData——发送的消息数据

以下是继承关系

```
MessageData
ScopedMessageData			MessageLikeTask
											MessageWithFunctor
```



### 8.1 MessageData——基类

rtc_base/thread_message.h

```cpp
class MessageData {
 public:
  MessageData() {}
  virtual ~MessageData() {}
};
```



### 8.2 !!! ScopedMessageData——任何数据类型，比如Task

存放对应的数据，可以任何数据类型，这里就可以翻入 Task（小节10）

rtc_base/thread_message.h

```cpp
// Like TypedMessageData, but for pointers that require a delete.
template <class T>
class ScopedMessageData : public MessageData {
 public:
  explicit ScopedMessageData(std::unique_ptr<T> data)
      : data_(std::move(data)) {}
  // Deprecated.
  // TODO(deadbeef): Remove this once downstream applications stop using it.
  explicit ScopedMessageData(T* data) : data_(data) {}
  // Deprecated.
  // TODO(deadbeef): Returning a reference to a unique ptr? Why. Get rid of
  // this once downstream applications stop using it, then rename inner_data to
  // just data.
  const std::unique_ptr<T>& data() const { return data_; }
  std::unique_ptr<T>& data() { return data_; }

  const T& inner_data() const { return *data_; }
  T& inner_data() { return *data_; }

 private:
  std::unique_ptr<T> data_;
};
```



### 8.3 !!! MessageLikeTask

```cpp
class MessageLikeTask : public MessageData {
 public:
  virtual void Run() = 0;
};
```



### 8.4 !!! MessageWithFunctor——自定义function

用户定义的functon，切换到对应线程执行。

rtc_base/thread.h

```cpp
template <class FunctorT>
class MessageWithFunctor final : public MessageLikeTask {
 public:
  explicit MessageWithFunctor(FunctorT&& functor)
      : functor_(std::forward<FunctorT>(functor)) {}

  void Run() override { functor_(); }

 private:
  ~MessageWithFunctor() override {}

  typename std::remove_reference<FunctorT>::type functor_;

  RTC_DISALLOW_COPY_AND_ASSIGN(MessageWithFunctor);
};

}  // namespace rtc_thread_internal
```



## 9. MessageHandler——消息接收者

收到的消息进行处理，主要是`OnMessage`。
以下是继承关系：

```
MessageHandler
MessageHandlerWithTask		QueuedTaskHandler
```



### 9.1 MessageHandler——基类

```cpp
class RTC_EXPORT MessageHandler {
 public:
  virtual ~MessageHandler() {}
  virtual void OnMessage(Message* msg) = 0;
};
```



### 9.2 !!! MessageHandlerWithTask——对应MessageWithFunctor

rtc_base/thread.cc

```cpp
class MessageHandlerWithTask final : public MessageHandler {
 public:
  MessageHandlerWithTask() {}

  void OnMessage(Message* msg) override {
    static_cast<rtc_thread_internal::MessageLikeTask*>(msg->pdata)->Run();
    delete msg->pdata;
  }

 private:
  ~MessageHandlerWithTask() override {}

  RTC_DISALLOW_COPY_AND_ASSIGN(MessageHandlerWithTask);
};
```



### 9.3 !!! QueuedTaskHandler

rtc_base/thread.h

需要和Task相结合，

```cpp
  class QueuedTaskHandler final : public MessageHandler {
   public:
    QueuedTaskHandler() {}
    void OnMessage(Message* msg) override;
  };

void Thread::QueuedTaskHandler::OnMessage(Message* msg) {
  RTC_DCHECK(msg);
  auto* data = static_cast<ScopedMessageData<webrtc::QueuedTask>*>(msg->pdata);
  std::unique_ptr<webrtc::QueuedTask> task = std::move(data->data());
  // Thread expects handler to own Message::pdata when OnMessage is called
  // Since MessageData is no longer needed, delete it.
  delete data;

  // QueuedTask interface uses Run return value to communicate who owns the
  // task. false means QueuedTask took the ownership.
  if (!task->Run())
    task.release();
}
```



## 10. Task——自定义任务，实现Run方法

对应PostTask，中输入的参数是QueuedTask的接口。其实就是需要用户实现Run方法；
以下是继承关系：

```
QueuedTask
ClosureTask					SafetyClosureTask
ClosureTaskWithCleanup
```



### 10.1 QueuedTask——基类

api/task_queue/queued_task.h

```cpp
// Base interface for asynchronously executed tasks.
// The interface basically consists of a single function, Run(), that executes
// on the target queue.  For more details see the Run() method and TaskQueue.
class QueuedTask {
 public:
  virtual ~QueuedTask() = default;

  // Main routine that will run when the task is executed on the desired queue.
  // The task should return |true| to indicate that it should be deleted or
  // |false| to indicate that the queue should consider ownership of the task
  // having been transferred.  Returning |false| can be useful if a task has
  // re-posted itself to a different queue or is otherwise being re-used.
  virtual bool Run() = 0;
};
```



### 10.2 ClosureTask——闭包

```cpp
template <typename Closure>
class ClosureTask : public QueuedTask {
 public:
  explicit ClosureTask(Closure&& closure)
      : closure_(std::forward<Closure>(closure)) {}

 private:
  bool Run() override {
    closure_();
    return true;
  }

  typename std::decay<Closure>::type closure_;
};
```



### 10.3 !!! ClosureTaskWithCleanup

```cpp
// Extends ClosureTask to also allow specifying cleanup code.
// This is useful when using lambdas if guaranteeing cleanup, even if a task
// was dropped (queue is too full), is required.
template <typename Closure, typename Cleanup>
class ClosureTaskWithCleanup : public ClosureTask<Closure> {
 public:
  ClosureTaskWithCleanup(Closure&& closure, Cleanup&& cleanup)
      : ClosureTask<Closure>(std::forward<Closure>(closure)),
        cleanup_(std::forward<Cleanup>(cleanup)) {}
  ~ClosureTaskWithCleanup() override { cleanup_(); }

 private:
  typename std::decay<Cleanup>::type cleanup_;
};
```



### 10.4 SafetyClosureTask

```cpp
template <typename Closure>
class SafetyClosureTask : public QueuedTask {
 public:
  explicit SafetyClosureTask(rtc::scoped_refptr<PendingTaskSafetyFlag> safety,
                             Closure&& closure)
      : closure_(std::forward<Closure>(closure)),
        safety_flag_(std::move(safety)) {}

 private:
  bool Run() override {
    if (safety_flag_->alive())
      closure_();
    return true;
  }

  typename std::decay<Closure>::type closure_;
  rtc::scoped_refptr<PendingTaskSafetyFlag> safety_flag_;
};
```





## 12. 调用堆栈

### 12.1 PostTask

```cpp
TaskQueueBase::PostTask
  	RTC_FROM_HERE									// 对应post的第一参数
  	GetPostTaskMessageHandler			// 对应post的第二参数
  		new MessageHandlerWithTask	
  	new rtc_thread_internal::MessageWithFunctor // 对应post的第三参数
Thread::Post
	messages_::push_back // 消息队列
Thread::WakeUpSocketServer
  Win32SocketServer::WakeUp
  PostMessage // 系统Api
```



### 12.2 PostDelayedTask

```js
TaskQueueBase::PostDelayedTask
  Thread::PostDelayed
  Thread::DoDelayPost
  Thread::WakeUpSocketServer
```



### 12.3 Invoke

```js
Thread::Invoke
Thread::InvokeInternal
Thread::Send
  PostTask
```



## 13. 参考

[WebRTC源码分析-线程基础之消息循环，消息投递](https://www.jianshu.com/p/a3547ffefbeb)

[WebRTC源码分析-线程基础之MessageQueue](https://www.jianshu.com/p/b7372d01b819)
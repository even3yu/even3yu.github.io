---
layout: post
title: android SurfaceTextureHelper
date: 2023-12-07 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android SurfaceTexture
---


* content
{:toc}

---


## 1. SurfaceTextureHelper.create

sdk/android/api/org/webrtc/SurfaceTextureHelper.java

```java
  /**
   * Construct a new SurfaceTextureHelper sharing OpenGL resources with |sharedContext|. A dedicated
   * thread and handler is created for handling the SurfaceTexture. May return null if EGL fails to
   * initialize a pixel buffer surface and make it current. If alignTimestamps is true, the frame
   * timestamps will be aligned to rtc::TimeNanos(). If frame timestamps are aligned to
   * rtc::TimeNanos() there is no need for aligning timestamps again in
   * PeerConnectionFactory.createVideoSource(). This makes the timestamps more accurate and
   * closer to actual creation time.
   */
  public static SurfaceTextureHelper create(
    	// 线程名称
      final String threadName,
			// OpenGL 上下文
      final EglBase.Context sharedContext, 
    	// 时间戳对齐
      boolean alignTimestamps, 
    	// yuv数据转换，把纹理转换为yuv数据
      final YuvConverter yuvConverter,
    	// 监听数据帧
      FrameRefMonitor frameRefMonitor) {
    
    // 创建线程，线程安全
    final HandlerThread thread = new HandlerThread(threadName);
    thread.start();
    final Handler handler = new Handler(thread.getLooper());

    // The onFrameAvailable() callback will be executed on the SurfaceTexture ctor thread. See:
    // http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/graphics/SurfaceTexture.java#195.
    // Therefore, in order to control the callback thread on API lvl < 21, the SurfaceTextureHelper
    // is constructed on the |handler| thread.
    // hreadUtils.invokeAtFrontUninterruptibly同步执行， 确保SurfaceTextureHelper 在HandlerThread 执行
    return ThreadUtils.invokeAtFrontUninterruptibly(handler, new Callable<SurfaceTextureHelper>() {
      @Nullable
      @Override
      public SurfaceTextureHelper call() {
        try {
          return new SurfaceTextureHelper(
              sharedContext, handler, alignTimestamps, yuvConverter, frameRefMonitor);
        } catch (RuntimeException e) {
          Logging.e(TAG, threadName + " create failure", e);
          return null;
        }
      }
    });
  }
```

1. 创建`HandlerThread`， 提供给`Handler`; 用于线程安全使用；
2. 同步执行`ThreadUtils.invokeAtFrontUninterruptibly`，确保`SurfaceTextureHelper` 在 `HandlerThread` 创建； 这里很重要，确保`SurfaceTextureHelper`， 确保 `SurfaceTexture` 在`HandlerThread`， 这样 `setOnFrameAvailableListener` 回调到 `HandlerThread`;



## 2. SurfaceTextureHelper.SurfaceTextureHelper

```java
private SurfaceTextureHelper(Context sharedContext, 
                             Handler handler, 
    												 // 时间戳对齐
                             boolean alignTimestamps,
    												 // yuv数据转换，把纹理转换为yuv数据
                             YuvConverter yuvConverter, 
    												 // 监听数据帧
                             FrameRefMonitor frameRefMonitor) {
    if (handler.getLooper().getThread() != Thread.currentThread()) {
      throw new IllegalStateException("SurfaceTextureHelper must be created on the handler thread");
    }
    this.handler = handler;
    this.timestampAligner = alignTimestamps ? new TimestampAligner() : null;
    this.yuvConverter = yuvConverter;
    this.frameRefMonitor = frameRefMonitor;

  	// EglBase.CONFIG_PIXEL_BUFFER？？？？
    eglBase = EglBase.create(sharedContext, EglBase.CONFIG_PIXEL_BUFFER);
    try {
      // Both these statements have been observed to fail on rare occasions, see BUG=webrtc:5682.
      eglBase.createDummyPbufferSurface();
      eglBase.makeCurrent();
    } catch (RuntimeException e) {
      // Clean up before rethrowing the exception.
      eglBase.release();
      handler.getLooper().quit();
      throw e;
    }

  	// 创建 oesTextureId
    oesTextureId = GlUtil.generateTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES);
  	// 创建 SurfaceTexture
    surfaceTexture = new SurfaceTexture(oesTextureId);
  
  	// 设置监听，通知有数据放入到 SurfaceTexture，
    setOnFrameAvailableListener(surfaceTexture, (SurfaceTexture st) -> {
      if (hasPendingTexture) {
        Logging.d(TAG, "A frame is already pending, dropping frame.");
      }
			//////！！！！！！
			//////！！！！！！
			//////！！！！！！
      // 这里就说，有数据回调回来了，输入到SurfaceTexture
			//////！！！！！！
			//////！！！！！！
			//////！！！！！！
      //
      // hasPendingTexture = true 正在处理Texture
      hasPendingTexture = true;
      tryDeliverTextureFrame();
    }, handler);
  }
```

1. 这里确保`SurfaceTextureHelper` 在 `HandlerThread` 创建；

1. 为什么要创建EglBase？？？？

   ```java
       eglBase = EglBase.create(sharedContext, EglBase.CONFIG_PIXEL_BUFFER);
       try {
         // Both these statements have been observed to fail on rare occasions, see BUG=webrtc:5682.
         eglBase.createDummyPbufferSurface();
         eglBase.makeCurrent();
       } catch (RuntimeException e) {
         // Clean up before rethrowing the exception.
         eglBase.release();
         handler.getLooper().quit();
         throw e;
       }
   ```

2. 创建纹理 `GLES11Ext.GL_TEXTURE_EXTERNAL_OES`

   ```java
     	// 创建 oesTextureId
       oesTextureId = GlUtil.generateTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES);
   ```

3. 创建`SurfaceTexture`

4. 数据监听`setOnFrameAvailableListener`， 数据回调回来，就是从`SurfaceTexture` 输出，输出的是opengl 的external纹理。



### 2.1 ??? tryDeliverTextureFrame

参考

### 2.2 属性

```java
  private final Handler handler;
  private final EglBase eglBase;
  private final SurfaceTexture surfaceTexture;
	// exteranl 纹理，就是存放了SurfaceTexture 输出的纹理
  private final int oesTextureId;
	// 纹理转yuv
  private final YuvConverter yuvConverter;
	// ？？？
  @Nullable private final TimestampAligner timestampAligner;
	// frame 的生命周期
  private final FrameRefMonitor frameRefMonitor;
```



### 2.3 setOnFrameAvailableListener

```java
  private static void setOnFrameAvailableListener(SurfaceTexture surfaceTexture,
      SurfaceTexture.OnFrameAvailableListener listener, Handler handler) {
    // android 5.0
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
      // 这里是确保回调的线程，都回调到HandlerThread
      surfaceTexture.setOnFrameAvailableListener(listener, handler);
    } else {
      // 回调到 SurfaceTexture 创建的线程
      // The documentation states that the listener will be called on an arbitrary thread, but in
      // pratice, it is always the thread on which the SurfaceTexture was constructed. There are
      // assertions in place in case this ever changes. For API >= 21, we use the new API to
      // explicitly specify the handler.
      surfaceTexture.setOnFrameAvailableListener(listener);
    }
  }
```

1. 注册回调，确保回调的线程 ，在`HandlerThread`。
2. 触发这个回调，说明有数据输入到`SurfaceTexture`。



## 3. 设置VideoSink监听

### 3.1 VideoSink pendingListener

这个是外部传入的临时保存的变量，真正的保存到  `VideoSink listener`; 



###  3.2 VideoSink listener

这个是真正内部是用的listener，具体的看`Runnable setListenerRunnable` 代码。



### 3.3 startListening

```java
  /**
   * Start to stream textures to the given |listener|. If you need to change listener, you need to
   * call stopListening() first.
   */
  public void startListening(final VideoSink listener) {
    if (this.listener != null || this.pendingListener != null) {
      throw new IllegalStateException("SurfaceTextureHelper listener has already been set.");
    }
    // 先临时保存
    this.pendingListener = listener;
    // 最终在HandlerThread线程了赋值给this.listener；
    handler.post(setListenerRunnable);
  }
```

注意，如果需要变化listener，则需要先调用 `stopListening`。



### 3.4 stopListening

```java
 /**
   * Stop listening. The listener set in startListening() is guaranteded to not receive any more
   * onFrame() callbacks after this function returns.
   */
  public void stopListening() {
    Logging.d(TAG, "stopListening()");
    handler.removeCallbacks(setListenerRunnable);
    // 这里是同步执行， 这样能确保设置之后，onframe 不会再被调用
    ThreadUtils.invokeAtFrontUninterruptibly(handler, () -> {
      listener = null;
      pendingListener = null;
    });
  }
```



### 3.5 Runnable setListenerRunnable

```java
  final Runnable setListenerRunnable = new Runnable() {
    @Override
    public void run() {
      Logging.d(TAG, "Setting listener to " + pendingListener);
      // 把临时pendingListener 存到了 this.listener， 清空pendingListener
      listener = pendingListener;
      pendingListener = null;
      // May have a pending frame from the previous capture session - drop it.
      if (hasPendingTexture) {
        // Calling updateTexImage() is neccessary in order to receive new frames.
        updateTexImage();
        hasPendingTexture = false;
      }
    }
  };
```

1. 把临时pendingListener 存到了 this.listener， 清空pendingListener；
2. `updateTexImage`获取新的 frame；
3. 关于变量`hasPendingTexture` 后面回说明；【参考】



## 4. 设置图像属性

### 4.1 setTextureSize

```java
  /**
   * Use this function to set the texture size. Note, do not call setDefaultBufferSize() yourself
   * since this class needs to be aware of the texture size.
   */
  public void setTextureSize(int textureWidth, int textureHeight) {
    if (textureWidth <= 0) {
      throw new IllegalArgumentException("Texture width must be positive, but was " + textureWidth);
    }
    if (textureHeight <= 0) {
      throw new IllegalArgumentException(
          "Texture height must be positive, but was " + textureHeight);
    }
    surfaceTexture.setDefaultBufferSize(textureWidth, textureHeight);
    // 确保线程安全
    handler.post(() -> {
      this.textureWidth = textureWidth;
      this.textureHeight = textureHeight;
      tryDeliverTextureFrame();
    });
  }
```



### 4.2 setFrameRotation

```java
  /** Set the rotation of the delivered frames. */
  public void setFrameRotation(int rotation) {
    // 确保线程安全
    handler.post(() -> this.frameRotation = rotation);
  }
```



## ------资源回收------

## 5. dispose

```java
  public void dispose() {
    Logging.d(TAG, "dispose()");
    // 同步释放
    ThreadUtils.invokeAtFrontUninterruptibly(handler, () -> {
      // 标识停止继续获取frame
      isQuitting = true;
      // 如果isTextureInUse 没在使用，则直接释放，否则就不释放
      if (!isTextureInUse) {
        release();
      }
    });
  }
```

1. `isQuitting` 标识停止继续获取frame；
2. 如果`isTextureInUse` 没在使用，则直接释放，否则就不释放， 那么什么释放，在`returnTextureFrame` 释放；



### 5.1 release

```java
	private void release() {
    if (handler.getLooper().getThread() != Thread.currentThread()) {
      throw new IllegalStateException("Wrong thread.");
    }
    if (isTextureInUse || !isQuitting) {
      throw new IllegalStateException("Unexpected release.");
    }
    yuvConverter.release();
    GLES20.glDeleteTextures(1, new int[] {oesTextureId}, 0);
    surfaceTexture.release();
    eglBase.release();
    handler.getLooper().quit();
    if (timestampAligner != null) {
      timestampAligner.dispose();
    }
  }
```





## ------Buffer------

## 6. 类的继承关系

```less
RefCounted		// 接口
VideoFrame
```



```less
RefCounted		// 接口
VideoFrame.Buffer				// 接口
VideoFrame.I420Buffer			VideoFrame.TextureBuffer 		// 接口
													TextureBufferImpl		// 实现
```





## 7. RefCountMonitor

```java
  private final RefCountMonitor textureRefCountMonitor = new RefCountMonitor() {
    @Override
    public void onRetain(TextureBufferImpl textureBuffer) {
      if (frameRefMonitor != null) {
        frameRefMonitor.onRetainBuffer(textureBuffer);
      }
    }

    @Override
    public void onRelease(TextureBufferImpl textureBuffer) {
      if (frameRefMonitor != null) {
        frameRefMonitor.onReleaseBuffer(textureBuffer);
      }
    }

    @Override
    public void onDestroy(TextureBufferImpl textureBuffer) {
      // !!!!!!!!!!
      returnTextureFrame();
      if (frameRefMonitor != null) {
        frameRefMonitor.onDestroyBuffer(textureBuffer);
      }
    }
  };
```

监听` TextureBufferImpl ` 的生命周期；引用计数计算；



### 7.1 !!! returnTextureFrame

```java
  /**
   * This function is called when the texture frame is released. Only one texture frame can be in
   * flight at once, so this function must be called before a new frame is delivered.
   */
  private void returnTextureFrame() {
    handler.post(() -> {
      isTextureInUse = false;
      // 如果调用了dispose，isQuitting=true，如果 isTextureInUse = true，则不会调用release()
      // 那么这里就会relase
      // 否则 就调用tryDeliverTextureFrame
      if (isQuitting) {
        release();
      } else {
        tryDeliverTextureFrame();
      }
    });
  }
```

上一帧`TextureBufferImpl` 回收了，则就分发下一站数据。



#### 7.1.1 !!! 数据循环堆栈

```less
tryDeliverTextureFrame	// frame 使用完成就release frame
VideoFrame.release	sdk/android/api/org/webrtc/VideoFrame.java
TextureBufferImpl.release	sdk/android/api/org/webrtc/VideoFrame.java
RefCountDelegate.release 		sdk/android/src/java/org/webrtc/RefCountDelegate.java
Runnable.run
RefCountMonitor.onDestroy
returnTextureFrame		// 进入下一个获取数据 tryDeliverTextureFrame
```





## 8. !!! tryDeliverTextureFrame

```java
	private void tryDeliverTextureFrame() {
    if (handler.getLooper().getThread() != Thread.currentThread()) {
      throw new IllegalStateException("Wrong thread.");
    }
    if (isQuitting || !hasPendingTexture || isTextureInUse || listener == null) {
      return;
    }
    if (textureWidth == 0 || textureHeight == 0) {
      // Information about the resolution needs to be provided by a call to setTextureSize() before
      // frames are produced.
      Logging.w(TAG, "Texture size has not been set.");
      return;
    }
    // 设置isTextureInUse = true
    isTextureInUse = true;
    hasPendingTexture = false;

    updateTexImage();

    // 转换矩阵
    final float[] transformMatrix = new float[16];
    surfaceTexture.getTransformMatrix(transformMatrix);
    // 时间戳转换
    long timestampNs = surfaceTexture.getTimestamp();
    if (timestampAligner != null) {
      timestampNs = timestampAligner.translateTimestamp(timestampNs);
    }
    // 构建VideoFrame.TextureBuffer
    final VideoFrame.TextureBuffer buffer =
        new TextureBufferImpl(textureWidth, textureHeight, 
            TextureBuffer.Type.OES, oesTextureId,
            RendererCommon.convertMatrixToAndroidGraphicsMatrix(transformMatrix),
            handler,
						// yuv转换
            yuvConverter,
            // 监听TextureBuffer 生命周期
            textureRefCountMonitor);
    if (frameRefMonitor != null) {
      frameRefMonitor.onNewBuffer(buffer);
    }
    final VideoFrame frame = new VideoFrame(buffer, frameRotation, timestampNs);
    //！！！！！！！！！！！
    //！！！！！！！！！！！
    //！！！！！！！！！！！
    // 回调VideoSink
    //！！！！！！！！！！！
    //！！！！！！！！！！！
    //！！！！！！！！！！！
    listener.onFrame(frame);
    frame.release();
	}
```

数据构造 通知给应用层使用。这里输出的是纹理id。



```java
  private final int oesTextureId; 
```





### !!! updateTexImage

```java
  private void updateTexImage() {
    // SurfaceTexture.updateTexImage apparently can compete and deadlock with eglSwapBuffers,
    // as observed on Nexus 5. Therefore, synchronize it with the EGL functions.
    // See https://bugs.chromium.org/p/webrtc/issues/detail?id=5702 for more info.
    synchronized (EglBase.lock) {
      surfaceTexture.updateTexImage();
    }
  }

```



## ------ 状态 ------

## 9. !!! hasPendingTexture

1. 设置为true， 强制获取当前一帧数据，在准备调用 tryDeliverTextureFrame;

```java
  public void forceFrame() {
    handler.post(() -> {
      hasPendingTexture = true;
      tryDeliverTextureFrame();
    });
  }
```



2. 设置为true， SurfaceTexture 接收到数据，然后回调了`OnFrameAvailableListener`,  可以处理当前数据了， 在准备调用 tryDeliverTextureFrame;

```java
private SurfaceTextureHelper(Context sharedContext, Handler handler, boolean alignTimestamps,
      YuvConverter yuvConverter, FrameRefMonitor frameRefMonitor) {
      ...
     setOnFrameAvailableListener(surfaceTexture, (SurfaceTexture st) -> {
      if (hasPendingTexture) {
        Logging.d(TAG, "A frame is already pending, dropping frame.");
      }

      hasPendingTexture = true;
      tryDeliverTextureFrame();
    }, handler);
    ...
}
```



3. 设置为false，`OnFrameAvailableListener` 已经回调过了，但是没有注册`VideoSink`， 所以没有获取当前帧的数据；一直到 调用了   `startListening`, 发现`hasPendingTexture=true`， 则调用 `updateTexImage`  获取下一帧数据的时候，

```java
 final Runnable setListenerRunnable = new Runnable() {
    @Override
    public void run() {
      Logging.d(TAG, "Setting listener to " + pendingListener);
      listener = pendingListener;
      pendingListener = null;
      // May have a pending frame from the previous capture session - drop it.
      // hasPendingTexture 是在setOnFrameAvailableListener 为true
      if (hasPendingTexture) {
        // Calling updateTexImage() is neccessary in order to receive new frames.
        updateTexImage();
        // 获取完数据，清空状态
        hasPendingTexture = false;
      }
    }
  };
```



4. 设置为false，`updateTexImage`  获取下一帧数据的时候，

```java
private void tryDeliverTextureFrame() {
    if (handler.getLooper().getThread() != Thread.currentThread()) {
      throw new IllegalStateException("Wrong thread.");
    }
  	// isQuitting 退出了Handler的loop，就是退出线程HandlerThread
  	// hasPendingTexture，有数据可以处理，false就是没有数据，则没必要调用updateTexImage
  	// isTextureInUse，
  	// listener， VideoSink
    if (isQuitting 
        || !hasPendingTexture 
        || isTextureInUse 
        || listener == null) {
      return;
    }
    if (textureWidth == 0 || textureHeight == 0) {
      // Information about the resolution needs to be provided by a call to setTextureSize() before
      // frames are produced.
      Logging.w(TAG, "Texture size has not been set.");
      return;
    }
    isTextureInUse = true;
    hasPendingTexture = false;

    updateTexImage();
  	...
}
```




##  10. !!! isTextureInUse

```java
private volatile boolean isTextureInUse;
```



1. 设置为false，在`TextureBufferImpl` 用完释放的时候， `isTextureInUse`  设置为false，表示该帧已经结束，获取下一帧数据  ，`returnTextureFrame`；

```java
  private void returnTextureFrame() {
    handler.post(() -> {
      isTextureInUse = false;
      if (isQuitting) {
        release();
      } else {
        tryDeliverTextureFrame();
      }
    });
  }

  public boolean isTextureInUse() {
    return isTextureInUse;
  }
```



2. 设置为true，在`tryDeliverTextureFrame` 从SurfaceTexture 获取当前帧数据；

```java
  private void tryDeliverTextureFrame() {
    if (handler.getLooper().getThread() != Thread.currentThread()) {
      throw new IllegalStateException("Wrong thread.");
    }
    if (isQuitting || !hasPendingTexture || isTextureInUse || listener == null) {
      return;
    }
    if (textureWidth == 0 || textureHeight == 0) {
      // Information about the resolution needs to be provided by a call to setTextureSize() before
      // frames are produced.
      Logging.w(TAG, "Texture size has not been set.");
      return;
    }
    isTextureInUse = true;
    hasPendingTexture = false;

    updateTexImage();
    ...
	}
```



3. 释放SurfaceTextureHelper的时候， 如果请求正在获取textImage，则不释放，等到数据处理完，在`TextureBufferImpl` 用完释放的时候，进行relase；

```java
  public void dispose() {
    Logging.d(TAG, "dispose()");
    ThreadUtils.invokeAtFrontUninterruptibly(handler, () -> {
      isQuitting = true;
      if (!isTextureInUse) {
        release();
      }
    });
  }
```



```java
  private void release() {
    if (handler.getLooper().getThread() != Thread.currentThread()) {
      throw new IllegalStateException("Wrong thread.");
    }
    if (isTextureInUse || !isQuitting) {
      throw new IllegalStateException("Unexpected release.");
    }
    yuvConverter.release();
    GLES20.glDeleteTextures(1, new int[] {oesTextureId}, 0);
    surfaceTexture.release();
    eglBase.release();
    handler.getLooper().quit();
    if (timestampAligner != null) {
      timestampAligner.dispose();
    }
  }
```



##  11. isQuitting

```java
  private boolean isQuitting;
```

- 设置为true， `isQuitting=true, isTextureInUse = false`,  这时候是不会调用 `release` ，直到 `returnTextureFrame` 的时候会`release`;

```java
  public void dispose() {
    Logging.d(TAG, "dispose()");
    ThreadUtils.invokeAtFrontUninterruptibly(handler, () -> {
      isQuitting = true;
      if (!isTextureInUse) {
        release();
      }
    });
  }
```



## 12. 总结

1. 通过`startListening` 来进行数据监听，回调`VideoFrame`;
2. 在有数据(camera，decode，mediaproject等)输入到SurfaceTexture， 通过`setOnFrameAvailableListener` ；
3. 在回调里设置`hasPendingTexture`,  调用`tryDeliverTextureFrame` ，在这里开始获取texture了，通过`updateTexture`，同时设置了`isTextureInUse=true`； 
4. 获取完数据后`hasPendingTexture`，代表数据获取完成；
5. 这里就把数据回调出去`onFrame`, 等到VideoFrame 处理完成了，不在使用了，就把`isTextureInUse=true`; 还会通过`returnTextureFrame` ，调用`tryDeliverTextureFrame` 继续循环，获取数据；
6. 线程安全问题，通过`HandlerThead`来处理;

## 参考

1. [SurfaceTexture api解读](https://blog.csdn.net/lyzirving/article/details/79051437)


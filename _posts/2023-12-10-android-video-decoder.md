---
layout: post
title: android 硬件解码器
date: 2023-12-10 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android decode codec
---


* content
{:toc}

---


## 0. 概要

1. decode 是入口，通过网络接经过拆包组帧为完整帧的需要解码的数据，EncodedImage。

   把数据加入到meidecodec 的解码队列中queue；

2. 创建一个`AndroidVideoDecoder.outputThread` 线程，从`meidecodec` 获取数据，具体参考`deliverDecodeFrame`。如果是通过texuture的，则走`deliverTextureFrame`。解码后的数据从`SurfaceTexture` 通过VideoSink回调回来`OnFrame`（参考3.5）;

3. 

## 1. 继承关系

```less
VideoDecoder VideoSink
AndroidVideoDecoder  WrappedNativeVideoDecoder
											LibvpxVp8Decoder LibvpxVp9Decoder	VideoDecoderFallback
```

- VideoDecoder 接口；
- AndroidVideoDecoder，硬件解码；
- WrappedNativeVideoDecoder， 抽象类，子类 LibvpxVp8Decoder，LibvpxVp9Decoder，具体的实现是在c++层实现；
- VideoSink 是从SurfaceTextureHelper回调 `OnFrame`;



## 2. VideoDecoder

```java
public interface VideoDecoder {
  /** Settings passed to the decoder by WebRTC. */
  public class Settings {
    public final int numberOfCores;
    public final int width;
    public final int height;

    @CalledByNative("Settings")
    public Settings(int numberOfCores, int width, int height) {
      this.numberOfCores = numberOfCores;
      this.width = width;
      this.height = height;
    }
  }

  /** Additional info for decoding. */
  public class DecodeInfo {
    public final boolean isMissingFrames;
    public final long renderTimeMs;

    public DecodeInfo(boolean isMissingFrames, long renderTimeMs) {
      this.isMissingFrames = isMissingFrames;
      this.renderTimeMs = renderTimeMs;
    }
  }

  public interface Callback {
    /**
     * Call to return a decoded frame. Can be called on any thread.
     *
     * @param frame Decoded frame
     * @param decodeTimeMs Time it took to decode the frame in milliseconds or null if not available
     * @param qp QP value of the decoded frame or null if not available
     */
    void onDecodedFrame(VideoFrame frame, Integer decodeTimeMs, Integer qp);
  }

  /**
   * The decoder implementation backing this interface is either 1) a Java
   * decoder (e.g., an Android platform decoder), or alternatively 2) a native
   * decoder (e.g., a software decoder or a C++ decoder adapter).
   *
   * For case 1), createNativeVideoDecoder() should return zero.
   * In this case, we expect the native library to call the decoder through
   * JNI using the Java interface declared below.
   *
   * For case 2), createNativeVideoDecoder() should return a non-zero value.
   * In this case, we expect the native library to treat the returned value as
   * a raw pointer of type webrtc::VideoDecoder* (ownership is transferred to
   * the caller). The native library should then directly call the
   * webrtc::VideoDecoder interface without going through JNI. All calls to
   * the Java interface methods declared below should thus throw an
   * UnsupportedOperationException.
   */
  @CalledByNative
  default long createNativeVideoDecoder() {
    return 0;
  }

  /**
   * Initializes the decoding process with specified settings. Will be called on the decoding thread
   * before any decode calls.
   */
  @CalledByNative VideoCodecStatus initDecode(Settings settings, Callback decodeCallback);
  /**
   * Called when the decoder is no longer needed. Any more calls to decode will not be made.
   */
  @CalledByNative VideoCodecStatus release();
  /**
   * Request the decoder to decode a frame.
   */
  @CalledByNative VideoCodecStatus decode(EncodedImage frame, DecodeInfo info);
  /**
   * The decoder should return true if it prefers late decoding. That is, it can not decode
   * infinite number of frames before the decoded frame is consumed.
   */
  // TODO(bugs.webrtc.org/12271): Remove when downstream has been updated.
  default boolean getPrefersLateDecoding() {
    return true;
  }
  /**
   * Should return a descriptive name for the implementation. Gets called once and cached. May be
   * called from arbitrary thread.
   */
  @CalledByNative String getImplementationName();
}
```



### 2.1 VideoDecoder.Settings

```java
  /** Settings passed to the decoder by WebRTC. */
  public class Settings {
    public final int numberOfCores;
    public final int width;
    public final int height;
  }
```



### 2.2 ??? VideoDecoder.DecodeInfo

```java
  /** Additional info for decoding. */
  public class DecodeInfo {
    public final boolean isMissingFrames;
    public final long renderTimeMs;
  }
```



### 2.3 VideoDecoder.Callback

```java
  public interface Callback {
    /**
     * Call to return a decoded frame. Can be called on any thread.
     *
     * @param frame Decoded frame
     * @param decodeTimeMs Time it took to decode the frame in milliseconds or null if not available
     * @param qp QP value of the decoded frame or null if not available
     */
    void onDecodedFrame(VideoFrame frame, Integer decodeTimeMs, Integer qp);
  }
```



### 2.4 !!! createNativeVideoDecoder

这个是针对软件解码的，返回的是native对象的指针地址；对于AndroidVideoDecoder 这个接口不用实现。



### 2.5 initDecode

初始化

### 2.6 release



### 2.7 !!! decode

```
VideoCodecStatus decode(EncodedImage frame, DecodeInfo info);
```



### 2.8 ??? getPrefersLateDecoding



### 2.9 getImplementationName

codec 的名称。



## 3. AndroidVideoDecoder

sdk/android/src/java/org/webrtc/AndroidVideoDecoder.java

### 3.1 initDecode

```java
  @Override
  public VideoCodecStatus initDecode(Settings settings, Callback callback) {
    int supportHwDecodeCount = HardwareVideoDecoderFactory.getSupportHwDecodeCount();

    this.decoderThreadChecker = new ThreadChecker();

    this.callback = callback;
    if (sharedContext != null) {
      if(hwDecodeCount >= supportHwDecodeCount) {
        Logging.d(TAG, "initDecode supportHwDecodeCount=" + supportHwDecodeCount +
                "   hwDecodeCount="+hwDecodeCount);
        return VideoCodecStatus.FALLBACK_SOFTWARE;
      }
      hwDecodeCount++;
      surfaceTextureHelper = createSurfaceTextureHelper();
      surface = new Surface(surfaceTextureHelper.getSurfaceTexture());
      surfaceTextureHelper.startListening(this);
    }
    return initDecodeInternal(settings.width, settings.height);
  }
```

初始化解码器。

- 创建`SurfaceTexture`;  关于`SurfaceTextureHelper后面会分析；
- 创建`Surface`;

- 接着分析`initDecodeInternal`,  主要是创建解码器(通过`MediaCodecWrapperFactory`)，配置解码器，启动解码器；同时还创建解码线程；



#### 3.1.1 initDecodeInternal

```java
	// Internal variant is used when restarting the codec due to reconfiguration.
  private VideoCodecStatus initDecodeInternal(int width, int height) {
    decoderThreadChecker.checkIsOnValidThread();
    Logging.d(TAG,
        "initDecodeInternal name: " + codecName + " type: " + codecType + " width: " + width
            + " height: " + height);
    if (outputThread != null) {
      Logging.e(TAG, "initDecodeInternal called while the codec is already running");
      return VideoCodecStatus.FALLBACK_SOFTWARE;
    }

    // Note:  it is not necessary to initialize dimensions under the lock, since the output thread
    // is not running.
    this.width = width;
    this.height = height;

    stride = width;
    sliceHeight = height;
    hasDecodedFirstFrame = false;
    keyFrameRequired = true;

    // 创建解码器
    try {
      // MediaCodecWrapperFactory mediaCodecWrapperFactory
      // 返回 MediaCodecWrapper codec
      codec = mediaCodecWrapperFactory.createByCodecName(codecName);
    } catch (IOException | IllegalArgumentException | IllegalStateException e) {
      Logging.e(TAG, "Cannot create media decoder " + codecName);
      //创建失败，则用软件解码
      return VideoCodecStatus.FALLBACK_SOFTWARE;
    }
    try {
      MediaFormat format = MediaFormat.createVideoFormat(codecType.mimeType(), width, height);
      if (sharedContext == null) {
        format.setInteger(MediaFormat.KEY_COLOR_FORMAT, colorFormat);
      }
      // MediaCodecWrapper codec
      codec.configure(format, surface, null, 0);
      codec.start();
    } catch (IllegalStateException | IllegalArgumentException e) {
      Logging.e(TAG, "initDecode failed", e);
      release();
      return VideoCodecStatus.FALLBACK_SOFTWARE;
    }
    running = true;
    // 创建AndroidVideoDecoder.outputThread， 解码线程
    outputThread = createOutputThread();
    outputThread.start();

    Logging.d(TAG, "initDecodeInternal done");
    return VideoCodecStatus.OK;
  }
```

-  主要是创建解码器(通过`MediaCodecWrapperFactory`)，`MediaCodecWrapperFactory.createByCodecName`;
- 配置解码器；`MediaFormat.createVideoFormat`;
- 启动解码器；
- 同时还创建解码线程，并启动，内部是个while死循环；通过`running` 来判断是否继续解码；



#### 3.1.2 !!! MediaCodecWrapperFactoryImpl.createByCodecName

sdk/android/src/java/org/webrtc/MediaCodecWrapperFactoryImpl.java

```java
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
    // 内部类 MediaCodecWrapperImpl
    // android.media.MediaCodec.createByCodecName 创建MediaCodec
    return new MediaCodecWrapperImpl(MediaCodec.createByCodecName(name));
  }
}
```

创建MediaCodec， 一起对MediaCodec 的方法的封装；



#### 3.1.3 createOutputThread

```java
  private Thread createOutputThread() {
    return new Thread("AndroidVideoDecoder.outputThread") {
      @Override
      public void run() {
        outputThreadChecker = new ThreadChecker();
        while (running) {
          deliverDecodedFrame();
        }
        releaseCodecOnOutputThread();
      }
    };
  }
```

```java
  // Output thread runs a loop which polls MediaCodec for decoded output buffers.  It reformats
  // those buffers into VideoFrames and delivers them to the callback.  Variable is set on decoder
  // thread and is immutable while the codec is running.
  @Nullable private Thread outputThread;
```



##### --deliverDecodedFrame

详见 3.4。

### 3.2 release

```java
  @Override
  public VideoCodecStatus release() {
    // TODO(sakal): This is not called on the correct thread but is still called synchronously.
    // Re-enable the check once this is called on the correct thread.
    // decoderThreadChecker.checkIsOnValidThread();
    Logging.d(TAG, "release");
    VideoCodecStatus status = releaseInternal();
    if (surface != null) {
      hwDecodeCount--;
      releaseSurface();
      surface = null;
      surfaceTextureHelper.stopListening();
      surfaceTextureHelper.dispose();
      surfaceTextureHelper = null;
    }
    synchronized (renderedTextureMetadataLock) {
      renderedTextureMetadata = null;
    }
    callback = null;
    frameInfos.clear();
    return status;
  }
```

- 停止解码，`running = false`;
- 退出解码线程；`releaseInternal`
- 释放相关资源，比如surfaceTextureHelper， surface等；



#### 3.2.1 releaseInternal

```java
  // Internal variant is used when restarting the codec due to reconfiguration.
  private VideoCodecStatus releaseInternal() {
    if (!running) {
      Logging.d(TAG, "release: Decoder is not running.");
      return VideoCodecStatus.OK;
    }
    try {
      // The outputThread actually stops and releases the codec once running is false.
      running = false;
      // 退出 AndroidVideoDecoder.outputThread
      // MEDIA_CODEC_RELEASE_TIMEOUT_MS = 5 000
      if (!ThreadUtils.joinUninterruptibly(outputThread, MEDIA_CODEC_RELEASE_TIMEOUT_MS)) {
        // Log an exception to capture the stack trace and turn it into a TIMEOUT error.
        Logging.e(TAG, "Media decoder release timeout", new RuntimeException());
        return VideoCodecStatus.TIMEOUT;
      }
      if (shutdownException != null) {
        // Log the exception and turn it into an error.  Wrap the exception in a new exception to
        // capture both the output thread's stack trace and this thread's stack trace.
        Logging.e(TAG, "Media decoder release error", new RuntimeException(shutdownException));
        shutdownException = null;
        return VideoCodecStatus.ERROR;
      }
    } finally {
      codec = null;
      outputThread = null;
    }
    return VideoCodecStatus.OK;
  }
```

- 停止解码，`running = false`;
- 退出解码线程；



#### 3.2.2 releaseSurface

```java
  protected void releaseSurface() {
    surface.release();
  }
```



### 3.3 !!! decode

```java
 @Override
  public VideoCodecStatus decode(EncodedImage frame, DecodeInfo info) {
    decoderThreadChecker.checkIsOnValidThread();
    // MediaCodecWrapper codec;
    if (codec == null || callback == null) {
      Logging.d(TAG, "decode uninitalized, codec: " + (codec != null) + ", callback: " + callback);
      return VideoCodecStatus.UNINITIALIZED;
    }

    if (frame.buffer == null) {
      Logging.e(TAG, "decode() - no input data");
      return VideoCodecStatus.ERR_PARAMETER;
    }

    int size = frame.buffer.remaining();
    if (size == 0) {
      Logging.e(TAG, "decode() - input buffer empty");
      return VideoCodecStatus.ERR_PARAMETER;
    }

    // Load dimensions from shared memory under the dimension lock.
    final int width;
    final int height;
    synchronized (dimensionLock) {
      width = this.width;
      height = this.height;
    }

    // Check if the resolution changed and reset the codec if necessary.
    // 分辨率变化，重新创建decode
    if (frame.encodedWidth * frame.encodedHeight > 0
        && (frame.encodedWidth != width || frame.encodedHeight != height)) {
      VideoCodecStatus status = reinitDecode(frame.encodedWidth, frame.encodedHeight);
      if (status != VideoCodecStatus.OK) {
        return status;
      }
    }

    // 如果需要关键帧，不是关键帧，则丢弃
    if (keyFrameRequired) {
      // Need to process a key frame first.
      if (frame.frameType != EncodedImage.FrameType.VideoFrameKey) {
        Logging.e(TAG, "decode() - key frame required first");
        return VideoCodecStatus.NO_OUTPUT;
      }
    }

    int index;
    try {
      // 获取可用的输入缓冲区的索引
      // private static final int DEQUEUE_INPUT_TIMEOUT_US = 500 000; 500ms
      index = codec.dequeueInputBuffer(DEQUEUE_INPUT_TIMEOUT_US);
    } catch (IllegalStateException e) {
      Logging.e(TAG, "dequeueInputBuffer failed", e);
      return VideoCodecStatus.ERROR;
    }
    // 获取输入缓冲区失败
    if (index < 0) {
      // Decoder is falling behind.  No input buffers available.
      // The decoder can't simply drop frames; it might lose a key frame.
      Logging.e(TAG, "decode() - no HW buffers available; decoder falling behind");
      return VideoCodecStatus.ERROR;
    }

    // 获取ByteBuffer， 
    ByteBuffer buffer;
    try {
      buffer = codec.getInputBuffers()[index];
    } catch (IllegalStateException e) {
      Logging.e(TAG, "getInputBuffers failed", e);
      return VideoCodecStatus.ERROR;
    }

    // buffer大小校验
    if (buffer.capacity() < size) {
      Logging.e(TAG, "decode() - HW buffer too small");
      return VideoCodecStatus.ERROR;
    }
    // 把需要解码数据放入到ByteBuffer
    buffer.put(frame.buffer);

    // ???
    frameInfos.offer(new FrameInfo(SystemClock.elapsedRealtime(), frame.rotation));
    try {
      // 将填满数据的inputBuffer提交到编码队列
      codec.queueInputBuffer(index, 0 /* offset */, size,
          TimeUnit.NANOSECONDS.toMicros(frame.captureTimeNs), 0 /* flags */);
    } catch (IllegalStateException e) {
      Logging.e(TAG, "queueInputBuffer failed", e);
      frameInfos.pollLast();
      return VideoCodecStatus.ERROR;
    }
    if (keyFrameRequired) {
      keyFrameRequired = false;
    }
    return VideoCodecStatus.OK;
  }
```

- 获取可用的输入缓冲区的索引 public int dequeueInputBuffer (long timeoutUs) 
- 获取输入缓冲区 public ByteBuffer getInputBuffer(int index) 
- 将填满数据的inputBuffer提交到编码队列 public final void queueInputBuffer(int index,int offset, int size, long presentationTimeUs, int flags) 

#### 3.3.0 EncodedImage

sdk/android/api/org/webrtc/EncodedImage.java

```java

public class EncodedImage implements RefCounted {
  // Must be kept in sync with common_types.h FrameType.
  public enum FrameType {
    EmptyFrame(0),
    VideoFrameKey(3),
    VideoFrameDelta(4);

    private final int nativeIndex;

    private FrameType(int nativeIndex) {
      this.nativeIndex = nativeIndex;
    }

    public int getNative() {
      return nativeIndex;
    }

    @CalledByNative("FrameType")
    static FrameType fromNativeIndex(int nativeIndex) {
      for (FrameType type : FrameType.values()) {
        if (type.getNative() == nativeIndex) {
          return type;
        }
      }
      throw new IllegalArgumentException("Unknown native frame type: " + nativeIndex);
    }
  }

  private final RefCountDelegate refCountDelegate;
  public final ByteBuffer buffer;
  public final int encodedWidth;
  public final int encodedHeight;
  public final long captureTimeMs; // Deprecated
  public final long captureTimeNs;
  public final FrameType frameType;
  public final int rotation;
  public final @Nullable Integer qp;
  ...
}
```



#### 3.3.1 reinitDecode

```java
  private VideoCodecStatus reinitDecode(int newWidth, int newHeight) {
    decoderThreadChecker.checkIsOnValidThread();
    // 先释放
    VideoCodecStatus status = releaseInternal();
    if (status != VideoCodecStatus.OK) {
      return status;
    }
    // 重新创建
    return initDecodeInternal(newWidth, newHeight);
  }
```

重启解码器。



### 3.4 deliverDecodedFrame

```java
protected void deliverDecodedFrame() {
    outputThreadChecker.checkIsOnValidThread();
    try {
      MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
      // Block until an output buffer is available (up to 100 milliseconds).  If the timeout is
      // exceeded, deliverDecodedFrame() will be called again on the next iteration of the output
      // thread's loop.  Blocking here prevents the output thread from busy-waiting while the codec
      // is idle.
      // 获取已成功编解码的输出缓冲区的索引
      // DEQUEUE_OUTPUT_BUFFER_TIMEOUT_US = 100 000， 100ms
      int result = codec.dequeueOutputBuffer(info, DEQUEUE_OUTPUT_BUFFER_TIMEOUT_US);
      if (result == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
        reformat(codec.getOutputFormat());
        return;
      }

      if (result < 0) {
        Logging.v(TAG, "dequeueOutputBuffer returned " + result);
        return;
      }

      FrameInfo frameInfo = frameInfos.poll();
      Integer decodeTimeMs = null;
      int rotation = 0;
      if (frameInfo != null) {
        // 注意这里的dts
        decodeTimeMs = (int) (SystemClock.elapsedRealtime() - frameInfo.decodeStartTimeMs);
        rotation = frameInfo.rotation;
      }

      hasDecodedFirstFrame = true;

      if (surfaceTextureHelper != null) {
        deliverTextureFrame(result, info, rotation, decodeTimeMs);
      } else {
        deliverByteFrame(result, info, rotation, decodeTimeMs);
      }

    } catch (IllegalStateException e) {
      Logging.e(TAG, "deliverDecodedFrame failed", e);
    }
  }
```

- 获取已成功编解码的输出缓冲区的索引 public final int dequeueOutputBuffer(BufferInfo info, long timeoutUs) 
- 获取输出缓冲区 public ByteBuffer getOutputBuffer(int index) 
- 释放输出缓冲区 public final void releaseOutputBuffer(int index, boolean render)



#### 3.4.1 !!! deliverTextureFrame

这里走的是texture模式。这里的数据会异步SurfaceTextureHelper 回调回来，到`onFrame`；具体参考 3.5





#### 3.4.2 !!! deliverByteFrame

```java
private void deliverByteFrame(
      int result, MediaCodec.BufferInfo info, int rotation, Integer decodeTimeMs) {
    // Load dimensions from shared memory under the dimension lock.
    int width;
    int height;
    int stride;
    int sliceHeight;
    synchronized (dimensionLock) {
      width = this.width;
      height = this.height;
      stride = this.stride;
      sliceHeight = this.sliceHeight;
    }

    // Output must be at least width * height bytes for Y channel, plus (width / 2) * (height / 2)
    // bytes for each of the U and V channels.
    if (info.size < width * height * 3 / 2) {
      Logging.e(TAG, "Insufficient output buffer size: " + info.size);
      return;
    }

    if (info.size < stride * height * 3 / 2 && sliceHeight == height && stride > width) {
      // Some codecs (Exynos) report an incorrect stride.  Correct it here.
      // Expected size == stride * height * 3 / 2.  A bit of algebra gives the correct stride as
      // 2 * size / (3 * height).
      stride = info.size * 2 / (height * 3);
    }

  	// 获取输出缓冲区
    ByteBuffer buffer = codec.getOutputBuffers()[result];
    buffer.position(info.offset);
    buffer.limit(info.offset + info.size);
    buffer = buffer.slice();

    final VideoFrame.Buffer frameBuffer;
    if (colorFormat == CodecCapabilities.COLOR_FormatYUV420Planar) {
      frameBuffer = copyI420Buffer(buffer, stride, sliceHeight, width, height);
    } else {
      // All other supported color formats are NV12.
      frameBuffer = copyNV12ToI420Buffer(buffer, stride, sliceHeight, width, height);
    }
  	// 释放输出缓冲区 
    codec.releaseOutputBuffer(result, /* render= */ false);

    long presentationTimeNs = info.presentationTimeUs * 1000;
    VideoFrame frame = new VideoFrame(frameBuffer, rotation, presentationTimeNs);

    // Note that qp is parsed on the C++ side.
    callback.onDecodedFrame(frame, decodeTimeMs, null /* qp */);
    frame.release();
  }
```



### 3.5 !!! deliverTextureFrame

```java
// decodeTimeMs, 是“在decode赋值给FrameInfo的，获取当前本地时间戳” 
// 与 “在deliverDecodedFrame解码结束获取当前本地时间戳” 之差
private void deliverTextureFrame(final int index, final MediaCodec.BufferInfo info,
      final int rotation, final Integer decodeTimeMs) {
    // Load dimensions from shared memory under the dimension lock.
    final int width;
    final int height;
    synchronized (dimensionLock) {
      width = this.width;
      height = this.height;
    }

    synchronized (renderedTextureMetadataLock) {
      if (renderedTextureMetadata != null) {
        codec.releaseOutputBuffer(index, false);
        return; // We are still waiting for texture for the previous frame, drop this one.
      }
      surfaceTextureHelper.setTextureSize(width, height);
      surfaceTextureHelper.setFrameRotation(rotation);
      renderedTextureMetadata = new DecodedTextureMetadata(info.presentationTimeUs, decodeTimeMs);
      // 释放输出缓冲区 
      codec.releaseOutputBuffer(index, /* render= */ true);
    }
  }
```

这里走的是texture模式。这里的数据会异步SurfaceTextureHelper 回调回来，到`onFrame`；



#### 3.5.1 onFrame

```java
  @Override
  public void onFrame(VideoFrame frame) {
    final VideoFrame newFrame;
    final Integer decodeTimeMs;
    final long timestampNs;
    synchronized (renderedTextureMetadataLock) {
      if (renderedTextureMetadata == null) {
        throw new IllegalStateException(
            "Rendered texture metadata was null in onTextureFrameAvailable.");
      }
      timestampNs = renderedTextureMetadata.presentationTimestampUs * 1000;
      decodeTimeMs = renderedTextureMetadata.decodeTimeMs;
      renderedTextureMetadata = null;
    }
    // Change timestamp of frame.
    final VideoFrame frameWithModifiedTimeStamp =
        new VideoFrame(frame.getBuffer(), frame.getRotation(), timestampNs);
    // VideoDecoder.Callback
    callback.onDecodedFrame(frameWithModifiedTimeStamp, decodeTimeMs, null /* qp */);
  }
```

这里的数据从SurfaceTextureHelper 回调来的。
在initDecode的时候注册了监听`surfaceTextureHelper.startListening(this)`。



#### 3.5.2 VideoDecoder.Callback.onDecodedFrame

在`initDecode` 设置了Callback。



### 3.6 关键帧

```java
  // Whether the decoder has seen a key frame.  The first frame must be a key frame.  Only accessed
  // on the decoder thread.
  private boolean keyFrameRequired;// 默认是true
```

第一帧进行解码必须是关键帧，非关键帧丢弃。

```java
public VideoCodecStatus decode(EncodedImage frame, DecodeInfo info) {
...
    if (keyFrameRequired) {
      // Need to process a key frame first.
      if (frame.frameType != EncodedImage.FrameType.VideoFrameKey) {
        Logging.e(TAG, "decode() - key frame required first");
        return VideoCodecStatus.NO_OUTPUT;
      }
    }
...
  
    if (keyFrameRequired) {
      keyFrameRequired = false;
    }
...
}
```



### 3.7 BlockingDeque\<FrameInfo\> frameInfos;

```java
  private static class FrameInfo {
    final long decodeStartTimeMs; // 开始解码的时间
    final int rotation; // 角度

    FrameInfo(long decodeStartTimeMs, int rotation) {
      this.decodeStartTimeMs = decodeStartTimeMs;
      this.rotation = rotation;
    }
  }
```



- 这里进行赋值

```java
 public VideoCodecStatus decode(EncodedImage frame, DecodeInfo info) {
  ...
   frameInfos.offer(new FrameInfo(SystemClock.elapsedRealtime(), frame.rotation));
    try {
      codec.queueInputBuffer(index, 0 /* offset */, size,
          TimeUnit.NANOSECONDS.toMicros(frame.captureTimeNs), 0 /* flags */);
    } catch (IllegalStateException e) {
      Logging.e(TAG, "queueInputBuffer failed", e);
      frameInfos.pollLast();
      return VideoCodecStatus.ERROR;
    }
    ...
}
```

- 这里获取

```java
protected void deliverDecodedFrame() {
	...
      FrameInfo frameInfo = frameInfos.poll();
      Integer decodeTimeMs = null;
      int rotation = 0;
      if (frameInfo != null) {
        decodeTimeMs = (int) (SystemClock.elapsedRealtime() - frameInfo.decodeStartTimeMs);
        rotation = frameInfo.rotation;
      }
      ...
    	if (surfaceTextureHelper != null) {
        deliverTextureFrame(result, info, rotation, decodeTimeMs);
      } else {
        deliverByteFrame(result, info, rotation, decodeTimeMs);
      }
  		...
}
```



### 3.8 !!! DecodedTextureMetadata

```java
  private static class DecodedTextureMetadata {
    final long presentationTimestampUs; // 设置解码器给的时间，具体参考deliverTextureFrame
    final Integer decodeTimeMs; // 解码所用时长， 具体参考 deliverDecodedFrame

    DecodedTextureMetadata(long presentationTimestampUs, Integer decodeTimeMs) {
      this.presentationTimestampUs = presentationTimestampUs;
      this.decodeTimeMs = decodeTimeMs;
    }
  }
```



## 4. WrappedNativeVideoDecoder

sdk/android/api/org/webrtc/WrappedNativeVideoDecoder.java

```java
public abstract class WrappedNativeVideoDecoder implements VideoDecoder {
  @Override public abstract long createNativeVideoDecoder();

  @Override
  public final VideoCodecStatus initDecode(Settings settings, Callback decodeCallback) {
    throw new UnsupportedOperationException("Not implemented.");
  }

  @Override
  public final VideoCodecStatus release() {
    throw new UnsupportedOperationException("Not implemented.");
  }

  @Override
  public final VideoCodecStatus decode(EncodedImage frame, DecodeInfo info) {
    throw new UnsupportedOperationException("Not implemented.");
  }

  @Override
  public final String getImplementationName() {
    throw new UnsupportedOperationException("Not implemented.");
  }
}
```

抽象类，主要是给非AndroidVideoDecoder的使用。主要是软件解码，例如`LibvpxVp8Decoder`, `LibvpxVp9Decoder`。



## 5. LibvpxVp8Decoder

sdk/android/api/org/webrtc/LibvpxVp8Decoder.java

```java
public class LibvpxVp8Decoder extends WrappedNativeVideoDecoder {
  @Override
  public long createNativeVideoDecoder() {
    return nativeCreateDecoder();
  }

  static native long nativeCreateDecoder();
}
```

Vp8 软件解码，在c++层实现。



## 6. LibvpxVp9Decoder

sdk/android/api/org/webrtc/LibvpxVp9Decoder.java

```java
public class LibvpxVp9Decoder extends WrappedNativeVideoDecoder {
  @Override
  public long createNativeVideoDecoder() {
    return nativeCreateDecoder();
  }

  static native long nativeCreateDecoder();

  static native boolean nativeIsSupported();
}
```

Vp9 软件解码，在c++层实现。



## 7. !!! VideoDecoderFallback

sdk/android/api/org/webrtc/VideoDecoderFallback.java

主要备用的，在主要的编码方案失败了，则启动备用；主要是AndroidVideoDecoder编码失败，备用会启动软件解码，就会调用到VideoDecoderFallback。

```java
/**
 * A combined video decoder that falls back on a secondary decoder if the primary decoder fails.
 */
public class VideoDecoderFallback extends WrappedNativeVideoDecoder {
  private final VideoDecoder fallback;
  private final VideoDecoder primary;

  public VideoDecoderFallback(VideoDecoder fallback, VideoDecoder primary) {
    this.fallback = fallback;
    this.primary = primary;
  }

  @Override
  public long createNativeVideoDecoder() {
    return nativeCreateDecoder(fallback, primary);
  }

  private static native long nativeCreateDecoder(VideoDecoder fallback, VideoDecoder primary);
}
```


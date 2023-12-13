---
layout: post
title: SurfaceTexture
date: 2023-12-05 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android surface
---


* content
{:toc}

---

## 1. 什么是SurfaceTexture

Android 3.0（API 11）新加入的一个类，**不同于 SurfaceView 会将图像显示在屏幕上，SurfaceTexture 对图像流的处理并不直接显示，而是转为 GL 外部纹理。**

- SurfaceTexture 可以从图像流（相机、视频）中捕获帧数据用作 OpenGL ES 外部纹理（GL_TEXTURE_EXTERNAL_OES），实现无缝连接。
- 应用层可以很方便的对这个外部纹理进行二次处理（如添加滤镜等）。
- **输出目标是 Camera 或 MediaPlayer 时，可以用 SurfaceTexture 代替 SurfaceHolder，这样图像流将把所有帧传给 SurfaceTexture 而不是显示在设备上。**
- 使用 updateTexImage() 方法更新最新的图像。



## 2. SurfaceTexture的作用

把从Camera，硬件解码，MediaProject的数据 输入到SurfaceTexture，通过OpengGL把数据输出到纹理上；最终输出的是纹理，可以对纹理做二次处理，比如美颜，修图, 水印等。



## 3. SurfaceView, SurfaceTexture区别

不同于 SurfaceView 会将图像显示在屏幕上，SurfaceTexture 对图像流的处理并不直接显示，而是转为 GL 外部纹理。



## 4. 使用场景

### 4.1 硬件解码

sdk/android/src/java/org/webrtc/AndroidVideoDecoder.java

```java
  @Override
  public VideoCodecStatus initDecode(Settings settings, Callback callback) {
    this.decoderThreadChecker = new ThreadChecker();

    this.callback = callback;
    if (sharedContext != null) {
      // 创建SurfaceTextureHelper，内部封装了SurfaceTexture，包括生命周期，以及相关的api
      // @Nullable private SurfaceTextureHelper surfaceTextureHelper;
      surfaceTextureHelper = createSurfaceTextureHelper();
      // 得到SurfaceTexture，创建Surface
      // @Nullable private Surface surface;
      surface = new Surface(surfaceTextureHelper.getSurfaceTexture());
      surfaceTextureHelper.startListening(this);
    }
    return initDecodeInternal(settings.width, settings.height);
  }
```



#### 4.1.1 createSurfaceTextureHelper

```java
protected SurfaceTextureHelper createSurfaceTextureHelper() {
  	// EglBase.Context sharedContext;
    return SurfaceTextureHelper.create("decoder-texture-thread", sharedContext);
  }
```



#### 4.1.2 initDecodeInternal

```java
private VideoCodecStatus initDecodeInternal(int width, int height) {
   ...

    try {
      codec = mediaCodecWrapperFactory.createByCodecName(codecName);
    } catch (IOException | IllegalArgumentException | IllegalStateException e) {
      Logging.e(TAG, "Cannot create media decoder " + codecName);
      return VideoCodecStatus.FALLBACK_SOFTWARE;
    }
    try {
      MediaFormat format = MediaFormat.createVideoFormat(codecType.mimeType(), width, height);
      if (sharedContext == null) {
        format.setInteger(MediaFormat.KEY_COLOR_FORMAT, colorFormat);
      }
      
      // 传入surface， 这样把解码后的数据会绘制到该Surface，
      // 从而把数据就导入SurfaceTexture，从而导出纹理
      //!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!! 
      //!!!!!!!!!!!!!!!!!
      codec.configure(format, surface, null, 0);
      //!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!
      //!!!!!!!!!!!!!!!!!
      codec.start();
    } catch (IllegalStateException | IllegalArgumentException e) {
     ...
    }
  	...
    return VideoCodecStatus.OK;
  }
```





### 4.2 Camera 采集

1. Camera1 用texture的采集方式；
   `startCapture`的时候，通过`Camera.setPreviewTexture`,  传入SurfaceTexture；
   `listenForTextureFrames` ，`SurfaceTexture.setOnFrameAvailableListener`, 回调`OnFrameAvailable`。

2. Camera2
   `startCapture`的时候，通过`Camera2Session.openCamera()`, `  通过 android.hardware.camera2.CameraManager.openCamera(cameraId, new CameraStateCallback(), cameraThreadHandler);`，
   这里会回调`Camera2Session.CameraStateCallback.onOpened` ;
   接着调用 `Camera.createCaptureSession`,  会回调`CaptureSessionCallback.onConfigured`, 
   `SurfaceTexture.setOnFrameAvailableListener`, 回调`OnFrameAvailable`。

   

#### 4.2.1 继承关系

```less
CameraVideoCapturer
CameraCapturer // 抽象类
Camera1Capturer		Camera2Capturer
```



#### 4.2.2 堆栈

#### 

```less
CameraCapturer.startCapture
CameraCapturer.createSessionInternal
Camera1Capturer.createCameraSession				
Camera1Session.create											
Camera1Session.Camera1Session							
Camera1Session.startCapturing							
Camera1Session.listenForTextureFrames																					
```



```less
CameraCapturer.startCapture
CameraCapturer.createSessionInternal
Camera2Capturer.createCameraSession
Camera2Session.create
Camera2Session.Camera2Session
Camera2Session.start
Camera2Session.openCamera
Camera2Session.CameraStateCallback
Camera2Session.CameraStateCallback.onOpened
Camera2Session.CaptureSessionCallback.onConfigured
```


#### 4.2.3 Camera1Capturer.createCameraSession

sdk/android/api/org/webrtc/Camera1Capturer.java

```java
  protected void createCameraSession(CameraSession.CreateSessionCallback createSessionCallback,
      CameraSession.Events events, Context applicationContext,
			//
			// SurfaceTextureHelper surfaceTextureHelper
			// 
      SurfaceTextureHelper surfaceTextureHelper, String cameraName, int width, int height,
      int framerate) {
    try {
      Camera1Session.create(createSessionCallback, events, captureToTexture, applicationContext,
          surfaceTextureHelper, Camera1Enumerator.getCameraIndex(cameraName), width, height,
          framerate);
    } catch (RuntimeException e) {
      createSessionCallback.onFailure(CameraSession.FailureType.ERROR, e.getMessage());
    }
  }
```





##### !!! Camera1Session.create

sdk/android/src/java/org/webrtc/Camera1Session.java

Camera1 的 texture 模式。

```java
 // TODO(titovartem) make correct fix during webrtc:9175
  @SuppressWarnings("ByteBufferBackingArray")
  public static void create(final CreateSessionCallback callback, final Events events,
      final boolean captureToTexture, final Context applicationContext,
      final SurfaceTextureHelper surfaceTextureHelper, final int cameraId, final int width,
      final int height, final int framerate) {
		...
      camera = android.hardware.Camera.open(cameraId);
    ...

    final android.hardware.Camera.CameraInfo info;
    final CaptureFormat captureFormat;
    try {

      try {
        //!!!!!!!!!!
        //!!!!!!!!!!
        //!!!!!!!!!!
        camera.setPreviewTexture(surfaceTextureHelper.getSurfaceTexture());
        //!!!!!!!!!!
        //!!!!!!!!!!
        //!!!!!!!!!!
      } catch (IOException | RuntimeException e) {
        camera.release();
        callback.onFailure(FailureType.ERROR, e.getMessage());
        return;
      }
			...

      // 非texture模式，byte流
      if (!captureToTexture) {
        final int frameSize = captureFormat.frameSize();
        for (int i = 0; i < NUMBER_OF_CAPTURE_BUFFERS; ++i) {
          final ByteBuffer buffer = ByteBuffer.allocateDirect(frameSize);
          camera.addCallbackBuffer(buffer.array());
        }
      }
			...
    } catch (RuntimeException e) {
      camera.release();
      callback.onFailure(FailureType.ERROR, e.getMessage());
      return;
    }
		...
  }
```



##### !!! Camera1Session.listenForTextureFrames

sdk/android/src/java/org/webrtc/Camera1Session.java

```java
 private void listenForTextureFrames() {
    surfaceTextureHelper.startListening((VideoFrame frame) -> {
      ...

      // Undo the mirror that the OS "helps" us with.
      // http://developer.android.com/reference/android/hardware/Camera.html#setDisplayOrientation(int)
      final VideoFrame modifiedFrame = new VideoFrame(
          CameraSession.createTextureBufferWithModifiedTransformMatrix(
              (TextureBufferImpl) frame.getBuffer(),
              /* mirror= */ info.facing == android.hardware.Camera.CameraInfo.CAMERA_FACING_FRONT,
              /* rotation= */ 0),
          /* rotation= */ getFrameOrientation(), frame.getTimestampNs());
      events.onFrameCaptured(Camera1Session.this, modifiedFrame);
      modifiedFrame.release();
    });
  }
```



###### SurfaceTextureHelper.tryDeliverTextureFrame

从这里回调了`TextureBufferImpl`。

sdk/android/api/org/webrtc/SurfaceTextureHelper.java

```java
private void tryDeliverTextureFrame() {
  ...
    final float[] transformMatrix = new float[16];
    surfaceTexture.getTransformMatrix(transformMatrix);
    long timestampNs = surfaceTexture.getTimestamp();
    if (timestampAligner != null) {
      timestampNs = timestampAligner.translateTimestamp(timestampNs);
    }
    final VideoFrame.TextureBuffer buffer =
        new TextureBufferImpl(textureWidth, textureHeight, TextureBuffer.Type.OES, oesTextureId,
            RendererCommon.convertMatrixToAndroidGraphicsMatrix(transformMatrix), handler,
            yuvConverter, textureRefCountMonitor);
    if (frameRefMonitor != null) {
      frameRefMonitor.onNewBuffer(buffer);
    }
    final VideoFrame frame = new VideoFrame(buffer, frameRotation, timestampNs);
    listener.onFrame(frame);
    frame.release();
  }
```





#### 4.2.4 Camera2Capturer.createCameraSession

```java
  protected void createCameraSession(CameraSession.CreateSessionCallback createSessionCallback,
      CameraSession.Events events, Context applicationContext,
		  //
			// SurfaceTextureHelper surfaceTextureHelper
			// 
      SurfaceTextureHelper surfaceTextureHelper, String cameraName, int width, int height,
      int framerate) {
    Camera2Session.create(createSessionCallback, events, applicationContext, cameraManager,
        surfaceTextureHelper, cameraName, width, height, framerate);
  }
```



#####  Camera2Session.openCamera

sdk/android/src/java/org/webrtc/Camera2Session.java

```java
  private void openCamera() {
    checkIsOnCameraThread();

    Logging.d(TAG, "Opening camera " + cameraId);
    events.onCameraOpening();

    try {
      cameraManager.openCamera(cameraId, new CameraStateCallback(), cameraThreadHandler);
    } catch (CameraAccessException e) {
      reportError("Failed to open camera: " + e);
      return;
    }
  }
```



##### !!! Camera2Session.CameraStateCallback.onOpened

sdk/android/src/java/org/webrtc/Camera2Session.java

```java
private class CameraStateCallback extends CameraDevice.StateCallback {
    ...

    @Override
    public void onOpened(CameraDevice camera) {
      checkIsOnCameraThread();

      Logging.d(TAG, "Camera opened.");
      cameraDevice = camera;
			//!!!!!!!!!!!!!!!!!!!!!!!!
			//!!!!!!!!!!!!!!!!!!!!!!!!
			//!!!!!!!!!!!!!!!!!!!!!!!!
      surfaceTextureHelper.setTextureSize(captureFormat.width, captureFormat.height);
      surface = new Surface(surfaceTextureHelper.getSurfaceTexture());
			...
        camera.createCaptureSession(
            Arrays.asList(surface), new CaptureSessionCallback(), cameraThreadHandler);
    	...
			//!!!!!!!!!!!!!!!!!!!!!!!!
			//!!!!!!!!!!!!!!!!!!!!!!!!
			//!!!!!!!!!!!!!!!!!!!!!!!!
    }
		...
  }
```



##### !!! Camera2Session.CaptureSessionCallback.onConfigured

sdk/android/src/java/org/webrtc/Camera2Session.java

```java
private class CaptureSessionCallback extends CameraCaptureSession.StateCallback {
...

    @Override
    public void onConfigured(CameraCaptureSession session) {
      ...

      surfaceTextureHelper.startListening((VideoFrame frame) -> {
        checkIsOnCameraThread();

    	...

        // Undo the mirror that the OS "helps" us with.
        // http://developer.android.com/reference/android/hardware/Camera.html#setDisplayOrientation(int)
        // Also, undo camera orientation, we report it as rotation instead.
        final VideoFrame modifiedFrame =
            new VideoFrame(CameraSession.createTextureBufferWithModifiedTransformMatrix(
                               (TextureBufferImpl) frame.getBuffer(),
                               /* mirror= */ isCameraFrontFacing,
                               /* rotation= */ -cameraOrientation),
                /* rotation= */ getFrameOrientation(), frame.getTimestampNs());
        events.onFrameCaptured(Camera2Session.this, modifiedFrame);
        modifiedFrame.release();
      });
      Logging.d(TAG, "Camera device successfully started.");
      callback.onDone(Camera2Session.this);
    }
    ...
 }
```



#### 4.2.5 ??? createTextureBufferWithModifiedTransformMatrix

sdk/android/src/java/org/webrtc/CameraSession.java

```java
  static VideoFrame.TextureBuffer createTextureBufferWithModifiedTransformMatrix(
      TextureBufferImpl buffer, boolean mirror, int rotation) {
    final Matrix transformMatrix = new Matrix();
    // Perform mirror and rotation around (0.5, 0.5) since that is the center of the texture.
    transformMatrix.preTranslate(/* dx= */ 0.5f, /* dy= */ 0.5f);
    if (mirror) {
      transformMatrix.preScale(/* sx= */ -1f, /* sy= */ 1f);
    }
    transformMatrix.preRotate(rotation);
    transformMatrix.preTranslate(/* dx= */ -0.5f, /* dy= */ -0.5f);

    // The width and height are not affected by rotation since Camera2Session has set them to the
    // value they should be after undoing the rotation.
    return buffer.applyTransformMatrix(transformMatrix, buffer.getWidth(), buffer.getHeight());
  }
```



##### TextureBufferImpl.applyTransformMatrix

sdk/android/api/org/webrtc/TextureBufferImpl.java

```java
private TextureBufferImpl applyTransformMatrix(Matrix transformMatrix, int unscaledWidth,
      int unscaledHeight, int scaledWidth, int scaledHeight) {
    final Matrix newMatrix = new Matrix(this.transformMatrix);
    newMatrix.preConcat(transformMatrix);
    retain();
    return new TextureBufferImpl(unscaledWidth, unscaledHeight, scaledWidth, scaledHeight, type, id,
        newMatrix, toI420Handler, yuvConverter, new RefCountMonitor() {
          @Override
          public void onRetain(TextureBufferImpl textureBuffer) {
            refCountMonitor.onRetain(TextureBufferImpl.this);
          }

          @Override
          public void onRelease(TextureBufferImpl textureBuffer) {
            refCountMonitor.onRelease(TextureBufferImpl.this);
          }

          @Override
          public void onDestroy(TextureBufferImpl textureBuffer) {
            release();
          }
        });
  }
```








---
layout: post
title: SurfaceView, GLSurfaceView, SurfaceTexture, TextureView
date: 2023-12-04 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android surface
---


* content
{:toc}

---

## 1. SurfaceView = Surface + View

[SurfaceView 和 GLSurfaceView](https://source.android.google.cn/docs/core/graphics/arch-sv-glsv?hl=zh-cn)。SurfaceView 结合了 Surface 和 View。SurfaceView 的 View 组件由 SurfaceFlinger（而不是应用）合成，从而可以通过单独的线程/进程渲染，并与应用界面渲染隔离。



## 2. GLSurfaceView = SurfaceView+OpenGL

[SurfaceView 和 GLSurfaceView](https://source.android.google.cn/docs/core/graphics/arch-sv-glsv?hl=zh-cn)。GLSurfaceView 提供了用于管理 EGL 上下文、线程间通信以及与 activity 生命周期的交互的帮助程序类（但不是必须使用 GLES）。



## 3. SurfaceTexture

[SurfaceTexture](https://source.android.google.cn/docs/core/graphics/arch-st?hl=zh-cn)。 SurfaceTexture 将 Surface 和 GLES 纹理相结合来创建 BufferQueue，而您的应用是 BufferQueue 的消费者。当生产者将新的缓冲区排入队列时，它会通知您的应用。您的应用会依次释放先前占用的缓冲区，从队列中获取新缓冲区并执行 EGL 调用，从而使 GLES 可将此缓冲区作为外部纹理使用。Android 7.0 添加了对安全纹理视频播放的支持，以便对受保护的视频内容进行 GPU 后处理。

SurfaceTexture是离屏渲染和TextureView的核心，内部包含了一个BufferQueue，可以把Surface生成的图像流，转换为纹理，供业务方进一步加工使用。

把SurfaceTexture设置给Camera接收摄像头图像数据，并转换为OES纹理，然后可以利用OpenGL对OES纹理做进一步特效处理，最后上屏或者录制成视频。

说白了，

- 首先，通过Canvas、OpenGL、Camera或者Video Decoder生成图像流。
- 接着，图像流通过Surface入队到BufferQueue，并通知到GLConsumer。创建 BufferQueue， 是生产者；
- 然后，GLConsumer从BufferQueue获取图像流GraphicBuffer，并转换为纹理。
- 最后，应用层可以对纹理进一步处理，例如：上特效或者上屏。应用层就是BufferQueue 的消费者；

![surfacetexture]({{ site.url }}{{ site.baseurl }}/images/android-surface-xx.assets/surfacetexture.png)

参考：[Android图形系统之SurfaceTexture](https://juejin.cn/post/6844904161645953038)
[谈一谈Android上的SurfaceTexture](https://juejin.cn/post/6854573221036392461)



## 4. TextureView = View + SurfaceTexture

[TextureView](https://source.android.google.cn/docs/core/graphics/arch-tv?hl=zh-cn)。 TextureView 结合了 View 和 SurfaceTexture。TextureView 对 SurfaceTexture 进行包装，并负责响应回调以及获取新的缓冲区。在绘图时，TextureView 使用最近收到的缓冲区的内容作为其数据源，根据 View 状态指示，在它应该渲染的任何位置和以它应该采用的任何渲染方式进行渲染。View 合成始终通过 GLES 来执行，这意味着内容更新可能会导致其他 View 元素重绘。
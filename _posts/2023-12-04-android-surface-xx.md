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
## 0. Surface 和 SurfaceHolder

### 0.1 Surface

**Surface 可以把它理解为一个Buffer，它是一块屏幕缓冲区**。每个Window(窗口)对应一个Surface，任何View都是画在Surface上的。

Surface是纵深排序(Z-ordered)的，这表明它总在自己所在窗口的后面。Surfaceview 提供了一个可见区域，只有在这个可见区域内的Surface部分内容才可见，可见区域外的部分不可见。
传统的View共享一块屏幕缓冲区，所有的绘制必须在UI线程中进行。由于UI线程是主线程，如果视频的绘制也与UI放在一个线程中，那么它将严重影响主线程工作。所以Android又提供了其它View，这些View可以通过其它线程进行渲染。

### 0.2 SurfaceHolder

它是 Surface 的抽象接口，它使你可以控制Surface的大小和格式，以及在Surface上编辑像素和监视Surace的改变。这个接口通常通过SurfaceView类实现。



## 1. SurfaceView = Surface + View

[SurfaceView 和 GLSurfaceView](https://source.android.google.cn/docs/core/graphics/arch-sv-glsv?hl=zh-cn)。SurfaceView 结合了 Surface 和 View。SurfaceView 的 View 组件由 SurfaceFlinger（而不是应用）合成，从而可以通过单独的线程/进程渲染，并与应用界面渲染隔离。

SurfaceView 是一个**可以在子线程中更新 UI 的 View，且不会影响到主线程**。它为自己创建了一个窗口（window），就好像在视图层次（View Hierarchy）上穿了个“洞”，让绘图层（Surface）直接显示出来。但是，**和常规视图（view）不同，它没有动画或者变形特效，一些 View 的特性也无法使用**。

- SurfaceView 独立于视图层次（View Hierarchy），拥有自己的绘图层（Surface），但也没有一些常规视图（View）的特性，如动画等。
- SurfaceView 的实现中具有两个绘图层（Surface），即我们所说的双缓冲机制。我们的绘制发生在后台画布上，并通过交换前后台画布来刷新画面，可避免局部刷新带来的闪烁，也提高了渲染效率。
- SurfaceView 中的 SurfaceHolder 是 Surface 的持有者和管理控制者。
- **SurfaceHolder.Callback 的各个回调发生在主线程。**



## 2. GLSurfaceView = SurfaceView+EGL

[SurfaceView 和 GLSurfaceView](https://source.android.google.cn/docs/core/graphics/arch-sv-glsv?hl=zh-cn)。GLSurfaceView 提供了用于管理 EGL 上下文、线程间通信以及与 activity 生命周期的交互的帮助程序类（但不是必须使用 GLES）。

GLSurfaceView 继承 SurfaceView，除了拥有 SurfaceView 所有特性外，**还加入了 EGL（EGL 是 OpenGL ES 和原生窗口系统之间的桥梁） 的管理，并自带了一个单独的渲染线程**。

- 继承自 SurfaceView，拥有其所有特性。
- 加入了 EGL 管理，是 SurfaceView 应用 OpenGL ES 的典型场景。
- 有单独的渲染线程 GLThread。
- 单独出了 Renderer 接口负责实际渲染，不同的 Renderer 实现相当于不同的渲染策略，使用方式灵活（策略模式）。



## 3. SurfaceTexture

[SurfaceTexture](https://source.android.google.cn/docs/core/graphics/arch-st?hl=zh-cn)。 SurfaceTexture 将 Surface 和 GLES 纹理相结合来创建 BufferQueue，而您的应用是 BufferQueue 的消费者。当生产者将新的缓冲区排入队列时，它会通知您的应用。您的应用会依次释放先前占用的缓冲区，从队列中获取新缓冲区并执行 EGL 调用，从而使 GLES 可将此缓冲区作为外部纹理使用。Android 7.0 添加了对安全纹理视频播放的支持，以便对受保护的视频内容进行 GPU 后处理。

SurfaceTexture是离屏渲染和TextureView的核心，内部包含了一个BufferQueue，可以把Surface生成的图像流，转换为纹理，供业务方进一步加工使用。可**以做各种滤镜。像直播中经常用到的美颜，水印等都可以通过它来处理**。

把SurfaceTexture设置给Camera接收摄像头图像数据，并转换为OES纹理，然后可以利用OpenGL对OES纹理做进一步特效处理，最后上屏或者录制成视频。

说白了，

- 首先，通过Canvas、OpenGL、Camera或者Video Decoder生成图像流。
- 接着，图像流通过Surface入队到BufferQueue，并通知到GLConsumer。创建 BufferQueue， 是生产者；
- 然后，GLConsumer从BufferQueue获取图像流GraphicBuffer，并转换为纹理。
- 最后，应用层可以对纹理进一步处理，例如：上特效或者上屏。应用层就是BufferQueue 的消费者；

![surfacetexture]({{ site.url }}{{ site.baseurl }}/images/android-surface-xx.assets/surfacetexture.png)

参考：[Android图形系统之SurfaceTexture](https://juejin.cn/post/6844904161645953038)
[谈一谈Android上的SurfaceTexture](https://juejin.cn/post/6854573221036392461)



Android 3.0（API 11）新加入的一个类，**不同于 SurfaceView 会将图像显示在屏幕上，SurfaceTexture 对图像流的处理并不直接显示，而是转为 GL 外部纹理。**

- SurfaceTexture 可以从图像流（相机、视频）中捕获帧数据用作 OpenGL ES 外部纹理（GL_TEXTURE_EXTERNAL_OES），实现无缝连接。
- 应用层可以很方便的对这个外部纹理进行二次处理（如添加滤镜等）。
- **输出目标是 Camera 或 MediaPlayer 时，可以用 SurfaceTexture 代替 SurfaceHolder，这样图像流将把所有帧传给 SurfaceTexture 而不是显示在设备上。**
- 使用 updateTexImage() 方法更新最新的图像。



## 4. TextureView = View + SurfaceTexture

[TextureView](https://source.android.google.cn/docs/core/graphics/arch-tv?hl=zh-cn)。 TextureView 结合了 View 和 SurfaceTexture。TextureView 对 SurfaceTexture 进行包装，并负责响应回调以及获取新的缓冲区。在绘图时，TextureView 使用最近收到的缓冲区的内容作为其数据源，根据 View 状态指示，在它应该渲染的任何位置和以它应该采用的任何渲染方式进行渲染。View 合成始终通过 GLES 来执行，这意味着内容更新可能会导致其他 View 元素重绘。

TextureView 是 Android 4.0（API 14）引入，它必须使用在开启了硬件加速的窗体中。除了拥有 SurfaceView 的特性外，它还可以进行像常规视图（View）那样进行平移、缩放等动画。

- 必须开启硬件加速（这个默认就是开启的）。
- 可以像常规视图（View）那样使用它，包括进行平移、缩放等操作。
- TextureView 重载了 draw() 方法，主要是使用 SurfaceTexture 中收到的图像数据作为纹理更新到对应的 HardwareLayer 中。
- 通过 SurfaceTextureListener 接口让使用者知道 SurfaceTexture 的各种状态



## 5. SurfaceView 对比 TextureView

|            | SurfaceView | TextureView |
| ---------- | ----------- | ----------- |
| 内存       | 较低        | 较高        |
| 绘制       | 及时        | 1-3 帧延迟  |
| 耗电       | 较低        | 较高        |
| 动画和截图 | 不支持      | 支持        |



## 6. SurfaceView, SurfaceTexture区别

不同于 SurfaceView 会将图像显示在屏幕上，SurfaceTexture 对图像流的处理并不直接显示，而是转为 GL 外部纹理。





## 参考

[浅谈 SurfaceView、TextureView、GLSurfaceView、SurfaceTexture](https://blog.csdn.net/afei__/article/details/100023701)

[Android 5.0(Lollipop)中的SurfaceTexture，TextureView, SurfaceView和GLSurfaceView](https://blog.csdn.net/jinzhuojun/article/details/44062175)
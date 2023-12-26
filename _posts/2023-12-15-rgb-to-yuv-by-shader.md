---
layout: post
title: 通过opengles 将 rgb 转换为 yuv
date: 2023-12-15 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: opengl
---


* content
{:toc}

---


## 1. 前言

在视频采集、美颜以及其他前处理之后，需要将数据转换成I420格式，方便后续处理，如果使用CPU去做会导致性能瓶颈。因为我们在前处理过程都是采用GPU去做，因此这个转换使用GPU去做不仅方便，而且能充分利用GPU的优势。

转换步骤：

- 先将 RGBA 按照公式转换为 YUV 如（YUYV）；

- 然后将 YUYV 按照 RGBA 进行排布；

- 最后使用 glReadPixels 读取 YUYV 数据，


注意：

- 由于 YUYV 数据量为 RGBA 的一半，需要注意输出 buffer 的大小，
- 以及 viewport 的宽度（宽度为原来的一半）。



## 2. 转换公式

```less
Y = 0.256788 * R + 0.504129 * G + 0.0979059 * B
U = -0.148223 * R - 0.290993 * G + 0.439216 * B + 0.5
V = 0.439216 * R - 0.367788 * G - 0.0714274 * B + 0.5
```



## 3. 输入输出数据格式

在编写OpenGL ES的shader前，先需要确定好fragment shader的输入和输出格式。

### 3.1 输入

输入可以是一个包含RGBA的texture，或者是分别包含Y、U、V的三个texture，也可以使包含Y和UV的两个texture（UV分别放在texture rgba的r和a中，NV21和NV12都可以用这种方式）。

### 3.3 输出

输出的texture不仅要包含所有的YUV信息，还要方便我们一次性读取I420格式数据（glReadPixels）。因此输出数据的YUV紧凑地分布：

```less
    +---------+
    |         |
    |  Y      |
    |         |
    |         |
    +----+----+
    | U  | V  |
    |    |    |
    +----+----+
```

而对于OpenGL ES来说，目前它输入只认RGBA、lumiance、luminace alpha这几个格式，输出大多数实现只认RGBA格式，因此输出的数据格式虽然是I420格式，但是在存储是我们仍然要按照RGBA方式去访问texture数据。

对于上述存储布局，输出的texture宽度为width/4，高度为height+height/2。这样一张1280*720的图，需要申请的纹理大小为：360x1080。



### 3.3 !!! 原理说明

以NV21的YUV数据为例，它的内存大小为width x height * 3 / 2。如果是RGBA的格式存储的话，占用的内存空间大小是width x height x 4（因为 RGBA 一共4个通道）。很显然它们的内存大小是对不上的，
那么该如何调整Opengl buffer的大小让RGBA的输出能对应上YUV的输出呢？我们可以设计输出的宽为width / 4，高为height * 3 / 2即可。

**为什么是这样的呢？虽然我们的目的是将RGB转换成YUV，但是我们的输入和输出时读取的依然是RGBA，也就是说：width x height x 4 = (width / 4) x (height * 3 / 2) * 4**

而YUV数据在内存中的分布以下这样子的：

```ruby
width / 4
|--------------|
|              |
|              | h
|      Y       |
|--------------|            
|   U   |  V   |
|       |      |  h / 2
|--------------|
```

那么上面的排序如果进行了归一化之后呢，就变成了下面这样子了：

```ruby
(0,0) width / 4  (1,0)
|--------------|
|              |
|              |  h
|      Y       |
|--------------|  (1,2/3)          
|   U   |  V   |
|       |      |  h / 2
|--------------|
(0,1)           (1,1)
```

从上面的排布可以看出看出，

- 在纹理坐标`y < (2/3)`时，需要完成一次对整个纹理的采样，用于生成Y数据，

- 当纹理坐标 `y > (2/3)`时，同样需要再进行一次对整个纹理的采样，用于生成UV的数据。

- 同时还需要将我们的视窗设置为`glViewport(0, 0, width / 4, height * 1.5);`

  **由于视口宽度设置为原来的 1/4 ，可以简单的认为相对于原来的图像每隔4个像素做一次采样，由于我们生成Y数据是要对每一个像素都进行采样，所以还需要进行3次偏移采样。**

  ![img]({{ site.url }}{{ site.baseurl }}/images/rgb-to-yuv-by-shader.assets/y-capture.png)

  

  同理，生成对于UV数据也需要进行3次额外的偏移采样。

  ![img]({{ site.url }}{{ site.baseurl }}/images/rgb-to-yuv-by-shader.assets/vu-capture.png)

  

  1. **在着色器中offset变量需要设置为一个归一化之后的值：`1.0/width`, 步长， 按照原理图，在纹理坐标 y < (2/3) 范围，一次采样（加三次偏移采样）4 个 RGBA 像素（R,G,B,A）生成 1 个（Y0,Y1,Y2,Y3），整个范围采样结束时填充好 `width*height` 大小的缓冲区；**

  2. **当纹理坐标 y > (2/3) 范围，一次采样（加三次偏移采样）4 个 RGBA 像素（R,G,B,A）生成 1 个（V0,U0,V0,U1），**


     **又因为 UV 缓冲区的高度为 height/2 ，VU plane 在垂直方向的采样是隔行进行。可以理解为， 4个像素RGB对应 一对VU，所以隔行，就相当于少了一半的高度，所以高度为height/2；在要采集的那一行里，进行一次原始采集和3次偏移采集 ；从而得到V0U0V1U1。**


     **整个范围采样结束时填充好 `width*height/2` 大小的缓冲区。**

  

  https://www.jianshu.com/p/1a231aac47d7
  

#### GPU 上面的实现

考虑在 GPU 上执行 RGB --> YUV 转换。GPU 的流水线操作：

```less
vertices 
        ----> Pipeline ----> Out color
texture
```

所以将 RGB 图像作为纹理输入，流水线输出我们需要的 YUV 数据。前面一部分很好理解，图像作为唯一的纹理输入，没有别的选项。后面一部分的话，需要在输出的时候输出我们需要的 YUV 数据即可，在 fragment shader 中的输出按常理就是每一个 fragment 的颜色，为实现读取像素是 YUV 的目标，要调整输出的数据。

考虑 YUV 格式内存分布，以 NV12 为例，一张图片占用内存大小为：`width x height * 3 / 2` (我们认为图像的宽为 width 高为 height). 如果是 RGBA 的格式存储的话，占用的内存空间大小是：`width x height x 4` （因为 RGBA 一共4个通道）。如果我们把 OpenGL renderbuffer 大小设置成等于图像的大小，那输出的大小就是 RGBA 那一种的大小，和 YUV 格式的是对不上的。考虑 YUV 的分布特点，设计输出的宽高为 `(width / 4, height * 3 / 2)`. 示意图如下：

```less
Memory of a frame (yuv format)

  width / 4
|-------------|
|             |
|             | h
| chrominance |
|             |
|-------------|
|             | 
| luminance   | h / 2
|-------------|
```

因为每一个 out color 含有四个分量 RGBA 所以将宽度设为 width / 4, 那么正好每一行的像素就是原来 width 的数量。在 fragment shader 内部计算的时候，需要考虑当前处理的单个 fragment 是属于 chrominance OR luminance, 可以用纹理坐标的 t 值的大小来判断。

#### !!! Chrominance

所谓的 RGBA 四个分量实际上代表四个不同的像素的 chrominance 值，也就是说需要做一定的 offset, 来获取到当前像素附近的像素的值，我先假定 offset 为 1.0f / width. 故四个分量如下：

1. (s, t)
2. (s + off, t)
3. (s + off x 2.0f, t)
4. (s + off x 3.0f, t)

**根据四个像素的 RGBA 值计算出四个 Y 通道的数据作为这个 fragment 的输出颜色。**

#### !!! Luminance

仍然是一个像素四个分量，但是现在代表的是两对 UV 分量。因为根据一个 RGBA 就可以算出 YUV 值，所以此处只需要做一个偏移。

1. (s, t)
2. (s + off x 2, t)

**这里 offset 的设置可以乘 1 或 2 或 3，我觉得都可以，我只是取中道选择了 2. 将上面两个像素的 UV 分量作为这个 fragment 的输出颜色。**

#### readback pixel

最终用 `glReadpixels()` 函数，将我们输出的颜色读回来，就完成了。

#### 补充

实际操作中遇到的一个问题是，如果设置了 `GL_BLEND`, 最终输出的颜色会是混合以后的颜色，记得一定要确认关闭了 blending.

https://www.cnblogs.com/psklf/p/8399726.html



## 4. 片段着色器脚本

```glsl
// 片元着色器
static const char *fragment = "#version 300 es\n"
                              "precision mediump float;\n"
                              "in vec2 v_texCoord;\n"
                              "layout(location = 0) out vec4 outColor;\n"
                              "uniform sampler2D s_TextureMap;\n"
                              "uniform float u_Offset;\n"
                              "const vec3 COEF_Y = vec3(0.299, 0.587, 0.114);\n"
                              "const vec3 COEF_U = vec3(-0.147, -0.289, 0.436);\n"
                              "const vec3 COEF_V = vec3(0.615, -0.515, -0.100);\n"
                              "const float UV_DIVIDE_LINE = 2.0 / 3.0;\n"
                              "void main(){\n"
                              "    vec2 texelOffset = vec2(u_Offset, 0.0);\n"
                              "    if (v_texCoord.   y <= UV_DIVIDE_LINE) {\n"
                              "        vec2 texCoord = vec2(v_texCoord.   x, v_texCoord.   y * 3.0 / 2.0);\n"
                              "        vec4 color0 = texture(s_TextureMap, texCoord);\n"
                              "        vec4 color1 = texture(s_TextureMap, texCoord + texelOffset);\n"
                              "        vec4 color2 = texture(s_TextureMap, texCoord + texelOffset * 2.0);\n"
                              "        vec4 color3 = texture(s_TextureMap, texCoord + texelOffset * 3.0);\n"
                              "        float y0 = dot(color0.   rgb, COEF_Y);\n"
                              "        float y1 = dot(color1.   rgb, COEF_Y);\n"
                              "        float y2 = dot(color2.   rgb, COEF_Y);\n"
                              "        float y3 = dot(color3.   rgb, COEF_Y);\n"
                              "        outColor = vec4(y0, y1, y2, y3);\n"
                              "    } else {\n"
                              "        vec2 texCoord = vec2(v_texCoord.x, (v_texCoord.y - UV_DIVIDE_LINE) * 3.0);\n"
                              "        vec4 color0 = texture(s_TextureMap, texCoord);\n"
                              "        vec4 color1 = texture(s_TextureMap, texCoord + texelOffset);\n"
                              "        vec4 color2 = texture(s_TextureMap, texCoord + texelOffset * 2.0);\n"
                              "        vec4 color3 = texture(s_TextureMap, texCoord + texelOffset * 3.0);\n"
                              "        float v0 = dot(color0.   rgb, COEF_V) + 0.5;\n"
                              "        float u0 = dot(color1.   rgb, COEF_U) + 0.5;\n"
                              "        float v1 = dot(color2.   rgb, COEF_V) + 0.5;\n"
                              "        float u1 = dot(color3.   rgb, COEF_U) + 0.5;\n"
                              "        outColor = vec4(v0, u0, v1, u1);\n"
                              "    }\n"
                              "}";
```



### 方案一

```glsl
#extension GL_OES_EGL_image_external : require
precision mediump float;

// tc是texture coordinate，可以理解成输出纹理坐标
varying vec2 tc;
uniform samplerExternalOES tex;
// 在x方向上，一个像素的步长（纹理已经做过归一化，这个步长不是像素个数）
uniform vec2 xUnit;
// rgb转yuv的向量， RGB to YUV的颜色转换系数
uniform vec4 coeffs;

// 虽然alpha通道值总是1，我们可以写成一个vec4xvec4的矩阵乘法，但是这样做实际
// 导致了较低帧率，这里用了vec3xvec3乘法。
void main() {
	gl_FragColor.r = coeffs.a + dot(coeffs.rgb,
              texture2D(tex, tc - 1.5 * xUnit).rgb);
	gl_FragColor.g = coeffs.a + dot(coeffs.rgb,
							texture2D(tex, tc - 0.5 * xUnit).rgb);
	gl_FragColor.b = coeffs.a + dot(coeffs.rgb,
							texture2D(tex, tc + 0.5 * xUnit).rgb);
	gl_FragColor.a = coeffs.a + dot(coeffs.rgb,
							texture2D(tex, tc + 1.5 * xUnit).rgb);
}
```



### 方案二

```glsl
#version 300 es
precision mediump float;
in vec2 v_texCoord;
layout(location = 0) out vec4 outColor;
uniform sampler2D s_TextureMap;//RGBA 纹理
uniform float u_Offset;//采样偏移

const vec3 COEF_Y = vec3( 0.256788,  0.504129,  0.0979059);
const vec3 COEF_U = vec3(-0.148223, -0.290993,  0.439216);
const vec3 COEF_V = vec3( 0.439216, -0.367788, -0.0714274);

void main()
{
    vec2 texelOffset = vec2(u_Offset, 0.0);
    vec4 color0 = texture(s_TextureMap, v_texCoord);
    #偏移 offset 采样
    vec4 color1 = texture(s_TextureMap, v_texCoord + texelOffset);

    // 采集color0
    float y0 = dot(color0.rgb, COEF_Y);
    float u0 = dot(color0.rgb, COEF_U) + 0.5;
    float v0 = dot(color0.rgb, COEF_V) + 0.5;
    // 采集 color1
    float y1 = dot(color1.rgb, COEF_Y);

    outColor = vec4(y0, u0, y1, v0);
}
```





## 5. 纹理绘制

### 转换的向量

```java
private static class ShaderCallbacks implements GlGenericDrawer.ShaderCallbacks {
    // Y'UV444 to RGB888, see https://en.wikipedia.org/wiki/YUV#Y%E2%80%B2UV444_to_RGB888_conversion
    // We use the ITU-R BT.601 coefficients for Y, U and V.
    // The values in Wikipedia are inaccurate, the accurate values derived from the spec are:
    // Y = 0.299 * R + 0.587 * G + 0.114 * B
    // U = -0.168736 * R - 0.331264 * G + 0.5 * B + 0.5
    // V = 0.5 * R - 0.418688 * G - 0.0813124 * B + 0.5
    // To map the Y-values to range [16-235] and U- and V-values to range [16-240], the matrix has
    // been multiplied with matrix:
    // {219 / 255, 0, 0, 16 / 255},
    // {0, 224 / 255, 0, 16 / 255},
    // {0, 0, 224 / 255, 16 / 255},
    // {0, 0, 0, 1}
    private static final float[] yCoeffs =
        new float[] {0.256788f, 0.504129f, 0.0979059f, 0.0627451f};
    private static final float[] uCoeffs =
        new float[] {-0.148223f, -0.290993f, 0.439216f, 0.501961f};
    private static final float[] vCoeffs =
        new float[] {0.439216f, -0.367788f, -0.0714274f, 0.501961f};

    private int xUnitLoc;
    private int coeffsLoc;

    private float[] coeffs;
    private float stepSize;

    public void setPlaneY() {
      coeffs = yCoeffs;
      stepSize = 1.0f;
    }

    public void setPlaneU() {
      coeffs = uCoeffs;
      stepSize = 2.0f;
    }

    public void setPlaneV() {
      coeffs = vCoeffs;
      stepSize = 2.0f;
    }

    @Override
    public void onNewShader(GlShader shader) {
      xUnitLoc = shader.getUniformLocation("xUnit");
      coeffsLoc = shader.getUniformLocation("coeffs");
    }

    @Override
    public void onPrepareShader(GlShader shader, float[] texMatrix, int frameWidth, int frameHeight,
        int viewportWidth, int viewportHeight) {
      GLES20.glUniform4fv(coeffsLoc, /* count= */ 1, coeffs, /* offset= */ 0);
      // Matrix * (1;0;0;0) / (width / stepSize). Note that OpenGL uses column major order.
      GLES20.glUniform2f(
          xUnitLoc, stepSize * texMatrix[0] / frameWidth, stepSize * texMatrix[1] / frameWidth);
    }
  }

```



### 绘制

```java
   final int frameWidth = preparedBuffer.getWidth();
    final int frameHeight = preparedBuffer.getHeight();
    final int stride = ((frameWidth + 7) / 8) * 8;
    final int uvHeight = (frameHeight + 1) / 2;
    // Total height of the combined memory layout.
    final int totalHeight = frameHeight + uvHeight;
    final ByteBuffer i420ByteBuffer = JniCommon.nativeAllocateByteBuffer(stride * totalHeight);
    // Viewport width is divided by four since we are squeezing in four color bytes in each RGBA
    // pixel.
    final int viewportWidth = stride / 4;

    // Produce a frame buffer starting at top-left corner, not bottom-left.
    final Matrix renderMatrix = new Matrix();
    renderMatrix.preTranslate(0.5f, 0.5f);
    renderMatrix.preScale(1f, -1f);
    renderMatrix.preTranslate(-0.5f, -0.5f);

    i420TextureFrameBuffer.setSize(viewportWidth, totalHeight);

    // Bind our framebuffer.
    GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, i420TextureFrameBuffer.getFrameBufferId());
    GlUtil.checkNoGLES2Error("glBindFramebuffer");

    // Draw Y.
    shaderCallbacks.setPlaneY();
    VideoFrameDrawer.drawTexture(drawer, preparedBuffer, renderMatrix, frameWidth, frameHeight,
        /* viewportX= */ 0, /* viewportY= */ 0, viewportWidth,
        /* viewportHeight= */ frameHeight);

    // Draw U.
    shaderCallbacks.setPlaneU();
    VideoFrameDrawer.drawTexture(drawer, preparedBuffer, renderMatrix, frameWidth, frameHeight,
        /* viewportX= */ 0, /* viewportY= */ frameHeight, viewportWidth / 2,
        /* viewportHeight= */ uvHeight);

    // Draw V.
    shaderCallbacks.setPlaneV();
    VideoFrameDrawer.drawTexture(drawer, preparedBuffer, renderMatrix, frameWidth, frameHeight,
        /* viewportX= */ viewportWidth / 2, /* viewportY= */ frameHeight, viewportWidth / 2,
        /* viewportHeight= */ uvHeight);

    GLES20.glReadPixels(0, 0, i420TextureFrameBuffer.getWidth(), i420TextureFrameBuffer.getHeight(),
        GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, i420ByteBuffer);
```



1. draw y

```java
		shaderCallbacks.setPlaneY();
    VideoFrameDrawer.drawTexture(drawer, preparedBuffer, renderMatrix, frameWidth, frameHeight,
        /* viewportX= */ 0, /* viewportY= */ 0, viewportWidth,
        /* viewportHeight= */ frameHeight);
```

setPlaneY的时候把`{0.256788f, 0.504129f, 0.0979059f, 0.0627451f} ` 赋值给 glsl 中的`coeffs`;

`stepSize = 1.0f;`  赋值给 glsl 中的`xUnit`;

drawTexture





```java
@Override
public void onPrepareShader(GlShader shader, float[] texMatrix, int frameWidth, int frameHeight,
    int viewportWidth, int viewportHeight) {
  GLES20.glUniform4fv(coeffsLoc, /* count= */ 1, coeffs, /* offset= */ 0);
  // Matrix * (1;0;0;0) / (width / stepSize). Note that OpenGL uses column major order.
  GLES20.glUniform2f(
      xUnitLoc, stepSize * texMatrix[0] / frameWidth, stepSize * texMatrix[1] / frameWidth);
}
```







## u_Offset

要将 RGBA 转成 YUYV，数据量相比于 RGBA 少了一半，这就**相当于将两个像素点合并成一个像素点**。

如图所示，我们在 shader 中执行两次采样，

- RGBA 像素（R0,G0,B0,A0）转换为（Y0，U0，V0），
- 像素（R1,G1,B1,A1）转换为（Y1），然后组合成（Y0，U0，Y1，V0），

这样 8 个字节表示的 2 个 RGBA 像素就转换为 4 个字节表示的 2 个 YUYV 像素。



**转换成 YUYV 时数据量减半，那么 glViewPort 时 width 变为原来的一半，同样 glReadPixels 时 width 也变为原来的一半。**

实现 RGBA 转成 YUYV 要保证原图分辨率不变，**建议**[**使用 FBO 离屏渲染**](https://cloud.tencent.com/developer/tools/blog-entry?target=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIwNTIwMzAzNg%3D%3D%26mid%3D2654164511%26idx%3D1%26sn%3D17fa1bba43703662803ea763b741cbfa%26chksm%3D8cf3852cbb840c3a2855fe7d0d8f68693e8346b9c41eb9e51534400768dc28257751faaa0b95%26scene%3D21%23wechat_redirect&source=article&objectId=1832837) **，这里注意绑定给 FBO 的纹理是用来容纳 YUYV 数据，其宽度应该设置为原图的一半**。



## 参考

[Convert RGBA to YUV by OpenGL ES shader](https://www.cnblogs.com/tkorays/p/11644836.html)

[请使用 OpenGL ES 将 RGB 图像转换为 YUV 格式](https://cloud.tencent.com/developer/article/1832837)

https://www.jianshu.com/p/878affffe413

[使用OpenGL ES shader做RGBA转YUV(I420](https://blog.51cto.com/u_12127193/5739323)

https://blog.csdn.net/a360940265a/article/details/121532966

[Opengl ES之RGB转NV21](https://www.jianshu.com/p/1a231aac47d7)

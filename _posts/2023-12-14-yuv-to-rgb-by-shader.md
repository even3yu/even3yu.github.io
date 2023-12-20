---
layout: post
title: 通过opengles 将 yuv 转换为 rgb
date: 2023-12-14 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: opengl
---


* content
{:toc}

---


## 0. 前言

软件解码的时候，得到yuv数据，opengl一般是支持rgb渲染，所以需要通过glsl 进行yuv到rgb的转换。
硬件解码久不需要了，出来就是纹理，只要对纹理做渲染久可以了。???

## 1. NV21 格式介绍

当前文章主要是以NV21来说明的。

实际上就是利用 shader 实现了 YUV（NV21） 到 RGBA 的转换，然后渲染到屏幕上。

以渲染 NV21 格式的图像为例，下面是 (4x4) NV21 图像的 YUV 排布：

```javascript
(0  ~  3) Y00  Y01  Y02  Y03  
(4  ~  7) Y10  Y11  Y12  Y13  
(8  ~ 11) Y20  Y21  Y22  Y23  
(12 ~ 15) Y30  Y31  Y32  Y33  

(16 ~ 19) V00  U00  V01  U01 
(20 ~ 23) V10  U10  V11  U11
```



## 2. yuv转rgb的公式

```less
R = 1.16438 * Y 								+ 1.59603 * Cr - 0.874202
G = 1.16438 * Y - 0.391762 * Cb - 0.812968 * Cr - 0.531668
B = 1.16438 * Y + 2.01723 * Cb - 1.08563
```



## 3. YUV 渲染步骤

- 生成 2 个纹理，编译链接着色器程序；
- 确定纹理坐标及对应的顶点坐标；
- 分别加载 NV21 的两个 Plane 数据到 2 个纹理，加载纹理坐标和顶点坐标数据到着色器程序；
- 绘制。

[NDK OpenGLES 3.0 开发（三）：YUV 渲染 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzIwNTIwMzAzNg==&mid=2654161548&idx=1&sn=50ba9f1fe3ac66321a6e7f0f8334a371&chksm=8cf399bfbb8410a9ff44bc62af3f8d5fb7208be94f619b29c6b934a0aeff4f557f84abafcd84&scene=21#wechat_redirect)



## 4. !!! 片段着色器脚本

片段着色器实现了yuv转rgb的功能。

```glsl
#version 300 es                                     
precision mediump float;   
# 纹理桌标
in vec2 v_texCoord;   

layout(location = 0) out vec4 outColor;   
# 两个纹理
uniform sampler2D y_texture;                        
uniform sampler2D uv_texture;                        
void main()                                         
{                                                   
    vec3 yuv;      
    # y_texture取得是r
    yuv.x = texture(y_texture, v_texCoord).r * 1.16438;  
    #uv_texture取得是a， 
    yuv.y = texture(uv_texture, v_texCoord).a; 
    #uv_texture取得是r，
    yuv.z = texture(uv_texture, v_texCoord).r;    
    vec3 rgb =mat3( 1.0,       1.0,           1.0,                    
                    0.0,       -0.391762,     2.01723,                  
                    1.59603,   -0.812968,     0.0) * yuv;             
    outColor = vec4(rgb.x - 0.874202, rgb.y - 0.531668, rgb.z - 1.08563,  1);                        
}
```

- y_texture 和 uv_texture 分别是 NV21 Y Plane 和 UV Plane 纹理的采样器，对两个纹理采样之后组成一个（y,u,v）三维向量

- 左乘变换矩阵转换为（r,g,b）三维向量。

- 转换公式，注意的是 OpenGLES 的内置矩阵实际上是一列一列地构建的

  ```glsl
  mat3 convertMat = mat3(1.0, 1.0, 1.0,      //第一列
                         0.0，-0.391762，2.01723， //第二列
                         1.59603，-0.812968， 0.0);//第三列
  ```

- YUV 转 RGB shader 中，面试官喜欢问的问题（一脸坏笑）：**为什么 UV 分量要减去 0.874202, 0.531668 啊？**

  **因为归一化**。

  Y [16,235]， U，V [16,240]

  **YUV 格式图像 UV 分量的默认值分别是 127 ，Y 分量默认值是 16 ，8 个 bit 位的取值范围是 16~235，由于在 shader 中纹理采样值需要进行归一化，所以 UV 分量的采样值需要分别减去 0.874202, 0.531668 ，确保 YUV 到 RGB 正确转换**。

- `uv_texture` 为什么取的 是a，和r？？？？？
  
  ```less
  nv12 , r and a
  nv21, a and r
  i420, r
  ```

> 需要注意的是 OpenGL ES 实现 YUV 渲染需要用到 GL_LUMINANCE 和 GL_LUMINANCE_ALPHA 格式的纹理。
>**其中 GL_LUMINANCE 纹理用来加载 NV21 Y Plane 的数据，GL_LUMINANCE_ALPHA 纹理用来加载 UV Plane 的数据。**

关于 shader 实现 YUV 转 RGB （NV21、NV12、I420 格式图像渲染）可以参考文章：[OpenGL ES 3.0 开发（三）：YUV 渲染](https://cloud.tencent.com/developer/tools/blog-entry?target=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIwNTIwMzAzNg%3D%3D%26mid%3D2654161548%26idx%3D1%26sn%3D50ba9f1fe3ac66321a6e7f0f8334a371%26chksm%3D8cf399bfbb8410a9ff44bc62af3f8d5fb7208be94f619b29c6b934a0aeff4f557f84abafcd84%26scene%3D21%23wechat_redirect&source=article&objectId=1832837) 和 [FFmpeg 播放器视频渲染优化](https://cloud.tencent.com/developer/tools/blog-entry?target=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIwNTIwMzAzNg%3D%3D%26mid%3D2654163238%26idx%3D1%26sn%3D3f778082ee5a278bdd05743c33247127%26chksm%3D8cf38215bb840b03fb6de375e22521fa3db762acdc09a37f9240648845e1057ff4c65d4a5661%26scene%3D21%23wechat_redirect&source=article&objectId=1832837)，本文主要重点讲 shader 如何实现 RGB 转 YUV 。



## 5. OpenGLES 常用纹理的格式类型

![图片]({{ site.url }}{{ site.baseurl }}/images/yuv-to-rgb-by-shader.assets/texture.png)

- GL_LUMINANCE 纹理在着色器中采样的纹理像素格式是（L，L，L，1），L 表示亮度。
- GL_LUMINANCE 纹理在着色器中采样的纹理像素格式是（L，L，L，A），A 表示透明度。



```
GL_LUMINANCE是将单个u或v打包到一个纹理的vec4里面，放到第一位：vec4(u,0,0,0)
GL_LUMINANCE_ALPHA是将相邻的两个uv打包到一个纹理的vec4里面，放到第一和最后一位：vec4(u,0,0,v)

https://www.jianshu.com/p/878affffe413
```







## 6. 不同格式的yuv 片段着色器

[FFmpeg 播放器视频渲染优化 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzIwNTIwMzAzNg==&mid=2654163238&idx=1&sn=3f778082ee5a278bdd05743c33247127&chksm=8cf38215bb840b03fb6de375e22521fa3db762acdc09a37f9240648845e1057ff4c65d4a5661&scene=21#wechat_redirect)

```glsl
//顶点着色器
#version 300 es
layout(location = 0) in vec4 a_Position;
layout(location = 1) in vec2 a_texCoord;
out vec2 v_texCoord;
void main()
{
    gl_Position = a_Position;
    v_texCoord = a_texCoord;
}

//片段着色器
#version 300 es
precision highp float;
in vec2 v_texCoord;
layout(location = 0) out vec4 outColor;
uniform sampler2D s_texture0;
uniform sampler2D s_texture1;
uniform sampler2D s_texture2;
uniform int u_ImgType;// 1:RGBA, 2:NV21, 3:NV12, 4:I420

void main()
{

    if(u_ImgType == 1) //RGBA
    {
        outColor = texture(s_texture0, v_texCoord);
    }
    else if(u_ImgType == 2) //NV21
    {
        vec3 yuv;
        yuv.x = texture(s_texture0, v_texCoord).r;
        yuv.y = texture(s_texture1, v_texCoord).a - 0.5;
        yuv.z = texture(s_texture1, v_texCoord).r - 0.5;
        highp vec3 rgb = mat3(1.0,       1.0,     1.0,
        0.0,     -0.344,     1.770,
        1.403,  -0.714,     0.0) * yuv;
        outColor = vec4(rgb, 1.0);

    }
    else if(u_ImgType == 3) //NV12
    {
        vec3 yuv;
        yuv.x = texture(s_texture0, v_texCoord).r;
        yuv.y = texture(s_texture1, v_texCoord).r - 0.5;
        yuv.z = texture(s_texture1, v_texCoord).a - 0.5;
        highp vec3 rgb = mat3(1.0,       1.0,     1.0,
        0.0,     -0.344,     1.770,
        1.403,  -0.714,     0.0) * yuv;
        outColor = vec4(rgb, 1.0);
    }
    else if(u_ImgType == 4) //I420
    {
        vec3 yuv;
        yuv.x = texture(s_texture0, v_texCoord).r;
        yuv.y = texture(s_texture1, v_texCoord).r - 0.5;
        yuv.z = texture(s_texture2, v_texCoord).r - 0.5;
        highp vec3 rgb = mat3(1.0,       1.0,     1.0,
                              0.0,     -0.344,     1.770,
                              1.403,  -0.714,     0.0) * yuv;
        outColor = vec4(rgb, 1.0);
    }
    else
    {
        outColor = vec4(1.0);
    }
}
```

1. NV21, NV12 的区别， 

   ```glsl
   ...
   				yuv.y = texture(s_texture1, v_texCoord).r - 0.5;
           yuv.z = texture(s_texture1, v_texCoord).a - 0.5;
   ...
     
   ...
           yuv.y = texture(s_texture1, v_texCoord).r - 0.5;
           yuv.z = texture(s_texture1, v_texCoord).a - 0.5;
   ...
   ```

​	对y和z的取值不同，刚好相反； r，a。

> r,a 是向量的取值方法， 向量可以是 rgba 对应vec4的四个值

2. NV21，NV12 与i420的区别
   NV21，NV12 是 yTexture，uvTexture；
   i420是yTexture，uTexture，uTexture；






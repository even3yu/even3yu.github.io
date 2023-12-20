---
layout: post
title: rgb,yuv 转换的公式
date: 2023-12-13 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: opengl
---


* content
{:toc}

---


## 前言 

- RGB,YUV（YCbCr）是常用的颜色空间。RGB图像又称真彩色图像，R、G、B分别代表红、绿、蓝3种不同的颜色。
- [YCbCr](https://so.csdn.net/so/search?q=YCbCr&spm=1001.2101.3001.7020)模型广泛用于数字视频，Y表示亮度，Cb、Cr分别为蓝色分量和红色分量相对于参考值的坐标。



## 基础说明

YUV 是YUV颜色空间模式的总称，“Y”表示明亮度（Luminance或Luma），也就是灰阶值，“U”和“V”表示的则是色度（Chrominance或Chroma），作用是描述影像色彩及饱和度，用于指定像素的颜色。YUV模式有以下类型：

- YUV ： YUV是一种模拟型号， Y∈ [0,1]   U,V∈[-0.5,0.5] 

- YCbCr ：也叫YCC或者Y'CbCr，YCbCr 是数字信号，它包含两种形式，分别为TV range和full range，TV range 主要是广播电视采用的标准， full range主要是pc端采用的标准，所以full range 有时也叫 pc range。

  **(1) TV range 的各个分量的范围为： YUV  Y∈[16,235]   Cb∈[16-240]   Cr∈[16-240] 。**

  **(2) full range 的各个分量的范围均为：[0-255] 。PC机显卡输出的为full range模式。**

  

  我们平时接触到的绝大多数都是 **YCbCr （tv range）** ， ffmpeg 解码出来的数据绝大多数也是这个， 虽然ffmpeg 里面将它的格式描述成YUV420P ， 实际上它是**YCbCr420p tv range**



## 量化

什么是量化？**量化**就是让通过[线性变换](https://www.zhihu.com/search?q=线性变换&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A261286123})让Y 或 U 或V 处于一定的范围内， 比如让Y [0,1]变到 Y' [16,235] 就这样来实行： `Y' = Y* （235-16）/(1-0) + 16 即 Y' = 219*Y + 16`;

### 关于为什么要将YUV量化为tv range 16-235 ？

以下是维基百科摘抄的一段， 意思是**tv range是为了解决滤波（模数转换）后的过冲现象，**

　　Y′ values are conventionally shifted and scaled to the range [16, 235] (referred to as studio swing or "TV levels") rather than using the full range of [0, 255] (referred to as full swing or "PC levels"). This practice was standardized in SMPTE-125M in order to accommodate signal overshoots ("ringing") due to filtering. The value 235 accommodates a maximal black-to-white overshoot of 255 − 235 = 20, or 20 / (235 − 16) = 9.1%, which is slightly larger than the theoretical maximal overshoot ([Gibbs phenomenon](https://en.wikipedia.org/wiki/Gibbs_phenomenon)) of about 8.9% of the maximal step. The toe-room is smaller, allowing only 16 / 219 = 7.3% overshoot, which is less than the theoretical maximal overshoot of 8.9%. This is why 16 is added to Y′ and why the Y′ coefficients in the basic transform sum to 220 instead of 255.[[9\]](https://en.wikipedia.org/wiki/YUV#cite_note-9) U and V values, which may be positive or negative, are summed with 128 to make them always positive, giving a studio range of 16–240 for U and V. (These ranges are important in video editing and production, since using the wrong range will result either in an image with "clipped" blacks and whites, or a low-contrast image.)



### 关于如何判断像素格式是否为tv range  Y(16-235）？

　　1. 常见的一些解码帧结构体里面有color_range 参数， 如果为MPEG 或者 LIMITED 则表示为tv_range

　　2. 在完全黑画面的时候打印出图像的Y数据， 如果Y=16左右  说明YCbCr 为tv range ，如果Y=0左右  说明YCbCr为 full range



## YUV和RGB互转要考虑因素

- Matrix，BT601，是BT709, 4K时代的BT2020，
- range，YUV里TV和PC显示器的数值范围不一样8bit位深的情况下
  TV range是16-235(Y)、16-240(UV)；
  PC range是0-255；
  而RGB没有range之分，全是0-255；



## 转换公式

主要有BT601、BT709、BT2020三个标准



### BT601

#### 1. 模拟信号

 YUV   (  U∈[-0.5-0.5]  ,   R，G，B∈[0,1]  )

```less
R = Y + 1.4075 * V;  
G = Y - 0.3455 * U - 0.7169*V;  
B = Y + 1.779 * U;  


Y = 0.299 * R + 0.587 * G + 0.114 * B
U = -0.168736 * R - 0.331264 * G + 0.5 * B + 0.5
V = 0.5 * R - 0.418688 * G - 0.0813124 * B + 0.5
```



#### 2. 数字Full range，未量化

R，G，B [0,255] Y， U，V [-128,128]



#### 3. !!! 数字TV range

Y [16,235]， U，V [16,240]

```less
Y = 0.256788 * R + 0.504129 * G + 0.0979059 * B
U = -0.148223 * R - 0.290993 * G + 0.439216 * B + 0.5
V = 0.439216 * R - 0.367788 * G - 0.0714274 * B + 0.5

```



```less
R = 1.164 * (Y - 16) + 1.596 * (Cr - 128)
G = 1.164 * (Y- 16) - 0.392 * (Cb-128) - 0.812 *(Cr - 128)
B = 1.164 * (Y - 16) + 2.016 * (Cb - 128) 

===> 

R = Y 								+ 1.59603 * Cr - 0.874202
G = Y - 0.391762 * Cb - 0.812968 * Cr - 0.531668
B = Y + 2.01723 * Cb - 1.08563

```



![img](https://img-blog.csdnimg.cn/bb678db54fe840b68b3bd400cb7539f1.png)



### BT709

![img](https://img-blog.csdnimg.cn/c4e05798d04c475a97128cc14d3fa01b.png)



### BT2020

![img](https://img-blog.csdnimg.cn/f016971c8ca549389fa7eb14f0a2b497.png)



## 参考

[BT601/BT709/BT2020 YUV2RGB RGB2YUV 公式](https://blog.csdn.net/m18612362926/article/details/127667954)

[YUV与RGB互转各种公式](https://www.cnblogs.com/luoyinjie/p/7219319.html)
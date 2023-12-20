---
layout: post
title: opengl 坐标系
date: 2023-12-12 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: opengl
---


* content
{:toc}

---


## 1. Android 手机中的坐标系

<img src="{{ site.url }}{{ site.baseurl }}/images/android-coordinate-system.assets/android-cs-p.png" alt="img" style="zoom:50%;" />

  

## 2. OpenGL 顶点坐标系

<img src="{{ site.url }}{{ site.baseurl }}/images/android-coordinate-system.assets/opengl-cs-v.png" alt="img" style="zoom:50%;" />

顶点坐标是从-1到-1，坐标是x向右，y向上

 比如顶点坐标：

```java
static float vertexVertices[] = {  
        -1f, -1f,   /*左下角*/
        1f, -1f,    /*右下角*/
        -1f,  1f,   /*左上角*/
        1f,  1f,    /*右上角*/
    };
```
 

## 3. OpenGL 纹理坐标系 

 <img src="{{ site.url }}{{ site.baseurl }}/images/android-coordinate-system.assets/opengl-cs-texture.png" alt="img" style="zoom:50%;" />

纹理坐标是从0到1，它的坐标是x向右，y向下
当纹理坐标和顶点坐标的4个点相对应时，纹理图片是原始的位置

纹理坐标

```java
static float textureVertices[] = {  
        0.0f,  1f,      /*左下角*/
        1f,  1f,         /*右下角*/
        0.0f,  0.0f,   /*左上角*/
        1f,  0.0f,      /*右上角*/
    };
```

而对应的，如果用以下的纹理坐标，图片显示的与原始的会上下颠倒：

```java
static float textureVertices[] = {  
	0.0f,  0.0f,   /*左上角*/
        1f,  0.0f,      /*右上角*/        
        0.0f,  1f,      /*左下角*/
        1f,  1f,         /*右下角*/  
    };
```

https://blog.csdn.net/tomorrow_opal/article/details/70143093


## 4. 世界坐标系（World Coordinates）

右手笛卡尔坐标系统。

在OpenGL中，世界坐标系是以屏幕中心为原点(0, 0, 0)，且是始终不变的。x轴正方向为屏幕从左向右，y轴正方向为屏幕从下向上，z轴正方向为屏幕从里向外。长度单位这样来定：窗口范围按此单位恰好是(-1,-1)到(1,1)，即屏幕左下角坐标为（-1，-1），右上角 坐标为（1,1）。

![img]({{ site.url }}{{ site.baseurl }}/images/android-coordinate-system.assets/android-cs-world.png)



进行旋转操作时需要指定的角度θ的方向则由右手法则来决定，即右手握拳，大拇指直向某个坐标轴的正方向，那么其余四指指向的方向即为该坐标轴上的θ角的正方向（即θ角增加的方向），在上图中用圆弧形箭头标出。

坐标变换矩阵栈：

用来存储一系列的变换矩阵，栈顶就是当前坐标的变换矩阵，进入OpenGL管道的每个坐标(齐次坐标)都会先乘上这个矩阵，结果才是对应点在场景中的世界坐标。OpenGL中的坐标变换都是通过矩阵运算完成的。 
如图： 

![img]({{ site.url }}{{ site.baseurl }}/images/android-coordinate-system.assets/cs-transform.png)

对象坐标系（乘以模型视图矩阵）--->眼睛坐标系（乘以投影矩阵）--->裁剪坐标系（除以w）--->标准设备坐标系（视口变换）--->设备坐标系

## 5. 对象/模型/局部/绘图坐标系(object coordinate)

这是对象在被应用任何变换之前的初始位置和方向所在的坐标系，也就是当前绘图坐标系。该坐标系不是固定的，且仅对该对象适用。在默认情况下，该坐标系与世界坐标系重合。这里能用到的函数有glTranslatef()，glScalef(), glRotatef()，当用这些函数对当前绘图坐标系进行平移、伸缩、旋转变换之后， 世界坐标系和当前绘图坐标系不再重合。改变以后，再用glVertex3f()等绘图函数绘图时，都是在当前绘图坐标系进行绘图，所有的函数参数也都是相对当前绘图坐标系来讲的。如图则是对物体进行变换后，对象坐标系与世界坐标系的相对位置。

![img]({{ site.url }}{{ site.baseurl }}/images/android-coordinate-system.assets/object-cs.png)

## 6. 眼/照相机坐标系（eye coordinate）

模型变换：对象坐标系-->世界坐标系

视图变换：世界坐标系-->眼睛坐标系

GL_MODELVIEW矩阵是模型变换和试图变换矩阵的组合（view*model）,因为没有单独的模型变换和视图变化，所以使用GL_MODELVIEW矩阵可以使对象直接从对象坐标系转换到眼睛坐标系。

为什么要转换到眼睛坐标系？

因为我们的观察位置没定，如果我们的眼睛（照相机）的位置不同，那么观察物体的角度则不同，看到的样子也不同，所有要有这一步，把场景与我们的观察位置对应起来。

默认情况下，眼睛坐标系与世界坐标系也是重合的。使用gluLookAt()则可以指定眼睛（相机）的位置和眼睛看的方向。该函数的原型如下：

```c
void gluLookAt(GLdouble eyex, GLdouble eyey, GLdouble eyez, 
                        GLdouble centerx, GLdouble centery, GLdouble centerz,
                        GLdouble upx, GLdouble upy, GLdouble upz);
```

函数参数中,点(eyex, eyey, eyez)代表眼睛所在位置；
点(centerx, centery,centerz)代表眼睛看向的位置；
向量(upx, upy, upz)代表视线向上方向,其中视点和物体的连线与视线向上方向要保持。

> 注意：
>
> 使用glTranslatef()，glScalef(), glRotatef()这些函数是对对象坐标系进行变动；使用void gluLookAt()是对眼坐标系进行变动，两者可以达到相同的变换效果。相当于对象不动移动相机，和相机不动移动对象。比如场景向x轴正方向移动1个单位(相机不动)，相当于相机向x轴负方向移动一个单位(对象不动)，glTranslatef(1.0, 0.0, 0.0) <=> gluLookAt(-1.0, 0.0, 0.0, ..., ... )。



## 7. 裁剪坐标系（clip coordinate）

眼坐标到裁剪坐标是通过投影完成的。眼坐标通过乘以GL_PROJECTION矩阵变成了裁剪坐标。
投影分为透视投影（perspective projection）和正交投影（orthographic projection）

### 透视投影

类似日常生活看到的场景，远大近小。透视投影函数有两个：gluPerspective()和glFrustum()

```c
void glFrustum(GLdouble left, GLdouble right,
 　　　　　 GLdouble bottom, GLdouble top, 
 　　　　　 GLdouble near, GLdouble far)
void gluPerspective(GLdouble fovy,  GLdouble aspect,
　　　　　　　   GLdouble near, GLdouble far) 
```

far, near是指近裁剪面（），远剪裁面离视点的距离(>0),fovy视角(通常为45)，aspect = w/h

这个投影矩阵将给定的平截头体范围映射到裁剪空间，除此之外还修改了每个顶点坐标的w值，从而使得离观察者越远的顶点坐标w分量越大。被变换到裁剪空间的坐标都会在-w到w的范围之间（任何大于这个范围的坐标都会被裁剪掉）。OpenGL要求所有可见的坐标都落在-1.0到1.0范围内，作为顶点着色器最后的输出，因此，一旦坐标在裁剪空间内之后，透视除法就会被应用到裁剪空间坐标上：

 out=(x/wy/wz/w)

顶点坐标的每个分量都会除以它的w分量，距离观察者越远顶点坐标就会越小。这是也是w分量非常重要的另一个原因，它能够帮助我们进行透视投影。

### 正交投影

```c
void glOrtho(GLdouble left, GLdouble right,
　　　　       GLdouble bottom, GLdouble top, 
　　　　       GLdouble near, GLdouble far);
```

把物体直接映射到屏幕上，不影响它的相对大小。也就是图像反映物体的实际大小。

## 8. 归一化设备坐标系(normalized device coordinate)

在裁剪坐标系下通过除以w分量得到，这个操作称为透视除法。得到的坐标值均为[-1,1]

Vclip=Mprojection⋅Mview⋅Mmodel⋅Vlocal。最后的顶点应该被赋值到顶点着色器中的gl_Position，OpenGL将会自动进行透视除法和裁剪。

## 9. 屏幕坐标（screen coordinate）

屏幕坐标的x轴向右为正，y轴向下为正，坐标原点位于窗口的左上角。是归一化设备坐标系通过视口变换得到（viewport）



## 10 . 几何变换

OpenGL中可以使用的几何变换有平移、旋转、缩放三种。

glTranslatef(x, y, z);

该函数可以实现平移变换，x、y、z为各坐标轴上的平移量。

glRotatef(θ, x, y, z);

该函数实现旋转变换。θ为旋转角度，x、y、z为旋转轴。旋转方向由右手法则决定（参见第一节“坐标系”）。

glScalef(x, y, z);

该函数实现缩放变换。x、y、z为各轴方向的扩大量。若为负值，则沿着坐标轴的反方向进行缩放。



## 参考

https://blog.csdn.net/chaoyu168/article/details/126904000
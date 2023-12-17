---
layout: post
title: matrix的pre， post， set
date: 2023-12-08 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android matrix
---


* content
{:toc}

---

## 0. 前言

Matrix包含一个3X3的矩阵,专门用于图像变换匹配



## 1. 相关矩阵操作

- translate，平移
- rotate，旋转
- scale，缩放
- skew，错切

每种操作都有3种方式

| 方式 |             注释 |
| ---- | ---------------: |
| pre  | 在队列最前面插入 |
| post | 在队列最后面追加 |
| set  | 先清空队列在添加 |

下面通过例子具体说明，具体例子下载地址：
[ImageingStudy/MatrixActivity.java at master · zhongjhATC/ImageingStudy (github.com)](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FzhongjhATC%2FImageingStudy%2Fblob%2Fmaster%2Fapp%2Fsrc%2Fmain%2Fjava%2Fcom%2Fzhongjh%2Fimageingstudy%2FMatrixActivity.java)



## 2. !!! 左乘右乘

1. post 相当于左乘，pre相当于右乘；
2. **post 和 pre 可以理解为针对矩阵自己**，post就是说矩阵自己放在 右侧，pre就是说矩阵放在左侧；
3. **set的相关操作，就是把前面的操作都清除了**；

https://blog.csdn.net/zzhou910/article/details/26379795



```java
matrix.postScale(0.5f, 0.5f);    //T1
matrix.preTranslate(-pivotX, -pivotY);  //T1*T2
matrix.postTranslate(pivotX, pivotY); 	//T3*T1*T2
```

1. 在当前的矩阵左边加一个Scale的操作，因为之前没有其的操作（reset了），那么当前的矩阵其实就只是 T1 了。
2. 在当前的矩阵（T1）执行**前**，先执行一个平移到原点的操作（preTranslate），其实也就是进行右乘（T1 * T2）。
3. 在当前的矩阵（T1 * T2）执行**后**，再执行一个从原点平移到中心点的操作（postTranslate），也就是进行一个左乘（T3 * T1 * T2）。



> **知识小讲堂**
>
> 在Android中，有关矩阵的操作都是成对的，比如preTranslate(float dx, float dy)和postTranslate(float dx, float dy)，通过看api的介绍，如果原矩阵为M，那么pre表示的是左乘，post表示右乘：
>
> ```java
> preTranslate : M' = M * T(dx, dy) // 左乘
> postTranslate: M' = T(dx, dy) * M // 右乘
> ```
>
> 因为矩阵的变换是顺序执行的，所以在平时最常用的应该是pre左乘，所有的变换操作都依次执行，比如canvas常用的translate等变换方法其实就是左乘。右乘其实就是在所有操作之前增加一步操作，合理的运用右乘可以方便代码的编写。
>
> https://www.cnblogs.com/itlgl/p/10419252.html



## 3. !!! 理解

```
matrix = new Matrix();
matrix.setTranslate(300, 0);
matrix.preTranslate(100, 0);
matrix.postTranslate(50, 0);
matrix.preTranslate(-50, 0);
matrix.preTranslate(10, 0);
matrix.postTranslate(20, 0);
```



```java
0、初始的矩阵为单位矩阵I
1、setTranslate(300, 0)-> 矩阵变为：T1
2、matrix.preTranslate(100, 0)->矩阵变为：T1 * T2
3、matrix.postTranslate(50, 0)->矩阵变为：T3 * T1 * T2
4、matrix.preTranslate(-50, 0)->矩阵变为：T3 * T1 * T2 * T4
5、matrix.preTranslate(10, 0)->矩阵变为：T3 * T1 * T2 * T4 * T5
6、matrix.postTranslate(20, 0)->矩阵变为：T6 * T3 * T1 * T2 * T4 * T5


// 最终展开：(T6) * (T3) * (T1) * (T2) * (T4) * (T5)
// 从知识理论角度，可以如下理解
// 注意，这个代码对应是从右边开始，往左边走
// (1) pre是从最后一个开始执行, 5,4,2
// (2) set
// (3) post 是 按顺序执行，就是 3，6
```





## 4. 举例

### 4.1 初始

如图，每条线的间距是50dp，

![img]({{ site.url }}{{ site.baseurl }}/images/android-matrix.assets/init.png)

### 4.2 先pre-后post

```java
matrix.preScale(2,2); // T1
matrix.postTranslate(100, 100); // T2*T1
```

![img]({{ site.url }}{{ site.baseurl }}/images/android-matrix.assets/pre-post.png)



步骤：

- pre是放大，先放大两倍
- post是移动，后移动（100，100）



### 4.3 先post-后pre

```java
matrix.postScale(2,2); // T1
matrix.preTranslate(100, 100); //T1*T2
```

![img]({{ site.url }}{{ site.baseurl }}/images/android-matrix.assets/post-pre.png)

步骤：

- pre 先移动,先平移到（100,100）

- post是放大，再放大一倍。 


> 跟上面例子一样数值。为啥差距这么大，原因是执行顺序不同导致的，因为先移动到（100，100）后，再进行放大是从整个移动后的画布放大的。所以到达了（200,200）坐标处。



### 4.4 多次转换

```java
matrix.preScale(2f, 2f);
matrix.preTranslate(50, 50);
matrix.postScale(0.5f, 0.5f);
matrix.postTranslate(10, 10);
```

![img]({{ site.url }}{{ site.baseurl }}/images/android-matrix.assets/multi.png)

步骤：

- `preTranslate(50, 50)` ， 移动

-  `preScale(2f,2f)` ，放大2倍；

- `postScale(0.5f, 0.5f)`，缩小2倍；

- `postTranslate(10, 10)`，移动；

> 为什么先执行preTranslate再执行preScale呢，因为是pre是执行在队列最前面。



### 4.5 !!! setScale

```java
matrix.postTranslate(50, 50); 
matrix.preScale(0.5f, 0.5f);
matrix.setScale(1f, 1f); // 前面两句执行是没有意义的

matrix.postScale(3f, 3f);
matrix.preTranslate(10, 10);
```

![img]({{ site.url }}{{ site.baseurl }}/images/android-matrix.assets/set.png)

> 注意第三句代码是setScale，也就是说，不管前面执行多少句转换代码都不起作用了。

- `preTranslate(10, 10)` 
- `setScale(1f, 1f)` 
- `postScale(3f, 3f)`

> 注意，pre仍然是在set队列前面



### 4.6 !!! 负数转换

```java
postScale(-1, -1);
postTranslate(100,100);
```

![img]({{ site.url }}{{ site.baseurl }}/images/android-matrix.assets/mirror.png)

这是一种特殊的缩放变换。X为负数，则表示以X轴翻转后，再进行缩放。
注意： 没有Translate，只有matrix.setScale(-1,1)。 图像会翻转到X轴返方向。会看不到图像。
Translate 最好用 postTranslate 。 如果用了setScale 和 preTranslate ，是不会看到图像的。
理解：因为是preTranslate，会将图像先平移，距离X轴一段距离。然后再将图像沿X轴翻转。然后再放大（整体放大，包括刚才的平移的距离。）导致图像离X的负半轴更远。

## 5. Matrix.mapRect()理解

有时候不针对View或者ImageView进行整个的Matrix处理，可能会对View里面的某块RectF进行处理，那么就需要这个了。

```java
    public void scaleBitmap(View view) {
        RectF r = new RectF(0, 0, imageView.getWidth(), imageView.getHeight());
        Log.d("test", "-r.left = " + r.left + ", right = " + r.right + ", top = "
                + r.top + ", bottom = " + r.bottom);
        Matrix matrix = new Matrix();
        matrix.setScale(2,3);
        matrix.mapRect(r);
        // 可以看到宽度涨了2倍，高度涨了三倍
        Log.d("test", "-r.left = " + r.left + ", right = " + r.right + ", top = "
                + r.top + ", bottom = " + r.bottom);
        imageView.setImageMatrix(matrix);
    }
```

上面这段代码log如下：

```less
-r.left = 0, right = 100.0, top = 0.0, bottom = 100.0
-r.left = 0, right = 200.0, top = 0.0, bottom = 300.0
```

所以mapRect是单独对RectF的坐标点进行矩阵变换的



## 6. 相关 api

| Constants |            |
| :-------- | ---------- |
| `int`     | `MPERSP_0` |
| `int`     | `MPERSP_1` |
| `int`     | `MPERSP_2` |
| `int`     | `MSCALE_X` |
| `int`     | `MSCALE_Y` |
| `int`     | `MSKEW_X`  |
| `int`     | `MSKEW_Y`  |
| `int`     | `MTRANS_X` |
| `int`     | `MTRANS_Y` |



| Public methods |                                                              |
| :------------- | ------------------------------------------------------------ |
| `boolean`      | `equals(Object obj) `如果obj是矩阵并且其值等于我们的值，则返回true。 |
| `void`         | `getValues(float[] values) `将矩阵中的9个值复制到数组中。    |
| `int`          | `hashCode() `返回对象的哈希码值。                            |
| `boolean`      | `invert(Matrix inverse) `如果这个矩阵可以反转，则返回true，如果inverse不为null，则将inverse设置为该矩阵的逆矩阵。 |
| `boolean`      | `isAffine() `获取这个矩阵是否仿射。                          |
| `boolean`      | `isIdentity() `如果矩阵是标识，则返回true。                  |
| `void`         | `mapPoints(float[] dst, int dstIndex, float[] src, int srcIndex, int pointCount) `将此矩阵应用于由src指定的2D点阵列，并将变换后的点写入由dst指定的点阵列中。 |
| `void`         | `mapPoints(float[] dst, float[] src) `将此矩阵应用于由src指定的2D点阵列，并将变换后的点写入由dst指定的点阵列中。 |
| `void`         | `mapPoints(float[] pts) `将此矩阵应用于2D点阵列，并将变换后的点写回到数组中 |
| `float`        | `mapRadius(float radius) `在矩阵映射后返回圆的平均半径。     |
| `boolean`      | `mapRect(RectF rect) `将这个矩阵应用到矩形中，然后将变换后的矩形写回到矩形中。 |
| `boolean`      | `mapRect(RectF dst, RectF src) `将此矩阵应用于src矩形，并将转换后的矩形写入dst。 |
| `void`         | `mapVectors(float[] vecs) `将这个矩阵应用到2D矢量阵列中，并将变换后的矢量写回到数组中。 |
| `void`         | `mapVectors(float[] dst, int dstIndex, float[] src, int srcIndex, int vectorCount) `将此矩阵应用于由src指定的2D矢量阵列，并将变换后的矢量写入由dst指定的矢量阵列中。 |
| `void`         | `mapVectors(float[] dst, float[] src) `将此矩阵应用于由src指定的2D矢量阵列，并将变换后的矢量写入由dst指定的矢量阵列中。 |
| `boolean`      | `postConcat(Matrix other) `用指定的矩阵后验矩阵。            |
| `boolean`      | `postRotate(float degrees, float px, float py) `用指定的旋转后置矩阵。 |
| `boolean`      | `postRotate(float degrees) `用指定的旋转后置矩阵。           |
| `boolean`      | `postScale(float sx, float sy, float px, float py) `使用指定的比例后矩阵。 |
| `boolean`      | `postScale(float sx, float sy) `使用指定的比例后矩阵。       |
| `boolean`      | `postSkew(float kx, float ky) `用指定的偏斜后矩阵。          |
| `boolean`      | `postSkew(float kx, float ky, float px, float py) `用指定的偏斜后矩阵。 |
| `boolean`      | `postTranslate(float dx, float dy) `用指定的翻译后置矩阵。   |
| `boolean`      | `preConcat(Matrix other) `用指定的矩阵预处理矩阵。           |
| `boolean`      | `preRotate(float degrees) `用指定的旋转预处理矩阵。          |
| `boolean`      | `preRotate(float degrees, float px, float py) `用指定的旋转预处理矩阵。 |
| `boolean`      | `preScale(float sx, float sy) `使用指定的比例预处理矩阵。    |
| `boolean`      | `preScale(float sx, float sy, float px, float py) `使用指定的比例预处理矩阵。 |
| `boolean`      | `preSkew(float kx, float ky) `用指定的偏斜预处理矩阵。       |
| `boolean`      | `preSkew(float kx, float ky, float px, float py) `用指定的偏斜预处理矩阵。 |
| `boolean`      | `preTranslate(float dx, float dy) `用指定的翻译预处理矩阵。  |
| `boolean`      | `rectStaysRect() `如果将矩形映射到另一个矩形，则返回true。   |
| `void`         | `reset() `将矩阵设置为标识                                   |
| `void`         | `set(Matrix src) `（深）将src矩阵复制到此矩阵中。            |
| `boolean`      | `setConcat(Matrix a, Matrix b) `将矩阵设置为两个指定矩阵的连接并返回true。 |
| `boolean`      | `setPolyToPoly(float[] src, int srcIndex, float[] dst, int dstIndex, int pointCount) `设置矩阵，使得指定的src点将映射到指定的dst点。 |
| `boolean`      | `setRectToRect(RectF src, RectF dst, Matrix.ScaleToFit stf) `将矩阵设置为比例并转换将源矩形映射到目标矩形的值，如果可以表示结果，则返回true。 |
| `void`         | `setRotate(float degrees, float px, float py) `将矩阵设置为以指定的度数旋转，其中的轴心点位于（px，py）处。 |
| `void`         | `setRotate(float degrees) `将矩阵设置为以（0,0）为中心旋转指定的度数。 |
| `void`         | `setScale(float sx, float sy) `将矩阵设置为通过sx和sy进行缩放。 |
| `void`         | `setScale(float sx, float sy, float px, float py) `将矩阵设置为通过sx和sy进行缩放，其中的轴心点位于（px，py）处。 |
| `void`         | `setSinCos(float sinValue, float cosValue, float px, float py) `将矩阵设置为以指定的正弦和余弦值旋转，其中的轴心点位于（px，py）处。 |
| `void`         | `setSinCos(float sinValue, float cosValue) `将矩阵设置为按指定的正弦和余弦值旋转。 |
| `void`         | `setSkew(float kx, float ky) `将矩阵设置为通过sx和sy偏斜。   |
| `void`         | `setSkew(float kx, float ky, float px, float py) `将矩阵设置为通过sx和sy偏斜，其中的轴心点位于（px，py）处。 |
| `void`         | `setTranslate(float dx, float dy) `将矩阵设置为通过（dx，dy）进行平移。 |
| `void`         | `setValues(float[] values) `将数组中的9个值复制到矩阵中。    |
| `String`       | `toShortString() `                                           |
| `String`       | `toString() `返回对象的字符串表示形式。                      |



## 参考

1. [如何计算矩阵乘法](https://links.jianshu.com/go?to=https%3A%2F%2Fzh.wikihow.com%2F%E8%AE%A1%E7%AE%97%E7%9F%A9%E9%98%B5%E4%B9%98%E6%B3%95)
2. [android matrix 最全方法详解与进阶（完整篇）](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fcquwentao%2Farticle%2Fdetails%2F51445269)
3. [Android Matrix 最全方法详解与进阶](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fgb702250823%2Farticle%2Fdetails%2F53526149)
4. [1-4 Canvas 对绘制的辅助 clipXXX() 和 Matrix](https://links.jianshu.com/go?to=http%3A%2F%2Fhencoder.com%2Fui-1-4%2F)
5. [Android中的Matrix(矩阵) - itlgl - 博客园 (cnblogs.com)](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fitlgl%2Fp%2F10419252.html)
5. https://www.jianshu.com/p/3be6c111b735


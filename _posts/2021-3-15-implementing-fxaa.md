---
layout: post
title:  "实现FXAA（翻译）"
categories: Unity
tags: 游戏 Unity 表格 配置
author: zack.zhang
mathjax: true
---

* content
{:toc}

在计算机屏幕上渲染3D场景时，可能会发生走样。由于每个像素只能属于覆盖其屏幕区域的特定对象，并从里面得到一种独特的颜色，这种对象的边缘就会产生锯齿状的效果。当有细线(如线框)显示时，也可以看到同样的效果。

![dragon1](https://zd304.github.io/assets/img/fxaa/dragon1.png)<br/>

<!-- more -->

## 对抗走样

已经有很多抗锯齿技术（anti-aliasing techniques）被开发出来，用来缓解这种视觉上的瑕疵。有些依赖于渲染更大分辨率的图片，以便在下采样（downsampling）和在屏幕上显示结果之前从场景中获得更多的细节，例如超级采样抗锯齿(SSAA)。虽然开发了各种改进方案来降低性能和内存成本，但仍然受到一些限制。多重采样抗锯齿(MSAA)就是其中之一，在OpenGL中非常容易实现，但在现代的延迟渲染管道中却很难使用（光照计算是在特定的渲染通道中执行的）。

还有其他的一些技术，使用先前帧的信息来增强当前帧的质量，这类算法被称为时间反走样（temporal anti-aliasing）。其他一些方法是应用于最终渲染图像的后期处理效果（post-processing effects）。其中，子像素形态反走样(SMAA)是最先进的算法之一，但实现起来相当复杂。2009年，来自Nvidia的Timothy Lottes描述了一种更为简单但仍然非常有效的算法，并迅速应用于许多游戏中：快速近似反走样(fast approximate anti-aliasing)，即FXAA。

## 进入FXAA

FXAA很容易添加到现有的渲染器中：它作为最终的渲染通道，只接受渲染的图像作为输入，并输出反走样版本。主要的想法是检测渲染图片的边缘和平滑这些边缘。这种方法是快速、有效的，但会模糊纹理的细节。我将尝试一步一步地解释这个算法，但首先是一个示例。



这是一个特写：当反锯齿被启用时，所有的边缘都被平滑，以及一些纹理细节（特别是在龙皮肤上）。

![comparison1](https://zd304.github.io/assets/img/fxaa/comparison1.png)<br/>

## 先决条件

为了便于解释，我将假设首先将整个场景渲染成一个纹理图像，其分辨率与窗口相同。然后渲染一个覆盖整个窗口的矩形来显示这个纹理。对于这个矩形的每个像素，FXAA算法在所谓的片段着色器(fragment shader)中执行，这是一个针对每个像素在GPU上执行的小程序。

### 亮度（Luma）

在FXAA着色器中的大多数计算，将依赖于从纹理中读取的像素的亮度，表示为0.0到1.0之间的灰色级别。为此将使用亮度，由公式L = 0.299 \* R + 0.587 \* G + 0.114 \* B定义。

它是红、绿、蓝成分的加权总和，考虑到我们眼睛对每个波长范围的敏感性。此外，我们将在感知空间（而不是线性空间）中使用它的值，并且我们用平方根近似反伽玛变换。

```glsl
float rgb2luma(vec3 rgb) {
	return sqrt(dot(rgb, vec3(0.299, 0.587, 0.114)));
}
```

### 纹理过滤（Texture filtering）

为了在OpenGL中读取纹理，我们通常使用[0,1]中表示为浮点数的UV坐标。但是纹理在每个维度上是由有限的像素组成的，每个像素都有一个恒定的颜色；如果一个人试图在两个像素之间的UV坐标上读取颜色，会发生什么情况？有两种主要的方法来处理这个问题：

* 确定最近的像素，并使用其颜色。这是最近邻算法，如下图左边所示。

* 在四个最接近的像素颜色之间进行线性插值，以每个像素的距离为权重。右边显示的是双线性过滤。

![comparison2](https://zd304.github.io/assets/img/fxaa/comparison2.png)<br/>

注意，当在相同大小的窗口中显示渲染的场景纹理时，使用的过滤方法不会修改它的外观。它只会影响在FXAA计算中读取的值，我们将选择双线性滤波来得到自由的插值。

## 一步一步计算

唯一需要的输入是screenTexture，片段的UV坐标In.uv，和窗口尺寸的倒数inverseScreenSize (1.0/width, 1.0/height)；唯一的输出是RGB颜色矢量fragColor。

我将使用下面的图像作为一个简单的例子：黑色和白色，8x5像素的网格，夹在其边缘；我们将关注红色描边的像素。

![exp1](https://zd304.github.io/assets/img/fxaa/exp1.png)<br/>

### 检测在何处应用AA

首先，需要检测边缘：为了做到这一点，计算当前片段和它的四个直接相邻像素的亮度。提取最小和最大亮度，并将两者的差值作为局部对比度值。在边缘处的对比是强烈的，因为边缘的颜色变化很明显。因此，如果对比度低于与最大亮度成比例的阈值，则不会执行反走样。此外，在黑暗区域走样是不太明显的，因此，如果对比度低于一个绝对阈值，我们也不执行反走样。在这些情况下，把从贴图的当前像素读出来的颜色输出出来。

```glsl
vec3 colorCenter = texture(screenTexture, In.uv).rgb;

// 当前像素的亮度
float lumaCenter = rgb2luma(colorCenter);

// 跟当前像素直接相邻的四个像素的亮度
float lumaDown = rgb2luma(textureOffset(screenTexture, In.uv, ivec2(0,-1)).rgb);
float lumaUp = rgb2luma(textureOffset(screenTexture, In.uv, ivec2(0,1)).rgb);
float lumaLeft = rgb2luma(textureOffset(screenTexture, In.uv, ivec2(-1,0)).rgb);
float lumaRight = rgb2luma(textureOffset(screenTexture, In.uv, ivec2(1,0)).rgb);

// 找到当前像素周围最大和最小的亮度
float lumaMin = min(lumaCenter, min(min(lumaDown, lumaUp), min(lumaLeft, lumaRight)));
float lumaMax = max(lumaCenter, max(max(lumaDown, lumaUp), max(lumaLeft, lumaRight)));

// 计算最大亮度和最小亮度的差
float lumaRange = lumaMax - lumaMin;

// 如果亮度变化低于阈值（或者在一个非常黑暗的区域），像素就不在边缘上，不执行任何AA。
if(lumaRange < max(EDGE_THRESHOLD_MIN,lumaMax * EDGE_THRESHOLD_MAX)) {
    fragColor = colorCenter;
    return;
}
```

这两个全大写常量的推荐值为EDGE_THRESHOLD_MIN = 0.0312和EDGE_THRESHOLD_MAX = 0.125。

对于我们的示例像素，最小值是0，最大值是1，因此范围是1，由于我们有1.0 > max(1 \* 0.125, 0.0312)，我们将执行AA。

### 估计梯度和选择边的方向

然后，对于检测到的边缘的每个像素，我们检查边缘是垂直的还是水平的。为此，使用中央亮度和8个相邻像素的亮度计算一系列的局部差值，在水平和垂直方向都使用以下公式:

* 水平方向梯度：[(upleft - left) - (left - downleft)] + 2 \* [(up - center) - (center - down)] + [(upright - right) - (right - downright)]

* 垂直方向梯度：[(upright - up) - (up - upleft)] + 2 \* [(right - center) - (center - left)] + [(downright - down) - (down - downleft)]

以上两个量中最大的一个将得出边的主要方向。

```glsl
// 查询剩余的4个角的亮度。
float lumaDownLeft = rgb2luma(textureOffset(screenTexture,In.uv,ivec2(-1,-1)).rgb);
float lumaUpRight = rgb2luma(textureOffset(screenTexture,In.uv,ivec2(1,1)).rgb);
float lumaUpLeft = rgb2luma(textureOffset(screenTexture,In.uv,ivec2(-1,1)).rgb);
float lumaDownRight = rgb2luma(textureOffset(screenTexture,In.uv,ivec2(1,-1)).rgb);

// 合并四条边的亮度（使用中间变量为未来计算相同的值）。
float lumaDownUp = lumaDown + lumaUp;
float lumaLeftRight = lumaLeft + lumaRight;

// 角落同样的处理
float lumaLeftCorners = lumaDownLeft + lumaUpLeft;
float lumaDownCorners = lumaDownLeft + lumaDownRight;
float lumaRightCorners = lumaDownRight + lumaUpRight;
float lumaUpCorners = lumaUpRight + lumaUpLeft;

// 沿水平和垂直轴向计算梯度的估计值。
float edgeHorizontal =  abs(-2.0 * lumaLeft + lumaLeftCorners)  + abs(-2.0 * lumaCenter + lumaDownUp ) * 2.0    + abs(-2.0 * lumaRight + lumaRightCorners);
float edgeVertical =    abs(-2.0 * lumaUp + lumaUpCorners)      + abs(-2.0 * lumaCenter + lumaLeftRight) * 2.0  + abs(-2.0 * lumaDown + lumaDownCorners);

// 边是水平的还是垂直的？
bool isHorizontal = (edgeHorizontal >= edgeVertical);
```

如我们的例子，可得：

* 横向梯度 = (-2 \* 0 + 0 + 1) + 2 \* (-2 \* 0 + 0 + 1) + (-2 \* 0 + 1 + 0) = 4

* 纵向梯度 = (-2 \* 0 + 0 + 0) + 2 \* (-2 \* 1 + 1 + 1) + (-2 \* 0 + 0 + 0) = 0

因此，边是水平的。

### 选择边的方向

当前像素不一定恰好位于边上。因此，下一步是确定与边方向正交的哪个方向是实边边界。计算当前像素每边的梯度，其最陡的位置可能位于边的边界上。

```glsl
// 在与边相反的方向上选择两个相邻的纹理亮度。
float luma1 = isHorizontal ? lumaDown : lumaLeft;
float luma2 = isHorizontal ? lumaUp : lumaRight;
// 计算这个方向的梯度。
float gradient1 = luma1 - lumaCenter;
float gradient2 = luma2 - lumaCenter;

// 哪个方向最陡？
bool is1Steepest = abs(gradient1) >= abs(gradient2);

// 对应方向的梯度，归一化。
float gradientScaled = 0.25*max(abs(gradient1),abs(gradient2));
```

在我们的例子中，我们有gradient1 = 0 - 0 = 0和gradient2 = 1 - 0 = 1，因此向上的邻居的变化更强，并且gradientScaled = 0.25。

最后，我们向这个方向移动半个像素，并计算此时的平均亮度。

```glsl
// 根据边的方向选择步长（一个像素）。
float stepLength = isHorizontal ? inverseScreenSize.y : inverseScreenSize.x;

// 在正确方向上的平均亮度。
float lumaLocalAverage = 0.0;

if(is1Steepest){
    // 转换方向
    stepLength = - stepLength;
    lumaLocalAverage = 0.5*(luma1 + lumaCenter);
} else {
    lumaLocalAverage = 0.5*(luma2 + lumaCenter);
}

// 将UV向正确的方向移动半像素。
vec2 currentUv = In.uv;
if(isHorizontal){
    currentUv.y += stepLength * 0.5;
} else {
    currentUv.x += stepLength * 0.5;
}
```

对于我们的像素，平均局部亮度是0.5\* (1 + 0) = 0.5，并且沿着Y轴的正方向偏移0.5的偏移量。

![exp3](https://zd304.github.io/assets/img/fxaa/exp3.png)<br/>

## 引用

原文地址：http://blog.simonrodriguez.fr/articles/30-07-2016_implementing_fxaa.html
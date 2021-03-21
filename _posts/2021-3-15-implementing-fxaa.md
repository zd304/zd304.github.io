---
layout: post
title:  "实现FXAA（翻译）"
categories: 渲染
tags: 渲染 FXAA 反走样
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
float lumaDownLeft = rgb2luma(textureOffset(screenTexture, In.uv,ivec2(-1, -1)).rgb);
float lumaUpRight = rgb2luma(textureOffset(screenTexture, In.uv,ivec2(1, 1)).rgb);
float lumaUpLeft = rgb2luma(textureOffset(screenTexture, In.uv,ivec2(-1, 1)).rgb);
float lumaDownRight = rgb2luma(textureOffset(screenTexture, In.uv,ivec2(1, -1)).rgb);

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

### 选择边的取向

当前像素不一定恰好位于边上。因此，下一步是确定与边方向正交的哪个方向是“真正”边界。计算当前像素每边的梯度，其最陡的位置可能位于边的边界上。

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
float gradientScaled = 0.25 * max(abs(gradient1), abs(gradient2));
```

在我们的例子中，我们有gradient1 = 0 - 0 = 0和gradient2 = 1 - 0 = 1，因此向上的相邻像素的变化更强，并且gradientScaled = 0.25。

最后，我们向这个方向移动半个像素，并计算此时的局部平均亮度。

```glsl
// 根据边的方向选择步长（一个像素）。
float stepLength = isHorizontal ? inverseScreenSize.y : inverseScreenSize.x;

// 在正确方向上移动像素的亮度到中心像素亮度的局部平均亮度。
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

对于我们的像素，局部平均亮度是0.5\* (1 + 0) = 0.5，并且沿着Y轴的正方向偏移0.5的偏移量。

![exp3](https://zd304.github.io/assets/img/fxaa/exp3.png)<br/>

### 第一次迭代探测

下一步是沿着边的主轴进行探测。我们在两个方向上移动一个像素，在新坐标处查询亮度，计算亮度相对于上一步的局部平均亮度的变化。如果这个变化大于局部梯度，说明我们在这个方向已经到达边缘的末端，就停止。否则，将UV偏移量增加一个像素。

```glsl
// 在正确的方向上计算偏移量（每个迭代步骤）。
vec2 offset = isHorizontal ? vec2(inverseScreenSize.x,0.0) : vec2(0.0,inverseScreenSize.y);
// 计算uv以探索边缘的每一个方向。QUALITY值允许我们加快步伐。
vec2 uv1 = currentUv - offset;
vec2 uv2 = currentUv + offset;

// 读取探测段的两端亮度，并且计算亮度到局部平均亮度的
float lumaEnd1 = rgb2luma(texture(screenTexture,uv1).rgb);
float lumaEnd2 = rgb2luma(texture(screenTexture,uv2).rgb);
lumaEnd1 -= lumaLocalAverage;
lumaEnd2 -= lumaLocalAverage;

// 如果当前某一端的亮度差大于局部梯度，则说明已经达到边缘的一侧。
bool reached1 = abs(lumaEnd1) >= gradientScaled;
bool reached2 = abs(lumaEnd2) >= gradientScaled;
bool reachedBoth = reached1 && reached2;

// 如果没有到达边缘一侧，我们将继续朝着这个方向进行探测
if(!reached1){
    uv1 -= offset;
}
if(!reached2){
    uv2 += offset;
}
```

在这个例子中，我们得到lumaEnd1 = 0.5 - 0.5 = lumaEnd2 = 0.0 < gradientScaled（亮度是0.5，因为在读取纹理时使用了双线性插值），因此我们在两边都继续进行迭代。

![exp4](https://zd304.github.io/assets/img/fxaa/exp4.png)<br/>

### 继续迭代

我们继续迭代，直到到达边缘的两端，或者直到达到最大迭代次数(12)。为了加快速度，我们开始在第五次迭代后增加像素QUALITY(i): 1.5, 2.0, 2.0, 2.0, 2.0, 4.0, 8.0。

```glsl
// 如果两边都还没有到达，则继续探测。
if(!reachedBoth){

    for(int i = 2; i < ITERATIONS; i++){
        // 如果没有到达一侧，读取第一个方向的亮度，计算差值。
        if(!reached1){
            lumaEnd1 = rgb2luma(texture(screenTexture, uv1).rgb);
            lumaEnd1 = lumaEnd1 - lumaLocalAverage;
        }
        // 如果没有到达另一侧，读取相反方向的亮度，计算差值。
        if(!reached2){
            lumaEnd2 = rgb2luma(texture(screenTexture, uv2).rgb);
            lumaEnd2 = lumaEnd2 - lumaLocalAverage;
        }
		// 如果当前端的亮度差值大于局部梯度，说明我们已经达到了边的一侧。
        // If the luma deltas at the current extremities is larger than the local gradient, we have reached the side of the edge.
        reached1 = abs(lumaEnd1) >= gradientScaled;
        reached2 = abs(lumaEnd2) >= gradientScaled;
        reachedBoth = reached1 && reached2;

        // 如果这一侧没有到达，我们用变量QUALITY继续沿着这个方向探测。
        if(!reached1){
            uv1 -= offset * QUALITY(i);
        }
        if(!reached2){
            uv2 += offset * QUALITY(i);
        }

        // 如果两侧都已经到达了，则停止探测。
        if(reachedBoth){ break;}
    }
}
```

最好的情况下，lumaEnd1和lumaEnd2现在包含边缘末端亮度与局部平均亮度之间的差值，以及对应的UV坐标uv1和uv2。

在这个例子中，我们获得了lumaEnd1 = 1-0.5 = 0.5 >= gradientscaling，所以我们可以停止左侧的探索。在右边，我们必须再迭代两次以满足条件。

![exp7](https://zd304.github.io/assets/img/fxaa/exp7.png)<br/>

### 评估偏移

接下来，我们计算当前像素在两个方向上离端点的距离，并找到最近的那个距离值。UV偏移量是一个估算值，也就是最近距离值和边的总长度的比值。这个值将告诉我们当前像素是在边的中间，还是靠近某一个端点。越接近某一个端点，UV偏移量越大。

```glsl
// 计算当前像素到每一个端点的距离。
float distance1 = isHorizontal ? (In.uv.x - uv1.x) : (In.uv.y - uv1.y);
float distance2 = isHorizontal ? (uv2.x - In.uv.x) : (uv2.y - In.uv.y);

// 到哪个方向的端点最近？
bool isDirection1 = distance1 < distance2;
float distanceFinal = min(distance1, distance2);

// 边的长度。
float edgeThickness = (distance1 + distance2);

// UV偏移：读取离边的端点最近的一侧。
// distanceFinal / edgeThickness < 0.5；由于越接近端点，UV偏移需要越大，则取负数；+0.5使取值范围为正数。
float pixelOffset = - distanceFinal / edgeThickness + 0.5;
```

对于示例像素，我们得到distance1 = 2, distance2 = 4，因此边缘的端点在左边的方向更近(direction1)，我们得到pixelOffset = - 2 / 6 + 0.5 = 0.1666。

还有一个附加检查，以确保在端点观察到的亮度变化与当前像素处的亮度一致。否则我们可能走得太远了，我们没有应用任何偏移量。

```glsl
// 是否中心亮度小于局部平局亮度？
bool isLumaCenterSmaller = lumaCenter < lumaLocalAverage;

// 如果中心的亮度比邻近的小，则两端的亮度差应均为正（变化相同）
// （在边缘较近的一边的方向。）
bool correctVariation = ((isDirection1 ? lumaEnd1 : lumaEnd2) < 0.0) != isLumaCenterSmaller;

// 如果亮度变化不正确，不要偏移。
float finalOffset = correctVariation ? pixelOffset : 0.0;
```

对于示例像素，中心亮度更小，并且结束亮度不是负的，我们确实有(0.5 < 0.0)!= isLumaCenterSmaller，偏移量计算是有效的。

### 子像素反走样

这个额外的计算步骤允许我们处理子像素走样，例如当细线在屏幕上出现锯齿时。在这些情况下，平均亮度是在3x3附近的像素计算的。用它减去中心亮度，并且除以第一步里计算出来的lumaRange，得到一个子像素的偏移。相对于整个相邻的范围，平均值和中心值之间的对比度差越小，区域就越均匀(即没有单个像素点)，偏移量也就越小。然后对这个偏移量进行细化，我们保留前一步和这一步中较大的偏移量。

```glsl
// 子像素转移
// 相邻的3x3像素的加权平均亮度。
float lumaAverage = (1.0/12.0) * (2.0 * (lumaDownUp + lumaLeftRight) + lumaLeftCorners + lumaRightCorners);
// 求全局平均亮度和中心亮度的差，和3x3相邻的lumaRange的比值。
float subPixelOffset1 = clamp(abs(lumaAverage - lumaCenter)/lumaRange,0.0,1.0);
float subPixelOffset2 = (-2.0 * subPixelOffset1 + 3.0) * subPixelOffset1 * subPixelOffset1;
// 计算基于此差值的子像素偏移量。
float subPixelOffsetFinal = subPixelOffset2 * subPixelOffset2 * SUBPIXEL_QUALITY;

// 从两个偏移量中选最大偏移量。
finalOffset = max(finalOffset,subPixelOffsetFinal);
```

其中SUBPIXEL_QUALITY = 0.75。

在这个例子中，lumaAverage = (1/12)(2\*(1+0+0+0)+1+1+0+0) = 4/12 = 0.333, subPixelOffset1 = 0.333-0.0/1.0 = 0.333，因此subPixelOffsetFinal = 0.75\*((-2\*0.333+3.0)\*(0.3333)^2)^2 = 0.0503，最大偏移量为0.1666。因此，没有检测或处理子像素走样。

### 最后的读取

最后，我们相应地在与边缘正交的方向偏移UV，并在纹理中最后读取一次。

```glsl
// 计算最后的UV坐标。
vec2 finalUv = In.uv;
if(isHorizontal){
    finalUv.y += finalOffset * stepLength;
} else {
    finalUv.x += finalOffset * stepLength;
}

// 在新的UV坐标上读取颜色，并使用它。
vec3 finalColor = texture(screenTexture,finalUv).rgb;
fragColor = finalColor;
```

![exp8](https://zd304.github.io/assets/img/fxaa/exp8.png)<br/>

对于所研究的像素，其强度为0.1666*1 +(1-0.1666)*0≈0.1666。

如果我们将这个方法应用到小图像的所有像素，我们得到以下值：

![exp9](https://zd304.github.io/assets/img/fxaa/exp9.png)<br/>

视觉效果如下：

![exp10](https://zd304.github.io/assets/img/fxaa/exp10.png)<br/>

这些像素的平滑程度取决于它们与边缘的相近程度，以及它们在边缘上的位置。由于这个例子过于简单，结果很难验证。在更复杂和明确的图像上，可以获得以下反走样效果：柔软但有效的边缘平滑，对纹理细节的影响最小，以及良好的时间一致性。（左图无AA，右图为FXAA，放大图片查看差异）

![comp2](https://zd304.github.io/assets/img/fxaa/comp2.png)<br/>

![comp3](https://zd304.github.io/assets/img/fxaa/comp3.png)<br/>

![comp4](https://zd304.github.io/assets/img/fxaa/comp4.png)<br/>

## 引用

原文地址：<a href="http://blog.simonrodriguez.fr/articles/30-07-2016_implementing_fxaa.html">《Implementing FXAA》</a>
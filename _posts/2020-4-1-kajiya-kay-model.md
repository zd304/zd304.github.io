---
layout: post
title:  "基于Kajiya-Kay模型的毛发渲染"
categories: 渲染
tags: Unity 渲染 各向异性 毛发
author: zack.zhang
mathjax: true
---

* content
{:toc}

Kajiya-Kay Shading Model（卡吉雅模型）是一种经典的各向异性的头发渲染着色模型，《神秘海域4（Uncharted 4）》中的角色头发渲染就是在卡吉雅模型的基础上实现的。1997年，因为对毛发渲染的研究，Jim Kajiya和Timothy Kay一起获得奥斯卡技术认证证书。
<!-- more -->

<a href="http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2012/10/Scheuermann_HairRendering.pdf">原文传送门</a>

## 头发渲染

大部分人头上都有头发，可见头发渲染是非常重要的。但是头发渲染同时也是困难的。

* 头发数量巨大，一个人的头上大约有100K-150K根头发
* 发型界存在着大量不同的发型
* 《最终幻想：灵魂深处》大约25%的渲染时间都花费在主角头发的渲染上。

因此，需要一种高效且真实的模型来模拟发丝质感.

## Blinn-Phong高光

Blinn-Phong高光是一种非常普遍的高光模型，渲染有时比Phong高光更柔和平滑，最关键的是处理速度快。在Phong模型中，必须计算V·R的值，其中R为反射光线的单位向量，V为视线方向的单位向量，但是在Blinn-Phong模型中，用N·H的值来取代V·R。其中H就是半角向量。

Blinn-Phong光照模型公式：

```glsl
half3 Is = Ks * lightColor * pow(dot(N, H), shininess);
```

这个光照模型，可以产生出一片圆形或者椭圆形的高光，但是无法模拟头发反射出“天使环”。因此该模型不能直接用于头发渲染，需要在此基础上改进。

这里解释一下什么是“天使环”。“天使环”是对各项异性高光的一种俗称，因为各项异性高光的形状通常是一个环形高光，而且就出现在头顶，类似于西方神话里天使头顶的光环，所以把各向异性高光形象地称作“天使环”。

![angle_ring](https://zd304.github.io/assets/img/angel_ring.png)<br/>

## 各向异性材质

各向异性光照产生的原理，简单来说，就是某些材质上的围观细丝反射光线形成的，比如头发、光盘等。

由于头发是由无数发丝组成，每一根发丝可以看做一根非常细长的圆柱体，那么头发整体的高光就是由这每一根圆柱形发丝的高光组成的。

为了表现每一根头发的高光我们需要做以下工作。

### 1. 贴图

![anisotropic](https://zd304.github.io/assets/img/hair_shift.jpg)<br/>

头发颜色贴图（Base Map）、法线贴图（Normal Map），以及偏移贴图（Shift Map，如上图）。

### 2. 高光

基于Blinn-Phong高光的思想，同样使用的是半角向量，代码如下。

```glsl
float StrandSpecular(float3 T, float3 V, float L, float exponent)
{
	float3 H = normalize(L + V);
	float dotTH = dot(T, H);
	float sinTH = sqrt(1.0 - dotTH * dotTH);
	float dirAtten = smoothstep(-1.0, 0.0, dot(T, H));
	
	return dirAtten * pow(sinTH, exponent);
}
```

#### 参数解释

参数V指的是视线方向，参数L是灯光方向，参数exponent也就是Blinn-Phong高光中的shininess。

重点要解释的是参数T，指的是模型切线（tangent）。

![hair_tbn](https://zd304.github.io/assets/img/hair_tbn.png)<br/>

上图展示了一个表面的三个向量：N-法线-Normal，T-切线-Tangent，B-副切线（副法线）-Bitangent（Binormal）。法线为垂直于表面的向量，切线为沿着<u>表面贴图UV坐标的V增长方向</u>的向量。

美术制作头发贴图的时候，最好<u>发丝方向</u>和<u>UV坐标中V增长方向</u>保持一致，因为**在高光计算过程中头发方向至关重要**。当然也可以不一致，但是需要多一张Flowmap贴图，美术可以使用<a href="https://krita.org/zh/">Krita</a>来制作Flowmap，该做法这里不赘述。

#### 算法推导

由于标准Blinn-Phong高光需要用到法线，然而每根圆柱体发丝的法线数量庞大，如下图所示。

![anisotropic](https://zd304.github.io/assets/img/anisotropic.png)<br/>

如果要用法线计算高光就需要细分发丝表面，**计算量巨大**。我们发现垂直于法线的另外一个向量是唯一的，也就是发丝延伸方向的向量，即**切线**。**过切线的起点并且与切线和光线共面**，可以**找到唯一的一条法线**，我们使用这条法线就可以计算出一个近似的高光。

通过方向光向量，和切线方向，可以确定一条法线，如下图所示。

![strand](https://zd304.github.io/assets/img/strand.png)<br/>

其中T为切线，L为灯光方向的反方向，通过T和L的平面可以确定一条法线N。N和L的夹角为θ，那么可得如下公式。

$$
N·L = cos(θ)
$$

又由于N垂直于T，所以公式形式可以改为如下所示。

$$
N·L = sin(\frac{\pi}{2} - \theta) = \sqrt{1 - (L·T)^2}
$$

如果按照上述方式找到一条法线会产生一个问题，**似乎高光位置跟视线（V）没有关系**，也就是视线移动，“天使环”不会跟着移动位置。为了解决这个问题，我们模仿Blinn-Phong**使用半角向量**，这样“天使环”位置就会同时受平行光和视线影响了。

同理可得如下公式，即Blinn-Phong高光的基底N·H。

$$
N·H = \sqrt{1 - (H·T)^2}
$$

代码表示如下。

```glsl
float sinTH = sqrt(1.0 - dotTH * dotTH);
```

由Blinn-Phong高光模型可以计算出头发高光，代码表示如下。

```glsl
Is = pow(sinTH, exponent);
```

从上述找一条具有代表性的法线来计算高光的过程中可以发现，卡吉雅高光模型的思想如下：

* 每条发丝单独生成一个高光光斑
* 每条发丝的高光光斑通过一根具有代表性的法线计算而来，而这根法线需要靠切线和半角向量来找到
* 千万条发丝在差不多位置的高光连成一片，就形成了“天使环”。

最后还有一个问题，代码里的dirAtten是什么？

dirAtten是对高光的一个衰减系数，当半角向量与切线向量的夹角过大时，高光亮度将会被衰减。

我们知道$$cos(\frac{\pi}{2}) = cos (\frac{3}{2}\pi) = 0$$，角度大于$$\frac{\pi}{2}$$并且小于$$\frac{3}{2}\pi$$的时候余弦值在-1到0之间。

![strand](https://zd304.github.io/assets/img/dirAtten.jpg)<br/>

如上图所示，如果半角向量H朝向红色平面以下，则高光亮度将会被减弱。

### 3. 高光偏移

如果根据上述公式，不经过任何偏移处理，我们会计算出一个非常规则的圆环型高光，如下图所示。

![bleach_sf](https://zd304.github.io/assets/img/bleach_sf.png)<br/>

但是由于发丝不是紧密的排布在同一个平面上的，现实生活中很难看到这种非常规则的圆环型高光，游戏中用得更多的其实是一种经过偏移计算后的W型高光，如下图所示。

![bleach_sfyy](https://zd304.github.io/assets/img/bleach_sfyy.png)<br/>

为了计算出这种W型高光，需要美术制作一张偏移贴图，也就是上文提到的Shift Map。

为了实现高光沿着发丝方向偏移，我们让切线沿着法线偏移即可。如下图所示，如果沿着法线的正方向偏移，得到新的切线T'，那么T'和H确定的新法线N'将更加偏向发根方向，也就是说高光将会更加靠近发根；反之高光更加靠近发梢。

![hair_shift](https://zd304.github.io/assets/img/hair_shift.png)<br/>

总结起来就是：

* 沿着法线正方向偏移切线，高光更加靠近发根
* 沿着法线负方向偏移切线，高光更加靠近发梢

那么为了取出的Shift Map值有正有负，代码可以如下实现。

```glsl
float shift = tex2D(tSpecShift, uv) - 0.5;
```

切线偏移方法代码可以如下实现。

```glsl
float3 ShiftTangent(float T, float N, float shift)
{
	float3 shiftedT = T + shift * N;
	return normalize(shiftedT);
}
```

## 总结

至此，Kajiya-Kay模型已经全部推导完毕，卡吉雅模型是一种经验模型，是对头发高光的一种拟合模型，具体代码请参考ATI的文档《Scheuermann_HairRendering.pdf》。

该渲染模型解决了以下问题：

* 头发各项异性高光，即“天使环”效果
* W型高光效果

因为其计算简单而广为传播和使用，同时基于该模型演变出很多头发高光的解决方案。由于Kajiya-Kay模型是肉眼观察出来的模型，同时也是能量不守恒的，因此《GDC2004头发渲染和着色》提出了另一种头发渲染方式，我们将在后续博客详细介绍，让头发渲染更加趋近于真实。
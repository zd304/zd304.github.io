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

Kajiya-Kay Shading Model（卡吉雅模型）是一种经典的各向异性的头发渲染着色模型，《神秘海域4（Uncharted 4）》中的角色头发渲染就是在卡吉雅模型的基础上实现的。1997年，因为对毛发渲染的研究，Jim Kajiya和Timothy Kay一起获得奥斯卡技术认证整数。
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
half3 specular = Ks * lightColor * pow(dot(N, H), shininess);
```

这个光照模型，可以反射出一片圆形高光，但是无法模拟头发反射出“天使环”高光。因此该模型不能直接用于头发渲染，需要在此基础上改进。

## 各向异性材质

各向异性光照产生的原理，简单来说，就是某些材质上的围观细丝反射光线形成的，比如头发、光盘等。

由于头发是由无数发丝组成，每一根发丝可以看做一根非常细长的圆柱体，那么头发整体的高光就是由这每一根圆柱形发丝的高光组成的。

为了表现每一根头发的高光我们需要做以下工作。

### 1. 贴图

![anisotropic](https://zd304.github.io/assets/img/hair_shift.jpg)<br/>

头发颜色贴图（Base Map）、法线贴图（Normal Map），以及偏移贴图（Shift Map，如上图）。

### 2. 高光

基于Blinn-Phong高光的思想，同样使用的是半角向量，代码如下。

```cg
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

![hair_tbn](https://zd304.github.io/assets/img/hair_tbn.jpg)<br/>

上图展示了一个表面的三个向量：N-法线-Normal，T-切线-Tangent，B-副切线（副法线）-Bitangent（Binormal）。法线为垂直于表面的向量，切线为沿着<u>表面贴图UV坐标的V增长方向</u>的向量。

因此美术制作头发贴图的时候，最好<u>发丝方向</u>和<u>UV坐标中V增长方向</u>保持一致。当然也可以不一致，但是需要多一张Flowmap贴图，美术可以使用<a href="https://krita.org/zh/">Krita</a>来制作Flowmap，该做法这里不赘述。

#### 算法推导

由于标准Blinn-Phong高光需要用到法线，然而每根圆柱体发丝的法线数量庞大，如下图所示。

![anisotropic](https://zd304.github.io/assets/img/anisotropic.png)<br/>

如果要用法线计算高光就需要细分发丝表面，计算量巨大。然而我们发现垂直于法线的另外一个向量却是唯一的，也就是发丝延伸方向的向量，即切线，于是我们可以使用切线向量来模拟计算高光。

通过方向光的方向向量，和切线方向，可以确定一条法线，如下图所示。

![strand](https://zd304.github.io/assets/img/strand.png)<br/>

其中T为切线，L为灯光方向的反方向，通过T和L的平面可以确定一条法线N。N和L的夹角为θ，那么$$ N·L = cos(θ) $$。
又由于N垂直于T，所以$$ N·L = sin(\frac{\pi}{2} - \theta) = \sqrt{1 - (L·T)^2}$$。


---
layout: post
title:  "基于Kajiya-Kay模型的毛发渲染"
categories: 渲染
tags: Unity 渲染 各向异性 毛发
author: zack.zhang
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

![anisotropic](https://zd304.github.io/assets/img/anisotropic.png)<br/>


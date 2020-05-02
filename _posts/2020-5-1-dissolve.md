---
layout: post
title:  Unity小工具：Unity溶解效果（Dissolve）
categories: Unity小工具
tags: 溶解 Dissolve Unity
author: zack.zhang
---

* content
{:toc}

溶解效果是一个很基础的效果，本文是对溶解效果的一个简单的总结，作为一个记录以达到技术积累的目的。同时会在GitHub实现一个小例子，来验证理论。

<!-- more -->

## 溶解效果

要实现溶解效果，最重要的就是让某些像素显示，某些像素消失，如下图所示。

![dissolve_effect](https://zd304.github.io/assets/img/dissolve_effect.jpg)<br/>

为了实现这种效果，我们需要标记哪些像素要显示，哪些像素要消失，于是使用一张噪声图来给每一个像素做标记（Mask）。如下图所示，这个噪声图就是在Photoshop里用“云彩”工具生成的。由于噪声图的随机性，使得像素的显示和消失显得非常不规则，也就满足了对不规则溶解效果的需求。

![mask2](https://zd304.github.io/assets/img/mask2.png)<br/>

在shader中“抠掉”一个像素，可以使用clip函数。在片段着色器中调用clip(x)即可丢弃这个像素，其中x为数字，当x小于0.0，该像素则会被丢弃。

在片段着色器中根据当前uv，采样噪声图，就可以获得一个遮罩值。通常的实现方式为clip(遮罩值 - 阈值)，这句代码表示当遮罩值小于阈值时，则当前像素会被丢弃。如果阈值会从0到1动态变化，那么在游戏里将会看到一个动态溶解的动画过程。

以上就是溶解的基本原理。

```glsl
half dissove = tex2D(_DissTex, i.uv).r;

clip(dissolve - _Clip);
```

## 定向溶解

有的时候，溶解需要有一定的方向性，比如纸张从左下角往右上角燃烧的效果。设clip = dissolve - \_Clip，那么我们可以在clip的基础上，给一个方向相关的因子，即clip = clip + worldFactor。其中worldFactor就是方向相关的因子。

这里给出一种方向相关因子的计算方法。以下为计算worldFactor的顶点着色器代码。

```glsl
float4 worldPos = mul(unity_ObjectToWorld, v.vertex);
float3 rootPos = float3(unity_ObjectToWorld[0].w, unity_ObjectToWorld[1].w, unity_ObjectToWorld[2].w);
float3 pos = worldPos.rgb - rootPos;
float posOffset = dot(normalize(_DissolveDir), pos);
o.worldFactor = posOffset;
```

其中rootPos为物体质心所在的坐标值，因为世界变换矩阵的w行向量即为世界位置（详细请搜索世界矩阵介绍和推导相关文章）。worldPos为当前顶点所处世界位置。那么worldPos - rootPos就可以求得顶点偏离模型质心的偏移向量，最后偏移向量在_DissolveDir方向上的投影长度就是我们所需要的方向因子，因为投影越长，那么在_DissolveDir方向上像素离质心越远。

最终在片段着色器里如下实现即可。

```glsl
fixed dissove = tex2D(_DissTex, i.uv).r;
dissove = (dissove - _Clip) + i.worldFactor * _WorldSpaceScale;
clip(dissove);
```

## 顶点溶解

有的时候，溶解需要从中心点向外溶解，或者从某一点开始向外溶解。这个就比较简单了，根据定向溶解的原理，最重要的是找到一个因子。顶点溶解的因子根据像素离中心点距离来求得即可，也就是使用distance函数就可以求得。代码如下。

```glsl
fixed dissove = tex2D(_DissTex, i.uv).r;
float dist = distance(_DissolveCenterUV, i.uv);
dissove = dissove + dist * _WorldSpaceScale;
```

## 总结

本文实现的效果如下。

![screenshot1](https://zd304.github.io/assets/img/dissolve_screenshot.gif)

该文章总结了以下知识点。

* 溶解基础原理

* 定向溶解和定点溶解算法

验证详见<a href="https://github.com/zd304/Dissolve">GitHub工程</a>。
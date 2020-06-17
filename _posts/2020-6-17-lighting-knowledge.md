---
layout: post
title:  照明基本知识
categories: 渲染
tags: 照明 光照度 光通量
author: zack.zhang
mathjax: true
---

* content
{:toc}

<!-- more -->

**立体角**，三维空间的角度，类比平面角。单位球面度(sr)。计算公式变量记作Ω。设面积为A，半径为r，则$$Ω =\frac{A}{r^2}$$。因为完整球体的表面积为$$4{\pi}r^2$$，所以球体的立体角为$$4{\pi}$$。

**辐射功率**，单位时间内，光源发射的总辐射能量，单位瓦特(W)。

**光通量**，人眼能感觉到的辐射功率。单位流明(lm)。计算公式变量记作Φ。由于人眼对不同波段的光视见率不同，所以虽然辐射功率相同的，但是人眼感觉的到的光的感觉不相同。例如，一只40W的白炽灯光通量约为460lm，一只40W的日光色荧光灯光通量约为2100lm。

**发光强度**（光强），光源给定方向单位立体角内的光通量。单位坎德拉(cd)。计算公式变量记作I。$$I =frac{Φ}{Ω}$$。故 1cd = 1lm/sr。

![lighting-knowledge](https://zd304.github.io/assets/img/lighting-knowledge.jpg)<br/>
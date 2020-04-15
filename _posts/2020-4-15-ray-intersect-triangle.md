---
layout: post
title:  "射线与三角面相交计算"
categories: 数学
tags: 射线 三角形 物理 碰撞
author: zack.zhang
mathjax: true
---

* content
{:toc}

<!-- more -->

![ray-triangle](https://zd304.github.io/assets/img/ray-triangle.png)<br/>

射线通常由起点和方向两部分数据组成，射线上的某个点表示为P<sub>ray</sub> = O + Dt。

设射线和三角面的交点为P，那么P = O + Dt = u(V<sub>1</sub> - V<sub>0</sub>) + v(V<sub>2</sub> - V<sub>0</sub>) + V<sub>0</sub>。

∴ u(V<sub>1</sub> - V<sub>0</sub>) + v(V<sub>2</sub> - V<sub>0</sub>) - Dt = O - V<sub>0</sub>。

设 E<sub>1</sub> = V<sub>1</sub> - V<sub>0</sub>，E<sub>2</sub> = V<sub>2</sub> - V<sub>0</sub>，T = O - V<sub>0</sub>。

可得 uE<sub>1</sub> + vE<sub>2</sub> - Dt = T。

再设 E<sub>1</sub>(x<sub>e1</sub>, y<sub>e1</sub>, z<sub>e1</sub>)，E<sub>2</sub>(x<sub>e2</sub>, y<sub>e2</sub>, z<sub>e2</sub>)，D(x<sub>d</sub>, y<sub>d</sub>, z<sub>d</sub>)，T(x<sub>t</sub>, y<sub>t</sub>, z<sub>t</sub>)。

那么联立做方程组：

$$\begin{cases}
x_{e1}u + x_{e2}v - x_d{t} = x_t\\
y_{e1}u + y_{e2}v - y_d{t} = y_t\\
z_{e1}u + z_{e2}v - z_d{t} = z_t\\
 \end{cases}
$$
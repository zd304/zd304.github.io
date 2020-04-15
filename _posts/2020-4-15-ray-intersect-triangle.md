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
ux_{e1} + vx_{e2} - x_d{t} = x_t\\
uy<sub>e1</sub> + vy<sub>e2</sub> - y<sub>d</sub>t = y<sub>t</sub>\\
uz<sub>e1</sub> + vz<sub>e2</sub> - z<sub>d</sub>t = z<sub>t</sub>\\
 \end{cases}
$$
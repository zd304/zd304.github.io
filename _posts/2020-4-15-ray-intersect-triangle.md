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

（下文中粗体代表向量）

![ray-triangle](https://zd304.github.io/assets/img/ray-triangle.png)<br/>

射线通常由起点和方向两部分数据组成，射线上的某个点表示为P<sub>ray</sub> = O + **D**t。

设射线和三角面的交点为P，那么P = O + **D**t = u(V<sub>1</sub> - V<sub>0</sub>) + v(V<sub>2</sub> - V<sub>0</sub>) + V<sub>0</sub>。

∴ u(V<sub>1</sub> - V<sub>0</sub>) + v(V<sub>2</sub> - V<sub>0</sub>) - **D**t = O - V<sub>0</sub>。

设 **E<sub>1</sub>** = V<sub>1</sub> - V<sub>0</sub>，**E<sub>2</sub>** = V<sub>2</sub> - V<sub>0</sub>，**T** = O - V<sub>0</sub>。

可得 u**E<sub>1</sub>** + v**E<sub>2</sub>** - **D**t = **T**。

再设 **E<sub>1</sub>**(x<sub>E1</sub>, y<sub>E1</sub>, z<sub>E1</sub>)，**E<sub>2</sub>**(x<sub>E2</sub>, y<sub>E2</sub>, z<sub>E2</sub>)，**D**(x<sub>D</sub>, y<sub>D</sub>, z<sub>D</sub>)，**T**(x<sub>T</sub>, y<sub>T</sub>, z<sub>T</sub>)。

那么联立做方程组：

$$
\begin{cases}
x_{E1}u + x_{E2}v - x_D{t} = x_T\\
y_{E1}u + y_{E2}v - y_D{t} = y_T\\
z_{E1}u + z_{E2}v - z_D{t} = z_T\\
 \end{cases}
$$

根据克莱姆法则得

$$u = \frac{D_1}{D}$$

$$v = \frac{D_2}{D}$$

$$t = \frac{D_3}{D}$$

即

$$
D = \begin{vmatrix}
x_{E1} & x_{E2} & -x_D \\
y_{E1} & y_{E2} & -y_D \\
z_{E1} & z_{E2} & -z_D \\
\end{vmatrix} = \begin{vmatrix} \bf{E_1} & \bf{E_2} & -\bf{D}\\ \end{vmatrix}
$$

$$
D_1 = \begin{vmatrix}
x_T & x_{E2} & -x_D \\
y_T & y_{E2} & -y_D \\
z_T & z_{E2} & -z_D \\
\end{vmatrix} = \begin{vmatrix} \bf{T} & \bf{E_2} & -\bf{D}\\ \end{vmatrix}
$$

$$
D_2 = \begin{vmatrix}
x_{E1} & x_T & -x_D \\
y_{E1} & y_T & -y_D \\
z_{E1} & z_T & -z_D \\
\end{vmatrix} = \begin{vmatrix} \bf{E_1} & \bf{T} & -\bf{D}\\ \end{vmatrix}
$$

$$
D_3 = \begin{vmatrix}
x_{E1} & x_{E2} & x_T \\
y_{E1} & y_{E2} & y_T \\
z_{E1} & z_{E2} & z_T \\
\end{vmatrix} = \begin{vmatrix} \bf{E_1} & \bf{E_2} & \bf{T}\\ \end{vmatrix}
$$

又根据混合积公式$$\begin{vmatrix} \bf{A} & \bf{B} & \bf{C}\\ \end{vmatrix}$$ = **A** × **B** · **C** = -**A** × **C** · **B**

得

D = **D** × **E<sub>2</sub>** · **E<sub>1</sub>**

D<sub>1</sub> = **D** × **E<sub>2</sub>** · **T**

D<sub>2</sub> = **T** × **E<sub>1</sub>** · **D**

D<sub>3</sub> = **T** × **E<sub>1</sub>** · **E<sub>2</sub>**

设

**P** = **D** × **E<sub>2</sub>**

**Q** = **T** × **E<sub>1</sub>**

得到最终公式：

u = $$\frac{\bf{P} · \bf{T}}{\bf{P} · \bf{E_1}}$$

v = $$\frac{\bf{Q} · \bf{D}}{\bf{P} · \bf{E_1}}$$

t = $$\frac{\bf{Q} · \bf{E_2}}{\bf{P} · \bf{E_1}}$$

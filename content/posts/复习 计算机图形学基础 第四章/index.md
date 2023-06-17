---
title: "复习 计算机图形学基础 第四章"
date: 2023-06-17T22:14:39+08:00
tags: ["计算机图形学"]
categories: ["期末复习"]
series: ["复习 计算机图形学基础"]
---

{{<katex>}}

## 二维几何变换
### 向量矩阵
基本几何变换都是相对于坐标原点和坐标轴进行的几何变换。  
在计算机中，若要对整个图形进行几何变换，只需要将该图形的所有顶点进行相同的几何变换即可。  
一般使用一个**向量矩阵**来存储图形的一个顶点，如：
\\(
P =
\begin{bmatrix}
x \\\
y
\end{bmatrix}
\\)  
通过对图形每个顶点的向量矩阵进行各种计算，即可对该图形进行几何变换

### 平移变换
对一个图形进行平移，只需对其顶点的向量矩阵加上各个轴的偏移量即可：
![2D平移变换](./Translate2D.png "2D平移变换")
$$
P' = P + T \\\
P = 
\begin{bmatrix}
x \\\
y
\end{bmatrix}
\ \ \ 
T = 
\begin{bmatrix}
t_x \\\
t_y
\end{bmatrix}
\\\ 
P' = 
\begin{bmatrix}
x+t_x \\\
y+t_x
\end{bmatrix}=
\begin{bmatrix}
x' \\\
y'
\end{bmatrix}
$$

### 旋转变换
**绕坐标原点的旋转变换**
设旋转角度为 θ
![旋转变换](./Rotation2D.png "旋转变换")
$$
P' = RP \\\
R = 
\begin{bmatrix}
cos θ & -sin θ\\\
sin θ & cos θ
\end{bmatrix}
\ \ \ 
P =
\begin{bmatrix}
x \\\
y
\end{bmatrix}
\\\
P' = 
\begin{bmatrix}
cos θ & -sin θ\\\
sin θ & cos θ
\end{bmatrix}
\begin{bmatrix}
x \\\
y
\end{bmatrix}=
\begin{bmatrix}
xcosθ-ysinθ \\\
xsinθ+ycosθ
\end{bmatrix}
$$

> 如果某个顶点 \\(P_0(X_0,0_Y)\\) 要绕某个特定点 \\(P_r(X_r,Y_r)\\) 旋转θ度，只需要将该点设为原点，接着计算 \\(P_0\\) 点对 \\(P_r\\) 点的相对坐标 \\(P_t(X_t,Y_t)\\) ，使用 \\(P_t\\) 点进行绕坐标原点的旋转变换得出目标点 \\(P_1(X_1,Y_1)\\)，然后以该 \\(P_1\\) 点作为偏移量对 \\(P_0\\) 点进行平移变换即可。

### 缩放变换
**以坐标原点为基准点的缩放变换**
设缩放率为 \\(S_x, S_y\\)
![缩放变换](./Scale2D.png "缩放变换")
$$
P' = SP \\\
P =
\begin{bmatrix}
x \\\
y
\end{bmatrix}
\ \ \ 
S =
\begin{bmatrix}
S_x & 0 \\\
0 & S_y
\end{bmatrix}
\\\
P' = 
\begin{bmatrix}
S_x & 0 \\\
0 & S_y
\end{bmatrix}
\begin{bmatrix}
x \\\
y
\end{bmatrix}
\begin{bmatrix}
S_x \centerdot x \\\
S_y \centerdot y
\end{bmatrix}
$$

> 注意：  
> 这样的缩放变换以坐标原点为放缩参照点  
> 它不仅改变了物体的大小和形状，也改变了它离原点的距离  


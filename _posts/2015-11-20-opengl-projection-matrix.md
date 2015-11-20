---
layout: post
title: "OpenGL投影矩阵"
tagline: ""
category : doc
tags : ["OpenGL"]
---

- [概述](#Overview)
- [透视投影](#PerspectiveProjection)
- [正交投影](#OrthographicProjection)


## <a name="Overview"></a>一、概述

计算机显示器是二维平面，OpenGL渲染的三维场景需要首先转换为二维图像投影到计算机屏幕上。GL_PROJECTION矩阵正是用于这种投影变换。首先，将所有的顶点数据从视图坐标系变换到裁剪坐标系。然后通过透视除法(将裁剪坐标除以w分量)将裁剪坐标系变换到归一化的设备坐标系(NDC)。

因此，GL_PROJECTION矩阵变换包括了视锥体裁剪和NDC变换这两者。以下部分描述了如何从6个参数建立投影矩阵：左、右、上、下、远、近平面边界值。

请注意视锥体裁剪是在裁剪坐标系下进行的，位于除以$w_c$之前。顶点的裁剪坐标$x_c$，$y_c$，$z_c$需要和$w_c$分别比较进行测试，如果坐标值小于$-w_c$或者大于$w_c$，顶点将会被丢弃。

$$-w_c<x_c,y_c,z_c<w_c$$

随后，OpenGL将重建出裁剪后的多边形边界，如图1示。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_frustumclip.png" width="auto"/>
	<div class="text"><p>Fig.1 视锥体裁剪三角形</p></div>
</div>

## 二、<a name="PerspectiveProjection"></a>透视投影

透视投影将截断的棱台(视图坐标系)映射到一个立方体(NDC)，如图2示。映射坐标范围分别是：X坐标从[l,r]到[-1,1]，Y坐标从[b,t]到[-1,1]，Z坐标从[n,f]到[-1,1]。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix01.png" width="auto"/>
	<div class="text"><p>Fig.2 透视视锥体和归一化设备坐标系(NDC)</p></div>
</div>

注意视图坐标系是右手坐标系，而NDC使用左手坐标系。也就是说，相机在视图坐标系下是沿-Z轴观察，在NDC是沿+Z轴观察。由于**glFrustum()**只接受正的近平面值和远平面值，在构建GL_PROJECTION矩阵时我们需要对它们取反。

#### 1、$x_n$和$y_n$的推导

OpenGL将视图坐标系下的一个三维空间的点投影到近投影平面。下图展示视图坐标系的点$(x_e,y_e,z_e)$是如何投影到近平面$(x_p,y_p,z_p)$。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix02.png" width="auto"/>
	<div class="text"><p>Fig.3 左:视锥体上视角，右:视锥体侧面视角</p></div>
</div>

从视锥体上视角观察，视图空间的X坐标$x_e$映射到$x_p$，由相似三角形可得到：

$$\frac{x_p}{x_e}=\frac{-n}{z_e}$$
$$x_p=\frac{-n \cdot x_e}{z_e}=\frac{n \cdot x_e}{-z_e}$$

从视锥体侧边观察，采用同样的方式计算$y_p$：

$$\frac{y_p}{y_e}=\frac{-n}{z_e}$$
$$y_p=\frac{-n \cdot y_e}{z_e}=\frac{n \cdot y_e}{-z_e}$$

注意到$x_p$和$y_p$都和$z_e$相关，都与$-z_e$成反比。换句话说，它们都需要除以$-z_e$，这是构建GL_PROJECTION矩阵的第一条线索。视图坐标系经过GL_PROJECTION矩阵变换得到的裁剪坐标系仍然是齐次坐标系。最后将裁剪坐标系除以w分量就得到了归一化的设备坐标系(NDC)。

<p>
$$
\begin{pmatrix}
x_{clip} \\
y_{clip} \\
z_{clip} \\
w_{clip} \\
\end{pmatrix} = M_{projection} \cdot
\begin{pmatrix}
x_{eye} \\
y_{eye} \\
z_{eye} \\
w_{eye} \\
\end{pmatrix},
\begin{pmatrix}
x_{ndc} \\
y_{ndc} \\
z_{ndc} \\
\end{pmatrix} =
\begin{pmatrix}
x_{clip} / w_{clip} \\
y_{clip} / w_{clip} \\
z_{clip} / w_{clip} \\
\end{pmatrix}
$$
</p>

因此，我们可以将裁剪坐标系的w分量设为$-z_e$，于是，GL_PROJECTION矩阵的第4列就成为(0,0,-1,0)。

<p>
$$
\begin{pmatrix}
x_c \\
y_c \\
z_c \\
w_c \\
\end{pmatrix} = 
\begin{pmatrix}
. & . & . & . \\
. & . & . & . \\
. & . & . & . \\
0 & 0 & -1 & 0 \\
\end{pmatrix}
\begin{pmatrix}
x_e \\
y_e \\
z_e \\
w_e \\
\end{pmatrix}, \therefore w_c=-z_e
$$
</p>

下一步我们得用线性关系将$x_p$和$y_p$分别映射到$x_n$，$y_n$：$[l,r] \Rightarrow [-1,1]$和$[b,t] \Rightarrow [-1,1]$。

<div class="illustration_float">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix06.png" width="auto" height="190px" />
	<div class="text"><p>Fig.4 X坐标的映射</p></div>
</div>

$$x_n=\frac{1-(-1)}{r-l} \cdot x_p + \beta$$

$$将点(r,1)代入方程(x_p,x_n)，可得到$$

$$1=\frac{2r}{r-l} + \beta$$
$$\beta=1-\frac{2r}{r-l}=-\frac{r+l}{r-l}$$
$$\therefore x_n=\frac{2x_p}{r-l}-\frac{r+l}{r-l}$$

$y_n$的推导同样

<div class="illustration_float">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix07.png" width="auto" height="190px" />
	<div class="text"><p>Fig.4 Y坐标的映射</p></div>
</div>

$$y_n=\frac{1-(-1)}{t-b} \cdot y_p + \beta$$

$$将点(t,1)代入方程(y_p,y_n)，可得到$$

$$1=\frac{2t}{t-b} + \beta$$
$$\beta=1-\frac{2t}{t-b}=-\frac{t+b}{t-b}$$
$$\therefore y_n=\frac{2y_p}{t-b}-\frac{t+b}{t-b}$$

下一步，我们将$x_p$和$y_p$代入上述方程：

<p>
$$\begin{align}
x_n & = \frac{2x_p}{r-l}-\frac{r+l}{r-l} \\
& = \frac{2 \cdot \frac{n \cdot x_e}{-z_e}}{r-l}-\frac{r+l}{r-l} \\
& = \frac{2n \cdot x_e}{(r-l)(-z_e)} - \frac{r+l}{r-l} \\
& = \frac{\frac{2n}{r-l} \cdot x_e}{-z_e} - \frac{r+l}{r-l} \\
& = \frac{\frac{2n}{r-l} \cdot x_e}{-z_e} + \frac{\frac{r+l}{r-l} \cdot z_e}{-z_e} \\
& = (\underbrace{\frac{2n}{r-l} \cdot x_e + \frac{r+l}{r-l} \cdot z_e}_{x_c}) / -z_e
\end{align}$$
</p>

同样可得到$y_n$:

<p>
$$\begin{align}
y_n & = (\underbrace{\frac{2n}{t-b} \cdot y_e + \frac{t+b}{t-b} \cdot z_e}_{y_c}) / -z_e
\end{align}$$
</p>

注意到我们将两个方程都构建成除以$-z_e$的形式，是为了透视除法$(x_c/w_c,y_x/w_c)$。因为我们在之前将$w_c$设置为$-z_e$，括号内的项就是裁剪坐标$x_c$和$y_c$。

从以上这些方程，我们可以得到GL_PROJECTION矩阵的第一和第二列。

<p>
$$
\begin{pmatrix}
x_c \\
y_c \\
z_c \\
w_c \\
\end{pmatrix} = 
\begin{pmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\
0 & \frac{2n}{t-b} & \frac{t+b}{t-b} & 0 \\
. & . & . & . \\
0 & 0 & -1 & 0 \\
\end{pmatrix}
\begin{pmatrix}
x_e \\
y_e \\
z_e \\
w_e \\
\end{pmatrix}
$$
</p>

#### 2、$z_n$的推导

现在，我们只需要解决GL_PROJECTION矩阵的第三列。$z_n$的计算和之前的会稍有不同，因为在视图空间中$z_e$总是投影到近平面-n。但我们需要一个唯一的z值用于裁剪和深度测试。另外，我们需要保证能够进行反投影(逆变换)。已知z值和x,y无关，我们借用w分量来找到$z_n$和$z_e$间的关系。因此，可以如下指定GL_PROJECTION矩阵第三列。

<p>
$$
\begin{pmatrix}
x_c \\
y_c \\
z_c \\
w_c \\
\end{pmatrix} = 
\begin{pmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\
0 & \frac{2n}{t-b} & \frac{t+b}{t-b} & 0 \\
0 & 0 & A & B \\
0 & 0 & -1 & 0 \\
\end{pmatrix}
\begin{pmatrix}
x_e \\
y_e \\
z_e \\
w_e \\
\end{pmatrix},
z_n=z_c/w_c=\frac{Az_e+Bw_e}{-z_e}
$$
</p>

在视图空间中，$w_e$等于1。因此，等式变形为：

$$z_n=z_c/w_c=\frac{Az_e+B}{-z_e}$$

系数A和B的计算，我们使用关系$(z_e,z_n)$：$(-n,-1)$，$(-f,1)$，将其代入方程，得到

<p>
$$\begin{cases}
\frac{-An+B}{n}=-1 \\
\frac{-Af+B}{f}=1
\end{cases}
$$
</p>

求解可得到

$$A=-\frac{f+n}{f-n}$$
$$B=-\frac{2fn}{f-n}$$

代入$z_n$方程可得到

$$z_n=\frac{-\frac{f+n}{f-n}z_e - \frac{2fn}{f-n}}{-z_e}$$

最终，我们就得到了**完整的GL_PROJECTION矩阵**：

<p>
$$
\begin{pmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\
0 & \frac{2n}{t-b} & \frac{t+b}{t-b} & 0 \\
0 & 0 & \frac{-(f+n)}{(f-n)} & \frac{-2fn}{f-n} \\
0 & 0 & -1 & 0 \\
\end{pmatrix}
$$
</p>

这是一个通用的视锥体投影矩阵。如果视场范围是对称的，即$r=-l$，$t=-b$，矩阵可以简化为：

<p>
$$
\begin{pmatrix}
\frac{n}{r} & 0 & 0 & 0 \\
0 & \frac{n}{t} & 0 & 0 \\
0 & 0 & \frac{-(f+n)}{(f-n)} & \frac{-2fn}{f-n} \\
0 & 0 & -1 & 0 \\
\end{pmatrix}
$$
</p>

#### 3、Z-fighting

在我们继续之前，请再注意下$z_e$和$z_n$的关系。$z_e$和$z_n$是非线性关系。这意味着在近平面上有非常高的精度，但远平面精度较差。随着范围$[-n,-f]$逐渐变大，会造成深度精度问题(通常称为**Z-fighting**)，最终会导致在远平面附近$z_e$的一个较小的变化不会影响到$z_n$。因此应尽可能减小n和f的距离，以最小化深度缓冲区精度问题。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix04.png" width="80%"/>
	<div class="text"><p>Fig.5 深度缓冲区的精度对比</p></div>
</div>

## <a name="OrthographicProjection"></a>三、正交投影

构建正交投影GL_PROJECTION矩阵比透视投影要简单的多。

视图空间下的$x_e$，$y_e$，$z_e$分量都被线性映射到NDC。我们只需要将一个长方体缩放到立方体，并将其平移到原点。下面我们使用线性关系来得到GL_PROJECTION矩阵的元素。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix05.png" width="auto"/>
	<div class="text"><p>Fig.1 正交投影和归一化设备坐标系(NDC)</p></div>
</div>

#### 1、$x_n$的推导

<div class="illustration_float">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix08.png" width="auto" height="190px"/>
	<div class="text"><p>Fig.2 X坐标的映射</p></div>
</div>

$$x_n = \frac{1-(-1)}{r-l} \cdot x_e + \beta$$

$$将(r,1)代入上述方程(x_e,x_n)，$$

$$1=\frac{2r}{r-l}+\beta$$
$$\beta=1-\frac{2r}{r-l}=-\frac{r+l}{r-l}$$
$$\therefore x_n=\frac{2}{r-l} \cdot x_e - \frac{r+l}{r-l}$$

#### 2、$y_n$的推导

<div class="illustration_float">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix09.png" width="auto" height="190px"/>
	<div class="text"><p>Fig.3 Y坐标的映射</p></div>
</div>

$$y_n = \frac{1-(-1)}{t-b} \cdot y_e + \beta$$

$$将(t,1)代入上述方程(y_e,y_n)，$$

$$1=\frac{2t}{t-b}+\beta$$
$$\beta=1-\frac{2t}{t-b}=-\frac{t+b}{t-b}$$
$$\therefore y_n=\frac{2}{t-b} \cdot y_e - \frac{t+b}{t-b}$$

#### 3、$z_n$的推导

<div class="illustration_float">
	<img src="{{ BASE_PATH }}/images/gl_projectionmatrix10.png" width="auto" height="190px"/>
	<div class="text"><p>Fig.4 Z坐标的映射</p></div>
</div>

$$z_n = \frac{1-(-1)}{-f-(-n)} \cdot z_e + \beta$$

$$将(-f,1)$代入上述方程$(z_e,z_n)，$$

$$1=\frac{2f}{f-n}+\beta$$
$$\beta=1-\frac{2f}{f-n}=-\frac{f+n}{f-n}$$
$$\therefore z_n=\frac{-2}{f-n} \cdot z_e - \frac{f+n}{f-n}$$

#### 4、合并

正交投影不需要w分量，因此GL_PROJECTION矩阵的第4行保留为$(0,0,0,1)$。得到**完整的正交投影矩阵GL_PROJECTION**为:

<p>
$$
\begin{pmatrix}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} & 0 & -\frac{t+b}{t-b} \\
0 & 0 & \frac{-2}{f-n} & -\frac{f+n}{f-n} \\
0 & 0 & 0 & 1 \\
\end{pmatrix}
$$
</p>

如果视口范围是对称的，可以对该矩阵进一步简化，$r=-l$，$t=-b$。

<p>
$$\begin{cases}
r+l=0 \\
r-l=2r (width)
\end{cases}, 
\begin{cases}
t+b=0 \\
t-b=2t (height)
\end{cases}
$$
</p>

<p>
$$
\begin{pmatrix}
\frac{1}{r} & 0 & 0 & 0 \\
0 & \frac{1}{t} & 0 & 0 \\
0 & 0 & \frac{-2}{f-n} & -\frac{f+n}{f-n} \\
0 & 0 & 0 & 1 \\
\end{pmatrix}
$$
</p>


以上翻译自[这儿](http://www.songho.ca/opengl/gl_projectionmatrix.html)。
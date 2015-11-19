---
layout: post
title: "OpenGL变换"
tagline: ""
category : doc
tags : ["OpenGL"]
---

- [概述](#Overview)
- [OpenGL变换矩阵](#MatrixTransform)
	- [模型视图矩阵](#ModelViewMatrix)
	- [投影矩阵](#ProjectionMatrix)

## <a name="Overview"></a>一、概述

顶点坐标、法线向量等几何数据，需要通过顶点操作和图元装配`Primitive Assembly`进行变换后，再进行光栅化`raterization`处理，这是一整套OpenGL管道线处理流程。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_transform02.png" width="auto"/>
	<div class="text"><p>Fig.1 OpenGL顶点变换</p></div>
</div>

#### 1、对象坐标系(Object Coordinates)

这是对象的局部坐标系，是对象在所有变换前的初始位置和方向。可以使用**glRotatef()**, **glTranslatef()**, **glScalef()**来变换对象。

#### 2、视图坐标系(Eye Coordinates)

将`GL_MODELVIEW`矩阵和对象坐标系相乘可以得到该坐标系。OpenGL使用GL_MODELVIEW矩阵将对象从对象空间变换到视图空间。GL_MODELVIEW矩阵是模型矩阵和视图矩阵的结合($M_{view} \cdot M_{model}$)。模型变换是从对象空间到世界空间的转换，视图变换是从世界空间到视图空间的转换。

<p>
$$
\begin{pmatrix}
x_{eye} \\
y_{eye} \\
z_{eye} \\
w_{eye} \\
\end{pmatrix} = M_{modelView} \cdot
\begin{pmatrix}
x_{obj} \\
y_{obj} \\
z_{obj} \\
w_{obj} \\
\end{pmatrix} = M_{view} \cdot M_{model} \cdot
\begin{pmatrix}
x_{obj} \\
y_{obj} \\
z_{obj} \\
w_{obj} \\
\end{pmatrix}
$$
</p>

需要注意的是，OpenGL中没有单独的<abbr title="或称之为相机矩阵">视图矩阵</abbr>。因此，为了模拟来自相机或视图的变换，需要对场景（包括三维物体和光源）使用逆视图变换。也就是说，OpenGL把相机放在$(0,0,0)$处，朝向$-Z$轴，且不允许变换。更详细的描述请查看[模型视图变换矩阵](#ModelViewMatrix)。

法向量也需要从对象坐标系变换到视图坐标系，以用于光照计算。注意法线（即向量）的变换方式同顶点的变换不同。对法线进行变换需要乘以GL_MODELVIEW矩阵的逆的转置。

<p>
$$
\begin{pmatrix}
nx_{eye} \\
ny_{eye} \\
nz_{eye} \\
nw_{eye} \\
\end{pmatrix} = ({M_{modelView}}^{-1})^T \cdot
\begin{pmatrix}
nx_{obj} \\
ny_{obj} \\
nz_{obj} \\
nw_{obj} \\
\end{pmatrix}
$$
</p>

#### 3、裁剪坐标系(Clip Coordinates)

将GL_PROJECTION矩阵乘以视图坐标系就得到了裁剪坐标系。GL_PROJECTION矩阵定义了可视空间范围（frustum）：顶点数据是如何被投影到屏幕上的（是透视投影或平行投影）。我们之所以称之为裁剪坐标系是因为变换顶点$(x,y,z)$通过比较$\pm w$来决定是否被裁掉。更详细的描述请查看[投影变换矩阵](#ProjectionMatrix)。

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
\end{pmatrix}
$$
</p>

#### 4、归一化设备坐标系(Normalized Device Coordinates, NDC)

将裁剪坐标系除以$w$就得到NDC坐标系，我们称之为`透视除法`。现在更像是窗口（屏幕）坐标系了，但仍需要进行平移和缩放到屏幕像素。当前坐标系3个轴的值的范围被归一化为-1到1。

<p>
$$
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

#### 5、窗口坐标系(Window Coordinates, or Screen Coordinates)

将NDC坐标系缩放平移以适应渲染屏幕。窗口坐标系最后通过OpenGL管道线的光栅化处理成为片段（fragment）。**glViewport(x,y,w,h)**用于定义图像最终映射的渲染区域，**glDepthRange(n,f)**用于确定窗口坐标系的Z值。窗口坐标系由以上两个函数的参数计算得到。

<p>
$$
\begin{pmatrix}
x_w \\
y_w \\
z_w \\
\end{pmatrix} = 
\begin{pmatrix}
\frac{w}{2} x_{ndc} + (x+\frac{w}{2}) \\
\frac{h}{2} y_{ndc} + (y+\frac{h}{2}) \\
\frac{f-n}{2} z_{ndc} + \frac{f+n}{2} \\
\end{pmatrix}
$$
</p>

视口变换公式是NDC和窗口坐标系的简单的线性映射。

<p>
$$
\begin{cases}
-1 \rightarrow x \\
+1 \rightarrow x+w
\end{cases}

\begin{cases}
-1 \rightarrow y \\
+1 \rightarrow y+h
\end{cases}

\begin{cases}
-1 \rightarrow n \\
+1 \rightarrow f
\end{cases}
$$
</p>

## <a name="MatrixTransform"></a>二、OpenGL变换矩阵

OpenGL使用$4\times4$矩阵进行变换运算。注意，矩阵的16个元素以列优先（column-major）的顺序存储在一维数组。OpenGL有4个不同类型的矩阵：GL_MODELVIEW, GL_PROJECTION, GL_TEXTURE, GL_COLOR。你可以在代码中使用**glMatrixMode()**来选择当前矩阵类型。例如，使用glMatrixMode(GL_MODELVIEW)就选中了GL_MODELVIEW矩阵。

<p>
$$
\begin{pmatrix}
m_0 & m_4 & m_8 & m_{12} \\
m_1 & m_5 & m_9 & m_{13} \\
m_2 & m_6 & m_{10} & m_{14} \\
m_3 & m_7 & m_{11} & m_{15} \\
\end{pmatrix}
$$
</p>

#### <a name="ModelViewMatrix"></a>1、模型视图矩阵(GL_MODELVIEW)

GL_MODELVIEW矩阵将视图矩阵和模型矩阵合并为一个矩阵。为了进行视图变换，需要对整个场景进行逆变换。**gluLookAt()**就是用来设置视图变换。

GL_MODELVIEW矩阵右侧的3个元素$(m_{12},m_{13},m_{14})$用于平移变换**glTranslatef**。元素$m_{15}$是齐次坐标，专门用于投影变换。

3个元素集，$({m_0,m_1,m_2})$，$({m_4,m_5,m_6})$，$({m_8,m_9,m_{10}})$是欧几里德和仿射变换，如旋转**glRotatef**和缩放**glScalef**。注意这3个集合实际上表示3个正交轴。

- $({m_0,m_1,m_2})$: +X轴，左向量，默认是$(1,0,0)$
- $({m_4,m_5,m_6})$: +Y轴，上向量，默认是$(0,1,0)$
- $({m_8,m_9,m_{10}})$: +Z轴，前向量，默认是$(0,0,1)$

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_anglestoaxes01.png" width="auto"/>
	<div class="text"><p>Fig.2 GL_MODELVIEW矩阵的4列</p></div>
</div>

我们可以直接从角度或lookat向量来直接推导GL_MODELVIEW矩阵，而不是使用OpenGL内置的变换函数。这部分推导以后有空再介绍吧。

需要注意的是OpenGL进行矩阵相乘运算是按照相反顺序来对顶点做多次变换。例如，有一个顶点首先经过矩阵$M_A$变换，然后矩阵$M_B$变换，OpenGL实际上在对顶点变换前首先计算$M_B \times M_A$。因此，在代码中最后的变换总是第一个出现，第一个变换则出现在最后。

<p>
$$
v'=M_B \cdot (M_A \cdot v)=(M_B \cdot M_A) \cdot v
$$
</p>

``` cpp
// Note that the object will be translated first then rotated
glRotatef(angle, 1, 0, 0);   // rotate object angle degree around X-axis
glTranslatef(x, y, z);       // move object to (x, y, z)
drawObject();
```

#### <a name="ProjectionMatrix"></a>2、投影矩阵(GL_PROJECTION)

GL_PROJECTION矩阵用于定义视锥体`frustum`，视锥体决定了物体哪些部分将被裁剪。此外，也定义了三维场景是如何投影到屏幕上的。

OpenGL提供两个函数用于GL_PROJECTION变换。`glFrustum()`用于产生透视投影， `glOrtho()`用于产生正交(平行)投影。这两个函数都需要6个参数来指定6个裁剪平面：左、右、上、下、远、近平面。视锥体的8个顶点如下图所示。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_transform09.png" width="auto"/>
	<div class="text"><p>Fig.3 OpenGL透视投影视锥体</p></div>
</div>

我们可以使用相似三角形来计算远平面的顶点，例如,远平面左侧：

$$
\frac{far}{near} = \frac{left_{far}}{left}, 
left_{far} = \frac{far}{near} \cdot left
$$

对于正交投影，如下图4左，该比例是1。所以前后平面的左右上下是完全相同的。

你也可以使用**gluPerspective()**和**gluOrtho2D()**等较少的参数的函数来设置投影。gluPerspective()仅需要4个参数：垂直视场角（field of view, FOV），长宽比，以及分别到前后表面的距离。下述代码是从gluPerspective()到glfrustum()的等效转换。

``` cpp
// This creates a symmetric frustum.
// It converts to 6 params (l, r, b, t, n, f) for glFrustum()
// from given 4 params (fovy, aspect, near, far)
void makeFrustum(double fovY, double aspectRatio, double front, double back)
{
    const double DEG2RAD = 3.14159265 / 180;

    double tangent = tan(fovY/2 * DEG2RAD);   // tangent of half fovY
    double height = front * tangent;          // half height of near plane
    double width = height * aspectRatio;      // half width of near plane

    // params: left, right, bottom, top, near, far
    glFrustum(-width, width, -height, height, front, back);
}
```

如果需要创建一个非对称的视锥体，就只能使用glFrustum()。例如，我们打算渲染一个大场景到2个相邻的屏幕上，可以将视锥体分成2个不对称的视锥体（一左一右），使用2个视锥体分别渲染场景到不同的屏幕上。

<div class="illustration">
	<img src="{{ BASE_PATH }}/images/gl_transform16.png" width="auto"/>
	<div class="text"><p>Fig.4 左图：OpenGL正交投影视锥体，右图：非对称视锥体</p></div>
</div>

#### 3、纹理矩阵(GL_TEXTURE)

在进行纹理映射前纹理坐标$(s,t,r,q)$首先要经过GL_TEXTURE矩阵变换。该矩阵默认是单位矩阵，因此纹理被映射到物体上完全按照你所分配的纹理坐标。通过修改GL_TEXTURE，可以做到移动，旋转，拉伸和收缩纹理。

``` cpp
// rotate texture around X-axis
glMatrixMode(GL_TEXTURE);
glRotatef(angle, 1, 0, 0);
```

#### 4、颜色矩阵(GL_COLOR)

颜色分量$(r,g,b,a)$需要经过GL_COLOR矩阵变换。可以用于颜色空间变换和颜色分量交换。GL_COLOR矩阵不经常使用，并且需要**GL_ARB_imaging**扩展。

#### 5、其他矩阵函数

- **glPushMatrix()**：把当前矩阵压入当前矩阵堆栈
- **glPopMatrix()**：把当前矩阵从当前矩阵堆栈弹出
- **glLoadIdentity()**：设置当前矩阵为单位矩阵
- **glLoadMatrix{fd}(m)**：将当前矩阵替换为矩阵m
- **glLoadTransposeMatrix{fd}(m)**：将当前矩阵替换为行优先的矩阵m
- **glMultMatrix{fd}(m)**：将当前矩阵乘以矩阵m，并更新结果到当前矩阵
- **glMultTransposeMatrix{fd}(m)**：将当前矩阵乘以行优先的矩阵m，并更新结果到当前矩阵
- **glGetFloatv(GL_MODELVIEW_MATRIX, m)**：返回GL_MODELVIEW_MATRIX矩阵的16个值。

以上翻译自[这儿](http://www.songho.ca/opengl/gl_transform.html)。详细的模型视图矩阵、投影矩阵的推导以后会补上。

















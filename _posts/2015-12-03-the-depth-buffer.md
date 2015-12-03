---
layout: post
title: "深度缓冲区"
tagline: ""
category : doc
tags : ["OpenGL"]
---

## 一、如何确保深度缓冲区正常工作？

你的应用程序需要至少做以下工作来确保深度缓冲区正常工作：

1. 在创建窗口时建立深度缓冲区；
2. 在OpenGL渲染上下文已建立并设置为当前上下文，在程序初始化时调用`glEnable(GL_DEPTH_TEST)`；
3. 确保远近裁剪平面`zNear`和`zFar`设置正确(确保都大于0)，提供足够的深度精度信息；
4. 将`GL_DEPTH_BUFFER_BIT`作为参数传递给`glClear`，通常是以位或形式与其它值组合，如`GL_COLOR_BUFFER_BIT`。

## 二、为什么深度缓冲区精度这么差？

视觉坐标系下的深度缓冲区精度受以下几点影响：zFar和zNear的比例，zFar裁剪平面，以及物体距离zNear裁剪平面的距离。

你应该尽可能的拉近zFar平面，并推远zNear平面。

具体来说，考虑以下情况，通过由`glFrustum(l,r,b,t,n,f)`指定的透视投影矩阵，将视觉坐标系下深度信息$x_e, y_e, z_e, w_e$变换到窗口坐标系$x_w, y_w, z_w$，交假设采用默认的视口变换。裁剪坐标系$z_c$和$w_c$为

$$z_c=-z_e \cdot (f+n)/(f-n)-w_e \cdot 2fn(f-n)$$
$$w_c=-z_e$$

为什么是负的？OpenGL想要告诉我们在投影前是右手坐标系，投影后则变成左手坐标系。ndc坐标系为：

<p>$$\begin{align}
z_{ndc} & = z_c/w_c=[-z_e \cdot (f+n)/(f-n)-w_e \cdot 2fn(f-n)]/-z_e \\
& = (f+n)(f-n)+(w_e/z_e) \cdot 2fn(f-n)
\end{align}$$</p>

视口变换使用默认深度范围(为$[0,1]$)进行缩放和平移，即$[-1,1]\rightarrow[0,1]$，然后再使用$s=(2\^n-1)$进行缩放，其中n是深度缓冲区的位宽：

$$z_w=s \cdot [(w_e/z_e) \cdot f \cdot n(f-n) + 0.5(f+n)(f-n)+0.5]$$

我们来重新调整下这个方程，

<p>$$\begin{align}
z_e/w_e & = f \cdot n/(f-n)/((z_w/s)-0.5(f+n)(f-n)-0.5) \\
& = f \cdot n/((z_w/s) \cdot (f-n) - 0.5(f+n) - 0.5(f-n)) \\
& = f \cdot n/((z_w/s) \cdot (f-n) - f)
\end{align}$$</p>

现在我们将以下两点代入上式：zNear裁剪平面和zFar裁剪平面，

$$z_w=0 \Rightarrow z_e/w_e=f \cdot n / (-f)=-n$$
$$z_w=s \Rightarrow z_e/w_e=f \cdot n/((f-n)-f)=-f$$

在定点缓冲区中，$z_w$被量化为整数。以下表示深度缓冲区位于裁剪平面1到s-1之间：

$$z_w=1 \Rightarrow z_e/w_e=f \cdot n/((1/s) \cdot (f-n)-f)$$
$$z_w=s-1 \Rightarrow z_e/w_e=f \cdot n/((s-1)/s \cdot (f-n)-f)$$

下面我们代入一些数据，如，n=0.01，f=1000，s=65535(表示16位深度缓冲区)

$$z_w=1 \Rightarrow z_e/w_e=-0.01000015$$
$$z_w=s-1 \Rightarrow z_e/w_e=-395.90054$$

我们来考虑下以上示例。在视觉坐标系下从-359.5到-1000都只能被映射到深度缓冲区的值为65534或65535。几乎zNear到zFar裁剪平面之间三分之二的距离都只能有两个深度值。

为了进一步分析z-buffer分辨率，我们来计算下&z_e/w_e&的导数

$$d(z_e/w_e)/dz_w=-f \cdot n \cdot (f-n) \cdot (1/s)/((z_w/s) \cdot (f-n) -f)\^2$$

来看一下当$z_w=s$时，

$$d(z_e/w_e)/dz_w=-f \cdot (f-n) \cdot (1/s)/n=-f \cdot (f/n-1)/s$$

如果你想要深度缓冲区在靠近zFar面处有意义，就需要保证上式值小于视觉空间下物体的大小(通常情况下是世界空间)。

原文请参考[这儿](https://www.opengl.org/archives/resources/faq/technical/depthbuffer.htm)。

#### CODE

下面附上一段代码实现：已知入点和出点的深度值，插值出中间某位置处的深度值。以下函数参数`entryPointsDepth`和`exitPointsDepth`分别是入点和出点的深度值，`t`是当前插值点所在位置占百分比(位于0~1间)。一般可能直接使用线性插值直接得到

`entryPointsDepth + (exitPointsDepth-entryPointsDepth)*t`

但这样计算是不够准确的。因为深度信息在窗口空间不是线性分布的，我们需要将其首先反变换到视觉空间下，在视觉空间下使用线性插值，再将插值后的结果变换回窗口空间。下面这段代码就是这样实现的。

``` c++
/**
* Calculates the depth value for the current sample specified by the parameter t.
**/
float calculateDepthValue(float t, float entryPointsDepth, float exitPointsDepth) {
  /*
  Converting eye coordinate depth values to windows coordinate depth values:
  (see http://www.opengl.org/resources/faq/technical/depthbuffer.htm 12.050, assuming w_e = 1)

  z_w = (1.0/z_e)*((f*n)/(f-n)) + 0.5*((f+n)/(f-n))+0.5; (f=far plane, n=near plane)

  We calculate constant terms outside:
  const_to_z_w_1 = ((f*n)/(f-n))
  const_to_z_w_2 = 0.5*((f+n)/(f-n))+0.5

  Converting windows coordinates to eye coordinates:

  z_e = 1.0/([z_w - 0.5 - 0.5*((f+n)/(f-n))]*((f-n)/(f*n)));

  with constant terms
  const_to_z_e_1 = 0.5 + 0.5*((f+n)/(f-n))
  const_to_z_e_2 = ((f-n)/(f*n))
  */

  // assign front value given in windows coordinates
  float zw_front = entryPointsDepth;
  // and convert it into eye coordinates
  float ze_front = 1.0 / ((zw_front - const_to_z_e_1)*const_to_z_e_2);

  // assign back value given in windows coordinates
  float zw_back = exitPointsDepth;
  // and convert it into eye coordinates
  float ze_back = 1.0 / ((zw_back - const_to_z_e_1)*const_to_z_e_2);

  // interpolate in eye coordinates
  float ze_current = ze_front + t*(ze_back - ze_front);

  // convert back to window coordinates
  float zw_current = (1.0 / ze_current)*const_to_z_w_1 + const_to_z_w_2;

  return zw_current;
}
```

<div class="illustration_float">
	<img src="{{ BASE_PATH }}/images/depth-information.png" width="80%" />
	<div class="text"><p>Fig.1 深度信息</p></div>
</div>

`calculateDepthValue`函数用于计算体绘制的深度信息，体绘制已知前表面和后表面深度信息(这通常是由渲染颜色立方体得到)，以及光线终止位置，通过该函数就可以得到光线终止时的深度值。


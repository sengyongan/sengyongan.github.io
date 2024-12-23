---
title: UnityShade入门精要（2·基础LightModel）
date: 2024-11-21 12:00:00 +0800
categories: [Unity_Shader]
tags: [shader]     # TAG names should always be lowercase
math: true
---
# 光照

如果它看起来是对的， 那么它就是对的

### 理论

颜色的本质？
看到的物体颜色，是光源发出的光线经过物体散射后到人眼

如何量化光？（辐射度量学）
一个物体表面的辐照度，是垂直于单位面积单位时间通过的光能量，对于不垂直的光能量，物体辐照度和costheta（光线和物体法线）成正比

光线与物体相交时？
会散射（反射（外部）和折射（内部））（不改变光线颜色）与吸收（改变光线颜色）

光照模型中，分为两部分计算物体颜色：
镜面反射 (specular) 部分表示物体表面是如何反射光线的，
漫反射 (diffuse) 部分则表示有多少光线会被折射、吸收和散射出表面。（通常假设散射出来的光线是均匀方向分布的）

着色，光照模型？
根据物体材质（如何与光相互作用），光源信息（如光源方向、辐照度等），去计算观察方向的出射光线数量（颜色）和方向的过程 / 算法，这就是着色 / 光照模型

对真实散射吸收过程的简化，真实散射吸收过程是很复杂的，涉及很多变化

### 标准光照模型

只关心直接光照(direct light)：也就是那些直接从光源发射出来照射到物体表面后，经过物体表面的一次反射直接进入摄像机的光线

把进入到摄像机/人眼内的光线分为4个部分，每个部分使用一种方法来计算它的贡献度：

* 自发光(emissive)部分，这个部分用千描述当给定一个方向时，一个表面本身会向该方向发射多少辐射量，而不需要经过任何物体的反射（非全局光照下，并不会影响其他物体），它用**材质颜色**表示
* 环境光(ambient)部分， 它用于描述其他所有的间接光照，在光线进入摄像机之前， 经过了不止一次的物体反射，通常使用一个**全局变量**描述
* 漫反射(diffuse)部分，这个部分用于描述，当光线从光源照射到模型表面时， 该表面会向每个方向散射多少辐射量，符合兰伯特定律 (Lambert's law):反射光线的强度与表面法线和光源方向之间夹角的余弦值成正比：**漫反射颜色 = (光源颜色 * 漫反射颜色) max（0，法线n 点乘 光线I矢量）**，其中0保证颜色不为负值（当夹角>90度）
* 镜面反射(specular)部分， 这个部分用千描述当光线从光源照射到模型表面时， 该表面会在完全镜面反射方向散射多少辐射量。**镜面反射颜色 =  (光源颜色 * shininess反光度) max（0，视线v 点乘 反射光线r） gloss光泽度** ，


标准光照模型是最基础的光照模型，但很多重要的物理现象，菲涅耳反射 (Fresnel reflection)，各向异性 (anisotropic) 等，都无法用 标准光照模型 表现出来

### vert？ vs. frag？
计算光照模型可以在顶点着色器中，也可以在片段着色器中。

逐顶点光照得到的不平滑的光照效果，运行次数一般较少

逐像素光照可以得到更加平滑的光照效果，但要运行的像素次数一般较大

### Blinn-Phong

利用blinn——phong光照模型：**镜面反射颜色 =  (光源颜色 * shininess反光度) max（0，法线n 点乘 半程h（(视线+入射光) / 2 的归一化）） gloss光泽度**

### BRDF 光照模型

使用一个数学公式式来表示， 并且提供了一些参数来调整材质属性,即算出某个出 射方向上的光照能量分布
……

### 逐顶点的漫反射实战

漫反射颜色作为Properties语义块中的属性，方便修改

通过Tags指明光照模式：LightMode 标签它用于定义该Pass 在Unity 的光照流水线中的角色，，只有定义了正确的LightMode（例如前向渲染）,我们才能得到一些Unity 的内置光照变量，

mul（）矩阵乘法（行* 列）或向量与矩阵乘法

定义输入输出的struct

在vertex shader中，使用漫反射标准光照模型，计算顶点颜色

* 通过UNITY_LIGHTMODEL_AMBIENT内置变量得到了环境光部分。
* _LightColorO获取光源的颜色和强度信息，正规化/归一化（只关心方向，不关心大小）
* 光源方向可以由_WorldSpaceLightPosO 来得到（这种方式并不具有通用性，只们假设场景中只有一个光源且该光源的类型是平行光Directional Light），归一化
* 需要将法线与 unity_WorldToObject（将世界空间中的坐标转换为模型空间中的坐标）法线矩阵相乘，变换到世界坐标，
* 通过saturate （）保证dot（）点乘结果为[O, l]的范围内

### 法线矩阵

对于postion来说直接*model矩阵，即可变换到世界坐标系下，进行光照计算

但是对于法线来说，不能简单的*model（SRT）矩阵，需要考虑两点mat3(transpose(inverse(model)))

* 法向量只是一个方向向量，不能表达空间中的特定位置，因此位移不应该影响到法向量，所以法线向量只需要前3个分量，矩阵只需要3*3
* 当模型不等比缩放时，法线的方向不再垂直于表面（因为矩阵 * 向量会改变原有向量的方向），会造成错误的渲染效果，为了法线依旧垂直于表面需要model的转置逆矩阵
* inverse逆矩阵，负责变换到原始空间，即从世界空间变换到局部空间（WorldToObject）
* transpose转置矩阵，负责保证正交性关系，以保证法线垂直于表面

### 逐像素的漫反射实战

仅需要将光照模型放到fragment shader中即可，其他做一些修改

### 半兰伯特 (HalfLambert) 光照模型

标准光照模型没有模拟间接光照，因此在光照无法到达的区域， 模型的外观通常是全黑的，有一种改善技术HalfLambert光照模型（之前使用的是兰伯特模型）

半兰伯特光照模型没有使用 max操作来防止点积结果为负值， 而是对其结果进行了一个a倍的缩放再加上一个b小的偏移，绝大多数情况下， a和b的值均为 0.5

本质就是将点乘的结果范围从[-1, 1]映射到[O, 1]范围内，从而，背光面也可以有明暗变化

```c++
fixed halfLambert = dot(worldNorrnal, worldLightDir) * 0.5 + 0.5; 
fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * halfLambert;
```

逐顶点，逐像素，半兰伯特：

![1732981051479](/assets/img/blog/unityshader/漫反射.png)

### 逐顶点的镜面反射实战

如何获得反射光线向量？

I - 2* dot(I,n)*n;   反射向量 = I + 2(I在n 的投影长度)

点积的几何意义是一个向量在另一个向量方向上的投影长度，（- 2* dot(I,n)*n）构造一个与法线方向相同，长度是投影的2倍的向量

加法求反射向量[可以参考推导](https://blog.csdn.net/a1047120490/article/details/106711036)，但是CG提供了计算反射方向的函数reflect(i, n)

//

在shader中，首先公开3个属性

其他的先掠过，主要说一下不同的地方，环境光和漫反射部分不变，反射向量通过reflect（）函数，视线通过_WorldSpaceCameraPos - 世界坐标的顶点（获得起点为顶点，终点为相机方向的向量），之后带入镜面反射公式就行

### 逐像素的镜面反射实战

……

### Blinn-Phong 光照模型实战

非常简单，只需要将原有的视线和反射光线点乘，替换为半程h（视线+入射光）和法线点乘

逐顶点，逐像素，半程h：

![1732981135891](/assets/img/blog/unityshader/镜面反射.png)

### 内置函数

UnityCG.cginc 中一些常用的帮助函数，可以帮助我们避免手动计算（例如光源方向、视角方向等）
![1732465914750](/assets/img/blog/unityshader/常用的帮助函数.png)

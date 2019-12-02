---
layout: post
title: "Unity渲染路径——延迟渲染路径"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

在之前的描述中，我们知道每个Pass都需要指定渲染路径，否则，一些光照变量很可能不会被正确赋值，计算得到的效果也就很可能是错误的。那么接下来的内容将涉及具体的渲染路径解释。

前向渲染的问题是：当场景中包含大量实时光源时，前向渲染的性能会急速下降。

 ![img]({{site.url}}/assets/image/posts_images/deferred_example.png) 

例如，如果我们在场景的某一块区域内放置了多个光源，这些光源的影响区域又互相重叠，那么为了得到最终的光照效果，我们就需要为该区域的每个物体执行多个Pass来计算不同光源对该物体的光照效果，然后在颜色缓存中把这些结果混合起来得到最终的光照。然而每执行一个Pass我们都需要重新渲染一遍物体，但很多计算实际上是重复的。

但是延迟渲染可以有效地避免这个问题，它与前向渲染的不同之处是，除了前向渲染中使用的颜色缓冲和深度缓冲之外，延迟渲染还会利用额外的缓冲区，即G缓冲，其中G是英文Geometry的缩写。G缓冲存储了我们所关心的表面（通常是离摄像机最近的表面）的其他信息，例如法线、位置、用于光照计算的材质属性等。

#### 延迟渲染原理

延迟渲染的主要包含两个Pass。在第一个Pass中，不进行任何光照计算，而是仅仅计算哪些片元可见，这主要通过深度缓冲技术实现，当发现一个片元可见，就将其相关信息存储到G缓冲区中。然后，在第二个Pass中，利用G缓冲的各个片元信息，如表面法线、视角方向、漫反射系数等，进行真正的光照计算。伪代码描述如下：

```presudo
Pass 1 {
	//第一个Pass不进行光照计算，仅存储光照计算所需信息到G缓冲中
	
	for (each primitive in this model) {
 		for (each fragment covered by this primitive) {
 			if (failed in depth test) {
 				discard;//若没有通过深度测试，则该片元不可见
 			}
 			else {
 				//若该片元可见，则进行光照计算，则将相关信息存储到G缓冲中
 				writeBuffer(materialInfo, pos, normal, lightDir, viewDir);
 			}
 		}
	}
}
Pass 2 {
	//利用G缓冲中的信息进行真正的光照计算
	
	for (each pixel in the screen) {
		if (the pixel is valid) {
        	//若该像素有效，则读取其G缓冲中的信息
        	readBuffer(pixel, materialInfo, pos, normal, lightDir, viewDir);
        	
        	//根据读取到的信息进行光照计算
        	float4 color = Shading(materialInfo, pos, normal, lightDir, viewDir);
        	//更新帧缓冲
        	writeFrameBuffer(pixel, color);
        }
    }
}
```

可以看出，延迟渲染使用的Pass数通常是两个，而与场景中包含的光源数目无关。

#### Unity中的实施细节和要求

对于延迟路径来说，它最适合光源数目较多，使用前向渲染会造成性能瓶颈的场景。由于延迟渲染路径的每个光源都可以按逐像素方式处理，其性能取决于场景中灯光的大小，因此可以通过保持较小的灯光来提高性能。但延迟渲染也有一些缺点：

- 对抗锯齿（anti-aliasing）没有真正的支持，也不能处理半透明的物体对象。
- 不支持Mesh渲染器(Mesh Renderer)的`RecieveShadow`（接受阴影）tag，并且支持的`culling masks`（剔除遮罩数）也是有限的。最多只能使用4个`culling mask`，也就是说，`culling layer mask`（剔除遮罩层）的设定不能少于所有层减去4个任意层的个数，即，32个层中的28个。
- 对显卡有一定要求，显卡需要支持**MRT**（Mutlple Render Targets）、Shader-Model 3.0及以上、深度渲染纹理（Depth render texture）。 2006年以后生产的大多数PC（从GeForce 8xxx，Radeon X2400和Intel G45开始）显卡都支持延迟着色。 

当使用延迟渲染时，Unity要求我们提供两个Pass。

1. 第一个Pass（G-Buffer pass） 将每个物体对象渲染1次 。在这个pass中，物体对象的漫反射和高光反射的颜色、表面平滑度、世界空间的法线以及自发光+环境光+反射+光照贴图等信息被渲染到屏幕空间的G缓冲中。将G缓冲纹理设置为全局着色器属性，以供以后由着色器（ *CameraGBufferTexture0 ..* CameraGBufferTexture3 names ）访问。

2. 第二个Pass（Lighting pass） 基于G缓冲区和深度来计算光照 。 由于光照是在屏幕空间中计算的，因此光照处理所需的时间与场景的复杂性无关。 最终的光照颜色被存储到帧缓冲中。

   未穿过相机**近平面（near plane）**的**点（point）**光源和**聚光灯(spot)**光源被渲染为3D形状，并启用了针对场景的Z缓冲区测试。这使得部分或完全遮挡的点光源和聚光灯渲染起来非常便宜。穿过近平面的**平行光（directional lights）**,**点（point）**光源和**聚光灯(spot)**光源被渲染为全屏**四边形(quads)**。 

G缓冲中**渲染目标（Render Target, RT）**（RT0-RT4）的默认**布局(layout)**列举如下,其中数据类型放置在每个渲染目标的各个通道（括号内字母表示通道）中:

- RT0，ARGB32格式：（RGB）存储漫反射颜色，（A）被遮挡。
- RT1，ARGB32格式：（RGB）存储高光反射颜色（RGB），（A）存储高光反射的指数部分，通常认为是高光的粗糙度。
- RT2，ARGB2101010格式：（RGB）存储世界空间的法线信息，未使用（A）。
- RT3，ARGB2101010（非HDR）或ARGBHalf（HDR）格式：用于存储自发光+光照+ 光照贴图
   \+ 反射探针（reflection probes）。
- 深度缓冲+ **模板缓冲（Stencil buffer）**。

 因此，默认的g缓冲区布局为160位/像素（非HDR）或192位/像素（HDR） 。 

如果将**阴影蒙版（Shadowmask）**或**距离阴影蒙版（Distance Shadowmask）**模式用于混合光照(Mixed lighting)，则使用第五个目标：

- RT4，ARGB32格式：光遮挡值（RGBA）。

因此，G缓冲区布局为192位/像素（非HDR）或224位/像素（HDR）。

如果硬件不支持五个并发渲染目标，则使用阴影遮罩的对象将回退到正向渲染路径。

当照相机不使用**HDR**动态范围时，**RT3**将进行对数编码，以提供比通常使用ARGB32纹理可能提供的更大的动态范围。当相机使用**HDR**渲染时，不会为**RT3**创建单独的渲染目标。而是将Camera渲染到的渲染目标（即传递到图像效果的渲染目标）用作RT3。

#### 可访问的内置变量和函数



| 名称          | 类型     | 描述                                                         |
| ------------- | -------- | ------------------------------------------------------------ |
| _LightColor0  | float4   | 该pass处理逐像素光源的颜色                                   |
| _LightMatrix0 | float4x4 | 从世界空间到光源空间的变换矩阵。可以用于采样cookie和光强衰减（attenuation）纹理 |



#### 渲染路径的选择

Unity的官方网站上其实还给出了其他的渲染路径以及它们的详细比较，在之前的介绍中也给出了这种表格。总体来说，渲染路径的选择依赖于游戏发布的目标平台；若当前显卡不支持所选渲染路径，Unity会自动降级渲染。

接下来的实验章节中主要使用Unity的前向渲染路径。

### 参考

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[LearnOpenGL-CN/Advanced Lighting/Deferred Shading](https://learnopengl-cn.readthedocs.io/zh/latest/05 Advanced Lighting/08 Deferred Shading/) 


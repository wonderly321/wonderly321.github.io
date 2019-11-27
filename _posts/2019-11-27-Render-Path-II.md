---
layout: post
title: "Unity渲染路径-前向渲染路径"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

在之前的描述中，我们知道每个Pass都需要指定渲染路径，否则，一些光照变量很可能不会被正确赋值，计算得到的效果也就很可能是错误的。那么接下来的内容将涉及具体的渲染路径解释。

### 前向渲染路径

前向渲染路径是Unity默认的渲染方式，也是最常用的一种。

#### 原理

每进行一次完整的前向渲染，我们都需要渲染该对象的渲染图元，并计算颜色缓冲区和深度缓冲区的信息，其中深度缓冲区的信息决定了一个片元是否可见，若可见则颜色缓冲区中的值会被更新。伪代码描述如下：

```presudo
Pass {
	for (each primitive in this model){
 		for(each fragment covered by this primitive){
 			if(failed in depth test){
 				discard;//若没有通过深度测试，则该片元不可见
 			}
 			else {
 				float4 color = Shading(materialInfo, pos, normal, lightDir, viewDir);//若该片元可见，则进行光照计算
 				writeFrameBuffer(fragment, color);//更新帧缓冲
 			}
 		}
	}
}
```

对于每个逐像素光源，都需要进行一次上述渲染流程。若一个物体在多个逐像素光源的影响区域内，则该物体会执行多个Pass，每个Pass计算一个逐像素光源的光照结果，然后在帧缓冲中把这些光照结果混合起来得到最终的颜色值。假设，场景中有N个物体，每个物体受M个光源的影响，那么要渲染整个场景一共需要N*M个Pass。也就是逐像素光照越多执行的Pass数也有可能越变越大，因此渲染引擎通常限制每个物体逐像素光照的数目。

#### Unity中的实施细节和要求

事实上，一个Pass不仅可以用来计算逐像素光照，也可以用来计算逐顶点等其他光照，这取决于光照所处的流水线阶段以及计算时使用的数学模型。当我们渲染一个物体时，Unity会计算哪些光源照亮了它，以及这些光源照亮该物体的方式。

在Unity中，前向渲染路径有3种处理光照（即，照亮物体）的方式：**逐像素处理**，**逐顶点处理**，**球谐函数（Spherical Harmonics, SH）处理**。而决定一个光源使用哪种处理模式取决于它的类型和渲染模式。光源类型指的是该光源是平行光还是其他类型的光源，而 光源的渲染模式指的是该光源是否是**重要的（Important）**。如果把一个光照的模式设置为Important，则其处理方式为逐像素光照，具体设置的面板如图：

![image-20191127214653565]({{site.url}}/assets/image/posts_images/LightSetting)

在前向渲染中，Unity会根据场景中各个光源的设置以及这些光源对物体的影响程度（比如，距离该物体的远近、光源强度等）对这些光源进行一个重要度排序。其中每个顶点最多计算4个逐顶点光照。其他的光以球谐函数（SH）的形式计算。这样处理，虽然速度更快，但其实是一个近似的值Unity使用的判断规则如下：

- 渲染模式被设置成`Not Important`的光源，会按逐顶点或者SH处理
- 场景中的最亮的平行光总是按逐像素处理的
- 渲染模式被设置成`Important`的光源会按照逐像素处理
- 若根据以上规则得到的逐像素光源少于`Quality Setting`的逐像素光源数量（Pixel Light Count),则会有更多的光源以逐像素的方式进行渲染

例如，如果某些对象受到许多光源的影响（下图中的圆圈代表物体对象，其受光源A到H的影响）：



 ![img](https://docs.unity3d.com/uploads/Main/ForwardLightsExample.png) 

假设光源A到H具有相同的颜色和强度，并且所有光源都具有**自动（`Auto`）**渲染模式，因此为这个对象按照上述规则对其进行排序和渲染。则最亮的灯光将在A到D中以逐像素处理的方式渲染，然后在D到G中以逐顶点处理的方式最多渲染4个，最后以SH去渲染其余（G-H）所有的光照：



 ![img](https://docs.unity3d.com/uploads/Main/ForwardLightsClassify.png) 

那么设置的光照将在哪里进行计算呢？答案是在Pass中。前向渲染一共有两种Pass: **Base Pass**和**Additional Pass**。其设置如下：

**Base Pass**: 

BasePass 可计算的光照包括：一个逐像素处理的平行光和所有SH 或者逐顶点处理的光源，可实现的光照效果有光照纹理（lightmap），环境光，自发光，阴影（主要指平行光的阴影）。

请注意，（1）lightmap对象不会从SH光源获得光照。（2）在着色器中使用`OnlyDirectional`作为该pass的Tag值时 ，前向渲染的Base Pass仅渲染主方向光、环境光、光照探针（lightprobe）和光照纹理，SH和顶点光不包括在该pass数据中。

**Additional Pass**：

BasePass 可计算的光照包括：其他影响该物体对象的逐像素光源，可实现的光照效果不支持阴影。

除非使用multi_compile_fwdadd_fullshadows变体快捷方式，否则默认情况下，这些Pass中的光照没有阴影，所以总的来看，正向渲染只支持一个带阴影的平行光。

#### 多说一句关于性能的考量

球形谐波光照（SH）的渲染速度非常快。它们在CPU上的成本很小，甚至实际上可以免费供GPU应用（也就是说，base pass总是计算SH光照；但是由于SH光照的工作方式，其计算成本光照数目的增加而增加）。

不过SH光照也有非常明显的缺点：

- 它们是在对象的顶点而不是像素处计算的，因此不支持light Cookies或**法线贴图(normal maps)**
- SH光照频率很低，不能表现尖锐的灯光过渡，也仅影响漫反射光照（对于高光而言，频率太低）。
- SH光照不在本地；靠近某个表面的点光源的光照计算效果看起来是错误的。

总而言之，SH光照通常可以应付小型动态物体。

#### Unity中的内置变量和函数

根据所设置的渲染路径（即 Pass 的标签`LightMode`的值），Unity会把不同的光照变量传递给Shader。对于前向渲染（即`LightMode`设置为`ForwardBase`或`ForwardAdd`）来说，有以下一些可以在Shader中访问的光照变量以供参考：

| 名称                                                    | 类型     | 描述                                                         |
| ------------------------------------------------------- | -------- | ------------------------------------------------------------ |
| _LightColor0                                            | float4   | 该Pass处理逐像素光源的颜色                                   |
| _WorldSpaceLightPos0                                    | float4   | _WorldSpaceLightPos0.xyz是该Pass处理逐像素光源的位置。如果该光源是平行光，那么_WorldSpaceLightPos0.w是0，其他光源类型的w是1 |
| _LightMatrix0                                           | float4x4 | 从世界空间到光源空间的变换矩阵。可以用于采样cookie和光强衰减（attenuation）纹理 |
| unity_4LightPosX0, unity_4LightPosY0, unity_4LightPosZ0 | float4   | 仅用于Base Pass。前4个非重要（Not Important）的点光源在世界中的位置 |
| unity_4LightAtten0                                      | float4   | 仅用于Base Pass。前4个非重要（Not Important）的点光源的衰减因子 |
| unity_LightColor                                        | float4   | 仅用于Base Pass。前4个非重要（Not Important）的点光源的颜色  |

一些可调用的内置函数仅供参考：

| 函数名                                      | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| float3    WorldSpaceLightDir(float4 v)      | 仅可用于前向渲染中。输入一个模型空间中的顶点位置，返回世界空间中的从该点到光源的光照方向。内部实现用了UnityWorldSpaceLightDir函数，没有被归一化 |
| float3    UnityWorldSpaceLightDir(float4 v) | 仅可用于前向渲染中。输入一个世界空间中的顶点位置，返回世界空间中的从该点到光源的光照方向，没有被归一化 |
| cfloat3    ObjSpaceLightDir(float4 v)       | 仅可用于前向渲染中。输入一个模型空间中的顶点位置，返回模型空间中的从该点到光源的光照方向，没有被归一化 |
| float3    Shader4PointLights(...)           | 仅可用于前向渲染中。计算4个点光源的光照，其参数是已经打包进矢量的光照数据，通常是上述的内置变量，如unity_4LightPosX0等，用于计算逐顶点光照 |

前向渲染路径的知识点差不多就这些，下一节是延迟渲染路径，它将覆盖掉一些前向渲染的弊端，下节见~

### 参考

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)
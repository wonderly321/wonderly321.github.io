---
layout: post
title: "Unity中的基础光照"
tagline: "通常来讲，想要模拟真实的环境光照来生成一张图像需要考虑这样一种流程：首先，光线从光源中发射出来；然后，光线和场景中的一些物体相交，其中一部分光线被吸收，一部分被散射；最后，摄像机吸收了一些光，产生了一张图像"
categories: shader
image:
author: "Wonder"
meta: "Unity Shader"
---

<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
 
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  tex2jax: {
    inlineMath: [['$','$'], ['\\(','\\)']],
    processEscapes: true
  }
});
</script>
 
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
      }
    });
</script>

#### 通常来讲，想要模拟真实的环境光照来生成一张图像需要考虑这样一种流程：首先，光线从**光源**中发射出来；然后，光线和场景中的一些物体相交，其中一部分光线被吸收，一部分被散射；最后，摄像机吸收了一些光，产生了一张图像。

#### 光源

在实时渲染中，光源可以看作是一个没有体积的点，用l表示其方向，用**辐射度(irradiance)** 来量化。比如平行光，其辐射度可通过计算在垂直于l的单位面积上单位时间内穿过的能量来得到。而计算物体表面的辐射度，可以利用光源方向l和表面法线n之间的夹角的余弦值来得到。（ps. 方向矢量默认模长为1）。实际计算中，辐射度与cosθ成正比，所以使用**点积**来计算。

#### 吸收与散射

光源与物体相交的结果有两个：**散射**和**吸收**。散射只改变光线方向，不改变光线密度和颜色，而吸收相反。光线在物体表面的散射又分为折射（或说透射）和反射两种。光照模型中，**高光反射**属性表示物体表面的反射，**漫反射**属性表示物体表面的折射，吸收和散射。**出射度**用来描述出射光线的数量和方向。辐射度与出射度的比值也就是材质的漫反射和高光属性。

#### 着色

着色是指根据材质属性，光源信息，使用一个等式去计算沿着某观察方向的出射度的过程。此等式就是**光照模型(Lighting Model)**.

#### BRDF光照模型

BRDF全称**Bidirectional Reflectance Distribution Function**。当给定模型表面一个点时，BRDF包含了对该点外观的完整描述。在给出入射光线和辐射度时，BRDF可以给出在某个出射方向上的光照能量分布。由于它是理想和简化后的模型，它并不能真实反映物体和光线的交互，故作为经验模型。
此时不得不说一句计算机图形学第一定律：如果它看起来是对的，那么它就是对的。

### 标准光照模型

标准光照模型的基本方法将进入到摄像机内的光线分为4部分：

- **自发光(emissive)**：描述当给定一个方向时，一个表面本身会向该方向发射多少辐射量。
- **高光反射(specular)**：描述当光线从光源照射到模型表面时，该表面会在完全镜面反射方向上辐射出多少辐射量。
- **漫反射(diffuse)**：描述当光线从光源照射到模型表面时，该表面会向每个方向散射多少辐射量。
- **环境光(ambient)**：描述其他所有的间接光照。

#### 环境光

间接光照是指光线在进入摄像机之前经过了多次物体的反射。在标准光照模型中，使用一种被称为**环境光**的部分来近似间接光照。

### 自发光

自发光是指光线没有经过任何反射之间进入到摄像机中。标准光照模型使用**自发光（颜色）**来计算这个部分的贡献，但该物体并不被当成一个光源，去照亮周围的其他表面。

### 漫反射

漫反射光照符合**兰伯特定律(Lambert's Law)**,反射强度与表面法线和光源方向之间的夹角的余弦值成正比

### 高光反射

高光可以使得物体具有金属的材质。一般采用**Phong模型**或者**Blinn模型**去计算，其中Blinn模型在摄像机和光源距离够远时会快于Phong模型。两者都是经验模型

### 逐像素/逐顶点

通常来讲，在片元着色器中计算光照被称为逐像素光照，而在顶点着色器中计算被称为逐顶点光照。逐像素光照以每个像素为基础得到其法线，然后计算光照模型，又称**Phong着色(Phong Shading,不同于Phong光照模型)**。逐顶点光照又称为**高洛德着色**，它在每个顶点上计算光照，在渲染图元内部进行线性插值，输出成像素颜色。很显然其计算量小于逐像素着色，但由于使用线性插值，将导致明显的棱角现象。

### 实践一 实现漫反射光照模型

- 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)

#### 计算公式

首先给出基本光照模型中漫反射部分的计算公式:



$$c_{diffuse} = (c_{light}·m_{diffuse})max(0, \hat n·I)$$



从公式可以看出，要计算漫反射需要知道4个参数：入射光线的颜色和强度 $c_{light}$ ，材质的漫反射系数 $m_{diffuse}$ ，表面法线$\hat n$以及光源方向 $I$ 。

为防止点积结果为负值，需使用max操作，而CG提供的saturate函数可以达到同样的目的。

#### 逐顶点光照

最终效果类似于下图：

 ![]({{site.url}}\assets\image\illustrations\1_1.png) 

准备工作：

1.  在Unity中新建一个场景，命名为Scene_6_4。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在Window->Rendering->Lighting Seting->Skybox中去掉场景中的天空盒子。
2. 新建Shader(右键Create->Shader->UnitySurfaceShader)并命名为DiffuseVertexLevel；新建材质(右键Create->Material)并命名为DiffuseVertexLevelMat，将新建的Shader拖拽赋给新建材质。
3.  在场景中新建一个胶囊体(菜单栏GameObject->3D Object->Capsule)，将其材质修改为新建材质。
4. 保存场景。

Shader实现：

打开新建的DiffuseVertexLevel(可以在VS Code中打开哦),删除所有已有代码并写入如下代码：

    Shader "Custom/DiffuseVertexLevel"{
        //声明属性
        Properties{
            _Diffuse ("Diffuse", Color) = (1, 1, 1, 1) //可在编辑器面板定义材质自身色彩
        }
        SubShader{
            //顶点或者片元着色器的代码需要写在Pass语义块中
            Pass{
    
                Tags { "LightMode" = "ForwardBase"} //标签，用于定义该Pass在Unity的光照流水线中的角色
                //接下来使用CGPROGRAM和ENDCG包围CG代码片，以定义最重要的顶点着色器和片元着色器代码
                CGPROGRAM
                // #pragma指令告诉Unity定义的着色器的名字：vert 和 frag
                #pragma vertex vert
                #pragma fragment frag
                //为使用Unity的一些内置变量，如_LightColor0
                #include "Lighting.cginc" 
                //定义与声明属性相匹配的变量，fixed精度，范围在0~1之间，表示材质的漫反射属性
                fixed4 _Diffuse; 
                //定义顶点着色器的输入输出结构体
                //为访问顶点法线，在a2v中定义normal变量，为把顶点着色器中计算得到的光照颜色传递给片元着色器，在v2f中定义了一个color变量
                struct a2v {
                    float4 vertex : POSITION;
                    float3 normal : NORMAL;
                };
                struct v2f{
                    float4 pos : SV_POSITION;
                    fixed3 color : COLOR;
                };
                //实现顶点着色器
                v2f vert(a2v v){
                    v2f o;
                    o.pos = UnityObjectToClipPos(v.vertex);
                    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                    fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
                    fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
                    //漫反射颜色 = 直射光颜色 * max(0, cos(光源方向和法线方向夹角)) * 材质自身颜色
                    //其中 max(0, cos(光源方向和法线方向夹角))部分可以改用半兰伯特光照模型以增强背光面的光照效果
                    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));
                    o.color = ambient + diffuse;
                    return o;
                }
                //实现片元着色器
                fixed4 frag(v2f i) : SV_Target {
                    return fixed4(i.color, 1.0);
                }           
                ENDCG
            }
        }
        Fallback "Diffuse"
    }

因为要实现逐顶点的漫反射光照，所以最重要的是顶点着色器的实现：

- 首先定义返回值o;
- 使用Unity内置矩阵UnityObjectToClipPos完成顶点位置从模型空间到裁剪空间的转换；
- 通过UNITY_LIGHTMODEL_AMBIENT得到环境光部分；
- 已知材质漫反射颜色_Diffuse以及顶点法线v.normal,再加上Unity内置的_LightColor0提供光源的颜色和强度信息，_WorldSpaceLightPos0提供光源方向，就可以进行漫反射光照计算了。首先把法线和光源转换到世界空间并用saturate归一化，然后根据公式进行计算，最后与环境光相加得到最终光照

评价：

对于细分程度较高的模型，逐顶点光照的效果已比较好了，然而对于一些细分程度较低的模型，则会产生一些视觉问题，比如，胶囊体的背光面和向光面交界处有一些锯齿。使用接下来的逐像素光照可以一定程度改善这个问题。

另外，我们可以从代码中总结出Unity Shader的基本结构：

    Shader "MyShaderName" {
        Properties {
            //属性
        }
        SubShader {
            //针对显卡A的SubShader
            Pass { 
                //设置渲染状态和标签
                
                //开始CG代码片段
                CGPROGRAM
                //该代码片段的编译指令，例如
    
                #pragma vertex vert
                #pragma fragment frag
                
                //CG代码写在此处
    
                ENDCG
            }
            //其他需要的Pass   
        }
        SubShader {
            //针对显卡B的SubShader
        }
    
        //上述SubShader都失败后用于回调的Unity Shader
        FallBack "VertexLit"
    }


#### 逐像素光照

最终效果类似于下图：

![]({{site.url}}\assets\image\illustrations\1_2.png)



准备工作：

1. 使用Scene_6_4和场景中添加的模型。
2. 新建Shader(右键Create->Shader->UnitySurfaceShader)并命名为DiffusePixelLevel；新建材质(右键Create->Material)并命名为DiffusePixelLevelMat，将新建的Shader拖拽赋给新建材质。
3. 保存场景。

Shader实现：

    Shader "Custom/DiffusePixelLevel"{
        Properties{
            _Diffuse ("Diffuse", Color) = (1, 1, 1, 1) 
        }
        SubShader{
            Pass{
                Tags { "LightMode" = "ForwardBase"}
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag
    
                #include "Lighting.cginc"
    
                fixed4 _Diffuse;
                struct a2v {
                    float4 vertex : POSITION;
                    float3 normal : NORMAL;
                };
                struct v2f{
                    float4 pos : SV_POSITION;
                    fixed3 worldNormal : TEXCOORD0;
                };
                v2f vert(a2v v){
                    v2f o;
                    o.pos = UnityObjectToClipPos(v.vertex);
                    
                    o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
                    return o;
                }
                fixed4 frag(v2f i) : SV_Target {
                    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                    fixed3 worldNormal = normalize(i.worldNormal);
                    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
                    fixed3 color = ambient + diffuse;
    
                    return fixed4(color, 1.0);
                }           
                ENDCG
            }
        }
        Fallback "Diffuse"
    }

其结构与逐顶点的实现比较相似，只不过，计算光照的部分由放在顶点着色器修改到了片元着色器中。

评价：
相较于逐顶点光照，逐像素光照可以得到更加平滑的效果，但是，出现了其他的视觉问题，比如，在光线无法到达的区域，模型外观通常是全黑的，无明暗变化，这使得模型丧失了许多细节。因此有一种改进的模型被提出来去优化这个问题，它就是**半兰伯特(Half Lambert)光照模型**。

#### 半兰伯特光照模型

广义的半兰伯特光照模型公式如下：



$$c_{diffuse} = (c_{light}·m_{diffuse})(\alpha(\hat n ·I) + \beta)$$




其主要特点是没有用max操作来防止点积为负，而是对其结果进行了 $\alpha$ 的缩放再加上一个 $\beta$ 大小的偏移。一般都取0.5。

效果图：

 ![]({{site.url}}\assets\image\illustrations\2_1.png)

准备工作：

1. 使用Scene_6_4和场景中添加的胶囊模型。
2. 新建Shader(右键Create->Shader->UnitySurfaceShader)并命名为HalfLambert；新建材质(右键Create->Material)并命名为HalfLambertMat，将新建的Shader拖拽赋给新建材质。
3. 保存场景。

Shader实现：
将DiffusePixelLevel Shader中的代码粘贴进去，然后修改为：

    Shader "Custom/HalfLambert"{
        Properties{
            _Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
        }
        SubShader{
            Pass{
                Tags { "LightMode" = "ForwardBase"}
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag
    
                #include "Lighting.cginc"
    
                fixed4 _Diffuse;
                struct a2v {
                    float4 vertex : POSITION;
                    float3 normal : NORMAL;
                };
                struct v2f{
                    float4 pos : SV_POSITION;
                    fixed3 worldNormal : TEXCOORD0;
                };
                v2f vert(a2v v){
                    v2f o;
                    o.pos = UnityObjectToClipPos(v.vertex);
                    
                    o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
                    return o;
                }
                fixed4 frag(v2f i) : SV_Target {
                    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                    fixed3 worldNormal = normalize(i.worldNormal);
                    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                    //Compute diffuse term
                    fixed3 HalfLambert = dot(worldNormal, worldLightDir) * 0.5 + 0.5;
                    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * HalfLambert;
                    fixed3 color = ambient + diffuse;
    
                    return fixed4(color, 1.0);
                }           
                ENDCG
            }
        }
        Fallback "Diffuse"
    }

重点是使用半兰伯特公式修改了片元着色器中计算漫反射光照的部分计算得出并使用了HalfLambert变量

最后，让我们看一下三种效果的对比吧 :p

| ![]({{site.url}}\assets\image\illustrations\1_1.png) | ![]({{site.url}}\assets\image\illustrations\1_2.png) | ![]({{site.url}}\assets\image\illustrations\1_3.png) |
|:----------:|:---:|:--------:|
| 逐顶点反射  | 逐像素反射 | 半兰伯特反射|


### 实践二 实现高光反射模型

 - 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)

#### 计算公式

 首先给出基本光照模型中高光反射部分的计算公式:



$$c_{specular} = (c_{light}·m_{specular})max(0, \hat v · r)$$



从公式可以看出，要计算高光反射需要知道4个参数：入射光线的颜色和强度 $c_{light}$ ，材质的漫反射系数 $m_{specular}$ ，视角方向 $\hat v$ 以及反射方向 $r$ 。其中反射方向 $r$ 可以由表面法线 $\hat n$ 和光源方向 $I$ 计算出：



$$r =2(\hat n · \hat I)\hat n - \hat I$$



此外，CG提供了计算反射方向的函数**Reflect**可以直接使用。

#### 逐顶点光照

 最终效果类似于下图：

![]({{site.url}}\assets\image\illustrations\2_1.png)


准备工作：

1. 在Unity中新建一个场景，命名为Scene_6_5。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在Window->Rendering->Lighting Seting->Skybox中去掉场景中的天空盒子。
2. 新建Shader(右键Create->Shader->UnitySurfaceShader)并命名为SpecularVertexLevel；新建材质(右键Create->Material)并命名为SpecularVertexLevelMat，将新建的Shader拖拽赋给新建材质。
3.  在场景中新建一个胶囊体(菜单栏GameObject->3D Object->Capsule)，将其材质修改为新建材质。
4. 保存场景。

Shader实现：

打开新建的SpecularVertexLevel,删除所有已有代码并写入如下代码：

    Shader "Custom/SpecularVertexLevel" { 
        Properties{
            _Diffuse("Diffuse Color", Color) = (1, 1, 1, 1) // 可在编辑器面板定义材质自身色彩
            _Specular("Specular Color", Color) = (1, 1, 1, 1)
            _Gloss("Gloss", Range(8.0, 256)) = 20 // 高光的参数
        }
        SubShader{
            Pass {           
                // 只有定义了正确的LightMode才能得到一些Unity的内置光照变量
                Tags{"LightMode" = "ForwardBase"}
    
                CGPROGRAM
    
                // 包含unity的内置的文件，才可以使用Unity内置的一些变量
                #include "Lighting.cginc" 
                #pragma vertex vert
                #pragma fragment frag
    
                fixed4 _Diffuse;
                fixed4 _Specular;
                float _Gloss;
    
                struct a2v
                {
                    float4 vertex : POSITION; // 告诉Unity把模型空间下的顶点坐标填充给vertex属性
                    float3 normal : NORMAL; // 告诉Unity把模型空间下的法线方向填充给normal属性
                };
    
                struct v2f
                {
                    float4 pos : SV_POSITION; // 声明用来存储顶点在裁剪空间下的坐标
                    float3 color : COLOR; // 用于传递计算出来的漫反射颜色
                };
    
                // 计算顶点坐标从模型坐标系转换到裁剪面坐标系
                v2f vert(a2v v)
                {
                    v2f o;
                    o.pos = UnityObjectToClipPos(v.vertex);
                    // 环境光
                    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                    // 法线方向
                    fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
                    // 光源方向
                    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz); 
                    //漫反射
                    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir)) ; 
                    
                    // 反射方向
                    fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
                    // 视角方向
                    fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(unity_WorldToObject, v.vertex).xyz);
                    //高光反射
                    fixed3 specular = _LightColor0.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
    
                    // 最终颜色 = 漫反射 + 环境光 + 高光反射
                    o.color = diffuse + ambient + specular; // 颜色叠加用加法（亮度通常会增加）
    
                    return o;
                }
    
                // 计算每个像素点的颜色值
                fixed4 frag(v2f i) : SV_Target 
                {
                    return fixed4(i.color, 1);
                }
    
                ENDCG
            }
            
        }
        FallBack "Specular"
    }

需要注意：

- 漫反射部分与之前代码完全一致
- 高光反射部分，首先计算了入射光线关于表面法线的反射方向reflectDir;然后变换后的顶点位置与世界空间下的相机位置相减得到世界空间下的视角方向，最后带入公式得到高光反射部分。
- 此时的回调函数修改为Specular


评价：

使用逐顶点光照可以看出高光部分不平滑，有较大问题。这主要是因为高光反射部分的计算非线性，而顶点着色器中的插值计算是线性的，因此使用逐像素光照可以得到更平滑的效果。


#### 逐像素光照

最终效果类似于下图：

![]({{site.url}}\assets\image\illustrations\2_2.png)


准备工作：

1. 使用Scene_6_5和场景中添加的模型。
2. 新建Shader(右键Create->Shader->UnitySurfaceShader)并命名为SpecularPixelLevel；新建材质(右键Create->Material)并命名为SpecularPixelLevelMat，将新建的Shader拖拽赋给新建材质。
3.  保存场景。

Shader实现：

    Shader "Custom/SpecularPixelLevel" { 
        Properties{
            _Diffuse("Diffuse Color", Color) = (1, 1, 1, 1) 
            _Specular("Specular Color", Color) = (1, 1, 1, 1)
            _Gloss("Gloss", Range(8.0, 256)) = 20 // 高光的参数
        }
        SubShader{
            Pass {           
                // 只有定义了正确的LightMode才能得到一些Unity的内置光照变量
                Tags{"LightMode" = "ForwardBase"}
    
                CGPROGRAM
    
                // 包含unity的内置的文件，才可以使用Unity内置的一些变量
                #include "Lighting.cginc" 
                #pragma vertex vert
                #pragma fragment frag
    
                fixed4 _Diffuse;
                fixed4 _Specular;
                float _Gloss;
    
                struct a2v
                {
                    float4 vertex : POSITION; // 告诉Unity把模型空间下的顶点坐标填充给vertex属性
                    float3 normal : NORMAL; // 告诉Unity把模型空间下的法线方向填充给normal属性
                };
    
                struct v2f
                {
                    float4 pos : SV_POSITION; // 声明用来存储顶点在裁剪空间下的坐标
                    float3 worldNormal : TEXCOORD0; 
                    float3 worldPos : TEXCOORD1;
                };
    
                // 计算顶点坐标从模型坐标系转换到裁剪面坐标系
                v2f vert(a2v v)
                {
                    v2f o;
                    o.pos = UnityObjectToClipPos(v.vertex); 
                    o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject); 
                    o.worldPos = mul(unity_WorldToObject, v.vertex).xyz; 
    
                    return o;
                }
    
                // 计算每个像素点的颜色值
                fixed4 frag(v2f i) : SV_Target 
                {
                    // 环境光
                    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                    // 法线方向
                    fixed3 worldNormal = normalize(i.worldNormal); 
                    // 光源方向
                    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz); 
                    //漫反射
                    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir)) ;
                    
                    // 反射光的方向
                    fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
                    // 视角方向
                    fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                    //高光反射
                    fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
    
                    // 最终颜色 = 漫反射 + 环境光 + 高光反射
                    return fixed4(diffuse + ambient + specular, 1.0); 
                }
    
                ENDCG
            }
            
        }
        FallBack "Specular"
    }

其结构与逐顶点的实现比较相似，只不过，计算光照的部分由放在顶点着色器修改到了片元着色器中。

评价：
相较于逐顶点光照，逐像素光照可以得到更加平滑的效果。至此，已经实现了一个完整的Phong光照模型。（Yeah！鼓掌撒花~）

#### Blinn-Phong模型

之前提到还有另一种高光反射的实现方法——Blinn模型。它引入了一个新的矢量 $\hat h$ ，由对视角方向 $\hat v$ 和光源方向 $I$ 相加再归一化得到:

$$\hat h = \frac{\hat v + \hat I}{|\hat v + \hat I |}$$



其计算公式如下：

$$c_{specular} = (c_{light}· m_{specular})max(0, \hat n · \hat h)^{m_{gloss}} $$  



效果图：
 ![]({{site.url}}\assets\image\illustrations\2_3.png)

准备工作：

1. 使用Scene_6_4和场景中添加的胶囊模型。
2. 新建Shader(右键Create->Shader->UnitySurfaceShader)并命名为BlinnPhong；新建材质(右键Create->Material)并命名为BlinnPhongMat，将新建的Shader拖拽赋给新建材质。
3. 保存场景。

Shader实现：
将SpecularPixelLevel Shader中的代码粘贴进去，然后修改为：

    Shader "Custom/BlinnPhong" { 
        Properties{
            _Diffuse("Diffuse Color", Color) = (1, 1, 1, 1) 
            _Specular("Specular Color", Color) = (1, 1, 1, 1)
            _Gloss("Gloss", Range(8.0, 256)) = 20 
        }
        SubShader{
            Pass {           
    
                Tags{"LightMode" = "ForwardBase"}
    
                CGPROGRAM
    
                #include "Lighting.cginc" 
                #pragma vertex vert
                #pragma fragment frag
    
                fixed4 _Diffuse; // 使用属性
                fixed4 _Specular;
                float _Gloss;
    
                struct a2v
                {
                    float4 vertex : POSITION; 
                    float3 normal : NORMAL;   
                };
    
                struct v2f
                {
                    float4 pos : SV_POSITION;
                    float3 worldNormal : TEXCOORD0; 
                    float3 worldPos : TEXCOORD1;
                };
    
                v2f vert(a2v v)
                {
                    v2f o;
                    o.pos = UnityObjectToClipPos(v.vertex);
                    o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject); 
                    o.worldPos = mul(unity_WorldToObject, v.vertex).xyz; 
    
                    return o;
                }
    
                // 计算每个像素点的颜色值
                fixed4 frag(v2f i) : SV_Target 
                {
                    // 环境光
                    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                    // 法线方向
                    fixed3 worldNormal = normalize(i.worldNormal); 
                    // 光照方向
                    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                    //漫反射
                    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir)) ;
                    
                    // 反射光的方向
                    fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal)); 
                    // 视野方向
                    fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                    fixed3 halfDir = normalize(worldLightDir + viewDir);
                    //高光反射
                    fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(dot(worldNormal, halfDir), 0), _Gloss);
    
                    // 最终颜色 = 漫反射 + 环境光 + 高光反射
                    return fixed4(diffuse + ambient + specular, 1.0); 
                }
    
                ENDCG
            }
            
        }
        FallBack "Specular"
    }


最后，让我们看一下三种效果的对比吧 :p

| ![]({{site.url}}\assets\image\illustrations\2_1.png) | ![]({{site.url}}\assets\image\illustrations\2_2.png) | ![]({{site.url}}\assets\image\illustrations\2_3.png) |
|:----------:|:---:|:--------:|
| 逐顶点反射  | 逐像素反射 | 半兰伯特反射|

评价：

可以看出，BlinnPhong模型的高光部分看起来更大和亮一些。在实际渲染中也倾向于选则BlinnPhong模型。

### 内置函数的妙用

在计算光照模型时，往往需要得到光源方向和视角方向这两个基本信息等。且光源方向其实在不同的光源类型下很不相同。因此可以使用一些Unity的内置函数方便的得到一些基本信息：

- UnityWorldSpaceLightDir : 输入一个模型空间的顶点位置返回世界空间从该顶点到摄像机的观察方向
- UnityWorldSpaceViewDir : 输入一个世界空间的顶点位置返回世界空间从该顶点到摄像机的观察方向
- UnityObjectToWorldNormal : 计算世界空间下的方向

p.s.： 使用时需归一化

#### 示例1

    v2f vert(a2v v) {
        v2f o;
        ...
        o.worldNormal = UnityObjectToWorldNormal(v.normal);
        
        ...
    
    }

#### 示例2

    fixed4 frag(v2f i) : SV_Target {
        ...
        
        fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
    
        ...
    
        fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
    
        ...
    
    }

### 参考

[Unity_Shaders_Book]: https://github.com/candycat1992/Unity_Shaders_Book
[Unity Scripting Reference]: https://docs.unity3d.com/ScriptReference/index.html


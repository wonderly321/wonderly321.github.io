---
layout: page
title: "Lighting Model II 实现漫反射光照模型"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---
本文是Unity基础光照的第二部分，主要内容是实现漫反射光照模型。

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

 ![]({{site.url}}/assets/image/illustrations/1_1.png) 

准备工作：

1.  在Unity中新建一个场景，命名为Scene_6_4。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->UnitySurfaceShader`)并命名为DiffuseVertexLevel；新建材质(右键`Create->Material`)并命名为DiffuseVertexLevelMat，将新建的Shader拖拽赋给新建材质。
3.  在场景中新建一个胶囊体(菜单栏`GameObject->3D Object->Capsule`)，将其材质修改为新建材质。
4. 保存场景。

Shader实现：

打开新建的DiffuseVertexLevel(可以在VS Code中打开哦),删除所有已有代码并写入如下代码：

```glsl
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
```

因为要实现逐顶点的漫反射光照，所以最重要的是顶点着色器的实现：

- 首先定义返回值o;
- 使用Unity内置矩阵UnityObjectToClipPos完成顶点位置从模型空间到裁剪空间的转换；
- 通过UNITY_LIGHTMODEL_AMBIENT得到环境光部分；
- 已知材质漫反射颜色_Diffuse以及顶点法线v.normal,再加上Unity内置的_LightColor0提供光源的颜色和强度信息，_WorldSpaceLightPos0提供光源方向，就可以进行漫反射光照计算了。首先把法线和光源转换到世界空间并用saturate归一化，然后根据公式进行计算，最后与环境光相加得到最终光照

评价：

对于细分程度较高的模型，逐顶点光照的效果已比较好了，然而对于一些细分程度较低的模型，则会产生一些视觉问题，比如，胶囊体的背光面和向光面交界处有一些锯齿。使用接下来的逐像素光照可以一定程度改善这个问题。

另外，我们可以从代码中总结出Unity Shader的基本结构：

```glsl
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
```

#### 逐像素光照

最终效果类似于下图：

![]({{site.url}}/assets/image/illustrations/1_2.png)



准备工作：

1. 使用Scene_6_4和场景中添加的模型。
2. 新建Shader(右键`Create->Shader->UnitySurfaceShader`)并命名为DiffusePixelLevel；新建材质(右键`Create->Material`)并命名为DiffusePixelLevelMat，将新建的Shader拖拽赋给新建材质。
3. 保存场景。

Shader实现：

```glsl
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
```

其结构与逐顶点的实现比较相似，只不过，计算光照的部分由放在顶点着色器修改到了片元着色器中。

评价：
相较于逐顶点光照，逐像素光照可以得到更加平滑的效果，但是，出现了其他的视觉问题，比如，在光线无法到达的区域，模型外观通常是全黑的，无明暗变化，这使得模型丧失了许多细节。因此有一种改进的模型被提出来去优化这个问题，它就是**半兰伯特(Half Lambert)光照模型**。

#### 半兰伯特光照模型

广义的半兰伯特光照模型公式如下：



$$c_{diffuse} = (c_{light}·m_{diffuse})(\alpha(\hat n ·I) + \beta)$$




其主要特点是没有用max操作来防止点积为负，而是对其结果进行了 $\alpha$ 的缩放再加上一个 $\beta$ 大小的偏移。一般都取0.5。

效果图：

 ![]({{site.url}}/assets/image/illustrations/2_1.png)

准备工作：

1. 使用Scene_6_4和场景中添加的胶囊模型。
2. 新建Shader(右键`Create->Shader->UnitySurfaceShader`)并命名为HalfLambert；新建材质(右键`Create->Material`)并命名为HalfLambertMat，将新建的Shader拖拽赋给新建材质。
3. 保存场景。

Shader实现：
将DiffusePixelLevel Shader中的代码粘贴进去，然后修改为：

```glsl
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
```

重点是使用半兰伯特公式修改了片元着色器中计算漫反射光照的部分计算得出并使用了HalfLambert变量

最后，让我们看一下三种效果的对比吧 :p

| ![]({{site.url}}/assets/image/illustrations/1_1.png) | ![]({{site.url}}/assets/image/illustrations/1_2.png) | ![]({{site.url}}/assets/image/illustrations/1_3.png) |
|:----------:|:---:|:--------:|
| 逐顶点反射  | 逐像素反射 | 半兰伯特反射|



### 参考


[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)


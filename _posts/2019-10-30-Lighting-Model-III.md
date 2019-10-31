---
layout: page
title: "Lighting Model III 实现高光反射光照模型"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

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

![]({{site.url}}/assets/image/illustrations/2_1.png)


准备工作：

1. 在Unity中新建一个场景，命名为Scene_6_5。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->UnitySurfaceShader`)并命名为SpecularVertexLevel；新建材质(右键`Create->Material`)并命名为SpecularVertexLevelMat，将新建的Shader拖拽赋给新建材质。
3.  在场景中新建一个胶囊体(菜单栏`GameObject->3D Object->Capsule`)，将其材质修改为新建材质。
4. 保存场景。

Shader实现：

打开新建的SpecularVertexLevel,删除所有已有代码并写入如下代码：

```glsl
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
```

需要注意：

- 漫反射部分与之前代码完全一致
- 高光反射部分，首先计算了入射光线关于表面法线的反射方向reflectDir;然后变换后的顶点位置与世界空间下的相机位置相减得到世界空间下的视角方向，最后带入公式得到高光反射部分。
- 此时的回调函数修改为Specular


评价：

使用逐顶点光照可以看出高光部分不平滑，有较大问题。这主要是因为高光反射部分的计算非线性，而顶点着色器中的插值计算是线性的，因此使用逐像素光照可以得到更平滑的效果。


#### 逐像素光照

最终效果类似于下图：

![]({{site.url}}/assets/image/illustrations/2_2.png)


准备工作：

1. 使用Scene_6_5和场景中添加的模型。
2. 新建Shader(右键`Create->Shader->UnitySurfaceShader`)并命名为SpecularPixelLevel；新建材质(右键`Create->Material`)并命名为SpecularPixelLevelMat，将新建的Shader拖拽赋给新建材质。
3.  保存场景。

Shader实现：

```glsl
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
```

其结构与逐顶点的实现比较相似，只不过，计算光照的部分由放在顶点着色器修改到了片元着色器中。

评价：
相较于逐顶点光照，逐像素光照可以得到更加平滑的效果。至此，已经实现了一个完整的Phong光照模型。（Yeah！鼓掌撒花~）

#### Blinn-Phong模型

之前提到还有另一种高光反射的实现方法——Blinn模型。它引入了一个新的矢量 $\hat h$ ，由对视角方向 $\hat v$ 和光源方向 $I$ 相加再归一化得到:

$$\hat h = /frac{\hat v + \hat I}{|\hat v + \hat I |}$$



其计算公式如下：

$$c_{specular} = (c_{light}· m_{specular})max(0, \hat n · \hat h)^{m_{gloss}} $$  



效果图：

 ![]({{site.url}}/assets/image/illustrations/2_3.png)

准备工作：

1. 使用Scene_6_4和场景中添加的胶囊模型。
2. 新建Shader(右键`Create->Shader->UnitySurfaceShader`)并命名为BlinnPhong；新建材质(右键`Create->Material`)并命名为BlinnPhongMat，将新建的Shader拖拽赋给新建材质。
3. 保存场景。

Shader实现：
将SpecularPixelLevel Shader中的代码粘贴进去，然后修改为：

```glsl
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
```

最后，让我们看一下三种效果的对比吧 :p

| ![]({{site.url}}/assets/image/illustrations/2_1.png) | ![]({{site.url}}/assets/image/illustrations/2_2.png) | ![]({{site.url}}/assets/image/illustrations/2_3.png) |
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


[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)


---
layout: post
title: "Unity纹理-渐变纹理"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

上节介绍了凹凸映射(bump mapping)的实践。本节将实践使用渐变纹理控制漫反射光照。

首先回顾一下之前漫反射光照的计算过程：使用表面法线和光照方向的点积结果与材质的反射率相乘来得到表面的漫反射光照。但这种方法可能不够灵活，因此使用渐变纹理的方法逐渐流行起来。



#### 效果图

| ![]({{site.url}}/assets/image/illustrations/4_1.png)   | ![]({{site.url}}/assets/image/illustrations/4_2.png)   | ![]({{site.url}}/assets/image/illustrations/4_3.png)   |
| -------------------------------------------- | -------------------------------------------- | -------------------------------------------- |
| ![]({{site.url}}/assets/image/illustrations/4_1_1.png) | ![]({{site.url}}/assets/image/illustrations/4_2_1.png) | ![]({{site.url}}/assets/image/illustrations/4_3_1.png) |





### 实践



- 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)



**准备工作**



1. 在Unity中新建一个场景，命名为Scene_7_3。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为RampTexture；新建材质(右键`Create->Material`)并命名为RampTextureMat，将新建的Shader拖拽赋给新建材质。
3. 向场景中拖拽一个Suzzane模型(在项目资源Model文件夹下)，将其材质修改为新建材质。
4. 保存场景。

Shader实现:

打开新建的RampTexture，删除所有已有代码并写入如下代码：

```glsl
Shader "Custom/RampTexture"{
    Properties {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        //声明渐变纹理属性_RampTex
        _RampTex ("Ramp Tex", 2D) = "white" {}
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    Subshader {
        Pass {
            Tags { "LightMode" = "ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"
            fixed4 _Color;
            //定义与Properties中相应的纹理属性变量
            sampler2D _RampTex;
            float4 _RampTex_ST;
            float4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0; 
                float3 worldPos : TEXCOORD1;
                float2 uv : TEXCOORD2;
            };

            // 计算顶点坐标从模型坐标系转换到裁剪面坐标系
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal); 
                o.worldPos = mul(unity_WorldToObject, v.vertex).xyz;
                使用内置的TRANSFORM_TEX宏来计算经过平铺和偏移后的纹理坐标
                o.uv = TRANSFORM_TEX(v.texcoord, _RampTex);

                return o;
            }
            //通过 scale/bias 属性转换2D UV
            #define TRANSFORM_TEX(tex, name)(tex.xy * name##_ST.xy + name##_ST.zw)


            // 计算每个像素点的颜色值
            fixed4 frag(v2f i) : SV_Target {
                
                // 法线方向，反过来相乘就是从模型到世界的变换
                fixed3 worldNormal = normalize(i.worldNormal); 
                // 光照方向。
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                //使用纹理去采样漫反射颜色 ，通过0.5倍的缩放和偏移将halfLambert范围映射到[0,1]之间
                //采用半兰伯特 mo'x
                fixed halfLambert = 0.5 * dot(worldNormal, worldLightDir) + 0.5;
                //使用halfLambert来构建u，v方向上的纹理坐标
                fixed3 diffuseColor = tex2D(_RampTex, fixed2(halfLambert, halfLambert)).rgb * _Color.rgb;         
                //漫反射
                fixed3 diffuse = _LightColor0.rgb * diffuseColor; 
                // 视野方向
                fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
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



保存返回后，在RampTextureMat面板上可以使用项目中的纹理资源RampTexture1~3(或其他图片)对Main Tex属性进行赋值并查看不同变化。



### 参考

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)
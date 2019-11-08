---
layout: post
title: "Unity纹理-遮罩纹理"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

之前的部分记录了凹凸纹理，渐变纹理等，接下来是纹理记录的最后一部分——遮罩纹理。那么什么是遮罩呢？举个例子，之前光照模型的实现中，高光的计算是全局的，也就是所有的像素使用同样大小的高光强度和指数。但有时，我们希望模型表面的反光是各处有些不同的。为了得到这种更细腻的效果，遮罩纹理就出场了~

#### 基本思路

通过采样得到遮罩纹理的纹素值，然后使用其中某个（或几个）通道的值与某种表面属性进行相乘，当该通道的值为0时，可以保护表面不受该属性影响。





#### 效果对比图

| ![]({{site.url}}/assets/image/illustrations/5_1.png) | ![]({{site.url}}/assets/image/illustrations/5_2.png) |
| ------------------------------------------ | ------------------------------------------ |
| 漫反射+未遮罩的高光反射                    | 漫反射+遮罩的高光反射                    |

我的主观视觉上觉得加了遮罩确实看上去更逼真和看上去更舒服了呀~


### 实践



- 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)



**准备工作**



1. 在Unity中新建一个场景，命名为Scene_7_4。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为MaskTexture；新建材质(右键`Create->Material`)并命名为MaskTextureMat，将新建的Shader拖拽赋给新建材质。
3. 在场景中创建一个Capsule模型并拖拽到合适位置，将其材质修改为新建材质。
4. 保存场景。

Shader实现:

打开新建的MaskTexture，删除所有已有代码并写入如下代码：

```glsl
Shader "Custom/MaskTexture" { 
    Properties{
        _Color("Color Tint", Color) = (1, 1, 1, 1)
        _MainTex("Main Tex", 2D) = "white"{}
        _BumpMap("Normal Map", 2D) = "bump" {}
        _BumpScale("Bump Scale", Float) = 1.0
        //声明需要使用的高光反射遮罩纹理
        _SpecularMask("Specular Mask", 2D) = "white" {}
        //声明用于控制遮罩影响度的系数
        _SpecularScale("Specular Scale",Float) = 1.0
        _Specular("Specular", Color) = (1, 1, 1, 1)
        _Gloss("Gloss", Range(8.0, 256)) = 20
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
 
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;//此处纹理属性变量是3个纹理共用的，目的是节省需要存储的纹理坐标数目
            sampler2D _BumpMap;
            float _BumpScale;
            //定义对应于属性的变量
            sampler2D _SpecularMask;
            float _SpecularScale;
            fixed4 _Specular;
            float _Gloss;

            struct a2v
            {
                float4 vertex : POSITION;   
                float3 normal : NORMAL;   
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION; 
                float2 uv : TEXCOORD0;
                float3 lightDir : TEXCOORD1; 
                float3 viewDir : TEXCOORD2;
            };

            // 对光照和视角方向进行了坐标空间的转换
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex); //坐标从模型空间转换到剪裁空间
                o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                TANGENT_SPACE_ROTATION;
                o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz; 
                o.viewDir = mul(rotation, ObjSpaceViewDir(v.vertex)).xyz;

                return o;
            }
            // 遮罩纹理使用在片元着色器中
            fixed4 frag(v2f i) : SV_Target {
                fixed3 tangentLightDir = normalize(i.lightDir);
                fixed3 tangentViewDir = normalize(i.viewDir); 
                fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uv));
                tangentNormal.xy *= _BumpScale;
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

                //使用纹理去采样漫反射颜色
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir)) ; // 颜色融合用乘法
                // 视野方向
                fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
                //高光反射
                计算时，先对遮罩纹理_SpecularMask进行采样，计算掩码值并和_SpecularScale相乘，一起控制高光反射强度
                fixed specularMask = tex2D(_SpecularMask, i.uv).r * _SpecularScale;
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(dot(tangentNormal, halfDir), 0), _Gloss)*specularMask;//此处rgb分量存储的是同一个值，而实际时每个颜色通道应该储存不同的表面属性

                // 最终颜色 = 漫反射 + 环境光 + 高光反射
                return fixed4(diffuse + ambient + specular, 1.0); 
            }

            ENDCG
        }
        
    }
    FallBack "Specular"
}
```



保存返回后，在MaskTextureMat面板上可以使用项目中的纹理资源RampTexture1~3(或其他图片)对Main Tex以及其他Tex属性进行赋值并查看不同变化。



### 参考

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)
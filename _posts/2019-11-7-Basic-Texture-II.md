---
layout: post
title: "Unity纹理-凹凸映射"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

上节介绍了许多纹理的导入，使用和属性的知识点。本节将就凹凸映射(bump mapping)进行实践。

凹凸映射是指用一张纹理去修改模型表面的法线信息，从而使模型表面产生看上去“凹凸不平”的效果。这种方法不会真正的改变模型的顶点位置。

目前有两种主要的方法可以用来进行凹凸映射：**高度映射(height mapping)** 和 **法线映射(notmal mapping)**。前者是使用**高度纹理(height map)**模拟**表面位移(displacement)**,得到修改后的法线值。后者则是使用一张**法线纹理(normal map)**直接存储表面法线。

### 高度纹理

使用高度图实现凹凸映射时，高度图中存储的是表示模型表面局部的海拔高度的强度值(intensity),颜色越深，位置越深凹，颜色越浅，位置越外凸。该方法很直观的展示了模型表面的凹凸，但是计算偏复杂，实时计算时因需要计算像素灰度值来得到法线信息，所以十分消耗性能。实际应用中，高度图通常和法线映射一起用。用于给出表面凹凸的额外信息。

### 法线纹理

法线纹理中存储的是表面的法线方向。但法线方向的分量范围在[-1,1],而像素范围为[0,1],因此需要做一个映射：



$$pixel = \frac{normal+1}{2}$$



这就要求我们在Shader中对法线纹理进行采样后，还需对结果进行一次映射逆运算以得到原来的法线方向，即：



$$nomral = pixel \times 2 - 1 $$



#### 模型空间和切线空间的法线纹理



由于方向是相对于坐标空间的，所以必须确认法线纹理存储的法线方向是在哪个坐标空间当中。例如，模型自带的法线是定义在模型空间中的。接下来需要明确两个概念：

- 模型空间的法线纹理(object-space normal map)

它是指将修改后的模型空间中的表面法线直接地存储在一张纹理中。其所有法线所在的坐标空间是同一个坐标空间，即，模型空间，而每个点存储的法线方向是各异的。

- 切线空间的法线纹理(tangent-space normal map)

它是指采用模型顶点的切线空间来存储法线，其实模型的每个顶点都有一个属于自己的切线空间，其原点就是该顶点本身，z轴是顶点的法线方向($n$)，x轴是顶点的切线方向($t$),,y轴由法线与切线叉积而得，因此也被称为副切线(bitangent, $b$)或副法线。其所有法线方向所在的坐标空间是表面每点各自的切线空间，是各异的。所有，这种法线纹理存储的是每个点在各自切线空间中的法线扰动方向，换句话说， 若一个点的法线方向不变，那么在它切线空间中新的法线方向就是z轴方向。

总结一下：

模型空间下的法线纹理具有以下优点：

- 实现简单，更直观，不需要模型原始的法线和切线信息，所以计算量小。
- 因为模型空间下的法线纹理存储的是同一坐标系下的法线信息，因此边界处插值平滑，所以在纹理坐标的缝合处和尖锐的边角部分，可见的突变(缝隙)较少，可以提供平滑的边界。

但美术人员更喜欢使用切线空间下的法线纹理，因为使用切线空间具有更多的优势：

- 由于切线空间记录的法线信息是相对的，所以即使把该纹理用到一个完全不同的网格上也可以得到合理的结果，所以自由度很高。
- 同理，使用切线空间下的纹理记录可以移动一个纹理的UV坐标来实现动画效果。
- 由于自由度高，所以一个法线纹理是可以重用在不同的网格上的。
- 由于切线空间下的法线的z方向总为正，可由x，y方向推导出来，所以可以不存储，实现压缩的效果。

接下来的实践中使用的是切线空间下的法线纹理。



### 凹凸纹理实践



- 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)

#### 前言

计算光照模型时，需要将各个方向矢量统一到统一坐标空间下。实践时有两种思路：

- 在切线空间下进行光照计算。此时需要把光照方向，视角方向变换到切线空间下。
- 在世界空间下进行光照计算。此时需要把采样得到的法线方向变换到世界空间下，再和世界空间下的光照方向，视角方向进行计算

第一种思路在顶点着色器中就可以完成对光照方向和视角方向的变换，因而效率高于第二种；但是就通用性来看，第二种方法允许我们在世界空间下进行一些计算，比如Cubemap的使用。另外，还有其他的思路，比如使用模型空间等。

但模型空间和切线空间仍是最为常用的两种空间，以下将分别进行实现。



#### 切线空间下的光照模型计算

**基本思路**

首先在片元着色器中，把视角方向、光线方向从模型空间变换到切线空间，通过纹理采样得到切线空间下的法线，然后再与切线空间下的视角方向、光线方向等进行计算，得到最终光照结果。

**效果图**



![]({{site.url}}/assets/image/illustrations/3_1.png)



**准备工作**



1. 在Unity中新建一个场景，命名为Scene_7_2_3。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为NormalMapTangentSpace；新建材质(右键`Create->Material`)并命名为NormalMapTangentSpaceMat，将新建的Shader拖拽赋给新建材质。
3. 在场景中新建一个胶囊体(菜单栏`GameObject->3D Object->Capsule`)，将其材质修改为新建材质。
4. 保存场景。

Shader实现:

打开新建的NormalMapTangentSpace，删除所有已有代码并写入如下代码：

```glsl
Shader "Custom/NormalMapTangentSpace"{
    Properties{
        _Color ("Color Tint", Color) = (1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
        
        //增加法线纹理属性，以及用于控制凹凸程度的属性，“bump”是Unity内置的法线纹理，_BumpScale为0时，表示法线纹理不会对光照产生影响
        _BumpMap ("Normal Map", 2D) = "Bump" {}
        _BumpScale ("Bump Scale", Float) = 1.0     
    }
    SubShader{
        Pass{
            Tags{"LightMode" = "ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;          
            fixed4 _Specular;
            float _Gloss;   
            
            //在CG代码块中声明和上述属性类型匹配的变量
            //为得到该纹理的属性(平铺和偏移系数)，定义_MainTex_ST和_BumpMap_ST变量
            sampler2D _MainTex;
            float4 _MainTex_ST;
            sampler2D _BumpMap;
            float4 _BumpMap_ST;
            float _BumpScale;
            
            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                //使用TANGENT语义描述float4类型的tangent变量，以告诉Unity把顶点的切线方向填充到tangent变量
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };
            struct v2f {
                float4 pos : POSITION;
                float4 uv : TEXCOORD0;
                //同理，添加两个变量存储变换后的光照和视角方向
                float3 lightDir : TEXCOORD1;
                float3 viewDir : TEXCOORD2;
            };
            v2f vert (a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                
                //由于使用了两张纹理故xy分量存储_MainTex的纹理坐标，而zw分量存储_BumpMap的纹理坐标
                o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                o.uv.zw = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                
                //Unity提供的内置函数，用来得到旋转变换矩阵
                TANGENT_SPACE_ROTATION;
                
                //用Unity内置函数得到模型空间下的光照和视角方向，再利用变换矩阵rotation把它们从模型空间变换到切线空间
                o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
                o.viewDir = mul(rotation, ObjSpaceViewDir(v.vertex)).xyz;

                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target 
            {
                fixed3 tangentLightDir = normalize(i.lightDir);
                fixed3 tangentViewDir = normalize(i.viewDir);
                
                //利用tex2D对法线纹理进行采样
                fixed4 packedNormal = tex2D(_BumpMap, i.uv.zw);
                //然后映射逆运算得到实际的法线分量
                fixed3 tangentNormal;
                tangentNormal = UnpackNormal(packedNormal);
                tangentNormal.xy *= _BumpScale;
                //z分量由xy推导而得
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

                //使用纹理去采样漫反射颜色
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                // 漫反射 
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir)) ;
                // 视野方向
                fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
                // 高光反射
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(dot(tangentNormal, halfDir), 0), _Gloss);
                // 最终颜色 = 漫反射 + 环境光 + 高光反射
                return fixed4(diffuse + ambient + specular, 1.0); 
            }

            ENDCG
        }
    }
    Fallback "Specular"  
}
```



保存返回后，在NormalMapTangentSpaceMat面板上可以使用项目中的纹理资源Brick_Diffuse.jpg，Brick_Normal.jpg纹理(或其他图片)对Main Tex属性进行赋值。可以通过调试面板中的Bump Scale属性来改变模型的凹凸程度。

调整效果图如下：

| ![]({{site.url}}/assets/image/illustrations/3_1.png) | ![]({{site.url}}/assets/image/illustrations/3_2.png) | ![]({{site.url}}/assets/image/illustrations/3_3.png) |
| :----------------------------------------: | :----------------------------------------: | :----------------------------------------: |
|              Bump Scale = -1               |               Bump Scale = 0               |               Bump Scale = 1               |



#### 世界空间下的光照模型计算

**基本思路**

在顶点着色器中，计算从切线空间到世界空间的变换矩阵并传递给片元着色器，然后在片元着色器中把法线纹理的法线方向从切线空间变换到世界空间，最终进行计算得到光照结果。



**效果图**

与上一个实践完全一样



**准备工作**



1. 使用上一实践中的场景3。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为NormalMapWorldSpace；新建材质(右键`Create->Material`)并命名为NormalMapWorldSpaceMat，将新建的Shader拖拽赋给新建材质。
3. 将场景中胶囊体材质修改为新建材质。
4. 保存场景。

Shader实现:

打开新建的NormalMapWorldSpace，粘贴入上节代码并修改如下：

```glsl
Shader "Custom/NormalMapWorldSpace"{
    Properties{
        _Color ("Color Tint", Color) = (1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _BumpMap ("Normal Map", 2D) = "Bump" {}
        _BumpScale ("Bump Scale", Float) = 1.0
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader{
        Pass{
            Tags{"LightMode" = "ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            sampler2D _BumpMap;
            float4 _BumpMap_ST;
            float _BumpScale;
            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };
            struct v2f {
                float4 pos : POSITION;
                float4 uv : TEXCOORD0;
                //修改处(1):使v2f结构体包含从切线空间到世界空间的变换矩阵
                float4 TtoW0 : TEXCOORD1;
                float4 TtoW1 : TEXCOORD2;
                float4 TtoW2 : TEXCOORD3;
            };
            v2f vert (a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                o.uv.zw = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				//修改处(2):计算世界空间下的顶点切线、副切线和法线的矢量表示
                float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
                fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);
                fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w;
                //按列摆放得到切线空间到世界空间的变换矩阵
                o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
                o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
                o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);

                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target {
            	//首先从TtoW系列的w分量中构建世界空间下的坐标
                float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
                //然后使用内置函数得到世界空间下的光照和视角函数
                fixed3 LightDir = normalize(UnityWorldSpaceLightDir(worldPos));
                fixed3 ViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
                //使用内置的UnpackNormal函数对法线纹理进行采样和解码，并使用_BumpScale进行缩放
                fixed3 bump = UnpackNormal(tex2D(_BumpMap,i.uv.zw));
                bump.xy *= _BumpScale;
                //最后通过点乘操作把法线变换到世界空间下
                bump.z = sqrt(1.0 - saturate(dot(bump.xy, bump.xy)));
                bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.x, bump), dot(i.TtoW2.xyz, bump)));

                //使用纹理去采样漫反射颜色
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                // 漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(bump, LightDir)) ;
                // 视野方向
                fixed3 halfDir = normalize(LightDir + ViewDir);
                // 高光反射
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(dot(bump, halfDir), 0), _Gloss);
                // 最终颜色 = 漫反射 + 环境光 + 高光反射
                return fixed4(diffuse + ambient + specular, 1.0);
            }

            ENDCG
        }
    }
    Fallback "Specular"  
}
```



保存返回后，在NormalMapWorldSpaceMat面板上可以使用项目中的纹理资源Brick_Diffuse.jpg，Brick_Normal.jpg纹理(或其他图片)对Main Tex属性进行赋值。可以通过调试面板中的Bump Scale属性来改变模型的凹凸程度。

就不放图啦，和模型空间下的光照模型效果一样哦~

### 参考

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)
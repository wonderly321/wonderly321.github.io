---
layout: post
title: "Unity透明效果-透明度融合"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

透明度测试的实现实际上与对待普通的不透明物体几乎是一样的，只是在片元着色器中增加了对透明度阈值的判断和对片元进行裁剪的代码；而透明度混合的实现要比透明度测试复杂一些。



**原理回顾**

透明度混合可以得到真正的半透明效果。它会使用当前片元的透明度作为混合因子，与已经存储在颜色缓冲中的颜色值进行混合，得到新的颜色。它需要关闭深度写入，因而我们需要非常注意渲染的顺序。另外，透明度混合虽然关闭了深度写入，但没有关闭深度测试，只不过此时比较深度值将决定是否进行混合操作。在这种情况下，不透明物体出现在透明物体前的场景中，即使先渲染了不透明物体，也可以正常地遮挡住透明物体的部分。换句话说，对于透明度混合而言，深度是只读的。需要注意的是，要格外小心物体的渲染顺序



> 通常使用Unity提供的Blend命令进行透明度混合。其命令语义如下表：
>
> - Blend Off : 关闭混合
> - Blend SrcFactor DstFactor : 开启混合，并设置混合因子。颜色缓冲中存入的值是由源颜色（该片元产生的颜色）和SrcFactor的乘积与目标颜色（已经存在于颜色缓冲中的颜色）与DstFactor的乘积相加得到的。
> - Blend SrcFactor DstFactor, SrcFactorA DstFactorA : 和上面几乎一样只是混合因子不同
> - BlendOp BlendOperation : 使用BlendOperation而不是简单乘加去做混合



本节使用第二种语义。



**实践**



- 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)



**准备工作**



1. 在Unity中新建一个场景，命名为Scene_8_4。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为AlphaBlend；新建材质(右键`Create->Material`)并命名为AlphaBlendMat，将新建的Shader拖拽赋给新建材质。
3. 在场景中创建一个立方体，并拖拽到合适位置，将其材质修改为新建材质。
4. 保存场景。

其他准备：一张透明纹理，其中每个方格的透明度不同（从左到右，从上到下依次是80%，70%，60%，50%）

![]({{site.url}}/assets/image/illustrations/6_2.png)

**Shader实现**:

打开新建的AlphaBlend，删除所有已有代码并把AlphaTest代码粘贴进来，并作修改得到：

```glsl
Shader "Custom/AlphaBlend"
{
    Properties
    {
        _Color ("Main Tint", Color) = (1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        //使用新的属性_AlphaScale来代替原先的_Cutoff属性，用以控制透明纹理的整体透明度
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 1 
    }
    SubShader
    {
    	//如综述中所示，需要注意，透明度混合用到的渲染队列和类型应该是Transparent
        Tags { "Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
        //在pass中，为透明度混合进行合适的混合状态设置
        Pass {
            Tags { "LightMode"="ForwardBase" }
            ZWrite Off //关闭深度写入
            Blend SrcAlpha OneMinusSrcAlpha //开启Blend混合，设置源颜色和目标颜色混合因子

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _AlphaScale;

            struct a2v{
                float4 vertex : POSITION; 
                float3 normal : NORMAL;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f{
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0; 
                float3 worldPos : TEXCOORD1;
                float2 uv : TEXCOORD2;
            };
            v2f vert(a2v v){
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);

                return o;
            }
            //移除透明度测试的代码，并设置该片元着色器返回值中的透明通道值为纹理像素透明通道的值玉与材质参数_AlphaScale的乘积
            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                
                fixed4 texColor = tex2D(_MainTex, i.uv);

                //使用纹理去采样漫反射颜色
                fixed3 albedo = texColor.rgb * _Color.rgb;
                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir)) ; // 颜色融合用乘法
        
                // 最终颜色 = 漫反射 + 环境光 + 高光反射
                return fixed4(diffuse + ambient, texColor.a * _AlphaScale); 
            }       
            ENDCG
        }
    }
    FallBack "Transparent/VertexLit"
}

```



保存返回后，使用之前准备的透明纹理，并调节Alpha Scale参数可以得到如下效果：

| ![]({{site.url}}/assets/image/illustrations/7_1_1.png) | ![]({{site.url}}/assets/image/illustrations/7_1_2.png) | ![]({{site.url}}/assets/image/illustrations/7_1_3.png) |
| :----------------------------------------------------: | :----------------------------------------------------: | :----------------------------------------------------: |
|                      Alpha = 0.55                      |                      Alpha = 0.65                      |                      Alpha = 0.75                      |



从图中可以看出，透明度融合得到的效果要更真实一些~

但是！由于关闭深度写入，若模型本身具有复杂的结构遮挡，即模型网格之间有相互交叉的结构时，就会产生因排序错误而导致的错误透明效果。比如下图：

![]({{site.url}}/assets/image/illustrations/7_2.png)



**双面渲染**

上一小节中已经实践了透明度测试的双面渲染效果，但实现透明度混合的双面效果就比较麻烦了。这是因为直接关闭剔除功能无法保证同一个物体正面和背面图元的渲染顺序。因此实现透明度融合的双面渲染需要两个Pass，其中第一个Pass只渲染背面，第二个Pass只渲染正面，由于Unity会顺序执行SubShader中的各个Pass,因此可以保证背面总是在正面之前被渲染，从而可以得到正确的深度渲染关系。

**实践** 

**准备工作**



1. 在Unity中新建一个场景，命名为Scene_8_7_2。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为AlphaBlendBothSide；新建材质(右键`Create->Material`)并命名为AlphaBlendBothSideMat，将新建的Shader拖拽赋给新建材质。
3. 在场景中创建一个立方体，并拖拽到合适位置，将其材质修改为新建材质。
4. 保存场景。

其他准备：一张透明纹理，其中每个方格的透明度不同（从左到右，从上到下依次是80%，70%，60%，50%）

![]({{site.url}}/assets/image/illustrations/6_2.png)

**Shader实现**:

打开新建的AlphaBlendBothSide，删除所有已有代码并把AlphaBlend代码粘贴进来，并作修改得到：

```glsl
Shader "Custom/AlphaBlendBothSided"
{
    Properties
    {
        _Color ("Main Tint", Color) = (1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 1 
    }
    SubShader
    {
        Tags { "Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
        Pass {
            Tags { "LightMode"="ForwardBase" }
            //剔除操作
            Cull Front
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _AlphaScale;

            struct a2v{
                float4 vertex : POSITION;  
                float3 normal : NORMAL; 
                float4 texcoord : TEXCOORD0;
            };

            struct v2f{
                float4 pos : SV_POSITION; 
                float3 worldNormal : TEXCOORD0; 
                float3 worldPos : TEXCOORD1;
                float2 uv : TEXCOORD2;
            };
            v2f vert(a2v v){
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);

                return o;
            }
            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                
                fixed4 texColor = tex2D(_MainTex, i.uv);

                //使用纹理去采样漫反射颜色
                fixed3 albedo = texColor.rgb * _Color.rgb;
                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir)) ;
        
                // 最终颜色 = 漫反射 + 环境光 + 高光反射
                return fixed4(diffuse + ambient, texColor.a * _AlphaScale); 
            }       
            ENDCG
        }
        Pass {
            Tags { "LightMode"="ForwardBase" }
            //剔除操作
            Cull Back
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _AlphaScale;

            struct a2v{
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f{
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0; 
                float3 worldPos : TEXCOORD1;
                float2 uv : TEXCOORD2;
            };
            v2f vert(a2v v){
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);

                return o;
            }
            // 计算每个像素点的颜色值
            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                
                fixed4 texColor = tex2D(_MainTex, i.uv);

                //使用纹理去采样漫反射颜色
                fixed3 albedo = texColor.rgb * _Color.rgb;
                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                //漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir)) ;
        
                // 最终颜色 = 漫反射 + 环境光 + 高光反射
                return fixed4(diffuse + ambient, texColor.a * _AlphaScale); 
            }       
            ENDCG
        }
    }
    FallBack "Transparent/VertexLit"
}

```



很简单，只需新建一个AlphaTestBothSideMat和对应得AlphatTestBothSide Shader的代码并复制粘贴AlphaTest代码，在pass中添加一行:

此时我们可以透过镂空区域看到立方体内部的结构，效果如下：

| ![]({{site.url}}/assets/image/illustrations/7_1_4.png) | ![]({{site.url}}/assets/image/illustrations/7_1_5.png) | ![]({{site.url}}/assets/image/illustrations/7_1_6.png) |
| :----------------------------------------------------: | :----------------------------------------------------: | :----------------------------------------------------: |
|                      Alpha = 0.55                      |                      Alpha = 0.65                      |                      Alpha = 0.75                      |





### 参考

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)
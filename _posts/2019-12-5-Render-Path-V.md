---
layout: post
title: "Unity渲染路径——光源种类"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

了解过光源种类和它们的几何定义后，我们来探索一下在Unity Shader中访问它们的5个属性：位置、方向、颜色、强度以及衰减。（本节使用前向渲染路径）

#### 实践 一 ：前向渲染中处理不同光源

- 运行平台：

  **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

  [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)



**准备工作**



1. 在Unity中新建一个场景，命名为Scene_9_2_2_1。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为ForwardRendering；新建材质(右键`Create->Material`)并命名为ForwardRenderingMat，将新建的Shader拖拽赋给新建材质。
3. 在场景中新建一个胶囊体（Capsule），并拖拽到合适位置，将其材质修改为新建材质。
4. 为了让物体受多个光源影响，我们再新建一个点光源，设定颜色为绿色，以和平行光作区分。
5. 保存场景。



**Shader实现**:

打开新建的ForwardRendering，删除所有已有代码并写入如下代码：

```glsl
Shader "Custom/ForwardRendering" { 
    Properties {
		_Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Tags { "RenderType"="Opaque" }
		//首先定义第一个Pass:Base Pass。为此需要设立该pass的渲染路径标签
		Pass {
			// 该Pass中计算环境光和最重要的平行光（该平行光以逐像素方式处理）
			Tags { "LightMode"="ForwardBase" }
		    
			CGPROGRAM
			//不可缺少的编译指令
			#pragma multi_compile_fwdbase	//保证在Shaderb中使用光照衰减等光照变量可以被正确赋值
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
			};
            v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {				
				//环境光的计算只有一次，自发光也是
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				//处理平行光，若场景中包含多个平行光，则Unity会选择最亮的平行光传递给Base Pass进行逐像素处理
				//其他平行光会按照逐顶点或者在Additional Pass中按逐像素方式处理
				//如果场景中没有平行光，则Base Pass会当成全黑的光源处理
				//对于Base Pass来说，其处理的逐像素光源一定是平行光
				fixed3 worldNormal = normalize(i.worldNormal);
				//使用_WorldSpaceLightPos0可以得到该平行光的方向
				fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				//使用_LightColor0可以得到该平行光的颜色和强度
			 	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));

			 	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
			 	fixed3 halfDir = normalize(worldLightDir + viewDir);
			 	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
                //由于平行光没有衰减，直接设定衰减值为1.0
				fixed atten = 1.0;
				
				return fixed4(ambient + (diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
        Pass {
			//Additional Pass 首先设定渲染路径标签,该Pass中处理的光源类型为平行光，点光源或是聚光灯光源。
			Tags { "LightMode"="ForwardAdd" }
			//使用blend开启和设置混合模式，以便additional pass中得到的光照结果可以和帧缓存之中的光照结果相叠加，不开启则意味着结果的覆盖
			Blend One One //Blend还有许多其他的混合系数，比如Blend SrcAlpha One
		
			CGPROGRAM
			
			// 使用必不可少的Additional Pass的编译指令
			#pragma multi_compile_fwdadd
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed3 worldNormal = normalize(i.worldNormal);
				//方向需要不同光源类型区别对待
				#ifdef USING_DIRECTIONAL_LIGHT
					//平行光的方向由_WorldSpaceLightPos0直接得到
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				#else
					//点光源或是聚光灯则是用_WorldSpaceLightPos0减去世界空间下的顶点位置
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
				#endif
				
				// 颜色与强度仍使用_LightColor0来得到
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
				
				fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
				fixed3 halfDir = normalize(worldLightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				//衰减也是区分不同光源来处理
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed atten = 1.0;
				#else
					#if defined (POINT)
				        float3 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1)).xyz;
				        //为避免计算量较大的公式定义计算，Unity提供纹理查找表（Lookup Table,LUT）以在片元着色器中得到光源的衰减
				        fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #elif defined (SPOT)
				    	//比如聚光灯光源的计算，首先得到光源空间下的坐标lightCoord
				        float4 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1));
				        //然后使用该坐标对衰减纹理进行采样得到衰减值
				        fixed atten = (lightCoord.z > 0) * tex2D(_LightTexture0, lightCoord.xy / lightCoord.w + 0.5).w * tex2D(_LightTextureB0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #else
				        fixed atten = 1.0;
				    #endif
				#endif

				return fixed4((diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
	}
	FallBack "Specular"
}
```



实验结果如下：

![]({{site.url}}/assets/image/illustrations/9_1.png)

#### 实践 二 ： Base Pass和Additional Pass 的调用 

实践一中给出了前向渲染中Unity是如何决定各个不同光源的不同处理方式。本次实践中包含了更多的光源

- 运行平台：

  **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

  [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)



**准备工作**



1. 在Unity中新建一个场景，命名为Scene_9_2_2_2。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 调节平行光的颜色为绿色。
3. 在场景中新建一个胶囊体（Capsule），并拖拽到合适位置，将其材质修改为实践一中建立的ForwardRenderingMat材质。
4. 新建3个点光源，设定颜色为红色，以和平行光作区分。
5. 保存场景。

此处需要想要纠正之前描述中的一个错误，Unity3d 默认的`Edit`->`Project Setting` ->`Quality`中`Pixel Light Count` 属性的默认值是3，（之前一直错认为4，所以实验中出现了若设置4个点光源则总有一个的效果是看不到的，因为按重要度排序后只有3个可以被逐像素处理）如图：

![Edit the settings for a specific Quality level]({{site.url}}/assets/image/posts_images/QualitySettings2019-1.png)

可以看到实验结果如下：

![]({{site.url}}/assets/image/illustrations/9_2.png) ![]({{site.url}}/assets/image/illustrations/9_2_1.png)

#### 分析

当我们创建一个光源时，默认情况下其`Render Mode`设置为`Auto`，这意味着Unity会自动的判断光源的处理类型。由于没有更改上述`Pixel Light Count`的值（默认是3），因此一个物体可以接受除最亮平行光之外的3个逐像素光源。在上面的例子中，场景中共包含了4个光源，其中一个是平行光，它会在**Base Pass**中按照逐像素方式被处理；其余4个点光源由于`Render Mode`设置为`Auto`且数目正好等于3，因此会在**Additional Pass**中以逐像素的方式进行处理。 每个光源会调用一次Additional Pass。

#### 关于帧调试器

在Unity 5中，可以使用**帧调试器**去查看场景的绘制过程。使用方法是`Window`->`Analysis`->`Frame ebugger`打开调试器，然后Enable分析就可以看到啦。

![]({{site.url}}/assets/image/posts_images/FrameDebugger.png)

我们截取渲染光照的部分分析如下：

| ![]({{site.url}}/assets/image/posts_images/FD_clear.png) | ![]({{site.url}}/assets/image/posts_images/FD_draw_1.png) | ![]({{site.url}}/assets/image/posts_images/FD_draw_2.png) | ![]({{site.url}}/assets/image/posts_images/FD_draw_3.png) | ![]({{site.url}}/assets/image/posts_images/FD_draw_4.png) |
| -------------------------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------- |
| clear                                                    | draw                                                      | draw                                                      | draw                                                      | draw                                                      |

调节点光源的强度和范围可以发现渲染的事件和视觉效果会有所变化，大家可以自行探索一下~

设置5个光源的`Render Mode`为`Not Important`也可以得到不同的结果~

本节主要介绍了光源在前向渲染中的不同处理方式和两个Pass的具体内容和不同，下节将重点关注光照衰减~

### 参考

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)


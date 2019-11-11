



上一部分中提到了由于关闭深度写入而产生的遮挡问题，本小节将进行一种解决上述问题的实践——开启深度写入的两个Pass的透明效果实现。



**基本思想**

使用两个Pass进行渲染：

- 第一个Pass开启深度写入，但不输出颜色，其目的仅仅是为了写入模型的深度值；
- 第二个Pass进行正常的透明度混合，因为上一个Pass已经得到了逐像素的正确深度信息，所以在这个Pass里可以根据逐像素深度值排序的结果进行透明渲染。

利用这种方法就可以得到遮挡结构的正确透明效果啦，但是，相应的会增加一些在所难免的性能开支~



**实践**



- 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)



**准备工作**



1. 在Unity中新建一个场景，命名为Scene_8_5。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->任一个Shader`)并命名为AlphaBlendZWrite；新建材质(右键`Create->Material`)并命名为AlphaBlendZWriteMat，将新建的Shader拖拽赋给新建材质。
3. 向场景中拖拽一个knot模型（在项目资源的model文件夹中），并拖拽到合适位置，将其材质修改为新建材质。
4. 保存场景。



**Shader实现**:

打开新建的AlphaBlendZWrite，删除所有已有代码并把AlphaBlend代码粘贴进来，并作修改得到：

```glsl
Shader "Custom/AlphaBlendZWrite"
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
        //新加一个Pass,实现写入深度信息，从而剔除模型中被自身遮挡的片元的作用
        Pass {
            ZWrite On
            ColorMask 0//在Shaderlab中,ColorMask用于设置颜色通道的写掩码(write mask)，值为0时，表示该Pass不写入任何颜色通道
        }
        //与AlphaBlend相同
        Pass {
            Tags { "LightMode"="ForwardBase" }
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
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir)) ; // 颜色融合用乘法
        
                // 最终颜色 = 漫反射 + 环境光 + 高光反射
                return fixed4(diffuse + ambient, texColor.a * _AlphaScale); 
            }       
            ENDCG
        }
        
    }
    FallBack "Diffuse"
}
```



> 多说一句ColorMask,其语义可表示如下：
>
> ​	ColorMask RGB | A | 0 | 其他任何R, G, B, A的组合



保存返回后，可以得到鲜明对比如下：

| ![]({{site.url}}/assets/image/illustrations/8_1.png) | ![]({{site.url}}/assets/image/illustrations/8_2.png) |
| :--------------------------------------------------: | :--------------------------------------------------: |
|                      AlphaBlend                      |                   AlphaBlendZWrite                   |



至此，透明效果的实践接近尾声了，但在之前的实践中`clip`,  `Blend` 等Unity内置的工具函数功不可没，还有许多没有提到的混合类型合效果，大家可以参考文末的链接到官网搜索以解疑惑~



### 参考

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)
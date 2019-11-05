---
layout: post
title: "Unity纹理-引入和单张纹理"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

纹理映射脑洞一下，可以理解为，把一张图贴在模型表面，去控制模型的颜色和外观。本主题将记录在unity中利用纹理采样实现更加丰富的视觉效果。

通常美术人员会在建模软件中利用纹理展开技术把**纹理映射坐标（texture-mapping coordinates）**存储在每个顶点上。而纹理映射坐标定义了该顶点在纹理中对应的2D坐标，通常用一个二维变量(u,v)来表示，其中u是横向坐标，v是纵向坐标，故纹理映射坐标又称UV坐标。

纹理的大小可变(如256x256, 1028x1028),但顶点UV坐标通常归一化到[0,1]范围。特殊的情况是，纹理采样时，坐标范围不在[0,1]之内，会有特殊处理。

Unity使用的坐标系是符合OpenG传统的，坐标原点位于纹理左下角。

### 单张纹理

- 运行平台：

    **Unity 2018.4.2f1 (64-bit)**

- 项目地址：

    [Unity_Shader_GetIn](https://github.com/wonderly321/Unity_Shader_GetIn)

#### 目标

使用单张纹理作为物体的漫反射颜色

#### 实践

在本例中仍使用Blinn-Phong光照模型。

准备工作：

1. 在Unity中新建一个场景，命名为Scene_7_1。默认场景中将包含一个摄像机和一个平行光，并使用内置的天空盒子。为便于查看效果，在`Window->Rendering->Lighting Seting->Skybox`中去掉场景中的天空盒子。
2. 新建Shader(右键`Create->Shader->SingleTextureShader`)并命名为SingleTexture；新建材质(右键`Create->Material`)并命名为SingleTextureMat，将新建的Shader拖拽赋给新建材质。
3. 在场景中新建一个胶囊体(菜单栏`GameObject->3D Object->Capsule`)，将其材质修改为新建材质。
4. 保存场景。

Shader实现:

打开新建的SingleTexture，删除所有已有代码并写入如下代码：

```glsl
Shader "Custom/SingleTexture" {
    Properties{
        _Specular("Specular Color", Color) = (1, 1, 1, 1)
        _Gloss("Gloss", Range(8.0, 256)) = 20
        //除了已有属性之外，应添加一个名为_MainTex的纹理属性，2D是纹理属性的声明方式，“white”是内置纹理的名字，意为全白
        _MainTex("Main Tex", 2D) = "white"{}
        //wei控制物体的整体色调，还可声明_color属性
        _Color("Color Tint", Color) = (1, 1, 1, 1)
    }
    SubShader{
        Pass {
            //LightMode标签是Pass标签的一种，它用于定义该Pass在Unity的光照流水线的颜色
            Tags{"LightMode" = "ForwardBase"}

            //CGPROGRAM和ENDCG共同包裹最重要的顶点着色器和片元着色器的代码
            CGPROGRAM

            #include "Lighting.cginc"

            #pragma vertex vert
            #pragma fragment frag
 
            fixed4 _Color;
            sampler2D _MainTex;
            fixed4 _Specular;
            float _Gloss;
            //需要为纹理类型的属性声明一个float4类型的变量_MainTex_ST
            //p.s.:Unity中需要使用纹理名_ST的方式来声明某个纹理的属性。ST为缩放(scale)和平移(translation)的缩写
            float4 _MainTex_ST;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                //用TEXCORD0将模型的第一组纹理坐标存储到该变量中
                float4 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
                //用uv存储纹理坐标,以便在片元着色器中使用该纹理坐标进行纹理采样
                float2 uv : TEXCOORD2;
            };

            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_WorldToObject, v.vertex).xyz;
                //_MainTex_ST.xy存储的是缩放值，_MainTex_ST.zw存储的是偏移值，这些值可以在材质面板的纹理属性中调节，分别作用于顶点纹理坐标上
                o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;

                return o;
            }

            //通过 scale/bias 属性转换2D UV，其中第一个参数是顶点纹理坐标，第二个是纹理名
            #define TRANSFORM_TEX(tex, name)(tex.xy * name##_ST.xy + name##_ST.zw)


            // 计算每个像素点的颜色值
            fixed4 frag(v2f i) : SV_Target 
            {
                // 计算世界空间下的法线方向和光照方向
                fixed3 worldNormal = normalize(i.worldNormal); 
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));

                //使用CG的tex2D函数对纹理进行采样，第一个参数是需要被采样的纹理，第二个参数是一个float2类型的纹理坐标，用于返回计算得到的纹素值
                //采样结果* 颜色属性_Color作为材质反射率albedo
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;

                // 环境光
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;

                //漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir)) ;

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

在SingleTextureMat面板上可以使用资源Brick_Diffuse.jpg纹理(或其他图片)对Main Tex属性进行赋值

效果图如下：

![]({{site.url}}/assets/image/illustrations/3_2.png)

#### 纹理的属性

![]({{site.url}}/assets/image/illustrations/3_4.png)

如图所示为材质面板，接下来就其中几个属性进行声明：

- 纹理类型（Texture Type）:使用此选项可以从源图像文件中选择要创建的纹理的类型
    - Default
    - Normal Map
    - Sprite (2D and UI)

    ...

本节使用`Default`类型，它是用于所有纹理的最常见设置。它提供对大多数属性的访问以进行纹理导入。另外，法线纹理通常使用`Normal map`类型将颜色通道转换为适合实时法线贴图的格式；在2D游戏中使用`Sprite(2D和UI)`。

- 纹理形状（Texture Shape）:使用它来选择和定义纹理的形状和结构。
    - 2D
    - Cube

    ...
    
本节使用`2D`类型，这是所有纹理的最常见设置。它将图像文件定义为2D纹理。它提供对大多数属性的访问以进行纹理导入。另外，`Cube`将纹理定义为立方体(cubemap)。用户可以将其用于“天空盒子(skybox)”或“ 反射探测器(Reflection Probes)”， 例如。选择cube去显示不同的映射选项。

- Alpha is Transparency:

如果指定的Alpha通道为透明度，则启用Alpha为透明度可放大颜色并避免过滤边缘上的瑕疵。

另提几个重要的高级属性：

- Wrap Mode: 决定了纹理坐标超过[0,1]时如何平铺。
Repeat模式下纹理坐标超过了1，则其整数部分会被舍弃，而直接使用小数部分进行采样。这样使得纹理结果不断重复；Clamp模式下，若纹理坐标大于1，则直接截取到1，若小于0，则截取到0。其他还有Mirror，Mirror Once等模式,不作详述。

p.s.: 说起平铺模式，则可以在材质面板中设置`Tilling`和`Offset`两种属性来调整纹理的重复次数和偏移位置

- Filter Mode:决定了纹理由于变换而产生拉伸时将采用哪种滤波模式

它支持3种：`Point`， `BIlinear`,`Trilinear`。得到图片的滤波效果依次提升，需要耗费的性能也依次增大。纹理滤波会影响放大或缩小纹理时得到的图片质量

p.s. Point滤波内部实现使用了**最近邻**滤波，在放缩时采样像素数目仅有一个，所以看上去有像素风效果（23333，好复古！）。Bilinear使用线性滤波，图像看起来有模糊的效果，Trilinear与之大致相同，只不过混合时采用了多级渐远纹理。

- 最大尺寸(Max Size) :导入的最大纹理尺寸(以像素为单位)。美工人员通常喜欢使用巨大的尺寸大小的纹理；使用最大尺寸将纹理缩小到合适的尺寸大小。值得一提的是，若导入纹理的长宽大小不是2的幂次则会占用更多内存，降低读取速度。

- 纹理模式(Format) : 这将绕过自动系统来指定用于纹理的内部表示.使用的纹理格式精度越高(如，RGBA 32 bit)，占用内存空间越大，但得到效果越好。因此需要具体需求具体分析。

### 参考

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)
---
layout: post
title: "Unity渲染路径——光源种类"
categories: shader
author: "Wonder"
meta: "Unity Shader"
---

在之前的实践场景中，一般只有一个光源且光源类型是平行光。但实际光源类型不止一种，且需要被区别对待。

#### 光源种类

在之前的文章中多多少少地提到了Unity中支持几种不同的光源，其中包括点光源（point lights），聚光灯光源（direct lights），平行光光源（directional lights）， 面光源（Area lights）以及发光物质（Emissive materials） 和 环境光（Ambient light）。

由于每种光的几何定义不同，因此它们对应的光源属性也就各不相同。因此需要区别对待。

**点光源**

![]({{site.url}}/assets/image/posts_images/PointLightDiagram.svg)



点光源位于空间中的某个点，并且均匀地向所有方向发出光。入射到表面的光的方向是从接触点回到光源对象中心的线。强度随着与光的距离而减小，在指定范围内达到零。光强度与到光源的距离的平方成反比。这就是所谓的“平方反比定律”，类似于光在现实世界中的行为。

其在场景中效果如下：



![]({{site.url}}/assets/image/posts_images/Light-Point.jpg)



其属性面板示例如下：



![]({{site.url}}/assets/image/posts_images/pointlight_setting.png)



点光源的球体半径可以由属性面板上的`Range`来调整，也可以直接拖拽球体的线框。其位置属性由面板中`Transform`组件中的`Position`属性来定义，其他属性如光源的颜色，强度都是可以在面板中进行设置的。

**聚光灯光源**

与点光源相同，聚光灯光源具有特定的位置和范围，但是被限制在一个角度上，所以形成锥形照明区域。圆锥体的中心指向灯光对象的向前（Z）方向。聚光灯圆锥体的边缘处的光线也会减少。加宽角度会增加圆锥的宽度，并随之增加这种渐隐的大小，称为“半影”（penumbra）。

![]({{site.url}}/assets/image/posts_images/SpotLightDiagram.svg)

聚光灯通常用于人造光源，例如手电筒，汽车前灯和探照灯。通过脚本或动画控制方向，移动的聚光灯将仅照亮场景的一小部分并产生戏剧性的照明效果。

![]({{site.url}}/assets/image/posts_images/Light-Spot.jpg)

锥形照明区域的半径由`Range`决定，张开角度由`SpotAngle`决定，其属性面板示例如下：

![]({{site.url}}/assets/image/posts_images/spotlight_setting.png)

**平行光**

平行光对于在场景中创建诸如日光之类的效果非常有用。像太阳一样，平行光在许多方面都可以看作是无限远地存在的遥远光源。平行光没有一个唯一的光源位置，因此可以将光源对象放置在场景中的任何位置。场景中的所有对象都被照亮。光线总是来自同一方向。光线到目标物体的距离没有定义，因此光线不会衰减，即，光照强度不会随着距离的改变而发生改变。

![]({{site.url}}/assets/image/posts_images/DirectionalLightDiagram.svg)

平行光表示来自游戏世界范围以外位置的大型的远距离光源。在逼真的场景中，它们可以用于模拟太阳或月亮。在抽象的游戏世界中，平行光是在不精确指定光源方向的情况下向物体对象添加令人信服的阴影的有用方法。

其在真实场景中的效果如下：

![]({{site.url}}/assets/image/posts_images/Light-Direct.jpg)

默认情况下，每个新的Unity场景都包含一个平行光。可以通过删除默认的“平行光”并创建新的光源。调解面板中的`Rotation`属性时，将灯光指向上方会导致天空变黑，就像在夜间一样。当光线从上方向下倾斜时，类似于日光。

属性面板如下：

![]({{site.url}}/assets/image/posts_images/DirectionalLight_setting.png)

**面光源**

一个面光源由空间中的矩形限定。在所有方向上均匀地在其表面区域上发出光，但仅从矩形的一侧发出。对于面光源的范围没有手动控制，但是强度会随着距离光源的距离的平方成反比而减小。由于照明计算需要大量处理器，因此面光源在运行时不可用，只能烘焙为光照贴图。

![]({{site.url}}/assets/image/posts_images/AreaLightDiagram.svg)

由于面光源会同时从几个不同的方向照亮对象，因此阴影比其他类型的光更柔和细微，可用于创建逼真的路灯或靠近播放器的一排灯。小面积的光源可以模拟较小的光源（例如室内照明），比点光源具有更逼真的效果。

其效果如下：

![]({{site.url}}/assets/image/posts_images/AreaLights.png)

**发光材料**

![]({{site.url}}/assets/image/posts_images/EmissiveMaterial.png)

与面光源相同，发光材料会在其表面积上发出光。它们有助于场景中的反射光，并且在游戏过程中可以更改相关的属性，例如颜色和强度。尽管预计算的实时GI不支持区域照明，但是使用发光材料仍然可以实时获得类似的柔和照明效果。

自发光是**标准着色器（Standard Shader）**的属性。它允许场景中的静态对象发光。默认情况下，自发光的值设置为零。这意味着使用“ 标准着色器”分配了材质的对象将不会发光。

发光材料没有范围值，但是发出的光将再次以二次速率衰减。同样，应用于非静态或动态几何图形（例如角色）的发光材料也不会有助于场景照明。但是，自发光值大于零的材质即使对场景照明没有帮助，仍会在屏幕上明亮地发光。像这样的自发光材料是创建效果（例如氖气或其他可见光源）的有用方法。

发光材质仅直接影响场景中的静态几何体。如果您需要动态或非静态的几何图形（例如角色）来从发光材料中拾取光，则必须使用“ **光探测器”（Light Probes）“**。

**环境光**

环境光是存在于整个场景中的光，而不是来自任何特定的源对象。它可能是场景整体外观和亮度的重要因素。

根据您选择的艺术风格，环境光在许多情况下都可能有用。如果需要在不调整单个灯光的情况下增加场景的整体亮度，则环境光也很有用。

在菜单栏中的`Window`->`Rendering`->`Lighting setting`中可以找到环境光设置。



以上就是Unity支持的各种光源类型了，之后将会介绍在前向渲染中对不同光源的不同处理。

### 参考

[Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://docs.unity3d.com/Manual/TextureTypes.html)

[Unity Scripting Reference : https://docs.unity3d.com/ScriptReference/index.html](https://docs.unity3d.com/ScriptReference/index.html)

[Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)


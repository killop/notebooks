# Unity技术美术日记——多种方式实现布料飘动 - 知乎
0\. 前言
------

在游戏制作中经常会遇到一个问题：在茫茫多的技术方案中如何选择最适合项目的方案？

我总结了以下几种我自己做过的技术方案（本文将不考虑常规的绑定蒙皮K动画的布料方案）

1.[顶点动画](https://zhida.zhihu.com/search?content_id=239920798&content_type=Article&match_order=1&q=%E9%A1%B6%E7%82%B9%E5%8A%A8%E7%94%BB&zhida_source=entity)

2.Magica Cloth

3.VAT （Vertex Animation Texture）

**1\. 顶点动画**
------------

顶点动画的方案在网上有很多，一般指通过顶点着色器对模型每个顶点进行位移操作从而模拟不同的运动效果。在游戏中遇到大量顶点需要进行规则或有规律的运动时，顶点动画就是一种非常有效率的技术方案

首先我们给旗面绘制顶点色，控制旗子飘动的区域

![](https://pic1.zhimg.com/v2-55000b66718f22c5ba6d0734ec61f6c2_1440w.jpg)

首先我们先对旗子本身添加一个简单的自飘动效果

```glsl
                //自身的顶点位移
                half sinOff = positionOS.x + positionOS.y + positionOS.z; 
                half t = -_Time.x * 50; 

                #if defined(_Swing_ON)//自摆动keyword
                positionOS.x += sin(t * 1.45 + sinOff) * vertexColor.r * (_ClothAmplitudeX * 0.5); 
                positionOS.z += sin(t * 2 + sinOff) * vertexColor.r * (_ClothAmplitudeY * 0.5 )  * 0.3; 
                #endif

                //给一个固定位移值，防止旗子与旗杆穿插穿帮
                positionOS.x -= vertexColor.r * _xOffset;   
                positionOS.z -= vertexColor.r * _yOffset;
```

让我们来看看效果：

![](https://pica.zhimg.com/v2-b25ce5a5479f3de31d1fb55fe5918f2e_b.gif)

基础的效果有了，但当存在多个搭配同材质球的旗子一起出现时会出现飘动轨迹完全一样的效果，会显得重复感太强：

![](https://picx.zhimg.com/v2-cd22b554380b0d213e95742784a325bb_b.gif)

因此我们来给旗子再加一层额外的基于世界空间的顶点位移：

```glsl
        //基于世界空间的顶点位移
        float3 worldPos = TransformObjectToWorld(positionOS.xyz);

        float2 sampleUV = float2(worldPos.x *rcp(_WaveSize) , worldPos.z *rcp(_WaveSize));    
        sampleUV.x+= _Time.x ;
    
        //用一张noise贴图扰动世界空间的顶点位移
        float4 windTex = tex2Dlod(_NoiseRipplingMap, float4(sampleUV,0,0));  
        half summedBlend = 0.33 * windTex.r + 0.33 * windTex.g + 0.33 * windTex.b ;
        worldPos += sin(0.1 * summedBlend) * _WindOffset.rgb * vertexColor.r;

        positionOS.xyz = TransformWorldToObject(worldPos.xyz);
```

在Substance Design里找了一张Noise贴图用来扰动世界空间的顶点位移效果

![](https://pic2.zhimg.com/v2-552df4a4cf21421a1d96aff47d8455e3_1440w.jpg)

最终效果如图：

![](https://picx.zhimg.com/v2-06e6369d941070c60dd2adf80bc94631_b.gif)

完整HLSL代码如下：

```csharp
void VertexMove_Cloth(half4 vertexColor,half2 uv,inout half4 positionOS, half3 normalOS)
{
    #if defined (Cloth_Render)

        //自身的顶点位移
        half sinOff = positionOS.x + positionOS.y + positionOS.z; 
        half t = -_Time.x * 50; 

        #if defined(_Swing_ON)
        positionOS.x += sin(t * 1.45 + sinOff) * vertexColor.r * (_ClothAmplitudeX * 0.5); 
        positionOS.z += sin(t * 2 + sinOff) * vertexColor.r * (_ClothAmplitudeY * 0.5 )  * 0.3; 
        #endif

        positionOS.x -= vertexColor.r * _xoffset;   
        positionOS.z -= vertexColor.r * _yoffset;
    
        //基于世界空间的风力效果
        float3 worldPos = TransformObjectToWorld(positionOS.xyz);

        float2 sampleUV = float2(worldPos.x *rcp(_WaveSize) , worldPos.z *rcp(_WaveSize));    
        sampleUV.x+= _Time.x ;
    
        //用一张noise贴图扰动全局风力
        float4 windTex = tex2Dlod(_NoiseRipplingMap, float4(sampleUV,0,0));  
        half summedBlend = 0.33 * windTex.r + 0.33 * windTex.g + 0.33 * windTex.b ;
        worldPos += sin(0.1 * summedBlend) * _WindOffset.rgb * vertexColor.r;

        positionOS.xyz = TransformWorldToObject(worldPos.xyz);

    #endif
}

```

P.S. 如果觉得旗子的飘动比较规律，可以更改用到Sin函数的地方，使其飘动距离变得不规律

![](https://picx.zhimg.com/v2-a4d8fe15494398b2df0ed99a233cf5bd_1440w.jpg)

y = Sin(a \* x) + Sin (x)

2.Magica Cloth
--------------

Magica Cloth 是一种由 Unity DOTS（面向数据的技术堆栈）提供支持的快速布料模拟。 它可用于变换和网格，BoneCloth 控制变换，MeshCloth 控制网格顶点。Unity内置Cloth组件和MagicaCloth这两种方法的在该案例下的美术表现基本一样，区别是Magica Cloth参数配置更简单，更适合新手上手学习，开发效率更高。因此本次我们就简单介绍MagicaCloth的制作方式和效果。

事实上MagicaCloth已经有很完善的官方技术文档了，可以很轻松的学习如何开始制作动态布料和每一个参数的具体含义。

首先先把这五个组件加上，从上到下分别是：Magica Render Deformer，Magica Virtual Deformer，Magica Mesh Cloth，Magica Capsule Collider和Magica Directional Wind。组件的具体功能在此[Component - Magica Soft](https://link.zhihu.com/?target=https%3A//magicasoft.jp/en/magicaclothcomponent-2/)

![](https://pic3.zhimg.com/v2-45f7d954d218c735c575d726ee6b79c2_1440w.jpg)

分别点击Magica Render Deformer和Magica Virtual Deformer内的Create来获取模型顶点数据，注意模型需要打开读写才能被Magica组件获取到数据。

接下来点击Magica Mesh Cloth组件里的Start Point Selection开始为模型设置固定点，并为模型调整它的物理属性参数。

![](https://pic1.zhimg.com/v2-695955669a27ece60edf12bc0cb4aaa8_1440w.jpg)

![](https://pic2.zhimg.com/v2-7c9edc3188644919590455755b433ffd_1440w.jpg)

为了防止旗面穿过旗杆，我们用Magica Capsule Collider来覆盖旗杆的模型，并在最后调整Magica Directional Wind组件中的风力方向和强度，最终效果如下

![](https://pic3.zhimg.com/v2-66024f6ebec9919c8d398a5b484dace4_b.gif)

其实能很明显的看到使用Magica Cloth插件模拟的布料效果相比第一种顶点动画要更真实，更像真正的布料，Magica Cloth提供了大量的参数，满足了不同对物理属性物体的模拟需求。理论上DynamicBone也可以做出类似的效果，但其技术原理更接近于常规的骨骼绑定蒙皮，因此不做过多介绍。

3.VAT （Vertex Animation Texture）
--------------------------------

Magica虽然更真实效果更好，但相比于顶点动画来说性能消耗也是大大提高的，哪有没有一种办法既可以满足高品质的布料飘动效果并且在性能上也有所优化呢？

虽然游戏是实时渲染的艺术，但并非所有的物体都需要实时渲染，VAT就是一种把离线渲染融合进游戏的渲染技术，这是一种用空间换时间的做法，其运动的技术原理和第一节提到的顶点动画基本相同，只不过他将实时计算的顶点位置存到一张图里，相当于预烘焙了顶点动画。

本节我将使用[Houdini](https://zhida.zhihu.com/search?content_id=239920798&content_type=Article&match_order=1&q=Houdini&zhida_source=entity)+Unity+VAT的流程来制作旗帜飘动的效果，本文中使用的houdini版本是20.0.547。

首先我们先在houdini里制作旗帜飘动的效果：

**1.布料模拟**

将模型导入后，先给旗子添加一个group，方便后续制作风力动画时固定这部分顶点

![](https://pic4.zhimg.com/v2-bec49fdc65bfdb386a0192e47f871d51_1440w.jpg)

为了表现布料细微的褶皱表现，需要对原有模型进行加面，这里remesh一下模型。

![](https://pic4.zhimg.com/v2-0f31af91f6e6152e8587823b3b679e51_1440w.jpg)

接下来开始模拟布料的运动轨迹，首先先把vellum相关节点加上，在vellum Constraints里来调整一下mesh的相关属性，并在Pin to Animation里把我们之前准备好的Group加上，用于固定顶点。

一些布料参数可以参考：

![](https://pic1.zhimg.com/v2-f8683dfb33345c75fda530d15b50cc1e_1440w.jpg)

然后在solver里调整一下风力的相关参数，不断调试到我们想要的样子

![](https://pic1.zhimg.com/v2-3a4a1b54f539b0537566418b2262d51c_1440w.jpg)

做到这步时发现了一个问题，当我们循环播放这段动画时，动画的首帧和尾帧无法很好的过渡，也就是无法做成循环动画

![](https://pica.zhimg.com/v2-98b64803575b5ac11d1c335c6e63befc_b.gif)

因此我们需要为其加个过度节点，这里我们用一个自定义节点来为首尾帧做插值[\[1\]](#ref_1)

![](https://picx.zhimg.com/v2-f656112a38840bfa05c695513e688b7d_1440w.jpg)

输入端我们来定义选用哪帧为首尾帧，和插值迭代次数

![](https://pic3.zhimg.com/v2-2deb5c77b21fff26a5bc69a9cdb08258_1440w.jpg)

看一下结果

![](https://pic4.zhimg.com/v2-98b645a003810cc3b57a92eeb8a06b3d_b.gif)

**2.Unity导入**

接下来我们来导入到unity，首先在后面添加一个out节点，作为整段模拟的输出节点

![](https://pic2.zhimg.com/v2-90da398d33d41585e429a6314d38be19_1440w.jpg)

houdini已经为VAT提供了简化的导入引擎流程，首先要将Houdini的[VertexAnimationTexture](https://zhida.zhihu.com/search?content_id=239920798&content_type=Article&match_order=1&q=VertexAnimationTexture&zhida_source=entity)的Package导入到Unity引擎中，具体方式可以看节点里的guide

![](https://pic1.zhimg.com/v2-111ecffc7b5055a9eab7f2b6f7bea51c_1440w.jpg)

Unity导入正常后，需要在VertexAnimationTexture的节点里设置我们导入的节点，和导出的资源位置

![](https://pic1.zhimg.com/v2-988d0c09fd12bbc2cf0e0901c23d7226_1440w.jpg)

导出后我们会得到三个文件夹，分别是模型，贴图和材质球，把它们导入到unity内。

![](https://pic3.zhimg.com/v2-39da1c23e6e3887d80ea1382f5575854_1440w.jpg)

根据我们导出的图，可以看到图的高度像素数正好等于我们的帧数（1680-60），也就是说高度代表动画总帧数，宽度代表我们旗帜的顶点数，像素内的RGB值则代表了坐标轴的XYZ偏移值，这样通过一张图我们就得到了每一个顶点在每一帧的偏移值。

![](https://picx.zhimg.com/v2-09bb3a1e8ffbc0d55e986f67357c6887_1440w.jpg)

**3.在Unity中实现**

根据houdini提供的说明文档，先将刚导入的贴图和模型的import setting按照下图设置好

![](https://pic4.zhimg.com/v2-987d43c137a250fa3285c65b75f5afdb_1440w.jpg)

接下来就简单了，把houdini提供给我们的贴图和材质球结合起来，可以看到在导出的材质球里，顶点动画的最大最小值已经设置好了

![](https://pica.zhimg.com/v2-a615ab1bd7970e0233d91fa4a974a12e_1440w.jpg)

在材质面板上并未发现basemap和法线贴图的位置，根据houdini提供的shader我们稍作修改，最终结果如下

![](https://pic1.zhimg.com/v2-394e5fc543a94e1a94c00af1e2aa2f30_b.gif)

P.S. 搞个碰撞玩一下:)

![](https://pic3.zhimg.com/v2-c909e70761505f1d634f48aba812488a_b.gif)

**结语**
------

游戏中的技术在实现的过程中都有很多种备选方案，在本文中我尝试了三种方式实现布料模拟，不同的实现方式有不同的优点：

画面表现（从好到坏）：VAT ≥ Magica Cloth > 顶点动画

性能表现（以帧耗时为参考依据，从低到高）：顶点动画 ＞ VAT ≥ Magica Cloth

效果调整难度（从易到难）：顶点动画 ＞ Magica Cloth ＞＞＞ VAT

对模型面数的需求（从低到高）：顶点动画 ＞ Magica Cloth ＞＞＞ VAT

另外不同技术还有许多可以扩展的方向，比如Magica Cloth对于游戏过程中进行实时布料模拟会有更高的适配性[\[2\]](#ref_2)

![](https://picx.zhimg.com/v2-cabeab474f1218996512748c2245dbed_b.gif)

VAT也可以做除了布料模拟以外的顶点动画[\[3\]](#ref_3)

![](https://picx.zhimg.com/v2-43fd80a181548e699a40cf0a1ba7a453_b.gif)

![](https://pic1.zhimg.com/v2-2956adc805f29382c0bbabf30412d5a0_b.gif)

本文仅实现了基础的工作流，在美术效果上还有很大的优化空间。总之，根据项目需求选择最合适的技术方案才是最重要的。

参考
--

1.  [^](#ref_1_0)过度节点参考 [https://blog.csdn.net/u013412391/article/details/120439373](https://blog.csdn.net/u013412391/article/details/120439373)
2.  [^](#ref_2_0)MagicaCloth案例 [https://www.youtube.com/watch?time\_continue=83&v=dPJYg8kTZjM&embeds\_referring\_euri=https%3A%2F%2Fwww.gameassetdeals.com%2F&source\_ve\_path=MTM5MTE3LDEzOTExNywyODY2MywxMzc3MjEsMzY4NDIsMzY4NDIsMzY4NDIsMjg2NjY&feature=emb\_logo](https://www.youtube.com/watch?time_continue=83&v=dPJYg8kTZjM&embeds_referring_euri=https%3A%2F%2Fwww.gameassetdeals.com%2F&source_ve_path=MTM5MTE3LDEzOTExNywyODY2MywxMzc3MjEsMzY4NDIsMzY4NDIsMzY4NDIsMjg2NjY&feature=emb_logo)
3.  [^](#ref_3_0)VAT案例 [https://github.com/Bonjour-Interactive-Lab/Unity3D-VATUtils?tab=readme-ov-file](https://github.com/Bonjour-Interactive-Lab/Unity3D-VATUtils?tab=readme-ov-file)
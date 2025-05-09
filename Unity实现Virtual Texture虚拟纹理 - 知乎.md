# Unity实现Virtual Texture虚拟纹理 - 知乎
技术背景
----

Virtual Texture(虚拟纹理)主要用于解决实时渲染中由于各种原因限制导致的无法直接显示超大分辨率纹理的问题。有可能是受限于内存，有可能是受限平台或图形API等。一般超大纹理不会完整被渲染到屏幕中，远景更多的情况是渲染低分辨率的Mipmap。

VT的思路就是把这张超大纹理切分成多个块(Page)。渲染时，根据屏幕绘制的需要，去加载或绘制对应的Page到物理缓存中。使用的时候通过间接列表(PageTable)的方式去找到这个Page。

VT有多种技术拓展：

### **Streaming Virtual Texture(SVT)  
流式虚拟纹理(SVT)**

纹理是已有资产，如模型贴图等，需要离线切割Page。运行时按需要加载Page而不需要整个大纹理加载，节省内存。

![](https://pic1.zhimg.com/v2-d6cfd4ce62fa0d3a5489bceee8f89232_1440w.jpg)

### **[Runtime Virtual Texture](https://zhida.zhihu.com/search?content_id=251345179&content_type=Article&match_order=1&q=Runtime+Virtual+Texture&zhida_source=entity)(RVT):  
运行时虚拟纹理(RVT):**

纹理是实时生成的，运行时按需要渲染Page，典型的案例是地型渲染，地型渲染需要混合多层纹理贴图开销较大，RVT可以把混合结果缓存起来，节省性能。

![](https://pica.zhimg.com/v2-7557b837ab672581666467f229bc00a0_1440w.jpg)

### **[Adaptive Virtual Texture](https://zhida.zhihu.com/search?content_id=251345179&content_type=Article&match_order=1&q=Adaptive+Virtual+Texture&zhida_source=entity)(AVT):  
自适应虚拟纹理(AVT):**

解决追求高精度的VT时 Indirection Texture过大的问题。 在标准RVT上引入了Sectors概念，VirtureTexture被划分到多个Sectors，根据相机距离动态调整Sectors大小，每个Sectors对应一份固定大小的Indirection Texture 去索引物理页。相当于多套了一层转换。

![](https://pica.zhimg.com/v2-ac497681223fafe542f0ef2ba05375ac_1440w.jpg)

### **Clipmap Texture:剪影纹理:**

使用相机信息来收集需要渲染的Page，而不是屏幕回读方式。避免了的GPU回读慢的问题和异步回读带来的延迟性。指定了特定区域的精度，解决了IndirectionTexture过大的问题。 更适合低端设备使用。其实和Cascade Shadowmap类似。

![](https://pic1.zhimg.com/v2-32f19449e6dad0e6792e3c607693501c_1440w.jpg)

多种VT部分流程是大致相同的。

篇幅有限本文主要介绍和实现标准的RVT和Clipmap Texture来渲染地型。

![](https://pic1.zhimg.com/v2-570884b74c2c79c98c7ee4cad71374d0_1440w.jpg)

Virtual Texture Terrain虚拟纹理地形

![](https://pic3.zhimg.com/v2-c6db6683485301ba16d3a06d02456534_1440w.jpg)

Virtual Texture Terrain虚拟纹理地形

VT流程
----

VT主要流程其实大同小异:

### 1.划分Virtual Texture Page1.划分虚拟纹理页

把整个巨大的虚拟纹理，每级mipmap都均匀划分成多个Virtual Texture Page。

越高级的Mip，Page数量越少，对应Virtual texture的像素区域越大。

如单个Pages Mip0:128 \* 128 , Mip1: 256 \* 256。

![](https://picx.zhimg.com/v2-e7fb8abd867b884f4a0d92cc2128e469_1440w.jpg)

VT Mipmap PagesVT Mipmap 页

### 2.收集可视Virtual Texture Page  
2.收集可视虚拟纹理页

目的是收集当前需要的VTPage，主要分CPU和GPU两种流派:

**CPU流派：** 根据相机位置估算所需VTPage的Mipmap，ClipmapTexture用的方式。

优点：快速，没有延迟性。

缺点：VTPage没有遮挡剔除，没有准确的mip信息。

![](https://picx.zhimg.com/v2-210ccaf70d04add9de2a624b7f5ac79f_1440w.jpg)

**GPU流派：** 在屏幕中渲染Feedback Pass（输出Page和mip信息）到指定Buffer，然后通过异步回读的方式把VTPage和mip传到CPU处理。通常为了加快回读速度，会额外对Buffer做降采样。

优点：可以比较精准获取可视的VTpage，享有遮挡剔除。

缺点：需要额外渲染Feedback Pass，异步回读比较慢，且有一定延迟性，需要解决降采样带来的信息丢失问题。

![](https://pic3.zhimg.com/v2-bc075830425dc5a12258ef072ae7cc52_1440w.jpg)

Feedback反馈

### 2.更新Physical Texture2.更新物理纹理

获取到需要的Virtual Texture Page信息后，就可以在Physical Texture申请一块区域(Physical Page)来渲染生成对应的Virtual Texture Page，通常使用LRU（Least Recently Used)双向循环链表来维护这些物理页，经常看到的Page在链尾，分配时候尽量取链头的Physical Page，这些Physical Page是循环复用的。

![](https://picx.zhimg.com/v2-b9798f43de0466018d1bdbbcb98d1465_1440w.jpg)

Physical Page（LRU）物理页面（LRU）

然后在Physical Texture上的Physical Page区域，通过混合各种地型Splat Map渲染对应Virtual Page的Virtual Texture区域。

![](https://picx.zhimg.com/v2-5e5bba17732e592811c0e85dc65aeda9_1440w.jpg)

### 3.更新IndirectTexture(PageTable)  
3.更新间接纹理（页表）

绘制了物理页后，需要把Physical Page在 Physical Texture的位置信息，写入到对应的Indirection Texture里。Indirection Texture的每个纹素代表一个Page的寻址转换。这样实际用的时候我们采样Indirect Texture就可以找到实际对应的Physical Page。

![](https://pic4.zhimg.com/v2-ba74f065b14b57be6763904bf137c6b3_1440w.jpg)

RVT实现
-----

### 1.初始化

**Indirection Texture:**

我们需要先把VT划分多个Virtual Page，每个Page对于Indirection Texture的一个纹素。我们可以拟定一个mip0的Page大小是128个像素，Indirection Texture的大小1024(即1024个Page)，那个VT的分辨率是1024 \*128。有了这些数据就可以初始化我们的Indirection Texture。

![](https://pic3.zhimg.com/v2-b6ddaf7a3d2336079734c904961b747c_1440w.jpg)

Indirection Texture是带mipmap的，我们需要把Mipmap中的Page数据也初始化。每级的Mipmap中有对应分辨率的Page，越高级的Mip对应Page的数量就越少，单个Page对应VT的区域越大。

![](https://pic3.zhimg.com/v2-b8963929081101ae94586b15f762f1e0_1440w.jpg)

![](https://picx.zhimg.com/v2-87bf395076ac0283d18a3f5fc8b64fa9_1440w.jpg)

VTPage的ID使用莫顿码来编码，可以让Page按照“Z”字排序的，用来后续快速查该页对应其他mip的对应位置。

![](https://pic2.zhimg.com/v2-8ce01d0d30fab4def07173e3b5cc90d9_1440w.jpg)

Z形编码

```csharp
public static class MortonCode
{
    public static int MortonCode2(int x)
    {
        x &= 0x0000ffff;
        x = (x ^ (x << 8)) & 0x00ff00ff;
        x = (x ^ (x << 4)) & 0x0f0f0f0f;
        x = (x ^ (x << 2)) & 0x33333333;
        x = (x ^ (x << 1)) & 0x55555555;
        return x;
    }

    // Encodes two 16-bit integers into one 32-bit morton code
    public static int MortonEncode(int x,int y)
    {
        int Morton = MortonCode2(x) | (MortonCode2(y) << 1);
        return Morton;
    }
    public static int ReverseMortonCode2(int x)
    {
        x &= 0x55555555;
        x = (x ^ (x >> 1)) & 0x33333333;
        x = (x ^ (x >> 2)) & 0x0f0f0f0f;
        x = (x ^ (x >> 4)) & 0x00ff00ff;
        x = (x ^ (x >> 8)) & 0x0000ffff;
        return x;
    }

    public static void MortonDecode(int Morton,out int x,out int y)
    {
        x = ReverseMortonCode2(Morton);
        y = ReverseMortonCode2(Morton >> 1);
    }

}

```

**Physical Texture:**

物理贴图是存放实际存放Page数据的地方，大小是我们指定的，一般地型渲染的话Physical Texture是4k - 8k左右的分辨率，有2张(基色图,法线图)，这里代码中物理页我用Tile来表示，Tile的数量是 : 物理页分辨率 / 单页的分辨率，Tile是使用了上面提到的LRU管理。

本文的Physical Texture是使用的[TextureArray](https://zhida.zhihu.com/search?content_id=251345179&content_type=Article&match_order=1&q=TextureArray&zhida_source=entity)纹理数组格式，主要是想着不希望更新一小块的Page导致整张RT被Load/Store，还有就是提高实时压缩的效率(后面会提到)。

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Experimental.Rendering;

public class VTPhysicalTable
{
    public class Tile
    {
        public Vector4 rect;
        public VTPage cachePage;
        public byte depth;
        public Tile prev;
        public Tile next;
        public bool SetCachePage(VTPage page, out VTPage oldPage)
        {
            if (cachePage != null && cachePage.alwayInCache)
            {
                oldPage = null;
                return false;
            }
            else
            {
                oldPage = ClearCache();
                cachePage = page;
                return true;
            }
        }
        public VTPage ClearCache()
        {
            if (cachePage != null && !cachePage.alwayInCache)
            {
                var oldPage = cachePage;
                cachePage.loadFlag = VTPage.FLAG_NONE;
                cachePage.physTile = null;
                cachePage = null;
                return oldPage;

            }
            else
            {
                return null;
            }
        }

    }

    public int realWidth;
    public int realHeight;
    int tileCountX;
    int tileCountY;
    byte depth;
    public RenderTexture[] rts;
    public int[] matPass;
    public Tile lruFirst;
    public Tile lruLast;

    void AddLRU(Tile tile)
    {
        if (lruFirst == null)
        {
            lruFirst = tile;
            lruLast = tile;
        }
        else
        {
            var last = lruLast;
            tile.prev = last;
            last.next = tile;
            lruLast = tile;
        }
    }

    void RemoveLRU(Tile tile)
    {
        if (lruFirst == tile && lruLast == tile)
        {
            lruFirst = null;
            lruLast = null;
        }
        else if (lruFirst == tile)
        {
            lruFirst = tile.next;
        }
        else if (lruLast == tile)
        {
            lruLast = lruLast.prev;
        }
        else
        {
            tile.prev.next = tile.next;
            tile.next.prev = tile.prev;
            tile.prev = null;
            tile.next = null;
        }
    }

    public VTPhysicalTable(Vector2Int size,byte depth,int pageWidth,int rtCount = 1,GraphicsFormat rtFormat = GraphicsFormat.R8G8B8A8_UNorm) 
    {
        tileCountX = Mathf.CeilToInt(size.x / (float)pageWidth);
        tileCountY=  Mathf.CeilToInt(size.y / (float)pageWidth);   
        realWidth = tileCountX * pageWidth;
        realHeight = tileCountY * pageWidth;
        this.depth = depth;
        for (int z = 0; z < depth; z++)
        {
            for (int y = 0; y < tileCountX; y++)
            {
                for (int x = 0; x < tileCountY; x++)
                {
                    var t = new Tile();
                    t.rect = new Vector4(x * pageWidth, y * pageWidth, pageWidth, pageWidth);
                    t.depth = (byte)z;
                    AddLRU(t);
                }
            }
        }
        CreateRenderTexture(rtCount, rtFormat);
    }
    public virtual void CreateRenderTexture(int rtCount , GraphicsFormat rtFormat )
    {
        rts = new RenderTexture[rtCount];
        matPass = new int[rtCount];
        for (int i = 0; i < rtCount; i++)
        {
            var rt = new RenderTexture(realWidth, realHeight, 0, rtFormat);
            rt.dimension = UnityEngine.Rendering.TextureDimension.Tex2DArray;
            rt.volumeDepth = depth;
            rt.filterMode = FilterMode.Bilinear;
            rt.name = "VTPhysicalMap" + i;
            rt.Create();
            rts[i] = rt;
        }
    }

    /// <summary>
    /// 标识最近使用
    /// </summary>
    public void MarkRecently(Tile tile)
    {
        RemoveLRU(tile);
        AddLRU(tile);
    }

    /// <summary>
    /// 分配Tile
    /// </summary>
    public void AllocAddress(VTPage page,out VTPage oldPage) 
    {
        oldPage = null;
        var tile = lruFirst;
        while (tile != null)
        {
            if (tile.SetCachePage(page,out oldPage))
            {
                page.physTile = tile;
                MarkRecently(tile);
                return;
            }
            tile = tile.next;
        }
#if UNITY_EDITOR
            Debug.LogError("物理贴图空间不足,可能导致渲染错误");
#endif
    }
 
}

```

### 2.Feedback

我们需要收集场景用到的VT Page。Feedback机制就是通过在GPU上计算出Page ID和Mipmap输出到Feedback Buffer。然后通过CPU的异步回读来解析这个Buffer，得到Page信息。

**Feedback Buffer**

我们Buffer格式是32位RGBA8。一般VT划分的Page是1024或2048左右，所以PageID的xy坐标是大于256，那么8位是不够了，一般会把 B通道的 8位拆分到RG里去，即PageID的xy用12位来表示。然后A通道8位来存mipLevel。

| IR8 | G8 | B8 | A8 |
| --- | --- | --- | --- |
| PageID X (低8位) | PageID Y (低8位) | 4位 PageID X(高4位)  
4位 PageID Y(高4位) | mipLevel |

有了这个Feedback Buffer，还需要把数据输出的Buffer上。

一种老派的做法是把Feedback Buffer作为Gbuffer的其中一个渲染目标，利用MRT的特性来输出。

而UE则在PS直接输出到一个无序访问的UAV Buffer，这个Buffer的大小基于屏幕分辨率缩小的，主要是为了加快回读的速度。缩小了分辨率会导致信息丢失，所以看到代码用利用JitterOffset的随机偏移机制，利用多帧把丢失的信息补充回来。

![](https://pica.zhimg.com/v2-38606bf0e263ea501bd4e7c1a8dd14a0_1440w.jpg)

UE Feedback

与 UE和MRT 的不同，为了减少代码的入侵性和方便展示，本文额外绘制了一次地型的Feedback Pass到Feed Buffer中。实际项目中不建议这么处理，因为需要额外的地型绘制，同时这样只会有地型的自遮挡而没有场景的遮挡信息，导致多余的Page被显示。

**Feedback Pass**

本文Feedback Pass的Shader代码如下，UV \* PageCount 得出所属的PageID。然后通过DDX,DDY计算出MipLevel。

![](https://pic4.zhimg.com/v2-1f69320cfa29c8fe149fd1614916cac5_1440w.jpg)

输出Page信息

![](https://pic4.zhimg.com/v2-89ca9003134add63425ebb451377ec0f_1440w.jpg)

计算MipLevel

![](https://pic1.zhimg.com/v2-bf3d091ef6dd6b22b21cece49474ca94_1440w.jpg)

左:Feedback Buffer

**Feedback 降采样**

需要注意的是Feedback Pass是需要在原始屏幕分辨率下渲染的，不能直接使用低分辨率的Buffer来渲染，因为我们的MipLevel是通过DDX,DDY计算的，分辨率改变了这个mip就不正确了。

如果像UE那样用全分辨率渲染，输出到低分辨率的UAV Buffer上就没有这个问题。但是我们并不是，所以需要额外做一次降采样到低分辨率的RenderTarget上。

这里降采样的方法是，在原始Feedback Buffer上圆盘随机采样，避免数据的丢失，随机数每帧变化，多帧后能接近原始数据。

![](https://pica.zhimg.com/v2-686634232cb0120536ede94c1e56d68a_1440w.jpg)

降采样

![](https://picx.zhimg.com/v2-8c49bd7a60b5283fe605918ed23b523b_1440w.jpg)

降采样后

### 3.收集可见Page

有了Feedback Buffer，我们就可以利用异步回读的方式获取他，在Unity里用的是CommandBuffer.RequestAsyncReadback 的API，这里加了同步回读的开关方便FrameDebuger调试

![](https://pic4.zhimg.com/v2-a153846fef8a1b76ed4a574c291a76df_1440w.jpg)

回读成功后，遍历这个Feedback Buffer的每个像素，解析出Page和MipLevel，加到处理列表里。处理列表按照Mip和出现次数排序，可以按照项目需要调整优先处理策略。

```csharp

    Dictionary<VTPage,int> pageReadPixelCounter = new Dictionary<VTPage, int>();
 

    private void OnReadback(AsyncGPUReadbackRequest obj)
    {
        if (!obj.hasError && readyReadback) 
        {
            ClearWaitList();
            var colors = obj.GetData<Color32>();
            for (int i = 0; i < colors.Length; i++) 
            {
                var color = colors[i];
                UnpackPage(color.r, color.g, color.b,out int pageIndexX,out int pageIndexY);
                int pageMip = (int)color.a;
                
                //无效像素
                if (pageMip < 0 || pageIndexX < 0 || pageIndexY < 0 || pageIndexX >= indirectTable.texWidth || pageIndexY>= indirectTable.texHeight || pageMip > maxMip)
                    continue;
              
                var page = indirectTable.GetPage(pageIndexX, pageIndexY, pageMip);
               
                if (pageReadPixelCounter.TryGetValue(page, out var count))//记录Page出现次数
                    pageReadPixelCounter[page]++;
                else
                    pageReadPixelCounter[page] = 0;
                
                if (page.loadFlag == VTPage.FLAG_LOADED)
                {
                    physicalTable.MarkRecently(page.physTile);//加载过,更新LRU列表顺序
                }
                else if (page.loadFlag == VTPage.FLAG_NONE)
                {
                    page.loadFlag = VTPage.FLAG_LOADING;
                    waitLoadPageList.Add(page);
                }

            }
            waitLoadPageList.Sort(ReadbackSortComparer);
            pageReadPixelCounter.Clear();
        }
    }

    private int ReadbackSortComparer(VTPage x, VTPage y)
    {
        if (y.mip != x.mip)
            return y.mip.CompareTo(x.mip);//优先高级
       return pageReadPixelCounter[y].CompareTo(pageReadPixelCounter[x]);//像素多的
    }
    /// <summary>
    /// 解析Page Buffer Color
    /// </summary>
    void UnpackPage(byte x, byte y, byte z,out int X,out int Y)
    {
        uint ix = x;
        uint iy = y;
        uint iz = z;
        // 8 bit in lo, 4 bit in hi
        uint hi = iz >> 4;
        uint lo = iz & 15;
        X = (int)(ix | lo << 8);
        Y = (int)(iy | hi << 8);
    }

```

### 4.更新物理页

有了可视Page之后，我们就可以对可视的Page进行绘制了，为了防止单帧中生成的Page过多，需要限制单帧中最大绘制数量。

通过VTPage去申请Physical Texture的Tile，如果Tile是已经储存了内容，理论上已存内容已经是不需要的了，但是并不能暴力覆盖上面的内容。因为Feedback是异步和延迟的，如果当前帧又刚好渲染到了移除的地方，那么就会造成渲染错误。为了避免这种情况，需要把Indirection Texture旧的Page映射到其他的Mip上后，才能覆盖物理页上的内容。

```csharp
    private List<VTPage> remapPages = new List<VTPage>();
    void CollectLoadPage(bool limit = true)
    {
        for (int i=0;i< physicalUpdateList.Length;i++)
            physicalUpdateList[i].Clear();
        int curRenderSize = 0;
        //分批生成page
        {
            for (int i = 0; i < waitLoadPageList.Count; i++)
            {
                var page = waitLoadPageList[i];
                curRenderSize += pageWidth;
                if (curRenderSize > maxRenderSize && !forceUpdate && limit) //性能限制导致分批
                {
                    break;
                }
                else
                {
                    physicalTable.AllocAddress(page, out var oldPage);
                    page.loadFlag = VTPage.FLAG_LOADED;
                    physicalUpdateList[page.physTile.depth].Add(page);
                    if (oldPage != null) //替换掉旧的Page
                    {
                        remapPages.Add(oldPage);
                    }
                }
            }
        }
        
        for (int i = 0; i < remapPages.Count; i++)
        {
            RemapSubPageIndirect(remapPages[i]);
        }
        remapPages.Clear();
    }  

```

实际更新物理页的时候，使用Instance实例化渲染正方面片在Physical Texture上绘制对应的VT区域。

```csharp
  void UpdatePhysicalTexture() 
    {
        if (compress)
            InitCompress();
        //绘制数据
        int renderCount = 0;
        if (renderPageTBS == null)
        {
            renderPageTBS = new Matrix4x4[MAX_RENDER_BATCH];
            for (int i = 0; i < renderPageTBS.Length; i++)
                renderPageTBS[i] = Matrix4x4.identity;

            renderPageData = new Vector4[MAX_RENDER_BATCH];
        }
        float padding = border;
        float scalePadding = (pageWidth + padding * 2f) / pageWidth;
        float offsetPadding = padding / (pageWidth + padding * 2f);
        //绘制到物理贴图
        cmd.BeginSample(profilerTag);
        var rtArray = physicalTable.rts;
        float pScaleX = pageWidth / (float)physicalTable.realWidth;
        float pScaleY = pageWidth / (float)physicalTable.realHeight;
        cmd.SetViewProjectionMatrices(Matrix4x4.identity, Matrix4x4.identity);
        for (int r = 0; r < rtArray.Length; r++) //rt (baseColor,normal)
        {
            var rt = physicalTable.rts[r];
            int pass = physicalTable.matPass[r];
            for (int t = 0; t < physicalUpdateList.Length; t++) // Texture array index
            {
                var list = physicalUpdateList[t];
                if (list.Count == 0)
                    continue;
                RenderTargetIdentifier depthIdentifier = new RenderTargetIdentifier(rt, 0, CubemapFace.Unknown, t);
                cmd.SetRenderTarget(depthIdentifier, RenderBufferLoadAction.Load, RenderBufferStoreAction.Store, RenderBufferLoadAction.DontCare, RenderBufferStoreAction.DontCare);
                for (int i = 0, len = list.Count, lastIndex = list.Count - 1; i < len; i++) // pages per texture array slice 
                {
                    var page = list[i];
                    var mipmap = indirectTable.mips[page.mip];
                    float scaleX = (mipmap.virtualWidth) / (float)virtualTextureSize.x;
                    float scaleY = (mipmap.virtualWidth) / (float)virtualTextureSize.y;
                    AddIndirectUpatePage(page, page.mip);
                    UpdateSubPageIndirect(page);//更新子页
                    /* Page UV 转 VT UV 数据
                        (uv * pageMipSize + pageStart) /virtualTextureSize
                        pageStart = pageMipIndex * pageMipSize
                    */
                    MortonCode.MortonDecode(page.mortonID,out var localIndexX,out var localIndexY);
                    float biasX = localIndexX * scaleX;
                    float baseY = localIndexY * scaleY;
                    // (uv  - offsetPadding) * scalePadding  * S + B
                    // uv * scalePadding * S -  offsetPadding * scalePadding * S + B
                    float vScaleX = scaleX * scalePadding;
                    float vScaleY = scaleY * scalePadding;
                    float vPosX = -offsetPadding * scalePadding * scaleX + biasX;
                    float vPosY = -offsetPadding * scalePadding * scaleY + baseY;
                    renderPageData[renderCount] = new Vector4(vScaleX, vScaleY, vPosX, vPosY);
                    //物理页矩阵
                    float pPosX = page.physTile.rect.x / (float)physicalTable.realWidth + pScaleX * 0.5f;
                    float pPosY = page.physTile.rect.y / (float)physicalTable.realWidth + pScaleY * 0.5f;
                    pPosX = pPosX * 2f - 1f; //[0,1] =>  [-1,1]
                    pPosY = pPosY * 2f - 1f;
                    ref var tbs = ref renderPageTBS[renderCount];
                    tbs.m03 = pPosX;
                    tbs.m13 = pPosY;
                    tbs.m00 = pScaleX;
                    tbs.m11 = pScaleY;
                    renderCount++;
                    if ((renderCount == MAX_RENDER_BATCH || i == lastIndex) && renderCount > 0)
                    {
                        block.SetVectorArray(_VirtualTextureUVTransform, renderPageData);
                        cmd.DrawMeshInstanced(RenderingUtils.fullscreenMesh, 0, renderPageMat, pass, renderPageTBS, renderCount, block);
                        renderCount = 0;
                    }
                }
                if(compress)
                    gpuCompress.Compress4x4(cmd,rt,r == 0? pBaseColorCompreeTex : pNormalCompreeTex,true,t);
            }
        }
        cmd.EndSample(profilerTag);
        Graphics.ExecuteCommandBuffer(cmd);
        cmd.Clear();
        for (int t = 0; t < physicalUpdateList.Length; t++)
        {
            var list = physicalUpdateList[t];
            for(int i = 0; i < list.Count;i++)
                tempRemoveList.Add(list[i]);
            list.Clear();
        }
        for (int i = 0; i < tempRemoveList.Count; i++)
        {
            var p = tempRemoveList[i];
            waitLoadPageList.Remove(p);
        }
        tempRemoveList.Clear();

    }

```

物理页更新后，需要把Page添加到Indirection Texture更新列表里。

![](https://pic3.zhimg.com/v2-4e23bc1e7725ba871d5965c397a39056_1440w.jpg)

除了更新Page对应Mip的像素外，还要对其更低级的Mip做映射，否则在Feedback回读期间，如果渲染到没被映射的Page(如下图蓝线Page)，也会导致渲染错误。本文的处理方案是让没被渲染的Page映射到已存在于物理贴图的更加高级Page的物理位置，如下图：

![](https://pic1.zhimg.com/v2-108d3294df1ebd2dd1c9248e916c79b6_1440w.jpg)

这里可以通过上面介绍的莫顿码就能快速找到低级mip的Page，然后每级Mip的pages遍历。这部开销其实比较大，或许存在更好的算法去处理这个问题。为了加速遍历和提高Indirect Texture的更新效率，本文使用了重叠绘制的方式，正方形会覆盖整个mip的区域，如果低级Mip已经存于物理贴图，那么会重新更新这个Page像素。

如上图，3个绿色像素不会分3个正方形更新，而是使用一个绿色正方形来更新，然后把红色正方形覆盖回去。

![](https://pic1.zhimg.com/v2-209466e86c76ac24b36967d117a31434_1440w.jpg)

Physical Page绘制的Shader如下

![](https://pic1.zhimg.com/v2-b862546655d80f79f5db84c584450406_1440w.jpg)

![](https://pic4.zhimg.com/v2-32b3f5d4fa248d8f9217abfbcbcc308d_1440w.jpg)

Physcial Texture

### 5.更新间接表

Indirection Texture格式是RGBA32，每通道8位的纹理。通道信息如下：

| R | G | B | A |
| --- | --- | --- | --- |
| Physical TileID X | Physical TileID Y | MipLevel | TextureArray Index |

上面提到了我们使用纹理数组的方式来实现Physical Texture Array，假设我们是1024 \* 1024 \* 8的大小，而每个Pages是128 \* 128，TileID XY = 1024 /128 = 8，即TileID是\[0,7\]区间，8位足够了。

更新Indirection Texture的方式和上面类似。

```csharp
void UpdateIndirectTexture() 
    {
        //绘制间接表信息
        int renderCount = 0;
        cmd.BeginSample(profilerTag);
        cmd.SetViewProjectionMatrices(Matrix4x4.identity, Matrix4x4.identity);
        for (int m = 0; m < indirectUpdateList.Length; m++)
        {
            var list = indirectUpdateList[m];
            if (list.Count == 0)
                continue;
            list.Sort(PageMipComparison);//高级的Mip先画，低级的mip覆盖
            var mipmap = indirectTable.mips[m];
            cmd.SetRenderTarget(indirectTable.rt, mipmap.mipLevel);
            uint bitMask = ~(1u << m);
            for (int i = 0, len = list.Count, lastIndex = list.Count - 1; i < len; i++)
            {
                var page = list[i];
                page.mipMask &= bitMask;//移除需要绘制mip标记
                {
                    float phyBlockIndexX = (int)(page.physTile.rect.x / pageWidth) / 255f; //8bit
                    float phyBlockIndexY = (int)(page.physTile.rect.y / pageWidth) / 255f; //8bit
                    float mip = page.mip / 255f; //8bit
                    float phyTexArrayIndex = page.physTile.depth / 255f; // 纹理数组Index
                    ref var data = ref renderPageData[renderCount];
                    data.x = phyBlockIndexX;
                    data.y = phyBlockIndexY;
                    data.z = mip;
                    data.w = phyTexArrayIndex;
                    
                    MortonCode.MortonDecode(page.mortonID,out var localIndexX,out var localIndexY);
                    float vScaleX, vScaleY, vPosX, vPosY;
                    if (page.mip != (byte)m) //处理回退的page,整个区域都映射到该Mip上
                    {
                        var pageMip = indirectTable.mips[page.mip];
                        float w = pageMip.virtualWidth;
                        vScaleX = w / virtualTextureSize.x;
                        vScaleY = w / virtualTextureSize.y;
                        vPosX = (w * localIndexX) / virtualTextureSize.x;
                        vPosY = (w * localIndexY) / virtualTextureSize.y;
                    }
                    else
                    {
                        vScaleX = 1f / mipmap.size;
                        vScaleY = 1f / mipmap.size;
                        vPosX = localIndexX  * vScaleX;
                        vPosY = localIndexY  * vScaleY;
                    }
                    float vHalfScaleX = vScaleX * 0.5f;
                    float vHalfScaleY = vScaleY * 0.5f;
                    vPosX += vHalfScaleX;
                    vPosY += vHalfScaleY;
                    vPosX = vPosX * 2f - 1f; //[0,1] =>  [-1,1]
                    vPosY = vPosY * 2f - 1f;
                    ref var tbs = ref renderPageTBS[renderCount];
                    tbs.m03 = vPosX;
                    tbs.m13 = vPosY;
                    tbs.m00 = vScaleX;
                    tbs.m11 = vScaleY;
                    renderCount++;
                }
                if (renderCount == MAX_RENDER_BATCH || i == lastIndex)
                {
                    block.SetVectorArray(_VirtualTextureUVTransform, renderPageData);
                    cmd.DrawMeshInstanced(RenderingUtils.fullscreenMesh, 0, renderPageMat, 1, renderPageTBS, renderCount, block);
                    renderCount = 0;
                }
            }
            list.Clear();
        }
        cmd.EndSample(profilerTag);
        Graphics.ExecuteCommandBuffer(cmd);
        cmd.Clear();
    }

```

Shader就很简单了，直接输出的是地址的缩放和映射。

![](https://picx.zhimg.com/v2-01de9ea30cb76aa863c5c6d8bc119b35_1440w.jpg)

![](https://pic3.zhimg.com/v2-47693b1807bfcee431f767546ab45644_1440w.jpg)

IndirectionTexture Mipmap

### 6.使用VT

使用VT就比较简单了, 需要小心的是Physical UV的计算就是了。

Physical UV = ( VTPageUV \* TileSize+ PhysicalTileIndex \* TileSize) / PhyscialTextureSize

VTPageUV 利用MipLevel和 VTPage的大小就可以计算出来了。 这样RVT就基本完成了。

```glsl

float2 ComputePhysicalUV(float4 physicalData,float2 px)
{
    int2 phyBlockInexXY = physicalData.xy;
    int pageMip = physicalData.z;
    int pageWidth = _VTPhysTexParma.x;
    int vtBlockWidth = pageWidth << pageMip;
    int2 vtBlockIndex = px / vtBlockWidth;
    float2 tileUV = (px - vtBlockIndex * vtBlockWidth) / vtBlockWidth;
    tileUV = tileUV * _VTPaddingParam.xy + _VTPaddingParam.zw;//边界处理
    float2 physicalUV = (tileUV * pageWidth + phyBlockInexXY * pageWidth) * _VTPhysTexParma.y;//_VTPhysTexParma: 1.0/PhyscialTextureSize
    return physicalUV;
}


void SampleDiffuseVT(float2 positionSS,float2 uv, inout half3 diffuse,inout half3 normal)
{
    float2 px = uv * _VTParam.y;
    float2 dx = ddx(px);
    float2 dy = ddy(px);
    float mip, nextMip, mipFrac;
    ComputeVTMipData(dx, dy, mip, nextMip, mipFrac);
    float4 physicalData = SAMPLE_TEXTURE2D_LOD(_VTIndirectTex, sampler_VTIndirectTex, uv, mip) * 255.5;
    float2 physicalUV = ComputePhysicalUV(physicalData, px);
    float4 color = SAMPLE_TEXTURE2D_ARRAY_LOD(_VTDiffuse, sampler_VTDiffuse, physicalUV, physicalData.w, 0);
    #ifdef _NORMALMAP
        normal = SAMPLE_TEXTURE2D_ARRAY_LOD(_VTNormal, sampler_VTNormal, physicalUV, physicalData.w, 0).xyz * 2.0 -1.0;
    #endif
    diffuse = color.xyz;
}
```

出来效果和直接混合SplatMap基本差不多。

![](https://pic3.zhimg.com/v2-7bb146ac31d25cf3d877f91898bf8f36_b.gif)

VT开启

在SceneView下可以看出高精度的地方都是靠近相机且在视锥内的。

![](https://pic4.zhimg.com/v2-caf5495e3ef2a7bf1b0bd859318d7583_1440w.jpg)

SceneView视角

Clipmap实现
---------

Clipmap和RVT差异主要在收集Page的步骤。

Indirection Texture也有所不同，每级mip使用相同数量的Page表示，即Indirection Texture不再使用mipmap了，而是使用TextureArray来存放每级Mip信息，每级mip的Page数量是相同的。

Clipmap可以是围绕相机或是围绕主角，也可以不是原定对称而是往相机方向前倾，毕竟灵活。

![](https://pic1.zhimg.com/v2-8b722ee2b43b595c88ac1fc79dead460_1440w.jpg)

左图mip2 右图mip1

### 1.数据准备

这里我用Layer来表示每级Mip。一般6-8级Layer就足够了，每级有 8 \* 8或 16 \* 16的pages。

由于pages数量毕竟少，Indirection Texture使用CPU来更新，所以会有一个Colors数组来存放贴图颜色数据。

![](https://pica.zhimg.com/v2-f3647b6283a2fa12416a2aea7ee8f5ba_1440w.jpg)

![](https://pic1.zhimg.com/v2-034ac26cd079d17f390242b6ea82417a_1440w.jpg)

### 2.滚动更新可见Page

Clipmap一个核心思路是使用有限的格子(page)来表示有限的区域，因为格子是固定数量的，当我们移动的时候，有些格子是相同的不需要更新，有部分需要更新到新的位置，这样格子是可以循环使用的，如下图:

![](https://pica.zhimg.com/v2-4f025919f509046ad5a4ac9ae0f29098_1440w.jpg)

![](https://picx.zhimg.com/v2-261ae4f510e7f97e17f421257ba0d97f_1440w.jpg)

这是个循环寻址的过程，把废弃的Page围绕中心镜像到新的位置，移动后重新调整锚点，这样个过程一直重复，参考上图。思路简单，但是代码毕竟麻烦:

```csharp
    void CollectDirtyPages()
   {
      var terrainBounds = terrain.terrainData.bounds;
      var postion = Camera.main.transform.position;
      var terrainMin = terrain.transform.position;// terrainBounds.min;
      var terrainSize = terrainBounds.size;
      Vector2 terrainUV = new Vector2(postion.x -terrainMin.x,postion.z-terrainMin.z);
      terrainUV.x /= terrainSize.x;
      terrainUV.y /= terrainSize.z;
      terrainUV.x = Mathf.Clamp01(terrainUV.x);
      terrainUV.y = Mathf.Clamp01(terrainUV.y);
      bool anyLayerUpdate = false;
      for (int i = 0; i < layers.Length; i++)
      {
         var layer = layers[i];
         int blockCount = Mathf.CeilToInt(virtualTextureSize / (float)layer.pixelSize);
         var curPos=  new Vector2Int((int)(terrainUV.x * blockCount),(int)(terrainUV.y * blockCount));
         var layerMinX = Mathf.Clamp(curPos.x - pagePerLayer / 2,0,blockCount-pagePerLayer);
         var layerMinY = Mathf.Clamp(curPos.y - pagePerLayer / 2,0,blockCount-pagePerLayer);
         var preRect = layer.prevRectXY;
         if (layerMinX != preRect.x || layerMinY != preRect.y || forceUpdate || !isInit)
         {
            var pages = layer.pages;
            var layerMaxX = layerMinX + pagePerLayer -1;
            var layerMaxY = layerMinY + pagePerLayer -1;

            int offsetX = layerMinX - preRect.x;
            int offsetY = layerMinY - preRect.y;
            if (forceUpdate || !isInit)
            {
               offsetX = pagePerLayer;
               offsetY = pagePerLayer;
            }
            offsetX = Mathf.Clamp(offsetX,-pagePerLayer,pagePerLayer);
            offsetY = Mathf.Clamp(offsetY,-pagePerLayer,pagePerLayer);
            if (offsetX >= 0)
            {
               for (int x = 0; x < offsetX; x++)
               {
                  for (int y = 0; y < pagePerLayer; y++)
                  {
                     int localX = (x + layer.rectOffset.x) % pagePerLayer;
                     int localY = (y + layer.rectOffset.y) % pagePerLayer;
                     int pageIndex = localX + localY * pagePerLayer;
                     var p = pages[pageIndex];
                     int newLocalX = pagePerLayer - offsetX + x;
                     p.gx = layerMinX + newLocalX; //new global X
                     AddLoadPadge(p);
                  }
               }
            }
            else // offSetX <= 0
            {
               for (int x = pagePerLayer + offsetX; x < pagePerLayer; x++)
               {
                  for (int y = 0; y < pagePerLayer; y++)
                  {
                     int localX = (x + layer.rectOffset.x) % pagePerLayer;
                     int localY = (y + layer.rectOffset.y) % pagePerLayer;
                     int pageIndex = localX + localY * pagePerLayer;
                     var p = pages[pageIndex];
                     int newLocalX = x- (pagePerLayer + offsetX);
                     p.gx = layerMinX + newLocalX;
                     AddLoadPadge(p);
                  }
               }
            }
            
            if (offsetY >= 0)
            {
               for (int y = 0; y < offsetY; y++)
               {
                  for (int x = 0; x < pagePerLayer; x++)
                  {
                     int localX = (x + layer.rectOffset.x) % pagePerLayer;
                     int localY = (y + layer.rectOffset.y) % pagePerLayer;
                     int pageIndex = localX + localY * pagePerLayer;
                     var p = pages[pageIndex];
                     int newLocalY = pagePerLayer - offsetY + y;
                     p.gy = layerMinY + newLocalY;
                     AddLoadPadge(p);
                  }
               }
            }
            else // offSetY <= 0
            {
               for (int y = pagePerLayer + offsetY; y < pagePerLayer; y++)
               {
                  for (int x = 0; x < pagePerLayer; x++)
                  {
                     int localX = (x + layer.rectOffset.x) % pagePerLayer; 
                     int localY = (y + layer.rectOffset.y) % pagePerLayer;
                     int pageIndex = localX + localY * pagePerLayer;
                     var p = pages[pageIndex];
                     int newLocalY =  y- (pagePerLayer + offsetY); 
                     p.gy = layerMinY + newLocalY;
                     AddLoadPadge(p);
                  }
               }
            }

            if (!isInit)
            {
               layer.rectOffset.x = 0;
               layer.rectOffset.y = 0;
            }
            else
            {
               layer.rectOffset.x += offsetX;
               layer.rectOffset.y += offsetY;
            }
            
            if(layer.rectOffset.x<0)
               layer.rectOffset.x = layer.rectOffset.x % pagePerLayer + pagePerLayer;
            if(layer.rectOffset.y<0)
               layer.rectOffset.y = layer.rectOffset.y % pagePerLayer + pagePerLayer;
            layer.prevRectXY = new RectInt(layerMinX, layerMinY,layerMaxX,layerMaxY);
          float pageMinPX = layerMinX * layer.pixelSize;
          float pageMinPY = layerMinY * layer.pixelSize;

          float pageMaxPX = (layerMaxX+1) * layer.pixelSize;
          float pageMaxPY = (layerMaxY+1) * layer.pixelSize;
          clipmapLayerRectArray[i] = new Vector4(pageMinPX, pageMinPY, pageMaxPX, pageMaxPY);
          float clipXS = 1f/ (layer.pixelSize * pagePerLayer);
          float clipYS = 1f/ (layer.pixelSize);
          float clipTX = -layerMinX / (float)pagePerLayer;
          float clipTY = -layerMinY / (float)pagePerLayer;
          clipmapLayerUVSTArray[i] = new Vector4(clipXS, clipYS, clipTX, clipTY);
          if (!anyLayerUpdate)
             anyLayerUpdate = true;
         }
      }

      if (anyLayerUpdate|| true)
      {
         var mat = terrain.materialTemplate;
         Shader.SetGlobalVectorArray(_CLIPMAP_LAYER_RECT_ARRAY, clipmapLayerRectArray);
         Shader.SetGlobalVectorArray(_CLIPMAP_LAYER_UVST_ARRAY, clipmapLayerUVSTArray);
      }
      if(!isInit)
         isInit = true;
   }

```

### 3.更新间接贴图

通过上述步骤知道更新的Page后，直接调度Job来更新间接贴图。

```csharp
   void ScheduleJob()
   {
      for (int i = 0; i < layers.Length; i++)
      {
         var layer = layers[i];
         if(layer.needUpdate == false)
            continue;
         var job = new IndirectTexColorJob()
         {
            colors = layer.colors,
            layerRect = layer.prevRectXY,
            pagesDatas = layer.pageJobDatas,
            pageWidth = pageWidth,
            pagePerLayer = pagePerLayer
         };
         layer.colorJobHandle = job.Schedule(layer.pages.Length,layer.pages.Length);
      }
   }  

   struct IndirectTexColorJob : IJobParallelFor
   {
      [WriteOnly]
      public NativeArray<Color32> colors;
      [ReadOnly]
      public NativeArray<ClipmapPageData> pagesDatas; 
      public RectInt layerRect;
      public int pagePerLayer;
      public int pageWidth;
      public void Execute(int index)
      {
         var pageData = pagesDatas[index];
         int x = pageData.virturalPageIndex.x - layerRect.x;
         int y = pageData.virturalPageIndex.y - layerRect.y;
         int pageIndex = y*pagePerLayer + x; 
         byte phyBlockIndexX = (byte)(pageData.physicalRect.x / pageWidth) ; //8bit
         byte phyBlockIndexY = (byte)(pageData.physicalRect.y / pageWidth) ; //8bit
         byte phyTexArrayIndex = pageData.physicalIndex ; // 纹理数组Index
         colors[pageIndex] = new Color(phyBlockIndexX/255f,phyBlockIndexY/255f,1f,phyTexArrayIndex/255f);
      }
   }

```

在Job更新的同时，我们可以并行开始更新Physical Texture，物理贴图更新步骤和RVT部分大同小异，不再赘述了。Physical Texture更新完成后，就可以同步Job的数据，填充给TextureArray了。

![](https://pic4.zhimg.com/v2-37bc97e4518d8d630e58a5b05ca88cb9_1440w.jpg)

调度Job

![](https://pic4.zhimg.com/v2-3bdc71cf4d1cbecee049ef60ab9df58d_1440w.jpg)

更新Indirection Texture

### 4.渲染Clipmap Texture

采样的方法很简单，先确定像素在哪个Layer上，然后采样对应Layer的 Indriection Texture Index。

```glsl
#define MAX_LAYER_COUNT 6

float4 _CLIPMAP_LAYER_RECT_ARRAY[MAX_LAYER_COUNT];
float4 _CLIPMAP_LAYER_UVST_ARRAY[MAX_LAYER_COUNT];

void SampleClipmapVT(float2 positionSS,float2 uv, inout half3 diffuse,inout half3 normal)
{
  
    float2 px = uv * _VTParam.y;
    int layer = MAX_LAYER_COUNT - 1;
    float4 rect;
    //判断所属Layer
    for(int i=0;i<MAX_LAYER_COUNT;i++)
    {
        rect =_CLIPMAP_LAYER_RECT_ARRAY[i];
        if(all(px >= rect.xy &&  px <= rect.zw))
        {
            layer = i;
            break;
        }
    }
    layer = min(layer, _ClipmapParm.y);//_ClipmapParm.y是实际Layer数量，防止越界
    float4 indirectUVST =_CLIPMAP_LAYER_UVST_ARRAY[layer];
    float2 indirectUV = px * indirectUVST.xx +indirectUVST.zw;
    float4 physicalData = SAMPLE_TEXTURE2D_ARRAY_LOD(_VTIndirectTexArray, sampler_VTIndirectTexArray, indirectUV,layer,0) * 255.5;
    int2 phyBlockInexXY = physicalData.xy;
    int pageWidth = _VTPhysTexParma.x;
    float2 tileUV = px * indirectUVST.y;
    tileUV = tileUV - floor(tileUV);//取余
    tileUV = tileUV * _VTPaddingParam.xy + _VTPaddingParam.zw;//padding
    float2 physicalUV = (tileUV * pageWidth + phyBlockInexXY * pageWidth) * _VTPhysTexParma.y;
    float4 color = SAMPLE_TEXTURE2D_ARRAY_LOD(_VTDiffuse, sampler_VTDiffuse, physicalUV, physicalData.w, 0);
    #ifdef _NORMALMAP
    normal = SAMPLE_TEXTURE2D_ARRAY_LOD(_VTNormal, sampler_VTNormal, physicalUV, physicalData.w, 0).xyz * 2.0 -1.0;
    #endif

    diffuse = color.xyz;
}
```

这样Clipmap Texture就大致完成了

![](https://pic2.zhimg.com/v2-7d1190b75987a874b90f54190ab64165_1440w.jpg)

Clipmap Texture地型

纹理过滤
----

### 1.双线插值

因为PhysicalTexture是多张Page并存的，当采样边界时如果没有没有Padding边界就会错误采样到隔壁的Page,导致错误。

一种解决方法就是加的边界，即Page如果是128加入4边界后是132。

本文采样另外一种方法是UV缩放来解决，这招之前ProbeBaseGI文章已经用过。

```text
//生成Page时候缩小
float2 scalePadding = ((_BlitPaddingSize.xy + float(_BlitPaddingSize.z)) / _BlitPaddingSize.xy);
float2 offsetPadding = (float(_BlitPaddingSize.z) / 2.0) / (_BlitPaddingSize.xy + _BlitPaddingSize.z);
uv = (uv - offsetPadding) * scalePadding;


//采样时候放大回来
float2 scalePadding = ((_BlitPaddingSize.xy + float(_BlitPaddingSize.z)) / _BlitPaddingSize.xy);
float2 offsetPadding = (float(_BlitPaddingSize.z) / 2.0) / (_BlitPaddingSize.xy + _BlitPaddingSize.z);
uv = uv / scalePadding + offsetPadding;
```

### 2.三线性插值

三线性指Mipmap之间的插值。

一种方案利用Dither噪点采样，省性能但有噪点，配合TAA可以把噪点干掉。

![](https://pic3.zhimg.com/v2-684befc4c19e2bf8799fd65680de98a6_1440w.jpg)

这里我把Mipmap的精度差异放大，方便观察效果。

![](https://pica.zhimg.com/v2-701ea984492d1abc827156e3e4db4466_1440w.jpg)

左:Clipmap过滤区域 右:噪点伪过度

另一种是老老实实采样两次(mip和mip+1)VT做插值:

![](https://pica.zhimg.com/v2-3f1adcbafd6f46a723986ea8b2e18f76_1440w.jpg)

### 3.各项异性过滤

TODO

VT压缩
----

为了提高渲染效率，Physical Texture可以压缩成对应平台的格式，所以我们把Physical Texture设置成TextureArray压缩效率也更高了，只需对应更新过的单层做压缩。

本文压缩使用的是UE源码中的ETCCompressionCommon.ush 和 BCCompressionCommon.ush，然后使用的PS 来压缩。需要注意的是格式需要时Uint的。

![](https://pic2.zhimg.com/v2-96f017f6da85b5e2703705f63580d3f3_1440w.jpg)

![](https://picx.zhimg.com/v2-523201f6dcfd204c11bbdb99a873b07b_1440w.jpg)

![](https://pic3.zhimg.com/v2-66e9396e9d57ae3dcb559cef84f1a4e6_1440w.jpg)

还有如果用PS来压缩，采样中心在原点，4x4压缩，需要-1.5回起始点。

![](https://pic1.zhimg.com/v2-0f08bde7099af2159e794e4a541b8ad6_1440w.jpg)

压缩后可以通过Renderdoc和FramgeDebuger看到贴图是BC7的了。

![](https://pica.zhimg.com/v2-f2774fa96a679d8d131929dfabf00e1a_1440w.jpg)

![](https://pica.zhimg.com/v2-d65222758664b31ad515948a7617c7e0_1440w.jpg)

总结
--

![](https://picx.zhimg.com/v2-6dcf2f8bd1b9c9e5f751eb98b1c2c5df_1440w.jpg)

SVT可以解决纹理对内存压力，

RVT可以为程序纹理生成缓存。

不得不说整套下来涉及的工程细节真的太多了。

本文仅实现了RVT，想过去SVT似乎也可以实现，把图片切割后打成一个Assetbundle，通过LoadObject("page\_"+N)的方式来管理。

### 参考

GPU Pro1 ：Virtual Texture Mapping

GPU Pro7 ：Adaptive Virtual Textures

[Terrain Rendering in 'Far Cry 5'](https://link.zhihu.com/?target=https%3A//gdcvault.com/play/1025480/Terrain-Rendering-in-Far-Cry)

[Terrain in Battlefield3](https://link.zhihu.com/?target=https%3A//www.gamedevs.org/uploads/battlefield-3-terrain.pdf)

[Chen Ka AdaptiveVirtualTexture](https://link.zhihu.com/?target=https%3A//ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2015/presentations/Chen_Ka_AdaptiveVirtualTexture.pdf)

[The Clipmap: A Virtual Mipmap](https://link.zhihu.com/?target=https%3A//notkyon.moe/vt/Clipmap.pdf)

[Software-Virtual-Textures](https://link.zhihu.com/?target=https%3A//notkyon.moe/vt/Software-Virtual-Textures.pdf)

[Virtual Texture（虚拟纹理）的理解和应用 | Epic 李文磊](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1KK411L7Rg/%3Fspm_id_from%3D333.337.search-card.all.click%26vd_source%3D07d4f665c85998941c935676c2e50d81)
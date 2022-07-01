本文记录UE中Texture的用法(Buffer基本等价)
包括RDG新用法及原先的用法

#### Texture基本概念 
Desc: 纹理的格式描述
Texture: 纹理资源(RHI层面上的)
UAV/SRV/RTV/DSV:  纹理视图

通常先新建纹理描述，再创建纹理，最后创建纹理视图传入shader,一个纹理可以创建多个纹理视图，纹理视图只是纹理的硬件解释，可以理解成C++的引用，同时创建多个纹理视图(例如创建多个UAV)，最后的修改都会覆盖在同一个纹理资源上。


#### 基本用法 
cpp:

step1:  创建纹理资源
```
FTextureRHIRef Tex0 = RHICreateTexture2D(Width, Height, Format, NumMips, NumSamples, TextureFlags, RHIAccess, CreateInfo);
// step1参数就是纹理的Desc分别是: Width(宽), Height(高), Format(格式), NumMips(mip数量), 
// NumSamples(采样器数量), TextureFlags(纹理创建的用途), RHIAccess(是否允许读写的标志，可以不填 通过RHICmdList.Transition修改), CreateInfo(纹理额外信息，通常是纹理的名字)
//
// 具体的用法 
// FTextureRHIRef Tex0 = RHICreateTexture2D(100, 100, PF_R16G16B16A16_UNORM, 1, 1, TexCreate_ShaderResource | TexCreate_UAV, TEXT("Texture0"));
// 该纹理将会被创建成长宽100*100，格式RGBA各16位范围在0-1.0，mip只有1级，采样器使用1个，纹理可以被创建成SRV与UAV即可以被shader读写，
// 名字叫做Texture0 可以在renderdoc nsight等工具的纹理资源视窗中看到。

```
这里需要注意的是TextureFlags(类型ETextureCreateFlags)，通常使用如下的Define来设定
|   类型ETextureCreateFlags  | 用途  |
|  ----  | ----  |
|TexCreate_None                           |        空占位，创建RT的时候有用，历史原因，RT的参数要写2个flag InFlags\|InTargetableFlags，通常第一个Flags固定写TexCreate_None，后一个Flags写纹理需要的功能
|TexCreate_RenderTargetable              |         纹理可作为渲染目标(RT)
|TexCreate_ResolveTargetable              |        
|TexCreate_DepthStencilTargetable         |       纹理可作为深度模板
|TexCreate_ShaderResource                 |         纹理可作为SRV
|TexCreate_SRGB                           |         纹理将在sRGB伽马空间中编码
|TexCreate_CPUWritable                    |         纹理CPU可写，通过FMemory::Memcpy将buffer的内容写入到texture中
|TexCreate_NoTiling                       |         纹理将以非平铺格式创建
|TexCreate_VideoDecode                    |         纹理将用于视频解码
|TexCreate_Dynamic                        |         纹理可以每帧更新
|TexCreate_InputAttachmentRead            |
|TexCreate_Foveation                     |
|TexCreate_3DTiling                       |       纹理将以3D平铺格式创建
|TexCreate_Memoryless                     |       仅Mobile支持，纹理将在TBDR GPUs架构的内存上创建
|TexCreate_GenerateMipCapable             |
|TexCreate_FastVRAMPartialAlloc           |
|TexCreate_DisableSRVCreation            |       
|TexCreate_DisableDCC                     |      不允许该纹理使用DCC压缩
|TexCreate_UAV                            |      纹理可作为UAV
|TexCreate_Presentable                   |
|TexCreate_CPUReadback                   |       纹理可写回CPU，通过RHICmdList.MapStagingSurface接口将纹理写回到buffer中
|TexCreate_OfflineProcessed              |        纹理可以离线处理
|TexCreate_FastVRAM                       |      纹理可以在FastVram中分配。
|TexCreate_HideInVisualizeTexture         |      纹理将不在debug工具中显示
|TexCreate_Virtual                        |      纹理将在虚拟内存中创建，不进行物理内存分配 需要自己调用 RHIVirtualTextureSetFirstMipInMemory分配物理内存
|TexCreate_TargetArraySlicesIndependently |
|TexCreate_Shared                        |       TexCreate_Shared
|TexCreate_NoFastClear                    |      纹理不会被快速清除
|TexCreate_DepthStencilResolveTarget      |      
|TexCreate_Streamable                     |     纹理可以进行流式传输
|TexCreate_NoFastClearFinalize            |     纹理会被刷新，但不会进行快速清除
|TexCreate_AFRManual                      |     纹理支持多GPU交替帧渲染
|TexCreate_ReduceMemoryWithTilingMode     |     PS4/PS5平台的平铺模式(可用于减少纹理内存占用)
|TexCreate_Transient                      |     5.0正式版本已经弃用
|TexCreate_AtomicCompatible               |                             支持原子操作比较，如果没定义会导致比较结果永远是true,例如InterlockedMax
|TexCreate_External               		 |                纹理仅限Vulkan使用，将纹理分配给外部访问 能提高访问效率 参考AllocateDedicatedImageMemory
|TexCreate_MultiGPUGraphIgnore                        |   禁止RDG在多GPU间自动传递该纹理视图
|还有个没degfine的 ETextureCreateFlags::Atomic64Compatible |            支持64位原子操作比较，如果没定义会导致比较结果永远是true，例如NvInterlockedMaxUint64

step2:  创建纹理视图
```
FUnorderedAccessViewRHIRef UAV0 = RHICreateUnorderedAccessView(Tex0, MipLevel);
// step2的参数Tex0就是第一步的Texture资源，MipLevel是指使用第几级Mip，通常用0即可
// 具体的用法 FUnorderedAccessViewRHIRef UAV0 = RHICreateUnorderedAccessView(Tex0, 0);
```

step3: 在cpp的shader类内定义传入shader的参数
```
class FCustomCS: public FGlobalShader
{
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
            SHADER_PARAMETER(int32, KernelHWidth)
            SHADER_PARAMETER_UAV(RWTexture2D<float4>, UAV0)
    END_SHADER_PARAMETER_STRUCT()
}
```

step4:  构造传入shader的参数并赋值
```
FCustomCS::FParameters PassParameters;
PassParameters.KernelHWidth = 1;
PassParameters.UAV0 = UAV0;
SetShaderParameters(RHICmdList, Shader, RHIShader, PassParameters)
// 将step2创建的视图传入到PassParameters结构中
```



shader:
step5:  在usf/ush文件内定义相同的参数，名字一样就行，顺序没要求
```
RWTexture2D<float4> UAV0;
int KernelHWidth;
```




#### RDG(Render Dependency Graph)
UE5中已经基本普及,核心功能是将编译执行的渲染命令加入到一个有向无环图中，同时会自动剔除掉不需要的指令，提高渲染效率。
在Debug时需要将```ConsoleVariables.ini```文件中的```r.RDG.Debug=1```以及```r.RDG.ImmediateMode=1```打开，
这样就可以在render thread中debug各pass流程，否则会使用多线程调用，不适合调试。

#### RDG的用法

step0:  创建RDG纹理描述
```
FRDGTextureDesc Desc0 = FRDGTextureDesc::Create2D(FIntPoint(Width, Height), Format, FClearValueBinding::None, TextureFlags, NumMips，NumSamples);
与基本用法的区别在多了一个FClearValueBinding，用于清理Texture
```
step1:  创建RDG纹理资源
```
FRDGTextureRef RDGTex0 = GraphBuilder.CreateTexture(Desc0, TEXT("Texture0"));
```
step2:  创建RDG纹理视图
```
FRDGTextureUAVRef RDGUAV0 = GraphBuilder.CreateUAV(RDGTex0,ERDGUnorderedAccessViewFlags Flag = ERDGUnorderedAccessViewFlags::None);
// 第二个参数Flag在确保该UAV可以被多个shader无序访问时，可以设为1减少Barrier数量, 通常不需要修改。
// 也可以下面这样调用(但不推荐，上面的用法是UE包装过的)
// FRDGTextureUAVRef RDGUAV0 = GraphBuilder.CreateUAV(FRDGTextureUAVDesc(RDGTex0), Flag = ERDGUnorderedAccessViewFlags::None)
```
step3: 在cpp的shader类内定义传入shader的参数
```
class FCustomCS: public FGlobalShader
{
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
            SHADER_PARAMETER(int32, KernelHWidth)
            SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, RDGUAV0)   // 注意这里与默认用法不同SHADER_PARAMETER_RDG_TEXTURE_UAV
    END_SHADER_PARAMETER_STRUCT()
}
```

step4:  构造传入shader的参数并赋值
```
FCustomCS::FParameters PassParameters = GraphBuilder.AllocParameters<FCustomCS::FParameters>();
PassParameters.KernelHWidth = 1;
PassParameters.UAV0 = UAV0;
SetShaderParameters(RHICmdList, Shader, RHIShader, PassParameters)
// 将step2创建的视图传入到PassParameters结构中
```


shader:
step5:  在usf/ush文件内定义相同的参数，名字一样就行，顺序没要求
```
RWTexture2D<float4> UAV0;
int KernelHWidth;
```  

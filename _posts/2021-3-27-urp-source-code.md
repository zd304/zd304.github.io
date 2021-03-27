---
layout: post
title:  "URP源码解析"
categories: 渲染
tags: Unity 渲染 URP
author: zack.zhang
---

* content
{:toc}

URP（Universal Render Pipeline）作为一个Unity预先构建好的SRP（Scriptable Render Pipeline），提供了一个对美术友好的工作流，让美术能够方便且快速地跨平台创造优化的图形。<a href="https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@8.1/manual/index.html">官方文档</a>

<!-- more -->

## UniversalRenderPipelineAsset

要使用URP，需要先创建一个URP Asset，并且将它赋给Graphics Settings。URP Asset便是UniversalRenderPipelineAsset这个类的实例。其保存了URP的一些设置，这里我们不去详细介绍。这里要去关注的是CreatePipeline()方法，它继承于父类RenderPipelineAsset。

```csharp
//
// 摘要:
//     Create a IRenderPipeline specific to this asset.
//
// 返回结果:
//     Created pipeline.
protected abstract RenderPipeline CreatePipeline();
```

URP Asset实现了父类的方法CreatePipeline()，在方法中URP Asset创建了管线对象UniversalRenderPipeline。

```csharp
protected override RenderPipeline CreatePipeline()
{
	...
	return new UniversalRenderPipeline(this);
}
```

该方法可以理解为URP管线的总的入口函数。

## UniversalRenderPipeline

管线类UniversalRenderPipeline继承自RenderPipeline，其核心方法为Render(...)。

```csharp
//
// 摘要:
//     Defines custom rendering for this RenderPipeline.
//
// 参数:
//   context: 可编程渲染的上下文
//
//   cameras: 本帧所有需要渲染的相机
protected abstract void Render(ScriptableRenderContext context, Camera[] cameras);
```

### Render(...)

Render(...)方法每帧都会被自动调用，在方法中，会处理本帧需要执行的所有渲染命令，来绘制本帧图像。以下为该方法的主要调用。

```csharp
protected override void Render(ScriptableRenderContext renderContext, Camera[] cameras)
{
	// 1.
	BeginFrameRendering(renderContext, cameras);
	
	...
	
	// 2.
	SortCameras(cameras);
	// 3.
	for (int i = 0; i < cameras.Length; ++i)
	{
		var camera = cameras[i];
		if (IsGameCamera(camera))
		{
			// 4.
			RenderCameraStack(renderContext, camera);
		}
		else
		{
			// 5.
			BeginCameraRendering(renderContext, camera);
			
			UpdateVolumeFramework(camera, null);
			
			RenderSingleCamera(renderContext, camera);
			EndCameraRendering(renderContext, camera);
		}
	}
	// 6.
	
	// 7.
	EndFrameRendering(renderContext, cameras);
}
```

1. BeginFrameRendering：表示该帧即将开始渲染。

2. SortCameras：根据所有要渲染的相机的depth值进行排序，depth越小越先渲染。

3. 遍历每一个相机。

4. 如果当前相机是主相机（也就是cameraType == CameraType.Game且renderType != CameraRenderType.Overlay），则调用RenderCameraStack，这一步在接下来会详细描述。

5. 如果当前相机不是游戏相机，比如SceneView相机、预览相机等，则调用相机渲染的通常步骤，BeginCameraRendering → UpdateVolumeFramework → RenderSingleCamera → EndCameraRendering，这些步骤暂且称作**“相机渲染常规步骤”**，后续再详细描述。

6. 遍历相机完成。

7. EndFrameRendering：表示该帧渲染结束，提交后备缓冲区。

### RenderCameraStack(...)

如上文所述，Render(...)方法里如果遍历到主相机，就会去调用RenderCameraStack(...)。RenderCameraStack(...)方法主要是去遍历主相机的CameraStack里的每一个Overlay相机，并且把主相机和所有生效的Overlay相机全部渲染出来。以下为该方法的主要调用。

```csharp
static void RenderCameraStack(ScriptableRenderContext context, Camera baseCamera)
{
	// 1.
	baseCamera.TryGetComponent<UniversalAdditionalCameraData>(out var baseCameraAdditionalData);
	
	List<Camera> cameraStack = baseCameraAdditionalData.cameraStack;
	bool anyPostProcessingEnabled = renderer.supportedRenderingFeatures.cameraStacking;
	
	int lastActiveOverlayCameraIndex = -1;
	
	// 2.
	if (cameraStack != null && cameraStack.Count > 0)
	{
		for (int i = 0; i < cameraStack.Count; ++i)
		{
			Camera currCamera = cameraStack[i];
			
			//<更新设置anyPostProcessingEnabled和lastActiveOverlayCameraIndex>...</>//
		}
	}
	
	// 3.
	bool isStackedRendering = lastActiveOverlayCameraIndex != -1;
	
	// 4.
	BeginCameraRendering(context, baseCamera);
	UpdateVolumeFramework(baseCamera, baseCameraAdditionalData);
	InitializeCameraData(baseCamera, baseCameraAdditionalData, out var baseCameraData);
	RenderSingleCamera(context, baseCameraData, !isStackedRendering, anyPostProcessingEnabled);
	EndCameraRendering(context, baseCamera);
	
	// 5.
	if (!isStackedRendering)
		return;
	
	// 6.
	for (int i = 0; i < cameraStack.Count; ++i)
	{
		var currCamera = cameraStack[i];
		
		if (!currCamera.isActiveAndEnabled)
			continue;
		
		currCamera.TryGetComponent<UniversalAdditionalCameraData>(out var currCameraData);
		if (currCameraData != null)
		{
			// 7.
			bool lastCamera = i == lastActiveOverlayCameraIndex;
			
			BeginCameraRendering(context, currCamera);
			UpdateVolumeFramework(currCamera, currCameraData);
			InitializeAdditionalCameraData(currCamera, currCameraData, ref overlayCameraData);
			RenderSingleCamera(context, overlayCameraData, lastCamera, anyPostProcessingEnabled);
			EndCameraRendering(context, currCamera);
		}
	}
	// 8.
}
```

1. 初始化anyPostProcessingEnabled和lastActiveOverlayCameraIndex，用于当作参数传入接下来所有生效相机的“相机渲染常规步骤”里面。

2. 遍历主相机的CameraStack的每一个相机，如果其中任何一个生效的相机需要渲染后期效果（Post-Processing），则anyPostProcessingEnabled为true；将最后一个相机的遍历序号i记录在lastActiveOverlayCameraIndex变量里。遍历每一个相机时候都会去检查它的“生效”性，如果不是“生效”的相机就会提示警告。“生效”的检查范围包括：是否Active，是否Enabled，是否是Overlay相机，是否scriptableRenderer类型和主相机保持一致。

3. 如果没有遍历访问到任何Overlay相机，则lastActiveOverlayCameraIndex依然为-1，反之则大于-1。根据lastActiveOverlayCameraIndex的值就可以判断有没有后续生效相机需要渲染，需不需要继续遍历CameraStack的所有相机，将这个判断得出的bool值保存到变量isStackedRendering里。

4. 对主相机做“相机渲染常规步骤”，传入参数为“是否是最后一个相机”和“是否需要处理后期效果”。如果isStackedRendering == false则说明主相机已经是最后一个相机了，后面没有可以渲染的相机了；如果anyPostProcessingEnabled == true说明需要处理后期效果。

5. 如果CameraStack里没有相机了，就返回吧。

6. 开始遍历CameraStack里每一个相机。

7. 对CameraStack里每一个相机做“相机渲染常规步骤”，传入参数也是“是否是最后一个相机”和“是否需要处理后期效果”。如果当前相机的遍历序号i等于lastActiveOverlayCameraIndex，那就说明这个相机已经是最后一个相机了，后面没有可以渲染的相机了；如果anyPostProcessingEnabled == true说明需要处理后期效果。

8. 结束遍历CameraStack里每一个相机。

### “相机渲染常规步骤”

#### BeginCameraRendering(...)

RenderPipeline的protected方法，表示某一个相机即将开始渲染，渲染相机的固定调用。

#### UpdateVolumeFramework(...)

更新当前相机是否在某一个后期效果的Volume内，如果在Volume内则触发对应的后期效果。

#### InitializeCameraData(...)

首先根据官方文档，在URP里相机上会绑一个叫做UniversalAdditionalCameraData的脚本。该脚本包含的变量，描述了该相机是否有某些渲染特性，比如是否需要渲染DepthTexture，是否需要渲染OpaqueTexture等。InitializeCameraData(...)就是将这些变量值从当前相机的UniversalAdditionalCameraData脚本里提取出来，供相机内的渲染使用。

#### RenderSingleCamera(...)

该方法用于渲染一个相机，其过程主要包括剪裁、设置渲染器、执行渲染器三步。后续会详细描述。

#### EndCameraRendering(...)

RenderPipeline的protected方法，表示某一个相机已经结束渲染，渲染相机的固定调用。

### RenderSingleCamera(...)

如上文所述，方法过程主要包括剪裁、设置渲染器、执行渲染器三步。以下为该方法的主要调用。

```csharp
/// <summary>
/// 渲染一个相机，其过程主要包括剪裁、设置渲染器、执行渲染器三步。
/// </summary>
/// <param name="context">渲染上下文用于记录执行过程中的命令。</param>
/// <param name="cameraData">相机渲染数据，里面可能包含了继承自相机的一些参数</param>
/// <param name="requiresBlitToBackbuffer">如果这是相机渲染栈里的最后一个相机则为true，否则为false</param>
/// <param name="anyPostProcessingEnabled">如果相机需要做后期效果处理则为true，否则为false</param>
static void RenderSingleCamera(ScriptableRenderContext context, CameraData cameraData, bool requiresBlitToBackbuffer, bool anyPostProcessingEnabled)
{
	// 1.
	var renderer = cameraData.renderer;
	// 2.
	if (!camera.TryGetCullingParameters(IsStereoEnabled(camera), out var cullingParameters))
		return;
	
	// 3.
	CommandBuffer cmd = CommandBufferPool.Get(sampler.name);
	
	// 4.
	renderer.Clear(cameraData.renderType);
	// 5.
	renderer.SetupCullingParameters(ref cullingParameters, ref cameraData);
	
	// 6.
	context.ExecuteCommandBuffer(cmd);
	// 7.
    cmd.Clear();
	
	// 8.
	var cullResults = context.Cull(ref cullingParameters);
	// 9.
	InitializeRenderingData(asset, ref cameraData, ref cullResults, requiresBlitToBackbuffer, anyPostProcessingEnabled, out var renderingData);
	
	// 10.
	renderer.Setup(context, ref renderingData);
	// 11.
    renderer.Execute(context, ref renderingData);
	
	context.ExecuteCommandBuffer(cmd);
	CommandBufferPool.Release(cmd);
	context.Submit();
}
```

1. 获取当前相机的渲染器renderer，其基类类型是ScriptableRenderer，可以继承以作扩展。

2. 获得当前相机的剪裁参数，保存在变量cullingParameters里。

3. 申请一个CommandBuffer来执行渲染命令。

4. 清空渲染器，也就是重置里面的一些数据。

5. 根据相机再去修改一下变量cullingParameters里的信息。

6. 执行当前的渲染命令。

7. 清空CommandBuffer，以供接下来渲染使用。

8. 根据剪裁参数cullingParameters执行相机剪裁，并将剪裁结果储存在cullResults里面。

9. 根据当前帧的剪裁结果、灯光状态等每帧可能会改变的数据，来初始化本帧渲染需要用到的渲染数据renderingData。

10. 调用渲染器的Setup(...)，主要是根据当前渲染数据，去设置本帧渲染需要用到的渲染过程到队列中，这些渲染过程在这里被命名为Pass，其基类类型为ScriptableRenderPass，可以继承扩展。后面会针对前向渲染器（ForwardRenderer）作详细描述。

11. 调用渲染器的Execute(...)，执行已经在队列中的渲染过程。后面会针对前向渲染器（ForwardRenderer）作详细描述。

## ForwardRenderer

ForwardRenderer继承于ScriptableRenderer，这个渲染器被所有支持URP的平台所支持。这个渲染器维护了一个ScriptableRenderPass的列表，每一帧都会往列表里加入Pass，帧中执行Pass得到每一个过程的渲染结果，帧末清空列表，等待下一帧的填充。它渲染的资源被序列化成ScriptableRendererData。

ScriptableRenderer里面最核心的两个方法是Setup(...)和Execute(...)，这两个方法在每一帧里都会被执行。Setup(...)会根据渲染数据，将本帧要执行的Pass加入到ScriptableRenderPass的列表中；Execute(...)从ScriptableRenderPass的列表中将Pass按照渲染时序分类（即RenderPassEvent）取出来，并执行这个过程。

### Setup(...)

该方法在ScriptableRenderer里面是一个虚方法，任何继承于ScriptableRenderer的子渲染器都需要去实现它。实现它的过程也就是将Pass加入队列的过程，由于队列是FIFO的，所以这个入队的过程也就是本帧内渲染的过程。

以下为该方法的主要调用。

```csharp
public override void Setup(ScriptableRenderContext context, ref RenderingData renderingData)
{
	// 1.
	if (cameraData.renderType == CameraRenderType.Base)
	{
		m_ActiveCameraColorAttachment = (createColorTexture) ? m_CameraColorAttachment : RenderTargetHandle.CameraTarget;
		m_ActiveCameraDepthAttachment = (createDepthTexture) ? m_CameraDepthAttachment : RenderTargetHandle.CameraTarget;
		
		...
	}
	else
	{
		m_ActiveCameraColorAttachment = m_CameraColorAttachment;
		m_ActiveCameraDepthAttachment = m_CameraDepthAttachment;
	}
	ConfigureCameraTarget(m_ActiveCameraColorAttachment.Identifier(), m_ActiveCameraDepthAttachment.Identifier());
	
	// 2.
	for (int i = 0; i < rendererFeatures.Count; ++i)
	{
		if(rendererFeatures[i].isActive)
			rendererFeatures[i].AddRenderPasses(this, ref renderingData);
	}
	
	//3.
	
	if (mainLightShadows)
		EnqueuePass(m_MainLightShadowCasterPass);
	
	if (additionalLightShadows)
		EnqueuePass(m_AdditionalLightsShadowCasterPass);
	
	...
	
	EnqueuePass(m_RenderOpaqueForwardPass);
	
	...
	
	// 如果创建了DepthTexture，我们需要复制它，否则我们可以将它渲染到renderbuffer。
	if (!requiresDepthPrepass && renderingData.cameraData.requiresDepthTexture && createDepthTexture)
	{
		m_CopyDepthPass.Setup(m_ActiveCameraDepthAttachment, m_DepthTexture);
		EnqueuePass(m_CopyDepthPass);
	}
	
	if (renderingData.cameraData.requiresOpaqueTexture)
	{
		Downsampling downsamplingMethod = UniversalRenderPipeline.asset.opaqueDownsampling;
		m_CopyColorPass.Setup(m_ActiveCameraColorAttachment.Identifier(), m_OpaqueColor, downsamplingMethod);
		EnqueuePass(m_CopyColorPass);
	}
	
	...
	
	EnqueuePass(m_RenderTransparentForwardPass);
	
	...
	
	// 4.
	if (lastCameraInTheStack)
	{
		// Post-processing将得到最终的渲染目标，不需要final blit pass。
		if (applyPostProcessing)
		{
			m_PostProcessPass.Setup(...);
			EnqueuePass(m_PostProcessPass);
		}
		
		...
		
		// 执行FXAA或任何其他可能需要在AA之后运行的Post-Processing效果。
		if (applyFinalPostProcessing)
		{
			m_FinalPostProcessPass.SetupFinalPass(sourceForFinalPass);
			EnqueuePass(m_FinalPostProcessPass);
		}
		
		...
		
		// 我们需要FinalBlitPass来得到最终的屏幕。
		if (!cameraTargetResolved)
		{
			m_FinalBlitPass.Setup(cameraTargetDescriptor, sourceForFinalPass);
			EnqueuePass(m_FinalBlitPass);
		}
	}
	else if (applyPostProcessing)
	{
		m_PostProcessPass.Setup(...);
		EnqueuePass(m_PostProcessPass);
	}
}
```

1. 如果当前相机是主相机，判断是否需要渲染到DepthTexture，如果需要就设置当前深度缓冲为m_CameraDepthAttachment，否则就渲染到相机默认渲染目标；判断是否需要渲染到ColorTexture，如果需要就设置当前颜色缓冲为m_CameraColorAttachment，否则就渲染到相机默认渲染目标。

>> 需要渲染到ColorTexture的条件包括：打开MSAA、打开RenderScale、打开HDR、打开Post-Processing、打开渲染到OpaqueTexture、添加了自定义ScriptableRendererFeature等。

>> 需要渲染到DepthTexture的条件主要是打开渲染到DepthTexture。

2. 将所有自定义的ScriptableRendererFeature加入到ScriptableRenderPass的队列中。

3. 将各种通用Pass根据各自条件加入到ScriptableRenderPass的队列中。

4. 如果当前相机是本帧最后一个渲染的相机，则将一些需要最后Blit的Pass加入到ScriptableRenderPass的队列中。

### Execute(...)

该方法在ScriptableRenderer里面是一个不用重写的公共方法，由于各个Pass的执行顺序在Setup(...)里已经确定，所以该方法已经没有重写的必要了。不过还是可以看一下Execute(...)里面发生了什么。以下为该方法的主要调用。

```csharp
public void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
{
	...
	
	// 1.
	FillBlockRanges(blockEventLimits, blockRanges);
	
	...
	
	// 2.
	ExecuteBlock(RenderPassBlock.BeforeRendering, blockRanges, context, ref renderingData);
	
	...
	
	// Opaque blocks...
	ExecuteBlock(RenderPassBlock.MainRenderingOpaque, blockRanges, context, ref renderingData, eyeIndex);
	
	// Transparent blocks...
	ExecuteBlock(RenderPassBlock.MainRenderingTransparent, blockRanges, context, ref renderingData, eyeIndex);
	
	// Draw Gizmos...
	DrawGizmos(context, camera, GizmoSubset.PreImageEffects);
	
	// In this block after rendering drawing happens, e.g, post processing, video player capture.
	ExecuteBlock(RenderPassBlock.AfterRendering, blockRanges, context, ref renderingData, eyeIndex);
}
```

1. 每一个ScriptableRenderPass里都有一个RenderPassEvent字段，FillBlockRanges(...)根据这个字段，将ScriptableRenderPass分配到不同的Block里。也就是给Pass根据渲染阶段进行了一下分类，这样开发者可以比较直观地在某一个阶段插入渲染过程。

2. 根据不同的渲染阶段，取出这个阶段所有Pass依次执行其中的渲染过程。

## 总结

以上所述，即为URP的主体代码，详细代码（比如每一个ScriptableRenderPass，不同的渲染器，细节的渲染过程）可以细读com.unity.render-pipelines.universal里面的代码。

本文主旨是记录并描述URP的主体结构，为定制渲染管线、优化性能等需求提供参考。
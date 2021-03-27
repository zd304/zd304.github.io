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

## UniversalRenderPipelineAsset.cs

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
			
			<更新设置anyPostProcessingEnabled和lastActiveOverlayCameraIndex>...</>
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


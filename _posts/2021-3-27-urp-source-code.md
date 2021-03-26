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

管线类UniversalRenderPipeline继承自RenderPipeline，其核心方法为Render()。

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

Render()方法每帧都会被自动调用，在方法中，会处理本帧需要执行的所有渲染命令，来绘制本帧图像。以下为该方法的主要调用。

```csharp
protected override void Render(ScriptableRenderContext renderContext, Camera[] cameras)
{
	BeginFrameRendering(renderContext, cameras);
	
	...
	
	SortCameras(cameras);
	for (int i = 0; i < cameras.Length; ++i)
	{
		var camera = cameras[i];
		if (IsGameCamera(camera))
		{
			RenderCameraStack(renderContext, camera);
		}
		else
		{
			BeginCameraRendering(renderContext, camera);
			
			UpdateVolumeFramework(camera, null);
			
			RenderSingleCamera(renderContext, camera);
			EndCameraRendering(renderContext, camera);
		}
	}
	
	EndFrameRendering(renderContext, cameras);
}
```


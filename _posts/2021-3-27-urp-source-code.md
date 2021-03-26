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

## UniversalRenderPipeline


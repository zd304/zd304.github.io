---
layout: post
title:  Unity ECS 文档 —— IComponentData
categories: Unity
tags: DOTS Unity ECS
author: zack.zhang
---

* content
{:toc}

语法

```csharp
public interface IComponentData
```

<!-- more -->

## 附注

IComponentData的实现必须是一个结构体，并且仅能包含非托管、可位传输的数据类型，包括：

* C# 定义的<a href="https://docs.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types">可位传输的数据类型</a>

* bool

* char

* <a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.BlobAssetReference-1.html">BlobAssetReference\<T\></a>（Blob数据结构的引用）

* <a href="https://docs.unity3d.com/Packages/com.unity.collections@0.11/api/Unity.Collections.FixedString.html">FixedString</a>（固定大小的字符缓冲区）

* Unity.Collections.FixedList

* <a href="https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/fixed-statement">fixed arrays</a>（在unsafe的环境下）

* 包含这些非托管、可位传输的数据的结构体（struct）

注意，你还可以在DynamicBuffer\<T\>中使用单独的IBufferElementData组件作为类数组的数据结构。

单个IComponentData实现应该只包含总是同时会被访问到的数据字段，或几乎总是同时会被访问到的数据字段。通常，使用更多的小组件类型，比使用更少、更大的组件类型更有效。

使用<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html">EntityManager</a>或<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityCommandBuffer.html">EntityCommandBuffer</a>添加、设置和删除实体的组件。（当你持有一个对IComponentData结构的引用时，你也可以正常地更新它的字段）

IComponentData对象存储在chunks(ArchetypeChunk)中，并按实体索引。您可以实现System（ComponentSystemBase）来选择和遍历一组具有特定组件的实体。对于非基于Job的系统，可以使用ComponentSystem的EntityQueryBuilder。对基于Unity.Entities.IJobForEach\`1和IJobChunk的系统，使用JobComponentSystem的EntityQuery。实体的所有组件大小必须适合一个单一的块，因此不能超过16 KB。（一些组件，如DynamicBuffer\<T\>和BlobArray\<T\>可以将数据存储在chunk之外，因此不能算在该限制范围内）

虽然添加到实体中的大多数组件都实现了IComponentData，但ECS还提供了几种特殊的组件类型。这些特殊的类型包括：

* <a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.IBufferElementData.html">IBufferElementData</a>——用于<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.DynamicBuffer-1.html">DynamicBuffer\<T\></a>

* <a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISharedComponentData.html">ISharedComponentData</a>——该组件的值由同一块中的所有实体共享。

* <a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISystemStateComponentData.html">ISystemStateComponentData</a>——存储与实体关联的内部系统状态的组件。

* <a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISystemStateSharedComponentData.html">ISystemStateSharedComponentData</a>——共享组件接口的系统状态版本。

* <a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ISystemStateBufferElementData.html">ISystemStateBufferElementData</a>——缓冲区元素接口的系统状态版本。

注意：块组件可以用来存储与块相关的数据(参见<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_AddChunkComponentData__1_Unity_Entities_Entity_">AddChunkComponentData\<T\>(Entity)</a>)和单例组件（只允许实例化一个该类型的实例，参见<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentSystemBase.html#Unity_Entities_ComponentSystemBase_SetSingleton__1___0_">SetSingleton\<T\>(T)</a>），使用IComponentData接口。

有关更多信息，请参见<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/manual/component_data.html">通用组件</a>。
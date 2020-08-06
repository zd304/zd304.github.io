---
layout: post
title:  Unity ECS 文档 —— 组件
categories: Unity
tags: DOTS Unity ECS
author: zack.zhang
---

* content
{:toc}

<!-- more -->

## 组件

组件是实体组件系统体系结构的三个主要元素之一，它们代表着游戏或应用程序的数据。实体是索引组件集合的标识符，而系统提供行为。

ECS中的组件是一个结构体，它具有以下“标记接口”中的一个:

* <a href="">IComponentData</a>——用于一般用途和\[chunk components\]。

* <a href="">IBufferElementData</a>——将[dynamic buffers]与一个实体关联。

* <a href="">IBufferElementData</a>——根据原型中的值对实体进行分类或分组。有关更多信息，请参见\[Shared Component Data\]。

* <a href="">ISystemStateComponentData</a>——将系统特定状态与实体关联，并检测何时创建或销毁单个实体。有关更多信息，请参见系统状态组件（System State Components）。

* <a href="">ISharedSystemStateComponentData</a>——共享数据和系统状态数据的组合。参见系统状态组件。参见系统状态组件（System State Components）。

* <a href="">Blob assets</a>——虽然从技术上讲不是一个“组件”，但你可以使用Blob assets来存储数据。Blob assets可以由使用<a href="">BBlobAssetReference</a>的一个或多个组件引用，并且是不可变的。您可以使用blob资产在资产之间共享数据，并在c# Job中访问该数据。

EntityManager将组件的独特组合组织为原型。它将具有相同原型的所有实体的组件一起存储在称为块（chunks）的内存块中，给定块中的实体都具有相同的组件原型。

![ArchetypeChunkDiagram](https://zd304.github.io/assets/img/ECS/ArchetypeChunkDiagram.png)<br/>

此图说明了ECS如何按其原型存储组件数据块。共享组件和块组件是例外，因为ECS将它们存储在块之外。这些组件类型的单个实例应用于可应用的块中的所有实体上。此外，您还可以选择在块之外存储动态缓冲区，尽管ECS不将这些类型的组件存储在块中，但在查询实体时，通常可以将它们与其他组件类型按照相同的方式处理。
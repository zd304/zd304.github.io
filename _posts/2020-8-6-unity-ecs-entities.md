---
layout: post
title:  Unity ECS 文档 —— 实体
categories: Unity
tags: DOTS Unity ECS
author: zack.zhang
---

* content
{:toc}

<!-- more -->

## 实体

实体是实体组件系统体系结构的三个主要元素之一，它们代表你的游戏或应用中的每一个“事物”。实体既没有行为也没有数据，相反，它标识哪些数据归在一起。<a href="https://zd304.github.io/2020/08/06/unity-ecs-systems/">系统</a>提供行为，<a href="https://zd304.github.io/2020/08/06/unity-ecs-components/">组件</a>存储数据。

实体本质上是一个ID，最简单的方法是把它看作一个超级轻量级的游戏对象，甚至默认没有名字。实体id稳定，你可以使用它们存储对另一个组件或实体的引用。例如，层次结构（hierarchy）中的子实体可能需要引用其父实体。

EntityManager管理世界中的所有实体。EntityManager维护实体列表，并组织与实体关联的数据，以获得最佳性能。

尽管实体没有类型，但实体组可以按照与其关联的数据组件的类型进行分类。当你创建实体并向其添加组件时，EntityManager会跟踪现有实体上的组件的惟一组合，这种独特的组合称为原型（Archetype）。往实体上添加组件时，EntityManager会创建一个<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityArchetype.html">EntityArchetype</a>。你可以使用现有的EntityArchetype来创建符合该原型的新实体，也可以预先创建EntityArchetype并使用它创建实体。

## 创建实体

创建实体最简单的方法是在Unity编辑器中，你可以设置ECS在运行时将放置在场景中的GameObjects和Prefabs转换为实体。对于游戏或应用程序的更动态的部分，你可以在Job中创建多个实体的生成系统。最后，您可以使用其中一个<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_CreateEntity">EntityManager.CreateEntity</a>函数一次创建实体。

## 使用EntityManager创建实体

使用<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_CreateEntity">EntityManager.CreateEntity</a>函数来创建实体。ECS在与EntityManager相同的World中创建实体。

你可以通过以下方式一个一个地创建实体：

* 用组件来创建实体，这些组件使用了<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.ComponentType.html">ComponentType</a>对象数组

* 用组件来创建实体，这些组件使用了<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityArchetype.html">EntityArchetype</a>

* 使用<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_Instantiate_Unity_Entities_Entity_">Instantiate</a>复制现有实体，包括其当前数据

* 创建一个没有组件的实体，然后向其添加组件。(你可以立即添加组件，或者在需要添加组件时添加组件)

你也可以在同一时间创建多个实体：

* 使用<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_CreateEntity">CreateEntity</a>，用具有相同原型的新实体填充NativeArray

* 使用<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.EntityManager.html#Unity_Entities_EntityManager_Instantiate_Unity_Entities_Entity_">Instantiate</a>，使用现有实体的副本（包括其当前数据）填充NativeArray。

## 添加和删除组件

创建实体后，你可以添加或删除组件。当你这样做时，受影响实体的原型将发生更改，并且EntityManager必须将更改的数据移动到新的内存块（chunks），并压缩原始块中的组件数组。

导致结构更改的那些对实体进行的更改操作——即添加或删除更改SharedComponentData值的组件，以及破坏实体——不能在Job内部进行，因为这些操作可能会使Job正在处理的数据无效。相反，你可以添加命令来对EntityCommandBuffer进行这些类型的更改，并在Job完成后执行此命令缓冲区。

EntityManager提供了从<u>单个实体</u>以及<u>NativeArray中的所有实体</u>中删除组件的功能。

## 遍历实体

遍历具有匹配组件集的所有实体是ECS体系结构的核心，参见<a href="">访问实体数据</a>。
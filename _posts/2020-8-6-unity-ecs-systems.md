---
layout: post
title:  Unity ECS 文档 —— 系统
categories: Unity
tags: DOTS Unity ECS
author: zack.zhang
---

* content
{:toc}

<!-- more -->

## 系统

系统(ECS中的S)提供了将组件数据从当前状态转换为下一个状态的逻辑——例如，系统可能通过速度乘以上一次更新后的时间间隔更新所有移动实体的位置。

BasicSystem.png

![BasicSystem](https://zd304.github.io/assets/img/ECS/BasicSystem.png)<br/>

## 实例化系统

Unity ECS自动发现项目中的系统类，并在运行时实例化它们。它将每个发现的系统添加到一个默认系统组中，你可以使用<a href="">系统属性（system attributes）</a>来指定系统的父组，以及该系统在组中的顺序。如果你没有指定父级，Unity将System以确定但未指定的顺序，添加到默认World的仿真系统组。你还可以使用属性来禁用自动创建。

系统的update循环是由它的父<a href="">组件系统组（ComponentSystemGroup）</a>驱动的。ComponentSystemGroup本身就是一种专门的系统，它负责更新其子系统，Groups可以嵌套。系统从运行的<a href="">World</a>中获得时间数据，时间由<a href="https://docs.unity3d.com/Packages/com.unity.entities@0.13/api/Unity.Entities.UpdateWorldTimeSystem.html">UpdateWorldTimeSystem</a>更新。

您可以使用实体调试器窗口查看系统配置(菜单:**Window** > **Analysis** > **Entity Debugger**)。

## 系统类型

Unity ECS提供了几种类型的系统。通常，你为实现游戏行为和数据转换而编写的系统会扩展<a href="">SystemBase</a>。其他系统类有专门的用途。通常使用<a href="">EntityCommandBufferSystem</a>和<a href="">ComponentSystemGroup</a>类的现有实例。

* <a href="">SystemBase</a>——创建系统时要实现的基类。

* <a href="">EntityCommandBufferSystem</a>——为其他系统提供EntityCommandBuffer实例。每个默认系统组在其子系统列表的开头和结尾维护一个**实体命令缓冲区系统**。这允许您对结构更改进行分组，以便它们在一个框架中导致更少的同步点。

* <a href="">ComponentSystemGroup</a>——为其他系统提供嵌套的组织和更新顺序。默认情况下，Unity ECS会创建几个**组件系统组**。

* GameObjectConversionSystem——将基于GameObject的编辑器内表示转换为高效的、基于实体的运行时表示。游戏转换系统运行在Unity编辑器环境下。

**重要提示**：ComponentSystem和JobComponentSystem类，以及IJobForEach，正在逐步退出DOTS API，但还没有正式被弃用。使用SystemBase和Entities.ForEach代替。
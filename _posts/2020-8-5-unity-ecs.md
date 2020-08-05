---
layout: post
title:  Unity ECS 文档 —— 概述
categories: Unity
tags: DOTS Unity ECS
author: zack.zhang
---

* content
{:toc}

<!-- more -->

## Entity Component System

实体组件系统(ECS)是Unity面向数据技术栈的核心。顾名思义，ECS有三个主要部分:

* Entities — 填充你的游戏程序的实体或者事物。

* Components — 与实体相关联，按数据本身组织的数据，而不是按实体组织的（这种组织上的差异是面向对象和面向数据设计之间的关键区别之一）。

* Systems — 将组件数据从当前状态转换为下一个状态的逻辑

> 例如，系统可能会根据所有移动实体的速度，乘以从上一帧开始的时间间隔，来更新它们的位置。
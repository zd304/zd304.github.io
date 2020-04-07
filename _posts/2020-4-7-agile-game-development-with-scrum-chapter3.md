---
layout: post
title:  "《Scrum敏捷游戏开发》读书笔记-第三章"
categories: 管理
tags: Scrum 敏捷开发 游戏开发
author: zack.zhang
---

* content
{:toc}

<!-- more -->

## 一、概述

### 1. 宏观大局

![2.1](https://zd304.github.io/assets/img/scrum-3.1.png)<br/>

PBI分解为任务实例

<div class="mermaid">
graph LR
玩家跳跃功能
动作：做动画【10h】
程序：跳跃机制【5h】
动作：调整动作【6h】
玩家跳跃功能-->动作：做动画【10h】
玩家跳跃功能-->程序：跳跃机制【5h】
玩家跳跃功能-->动作：调整动作【6h】
</div>

### 2. Scrum原则

* 经验论
> “检验和适应”及时响应变化
* 浮现论
> 只是是开发过程中浮现出来的
> 没有人一开始就知道所有知识
* 限时
> 定期交付价值
* 优先级
> “实现设计中的所有需求”×
> 根据 <u>游戏特征</u> 之于 <u>玩家价值</u> 来判断
* 自组织
> 授权 → 小型团队 → 管理时间盒 → Sprint回顾

## 二、Scrum组成

### 1. 产品Backlog
		故事池
**每个Sprint后**
	* 没预期到的PBI → 添加
	* 不需要的PBI → 删除
	* 优先级调整
**Sprint前**
	* 制定优先级
	* 筛选评估PBI
	* <u>分解成</u> 一个Sprint <u>能完成</u> 的 <u>特性</u>
	

---
layout: post
title:  Unity多开工具
categories: JavaScript
tags: Unity 多开 Windows
author: HyG
---

* content
{:toc}

当我们在使用Unity进行开发的时候，尤其是网络游戏开发，很多时候需要启动多个Unity客户端进程。当然有一种最粗暴的办法是将这个工程复制多个副本，分别打开。这里介绍Windows下另一种巧妙的办法。

<!-- more -->

## 多开思路

windows下有一个传感符号链接的工具，使用方式：MKLINK \[\[/D\] \| \[/H\] \| \[/J\]\] &lt;链接名称&gt; &lt;目标&gt;

参数

<table>
  <thead>
    <tr>
      <th>参数</th>
      <th>描述</th>
    </tr>
  </thead>
  <tbody>
	<tr>
      <td>&#47;D</td>
      <td>创建目录符号链接。默认情况下，mklink会创建文件符号链接。</td>
    </tr>
	<tr>
      <td>&#47;H</td>
      <td>创建硬链接而不是符号链接。</td>
    </tr>
	<tr>
      <td>&#47;J</td>
      <td>创建目录连接。</td>
    </tr>
	<tr>
      <td>&lt;链接名称&gt</td>
      <td>指定正在创建的符号链接的名称。</td>
    </tr>
	<tr>
      <td>&lt;目标&gt;</td>
      <td>指定新符号链接引用的路径（相对或绝对）。</td>
    </tr>
	<tr>
      <td>&#47;?</td>
      <td>在命令提示符下显示帮助。</td>
    </tr>
  </tbody>
</table>

鉴于此工具，我们可以用命令行创建一个当前工程的链接。例如当前工程路径为E:\MHCT\Client\MProject，我们可以创建一个空文件夹E:\MHCT_MKLINK，最后输入命令：

```c
mklink /D E:\ProjectD\Client E:\ProjectD_MKLINK\Client
```

即可。

但是有的时候并不需要把整个Unity工程文件夹链接过去，通常只需要链接Asstes、Packages和ProjectSettings三个文件夹即可。如果觉得每次都要输入三次bat指令很麻烦，可以自己写一个小工具来一键处理。

这里附上一个自己开发的<a href="https://github.com/zd304/UnityMultiOpen">小工具</a>

## 总结

该文章解决了以下问题：

* Unity客户端多开问题。
* 一键多开工具，使团队中其他不会写代码的同事也可以多开。

后续如果有时间可以把小工具继续优化，比如加一个交互性更强的GUI，降低用户使用的学习成本。
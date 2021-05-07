---
layout: post
title:  "Unity移动游戏纹理压缩格式优化"
categories: Unity
tags: 游戏 Unity 优化 贴图
author: zack.zhang
---

* content
{:toc}

移动游戏开发中，性能优化是一个很重要的环节，没有人能在设计之初就想到所有可能发生的事情，并在开发过程中完全避免浪费，这就需要在每一个开发的跌代里去对游戏性能做优化。游戏性能优化通常会针对CPU、渲染、内存去做优化，其中内存优化主要是为了保证游戏运行的稳定性，让更多设备能够运行游戏。内存优化主要是对游戏资源进行优化，这里我们主要讨论对贴图压缩格式的优化，这一点对于2D游戏和UI系统的内存占用至关重要。

<!-- more -->

## 图片文件格式和纹理格式的区别

常用的图片文件格式有BMP，TGA，JPG，GIF，PNG等；

常用的纹理格式有RGB565，ARGB4444，ARGB1555，RGB24，ARGB32等。

图片文件格式是图片为了储存信息而使用的，不同的图片文件格式有不同的编码方式。但是图片文件格式是不能直接被GPU识别的，也就是说如果读取的图片是以图片文件格式来进行编码的，那么就需要将这些图片解码成GPU识别的格式，再传送到GPU。游戏引擎通常直接使用能被GPU识别的格式来保存图片。

例如Unity中使用图片时，无论导入的是什么样的图片，最终都会根据平台的不同转换成不同的纹理格式。

![unity_compressd_list](https://zd304.github.io/assets/img/TexFormat/unity_compressd_list.png)<br/>

## 像素深度

像素深度对应的英文名为bits per pixel，简称bpp。从英文名上可以更直观地了解到，像素深度其实描述的是每个像素在内存中占用多少字节。例如ARGB32的像素深度就是32bpp，因为每个像素为R、G、B、A分别分配了8bit的空间，所以像素深度 = 8 + 8 + 8 + 8 = 32。

## 常用纹理压缩格式

对于ARGB32的纹理来说，一张1024\*1024大小的贴图，所占的内存空间为1024\*1024\*32/8 = 4,194,304 ≈ 4MB。游戏中会大量用到这么大尺寸的贴图，甚至还有很多更大的贴图，如此大量的贴图，如果都用ARGB32显然是不明智的。

那么有没有什么格式是像png、jpg这种经过压缩而又能最大程度保持原图效果的格式呢？答案是有的。常用的纹理压缩格式有ETC1、ETC2、PVRTC、ASTC等。

## 纹理压缩格式对比

### ETC1

像素深度：4bpp

图像质量：★★★

支持通道：RGB

适配性：支持OpenGL ES 2.0的设备

GL扩展名：GL_OES_compressed_ETC1_RGB8_texture

纹理尺寸：按4×4的块Block来划分，每个Block占64bit，所以纹理长度和宽度都必须为4的倍数

### ETC2-RGB

像素深度：4bpp

图像质量：★★★

支持通道：RGB

适配性：支持OpenGL ES 3.0的设备

GL扩展名：GL_compressed_RGB8_ETC2

纹理尺寸：按4×4的块Block来划分，每个Block占64bit，所以纹理长度和宽度都必须为4的倍数

### ETC2-RGBA

像素深度：8bpp

图像质量：★★★

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备

GL扩展名：GL_compressed_RGBA8_ETC2_EAC

纹理尺寸：按4×4的块Block来划分，每个Block占128bit（包含alpha通道64bit），所以纹理长度和宽度都必须为4的倍数

### PVRTC2 2-bpp

像素深度：2bpp

图像质量：★★

支持通道：RGB/RGBA

适配性：PowerVR架构

GL扩展名：GL_IMG_texture_compression_pvrtc

纹理尺寸：任何尺寸纹理

### PVRTC2 4-bpp

像素深度：4bpp

图像质量：★★★

支持通道：RGB/RGBA

适配性：PowerVR架构

GL扩展名：GL_IMG_texture_compression_pvrtc

纹理尺寸：任何尺寸纹理

### ASTC 4x4

像素深度：8bpp

图像质量：★★★★★

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备，不支持设备包括Qualcomm Adreno4xx / Snapdragon 415 (2015) 以下、 ARM Mali T624 (2012)以下、NVIDIA Tegra K1 (2014) 以下、PowerVR GX6250 (2014)以下

GL扩展名：GL_KHR_texture_compression_astc_ldr

纹理尺寸：任何尺寸纹理

### ASTC 5x4

像素深度：6.4bpp

图像质量：★★★★☆

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备，不支持设备包括Qualcomm Adreno4xx / Snapdragon 415 (2015) 以下、 ARM Mali T624 (2012)以下、NVIDIA Tegra K1 (2014) 以下、PowerVR GX6250 (2014)以下

GL扩展名：GL_KHR_texture_compression_astc_ldr

纹理尺寸：任何尺寸纹理

### ASTC 5x5

像素深度：5.12bpp

图像质量：★★★★

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备上，不支持设备包括Qualcomm Adreno4xx / Snapdragon 415 (2015) 以下、 ARM Mali T624 (2012)以下、NVIDIA Tegra K1 (2014) 以下、PowerVR GX6250 (2014)以下

GL扩展名：GL_KHR_texture_compression_astc_ldr

纹理尺寸：任何尺寸纹理

### ASTC 6x5

像素深度：4.27bpp

图像质量：★★★☆

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备上，不支持设备包括Qualcomm Adreno4xx / Snapdragon 415 (2015) 以下、 ARM Mali T624 (2012)以下、NVIDIA Tegra K1 (2014) 以下、PowerVR GX6250 (2014)以下

GL扩展名：GL_KHR_texture_compression_astc_ldr

纹理尺寸：任何尺寸纹理

### ASTC 6x6

像素深度：3.56bpp

图像质量：★★★

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备上，不支持设备包括Qualcomm Adreno4xx / Snapdragon 415 (2015) 以下、 ARM Mali T624 (2012)以下、NVIDIA Tegra K1 (2014) 以下、PowerVR GX6250 (2014)以下

GL扩展名：GL_KHR_texture_compression_astc_ldr

纹理尺寸：任何尺寸纹理

### ASTC 8x5

像素深度：3.2bpp

图像质量：★★☆

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备上，不支持设备包括Qualcomm Adreno4xx / Snapdragon 415 (2015) 以下、 ARM Mali T624 (2012)以下、NVIDIA Tegra K1 (2014) 以下、PowerVR GX6250 (2014)以下

GL扩展名：GL_KHR_texture_compression_astc_ldr

纹理尺寸：任何尺寸纹理

### ASTC 8x8

像素深度：2bpp

图像质量：★★☆

支持通道：RGBA

适配性：支持OpenGL ES 3.0的设备上，不支持设备包括Qualcomm Adreno4xx / Snapdragon 415 (2015) 以下、 ARM Mali T624 (2012)以下、NVIDIA Tegra K1 (2014) 以下、PowerVR GX6250 (2014)以下

GL扩展名：GL_KHR_texture_compression_astc_ldr

纹理尺寸：任何尺寸纹理
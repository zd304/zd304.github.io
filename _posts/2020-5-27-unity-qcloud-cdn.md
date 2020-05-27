---
layout: post
title:  Unity接入腾讯云CDN
categories: Unity小工具
tags: 腾讯云 CDN SDK
author: zack.zhang
---

* content
{:toc}

## 背景

2019年的GDC游戏开发者大会上，Unity CEO约翰·里奇蒂洛，其公司将于今年晚些时候与腾讯进行合作，新的合作将会为腾讯旗下的“腾讯云”提供支持。详情见<a href="https://cloud.tencent.com/solution/ucg">腾讯云的Unity游戏服务</a>官网。

<!-- more -->

![ugc-assetstreaming](https://zd304.github.io/assets/img/qcloud/ugc-assetstreaming.png)<br/>

在此大背景下，为何还会手动接入腾讯云呢？

* 看样子目前只是一个方向，还没真正开始合作？

![this-link-is-no-longer-valid](https://zd304.github.io/assets/img/qcloud/this-link-is-no-longer-valid.png)<br/>

目前找到了这么一篇<a href="https://unity-1258948065.cos.ap-shanghai.myqcloud.com/AssetStreaming/CloudContentDeliveryUserManual.pdf">文档</a>

* 还有很多低版本的用户在使用Unity（比如我还在使用Unity2018.4.21f1）

## 使用

搭建CDN的文章网上比比皆是，就不赘述。

本文重点在于记录Unity如何接入腾讯云CDN的过程。

首先有两个重要的网站需要记住：<a href="https://cloud.tencent.com/document/product/436/11366">COSBrowser工具</a>和<a href="https://cloud.tencent.com/document/product/436/7751">API使用文档</a>。

接下来，通常我们需要以下信息去操作CDN：

* SecretID和SecretKey，用于识别开发者身份的ID和秘钥

* Bucket，存储桶，COS中用于存储数据的容器，比如一个项目分一个桶

* Region，地域信息，枚举值可参见可用地域文档 https://cloud.tencent.com/document/product/436/6224

* APPID，开发者访问COS服务时拥有的用户维度唯一资源标识，用以标识资源

有了以上信息初步就足够了。

如下图，我们可以使用以上信息，去登录COSBrowser。

COSBrowser是腾讯云对象存储COS推出的可视化界面工具，让用户可以使用更简单的UI轻松实现对COS资源的查看、传输和管理。

![COSBrowser](https://zd304.github.io/assets/img/qcloud/COSBrowser.png)<br/>

COSBrowser的操作很傻瓜，很像百度网盘之类的东西，就不再赘述。

接下来，Unity如何去对接SDK，就需要看<a href="https://cloud.tencent.com/document/product/436/32819">.NET SDK文档</a>。

下载文档里的<a href="https://cos-sdk-archive-1253960454.file.myqcloud.com/qcloud-sdk-dotnet/latest/qcloud-sdk-dotnet.zip?_ga=1.216336144.766970255.1590125456">SDK</a>，我下载的版本是5.4.9。

下载下来后，会发现有一个C#工程，例如我本地的工程文件地址为 D:/QCloudSDK/qcloud-sdk-dotnet-5.4.9/QCloudCSharpSDK/QCloudCSharpSDK.sln。

接下来的用法，如果是Unity的老用户就很熟悉了，要么把C#代码直接放到Unity工程里，要么把工程编译成DLL放到工程Plugins下即可。我选择了编译DLL。

最后，根据.NET SDK文档的介绍来敲代码吧。

```cs
using COSXML;
using COSXML.Auth;
using COSXML.Model.Object;
using COSXML.Model.Bucket;
using COSXML.CosException;
using COSXML.Model.Service;
using COSXML.Utils;
using System.Collections.Generic;
using COSXML.Model.Tag;
using UnityEngine;
using System.Text;

public class TestQCloud
{
    CosXmlConfig config = null;

    public TestQCloud()
    {
        //初始化 CosXmlConfig 
        string appid = "1251000000";//设置腾讯云账户的账户标识 APPID
        string region = "ap-hongkong"; //设置一个默认的存储桶地域
        config = new CosXmlConfig.Builder()
          .SetConnectionTimeoutMs(60000)  //设置连接超时时间，单位毫秒，默认45000ms
          .SetReadWriteTimeoutMs(40000)  //设置读写超时时间，单位毫秒，默认45000ms
          .IsHttps(true)  //设置默认 HTTPS 请求
          .SetAppid(appid)  //设置腾讯云账户的账户标识 APPID
          .SetRegion(region)  //设置一个默认的存储桶地域
          .SetDebugLog(true)  //显示日志
          .Build();  //创建 CosXmlConfig 对象
    }

    public void Test()
    {
        string secretId = "AKIDMTP********************"; //"云 API 密钥 SecretId";
        string secretKey = "HJ4bR1*****************"; //"云 API 密钥 SecretKey";
        long durationSecond = 600;  //每次请求签名有效时长，单位为秒

        QCloudCredentialProvider qCloudCredentialProvider = new DefaultQCloudCredentialProvider(secretId,
          secretKey, durationSecond);

        //初始化 CosXmlServer
        CosXmlServer cosXml = new CosXmlServer(config, qCloudCredentialProvider);

        try
        {
            {
                string bucket = "example-1251000000"; //格式：BucketName-APPID
                HeadBucketRequest request = new HeadBucketRequest(bucket);
                //设置签名有效时长
                request.SetSign(TimeUtils.GetCurrentTime(TimeUnit.SECONDS), 600);
                //执行请求
                HeadBucketResult result = cosXml.HeadBucket(request);
                //请求成功
                Debug.Log("++++++++++++ " + result.GetResultInfo());
            }
            {
                string bucket = "nwrz-1251000000"; //存储桶，格式：BucketName-APPID
                string key = "test/logo.jpg"; //对象在存储桶中的位置，即称对象键
                string localDir = Application.persistentDataPath + "/Archive";//本地文件夹
                Debug.Log("************ dir: " + localDir);
                string localFileName = "logo.jpg"; //指定本地保存的文件名
                GetObjectRequest request = new GetObjectRequest(bucket, key, localDir, localFileName);
                //设置签名有效时长
                request.SetSign(TimeUtils.GetCurrentTime(TimeUnit.SECONDS), 600);
                //设置进度回调
                request.SetCosProgressCallback(delegate (long completed, long total)
                {
                    Debug.LogFormat("**************** progress = {0:##.##}%", completed * 100.0 / total);
                });
                //执行请求
                GetObjectResult result = cosXml.GetObject(request);
                //请求成功
                Debug.Log("**************** " + result.GetResultInfo());
            }
        }
        catch (COSXML.CosException.CosClientException clientEx)
        {
            //请求失败
            Debug.LogError("CosClientException: " + clientEx);
        }
        catch (COSXML.CosException.CosServerException serverEx)
        {
            //请求失败
            Debug.LogError("CosServerException: " + serverEx.GetInfo());
        }
    }
}
```
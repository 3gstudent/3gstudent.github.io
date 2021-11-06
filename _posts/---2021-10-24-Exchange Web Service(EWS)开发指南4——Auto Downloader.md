---
layout: post
title: Exchange Web Service(EWS)开发指南4——Auto Downloader
---


## 0x00 前言
---

我在之前的文章[《Exchange Web Service(EWS)开发指南》](https://3gstudent.github.io/Exchange-Web-Service(EWS)%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97)和[《Exchange Web Service(EWS)开发指南2——SOAP XML message》](https://3gstudent.github.io/Exchange-Web-Service(EWS)%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%972-SOAP-XML-message)详细介绍了通过SOAP XML message实现利用hash对Exchange资源的访问。

因为是较为底层的通信协议，在功能实现上相对繁琐，例如下载邮件附件的操作，需要依次完成以下操作：

- 读取文件夹信息，获得邮件对应的ItemId和ChangeKey
- 读取邮件信息，获得附件的ItemId
- 通过附件的ItemId获得每个附件对应的AttachmentId
- 通过AttachmentId下载邮件内容，将内容作Base64解码得到实际内容

如果想要完全自动化实现下载邮件和提取附件，原有的[ewsManage.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage.py)需要作一些改动

因此，本文将要介绍自动化下载邮件和提取附件的实现细节，开源代码ewsManage_Downloader。

## 0x01 简介
---

本文将要介绍以下内容：

- 设计思路
- 开发细节
- 开源代码

## 0x02 设计思路
---

ewsManage_Downloader需要满足以下功能：

- 支持明文和NTLM Hash的登录
- 支持关键词搜索
- 支持日期搜索
- 下载时可以指定下载数量
- 能够自动下载邮件和提取附件

程序在通信过程中，每次SOAP XML message请求都需要完整的NTLM验证，无法借助Session机制简化登录流程

因此，原有的[ewsManage.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage.py)代码结构需要重新设计，减少代码冗余。

## 0x03 开发细节
---

### 1.修复登录用户Domain参数的bug

原有的[ewsManage.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage.py)，需要指定登录用户的Domain作为参数

例如登录用户为`test.com\administrator`，Domain参数需要设置为`test.com`

但是，如果登录用户为`administrator`，那么无法指定Domain参数

这是我之前在使用NTLM认证时没有考虑到的一个地方，解决方法如下：

添加参数判断，如果Domain参数为`NULL`，那么在NTML认证时不指定Domian参数

代码示例：

```
    if domain == "NULL":
        ntlm_nego = ntlm.getNTLMSSPType1(host)
    else:    
        ntlm_nego = ntlm.getNTLMSSPType1(host, domain)
```


### 2.支持关键词搜索

发送的SOAP格式：

```
?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:m="http://schemas.microsoft.com/exchange/services/2006/messages" xmlns:t="http://schemas.microsoft.com/exchange/services/2006/types" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <t:RequestServerVersion Version="Exchange2013_SP1" />
  </soap:Header>
  <soap:Body>
    <m:FindItem Traversal='Shallow'>
      <m:ItemShape>
        <t:BaseShape>AllProperties</t:BaseShape>
      </m:ItemShape>
      <m:ParentFolderIds>
        <t:DistinguishedFolderId Id='{folderpath}'>
        </t:DistinguishedFolderId>
      </m:ParentFolderIds>
      <m:QueryString>{querystring}</m:QueryString>
    </m:FindItem>
  </soap:Body>
</soap:Envelope>
```

其中`{querystring}`为搜索的关键词，返回结果为带有指定关键词邮件对应的ItemId和ChangeKey


### 3.支持日期搜索

发送的SOAP格式同上，区别是`{querystring}`不同

参考资料1：

https://docs.microsoft.com/en-us/windows/win32/lwef/-search-2x-wds-aqsreference?redirectedfrom=MSDN

参考资料1中的时间格式为`MM/DD/YY`，示例如下：

2021年3月1日对应的格式为`03/01/21`

但是经过实际测试，`{querystring}`中关于时间的语法不能按照参考资料1的格式

参考资料2：

https://support.microsoft.com/en-us/office/learn-to-narrow-your-search-criteria-for-better-searches-in-outlook-d824d1e9-a255-4c8a-8553-276fb895a8da?ocmsassetid=ha010238831&correlationid=bf4cdcf9-abb8-4d43-930e-d0909de76728&ui=en-us&rs=en-us&ad=us

参考资料2中的时间格式为`Year/Month/Day`，示例如下：

2021年3月1日对应的格式为`2021/3/1`

在时间格式上面，正确的语法为参考资料2

综上，筛选出发送时间为2021年1月1日至2021年12月30日的参数如下：

```
sent:>=2021/1/1 AND sent:<=2021/12/30
```

筛选出接收时间为2021年1月1日至2021年12月30日的参数如下：

```
received:>=2021/1/1 AND received:<=2021/12/30
```

### 4.支持长度搜索

正常情况下，筛选出长度小于2000的参数为：`size:<2000`

但是需要考虑XML格式转义，实际的参数内容为：`size:&lt;2000`

## 0x04 开源代码
---

完整代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_Downloader.py

支持明文和NTLM Hash的登录，代码支持以下功能：

- download，下载邮件并提取附件，可指定邮箱文件夹和下载数量
- findallpeople，导出联系人列表
- search，邮件搜索并下载，支持关键词、时间、长度等语法

在下载邮件时，以邮件用户名作为父文件夹，不同操作会创建不同的子文件夹，在使用搜索功能创建子文件夹时，为了避免特殊字符(例如字符`>`)无法作为文件夹名称，这里会去掉特殊字符


## 0x05 小结
---

本文介绍了自动化下载邮件和提取附件的开发细节，开源代码[ewsManage_Downloader.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_Downloader.py)，实现了利用hash对Exchange资源的访问。

由于采用了较为底层的通信协议，在功能实现上相对繁琐，但是有助于理解通信协议原理和漏洞利用。



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



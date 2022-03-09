---
layout: post
title: 渗透基础——在Win7下运行csvde
---


## 0x00 前言
---

在之前的文章[《渗透基础——活动目录信息的获取2:Bypass AV》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-%E6%B4%BB%E5%8A%A8%E7%9B%AE%E5%BD%95%E4%BF%A1%E6%81%AF%E7%9A%84%E8%8E%B7%E5%8F%962_Bypass-AV)介绍了使用csvde获取活动目录信息的方法，优点是Windows Server系统自带，能够导出csv格式便于查看。但是在Win7系统下，默认不支持这个命令。

本文将要介绍在Win7下运行csvde的方法，提高适用范围。

## 0x01 简介
---

本文将要介绍以下内容：

- 背景知识
- 移植思路
- 实现方法

## 0x02 背景知识
---

参考资料：

https://docs.microsoft.com/en-us/previous-versions/orphan-topics/ws.10/cc772704(v=ws.10)?redirectedfrom=MSDN

https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc732101(v=ws.11)

### 1.csvde的依赖库

需要理清以下结构：

- Windows Server 2003，默认支持csvde
- Windows Server 2008及更高版本，需要开启Active Directory Domain Services (AD DS)或Active Directory Lightweight Directory Services (AD LDS)服务器角色
- Windows XP Professional，需要安装Active Directory Application Mode (ADAM)
- Windows 7及更高版本，需要安装Remote Server Administration Tools (RSAT)

### 2.安装Remote Server Administration Tools (RSAT)

Remote Server Administration Tools for Windows 7：微软已经不再提供下载

Remote Server Administration Tools for Windows 8下载地址：https://www.microsoft.com/en-us/download/details.aspx?id=28972

Remote Server Administration Tools for Windows 10下载地址：https://www.microsoft.com/en-us/download/details.aspx?id=45520

### 3.Win7安装Remote Server Administration Tools (RSAT)

#### (1) 下载安装KB958830

微软已经不再提供手动下载，可选择安装Win7自动更新补丁

#### (2) 安装功能

打开`控制面板`，选择`Turn Windows features on or off`

在`Windows Features`界面中能够找到`Remote Server Administration Tools`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-3-8/2-1.png)

为了能够支持csvde，需要安装`AD DS Snap-ins and Command-line Tools`，路径如下：
`Remote Server Administration Tools` -> `Role Administration Tools` -> `AD DS and AD LDS Tools` -> `AD DS Tools` -> `AD DS Snap-ins and Command-line Tools`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-3-8/2-2.png)

安装成功后，当前Win7系统支持csvde命令

## 0x03 移植思路
---

csvde默认安装路径为`c:\windows\system32`，可以使用Process Monitor监控csvde的启动过程，定位csvde需要的依赖文件，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-3-8/3-1.png)

从图中可以看出，csvde启动时需要依赖文件`C:\Windows\System32\en-US\csvde.exe.mui`

经过一段时间的测试，最终得出以下移植思路：

- 复制文件`C:\Windows\System32\csvde.exe`
- 复制文件`C:\Windows\System32\en-US\csvde.exe.mui`

## 0x04 实现方法
---

我们知道，在`C:\Windows\System32\`下创建文件需要管理员权限，为了能够在普通用户权限下进行移植，这里可以采取相对路径的方法实现：

- 复制csvde.exe至任意普通用户权限可访问的路径
- 在同级目录创建文件夹en-US，复制csvde.exe.mui

为了便于测试，我将自己测试系统的csvde已上传至github，地址如下：

https://github.com/3gstudent/test/blob/master/csvde.zip

## 0x05 小结
---

本文介绍了在Win7下运行csvde的方法，提高适用范围，可按照同样的方法，分别实现在Win8和Win10下运行。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




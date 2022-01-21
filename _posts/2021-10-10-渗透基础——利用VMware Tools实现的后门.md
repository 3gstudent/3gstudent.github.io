---
layout: post
title: 渗透基础——利用VMware Tools实现的后门
---


## 0x00 前言
---

在渗透测试中，我们经常会碰到Windows虚拟机，这些虚拟机往往会安装VMware Tools，利用VMware Tools的脚本执行功能可以实现一个开机自启动的后门。

关于这项技术的文章：

https://bohops.com/2021/10/08/analyzing-and-detecting-a-vmtools-persistence-technique/

https://www.hexacorn.com/blog/2017/01/14/beyond-good-ol-run-key-part-53/


本文将要在参考资料的基础上，分析利用思路，给出防御建议。


## 0x01 简介
---

本文将要介绍以下内容：

- 利用思路
- 利用分析
- 防御建议

## 0x02 利用思路
---

VMware Tools的脚本执行功能支持在以下四种状态时运行：

- power，开机状态
- resume，恢复状态
- suspend，挂起状态
- shutdown，关机状态

可以选择以下两种方法进行配置脚本执行的功能：

### 1.使用VMwareToolboxCmd.exe

默认安装路径:`"C:\Program Files\VMware\VMware Tools\VMwareToolboxCmd.exe"`

命令示例1：

```
"C:\Program Files\VMware\VMware Tools\VMwareToolboxCmd.exe" script power enable
```

命令执行后，将在默认安装路径下创建文件`C:\ProgramData\VMware\VMware Tools\tools.conf`，内容为：

```
[powerops]
poweron-script=poweron-vm-default.bat
```

实现效果：

当系统开机时，将会以System权限执行`"C:\Program Files\VMware\VMware Tools\poweron-vm-default.bat"`

**注：**

对于power命令，只能是开机操作，重启操作无法触发

命令示例2：

```
"C:\Program Files\VMware\VMware Tools\VMwareToolboxCmd.exe" script suspend set "c:\test\1.bat"
```

命令执行后，将在默认安装路径下创建文件`C:\ProgramData\VMware\VMware Tools\tools.conf`，内容为：

```
[powerops]
suspend-script=c:\\test\\1.bat
```

实现效果：

当系统进入挂起状态时，将会以System权限执行`"c:\test\1.bat"`

### 2.使用tools.conf

直接创建文件`C:\ProgramData\VMware\VMware Tools\tools.conf`

文件内容示例：

```
[powerops]
poweron-script=poweron-vm-default.bat
suspend-script=c:\\test\\1.bat
```

实现效果：

当系统开机时，将会以System权限执行`"C:\Program Files\VMware\VMware Tools\poweron-vm-default.bat"`，当系统进入挂起状态时，将会以System权限执行`"c:\test\1.bat"`

### 补充：

查看VMwareToolboxCmd.exe的帮助说明：

```
"C:\Program Files\VMware\VMware Tools\VMwareToolboxCmd.exe" help
```

查看开机启动脚本的默认路径：

```
"C:\Program Files\VMware\VMware Tools\VMwareToolboxCmd.exe" script power default
```

查看开机启动脚本的当前路径：

```
"C:\Program Files\VMware\VMware Tools\VMwareToolboxCmd.exe" script power current
```

## 0x03 利用分析
---

创建文件`C:\ProgramData\VMware\VMware Tools\tools.conf`需要管理员权限

通过VMware Tools的脚本执行功能，启动脚本的执行权限为System

为了提高隐蔽性，可以设置默认启动脚本为`poweron-vm-default.bat`，在`poweron-vm-default.bat`添加通过rundll32加载dll的命令


## 0x04 防御检测
---

默认配置下，VMware Tools不会开启脚本执行功能，也就是说不存在文件`C:\ProgramData\VMware\VMware Tools\tools.conf`

### 1.识别脚本执行功能是否开启

查看文件`C:\ProgramData\VMware\VMware Tools\tools.conf`的内容

如果文件不存在，代表脚本执行功能未开启

### 2.识别脚本执行的内容

查看文件`C:\ProgramData\VMware\VMware Tools\tools.conf`的内容

如果未指明脚本文件的绝对路径，脚本文件默认的绝对路径为`"C:\Program Files\VMware\VMware Tools\"`

## 0x05 小结
---

本文分析了VMware Tools脚本执行功能的利用思路，给出防御建议。



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




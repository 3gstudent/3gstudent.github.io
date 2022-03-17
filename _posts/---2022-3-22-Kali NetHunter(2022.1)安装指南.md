---
layout: post
title: Kali NetHunter(2022.1)安装指南
---


## 0x00 前言
---

Kali NetHunter是首个针对Android设备的开源渗透测试平台，允许从Android设备运行Kali工具集。

目前最新的版本为2022月1月，对于这个版本，还没有一个完整的安装指南。于是本文从零开始，完整介绍Android设备Nexus6P安装Kali NetHunter的方法，记录细节。

## 0x01 简介
---

本文将要介绍以下内容：

- Kali NetHunter的不同版本
- Nexus6P安装Kali NetHunter


## 0x02 Kali NetHunter的不同版本
---

参考资料：

https://www.kali.org/docs/nethunter/

Kali NetHunter分为三个不同的版本：

- NetHunter Rootless，不需要root，不需要TWRP，可以从https://store.nethunter.com/下载apk安装，不支持Wifi和HID攻击
- NetHunter Lite，不需要root，需要TWRP，功能不完整
- NetHunter，需要root，需要TWRP，只支持部分Android设备，功能最完整

为了能够完整的体验Kali NetHunter的功能，我们需要安装NetHunter，支持的设备型号可参考：https://stats.nethunter.com/nethunter-images.html

这里选取官方首推的低端设备Nexus6P (Oreo)，介绍安装方法


## 0x03 Nexus6P (Oreo)安装Kali NetHunter
---

基本概念：

- adb：全称Android Debug Bridge，用来调试设备
- fastboot：常用功能为设备解锁，刷写img文件，格式化系统分区和运行img文件
- TWRP：全称Team Win Recovery Project，常用功能为刷机、备份和恢复
- Magisk：常用功能为获得root权限

总体流程如下：

1.开启Nexus6P的`OEM unlocking`和`USB debugging`
2.使用adb进入Bootloder模式
3.使用fastboot刷入TWRP
4.通过TWRP安装Android系统镜像Oreo
5.通过TWRP安装Kali NetHunter和Magisk

具体步骤如下：

### 1.下载文件

#### (1)adb和fastboot

需要下载到Windows系统并配置环境变量

这里可以选择一键下载配置，下载地址：https://forum.xda-developers.com/t/official-tool-windows-adb-fastboot-and-drivers-15-seconds-adb-installer-v1-4-3.2588979/#post-48915118

运行adb-setup-1.4.3.exe按照提示即可

#### (2)TWRP

下载页面：https://dl.twrp.me/angler/

这里选择twrp-3.6.1_9-0-angler.img，下载地址：https://dl.twrp.me/angler/twrp-3.6.1_9-0-angler.img.html

将twrp-3.6.1_9-0-angler.img保存在Windows系统中，可通过fastboot刷入Nexus6P

#### (3)Oreo

Oreo是指Android 8的系统镜像

下载页面：https://developers.google.com/android/images

这里选择8.0.0 (OPR5.170623.014, Dec 2017)，下载地址：https://dl.google.com/dl/android/aosp/angler-ota-opr5.170623.014-234956cb.zip

#### (4)Magisk

下载页面：https://github.com/topjohnwu/Magisk

这里选择Magisk-v21.4.zip，下载地址：https://github.com/topjohnwu/Magisk/releases/download/v21.4/Magisk-v21.4.zip

#### (5)Kali NetHunter

下载地址：https://kali.download/nethunter-images/kali-2022.1/nethunter-2022.1-angler-oreo-kalifs-full.zip

### 2.解锁Nexus6P的Bootloader

#### (1)启动开发者选项

打开Nexus6P，依次选择`Settings` -> `System` -> `About phone`，多次点击`Build number`可启动`Developer options`

#### (2)修改手机设置

点击`Developer options`，打开`OEM unlocking`和`USB debugging`

#### (3)连接设备

通过USB数据线将Nexus6P连接Windows系统

#### (4)解锁

Windows系统的命令行执行：

```
adb reboot bootloader
```

等待Nexus6P重启，进入Bootloader模式

Windows系统的命令行执行：

```
fastboot flashing unlock
```

Nexus6P用`音量+`选择`yes`，按`电源键`进行确认

至此，解锁完成。解锁后每次开机会出现提示`Your device software can't be checked for corruption. Please lock the bootloader. PRESS POWER TO PAUSE BOOT`

### 3.刷入TWRP

#### (1)进入Bootloader模式

关机状态下，同时按住`电源键`和`音量-`

#### (2)连接设备

通过USB数据线将Nexus6P连接Windows系统

通过Windows系统命令行查看设备：

```
fastboot devices
```

能够获得回显

#### (3)刷入TWRP

Windows系统的命令行执行：

```
fastboot flash recovery twrp-3.6.1_9-0-angler.img
```

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-3-22/2-1.png)

#### (4)Nexus6P启动TWRP

Nexus6P用`音量-`切换到`Recovery mode`，按`电源键`进行确认

等待Nexus6P启动TWRP

### 4.将文件复制到Nexus6P

#### (1)Oreo

将镜像文件angler-ota-opr5.170623.014-234956cb.zip复制到手机的根目录下

在Nexus6P启动TWRP后，Windows系统可以访问手机内的文件，可以进行文件复制操作，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-3-22/3-1.png)

也可以通过adb的push命令复制：`adb push angler-ota-opr5.170623.014-234956cb.zip /sdcard`

**注：**

push命令需要等待很长时间

#### (2)Kali NetHunter

将nethunter-2022.1-angler-oreo-kalifs-full.zip复制到手机的根目录下

#### (3)Magisk-v21.4.zip

将Magisk-v21.4.zip复制到手机的根目录下

### 5.使用TWRP安装Oreo 8.0

在Nexus6P的TWRP页面，选择`Install`，选择angler-ota-opr5.170623.014-234956cb.zip进行安装

安装成功后选择`Reboot System`

至此，Android Oreo 8.0系统安装完成

### 6.使用TWRP安装Kali NetHunter和Magisk

#### (1)进入TWRP

Nexus6P关机状态下，同时按住`电源键`和`音量-`

通过USB数据线将Nexus6P连接Windows系统

Windows系统命令行：

```
fastboot.exe flash recovery twrp-3.6.1_9-0-angler.img
```

Nexus6P用`音量-`切换到`Recovery mode`，按`电源键`进行确认

等待Nexus6P启动TWRP

#### (2)安装Kali NetHunter

在Nexus6P的TWRP页面，选择`Install`，选择nethunter-2022.1-angler-oreo-kalifs-full.zip

需要取消选择`Reboot after installation is complete`避免安装后Nexus6P自动重启

经过漫长的等待，安装成功

#### (3)安装Magisk-v21.4.zip

在Nexus6P的TWRP页面，选择`Install`，选择Magisk-v21.4.zip

安装成功后选择`Reboot System`

至此，Kali NetHunter安装完成

在Nexus6P的应用列表中，能够看到新安装的NetHunter、NetHunter Terminal 和NetHunterKeX

## 0x04 小结
---

本文介绍了在Nexus6P安装Kali NetHunter的方法，其他设备可依次类推。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



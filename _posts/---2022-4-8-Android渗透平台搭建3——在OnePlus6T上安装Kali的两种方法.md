---
layout: post
title: Android渗透平台搭建3——在OnePlus6T上安装Kali的两种方法
---


## 0x00 前言
---

Android渗透平台搭建的系列文章第三篇，介绍Android设备OnePlus6T上安装Kali的两种方法，记录细节。

方法一：在Android系统安装Kali NetHunter(2022.1)

方法二：在Win11系统安装Linux子系统kali-linux

测试设备：OnePlus 6T 10g+256g 迈凯伦

测试设备系统：Android 11+Win11

## 0x01 简介
---

本文将要介绍以下内容：

- 在Android系统安装Kali NetHunter(2022.1)
- 在Win11系统安装Linux子系统kali-linux

## 0x02 在Android系统安装Kali NetHunter(2022.1)
---

流程可参考之前的文章《Android渗透平台搭建——在Nexus6P安装Kali NetHunter(2022.1)》

Win11切换至Android的方法：

开机时按`音量+`，选择`UEFI BootMenu`，再选择`Reboot to other slot`

### 1.下载文件

(1)Kali NetHunter

需要Android版本10或Android版本11

OnePlus6T Android 11的下载地址：https://kali.download/nethunter-images/kali-2022.1/nethunter-2022.1-oneplus6-oos-eleven-kalifs-full.zip

OnePlus6T Android 10的下载地址：https://kali.download/nethunter-images/kali-2022.1/nethunter-2022.1-oneplus6-oos-ten-kalifs-full.zip

(2)Magisk

下载页面：https://github.com/topjohnwu/Magisk

这里选择Magisk-v21.4.zip，下载地址：https://github.com/topjohnwu/Magisk/releases/download/v21.4/Magisk-v21.4.zip

(3)TWRP

下载页面：https://twrp.me/oneplus/oneplus6t.html

下载地址：

https://dl.twrp.me/fajita/twrp-3.6.1_9-0-fajita.img

https://dl.twrp.me/fajita/twrp-installer-3.6.1_9-0-fajita.zip

下载得到文件twrp-3.6.1_9-0-fajita.img和twrp-installer-3.6.1_9-0-fajita.zip

twrp-3.6.1_9-0-fajita.img用于通过fastboot启动TWRP，twrp-installer-3.6.1_9-0-fajita.zip用于永久安装TWRP

### 2.安装Kali NetHunter

(1)进入Recovery模式

在开机状态下，按住`电源键`，选择`恢复模式`，进入Recovery模式

在TWRP页面，选择`Install`，我的OnePlus6T为Android 11，这里选择nethunter-2022.1-oneplus6-oos-eleven-kalifs-full.zip

需要取消选择`Reboot after installation is complete`避免安装后自动重启

经过漫长的等待，安装成功

安装Magisk-v21.4.zip

在TWRP页面，选择`Install`，选择Magisk-v21.4.zip

安装成功后选择`Reboot System`

至此，Kali NetHunter安装完成

在应用列表中，能够看到新安装的NetHunter、NetHunter Terminal 和NetHunterKeX

## 0x03 在Win11系统安装Linux子系统kali-linux
---

Android切换至Win11的方法：

进入Recovery模式，在TWRP中，选择`Reboot` -> `SlotB`

进入Win11系统后，可选择在微软商店搜索kali进行安装，具体流程如下：

### 1.开启子系统Linux

Powershell：

```
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

安装完会提示需要重启操作系统

### 2.在微软商店搜索kali并安装

安装后直接启动kali会报错`0x80370102`，这里需要手动设置wsl版本为1

wsl命令可参考：https://docs.microsoft.com/en-us/windows/wsl/basic-commands

### 3.设置wsl版本为1

命令如下：

```
wsl --set-default-version 1
```

查看已安装系统的命令如下:

```
wsl --list --verbose
```

返回结果能看到`kali-linux`

### 4.配置Kali默认登录用户

设置默认登录用户为root的命令如下：

```
kali config --default-user root
```

配置root用户密码，命令如下：

```
kali
passwd root
toor
toor
```

### 5.更新kali

```
sudo apt update && sudo apt upgrade -y
```

如果无法更新，可以修改成国内的源

```
sudo vi /etc/apt/sources.list 
```

添加：

```
deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
```

### 6.安装Kali GUI

本地Win11可通过远程桌面连接至Kali

首先在命令行输入`kali`进入kali控制台

(1)安装kali-desktop-xfce

```
sudo apt install kali-desktop-xfce -y
```

(2)安装xrdp

```
sudo apt install xrdp -y
```

(3)修改端口为3390

```
sed -i 's/port=3389/port=3390/g' /etc/xrdp/xrdp.ini
```

(4)开启服务

```
sudo service xrdp start
```

(5)连接远程桌面

```
mstsc
127.0.0.1:3390
```

#### 补充：常见问题

(1)修改kali分辨率

在kali中依次选择`Settings` -> `Appearance` -> `Settings` -> `Window Scaling`，改成`2x`

(2)无法打开命令行终端，提示：`Failed to execute default Terminal Emulator. Input/output error.`

先安装xfce的终端：

```
sudo apt install xfce4-terminal
```

在kali中依次选择`Settings` -> `Settings Manager` -> `DefaultApplications` -> `Utilities`，将设置`Terminal Emulator`设置为`Xfce Terminal`

(3)Win11重启后重新连接kali的远程桌面

需要先开启服务xrdp：

```
sudo service xrdp restart
```

(4)Win11 Linux子系统重启

```
net stop LxssManager
net start LxssManager
```

(5)安装msfconsole

```
sudo apt install metasploit-framework
```

(6)启动msfconsole

需要在Windows Defender中添加排除目录：`\\wsl.localhost\kali-linux`

## 0x04 小结
---

本文将介绍了在Android设备OnePlus6T上安装Kali的两种方法。骁龙845处理器的设备可以选择安装Win11再安装Kali，其他Android设备可选择安装Kali NetHunter。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)

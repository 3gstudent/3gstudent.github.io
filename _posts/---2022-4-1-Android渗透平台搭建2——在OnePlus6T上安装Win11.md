---
layout: post
title: Android渗透平台搭建2——在OnePlus6T上安装Win11
---


## 0x00 前言
---

Android渗透平台搭建的系列文章第二篇，介绍Android设备OnePlus6T上安装Win11操作系统的方法，记录细节。

测试设备：OnePlus 6T 10g+256g 迈凯伦

简单理解：采用骁龙845处理器的手机设备能够安装Arm版的Win11

完整资料：https://renegade-project.cn/#/README

参考资料：

http://www.oneplusbbs.com/thread-4446250-1.html

https://forum.renegade-project.org/t/6-windows/194

https://www.bilibili.com/video/BV1kM4y137bR

https://silime.gitee.io/2021/05/20/Windows10-on-arm64/

https://baijiahao.baidu.com/s?id=1721563590612500439&wfr=spider&for=pc

## 0x01 简介
---

本文将要介绍以下内容：

- 深度刷机的方法
- 安装Win11的准备
- 安装Win11的方法

## 0x02 深度刷机的方法
---

这里把深度刷机放在第一部分，是因为在刷机过程中很容易黑砖，只能通过深度刷机进行还原

在刷机过程中，错误的操作有可能导致手机无法开机，即9008 download模式，即常说的黑砖

这时只能通过深度刷机的方法重新刷入系统，也就是常说的救砖

救砖教程参考资料：http://www.oneplusbbs.com/thread-4446250-1.html

### 1.下载文件

在救砖教程中提供的网盘进行下载

(1)9008驱动

网盘中的高通9008驱动（推荐）.exe

(2)线刷救砖包

OnePlus 6T迈凯伦定制版有专用的救砖包，网盘中提供的迈凯伦救砖包是氧OS版，后续还需要升级成氢OS

(3)一加万能工具包

如果无法识别OnePlus 6T，可以安装`一加万能工具包` -> `驱动安装` -> `黑砖驱动`

(4)氢OS系统安装包

文件列表如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-1/1-0.png)

### 2.安装9008驱动

运行`高通9008驱动（推荐）.exe`

### 3.安装底层驱动

(1)在Windows系统打开设备管理器，位置：`我的电脑`->`右键`->`管理`，在计算机管理中选择`设备管理器`

(2)OnePlus 6T在关机状态下，同时按住`音量+`和`音量-`不放，通过USB数据线将OnePlus 6T连接Windows系统

等待Windows系统自动安装驱动

在设备管理器中，查看"端口（COM和LPT）"，如果出现`Qualcomm HS-USB QDLoader 9008（COM3）`代表底层驱动安装成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-1/1-1.png)

(3)管理员身份运行MsmDownloadTool V4.0.exe

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-1/1-2.png)

点击`Start`开始刷机，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-1/1-3.png)

等待一段时间，刷机成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-1/1-4.png)

OnePlus 6T会自动开机，进行初始化，默认安装氧OS

### 4.刷入氢OS

网盘中提供的迈凯伦救砖包是氧OS版，需要刷成氢OS，可以使用OnePlus 6T内置的本地升级功能

(1)将OnePlus6THydrogen_41_OTA_032_all_1903251445_5c8a300ab3b84fa5.zip复制到OnePlus 6T的根目录

(2)在OnePlus 6T上依次选择`设置` -> `系统` -> `系统更新`-> `右上角设置` -> `本地升级`，选择OnePlus6THydrogen_41_OTA_032_all_1903251445_5c8a300ab3b84fa5.zip

等待升级完成，点击`重启手机`

### 5.升级氢OS Android 10

在OnePlus 6T上依次选择`设置` -> `系统` -> `系统更新`，进行在线升级

在线升级后，最新版本为Android 11，在安装Win11之前我们先需要降级到Android 10，可以采用以下方法进行降级：

下载降级包：https://download.h2os.com/OnePlus6T/Back/OnePlus6THydrogen_34.K.51_OTA_051_all_2105262300_downgrade_e10c56ab63f04596.zip

在OnePlus 6T上依次选择`设置` -> `系统` -> `系统更新`-> `右上角设置` -> `本地升级`，选择OnePlus6THydrogen_34.K.51_OTA_051_all_2105262300_downgrade_e10c56ab63f04596.zip

补充：官方OnePlus 6T系统安装包的下载地址：

https://www.oneplus.com/cn/support/softwareupgrade/details?code=PM1574150307705

**注：**

我也考虑过在氢OS Android 9进行卡刷直接升级到OS Android 10的方法和在氢OS Android 11进行卡刷直接降级到OS Android 10的方法，但是这两种方法我在测试过程中失败了，都是因为无法通过Bootloader模式安装TWRP


## 0x03 安装Win11的准备
---

Windows系统只需要配置adb和fastboot，然后是一些文件的下载

### 1.adb和fastboot

需要下载到Windows系统并配置环境变量

这里可以选择一键下载配置，下载地址：https://forum.xda-developers.com/t/official-tool-windows-adb-fastboot-and-drivers-15-seconds-adb-installer-v1-4-3.2588979/#post-48915118

运行adb-setup-1.4.3.exe按照提示即可

### 2.TWRP下载

下载页面：https://twrp.me/oneplus/oneplus6t.html

下载地址：

https://dl.twrp.me/fajita/twrp-3.6.1_9-0-fajita.img

https://dl.twrp.me/fajita/twrp-installer-3.6.1_9-0-fajita.zip

下载得到文件twrp-3.6.1_9-0-fajita.img和twrp-installer-3.6.1_9-0-fajita.zip

twrp-3.6.1_9-0-fajita.img用于通过fastboot启动TWRP，twrp-installer-3.6.1_9-0-fajita.zip用于永久安装TWRP

### 3.parted下载

Linux下的分区工具

源码下载地址：https://alpha.gnu.org/gnu/parted/parted-3.3.52.tar.xz

需要手动编译

也可以下载编译好的文件：https://pwdx.lanzoux.com/iUgSEmkrlmh

下载得到文件parted

### 4.驱动下载

项目页面:https://github.com/edk2-porting/WOA-Drivers

一加6T的下载地址：https://github.com/edk2-porting/WOA-Drivers/releases/download/v1.1.1/fajita.tar.gz

下载得到文件fajita.tar.gz

### 5.UEFI固件下载

项目地址：https://github.com/edk2-porting/edk2-sdm845

一加6T的下载地址：https://github.com/edk2-porting/edk2-sdm845/releases/download/v1.1.1/boot-fajita-10g.img

下载得到文件boot-fajita-10g.img

### 6.Win11镜像下载

下载arm版的Win11镜像文件，解压后将sources\install.wim复制提取出来

最终得到文件install.wim

### 7.Dism++下载

项目地址：https://github.com/Chuyu-Team/Dism-Multi-language

下载地址：https://github.com/Chuyu-Team/Dism-Multi-language/releases/download/v10.1.1002.1/Dism++10.1.1002.1.zip

解压缩得到文件夹Dism++10.1.1002.1

### 8.WinPE 下载

在参考资料中的百度网盘中下载

解压缩后的文件列表如下：

- boot文件夹
- efi文件夹
- sources文件夹
- bootmgr.efi文件

## 0x04 安装Win11的方法
---

### 1.解锁Bootloader

**注：**

解锁Bootloader将擦除Android系统的所有数据

#### (1)启动开发者选项

打开OnePlus 6T，依次选择`设置` -> `关于手机`，多次点击`版本号`可启动`开发者模式`

#### (2)修改手机设置

依次选择`设置` -> `系统` -> `开发者选项`，打开`OEM解锁`、`USB调试`和`高级重启`

按住`电源键`，选择`引导加载器`，进入Bootloader模式，此时`DEVICE STATE`状态为`locked`

#### (3)连接设备

通过USB数据线将OnePlus 6T连接Windows系统

#### (4)解锁

Windows系统的命令行执行：

```
fastboot oem unlock
```

OnePlus 6T用`音量+`选择`yes`，按`电源键`进行确认

至此，解锁完成。

解锁操作将会清空所有数据，此时需要重新启动`开发者模式`，打开`USB调试`和`高级重启`

解锁后每次开机会出现提示`The bootloader is unlocked`

### 2.刷入TWRP

#### (1)进入Bootloader模式

在关机状态下，同时按住`电源键`和`音量-`

也可以在开机状态下，按住`电源键`，选择`引导加载器`，进入Bootloader模式

此时`DEVICE STATE`状态为`unlocked`

#### (2)连接设备

通过USB数据线将OnePlus 6T连接Windows系统

通过Windows系统命令行查看设备：

```
fastboot devices
```

能够获得回显

#### (3)刷入TWRP

Windows系统的命令行执行：

```
fastboot boot twrp-3.6.1_9-0-fajita.img
```

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-1/2-1.png)

等待OnePlus 6T启动TWRP

### 3.分区

进入TWRP后，将parted复制到OnePlus 6T的根目录，将twrp-installer-3.6.1_9-0-fajita.zip复制到OnePlus 6T的根目录

在TWRP中，安装twrp-installer-3.6.1_9-0-fajita.zip，这是为了方便以后在进入Recovery模式会自动启动TWRP

在TWRP中，选择`Reboot` -> `Recovery`，重新进入Recovery模式

此时可选择两种方式运行parted进行分区：

#### (1)通过Windows的命令行执行

```
adb shell
cp /sdcard/parted /sbin/
chmod 755 /sbin/parted
umount /data && umount /sdcard
parted /dev/block/sda
```

#### (2)在OnePlus 6T的TWRP中直接操作

依次选择`Advanced` -> `Terminal`

```
cp /sdcard/parted /sbin/
chmod 755 /sbin/parted
umount /data && umount /sdcard
parted /dev/block/sda
```

执行`cp /sdcard/parted /sbin/`的原因是因为在执行`umount /sdcard`后，无法访问/sdcard下的文件

查看分区：

```
(parted) p
```

删除分区userdata：

```
(parted) rm 17
```

创建分区：

```
(parted) mkpart esp fat32 6559MB 7000MB
(parted) mkpart pe fat32 7000MB 17000MB
(parted) mkpart win ntfs 17000MB 200GB
(parted) mkpart userdata ext4 200GB 246GB
```

我的环境下，esp对应的分区号为17，对应的命令如下:

```
(parted) set 17 esp on
```

在TWRP中，选择`Reboot` -> `Recovery`

### 4.格式化

重新进入Recovery后依次选择Advanced -> Terminal，命令如下：

```
mkfs.fat -F32 -s1 /dev/block/by-name/pe
mkfs.fat -F32 -s1 /dev/block/by-name/esp
mkfs.ntfs -f /dev/block/by-name/win
mke2fs -t ext4 /dev/block/by-name/userdata
```

在TWRP中，选择`Reboot` -> `Recovery`，重新进入Recovery模式

### 5.挂载PE

重新进入Recovery后，将以下文件复制到手机中：

- install.wim，Win11 ISO文件中的sources\install.wim
- boot，解压自winpe
- efi，解压自winpe
- sources，解压自winpe
- bootmgr.efi，解压自winpe
- Dism++10.1.1002.1
- fajita，解压自https://github.com/edk2-porting/WOA-Drivers/releases/download/v1.1.1/fajita.tar.gz
boot-fajita-10g.img，下载自https://github.com/edk2-porting/edk2-sdm845/releases/download/v1.1.1/boot-fajita-10g.img

文件如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-1/2-2.png)

**注：**

手机使用fat32格式，无法直接复制大于4G的文件，可以选择将其复制到U盘中

选择`Advanced` -> `Terminal`，将文件复制到PE分区的命令如下：

```
mount /dev/block/by-name/pe /mnt
cp -r /sdcard/* /mnt
```

### 6.切换分区，选择`Slot B`

在TWRP中，选择`Reboot` -> `SlotB`

### 7.启动PE

在TWRP中，选择`Install` -> `Install Image` -> `/mnt/boot-fajita-10g.img`，刷入镜像的分区选择`Boot`

手机重启后进入PE系统，在手机上接入键盘、鼠标和U盘

### 8.安装Win11

打开PE中的C:\Dism++10.1.1002.1\Dism++ARM64.exe

在Dism++的页面，依次选择`文件` -> `释放镜像`

第一个参数设置为install.wim

第二个参数安装路径设置为D盘

勾选`添加引导`

点击`确定`

释放完毕后需要修复引导，依次选择`工具箱` -> `修复引导`

### 9.安装Win11驱动

在Dism++的页面，依次选择`打开会话` -> `驱动管理` -> `添加驱动`，选择fajita文件夹即可

### 10.设置盘符

我的环境下，esp对应的分区号为17，在PE中的CMD输入以下命令：

```
diskpart
select disk 0
list part
select part 17
assign letter=Y
exit
```

打开Y盘确认是否成功创建文件夹EFI

### 11.关闭签名验证

关闭签名的命令如下：

```
bcdedit /store Y:\efi\microsoft\boot\bcd /set {Default} testsigning on
bcdedit /store Y:\efi\microsoft\boot\bcd /set {Default} nointegritychecks on
```

关闭PE系统:

```
shutdown -s -t 0
```

### 12.进入Win11

按电源键进行开机，等待安装即可

### 补充1：Win11切换至Android

开机时按`音量+`，选择`UEFI BootMenu`，再选择`Reboot to other slot`

### 补充2：Android切换至Win11

进入Recovery模式，在TWRP中，选择`Reboot` -> `SlotB`

## 0x05 小结
---

本文介绍了在OnePlus6T上安装Win11的完整方法。



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)

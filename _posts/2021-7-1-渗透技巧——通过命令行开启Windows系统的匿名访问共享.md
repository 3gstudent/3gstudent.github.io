---
layout: post
title: 渗透技巧——通过命令行开启Windows系统的匿名访问共享
---


## 0x00 前言
---

在渗透测试中，尤其是内网渗透，通常需要在内网开启一个支持匿名访问的文件共享，配合漏洞利用。

所以我们需要一种通用的方法，不仅需要使用便捷，还需要能够在命令行下运行。

## 0x01 简介
---

本文将要介绍以下内容：

- 利用场景
- 通过界面开启可匿名访问的文件共享服务器
- 通过命令行开启可匿名访问的文件共享服务器
- 开源代码


## 0x02 利用场景
---

开启支持匿名访问的文件共享后，其他用户不需要输入用户名和口令，可以直接访问文件服务器的共享文件

通常有以下两种用法：

1. 作为数据传输的通道
2. 配合漏洞利用，作为Payload的下载地址

文件共享服务器需要能够在不同操作系统上搭建

对于Linux系统，可借助Samba服务搭建可匿名访问的文件共享服务器

这里给出Kali系统下的使用方法：

修改文件`/etc/samba/smb.conf`，内容如下：

```
[global]
    map to guest = test1
    server role = standalone server
    usershare allow guests = yes
    idmap config * : backend = tdb
    smb ports = 445

[smb]
    comment = Samba
    path = /tmp/
    guest ok = yes
    read only = no
    browsable = yes
```

开启服务：

```
service smbd start 
service nmbd start 
```

对于Windows系统，需要考虑域环境和工作组环境。为了支持匿名访问，需要开启Guest用户，允许Guest用户访问文件共享服务器的内容

## 0x03 通过界面开启可匿名访问的文件共享服务器
---

具体方法如下：

#### 1.启用Guest用户

运行`gpedit.msc`，打开组策略

位置：`Computer Configuration`->`Windows Settings`->`Security Settings`->`Local Policies`->`Security Options`

选择策略`Accounts: Guest account status`，设置为`Enabled`

#### 2.将Everyone权限应用于匿名用户

位置：`Computer Configuration`->`Windows Settings`->`Security Settings`->`Local Policies`->`Security Options`

选择策略`Network access:Let Everyone permissions apply to anonymous users`，设置为`Enabled`

#### 3.指定匿名共享文件的位置

位置：`Computer Configuration`->`Windows Settings`->`Security Settings`->`Local Policies`->`Security Options`

选择策略`Network access:Shares that can be accessed anonymously`，设置名称，这里可以填入`smb`

#### 4.将Guest用户从策略“拒绝从网络访问这台计算机”中移除

位置：`Computer Configuration`->`Windows Settings`->`Security Settings`->`Local Policies`->`User Rights Assignment`

选择策略`Deny access to this computer from the network`，移除用户Guest

#### 5.设置文件共享

选择要共享的文件夹，设置高级共享，共享名为`smb`，共享权限组或用户名为`Everyone`

至此，可匿名访问的文件共享服务器开启成功，访问的地址为`//<ip>/smb`

## 0x04 通过命令行开启可匿名访问的文件共享服务器
---

具体方法对应的命令如下：

#### 1.启用Guest用户

```
net user guest /active:yes
```

#### 2.将Everyone权限应用于匿名用户

```
REG ADD "HKLM\System\CurrentControlSet\Control\Lsa" /v EveryoneIncludesAnonymous /t REG_DWORD /d 1 /f
```

#### 3.指定匿名共享文件的位置

```
REG ADD "HKLM\System\CurrentControlSet\Services\LanManServer\Parameters" /v NullSessionShares /t REG_MULTI_SZ /d smb /f
```

#### 4.将Guest用户从策略“拒绝从网络访问这台计算机”中移除

导出组策略：

```
secedit /export /cfg gp.inf /quiet
```

修改文件gp.inf，将`SeDenyNetworkLogonRight = Guest`修改为`SeDenyNetworkLogonRight =`，保存

重新导入组策略：

```
secedit /configure /db gp.sdb /cfg gp.inf /quiet
```

强制刷新组策略，立即生效(否则，重启后生效)：

```
gpupdate/force
```

#### 5.设置文件共享

```
icacls C:\share\ /T /grant Everyone:r
net share share=c:\share /grant:everyone,full
```

至此，可匿名访问的文件共享服务器开启成功，访问的地址为`//<ip>/smb`

## 0x05 开源代码
---

完整的Powershell代码已开源，地址如下：

https://github.com/3gstudent/Invoke-BuildAnonymousSMBServer

代码在以下操作系统测试成功：

- Windows 7
- Windows 8
- Windows 10
- Windows Server 2012
- Windows Server 2012 R2
- Windows Server 2016

支持域环境和工作组环境的Windows操作系统

需要本地管理员权限执行

开启可匿名访问的文件共享服务器：

```
Invoke-BuildAnonymousSMBServer -Path c:\share -Mode Enable
```

关闭可匿名访问的文件共享服务器：

```
Invoke-BuildAnonymousSMBServer -Path c:\share -Mode Disable
```

**注：**

关闭可匿名访问的文件共享服务器实现了以下操作：

- 关闭指定目录的共享权限
- 禁用Guest用户
- 禁用将Everyone权限应用于匿名用户
- 删除组策略中指定的匿名共享文件位置
- 将Guest用户添加至策略“拒绝从网络访问这台计算机”

在导出组策略时，如果策略“拒绝从网络访问这台计算机”中的内容为空，那么不会有这一选项，当我们需要添加这个策略时，需要手动添加一行内容`SeDenyNetworkLogonRight = Guest`

在代码实现上，我采用了以下方法：

将`SeDenyInteractiveLogonRight = Guest`

替换为

```
SeDenyNetworkLogonRight = Guest
SeDenyInteractiveLogonRight = Guest
```

对应的Powershell示例代码：

```
(Get-Content a.txt) -replace "SeDenyInteractiveLogonRight = Guest","SeDenyNetworkLogonRight = Guest`r`nSeDenyInteractiveLogonRight = Guest" | Set-Content "a.txt"
```
    
## 0x06 小结
---

本文实现了命令行下对匿名访问共享的开启和关闭，开源代码，可用于测试CVE-2021-1675和CVE-2021-34527。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




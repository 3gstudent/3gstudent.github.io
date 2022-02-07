---
layout: post
title: 渗透基础——Windows Defender
---


## 0x00 前言
---

Windows Defender是一款内置在Windows操作系统的杀毒软件程序，本文仅在技术研究的角度介绍Windows Defender相关的渗透方法，分析利用思路，给出防御建议。

## 0x01 简介
---

本文将要介绍以下内容：

- 查看Windows Defender版本
- 查看已存在的查杀排除列表
- 关闭Windows Defender的Real-time protection
- 添加查杀排除列表
- 移除Token导致Windows Defender失效
- 恢复被隔离的文件

## 0x02 查看Windows Defender版本
---

### 1.通过面板查看

依次选择`Windows Security`->`Settings`->`About`，`Antimalware Client Verions`为Windows Defender版本，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-2-3/1-1.png)

### 2.通过命令行查看

```
dir "C:\ProgramData\Microsoft\Windows Defender\Platform\" /od /ad /b
```

数字大的为最新版本

## 0x03 查看已存在的查杀排除列表
---

### 1.通过面板查看

依次选择`Windows Security`->`Virus & theat protection settings`->`Add or remove exclusions`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-2-3/1-2.png)

### 2.通过命令行查看

```
reg query "HKLM\SOFTWARE\Microsoft\Windows Defender\Exclusions" /s
```

### 3.通过Powershell查看

```
Get-MpPreference | select ExclusionPath
```

## 0x04 关闭Windows Defender的Real-time protection
---

### 1.通过面板关闭

依次选择`Windows Security`->`Virus & theat protection settings`，关闭`Real-time protection`

### 2.通过命令行关闭

利用条件：

- 需要TrustedInstaller权限
- 需要关闭Tamper Protection

```
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Real-Time Protection" /v "DisableRealtimeMonitoring" /d 1 /t REG_DWORD /f
```

**注：**

运行成功时，桌面右下角会弹框提示Windows Defender已关闭

### 补充1：开启Windows Defender的Real-time protection

利用条件：

- 需要TrustedInstaller权限
- 需要关闭Tamper Protection

```
reg delete "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Real-Time Protection" /v "DisableRealtimeMonitoring" /f
```

### 补充2：获得TrustedInstaller权限

可参考之前的文章[《渗透技巧——Token窃取与利用》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Token%E7%AA%83%E5%8F%96%E4%B8%8E%E5%88%A9%E7%94%A8)

也可以借助[AdvancedRun](https://www.nirsoft.net/utils/advanced_run.html)，命令示例：

```
AdvancedRun.exe /EXEFilename "%windir%\system32\cmd.exe" /CommandLine '/c reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Real-Time Protection" /v "DisableRealtimeMonitoring" /d 1 /t REG_DWORD /f' /RunAs 8 /Run
```

### 补充3：Tamper Protection

参考资料：
https://docs.microsoft.com/en-us/microsoft-365/security/defender-endpoint/prevent-changes-to-security-settings-with-tamper-protection?view=o365-worldwide

当开启Tamper Protection时，用户将无法通过注册表、Powershell和组策略修改Windows Defender的配置

开启Tamper Protection的方法：

依次选择`Windows Security`->`Virus & theat protection settings`，启用`Tamper Protection`

该操作对应的cmd命令：`reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Features" /v "TamperProtection" /d 5 /t REG_DWORD /f`

关闭Tamper Protection的方法：

依次选择`Windows Security`->`Virus & theat protection settings`，禁用`Tamper Protection`

该操作对应的cmd命令：`reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Features" /v "TamperProtection" /d 4 /t REG_DWORD /f`，当然，我们无法通过修改注册表的方式去设置Tamper Protection，只能通过面板进行修改

查看Tamper Protection的状态：

```
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Features" /v "TamperProtection"
```

返回结果中的数值`5`代表开启，数值`4`代表关闭

### 补充4：通过Powershell关闭Windows Defender的Real-time protection

```
Set-MpPreference -DisableRealtimeMonitoring $true
```

**注：新版本的Windows已经不再适用**

### 补充5：通过组策略关闭Windows Defender的Real-time protection

依次打开`gpedit.msc`->`Computer Configuration`->`Administrative Templates`->`Windows Components`->`Microsoft Defender Antivirus`->`Real-time Protection`，选择`Turn off real-time protection`，配置成`Enable`

**注：新版本的Windows已经不再适用**

## 0x05 添加查杀排除列表
---

### 1.通过面板添加

依次选择`Windows Security`->`Virus & theat protection settings`->`Add or remove exclusions`，选择`Add an exclusion`，指定类型

该操作等价于修改注册表`HKLM\SOFTWARE\Microsoft\Windows Defender\Exclusions\`的键值，具体位置如下：

- 类型File对应注册表项Paths
- 类型Folder对应注册表项Paths
- 类型File type对应注册表项Extensions
- 类型Process对应注册表项Processes

### 2.通过命令行添加

利用条件：

- 需要TrustedInstaller权限

cmd命令示例：

```
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Exclusions\Paths" /v "c:\test" /d 0 /t REG_DWORD /f
```

### 3.通过Powershell添加

利用条件：

- 需要管理员权限

参考资料：

https://docs.microsoft.com/en-us/powershell/module/defender/add-mppreference?view=windowsserver2022-ps

Powershell命令示例：

```
Add-MpPreference -ExclusionPath "C:\test"
```

补充：删除排除列表

```
Remove-MpPreference -ExclusionPath "C:\test"
```

## 0x06 移除Token导致Windows Defender失效
---

学习地址：

https://elastic.github.io/security-research/whitepapers/2022/02/02.sandboxing-antimalware-products-for-fun-and-profit/article/

简单理解：

- Windows Defender进程为MsMpEng.exe
- MsMpEng.exe是一个受保护的进程(Protected Process Light，简写为PPL)
- 非PPL进程无法获取PPL进程的句柄，导致我们无法直接结束PPL进程MsMpEng.exe
- 但是我们能够以SYSTEM权限运行的线程修改进程MsMpEng.exe的token
- 当我们移除进程MsMpEng.exe的所有token后，进程MsMpEng.exe无法访问其他进程的资源，也就无法检测其他进程是否有害，最终导致Windows Defender失效

POC地址：https://github.com/pwn1sher/KillDefender

利用条件：

- 需要System权限

测试如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-2-3/2-1.png)

## 0x07 恢复被隔离的文件
---

参考资料：

https://docs.microsoft.com/en-us/microsoft-365/security/defender-endpoint/command-line-arguments-microsoft-defender-antivirus?view=o365-worldwide

### 1.定位MpCmdRun

```
dir "C:\ProgramData\Microsoft\Windows Defender\Platform\" /od /ad /b
```

得到`<antimalware platform version>`

MpCmdRun的位置为：`C:\ProgramData\Microsoft\Windows Defender\Platform\<antimalware platform version>`

### 2.常用命令

查看被隔离的文件列表：

```
MpCmdRun -Restore -ListAll
```

恢复指定名称的文件至原目录：

```
MpCmdRun -Restore -FilePath C:\test\mimikatz_trunk.zip
```

恢复所有文件至原目录：

```
MpCmdRun -Restore -All
```

查看指定路径是否位于排除列表中：

```
MpCmdRun -CheckExclusion -path C:\test
```

## 0x08 防御建议
---

阻止通过命令行关闭Windows Defender：开启Tamper Protection

阻止通过移除Token导致Windows Defender失效：阻止非PPL进程修改PPL进程MsMpEng.exe的token，工具可参考：https://github.com/elastic/PPLGuard

## 0x09 小结
---

本文在仅在技术研究的角度介绍Windows Defender相关的渗透方法，分析利用思路，给出防御建议。对于移除Token导致Windows Defender失效的利用方法，可能会在未来版本的Windows中默认解决这个问题。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



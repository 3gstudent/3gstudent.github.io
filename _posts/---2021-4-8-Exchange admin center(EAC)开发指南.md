---
layout: post
title: Exchange admin center(EAC)开发指南
---

## 0x00 前言
---

Exchange admin center(EAC)是Exchange Server中基于Web的管理控制台，在渗透测试和漏洞利用中，通常需要通过代码实现对EAC的操作，本文将要开源一份
操作EAC的实现代码[eacManage](https://github.com/3gstudent/Homework-of-Python/blob/master/eacManage.py)，记录开发细节，便于后续的二次开发。

## 0x01 简介
---

本文将要介绍以下内容：

- 程序实现原理
- 开源代码eacManage
- eacManage功能介绍

## 0x02 EAC的基本操作
---

介绍EAC的资料：

https://docs.microsoft.com/en-us/Exchange/architecture/client-access/exchange-admin-center?view=exchserver-2019

### 1.添加邮箱用户和设置邮箱用户的权限

添加邮箱用户需要在recipients->mailboxes页面下进行操作，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-4-8/2-1.png)

设置邮箱用户的权限需要在permissions->admin roles页面下进行操作，admin roles页面下默认建立了多个管理员角色组(Role Group)，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-4-8/2-2.png)

每个管理员角色组(Role Group)可以通过设置角色(Roles)属性来设定具体的权限，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-4-8/2-3.png)

将邮箱用户添加至指定的管理员角色组就可以获得对应的权限，修改权限可以通过新建管理员角色组(Role Group)或者设置已有管理员角色组(Role Group)的角色(Roles)属性实现

### 2.导出所有邮箱用户列表

需要在recipients->mailboxes页面下进行操作，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-4-8/2-4.png)

## 0x03 程序实现原理
---

目前，Exchange Server并未开放程序实现的接口，但我们可以通过构造特定格式的POST数据包实现

抓包可以选择以下两种方式：

1. Chrome浏览器自带的抓包工具，可直接抓取明文数据，在Chrome界面按F12选择Network即可，具体细节可参考之前的文章[《渗透基础——通过Outlook Web Access(OWA)读取Exchange邮件的命令行实现》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-%E9%80%9A%E8%BF%87Outlook-Web-Access(OWA)%E8%AF%BB%E5%8F%96Exchange%E9%82%AE%E4%BB%B6%E7%9A%84%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%AE%9E%E7%8E%B0)
2. Wireshark，抓取明文数据需要配置证书，方法可参考之前的文章[《渗透技巧——Pass the Hash with Exchange Web Service》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Pass-the-Hash-with-Exchange-Web-Service)

具体的POST数据包格式如下：

### 1.查看所有管理员角色组

请求url：/ecp/UsersGroups/AdminRoleGroups.svc/GetList

参数：

- msExchEcpCanary

数据格式：application/json

发送内容：

```
{"filter":{"SearchText":""},"sort":{"Direction":0,"PropertyName":"Name"}}
```

### 2.新建管理员角色组

创建管理员角色组(Role Group)时，需要设置(Roles)属性，即设置该管理员角色组的权限

分为以下步骤：

#### (1)获得每个角色的RawIdentity

请求url：/ecp/UsersGroups/ManagementRoles.svc/GetList

参数：

- msExchEcpCanary

数据格式：application/json

发送内容：

```
{"filter":{},"sort":{"Direction":0,"PropertyName":"DisplayName"}}
```

#### (2)创建管理员角色组

请求url：/ecp/UsersGroups/AdminRoleGroups.svc/NewObject

参数：

- msExchEcpCanary

数据格式：application/json

发送内容：

```
{
      "properties":
      {
        "Name":<newName>,
        "Description":"",
        "AggregatedScope":
        {
          "IsOrganizationalUnit":false,
          "ID":"00000000-0000-0000-0000-000000000000"
        },
        "Roles":
        [
          {
            "__type":"Identity:ECP",
            "DisplayName":<newRole>,
            "RawIdentity":<newRawIdentity>
          }
        ],
      }
}
```

### 3.编辑管理员角色组：

将指定用户添加至指定管理员角色组，使该用户获得对应的权限

分为以下步骤：

#### (1)获得指定邮箱用户的RawIdentity

请求url：/ecp/DDI/DDIService.svc/GetList

参数：

- schema
- msExchEcpCanary

数据格式：application/json

发送内容：

```
{
      "filter":
      {
        "Parameters":
        {
          "__type":"JsonDictionaryOfanyType:#Microsoft.Exchange.Management.ControlPanel",
          "SearchText":"[[\"anr\",\"startsWith\",[\"" + <mailbox> + "\"]]]"
        }
      },
      "sort":{}
}
```

#### (2)获得指定管理员角色组的RawIdentity

格式见“1.查看所有管理员角色组”

#### (3)编辑管理员角色组

请求url：/ecp/UsersGroups/AdminRoleGroups.svc/SetObject

参数：

- msExchEcpCanary

数据格式：application/json

发送内容：

```
{
      "identity":
      {
        "__type":"Identity:ECP",
        "DisplayName":<editRole>,
        "RawIdentity":<editRoleRawIdentity>
      },
      "properties":
      {
        "Members":
        [
          {
            "__type":"Identity:ECP",
            "DisplayName":<addUser>,
            "RawIdentity":<addUserRawIdentity>
          }
        ],
        "ReturnObjectType":1
      }
}
```

### 4.删除管理员角色组

请求url：/ecp/UsersGroups/AdminRoleGroups.svc/RemoveObjects

参数：

- msExchEcpCanary

数据格式：application/json

发送内容：

```
{
            "identities":
            [
                {
                    "__type":"Identity:ECP",
                    "DisplayName":<removeRole>,
                    "RawIdentity":<removeRoleRawIdentity>
                }
            ],
            "parameters":{}
}
```

### 5.新建邮箱用户

请求url：/ecp/DDI/DDIService.svc/NewObject

参数：

- msExchEcpCanary
- schema

数据格式：application/json

发送内容：

```
{
    "properties":{
        "Parameters":
        {
                "__type":"JsonDictionaryOfanyType:#Microsoft.Exchange.Management.ControlPanel",
                "RemoteArchive":false,
                "UserPrincipalName":<newUser>,
                "IsNewMailbox":"true",
                "DisplayName":<newUsername>,
                "Name":<newUsername>,
                "PlainPassword":<newUserpassword》,
                "ResetPasswordOnNextLogon":false,
                "EnableArchive":false
        }
    },
    "sort":{}
}
```


### 6.删除指定邮箱用户

#### (1)获得指定用户的RawIdentity

格式见“3.编辑管理员角色组->(1)获得指定邮箱用户的RawIdentity”

#### (2)删除指定邮箱用户

请求url：/ecp/DDI/DDIService.svc/MultiObjectExecute

参数：

- msExchEcpCanary
- schema
- workflow

数据格式：application/json

发送内容：

```
{
            "identities":
            [

                {
                    "__type":"Identity:ECP",
                    "DisplayName":<removeUser>,
                    "RawIdentity":<removeUserRawIdentity>
                }
            ],
            "parameters":{}
}
```

### 7.导出所有邮箱信息

请求url：/ecp/UsersGroups/Download.aspx

参数：

- msExchEcpCanary
- schema
- handlerClass

数据格式：application/x-www-form-urlencoded

发送内容：

```
workflowOutput=DisplayName%2CMailboxType%2CPrimarySmtpAddress&titlesCSV=DISPLAY+NAME%2CMAILBOX+TYPE%2CEMAIL+ADDRESS&PropertyList=DisplayName%2CRecipientTypeDetails%2CPrimarySmtpAddress
```

 

## 0x04 开源代码
---

完整实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/eacManage.py

代码支持以下功能：

- ListAdminRoles
- NewAdminRoles
- EditAdminRoles
- DeleteAdminRoles
- AddMailbox
- RemoveMailbox
- ExportAllMailbox

在程序实现上，首先需要登录操作获得参数msExchEcpCanary的内容，具体细节如下：

#### (1)访问/owa/auth.owa

细节可参考之前的文章[《渗透基础——通过Outlook Web Access(OWA)读取Exchange邮件的命令行实现》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-%E9%80%9A%E8%BF%87Outlook-Web-Access(OWA)%E8%AF%BB%E5%8F%96Exchange%E9%82%AE%E4%BB%B6%E7%9A%84%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%AE%9E%E7%8E%B0)

这里新加了一个自动识别用户是否初次登录的功能，如果是初次登录的邮箱用户，需要选择时区和语言才能生效

#### (2)访问/ecp

在返回的Cookie中获得参数msExchEcpCanary的内容

## 0x05 小结 
---

本文开源了操作EAC的实现代码[eacManage](https://github.com/3gstudent/Homework-of-Python/blob/master/eacManage.py)，记录开发细节，便于后续的二次开发，可应用到多个漏洞(如CVE-2020-16875，CVE-2021-24085，CVE-2021-26855，CVE-2021-27065)



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




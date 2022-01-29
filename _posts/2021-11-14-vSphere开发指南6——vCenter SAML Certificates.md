---
layout: post
title: vSphere开发指南6——vCenter SAML Certificates
---


## 0x00 前言
---

我最近学到的一个利用方法：在vCenter上使用管理员权限，从`/storage/db/vmware-vmdir/data.mdb`提取IdP证书，为管理员用户创建SAML请求，最后使用vCenter server进行身份验证并获得有效的管理员cookie。

直观理解：从vCenter本地管理员权限到VCSA管理面板的管理员访问权限。

学习资料：

https://www.horizon3.ai/compromising-vcenter-via-saml-certificates/
https://github.com/horizon3ai/vcenter_saml_login

本文将要在学习资料的基础上，完善代码，增加通用性，结合利用思路给出防御建议。

## 0x01 简介
---

本文将要介绍以下内容：

- 方法复现
- 脚本优化
- 利用思路
- 防御建议

## 0x02 方法复现
---

在Kali系统下进行测试

安装Openssl：

```
apt install python3-openssl
```

### 1.从vCenter获得数据库文件

路径：`/storage/db/vmware-vmdir/data.mdb`

需要vCenter管理员权限

### 2.运行脚本

下载地址：

https://github.com/horizon3ai/vcenter_saml_login/blob/main/vcenter_saml_login.py

命令参数示例：

```
python3 ./vcenter_saml_login.py -t 192.168.1.1 -p data.mdb
```

命令行返回结果：

```
JSESSIONID=XX533CDFA344DE842517C943A1AC7611
```

3.登录VCSA管理面板

访问`https://192.168.1.1/ui`

设置Cookie: `JSESSIONID=XX533CDFA344DE842517C943A1AC7611`

成功以管理员身份登录管理面板

## 0x03 脚本优化
---

通常data.mdb的大小至少为20MB

为了减少交互流量，选择将[vcenter_saml_login.py](https://github.com/horizon3ai/vcenter_saml_login/blob/main/vcenter_saml_login.py)修改成能够直接在vCenter下使用

**注：**

vCenter默认安装Python

在脚本修改上具体需要考虑以下问题：

### 1.去掉引用第三方包bitstring

我采用的方式是将第三方包bitstring的内容进行精简，直接插入到Python脚本中

### 2.避免使用f-字符串格式化

Python3.6新增了一种f-字符串格式化

vCenter 6.7的版本为Python 3.5.6，不支持格式化的字符串文字前缀为"f"

我采用的方式是使用format实现格式化字符串

例如：

```
cn = stream.read(f'bytes:{cn_len}').decode()
```

替换为：

```
cn = stream.read('bytes:{}'.format(cn_len)).decode()
```

完整代码已上传至Github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/vCenter_ExtraCertFromMdb.py

vCenter_ExtraCertFromMdb.py可上传至vCenter后直接执行，执行后会得到以下四个重要的参数：

- domain，在命令行显示
- idp_cert，保存为idp_cert.txt
- trusted_cert_1，保存为trusted_cert_1.txt
- trusted_cert_2，保存为trusted_cert_2.txt

接下来，可在任意主机上为管理员用户创建SAML请求，使用vCenter server进行身份验证并获得有效的管理员cookie，完整代码已上传至Github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/vCenter_GenerateLoginCookie.py

参数说明如下：

- target: VCSA管理面板的URL
- hostname: 对应VCSA管理面板的证书Subject属性中的CN
- domain: 可以使用vCenter_ExtraCertFromMdb.py从data.mdb中获得
- idp_cert path: 可以使用vCenter_ExtraCertFromMdb.py从data.mdb中获得
- trusted_cert_1 path: 可以使用vCenter_ExtraCertFromMdb.py从data.mdb中获得
- trusted_cert_2 path: 可以使用vCenter_ExtraCertFromMdb.py从data.mdb中获得

## 0x04 利用思路
---

### 1.从vCenter本地管理员权限到VCSA管理面板的管理员访问权限

前提：通过漏洞获得了vCenter本地管理员权限

利用效果：

获得VCSA管理面板的管理员访问权限，能够同vCenter可管理的虚拟机进行交互

**注：**

此时还可以通过《vSphere开发指南5——LDAP》中介绍的方法通过LDAP数据库添加管理员用户，进而同vCenter可管理的虚拟机进行交互

### 2.从vCenter备份文件中得到data.mdb

前提：需要获得正确的data.mdb文件

利用效果：

获得VCSA管理面板的管理员访问权限，能够同vCenter可管理的虚拟机进行交互

## 0x05 防御建议
---

1.更新补丁，避免攻击者获得vCenter本地管理员权限

2.避免在用的vCenter备份文件泄露

## 0x06 小结
---

本文介绍了[vcenter_saml_login](https://github.com/horizon3ai/vcenter_saml_login)的优化思路，增加通用性，结合利用思路给出防御建议。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



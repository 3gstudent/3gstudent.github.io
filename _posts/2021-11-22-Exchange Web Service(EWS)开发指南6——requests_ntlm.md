---
layout: post
title: Exchange Web Service(EWS)开发指南6——requests_ntlm
---


## 0x00 前言
---

在之前的文章《Exchange Web Service(EWS)开发指南4——Auto Downloader》和《Exchange Web Service(EWS)开发指南5——exchangelib》介绍了两种利用hash访问Exchange资源的方法，各有特点。
前者采用了较为底层的通信协议，在功能实现上相对繁琐，但是有助于理解通信协议原理和漏洞利用。后者借助第三方库exchangelib，开发便捷，但是不太适用于漏洞利用。

站在漏洞利用的角度，如果仅使用封装NTLM认证的第三方包，既不影响漏洞利用，又能兼顾效率。

所以本文选取了第三方包requests_ntlm，以自动化下载邮件和提取附件为例，开源代码，介绍用法。

## 0x01 简介
---

本文将要介绍以下内容：

- requests_ntlm用法
- 开发细节
- 开源代码

## 0x02 requests_ntlm用法
---

说明文档：

https://github.com/requests/requests-ntlm

### 1.两种登录方法

我在低于1.0.0版本的requests_ntlm.py找到了使用Hash登录的方法，代码位置：

https://github.com/requests/requests-ntlm/blob/v0.3.0/requests_ntlm/requests_ntlm.py#L16

这里可以找到使用Hash登录的参数格式为`ABCDABCDABCDABCD:ABCDABCDABCDABCD`

两种登录Exchange的示例代码如下：

#### (1)明文登录

```
target = "192.168.1.1"
username = "administrator@test.com"
password = "password1"
res = requests.post("https://" + target + "/ews/exchange.asmx", data=POST_BODY, headers=headers, verify=False, auth=HttpNtlmAuth(username, password))
print(res.status_code)
print(res.text)
```

#### (2)Hash登录

```
target = "192.168.1.1"
username = "administrator@test.com"
hash = "00000000000000000000000000000000:5835048CE94AD0564E29A924A03510EF"
res = requests.post("https://" + target + "/ews/exchange.asmx", data=POST_BODY, headers=headers, verify=False, auth=HttpNtlmAuth(username, hash))
print(res.status_code)
print(res.text)
```

### 2.Session机制

requests_ntlm支持Session机制，能够减少身份验证的次数，缩短通信数据包，效率更高

## 0x03 开发细节
---

### 1.使用requests_ntlm重新实现[ewsManage_Downloader.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_Downloader.py)

在原有脚本[ewsManage_Downloader.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_Downloader.py)的基础上，只需做以下替换即可

(1)

原代码：

```
status, responsetext = ntlm_auth_login(host, port, mode, domain, user, data, POST_BODY)
```

替换为：

```
res  = requests.post("https://" + host + "/ews/exchange.asmx", data=POST_BODY, headers=headers, verify=False, auth=HttpNtlmAuth(user, data))
```

(2)

原代码：

```
status
```

替换为：

```
res.status_code
```

(3)

原代码：

```
if res.status_code == "200" and "NoError" in responsetext:
```

替换为：

```
if res.status_code == 200 and "NoError" in res.text:
```

(4)

原代码：

```
host, port, mode, domain, user, data
```

替换为：

```
host, mode, user, data
```

(5)

原代码：

```
responsetext
```

替换为：

```
res.text
```

(6)

原代码：

```
sys.argv[1], int(sys.argv[2]), sys.argv[3], sys.argv[4], sys.argv[5], sys.argv[6]
```

替换为：

```
sys.argv[1], sys.argv[2], sys.argv[3], sys.argv[4]
```

(7)

原代码：

```
path1 + '\\' + sys.argv[5]
```

替换为：

```
path1 + '\\' + sys.argv[3]
```

最后再去除一些多余的代码即可

完整的代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_requests_ntlm_Downloader.py.py

### 2.使用Session机制重新实现[ewsManage_Downloader.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_Downloader.py)

通过Session机制能够减少身份验证的次数，缩短通信数据包，效率更高

所以在这里使用Session机制重新实现[ewsManage_Downloader.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_Downloader.py)

换用Session机制的方法示例：

原代码：

```
target = "192.168.1.1"
username = "administrator@test.com"
password = "password1"
res = requests.post("https://" + target + "/ews/exchange.asmx", data=POST_BODY, headers=headers, verify=False, auth=HttpNtlmAuth(username, password))
print(res.status_code)
print(res.text)
res = requests.post("https://" + target + "/ews/exchange.asmx", data=POST_BODY2, headers=headers, verify=False, auth=HttpNtlmAuth(username, password))
print(res.status_code)
print(res.text)
```

新代码：

```
target = "192.168.1.1"
username = "administrator@test.com"
password = "password1"
session = requests.Session()
session.auth = HttpNtlmAuth(username, password)
res = session.post("https://" + target + "/ews/exchange.asmx", data=POST_BODY, headers=headers, verify=False)
print(res.status_code)
print(res.text)
res = session.post("https://" + target + "/ews/exchange.asmx", data=POST_BODY2, headers=headers, verify=False)
print(res.status_code)
print(res.text)
```

完整的代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_requests_ntlm_Session_Downloader.py

## 0x04 小结
---

在Exchange Web Service(EWS)开发上，使用第三方包[requests_ntlm](https://pypi.org/project/requests_ntlm/)，封装了NTLM认证过程，不仅可以提高开发效率，同时不影响漏洞利用代码的编写，更可以借助Session机制减少认证过程的通信数据，十分推荐。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)







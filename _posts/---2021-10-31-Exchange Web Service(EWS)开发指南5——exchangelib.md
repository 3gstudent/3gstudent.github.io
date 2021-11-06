# Exchange Web Service(EWS)开发指南5——exchangelib

## 0x00 前言
---

在之前的文章介绍了通过SOAP XML message实现利用hash对Exchange资源的访问，由于采用了较为底层的通信协议，在功能实现上相对繁琐，但是有助于理解通信协议原理和漏洞利用。

如果仅仅为了更高效的开发一个资源访问的程序，可以借助Python库exchangelib实现。

本文将要介绍exchangelib的用法，开源代码，实现自动化下载邮件和提取附件。

## 0x01 简介
---

本文将要介绍以下内容：

- exchangelib用法
- 开发细节
- 开源代码

## 0x02 exchangelib用法
---

参考资料：

https://github.com/ecederstrand/exchangelib

https://ecederstrand.github.io/exchangelib/

### 1.简单的登录测试

代码如下：

```
from exchangelib import Credentials, Account, Configuration, DELEGATE
credentials = Credentials(username='MYWINDOMAIN\\myuser', password='topsecret')
config = Configuration(server='outlook.office365.com', credentials=credentials)
account = Account(primary_smtp_address='john@example.com', config=config,
                  autodiscover=False, access_type=DELEGATE)
for item in account.inbox.all().order_by('-datetime_received')[:100]:
    print(item.subject, item.sender, item.datetime_received)
```

如果Exchange服务器证书不可信，需要忽略证书验证，加入以下代码：

```
from exchangelib.protocol import BaseProtocol, NoVerifyHTTPAdapter
BaseProtocol.HTTP_ADAPTER_CLS = NoVerifyHTTPAdapter
```

屏蔽输出的提示信息InsecureRequestWarning，加入以下代码:

```
import urllib3
urllib3.disable_warnings()
```

完整代码如下：

```
from exchangelib import Credentials, Account, Configuration, DELEGATE
from exchangelib.protocol import BaseProtocol, NoVerifyHTTPAdapter
BaseProtocol.HTTP_ADAPTER_CLS = NoVerifyHTTPAdapter
import urllib3
urllib3.disable_warnings()
credentials = Credentials(username='MYWINDOMAIN\\myuser', password='topsecret')
config = Configuration(server='outlook.office365.com', credentials=credentials)
account = Account(primary_smtp_address='john@example.com', config=config,
                  autodiscover=False, access_type=DELEGATE)
for item in account.inbox.all().order_by('-datetime_received')[:100]:
    print(item.subject, item.sender, item.datetime_received)
```

### 2.使用明文或hash登录

使用明文登录：

```
credentials = Credentials('MYWINDOMAIN\\myuser', 'topsecret')
```

使用Hash登录：

```
credentials = Credentials('MYWINDOMAIN\\myuser', '00000000000000000000000000000000:7C451851EA87B63EC7692126416D01EB')
```

### 3.统计邮件数目

示例代码：

```
n = a.inbox.all().count()
```

### 4.指定时间范围进行搜索

示例代码：

```
for item in account.inbox.filter(datetime_received__gt=EWSDateTime(2021, 1, 20, tzinfo=account.default_timezone)):
    print(item.subject, item.sender, item.datetime_received)
```

### 5.指定下载数量：

指定前10个：

```
first_ten = a.inbox.all()[:10]
```

指定后10个：

```
last_ten = a.inbox.all()[:-10]
```

指定区间：

```
next_ten = a.inbox.all()[10:20]
```

### 6.文件夹枚举

能够遍历出邮箱用户下的所有文件夹，示例代码：

```
print(account.root.tree())
```

### 7.将Python脚本编译成exe

如果将使用exchangelib开发的Python脚本编译成exe格式，使用`pyinstaller -F test.py`命令会报错，提示：`No time zone found with key UTC`

解决方法：

```
pyinstaller --collect-all tzdata --onefile test.py
```

## 0x03 开发细节
---

### 1.通信协议

exchangelib也是通过SOAP XML message实现利用hash对Exchange资源的访问

### 2.邮件保存

exchangelib中会自动XML格式邮件内容进行解析，在保存邮件时，可以直接提取出对应的信息

这里需要注意的是可以将返回结果中的字符串`\r\n`替换成换行符，提高数据可读性

示例代码：

```
filtered_items = email.inbox.all()
for item in items:
with open(item.id, "w") as fw:
	fw.write(str(item).replace('\\r\\n','\r\n'))
```

### 3.条件匹配

exchangelib支持Advanced Query Syntax (AQS)

AQS参考资料：

https://docs.microsoft.com/en-us/exchange/client-developer/web-service-reference/querystring-querystringtype

利用AQS可以实现日期搜索，搜索格式示例：

```
sent:>=2021/1/1 AND sent:<=2021/12/30
received:>=2021/1/1 AND received:<=2021/12/30
```

对于关键词搜索，无法直接使用AQS，可以选择接收所有邮件再进行字符串匹配


## 0x04 开源代码
---

完整代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_exchangelib_Downloader.py

支持明文和NTLM Hash的登录，代码支持以下功能：

- 支持自己的Exchange服务器和Office365(outlook.office365.com)
- download，下载邮件并提取附件，可指定邮箱文件夹和下载数量
- search，邮件搜索并下载，支持关键词、时间、长度等语法
- listfolder，枚举用户所有文件夹

在下载邮件时，以邮件用户名作为父文件夹，不同操作会创建不同的子文件夹，在使用搜索功能创建子文件夹时，为了避免特殊字符(例如字符`>`)无法作为文件夹名称，这里会去掉特殊字符

## 0x05 小结
---

本文介绍了exchangelib的用法，开源代码[ewsManage_exchangelib_Downloader.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ewsManage_exchangelib_Downloader.py)，实现了利用hash对Exchange资源的访问。

如果想要快速开发一个EWS的资源访问程序，推荐选择exchangelib


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


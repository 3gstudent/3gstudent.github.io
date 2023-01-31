---
layout: post
title: Zimbra-SOAP-API开发指南5——邮件转发
---


## 0x00 前言
---

本文将要继续扩充开源代码[Zimbra_SOAP_API_Manage](https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py)的功能，通过`Zimbra SOAP API`修改配置实现邮件转发，分享开发细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 添加邮件转发
- 查看邮件转发的配置
- 查看文件夹共享的配置
- 开源代码

## 0x02 添加邮件转发
---

Zimbra支持将收到的邮件额外转发至另一邮箱，通过Web界面的操作方法如下：

登录邮箱后，依次选择`Preferences`->`Mail`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-3/2-1.png)

设置转发邮箱后，点击`Save`

如果想要转发多个邮箱，可以使用`,`进行分割，同时转发至两个邮箱的示例：`test1@test.com,test2@test.com`

接下来，通过抓包的方式分析实现流程，进而使用程序实现这部分功能

抓包获得的soap格式示例：

```
<soap:Body>
<BatchRequest xmlns="urn:zimbra" onerror="stop">
<ModifyPrefsRequest xmlns="urn:zimbraAccount" requestId="0">
<pref name="zimbraPrefMailForwardingAddress">test1@test.com</pref>
</ModifyPrefsRequest>
</BatchRequest>
</soap:Body>
```

实现代码示例：

```
def addforward_request(uri,token):
    print("[*] Input the mailbox to forward:")
    print("    Eg :test1@test.com,test2@@test.com")
    mailbox = input("[>]: ")
    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
       <soap:Header>
           <context xmlns="urn:zimbra">
               <authToken>{token}</authToken>
           </context>
       </soap:Header>
       <soap:Body>
            <BatchRequest xmlns="urn:zimbra" onerror="stop">
                <NoOpRequest xmlns="urn:zimbraMail" requestId="0"/>
                <ModifyPrefsRequest xmlns="urn:zimbraAccount" requestId="1">
                    <pref name="zimbraPrefMailForwardingAddress">{mailbox}</pref>
                </ModifyPrefsRequest>
            </BatchRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        r=requests.post(uri+"/service/soap",headers=headers,data=request_body.format(token=token,mailbox=mailbox),verify=False,timeout=15)
        if r.status_code == 200:
            print("[+] Add success")
        else:    
            print(r.status_code)
            print(r.text)        
        
    except Exception as e:
        print("[!] Error:%s"%(e))
```

对于清除邮件转发的设置，只需要将邮箱地址设置为空即可

## 0x03 查看邮件转发的配置
---

在我们添加邮件转发前，通常需要先获得邮箱转发的配置

通过抓包发现，在访问Web主页面时，如果存在邮件转发的设置，那么返回数据会增加以下内容：

```
"zimbraPrefMailForwardingAddress":"test@test.com"
```

如果不存在邮件转发的设置，返回数据不存在字符`zimbraPrefMailForwardingAddress`

在程序实现上，访问Web主页面需要添加Cookie，再通过正则表达式筛选出指定的内容即可

实现代码示例：

```
def getforward_request(uri,token):
    try:
        headers["Cookie"]="ZM_AUTH_TOKEN="+token+";"
        r=requests.get(uri,headers=headers,verify=False,timeout=15)
        if r.status_code == 200 and 'zimbraPrefMailForwardingAddress' in r.text:
            print("[+] Forward")
            pattern_name = re.compile(r"\"zimbraPrefMailForwardingAddress\":\"(.*?)\"")
            name = pattern_name.findall(r.text)
            print("    " + name[0])
        else:           
            print(r.status_code)
            print("[-] No Forward")
        
    except Exception as e:
        print("[!] Error:%s"%(e))
```

## 0x04 查看文件夹共享的配置
---

在上篇文章《Zimbra-SOAP-API开发指南4——邮件导出和文件夹共享》缺少了查看文件夹共享配置的方法，本文作为补充

通过抓包进行分析

发送的url示例： `https://<url>/service/soap/BatchRequest`

发送的内容示例：

```
{"Header":{"context":{"_jsns":"urn:zimbra","userAgent":{"name":"ZimbraWebClient - GC103 (Win)","version":"8.8.12_GA_3844"},"session":{"_content":123,"id":123},"account":{"_content":"admin@test.com","by":"name"},"csrfToken":"0_71c4fc5d29c57ec1863d1630a77bb4834f0cd67c"}},"Body":{"BatchRequest":{"_jsns":"urn:zimbra","onerror":"continue","GetFolderRequest":[{"_jsns":"urn:zimbraMail","folder":{"l":"2"},"requestId":0}]}}}
```

返回的内容示例：

```
{"Header":{"context":{"session":{"id":"123","_content":"123"},"change":{"token":151},"_jsns":"urn:zimbra"}},"Body":{"BatchResponse":{"GetFolderResponse":[{"folder":[{"id":"2","uuid":"68dd08c1-26ea-4460-9716-14eee9103a45","deletable":false,"name":"Inbox","absFolderPath":"/Inbox","l":"1","luuid":"0e366bb5-f76c-40ce-9a92-28def5720d67","f":"ui","u":14,"view":"message","rev":1,"ms":147,"webOfflineSyncDays":30,"activesyncdisabled":false,"n":14,"s":24088,"i4ms":112,"i4next":273,"acl":{"grant":[{"zid":"f87692f9-0ab9-441d-9870-ef5b6dd6f375","gt":"usr","perm":"r","d":"test1@test.com"}]}}],"requestId":"0","_jsns":"urn:zimbraMail"}],"_jsns":"urn:zimbra"}},"_jsns":"urn:zimbraSoap"}
```

从以上内容可以知道，相关的请求为`GetFolderRequest`

查看GetFolderRequest的用法：https://files.zimbra.com/docs/soap_api/8.8.15/api-reference/zimbraMail/GetFolder.html

经过前期的积累，这里也可以通过`Zimbra SOAP API`实现，发送GetFolderRequest，对返回内容进行筛选即可

收件箱存在文件共享的数据内容示例：

```
<folder i4ms="201"rev="1" i4next="282" f="ui" ms="147" deletable="0" l="1" uuid="68dd08c1-26ea-4460-9716-14eee9103a45" n="16" luuid="0e366bb5-f76c-40ce-9a92-28def5720d67" activesyncdisabled="0" absFolderPath="/Inbox" view="message" s="29224" u="16" name="Inbox" id="2" webOfflineSyncDays="30"><acl><grant zid="f87692f9-0ab9-441d-9870-ef5b6dd6f375" perm="r" d="test1@test.com" gt="usr"/></acl></folder>
```

在程序实现上，如果返回结果中存在字符`<acl>`，代表存在文件共享，提取出对应的数据即可

实现代码示例：

```
def getshare_request(uri,token):
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
        r=requests.post(uri+"/service/soap",headers=headers,data=request_body.format(token=token),verify=False,timeout=15)
        if r.status_code == 200 and '<acl>' in r.text:
            print("[+] Folder Share")
            pattern_name = re.compile(r"<folder(.*?)</folder>")
            folders = pattern_name.findall(r.text)
            for i in range(len(folders)):
                if '<acl>' in folders[i]:
                    pattern_name = re.compile(r"name=\"(.*?)\"")
                    name = pattern_name.findall(folders[i])
                    pattern_name = re.compile(r"<acl>(.*?)</acl>")
                    acl = pattern_name.findall(r.text)
                    print("    " + name[len(name)-1] + ":")
                    print("    " + acl[0])
        else:
            print(r.status_code)
            print(r.text)
            print("[-] No Folder Share")        
        
    except Exception as e:
        print("[!] Error:%s"%(e))
```

返回结果示例：

```
Inbox:
<grant zid="f87692f9-0ab9-441d-9870-ef5b6dd6f375" perm="rwidx" d="test1@test.com" gt="usr"/>
```

在删除文件夹共享的操作时，需要填入`zid`和`Inbox`对应的数字`2`即可

## 0x05 开源代码
---

新的代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py

添加以下四个功能：

- AddForward：添加邮件转发
- GetForward：查看邮件转发
- GetShare：查看文件夹共享
- RemoveForward：清除邮件转发的设置

## 0x05 小结
---

本文扩充了Zimbra SOAP API的调用方法，添加四个实用功能，实现方法和思路也可在XSS漏洞上进行测试。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


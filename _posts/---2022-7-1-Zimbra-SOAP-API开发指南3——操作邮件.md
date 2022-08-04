---
layout: post
title: Zimbra-SOAP-API开发指南3——操作邮件
---


## 0x00 前言
---

在之前的文章[《Zimbra SOAP API开发指南》](https://3gstudent.github.io/Zimbra-SOAP-API%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97)和[《Zimbra-SOAP-API开发指南2》](https://3gstudent.github.io/Zimbra-SOAP-API%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%972)介绍了Zimbra SOAP API的调用方法，开源代码[Zimbra_SOAP_API_Manage](https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py)。 本文将要在此基础上扩充功能，添加邮件操作的相关功能。

## 0x01 简介
---

本文将要介绍以下内容：

- 查看邮件
- 发送邮件
- 删除邮件

## 0x02 查看邮件
---

Zimbra SOAP API说明文档：https://files.zimbra.com/docs/soap_api/9.0.0/api-reference/index.html

结合Zimbra SOAP API说明文档和调试结果得出以下实现流程：

1. 调用`Search`命令获得邮件对应的`Item id`，通过`Item id`作为邮件的识别标志
2. 获得`Item id`后可以对邮件做进一步操作，如查看邮件细节、移动邮件、删除邮件等

### 1.获得邮件对应的Item id

需要使用`Search`命令

说明文档：https://files.zimbra.com/docs/soap_api/8.8.15/api-reference/zimbraMail/Search.html

需要用到以下参数：

(1)query

表示查看的位置，示例如下：

查看收件箱：`<query>in:inbox</query>`

查看发件箱：`<query>in:sent</query>`

查看垃圾箱：`<query>in:trash</query>`

(2)limit

表示返回的查询结果数量，示例如下：

指定返回数量为10：`<limit>10</limit>`

如果不指定该属性，默认为`10`

测试代码：

```
def searchinbox_request(uri,token):
    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
       <soap:Header>
           <context xmlns="urn:zimbra">
               <authToken>{token}</authToken>
           </context>
       </soap:Header>
       <soap:Body>
         <SearchRequest  xmlns="urn:zimbraMail">
            <query>in:inbox</query>
         </SearchRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        print("[*] Try to search")
        r=requests.post(uri+"/service/soap",data=request_body.format(token=token),verify=False,timeout=15)
        print(r.text)               
    except Exception as e:
        print("[!] Error:%s"%(e))
```

返回内容示例：

```
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope"><soap:Header><context xmlns="urn:zimbra"><change token="147"/></context></soap:Header><soap:Body><SearchResponse offset="0" more="1" sortBy="dateDesc" xmlns="urn:zimbraMail"><c sf="1657272073000" d="1657272073000" u="0" id="-271" n="1"><su>Service zimlet stopped on mail.test.com</su><fr>Jun 14 01:49:12 mail zmconfigd[34031]: Service status change: mail.test.com zimlet changed from running to stopped</fr><e a="admin@mail.test.com" d="admin" t="f"/><m s="1701" d="1657272073000" id="271" l="2"/></c><c sf="1657272073000" d="1657272073000" u="0" id="-272" n="1"><su>Service service stopped on mail.test.com</su><fr>Jun 14 01:49:10 mail zmconfigd[34031]: Service status change: mail.test.com service changed from running to stopped</fr><e a="admin@mail.test.com" d="admin" t="f"/><m s="1703" d="1657272073000" id="272" l="2"/></c></SearchResponse></soap:Body></soap:Envelope>
```

对以上格式分析，发现标签`<c***</c>`对应每个邮件的信息，提取数据如下：

```
<c sf="1657272073000" d="1657272073000" u="0" id="-271" n="1"><su>Service zimlet stopped on mail.test.com</su><fr>Jun 14 01:49:12 mail zmconfigd[34031]:Service status change: mail.test.com zimlet changed from running to stopped</fr><e a="admin@mail.test.com" d="admin" t="f"/><m s="1701" ="1657272073000" id="271" l="2"/></c>
```

格式分析如下：

- `id="-271"`对应这个邮件的`Item id`
- `<su>***</su>`对应这个邮件的标题
- `<fr>***</fr>`对应这个邮件的正文内容
- `<e a="admin@mail.test.com" d="admin" t="f"/>`对应这个邮件的发件人
- `sf="1657272073000"`对应这个邮件的收取时间，格式为unix时间戳，没有额外计算时差

时间格式转换的示例代码：

```
from datetime import datetime
print(str(datetime.fromtimestamp(1657272073)))
```

综合以上内容，得出提取`Item id`、发件人、标题、正文内容和发送时间的实现代码：

```
def searchinbox_request(uri,token):
    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
       <soap:Header>
           <context xmlns="urn:zimbra">
               <authToken>{token}</authToken>
           </context>
       </soap:Header>
       <soap:Body>
         <SearchRequest  xmlns="urn:zimbraMail">
            <query>in:inbox</query>
         </SearchRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        print("[*] Try to search")
        r=requests.post(uri+"/service/soap",data=request_body.format(token=token),verify=False,timeout=15)        
        pattern_c = re.compile(r"<c (.*?)</c>")
        maildata = pattern_c.findall(r.text)
        print("[+] Total: " + str(len(maildata)))
        for i in range(len(maildata)):
            pattern_data = re.compile(r"id=\"(.*?)\"")
            data = pattern_data.findall(maildata[i])[0]
            print("[+] Item id: " + data)
            pattern_data = re.compile(r"a=\"(.*?)\"")
            data = pattern_data.findall(maildata[i])[0]
            print("    From: " + data)
            pattern_data = re.compile(r"<su>(.*?)</su>")
            data = pattern_data.findall(maildata[i])[0]
            print("    Subject: " + data)
            pattern_data = re.compile(r"<fr>(.*?)</fr>")
            data = pattern_data.findall(maildata[i])[0]
            print("    Body: " + data)
            pattern_data = re.compile(r"sf=\"(.*?)\"")
            data = pattern_data.findall(maildata[i])[0]
            data = str(datetime.fromtimestamp(int(data[:-3])))
            print("    UnixTime: " + data)      
    except Exception as e:
        print("[!] Error:%s"%(e))
```

### 2.查看邮件内容

测试发现，查看邮件细节可以不依赖Zimbra SOAP API，访问固定url即可

url格式：`https://<url>/service/home/~/?auth=co&view=text&id=<Item id>`

通过这种方式可以获得完整的邮件内容，包括Base64编码的附件内容

实现代码：

```
def viewmail_request(uri,token):
    id = input("[*] Input the item id of the mail:")
    headers["Cookie"]="ZM_AUTH_TOKEN="+token+";"
    r = requests.get(uri+"/service/home/~/?auth=co&view=text&id="+id,headers=headers,verify=False)
    if r.status_code == 200:        
        print("[*] Try to save the details of the mail")
        path = id + ".txt"        
        with open(path, 'w+', encoding='utf-8') as file_object:
            file_object.write(r.text)
        print("[+] Save as " + path)
    else:
        print("[!]")
        print(r.status_code)
        print(r.text)
```

## 0x03 发送邮件
---

在发送带有附件的邮件时，需要先上传附件，再发送

### 1.上传附件

上传功能通过`FileUploadServlet`实现，对应代码位置：`/opt/zimbra/lib/jars/zimbrastore.jar中/com.zimbra/cs/service/FileUploadServlet.class`

上传细节可参考：https://github.com/Zimbra/zm-mailbox/blob/develop/store/docs/file-upload.txt

上传的url: `https://<url>/service/upload`，返回结果示例：

```
<html><head></head><body onload="window.parent._uploadManager.loaded(200,'client_token',[{"aid":"server_token","filename":"image.jpeg","ct":"image/jpeg"}]);"></body></html>
```

如果添加参数`fmt=raw,extended`，返回结果示例：

```
200,'client_token',[{"aid":"server_token","filename":"image.jpeg","ct":"image/jpeg"}]
```

经过比较，发现添加参数`fmt=raw,extended`能够额外获得文件类型，示例:`"ct":"image/jpeg"`

所以在上传时，使用url: `https://<url>/service/upload?fmt=raw,extended`

综合以上内容，得出以下实现代码：

```
def uploadattachment_request(uri,token):
    fileContent = 0;
    path = input("[*] Input the path of the file:")
    with open(path,'rb') as f:
        fileContent = f.read()
    filename = path
    print("[*] filepath:"+path)
    if "\\" in path:
        strlist = path.split('\\')
        filename = strlist[-1]
    if "/" in path:  
        strlist = path.split('/')
        filename = strlist[-1]
    headers = {
    "Content-Type":"text/plain",
    "Content-Disposition":"attachment; filename=\""+filename+"\"",
    }   
    headers["Cookie"]="ZM_AUTH_TOKEN="+token+";"
    files = {filename: fileContent}
    r = requests.post(uri+"/service/upload?fmt=raw,extended",files=files,headers=headers,verify=False)
    if "200" in r.text:
        print("[+] Success")
        pattern_id = re.compile(r"aid\":\"(.*?)\"")
        attachmentid = pattern_id.findall(r.text)[0]
        pattern_type = re.compile(r"ct\":\"(.*?)\"")
        attachmenttype = pattern_type.findall(r.text)[0]
        print("    name:"+filename)
        print("    Type:%s"%(attachmenttype))
        print("    Id:%s"%(attachmentid))
        return attachmentid
    else:
        print("[!]")
        print(r.text)
```

### 2.发送带有附件的邮件

需要使用`SendMsg`命令

说明文档：https://files.zimbra.com/docs/soap_api/8.8.15/api-reference/zimbraMail/SendMsg.html


需要用到以下参数：

(1)e

表示发件人和收件人等相关信息，示例如下：

指定发件人：`<e t="t" a="admin@mail.test.com"/>`

指定收件人：`<e t="f" a="admin@mail.test.com"/>`

指定cc：`<e t="c" a="admin@mail.test.com"/>`

(2)su

表示邮件标题，示例如下：

```
<su>subjecttest1</su>
```

(3)mp

表示正文内容，示例如下：

```
<mp>
    <ct>"text/plain"</ct>
    <content>bodytest123456</content>
</mp>
```

(4)noSave

如果设置为`1`，表示邮件发送后，不在发件箱保存副本，示例代码：

```
<noSave>1</noSave>
```

(5)attach

指定发送附件的`aid`，示例代码：

```
<attach>
    <aid>4e53e807-879c-482b-b56f-4c6d927fbab2:50e03447-418f-42e0-932c-c2d3c2616163</aid>           
</attach>
```

综合以上内容，得出发送带有附件邮件的实现代码：

```
def sendmsgwithattachment_request(uri,token):
    aid = input("[*] Input the id of the attachment:")
    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
       <soap:Header>
           <context xmlns="urn:zimbra">
               <authToken>{token}</authToken>
           </context>
       </soap:Header>
       <soap:Body>
         <SendMsgRequest xmlns="urn:zimbraMail">
            <noSave>1</noSave>
            <m>
                <e t="t" a="admin@mail.test.com"/>
                <e t="f" a="admin@mail.test.com"/>
                <su>subjecttest1</su>
                <mp>
                    <ct>"text/plain"</ct>
                    <content>bodytest123456</content>
                </mp>
                <attach>
                    <aid>{aid}</aid>           
                </attach>
            </m>
         </SendMsgRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        print("[*] Try to send msg")
        r=requests.post(uri+"/service/soap",data=request_body.format(token=token,aid=aid),verify=False,timeout=15)
        if "soap:Reason" not in r.text:
            print("[+] Success")
        else:
            print("[!]")
            print(r.text)        
        
    except Exception as e:
        print("[!] Error:%s"%(e))
```

## 0x04 删除邮件
---

需要使用`ConvAction`命令

说明文档：https://files.zimbra.com/docs/soap_api/8.8.15/api-reference/zimbraMail/ConvAction.html

需要用到以下参数：

(1)tcon

指定遍历所有位置：`<tcon>o</tcon>`

指定遍历垃圾箱：`<tcon>t</tcon>`

通过浏览器删除邮件的流程是先点击删除邮件，将邮件移动至垃圾箱，再从垃圾箱中点击删除邮件，完成邮件的彻底删除

通过Zimbra-SOAP-API可以简化以上流程，直接删除邮件

实现代码：

```
def deletemail_request(uri,token):
    id = input("[*] Input the item id of the mail:")
    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
       <soap:Header>
           <context xmlns="urn:zimbra">
               <authToken>{token}</authToken>
           </context>
       </soap:Header>
       <soap:Body>
         <ConvActionRequest  xmlns="urn:zimbraMail">           
            <action>
                <op>delete</op>
                <tcon>o</tcon>
                <id>{id}</id>           
            </action>
         </ConvActionRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        print("[*] Try to send msg")
        r=requests.post(uri+"/service/soap",data=request_body.format(token=token,id=id),verify=False,timeout=15)
        if "soap:Reason" not in r.text:
            print("[+] Success")
        else:
            print("[!]")
            print(r.text)
        
    except Exception as e:
        print("[!] Error:%s"%(e))
```

## 0x05 开源代码
---

新的代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py

优化了代码结构，增加了以下功能：

- DeleteMail，删除指定邮件
- SearchMail，获得邮箱信息，包括`Item id`、发件人、标题、正文内容和发送时间
- SendTestMailToSelf，向当前邮箱发送一封带有附件的邮件
- uploadattachment，上传附件
- uploadattachmentraw，上传附件的另一种实现，用于特定条件
- viewmail，查看邮件完整细节

## 0x06 小结
---

本文扩充了Zimbra SOAP API的调用方法，添加三个实用功能：查看邮件、发送邮件和删除邮件，记录实现细节。



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



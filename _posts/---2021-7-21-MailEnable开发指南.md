---
layout: post
title: MailEnable开发指南
---


## 0x00 前言
---

MailEnable提供端到端的解决方案，用于提供安全的电子邮件和协作服务。引用自官方网站的说法：最近的一项独立调查报告称MailEnable是世界上最受欢迎的Windows邮件服务器平台。
对于MailEnable的开发者API，我在官方网站上只找到了[AJAX API的说明文档](http://www.mailenable.com/developers/help/webframe.html)，所以本文将要尝试编写Python脚本，实现对MailEnable邮件的访问，记录开发细节，开源代码。

## 0x01 简介
---

本文将要介绍以下内容：

- 环境搭建
- 开发细节
- 开源代码MailEnableManage.py

## 0x02 环境搭建
---

### 1.安装

安装前需要安装IIS服务和.Net 3.5，否则无法正常配置Web访问

MailEnable下载地址：http://www.mailenable.com/download.asp

### 2.配置

启动MailEnableAdmin.msc，在`MailEnable Management`->`Messaging Manager`->`Post Offices`下配置邮件服务器信息

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-7-21/2-1.png)

默认登录页面：

http://mewebmail.localhost/mewebmail/Mondo/lang/sys/login.aspx

### 3.开启Web管理页面

参考资料：

http://www.mailenable.com/kb/content/article.asp?ID=ME020132

启动MailEnableAdmin.msc，选择`MailEnable Management`->`Servers`->`localhost`->`Services and Connectors`->`WebAdmin`，右键单击并从弹出菜单中选择`Properties`，选择`Configure...`按钮，进行安装

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-7-21/2-2.png)


启动MailEnableAdmin.msc，在`MailEnable Management`->`Messaging Manager`->`Post Offices`下选择已配置的`Post Office`，右键单击并从弹出菜单中选择`Properties`，切换到`Web Admin`标签，启用`web administration`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-7-21/2-3.png)

选择指定用户，将属性修改为管理员

默认管理页面：

http://mewebmail.localhost/meadmin/Mondo/lang/sys/login.aspx

**注：**

如果忘记了用户的明文口令，可以查看默认安装路径`C:\Program Files (x86)\Mail Enable\Config`下的Auth.tab文件，其中保存有每个邮箱用户的明文口令

## 0x03 开发细节
---

### 1.版本判断

经过多个版本的测试，总结出来的版本判断方法如下：

访问登录页面：http://<url>/mewebmail/Mondo/lang/sys/login.aspx

查看网页源码，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-7-21/3-1.png)

其中`<link rel="stylesheet" type="text/css" href="/MEWebMail/Mondo/skins/Arctic/me.css?v=9.84">`中的`v=9.84`对应MailEnable的版本

在脚本实现上，我采用了如下方法：

1. 找到`?v=`的位置
2. 向后截取固定长度的字符串
3. 以`"`作为分隔符，取出版本号

#### 补充：通过MailEnableAdmin.msc获得版本号

启动MailEnableAdmin.msc，选择`MailEnable Management`->`Servers`->`localhost`->`System`->`Diagnose`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-7-21/3-2.png)

**注：版本号列表**

http://www.mailenable.com/Premium-ReleaseNotes.txt

http://www.mailenable.com/Standard-ReleaseNotes.txt

### 2.用户登录

访问URL：/mewebmail/Mondo/Servlet/request.aspx

需要的部分关键参数：

- txtUsername
- txtPassword
- loginParam

返回结果为json格式，如果登录成功，`bReportLoginFailure`的值为`False`

对应的Python代码如下：

```
def Check(host, username, password):
    url = host + "/mewebmail/Mondo/Servlet/request.aspx?Cmd=LOGIN&Format=JSON"

    body = {
            "txtUsername": username,
            "txtPassword": password,
            "ddlLanguages": "en",
            "ddlSkins":"Arctic",
            "loginParam":"SubmitLogin"
            }  
    postData = urllib.parse.urlencode(body).encode("utf-8")           
    headers = {
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0",
        "Content-Type": "application/x-www-form-urlencoded",    
    }         
    r = requests.post(url, headers=headers, data=postData, verify = False)
    if r.status_code ==200 and r.json()["bReportLoginFailure"] == False:
        print("[+] Valid:%s  %s"%(username, password))      
    else:
        print(r.status_code)
        print(r.text)
    r.close()
```

### 3.查看邮箱文件夹

访问URL：/MEWebMail/Mondo/Servlet/asyncrequest.aspx

需要的部分关键参数：

- Folder，可以指定为inbox/sent/drafts/deleted/junk
- ME_VALIDATIONTOKEN，需要访问/mewebmail/Mondo/Servlet/request.aspx?Cmd=GET-MBX-OPTIONS&Scope=2，从返回结果中获得

返回结果为xml格式，包含该文件夹下所有邮件的数量和每个邮件的简要内容，ID作为每封邮件的唯一标志，在读取邮件时需要作为参数

为了提高效率，可以使用xml.dom解析xml

使用xml.dom解析xml的参考资料：

https://docs.python.org/3.8/library/xml.dom.minidom.html#xml.dom.minidom.parse

使用xml.dom解析xml，提取出TOTAL_ITEMS的Python代码如下：

```
from xml.dom import minidom
DOMTree = minidom.parseString(r.text)
collection = DOMTree.documentElement
total = collection.getAttribute("TOTAL_ITEMS")
print("[+] TOTAL_ITEMS: " + total)
```

### 4.查看邮件

访问URL：/MEWebMail/Mondo/Servlet/request.aspx

需要的部分关键参数：

- Folder，可以指定为inbox/sent/drafts/deleted/junk
- ME_VALIDATIONTOKEN，需要访问/mewebmail/Mondo/Servlet/request.aspx?Cmd=GET-MBX-OPTIONS&Scope=2，从返回结果中获得
- ID，需要发送查看邮箱文件夹的请求，在返回结果中获得

返回结果为xml格式，包含邮件的详细内容，如果存在附件，那么ATTACHMENTS的EXISTS属性值为1，如果不存在附件，那么ATTACHMENTS的EXISTS属性值为0

MESSAGEID作为附件的标志，如果包含多个附件，多个附件共享同一个MESSAGEID，FILENAME为附件的名称，MESSAGEID+FILENAME作为附件的唯一标志，在下载附件时需要作为参数

为了提高效率，可以使用xml.dom解析xml

xml数据示例如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-7-21/3-3.png)


解析xml提取邮件信息的Python代码如下：

```
DOMTree = minidom.parseString(r.text)
collection = DOMTree.documentElement
element = collection.getElementsByTagName("ELEMENT")
id = element[0].getAttribute("ID")
print("[+] ID      : " + id)
fromaddress = element[0].getElementsByTagName("FROM_ADDRESS")
print("    From    : " + fromaddress[0].childNodes[0].data)
to = element[0].getElementsByTagName("TO")
print("    To      : " + to[0].childNodes[0].data)
subject = element[0].getElementsByTagName("SUBJECT")
print("    Subject : " + subject[0].childNodes[0].data)
received = element[0].getElementsByTagName("RECEIVED")
print("    Received: " + received[0].childNodes[0].data)

body = element[0].getElementsByTagName("BODY")
if len(body[0].childNodes) > 0:
    print("    BODY    : " + body[0].childNodes[0].data)

attachments = element[0].getElementsByTagName("ATTACHMENTS")
exists =  attachments[0].getAttribute("EXISTS")
if exists == "1":
    messageid = attachments[0].getElementsByTagName("MESSAGEID")
    print("    Attachments: " + messageid[0].childNodes[0].data)
    items = attachments[0].getElementsByTagName("ITEM")
    for item in items:
        print(" name:" + item.getElementsByTagName("FILENAME")[0].childNodes[0].data)
        print(" size:" + item.getElementsByTagName("SIZE")[0].childNodes[0].data)
```

### 5.下载附件

访问URL：/MEWebMail/Mondo/lang/sys/Forms/MAI/GetAttachment.aspx

需要的部分关键参数：

- Folder，可以指定为inbox/sent/drafts/deleted/junk
- MessageID，需要发送查看邮件的请求，在返回结果中获得
- Filename，需要发送查看邮件的请求，在返回结果中获得

在保存附件上，需要区分文本格式和二进制格式

## 0x04 开源代码
---

完整代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/MailEnableManage.py

代码支持以下功能：

- GetVersion，版本判断
- Check，登录验证
- ListFolder，查看文件夹，命令行显示邮件数量，完整内容保存至文件
- ViewMail，查看邮件，命令行显示邮件信息，完整内容保存至文件
- DownloadAttachment，下载附件

## 0x05 小结
---

本文介绍了编写Python脚本访问MailEnable邮件的开发细节，开源代码[MailEnableManage.py](https://github.com/3gstudent/Homework-of-Python/blob/master/MailEnableManage.py)，实现了版本判断、登录验证、查看文件夹、查看邮件和下载邮件的功能



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)





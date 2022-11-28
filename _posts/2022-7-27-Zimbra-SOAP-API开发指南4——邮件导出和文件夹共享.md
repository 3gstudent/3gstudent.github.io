---
layout: post
title: Zimbra-SOAP-API开发指南4——邮件导出和文件夹共享
---


## 0x00 前言
---

本文将要继续扩充开源代码[Zimbra_SOAP_API_Manage](https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py)的功能，实现邮件导出和文件夹共享，分享开发细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 邮件导出
- 文件夹共享
- 开源代码

## 0x02 邮件导出
---

Zimbra支持导出当前邮箱的所有邮件，通过Web界面的操作方法如下：

登录邮箱后，依次选择`Preferences`->`Import/Export`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-7-27/2-1.png)

接下来，通过抓包的方式分析实现流程，进而使用程序实现这部分功能

### 1.默认配置导出邮件

默认配置下，会导出所有邮件，以压缩包的形式保存

访问URL示例：

```
https://192.168.1.1/home/admin%40test.com/?fmt=tgz&filename=All-2022-07-27-181056&emptyname=No+Data+to+Export&charset=UTF-8&callback=ZmImportExportController.exportErrorCallback__export1
```

参数解析：

- `admin%40test.com`为邮箱用户，可以用`~`替代
- `filename=All-2022-07-27-181056`为存在记录时保存的文件名，`2022-07-27-181056`对应的时间格式为`年-月-日-时分秒`，时间为带时区的时间，需要计算时差
- `emptyname=No+Data+to+Export`为空记录时保存的文件名

在程序实现上，需要同Web操作的格式保持一致，代码细节：

#### (1)构造保存的文件名

```
from time import localtime, strftime
exporttime = strftime("%Y-%m-%d-%H%M%S", localtime())
filename = "All-" + str(exporttime)
print(filename)
```

#### (2)保存文件

保存文件时使用binary写入

```
with open(path, 'wb+') as file_object:
    file_object.write(r.content)
```

实现代码示例：

```
def exportmailall_request(uri,token,mailbox):
    from time import localtime, strftime
    exporttime = strftime("%Y-%m-%d-%H%M%S", localtime())
    filename = "All-" + str(exporttime)
    url = uri + "/home/" + mailbox + "/?fmt=tgz&filename=" + filename + "&emptyname=No+Data+to+Export&charset=UTF-8&callback=ZmImportExportController.exportErrorCallback__export1"
    headers["Cookie"]="ZM_AUTH_TOKEN="+token+";"
    r = requests.get(url,headers=headers,verify=False)

    if r.status_code == 200:        
        print("[*] Try to export the mail")
        path = filename + ".tgz"        
        with open(path, 'wb+') as file_object:
            file_object.write(r.content)
        print("[+] Save as " + path)
    else:
        print("[!]")
        print(r.status_code)
        print(r.text)
```

### 2.加入筛选条件导出邮件

高级选项下，可以添加筛选条件，导出特定的邮件

访问URL示例：

```
https://192.168.1.1/home/admin%40test.com/?fmt=tgz&start=1658818800000&end=1658991600000&query=content%3Apassword&filename=All-2022-07-27-193148&emptyname=No+Data+to+Export&charset=UTF-8&callback=ZmImportExportController.exportErrorCallback__export1
```

参数解析，新增加了以下参数：

- `start=1658818800000`为筛选的起始时间，格式为unix时间戳，没有额外计算时差
- `end=1658991600000`为筛选的结束时间，格式为unix时间戳，没有额外计算时差
- `query=content%3Apassword`为筛选的关键词，作用是查询正文中带有`password`关键词的邮件

筛选条件的语法可参考：https://wiki.zimbra.com/wiki/Zimbra_Web_Client_Search_Tips

代码实现细节：

#### (1)时间格式转换的示例代码

时间转换成秒：

```
import datetime, time
search1 = datetime.datetime(2022, 7, 26)
search1Toseconds = int(time.mktime(search1.timetuple()))
print(search1Toseconds)
```

秒转换成时间：

```
from datetime import datetime
search1ToDate = str(datetime.fromtimestamp(search1Toseconds))
print(search1ToDate)
```

实现代码示例：

```
def exportmail_request(uri,token,mailbox):
    url = uri + "/home/" + mailbox + "/?fmt=tgz"
    print("[*] Advanced settings")
    print("    You can set the following:")
    print("    - start time:      eg:  2022-06-01")
    print("                       eg:  null")
    print("    - end   time:      eg:  2022-06-02")
    print("    - search fileter:  eg:  content:keyword")
    print("[*] Input the start time:")
    starttime = input("[>]: ")
    
    if len(starttime) != 0:
        if starttime != "null":
            starttimelist = starttime.split('-')
            import datetime, time
            search1 = datetime.datetime(int(starttimelist[0]), int(starttimelist[1]), int(starttimelist[2]))
            search1Toseconds = int(time.mktime(search1.timetuple()))
            print("[*] Input the end time:")
            endtime = input("[>]: ")
            if len(endtime) != 0:
                endtimelist = endtime.split('-')
                search2 = datetime.datetime(int(endtimelist[0]), int(endtimelist[1]), int(endtimelist[2]))
                search2Toseconds = int(time.mktime(search2.timetuple()))
                url = url + "&start=" + str(search1Toseconds) + "000&end=" + str(search2Toseconds) + "000"
            else:
                url = url + "&start=" + str(search1Toseconds) + "000"
        else:
            print("[*] Search all time")    
    else:
        print("[*] Search all time")
    print("[*] Input the search fileter:")
    searchfileter = input("[>]: ")    
    
    from time import localtime, strftime
    exporttime = strftime("%Y-%m-%d-%H%M%S", localtime())
    filename = "All-" + str(exporttime)

    url = url + "&query=" + searchfileter + "&filename=" + filename + "&emptyname=No+Data+to+Export&charset=UTF-8&callback=ZmImportExportController.exportErrorCallback__export1"
    print("[*] Export url:" + url)
    headers["Cookie"]="ZM_AUTH_TOKEN="+token+";"
    r = requests.get(url,headers=headers,verify=False)

    if r.status_code == 200:        
        print("[*] Try to export the mail")
        path = filename + ".tgz"        
        with open(path, 'wb+') as file_object:
            file_object.write(r.content)
        print("[+] Save as " + path)
    else:
        print("[!]")
        print(r.status_code)
        print(r.text)
```

## 0x03 文件夹共享
---

### 1.流程分析

Zimbra支持将当前邮箱的文件夹共享至其他用户，通过Web界面的操作方法如下：

登录邮箱后，依次选择`Preferences`->`Sharing`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-7-27/3-1.png)

文件夹共享可选择以下三个文件夹：

- Inbox
- Sent
- Junk

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-7-27/3-2.png)

设置共享属性如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-7-27/3-3.png)

需要区别以下设置：

#### (1)Role

- `Viewer`只能查看邮件
- `Manager`可以修改邮件

#### (2)Message

- `Send stanard message`，在设置后会向目的邮箱发送一份确认邮件
- `Do not send mail about this share`，不发送确认邮件

这里可以通过抓包分析每项设置对应的具体数值

示例数据包1：

```
<soap:Body>
<BatchRequest xmlns="urn:zimbra" onerror="continue">
<FolderActionRequest xmlns="urn:zimbraMail" requestId="0">
<action op="grant" id="2">
<grant gt="usr" inh="1" d="test1@test.com" perm="r" pw=""/>
</action>
</FolderActionRequest>
</BatchRequest>
</soap:Body>
```

格式分析：

#### (1)`<action op="grant" id="2">`

`id="2"`表示`Inbox`

`Sent`对应`id="5"`

`Junk`对应`id="4"`

通过测试，还可以指定`Drafts`，对应`id="6"`

#### (2)`<grant gt="usr" inh="1" d="test1@test.com" perm="r" pw=""/>`

`d="test1@test.com"`表示可访问共享的邮箱

`perm="r"`表示权限为可读，对应`Viewer`

`Manager`对应的配置为`perm="rwidx"`，表示权限为读、写、添加和删除

如果设置了`Send stanard message`，在设置后会向目的邮箱(例如test1@test.com)发送一份确认邮件，数据包格式示例：

```
<soap:Body>
<SendShareNotificationRequest xmlns="urn:zimbraMail">
<item id="2"/>
<e a="test1@test.com"/>
<notes></notes>
</SendShareNotificationRequest>
</soap:Body>
```

邮箱test1@test.com会收到一份邮件，确认是否接受文件夹共享

### 2.代码实现

#### (1)添加文件共享

需要指定目标邮箱和共享文件夹

添加文件共享成功的响应中返回共享文件夹对应的`zid`

实现代码示例：

```
def addshare_request(uri,token):
    print("[*] Input the target mailbox:")
    mailbox = input("[>]: ")
    print("[*] Input the share folder:")
    print("    2     Inbox")
    print("    4     Junk")
    print("    5     Sent")
    print("    6     Drafts")

    folder = input("[>]: ")

    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
        <soap:Header>
            <context xmlns="urn:zimbra">
                <authToken>{token}</authToken>
            </context>
        </soap:Header>
        <soap:Body>
            <BatchRequest xmlns="urn:zimbra" onerror="continue">
                <FolderActionRequest xmlns="urn:zimbraMail" requestId="0">
                <action op="grant" id="{folder}">
                <grant gt="usr" inh="1" d="{mailbox}" perm="rwidx" pw=""/>
                </action>
                </FolderActionRequest>
            </BatchRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        r=requests.post(uri+"/service/soap",headers=headers,data=request_body.format(token=token,folder=folder,mailbox=mailbox),verify=False,timeout=15)
        if r.status_code == 200 and 'zid' in r.text:        
            pattern_id = re.compile(r"zid=\"(.*?)\"")
            zid = pattern_id.findall(r.text)[0] 
            print("[+] Add success")
            print("    zid: %s"%(zid))
        else:
            print("[!]")
            print(r.status_code)
            print(r.text)

    except Exception as e:
        print("[!] Error:%s"%(e))
```

#### (2)发送文件共享请求

需要指定目标邮箱

实现代码示例：

```
def sendsharenotification_request(uri,token):
    print("[*] Input the target mailbox:")
    mailbox = input("[>]: ")

    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
        <soap:Header>
            <context xmlns="urn:zimbra">
                <authToken>{token}</authToken>
            </context>
        </soap:Header>
        <soap:Body>
            <SendShareNotificationRequest xmlns="urn:zimbraMail">
            <item id="2"/>
            <e a="{mailbox}"/>
            <notes></notes>
            </SendShareNotificationRequest>
        </soap:Body>
    </soap:Envelope>
    """
    try:
        r=requests.post(uri+"/service/soap",headers=headers,data=request_body.format(token=token,mailbox=mailbox),verify=False,timeout=15)
        if r.status_code == 200:   
            print("[+] Send success")
        elif r.status_code == 500 and 'no matching grant' in r.text:
            print("[-] You should add share first.") 

        else:
            print("[!]")
            print(r.status_code)
            print(r.text)

    except Exception as e:
        print("[!] Error:%s"%(e))
```

这里需要注意，只有在添加文件共享后，发送文件共享请求才能成功返回200，否则返回500，提示`invalid request: no matching grant`

#### (3)删除文件共享

需要指定目标邮箱对应的`zid`和共享文件夹，`zid`可在添加文件共享成功的响应中获得

实现代码示例：

```
def removeshare_request(uri,token):
    print("[*] Input the zid:")
    zid = input("[>]: ")
    print("[*] Input the share folder:")
    print("    2     Inbox")
    print("    4     Junk")
    print("    5     Sent")
    print("    6     Drafts")

    folder = input("[>]: ")

    request_body="""<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
        <soap:Header>
            <context xmlns="urn:zimbra">
                <authToken>{token}</authToken>
            </context>
        </soap:Header>
        <soap:Body>
            <FolderActionRequest xmlns="urn:zimbraMail">
            <action op="!grant" id="{folder}" zid="{zid}"/>
            </FolderActionRequest>
       </soap:Body>
    </soap:Envelope>
    """
    try:
        r=requests.post(uri+"/service/soap",headers=headers,data=request_body.format(token=token,folder=folder,zid=zid),verify=False,timeout=15)
        if r.status_code == 200:        
            print("[+] Send success") 
        else:
            print("[!]")
            print(r.status_code)
            print(r.text)

    except Exception as e:
        print("[!] Error:%s"%(e))
```

## 0x04 开源代码
---

新的代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Zimbra_SOAP_API_Manage.py

添加以下五个功能：

- AddShare：添加文件夹共享，默认权限为`rwidx`
- ExportMail：导出带有搜索条件的邮件，可指定日期和关键词
- ExportMailAll：导出所有邮件
- RemoveShare：删除当前邮箱的文件夹共享
- SendShareNotification：在添加文件夹共享后，向目标邮箱发送一封确认邮件

## 0x05 小结
---

本文扩充了Zimbra SOAP API的调用方法，添加五个实用功能，实现方法和思路还可在XSS漏洞上进行测试。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)

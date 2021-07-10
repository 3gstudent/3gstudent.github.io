---
layout: post
title: SolarWinds Orion开发指南
---


## 0x00 前言
---

SolarWinds Orion平台是一个统一的网络和系统管理产品套件，可用于监控IT基础架构。我们可以通过SolarWinds Information Service (SWIS)访问Orion平台中的数据。

在程序实现上，我们可以借助SolarWinds Orion API进行开发，但是在最近的漏洞利用上，我们无法直接使用SolarWinds Orion API。

本文将要介绍SolarWinds Orion API的用法，分析无法直接使用的原因，提供一种解决方法，开源两个测试代码

### 0x01 简介
---

本文将要介绍以下内容：

- SolarWinds Orion API的使用
- 模拟网页操作的实现
- 开发细节
- 开源代码


### 0x02 SolarWinds Orion API的使用
---

参考资料：

https://github.com/solarwinds/OrionSDK/wiki

Python语言可使用orionsdk库进行开发，地址如下：

https://github.com/solarwinds/orionsdk-python

在引入orionsdk库后，可以很容易的实现以下功能：

- query
- invoke
- create
- read
- update
- bulkupdate
- delete
- bulkdelete

为了研究SolarWinds Orion API的实现细节，决定不借助orionsdk库实现相同的功能

语法格式的参考资料：

https://github.com/solarwinds/OrionSDK/wiki/REST

对于SolarWinds Orion API，需要注意以下细节：

#### 1.接口地址

默认接口地址为`https://<url>:17778/SolarWinds/InformationService/v3/Json/`

#### 2.用户验证

添加Header： `Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=`

其中，`dXNlcm5hbWU6cGFzc3dvcmQ`为`username:password`作Base64编码后的结果

#### 3.数据查询

通过POST发送查询命令，格式为`application/json`类型

SolarWinds Orion API使用SolarWinds Query Language (SWQL)，类似于SQL语法

数据库的表项可以通过本地搭建测试环境，执行SolarWinds Orion下的DataBase Manager进行查看

查询数据库的示例代码：

```
def SWIS_query(api_host, username, password, query, **params):
    authentication = username + ":" + password
    authentication = authentication.encode("utf-8")
    credential = base64.b64encode(authentication).decode("utf8")
    url = "https://" + api_host + ":17778/SolarWinds/InformationService/v3/Json/Query"
    headers = {
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0",
        "Authorization": "Basic " + credential,
        "Content-Type": "application/json"
    }

    data =  {
                "query": query,
                "parameters":params   
            }
            
    r = requests.post(url, headers = headers, data=json.dumps(data), verify = False)
    if r.status_code ==200: 
        print("[+] query success")
        for i in r.json()["results"]:
            print(i)
             
    else:         
        print("[!]")
        print(r.status_code)
        print(r.text)
        exit(0) 
```
        

在Python代码开发上，需要考虑以下细节：

#### 1.将字典作为命令行参数传递

可以先通过命令行参数传入字符串，再将字符串转义为json字符串

固定参数的用法示例：

```
entity = "Orion.Pollers"
properties = {"PollerType":"hi from curl 2", "NetObject":"N:123"}
SWIS_create(api_host, username, password, entity, **params)
```

将字典作为命令行参数传递的用法示例：

```
entity = "Orion.Pollers"
properties = input("input the properties: (eg. {\"PollerType\":\"hi from curl 2\", \"NetObject\":\"N:123\"} )")
SWIS_create(sys.argv[1], sys.argv[2], sys.argv[3], entity, **json.loads(properties))
```

#### 2.将列表作为命令行参数传递

可以先通过命令行参数传入字符串，再将字符串转为列表

固定参数的用法示例：

```
uris = ["swis://WIN-KQ48K3S9B92/Orion/Orion.Nodes/NodeID=1/CustomProperties"]
properties = {"City": "Serenity Valley"}
SWIS_bulkupdate(api_host, username, password, uris, **properties)
```

将列表作为命令行参数传递的用法示例：

```
uris = input("input the uris: (eg. swis://Server1/Orion/Orion.Nodes/NodeID=1/CustomProperties,swis://Server1/Orion/Orion.Nodes/NodeID=2/CustomProperties )")
properties = {"City": "Serenity Valley"}
SWIS_bulkupdate(sys.argv[1], sys.argv[2], sys.argv[3], uris.split(','), **properties)
```

完整的实现代码已上传至Github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/SolarWindsOrionAPI_Manage.py

代码支持以下功能：

- query
- invoke
- create
- read
- update
- bulkupdate
- delete
- bulkdelete

为了便于使用，省去输入查询语句的过程，还支持以下功能：

- GetAccounts
- GetAlertActive
- GetAlertHistory
- GetCredential
- GetNodes
- GetOrionServers

猜测是出于安全考虑，SolarWinds Orion API的功能有限，有些数据库无法进行查询，例如VirtualMachines表(存储虚拟机信息)、CredentialProperty表(存储凭据信息)、Accounts表的PasswordHash项和PasswordSalt项

已公开的SolarWinds漏洞(CVE-2020-10148、CVE-2020-27870、CVE-2020-27871、CVE-2021-31474)涉及的均为网络协议接口(默认为`http://<ip>:8787/Orion/`)，同SolarWinds Orion API不同，所以无法结合利用


### 0x03 模拟网页操作的实现
---

为了结合漏洞利用，我们需要网页登录，通过抓取数据包的方式实现同SolarWinds Orion的数据交互，功能同SolarWinds Orion API的功能保持一致

#### 1.登录验证

通过抓取数据包发现，SolarWinds Orion基于ASP.NET平台使用了ViewState存储数据

但经过实际测试，我们的测试程序可以不带有ViewState，不影响功能

登录验证的数据包流程如下：

1. 访问：`http://<ip>:8787/Orion/Login.aspx?autologin=no`
2. 返回响应码为302
3. 自动跳转至`http://<ip>:8787/Orion/View.aspx`
4. 返回响应码为302
5. 自动跳转至`http://<ip>:8787/Orion/SummaryView.aspx`
6. 返回响应码为200，获得最终结果

在程序实现上，我们可以对返回的最终结果进行判断，如果Cookie中带有`__AntiXsrfToken`项，那么代表用户验证成功

示例代码：

```
def Check(api_host, username, password):
    url = api_host + "/Orion/Login.aspx?autologin=no"

    body = {
            "__EVENTTARGET": "ctl00$BodyContent$LoginButton",
            "ctl00$BodyContent$Username": username,
            "ctl00$BodyContent$Password": password
            }
    
    postData = urllib.parse.urlencode(body).encode("utf-8")
            
    headers = {
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0",
        "Content-Type": "application/x-www-form-urlencoded",    
    }
          
    r = requests.post(url, headers = headers, data=postData, verify = False)
    if r.status_code ==200 and "__AntiXsrfToken" in r.headers['set-cookie']:
        print("[+] Valid:%s  %s"%(username, password))
        r.close()
    else:         
        print("[!]")
        print(r.status_code)
        print(r.text)
        r.close()
        exit(0) 
```

#### 2.查询操作

Header中需要额外添加属性`X-XSRF-TOKEN`

属性`X-XSRF-TOKEN`的值在初次登录时访问`http://<ip>:8787/Orion/Login.aspx?autologin=no`返回的302结果中获得

在程序实现上，可以通过添加参数`allow_redirects=Faslse`来禁用跳转，在返回的Cookie中取出`X-XSRF-TOKEN`

查询接口地址：`http://<ip>:8787/api2/swis/query`


示例代码：

```
def QueryData(api_host, username, password, query, **params):
    session = requests.session()
    url = api_host + "/Orion/Login.aspx?autologin=no"

    body = {
            "__EVENTTARGET": "ctl00$BodyContent$LoginButton",
            "ctl00$BodyContent$Username": username,
            "ctl00$BodyContent$Password": password
            }
    
    postData = urllib.parse.urlencode(body).encode("utf-8")
            
    headers = {
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0",
        "Content-Type": "application/x-www-form-urlencoded",    
    }
          
    r = session.post(url, headers = headers, data=postData, verify = False)
    if r.status_code !=200 or "__AntiXsrfToken" not in r.headers['set-cookie']:
        print("[!]")
        print(r.status_code)
        print(r.text)
        r.close()
        exit(0)

    print("[+] Valid:%s  %s"%(username, password))

    r = session.post(url, headers = headers, data=postData, verify = False, allow_redirects=False)
    index = r.headers['Set-Cookie'].index('XSRF-TOKEN')
    xsrfToken = r.headers["Set-Cookie"][index+11:index+55]
    print("[+] XSRF-TOKEN: " + xsrfToken)
    
    url1 = api_host + "/api2/swis/query?lang=en-us&swAlertOnError=false"
    headers = {
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0",
        "Content-Type": "application/json",
        "X-XSRF-TOKEN":xsrfToken
    }

    data =  {
                "query": query,
                "parameters":params   
            }
    
    r = session.post(url1, headers = headers, data=json.dumps(data), verify = False)
    if "Result" in r.json():
        print("[+] Result: ")
        dic = r.json()['Result']
        for i in dic:       
            print(i)
    else:
        print("[!]")
        print(r.json())
```

数据格式同SolarWinds Orion API的query命令保持一致，所以我们可以直接对照SolarWinds Orion AP实现相同的功能

完整的实现代码已上传至Github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/SolarWindsOrionAPI_Manage.py

代码支持用户口令验证和数据库查询的功能

为了便于使用，省去输入查询语句的过程，支持以下功能：

- GetAccounts
- GetAlertActive
- GetAlertHistory
- GetCredential
- GetNodes
- GetOrionServers

### 0x04 小结
---

本文分别介绍了使用SolarWinds Orion API和模拟网页操作实现数据查询的方法，开源测试代码，便于同其他漏洞利用相结合。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)







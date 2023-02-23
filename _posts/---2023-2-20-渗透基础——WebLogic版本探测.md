---
layout: post
title: 渗透基础——WebLogic版本探测
---


## 0x00 前言
---

本文将要介绍WebLogic版本探测的两种方法，通过Python实现自动化，记录开发细节，开源代码。

## 0x01 简介
---

本文将要介绍以下内容：

- 实现思路
- 实现细节
- 开源代码

## 0x02 实现思路
---

探测WebLogic版本的方法有以下两种：

### 1.通过Web页面WebLogic Admin Console

默认配置下的URL:`http://<ip>:7001/console/login/LoginForm.jsp`

在返回结果中能够获得WebLogic的版本

这里需要注意以下问题：

#### (1)需要区别早期版本

早期版本的返回结果示例：`<TITLE>WebLogic Server 8.1 - Console Login</TITLE>`

目前常用版本的返回结果示例：`<p id="footerVersion">WebLogic Server Version: 14.1.1.0.0</p>`

#### (2)WebLogic Admin Console对应的路径和端口可被修改

WebLogic Admin Console可被关闭，也可修改URL，修改方法有以下两种：

通过浏览器访问WebLogic Admin Console，在`Configuration`->`General`->`Advanced`设置，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-2-20/2-1.png)

通过配置文件设置，默认路径：`Oracle_Home\user_projects\domains\base_domain\config\config.xml`，内容如下：

```
  <console-enabled>true</console-enabled>
  <console-context-path>console</console-context-path>
  <console-extension-directory>console-ext</console-extension-directory>
```

#### (3)关闭WebLogic Admin Console的情况

如果关闭了WebLogic Admin Console，访问URL:`http://<ip>:7001/console/login/LoginForm.jsp`，返回内容带有字符串`10.4.5 404 Not Found`，可作为判断使用了WebLogic的依据

### 2.通过T3协议

可以使用nmap的脚本weblogic-t3-info.nse，命令示例：

```
nmap -n -v -Pn -sV 192.168.1.1 -p7001 -script=weblogic-t3-info.nse
```

返回结果示例：

```
PORT     STATE    SERVICE       VERSION
7001/tcp open     ldap          WebLogic application server 14.1.1.0 (T3 enabled)
|_weblogic-t3-info: T3 protocol in use (WebLogic version: 14.1.1.0)
```

在原理上是通过建立socket连接，在返回结果中获得WebLogic的版本

这里需要注意以下问题：

#### (1)需要区别早期版本

早期版本的返回结果示例：`t3 10.3.6.0\nAS:2048\nHL:19\n\n`

目前常用版本的返回结果示例：`HELO:12.2.1.3.0.false\nAS:2048\nHL:19\nMS:10000000\nPN:DOMAIN\n\n`

#### (2)存在需要多次发送的情况

存在特殊情况，返回内容为`HELO`，此时需要重新发送直到返回完整的版本信息

#### (3)T3协议可被关闭

关闭方法有以下两种：

通过浏览器访问WebLogic Admin Console，在`Security`->`Filter`设置，配置如下：

`Connection Filter`设置为`weblogic.security.net.ConnectionFilterImpl`

`Connection Filter Rules`设置为：

```
127.0.0.1 * * allow t3 t3s
* * * deny t3 t3s
```

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-2-20/2-2.png)

通过配置文件设置，默认路径：`Oracle_Home\user_projects\domains\base_domain\config\config.xml`，内容如下：

```
    <connection-filter>weblogic.security.net.ConnectionFilterImpl</connection-filter>
    <connection-filter-rule>127.0.0.1 * * allow t3 t3s</connection-filter-rule>
    <connection-filter-rule>* * * deny t3 t3s</connection-filter-rule>
```


## 0x03 实现细节
---

综合以上探测方法，为了适应多种环境，在程序实现上选取了通过HTTP协议和T3协议两种方法实现

### 1.通过HTTP协议

选择默认配置下的URL:`http://<ip>:7001/console/login/LoginForm.jsp`

需要注意以下问题:

#### (1)第一次访问时存在一次跳转

首次启动WebLogic时，在访问默认配置下的URL:`http://<ip>:7001/console/login/LoginForm.jsp`时存在一次跳转，需要等待一段时间

在返回内容中以字符串`Deploying application`作为判断依据

#### (2)需要区别早期版本

早期版本的返回结果示例：`<TITLE>WebLogic Server 8.1 - Console Login</TITLE>`

目前常用版本的返回结果示例：`<p id="footerVersion">WebLogic Server Version: 14.1.1.0.0</p>`

在脚本实现上优先判断常用版本，使用正则匹配，如果失败，再从固定格式`<TITLE>`直接提取早期版本，示例代码如下：

```
try:
    versiondata = re.search('[0-9.]{8}', response.text)
    version = versiondata.group(0)    
    print("[+] Admin Console Version: " + version)
except Exception as e:
    print("    Possibly an earlier version")
    versiondata = re.compile(r"TITLE\>(.*?)\<")
    version = versiondata.findall(response.text)       
    print("[+] " + version[0])
```

#### (3)关闭WebLogic Admin Console的识别

如果关闭了WebLogic Admin Console，访问URL:`http://<ip>:7001/console/login/LoginForm.jsp`，返回内容带有字符串`10.4.5 404 Not Found`，可作为判断使用了WebLogic的依据，此时无法获得WebLogic的准确版本

完整示例代码如下：

```
def getversion_AdminConsole(ip, port):
    try:  
        url = "http://" + ip + ":" + str(port) + "/console/login/LoginForm.jsp"
        print("\n[*] Try to access Admin Console: " + url)
        headers={
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.190 Safari/537.36",
        }
        response = requests.get(url, headers=headers, verify=False, timeout=5)
        if response.status_code == 200 and 'Deploying application' in response.text:
            print("    Wait 3 seconds to deploy the application...")
            time.sleep(3)
            response = requests.get(url, headers=headers, verify=False, timeout=8)
        if response.status_code == 200 and 'WebLogic' in response.text:
            print("    Success")
            try:
                versiondata = re.search('[0-9.]{8}', response.text)
                version = versiondata.group(0)    
                print("[+] Admin Console Version: " + version)
            except Exception as e:
                print("    Possibly an earlier version")
                versiondata = re.compile(r"TITLE\>(.*?)\<")
                version = versiondata.findall(response.text)       
                print("[+] " + version[0])

        elif response.status_code == 404 and '10.4.5 404 Not Found' in response.text:
            print("    Weblogic detected, but unable to get version.")
            print("    It is possible to disable the Admin Console, or to modify the configuration of the Admin Console.")
        else:
            print("[-]")
            print(response.status_code)
            print(response.text)
    except Exception as e: 
        print(e)
```

### 2.通过T3协议

发送的socket数据内容为：`t3 12.1.2\nAS:2048\nHL:19\n\n`

需要注意以下问题:

#### (1)需要区别早期版本

早期版本的返回结果示例：`t3 10.3.6.0\nAS:2048\nHL:19\n\n`

目前常用版本的返回结果示例：`HELO:12.2.1.3.0.false\nAS:2048\nHL:19\nMS:10000000\nPN:DOMAIN\n\n`

为了提高准确性，这里使用正则提取版本信息，示例代码：

```
versiondata = re.search('[0-9.]{8}', response.decode('UTF-8'))
version = versiondata.group(0)
```

#### (2)存在需要多次发送的情况

存在特殊情况，返回内容为`HELO`，此时需要重新发送直到返回正确的版本信息

在重新发送的过程中，应关闭整个socket连接，重新初始化发送数据

#### (3)T3协议可被关闭

如果关闭了T3协议，返回内容示例：

```
LGIN:[Socket:000445]Connection rejected, filter blocked Socket, weblogic.security.net.FilterException: [Security:090220]rule 2\nweblogic.security.net.FilterException: [Security:090220]rule 2\r\n\tat weblogic.security.net.ConnectionFilterImpl.accept(ConnectionFilterImpl.java:163)\r\n\tat weblogic.socket.MuxableSocketDiscriminator.maybeFilter(MuxableSocketDiscriminator.java:253)\r\n\tat weblogic.socket.MuxableSocketDiscriminator.dispatch(MuxableSocketDiscriminator.java:139)\r\n\tat weblogic.socket.SocketMuxer.readReadySocketOnce(SocketMuxer.java:981)\r\n\tat weblogic.socket.SocketMuxer.readReadySocket(SocketMuxer.java:917)\r\n\tat weblogic.socket.NIOSocketMuxer.process(NIOSocketMuxer.java:599)\r\n\tat weblogic.socket.NIOSocketMuxer.processSockets(NIOSocketMuxer.java:563)\r\n\tat weblogic.socket.SocketReaderRequest.run(SocketReaderRequest.java:30)\r\n\tat weblogic.socket.SocketReaderRequest.execute(SocketReaderRequest.java:43)\r\n\tat weblogic.kernel.ExecuteThread.execute(ExecuteThread.java:147)\r\n\tat weblogic.kernel.ExecuteThread.run(ExecuteThread.java:119)\r\n\n
```

完整示例代码如下：

```
def getversion_T3(ip, port):
    try:
        print("[*] Try to use T3: " + ip + ":" + str(port))
        for i in range(5):
            s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
            s.settimeout(8)
            s.connect((ip, port))
            s.sendall('t3 12.1.2\nAS:2048\nHL:19\n\n'.encode())
            response = s.recv(2048)
            s.close()
            if response.decode('UTF-8') == "HELO":
                print("    T3 is turned on, send data again to get the version")
                continue
            elif "AS" in response.decode('UTF-8') or "HELO" in response.decode('UTF-8'):
                print("    OK")
                versiondata = re.search('[0-9.]{8}', response.decode('UTF-8'))
                version = versiondata.group(0)
                print("[+] T3 Version: " + version)
                break
            else:
                print("[-]")
                print(response.decode('UTF-8'))
                s.close()
                break      
    except Exception as e:
        print(e)
```


## 0x04 开源代码
---

完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/WebLogic_GetVersion.py

代码使用HTTP协议和T3协议探测版本信息

## 0x05 小结
---

本文介绍了WebLogic版本探测的两种方法，比较优缺点，选取有效的方法并通过Python实现自动化，记录开发细节，开源代码，作为一个很好的学习示例。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


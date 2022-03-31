---
layout: post
title: Windows Communication Foundation开发指南2——禁用元数据发布
---


## 0x00 前言
---

本文将要介绍在禁用元数据发布(MEX)时WCF开发的相关内容，给出文章[《Abusing Insecure Windows Communication Foundation (WCF) Endpoints》](https://versprite.com/blog/security-research/abusing-insecure-wcf-endpoints/)的完整代码示例。

### 0x01 简介
---

本文将要介绍以下内容：

- 禁用MEX实现WCF
- 文章完整代码示例

## 0x02 禁用MEX实现WCF
---

禁用MEX时，需要根据服务端的代码手动编写客户端代码

本节采用命令行实现WCF的方式作为示例

开发工具：Visual Studio 2015

### 1.服务端编写

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为NBTServer

#### (2)新建WCF服务

选择`Add`->`New Item...`，选择`WCF Service`，名称为Service1.cs

#### (3)修改service1.cs

添加DoWork的实现代码，代码示例：

```
namespace NTBServer
{
    public class Service1 : IService1
    {
        public void DoWork()
        {
            System.Diagnostics.Process.Start("calc.exe");
        }
    }
}
```

#### (4)修改Program.cs

添加引用`System.ServiceModel`

添加启动代码，代码示例：

```
using System;
using System.ServiceModel;
namespace basicHttpBindingWCFServer
{
    class Program
    {
        static void Main(string[] args)
        {
            ServiceHost Host = new ServiceHost(typeof(Service1));
            Host.Open();
            Console.WriteLine(Host.Description.Endpoints[0].Address);
            Console.ReadLine();
            Host.Close();
        }
    }
}
```

#### (5)修改App.config

```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.6.1" />
  </startup>
</configuration>
```

#### (6)编译运行

命令行输出地址：net.tcp://localhost/vulnservice/runme

#### (7)测试

此时无法使用WcfTestClient进行测试

### 2.客户端编写

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为NBTClient

#### (2)修改Program.cs

添加引用`System.ServiceModel`

代码示例：

```
using System.ServiceModel;
namespace NTBClient
{
    [ServiceContract]
    public interface IService1
    {
       [OperationContract]
       void DoWork();
    }
    class Program
    {
        static void Main(string[] args)
        {
            string address = $"net.tcp://localhost/vulnservice/runme";
            ChannelFactory<IService1> factory = new ChannelFactory<IService1>(new NetTcpBinding(), address);
            IService1 proxy = factory.CreateChannel();
            proxy.DoWork();
            factory.Close();
        }
    }
}
```

## 0x03 文章完整代码示例
---

代码示例：

https://github.com/VerSprite/research/tree/master/projects/wcf/VulnWCFService

相关介绍：

https://versprite.com/blog/security-research/abusing-insecure-wcf-endpoints/

代码示例实现了WCF的服务端，但是缺少安装部分和客户端的编写，这里给出完整示例

### 1.服务端编写

#### (1)下载代码

https://github.com/VerSprite/research/tree/master/projects/wcf/VulnWCFService

#### (2)新建Windows Service

选择`Add`->`New Item...`，选择`Windows Service`，名称为Service1.cs

#### (3)设置服务信息

选中`Service1.cs`，`右键`->`Add Installer`

项目中自动创建ProjectInstaller.cs文件，该文件会添加俩个组件serviceProcessInstaller1和serviceInstaller1

选中serviceProcessInstaller1组件，查看属性，设置account为LocalSystem

选中serviceInstaller1组件，查看属性，设置ServiceName为VulService1


#### (4)启动服务

编译生成VulnWCFService.exe

安装服务：

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil VulnWCFService.exe
```

启动服务：

```
sc start VulService1
```

**补充：卸载服务**

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil /u WCFService.exe
```

### 2.客户端编写

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为VulnWCFClient

#### (2)修改Program.cs

添加引用`System.ServiceModel`

代码示例：

```
using System.ServiceModel;
namespace VulnWCFClient
{
    [ServiceContract]
    public interface IVulnService
    {
        [OperationContract]
        void RunMe(string str);
    }
    class Program
    {
        static void Main(string[] args)
        {
            string address = $"net.pipe://localhost/vulnservice/runme";
            ChannelFactory<IVulnService> factory = new ChannelFactory<IVulnService>(new NetNamedPipeBinding(), address);
            IVulnService proxy = factory.CreateChannel();
            proxy.RunMe("calc.exe");
            factory.Close();
        }
    }
}
```

#### (3)编译运行

编译生成VulnWCFClient，运行后弹出System权限的计算器，测试成功

## 0x04 小结
---

本文介绍了禁用元数据发布(MEX)时WCF开发的相关内容，给出文章[《Abusing Insecure Windows Communication Foundation (WCF) Endpoints》](https://versprite.com/blog/security-research/abusing-insecure-wcf-endpoints/)的完整代码示例，便于WCF相关知识的研究。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




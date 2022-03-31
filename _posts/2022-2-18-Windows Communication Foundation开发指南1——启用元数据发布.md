---
layout: post
title: Windows Communication Foundation开发指南1——启用元数据发布
---


## 0x00 前言
---

Windows Communication Foundation (WCF)是用于在.NET Framework中构建面向服务的应用程序的框架。本文将要介绍WCF开发的相关内容，为后续介绍的内容作铺垫。

## 0x01 简介
---

本文将要介绍以下内容：

- 使用basicHttpBinding实现WCF
- 使用NetTcpBinding实现WCF
- 通过命令行实现WCF
- 通过IIS实现WCF
- 通过服务实现WCF

## 0x02 基础知识
---

参考资料：

https://docs.microsoft.com/en-us/dotnet/framework/wcf/whats-wcf

常用的传输协议：

- HTTP，http://localhost:8080/
- TCP，net.tcp://localhost:8080/
- IPC，net.pipe://localhost/

常用的Binding：

- BasicHttpBinding
- WSHttpBinding
- NetTcpBinding
- NetNamedPipeBinding

元数据发布(metadata exchange)，简称MEX

WCF默认禁用MEX，这样能够避免数据泄露

本着逐步深入的原则，本系列文章选择先介绍开启MEX的用法，这样能够提高客户端开发的效率，禁用MEX的用法将放在下篇文章进行介绍。

## 0x03 使用basicHttpBinding实现WCF
---

本节采用命令行实现WCF的方式作为示例

开发工具：Visual Studio 2015

### 1.服务端编写

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为basicHttpBindingWCFServer

#### (2)新建WCF服务

选择`Add`->`New Item...`，选择`WCF Service`，名称为Service1.cs

#### (3)修改service1.cs

添加DoWork的实现代码，代码示例：

```
using System;
namespace basicHttpBindingWCFServer
{
    public class Service1 : IService1
    {
        public void DoWork()
        {
            Console.Write("Run Server.DoWork()");
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

#### (5)编译运行

命令行输出服务地址：http://localhost:8733/Design_Time_Addresses/basicHttpBindingWCFServer/Service1/

服务地址也可以在工程中的`App.config`查看

#### (6)测试

此时开启了MEX，可选择以下方法进行测试：

- 通过浏览器访问服务地址：http://localhost:8733/Design_Time_Addresses/basicHttpBindingWCFServer/Service1/，能够返回服务信息
- 使用WcfTestClient进行测试，默认路径：`C:\Program Files(x86)\Microsoft Visual Studio 14\Common7\IDE\WcfTestClient.exe`，连接http://localhost:8733/Design_Time_Addresses/basicHttpBindingWCFServer/Service1/，调用`DoWork()`，此时服务端命令行输出`Run Server.DoWork()`，方法调用成功
- 使用Svcutil生成客户端配置代码，命令示例：`svcutil.exe http://localhost:8733/Design_Time_Addresses/basicHttpBindingWCFServer/Service1/ /out:1.cs`，相关代码可参考：https://github.com/dotnet/samples/tree/main/framework/wcf/Basic/Binding/Net/Tcp/Default/CS

**注：**

App.config由Visual Studio自动生成，服务地址由App.config随机指定，这里也可以通过代码的方式指定服务地址，不需要依赖App.config，方法如下：

Program.cs示例：

```
using System;
using System.ServiceModel;
using System.ServiceModel.Description;
namespace basicHttpBindingWCFServer
{
    class Program
    {
        static void Main(string[] args)
        {
            ServiceHost host = null;
            try
            {            
                Uri baseAddress = new Uri("http://localhost/TestService");
                host = new ServiceHost(typeof(Service1), baseAddress);
                BasicHttpBinding binding = new BasicHttpBinding();          
                host.AddServiceEndpoint(typeof(IService1), binding, baseAddress);
                if (host.Description.Behaviors.Find<ServiceMetadataBehavior>() == null)
                {
                    ServiceMetadataBehavior behavior = new ServiceMetadataBehavior();
                    behavior.HttpGetEnabled = true;                    
                    host.Description.Behaviors.Add(behavior);
                }
                host.Open();
                Console.WriteLine(host.Description.Endpoints[0].Address);
                Console.Read();
                host.Close();      
            }
            catch (CommunicationException ce)
            {
                host.Abort();
            }
        }
    }
}
```

App.config示例：

```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.6.1" />
  </startup>
</configuration>
```

### 2.客户端编写

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为basicHttpBindingWCFClient

#### (2)引用服务

选择`Add`->`Service Reference...`

填入URL：http://localhost:8733/Design_Time_Addresses/basicHttpBindingWCFServer/Service1/

#### (3)修改Program.cs

代码示例：

```
using basicHttpBindingWCFClient.ServiceReference1;
namespace basicHttpBindingWCFClient
{
    class Program
    {
        static void Main(string[] args)
        {
            Service1Client service1 = new Service1Client();
            service1.DoWork();
        }
    }
}
```

#### (4)编译运行

此时服务端命令行输出`Run Server.DoWork()`，方法调用成功

## 0x04 使用NetTcpBinding实现WCF
---

本节采用命令行实现WCF的方式作为示例

### 1.服务端编写

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为NetTcpBindingWCFServer

#### (2)新建WCF服务

选择`Add`->`New Item...`，选择`WCF Service`，名称为Service1.cs

#### (3)修改service1.cs

添加DoWork的实现代码，代码示例：

```
using System;
namespace NetTcpBindingWCFServer
{
    public class Service1 : IService1
    {
        public void DoWork()
        {
            Console.Write("Run Server.DoWork()");
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
namespace NetTcpBindingWCFServer
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

Line10：`<serviceMetadata httpGetEnabled="true" httpsGetEnabled="true" />`

修改为：`<serviceMetadata httpGetEnabled="false" httpsGetEnabled="false" />`
                    
Line17：`<endpoint address="" binding="basicHttpBinding" contract="NetTcpBindingWCFServer.IService1">`
                
修改为：`<endpoint address="" binding="netTcpBinding" contract="NetTcpBindingWCFServer.IService1">`
                
Line22:`<endpoint address="mex" binding="mexHttpBinding" contract="IMetadataExchange" />`

修改为：`<endpoint address="mex" binding="mexTcpBinding" contract="IMetadataExchange" />`

Line25:`<add baseAddress="http://localhost:8733/Design_Time_Addresses/NetTcpBindingWCFServer/Service1/" />`

修改为：`<add baseAddress="net.tcp://localhost:1111/Design_Time_Addresses/NetTcpBindingWCFServer/Service1/" />`


完整代码示例：

```
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <startup> 
        <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.5.2" />
    </startup>
    <system.serviceModel>
        <behaviors>
            <serviceBehaviors>
                <behavior name="">
                    <serviceMetadata httpGetEnabled="false" httpsGetEnabled="false" />
                    <serviceDebug includeExceptionDetailInFaults="false" />
                </behavior>
            </serviceBehaviors>
        </behaviors>
        <services>
            <service name="NetTcpBindingWCFServer.Service1">
                <endpoint address="" binding="netTcpBinding" contract="NetTcpBindingWCFServer.IService1">
                    <identity>
                        <dns value="localhost" />
                    </identity>
                </endpoint>
                <endpoint address="mex" binding="mexTcpBinding" contract="IMetadataExchange" />
                <host>
                    <baseAddresses>
                        <add baseAddress="net.tcp://localhost:1111/Design_Time_Addresses/NetTcpBindingWCFServer/Service1/" />
                    </baseAddresses>
                </host>
            </service>
        </services>
    </system.serviceModel>
</configuration>
```

#### (6)编译运行

命令行输出服务地址：net.tcp://localhost:1111/Design_Time_Addresses/NetTcpBindingWCFServer/Service1/

#### (7)测试

此时开启了MEX，可选择以下方法进行测试：

- 使用WcfTestClient进行测试，默认路径：`C:\Program Files(x86)\Microsoft Visual Studio 14\Common7\IDE\WcfTestClient.exe`，连接net.tcp://localhost:1111/Design_Time_Addresses/NetTcpBindingWCFServer/Service1/，调用`DoWork()`，此时服务端命令行输出`Run Server.DoWork()`，方法调用成功
- 使用Svcutil生成客户端配置代码，命令示例：`svcutil.exe net.tcp://localhost:1111/Design_Time_Addresses/NetTcpBindingWCFServer/Service1/ /out:1.cs`，相关代码可参考：https://github.com/dotnet/samples/tree/main/framework/wcf/Basic/Binding/Net/Tcp/Default/CS

### 2.客户端编写

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为NetTcpBindingWCFClient

方法同0x03中的2.客户端编写

## 0x05 通过IIS实现WCF
---

本节仅以服务端编写作为示例，客户端编写同命令行实现的方法一致

### 1.服务端编写

#### (1)新建项目

选择`Visual C#`->`WCF`->`WCF Service Library`，名称为WcfServiceLibrary1

#### (2)发布

选中项目，`右键`->`Publish...`，设置Target location为`c:\wcfdemo`

#### (3)在IIS管理页面下新建网站

设置以下参数：
- Site name:wcfdemo
- Physical path:c:\wcfdemo
- IP address: All unassigned
- Port:81

选中网站wcfdemo，进入`Content View`

选中`WcfServiceLibrary1.Service1.svc`，`右键`->`Browse`，得到URL：http://localhost:81/WcfServiceLibrary2.Service1.svc

#### (4)测试

此时开启了MEX，可选择以下方法进行测试：

- 通过浏览器访问服务地址：http://localhost:81/WcfServiceLibrary2.Service1.svc，能够返回服务信息
- 使用WcfTestClient进行测试，默认路径：`C:\Program Files(x86)\Microsoft Visual Studio 14\Common7\IDE\WcfTestClient.exe`，连接http://localhost:81/WcfServiceLibrary2.Service1.svc，调用`GetData()`，获得返回值，方法调用成功
- 使用Svcutil生成客户端配置代码，命令示例：`svcutil.exe http://localhost:81/WcfServiceLibrary2.Service1.svc`，相关代码可参考：https://github.com/dotnet/samples/tree/main/framework/wcf/Basic/Binding/Net/Tcp/Default/CS

## 0x06 通过服务实现WCF
---

本节仅以服务端编写作为示例，客户端编写同命令行实现的方法一致


### 1.使用basicHttpBinding实现服务端

#### (1)新建项目

选择`Visual C#`->`Console Application`，名称为WCFService

#### (2)新建Windows Service

选择`Add`->`New Item...`，选择`Windows Service`，名称为Service1.cs

#### (3)设置服务信息

选中`Service1.cs`，`右键`->`Add Installer`

项目中自动创建ProjectInstaller.cs文件，该文件会添加俩个组件serviceProcessInstaller1和serviceInstaller1

选中serviceProcessInstaller1组件，查看属性，设置account为LocalSystem

选中serviceInstaller1组件，查看属性，设置ServiceName为VulServiceTest1

#### (4)编辑Program.cs

```
using System;
using System.ServiceModel;
using System.ServiceProcess;
using System.ServiceModel.Description;
namespace WCFService
{
    [ServiceContract]
    public interface IVulnService
    {
        [OperationContract]
        void RunMe(string str);
    }

    public class VulnService : IVulnService
    {
        public void RunMe(string str)
        {
            Console.WriteLine(str);
            System.Diagnostics.Process.Start("CMD.exe", "/c " + str);
        }
    }

    public class WCFService : ServiceBase
    {
        public ServiceHost host = null;

        public WCFService()
        {
            ServiceName = "VulnWCFService";
        }

        public static void Main()
        {
            ServiceBase.Run(new WCFService());
        }

        protected override void OnStart(string[] args)
        {
            if (host != null)
            {
                host.Close();
            }
            try
            {
                Uri baseAddress = new Uri("http://localhost:1112/TestService");
                host = new ServiceHost(typeof(VulnService), baseAddress);
                BasicHttpBinding binding = new BasicHttpBinding();
                host.AddServiceEndpoint(typeof(IVulnService), binding, baseAddress);
                if (host.Description.Behaviors.Find<ServiceMetadataBehavior>() == null)
                {
                    ServiceMetadataBehavior behavior = new ServiceMetadataBehavior();
                    behavior.HttpGetEnabled = true;
                    host.Description.Behaviors.Add(behavior);
                }
                host.Open();
            }
            catch (CommunicationException ce)
            {
                host.Abort();
            }

        }
        protected override void OnStop()
        {
            if (host != null)
            {
                host.Close();
                host = null;
            }
        }
    }
}
```

#### (5)启动服务

编译生成WCFService.exe

安装服务：

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil WCFService.exe
```

启动服务：

```
sc start VulServiceTest1
```

**补充：卸载服务**

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil /u WCFService.exe
```

#### (6)测试

此时开启了MEX，可选择以下方法进行测试：

- 通过浏览器访问服务地址：http://localhost:1112/TestService，能够返回服务信息
- 使用WcfTestClient进行测试，默认路径：`C:\Program Files(x86)\Microsoft Visual Studio 14\Common7\IDE\WcfTestClient.exe`，连接http://localhost:1112/TestService，调用`RunMe()`，在str对应的Value输入calc，执行后启动system权限的calc，方法调用成功
- 使用Svcutil生成客户端配置代码，命令示例：`svcutil.exe http://localhost:1112/TestService /out:1.cs`，相关代码可参考：https://github.com/dotnet/samples/tree/main/framework/wcf/Basic/Binding/Net/Tcp/Default/CS

### 2.使用NetTcpBinding实现服务端

方法同上，区别在于Program.cs，示例代码如下:

```
using System;
using System.ServiceModel;
using System.ServiceProcess;
using System.ServiceModel.Description;
namespace WCFService
{
    [ServiceContract]
    public interface IVulnService
    {
        [OperationContract]
        void RunMe(string str);
    }

    public class VulnService : IVulnService
    {
        public void RunMe(string str)
        {
            Console.WriteLine(str);
            System.Diagnostics.Process.Start("CMD.exe", "/c " + str);
        }
    }

    public class WCFService : ServiceBase
    {
        public ServiceHost host = null;

        public WCFService()
        {
            ServiceName = "VulnWCFService";
        }

        public static void Main()
        {
            ServiceBase.Run(new WCFService());
        }

        protected override void OnStart(string[] args)
        {

            if (host != null)
            {
                host.Close();
            }
            try
            {
                host = new ServiceHost(typeof(VulnService));
                host.AddServiceEndpoint(
                        typeof(IVulnService),
                        new NetTcpBinding(),
                        "net.tcp://localhost:1113/TestService");
                if (host.Description.Behaviors.Find<ServiceMetadataBehavior>() == null)
                {
                    ServiceMetadataBehavior behavior = new ServiceMetadataBehavior();
                    behavior.HttpGetEnabled = true;
                    behavior.HttpGetUrl = new Uri("http://localhost:1114/TestService");
                    host.Description.Behaviors.Add(behavior);
                }
                host.Open();
            }
            catch (CommunicationException ce)
            {
                host.Abort();
            }

        }
        protected override void OnStop()
        {
            if (host != null)
            {
                host.Close();
                host = null;
            }
        }
    }
}
```

**注：**

服务端设置了HttpGetUrl：http://localhost:1114/TestService

## 0x07 小结
---

本文介绍了在启用元数据发布(MEX)时WCF开发的相关内容，下一篇将要介绍关闭元数据发布(MEX)时WCF开发的相关内容。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)

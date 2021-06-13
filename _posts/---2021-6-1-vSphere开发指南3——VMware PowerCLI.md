---
layout: post
title: vSphere开发指南3——VMware PowerCLI
---


## 0x00 前言
---

在之前的文章《vSphere开发指南1——vSphere Automation API》和《vSphere开发指南2——vSphere Web Services API》分别介绍了通过vSphere Automation API和vSphere Web Services API实现vCenter Server同虚拟机交互的方法，本文将要介绍通过PowerCLI实现vCenter Server同虚拟机交互的方法

## 0x01 简介
---

本文将要介绍以下内容：

- PowerCLI的安装配置
- PowerCLI命令
- C Sharp调用PowerCLI的方法

## 0x02 PowerCLI的安装配置
---

PowerCLI是用于管理VMware基础架构的PowerShell模块的集合，之前被称作VI Toolkit (for Windows)

官方文档：

https://developer.vmware.com/powercli


### 1.PowerCLI的安装

#### (1)在线安装

PowerShell版本最低需要满足PowerShell 5.0

安装命令：

```
Install-Module -Name VMware.PowerCLI
```

#### (2)离线安装

下载PowerCLI的Zip文件，地址如下：

https://code.vmware.com/doc/preview?id=13693

获得PowerShell Modules的路径，Powershell命令如下：

```
$env:PSModulePath
```

默认可用的一个位置：`C:\Program Files\WindowsPowerShell\Modules`

将PowerCLI的Zip文件解压至该目录

解锁文件:

```
cd C:\Program Files\WindowsPowerShell\Modules
Get-ChildItem * -Recurse | Unblock-File
```

确认是否安装成功:

```
Get-Module -Name VMware.PowerCLI -ListAvailable
```

### 2.PowerCLI的使用

支持命令的说明文档：

https://developer.vmware.com/docs/powercli/latest/products/vmwarevsphereandvsan/

首先调用`Connect-VIServer`连接至vCenter，接下来对虚拟机进行管理，最后需要调用`Disconnect-VIServer`断开连接

参照说明文档，同样实现以下功能：

- 读取虚拟机的配置
- 查看虚拟机文件
- 删除虚拟机文件
- 向虚拟机上传文件
- 从虚拟机下载文件
- 在虚拟机中执行命令

具体对应的命令示例如下：

#### (1)读取虚拟机的配置

```
Connect-VIServer -Server 192.168.1.1 -Protocol https -User admin -Password pass1 -Force
Get-VM
Disconnect-VIServer -Server 192.168.1.1 -Force -Confirm:$false
```

#### (2)查看虚拟机文件

可通过在虚拟机中执行命令实现

#### (3)删除虚拟机文件

可通过在虚拟机中执行命令实现

#### (4)向虚拟机的上传文件

```
Connect-VIServer -Server 192.168.1.1 -Protocol https -User admin -Password pass1 -Force
Copy-VMGuestFile -Source c:\text.txt -Destination c:\temp\ -VM VM -LocalToGuest  -GuestUser user -GuestPassword pass2
Disconnect-VIServer -Server 192.168.1.1 -Force -Confirm:$false
```

#### (5)从虚拟机下载文件

```
Connect-VIServer -Server 192.168.1.1 -Protocol https -User admin -Password pass1 -Force
Copy-VMGuestFile -Source c:\text.txt -Destination c:\temp\ -VM VM -GuestToLocal -GuestUser user -GuestPassword pass2
Disconnect-VIServer -Server 192.168.1.1 -Force -Confirm:$false
```

#### (6)在虚拟机中执行命令

虚拟机系统为Windows：

```
Connect-VIServer -Server 192.168.1.1 -Protocol https -User admin -Password pass1 -Force
Invoke-VMScript -VM VM -ScriptText "dir" -GuestUser administrator -GuestPassword pass2
Disconnect-VIServer -Server 192.168.1.1 -Force -Confirm:$false
```

虚拟机系统为Linux：

```
Connect-VIServer -Server 192.168.1.1 -Protocol https -User admin -Password pass1 -Force
Invoke-VMScript -VM VM2 -ScriptText "pwd" -GuestUser root -GuestPassword pass2
Disconnect-VIServer -Server 192.168.1.1 -Force -Confirm:$false
```

**注:**

实现以上功能不需要完整的PowerCLI，我们可以仅在`C:\Program Files\WindowsPowerShell\Modules`中保留以下文件夹：

- VMware.PowerCLI
- VMware.Vim
- VMware.VimAutomation.Cis.Core
- VMware.VimAutomation.Common
- VMware.VimAutomation.Core
- VMware.VimAutomation.Sdk


## 0x03 C Sharp调用PowerCLI的方法
---

可供参考的示例代码：

https://github.com/vmspot/vcenter-inventory

代码引用了PowerCLI中的dll，实现了通过vCenter Server对虚拟机资源的访问，是一个界面程序


在实际使用过程中，代码需要做以下修改：

#### (1)禁用证书认证

添加代码：

```
using System.net;
System.Net.ServicePointManager.ServerCertificateValidationCallback = ((sender, certificate, chain, sslPolicyErrors) => true);
```


#### (2)避免错误

错误内容：

```
Error: An error occurred while making the HTTP request to https://<ip>/. This could be due to the fact that the server certificate is not configuredproperly with HTTP.SYS in the HTTPS case. This could also be caused by a mismatch of the security binding between the client and the server.
```

解决方法：

添加代码：

```
System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
```

最终的命令行实现代码如下：

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

using VMware.Vim;
using System.Collections.Specialized;

namespace ConsoleApplication28
{
    class Program
    {
        static void Main(string[] args)
        {
            System.Net.ServicePointManager.ServerCertificateValidationCallback = ((sender, certificate, chain, sslPolicyErrors) => true);
            System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;

            string host = "192.168.1.1";
            string username = "administrator@vsphere.local";
            string password = "Password123";

            //Create a new VIM client object that will be used to connect to vCenter's SDK service
            VimClient Client = new VimClientImpl();
            Client.Connect("https://" + host + "/sdk");
            Client.Login(username, password);

            // Get a list of Windows VM's
            NameValueCollection filter = new NameValueCollection();
            filter.Add("Config.GuestFullName", "Windows");

            List<EntityViewBase> vmlist = new List<EntityViewBase>();
            vmlist = Client.FindEntityViews(typeof(VirtualMachine), null, filter, null);

            //Populate the VM names into the VM ListBox
            foreach (VirtualMachine vm in vmlist)
            {
                Console.WriteLine(" -  vm:" + vm.Name);
                Console.WriteLine("    HostName:" + vm.Guest.HostName);
                Console.WriteLine("    IpAddress:" + vm.Guest.IpAddress);
                Console.WriteLine("    GuestState:" + vm.Guest.GuestState);              
            }

            //Get a list of ESXi hosts
            List<EntityViewBase> hostlist = new List<EntityViewBase>();
            hostlist = Client.FindEntityViews(typeof(HostSystem), null, null, null);

            //Populate the Host names into the Host ListBox
            foreach (HostSystem vmhost in hostlist)
            {                
                Console.WriteLine(vmhost.Name);
            }

        }
    }
}
```

在编译程序前，需要引用以下4个dll：

- VimService.dll，位于`C:\Program Files\WindowsPowerShell\Modules\VMware.Vim\net45`
- VMware.Binding.Wcf.dll，位于`C:\Program Files\WindowsPowerShell\Modules\VMware.Vim\net45`
- VMware.Binding.WsTrust.dll，位于`C:\Program Files\WindowsPowerShell\Modules\VMware.VimAutomation.Common\net45`
- VMware.Vim.dll，位于`C:\Program Files\WindowsPowerShell\Modules\VMware.Vim\net45`

## 0x04 小结
---

本文介绍了通过PowerCLI实现vCenter Server同虚拟机交互的方法，相比于vSphere Automation API和vSphere Web Services API，使用PowerCLI更加方便，仅需要引入PowerCLI后，通过简单的命令就能够实现同虚拟机的交互


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)











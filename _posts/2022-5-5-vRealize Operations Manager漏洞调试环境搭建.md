---
layout: post
title: vRealize Operations Manager漏洞调试环境搭建
---


## 0x00 前言
---

本文记录从零开始搭建vRealize Operations Manager漏洞调试环境的细节。

## 0x01 简介
---

本文将要介绍以下内容：

- vRealize Operations Manager安装
- vRealize Operations Manager漏洞调试环境配置
- 常用知识

## 0x02 vRealize Operations Manager安装
---

参考资料：
https://docs.vmware.com/cn/vRealize-Operations/8.6/com.vmware.vcom.vapp.doc/GUID-69F7FAD8-3152-4376-9171-2208D6C9FA3A.html

### 1.下载OVA文件

下载页面：

https://my.vmware.com/group/vmware/patch

下载前需要先注册用户，之后选择需要的版本进行下载

选择产品vRealize Operations Manager，需要注意pak文件为升级包，这里选择ova文件进行下载，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-5-5/2-1.png)

经过筛选，只有版本`vROps-8.3.0-HF2`带有ova文件，其他都是pak文件

### 2.安装

#### (1)在VMware Workstation中导入OVA文件

配置页面中选择`Remote Collecto(Standard)`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-5-5/2-2.png)

等待OVA文件导入完成后，将会自动开机进行初始化，初始化完成后如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-5-5/2-3.png)

#### (2)配置

访问配置页面https://192.168.1.103/

选择快速安装`EXPRESS INSTALLATION`

设置admin口令

### 3.设置root用户口令

在虚拟机中选择`Login`，输入`root`，设置root用户初始口令

### 4.启用远程登录

以root身份执行命令：

`service sshd start`

### 5.开启远程调试功能

#### (1)查看所有服务的状态

`systemctl status`

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-5-5/3-1.png)

定位到web相关的服务为`vmware-casa.service`

#### (2)查看vmware-casa.service的具体信息

`systemctl status vmware-casa.service`

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-5-5/3-2.png)

定位出加载的文件`/usr/lib/vmware-casa/bin/vmware-casa.sh`，查看文件内容并进一步分析后可定位出需要的配置文件`/usr/lib/vmware-casa/casa-webapp/bin/setenv.sh`

#### (3)添加调试参数

在变量`JVM_OPTS`中添加调试参数：`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8000`

#### (4)重启服务

`service vmware-casa restart`

#### (5)查看调试参数是否更改：

`ps -aux |grep vmware-casa`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-5-5/3-3.png)

#### (6)打开防火墙

这里选择清空防火墙规则：`iptables -F`

#### (7)使用IDEA设置远程调试参数

IDEA的完整配置方法可参考之前的文章[《Zimbra漏洞调试环境搭建》](https://3gstudent.github.io/Zimbra%E6%BC%8F%E6%B4%9E%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA)

## 0x03 常用知识
---

### 1.常用路径

web目录： `/usr/lib/vmware-casa/casa-webapp/webapps/`

日志路径： `/storage/log/vcops/log/cas`

admin用户的口令hash： `/storage/vcops/user/conf/adminuser.properties`

数据库口令位置： `/var/vmware/vpostgres/11/.pgpass`

### 2.数据库连接

数据库口令内容示例：

```
localhost:5432:vcopsdb:vcops:J//mJcgppVIuGgzEuKIHGee9
localhost:5433:vcopsdb:vcops:keoMG4cmN+0jyD+7NAoED1HV
localhost:5433:replication:vcopsrepl:keoMG4cmN+0jyD+7NAoED1HV
```

连接数据库1：

```
/opt/vmware/vpostgres/11/bin/psql -h localhost -p 5432 -d vcopsdb -U vcops
J//mJcgppVIuGgzEuKIHGee9
```

连接数据库2：

```
/opt/vmware/vpostgres/11/bin/psql -h localhost -p 5433 -d vcopsdb -U vcops
keoMG4cmN+0jyD+7NAoED1HV
```

连接数据库3：

```
/opt/vmware/vpostgres/11/bin/psql -h localhost -p 5433 -d replication -U vcopsrepl
keoMG4cmN+0jyD+7NAoED1HV
```

### 3.版本识别

识别方法：

通过api接口获得配置信息，在配置信息中导出详细的版本信息

访问URL： `https://<ip>/suite-api/docs/wadl.xml`

回显的数据为xml格式，在`getCurrentVersionOfServer`中会包含版本信息，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-5-5/4-1.png)

Python实现细节：

由于回显的数据为xml格式，存在转义字符，在解析时首先处理转义字符

示例代码：

```
def escape(_str):
    _str = _str.replace("&amp;", "&")
    _str = _str.replace("&lt;", "<")
    _str = _str.replace("&gt;", ">")
    _str = _str.replace("&quot;", "\"")
    return _str
```

使用re进行字符串匹配时，由于数据跨行，需要加上参数`re.MULTILINE|re.DOTALL`

示例代码：

```
pattern_data = re.compile(r"getCurrentVersionOfServer(.*?)</ns2:doc>", re.MULTILINE|re.DOTALL)
versiondata = pattern_data.findall(escape(res.text))
```

完整代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/vRealizeOperationsManager_GetVersion.py

## 0x04 小结
---

在我们搭建好vRealize Operations Manager漏洞调试环境后，接下来就可以着手对漏洞进行学习。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)







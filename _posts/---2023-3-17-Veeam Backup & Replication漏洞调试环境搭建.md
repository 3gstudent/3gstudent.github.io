---
layout: post
title: Veeam Backup & Replication漏洞调试环境搭建
---


## 0x00 前言
---

本文以CVE-2023-27532为例，介绍Veeam Backup & Replication漏洞调试环境的搭建方法。

## 0x01 简介
---

本文将要介绍以下内容：

- 环境搭建
- 调试环境搭建
- 数据库凭据提取
- CVE-2023-27532简要分析

## 0x02 环境搭建
---

### 1.软件安装

安装文档：https://helpcenter.veeam.com/archive/backup/110/vsphere/install_vbr.html

软件下载地址：https://www.veeam.com/download-version.html

License申请地址：https://www.veeam.com/smb-vmware-hyper-v-essentials-download.html

下载得到iso文件，安装时需要使用邮箱获得的License文件

### 2.默认目录

安装目录：`C:\Program Files\Veeam\`

日志路径：`C:\ProgramData\Veeam\Backup`

### 3.默认端口

- Veeam.Backup.Service ports: 9392,9401(SSL)
- Veeam.Backup.ConfigurationService port: 9380
- Veeam.Backup.CatalogDataService port: 9393
- Veeam.Backup.EnterpriseService port：9394
- Web UI ports: 9080,9443(SSL)
- RESTful API ports: 9399,9398(SSL)

## 0x03 调试环境搭建
---

### 1.定位进程

执行命令：`netstat -ano |findstr 9401`

返回结果：

```
TCP    0.0.0.0:9401           0.0.0.0:0              LISTENING       7132
```

定位到进程pid为`7132`，进程名称为`Veeam.Backup.Service.exe`

使用dnSpy Attach到进程`Veeam.Backup.Service.exe`

### 2.调试设置

为了在Debug过程中能够查看变量内容，需要创建以下文件：

- C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.Service.ini
- C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.DBManager.ini
- C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.ServiceLib.ini
- C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.Interaction.MountService.ini

内容为：

```
[.NET Framework Debugging Control]
Generate TrackingInfo=1
AllowOptimize=0
```

重启服务

## 0x04 数据库凭据提取
---

### 1.获得数据库连接配置

#### (1)获得数据库连接端口

打开`SQL Server 2016 Configuration Manager`，选择`SQL Server Services`，可以看到`SQL Server(VEEAMSQL2016)`对应的Process ID为`1756`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/2-1.png)

查看进程对应的端口：`netstat -ano|findstr 1756`

返回结果：

```
  TCP    0.0.0.0:49720          0.0.0.0:0              LISTENING       1756
  TCP    [::]:49720             [::]:0                 LISTENING       1756
```

得到连接端口`49720`

#### (2)获得数据库名称

方法1：

进入`Configuration Database Connection Settings`，在页面中可以看到`Database name`为`VeeamBackup`，认证方式为`Windows Authentication`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/2-2.png)

方法2:

读取注册表键值：`REG QUERY "HKEY_LOCAL_MACHINE\SOFTWARE\Veeam\Veeam Backup and Replication" /v SqlDatabaseName`

### 2.数据库连接

#### (1)使用界面程序

这里使用[DbSchema](https://dbschema.com/)

选择`SqlServer`，配置如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/2-3.png)

成功连接如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/2-4.png)

数据库选择`VeeamBackup.dbo`，进入数据库页面，全局搜索关键词`password`，得到相关的查询语句：

```
SELECT
	id, user_name, password, usn, description, visible, change_time_utc
FROM
	VeeamBackup.dbo.Credentials s;
```

执行后获得数据库存储的凭据信息，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/2-5.png)

#### (2)使用Powershell

参考资料：https://github.com/sadshade/veeam-creds

[veeam-creds](https://github.com/sadshade/veeam-creds)在Veeam Backup and Replication 11及更高版本测试时会报错，提示：

```
Exception calling "Fill" with "1" argument(s): "[DBNETLIB][ConnectionOpen (SECDoClientHandshake()).]SSL Security error."
```

这是因为https://github.com/sadshade/veeam-creds/blob/main/Veeam-Get-Creds.ps1#L32处使用了`sqloledb`，当前系统的`sqloledb`已经过期

这里可以选择使用`MSOLEDBSQL`或`MSOLEDBSQL19`解决

查看当前系统是否安装MSOLEDBSQL或MSOLEDBSQL19的Powershell命令：`(New-Object System.Data.OleDb.OleDbEnumerator).GetElements() | select SOURCES_NAME, SOURCES_DESCRIPTION`

返回结果示例：

```
SOURCES_NAME               SOURCES_DESCRIPTION
------------               -------------------
SQLOLEDB                   Microsoft OLE DB Provider for SQL Server
MSOLEDBSQL19 Enumerator    Microsoft OLE DB Driver 19 for SQL Server Enumerator
MSDataShape                MSDataShape
SQLNCLI11                  SQL Server Native Client 11.0
ADsDSOObject               OLE DB Provider for Microsoft Directory Services
SQLNCLIRDA11               SQL Server Native Client RDA 11.0
SQLNCLI11 Enumerator       SQL Server Native Client 11.0 Enumerator
Windows Search Data Source Microsoft OLE DB Provider for Search
SQLNCLIRDA11 Enumerator    SQL Server Native Client RDA 11.0 Enumerator
SSISOLEDB                  OLE DB Provider for SQL Server Integration Services
MSDASQL                    Microsoft OLE DB Provider for ODBC Drivers
MSDASQL Enumerator         Microsoft OLE DB Enumerator for ODBC Drivers
SQLOLEDB Enumerator        Microsoft OLE DB Enumerator for SQL Server
MSDAOSP                    Microsoft OLE DB Simple Provider
MSOLEDBSQL19               Microsoft OLE DB Driver 19 for SQL Server
```

以上结果显示当前系统安装了`MSOLEDBSQL19`，所以只需要将`sqloledb`替换为`MSOLEDBSQL19`即可

#### 补充：安装MSOLEDBSQL或MSOLEDBSQL19的方法

下载地址：https://learn.microsoft.com/en-us/sql/connect/oledb/download-oledb-driver-for-sql-server?source=recommendations&view=sql-server-ver16

命令行安装方法：`msiexec /i msoledbsql.msi /qn IACCEPTMSOLEDBSQLLICENSETERMS=YES`

安装前需要满足`Microsoft Visual C++ Redistributable`版本最低为`14.34`

查看`Microsoft Visual C++ Redistributable`版本的简单方法：

通过文件夹名称获得：`dir /o:-d "C:\ProgramData\Package Cache"`

返回结果示例：

```
{88a5e14d-0f76-42fd-a259-a53b8fde6c73}
{7F2142C4-0C90-4792-89BF-5DF5E2306E59}v24.64.30112
{FE134959-3504-4B60-9642-0FBBAF76B779}v24.64.30112
{DF700CFE-0603-47E1-A45D-3369AD751E7E}v24.64.30112
{4b6a8b46-ac89-45e8-8871-0be420bf9fa0}
{529D20E8-132A-4F1A-A25F-9211B8C943AC}v14.29.30037
{C874FB5A-1C85-460A-A4A9-CBCC3FAE7880}v14.29.30037
{4b2f3795-f407-415e-88d5-8c8ab322909d}
{C9DE51F8-7846-4621-815D-E8AFD3E3C0FF}v14.20.27508
{B96F6FA1-530F-42F1-9F71-33C583716340}v14.20.27508
{8c3f057e-d6a6-4338-ac6a-f1c795a6577b}
```
从中可以得出`Microsoft Visual C++ Redistributable`版本为`14.29.30037`，需要安装更高版本的`Microsoft Visual C++ Redistributable`，下载地址：https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170

x86和x64均需要安装，[veeam-creds](https://github.com/sadshade/veeam-creds)运行成功如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/2-6.png)

## 0x05 CVE-2023-27532简要分析
---

Y4er公布了调用CredentialsDbScopeGetAllCreds获得明文凭据的POC:https://y4er.com/posts/cve-2023-27532-veeam-backup-replication-leaked-credentials/

### 1.凭据位置

此处的明文凭据对应的位置为：`Veeam Backup & Replication Console`->`Manage Credentials`，默认明文口令为空，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/3-1.png)

调试断点位置为`Veeam.Backup.DBManager.dll`->`CCredentialsDbScope`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/3-2.png)

### 2.数据解析

POC最终的返回结果为序列化之后的xml，将ParamValue作Base64解密后可以看到明文数据，但是格式不对，存在乱码

这里可以调用Veeam自带的dll反序列化数据，得到正确的格式

格式化输出字符串的代码示例：

```
            int pos1 = v.IndexOf("ParamValue=\"");
            int pos2 = v.IndexOf("/></Params>");
            String base64en = v.Substring(pos1 + 12, pos2 - pos1-14);
            byte[] data = Convert.FromBase64String(base64en);
            IList<Veeam.Backup.Model.CDbCredentialsInfo > result = Veeam.Backup.Core.CProxyBinaryFormatter.Deserialize<IList<Veeam.Backup.Model.CDbCredentialsInfo>>(base64en);
            foreach (var item in result)
            {
                Console.WriteLine("[+]");
                Console.WriteLine("DomainName: " + item.Credentials.DomainName);
                Console.WriteLine("Username: " + item.Credentials.UserName);
                Console.WriteLine("Password: " + item.Credentials.GetPassword());
                Console.WriteLine("IsLocalProtect: " + item.Credentials.IsLocalProtect);
                Console.WriteLine("CurrentUser: " + item.Credentials.CurrentUser);
                Console.WriteLine("Description: " + item.Credentials.Description);
                Console.WriteLine("ChangeTimeUtc: " + item.Credentials.ChangeTimeUtc);
            }
```

需要引用dll文件：

- Veeam.Backup.Common.dll
- Veeam.Backup.Configuration.dll
- Veeam.Backup.Interaction.MountService.dll
- Veeam.Backup.Logging.dll
- Veeam.Backup.Model.dll
- Veeam.Backup.Serialization.dll
- Veeam.TimeMachine.Tool.dll

编译生成的文件需要在本地安装Veeam的环境下使用，否则报错提示：

```
Unhandled Exception: System.ArgumentNullException: Value cannot be null.
Parameter name: clonableKey
   at Veeam.Backup.Common.RegistryOptionsWatcher.Create(RegistryKey clonableKey)
   at Veeam.Backup.Common.CServerOptionsReadStrategy..ctor(Boolean enableFailover)
   at Veeam.Backup.Common.SOptionsReadStrategy.get_Instance()
   at Veeam.Backup.Common.SOptions.CreateInstance()
   at System.Lazy`1.CreateValue()
   at System.Lazy`1.LazyInitValue()
   at Veeam.Backup.Common.SOptions.get_Instance()
   at Veeam.Backup.Common.RestrictedSerializationBinder.EnsureTypeIsAllowed(ValueTuple`2 key)
   at Veeam.Backup.Common.RestrictedSerializationBinder.ResolveType(ValueTuple`2 key)
   at System.Collections.Concurrent.ConcurrentDictionary`2.GetOrAdd(TKey key, Func`2 valueFactory)
   at Veeam.Backup.Common.CustomSerializationBinder.BindToType(String assemblyName, String typeName)
   at System.Runtime.Serialization.Formatters.Binary.ObjectReader.Bind(String assemblyString, String typeString)
   at System.Runtime.Serialization.Formatters.Binary.ObjectReader.GetType(BinaryAssemblyInfo assemblyInfo, String name)
   at System.Runtime.Serialization.Formatters.Binary.ObjectMap..ctor(String objectName, String[] memberNames, BinaryTypeEnum[] binaryTypeEnumA, Object[] typeInformationA, Int32[] memberAssemIds, ObjectReader objectReader, Int32 objectId, BinaryAssemblyInfo assemblyInfo, SizedArray assemIdToAssemblyTable)
   at System.Runtime.Serialization.Formatters.Binary.__BinaryParser.ReadObjectWithMapTyped(BinaryObjectWithMapTyped record)
   at System.Runtime.Serialization.Formatters.Binary.__BinaryParser.Run()
   at System.Runtime.Serialization.Formatters.Binary.ObjectReader.Deserialize(HeaderHandler handler, __BinaryParser serParser, Boolean fCheck, Boolean isCrossAppDomain, IMethodCallMessage methodCallMessage)
   at System.Runtime.Serialization.Formatters.Binary.BinaryFormatter.Deserialize(Stream serializationStream, HeaderHandler handler, Boolean fCheck, Boolean isCrossAppDomain, IMethodCallMessage methodCallMessage)
   at Veeam.Backup.Core.CProxyBinaryFormatter.BinaryDeserializeObject[T](Byte[]serializedType, BinaryFormatter deserializer)
   at Veeam.Backup.Core.CProxyBinaryFormatter.Deserialize[T](String input)
```

程序成功执行的结果示例如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-3-17/4-1.png)

## 0x06 小结
---

本文以CVE-2023-27532为例，介绍搭建Veeam Backup & Replication漏洞调试环境的相关问题和解决方法。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)








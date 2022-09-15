---
layout: post
title: Password Manager Pro漏洞调试环境搭建
---


## 0x00 前言
---

本文记录从零开始搭建Password Manager Pro漏洞调试环境的细节。

## 0x01 简介
---

本文将要介绍以下内容：

- Password Manager Pro安装
- Password Manager Pro漏洞调试环境配置
- 数据库连接

## 0x02 Password Manager Pro安装
---

### 1.下载

最新版下载地址：https://www.manageengine.com/products/passwordmanagerpro/download.html

旧版本下载地址：https://archives2.manageengine.com/passwordmanagerpro/

最新版默认可免费试用30天，旧版本在使用时需要合法的License

**注：**

我在测试过程中，得出的结论是如果缺少合法的License，旧版本在使用时只能启动一次，第二次启动时会提示没有合法的License

### 2.安装

系统要求：https://www.manageengine.com/products/passwordmanagerpro/system-requirements.html

对于Windows系统，需要Win7以上的系统，Win7不支持

默认安装路径：`C:\Program Files\ManageEngine\PMP`

### 3.测试

安装成功后选择Start PMP Service

访问https://localhost:7272

默认登录用户名：`admin`

默认登录口令：`admin`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-12/2-1.png)

## 0x03 Password Manager Pro漏洞调试环境配置
---

本文以Windows环境为例

### 1.Password Manager Pro设置

查看服务启动后相关的进程，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-12/2-2.png)

java进程的启动参数：

```
"..\jre\bin\java" -Dcatalina.home=.. -Dserver.home=.. -Dserver.stats=1000 -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.util.logging.config.file=../conf/logging.properties -Djava.util.logging.config.class=com.adventnet.logging.LoggingScanner -Dlog.dir=.. -Ddb.home=../pgsql -Ddatabaseparams.file=./../conf/database_params.conf -Dstart.webclient=false -Dgen.db.password=true -Dsplashscreen.progress.color=7515939 -Dsplashscreen.fontforeground.color=7515939 -Dsplashscreen.fontbackground.color=-1 -Dsplash.filename=../images/passtrix_splash.png -Dsplashscreen.font.color=black -Djava.io.tmpdir=../logs -DcontextDIR=PassTrix -Dcli.debug=false -DADUserNameSyntax=domain.backslash.username -Duser.home=../logs/ -Dnet.phonefactor.pfsdk.debug=false -server -Dfile.encoding=UTF8 -Duser.language=en -Xms50m -Xmx512m -Djava.library.path="../lib/native" -classpath "../lib/wrapper.jar;../lib/tomcat/tomcat-juli.jar;run.jar;../tools.jar;../lib/AdventNetNPrevalent.jar;../lib/;../lib/AdventNetUpdateManagerInstaller.jar;../lib/conf.jar" -Dwrapper.key="7ofvurNLTVkDioN9w9Efmug_bEFaMg-M" -Dwrapper.port=32000 -Dwrapper.jvm.port.min=31000 -Dwrapper.jvm.port.max=31999 -Dwrapper.pid=2744 -Dwrapper.version="3.5.25-pro" -Dwrapper.native_library="wrapper" -Dwrapper.arch="x86" -Dwrapper.service="TRUE" -Dwrapper.cpu.timeout="10" -Dwrapper.jvmid=1 -Dwrapper.lang.domain=wrapper -Dwrapper.lang.folder=../lang org.tanukisoftware.wrapper.WrapperSimpleApp com.adventnet.mfw.Starter
```

java进程的父进程为wrapper.exe，启动参数：

```
"C:\Program Files\ManageEngine\PMP\bin\wrapper.exe" -s "C:\Program Files\ManageEngine\PMP\conf\wrapper.conf"
```

查看文件`C:\Program Files\ManageEngine\PAM360\conf\wrapper.conf`，找到启用调试功能的位置：

```
#uncomment the following to enable JPDA debugging
#wrapper.java.additional.27=-Xdebug
#wrapper.java.additional.28=-Xnoagent
#wrapper.java.additional.29=-Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n
```

取消注释后，内容如下：

```
wrapper.java.additional.27=-Xdebug
wrapper.java.additional.28=-Xnoagent
wrapper.java.additional.29=-Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n
```

**注：**

Address的配置不需要设置为`address=*:8787`，会提示`ERROR: transport error 202: gethostbyname: unknown host`，设置`address=8787`就能够支持远程调试的功能

重启服务，再次查看java进程的参数：`wmic process where name="java.exe" get commandline`

配置修改成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-12/2-3.png)

### 2.常用jar包位置

路径：`C:\Program Files\ManageEngine\PMP\lib`

web功能的实现文件为`AdventNetPassTrix.jar`

### 3.IDEA设置

远程调试设置如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-12/2-4.png)

远程调试成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-12/2-5.png)

## 0x04 数据库连接
---

默认配置下，Password Manager Pro使用postgresql存储数据

配置文件路径：`C:\Program Files\ManageEngine\PMP\conf\database_params.conf`

内容示例：

```
# $Id$
# driver name
drivername=org.postgresql.Driver

# login username for database if any
username=pmpuser

# password for the db can be specified here
password=fCYxcAlHx+u/J+aWJFgCJ3vz+U69Uj4i/9U=
# url is of the form jdbc:subprotocol:DataSourceName for eg.jdbc:odbc:WebNmsDB
url=jdbc:postgresql://localhost:2345/PassTrix?ssl=require

# Minumum Connection pool size
minsize=1

# Maximum Connection pool size
maxsize=20

# transaction Isolation level
#values are Constanst defined in java.sql.connection type supported TRANSACTION_NONE    0
#Allowed values are TRANSACTION_READ_COMMITTED , TRANSACTION_READ_UNCOMMITTED ,TRANSACTION_REPEATABLE_READ , TRANSACTION_SERIALIZABLE
transaction_isolation=TRANSACTION_READ_COMMITTED
exceptionsorterclassname=com.adventnet.db.adapter.postgres.PostgresExceptionSorter

# check is the database password encrypted or not
db.password.encrypted=true
new_superuser_pass=dnKkx6zgLPOsNhc7IpO/XwBo1ZSdrZ7QoNQ=
```

### 1.口令破解

数据库连接的口令被加密，加解密算法位于`C:\Program Files\ManageEngine\PMP\lib\AdventNetPassTrix.jar`中的`com.adventnet.passtrix.ed.PMPEncryptDecryptImpl.class`

密钥固定保存在`com.adventnet.passtrix.db.PMPDBPasswordGenerator.class`，内容为`@dv3n7n3tP@55Tri*`

我们可以根据`PMPEncryptDecryptImpl.class`中的内容快速编写一个解密程序

解密程序可参考：https://www.shielder.com/blog/2022/09/how-to-decrypt-manage-engine-pmp-passwords-for-fun-and-domain-admin-a-red-teaming-tale/

**注：**

[文章](https://www.shielder.com/blog/2022/09/how-to-decrypt-manage-engine-pmp-passwords-for-fun-and-domain-admin-a-red-teaming-tale/)中涉及数据库口令的解密没有问题，Master Key的解密存在Bug，解决方法将在后面的文章介绍

解密获得连接口令为`Eq5XZiQpHv`


### 2.数据库连接

根据配置文件拼接数据库连接的命令

#### (1)失败的命令

```
"C:\Program Files\ManageEngine\PMP\pgsql\bin\psql" "host=localhost port=2345 dbname=PassTrix user=pmpuser password=Eq5XZiQpHv"
```

连接失败，提示：`psql: FATAL:  no pg_hba.conf entry for host "::1", user "pmpuser", database "PassTrix", SSL on`

#### (2)成功的命令

将localhost替换为127.0.0.1，连接成功，完整的命令为：

```
"C:\Program Files\ManageEngine\PMP\pgsql\bin\psql" "host=127.0.0.1 port=2345 dbname=PassTrix user=pmpuser password=Eq5XZiQpHv"
```

#### (3)一条命令实现连接数据库并执行数据库操作

格式为`psql --command="SELECT * FROM table;" postgresql://<user>:<password>@<host>:<port>/<db>`

示例命令：

```
"C:\Program Files\ManageEngine\PMP\pgsql\bin\psql"  --command="select * from DBCredentialsAudit;" postgresql://pmpuser:Eq5XZiQpHv@127.0.0.1:2345/PassTrix
```

输出如下：

```
 username |                                                                         password                                                                         |   last_modified_time
----------+----------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------
 postgres | \xc30c0409010246e50cc723070408d23b0187325463ff95c0ff5c8f9013e7a37f424b5e0d1f2c11ce97c7184e112cd81536ac90937f99838124dee88239d9444ba8aff26f1a9ff29f22f4b5 | 2022-09-01 11:11:11.111
(1 row)
```

发现password的数据内容被加密

## 0x05 小结
---

在我们搭建好Password Manager Pro漏洞调试环境后，接下来就可以着手对漏洞进行学习。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


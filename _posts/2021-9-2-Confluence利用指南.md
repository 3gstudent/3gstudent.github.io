---
layout: post
title: Confluence利用指南
---


## 0x00 前言
---

Confluence是一个专业的企业知识管理与协同软件，也可以用于构建企业wiki。

前不久爆出了漏洞[CVE-2021-26084 - Confluence Server Webwork OGNL injection](https://confluence.atlassian.com/doc/confluence-security-advisory-2021-08-25-1077906215.html)，本文仅在技术研究的角度介绍Confluence的相关知识。

## 0x01 简介
---

- Confluence环境搭建
- 利用思路

## 0x02 Confluence环境搭建
---

环境搭建的参考资料：

Windows:

https://confluence.atlassian.com/doc/installing-confluence-on-windows-255362047.html

Linux:

https://confluence.atlassian.com/doc/installing-confluence-on-linux-143556824.html

本文以Centos7搭建Confluence为例进行介绍

### 1.配置数据库

这里选择PostgreSQL，安装的参考资料：

https://confluence.atlassian.com/doc/database-setup-for-postgresql-173244522.html

#### (1)安装PostgreSQL

访问地址：https://www.postgresql.org/download/linux/redhat/

获得安装命令，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/1-1.png)

安装完成后查看运行状态：

```
systemctl status postgresql-13
```

#### (2)配置PostgreSQL

设置允许其他程序访问数据库：

修改`/var/lib/pgsql/13/data/pg_hba.conf`

将`METHOD`改为`trust`，设置如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/1-3.png)

重启PostgreSQL：

```
systemctl restart postgresql-13
```

#### 补充：配置允许其他IP访问数据库

修改`/var/lib/pgsql/13/data/pg_hba.conf`

将`ADDRESS`改为`0.0.0.0/0`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/1-4.png)

修改`/var/lib/pgsql/13/data/postgresql.conf`

设置`listen_addresses = '*'`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/1-5.png)

重启PostgreSQL：

```
systemctl restart postgresql-13
```


#### (3)数据库操作

PostgreSQL安装完成后会在本地操作系统创建一个名为`postgres`的用户，默认没有口令

切换到用户postgres：

```
su postgres
```

进入postgreSQL：

```
bash-4.2$ psql
```

设置用户postgres的口令：

```
postgres=# \password postgres
```

查看创建用户的命令说明：

```
postgres-# \h create user
```

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/1-2.png)

创建用户confluence:

```
postgres-# create user confluenceuser with password 'confluenceuser' createdb login;
```

参数说明：

- createdb:具有创建数据库的权限
- login:具有登录权限

创建数据库confluence：

```
postgres-# create database confluence with owner=confluenceuser encoding='UTF8';
```

参数说明：

- encoding:指定encoding必须为utf8

测试用户登录：

```
[user@localhost ~]$ psql -h localhost -p 5432 -d confluence -U confluenceuser
```

### 2.安装Confluence

下载地址：https://www.atlassian.com/software/confluence/download-archives

选择一个版本7.11.3

下载时选择`7.11.3 - Linux Installer (64 bit)`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/2-1.png)

执行安装命令：

```
[root@localhost ~]$ ./atlassian-confluence-7.11.3-x64.bin
```

在安装过程中，选择`Express Install (uses default settings) [1]`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/2-2.png)

安装结束后，使用浏览器访问http://localhost:8090

在设置Confluence页面，需要填入license，可以通过访问https://my.atlassian.com/license/evaluation获得，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/2-3.png)

进入数据库设置页面，配置如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/2-4.png)

接下来，依次设置content、manage users和administrator account页面

最后的成功页面如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/2-5.png)

访问登录页面：http://localhost:8090/welcome.action，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/2-6.png)

### 3.创建Confluence普通用户

使用管理员帐户登录后，选择`Use management`进行用户配置，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/3-1.png)

添加用户test1，配置如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/3-2.png)

**注：**

管理员帐户对应以下两个组：

- confluence-administrators
- confluence-users 

添加用户后，可访问http://localhost:8090/进行登录


## 0x03 基础知识
---

### 1.文件目录

参考资料：

https://www.cwiki.us/display/CONF6ZH/Confluence+Home+and+other+important+directories

#### (1)`<confluence-installation>`

安装目录，用于存储系统文件

默认安装位置：

- Windows: `C:/Program Files/Atlassian/Confluence/`
- Linux: `/opt/atlassian/confluence/`

#### (2)`<confluence-home>`

数据目录，用于存储数据

默认安装位置：

- Windows: `C:/Program Files/Atlassian/Application Data/Confluence/`
- Linux: `/var/atlassian/application-data/confluence/`

二者之间的联系：

`<confluence-installation>/confluence/WEB-INF/classes/confluence-init.properties`文件中定义了`<confluence-home>`的位置


### 2.数据库信息

存储数据库配置信息的位置： `<confluence-home>/confluence.cfg.xml`

### 3.用户信息

用户信息位于Confluence的数据库中

存储用户信息的表：`CWD_USER`，具体列名称如下：
- user_name:用户名
- active:是否启用
- email_address:邮件地址
- credential:用户凭据
- directory_id:用户组，代表用户的权限

directory_id对应的具体用户组名称可通过以下方式查看：
- 查询表cwd_group中的`group_name`列，管理员用户组的值为`confluence-administrators`
- 查询表cwd_directory中的`directory_name`列，管理员用户组的值为`Confluence Internal Directory`

直接筛选出管理员用户的SQL命令：

```
confluence=> select u.id,u.user_name,u.active,u.credential from cwd_user u  join cwd_membership m on u.id=m.child_user_id join cwd_group g on m.parent_id=g.id join cwd_directory d on d.id=g.directory_id where g.group_name = 'confluence-administrators' and d.directory_name='Confluence Internal Directory';
```

执行结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/5-1.png)



### 4.日志文件位置


`<confluence-home>/logs/`


### 5.Web路径

`<confluence-installation>/confluence/`

Windows: Confluence默认权限为network service，具有写权限

Linux： Confluence默认权限为confluence，没有写权限


## 0x04 利用思路
---

### 1.修改数据库，实现用户登录

#### (1)修改用户登录口令

利用实例：

查看用户关键信息，命令如下：

```
confluence=> select id,user_name,credential from cwd_user;
```

执行结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/4-1.png)

修改用户test2的口令信息，命令如下：

```
confluence=> UPDATE cwd_user SET credential= '{PKCS5S2}UokaJs5wj02LBUJABpGmkxvCX0q+IbTdaUfxy1M9tVOeI38j95MRrVxWjNCu6gsm' WHERE id = 458755;
```

确认数据库被修改，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-2/4-2.png)


**注：**

`{PKCS5S2}UokaJs5wj02LBUJABpGmkxvCX0q+IbTdaUfxy1M9tVOeI38j95MRrVxWjNCu6gsm`对应的明文为123456

#### (2)修改Personal Access Tokens

使用`Personal Access Tokens`可以实现免密登录

介绍资料：

https://confluence.atlassian.com/bitbucketserver0610/personal-access-tokens-989761177.html
https://developer.atlassian.com/server/confluence/confluence-server-rest-api/
https://docs.atlassian.com/ConfluenceServer/rest/7.11.6/

利用实例：

测试环境下，`Personal Access Tokens`对应表为`AO_81F455_PERSONAL_TOKEN`

查询语句：

```
confluence=> select * from "AO_81F455_PERSONAL_TOKEN";
```

修改Personal Access Tokens，命令如下：

```
confluence=> UPDATE "AO_81F455_PERSONAL_TOKEN" SET "HASHED_TOKEN"= '{PKCS5S2}Deoq/psifhVO0VE8qhJ6prfgOltOdJkeRH4cIxac9NtoXVodRQJciR95GW37gR7/' WHERE "ID" = 4;
```


**注：**


`{PKCS5S2}Deoq/psifhVO0VE8qhJ6prfgOltOdJkeRH4cIxac9NtoXVodRQJciR95GW37gR7/`对应的token为`MjE0NTg4NjQ3MTk2OrQ5JtSJgT/rrRBmCY4zu+N+NaWZ`


### 2.写文件


Web路径：`<confluence-installation>/confluence/`

Windows: Confluence默认权限为network service，具有写权限

Linux： Confluence默认权限为confluence，没有写权限，但可以尝试内存马


## 0x05 小结
---

本文介绍了Confluence在利用上的相关基础知识。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)









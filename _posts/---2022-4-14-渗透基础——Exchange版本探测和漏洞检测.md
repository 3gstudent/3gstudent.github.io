---
layout: post
title: 渗透基础——Exchange版本探测和漏洞检测
---


## 0x00 前言
---

Exchange的版本众多，历史漏洞数量也很多，因此需要通过程序实现版本探测和漏洞检测。本文将要介绍通过Python进行版本探测的两种方法，介绍漏洞检测的实现细节，开源代码。

## 0x01 简介
---

本文将要介绍以下内容：

- 实现思路
- 实现细节
- 开源代码

## 0x02 实现思路
---

### 1.版本识别

#### (1)获得精确版本(Build number)

访问EWS接口，在Response Headers中的`X-OWA-Version`可以获得精确版本，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-14/2-1.png)

优点：精确版本(Build number)能够对应到具体的发布日期

缺点：方法不通用，部分旧的Exchange版本不支持

#### (2)获得粗略版本

访问OWA接口，在回显内容可以获得粗略版本，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-14/2-2.png)

优点：方法通用

缺点：粗略版本无法对应到准确的发布日期，只能对应到一个区间

综上，在版本识别上，首先尝试获得精确版本，如果无法获得，再尝试获得粗略版本

获得版本号后，可以去官网查询对应的Exchange版本和发布日期，查询地址：https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019

### 2.漏洞检测

Exchange的漏洞详情可通过访问`https://msrc.microsoft.com/update-guide/vulnerability/<CVE>`查看，例如：

CVE-2020-0688对应的URL：https://msrc.microsoft.com/update-guide/vulnerability/CVE-2020-0688

在漏洞检测上，可以将补丁时间作为判定依据，如果识别到的Exchange版本发布日期低于某个补丁日期，那么判定该Exchange存在该补丁中描述的漏洞。

## 0x03 实现细节
---

### 1.版本识别

访问EWS接口获得版本的实现代码：

```
url1 = "https://" + host + "/ews"
req = requests.get(url1, headers = headers, verify=False)
if "X-OWA-Version" in req.headers:
    version = req.headers["X-OWA-Version"]
    print(version)
```

访问OWA接口获得版本的实现代码：

```
url2 = "https://" + host + "/owa"
req = requests.get(url2, headers = headers, verify=False)
pattern_version = re.compile(r"/owa/auth/(.*?)/themes/resources/favicon.ico")
version = pattern_version.findall(req.text)[0]
print(version)
```

获得版本号后，需要同已知的版本信息作匹配。为了提高效率，可以选择将已知的版本信息存储在列表中，元素包括Exchange版本，发布时间和版本号(Build number)

首先从[官网](https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)复制已知的版本信息，再通过字符串替换的方式将版本信息存储在列表中

在版本匹配时，需要区别精确版本和粗略版本，精确版本可以对应唯一的结果，而粗略版本需要筛选出所有可能的结果

Build number格式示例：`15.1.2375.24`

粗略版本格式示例：`15.1.2375`

粗略版本的筛选方法：

对Build number字符串进行截取，去除最后一个字符"."后面的数据，同粗略版本进行数据对比，输出所有结果

代码示例：

```
versionarray = [
["Exchange Server 2019 CU11 Mar22SU", "March 8, 2022", "15.2.986.22"],
["Exchange Server 2019 CU11 Jan22SU", "January 11, 2022", "15.2.986.15"],
["Exchange Server 2019 CU11 Nov21SU", "November 9, 2021", "15.2.986.14"],
["Exchange Server 2019 CU11 Oct21SU", "October 12, 2021", "15.2.986.9"],
["Exchange Server 2019 CU11", "September 28, 2021", "15.2.986.5"],
["Exchange Server 2019 CU10 Mar22SU", ""March 8, 2022", "15.2.922.27"]
]
version="15.2.986"
for value in versionarray:
    if version in value[2][:value[2].rfind(".")]:
        print("[+] Version: " + value[2])
       	print("    Product: " + value[0])
        print("    Date: " + value[1])
```

### 2.漏洞检测

将补丁时间作为判定依据，同样为了提高效率，将已知的漏洞信息存储的列表中，元素包括发布时间和漏洞编号

为了便于比较时间，需要改变时间格式，例如将`September 28, 2021`修改成`09/28/2021`

代码示例：

```
vularray = [
["CVE-2020-0688", "02/11/2020"],
["CVE-2021-26855+CVE-2021-27065", "03/02/2021"],
["CVE-2021-28482", "04/13/2021"]
]
date="03/01/2021"
for value in vularray:
    if (date.split('/')[2] <= value[1].split('/')[2]) & (date.split('/')[1] <= value[1].split('/')[1]) & (date.split('/')[0] < value[1].split('/')[0]):
        print("[+] " + value[0] + ", " + value[1])
```

## 0x04 开源代码
---

由于代码内容较长，完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Exchange_GetVersion_MatchVul.py

版本数据库的日期为`03/21/2022`

漏洞信息包括以下编号：

- CVE-2020-0688
- CVE-2021-26855+CVE-2021-27065
- CVE-2021-28482
- CVE-2021-34473+CVE-2021-34523+CVE-2021-31207
- CVE-2021-31195+CVE-2021-31196
- CVE-2021-31206
- CVE-2021-42321

代码能够自动识别出精确版本，如果无法识别，改为识别粗略版本，标记出所有匹配的漏洞

## 0x05 小结
---

本文介绍了通过Python进行Exchange版本探测的两种方法，介绍实现细节，开源代码，作为一个很好的学习示例。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



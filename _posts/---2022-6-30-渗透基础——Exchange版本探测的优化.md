---
layout: post
title: 渗透基础——Exchange版本探测的优化
---


## 0x00 前言
---

在上篇文章《渗透基础——Exchange版本探测和漏洞检测》介绍了通过Python进行版本探测的两种方法，在版本识别上，首先从[官网](https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)获得已知的版本信息，将版本信息存储在列表中，然后通过字符串匹配的方式获得Exchange版本的详细信息。开源的代码[Exchange_GetVersion_MatchVul.py](https://github.com/3gstudent/Homework-of-Python/blob/master/Exchange_GetVersion_MatchVul.py)反馈很好。但是这个方法存在一个缺点：需要定期访问[官网](https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)，手动更新扫描脚本中的版本信息列表。
为了进一步提高效率，本文介绍另外一种实现方法，通过访问[官网](https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)，从返回数据中直接提取出详细的版本信息，优点是不再需要定期更新脚本。

## 0x01 简介
---

本文将要介绍以下内容：

- 通过BeautifulSoup解析网页数据
- 实现细节
- 开源代码

## 0x02 通过BeautifulSoup解析网页数据
---

BeautifulSoup是一个可以从HTML或XML文件中提取数据的Python库，可以提高开发效率

安装：

```
pip install bs4
```

### 1.基本使用

在Python实现上，需要先通过requests库获取网页内容，再调用BeautifulSoup进行解析

测试代码：

```
from bs4 import BeautifulSoup
import requests
import urllib3
urllib3.disable_warnings()
url = "https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019"

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36",
} 
response = requests.get(url, verify=False, headers=headers)
soup = BeautifulSoup(response.text, features="html.parser")
print(soup.prettify())
```

以上代码将会访问https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019，将网页数据交由BeautifulSoup进行优化并显示

执行代码的部分输出结果示例：

       <tr>
        <td>
         Â Â Â
         <a data-linktype="external" href="https://support.microsoft.com/help/5014261">
          Exchange Server 2019 CU12 May22SU
         </a>
        </td>
        <td>
         May 10, 2022
        </td>
        <td style="text-align: center;">
         15.2.1118.9
        </td>
        <td style="text-align: center;">
         15.02.1118.009
        </td>
       </tr>
       <tr>
        <td>
         <a data-linktype="external" href="https://www.microsoft.com/download/details.aspx?familyID=a149e06c-62f4-4b62-adf8-7d382223a239">
          Exchange Server 2019 CU12 (2022H1)
         </a>
        </td>
        <td>
         April 20, 2022
        </td>
        <td style="text-align: center;">
         15.2.1118.7
        </td>
        <td style="text-align: center;">
         15.02.1118.007
        </td>
       </tr>

对于以上结果，每个`"tr"`节点对应一个版本信息，子节点`"td"`为具体的版本细节

### 2.只筛选出`"tr"`节点的内容

测试代码：

```
from bs4 import BeautifulSoup
import requests
import urllib3
urllib3.disable_warnings()
url = "https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019"

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36",
} 
response = requests.get(url, verify=False, headers=headers)
soup = BeautifulSoup(response.text, features="html.parser")
for tag in soup.find_all('tr'):
	print(tag)
	print("---")
```

执行代码的部分输出结果示例：

```
<tr>
<td>   <a data-linktype="external" href="https://support.microsoft.com/help/5014261">Exchange Server 2019 CU12 May22SU</a></td>
<td>May 10, 2022</td>
<td style="text-align: center;">15.2.1118.9</td>
<td style="text-align: center;">15.02.1118.009</td>
</tr>
---
<tr>
<td><a data-linktype="external" href="https://www.microsoft.com/download/details.aspx?familyID=a149e06c-62f4-4b62-adf8-7d382223a239">Exchange Server 2019 CU12 (2022H1)</a></td>
<td>April 20, 2022</td>
<td style="text-align: center;">15.2.1118.7</td>
<td style="text-align: center;">15.02.1118.007</td>
</tr>
---
<tr>
<td>   <a data-linktype="external" href="https://support.microsoft.com/help/5014261">Exchange Server 2019 CU11 May22SU</a></td>
<td>May 10, 2022</td>
<td style="text-align: center;">15.2.986.26</td>
<td style="text-align: center;">15.02.0986.026</td>
</tr>
```

接下来，尝试去除无效数据

### 3.提取出版本信息

测试代码：

```
from bs4 import BeautifulSoup
import requests
import urllib3
urllib3.disable_warnings()
url = "https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019"

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36",
} 
response = requests.get(url, verify=False, headers=headers)
soup = BeautifulSoup(response.text, features="html.parser")
for tag in soup.find_all('tr'):
	for string in tag.stripped_strings:
		print((string))
	print("---")
```

执行代码的部分输出结果示例：

```
---
Â Â Â
Exchange Server 2019 CU12 May22SU
May 10, 2022
15.2.1118.9
15.02.1118.009
---
Exchange Server 2019 CU12 (2022H1)
April 20, 2022
15.2.1118.7
15.02.1118.007
---
Â Â Â
Exchange Server 2019 CU11 May22SU
May 10, 2022
15.2.986.26
15.02.0986.026
---
Â Â Â
Exchange Server 2019 CU11 Mar22SU
March 8, 2022
15.2.986.22
15.02.0986.022
---
```

接下来，可以尝试对精确版本进行匹配

### 4.精确匹配版本

测试代码：

```
from bs4 import BeautifulSoup
import requests
import urllib3
urllib3.disable_warnings()
url = "https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019"

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36",
} 
response = requests.get(url, verify=False, headers=headers)
soup = BeautifulSoup(response.text, features="html.parser")

version = "15.2.986.26"
for tag in soup.find_all('tr'):
	if version in tag.stripped_strings:
		print("[+] Exchange Information")
		for versiondata in tag.stripped_strings:
			if (len(versiondata)==5):
				continue
			print("    " + versiondata)
```

执行代码的输出结果示例：

```
[+] Exchange Information
    Exchange Server 2019 CU11 May22SU
    May 10, 2022
    15.2.986.26
    15.02.0986.026
```

对于Exchange较老的版本，无法获得准确的版本号，所以还需要实现粗略匹配版本的功能

### 5.粗略匹配版本

测试代码：

```
from bs4 import BeautifulSoup
import requests
import urllib3
urllib3.disable_warnings()
url = "https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019"

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36",
} 
response = requests.get(url, verify=False, headers=headers)
soup = BeautifulSoup(response.text, features="html.parser")

version = "15.2.986"
for tag in soup.find_all('tr'):
	if version in tag.text:
		print("[+] Exchange Information")
		for versiondata in tag.stripped_strings:
			if (len(versiondata)==5):
				continue
			print("    " + versiondata)
```

执行代码的输出结果示例：

```
[+] Exchange Information
    Exchange Server 2019 CU11 May22SU
    May 10, 2022
    15.2.986.26
    15.02.0986.026
[+] Exchange Information
    Exchange Server 2019 CU11 Mar22SU
    March 8, 2022
    15.2.986.22
    15.02.0986.022
[+] Exchange Information
    Exchange Server 2019 CU11 Jan22SU
    January 11, 2022
    15.2.986.15
    15.02.0986.015
[+] Exchange Information
    Exchange Server 2019 CU11 Nov21SU
    November 9, 2021
    15.2.986.14
    15.02.0986.014
[+] Exchange Information
    Exchange Server 2019 CU11 Oct21SU
    October 12, 2021
    15.2.986.9
    15.02.0986.009
[+] Exchange Information
    Exchange Server 2019 CU11
    September 28, 2021
    15.2.986.5
    15.02.0986.005
```

### 6.提取出网页数据时间

为了能够准确获得版本信息，这里还需要提取出网页数据的更新时间

标记网页数据时间的位置：

       <li>
        <time aria-label="Article review date" class="is-invisible" data-article-date="" data-article-date-source="git" datetime="2022-06-29T22:54:00Z">
         06/29/2022
        </time>
       </li>


定位该时间的代码：

```
print(soup.find_all('time'))
```

执行代码的输出结果示例：

```
[<time aria-label="Article review date" class="is-invisible" data-article-date="" data-article-date-source="git" datetime="2022-06-29T22:54:00Z">06/29/2022</time>]
```

提取出时间的代码：

```
print(soup.find('time').text)
```

执行代码的输出结果示例：

```
06/29/2022
```

结合以上信息，我们可以写出新的识别Exchange版本的代码，通过从[官网](https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)读取数据信息来获得准确的版本，考虑自动化判断多个目标的情况下，为了避免多次访问网站读取数据信息，在代码结构上做了适当优化，只需访问一次https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019，将网页结果保存在变量中。代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Exchange_GetVersion_ParseFromWebsite.py


考虑到内网无法访问[官网](https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)的情况，实现了一个从本地解析网页文件来获得准确的版本，代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Exchange_GetVersion_ParseFromFile.py

可以先访问[官网](https://docs.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates?view=exchserver-2019)并将网页内容保存为exchange.data，再执行脚本Exchange_GetVersion_ParseFromFile.py即可

## 0x03 小结
---

本文介绍了在Exchange版本识别上的优化方法，可以不必手动更新扫描脚本中的版本信息列表，开源代码[Exchange_GetVersion_ParseFromWebsite.py](https://github.com/3gstudent/Homework-of-Python/blob/master/Exchange_GetVersion_ParseFromWebsite.py)和[Exchange_GetVersion_ParseFromFile.py](https://github.com/3gstudent/Homework-of-Python/blob/master/Exchange_GetVersion_ParseFromFile.py)


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



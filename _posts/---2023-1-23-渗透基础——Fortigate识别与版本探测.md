---
layout: post
title: 渗透基础——Fortigate识别与版本探测
---


## 0x00 前言
---

Fortigate的识别需要区分管理页面和VPN登陆页面，版本探测需要根据页面特征提取特征，根据特征匹配出精确的版本，本文将要介绍通过Python实现Fortigate识别与版本探测的方法，开源代码。

## 0x01 简介
---

本文将要介绍以下内容：

- 实现思路
- 实现细节
- 开源代码

## 0x02 实现思路
---

### 1.Fortigate的识别

可通过跳转的URL进行区分

管理页面跳转的url：`/login?redir=%2F`

vpn登陆页面跳转的url：`/remote/login?lang=en`

### 2.版本探测

页面源码中存在32位的16进制字符串可以作为版本识别的特征，每个版本对应不同的32位字符串

## 0x03 实现细节
---

### 1.Fortigate的识别

这里的方法是直接访问IP，根据页面返回结果进行判断

#### (1)管理页面

在返回结果中就能获得32位的16进制字符串

#### (2)vpn登陆页面

返回的内容为跳转地址，需要解析出跳转地址重新构造URL并访问，在返回结果中获得32位的16进制字符串

返回跳转地址的内容示例：

```
200
<html><script type="text/javascript">
if (window!=top) top.location=window.location;top.location="/remote/login";
</script></html>
```

因为跳转的url不固定，这里可以通过正则匹配取出需要跳转的url，示例代码：

```
redirect_name = re.compile(r"top.location=\"(.*?)\"")
redirect = redirect_name.findall(response.text)
print("    URL: " + redirect[0])
```

**注：**

在判断版本时无法在`requests`模块中使用`allow_redirects=False`参数来控制是否重定向，原因如下：

使用`requests`模块时，如果使用`allow_redirects=False`参数，只有在返回状态码为`301`或`302`时，才会关闭重定向，这里Fortigate返回的状态码为`200`，所以`allow_redirects=False`参数不起作用


### 2.版本探测

在实际测试过程中，不同版本的Fortigate，虽然都会返回32位16进制字符，但是格式不同，为了提高匹配的效率，减少工作量，这里在正则匹配时选择直接匹配32位的16进制字符，示例代码如下：

```
x = re.search('[0-9a-f]{32}', response.text)
print(x.group(0))
```

在实际测试过程中，存在`response.text`的输出为乱码的情况

研究解决方法的过程如下：

输出`response.headers`，示例代码：

```
print(response.headers)
```

返回结果：

```
{'Date': 'Fri, 20 Jan 2023 07:33:09 GMT', 'Server': 'Apache', 'X-Frame-Options': 'SAMEORIGIN', 'Content-Security-Policy': "frame-ancestors 'self'", 'X-XSS-Protection': '1; mode=block', 'Strict-Transport-Security': 'max-age=15552000', 'Vary': 'Accept-encoding', 'Cache-Control': 'no-cache', 'Last-Modified': 'Wed, 03 Nov 2021 22:39:42 GMT', 'Accept-Ranges': 'bytes', 'Content-Length': '2126', 'Keep-Alive': 'timeout=5, max=100', 'Connection': 'Keep-Alive', 'Content-Type': 'text/html', 'Content-Encoding': 'x-gzip'}
```

发现编码格式为`x-gzip`

所以这里可以对`response.text`额外做一次gzip解码，获得原始数据，代码如下：

```
import gzip
result = gzip.decompress(response.content)
print(result)
```

完整的实现代码如下：

```
def GetVersion(url):
    try:  
        response = requests.get(url, headers=headers, verify=False, timeout=10, )
        if response.status_code==200 and "top.location=" in response.text:
            print("[*] Try to redirect")
            redirect_name = re.compile(r"top.location=\"(.*?)\"")
            redirect = redirect_name.findall(response.text)
            print("    URL: " + redirect[0])
        
            url1 = url + redirect[0]
            response = requests.get(url1, headers=headers, verify=False, timeout=10)          
            if re.search("[0-9a-f]{32}", response.text):
                hash = re.search('[0-9a-f]{32}', response.text)
                print("[+] " + url)
                print("    Mode: SSL Vpn Client")
                print("    Hash: " + hash.group(0))                
            else:
                print("[-] " + url)
                print("Maybe an old version of Fortigate")
                print(response.status_code)
                print(response.content)

        else:
            if re.search("[0-9a-f]{32}", response.text) is None:
                result = gzip.decompress(response.content)
                hash = re.search('[0-9a-f]{32}', result.decode('utf8'))
            else:
                hash = re.search('[0-9a-f]{32}', response.text)
            print("[+] " + url)
            print("    Mode: Mode: Admin Management")
            print("    Hash: " + hash.group(0))                    
    except:
        print("[-] " + url)
        print("Maybe not Fortigate")
        print(sys.exc_info())
```

**注：**

如果遇到通过浏览器访问SSL Vpn Client页面提示`ERR_SSL_VERSION_OR_CIPHER_MISMATCH`的错误时，程序将返回如下结果：

```
(<class 'requests.exceptions.SSLError'>, SSLError(MaxRetryError("HTTPSConnectionPool(host='192.168.1.1', port=4443): Max retries exceeded with url: / (Caused by SSLError(SSLError(1, '[SSL: TLSV1_ALERT_PROTOCOL_VERSION] tlsv1 alert protocol version (_ssl.c:997)')))")), <traceback object at 0x7f4715bd4780>)
```

解决方法：

改用Python2即可

## 0x04 开源代码
---

完整的实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Python/blob/master/Fortigate_GetVersion.py

代码支持区分管理页面和VPN登陆页面，提供了VM版本的指纹库作为示例，代码能够从页面自动提取出指纹特征，同指纹库进行比对，识别出精确的版本

## 0x05 小结
---

本文介绍了通过Python实现Fortigate识别与版本探测的方法，介绍实现细节，开源代码，作为一个很好的学习示例。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



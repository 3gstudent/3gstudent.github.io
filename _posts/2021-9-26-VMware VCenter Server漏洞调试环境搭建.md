---
layout: post
title: VMware VCenter Server漏洞调试环境搭建
---


## 0x00 前言
---

本文记录从零开始搭建VMware VCenter Server漏洞调试环境的细节。

## 0x01 简介
---

本文将要介绍以下内容：

- 下载vCenter上的文件
- vCenter服务器开启调试模式
- 本地使用IDEA进行远程调试

## 0x02 下载vCenter上的文件
---

为了能够从vCenter上下载文件，这里选择通过SSH连接的方式实现文件下载

### 1.开启SSH

通常可选择以下两种方法：

#### (1)通过浏览器配置

访问`https://<url>:5480`

在`Access`页面下进行开启，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/2-1.png)

#### (2)通过虚拟机配置

访问虚拟机登录页面，按`F2`进入配置页面，在`Troubleshooting Mode Options`下进行开启，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/2-2.png)

### 2.切换到Bash shell

使用SSH登录至vCenter时，默认为Appliance Shell，需要输入`shell`命令才能进入Bash shell

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/2-3.png)

这就导致了无法直接使用scp等命令进行文件上传和下载

这里需要将默认的Appliance Shell切换到Bash shell，方法如下：

(1)使用SSH登录至vCente

(2)输入`shell`命令进入Bash shell

(3)输入以下命令设置默认环境：

```
chsh -s /bin/bash root
```

如果返回结果如下：

```
You are required to change your password immediately (password expired)
chsh: PAM: Authentication token is no longer valid; new one required
```

表示root密码已过期

可以使用passwd命令更改root密码，命令如下：

```
passwd root
```

**注：**

设置默认为Appliance Shell的命令如下：

```
chsh -s /bin/appliancesh root
```

至此，可以通过SSH连接的方式实现文件上传和下载

## 0x03 vCenter服务器开启调试模式
---

首先需要确定待调试的进程，不同漏洞对应的进程不同，例如

CVE-2021-21985对应的进程为vsphere-ui.launcher，CVE-2021-22005对应的进程为vmware-analytics.launcher

下面介绍两种开启调试的方法

#### (1)调试vsphere-ui.launcher

修改文件`/etc/vmware/vmware-vmon/svcCfgfiles/vsphere-ui.json`

将以下内容的注释取消：

```
        //"-Xdebug",
        //"-Xnoagent",
        //"-Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8002",
```

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/3-1.png)

重新启动vsphere-ui服务：

```
service-control --restart vsphere-ui
```

打开防火墙：

```
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
```

验证vsphere-ui启动参数是否修改成功，执行命令：

```
ps -aux | grep vsphere-ui.launcher
```

得到内容：

```
/usr/java/jre-vmware/bin/vsphere-ui.launcher -Xmx854m -XX:CompressedClassSpaceSize=256m -Xss320k -XX:ParallelGCThreads=1 -Djava.io.tmpdir=/usr/lib/vmware-vsphere-ui/server/work/tmp -Dorg.eclipse.virgo.kernel.home=/usr/lib/vmware-vsphere-ui/server -DPS_BASEDIR=/storage/vsphere-ui/ -Declipse.ignoreApp=true -Dcatalina.base=/usr/lib/vmware-vsphere-ui/server -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/vmware/vsphere-ui/ -XX:ErrorFile=/var/log/vmware/vsphere-ui/java_error%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintReferenceGC -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=1024K -XX:-OmitStackTraceInFastThrow -Xloggc:/var/log/vmware/vsphere-ui/vsphere-ui-gc.log -Xdebug -Xnoagent -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8002 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9876 -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dosgi.service.lookup.retry.count=1 -Djava.security.properties=/etc/vmware/java/vmware-override-java.security -Djava.ext.dirs=/usr/java/jre-vmware/lib/ext:/usr/java/packages/lib/ext:/opt/vmware/jre_ext/ -Dorg.osgi.framework.system.packages.extra=sun.misc -Dsun.zip.disableMemoryMapping=true -Dui.component.name=vsphere-ui -Dvlsi.client.vecs.certstore=false -DisFling=false -Dorg.apache.tomcat.websocket.DISABLE_BUILTIN_EXTENSIONS=true -Dlogback.configurationFile=/usr/lib/vmware-vsphere-ui/server/conf/serviceability.xml -Dlogs.dir=/var/log/vmware/vsphere-ui/logs/ -Dhttps.port=5443 -Dhttp.port=5090 -Dshutdown.port=-1 -classpath /usr/lib/vmware-vsphere-ui/server/bootstrap/server-launcher.jar:/usr/lib/vmware-vsphere-ui/server/bin/bootstrap.jar:/usr/lib/vmware-vsphere-ui/server/bin/tomcat-juli.jar com.vmware.vise.launcher.tomcat.TomcatLauncher start 
```

证实vsphere-ui的启动参数修改成功

#### (2)调试vmware-analytics.launcher

定位vmware-analytics.launcher，执行命令：

```
ps -aux | grep vmware-analytics.launcher
```

得到默认的启动参数：

```
root      2434 12.9  2.5 2730380 420720 ?      Sl   07:41   1:07 /usr/java/jre-vmware/bin/vmware-analytics.launcher -Xmx139m -XX:CompressedClassSpaceSize=64m -Xss256k -XX:ParallelGCThreads=1 -Dorg.apache.catalina.startup.EXIT_ON_INIT_FAILURE=TRUE -Danalytics.logDir=/var/log/vmware/analytics -Danalytics.dataDir=/storage/analytics -Danalytics.deploymentNodeTypeFile=/etc/vmware/deployment.node.type -Danalytics.buildInfoFile=/etc/vmware/.buildInfo -Danalytics.agentsDir=/etc/vmware-analytics/agents -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/vmware/analytics -XX:ErrorFile=/var/log/vmware/analytics/java_error%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintReferenceGC -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=1024K -Xloggc:/var/log/vmware/analytics/vmware-analytics-gc.log -Djava.security.properties=/etc/vmware/java/vmware-override-java.security -Djava.ext.dirs=/usr/java/jre-vmware/lib/ext:/usr/java/packages/lib/ext:/opt/vmware/jre_ext/ -classpath /etc/vmware-analytics:/usr/lib/vmware-analytics/lib/*:/usr/lib/vmware-analytics/lib:/usr/lib/vmware/common-jars/tomcat-embed-core-8.5.37.jar:/usr/lib/vmware/common-jars/tomcat-annotations-api-8.5.37.jar:/usr/lib/vmware/common-jars/antlr-2.7.7.jar:/usr/lib/vmware/common-jars/antlr-runtime.jar:/usr/lib/vmware/common-jars/aspectjrt.jar:/usr/lib/vmware/common-jars/bcprov-jdk16-145.jar:/usr/lib/vmware/common-jars/commons-codec-1.6.jar:/usr/lib/vmware/common-jars/commons-collections-3.2.2.jar:/usr/lib/vmware/common-jars/commons-collections4-4.1.jar:/usr/lib/vmware/common-jars/commons-compress-1.8.jar:/usr/lib/vmware/common-jars/commons-io-2.1.jar:/usr/lib/vmware/common-jars/commons-lang-2.6.jar:/usr/lib/vmware/common-jars/commons-lang3-3.4.jar:/usr/lib/vmware/common-jars/commons-logging-1.1.3.jar:/usr/lib/vmware/common-jars/commons-pool-1.6.jar:/usr/lib/vmware/common-jars/custom-rolling-file-appender-1.0.jar:/usr/lib/vmware/common-jars/featureStateSwitch-1.0.0.jar:/usr/lib/vmware/common-jars/guava-18.0.jar:/usr/lib/vmware/common-jars/httpasyncclient-4.1.3.jar:/usr/lib/vmware/common-jars/httpclient-4.5.3.jar:/usr/lib/vmware/common-jars/httpcore-4.4.6.jar:/usr/lib/vmware/common-jars/httpcore-nio-4.4.6.jar:/usr/lib/vmware/common-jars/httpmime-4.5.3.jar:/usr/lib/vmware/common-jars/jackson-annotations-2.9.5.jar:/usr/lib/vmware/common-jars/jackson-core-2.9.5.jar:/usr/lib/vmware/common-jars/jackson-databind-2.9.5.jar:/usr/lib/vmware/common-jars/jna.jar:/usr/lib/vmware/common-jars/log4j-1.2.16.jar:/usr/lib/vmware/common-jars/log4j-core-2.8.2.jar:/usr/lib/vmware/common-jars/log4j-api-2.8.2.jar:/usr/lib/vmware/common-jars/platform.jar:/usr/lib/vmware/common-jars/slf4j-api-1.7.2.jar:/usr/lib/vmware/common-jars/slf4j-log4j12-1.7.2.jar:/usr/lib/vmware/common-jars/spring-aop-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-beans-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-context-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-core-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-expression-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-web-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-webmvc-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/velocity-1.7.jar com.vmware.ph.phservice.service.Main ph-properties-loader.xml ph-featurestate.xml phservice.xml ph-web.xml
```

修改启动参数，加入调试参数：

```
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8003
```

在末尾使用`&`，将进程放置到后台运行

在重新启动前，先需要停止服务vmware-analytics：

```
service-control --stop vmware-analytics
```

使用新的参数启动vmware-analytics.launcher：

```
/usr/java/jre-vmware/bin/vmware-analytics.launcher -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8003 -Xmx139m -XX:CompressedClassSpaceSize=64m -Xss256k -XX:ParallelGCThreads=1 -Dorg.apache.catalina.startup.EXIT_ON_INIT_FAILURE=TRUE -Danalytics.logDir=/var/log/vmware/analytics -Danalytics.dataDir=/storage/analytics -Danalytics.deploymentNodeTypeFile=/etc/vmware/deployment.node.type -Danalytics.buildInfoFile=/etc/vmware/.buildInfo -Danalytics.agentsDir=/etc/vmware-analytics/agents -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/vmware/analytics -XX:ErrorFile=/var/log/vmware/analytics/java_error%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintReferenceGC -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=1024K -Xloggc:/var/log/vmware/analytics/vmware-analytics-gc.log -Djava.security.properties=/etc/vmware/java/vmware-override-java.security -Djava.ext.dirs=/usr/java/jre-vmware/lib/ext:/usr/java/packages/lib/ext:/opt/vmware/jre_ext/ -classpath /etc/vmware-analytics:/usr/lib/vmware-analytics/lib/*:/usr/lib/vmware-analytics/lib:/usr/lib/vmware/common-jars/tomcat-embed-core-8.5.37.jar:/usr/lib/vmware/common-jars/tomcat-annotations-api-8.5.37.jar:/usr/lib/vmware/common-jars/antlr-2.7.7.jar:/usr/lib/vmware/common-jars/antlr-runtime.jar:/usr/lib/vmware/common-jars/aspectjrt.jar:/usr/lib/vmware/common-jars/bcprov-jdk16-145.jar:/usr/lib/vmware/common-jars/commons-codec-1.6.jar:/usr/lib/vmware/common-jars/commons-collections-3.2.2.jar:/usr/lib/vmware/common-jars/commons-collections4-4.1.jar:/usr/lib/vmware/common-jars/commons-compress-1.8.jar:/usr/lib/vmware/common-jars/commons-io-2.1.jar:/usr/lib/vmware/common-jars/commons-lang-2.6.jar:/usr/lib/vmware/common-jars/commons-lang3-3.4.jar:/usr/lib/vmware/common-jars/commons-logging-1.1.3.jar:/usr/lib/vmware/common-jars/commons-pool-1.6.jar:/usr/lib/vmware/common-jars/custom-rolling-file-appender-1.0.jar:/usr/lib/vmware/common-jars/featureStateSwitch-1.0.0.jar:/usr/lib/vmware/common-jars/guava-18.0.jar:/usr/lib/vmware/common-jars/httpasyncclient-4.1.3.jar:/usr/lib/vmware/common-jars/httpclient-4.5.3.jar:/usr/lib/vmware/common-jars/httpcore-4.4.6.jar:/usr/lib/vmware/common-jars/httpcore-nio-4.4.6.jar:/usr/lib/vmware/common-jars/httpmime-4.5.3.jar:/usr/lib/vmware/common-jars/jackson-annotations-2.9.5.jar:/usr/lib/vmware/common-jars/jackson-core-2.9.5.jar:/usr/lib/vmware/common-jars/jackson-databind-2.9.5.jar:/usr/lib/vmware/common-jars/jna.jar:/usr/lib/vmware/common-jars/log4j-1.2.16.jar:/usr/lib/vmware/common-jars/log4j-core-2.8.2.jar:/usr/lib/vmware/common-jars/log4j-api-2.8.2.jar:/usr/lib/vmware/common-jars/platform.jar:/usr/lib/vmware/common-jars/slf4j-api-1.7.2.jar:/usr/lib/vmware/common-jars/slf4j-log4j12-1.7.2.jar:/usr/lib/vmware/common-jars/spring-aop-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-beans-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-context-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-core-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-expression-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-web-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/spring-webmvc-4.3.20.RELEASE.jar:/usr/lib/vmware/common-jars/velocity-1.7.jar com.vmware.ph.phservice.service.Main ph-properties-loader.xml ph-featurestate.xml phservice.xml ph-web.xml &
```

**注：**

如果想重新调试，需要做以下操作：
1. 停止服务：`vmware-analytics：service-control --stop vmware-analytics`
2. 结束进程：`kill -KILL pid`
3. 以新的参数启动vmware-analytics.launcher。如果想要以正常的参数启动，只需要重新启动服务vmware-analytics：`service-control --start vmware-analytics`


## 0x04 本地使用IDEA进行远程调试
---

### 1.下载jar文件

本地使用IDEA进行远程调试时，本地和远程的代码需要保持一致，也就是说，我们需要拿到vCenter服务器上待调试进程加载的jar文件


vCenter服务器的相关jar文件位于以下两个路径：

- /etc
- /usr/lib

可以通过以下命令将所有vCenter服务器相关的jar文件复制到同一路径下，再统一进行下载：

```
mkdir /tmp/jar
find /etc/ -name "*.jar" |xargs -n1 -i cp {} /tmp/jar
find /usr/lib/ -name "*.jar" |xargs -n1 -i cp {} /tmp/jar
```

**注：**

如果想要查找所有jar文件中的内容，可以通过以下命令将所有vCenter服务器相关的jar文件解压至同一路径：

```
find /etc -name "*.jar" | xargs -n 1 unzip -d /tmp/data/
find /usr/lib/ -name "*.jar" | xargs -n 1 unzip -d /tmp/data/
```

将所有vCenter服务器相关的jar文件统一下载后，保存文件夹为`c:\testjar\`


### 2.批量导入jar文件

新建java工程，依次选择`File`->`Project Structure...`，在`Libraries`下选择`New Project Library`->`Java`，设置为`c:\testjar\`，配置后的结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/4-1.png)

### 3.添加断点

在`External Libraries`->`testjar`下面打开.class文件，在合适的位置添加断点，示例如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/4-2.png)

### 4.设置远程调试参数

顶部菜单栏选择`Add Configuration...`，在弹出的页面中选择`Remote JVM Debug`，填入远程调试参数，需要修改以下参数：

- Host
- Port

使用的JDK选择`JDK 5-8`

示例如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/4-3.png)

### 5.开启Debug模式

回到IDEA主页面，选择刚才的配置文件，点击Debug图标(快捷键Shift+F9)


如果远程调试执行成功，断点图标会发生变化，增加一个对号，示例如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2021-9-26/4-4.png)

此时，Console页面显示如下：

```
Connected to the target VM, address: '<host>:<port>', transport: 'socket'
```


## 0x05 小结
---

在我们搭建好VMware VCenter Server漏洞调试环境后，接下来就可以着手对漏洞进行研究学习。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




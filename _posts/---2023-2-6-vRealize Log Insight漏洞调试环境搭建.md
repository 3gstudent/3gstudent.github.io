---
layout: post
title: vRealize Log Insight漏洞调试环境搭建
---


## 0x00 前言
---

本文记录从零开始搭建vRealize Log Insight漏洞调试环境的细节。

## 0x01 简介
---

本文将要介绍以下内容：

- vRealize Log Insight安装
- vRealize Log Insight漏洞调试环境配置
- 数据库操作

## 0x02 vRealize Log Insight安装
---

参考资料： https://docs.vmware.com/en/vRealize-Log-Insight/index.html

### 1.下载OVA文件

下载页面：https://customerconnect.vmware.com/evalcenter?p=vr-li

下载前需要先注册用户，之后选择需要的版本进行下载

### 2.安装

#### (1)在VMware Workstation中导入OVA文件

#### (2)配置

访问配置页面`https://<ip>/`

选择`Starting New Deployment`，设置admin用户口令

### 3.开启远程调试功能

#### (1)查看所有服务的状态

```
systemctl status
```

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-2-6/2-1.png)

定位到web相关的服务为`loginsight.service`

#### (2)查看loginsight.service的具体信息

```
systemctl status loginsight.service
```

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-2-6/2-2.png)

定位到服务启动文件：`/usr/lib/loginsight/application/bin/loginsight` 

#### (3)查看进程参数

执行命令：`ps aux|grep java`

返回结果：

```
root      1977  4.7 34.5 4687676 1396852 ?     Sl   00:04   6:55 /usr/lib/loginsight/application/3rd_party/bin/java -Xrs -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/storage/core/loginsight/var/heapdump/li_heapdump.hprof -XX:ErrorFile=/storage/core/loginsight/var/jvm_hs_err_pid.log -Djava.util.logging.config.level=SEVERE -Djdk.tls.ephemeralDHKeySize=2048 -Dorg.bouncycastle.fips.approved_only=false -Djavax.net.ssl.trustStorePassword=changeit -Djdk.http.auth.tunneling.disabledSchemes="" -DLOGINSIGHT_HOME=/usr/lib/loginsight -Dstrata.pgid=1961 -cp /usr/lib/loginsight/application/lib/* -Xmx1972m -Xms1972m -Xss256k -Xmn1024M -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+ScavengeBeforeFullGC -XX:TargetSurvivorRatio=80 -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=15 -XX:ParallelGCThreads=4 -XX:+UseCompressedOops -XX:+OptimizeStringConcat -XX:+AlwaysPreTouch com.vmware.loginsight.daemon.LogInsightDaemon --wait=120
root      2157  0.1  2.0 2685308 84276 ?       Sl   00:04   0:17 /usr/lib/loginsight/application/3rd_party/bin/java -Xms100m -Xmx256m -jar -Dapp.log.home=/storage/var/loginsight /usr/lib/loginsight/application/3rd_party/vI18nManager-logInsight-8.10.latest.jar --server.scheme=https -c --swagger-ui.enable=false
root      2327  0.6 15.1 3577064 612048 ?      Sl   00:04   0:57 /usr/lib/loginsight/application/3rd_party/bin/java -Dnop -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -server -DLOGINSIGHT_HOME=/usr/lib/loginsight -Dorg.bouncycastle.fips.approved_only=false -Dorg.apache.jasper.runtime.BodyContentImpl.LIMIT_BUFFER=true -Djava.awt.headless=true -Dorg.apache.jasper.runtime.JspFactoryImpl.USE_POOL=false -Djdk.tls.rejectClientInitiatedRenegotiation=true -Dorg.apache.el.parser.SKIP_IDENTIFIER_CHECK=true -Djavax.net.ssl.trustStorePassword=changeit -Djava.endorsed.dirs= -classpath /usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82/log4j2/lib/*:/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82/log4j2/conf:/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82/bin/bootstrap.jar:/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82/bin/tomcat-juli.jar -Dcatalina.base=/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82 -Dcatalina.home=/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82 -Djava.io.tmpdir=/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82/temp org.apache.catalina.startup.Bootstrap start
root      3097  1.7 33.0 2860656 1336624 ?     SLl  00:06   2:31 /usr/bin/java -Xloggc:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../logs/gc.log -ea -XX:+UseThreadPriorities -XX:ThreadPriorityPolicy=42 -XX:+HeapDumpOnOutOfMemoryError -Xss256k -XX:StringTableSize=1000003 -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+UseTLAB -XX:+ResizeTLAB -XX:+UseNUMA -XX:+PerfDisableSharedMem -Djava.net.preferIPv4Stack=true -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSWaitDuration=10000 -XX:+CMSParallelInitialMarkEnabled -XX:+CMSEdenChunksRecordAlways -XX:+CMSClassUnloadingEnabled -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -XX:+PrintTenuringDistribution -XX:+PrintGCApplicationStoppedTime -XX:+PrintPromotionFailure -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=10M -Xms1024M -Xmx1024M -Xmn200M -XX:+UseCondCardMark -XX:CompileCommandFile=/storage/core/loginsight/cidata/cassandra/config/hotspot_compiler -javaagent:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jamm-0.3.0.jar -Djava.rmi.server.hostname=192.168.112.156 -Dcassandra.jmx.local.port=7199 -Dcom.sun.management.jmxremote.authenticate=true -Dcom.sun.management.jmxremote.password.file=/storage/core/loginsight/cidata/cassandra/config/jmxremote.password -Djava.library.path=/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/sigar-bin -Dcassandra.consistent.rangemovement=false -XX:OnOutOfMemoryError=kill -9 %p -Dlogback.configurationFile=logback.xml -Dcassandra.logdir=/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../logs -Dcassandra.storagedir=/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../data -Dcassandra-foreground=yes -cp /storage/core/loginsight/cidata/cassandra/config:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../build/classes/main:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../build/classes/thrift:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/airline-0.6.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/antlr-runtime-3.5.2.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/apache-cassandra-3.11.11.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/apache-cassandra-thrift-3.11.11.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/asm-5.0.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/caffeine-2.2.6.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/cassandra-driver-core-3.0.1-shaded.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/commons-cli-1.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/commons-codec-1.9.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/commons-lang3-3.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/commons-math3-3.2.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/compress-lzf-0.8.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/concurrentlinkedhashmap-lru-1.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/concurrent-trees-2.4.0.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/disruptor-3.0.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/ecj-4.4.2.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/guava-18.0.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/HdrHistogram-2.1.9.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/high-scale-lib-1.0.6.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/hppc-0.5.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jackson-annotations-2.9.10.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jackson-core-2.9.10.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jackson-databind-2.9.10.8.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jamm-0.3.0.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/javax.inject-1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jbcrypt-0.3m.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jcl-over-slf4j-1.7.7.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jctools-core-1.2.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jflex-1.6.0.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jna-4.2.2.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/joda-time-2.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/json-simple-1.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/libthrift-0.9.2.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/log4j-api-2.17.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/log4j-core-2.17.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/log4j-over-slf4j-1.7.7.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/log4j-slf4j-impl-2.17.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/logback-classic-1.1.3.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/logback-core-1.1.3.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/lz4-1.3.0.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/metrics-core-3.1.5.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/metrics-jvm-3.1.5.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/metrics-logback-3.1.5.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/netty-all-4.0.44.Final.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/ohc-core-0.4.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/ohc-core-j8-0.4.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/reporter-config3-3.0.3.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/reporter-config-base-3.0.3.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/sigar-1.6.4.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/slf4j-api-1.7.30.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/slf4j-api-1.7.7.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/snakeyaml-1.11.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/snappy-java-1.1.1.7.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/snowball-stemmer-1.3.0.581.1.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/ST4-4.0.8.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/stream-2.5.2.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/thrift-server-0.3.7.jar:/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/../lib/jsr223/*/*.jar: org.apache.cassandra.service.CassandraDaemon
```

结果分析如下：

**进程1977：**

进程参数对应文件：`/usr/lib/loginsight/application/sbin/loginsight-daemon.sh`

对应lib文件夹：`/usr/lib/loginsight/application/lib/`

**进程2157：**

进程参数对应文件：`/usr/lib/loginsight/application/sbin/vI18nManager`

对应lib文件：`/usr/lib/loginsight/application/3rd_party/vI18nManager-logInsight-8.10.latest.jar`

**进程2327：**

进程参数对应文件：`/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82/bin/catalina.sh`

对应lib文件夹：`/usr/lib/loginsight/application/3rd_party/apache-tomcat-8.5.82/`

**进程3097：**

进程参数对应文件：`/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/conf/jvm.options`

对应lib文件夹：`/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/lib/`

对进程3097添加调试参数的方法：

修改文件`/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/conf/jvm.options`

添加内容: `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=1414`

重启服务：`service loginsight restart`

添加防火墙规则：`iptables -I INPUT -p tcp --dport 1414 -j ACCEPT`

## 0x03 数据库操作
---

### 1.重置web登陆用户admin口令

实现文件：`/usr/lib/loginsight/application/sbin/li-reset-admin-passwd.sh`

从文件中可以获得数据库操作的相关信息，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-2-6/3-1.png)

### 2.连接数据库的命令参数

实现文件：`/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/cqlsh-no-pass`

文件内容如下：

```
_bin() {
    local CASSANDRA_BIN=$(find $BASE -name nodetool | sort | tail -n 1)
    if [ ! -e "$CASSANDRA_BIN" ]; then
        echo "ERROR: Unable to locate Cassandra's bin directory!"
        exit 255
    else
        # Echo the bin directory
        echo ${CASSANDRA_BIN%/*}
    fi
    return 0
}

BASE="$(dirname $(readlink -f $0 2>/dev/null) 2>/dev/null)/"
if [ $? != 0  ]; then
    # If mac then no need to worry about symlink
    BASE="$(dirname $0)"
    BASE="${BASE%.*}"
    if [ ! -z "$BASE" ]; then
        BASE="$BASE/";
    fi
elif [ "$BASE" == "/opt/vmware/bin/" ]; then
    BASE="/usr/lib/loginsight/application/lib/apache-cassandra-*"
fi

CASSANDRA_BIN=$(_bin)
if [ $? != 0 ]; then
    echo $CASSANDRA_BIN
    exit 255
fi

user_password=`$CASSANDRA_BIN/credentials-look-up | tr "\n" " " | sed 's/.*"\(.*\)".*"\(.*\)".*/\1\t\2/g'`
cuser=`echo "${user_password}" | cut -f1`
cpassword=`echo "${user_password}" | cut -f2`

$CASSANDRA_BIN/cqlsh -u $cuser -p $cpassword --cqlshrc=/storage/core/loginsight/cidata/cassandra/config/cqlshrc "$@"r
```

从文件中可以获得数据库操作的具体参数：
- 从`$CASSANDRA_BIN/credentials-look-up`中获得用户名口令
- 配置文件为`/storage/core/loginsight/cidata/cassandra/config/cqlshrc`

### 3.连接数据库的用户名口令

实现文件：`/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/credentials-look-up`

执行后返回用户名口令：

```
<cassandra-user value="lisuper" />
<cassandra-password value="slshitn2@S" />
```

实现原理为读取文件`/storage/core/loginsight/config/loginsight-config.xml*`，从xml文件中提取出明文保存的用户名口令

### 4.连接数据库的配置信息

实现文件：`/storage/core/loginsight/cidata/cassandra/config/cqlshrc`

文件内容如下：

```
[connection]
hostname = 127.0.0.1
port = 9042
client_timeout = 120
ssl = true

;[authentication]
;username = cassandra
;password = cassandra

[ssl]
certfile = /storage/core/loginsight/cidata/cassandra/config/cacert.pem
;usercert = cert.pem
;userkey = cert_key.pem
```

综合以上内容，可以总结出连接数据库的两种方法：

#### (1)使用封装好参数的文件

```
/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/cqlsh-no-pass
```

#### (2)使用参数连接

```
/usr/lib/loginsight/application/lib/apache-cassandra-3.11.11/bin/cqlsh -u lisuper -p slshitn2@S --cqlshrc=/storage/core/loginsight/cidata/cassandra/config/cqlshrc
```

从返回结果可以看到数据库使用了CQL(Cassandra Query Language)

查询用户配置的命令：

```
select * from logdb.user;
select * from logdb.user_auth;
```

### 5.界面化操作数据库

这里使用软件TablePlus

需要修改防火墙配置打开端口9042：`iptables -I INPUT -p tcp --dport 9042 -j ACCEPT`

证书文件使用：`/storage/core/loginsight/cidata/cassandra/config/cacert.pem`

连接成功如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-2-6/3-2.png)

## 0x04 小结
---

在我们搭建好vRealize Log Insight漏洞调试环境后，接下来就可以着手对漏洞进行学习。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



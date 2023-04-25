---
layout: post
title: ADAudit Plus漏洞调试环境搭建
---


## 0x00 前言
---

本文记录从零开始搭建ADAudit Plus漏洞调试环境的细节，介绍数据库用户口令的获取方法。

## 0x01 简介
---

本文将要介绍以下内容：

- ADAudit Plus安装
- ADAudit Plus漏洞调试环境配置
- 数据库用户口令获取

## 0x02 ADAudit Plus安装
---

### 1.下载

全版本下载地址：https://archives2.manageengine.com/active-directory-audit/

### 2.安装

安装参考：https://www.manageengine.com/products/active-directory-audit/quick-start-guide-overview.html

### 3.测试

访问https://localhost:8081

## 0x03 ADAudit Plus漏洞调试环境配置
---

方法同Password Manager Pro漏洞调试环境配置基本类似

### 1.开启调试功能

#### (1)定位配置文件

查看java进程的信息，这里分别有两个java进程，对应两个不同的父进程wrapper.exe，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-1/1-1.png)

wrapper.exe的进程参数分别为：

- "C:\Program Files\ManageEngine\ADAudit Plus\bin\Wrapper.exe"  -c  "C:\Program Files\ManageEngine\ADAudit Plus\bin\\..\conf\wrapper.conf"
- "C:\Program Files\ManageEngine\ADAudit Plus\bin\wrapper.exe" -s "C:\Program Files\ManageEngine\ADAudit Plus\apps\dataengine-xnode\conf\wrapper.conf"

这里需要修改的配置文件为`C:\Program Files\ManageEngine\ADAudit Plus\conf\wrapper.conf`

#### (2)修改配置文件添加调试参数

找到启用调试功能的位置：

```
#uncomment the following to enable JPDA debugging
#wrapper.java.additional.3=-Xdebug
#wrapper.java.additional.4=-Xnoagent
#wrapper.java.additional.5=-Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n
```

将其修改为

```
wrapper.java.additional.25=-Xdebug
wrapper.java.additional.26=-Xnoagent
wrapper.java.additional.27=-Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n
```

**注：**

序号需要逐个递增，此处将`wrapper.java.additional.3=-Xdebug`修改为`wrapper.java.additional.25=-Xdebug`

#### (3)重新启动相关进程

关闭进程wrapper.exe和对应的子进程java.exe

在命令行下执行命令：

```
"C:\Program Files\ManageEngine\ADAudit Plus\bin\Wrapper.exe"  -c  "C:\Program Files\ManageEngine\ADAudit Plus\bin\\..\conf\wrapper.conf"
```

### 2.常用jar包位置

路径：`C:\Program Files\ManageEngine\ADAudit Plus\lib`

web功能的实现文件为`AdventNetADAPServer.jar`和`AdventNetADAPClient.jar`

### 3.IDEA设置

设置为`Remote JVM Debug`，远程调试成功如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-1/1-2.png)

## 0x04 数据库用户口令获取
---

默认配置下，ADAudit Plus使用postgresql存储数据，默认配置了两个登录用户：`adap`和`postgres`

### 1.用户`adap`的口令获取

配置文件路径：`C:\Program Files\ManageEngine\ADAudit Plus\conf\database_params.conf`，内容示例：

```
# $Id$
# driver name
drivername=org.postgresql.Driver

# login username for database if any
username=adaudit

# password for the db can be specified here
password=cb26b920b56fed8d085d71f63bdd79c55ea7b98f8794699562c06ea1bedbec52087b394f


# url is of the form jdbc:subprotocol:DataSourceName for eg.jdbc:odbc:WebNmsDB
url=jdbc:postgresql://localhost:33307/adap
#url=jdbc:mysql://localhost:33307/adsm?autoReconnect=true&characterEncoding=utf8
# Minumum Connection pool size
minsize=1

# Maximum Connection pool size
maxsize=20

# transaction Isolation level
#values are Constanst defined in java.sql.connection type supported TRANSACTION_NONE 	0
#Allowed values are TRANSACTION_READ_COMMITTED , TRANSACTION_READ_UNCOMMITTED ,TRANSACTION_REPEATABLE_READ , TRANSACTION_SERIALIZABLE
transaction_isolation=TRANSACTION_READ_COMMITTED


dbadapter=com.adventnet.db.adapter.postgres.PostgresDBAdapter
sqlgenerator=com.adventnet.db.adapter.postgres.PostgresSQLGenerator
exceptionsorterclassname=com.adventnet.db.adapter.postgres.PostgresExceptionSorter
```

其中，password被加密，加解密算法位于：`C:\Program Files\ManageEngine\ADAudit Plus\lib\framework-tools.jar`中的`com.zoho.framework.utils.crypto->CryptoUtil.class`

经过代码分析，得出以下解密方法：

密钥固定保存在`C:\Program Files\ManageEngine\ADAudit Plus\conf\customer-config.xml`，内容示例：

```
<?xml version="1.0" encoding="UTF-8"?><extended-configurations>
    <configuration name="ECTag" value="b1b4a1770dd36f314b0d7753216560f1ecaa4e70fc430dd09ca5b022c87dbc7db8e5bb3756cfe8088d5ccf2c5eaa62a8fa0bac12"/>
    <configuration name="CryptTag" value="8ElrDgofXtbrMAtNQBqy"/>
    <configuration name="mssql" value="">
        <property name="certificate.subject" value="767f95aba2ecb31574a3255bfc966a094181138f2fc4c3989b7b892bda31d3d918483ba2"/>
        <property name="masterkey.password" value="7d439da9842982f608ddf9e2fa26c6cc246e1f057dfb4ab812d794dafaa78e5e6c576f6a"/>
        <property name="certificate.name" value="7df03f3ef5eea4bfac3e6bff91c2837819f8c557179f45e2a4b37fa768a7414811bf4611"/>
        <property name="symmetrickey.name" value="fc9cea33ebb88939f287285956b7e05b08dde996dc81365aef6d3530227889426af8659f"/>
    </configuration>
</extended-configurations>
```

得到密钥：CryptTag为`8ElrDgofXtbrMAtNQBqy`

根据以上得到的密文`cb26b920b56fed8d085d71f63bdd79c55ea7b98f8794699562c06ea1bedbec52087b394f`和密钥`8ElrDgofXtbrMAtNQBqy`，编写解密程序，代码如下：

```
import javax.crypto.Cipher;
import javax.crypto.SecretKey;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.PBEKeySpec;
import javax.crypto.spec.SecretKeySpec;
import java.nio.ByteBuffer;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.security.spec.InvalidKeySpecException;

public class Main {
    private static int INT(char c) {
        return Integer.decode("0x" + c);
    }
    protected static byte[] BASE16_DECODE(String b16str) {
        int len = b16str.length();
        byte[] out = new byte[len / 2];
        int j = 0;

        for(int i = 0; i < len; i += 2) {
            int c1 = INT(b16str.charAt(i));
            int c2 = INT(b16str.charAt(i + 1));
            int bt = c1 << 4 | c2;
            out[j++] = (byte)bt;
        }
        return out;
    }
    protected static String B2S(byte[] bytes) {
        return new String(bytes);
    }
    public static String decryptAES256(String cipherText, String encryptionKey) {
        try {
            byte[] saltBytes = new byte[20];
            ByteBuffer saltbuffer = ByteBuffer.wrap(BASE16_DECODE(cipherText));
            saltbuffer.get(saltBytes, 0, saltBytes.length);
            byte[] cipherbytes = new byte[saltbuffer.capacity() - saltBytes.length];
            saltbuffer.get(cipherbytes);
            SecretKeyFactory KeyFactory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
            PBEKeySpec spec = new PBEKeySpec(encryptionKey.toCharArray(), saltBytes, 65556, 256);
            SecretKey secretkey = KeyFactory.generateSecret(spec);
            SecretKeySpec skc = new SecretKeySpec(secretkey.getEncoded(), "AES");
            Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
            byte[] iv = new byte[cipher.getBlockSize()];
            IvParameterSpec ivpa = new IvParameterSpec(iv);
            cipher.init(2, skc, ivpa);
            byte[] decryptbytes = cipher.doFinal(cipherbytes);
            String decryptedString = B2S(decryptbytes);
            return decryptedString;
        } catch (InvalidKeySpecException | NoSuchAlgorithmException var16) {
            System.out.println("Key generation failed" + var16.getMessage());
            throw new IllegalArgumentException(var16.getMessage(), var16);
        } catch (InvalidKeyException var17) {
            if (var17.getMessage().contains("Illegal key size")) {
                throw new RuntimeException("Possible reason for exception: The jre used should contain unlimited strength jce jars", var17);
            } else {
                throw new IllegalArgumentException(var17.getMessage(), var17);
            }
        } catch (Exception var18) {
            System.out.println("Decryption failed " + var18.getMessage());
            return cipherText;
        }
    }
    public static void main(String[] args) throws Exception, Exception {
        String password = "cb26b920b56fed8d085d71f63bdd79c55ea7b98f8794699562c06ea1bedbec52087b394f";
        String encryptionKey = "8ElrDgofXtbrMAtNQBqy";
        String out  = decryptAES256(password, encryptionKey);
        System.out.println(out);
    }
}
```

程序运行后得到解密结果：`Adaudit@123$`

拼接出数据库的连接命令：`"C:\Program Files\ManageEngine\ADAudit Plus\pgsql\bin\psql" "host=127.0.0.1 port=33307 dbname=adap user=adaudit password=Adaudit@123$"`

连接成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-1/2-1.png)

### 2.用户`postgres`的口令获取

口令硬编码于`C:\Program Files\ManageEngine\ADAudit Plus\lib\AdventnetADAPServer.jar`中的`com.adventnet.sym.adsm.common.server.mssql.tools->ChangeDBServer.class->isDBServerRunning()`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-1/2-2.png)

得到用户`postgres`的口令为`Stonebraker`

拼接出数据库的连接命令：`"C:\Program Files\ManageEngine\ADAudit Plus\pgsql\bin\psql" "host=127.0.0.1 port=33307 dbname=adap user=postgres password=Stonebraker"`

连接成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-1/2-3.png)

一条命令实现连接数据库并执行数据库操作的命令示例：`"C:\Program Files\ManageEngine\ADAudit Plus\pgsql\bin\psql" --command="SELECT * FROM public.aaapassword ORDER BY password_id ASC;" postgresql://postgres:Stonebraker@127.0.0.1:33307/adap`

返回结果示例：

```
 password_id |                           password                           | algorithm |                       salt                                         | passwdprofile_id | passwdrule_id |  createdtime  | factor
-------------+--------------------------------------------------------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------+---------------+---------------+--------
           1 | $2a$12$e.lzmqKXyh8m8KTd9WBkxeqr90qeTrDKwBhlSsdjMf7yIlj1ygbqm | bcrypt    | \xc30c040901028b43e9beabaec7d8d24e01623a1d3e917176f836602c04ee285d87c01c64174cf0bb7c11ab314b2cc8a8507cf04b297eb410d83ba29e6b3adc208fde8b7adfbc96be5dbf53485069cb4625a7ddf9d423b4c151c46367ba58 |                3 |             1 | 1680771305487 |     12
         301 | $2a$12$abyhGRg2fsQ27NoHsR2xae8vuOFVbONpayxfctUAFpSvbM68kL1q2 | bcrypt    | \xc30c04090102c69b4884afb59f93d24e01a926a128b8a91eb20272877d093518b7ef759db0e85117467c93244d3a832a517d34e1b120164bd6717e2d5aa07ec5d95c2f5ef1c6eaed91126b649d4ee06dba3f7233f61254d1b67f7e56903c |                3 |             1 | 1681264745532 |     12
(2 rows)
``` 

发现password的数据内容被加密

## 0x05 小结
---

在我们搭建好ADAudit Plus漏洞调试环境后，接下来就可以着手对漏洞进行学习。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)






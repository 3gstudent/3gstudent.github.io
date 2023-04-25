---
layout: post
title: ADManager Plus漏洞调试环境搭建
---


## 0x00 前言
---

本文记录从零开始搭建ADManager Plus漏洞调试环境的细节，介绍数据库用户口令的获取方法。

## 0x01 简介
---

本文将要介绍以下内容：

- ADManager Plus安装
- ADManager Plus漏洞调试环境配置
- 数据库用户口令获取
- 数据库加密算法

## 0x02 ADManager Plus安装
---

### 1.下载

全版本下载地址：https://archives2.manageengine.com/ad-manager/

### 2.安装

安装参考：https://www.manageengine.com/products/ad-manager/help/getting_started/installing_admanager_plus.html

### 3.测试

访问https://localhost:8080

## 0x03 ADManager Plus漏洞调试环境配置
---

方法同ADAudit Plus漏洞调试环境配置基本类似

### 1.开启调试功能

#### (1)定位配置文件

查看java进程的父进程wrapper.exe的进程参数为：`"C:\Program Files\ManageEngine\ADManager Plus\bin\Wrapper.exe"  -c  "C:\Program Files\ManageEngine\ADManager Plus\bin\\..\conf\wrapper.conf"`


这里需要修改的配置文件为`C:\Program Files\ManageEngine\ADManager Plus\conf\wrapper.conf`

#### (2)修改配置文件添加调试参数

找到启用调试功能的位置：

```
#uncomment the following to enable JPDA debugging
#wrapper.java.additional.16=-Xdebug
#wrapper.java.additional.17=-Xnoagent
#wrapper.java.additional.18=-Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n

```

将其修改为

```
wrapper.java.additional.16=-Xdebug
wrapper.java.additional.17=-Xnoagent
wrapper.java.additional.18=-Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n
```

#### (3)重新启动相关进程

关闭进程wrapper.exe和对应的子进程java.exe

在开始菜单依次选择`Stop ADManager Plus`和`Start ADManager Plus`

### 2.常用jar包位置

路径：`C:\Program Files\ManageEngine\ADManager Plus\lib`

web功能的实现文件为`AdventNetADSMServer.jar`和`AdventNetADSMClient.jar`

### 3.IDEA设置

设置为`Remote JVM Debug`，远程调试

## 0x04 数据库用户口令获取
---

默认配置下，ADManager Plus使用postgresql存储数据，默认配置了两个登录用户：`admanager`和`postgres`

### 1.用户`admanager`的口令获取

配置文件路径：`C:\Program Files\ManageEngine\ADManager Plus\conf\database_params.conf`，内容示例：

```
# $Id$
# driver name
drivername=org.postgresql.Driver

# login username for database if any
username=admanager

# password for the db can be specified here
password=28e3e4d73561031fa3a0100ea4bfb3617c7d66b631ff54ca719dd4ca3dcfb3c308605888


# url is of the form jdbc:subprotocol:DataSourceName for eg.jdbc:odbc:WebNmsDB
url=jdbc:postgresql://localhost:33306/adsm?useUnicode=true&characterEncoding=UTF-8
#url=jdbc:mysql://localhost:33306/adsm?autoReconnect=true&characterEncoding=utf8
# Minumum Connection pool size
minsize=1

# Maximum Connection pool size
maxsize=20

# transaction Isolation level
#values are Constanst defined in java.sql.connection type supported TRANSACTION_NONE 	0
#Allowed values are TRANSACTION_READ_COMMITTED , TRANSACTION_READ_UNCOMMITTED ,TRANSACTION_REPEATABLE_READ , TRANSACTION_SERIALIZABLE
transaction_isolation=TRANSACTION_READ_COMMITTED


#dbadapter=com.adventnet.db.adapter.mysql.MysqlDBAdapter
#sqlgenerator=com.adventnet.db.adapter.mysql.MysqlSQLGenerator

dbadapter=com.adventnet.db.adapter.postgres.PostgresDBAdapter
sqlgenerator=com.adventnet.db.adapter.postgres.PostgresSQLGenerator
exceptionsorterclassname=com.adventnet.db.adapter.postgres.PostgresExceptionSorter
```

其中，password被加密，加解密算法位于：`C:\Program Files\ManageEngine\ADManager Plus\lib\framework-tools.jar`中的`com.zoho.framework.utils.crypto->CryptoUtil.class`

经过代码分析，得出以下解密方法：

密钥固定保存在`C:\Program Files\ManageEngine\ADManager Plus\conf\customer-config.xml`，内容示例：

```
<?xml version="1.0" encoding="UTF-8"?><extended-configurations>
    <configuration name="ECTag" value="9bfb32bf760984b1b2d0e232e8232f52b48e088c23bb6031866c151a35d94759fde6b4bdb3dfdbabc329b900fff159750b164018"/>
    <configuration name="CryptTag" value="o0hV5KhXBIKRH2PAmnCx"/>
    <configuration name="mssql" value="">
        <property name="certificate.subject" value="b5c0d93cb17e1d87bcd58ccb41a9b93df4bcd654c46df05d4385f3fb21f969a7cbd25a34"/>
        <property name="masterkey.password" value="8604001d26ee7080b8e52f751c589196925faa0b40ed16f38f00fc418cef8074e3d4d780"/>
        <property name="certificate.name" value="a8f8fbfdb609d8a942ce04ab29ef1635396387ff4294218c6d7e16db849a3c119ede620f"/>
        <property name="symmetrickey.name" value="8d9738e2c28b848d2017fdfaebaaed677cff132e78a0d709ff84681831ac49e7f427eb0c"/>
    </configuration>
</extended-configurations>
```


得到密钥：CryptTag为`o0hV5KhXBIKRH2PAmnCx`

根据以上得到的密文`28e3e4d73561031fa3a0100ea4bfb3617c7d66b631ff54ca719dd4ca3dcfb3c308605888`和密钥`o0hV5KhXBIKRH2PAmnCx`，编写解密程序，代码如下：

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
        String password = "28e3e4d73561031fa3a0100ea4bfb3617c7d66b631ff54ca719dd4ca3dcfb3c308605888";
        String encryptionKey = "o0hV5KhXBIKRH2PAmnCx";
        String out  = decryptAES256(password, encryptionKey);
        System.out.println(out);
    }
}
```

程序运行后得到解密结果：`DFVpXge0NS`

拼接出数据库的连接命令：`"C:\Program Files\ManageEngine\ADManager Plus\pgsql\bin\psql" "host=127.0.0.1 port=33306 dbname=adsm user=admanager password=DFVpXge0NS"`

### 2.用户`postgres`的口令

默认口令为`Stonebraker`

## 0x05 数据库加密算法
---

### 1.相关数据库信息

#### (1)用户相关的表

```
SELECT * FROM public.aaalogin；

 login_id | user_id |    name     |          domainname
----------+---------+-------------+-------------------------------
        1 |       1 | admin       | ADManager Plus Authentication
        2 |       2 | helpdesk    | ADManager Plus Authentication
        3 |       3 | hrassociate | ADManager Plus Authentication
(3 rows)
```

#### (2)口令相关的表

```
SELECT * FROM public.aaapassword;

 password_id |                           password                           | algorithm |             salt              | passwdprofile_id | passwdrule_id |  createdtime  | factor
-------------+--------------------------------------------------------------+-----------+-------------------------------+------------------+---------------+---------------+--------
           1 | $2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi | bcrypt    | $2a$12$sdX7S5c11.9vZqC0OOPZQ. |                2 |             3 | 1672405184666 |
           2 | $2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi | bcrypt    | $2a$12$sdX7S5c11.9vZqC0OOPZQ. |                2 |             3 | 1672405184666 |
           3 | $2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi | bcrypt    | $2a$12$sdX7S5c11.9vZqC0OOPZQ. |                2 |             3 | 1672405184666 |
(3 rows)
```

#### (3)权限相关的表

```
SELECT * FROM public.aaarole;

 role_id |       name       | service_id |                             descripti
on
---------+------------------+------------+--------------------------------------
-------------------------------
       1 | ReadAllTables    |          1 | Kind of users who have permission to
read all tables
       2 | AccessAllTables  |          1 | Kind of users who have permission to
access  entries in all tables
       3 | AccessAllMethods |          1 | Kind of users who have permission to
invoke all methods on all EJBs
       4 | Administrator    |          1 | No Description
       5 | DomainUser       |          1 | No Description
       6 | Requester        |          1 | Workflow Requester Configuration
       7 | SdpUser          |          1 | ServiceDesk Plus
(7 rows)


SELECT * FROM public.admpusersrolemapping;

 admp_user_domain_role_id | login_id | admp_role_id | admp_domain_name | user_credential | append_groups | type_of_data | role_ou_mapping_id
--------------------------+----------+--------------+------------------+-----------------+---------------+--------------+--------------------
                        1 |        1 |            1 | All Domains      | true         | false         |            1 |                  1
                        4 |        1 |            1 | All Domains      | true         | false         |            3 |                  1
                      305 |        2 |            3 | test.com         | true         | false         |            1 |                  1
                      313 |        3 |            2 | test.com         | true         | false         |            1 |                  1
                      314 |        3 |            2 | test.com         | true         | false         |            3 |                  1
                      315 |        2 |            3 | test.com         | true         | false         |            3 |                  1
(6 rows)
```

### 2.口令加密算法

算法同ADAudit Plus一致，计算密文的测试代码如下：

```
import org.mindrot.jbcrypt.BCrypt;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import static com.adventnet.authentication.util.AuthUtil.convertToByteArray;
import static com.adventnet.authentication.util.AuthUtil.convertToString;
public class test1 {
    
    public static String getEncryptedPassword(String password, String salt, String algorithm) {
        if (algorithm.equals("bcrypt")) {
            String hashedPassword = null;
            if (salt.matches("\\$2a\\$.*\\$.*")) {
                hashedPassword = BCrypt.hashpw(password, salt);
                return hashedPassword;
            } else {
                throw new IllegalArgumentException("Invalid Salt value.. To use Bcrypt hashing, salt should be generated through 'Bcrypt.genSalt(workload)' or in the form '$2a$(workload)$(22char)' ");
            }
        } else {
            byte[] password_ba = convertToByteArray(password);
            byte[] salt_ba = convertToByteArray(salt);

            try {
                MessageDigest md = MessageDigest.getInstance(algorithm);
                md.update(password_ba);
                md.update(salt_ba);
                byte[] cipher = md.digest();
                return convertToString(cipher);
            } catch (NoSuchAlgorithmException var7) {
                System.out.println("Exception occured when getting MessageDigest Instance for Algorithm "+ algorithm + ". Returning unencrypted Password");
                return password;
            }
        }
    }
    public static void main(String[] args) throws Exception, Exception {
        String password = "admin";
        String salt = "$2a$12$sdX7S5c11.9vZqC0OOPZQ.";
        String encPass = getEncryptedPassword(password, salt, "bcrypt");
        System.out.println(encPass);
    }
}
```

计算结果为`$2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi`，同数据库得到的password项一致


### 3.语法示例

#### (1)查询用户及对应的权限

```
SELECT aaalogin.login_id,aaalogin.user_id,aaalogin.name,aaalogin.domainname,admpusersrolemapping.admp_role_id,admpusersrolemapping.admp_domain_name FROM public.aaalogin as aaalogin INNER JOIN public.admpusersrolemapping AS admpusersrolemapping on aaalogin.login_id=admpusersrolemapping.login_id;

 login_id | user_id |    name     |          domainname           | admp_role_id | admp_domain_name
----------+---------+-------------+-------------------------------+--------------+------------------
        1 |       1 | admin       | ADManager Plus Authentication |            1 | All Domains
        1 |       1 | admin       | ADManager Plus Authentication |            1 | All Domains
        3 |       3 | hrassociate | ADManager Plus Authentication |            2 | test.com
        3 |       3 | hrassociate | ADManager Plus Authentication |            2 | test.com
        2 |       2 | helpdesk    | ADManager Plus Authentication |            1 | test.com
        2 |       2 | helpdesk    | ADManager Plus Authentication |            1 | test.com
(6 rows)
```

#### (2)查询用户及对应的口令

```
SELECT aaalogin.login_id,aaalogin.name,aaapassword.password_id,aaapassword.password,aaapassword.salt FROM public.aaalogin as aaalogin INNER JOIN public.aaapassword AS aaapassword on aaalogin.login_id=aaapassword.password_id;

 login_id|    name     | password_id |                           password                    |             salt
---------+-------------+-------------+--------------------------------------------------------------+-------------------------------
       1 | admin       |           1 | $2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi | $2a$12$sdX7S5c11.9vZqC0OOPZQ.
       2 | helpdesk    |           2 | $2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi | $2a$12$sdX7S5c11.9vZqC0OOPZQ.
       3 | hrassociate |           3 | $2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi | $2a$12$sdX7S5c11.9vZqC0OOPZQ.
(3 rows)
```

#### (3)修改口令

```
UPDATE aaapassword SET password='$2a$12$sdX7S5c11.9vZqC0OOPZQ.9PLFBKubufTqUNyLbom2Ub13d573jhi' WHERE password_id=1;
```

## 0x06 小结
---

在我们搭建好ADManager Plus漏洞调试环境后，接下来就可以着手对漏洞进行学习。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




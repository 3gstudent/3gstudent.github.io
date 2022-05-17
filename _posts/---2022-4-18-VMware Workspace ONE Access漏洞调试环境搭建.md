---
layout: post
title: VMware Workspace ONE Access漏洞调试环境搭建
---


## 0x00 前言
---

本文记录从零开始搭建VMware Workspace ONE Access漏洞调试环境的细节。

## 0x01 简介
---

本文将要介绍以下内容：

- VMware Workspace ONE Access安装
- VMware Workspace ONE Access漏洞调试环境配置
- 常用知识

## 0x02 VMware Workspace ONE Access安装
---

参考资料：

https://docs.vmware.com/en/VMware-Workspace-ONE-Access/20.01/workspace_one_access_install.pdf

### 1.下载OVA文件

下载页面：

https://customerconnect.vmware.com/downloads/search?query=workspace%20one%20access

下载前需要先注册用户，之后选择需要的版本进行下载

VMware Workspace ONE Access 21.08.0.1的下载页面：https://customerconnect.vmware.com/downloads/details?downloadGroup=WS1A_ONPREM_210801&productId=1269

下载文件identity-manager-21.08.0.1-19010796_OVF10.ova

### 2.安装

#### (1)在VMware Workstation中导入OVA文件

**注：**

VMware Workstation版本需要大于14，否则报错提示无法导入

在安装页面设置`Host Name`，如果配置了DHCP，其他选项不用设置，我的配置指定了静态IP，配置如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-18/2-1.png)

等待OVA文件导入完成后，将会自动开机进行初始化，初始化完成后如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-18/2-2.png)

#### (2)配置

修改本机的hosts文件，将192.168.1.11指向workspaceone.test.com

访问配置页面https://workspaceone.test.com:8443

设置admin、root和sshuser用户的口令，口令需要包含大写字母、小写字母、数字和特殊字符

**注：**

我的测试结果显示，口令长度需要设置为14，否则无法登陆root和sshuser用户

我的测试环境设置口令为`Password@12345`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-18/2-3.png)

设置数据库，为了便于环境搭建，这里选择`Internal Database`

等待安装完成，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-18/2-4.png)

### 3.设置允许root用户远程登录ssh

需要登录VMware Workspace ONE Access，修改系统的配置文件，有以下两种登录方法：

#### (1)在虚拟机中直接登录root用户

选择`Login`，输入root和口令`Password@12345`

#### (2)通过ssh登录sshuser用户

登录后再切换至root

切换至root用户后，依次执行以下命令：

- vi /etc/ssh/sshd_config
- 设置PermitRootLogin从`no`变为`yes`
- systemctl restart sshd

### 4.开启远程调试功能

修改文件：`/opt/vmware/horizon/workspace/bin/setenv.sh`

修改参数`JVM_OPTS`，添加参数：`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8000`

如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-18/2-5.png)

重新启动系统

打开防火墙：`iptables -P INPUT ACCEPT && iptables -P OUTPUT ACCEPT`

IDEA设置远程调试参数，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-18/2-6.png)

**注：**

IDEA的完整配置方法可参考之前的文章[《Zimbra漏洞调试环境搭建》](https://3gstudent.github.io/Zimbra%E6%BC%8F%E6%B4%9E%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA)

## 0x03 常用知识
---

### 1.常用命令

查看系统服务状态： `chkconfig --list` 

查看所有服务状态： `systemctl status`

查看IP地址： `ip addr show`

查看Host Name: `hostname`

日志路径： `/opt/vmware/horizon/workspace/logs/`

### 2.查看系统版本

需要root权限执行命令： `vamicli version --appliance`

查看系统版本的实现细节：

```
#!/usr/bin/env python2
import sys
sys.path.append("/opt/vmware/lib/python/site-packages/")
import pywbem
def getCIMConnection (url, namespace):
    cred = {}
    cred ['cert_file'] = '/opt/vmware/etc/sfcb/client.pem'
    cred ['key_file'] = '/opt/vmware/etc/sfcb/file.pem'
    cliconn = pywbem.WBEMConnection (url, None, namespace, cred)
    return cliconn

def showVersion():
  try:
    cliconn = getCIMConnection ('https://localhost:5489', 'root/cimv2')
    esis = cliconn.EnumerateInstances ('VAMI_ElementSoftwareIdentity')
  except:
    print('error')
    return
  for esi in esis:
    ess = esi ['ElementSoftwareStatus']
    if (ess == [2, 6]):
      inst = cliconn.GetInstance (esi['Antecedent'])
      print ('Version - ' + inst ['VersionString'])
      print ('Description - ' + inst ['Description'])
showVersion()
```

需要root权限是因为访问文件`/opt/vmware/etc/sfcb/client.pem`和`/opt/vmware/etc/sfcb/file.pem`需要root权限

### 3.数据库连接口令

连接数据库的明文口令位置为:`/usr/local/horizon/conf/db.pwd`

连接数据库的口令加密保存在文件`/usr/local/horizon/conf/runtime-config.properties`中，文件内容示例：

```
datastore.jdbc.url=jdbc:postgresql://localhost/saas?stringtype=unspecified
datastore.jdbc.userName=horizon
secure.datastore.jdbc.password=BAACs8MW1xyMe7/8ONd2QwtG3mw37wF1/1pQ6D09xXqf56ncfRtCun6y8A1XFtjajhU60V1QNYnCOxk3t1m0dV0JvA==
```

其中，`BAACs8MW1xyMe7/8ONd2QwtG3mw37wF1/1pQ6D09xXqf56ncfRtCun6y8A1XFtjajhU60V1QNYnCOxk3t1m0dV0JvA==`为加密口令

需要以下文件作为解密密钥：

- /usr/local/horizon/conf/configkeystore.pass
- /usr/local/horizon/conf/configkeystore.bcfks



### 4.数据库中的加密信息

admin用户的口令加密存储在数据库中

查询命令：`saas=> SELECT "passwordAuthData" FROM "PasswordInformation";`

查询结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-4-18/2-7.png)

加密的主要实现代码1：

```
   private String AES_encrypt(@Nonnull byte[] clearData, @Nonnull byte[] key, @Nonnull EncryptionAlgorithms encAlg) throws EncryptionServiceException {
        Preconditions.checkNotNull(clearData);
        Preconditions.checkNotNull(key);
        Preconditions.checkNotNull(encAlg);
        Preconditions.checkArgument(clearData.length != 0);

        try {
            String cipherName = encAlg.getCipherName();
            Cipher cipher = Cipher.getInstance(cipherName, provider);
            int nonceSize = encAlg.getNonceSize(cipher.getBlockSize());
            IvParameterSpec ivSpec = null;
            String encodedIv;
            if (nonceSize > 0) {
                byte[] iv = new byte[nonceSize];
                srand.nextBytes(iv);
                ivSpec = new IvParameterSpec(iv);
                encodedIv = new String(Hex.encode(iv), StandardCharsets.US_ASCII);
            } else {
                encodedIv = "";
            }

            if (encAlg.forcePadding()) {
                clearData = ArrayUtils.add(clearData, (byte)1);
            }

            SecretKey secret = new SecretKeySpec(key, cipherName);
            cipher.init(1, secret, ivSpec, srand);
            byte[] data = cipher.doFinal(clearData);
            String output = Integer.toString(4) + ":" + encodedIv + ":" + new String(Hex.encode(data), StandardCharsets.US_ASCII);
            return output;
        } catch (InvalidKeyException | BadPaddingException | IllegalBlockSizeException | InvalidAlgorithmParameterException | NoSuchPaddingException | NoSuchAlgorithmException | FipsUnapprovedOperationError var12) {
            log.error("Failed to encrypt with AES: " + var12.getMessage());
            throw new EncryptionServiceException(var12);
        }
    }
```

加密的主要实现代码2：

```
String encryptedData = Integer.toString(1) + "," + encKey.getSafeUuid().toString() + "," + this.AES_encrypt(clearData, aesKey, encAlg);
```

### 5.8443端口登录口令

登录口令加密保存在文件`/usr/local/horizon/conf/config-admin.json`

加密的主要实现代码：

```
    private void setPassword(String newPassword, boolean isSet) throws AdminAuthException {
        int ic = this.passwordAuthenticationUtil.getIc(iterationCountBase, iterationCountRange);

        try {
            String newEncryptedPassword = this.passwordAuthenticationUtil.createPWInfo("admin", "admin", ic, newPassword);
            PasswordInfo newPasswordInfo = new PasswordInfo(newEncryptedPassword, isSet);
            if (this.passwordInfo != null) {
                newPasswordInfo.setAttemptDelay(this.passwordInfo.getAttemptDelay());
                newPasswordInfo.setMaxAttemptCount(this.passwordInfo.getMaxAttemptCount());
            }

            objectMapper.writeValue(this.passwordInfoFile, newPasswordInfo);
        } catch (IOException | EncryptionServiceException var7) {
            throw new AdminAuthException("Failed to set password" + var7.getMessage(), var7);
        }

        try {
            this.loadEncryptedPasswordFromFile();
        } catch (IOException var6) {
            throw new AdminAuthException("Failed to load stored password" + var6.getMessage(), var6);
        }
    }
```

## 0x04 小结
---

在我们搭建好VMware Workspace ONE Access漏洞调试环境后，接下来就可以着手对漏洞和数据库口令的解密方法进行学习。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)




---
layout: post
title: Server Backup Manager漏洞调试环境搭建
---


## 0x00 前言
---

Server Backup Manager(SBM)是一种快速、经济且高性能的备份软件，适用于物理和虚拟环境中的Linux和Windows服务器。本文将要介绍Server Backup Manager漏洞调试环境的搭建方法。

## 0x01 简介
---

本文将要介绍以下内容：

- 环境搭建
- 调试环境搭建
- 用户数据库文件提取
- CVE-2022-36537简要介绍

## 0x02 环境搭建
---

安装参考资料：http://wiki.r1soft.com/display/ServerBackupManager/Install+and+Upgrade+Server+Backup+Manager+on+Debian+and+Ubuntu.html

参考资料提供了两种安装方法，但是我在测试过程中均遇到了缺少文件`/etc/init.d/cdp-server`的错误

这里改用安装旧版本的Server Backup Manager，成功完成安装，具体方法如下：

### 1.下载安装包

http://r1soft.mirror.iweb.ca/repo.r1soft.com/release/6.2.2/78/trials/R1soft-ServerBackup-Manager-SE-linux64-6-2-2.zip

### 2.安装

```
unzip R1soft-ServerBackup-Manager-SE-linux64-6-2-2.zip
dpkg -i *.deb
```

### 3.配置

```
serverbackup-setup --user admin --pass 123456
serverbackup-setup --http-port 8080 --https-port 8443
```

### 4.启动服务

```
/etc/init.d/cdp-server restart
```

web管理页面有以下两个:

http://127.0.0.1:8080

https://127.0.0.1:8443

## 0x03 调试环境搭建
---

研究过程如下：

#### (1)

查看文件`/etc/init.d/cdp-server`的格式为文本文件，可在文件内容中得到默认安装路径为`/usr/sbin/r1soft`，主程序为`/usr/sbin/r1soft/bin/cdpserver`

web路径为：`/usr/sbin/r1soft/webapps`，默认有以下两个文件：

- r1soft-api.war
- zk-web.war

#### (2)

查看进程信息：`ps aux |grep cdp`

返回内容：

```
root        2250  0.3  0.1  20836  2488 ?        Sl   22:18   0:01 ./cdpserver /usr/sbin/r1soft/conf/server.conf
root        2252 22.6 51.2 8747176 1036408 ?     Sl   22:18   1:55 ./cdpserver /usr/sbin/r1soft/conf/server.conf
```

得到配置文件的路径：`/usr/sbin/r1soft/conf/server.conf`

#### (3)

通过修改文件`/usr/sbin/r1soft/conf/server.conf`添加Java调试参数，添加内容如下：

```
additional.19=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8000
```

**注：**

这里要使用`JDK5-8`对应的参数`address=8000`，不能使用`JDK9及更高版本`对应的参数`address=*:8000`

#### (4)

重启服务：`/etc/init.d/cdp-server restart`

#### (5)

从服务器下载源代码

源代码位置1：`/usr/sbin/r1soft/lib`中的jar文件

源代码位置2：提取`/usr/sbin/r1soft/webapps/zk-web.war`的文件夹`\WEB-INF\classes\`中的class文件

#### (6)

使用IDEA下断点并配置远程调试，远程调试成功如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-12-12/2-1.png)

## 0x04 用户数据库文件提取
---

研究过程如下：

#### (1)定位用户创建的操作

位置为`/usr/sbin/r1soft/webapps/zk-web.war`中的文件`\WEB-INF\classes\com\r1soft\backup\server\web\user\Controller.class`，核心代码如下：

```
   public void onDoCreate(Event evt) {
        Clients.clearBusy();
        if (this.createWindow == null) {
            throw new IllegalStateException("CreateWindow object is not available");
        } else {
            User currentUser = SessionUtil.getCurrentUser();
            User user = new User();
            user.setUsername(this.createWindow.getUsername());
            user.setPassword((new PasswordEncoder()).encodePassword(this.createWindow.getPassword(), (Object)null));
            user.setName(this.createWindow.getName());
            user.setEmailAddress(this.createWindow.getEmailAddress());
            user.setUserType(this.createWindow.getUserType());
            List<Volume> volumes = new ArrayList();
            Map<String, String> attributes = new HashMap();
            attributes.putAll(this.createWindow.getAttributesMap());
            if (!user.isSuperUser()) {
                if (this.createWindow.getUserType().equals(UserType.SUB_USER)) {
                    user.getAdministrators().addAll(this.createWindow.getAdministrators());
                    if (currentUser.isPowerUser() && !user.getAdministrators().contains(currentUser)) {
                        user.getAdministrators().add(currentUser);
                    }
                } else if (this.createWindow.getUserType().equals(UserType.POWER_USER)) {
                    user.getSubUsers().addAll(this.createWindow.getSubUsers());
                }

                user.getGroups().addAll(this.createWindow.getGroups());
                Set<UserAgentPermission> permissions = new HashSet(this.createWindow.getUserAgentPermissions());
                Iterator i$ = permissions.iterator();

                while(i$.hasNext()) {
                    UserAgentPermission permission = (UserAgentPermission)i$.next();
                    permission.setUser(user);
                }

                user.getUserAgentPermissions().addAll(permissions);
                volumes.addAll(this.createWindow.getVolumes());
                if (user.isPowerUser()) {
                    attributes.putAll(this.createWindow.getPowerUserAttributesMap());
                }
            }

            if (this.createWindow.getSelectedLocale() != null) {
                attributes.put(UserAttributes.SELECTED_LOCALE.getDataKey(), this.createWindow.getSelectedLocale().toString());
            }

            try {
                UserFacade.make().createUser(SessionUtil.buildActivitySource(), user, volumes, attributes);
                WebUtil.showSuccessBox(logger, "Messages.UI.successfully-created-user", new Object[]{user.getUsername()});
            } catch (UserException var9) {
                this.showErrorBox(var9, "Messages.UI.could-not-create-user", new Object[]{user.getUsername()});
            }

            this.createWindow.detach();
            this.createWindow = null;
        }
    }
```

跟进`(new PasswordEncoder()).encodePassword(this.createWindow.getPassword(), (Object)null)`，具体位置为`/usr/sbin/r1soft/lib/cdpserver.jar`->`com.r1soft.backup.server.facade`->`PasswordEncoder.class`，核心代码如下：

```
   public String encodePassword(String var1, Object var2) throws DataAccessException {
        MessageDigest var3 = null;

        try {
            var3 = MessageDigest.getInstance("SHA-1");
        } catch (NoSuchAlgorithmException var7) {
            throw new RuntimeException(var7);
        }

        try {
            var3.update(var1.getBytes("UTF-8"));
        } catch (UnsupportedEncodingException var6) {
            throw new RuntimeException(var6);
        }

        byte[] var4 = var3.digest();
        String var5 = (new BASE64Encoder()).encode(var4);
        return var5;
    }
```

从以上代码可以得出用户口令的加密算法

#### (2)定位用户创建的具体代码实现位置

跟进`UserFacade.make().createUser(SessionUtil.buildActivitySource(), user, volumes, attributes);`，具体位置为`/usr/sbin/r1soft/lib/cdpserver.jar`->`com.r1soft.backup.server.facade`->`UserFacade.class`，核心代码如下：

```
   public User createUser(ActivitySource var1, User var2, List<Volume> var3, Map<String, String> var4) throws UserException {
//Hide code
            this.validateUser(var2);

            try {
                if (var4 != null && !var4.isEmpty()) {
                    HashSet var5 = new HashSet();
                    Iterator var6 = Arrays.asList(UserFacade.PowerUserAttributes.values()).iterator();

                    while(var6.hasNext()) {
                        PowerUserAttributes var7 = (PowerUserAttributes)var6.next();
                        String var8 = var4.containsKey(var7.getDataKey()) ? (String)var4.get(var7.getDataKey()) : var7.getDefaultValue();
                        var5.add(new UserData(var2, var7.getDataKey(), var8));
                        var4.remove(var7.getDataKey());
                    }

                    var6 = var4.entrySet().iterator();

                    while(var6.hasNext()) {
                        Map.Entry var13 = (Map.Entry)var6.next();
                        var5.add(new UserData(var2, (String)var13.getKey(), (Serializable)var13.getValue()));
                    }

                    var2 = com.r1soft.backup.server.om.facade.UserFacade.getInstance().persistPOJOWithData(var2, var5);
                } else {
                    var2 = (User)EntityManagerFacade.persistPOJO(var2);
                }
//Hide code
    }
```

跟进`(User)EntityManagerFacade.persistPOJO(var2)`，具体位置为`/usr/sbin/r1soft/lib/cdpserver.jar`->`com.r1soft.backup.server.om.entity`->`EntityManagerFacade.class`，核心代码如下：

```
private static EntityManagerFactory emf = null;
private static Map<Class<? extends BaseEntity<?>>, Class<? extends BaseOMFacade<? extends BaseEntity<?>, ?, ? extends IPOJO<? extends BaseEntity<?>, ?, ?>, ?>>> facadeMap = new HashMap();
private static Map<Class<? extends IPOJO<? extends BaseEntity<?>, ?, ?>>, Class<? extends BaseEntity<?>>> pojoToEntityMap = new HashMap();
private static Map<Class<? extends IPOJO<? extends BaseEntity<?>, ?, ?>>, Class<? extends BaseOMFacade<? extends BaseEntity<?>, ?, ? extends IPOJO<? extends BaseEntity<?>, ?, ?>, ?>>> pojoToFacadeMap = new HashMap();
private static Map<Class<? extends Object>, Class<? extends BaseOMFacade<? extends BaseEntity<?>, ?, ? extends IPOJO<? extends BaseEntity<?>, ?, ?>, ?>>> pojoIDToFacadeMap = new HashMap();
protected static final QueryOrderByType DEFAULT_SORT_DIRECTION;
protected static final String PERSISTENCEUNITNAME = "CDP-PU";
protected static final String HIBERNATECONFFILE = "com/r1soft/backup/server/om/hibernate.cfg.xml";
protected static final String DBUSER = "r1derbyuser";
protected static final String DBPASS = "V?Rdp*eT6N9t8aW3KDoh";
protected static final String H2PATH = "h2/r1backup";
```

从以上的代码可以看到这里使用了JDBC H2数据库存储数据，配置文件为`"com/r1soft/backup/server/om/hibernate.cfg.xml"`，实际对应的文件位置为`/usr/sbin/r1soft/lib/cdpserver.jar`->`com.r1soft.backup.server.om.hibernate.cfg.xml`，其核心内容如下：

```
   <property name="hibernate.connection.url">jdbc:h2:./data/h2/r1backup</property>
   
   <property name="hibernate.connection.driver_class">org.h2.Driver</property>
```

综合以上内容，我们可以获得完整的数据库连接参数，并且实际的数据库文件位置为`/usr/sbin/r1soft/data/h2/r1backup.h2.db`

这里进行数据验证测试：手动创建用户`admin2`，口令设置为`123456`，通过动态调试得出加密的口令内容为`fEqNCco3Yq9h5ZUglD3CZJT4lBs=`，接着以二进制打开文件`/usr/sbin/r1soft/data/h2/r1backup.h2.db`，在文件内容中可以获得用户名和加密的口令内容，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-12-12/2-2.png)

## 0x05 CVE-2022-36537简要介绍
---

漏洞分析文章：https://medium.com/numen-cyber-labs/cve-2022-36537-vulnerability-technical-analysis-with-exp-667401766746

文章中提到触发RCE需要上传一个带有Payload的com.mysql.jdbc.Driver文件

这个操作只能利用一次，原因如下：

默认情况下，管理后台的的Database Driver页面存在可以上传的图标，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-12-12/3-1.png)

上传后不再显示可上传的图标，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-12-12/3-2.png)

触发RCE就是通过程序实现上传MySQL DataBase Driver，所以攻击一次后该功能失效，不可重复利用

攻击检测：触发RCE后会在系统中上传文件`/usr/sbin/r1soft/conf/database-drivers/mysql-connector.jar`，其中的`com\mysql\jdbc\Driver.class`为攻击者使用的Payload

## 0x06 小结
---

本文介绍了在搭建Server Backup Manager调试环境过程中一些问题的解决方法，分析用户数据库文件提取的方法，给出检测CVE-2022-36537的建议。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)



---
layout: post
title: ADAudit Plus利用分析——数据加密分析
---


## 0x00 前言
---

在上篇文章《ADAudit Plus漏洞调试环境搭建》介绍了漏洞调试环境的搭建细节，经测试发现数据库的部分数据做了加密，本文将要介绍数据加密的相关算法。

## 0x01 简介
---

本文将要介绍以下内容：

- 数据加密的位置
- 算法分析
- 算法实现

## 0x02 数据加密的位置
---

测试环境同《ADAudit Plus漏洞调试环境搭建》保持一致

数据库连接的完整命令：`"C:\Program Files\ManageEngine\ADAudit Plus\pgsql\bin\psql" "host=127.0.0.1 port=33307 dbname=adap user=postgres password=Stonebraker"`

查询加密口令的命令示例：`SELECT * FROM public.aaapassword ORDER BY password_id ASC;`

返回结果示例：

```
 password_id |                           password                           | algorithm |                       salt                                         | passwdprofile_id | passwdrule_id |  createdtime  | factor
-------------+--------------------------------------------------------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------------+---------------+---------------+--------
           1 | $2a$12$e.lzmqKXyh8m8KTd9WBkxeqr90qeTrDKwBhlSsdjMf7yIlj1ygbqm | bcrypt    | \xc30c040901028b43e9beabaec7d8d24e01623a1d3e917176f836602c04ee285d87c01c64174cf0bb7c11ab314b2cc8a8507cf04b297eb410d83ba29e6b3adc208fde8b7adfbc96be5dbf53485069cb4625a7ddf9d423b4c151c46367ba58 |                3 |             1 | 1680771305487 |     12
         301 | $2a$12$abyhGRg2fsQ27NoHsR2xae8vuOFVbONpayxfctUAFpSvbM68kL1q2 | bcrypt    | \xc30c04090102c69b4884afb59f93d24e01a926a128b8a91eb20272877d093518b7ef759db0e85117467c93244d3a832a517d34e1b120164bd6717e2d5aa07ec5d95c2f5ef1c6eaed91126b649d4ee06dba3f7233f61254d1b67f7e56903c |                3 |             1 | 1681264745532 |     12
(2 rows)
```

经测试，对应Web管理页面的位置为`Admin`->`Technicians`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-7/2-1.png)

点击`Add technicians`可以添加用户，这里可以选择添加自定义用户或者域用户

添加自定义用户需要输入口令，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-7/2-2.png)

添加域用户不需要输入域用户的口令，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-7/2-3.png)

## 0x03 算法分析
---

### 1.加密算法细节

经分析，加密算法细节位于`C:\Program Files\ManageEngine\ADAudit Plus\lib\AdvAuthentication.jar`中的`com.adventnet.authentication.util`->`AuthUtil.class`

添加用户的实现代码：

```
   public static DataObject createUserAccount(DataObject accountDO) throws DataAccessException, PasswordException {
        int workload = false;
        LOGGER.log(Level.FINEST, "createUserAccount invoked with dataobject : {0}", accountDO);
        Iterator accItr = accountDO.getRows("AaaAccount");
        int numOfAcc = getCount(accItr);
        Credential credential = getUserCredential();
        if (credential != null) {
            validateForAccountCreation(credential.getAccountId(), numOfAcc);
        }

        List requiredTables = Arrays.asList("AaaUser", "AaaLogin", "AaaAccount", "AaaPassword", "AaaAccPassword");
        List tablesFromDO = accountDO.getTableNames();
        if (!tablesFromDO.containsAll(requiredTables)) {
            throw new DataAccessException("In sufficient data for creating an account, required tables in dataobject " + requiredTables);
        } else {
            long now = System.currentTimeMillis();
            Row userRow = accountDO.getFirstRow("AaaUser");
            userRow.set("CREATEDTIME", new Long(now));
            Row userStatusRow = new Row("AaaUserStatus");
            userStatusRow.set("USER_ID", userRow.get("USER_ID"));
            userStatusRow.set("STATUS", "ACTIVE");
            userStatusRow.set("UPDATEDTIME", new Long(now));
            accountDO.addRow(userStatusRow);
            Row loginRow = accountDO.getFirstRow("AaaLogin");

            try {
                String domain = (String)loginRow.get("DOMAINNAME");
                loginRow.set("DOMAINNAME", domain != null && domain.trim().length() != 0 ? domain : (String)MetaDataUtil.getTableDefinitionByName("AaaLogin").getColumnDefinitionByName("DOMAINNAME").getDefaultValue());
            } catch (MetaDataException var28) {
                throw new DataAccessException("Exception occured while obtaining default value of [AAALOGIN.DOMAINNAME]" + var28);
            }

            Criteria c = new Criteria(Column.getColumn("AaaLogin", "NAME"), loginRow.get("NAME"), 0, false);
            c = c.and(new Criteria(Column.getColumn("AaaLogin", "DOMAINNAME"), loginRow.get("DOMAINNAME"), 0));
            DataObject dobj = DataAccess.get("AaaLogin", c);
            if (!dobj.isEmpty()) {
                LOGGER.log(Level.SEVERE, "Could not create new User :: {0}, as the user with given LoginName and DomainName already exists", new Object[]{loginRow.get("NAME")});
                throw new DataAccessException("Could not create new User ::" + loginRow.get("NAME") + ", as the given LoginName and DomainName :: " + loginRow.get("DOMAINNAME") + "already exists");
            } else {
                if (loginRow.get("USER_ID") == null) {
                    loginRow.set("USER_ID", userRow.get("USER_ID"));
                    accountDO.updateRow(loginRow);
                }

                accItr = accountDO.getRows("AaaAccount");
                Row accountRow = null;
                List serviceIds = new ArrayList();

                Row passwordRow;
                while(accItr.hasNext()) {
                    accountRow = (Row)accItr.next();
                    serviceIds.add(accountRow.get("SERVICE_ID"));
                    if (accountRow.get("LOGIN_ID") == null) {
                        accountRow.set("LOGIN_ID", loginRow.get("LOGIN_ID"));
                    }

                    accountRow.set("CREATEDTIME", new Long(now));
                    accountDO.updateRow(accountRow);
                    Row accOwnerProfileRow = null;

                    try {
                        accOwnerProfileRow = accountDO.getFirstRow("AaaAccOwnerProfile", accountRow);
                    } catch (DataAccessException var27) {
                    }

                    if (accOwnerProfileRow == null) {
                        accOwnerProfileRow = new Row("AaaAccOwnerProfile");
                        accOwnerProfileRow.set("ACCOUNT_ID", accountRow.get("ACCOUNT_ID"));
                        accOwnerProfileRow.set("ALLOWED_SUBACCOUNT", new Integer(0));
                        accountDO.addRow(accOwnerProfileRow);
                    }

                    if (!accountDO.containsTable("AaaAccountOwner") && credential != null) {
                        passwordRow = new Row("AaaAccountOwner");
                        passwordRow.set("ACCOUNT_ID", accountRow.get("ACCOUNT_ID"));
                        passwordRow.set("OWNERACCOUNT_ID", credential.getAccountId());
                        accountDO.addRow(passwordRow);
                    }

                    String accAdminProfile = (String)AuthDBUtil.getObject("AaaAccAdminProfile", "NAME", "ACCOUNTPROFILE_ID", accountRow.get("ACCOUNTPROFILE_ID"));
                    Row accStatusRow = constructAccStatusRow(accountRow, accAdminProfile);
                    accountDO.addRow(accStatusRow);
                }

                Iterator passItr = accountDO.getRows("AaaPassword");
                passwordRow = null;

                while(passItr.hasNext()) {
                    passwordRow = (Row)passItr.next();
                    Long passRuleId = (Long)passwordRow.get("PASSWDRULE_ID");
                    Long passProfileId = (Long)passwordRow.get("PASSWDPROFILE_ID");
                    if (passRuleId == null) {
                        String[] serviceNames = getServiceNames(serviceIds);
                        passRuleId = getCompatiblePassRuleId(serviceNames);
                        passwordRow.set("PASSWDRULE_ID", passRuleId);
                    }

                    Row passRuleRow = AuthDBUtil.getRowMatching("AaaPasswordRule", "PASSWDRULE_ID", passRuleId);
                    validateForPasswordRule((String)loginRow.get("NAME"), (String)passwordRow.get("PASSWORD"), passRuleRow);
                    String passwordProfile = (String)AuthDBUtil.getObject("AaaPasswordProfile", "NAME", "PASSWDPROFILE_ID", passProfileId);
                    Row passProfileRow = AuthDBUtil.getRowMatching("AaaPasswordProfile", "NAME", passwordProfile);
                    Object workFactor = passProfileRow.get("FACTOR");
                    if (!((String)passwordRow.get("ALGORITHM")).equalsIgnoreCase("bcrypt")) {
                        LOGGER.log(Level.INFO, "algorithm used to hash password should be bcrypt, hence updating algorithm as bcrypt");
                        passwordRow.set("ALGORITHM", "bcrypt");
                    }

                    int workload;
                    if (workFactor != null && Integer.parseInt(workFactor.toString()) > 0) {
                        workload = Integer.parseInt(workFactor.toString());
                    } else {
                        workload = PAM.workload;
                    }

                    String salt = BCrypt.gensalt(workload);
                    String encPass = getEncryptedPassword((String)passwordRow.get("PASSWORD"), salt, (String)passwordRow.get("ALGORITHM"));
                    passwordRow.set("PASSWORD", encPass);
                    passwordRow.set("FACTOR", workload);
                    passwordRow.set("ALGORITHM", passwordRow.get("ALGORITHM"));
                    passwordRow.set("CREATEDTIME", new Long(now));
                    passwordRow.set("SALT", salt);
                    accountDO.updateRow(passwordRow);
                    Row passStatusRow = constructPassStatusRow(passwordRow, passwordProfile);
                    accountDO.addRow(passStatusRow);
                }

                LOGGER.log(Level.FINEST, "account validated dataobject is : {0}", accountDO);
                DataObject addedDO = AuthDBUtil.getPersistence("Persistence").add(accountDO);
                LOGGER.log(Level.FINEST, "account added successfully");
                return addedDO;
            }
        }
    }
```

得到加密生成Password的代码：

```
String encPass = getEncryptedPassword((String)passwordRow.get("PASSWORD"), salt, (String)passwordRow.get("ALGORITHM"));
```

生成salt的代码：

```
String salt = BCrypt.gensalt(workload);
```

经动态调试，发现`workload`默认为`12`，生成的salt格式示例：`$2a$12$DVT1iwOoi3YwkHO6L6QSoe`，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2023-4-7/3-1.png)

具体加密算法`getEncryptedPassword()`的实现细节：

```
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
                LOGGER.log(Level.SEVERE, "Exception occured when getting MessageDigest Instance for Algorithm {0}. Returning unencrypted Password", algorithm);
                return password;
            }
        }
    }
```

在此处下断点，经动态调试得出以下结论：

- 如果是域用户，会使用默认口令`admin`作为明文，随机生成`salt`，算法使用`bcrypt`，通过固定算法计算得出密文，密文前29字节对应加密使用的`salt`
- 如果不是域用户，会使用用户口令作为明文去计算，例如默认用户`admin`，会使用实际的口令作为明文去加密得到密文

也就是说，在查询表`public.aaapassword`时，我们只需要取出password项前29字节作为加密使用的`salt`，不需要关注表`public.aaapassword`中的salt项

### 2.区分是否为域用户

查询命令示例：`SELECT * FROM public.aaalogin ORDER BY login_id ASC;`

返回结果示例：

```
 login_id | user_id |     name      |         domainname
----------+---------+---------------+----------------------------
        1 |       1 | admin         | ADAuditPlus Authentication
      301 |     301 | Administrator | TEST
      310 |     310 | test1         | TEST
      311 |     311 | testadmin     | ADAuditPlus Authentication
(4 rows)
```

其中，`domainname`为`ADAuditPlus Authentication`代表自定义添加的用户

这里使用inner join查询自动筛选出非域用户和对应的hash，命令示例：`SELECT aaalogin.login_id,aaalogin.name,aaalogin.domainname,aaapassword.password FROM public.aaalogin as aaalogin INNER JOIN public.aaapassword AS aaapassword on aaalogin.login_id=aaapassword.password_id WHERE aaalogin.domainname = 'ADAuditPlus Authentication';`

返回结果示例：

```
 login_id|   name    |         domainname         |                           password
---------+-----------+----------------------------+--------------------------------------------------------------
       1 | admin     | ADAuditPlus Authentication | $2a$12$1hKeH4aM2LY4BvYpKT9Z5.p9cD453FjBAPYjp0ek94n936WRRAYme
     311 | testadmin | ADAuditPlus Authentication | $2a$12$1hKeH4aM2LY4BvYpKT9Z5.6zPb3SEFvW6PbGrU11Ilc96/YqJAEua
(2 rows)
```

## 0x04 算法实现
---

测试参数如下：

- 已知明文为`123456`
- 查询数据库得到的password项为`$2a$12$1hKeH4aM2LY4BvYpKT9Z5.p9cD453FjBAPYjp0ek94n936WRRAYme`

从中可知`salt`为password项的前29字节，即`$2a$12$1hKeH4aM2LY4BvYpKT9Z5.`

计算密文的测试代码如下：

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
        String password = "123456";
        String salt = "$2a$12$uVCggMRqSCSEwi6wh06Kd.";
        String encPass = getEncryptedPassword(password, salt, "bcrypt");
        System.out.println(encPass);   
    }
}
```

计算结果为`$2a$12$1hKeH4aM2LY4BvYpKT9Z5.p9cD453FjBAPYjp0ek94n936WRRAYme`，同数据库得到的password项一致

综上，根据以上算法可以用来对用户口令进行暴破

## 0x05 小结
---

本文分析了ADAudit Plus数据加密的算法，区分域用户，编写实现代码，后续根据算法可以用来对用户口令进行暴破。


---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


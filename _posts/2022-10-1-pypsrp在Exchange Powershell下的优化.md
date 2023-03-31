---
layout: post
title: pypsrp在Exchange Powershell下的优化
---


## 0x00 前言
---

[pypsrp](https://github.com/jborean93/pypsrp)是用于PowerShell远程协议(PSRP)服务的Python客户端。我在研究过程中，发现在Exchange Powershell下存在一些输出的问题，本文将要介绍研究过程，给出解决方法。

## 0x01 简介
---

- Exchange PowerShell Remoting
- pypsrp的使用
- pypsrp存在的输出问题
- 解决方法

## 0x02 Exchange PowerShell Remoting
---

参考资料：

https://docs.microsoft.com/en-us/powershell/module/exchange/?view=exchange-ps

默认设置下，需要注意以下问题：

- 所有域用户都可以连接Exchange PowerShell
- 需要在域内主机上发起连接
- 连接地址需要使用FQDN，不支持IP

通过Powershell连接Exchange PowerShell的命令示例：

```
$User = "test\user1"
$Pass = ConvertTo-SecureString -AsPlainText Password1 -Force
$Credential = New-Object System.Management.Automation.PSCredential -ArgumentList $User,$Pass
$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri http://exchangeserver.test.com/PowerShell/ -Authentication Kerberos -Credential $Credential
Invoke-Command -Session $session -ScriptBlock {Get-Mailbox -Identity administrator}
```

通过[pypsrp](https://github.com/jborean93/pypsrp)连接Exchange PowerShell的命令示例：

```
from pypsrp.powershell import PowerShell, RunspacePool
from pypsrp.wsman import WSMan
host = 'exchangeserver.test.com'
username='test\\user1'
password='Password1'
wsman = WSMan(server=host, username=username, password=password, port=80, path="PowerShell", ssl=False, auth="kerberos", cert_validation=False)     
with wsman, RunspacePool(wsman, configuration_name="Microsoft.Exchange") as pool:
    ps = PowerShell(pool)                  
    ps.add_cmdlet("Get-Mailbox").add_parameter("-Identity", "administrator")
    output = ps.invoke()
    print("[+] OUTPUT:\n%s" % "\n".join([str(s) for s in output]))
```

如果想要加入调试信息，可以添加以下代码：

```
import logging
logging.basicConfig(level=logging.DEBUG)
```

## 0x03 pypsrp存在的输出问题
---

我们在Exchange PowerShell下执行命令的完整返回结果如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-10-1/2-1.png)

但是通过[pypsrp](https://github.com/jborean93/pypsrp)连接Exchange PowerShell执行命令时，输出结果不完整，无法获得命令的完整信息，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-10-1/2-2.png)

## 0x04 解决方法
---

### 1.定位问题

通过查看源码，定位到代码位置：https://github.com/jborean93/pypsrp/blob/704f6cc49c8334f71b12ce10673964f037656782/src/pypsrp/messages.py#L207

我们可以在这里添加输出`message_data`的代码，代码示例：

```
print(message_data)
message_data = serializer.deserialize(message_data)
print(message_data)
```

返回结果：

```
<Obj RefId="0"><TN RefId="0"><T>Microsoft.Exchange.Data.Directory.Management.Mailbox</T><T>Microsoft.Exchange.Data.Directory.Management.MailEnabledOrgPerson</T><T>Microsoft.Exchange.Data.Directory.Management.MailEnabledRecipient</T><T>Microsoft.Exchange.Data.Directory.Management.ADPresentationObject</T><T>Microsoft.Exchange.Data.Directory.ADObject</T><T>Microsoft.Exchange.Data.Directory.ADRawEntry</T><T>Microsoft.Exchange.Data.ConfigurableObject</T><T>System.Object</T></TN><ToString>Administrator</ToString><Props><S N="Database">Ex2016-DB2</S><Nil N="MailboxProvisioningConstraint" /><Nil N="MailboxRegion" /><Nil N="MailboxRegionLastUpdateTime" /><B N="MessageCopyForSentAsEnabled">false</B><B N="MessageCopyForSendOnBehalfEnabled">false</B><Obj N="MailboxProvisioningPreferences" RefId="1"><TN RefId="1"><T>Microsoft.Exchange.Data.Directory.ADMultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.MailboxProvisioningConstraint, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.MailboxProvisioningConstraint, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST /></Obj><B N="UseDatabaseRetentionDefaults">true</B><B N="RetainDeletedItemsUntilBackup">false</B><B N="DeliverToMailboxAndForward">false</B><B N="IsExcludedFromServingHierarchy">false</B><B N="IsHierarchyReady">true</B><B N="IsHierarchySyncEnabled">true</B><B N="HasSnackyAppData">false</B><B N="LitigationHoldEnabled">false</B><B N="SingleItemRecoveryEnabled">false</B><B N="RetentionHoldEnabled">false</B><Nil N="EndDateForRetentionHold" /><Nil N="StartDateForRetentionHold" /><S N="RetentionComment"></S><S N="RetentionUrl"></S><Nil N="LitigationHoldDate" /><S N="LitigationHoldOwner"></S><B N="ElcProcessingDisabled">false</B><B N="ComplianceTagHoldApplied">false</B><B N="WasInactiveMailbox">false</B><B N="DelayHoldApplied">false</B><Nil N="InactiveMailboxRetireTime" /><Nil N="OrphanSoftDeleteTrackingTime" /><S N="LitigationHoldDuration">Unlimited</S><Nil N="ManagedFolderMailboxPolicy" /><Nil N="RetentionPolicy" /><Nil N="AddressBookPolicy" /><B N="CalendarRepairDisabled">false</B><G N="ExchangeGuid">9b9387fe-e1b1-4695-97bd-b0a0bebe7ce2</G><Nil N="MailboxContainerGuid" /><Nil N="UnifiedMailbox" /><Obj N="MailboxLocations" RefId="2"><TN RefId="2"><T>System.Collections.Generic.List`1[[Microsoft.Exchange.Data.Directory.IMailboxLocationInfo, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>System.Object</T></TN><LST><S>1;9b9387fe-e1b1-4695-97bd-b0a0bebe7ce2;Primary;test.com;0f29ba80-17c3-4c51-9d94-d017c850e3be</S></LST></Obj><Obj N="AggregatedMailboxGuids" RefId="3"><TN RefId="3"><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[System.Guid, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST /></Obj><S N="ExchangeSecurityDescriptor">System.Security.AccessControl.RawSecurityDescriptor</S><S N="ExchangeUserAccountControl">None</S><S N="AdminDisplayVersion">Version 15.1 (Build 2507.6)</S><B N="MessageTrackingReadStatusEnabled">true</B><S N="ExternalOofOptions">External</S><Nil N="ForwardingAddress" /><Nil N="ForwardingSmtpAddress" /><S N="RetainDeletedItemsFor">14.00:00:00</S><B N="IsMailboxEnabled">true</B><Obj N="Languages" RefId="4"><TN RefId="4"><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[System.Globalization.CultureInfo, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST><Obj RefId="5"><TN RefId="5"><T>System.Globalization.CultureInfo</T><T>System.Object</T></TN><ToString>en-US</ToString><Props><I32 N="LCID">1033</I32><S N="Name">en-US</S><S N="DisplayName">English (United States)</S><S N="IetfLanguageTag">en-US</S><S N="ThreeLetterISOLanguageName">eng</S><S N="ThreeLetterWindowsLanguageName">ENU</S><S N="TwoLetterISOLanguageName">en</S></Props></Obj></LST></Obj><Nil N="OfflineAddressBook" /><S N="ProhibitSendQuota">Unlimited</S><S N="ProhibitSendReceiveQuota">Unlimited</S><S N="RecoverableItemsQuota">30 GB (32,212,254,720 bytes)</S><S N="RecoverableItemsWarningQuota">20 GB (21,474,836,480 bytes)</S><S N="CalendarLoggingQuota">6 GB (6,442,450,944 bytes)</S><B N="DowngradeHighPriorityMessagesEnabled">false</B><Obj N="ProtocolSettings" RefId="6"><TN RefId="6"><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[System.String, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST><S>RemotePowerShell§1</S></LST></Obj><S N="RecipientLimits">Unlimited</S><B N="ImListMigrationCompleted">false</B><Nil N="SiloName" /><B N="IsResource">false</B><B N="IsLinked">false</B><B N="IsShared">false</B><B N="IsRootPublicFolderMailbox">false</B><S N="LinkedMasterAccount"></S><B N="ResetPasswordOnNextLogon">false</B><Nil N="ResourceCapacity" /><Obj N="ResourceCustom" RefId="7"><TNRef RefId="6" /><LST /></Obj><Nil N="ResourceType" /><Nil N="RoomMailboxAccountEnabled" /><S N="SamAccountName">Administrator</S><Nil N="SCLDeleteThreshold" /><Nil N="SCLDeleteEnabled" /><Nil N="SCLRejectThreshold" /><Nil N="SCLRejectEnabled" /><Nil N="SCLQuarantineThreshold" /><Nil N="SCLQuarantineEnabled" /><Nil N="SCLJunkThreshold" /><Nil N="SCLJunkEnabled" /><B N="AntispamBypassEnabled">false</B><S N="ServerLegacyDN">/o=First Organization/ou=Exchange Administrative Group (FYDIBOHF23SPDLT)/cn=Configuration/cn=Servers/cn=EXCHANGE02</S><S N="ServerName">exchange02</S><B N="UseDatabaseQuotaDefaults">true</B><S N="IssueWarningQuota">Unlimited</S><S N="RulesQuota">256 KB (262,144 bytes)</S><S N="Office"></S><S N="UserPrincipalName">Administrator@test.com</S><B N="UMEnabled">false</B><Nil N="MaxSafeSenders" /><Nil N="MaxBlockedSenders" /><Nil N="NetID" /><Nil N="ReconciliationId" /><S N="WindowsLiveID"></S><S N="MicrosoftOnlineServicesID"></S><Nil N="ThrottlingPolicy" /><S N="RoleAssignmentPolicy">Default Role Assignment Policy</S><Nil N="DefaultPublicFolderMailbox" /><Nil N="EffectivePublicFolderMailbox" /><S N="SharingPolicy">Default Sharing Policy</S><Nil N="RemoteAccountPolicy" /><Nil N="MailboxPlan" /><Nil N="ArchiveDatabase" /><G N="ArchiveGuid">00000000-0000-0000-0000-000000000000</G><Obj N="ArchiveName" RefId="8"><TNRef RefId="6" /><LST /></Obj><S N="JournalArchiveAddress"></S><S N="ArchiveQuota">100 GB (107,374,182,400 bytes)</S><S N="ArchiveWarningQuota">90 GB (96,636,764,160 bytes)</S><Nil N="ArchiveDomain" /><S N="ArchiveStatus">None</S><S N="ArchiveState">None</S><B N="AutoExpandingArchiveEnabled">false</B><B N="DisabledMailboxLocations">false</B><S N="RemoteRecipientType">None</S><Nil N="DisabledArchiveDatabase" /><G N="DisabledArchiveGuid">00000000-0000-0000-0000-000000000000</G><Nil N="QueryBaseDN" /><B N="QueryBaseDNRestrictionEnabled">false</B><Nil N="MailboxMoveTargetMDB" /><Nil N="MailboxMoveSourceMDB" /><S N="MailboxMoveFlags">None</S><S N="MailboxMoveRemoteHostName"></S><S N="MailboxMoveBatchName"></S><S N="MailboxMoveStatus">None</S><S N="MailboxRelease"></S><S N="ArchiveRelease"></S><B N="IsPersonToPersonTextMessagingEnabled">false</B><B N="IsMachineToPersonTextMessagingEnabled">true</B><Obj N="UserSMimeCertificate" RefId="9"><TN RefId="7"><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[System.Byte[], mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST /></Obj><Obj N="UserCertificate" RefId="10"><TNRef RefId="7" /><LST /></Obj><B N="CalendarVersionStoreDisabled">false</B><S N="ImmutableId"></S><Obj N="PersistedCapabilities" RefId="11"><TN RefId="8"><T>Microsoft.Exchange.Data.Directory.ADMultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.Capability, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.Capability, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST /></Obj><Nil N="SKUAssigned" /><B N="AuditEnabled">false</B><S N="AuditLogAgeLimit">90.00:00:00</S><Obj N="AuditAdmin" RefId="12"><TN RefId="9"><T>Microsoft.Exchange.Data.Directory.ADMultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.MailboxAuditOperations, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.MailboxAuditOperations, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST><S>Update</S><S>Move</S><S>MoveToDeletedItems</S><S>SoftDelete</S><S>HardDelete</S><S>FolderBind</S><S>SendAs</S><S>SendOnBehalf</S><S>Create</S></LST></Obj><Obj N="AuditDelegate" RefId="13"><TNRef RefId="9" /><LST><S>Update</S><S>SoftDelete</S><S>HardDelete</S><S>SendAs</S><S>Create</S></LST></Obj><Obj N="AuditOwner" RefId="14"><TNRef RefId="9" /><LST /></Obj><DT N="WhenMailboxCreated">2022-10-19T20:31:31-07:00</DT><S N="SourceAnchor"></S><Nil N="UsageLocation" /><B N="IsSoftDeletedByRemove">false</B><B N="IsSoftDeletedByDisable">false</B><B N="IsInactiveMailbox">false</B><B N="IncludeInGarbageCollection">false</B><Nil N="WhenSoftDeleted" /><Obj N="InPlaceHolds" RefId="15"><TNRef RefId="6" /><LST /></Obj><Obj N="GeneratedOfflineAddressBooks" RefId="16"><TN RefId="10"><T>Microsoft.Exchange.Data.Directory.ADMultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.ADObjectId, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[Microsoft.Exchange.Data.Directory.ADObjectId, Microsoft.Exchange.Data.Directory, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST /></Obj><B N="AccountDisabled">false</B><Nil N="StsRefreshTokensValidFrom" /><Nil N="DataEncryptionPolicy" /><B N="DisableThrottling">false</B><Obj N="Extensions" RefId="17"><TNRef RefId="6" /><LST /></Obj><B N="HasPicture">false</B><B N="HasSpokenName">false</B><B N="IsDirSynced">false</B><Obj N="AcceptMessagesOnlyFrom" RefId="18"><TNRef RefId="10" /><LST /></Obj><Obj N="AcceptMessagesOnlyFromDLMembers" RefId="19"><TNRef RefId="10" /><LST /></Obj><Obj N="AcceptMessagesOnlyFromSendersOrMembers" RefId="20"><TNRef RefId="10" /><LST /></Obj><Obj N="AddressListMembership" RefId="21"><TNRef RefId="10" /><LST><S>\Mailboxes(VLV)</S><S>\All Mailboxes(VLV)</S><S>\All Recipients(VLV)</S><S>\Default Global Address List</S><S>\All Users</S></LST></Obj><Obj N="AdministrativeUnits" RefId="22"><TNRef RefId="10" /><LST /></Obj><S N="Alias">Administrator</S><Nil N="ArbitrationMailbox" /><Obj N="BypassModerationFromSendersOrMembers" RefId="23"><TNRef RefId="10" /><LST /></Obj><S N="OrganizationalUnit">test.com/Users</S><S N="CustomAttribute1"></S><S N="CustomAttribute10"></S><S N="CustomAttribute11"></S><S N="CustomAttribute12"></S><S N="CustomAttribute13"></S><S N="CustomAttribute14"></S><S N="CustomAttribute15"></S><S N="CustomAttribute2"></S><S N="CustomAttribute3"></S><S N="CustomAttribute4"></S><S N="CustomAttribute5"></S><S N="CustomAttribute6"></S><S N="CustomAttribute7"></S><S N="CustomAttribute8"></S><S N="CustomAttribute9"></S><Obj N="ExtensionCustomAttribute1" RefId="24"><TNRef RefId="6" /><LST /></Obj><Obj N="ExtensionCustomAttribute2" RefId="25"><TNRef RefId="6" /><LST /></Obj><Obj N="ExtensionCustomAttribute3" RefId="26"><TNRef RefId="6" /><LST /></Obj><Obj N="ExtensionCustomAttribute4" RefId="27"><TNRef RefId="6" /><LST /></Obj><Obj N="ExtensionCustomAttribute5" RefId="28"><TNRef RefId="6" /><LST /></Obj><S N="DisplayName">Administrator</S><Obj N="EmailAddresses" RefId="29"><TN RefId="11"><T>Microsoft.Exchange.Data.ProxyAddressCollection</T><T>Microsoft.Exchange.Data.ProxyAddressBaseCollection`1[[Microsoft.Exchange.Data.ProxyAddress, Microsoft.Exchange.Data, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedProperty`1[[Microsoft.Exchange.Data.ProxyAddress, Microsoft.Exchange.Data, Version=15.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]</T><T>Microsoft.Exchange.Data.MultiValuedPropertyBase</T><T>System.Object</T></TN><LST><S>SMTP:Administrator@test.com</S></LST></Obj><Obj N="GrantSendOnBehalfTo" RefId="30"><TNRef RefId="10" /><LST /></Obj><S N="ExternalDirectoryObjectId"></S><B N="HiddenFromAddressListsEnabled">false</B><Nil N="LastExchangeChangedTime" /><S N="LegacyExchangeDN">/o=First Organization/ou=Exchange Administrative Group (FYDIBOHF23SPDLT)/cn=Recipients/cn=d1ce5d43a9184ce3ba01e101a19d0a28-Admin</S><S N="MaxSendSize">Unlimited</S><S N="MaxReceiveSize">Unlimited</S><Obj N="ModeratedBy" RefId="31"><TNRef RefId="10" /><LST /></Obj><B N="ModerationEnabled">false</B><Obj N="PoliciesIncluded" RefId="32"><TNRef RefId="6" /><LST><S>e1a339c9-ac14-4226-9cf8-6de2e2b48d6d</S><S>{26491cfc-9e50-4857-861b-0cb8df22b5d7}</S></LST></Obj><Obj N="PoliciesExcluded" RefId="33"><TNRef RefId="6" /><LST /></Obj><B N="EmailAddressPolicyEnabled">true</B><S N="PrimarySmtpAddress">Administrator@test.com</S><S N="RecipientType">UserMailbox</S><S N="RecipientTypeDetails">UserMailbox</S><Obj N="RejectMessagesFrom" RefId="34"><TNRef RefId="10" /><LST /></Obj><Obj N="RejectMessagesFromDLMembers" RefId="35"><TNRef RefId="10" /><LST /></Obj><Obj N="RejectMessagesFromSendersOrMembers" RefId="36"><TNRef RefId="10" /><LST /></Obj><B N="RequireSenderAuthenticationEnabled">false</B><S N="SimpleDisplayName"></S><S N="SendModerationNotifications">Always</S><Obj N="UMDtmfMap" RefId="37"><TNRef RefId="6" /><LST><S>emailAddress:2364647872867</S><S>lastNameFirstName:2364647872867</S><S>firstNameLastName:2364647872867</S></LST></Obj><S N="WindowsEmailAddress">Administrator@test.com</S><Nil N="MailTip" /><Obj N="MailTipTranslations" RefId="38"><TNRef RefId="6" /><LST /></Obj><S N="Identity">test.com/Users/Administrator</S><B N="IsValid">true</B><S N="ExchangeVersion">0.20 (15.0.0.0)</S><S N="Name">Administrator</S><S N="DistinguishedName">CN=Administrator,CN=Users,DC=test,DC=com</S><G N="Guid">c07d2fbc-48f5-4a90-b7d7-5ec79d2844b4</G><S N="ObjectCategory">test.com/Configuration/Schema/Person</S><Obj N="ObjectClass" RefId="39"><TNRef RefId="6" /><LST><S>top</S><S>person</S><S>organizationalPerson</S><S>user</S></LST></Obj><DT N="WhenChanged">2022-10-30T18:01:36-07:00</DT><DT N="WhenCreated">2022-10-19T18:10:53-07:00</DT><DT N="WhenChangedUTC">2022-10-31T01:01:36Z</DT><DT N="WhenCreatedUTC">2022-10-20T01:10:53Z</DT><S N="OrganizationId"></S><S N="Id">test.com/Users/Administrator</S><S N="OriginatingServer">dc01.test.com</S><S N="ObjectState">Unchanged</S></Props></Obj>
Administrator
```

在调用`serializer.deserialize(message_data)`提取输出结果时，这里只提取到了一组数据，忽略了完整的结果

经过简单的分析，发现`<Props></Props>`标签内包含完整的输出结果，所以这里可先通过字符串截取提取出`<Props></Props>`标签内的数据，示例代码：

```
props_data = message_data[message_data.find('<Props'):message_data.rfind('</Props>')+8]
```

进一步分析提取出来的数据，发现每个标签`<S></S>`分别对应一项属性，为了提高效率，这里使用`xml.dom.minidom`解析成xml格式并提取元素，示例代码：

```
from xml.dom import minidom
dom = minidom.parseString(props_data)   
data_s = dom.getElementsByTagName("S")
for data_i in data_s:
    if data_i.firstChild and len(data_i.getAttribute('N'))>0:
        key = data_i.getAttribute('N')          #每个子节点属性'N'的名称
        value = data_i.firstChild.data          #每个子节点的值
        print('{:32s}: {}'.format(key,value))   #格式化输出
```

经测试，以上代码能够输出完整的结果

按照[pypsrp](https://github.com/jborean93/pypsrp)的代码格式，得出优化[pypsrp](https://github.com/jborean93/pypsrp)输出结果的代码：

        try:
            if "Microsoft.Exchange" in message_data:
                #try to output the complete results of the Exchange Powershell
                props_data = message_data[message_data.find('<Props'):message_data.rfind('</Props>')+8]
                from xml.dom import minidom
                dom = minidom.parseString(props_data)   
                data_s = dom.getElementsByTagName("S")
                temp_data = ""
                for data_i in data_s:
                    if data_i.firstChild and len(data_i.getAttribute('N'))>0:
                        key = data_i.getAttribute('N')          
                        value = data_i.firstChild.data
                        temp_data += '{:32s}: {}\r\n'.format(key,value)
                message_data = temp_data       
            else:
                message_data = serializer.deserialize(message_data)


使用修改过的[pypsrp](https://github.com/3gstudent/pypsrp)连接Exchange PowerShell执行命令时，能够返回完整的输出结果，如下图

![Alt text](https://raw.githubusercontent.com/3gstudent/BlogPic/master/2022-10-1/2-3.png)

经测试，在测试ProxyShell的过程中，使用修改过的[pypsrp](https://github.com/3gstudent/pypsrp)也能得到完整的输出结果

### 补充：

如果使用原始版本[pypsrp](https://github.com/jborean93/pypsrp)测试ProxyShell，可通过解析代理的返回结果实现，其中需要注意的是在作Base64解密时，由于存在不可见字符，无法使用`.decode('utf-8')`解码，可以换用`.decode('ISO-8859-1')`，还需要考虑数据被分段的问题，实现的示例代码如下：

```
data_base64 = ""
data = re.compile(r"<rsp:Stream(.*?)</rsp:Stream>").findall(res.text)                 
for i in range(len(data)):               
    data_base64 = data_base64 + data[i][64:]
data_decrypt = base64.b64decode(data_base64).decode('ISO-8859-1')
```

## 0x05 小结
---

本文介绍了通过[pypsrp](https://github.com/jborean93/pypsrp)连接Exchange PowerShell执行命令返回完整输出结果的解决方法。

---


[LEAVE A REPLY](https://github.com/3gstudent/feedback/issues/new)


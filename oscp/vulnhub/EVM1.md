# 信息搜集

补充下，这台机器需要修改一些配置才能得到 ip，[关于Vulnhub 靶机ip无法扫描到的解决方法_vmware 靶机net无法扫描_葫芦娃42的博客-CSDN博客](https://blog.csdn.net/weixin_63231007/article/details/125711686)

1、端口扫描

这里是分步骤扫描命令

<img src=".\图片\Snipaste_2023-08-17_15-13-19.png" alt="Snipaste_2023-08-17_15-13-19" style="zoom:80%;" />

```bash
nmap -sn -n --min-rate 1000 192.168.10.0/24
ip=192.168.10.7
ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a2d3341362b118a3dddb35c55ab7c078 (RSA)
|   256 8548532a50c5a0b71aeea4d8128e1cce (ECDSA)
|_  256 362292c73222e33451bc0e749f1cdbaa (ED25519)
53/tcp  open  domain      ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
110/tcp open  pop3?
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: CAPABILITY
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
MAC Address: 00:0C:29:D6:24:B3 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host: UBUNTU-EXTERMELY-VULNERABLE-M4CH1INE; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h19m58s, deviation: 2h18m33s, median: -1s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: UBUNTU-EXTERMEL, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-time: 
|   date: 2023-08-17T07:33:06
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: ubuntu-extermely-vulnerable-m4ch1ine
|   NetBIOS computer name: UBUNTU-EXTERMELY-VULNERABLE-M4CH1INE\x00
|   Domain name: \x00
|   FQDN: ubuntu-extermely-vulnerable-m4ch1ine
|_  System time: 2023-08-17T03:33:06-04:00
```

2、枚举 smb 用户

```bash
└─# enum4linux -A 192.168.10.7 
```

啥也没有

3、看 80 端口，发现一个 WordPress 站点

<img src=".\图片\Snipaste_2023-08-17_15-43-11.png" alt="Snipaste_2023-08-17_15-43-11" style="zoom:50%;" />

靶机坏了，gg

# 总结技巧

[EVM靶机通关 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/351650.html)

- wpscan 使用 [wpscan 工具使用教程_wpscan使用教程_liver100day的博客-CSDN博客](https://blog.csdn.net/liver100day/article/details/117585795)

  常用命令集合    `wpscan --url http://dc-2 --enumerate vp,vt,tt,u`

- `cewl` 工具可以对应网站生成社工字典
- `wpscan` 后台 编辑 `Theme` 内容写 shell，主题模版的网站路径在 `http://192.168.56.103/wordpress/wp-content/themes/twentynineteen/404.php`
- 得到 `www` 权限后记得找找网站的配置文件里的用户名密码信息
- 提取阶段记得递归看看 `/home` 目录下的敏感信息，可用 `ls -alR` 命令递归查看



配置 wpscan 的 `api token`

接下来，使用 wpscan 工具进行针对性扫描。在这之前，先要配置 API Token。在 [WPScan](https://wpscan.com/) 官网注册登录后，进入 Profile 页面获取 API Token，每天有 25 次免费调用机会

并在 Kali Linux 当前 root 用户的主目录下，创建 `~/.wpscan/scan.yml` 配置文件，写入以下内容：

```
cli_options:
  api_token: YOUR_API_TOKEN
```

扫 https 时要注意忽略证书检测 `--disable-tls-checks`



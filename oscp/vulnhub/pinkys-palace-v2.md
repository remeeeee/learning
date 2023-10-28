绑定 hosts

**Note From VulnHub**: Wordpress will not render correctly. You will need to alter your host file with the IP shown on the console: `echo 192.168.x.x pinkydb | sudo tee -a /etc/hosts`

[(161条消息) No.27-VulnHub-Pinky's Palace: v2-Walkthrough渗透学习_大余xiyou的博客-CSDN博客](https://blog.csdn.net/qq_34801745/article/details/104070421)

[[2023/05更新预告\]反汇编神器IDA Pro 7.7.220118 (SP1)汉化版 - 『逆向资源区』 - 吾爱破解 - LCG - LSG |安卓破解|病毒分析|www.52pojie.cn](https://www.52pojie.cn/thread-1640829-1-1.html)

[VulnHub-Pinky’s Palace: v2-靶机渗透学习 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/261184.html)

# 信息搜集

1、nmap 扫描端口

```bash
└─# nmap -sVC -O -A -p- 192.168.0.102 -oA nmap.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-27 04:38 EDT
Nmap scan report for pinkydb (192.168.0.102)
Host is up (0.00042s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE    SERVICE VERSION
80/tcp    open     http    Apache httpd 2.4.25 ((Debian))
|_http-generator: WordPress 4.9.4
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Pinky&#039;s Blog &#8211; Just another WordPress site
4655/tcp  filtered unknown
7654/tcp  filtered unknown
31337/tcp filtered Elite
MAC Address: 00:0C:29:9A:EF:4E (VMware)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
```

2、80 端口查看，显示了 `another WorldPress`，由此可见可能存在其它站点，发现用户名 `plink`，留心并加入到用户名的字典

<img src=".\图片\Snipaste_2023-06-27_16-42-33.png" alt="Snipaste_2023-06-27_16-42-33" style="zoom:80%;" />

3、上 wpscan 扫描

 [WPScan](https://wpscan.com/) 官网，配置，并在 Kali Linux 当前 root 用户的主目录下，创建 `~/.wpscan/scan.yml` 配置文件，写入以下内容：

```yml
cli_options:
  api_token: L6qtLJk0swdg0XRMKX6eU5BJE6I0fbV52H8wFLcZKXY
```

枚举用户名

```
wpscan --url http://pinkydb --enumerate u1-100
```

發現用户名 `pinky1337`

4、目录扫描

```bash
└─# gobuster dir -u http://pinkydb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js,php.bak,txt.bak,html.bak,json,git,git.bak 
```

```
/.html.bak            (Status: 403) [Size: 291]
/.html                (Status: 403) [Size: 287]
/index.php            (Status: 301) [Size: 0] [--> http://pinkydb/]
/.php                 (Status: 403) [Size: 286]
/wp-content           (Status: 301) [Size: 307] [--> http://pinkydb/wp-content/]
/wp-login.php         (Status: 200) [Size: 2239]
/wordpress            (Status: 301) [Size: 306] [--> http://pinkydb/wordpress/]
/license.txt          (Status: 200) [Size: 19935]
/wp-includes          (Status: 301) [Size: 308] [--> http://pinkydb/wp-includes/]
/readme.html          (Status: 200) [Size: 7413]
/wp-trackback.php     (Status: 200) [Size: 135]
/secret               (Status: 301) [Size: 303] [--> http://pinkydb/secret/]
/wp-admin             (Status: 301) [Size: 305] [--> http://pinkydb/wp-admin/]
/xmlrpc.php           (Status: 405) [Size: 42]
/.php                 (Status: 403) [Size: 286]
/.html                (Status: 403) [Size: 287]
/.html.bak            (Status: 403) [Size: 291]
/wp-signup.php        (Status: 302) [Size: 0] [--> http://pinkydb/wp-login.php?action=register]
/server-status        (Status: 403) [Size: 295]
```

查看下 `http://pinkydb/secret/`

<img src=".\图片\Snipaste_2023-06-27_16-58-59.png" alt="Snipaste_2023-06-27_16-58-59" style="zoom:80%;" />

# web打点

## 敲开端口

紧接上文，bambam.txt下 是三个端口号，但是 nmap 并没有扫描出这些端口，我这边用新的方式打开这三个隐藏端口

```bash
knock pinkydb 8890 7000 666
```

端口没有变化，使用python,根据上述的3个数字列出全部的排列组合，因为不知道其是根据哪个顺序来判断的

```python
from itertools import combinations, permutations
print(list(permutations([8890,7000,666])))
```

使用脚本将排列组合，knock测试

> \#!/bin/bash
>
> while read -r line
>
> do
>
> echo '---------------'
>
> knock -v 192.168.0.102 $line
>
> done < knock.txt

```bash
┌──(root💀kali)-[~/Pinkys_Palace_v2]
└─# ./port.sh  
---------------
hitting tcp 192.168.0.102:8890
hitting tcp 192.168.0.102:7000
hitting tcp 192.168.0.102:666
---------------
hitting tcp 192.168.0.102:8890
hitting tcp 192.168.0.102:666
hitting tcp 192.168.0.102:7000
---------------
hitting tcp 192.168.0.102:7000
hitting tcp 192.168.0.102:8890
hitting tcp 192.168.0.102:666
---------------
hitting tcp 192.168.0.102:7000
hitting tcp 192.168.0.102:666
hitting tcp 192.168.0.102:8890
---------------
hitting tcp 192.168.0.102:666
hitting tcp 192.168.0.102:8890
hitting tcp 192.168.0.102:7000
---------------
hitting tcp 192.168.0.102:666
hitting tcp 192.168.0.102:7000
hitting tcp 192.168.0.102:8890
                                                                                                                                                                  
┌──(root💀kali)-[~/Pinkys_Palace_v2]
└─# cat knock.txt            
8890 7000 666
8890 666 7000
7000 8890 666
7000 666 8890
666  8890 7000
666  7000 8890
```

再用nmap探测端口

好的，我敲不开端口，gg，失败

# 大佬wp

[(161条消息) No.27-VulnHub-Pinky's Palace: v2-Walkthrough渗透学习_大余xiyou的博客-CSDN博客](https://blog.csdn.net/qq_34801745/article/details/104070421)

[VulnHub-Pinky’s Palace: v2-靶机渗透学习 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/261184.html)



# 总结

可以试用 knock 命令敲开一些端口

`cewl`  工具 可以社工网站形成字典

`ssh2john` 将将密钥转换为可破解的哈希，使用`John the Ripper`破解这个哈希值，这样就可以破解私有ssh密钥的密码

```
python ssh2john.py  id_rsa > id_rsa.txt
john -w=/usr/share/wordlists/rockyou.txt id_rsa.txt
```

二进制不会，后来再慢慢学吧



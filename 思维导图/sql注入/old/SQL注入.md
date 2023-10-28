# SQL注入

## 1、基础知识

在 mysql5 版本以后，有一个内置的重点的数据库叫作 infomation_schema ， 这个库里面有很多表，重点是这三个表 SCHEMATA 、TABLES和COLUMNS 。

SCHEMATA表中重点的是记录着数据库名的字段SCHEMA_NAME。

<img src=".\image\image-20220602151103374.png" alt="image-20220602151103374" style="zoom:67%;" />



TABLES表中重点关注数据库库名TABLES_SCHEMA和表名的字段名TABLE_NAME。

<img src=".\image\Snipaste_2022-06-02_15-25-11.png" alt="Snipaste_2022-06-02_15-25-11" style="zoom: 50%;" />



COLUMNS表中重点关注数据库库名TABLE_SCHEMA、表名TABLE_NAME、字段名COLUMN_NAME。

<img src=".\image\Snipaste_2022-06-02_15-29-36.png" alt="Snipaste_2022-06-02_15-29-36" style="zoom: 50%;" />

## 2、union联合注入

### 判断注入：

确定为字符型，再 1‘ and ’1‘ = ’1 ，页面返回正常，再 1‘ and ’1‘ = ’2 页面返回不一样的内容，基本确定存在注入。

### 判断字段数：

利用order by，1' order by 2--+ 正常，1' order by 3--+ 出错，则字段数为2。

<img src=".\image\Snipaste_2022-06-02_15-45-10.png" alt="Snipaste_2022-06-02_15-45-10" style="zoom: 50%;" />

### 利用联合查询找出敏感信息：

version() ：mysql 版本； database()： 当前数据库 ；user()： 当前用户名； group_concat()：分组打印字符串。

<img src=".\image\Snipaste_2022-06-02_15-52-37.png" alt="Snipaste_2022-06-02_15-52-37" style="zoom: 50%;" />

### 通过information_schema 获取表:

limit 0,1时显示第一个表为guestbook，limit1,1时显示第二个表为users。

<img src=".\image\Snipaste_2022-06-02_15-59-55.png" alt="Snipaste_2022-06-02_15-59-55" style="zoom: 50%;" />

### 通过 information_schema 获取字段：

由于我们已经得知一个重要的表名为users，再利用infomation_schema库中的COLUMNS表来获取字段。从limit 0,1开始limit 

1,1再limit 2,1逐渐。。。最后得出想要的字段user 与password。

<img src=".\image\Snipaste_2022-06-02_16-11-14.png" alt="Snipaste_2022-06-02_16-11-14" style="zoom:50%;" />

### 最后再利用已知信息查询：

我们得到了库名dvwa，表名users，字段名为user与password。则利用查询语句，可得到改库中的用户名与密码。

<img src=".\image\Snipaste_2022-06-02_16-20-25.png" alt="Snipaste_2022-06-02_16-20-25" style="zoom:80%;" />



## 3、布尔盲注

### 判断盲注：

当我们遇到，在页面中不会显示数据库信息，只会显示对与错时，可考虑使用盲注。使用1' and '1'=1时显示User ID exists in the database，当使用-1' and '1'=1时显示User ID is MISSING from the database。我们可以用延时注入判断1'and sleep(10)，结果显示10秒后才响应。

### 判断当前库长度：

使用 if(length(database())=1,1,0)时显示User ID is MISSING from the databas，判断数据库长度不为1；再以此次if(length(database())=2,1,0)判断长度是否为2。。。最后得出当前数据库长度为4。

<img src=".\image\Snipaste_2022-06-02_16-37-24.png" alt="Snipaste_2022-06-02_16-37-24" style="zoom: 67%;" />

### 判断当前库名称：

SUBSTRING()字符串截取，第一个参数是字符串，第二个参数是开始截取位置， 第三个是截取的长度。id=1' and if(substring(database(),1,1)='d',1,0)是判断当前库的第一个字符是否为d。那么我们接下来借用brupsuite进行爆破可得到数据库名。

<img src=".\image\Snipaste_2022-06-02_16-43-11.png" alt="Snipaste_2022-06-02_16-43-11" style="zoom:67%;" />

### 再获取表名列名和查询：

接下来把substring()函数中的语句替换即可。盲注方法就是通过判断条件，页面会告诉你这个条件是否正确，然后再一点点测试，所以盲注也是比较费时间的。

## 3、报错注入


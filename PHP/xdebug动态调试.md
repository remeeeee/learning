# 前置说明

以下操作均在 wooyun 虚拟机里操作配置的

## 参考：

[(149条消息) phpStudy + PhpStorm + XDebug调试【绝对能用】_phpstudy用xdebug_大方子的博客-CSDN博客](https://blog.csdn.net/nzjdsds/article/details/100114242)

[xdebug phpstudy phpstorm 配置文件 pip.ini | (moonsec.com)](https://www.moonsec.com/4500.html)

PhpStudy+Apache(Fastcgi)+Xdebug调试时间过长出现500时解决方法：
https://www.moonsec.com/1881.html

# 步骤

## Step1

开启对应php版本的xdebug拓展

<img src=".\图片\Snipaste_2023-05-24_14-11-07.png" alt="Snipaste_2023-05-24_14-11-07" style="zoom:67%;" />

## Step2

修改对应版本的php.ini，增加或者替换如下为：

```
[XDebug]
xdebug.profiler_append = 0
;效能监测的设置开关
xdebug.profiler_enable = 1
xdebug.profiler_enable_trigger = 0
;profiler_enable设置为1的时候，效能监测信息写入文件所在的目录
xdebug.profiler_output_dir="C:\phpstudy_pro\temp\xdebug"
;设置的函数调用监测信息的输出路径
xdebug.trace_output_dir="C:\phpstudy_pro\temp\xdebug"
;生成的效能监测文件的名字
xdebug.profiler_output_name ="cache.out.%t-%s"
; IDE与XDebug协作
xdebug.remote_enable = 1
xdebug.remote_handler = "dbgp"
xdebug.remote_host = "127.0.0.1"
xdebug.remote_port = 9000
xdebug.idekey = phpstorm-xdebug
;.dll文件的路径
zend_extension="C:/phpstudy_pro/Extensions/php/php7.3.4nts/ext/php_xdebug.dll"
```

这个路径 `C:\phpstudy_pro\temp\xdebug` 不存在就新建这个目录

## Step3

phpstorm配置

### 1、选择PHP版本

<img src=".\图片\Snipaste_2023-05-24_14-22-32.png" alt="Snipaste_2023-05-24_14-22-32" style="zoom:67%;" />

### 2、Debug端口

与 php.ini 里配置得一致，都为 9000

<img src=".\图片\Snipaste_2023-05-24_14-23-40.png" alt="Snipaste_2023-05-24_14-23-40" style="zoom:67%;" />

### 3、设置服务器

<img src=".\图片\Snipaste_2023-05-24_14-26-17.png" alt="Snipaste_2023-05-24_14-26-17" style="zoom:80%;" />

### 4、配置host端口

IDE key 与 php.ini 中 xdebug. idekey 值一致

Port 与 php.ini 中 xdebug.remote_port 值一致

<img src=".\图片\Snipaste_2023-05-24_14-27-35.png" alt="Snipaste_2023-05-24_14-27-35" style="zoom:67%;" />

### 5、增加PHP Web Page

先点击：Run-> Edit Configurations–> PHP Web Application

<img src=".\图片\Snipaste_2023-05-24_14-29-56.png" alt="Snipaste_2023-05-24_14-29-56" style="zoom:67%;" />

### 6、点击小电话图标

点击小电话图标，打开监听（图中表示已经开启监听）

<img src=".\图片\Snipaste_2023-05-24_14-30-50.png" alt="Snipaste_2023-05-24_14-30-50" style="zoom:80%;" />

## Step4

1、下断点

<img src=".\图片\Snipaste_2023-05-24_14-33-08.png" alt="Snipaste_2023-05-24_14-33-08" style="zoom:67%;" />

2、访问网页调试

<img src=".\图片\Snipaste_2023-05-24_14-34-35.png" alt="Snipaste_2023-05-24_14-34-35" style="zoom:60%;" />

<img src=".\图片\Snipaste_2023-05-24_14-36-18.png" alt="Snipaste_2023-05-24_14-36-18" style="zoom:80%;" />



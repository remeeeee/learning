[VulnHub-Lin.Security: 1-靶机渗透学习 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/260506.html)

[(160条消息) [VulnHub靶机\]Lin.Security_linux提权_Stray.io的博客-CSDN博客](https://blog.csdn.net/qq_45927819/article/details/123464242)

# 提权靶场

https://gtfobins.github.io/

ssh登录测试用户 `bob/secret`，我这里直接登录了，没用ssh

## 信息搜集

```
find / -perm -u=s -type f 2>/dev/null
sudo -l
```

<img src=".\图片\Snipaste_2023-06-26_16-05-49.png" alt="Snipaste_2023-06-26_16-05-49" style="zoom:80%;" />

## 示范操作

```
sudo vi
```

<img src=".\图片\Snipaste_2023-06-26_16-17-45.png" alt="Snipaste_2023-06-26_16-17-45"  />
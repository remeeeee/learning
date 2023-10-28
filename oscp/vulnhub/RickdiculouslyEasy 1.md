[Vulnhub-靶机-RICKDICULOUSLYEASY: 1 - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13650621.html)

[vulnhub RickdiculouslyEasy: 1_仙女象的博客-CSDN博客](https://blog.csdn.net/elephantxiang/article/details/124955857)

# 总结

- 21 端口 匿名 访问 ftp，`Anonymous`

- 不熟悉的端口可以尝试用 nc 连接， `nc 192.168.101.70 60000`

- `dirb`  工具扫描 web 目录

- 发现命令注入时，类似 `127.0.0.1;whoami` ，无法 反弹 shell 时（这可以用 nc 反弹），那就顺从它，可尝试远程下载后门执行，可尝试读取服务器敏感文件如 `/etc/passwd` 

  <img src=".\图片\700440-20200911094148015-616418509.png" alt="700440-20200911094148015-616418509" style="zoom:67%;" />

- `cat` 命令还是不好使时，用 `more` 命令查看 文件
- 图片隐写，可用 `strings Safe_Password.jpg ` 命令
- 进入`/home/RickSanchez/RICKS_SAFE` 目录可以发现 `safe` 文件，是可执行的，但是当前用户 `Summer` 没有执行权限。由于Summer 不是 safe 文件的属主，所以也无法修改 safe 文件权限。将 safe 拷贝到 Summer 的家目录下，命名为 safe2，虽然其文件权限没变，属主却变成了 Summer，此时 Summer 便有 safe2 文件的执行权限了

<img src=".\图片\a8d9723d6cf1ae8ab80723ae8c4a44de.png" alt="a8d9723d6cf1ae8ab80723ae8c4a44de" style="zoom:80%;" />

<img src=".\图片\75629323b5ee4f6a9b1f6e26aa148ae5.png" alt="75629323b5ee4f6a9b1f6e26aa148ae5" style="zoom:80%;" />

- `crunch` 工具生成密码字典

  除了flag，上述打印还提示了 RickSanchez 用户的密码，依次是：

  1个大写字母，一个数字，RickSanchez 的老乐队名字中的一个单词

​       用 crunch 制作密码字典，其中 -t 表示指定格式，% 表示数字，逗号 表示大写字母

​       `crunch 7 7 -t ,%Flesh > rick.txt;crunch 10 10 -t ,%Curtains >> rick.txt`

- 九头蛇爆破 ssh，`hydra -l RickSanchez -P "rick.txt" ssh://192.168.101.70:22222`，端口可以指定，服务器可能换了 ssh 的端口，比如这里就换成了 `22222`

- `sudo -l` 发现 `RickSanchez` 可以以任何身份执行任何命令，于是 `sudo su` 提权

<img src=".\图片\e8e680330d5debbe001d1dd12c81b5d8.png" alt="e8e680330d5debbe001d1dd12c81b5d8" style="zoom:80%;" />





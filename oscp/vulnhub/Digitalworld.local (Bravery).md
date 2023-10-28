[(162条消息) 靶机渗透练习53-digitalworld.local：BRAVERY_digitalworld.local: electrical_hirak0的博客-CSDN博客](https://blog.csdn.net/Perpetual_Blue/article/details/124265751)

[Vulnhub-靶机-DIGITALWORLD.LOCAL: BRAVERY - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13786382.html)

总结：

1、`enum4linux` 枚举 linux 的相关用户

2、`smbclient`  `smbclient //192.168.9.51/anonymous`

3、nfs挂载目录可以查看信息

4、信息搜集过程中发现用户名和密码或其它关键信息时，注意保存为文件记录下来

5、suid 权限中 cp 命令提权，复制替换 /etc/passwd，在 su 提权
[vulnhub sar 靶场练习 - rpsate - 博客园 (cnblogs.com)](https://www.cnblogs.com/masses0x1/p/15780877.html)

[Vulnhub-靶机-SAR: 1 - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13673015.html)

[【学渗透靶机——sar 1】_懒惰的老鼠的博客-CSDN博客](https://blog.csdn.net/jieli0901227/article/details/131271829)



在一个文件夹递归查看其中的 文件或文件夹，更方便

```
ls -laRh
```



在计划任务提权里，编写修改脚本时

- 可以使用各种语言的反弹shell
- 可以修改 /etc/passwd
- 可以 `cp /bin/bash temp/bash_fake` 再添加 s 位 权限，`chmod +s temp/bash_fake`  ，再  `temp/bash_fake -p` 提权
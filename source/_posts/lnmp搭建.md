---
title: lnmp环境搭建
categories: 实践与总结
tag: 实践与总结
abbrlink: 110e9b1f
date: 2017-03-27 00:00:00
---
## 准备Linux环境
环境为CentOS6.7最小化安装
安装开发包组
```
yum groupinstall "Development Tools"
yum -y install pcre-devel bzip2-devel libmcrypt-devel gcc  gcc-c++ openssl-devel pcre-devel libxml2-devel libcurl-devel libpng-devel freetype-devel libxslt-devel libtool gd-devel
```

<!-- more -->
## 安装MySQL
```
1、准备MySQL数据存放的文件目录

数据目录为/data，而后需要创建/data/mysql目录做为mysql数据的存放目录。

2、新建用户以安全方式运行进程：

# groupadd -r mysql
# useradd -g mysql -r -s /sbin/nologin -M -d /data/mysql mysql
# chown -R mysql:mysql /data/mysql

3、安装并初始化mysql
https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.19-linux-glibc2.12-x86_64.tar.gz

# tar xf mysql-5.7.19-linux-glibc2.12-x86_64.tar.gz -C /usr/local
# cd /usr/local/
# ln -sv mysql-5.7.19-linux-glibc2.12-x86_64  mysql
# cd mysql 

# chown -R mysql:mysql  .
# bin/mysql_install_db --user=mysql --datadir=/data/mysql
# chown -R root  .

4、为mysql提供主配置文件：

参考链接https://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html
从MySQL 5.7.18开始，my-default.ini不再包含在分发包中或由分发包安装。
# vim /etc/my.cnf
[mysqld]
# innodb_buffer_pool_size = 128M
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/data/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid



5、为mysql提供sysv服务脚本：

# cd /usr/local/mysql
# cp support-files/mysql.server  /etc/rc.d/init.d/mysqld

添加至服务列表：
# chkconfig --add mysqld
# chkconfig mysqld on

而后就可以启动服务测试使用了。


为了使用mysql的安装符合系统使用规范，并将其开发组件导出给系统使用，这里还需要进行如下步骤：

6、输出mysql的man手册至man命令的查找路径：

编辑/etc/man.config，添加如下行即可：
MANPATH  /usr/local/mysql/man

7、输出mysql的头文件至系统头文件路径/usr/include：

这可以通过简单的创建链接实现：
# ln -sv /usr/local/mysql/include  /usr/include/mysql

8、输出mysql的库文件给系统库查找路径：

# echo '/usr/local/mysql/lib' > /etc/ld.so.conf.d/mysql.conf

而后让系统重新载入系统库：
# ldconfig

9、修改PATH环境变量，让系统可以直接使用mysql的相关命令。
在/etc/profile.d下新建mysql.sh
内容如下：
MYSQL_BASE=/usr/local/mysql
PATH=$MYSQL_BASE/bin:$PATH
执行source /etc/profile.d/mysql.sh

10、连接MySQL
首次使用执行mysql_secure_installation

执行完成后使用root连接
mysql -uroot -h127.0.0.1 -p
>password

```
## Nginx

1、下载
官网下载地址：`http://nginx.org/download/nginx-1.10.3.tar.gz`
2、安装
首先添加用户nginx，实现以之运行nginx服务进程：
```
# groupadd -r nginx
# useradd -r -g nginx nginx
```
接着开始编译和安装：
```
# ./configure \
  --prefix=/usr/local/nginx \
  --sbin-path=/usr/local/nginx/sbin/nginx \
  --conf-path=/etc/nginx/nginx.conf \
  --error-log-path=/var/log/nginx/error.log \
  --http-log-path=/var/log/nginx/access.log \
  --pid-path=/var/run/nginx/nginx.pid  \
  --lock-path=/var/lock/nginx.lock \
  --user=nginx \
  --group=nginx \
  --with-http_ssl_module \
  --with-http_flv_module \
  --with-http_stub_status_module \
  --with-http_gzip_static_module \
  --http-client-body-temp-path=/var/tmp/nginx/client/ \
  --http-proxy-temp-path=/var/tmp/nginx/proxy/ \
  --http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ \
  --http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \
  --http-scgi-temp-path=/var/tmp/nginx/scgi \
  --with-pcre
# make && make install
```
3、为nginx提供SysV init脚本:

新建文件/etc/rc.d/init.d/nginx，内容如下：
```
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15 
# description:  Nginx is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid
 
# Source function library.
. /etc/rc.d/init.d/functions
 
# Source networking configuration.
. /etc/sysconfig/network
 
# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0
 
nginx="/usr/sbin/nginx"
prog=$(basename $nginx)
 
NGINX_CONF_FILE="/etc/nginx/nginx.conf"
 
[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
 
lockfile=/var/lock/subsys/nginx
 
make_dirs() {
   # make required directories
   user=`nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   options=`$nginx -V 2>&1 | grep 'configure arguments:'`
   for opt in $options; do
       if [ `echo $opt | grep '.*-temp-path'` ]; then
           value=`echo $opt | cut -d "=" -f 2`
           if [ ! -d "$value" ]; then
               # echo "creating" $value
               mkdir -p $value && chown -R $user $value
           fi
       fi
   done
}
 
start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}
 
stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}
 
restart() {
    configtest || return $?
    stop
    sleep 1
    start
}
 
reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}
 
force_reload() {
    restart
}
 
configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}
 
rh_status() {
    status $prog
}
 
rh_status_q() {
    rh_status >/dev/null 2>&1
}
 
case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```
而后为此脚本赋予执行权限：
```
# chmod +x /etc/rc.d/init.d/nginx
```
添加至服务管理列表，并让其开机自动启动：
```
# chkconfig --add nginx
# chkconfig nginx on
```
而后就可以启动服务并测试了：
```
# service nginx start
```

## php
1、下载
http://php.net/downloads.php
这里选择5.6.31版本

2、编译安装
```
[root@lnmp php-5.6.31]#  ./configure --prefix=/usr/local/php --with-mysql=/usr/local/mysql --with-openssl --enable-fpm --enable-sockets --enable-sysvshm  --with-mysqli=/usr/local/mysql/bin/mysql_config --enable-mbstring --with-freetype-dir --with-jpeg-dir --with-png-dir --with-zlib-dir --with-libxml-dir=/usr --enable-xml  --with-mhash --with-mcrypt  --with-config-file-path=/etc --with-config-file-scan-dir=/etc/php.d --with-bz2 --with-curl 

[root@lnmp php-5.6.31]# make
[root@lnmp php-5.6.31]# make test  #此步骤可以不做，非常耗时间
[root@lnmp php-5.6.31]# make intall
```
3、为php提供配置文件：
```
[root@lnmp php-5.6.31]# cp php.ini-production /etc/php.ini
```
4、为php-fpm提供Sysv init脚本，并将其添加至服务列表：
```
[root@lnmp php-5.6.31]# cp sapi/fpm/init.d.php-fpm  /etc/init.d/php-fpm
[root@lnmp php-5.6.31]# chmod +x /etc/rc.d/init.d/php-fpm
[root@lnmp php-5.6.31]# chkconfig --add php-fpm
[root@lnmp php-5.6.31]# chkconfig php-fpm on
```
5、为php-fpm提供配置文件：
```
[root@lnmp php-5.6.31]# cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf 
```
6、编辑php-fpm的配置文件：
```
[root@lnmp php-5.6.31]# vim /usr/local/php/etc/php-fpm.conf
配置fpm的相关选项为你所需要的值，并启用pid文件（如下最后一行）：
pm.max_children = 150
pm.start_servers = 8
pm.min_spare_servers = 5
pm.max_spare_servers = 10
pid = /usr/local/php/var/run/php-fpm.pid 

# 可以设置运行用户和组
user = nginx
group = nginx
```
接下来就可以启动php-fpm了：
```
# service php-fpm start
```
使用如下命令来验正（如果此命令输出有中几个php-fpm进程就说明启动成功了）：
```
# ps aux | grep php-fpm
```

## 整合nginx和php5

1、编辑/etc/nginx/nginx.conf，启用如下选项：
```
    location ~ \.php$ {
          #root html;
          root   html; #可以自定义根路径
          fastcgi_pass 127.0.0.1:9000;
          fastcgi_index index.php;
          include /etc/nginx/fastcgi.conf;
       }
```

并在所支持的主页面格式中添加php格式的主页，类似如下：
```
location / {
            root   html; #可以自定义根路径
            index  index.php index.html index.htm;
        }
        
```
而后重新载入nginx的配置文件：
```
# service nginx reload
```
3、在/usr/local/nginx/html新建index.php的测试页面，测试php是否能正常工作：
```
# vim /usr/local/nginx/html/index.php
<?php
  phpinfo();
?>
```
接着就可以通过浏览器访问此测试页面了。

## 测试php和MySQL联通性
在/usr/local/nginx/html新建index2.php的测试页面，测试php是否能正常工作
```
# vim /usr/local/nginx/html/index2.php

<?php
$con=mysql_connect('127.0.0.1:3306','root','password');
    if($con)
    {
        die('ok');
    }
    else
    {
        die('Could not connect: ' . mysql_error());
    }
?>

```
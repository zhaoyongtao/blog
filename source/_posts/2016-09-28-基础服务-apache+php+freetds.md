---
title: Linux下Apache2.4+php5.5连接SQL Server2008
categories: 实践与总结
tag: web
abbrlink: 3d6aa6c1
date: 2016-09-28 00:00:00
---

> 此文档适用于CentOS6.7部署Apache+PHP连接SQL Server2008

<!-- more -->
软件版本：
- Apache: httpd-2.4.23
- PHP: php-5.5.12 
- Apr: apr apr-1.5.2
- Apr-util: apr-util-1.5.4
- libiconv: libiconv-1.14
- FreeTDS: freetds-1.00.15

> 先安装开发环境和依赖包

```
yum install gcc openssl openssl-devel pcre pcre-devel libxml2 libxml2-devel libcurl libcurl-devel libpng libpng-devel freetype-devel libxslt-devel libxslt
```


## Apache
> 由于Apache2.4.x依赖于apr1.4+和apr-util1.4+，所以在编译安装Apache2.4时要先编译安装这两个包

```
#安装apr
~]# ./configure  --prefix=/usr/local/apr
~]# make && make install


#安装apr-util
~]# ./configure  --prefix=/usr/local/apr-util  --with-apr=/usr/local/apr
~]# make && make install

#安装Apache(添加运行用户和组)
~]#  ./configure --prefix=/usr/local/apache24 --sysconfdir=/etc/httpd24  --enable-so --enable-ssl --enable-cgi --enable-rewrite --with-zlib --with-pcre --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util --enable-modules=most --enable-mpms-shared=all --with-mpm=prefork
~]# make && make install
```

### 添加Apache服务管理脚本

vim /etc/init.d/httpd
```
#!/bin/bash
#
# httpd        Startup script for the Apache HTTP Server
#
# chkconfig: - 85 15
# description: Apache is a World Wide Web server.  It is used to serve \
#        HTML files and CGI.
# processname: httpd
# config: /etc/httpd/conf/httpd.conf
# config: /etc/sysconfig/httpd
# pidfile: /var/run/httpd.pid
 
# Source function library.
. /etc/rc.d/init.d/functions
 
# Start httpd in the C locale by default.
HTTPD_LANG=${HTTPD_LANG-"C"}
 
# This will prevent initlog from swallowing up a pass-phrase prompt if
# mod_ssl needs a pass-phrase from the user.
INITLOG_ARGS=""
 
# Set HTTPD=/usr/sbin/httpd.worker in /etc/sysconfig/httpd to use a server
# with the thread-based "worker" MPM; BE WARNED that some modules may not
# work correctly with a thread-based MPM; notably PHP will refuse to start.
 
# Path to the apachectl script, server binary, and short-form for messages.
apachectl=/usr/local/apache24/bin/apachectl #根据自己安装路径
httpd=${HTTPD-/usr/local/apache24/bin/httpd}
prog=httpd
pidfile=${PIDFILE-/usr/local/apache24/logs/httpd.pid} #自己安装路径下找
lockfile=${LOCKFILE-/var/lock/subsys/httpd24}
RETVAL=0
STOP_TIMEOUT=${STOP_TIMEOUT-10}

start() {
        echo -n $"Starting $prog: "
        LANG=$HTTPD_LANG daemon --pidfile=${pidfile} $httpd $OPTIONS
        RETVAL=$?
        echo
        [ $RETVAL = 0 ] && touch ${lockfile}
        return $RETVAL
}

stop() {
  echo -n $"Stopping $prog: "
  killproc -p ${pidfile} -d 10 $httpd
  RETVAL=$?
  echo
  [ $RETVAL = 0 ] && rm -f ${lockfile} ${pidfile}
}
reload() {
    echo -n $"Reloading $prog: "
    if ! LANG=$HTTPD_LANG $httpd $OPTIONS -t >&/dev/null; then
        RETVAL=$?
        echo $"not reloading due to configuration syntax error"
        failure $"not reloading $httpd due to configuration syntax error"
    else
        killproc -p ${pidfile} $httpd -HUP
        RETVAL=$?
    fi
    echo
}
 
# See how we were called.
case "$1" in
  start)
  start
  ;;
  stop)
  stop
  ;;
  status)
        status -p ${pidfile} $httpd
  RETVAL=$?
  ;;
  restart)
  stop
  start
  ;;
  condrestart)
  if [ -f ${pidfile} ] ; then
    stop
    start
  fi
  ;;
  reload)
        reload
  ;;
  graceful|help|configtest|fullstatus)
  $apachectl $@
  RETVAL=$?
  ;;
  *)
  echo $"Usage: $prog {start|stop|restart|condrestart|reload|status|fullstatus|graceful|help|configtest}"
  exit 1
esac

exit $RETVAL
```

给/etc/init.d/httpd添加执行权限，并将httpd添加到开机自动启动
```
~]# chmod +x /etc/init.d/httpd
~]# chkconfig httpd --add
~]# chkconfig httpd on
```

## 编译安装PHP

> 后端PHP连接SQL Server时会出现乱码，解决办法：安装libiconv

### 编译安装libiconv

```
~]# ./configure --prefix=/usr/local/libiconv 
~]# make -j 4 && make install
```
### 编译安装PHP
```
~]# ./configure --prefix=/usr/local/php --with-curl --with-freetype-dir --with-gd --with-gettext --with-iconv-dir=/usr/local/libiconv --with-kerberos --with-libdir=lib64 --with-libxml-dir --with-openssl --with-pcre-regex --with-pdo-sqlite --with-pear --with-png-dir --with-xmlrpc --with-xsl --with-zlib --with-apxs2=/usr/local/apache24/bin/apxs --enable-libxml --enable-inline-optimization --enable-gd-native-ttf --enable-mbregex --enable-mbstring --enable-opcache --enable-pcntl --enable-shmop --enable-soap --enable-sockets --enable-sysvsem --enable-xml --enable-zip

~]# make -j 4 && make install -j 4
```

### 配置apache让它支持php
#### 修改apache的配置文件httpd.conf
```
~]# vim /usr/local/apache2.4/conf/httpd.conf
```

#### 然后查找文本，取消注释
```
LoadModule php5_module modules/libphp5.so #若已经取消进行下一步

#添加
AddType application/x-httpd-php .php
```
*** 注意，如果上面一条没配置好的话会导致，，访问http:localhost/*.php会直接下载，而不是打开 ***

#### 复制php配置文件
```
~]# cp php-5.6.3/php.ini-production /usr/local/php/lib/php.ini 
```

#### 重启httpd服务测试phpinfo页面

## Freetds编译安装
> Freetds作用是Linux下PHP支持连接SQL Server

```
#下载
~]# wget ftp://ftp.freetds.org/pub/freetds/stable/freetds-patched.tar.gz
~]# tar -zxvf freetds-patched.tar.gz
~]# cd freetds-1.00.15

#编译安装
~]# ./configure --prefix=/usr/local/freetds --with-tdsver=7.3 --enable-msdblib
~]# make && make install

配置
~]# cd ../
~]# echo "/usr/local/freetds/lib/" > /etc/ld.so.conf.d/freetds.conf
~]# ldconfig
```
输入以下命令，查看TDS Version是否和你的SQL Server版本一致
FreeTDS官网版本支持信息
http://www.freetds.org/userguide/choosingtdsprotocol.htm
```
~]# /usr/local/freetds/bin/tsql -C
```

### 测试数据库是否联通
```
~]# /usr/local/freetds/bin/tsql -H 数据库服务器IP  -p 端口号 -U 用户名 -P 密码
```
### 增加PHP的扩展mssql
```
~]# cd /opt/software/php-5.5.12/ext/mssql/

#linux下用phpize给PHP动态添加扩展
~]# /usr/local/php/bin/phpize
~]# ./configure --with-php-config=/usr/local/php/bin/php-config --with-mssql=/usr/local/freetds/
~]# make && make install
```
### 修改PHP配置文件
```
在php.ini配置文件中增加.so
~]# cd /usr/local/php/lib下的php.ini
#增加：
extension = "mssql.so"  
 
```
重启httpd服务即可

## 错误

### 有的页面正常打开，有的打开显示源码报错
> 刚开始以为是配置的环境问题，实际则是因为开发在windows环境下写php网页时，开头用的`<? ?>` 而不是Linux下的`<?php ?>`

解决办法：
```
第一种：
将php.ini里的
short_open_tag = Off
修改为on

第二种：
让开发将<? ?> 修改为Linux下可以识别的<?php ?>
```
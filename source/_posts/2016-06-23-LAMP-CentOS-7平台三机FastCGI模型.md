---
title: LAMP--CentOS-7平台三机FastCGI模型
categories: 实践与总结
tag: web
abbrlink: c3277995
date: 2016-06-23 13:27:02
---

# 需求 #
```
CentOS-7平台上搭建LAMP，其中php作为独立的服务工作；
(1) 三者分离于三台主机；
(2) 一个虚拟主机用于提供phpMyAdmin；另一个虚拟主机用于提供wordpress；
(3) 安装xcache，为php提供加速；
```
<!-- more -->
# 解析 #
+ 由于`CentOS-7`平台提供了以上各应用的`rpm`安装包，且均包含了各应用的新特性，所以各应用程序可以直接使用`yum`源安装，而无需通过编译源码来安装。
+ `xcache`为`EPEL`源中提供的应用程序，因此需要配置启用`EPEL`源。
+ 因为`php`要作为独立的服务单独运行于一台服务器上，因此不在是安装`php`，而是要安装`php-fpm`了。
+ `httpd`与`php`之间需要通过`FastCGI`协议来连接，`httpd`其实是作为作为反向代理来工作的，`httpd`中就需要加载`mod_proxy`和`mod_proxy_fcgi`模块了。

# 规划准备 #
+ 准备两台`CentOS-7`主机，均为最小化安装。为避免安全设置影响实验结果，将`iptables`和`SELinux`均设置为关闭状态。
+ `HostA`作为前端`Web`服务器，`IP`地址为: `172.18.71.201`。
+ `HostB`作为中间`App`服务器，`IP`地址为: `172.18.71.202`。
+ `HostC`作为后端`DB`服务器，`IP`地址为: `172.18.71.203`。
+ 需要配置`DNS`服务，或者修改主机`hosts`文件来实现名称解析。
+ 两台主机均需要配置好`yum`源，除发行版`base`源外还需要`EPEL`源。
* 配置名称解析和`yum`源不在本文介绍范围内，请参考其他资料。各程序安装包不提供，请自行下载。*

如图所示：
![CentOS-7-fastcgi](http://7xltax.com1.z0.glb.clouddn.com/538b2805-1156-4c78-8efa-ec3405ade4b7.png)

# 配置 #
配置时一般习惯按照从后端到前端的顺序来进行，便于测试验证前一步配置，思路上也比较清晰，容易梳理。所以首先配置后端`DB`服务器`HostC`，然后配置中间`App`服务器`HostB`，最后配置后端`Web`服务器`HostA`。

## HostC ##
安装`mariadb`
```
[root@localhost ~]# yum instal -y mariadb-server
```

在默认配置基础上增加一些配置，包括设置默认字符集、默认字符排序、默认存储引擎、独立表空间、不做域名反解。
```
[root@localhost ~]# vim /etc/my.cnf.d/client.cnf
[client]
default-character-set=utf8
[root@localhost ~]# vim /etc/my.cnf.d/mysql-clients.cnf
[mysql]
default-character-set=utf8
[root@localhost ~]# vim /etc/my.cnf.d/server.cnf
[mysqld]
character-set-server=utf8
collation-server=utf8_general_ci
default-storage-engine=InnoDB
innodb-file-per-table=TRUE
skip-name-resolve=TRUE
```

启动`mariadb`
```
[root@localhost ~]# systemctl start mariadb
```

初始化安全设置，主要这是设置`root`用户口令，删除匿名用户，不允许`root`远程登录，删除`test`数据库和访问`test`数据库的相关权限设置，重载权限表。
** 注：这里如果设置允许`root`远程登录，后面也还是要再次显式授权才可以。**
```
[root@localhost ~]# mysql_secure_installation
Set root password? [Y/n] y
New password:
Re-enter new password:
Remove anonymous users? [Y/n] y
 ... Success!
 Disallow root login remotely? [Y/n] y
  ... skipping.
  Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!
 Reload privilege tables now? [Y/n] y
 ... Success!
```

登录`mariadb`，做以下设置。
```
[root@localhost ~]# mysql -u root -p
# 授权root可从172.18.71.0/24网段内的主机登录操作所有数据库。
MariaDB [(none)]> grant all privileges on *.* to 'root'@'172.18.71.%' identified by 'magedu';
# 给wordpress创建数据wpdb。
MariaDB [(none)]> create database wpdb;
# 给wordpress创建用户wp。
MariaDB [(none)]> create user 'wp'@'172.18.71.%' identified by 'magedu';
# 授权wp可从172.18.71.0/24网段内的主机登录操作wpdb数据库。
MariaDB [(none)]> grant all privileges on wpdb.* to 'wp'@'172.18.71.%' identified by 'magedu';
# 重载权限表
MariaDB [(none)]> flush privileges;
# 退出
MariaDB [(none)]> \q
```

这样`HostC`就配置完成了，再来配置`HostB`。

## HostB ##
正式配置`HostB`之前，应首先验证以下`HostC`的配置，即测试一下能否从`HostB`上连接`HostC`的数据库。
```
# 安装mariadb的客户端
[root@localhost ~]# yum install -y mariadb
# 测试连接HostB数据库
[root@localhost ~]# mysql -u root -h 172.18.71.203 -p
```

安装`php-fpm`应用
```
[root@localhost ~]# yum install -y php-fpm php-mysql php-mbstring
```

修改`php-fpm`的连接池配置文件
```
[root@localhost ~]# vim /etc/php-fpm.d/www.conf
# 监听在本机的外部可访问地址上
listen = 172.18.71.202:9000
# 仅允许前端Web服务器连接
listen.allowed_clients = 172.18.71.201
```

创建`session`存放目录，并修改其权限。
```
[root@localhost ~]# mkdir /var/lib/php/session
[root@localhost ~]# chown -R apache:apache /var/lib/php/session
```

部署`php`程序代码`phpMyAdmin`和`wordpress`。
**  注：由于`php-fpm`是独立工作的服务，对于访问`php`页面的请求，前端的`httpd`仅作为反向代理将请求转发至`php-fpm`服务器上，由`php-fpm`找到对应的`php`程序代码文件进行解释执行，然后将结果返回给前端。所以需要将`php`程序代码放在`php-fpm`服务器上。**

建立`php`程序相应的存放目录。
```
[root@localhost ~]# mkdir -pv /htdocs/www.twoyang.{com,net}
mkdir: created directory ‘/htdocs’
mkdir: created directory ‘/htdocs/www.twoyang.com’
mkdir: created directory ‘/htdocs/www.twoyang.net’
```

将`phpMyAdmin`安装包放置在`/htdocs/www.twoyang.com`的目录下，并修改配置文件。
```bash
[root@localhost ~]# cd /htdocs/www.twoyang.com
[root@localhost www.twoyang.com]# unzip phpMyAdmin-4.4.14.1-all-languages.zip
[root@localhost www.twoyang.com]# ln -sv phpMyAdmin-4.4.14.1-all-languages pma
‘pma’ -> ‘phpMyAdmin-4.4.14.1-all-languages’
[root@localhost www.twoyang.com]# cd pma
[root@localhost pma]# cp config.sample.inc.php config.inc.php
[root@localhost pma]# openssl rand -base64 20
0fkgcykLDfyRgG71FOjE9W6LKa8=
[root@localhost pma]# vim config.inc.php
$cfg['blowfish_secret'] = '0fkgcykLDfyRgG71FOjE9W6LKa8'; /* YOU MUST FILL IN THIS FOR COOKIE AUTH! */
$cfg['Servers'][$i]['host'] = '172.18.71.203';
```

将`wordpress`安装包放置在`/htdocs/www.twoyang.com`的目录下，并修改配置文件。
```
[root@localhost ~]# cd /htdocs/www.twoyang.net
[root@localhost www.twoyang.net]# unzip wordpress-4.3.1-zh_CN.zip
[root@localhost www.twoyang.net]# ln -sv wordpress wp
‘wp’ -> ‘wordpress’
[root@localhost www.twoyang.net]# cd wp
[root@localhost wp]# cp wp-config-sample.php wp-config.php
[root@localhost wp]# vim wp-config.php

/** WordPress数据库的名称 */
define('DB_NAME', 'wpdb');

/** MySQL数据库用户名 */
define('DB_USER', 'wp');

/** MySQL数据库密码 */
define('DB_PASSWORD', 'magedu');

/** MySQL主机 */
define('DB_HOST', '172.18.71.203');
```

启动`php-fpm`服务
```
[root@localhost ~]# systemctl start php-fpm
```

查看`php-fpm`服务是否监听在指定套接字上
```
[root@localhost ~]# ss -tnl | grep 9000
LISTEN     0      128    172.18.71.202:9000                     *:*
```

## HostA ##
安装`httpd`
```
[root@localhost ~]# yum install -y httpd
```

修改主配置文件
```
[root@localhost ~]# vim /etc/httpd/conf/httpd.conf
# 设置服务器名，其实这项不设置也可以，但是启动服务时会有警告。
ServerName www.twoyang.net
# 由于准备使用虚拟主机，需要把MainServer的DocumentRoot关掉。
#DocumentRoot "/var/www/html"
# 另外需要增加php类型的主页和MIME
<IfModule dir_module>
    DirectoryIndex index.php index.html
</IfModule>
<IfModule mime_module>
    AddType application/x-httpd-php .php
</IfModule>
```

移除欢迎页面配置文件
```
[root@localhost ~]# mv /etc/httpd/conf.d/welcome.conf{,.bak}
```

新建虚拟主机配置文件`/etc/httpd/conf.d/vhosts.conf`
```
[root@localhost ~]# vim /etc/httpd/conf.d/vhosts.conf
<VirtualHost 172.18.71.201:80>
    ServerName www.twoyang.com
    DocumentRoot "/htdocs/www.twoyang.com/pma"
    <Directory "/htdocs/www.twoyang.com/pma">
        OPTIONS FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    # 由于正向代理和反向代理不能同时存在，这里关闭正向代理。
    ProxyRequests Off
    # 将直接访问/的url请求，通过fcgi协议指向后端php-fpm服务器对应目录的index.php。
    ProxyPassMatch ^/$ fcgi://172.18.71.202:9000/htdocs/www.twoyang.com/pma/index.php
    # 将访问以.php结尾的url请求，通过fcgi协议指向后端php-fpm服务器对应目录的对应文件。
    ProxyPassMatch ^/(.*\.php)$ fcgi://172.18.71.202:9000/htdocs/www.twoyang.com/pma/$1
</VirtualHost>

<VirtualHost 172.18.71.201:80>
    ServerName www.twoyang.net
    DocumentRoot "/htdocs/www.twoyang.net/wp"
    <Directory "/htdocs/www.twoyang.net/wp">
        OPTIONS FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    ProxyRequests Off
    ProxyPassMatch ^/$ fcgi://172.18.71.202:9000/htdocs/www.twoyang.net/wp/index.php
    ProxyPassMatch ^/(.*\.php)$ fcgi://172.18.71.202:9000/htdocs/www.twoyang.net/wp/$1
</VirtualHost>
```

建立虚拟主机`DocumentRoot`对应的目录。
```
[root@localhost ~]# mkdir -pv /htdocs/www.twoyang.{com,net}
mkdir: created directory ‘/htdocs’
mkdir: created directory ‘/htdocs/www.twoyang.com/’
mkdir: created directory ‘/htdocs/www.twoyang.net/’
```

将`phpMyAdmin`安装包放置在虚拟主机`www.twoyang.com`的`DocumentRoot`目录下。
```
[root@localhost ~]# cd /htdocs/www.twoyang.com
[root@localhost www.twoyang.com]# unzip phpMyAdmin-4.4.14.1-all-languages.zip
[root@localhost www.twoyang.com]# ln -sv phpMyAdmin-4.4.14.1-all-languages pma
‘pma’ -> ‘phpMyAdmin-4.4.14.1-all-languages’
```

将`wordpress`安装包放置在虚拟主机`www.twoyang.net`的`DocumentRoot`目录下。
```
[root@localhost ~]# cd /htdocs/www.twoyang.net
[root@localhost www.twoyang.net]# unzip wordpress-4.3.1-zh_CN.zip
[root@localhost www.twoyang.net]# ln -sv wordpress wp
‘wp’ -> ‘wordpress’
```

** 注: 这里将`phpMyAdmin`和`wordpress`安装包在`HostA`上重新部署一次，是因为`httpd`仅将有关`php`的请求转发至`php-fpm`服务器去处理。而事实上一个完整页面当中还存在图片等静态内容，这些内容是不会转发的，还是要在本地的`DocumentRoot`目录中去找。**

这么做以后，在`HostA`上的虚拟机`DocumentRoot`目录就会存在无用的`php`程序代码文件占用磁盘，可以将其删除。
```
[root@localhost ~]# cd /htdocs/www.twoyang.com/pma
[root@localhost pma]# find . -name "*.php" | xargs rm -rf
[root@localhost ~]# cd /htdocs/www.twoyang.com/wp
[root@localhost wp]# find . -name "*.php" | xargs rm -rf
```

你可以跳过在`HostA`上部署`phpMyAdmin`和`wordpress`安装包的步骤来验证以上结论，可预期的结果是一些页面当中的图片链接无法显示。

启动`httpd`服务，修改一下`/htdocs`目录的属主和属组，以免出现权限问题。
```
[root@localhost ~]# chown -R apache:apache /htdocs
```

这时便可以启动`httpd`服务，测试访问`phpMyAdmin`和`wordpress`了。
```
[root@localhost ~]# systemctl start httpd
```

## xcache ##
但是这个时候还是没有安装`xcache`的，可以用`ab`测试一下并发访问时服务器响应情况。
```
[root@localhost ~]# ab -n 1000 -c 100 http://www.twoyang.com/
```

这时在`HostB`上安装`xcache`和`php-cli`（后者提供`php`命令行工具，可查看当前加载的`php`模块有哪些）。
```
[root@localhost ~]# yum install -y php-xcache php-cli
```

修改`xcache`配置文件，打开`xcache.cacher`开关。
```
[root@localhost ~]# vim /etc/php.d/xcache.ini
xcache.cacher = On
```

重启服务
**注意因为此时`php`是作为独立服务来工作的，因此需要重启`HostB`上的`php-fpm`服务。**
```
[root@localhost ~]# systemctl restart php-fpm
[root@localhost ~]# php -m | grep XCache
XCache
XCache Cacher
XCache Coverager
XCache Optimizer
XCache
XCache Cacher
XCache Coverager
XCache Optimizer
```

再次使用`ab`测试
```
[root@localhost ~]# ab -n 1000 -c 100 http://www.twoyang.com/
```

> 注：本文转载自:[twoyang.net](twoyang.net)

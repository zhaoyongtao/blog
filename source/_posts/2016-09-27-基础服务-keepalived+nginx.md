---
title: KeepAlived高可用Nginx
categories: 实践与总结
tag: 高可用
abbrlink: bf9d2e83
date: 2016-09-27 00:00:00
---

# 主

## Nginx
> /etc/nginx/conf.d/default.conf
> 同时修改nginx.conf里监听的端口

<!-- more -->
```
~]# vim /etc/nginx/conf.d/default.conf
#
# The default server
#
 upstream backend {
          server 10.207.0.88 weight=1;
          server 10.207.0.89 weight=1;
    }

    server {
        listen       80;
        server_name  localhost;
        # 当nginx将php代码送至后端RS处理时请求头中的Host值会是backend.
        # php代码在RS上处理时,其内部代码会去请求图片/层叠样式表等静态资源以填充页面.
        # 而php代码去请求后端资源时使用的是如http://backend/xxxx.php这样的url,自然是取不到的.
        # 所以我们要在nginx向后代理遇到Host为backend时,将其转换为127.0.0.1.
        set $my_host $http_host;
        if ($http_host = "backend") {
            set $my_host "127.0.0.1";
        }

        location / {
              proxy_pass     http://backend;
              proxy_redirect off;
              proxy_set_header  Host  $my_host;
         }
    }

server {
    listen       8080 default_server;
    server_name  localhost;
    root         /usr/share/nginx/html;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }

}

```
## keepalived

```
! Configuration File for keepalived
global_defs {

   notification_email {
     root@localhost  # 邮件给本机root用户
   }
   notification_email_from admin@nginx-124.com
   smtp_server 127.0.0.1  # 使用本机作为smtp服务器
   smtp_connect_timeout 30
   router_id 8a028eb8  # 标识主机,可以使用主机名.
   vrrp_mcast_group4 224.0.71.18  # 多播地址,用于发送心跳信号.尽量让集群内的主机处于同一个独立的多播地址.
}

# nginx进程监控脚本.如果进程不在,降低自身权重,使从节点主机优先级高于自身,将VRRP漂移至从节点主机.
vrrp_script chk_nginx {
    script "killall -0 nginx"
    interval 2
    weight -8
}
vrrp_instance VI_1 {
    state MASTER  # vrrp实例VI_1中作为主
    interface eth0
    virtual_router_id 71 # 0-255范围内的数字,用于区分vrrp实例,所以两个实例不能一致.
    priority 100    # MASTER的优先级要高一些
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass uWjblY61  # 简单字符认证, 8位任意字符串.
    }
    virtual_ipaddress {
        10.207.0.164/24 brd 10.207.0.164 dev eth0 label eth0:0  # VIP1
    }

    # 在此处调用nginx进程监控脚本
    track_script {
        chk_nginx
    }

    # 关闭争用. 争用是指当高优先级节点上线会立即争夺成为MASTER
    # 而不管其它节点是否正在给用户提供服务.
    #nopreempt

    # 开启争用时,会延迟一段时间才开始.
    #preempt_delay 300
}
vrrp_instance VI_2 {
    state BACKUP    # vrrp实例VI_2中作为备
    interface eth0
    virtual_router_id 171
    priority 95     # MASTER的优先级要高一些
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass uWjblY62
    }
    virtual_ipaddress {
        10.207.0.164/24 brd 10.207.0.164 dev eth0 label eth0:0  # VIP1
    }

    # 在此处调用nginx进程监控脚本
    track_script {
        chk_nginx
    }
}


```

# 备
## Nginx
> /etc/nginx/conf.d/default.conf
> 同时修改nginx.conf里监听的端口

```

upstream backend {
          server 10.207.0.88 weight=1;
          server 10.207.0.89 weight=1;
    }

    server {
        listen       80;
        server_name  localhost;
        # 当nginx将php代码送至后端RS处理时请求头中的Host值会是backend.
        # php代码在RS上处理时,其内部代码会去请求图片/层叠样式表等静态资源以填充页面.
        # 而php代码去请求资源时使用的是如http://backend/xxxx.php这样的url,自然是取不到的.
        # 所以我们要在nginx向后代理遇到Host为backend时,将其转换为127.0.0.1.
       set $my_host $http_host;
        if ($http_host = "backend") {
            set $my_host "127.0.0.1";
        }

        location / {
              proxy_pass     http://backend;
              proxy_redirect off;
              proxy_set_header  Host  $my_host;
         }
    }


server {
     listen       8080 default_server;
     server_name  localhost;
     root         /usr/share/nginx/html;
     
     # Load configuration files for the default server block.
     include /etc/nginx/default.d/*.conf;
     
     location / {
     }
     
     error_page 404 /404.html;
         location = /40x.html {
     }   
     
     error_page 500 502 503 504 /50x.html;
        location = /50x.html { 
     }
 
}


```

## keepalived

```
! Configuration File for keepalived
global_defs {

   notification_email {
     root@localhost  # 邮件给本机root用户
   }
   notification_email_from admin@nginx-133.com
   smtp_server 127.0.0.1  # 使用本机作为smtp服务器
   smtp_connect_timeout 30
   router_id 8a028eb8  # 标识主机,可以使用主机名.
   vrrp_mcast_group4 224.0.71.18  # 多播地址,用于发送心跳信号.尽量让集群内的主机处于同一个独立的多播地址.
}

# nginx进程监控脚本.如果进程不在,降低自身权重,使从节点主机优先级高于自身,将VRRP漂移至从节点主机.
vrrp_script chk_nginx {
    script "killall -0 nginx"
    interval 2
    weight -8
}
vrrp_instance VI_1 {
    state BACKUP  # vrrp实例VI_1中HostA作为备
    interface eth0
    virtual_router_id 71 # 0-255范围内的数字,用于区分vrrp实例,所以两个实例不能一致.
    priority 95    # MASTER的优先级要高一些
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass uWjblY61  # 简单字符认证, 8位任意字符串.
    }
    virtual_ipaddress {
        10.207.0.164/24 brd 10.207.0.164 dev eth0 label eth0:0  # VIP1
    }

    # 在此处调用nginx进程监控脚本
    track_script {
        chk_nginx
    }

    # 关闭争用. 争用是指当高优先级节点上线会立即争夺成为MASTER
    # 而不管其它节点是否正在给用户提供服务.
    #nopreempt

    # 开启争用时,会延迟一段时间才开始.
    #preempt_delay 300
}
vrrp_instance VI_2 {
    state MASTER    # vrrp实例VI_2中HostA作为主
    interface eth0
    virtual_router_id 171
    priority 100     # BACKUP的优先级要低一些
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass uWjblY62
    }
    virtual_ipaddress {
        10.207.0.164/24 brd 10.207.0.164 dev eth0 label eth0:0  # VIP1
    }

    # 在此处调用nginx进程监控脚本
    track_script {
        chk_nginx
    }
}

```
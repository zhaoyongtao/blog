---
title: samba服务器搭建
categories: 实践与总结
tag: 实践与总结
abbrlink: eb0f8299
date: 2017-03-27 00:00:00
---
### 安装
```
~]# yum -y install samba samba-common samba-client 
```

### samba服务器访问过慢的问题
```
具体表现：
	samba服务正常启动，win下可以访问，打开时要等待很久
解决办法：
	1、查看/etc/sysconfig/network中hostname
	2、编辑/etc/hosts文件，添加hostname的值到127.0.0.1
	3、重启服务或服务器
```
<!-- more -->
### 配置

#### samba配置

```
~]# vim /etc/samba/smb.conf
[global]
	workgroup = WORKGROUP
	server string = Samba Server Version %v
	log file = /var/log/samba/log.%m
	max log size = 50
	security = user
	passdb backend = tdbsam
	load printers = yes
	hosts allow = 192.168.1.
	cups options = raw

[公共]
    comment = This is a public directory
    path = /data/public
    public = yes
    writable = yes
    create mask = 0755
    directory mask = 0755

[开发组]
    path = /data/kaifazu
    valid users = @kaifazu
    group = kaifazu
	read list = @kaifazu
    write list = @kaifazu
	create mode = 0664
	directory mode = 0775

[测试组]
    path = /data/ceshizu
    valid users = @ceshizu
    group = @ceshizu
    write list = @ceshizu
	create mode = 0664
	directory mode = 0775

```
#### 组配置
```
kaifazu:x:531:zhangsan,lisi
ceshizu:x:532:wangwu

```

### samba服务器磁盘检测脚本
```
#!/bin/bash
function fun_chksize() 
{
    # $1 file_name
    # $2 file_size
    # $3 max_size
    # $4 mail_addr
    # switch_file_size=file_size/1024  将K转换为M，方便阅读
    if [ $2 -ge $3 ];then
        switch_file_size=$(($2/1024))
        echo -n -e "samba服务器 ${1} 空间已使用${switch_file_size}M\n" | mail -s "samba服务器空间使用告警"  $4
    fi
}
#定义一些变量
smb_dir=/data
ip=`ifconfig | grep Bcast | awk '{print $2}' | awk -F: '{print $2}'` #IP地址
admin_mail_addr="123456@qq.com" #管理员邮箱
total_max_size=1024000 #总空间最大可用1G，单位K
total_use_size=`du -s $smb_dir | awk '{print $1}'` #总使用大小，单位K
max_size=10240 #子目录默认最大可用10M  
public_max_size=10240 #public最大可用10M
#检测总目录大小
fun_chksize $smb_dir $total_use_size $total_max_size $admin_mail_addr
#检测子目录大小
for list1 in ${smb_dir}/*; do
    file_list=`basename $list1` #列表
    cd $smb_dir
    file_size=`du -s $file_list | awk '{print $1}'` #大小，单位K
    switch_file_size=$(($file_size/1024)) #将K转换为M
    file_name=`du -s $file_list | awk '{print $2}'` #文件名或目录名
    case $file_name in
        public)
	    fun_chksize $file_name $file_size $public_max_size $admin_mail_addr
	    ;;
	    kaifazu)
	    fun_chksize $file_name $file_size $max_size $admin_mail_addr
	    ;;
        *)
	    fun_chksize $file_name $file_size $max_size $admin_mail_addr
	    ;;
    esac
done
```


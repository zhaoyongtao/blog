---
title: KVM虚拟Mac OS X Sierra
categories: 实践与总结
tag: 实践与总结
abbrlink: 638d5663
date: 2017-04-12 00:00:00
---
# KVM虚拟Mac OS X Sierra
 
> 大致可行的方法有两种：

```
第一种，重新编译内核、编译qemu、编译kvm、kvm-mod，加上OS X的支持。
详情：http://www.tuicool.com/articles/JBzANrU
在使用此种方法编译kvm时，报错，无法安装，文件内容都与作者标识的不一致。才疏学浅，未能成功。。。

第二种,GitHub上有OSX-KVM项目，相对来说较为简单，本次采用此种方法。
具体请参考：https://github.com/kholia/OSX-KVM
```
<!-- more -->
本文所需文件：
```
在Mac下制作的Install_macOS_Sierra_OS_X_10.12.iso
引导文件enoch_rev2839_boot
创建的磁盘mac_hdd.img
qemu配置文件OSX_KVM.xml

```
链接：http://pan.baidu.com/s/1qYbe12W 密码：6znh

## 环境准备
```
物理机系统说明：

作者在以下系统中测试过:
Ubuntu 15.10 running on i5-6500 CPU.
Ubuntu 16.10 running on i7-3960X CPU.
Fedora 24 running on i5-6500 + i7-6600U CPU.

QEMU版本：2.4.1, 2.5, 2.6.1, and 2.8.

AMD CPU有问题。AMD FX-8350 可以工作，但是Phenom II X3 720不工作
需要开启  Intel VT-x/AMD-v虚拟化技术

本次使用环境：
Ubuntu Server 16.04 LTS  
QEMU:2.5.0
```

## 安装qemu和virt-manager
```
sudo apt-get install qemu uml-utilities virt-manager
```

## 安装桥接网络管理管理工具
```
sudo apt-get install bridge-utils
```

## 配置桥接网络

```
root@fin75:~# vim /etc/network/interface
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

auto br0
iface br0 inet static
    address 172.16.0.75
    network 172.16.0.0
    netmask 255.255.255.0
    broadcast 172.16.0.255
    gateway 172.16.0.1
    dns-nameservers 223.5.5.5
    bridge_ports eno1
    bridge_stp off
    bridge_fd 0
    bridge_maxwait 0
	post-up ip link set br0 address aa:2b:3c:4d:5e:6f

root@fin75:~# reboot
```
注意：virbr0是安装kvm时系统建立的，为NAT网络专用，跟我们要使用的桥接完全不同
> 按照官方文档直接重启服务会失败，重启物理机就可以了。

参考：https://help.ubuntu.com/community/KVM/Networking


## 创建磁盘文件
```
qemu-img create -f qcow2 /u01/mac_hdd.img 200G
```

> 安装方法可以使用boot-macOS.sh/boot.sh，或者使用macOS-libvirt.xml
> 本次使用libvirt文件的方式

## 修改libvirt文件
```
#只需修改这几处即可。

#引导文件位置
<kernel>/u01/boot/enoch_rev2839_boot</kernel> 

#磁盘文件位置
<source file='/u01/mac_hdd.img'/> 

#ISO镜像位置
<qemu:arg value='id=MacDVD,if=none,snapshot=on,file=/opt/Install_macOS_Sierra_OS_X_10.12.iso'/>

#如果有多台OS X系统，还需修改

<uuid>c757b31e-115f-4d1a-b574-0ae7b3cc8a58</uuid>

#kvm中合理的Mac地址为52:54:00开头
<mac address='52:54:00:3d:f8:25'/> 

#可在shell中执行以下命令获取合理的Mac地址：
MACADDR="52:54:00:$(dd if=/dev/urandom bs=512 count=1 2>/dev/null | md5sum | sed 's/^\(..\)\(..\)\(..\).*$/\1:\2:\3/')"; echo $MACADDR

#vnc中的port需更改为与第一个虚拟机不同
<graphics type='vnc' port='5900' autoport='no' listen='127.0.0.1' keymap='en-us'>


```

## 重新定义libvirt文件
```
virsh define /somepath/OSX-KVM/macOS-libvirt.xml 

#定义后，在/etc/libvirt/qemu/目录下会有macOS-libvirt.xml文件，以后修改后只需重新定义此文件即可
```

## 安装OS X

在virt-manager中启动OSX

步骤：
![](https://cloud.githubusercontent.com/assets/731252/17645877/5136b1ac-61b2-11e6-8d90-29f5cc11ae01.png)

选择磁盘工具
![](https://cloud.githubusercontent.com/assets/731252/17645881/513b6918-61b2-11e6-91f2-026d953cbe0b.png)

格式化KVM磁盘
![](https://cloud.githubusercontent.com/assets/731252/17645878/51373d48-61b2-11e6-8740-69c86bf92d31.png)

![](https://cloud.githubusercontent.com/assets/731252/17645879/513ae704-61b2-11e6-9a54-109c37132783.png)

退出磁盘工具，打开终端
![](https://cloud.githubusercontent.com/assets/731252/17645876/5136ad6a-61b2-11e6-84cd-cb7851119292.png)
输入命令，拷贝安装文件：
```
cp -av /Extra /Volumes/KVMDisk
```

退出终端，启动安装即可！


#错误合集
## virt-manager启动虚拟机，打不开安装界面，显示boot，无限重启
解决办法：开启ignore_msrs
```
echo 1 > /sys/module/kvm/parameters/ignore_msrs
```

开机执行
```
vim /etc/rc.local
echo 1 > /sys/module/kvm/parameters/ignore_msrs
```

## 启动域时出错: 
internal error: process exited while connecting to monitor: 2017-04-05 T06\:\25:53.648209Z qemu-system-x86_64: -drive id=MacDVD,if=none,snapshot=on,file=/opt/Install_macOS_Sierra_OS_X_10.12.iso: Could not open '/opt/Install_macOS_Sierra_OS_X_10.12.iso': Permission denied

#解决办法
kvm需要selinux装载安全模块，默认的Ubuntu server没有安装selinux

```
sudo apt-get install selinux
#设置selinux=permissive
reboot
```

## virt-manager显示乱码

```
sudo apt install font-manager 
sudo apt install fonts-arphic-ukai 
sudo apt install ttf-wqy-zenhei xfonts-wqy ttf-wqy-microhei 
sudo apt install fonts-cwtex-fs 
sudo apt install ttf-hanazono 
sudo apt install ttf-mscorefonts-installer 
```




# Ubuntu虚拟Windows

需要将显示协议由`Spice`服务器更改为`VNC`服务器，并且将键映射改为：`en-us`，不然会出现键盘无法使用的情况

需要将显卡由`QXL`改为`Cirrus`，不然kvm安装windows系统时会卡在Starting Windows画面
http://serverfault.com/questions/776406/windows-7-setup-hangs-at-starting-windows-using-proxmox-4-2

需要将NIC网卡由`rtl8139`更改为`e1000`，不然会出现断网的情况


# 参考
https://github.com/kholia/OSX-KVM
http://www.tuicool.com/articles/JBzANrU
http://www.jianshu.com/p/e95c458d78bd
https://blog.ostanin.org/2014/02/11/playing-with-mac-os-x-on-kvm/
http://www.cnblogs.com/huntaiji/p/3918941.html
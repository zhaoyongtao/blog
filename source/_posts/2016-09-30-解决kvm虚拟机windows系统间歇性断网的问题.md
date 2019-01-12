---
title: 解决kvm虚拟windows系统间歇性网络中断的问题
categories: 实践与总结
tag: kvm
abbrlink: d6278d54
date: 2016-09-28 00:00:00
---


> 现象
- 突然之间，网络完全中断，无法从网络访问虚拟机
- 用virt-manager或者console登录虚拟机，发现虚拟机还在正常工作，没有崩溃
- 使用 service network restart重启物理机网络服务，可以立即恢复网络
- 网络负载越大，故障出现的频率越高。轻网络负载的机器，没有出现故障

<!-- more -->
## 解决
搜索了一下，发现ubuntu和centos都会出现这样的问题：

https://bugs.launchpad.net/ubuntu/+source/libvirt/+bug/997978
http://bugs.centos.org/view.php?id=5526

几个解决办法：

第1种：使用 e1000替代原有的windows网卡

第2种：使用 vhost_net 模块
```
echo vhost_net > /etc/modules
modprobe vhost_net
```
然后重新启动虚拟机，libvirtd就会自动使用 vhost_net

## 原因分析
在kvm虚拟机里，默认windows系统虚拟的网卡是RTL8139C的网卡，此网卡在网络重负载下易发生崩溃现象。
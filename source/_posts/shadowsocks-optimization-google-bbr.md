---
title: shadowsocks之采用google bbr算法优化
date: 2017-07-30 16:58:42
categories: 实用工具
tags: [shadowsocks, 翻墙]
---
转自[https://www.vultr.com/docs/how-to-deploy-google-bbr-on-centos-7](https://www.vultr.com/docs/how-to-deploy-google-bbr-on-centos-7)  
翻译by rou liu

上一篇介绍完了部署shadowsocks 大多数情况下已经满足需求，但是如果想看youtube高清视频
或者在线直播，那就要用到接下来的方法，感谢goolgle。

## 什么是bbr 
BBR（Bottleneck Bandwidth and RTT）是一个新的拥塞控制算法，由谷歌贡献的Linux内核的TCP协议栈。有了BBR，Linux服务器可以得到显著提高连接的吞吐量和减少延迟。此外，它很容易部署，因为这种算法只需要在发送方更新，而不需要在网络或在接收端部署BBR。

在本文中，我将向你展示如何在Vultr CentOS的7 KVM服务器实例上部署BBR。

## 前提条件
- 一个Vultr CentOS的7 x64的服务器实例。
- 一个sudo的用户。

## 升级bbr
### 使用ELRepo RPM库升级内核

为了使用BBR，你需要将你的CentOS 7机器的内核升级到4.9.0。你可以很容易得通过ELRepo RPM库中进行升级。  
在升级之前，你可以看看当前的linux内核版本:  
```bash
uname -r
```  

这个命令会输出类似的一个字符串:   
```bash
3.10.0-514.2.2.el7.x86_64
``` 
  
正如你看到的，目前linux的内核版本是3.10.0。

安装ELRepo repo：  
```bash
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm 
```

使用ELRepo安装4.9.0内核：  
```bash
sudo yum --enablerepo=elrepo-kernel install kernel-ml -y
```  
确认结果：  
```bash
rpm -qa | grep kernel  
```  
如果安装成功，你应该可以看到kernel-ml-4.9.0-1.el7.elrepo.x86_64在如下输出列表中：  
```bash
kernel-ml-4.9.0-1.el7.elrepo.x86_64
kernel-3.10.0-514.el7.x86_64
kernel-tools-libs-3.10.0-514.2.2.el7.x86_64
kernel-tools-3.10.0-514.2.2.el7.x86_64
kernel-3.10.0-514.2.2.el7.x86_64
```
现在，你需要通过设置默认的grub2的启动项来启用4.9.0内核。

显示在GRUB2菜单中的所有条目如下：  
```bash
sudo egrep ^menuentry /etc/grub2.cfg | cut -f 2 -d \'
```

结果应类似于： 
```bash
CentOS Linux 7 Rescue a0cbf86a6ef1416a8812657bb4f2b860 (4.9.0-1.el7.elrepo.x86_64)  
CentOS Linux (4.9.0-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (3.10.0-514.2.2.el7.x86_64) 7 (Core)
CentOS Linux (3.10.0-514.el7.x86_64) 7 (Core)
CentOS Linux (0-rescue-bf94f46c6bd04792a6a42c91bae645f7) 7 (Core)
```  
<!--more--> 

从0开始计数，4.9.0内核位于第二行，所以设置默认启动项为1：  
```bash
sudo grub2-set-default 1
```  

重新启动系统：  
```bash
sudo shutdown -r now
```   

当服务器重新上线，请重新登录并重新运行uname命令，以确认您使用的是正确的内核：  
```bash
uname -r
``` 

您应该看到如下结果：  
```bash
4.9.0-1.el7.elrepo.x86_64
```

### 启用BBR

为了使BBR算法，你需要修改sysctl配置： 
```bash
echo 'net.core.default_qdisc=fq' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_congestion_control=bbr' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
``` 

现在，你可以使用下面的命令来确认BBR是否启用：
```bash
sudo sysctl net.ipv4.tcp_available_congestion_control
``` 

输出应该类似于:  
```bash
net.ipv4.tcp_available_congestion_control = bbr cubic reno
``` 

接下来，验证有：  
```bash
sudo sysctl -n net.ipv4.tcp_congestion_control
``` 

输出应该是： 

```bash
bbr
``` 

最后，检查内核模块的加载位置:  
```bash
lsmod | grep bbr 
``` 

输出将类似于：  
```bash
tcp_bbr                16384  0
``` 

## (可选)：测试网络性能增强

为了测试BBR的网络性能的提升，你可以在Web服务器目录中的创建一个文件下载，然后从你的台式机上的浏览器测试下载速度。
```bash
sudo yum install httpd -y
sudo systemctl start httpd.service
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --reload
cd /var/www/html
sudo dd if=/dev/zero of=500mb.zip bs=1024k count=500
``` 
最后，请从您的桌面电脑上的网络浏览器访问网址http://[your-server-IP]/500mb.zip，然后评估的下载速度。

就这样。谢谢您的阅读。
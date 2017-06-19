---
title: Centos7下安装shadowsocks
date: 2017-06-03 09:38:40
categories: 实用工具
tags: [centos7,shadowsocks]
---

## 1. 安装pip
pip是 python 的包管理工具。在本文中将使用 python 版本的 shadowsocks，此版本的 shadowsocks 已发布到 pip 上，因此我们需要通过 pip 命令来安装。
```shell
yum -y update
yum install -y python-setuptools && easy_install pip
```
## 2.安装 shadowsocks
```shell
pip install shadowsocks
yum clean all
```
## 3.配置 shadowsocks
安装完成后，需要创建配置文件/etc/shadowsocks/shadowsocks.json
vi /etc/shadowsocks/shadowsocks.json
<!--more-->
输入以下内容
```shell
{
"server": "0.0.0.0",
"port_password":
 {
     "18071":"a123456",
     "18072":"b123456",
     "18073":"c123456"
 },
"method": "aes-256-cfb"
}
```
method:为加密方法，可选aes-128-cfb, aes-192-cfb, aes-256-cfb, des-cfb, rc4-md5, salsa20, chacha20

port:为服务监听端口

password:为密码

上述配置的是多个用户
## 4.防火墙设置
```shell
 firewall-cmd --permanent --add-port=18071-18073/tcp  
 firewall-cmd --reload
``` 
## 5.配置自启动
新建启动脚本文件/etc/systemd/system/shadowsocks.service
```shell
vi /etc/systemd/system/shadowsocks.service
```
输入以下内容
```shell
[Unit]
Description=Shadowsocks
[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/ssserver -c /etc/shadowsocks.json
[Install]
WantedBy=multi-user.target
```
执行以下命令启动 shadowsocks 服务：
```shell
systemctl enable shadowsocks       #加入开机启动
systemctl start shadowsocks        #启动
systemctl restart shadowsocks　    #重新启动服务
```
检查 shadowsocks 服务是否已成功启动

```shell
systemctl status shadowsocks -l
```
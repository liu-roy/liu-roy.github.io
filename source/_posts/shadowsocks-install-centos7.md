---
title: shadowsocks之在centos7上安装
date: 2017-06-03 09:38:40
categories: 实用工具
tags: [shadowsocks, 翻墙]
---
## 前提条件
- 一台境外centos7服务器。（推荐国外云服务器Vultr/DigitalOcean/Linode/ Bandwagon）  


## shadowsock简介
shadowsock可以说是目前最稳定的健康(bypass the firewall)上网方式，由clowwindy大神分享并开源了他的解决方案 [github](https://github.com/shadowsocks/shadowsocks/releases)（master分支已被约谈删除，release还有），其原理导致firewall无法根据特殊关键字和连接方式屏蔽它， 具体的可以自行google

## 安装pip
pip是 python 的包管理工具。在本文中使用 python 版本的 shadowsocks，此版本的 shadowsocks 已发布到 pip 上，因此我们需要通过 pip 命令来安装。
```shell
yum -y update
yum install -y python-setuptools && easy_install pip
```
## 安装 shadowsocks
```shell
pip install shadowsocks
yum clean all
```
## 配置 shadowsocks
安装完成后，需要创建配置文件/etc/shadowsocks/shadowsocks.json 

```shell
vi /etc/shadowsocks/shadowsocks.json
```
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

上述配置的是多个用户，每个端口可以给多人使用

## 防火墙设置
 firewall-cmd --permanent --add-port=18071-18073/tcp  
 firewall-cmd --reload  

## 配置自启动(可选)  
新建启动脚本文件/etc/systemd/system/shadowsocks.service  

vi /etc/systemd/system/shadowsocks.service  

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
## 科学上网
添加线路，填写云服务器ip，端口，和端口对应的密码，选择对应的加密方式，保存即可，
具体可百度。 

- ios10 苹果应用商店搜索 wingy
- windows端 https://github.com/shadowsocks/shadowsocks-windows
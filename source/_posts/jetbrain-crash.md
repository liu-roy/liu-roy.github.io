---
title: centos7 Intellij Idea 授权服务器搭建(Jetbrain 家族系列IDE)
date: 2017-04-20 10:36:30
categories: 实用工具
tags: [centos7,idea]
---

## 1.上传破解文件
我用的是Xshell客户端，有上传功能，但是linux必须先装lrzsz，也可以通过其他方式传到linux上
```shell
yum -y install lrzsz
```

安装完成后，在终端输入rz，弹出上传窗口，上传文件即可

## 2.安装破解服务器
```shell
mkdir /home/IntellijIdea                                                 
mv IntelliJIDEALicenseServer_linux_amd64 /home/IntellijIdea/IdeaServer  
chmod +x IdeaServer  
screen -dmS IdeaServer ./IdeaServer -l xx.xx.xx.xx -p 1024 -prolongationPeriod 999999999999
```

## 3.为防火墙添加端口
```shell
firewall-cmd --add-port=1024/tcp --permanent
```
## 4.设置自启动
```shell
vi /etc/systemd/system/IdeaServer.service 
```
写入如下内容

```shell
[Unit]  
Description=IdeaServer service  
After=network.target  
[Service]  
Type=simple  
User=nobody  
ExecStart=/home/IntellijIdea/IdeaServer -l xx.xx.xx.xx -p 1024 -prolongationPeriod 999999999999
ExecReload=/bin/kill -HUP $MAINPID   
ExecStop=/bin/kill -s QUIT $MAINPID  
PrivateTmp=true  
KillMode=process  
Restart=on-failure  
RestartSec=5s  
[Install]   
WantedBy=multi-user.target
```


加入自启动  
```shell
systemctl enable IdeaServer  
systemctl start IdeaServer
```

检查 服务是否已成功启动
```shell
systemctl status IdeaServer -l
```
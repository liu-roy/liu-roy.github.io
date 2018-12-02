---
title: 利用filezilla搭建ftp服务器
date: 2016-03-12 22:06:36
categories: 实用工具
tags: [ftp, filezilla]
---

之所以没选择serv-u,一是因为收费，虽说网上有破解版，但是使用过程中发现破解版很不稳定，经常异常死掉，随后改选用免费的filezilla。
------------
##1软件获取
从百度搜索 FileZilla Server，下载即可，此软件分为客户端和服务端，注意区分
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/1.jpg)
##2软件安装
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/2.jpg)

![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/3.jpg)

![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/4.jpg)

![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/5.jpg)

![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/6.jpg)

点击install完成安装

填写server address，也可以不写，密码无需设置，点击ok
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/7.jpg)
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/8.jpg)

当出现如上图所示的loggedon 表示ftp服务器已经开启


## 3软件配置
### 添加用户
点击edit->users->general->add 填写用户名，添加一个用户，供第三方软件登录，然后点击enable account 设置密码，除用户名外，与下图保持一致即可，点击ok
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/9.jpg)
### 创建共享文件夹
点击shared floders ,为用户添加一个共享文件夹，用于传数据，如下图所示，F:\FtpTest为此用户的根目录，此目录下的所有文件和文件夹对此用户可见。文件和文件夹的权限可以自行配置，或者按照图中的对号进行勾选，点击ok
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/10.jpg)
### 设置上传和下载速度。
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/11.jpg)
### 设置超时时间
有三个超时时间，其他两个不加赘述，注意TimeOut settings，中间的no transfertimeout如果不为0，则表示一段时间不传输数据就会自动下线。可以理解为会话的保活时间
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/12.jpg)
### 设置日志信息（可省略）
主要用于其他用户的操作记录和问题查看
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/13.jpg)

## 从客户端登录查看文件
![](http://d17znh8lvwja9e.cloudfront.net/ftp-server/14.jpg)
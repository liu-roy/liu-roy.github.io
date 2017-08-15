---
title: 跨站请求伪造防御
date: 2017-08-14 16:22:41
categories: 网络安全
tags: csrf
---
## 背景
最近安全问题越来越多，公司软件也面临出海，刚开始公司软件大部分部在公安内网，安全问题没有太多重视。最近买了安全公司的扫描软件，一扫扫出很多安全问题，其中有一个是跨站请求伪造问题。

## 常见的攻击模式
### GET请求利用
使用GET请求方式的利用是最简单的一种利用方式，其隐患的来源主要是由于在开发系统的时候没有按照HTTP动词的正确使用方式来使用造成的。对于GET请求来说，它所发起的请求应该是只读的，不允许对网站的任何内容进行修改。

但是事实上并不是如此，很多网站在开发的时候，研发人员错误的认为GET/POST的使用区别仅仅是在于发送请求的数据是在Body中还是在请求地址中，以及请求内容的大小不同。对于一些危险的操作比如删除文章，用户授权等允许使用GET方式发送请求，在请求参数中加上文章或者用户的ID，这样就造成了只要请求地址被调用，数据就会产生修改。

现在假设攻击者（用户ID=121）想将自己的身份添加为网站的管理员，他在网站A上面发了一个帖子，里面包含一张图片，其地址为 *http://a.com/user/grant_super_user/121*
```
<img src="http://a.com/user/grant_super_user/121" />
```
设想管理员看到这个帖子的时候，这个图片肯定会自动加载显示的。于是在管理员不知情的情况下，一个赋予用户管理员权限的操作已经悄悄的以他的身份执行了。这时候攻击者121就获取到了网站的管理员权限。
<!--more-->
### POST请求利用

相对于GET方式的利用，POST方式的利用更加复杂一些，难度也大了一些。攻击者需要伪造一个能够自动提交的表单来发送POST请求。
```js
<script>
$(function() {
    $('#CSRF_forCSRFm').trigger('submit');
});
</script>
<form action="http://a.com/user/grant_super_user" id="CSRF_form" method="post">
    <input name="uid" value="121" type="hidden">
</form>
```
只要想办法实现用户访问的时候自动提交表单就可以了。

### 网上的方案
- 验证 HTTP Referer 字段
- 在请求地址中添加 token 并验证
- 在 HTTP 头中自定义属性并验证 

## 三种方案优缺点
### 验证 HTTP Referer 字段
- 容易篡改，低版本浏览器不安全，隐私软件可能禁用referer
- 公司软件没有域名，无法针对域名进行过滤

### 在请求地址/参数中添加token并验证
- 改动接口较多
- 难以保证token本身安全性

### 在HTTP 头中自定义属性并验证
- 改动很大 几乎要重写网站

## 自己的方案
由于写代码的时候post get使用比较规范，get方法用于获取资源，没有对后台数据修改的，所以扫描的问题绝大多数在post表单提交
针对扫描软件的扫描报告，跨站脚本攻击都是由form表单提交的接口发起

为了能减少对代码的修改，采用注解加拦截器的方式，这样做的好处是通过拦截器和注解，对特定方法做出一致性处理，减少代码量
具体流程图如下

![这里写图片描述](http://img.blog.csdn.net/20170815171336749)


### 关键代码CSRFInterceptor 
```java
public class CSRFInterceptor implements HandlerInterceptor {

    private final Logger logger = LoggerFactory.getLogger(CSRFInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod)handler;
        Method method = handlerMethod.getMethod();
        VerifyCSRFToken annotation = method.getAnnotation(VerifyCSRFToken.class);
        if (annotation != null && annotation.verify()) {
            String token = (String)request.getParameter(Constants.CSRF_TOKEN);
            if (token == null || !token.equals(request.getSession(true).getAttribute(Constants.CSRF_TOKEN))) {
                RestResult restResult = new RestResult(false, "CSRF Token Verify fail");
                restResult.putError("message" ,"CSRF Token 验证失败，请刷新页面");
                response.setContentType("text/html; charset=UTF-8");
                response.setCharacterEncoding("UTF-8");
                response.getWriter().append(JsonUtlis.getJsonUtlis().object2String(restResult));
                return false;
            }
        }
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }
}

```

### 关键代码VerifyCSRFToken
``` java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface VerifyCSRFToken {
    boolean verify() default true;
}
```
### 关键代码Controller
```java 
public class ScriptController {

    private static final ObjectMapper mapper = new ObjectMapper();

    private Logger logger = LoggerFactory.getLogger(ScriptController.class);

    @GetMapping(value = "")
    public String indexView(HttpServletRequest request, Model model) {
        model.addAttribute(Constants.CSRF_TOKEN, request.getSession(false).getAttribute(Constants.CSRF_TOKEN));
        return "script/script";
    }

	@PostMapping(value = "/upload")
    @VerifyCSRFToken
    public @ResponseBody RestResult uploadScript(@RequestParam("file")MultipartFile file, String type, 	HttpServletResponse response){
    }
```

参考资料
https://www.ibm.com/developerworks/cn/web/1102_niugang_csrf/
https://segmentfault.com/a/1190000008505616
http://blog.csdn.net/jrn1012/article/details/52750883

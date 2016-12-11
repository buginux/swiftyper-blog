---
title: 配置 Nginx 的目录浏览功能
date: 2016-12-08 14:10:08
tags: [Nginx]
---

有时候我们需要将服务器上的某些目录共享出来，让其他人可以直接通过浏览器去访问、浏览或者下载这些目录里的一些文件。

最近我就正好需要将一些静态的 HTML 页面部署到服务器上，让自己的多台设备能随时随地进行查看。

经过搜索之后找到了两个方法：一是使用 Node 的 [http-server](https://www.npmjs.com/package/http-server)，二是使用 Nginx 自带的 ngx_http_autoindex_module 模块。由于我自己的博客就是使用 Nginx 部署的，所以我就选择了第二种方法。

本篇文章介绍如何打开 Nginx 的目录浏览功能，配置简单的密码保护，并对索引页面进行美化。

<!-- more -->

## 配置目录浏览

要开启 Nginx 的目录浏览功能很简单，只需要打开 `nginx.conf` 或者对应的虚拟主机配置文件，在 server 或 location 段里面中上 `autoindex on;` 就可以了。

除了 autoindex 外，该模块还有两个可用的字段：
```
autoindex_exact_size on;
# 默认为 on，以 bytes 为单位显示文件大小；
# 切换为 off 后，以可读的方式显示文件大小，单位为 KB、MB 或者 GB。

autoindex_localtime on;
# 默认为 off，以 GMT 时间作为显示的文件时间；
# 切换为 on 后，以服务器的文件时间作为显示的文件时间。
```

除此之外，如果二级目录使用的是虚拟目录，则需要使用 `alias` 字段进行配置。

下面是一个完整的配置文件：
```
location /download {
    alias /home/user/static_files/;
    
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
}
```

关于 ngx_http_autoindex 模块的更详细信息可以参考[官方文档](http://nginx.org/en/docs/http/ngx_http_autoindex_module.html)。

## 中文乱码问题

如果开启了 Nginx 的目录浏览功能后发现中文目录名或者文件名显示为乱码，则要加上 `charset` 字段：
```
location /download {
    # ... 其它同上
    charset utf-8,gbk; # 两个字符集间不要加空格
}
```

## 添加目录密码保护

如果该目录是隐私目录，就需要为其增加密码保护。方法如下：
```
location /download {
    # ... 其它同上
    
    auth_basic "Enter your name and password";
    auth_basic_user_file /var/www/html/.htpasswd;
}
```

其中，`auth_basic` 字段是用户名、密码弹框上显示的文字（貌似在 Chrome 和 Safari 上面都没有用到），而 `auth_basic_user_file` 指定了记录登录用户名与密码的文件 `.htpasswd`，这个文件需要使用 `htpasswd` 命令或者[在线工具](http://tool.oschina.net/htpasswd)来生成。

`htpasswd` 命令是 MacOS 系统自带的命令，如果是 Windows 系统，建议直接使用在线生成工具比较方便。

```bash
# 创建一个全新的文件，会清除文件里的全部用户
$ htpasswd -c /var/www/html/.htpasswd user1  
# 添加一个用户，如果用户已存在，则修改密码
$ htpasswd -b /var/www/html/.htpasswd user2 password
# 删除一个用户
$ htpasswd -D /var/www/html/.htpasswd user2
```
更具体的使用可以参考[官方文档](https://httpd.apache.org/docs/current/programs/htpasswd.html)。

到目前为止，我们已经完成了对 Nginx 目录浏览的全部配置。但是，默认的页面样式有点难看，我们要对其进行一些美化（装扮QQ空间即视感）。

> 如果对页面样式没有要求，下面的部分就不需要阅读了。

## 使用 FancyIndex 进行美化

### 安装 FancyIndex
网上的 FancyIndex 安装教程大多数是在编译 nginx 时，添加这个插件。但是，由于我的服务器是 ubuntu 系统，安装时图方便直接使用了 `apt-get install nginx` 安装了 nginx。如果说现在为了安装一个 FancyIndex 要重新进行一次 nginx 的编译和配置，我想我也没那个折腾的心情。

幸好 ubuntu 最好的地方就在于它的仓库源很多。在 ubuntu 系统上，我们可以通过安装 `nginx-extras` 来安装 FancyIndex 插件。

```bash
$ sudo apt-get install nginx-extras
```

安装完成之后，就要对页面进行美化了。

由于我不是前端，要真让我自己手写来对页面进行调整，那估计就不是美化，而是对页面的摧毁了。幸好，对于美化的东西，网上正常都能找到主题或者模板，FancyIndex 也不例外。[这里](https://github.com/TheInsomniac/Nginx-Fancyindex-Theme)就有一个简洁大方的主题可以直接拿来使用。

### 配置 FancyIndex

首先，将[这个主题](https://github.com/TheInsomniac/Nginx-Fancyindex-Theme)克隆下来。

在网站根目录（如 `/var/www/html` ）下新建一个 `fancyindex` 目录，然后将下面的文件复制到该目录中：
* header.html
* footer.html
* css/fancyindex.css
* fonts/*
* images/breadcrumb.png

最后修改 nginx 配置文件，下面是完整的配置文件：
```
location /download {
	alias /home/user/static_files/;
	charset utf-8,gbk;

	auth_basic "Enter your name and password";
	auth_basic_user_file /var/www/html/.htpasswd;

	fancyindex on;
	fancyindex_exact_size off;
	fancyindex_localtime on;
	fancyindex_header "/fancyindex/header.html";
	fancyindex_footer "/fancyindex/footer.html";
	fancyindex_ignore "fancyindex";
}
```
注意，使用 fancyindex 之后需要将 autoindex 相关的字段去掉，否则可能会造成冲突。

> [文档](https://github.com/aperezdc/ngx-fancyindex#directives)上面说明了有两个字段 `fancyindex_default_sort` 和 `fancyindex_name_length` 可以分别用来指定文件排序和文件名的最大长度，但是我试过之后都不起作用，可能是由于 nginx-extras 里面的 FancyIndex 版本比较低的缘故。

下图是配置完后的最终效果：
![Final Touch](http://7xqonv.com1.z0.glb.clouddn.com/nginx-autoindex-configuration-pic-1.png)

## 参考资料
* [Nginx配置索引（目录浏览），美化索引页面，配置目录密码保护](https://www.zhukun.net/archives/7343)
* [使用FancyIndex插件美化nginx文件目录列表](http://www.tennfy.com/2489.html)


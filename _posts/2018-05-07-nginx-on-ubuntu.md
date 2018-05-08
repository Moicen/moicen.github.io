---
layout: post
title: Ubuntu搭建nginx服务
abstract: 在Ubuntu上搭建nginx服务
category: technical
permalink: ubuntu-nginx
author: 木逸辰
tags: [tech, ubuntu, nginx]
---

### {{ page.title }}

使用nginx可以很方便地搭建服务器，反向代理功能还能让我们轻松使用自定义域名，非常实用。

首先在当然是安装nginx，ubuntu的packages里面有，所以直接使用`apt`方式安装就行了：

	apt get install nginx

等待安装，完成之后，测试下是否成功：

![nginx](/assets/images/2018-05-07-ubuntu-nginx-version.png)

运行`nginx`就启动了服务，打开浏览器访问localhost，会看到nginx的欢迎页面：

![nginx](/assets/images/2018-05-07-nginx-welcome.png)

如果提示权限不足，则需要使用`sudo`来操作。

要关闭nginx或者重启，运行：

	nginx -s signal

`nginx`是命令，`-s`是选项(option)，`signal`是参数，就是控制nginx执行何种命令，有以下几种：

- `stop` 快速关闭
- `quit` — 正常关闭
- `reload` — 重新加载配置文件
- `reopen` — 重新打开日志文件

控制nginx使用的是`nginx.conf`配置文件，该文件在一般在`/usr/local/nginx/conf`、`/etc/nginx`或`/usr/local/etc/nginx`。我在ubuntu中的目录是`etc/nginx`，mac中的目录则是`/usr/local/etc/nginx`。

当nginx接受到`reload`指令时，它会首先检查配置文件的正确性，然后开启一个新的子进程，并给旧的子进程发送消息告诉它该终止运行了。

接下来修改`nginx.conf`文件，尝试开启一个静态文件的WEB服务。

在`http`块中，添加一个`server`块，定义好服务器监听端口和路径：

	server {
		listen 8080;
	    location / {
	        root /data/static;
	    }

	    location /images/ {
	        root /data/static/img;
	    }
	}

在`/data/static`目录添加一个index.html文件，用于测试，运行命令加载刚才修改的配置：

	nginx -s reload

打开`localhost:8080`：

![nginx](/assets/images/2018-05-07-nginx-firefox.png)

完成！


---
layout: post
title: docker中使用supervisor管理nginx进程
abstract: docker中使用supervisor管理nginx进程
category: technical
permalink: supervisor-nginx
author: 木逸辰
tags: [tech, docker, nginx, supervisor]
---

### {{ page.title }}


首先安装nginx和supervisor

```sh
$ apk add supervisor nginx
```

给supervisor添加配置文件

```ini
[unix_http_server]
file=/run/supervisord.sock

[supervisord]
logfile=/root/logs/supervisord.log
loglevel=info
nodaemon=true
user=root

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///run/supervisord.sock ; use a unix:// URL  for a unix socket

[inet_http_server]
port=0.0.0.0:9001
[program:web]
command=/usr/sbin/nginx -g 'daemon off;'
stderr_logfile=/root/logs/err.log
stdout_logfile=/root/logs/out.log
```

注意，这里的command必须加 `-g ‘daemon off;’`参数，指定nginx运行在forground。因为nginx默认是使用daemon模式运行，supervisor无法管理daemon进程。

3. 启动supervisor：

```sh
$ supervisord -c /etc/supervisor.d/default.ini
```

supervisor默认使用9001端口提供一个web控制台，可以直接在里面对所管理的服务进行管理：

![supervisor](/assets/images/2019-09-10-supervisor-nginx.png)

这里supervisor的配置也有`nodaemon=true`，使其运行在forground，这里是为了方便在使用docker-compose的时候将supervisor的日志输出到docker-compose这一层，方便观察进程状态

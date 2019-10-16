---
layout: post
title: docker容器文件和端口映射
abstract: docker容器文件和端口映射
category: technical
permalink: docker-port-volume-map
author: 木逸辰
tags: [tech, docker, map, volume port]
---

### {{ page.title }}


在创建容器时，需要指定映射的目录（本地及容器）和端口：

    ```bash
    # 冒号前面为host目录/端口，后面为container目录/端口
    $ docker create -it -v ~/projs/mywork/:/mywork -p 3000:3000 moicen/ruby bash
    ```

如果要在容器创建完成后添加映射，参考[Attach a volume to a container while it is running](https://jpetazzo.github.io/2015/01/13/docker-mount-dynamic-volumes/)

如果要从host访问容器中的http服务，只是映射端口的话，从host可能依旧无法访问容器内的服务：

![port-map](/assets/images/2019-06-01-docker-port-map.png)

检查配置之后，发现是容器里的http服务只侦听里`127.0.0.1`的IP，也就是只能从这个容器里面才能访问到服务，如果要从host也能访问到，则需要侦听`0.0.0.0`IP，即所有IP。

```ini
[inet_http_server]
port=0.0.0.0:9001
```

重新启动，从host访问成功。

![port-map](/assets/images/2019-06-01-docker-port-map-supervisor.png)
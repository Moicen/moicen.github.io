---
layout: post
title: 在Mac OSX上编译免费不限流量的lantern
abstract: lantern是一款不错的开源翻墙软件，其免费版每月流量仅800MB，但因为是开源的，其实可以自己编译一个不限流量的版本。本文介绍自己在Mac上编译的过程。
category: technical
permalink: build-free-lantern-on-mac
author: 木逸辰
tags: [tech, lantern, mac]
---

### {{ page.title }}

[lantern](https://www.getlantern.org)最早由Google公司开发，完全免费，但早期效果不是很好，经常连不上。后来出来了收费版，免费版每月流量限制为800MB。但据[官方说法](https://github.com/getlantern/forum/issues/378)，每月800MB流量使用完了之后，其实还可以继续使用，只是限制了速度，看文字没问题，图片视频就不要想了。所以如果没大流量需求的话，免费版就够用了。

最早我使用的[Green VPN](https://www.igreenjsq.info)，速度还不错，但后来忽然就被干掉了，官网都打不开了，此后一直使用lantern免费版。Google基本没用，导致gmail也一直连不上，[GitHub](https://github.com)和[stackoverflow](http://stackoverflow.com)时不时被墙，也是很烦，至于[youtube](http://youtube.com)就更不用说了。

前阵子偶然看到这篇[《lantern编译过程》](http://blog.lanyus.com/archives/290.html)，介绍了使用docker编译Linux和windows版本的lantern，但没有Mac版的编译过程。于是想自己试试编译一个Mac版的。

过程其实很简单，按官网介绍来就行，先拉下lantern的repository:

    git clone https://github.com/getlantern/lantern.git

编译之前要先安装一些`Prerequisites`，其中[`go`](http://golang.org)和[`node`](http://nodejs.org)可以直接去各自官网下载安装文件安装，也可以用`homebrew`安装:

    brew install go //Go lang
    brew install node   //nodejs

而[`gulp-cli`](http://gulpjs.com)需要用[`npm`](http://npmjs.com)安装:

    npm i gulp-cli -g

`GNU Make`在`Mac OSX`中是自带的，然后进入lantern目录。先要设置版本号，以防止自动读取已发布的新版本:

    export VERSION=9.9.9

然后就可以编译了，这里注意，可以直接使用`make`命令:

    make darwin //make lantern for mac

结果报错:

    Build tags:
    Extra ldflags:
    # github.com/getlantern/flashlight/proxied
    src/github.com/getlantern/flashlight/proxied/proxied.go:391: tr.MaxIdleTime undefined (type *http.Transport has no field or method MaxIdleTime)
    src/github.com/getlantern/flashlight/proxied/proxied.go:392: tr.EnforceMaxIdleTime undefined (type *http.Transport has no field or method EnforceMaxIdleTime)
    # github.com/getlantern/flashlight/client
    src/github.com/getlantern/flashlight/client/reverseproxy.go:20: unknown field 'MaxIdleTime' in struct literal of type http.Transport
    src/github.com/getlantern/flashlight/client/reverseproxy.go:22: transport.EnforceMaxIdleTime undefined (type *http.Transport has no field or method EnforceMaxIdleTime)
    make: *** [darwin] Error 2

按照提示找到对应的文件对应位置
`src/github.com/getlantern/flashlight/proxied/proxied.go:391`
`src/github.com/getlantern/flashlight/client/reverseproxy.go:20`

把`MaxIdleTime`换成`IdleConnTimeout`，然后把下面一句调用`EnforceMaxIdleTime()`这个方法的语句注释掉。重新编译。成功。

但这条命令会只会生成一个叫`lantern_darwin_amd64`的可执行文件，这样:

![lantern_darwin_amd64](/assets/images/lantern_darwin_amd64.png)

而如果按照官网步骤:

    make lantern && ./lantern

或
    make clean-desktop lantern && ./lantern

这样会生成一个名为`lantern`的可执行文件，并且会直接自动运行。

到此lantern已经可以运行了，但我想生成一个`.app`文件，查看`Makefile`文件，有好几个`package`任务，找到Mac平台的`package-darwin`，重新编译:

    make clean-desktop package-darwin

完成！获得`lantern.app`文件，

![lantern_app](/assets/images/lantern_app.png)

跟官网上提供的下载版本一样，只是不再有流量限制了。

[![lantern](/assets/images/lantern.png)]({{site.url}}/assets/images/lantern.png)

来处理累积了很久的Gmail邮件

![gmail](/assets/images/gmail.png)

另外这个命令执行到最后报了个错:

    Generating distribution package for darwin/amd64 manoto...
    error: The specified item could not be found in the keychain.
    make: *** [package-darwin-manoto] Error 1

生成`distribution package`时出现错误，还跟`keychain`有关，暂时不清楚啥情况，不过需要的`lantern.app`文件已生成而且可用，暂时不管了。

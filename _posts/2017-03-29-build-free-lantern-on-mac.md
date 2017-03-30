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

之所以出现这个错误，是因为我安装的`go`是直接从`go lang`官网下载的，用`homebrew`安装的也是一样，在官方版本里`transport.go`文件里只有`IdleConnTimeout`:

    // IdleConnTimeout is the maximum amount of time an idle
	// (keep-alive) connection will remain idle before closing
	// itself.
	// Zero means no limit.
	IdleConnTimeout time.Duration

而[`lantern`作者使用了一个自定义的`go`](https://github.com/getlantern/lantern/issues/4692)，在`transport.go`文件中删除了`IdleConnTimeout`字段，增加了`MaxIdleTime`字段和`EnforceMaxIdleTime`方法的定义，应该是用来专门对`lantern`做优化支持的。

    // MaxIdleTime, if non-zero, controls how long to hold on to keep-alive
	// connections. Once a connection hits its MaxIdleTime, it will be closed.
	// You need to call EnforceMaxIdleTime() at least once for this to be
	// enforced on an ongoing basis.
	MaxIdleTime time.Duration

    // EnforceMaxIdleTime checks for idle connections that have exceeded the
    // MaxIdleTime and closes them.
    func (t *Transport) EnforceMaxIdleTime() {
    	if t.MaxIdleTime == 0 {
    		// Nothing to maintain
    		return
    	}
    	t.maintainIdleOnce.Do(func() {
    		go func() {
    			for {
    				time.Sleep(t.MaxIdleTime / 10)
    				now := time.Now()
    				toClose := make([]*persistConn, 0)
    				t.idleMu.Lock()
    				for key, conns := range t.idleConn {
    					retainedConns := make([]*persistConn, 0, len(conns))
    					for _, pconn := range conns {
    						if now.Sub(pconn.lastIdled) > t.MaxIdleTime {
    							toClose = append(toClose, pconn)
    						} else {
    							retainedConns = append(retainedConns, pconn)
    						}
    					}
    					if len(retainedConns) == 0 {
    						delete(t.idleConn, key)
    					} else {
    						t.idleConn[key] = retainedConns
    					}
    				}
    				t.idleMu.Unlock()

    				// Close connections outside of mutex to minimize locking time
    				for _, pconn := range toClose {
    					pconn.close(errCloseIdleConns)
    				}
    			}
    		}()
    	})
    }

因此如果使用他们提供的`go`语言编译就不会出现问题，而如果使用官方版本的`go`，就需要按照提示找到对应的文件对应位置
`src/github.com/getlantern/flashlight/proxied/proxied.go:391`
`src/github.com/getlantern/flashlight/client/reverseproxy.go:20`

把`MaxIdleTime`换成`IdleConnTimeout`，然后把下面一句调用`EnforceMaxIdleTime()`这个方法的语句注释掉。重新编译。成功。

但这条命令会只会生成一个叫`lantern_darwin_amd64`的可执行文件，这样:

![lantern_darwin_amd64](/assets/images/2017-03-29-lantern_darwin_amd64.png)

而如果按照官网步骤:

    make lantern && ./lantern

或
    make clean-desktop lantern && ./lantern

这样会生成一个名为`lantern`的可执行文件，并且会直接自动运行。

到此lantern已经可以运行了，但我想生成一个`.app`文件，查看`Makefile`文件，有好几个`package`任务，找到Mac平台的`package-darwin`，重新编译:

    make clean-desktop package-darwin

居然报了个错:

    Generating distribution package for darwin/amd64 manoto...
    error: The specified item could not be found in the keychain.
    make: *** [package-darwin-manoto] Error 1

生成`distribution package`时出现错误，还跟`keychain`有关。搜了一下，发现是在[编译分发包时出现签名问题](https://github.com/getlantern/lantern/issues/3942)，可以忽略掉，因为我们需要的`.app`文件已经生成了，而且可用。或者可以参考[这个方法](https://github.com/getlantern/lantern/issues/4202)，这样不仅生成`.app`文件，还会生成`.dmg`文件，用于分发安装。

![lantern_app](/assets/images/2017-03-30-lantern.png)

可以直接打开`Lantern.app`，或者拷贝进`Application`文件夹，也可以点击`lantern-installer.dmg`，就跟一般的安装过程一样了。
打开`Lantern.app`，跟官网上提供的下载版本一样，只是不再有流量限制了。

![lantern](/assets/images/2017-03-29-lantern-page.png)

来处理累积了很久的Gmail邮件

![gmail](/assets/images/2017-03-29-gmail.png)

到此为止基本可算完成，但发现每次打开`lantern.app`时，不光会在浏览器中打开默认的`127.0.0.1:port`页面，同时还会打开一个叫[manototv](https://www.manototv.com/iran?utm_campaign=manotolantern)的页面，看着眼烦，于是继续翻代码，在`src/github.com/getlantern/flashlight/ui/ui.go`中发现了打开这个地址的代码:

    // openExternalUrl opens an external URL of one of our partners automatically
    // at startup if configured to do so. It should only open the first time in
    // a given session that Lantern is opened.
    func openExternalUrl(u string) {
    	var url string
    	if u == "" {
    		return
    	} else if strings.HasPrefix(u, "https://www.manoto1.com/") || strings.HasPrefix(u, "https://www.facebook.com/manototv") {
    		// Here we make sure to override any old manoto URLs with the latest.
    		url = "https://www.manototv.com/iran?utm_campaign=manotolantern"
    	} else {
    		url = u
    	}
    	time.Sleep(4 * time.Second)
    	err := open.Run(url)
    	if err != nil {
    		log.Errorf("Error opening external page to `%v`: %v", uiaddr, err)
    	}
    }

可以直接把对应代码注释，重新编译，但既然是可配置的，于是想找到是在哪配置的，继续翻，在`src/github.com/getlantern/flashlight/config/bootstrap.go` 中找到读取配置的代码:

    func bootstrapPath(fileName string) (string, string, error) {
    dir, err := filepath.Abs(filepath.Dir(os.Args[0]))
    if err != nil {
        log.Errorf("Could not get current directory %v", err)
        return "", "", err
    }
    var yamldir string
    if runtime.GOOS == "windows" {
        yamldir = dir
    } else if runtime.GOOS == "darwin" {
        // Code signing doesn't like this file in the current directory
        // for whatever reason, so we grab it from the Resources/en.lproj
        // directory in the app bundle. See:
        // https://developer.apple.com/library/mac/technotes/tn2206/_index.html#//apple_ref/doc/uid/DTS40007919-CH1-TNTAG402
        yamldir = dir + "/../Resources/en.lproj"
        if _, err := ioutil.ReadDir(yamldir); err != nil {
            // This likely means the user originally installed with an older version that didn't include en.lproj
            // in the app bundle, so just look in the old location in Resources.
            yamldir = dir + "/../Resources"
        }
    } else if runtime.GOOS == "linux" {
        yamldir = dir + "/../"
    }
    fullPath := filepath.Join(yamldir, fileName)
    log.Debugf("Opening bootstrap file from: %v", fullPath)
    return yamldir, fullPath, nil
    }

然后找到这个`fileName`参数，最终指向本文件最顶部的一个定义:`name = ".packaged-lantern.yaml"`，原来是从这个文件里读取的，于是继续找，但在源代码里找不到，最后发现这个文件是最终打包时生成的，在`Lantern.app/Contents/Resources/en.lproj/`目录下，打开，就只有一行代码:

    startupurl: https://www.facebook.com/manototv/app_128953167177144

然后就简单了，直接把这行代码删除掉，不用重新编译。重新打开`Lantern.app`，果然没有弹出其他页面了。这里是一个比较懒的办法，直接修改的已打包的文件，下次再编译时还会生成同样的配置文件，所以要从源头解决问题，继续找在哪里做的配置。最后在`Makefile`里发现这样的代码:

    //package-darwin-manoto任务
	mkdir Lantern.app/Contents/Resources/en.lproj && \
	cp installer-resources/$(MANOTO_YAML) Lantern.app/Contents/Resources/en.lproj/$(PACKAGED_YAML) && \
	cp $(LANTERN_YAML_PATH) Lantern.app/Contents/Resources/en.lproj/$(LANTERN_YAML) && \

看到了熟悉的目录，原来是在这里生成了`app`中对应的目录，并把配置文件拷贝过去，于是找源配置文件，在`installer-resources`目录下有三个`.yaml`文件，`.packaged-lantern-manoto.yaml`、`.packaged-lantern.yaml`和`lantern.yaml`，都打看看看，`.packaged-lantern-manoto.yaml`里只有一行，就是上面那行定义`startupurl`的，另外两个文件都是空的。于是就简单了，把这行代码删除掉，重新编译，问题解决。

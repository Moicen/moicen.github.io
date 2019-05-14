---
layout: post
title: 使用nvm管理多版本node
abstract: 使用nvm管理多版本node
category: technical
permalink: use-nvm-to-manage-node
author: 木逸辰
tags: [tech, nvm, node, npm, yarn]
---

### {{ page.title }}

从[官方网站](https://nodejs.org)下载的NodeJS安装包，安装完成之后，使用`npm`进行`global`安装时，需要`root`权限，非常不方便。

![npm permission error](/assets/images/2019-05-14-npm-permission-error.jpg)

另外如果也没办法安装多个版本的NodeJS。使用[nvm](https://github.com/nvm-sh/nvm)可以解决这两个问题。

首先安装`nvm`，使用curl或者wget下载并运行安装脚本。


```bash
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
$ wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
```

![nvm install](/assets/images/2019-05-14-nvm-install.jpg)

因为`nvm`并不是一个可执行文件，而只是一个shell函数，因此，安装完后需要重新加载`~/.bash_profile`文件，并且`which nvm`也是无效的：

```bash
$ . ~/.bash_profile
```

![nvm use](/assets/images/2019-05-14-nvm-use.jpg)

现在`nvm`可以正常使用了，看看现在的node安装情况：

```bash
$ nvm  ls

->       system
iojs -> N/A (default)
node -> stable (-> N/A) (default)
unstable -> N/A (default)
$ nvm current
system
```

可以看到，我的系统里有一个`system`版本的NodeJS，这是我用从官方网站下载的安装包安装的，现在再安装一个其他版本：

```bash
$ nvm install 10.14.1
Downloading and installing node v10.14.1...
Downloading https://nodejs.org/dist/v10.14.1/node-v10.14.1-darwin-x64.tar.xz...
##################################################################################################################################################################################################### 100.0%
Computing checksum with shasum -a 256
Checksums matched!
Now using node v10.14.1 (npm v6.4.1)
Creating default alias: default -> 10.14.1 (-> v10.14.1)
$ npm -v
6.4.1
$ node -v
v10.14.1
```

再看下安装情况：

![nvm show](/assets/images/2019-05-14-nvm-show.jpg)

可以看到，现在已经使用刚才安装的`v10.14.1`版本作为当前版本了，使用`npm`安装全局工具：

```bash
$ npm install --global create-react-app@2.1.2
/Users/moicen/.nvm/versions/node/v10.14.1/bin/create-react-app -> /Users/moicen/.nvm/versions/node/v10.14.1/lib/node_modules/create-react-app/index.js
+ create-react-app@2.1.2
added 63 packages from 20 contributors in 12.675s
```

安装成功，不需要`sudo`运行，也没有权限问题。
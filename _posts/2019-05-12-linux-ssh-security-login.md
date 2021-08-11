---
layout: post
title: Linux使用ssh安全登录
abstract: Linux使用ssh安全登录
category: technical
permalink: linux-ssh-security-login
author: 木逸辰
tags: [tech, linux, ssh, security, login]
---

### {{ page.title }}

现在的服务器几乎都是使用Linux操作系统，linux是多用户系统，root用户拥有可以对系统做任何操作的权限。因此我们不能直接使用root来进行日常登录和文件传输等操作，而是应该为每个使用者建立相应的用户和用户组，并禁止root登录，提供密钥登录方式，确保安全。

### 1. 使用`root`登录，添加一个常用用户，并赋予`sudo`权限：

```bash
# 添加用户
[root ~] useradd -m moicen
# 设置密码
[root ~] passwd moicen
# 设置无密码使用sudo权限
[root ~] echo 'moicen ALL=(ALL) NOPASSWD: ALL'  >> /etc/sudoers
[root ~] mkdir /home/moicen/.ssh
# 新创建的目录，默认的权限是`drwxrwxr-x`，需修改为700，否则ssh会认为不安全而拒绝
[root ~] ls -al
drwxrwxr-x  2 moicen moicen 4096 May 12 14:39 .ssh
[root ~] chmod 700 /home/moicen/.ssh
# 将本机的~/.ssh/id_rsa.pub文件内容复制到这个authorized_keys文件中
[root ~] vi /home/moicen/.ssh/authorized_keys
[root ~] chmod og-rw /home/moicen/.ssh/authorized_keys
```


**注意**这里的文件和目录权限必须设置好，比如上面.ssh和authorized_keys的owner都是moicen，所以如上设置就可以了，但如果你的这两个目录文件的owner是root，那么还需要给其他用户相应的read和exec权限，否则还是无法登陆，如下图：

![ssh file privilege](/assets/images/2019-05-12-ssh-file-privilege.png)


这里，如果是使用有sudo权限的非root用户登录，可以添加用户，设置密码，但是无法设置sudo权限：

![set sudo without root](/assets/images/2019-05-12-set-sudo-without-root.jpg)

需要切换到root账号操作：

![su root](/assets/images/2019-05-12-su-root.jpg)


### 2. 将root用户的ssh登录禁用，修改`/etc/ssh/sshd_config`文件，设置`PermitRootLogin`为`yes`并取消注释，

```bash
#LoginGraceTime 2m
PermitRootLogin no
#StrictModes yes
MaxAuthTries 3
#MaxSessions 10
```

这里也可以将密码登录完全禁用，设置`PasswordAuthentication no`并取消注释

重启`ssh`服务：`systemctl restart sshd.service`，退出服务器。


### 3. 在本机登录新建用户`ssh moicen@server`:

![ssh login without password](/assets/images/2019-05-12-moicen-ssh-login.jpg)





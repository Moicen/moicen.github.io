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
[root ~] vi /home/admin/.ssh/authorized_keys
# 将本机的~/.ssh/id_rsa.pub文件内容复制到这个authorized_keys文件中
```

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





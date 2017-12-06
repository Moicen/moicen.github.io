---
layout: post
title: 使用NodeJS实现第三方登录
abstract: 第三方登录为用户提供了非常方便的注册登录方式，对于吸引用户帮助很大。
category: technical
permalink: third-party-login-by-nodejs
author: 木逸辰
tags: [tech, nodejs]
---

### {{ page.title }}

已集成第三方登录功能为插件，发布于[NPM](https://www.npmjs.com/package/third-party-login)，可直接安装使用，方便省事儿。

最近项目中要用到第三方登录，因为自己此前从未接触过，加之时间仓促，因而踩了不少坑，在此记录下。

项目中需要实现[微博](https://weibo.com)、[微信](http://weixin.qq.com)、[支付宝](https://www.alipay.com)三家平台的第三方登录。先翻看了下三家开放平台的文档，流程其实很简单：

>用户点击第三方登录按钮 → 跳转到第三方指定的授权登录页 → 用户登录授权 → 带着`auth_code`跳回自己在第三方设置的回调地址 → 使用`auth_code`换取`access_token` → 使用`access_token`调用第三方API获取需要的信息

**第一步**、平台设置

首先当然是要在各家社交站点的开放平台注册开发者账号，添加网站应用，获得相应的`app key`(`app id`)和`app secret`，然后配置相应的应用网关和回调地址，如图为支付宝的配置：

![alipay](/assets/images/2017-11-08-3rd-login-alipay-1.png)

应用网关配置为自己站点的域名，回调地址为授权登录之后跳转回来的地址。微博的设置也类似，一个域名一个回调地址。但是微信的有些不一样，只有一个 **授权回调域**，填上域名就可以了。我最初一直设置的是回调地址，结果一直报`redirect_uri`参数不正确的错误，改成域名就可以了。

**第二步**、程序开发

我的项目使用的是`NodeJS` + `express`，路由先写好：

    var express = require('express');
    var router = express.Router();

    var login = require("./login");

    router.get("/login/weibo", login.weibo.auth);
    router.get("/login/callback/weibo", login.weibo.token);
    router.get("/login/cancel/weibo", login.weibo.cancel);
    //支付宝微信类似
    ...

然后在`login.js`里写处理逻辑。第一步处理很简单，都是跳转到相应平台指定的登录授权地址，把需要的参数拼进`url`就行了。各平台对参数要求略有出入，以微博为例：

    auth: function () {
        var opt = platforms.opt("weibo");
        var url = [opt.url.auth];
        url.push("?client_id=" + opt.app.key);
        url.push("&response_type=code&state=" + (new Date()).valueOf());
        url.push("&redirect_uri=" + encodeURIComponent(callback_url + "weibo"));
        return url.join("");
    }

然后是获取`token`的处理，`token`获取的url也是根据文档要求拼出来就行了：

    token: function (req, res, next) {
        var code = req.query.code;
        //未授权
        if(!code){
            res.redirect("/");
            return;
        }
        var options = {
            url: tool.url("weibo", "token", code),
            method: "POST"
        };
        request(options, function(err, response, body) {
            if(err || !body){
                failure(res, "用户授权验证失败！");
                return;
            }
            if(typeof body === "string") body = JSON.parse(body);
            weibo.user(body.uid, body.access_token, req, res);
        });
    }

获取到`token`后，同时也能拿到用户在该平台的唯一ID：`uid`，如果只需要这些数据，一般就不需要下一步了。如果需要读取其他更多信息，比如获取用户的昵称等等，就拿`token`去调用相应的API就行了。如我这里需要获取用户昵称，所以还得再调用一次用户接口，接口地址和参数各平台文档都写的很清楚，照着写就行了：

    user: function (uid, token, req, res) {
        var opt = {
            url: tool.url("weibo", "user", token, uid),
            method: "GET"
        };
        request(opt, function (err, response, body) {
            if(err){
                failure(res, "获取用户信息失败！");
            }
            var user = JSON.parse(body);
            //获取到用户数据，接下来根据业务进行其他操作
            ...
        });
    }

**第三步**、注意事项

微博和微信的接口调用都很简单，参数也简单，基本上没什么可说的。支付宝为了安全起见，添加了一个签名参数，要做一些签名和验签操作：

    token: function (code) {
        var opt = platforms.opt("alipay");
        var params = {
            app_id: opt.app.key,
            method: "alipay.system.oauth.token",
            format: "json",
            charset: "utf-8",
            sign_type: "RSA2",
            timestamp: helper.timestamp(),
            version: "1.0",
            grant_type: "authorization_code",
            code: code
        };
        params["sign"] = secure.sign(params);

        return { url: opt.url.gateway, params: params };
    }

支付宝从获取`token`开始后面每一步都需要添加一个`sign`签名参数，签名方法文档写得很清楚，支付宝另外还提供了一个工具，用于生成密钥，并提供签名和验签的功能，非常方便：

![alipay](/assets/images/2017-11-08-3rd-login-alipay-2.jpg)

这里要注意的，一个是`charset`参数。在获取`token`的请求头里也要设置一样的`charset`：

    options: function (url, params) {
        return  {
            url: url,
            json: true,
            form: params,
            headers: {
                "content-type": "application/x-www-form-urlencoded;charset=utf-8"
            }
        };
    }

这两个地方必须都设置且保持一致，不然接口返回的就会是乱码。

另一个就是签名了，推荐使用RSA2签名算法，先把除`sign`之外的所有参数按字母表顺序排序，然后使用私钥进行签名，作为`sign`参数一起发送。从支付宝返回的数据也包含`sign`参数，使用同样的方式签名，这边也要对其验签，以保证安全性。

    var secure = {
        key: function (type) {
            var file = path.join(process.cwd(), "data/pem/ali-" + type + ".pem");
            if (!fs.existsSync(file)) {
                return "";
            }
            return fs.readFileSync(file).toString();
        },
        sign: function (params) {
            var sign = crypto.createSign('RSA-SHA256');
            var content = helper.parameter(params);
            sign.update(content);
            var key = this.key("private");
            return sign.sign(key, 'base64');
        },
        verify: function(params, signature){
            var verify = crypto.createVerify("RSA-SHA256");
            var content = helper.parameter(params);
            verify.update(content);
            var key = this.key("public");
            return verify.verify(key, signature, "base64");
        }
    };

下面是前面所用到的几个辅助函数：

    var helper = {
        parameter: function (params) {
            var names = [], parts = [];
            for (var name in params) {
                //剔除空值参数和sign参数
                if (params[name] && name !== "sign") {
                    names.push(name);
                }
            }
            names.sort().forEach(function (name) {
                parts.push(name + "=" + params[name]);
            });
            return parts.join("&");
        },
        padding: function (num) {
            return (num < 10 ? "0" + num : num).toString();
        },
        timestamp: function () {
            var date = new Date();
            return date.getFullYear() + "-" + this.padding(date.getMonth() + 1)
                + "-" + this.padding(date.getDate()) + " " + this.padding(date.getHours()) + ":"
                + this.padding(date.getMinutes()) + ":" + this.padding(date.getSeconds());
        }
    };

以上。如果要添加其他平台比如QQ、豆瓣等等的第三方登录，过程也是一致的。

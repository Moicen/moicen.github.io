---
layout: post
title: 微信公众号服务器接口配置及数字签名验证-ruby版
abstract: 微信公众号服务器接口配置及数字签名验证-ruby版
category: technical
permalink: weixin-server-signature-ruby
author: 木逸辰
tags: [tech, weixin, signature, ruby]
---

### {{ page.title }}

在微信平台开发时，如果配置了服务器地址，则微信会废除自定义菜单和自动回复，转而将用于的交互信息发送到开发者所配置的服务器地址，让开发者自行处理。

![weixin server](/assets/images/2019-05-06-wexin-server-url.png)


同时微信会对数据进行签名，用于安全性校验：

![weixin signature](/assets/images/2019-05-06-wexin-signature.jpg)

下面记录下使用ruby进行签名校验的过程。

先定义一个用于校验签名的方法：

```ruby
    # 根据参数校验请求是否合法，如果非法返回错误页面
    def verify
        require 'digest'
        # 这里是在公众号进行服务器配置的时候，配置的token
        token = Rails.application.config_for(:weixin)["service"]["token"]
        str = [token, params['timestamp'], params['nonce']].sort.join('')
        signature = Digest::SHA1.hexdigest(str)
        render html: "Forbidden", :status => 403 if params[:signature] != signature
    end
```

再定义两个方法，分别接收微信提交的`post`和`get`方法（路由地址相同），在`get`方法中校验签名

```ruby

    # 使用前置过滤器校验
    before_action :verify, only: [:receive]

    # 用于接收get请求，校验签名
    def receive
        # 校验成功返回微信需要的字段
        render plain: params[:echostr]
    end

    # 处理post请求
    def handle
        xml = Nokogiri::XML(request.body.read)

        msgType = xml.xpath("//MsgType").text
        event = xml.xpath("//Event").text
        openid = xml.xpath("//FromUserName").text
        account = xml.xpath("//ToUserName").text
        # get unionid and save
        follow(openid)
        if msgType == 'event'
          if event == 'subscribe'
            #  关注事件
            render xml: res_subscribe(account, openid)
            return
          elsif event.downcase == 'templatesendjobfinish'
            # 消息推送结束
            status = xml.xpath("//Status").text
            if status != "success"
              # 消息推送失败，记录日志
              wechat_logger.info "消息推送失败: " + xml.to_s
            end

          end
        end
        render xml: res_xml(account, openid)
    end
    
```

相应的路由配置如下：

```ruby
    # (config/routes.rb)
    get 'wechat/receive', to: 'wechat#receive'
    post 'wechat/receive', to: 'wechat#handle'
```



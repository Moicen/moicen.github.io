---
layout: post
title: 微信公众号服务器接口配置及数字签名验证-ruby版
abstract: 微信公众号服务器接口配置及数字签名验证-ruby版
category: technical
permalink: weixin-server-signature-ruby
author: 木逸辰
tags: [tech, weixin, signature, ruby, rails]
---

### {{ page.title }}

在微信平台开发时，可以配置一个服务器地址，自行处理用户在与服务号的交互事件，也可以添加自定义菜单。

![weixin server](/assets/images/2019-05-06-weixin-server-url.png)

微信在将数据转发到开发者配置的地址时，会发送get请求校验地址是否正确，下面记录下使用ruby进行签名校验的过程。

![weixin signature](/assets/images/2019-05-06-weixin-signature.jpg)

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


最后，微信在配置了服务器地址后，会将服务号中的交互事件全部转发到开发者配置的地址，同时也会废除在开放平台里配置的菜单，需要调用[自定义菜单创建接口](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421141013)另行创建。

菜单参数示例：
```json
{
  "button": [
    {
      "name": "关于我们",
      "sub_button": [
        {
          "type": "view",
          "name": "官方主页",
          "url": "https://my-domain.com",
          "sub_button": []
        },
        {
          "type": "view",
          "name": "服务一",
          "url": "https://my-domain.com/service1",
          "sub_button": []
        },
        {
          "type": "view",
          "name": "加入我们",
          "url": "https://my-doman.com/contract",
          "sub_button": []
        }
      ]
    },
    {
      "name": "企业服务",
      "sub_button": [
        {
          "type": "miniprogram",
          "name": "服务二",
          "url": "https://my-domain.com/service2",
          "appid": "miniprogram1 appid",
          "pagepath": "pages/index/index"
        },
        {
          "type": "miniprogram",
          "name": "服务三",
          "url": "https://my-domain.com/service3",
          "appid": "miniprogram2 appid",
          "pagepath": "pages/index/index"
        }
      ]
    }
  ]
}
```

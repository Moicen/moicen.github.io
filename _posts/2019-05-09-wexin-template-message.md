---
layout: post
title: 微信公众号模板消息推送
abstract: 记录下如何使用微信的模板消息给关注公众号的用户进行推送
category: technical
permalink: wexin-template-push
author: 木逸辰
tags: [tech, weixin, tempate message]
---

### {{ page.title }}

首先打开微信公众号的模板消息功能，从模板库中挑选自己需要的模板，添加进自己的模板库。

![weixin template](/assets/images/2019-05-09-wexin-template.jpg)

如果找不到合适的模板，可以自己提交新的模板，等微信审核通过，就可以使用。

![template new](/assets/images/2019-05-09-weixin-template-new.jpg)

如图可知，微信对模板消息的定义很严格，一般的群发通知公告是不允许的，所以在模板库里也找不到通用的【系统通知】之类的模板。不过微信目前并没有办法判断发送的模板消息是否满足标准，一般的群发是没有问题的，但一些简单的公告性质的模板基本上找不到通用的，比如一个最普通的【系统通知】，是没有的，只有异常、提醒、预警之类的模板。

![template example](/assets/images/2019-05-09-wexin-template-example.jpg)

选好模板之后，就可以使用模板ID来向微信请求，发送模板消息了。这里也是需要配置服务器的，参见前几天的博客[微信公众号服务器接口配置及数字签名验证-ruby版](http://moicen.com/weixin-server-signature-ruby)。


```ruby
  def announce
    # 组织消息内容
    param ={
      template_id: "template_id",
      # 旧版本微信不支持小程序，则可以设置url，点击详情可以进入该url
      url: 'http://moicen.com',
      miniprogram: {
        # 设置小程序的appid和页面路径，可以点击消息下方的详情直接进入小程序
        appid: "miniprogram appid", 
        pagepath: "pages/index/index"
      },
      # 要推送到的用户在当前公众号的openid
      touser: "touser_openid", 
      topcolor: '#FF00000',
      data: {
        first: {
          value: "系统消息播报\n",
          color: "#173177"
        },
        keyword1: {
          value: "2019-05-09"
        },
        keyword2: {
          value: "消息推送",
          color: "#173177"
        },
        remark: {
          value: "备注"
        }
      }
    }
    
    require 'net/http'
    require 'json'
    uri = URI("https://api.weixin.qq.com/cgi-bin/message/template/send?access_token=#{wechat_token}")
    res = Net::HTTP.post(uri, param.to_json, "Content-Type" => "application/json")
    # 将响应记录日志
    logger.info res.as_json
  end
```

从微信上收到的消息推送如下：

![template msg](/assets/images/2019-05-09-wexin-template-msg.jpeg)

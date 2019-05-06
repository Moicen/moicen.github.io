---
layout: post
title: 微信加密数据解密-ruby版
abstract: 微信加密数据解密-ruby版
category: technical
permalink: weixin-encrypt-ruby
author: 木逸辰
tags: [tech, weixin, encrypt, ruby]
---

### {{ page.title }}

微信平台的接口，如果设计敏感数据，微信会将这些敏感数据加密返回。

![weixin server](/assets/images/2019-05-06-wexin-encrypt.jpg)


下面记录下使用ruby解密小程序接口`wx.getUserInfo`返回的加密数据过程。

首先通过`https://api.weixin.qq.com/sns/jscode2session`接口获取`session_key`，这个是解密需要的秘钥，`iv`和`encryptedData`由接口`wx.getUserInfo`返回，原始数据如下：

```ruby
    encryptedData = "zvwk2PxUokScXLvuqAJ9AoF9EcFD2Zgl8Hl2QJnFT81z8cFCX2wNPnc5KERJYOUvq9MJIuw4oa2Kf841L2yXbDWLQhw6sRlTTOCoCyvBSYuFcmsMqDkQ/0m30AKnoFCWHp++VHW6fi82FPfQWYKHAmhCrbLq8cOTVb1ABvDlJixdgpJJDA/5V6IL34eJLay6M/r86sLBhYtOZDWGXxq4AWofJTWvVnPFPu5Zue1ETdBRBIANXYpNxqkhDuhRWXkgVzx7DaO/UXkEbWf3UDfoBXRxQx0dFQI74X1a5BTlNr2ghDbjYcwXhYU2qwClF2thnQ5dPbwwDvLvtlynGlQ+nxHnSWxKbAzJNp8omUt1sPG4Z8VOUG9+6OQXseHG3oeskskhwZFuVZhM164MFxo+D5K968LNA9ew793g1xgPKEwLs+tk+lbMJQaOJvwXq5S6yMFPbSR86VzbjIOsZJ8xKRNsEOGHuD8lFWvjC1GGPOT+xu+s35fE17LGtrlllEIVknrULlkXi7hEvqGMKr6C5g=="

    iv = "5vcM4VWJ9jYVz2GqqV949w=="

    session_key = "qGo4oIyigQwn4kfpSavOzg=="
```

按照微信文档所述，使用`AES-128-CBC`算法解密：

```ruby
    def decode(encryptedData, session_key, iv)
        require 'base64'
        require 'openssl'
        encrypted = Base64.decode64(encryptedData)
        decipher = OpenSSL::Cipher::AES.new("128-CBC")
        decipher.decrypt
        decipher.key = Base64.decode64(session_key)
        decipher.iv = Base64.decode64(iv)
        result = decipher.update(encrypted) << decipher.final
        puts result
    end
```

查看解密完成的数据

![weixin server](/assets/images/2019-05-06-weixin-decrypted.jpg)

`openid`和`unionid`都已获取到
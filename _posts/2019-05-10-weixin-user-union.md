---
layout: post
title: 微信服务号、小程序与项目自有账号体系关联绑定-ruby版
abstract: 如何将微信服务号的关注者、小程序使用者与自有账号体系关联
category: technical
permalink: weixin-user-union-ruby
author: 木逸辰
tags: [tech, weixin, user, union, ruby, rails]
---

### {{ page.title }}

在使用微信平台进行开发时，常常需要把用户的微信号与自有的账号体系关联起来，方便进行全局事件响应和消息推送。

这里就需要有一个唯一标识字段，来唯一确定一个用户。在自有账号体系中，一般会有一个`user_id`，但要跟微信体系打通，就需要使用微信提供的[UnionId机制](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/union-id.html)。

![weixin unionid](/assets/images/2019-05-10-weixin-unionid.jpg)

##### 首先要注册[微信开放平台](https://open.weixin.qq.com/)，并认证开发者资质，才能绑定微信公众号。

##### 在微信开放平台绑定公众号和小程序之后，根据UnionId机制，就可以从公众号和小程序获取到用户的`unionid`，接下来需要将此`unionid`与自有账号中的`user_id`关联。

主流方案是要求用户提供手机号，因为微信要求唯一手机号，因此该手机号即可作为全局唯一标识。这种方式比较简单，但是需要提供短信服务，用于手机号真实性的验证。同时也需要保证用户在微信绑定的手机号与在项目中绑定的一致。

如果没有短信服务，需要另一种方案，来处理不同体系账号的关联。

首先添加一张关联表，用于保存微信用户与自有账号的对应关系。

```ruby
class CreateFollowers < ActiveRecord::Migration[5.2]
  def change
    create_table :followers do |t|
      t.string :openid, unique: true
      t.string :unionid, unique: true
      t.string :mp_openid, unique: true
      t.integer :user_id, unique: true
    end
  end
end
```

`followers`表保存了用户在服务号的`openid`，微信平台`unionid`，小程序中的openid(`mp_openid`)，以及自有账号的`user_id`。

接下来需要将各个id更新到表中，并保证正确的对应关系。

公众号这边，通过用户与公众号交互的事件触发检测，来获取用户的`openid`和`unionid`，用户关注公众号，或者已关注用户在公众号里的操作，在配置完服务器后，微信都会把这些事件提交到开发者所配置的地址，此时就能拿到用户的公众号`openid`和`unionid`。

```ruby
  def handle
    # 解析微信发来的xml参数
    xml = Nokogiri::XML(request.body.read)

    msgType = xml.xpath("//MsgType").text
    event = xml.xpath("//Event").text
    openid = xml.xpath("//FromUserName").text
    account = xml.xpath("//ToUserName").text
    # 保存当前用户的`openid`
    follow(openid)
    # 接下来可以自行针对不同事件进行处理，比如在关注时回复自定义内容
    if msgType == 'event'
      if event == 'subscribe'
        #  关注事件
        render xml: res_subscribe(account, openid)
        return
      elsif event.downcase == 'templatesendjobfinish'
        # 消息推送事件结束，判断是否成功并记录日志
        status = xml.xpath("//Status").text
        if status != "success"
          # 消息推送失败，记录日志
          wechat_logger.info "消息推送失败: " + xml.to_s
        end

      end
    end
    render xml: res_xml(account, openid)
  end
  
  def follow(openid)
  
      require 'net/http'
      # 调用接口根据openid获取unionid信息
      url = URI("https://api.weixin.qq.com/cgi-bin/user/info?access_token=#{wechat_token}&openid=#{openid}&lang=zh_CN")
      res = JSON.parse(Net::HTTP.get(url))
      unionid = res["unionid"]
      if unionid.present?
        # 将订阅者的openid和unionid保存进数据库
        follower = Follower.find_by_unionid(unionid)
        if follower.nil?
          Follower.create!({openid: openid, unionid: unionid})
        else
          follower.update!({openid: openid})
        end
      end
  
    end
  
```

小程序这边，用于进入小程序之后，即可拿到`openid`，但是要获取其他信息，必须用户授权，或者关注了同主体公众号。
在小程序首页，提供授权按钮让用户授权，授权完毕，调用微信的登录方法，获取用户信息和临时认证`code`，发送到后台。
此时可以设置`withCredentials=true`，可以拿到敏感数据信息（`openid`、`unionid`、手机号等）。

```js
// /pages/index/index.js
const app = getApp()
const { warn, request } = require("../../utils/util.js")

const identify = (instance, params, server) => {
  request({
    url: server + "/wechat/identify",
    method: 'POST',
    data: params,
    success: function (res) {
      let { success, result } = res.data;
      if (success) {
        app.globalData.session = result
        wx.setStorageSync("session", result)
        wx.redirectTo({
          url: "../login/login"
        })
      } else {
        warn(res.data.message)
      }
    }
  })
}

const login = (instance) => {
  let { server } = app.globalData;

  wx.login({
    success: function (res) {
      let { code, errMsg } = res;
      if (code) {
        // 获取code成功，则读取用户信息
        wx.getUserInfo({
          withCredentials: true,
          success: function (res) {
            let data = {
              code: code,
              encryptedData: res.encryptedData,
              iv: res.iv
            }
            // 请求后台，换取openid和unionid
            identify(instance, data, server);
          }
        })
      } else {
        console.log('获取用户登录态失败！' + res.errMsg);
      }
    }
  })
}

Page({
  data: {
    canIUse: wx.canIUse('button.open-type.getUserInfo'),
  },
  bindGetUserInfo(e) {
    login(this)
  },
  onLoad: function () {

    let instance = this;
    wx.getSetting({
      success: function (res) {
        // 已授权
        if (res.authSetting["scope.userInfo"]) {
          // 从缓存读取用户数据
          let user = wx.getStorageSync("user")
          if (user) {
            app.globalData.user = user;
            wx.redirectTo({
              url: "../login/login"
            })
          } else {
            login(instance)
          }
        }
    })

  }
})

```
后端拿到临时凭证`code`和用户信息，调用微信接口`code2session`换取登录态`session_key`和用户的`openid`、`unionid`数据。

```ruby
  def identify
    require 'net/http'
    mp = Rails.application.config_for(:weixin)["miniprogram"]
    appid = mp["appid"]
    appSecret = mp["appsecret"]
    uri = URI("https://api.weixin.qq.com/sns/jscode2session?appid=#{appid}&secret=#{appSecret}&js_code=#{params[:code]}&grant_type=authorization_code")
    res = Net::HTTP.get(uri)
    wechat_logger.info "code2session data: #{res}"
    result = JSON.parse(res)

    code = result['errcode']

    if code == -1
      render json: failure('系统繁忙，请稍候再试')
    elsif code == 40029
      render json: failure('认证身份无效')
    elsif code == 45011
      render json: failure('您操作太快了，请稍后再试')
    else
      render json: success(decode(params[:encryptedData], params[:iv], result))
    end

  end
```

如果此时不符合UnionId机制，则只能获取到`openid`，无法获取到`unionid`，就需要解密用户信息中的加密数据，具体参见[微信加密数据解密-ruby版](http://moicen.com/weixin-encrypt-ruby)。

拿到用户的`openid`和`unionid`之后，返回给小程序端，接下来小程序进入登录页面，用户输入用户名密码登录时，则可以把前面获取到的`openid`和`unionid`一起发送给后端，将自有账号与小程序账号关联起来。

这里要注意的是，如何处理服务号那边的`openid`数据、已关联过的数据，以及，如果用户使用同一个微信小程序登录不同的账号，如何处理，这个就需要根据业务来决定，我这里只是简单地将其关联信息更新。具体逻辑如下：

- 小程序登录和公众号事件，都先判断`unionid`是否有，所以`followers`表中的数据必然都有`unionid`字段
- 如果小程序登录，则还有`user_id`
- 如果公众号事件，则还有`openid`
- 根据`unionid`查询
  1. `unionid`有记录，对比`user_id`
    1. 如果`user_id`不同，则需要先将当前`user_id`对应的数据的`user_id`和`mp_openid`清除掉，再将本条记录的`user_id`更新成当前用户
     2. 如果`user_id`相同，则判断`openid`是否存在，若不存在，则读取并更新
  2. `unionid`无记录，则根据`user_id`查,并根据`unionid`读取新的`openid`，
    1. 如果有记录，则更新对应的`unionid`和`openid`
    2. 如果没有记录，添加新记录

```ruby
  class AuthenticateUser
    prepend SimpleCommand
  
    def initialize(username, password, mp_openid = nil, unionid = nil)
      @username = username
      @password = password
      @mp_openid = mp_openid
      @unionid = unionid
    end
  
    def call
      JsonWebToken.encode(user_id: user.id) if user
    end
  
    attr_accessor :username, :password
  
    def user
      user = VUser.find_by_username(username)
      if user&.authenticate(password)
        begin
          user_id = user[:id]
          if @unionid.present?
            follower = Follower.find_by_unionid(@unionid)
            if follower.present?
              openid = follower[:openid].nil? ? get_openid : follower[:openid]
              if follower[:user_id] != user_id
                prev = Follower.find_by_user_id(user_id)
                # 清除旧数据
                if prev.present?
                  prev.update(user_id: nil, mp_openid: nil)
                end
                follower.update(user_id: user_id, mp_openid: @mp_openid, openid: openid)
              else
                if follower[:openid].nil? && openid.present?
                  follower.update(mp_openid: @mp_openid, user_id: user_id, openid: openid)
                end
              end
            else
              follower = Follower.find_by_user_id(user_id)
              if follower.present?
                follower.update(unionid: @unionid, openid: get_openid, mp_openid: @mp_openid)
              else
                Follower.create(user_id: user_id, mp_openid: @mp_openid, unionid: @unionid, openid: get_openid)
              end
            end
          end
        rescue => error
          ApplicationController.new.wechat_logger.error "用户微信信息关联发生错误：#{error.as_json}"
        end
        user
      else
        errors.add("登录失败：", '用户名或密码错误！')
      end
    end
  
  end
```

如果这里用户直接进入小程序，在数据库中并未保存在公众号那边的相关信息（比如旧数据），那就需要另外调用微信接口，根据小程序处得到的`unionid`来获取用户在公众号的`openid`。
因为微信未提供直接根据`unionid`获取`openid`的接口，因此只能读取公众号所有关注者`openid`，循环调接口读取对应的`unionid`，与已有数据判断。

```ruby

  def get_openid
    openid = nil
    get_followers.each do |fo|
      unionid = get_unionid_by_openid(fo)
      if @unionid == unionid
        openid = fo
        break
      end
    end
    openid
  end

  def get_followers
    openIds = []
    require 'net/http'
    uri = URI("https://api.weixin.qq.com/cgi-bin/user/get?access_token=#{wechat_token}&next_openid=")
    res = JSON.parse(Net::HTTP.get(uri))
    wechat_logger.info "Get followers: #{JSON.generate(res)}"
    if res['data']
      openIds = res['data']['openid']
    else
      if [41001, 40001, 40014, 42001].index(res['errcode']) >= 0
        # token problem
        @@wechat = {"access_token" => nil, "expired_time" => nil}
      end
    end
    openIds
  end
    
  def get_unionid_by_openid(openid)
    url = URI("https://api.weixin.qq.com/cgi-bin/user/info?access_token=#{wechat_token}&openid=#{openid}")
    res = JSON.parse(Net::HTTP.get(url))
    res["unionid"]
  end
  
```


以上就完成了整个微信平台账号体系与自有账号的关联，可以针对性地给不同产品用户发送消息，比如公众号推送、小程序推送等等。模板消息的发送参见上一篇[微信公众号模板消息推送](http://moicen.com/wexin-template-push)。

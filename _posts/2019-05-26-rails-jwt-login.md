---
layout: post
title: 使用jwt做用户登录及校验系统-rails版
abstract: 使用jwt做用户登录及校验系统-rails版
category: technical
permalink: rails-jwt-login
author: 木逸辰
tags: [tech, rails, jwt, login, authenticate]
---

### {{ page.title }}


`JsonWebToken(jwt)`目前最流行的会话保持机制，之前的rails项目里也使用了，现在分析一下它的完整流程和机制。

先看看代码里是如何使用的。我这里使用的是一个叫`jwt`的gem。

#### 从登录开始，拿到username和password之后，调用方法校验，并在成功后返回一个token：

```ruby
def login

  command = AuthenticateUser.call(params[:username], params[:password])

  if command.success?
    token = command.result
    render json: success(token), status: :ok
  else
    render json: error(command.errors), status: :unauthorized
  end
end

```

这个`AuthenticateUser`类很简单，就是调用了一下`JsonWebToken`的编码方法，使用用户名查询用户，并对密码进行校验，成功后则将`user_id`进行编码，然后返回：

```ruby
class AuthenticateUser
  prepend SimpleCommand

  def initialize(username, password)
    @username = username
    @password = password
  end

  def call
    JsonWebToken.encode(user_id: user.id) if user
  end

  attr_accessor :username, :password

  def user
    user = User.find_by_username(username)
    if user&.authenticate(password)
    else
      errors.add("登录失败：", '用户名或密码错误！')
    end
  end

end
```

这个`JsonWebToken`类也是自己实现的，主要是把对`JWT`的调用封装了一下，便于使用：

```ruby
class JsonWebToken
  class << self
    def encode(payload, exp = 24.hours.from_now)
      payload[:exp] = exp.to_i
      JWT.encode(payload, Rails.application.secret_key_base)
    end

    def decode(token)
      begin
        body = JWT.decode(token, Rails.application.secret_key_base)[0]
        HashWithIndifferentAccess.new body
      rescue
        nil
      end
    end
  end
end
```

`JWT`模块对外提供一个`encode`方法和一个`decode`方法，用来编码和解码数据，生成唯一标识token。

![jwt](/assets/images/2019-05-26-rails-jwt.png)

`JWT`的编码机制实现了 [jwt](https://jwt.io/) 规范

![jwt encode](/assets/images/2019-05-26-jwt-encode.png)

生成的token数据包含三个部分：`header`、`payload`、`signature`，并且自己维护了一套过期机制：

![jwt](/assets/images/2019-05-26-jwt.png)

#### 用Wireshark看看当前项目登录后生成的token：

![token](/assets/images/2019-05-26-login-token.png)

在后续请求中token被放到header中的Authorization字段

![authorization](/assets/images/2019-05-26-request-header-token.png)

server端添加一个过滤器，对所有需要权限校验的接口进行过滤：
```ruby
  def authenticate_request
    @current_user = AuthorizeRequest.call(request.headers).result
    if @current_user.nil?
      render json: failure("您没有权限进行此操作"), status: :unauthorized
    end
  end
```

`AuthorizeRequest`类调用`JsonWebToken`的`decode`方法，从token中获取`user_id`，如果成功，则使用该`user_id`从数据库读取数据，并返回，否则即返回401:

![auth request](/assets/images/2019-05-26-auth-req.png)

#### 接下来看看`jwt`的过期机制。先进行两次连续登录操作，模拟不同浏览器登录：

![parall login](/assets/images/2019-05-26-parall-login.png)

获得两个token，直接使用第一次登录后的token调用接口，执行成功，说明后续登录不会挤掉之前的会话。

将`jwt`过期时间设置为1分钟，用于测试，注意，这里的`1.minutes.from_now`是`active_support`提供的功能

```ruby
def encode(payload, exp = 1.minutes.from_now)
  payload[:exp] = exp.to_i
  JWT.encode(payload, Rails.application.secret_key_base)
end
```

![jwt expire](/assets/images/2019-05-26-jwt-exp.png)

1分钟之后，再次使用之前的两个token调用接口，返回401，说明`jwt`确实是自己维护了一套过期机制，查看它的代码：

![jwt verify](/assets/images/2019-05-26-jwt-verify.png)

可以看到这里有一系列的校验`verify_aud verify_expiration verify_iat verify_iss verify_jti verify_not_before verify_sub`，校验过期的代码：

```ruby
def verify_expiration
  return unless @payload.include?('exp')
  raise(JWT::ExpiredSignature, 'Signature has expired') if @payload['exp'].to_i <= (Time.now.to_i - exp_leeway)
end
```

可以看到`JWT`实际是使用token中解码得来的`exp`属性，与当前时间比较来判断是否过期。因此如果在`encode`的时候不设置`exp`，则对应生成的token就一直不会过期。

#### `JWT`实际只是提供了一个编码和解码的功能，而且自身不保存任何状态，因此如果登录拿到token之后，只要不重新登录，即便服务重启，之前的 token依旧有效。如将过期时间改回24小时，重新登录，然后立即重启服务器

![jwt restart expire](/assets/images/2019-05-26-jwt-restart-exp.png)

使用刚才登录得到的token调用接口，token依旧有效。

那么如果要实现服务重启后token失效，则需要自己另外处理，我这里是使用了一个`@@tokens`变量保存所有的token，做解码之前会先校验当前token是否存在与这个变量里，因此如果服务重启 ，则变量清空，之前的token失效。

![jwt](/assets/images/2019-05-26-token-save.png)
![jwt](/assets/images/2019-05-26-saved-token-check.png)

当然也可以把这些token保存在redis中，使用redis的expiration功能。

因此，如果要实现一个更完善的session机制，则需要自己实现对`token`的保存，另外还需要将session、jwt的过期时间与发送给浏览器的`ETag`header属性保持一致，也需要自己处理。


---
layout: post
title: 使用Resteasy代替Spring Security做权限校验
abstract: 使用Resteasy代替Spring Security做权限校验
category: technical
permalink: resteasy-filter
author: 木逸辰
tags: [tech, resteasy, spring, jwt, login, authenticate, filter]
---

### {{ page.title }}


上一篇中，我们使用了Spring Security为我们的系统提供权限控制。Spring Security功能非常强大，而且与Spring系列结合紧密，一般项目里使用起来也是非常方便的。
Spring Security提供了一个`UserDetailsService`接口和`UserDetails`类，要更好使用Spring Security，就必须按照它定义的标准来定义我们的用户数据，这在实际项目中常常是不太方便的。于是打算使用Resteasy替换Spring Securty。




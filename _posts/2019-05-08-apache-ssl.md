---
layout: post
title: Apache中SSL证书相关配置
abstract: Apache中SSL证书相关配置
category: technical
permalink: apache-ssl-conf
author: 木逸辰
tags: [tech, apache, ssl, ca]
---

### {{ page.title }}

在Apache配置HTTPS服务，主要有以下几个配置项：


![apache ssl conf](/assets/images/2019-05-08-apache-ssl-conf.jpg)


#### 第一组是服务器在https通信的时候出示给客户端的证书
1. `SSLCertificateFile(.pem/.crt)`: 证书文件(`2_domain.crt`)
2. `SSLCertificateKeyFile(.key)`: 密钥文件(`3_domain.key`)

#### `SSLCertificateChainFile(.crt)`: 证书链文件/路径(`1_root_bundle.crt`)，由直接签发服务器证书的CA证书开始，按证书链顺序回溯，一直到根CA的证书结束，这一系列的CA证书(PEM格式)就构成了服务器的证书链。这个证书链将被与服务器证书一起发送给客户端。这有利于避免在执行客户端认证时多个CA证书之间出现混淆或冲突。

#### `SSLCACertificatePath/SSLCACertificateFile`是用于验证他人提交过来的证书的CA证书路径/文件，即双向认证。

通常的SSL认证，比如访问一个网站，浏览器作为客户端，验证网站提交过来的证书，但网站不验证访问客户的身份。
如果服务端那边也要求你提交你的个人证书，那就是双向认证了，彼此确认身份。
服务端验证他人提交的证书和浏览器验证服务端提交的证书是一样的。在双向ssl认证时，他人出示证书给apache服务器，apache用此处配置的CA证书认证对方交过来的证书。此时服务器可以看作是客户端，跟浏览器一样，浏览器会用浏览器内置的ca证书验证https网站提交过来的证书

![browser-ca](/assets/images/2019-05-08-browser-ca.jpg)

如图所示都是权威机构，这些机构签发的证书，默认有效，即如果 服务端的证书是这一堆机构里面签发的，就有效。那么反过来，双向认证的话，提交给服务端的证书，服务端那边也得有个CA机构的证书列表，就是这里要配置的。openssl默认提供一套标准化的CA机构列表

这里也可以自己配置，比如内部机构有一整套自签名的生态，就要自己配置自己为CA机构，自己内部生成一张根证书，用这张根证书签名就行了。这样自己就是自己的CA，内部统一用自己的自签名证书，就不需要找第三方权威机构签名了（省钱）

#### SSLCACertificateRevocation*，是吊销的证书列表

#### SSLVerifyClient: 设置是否需要验证客户端的证书，
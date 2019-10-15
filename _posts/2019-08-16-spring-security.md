---
layout: post
title: Spring Security使用
abstract: 记录下如何使用Spring Security
category: technical
permalink: spring-security
author: 木逸辰
tags: [tech, spring, security, login, authenticate, filter]
---

### {{ page.title }}

Spring Security提供了一整套的用户授权鉴权机制，在Spring全家桶中使用起来也十分方便。

首先添加一个`security`的配置文件

![spring security config](/assets/images/2019-08-16-spring-config.jpg)

这个自定义的`SecurityJavaConfig`需要继承`WebSecurityConfigurerAdapter`类，并重写`configure`方法，在`configure`方法中实现对请求的过滤和校验。

![spring security config](/assets/images/2019-08-16-spring-security-config-2.jpg)

`addFilter**`即在指定的位置添加自己定义的filter：

![login filter](/assets/images/2019-08-16-spring-security-filter.jpg)

我这里定义了三个filter，`LoginAuthenticationFilter`用来处理登录，`JwtAuthenticationFilter`处理jwt校验，`WxAuthenticationFilter`处理微信相关的token校验。

1. `LoginAuthenticationFilter`定义。主要就是读取请求参数里的`username`和`password`，可以直接调用默认的`UsernamePasswordAuthenticationFilter`校验，基本不需要做太多自定义的事。：


  ```java

  package com.moicen.spring.rest.filter;

  import com.moicen.spring.rest.common.AuthenticationBean;
  import com.fasterxml.jackson.databind.ObjectMapper;
  import org.springframework.http.MediaType;
  import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
  import org.springframework.security.core.Authentication;
  import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
  import org.springframework.util.Assert;

  import javax.servlet.http.HttpServletRequest;
  import javax.servlet.http.HttpServletResponse;
  import java.io.IOException;

  public class LoginAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

     @Override
     public void afterPropertiesSet() {
        Assert.notNull(getAuthenticationManager(), "authenticationManager must be specified");
        Assert.notNull(getSuccessHandler(), "AuthenticationSuccessHandler must be specified");
        Assert.notNull(getFailureHandler(), "AuthenticationFailureHandler must be specified");
     }

     @Override
     public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) {

        ObjectMapper mapper = new ObjectMapper();
        UsernamePasswordAuthenticationToken authenticationToken = null;

        try {
           AuthenticationBean authenticationBean = mapper.readValue(getRequestPostStr(request), AuthenticationBean.class);
           authenticationToken = new UsernamePasswordAuthenticationToken(
                   authenticationBean.getUsername(), authenticationBean.getPassword());
        } catch (IOException e) {
           authenticationToken = new UsernamePasswordAuthenticationToken(null, null);
        }
        setDetails(request, authenticationToken);
        return this.getAuthenticationManager().authenticate(authenticationToken);
     }

     private static byte[] getRequestPostBytes(HttpServletRequest request)
             throws IOException {
        int contentLength = request.getContentLength();
        if (contentLength < 0) {
           return new byte[0];
        }
        byte[] buffer = new byte[contentLength];
        int i = 0;
        while (i < contentLength) {
           int cursor = request.getInputStream().read(buffer, i, contentLength - i);
           if (cursor == -1) {
              break;
           }
           i += cursor;
        }
        return buffer;
     }

     private static String getRequestPostStr(HttpServletRequest request)
             throws IOException {
        byte[] buffer = getRequestPostBytes(request);
        String charEncoding = request.getCharacterEncoding();
        if (charEncoding == null) {
           charEncoding = "UTF-8";
        }
        return new String(buffer, charEncoding);
     }

  }

  ```

2. `JwtAuthenticationFilter`

![jwt filter](/assets/images/2019-08-16-jwt-filter.jpg)


这里使用`@Value`从配置文件`resources/application.yaml`中读取`jwt`相关的配置。
重写`doFilterInternal`方法来做校验，主要就是通过`jwt`读取`userId`，然后从数据库读取用户信息，然后继续重用`UsernamePasswordAuthenticationToken`来保存`UserDetails`。注意这里我使用了自定义的`UserDetailsService`，主要是`Spring Security`自带的`UserDetailsService`不太顺手。一般的系统里都会有自己定义的一套权限系统，跟Spring Security默认的结构不太一样。

  ```java
  @Service
  public class WebUserDetailsService {

     @Autowired
     private UserService userService;

     public OcrUser loadUserByUsername(String username) {

        OcrUser user = userService.findByUsername(username);

        if (user == null) {
           throw new UsernameNotFoundException(username + " not found");
        }

        return user;
     }
  }
  ```
    
同时另外定义了一个`WebAuthenticationProvider`类，用于做实际的鉴权操作：

![jwt filter](/assets/images/2019-08-16-auth-provider.jpg)

注意这里因为密码加密时使用了随机的`salt`，因此每次生成的密码是不一样的，不能直接做相等比较，需要使用`encoder.matches`方法来对比。
`AuthenticationManager`使用`ProviderManager`来管理和使用provider，`AuthenticationProvider`需要重写`supports`方法来表明自己支持哪一种`Authentication`

![provider manager](/assets/images/2019-08-16-provider-manager.jpg)

在自定义的`SecurityJavaConfig`中配置使用上述provider：

![provider use](/assets/images/2019-08-16-auth-provider-use.jpg)

`permissiveRequest`和`setPermissiveUrl`可以用来设置规则让不需要校验的请求直接通过。这个是使用正则表达式对URL做匹配，对简单的规则比较方便，如果规则比较复杂那么写起来就很麻烦。因此这里使用另一种更简单灵活的方式：重写`shouldNotFilter`方法，在这里可以做任何方式的校验，只要最终返回一个`boolean`就可以了，比如我这里设置包含`/wx/`的路由都不需要走这个校验。
        
  ```java
  @Override
  protected boolean shouldNotFilter(HttpServletRequest request) {
     String uri = request.getRequestURI();
     return uri.contains("/wx/");
  }
  ```

3. `WxAuthenticationFilter`。因为我这里跟微信交互的比较多，而且校验方式不一样，因此额外写了一个专门处理微信相关请求的filter。整体跟`JwtAuthenticationFilter`基本一致，只是`shouldNotFilter`正好相反，另外从`jwt`读取用户信息时做了额外判断。


接下来定义一个`JwtUtil`类来专门处理`jwt`相关

![jwt util](/assets/images/2019-08-16-jwt-util.jpg)


最后测试以上filter

1. `LoginAuthenticationFilter`

  ```java
     @Test
     public void loginTest() throws Exception {
        Response response = given().contentType(ContentType.JSON)
                .body("{\"username\": \"bar_user\", \"password\": \"bar_pwd\"}")
                .post("/login");
        ObjectMapper objectMapper = new ObjectMapper();
        HttpWebResponse res = objectMapper.readValue(response.asString(), HttpWebResponse.class);
        System.out.println("----------token: " + res.getResult());
        assertTrue(res.isSuccess());
        assertEquals("bar_user", JwtUtil.getIdentifier(res.getResult().toString()));
     }
  ```

2. `JwtAuthenticationFilter`

  ```java
  @Test
  public void jwtTest() throws Exception {
      String token = "eyJhbGciOiJIUzI1NiIsImN0eSI6ImFwcGxpY2F0aW9uL2pzb24ifQ==.eyJpZGVudGlmaWVyIjoiYmFyX3VzZXIiLCJ0aW1lc3RhbXAiOjE1Njc2OTM0OTg5NjMsInR5cGUiOiJVU0VSTkFNRSJ9.lmZUE2XwVVqF4zdimrI8fVP6wEosxdsgqxYZQuDBh3c=";
      Response response = given()
            .header("Authorization", "Bearer " + token)
            .contentType(ContentType.JSON)
            .get("/users/current");
      ObjectMapper objectMapper = new ObjectMapper();
      System.out.println("----------------response: " + response.asString());
      HttpWebResponse res = objectMapper.readValue(response.asString(), HttpWebResponse.class);
      assertTrue(res.isSuccess());
      assertNotNull(res.getResult());
      SpringUser user = objectMapper.convertValue(res.getResult(), SpringUser.class);
      assertEquals("bar_user", user.getUsername());
      assertEquals("bar", user.getOpenid());
  }
  ```
    
3. `WxAuthenticationFilter`

  ```java
  @Test
  public void jwtOpenidTest() throws Exception {
      String token = JwtUtil.encode("foo");
      Response response = given()
            .header("Authorization", "Bearer " + token)
            .contentType(ContentType.JSON)
            .get("/photos/wx/query");
      System.out.println("-----------response: " + response.asString());

      HttpWebResponse res = objectMapper.readValue(response.asString(), HttpWebResponse.class);
      assertTrue(res.isSuccess());

  }
  ```


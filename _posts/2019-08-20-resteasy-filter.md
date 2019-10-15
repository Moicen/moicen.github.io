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


首先在POM文件里添加引用：

  ```xml
  <dependency>
      <groupId>javax.ws.rs</groupId>
      <artifactId>javax.ws.rs-api</artifactId>
      <version>2.1</version>
  </dependency>
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>resteasy-spring-boot-starter</artifactId>
      <version>${resteasy.springboot.version}</version>
      <scope>runtime</scope>
  </dependency>


  <!-- Adding RESTEasy support to Bean Validations -->
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>resteasy-validator-provider</artifactId>
      <version>${resteasy.version}</version>
  </dependency>
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>resteasy-core</artifactId>
      <version>${resteasy.version}</version>
  </dependency>
  <!-- JAXB support -->
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>resteasy-jaxb-provider</artifactId>
      <version>${resteasy.version}</version>
  </dependency>
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>resteasy-core-spi</artifactId>
      <version>${resteasy.version}</version>
  </dependency>
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>resteasy-client</artifactId>
      <version>${resteasy.version}</version>
  </dependency>
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>resteasy-jackson2-provider</artifactId>
      <version>${resteasy.version}</version>
  </dependency>
  <dependency>
      <groupId>org.jboss.resteasy</groupId>
      <artifactId>jose-jwt</artifactId>
      <version>${resteasy.version}</version>
  </dependency>
  ```

添加一个全局异常处理类：

  ```java

  package com.moicen.resteasy.configuration;

  import com.moicen.resteasy.rest.common.HttpWebResponse;
  import org.springframework.stereotype.Component;

  import javax.ws.rs.core.MediaType;
  import javax.ws.rs.core.Response;
  import javax.ws.rs.ext.ExceptionMapper;
  import javax.ws.rs.ext.Provider;

  @Component
  @Provider
  public class GlobalExceptionMapper implements ExceptionMapper<Exception> {
      public Response toResponse(Exception exception) {

          System.out.println("********************[START OF GLOBAL EXCEPTION MAPPER]********************");
          System.out.println(exception);
          System.out.println("********************[END OF GLOBAL EXCEPTION MAPPER]**********************");

          return Response
                  .serverError()
                  .entity(new HttpWebResponse(false, exception.getMessage()))
                  .type(MediaType.APPLICATION_JSON)
                  .build();
      }
  }
  ```

使用`ContainerRequestFilter`：

  ```java
  package com.moicen.resteasy.rest.filter;

  import com.moicen.resteasy.data.entity.ResteasyUser;
  import com.moicen.resteasy.rest.context.UserContext;
  import com.moicen.resteasy.rest.util.JwtUtil;
  import org.jboss.resteasy.core.ResourceMethodInvoker;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.stereotype.Component;

  import javax.annotation.security.DenyAll;
  import javax.annotation.security.PermitAll;
  import javax.servlet.http.HttpServletRequest;
  import javax.ws.rs.container.ContainerRequestContext;
  import javax.ws.rs.container.ContainerRequestFilter;
  import javax.ws.rs.core.Context;
  import javax.ws.rs.core.UriInfo;
  import javax.ws.rs.ext.Provider;
  import java.io.IOException;
  import java.lang.reflect.Method;

  import static com.moicen.resteasy.configuration.GlobalConstants.ACCESS_DENIED;
  import static com.moicen.resteasy.configuration.GlobalConstants.ACCESS_FORBIDDEN;
  import static com.moicen.resteasy.configuration.GlobalConstants.AUTHENTICATION_SCHEME;
  import static com.moicen.resteasy.configuration.GlobalConstants.AUTHORIZATION_PROPERTY;

  @Provider
  @Component
  public class MyContainerRequestFilter implements ContainerRequestFilter {

      @Context
      UriInfo info;

      @Context
      HttpServletRequest request;

      @Value("${jwt.name}")
      private String tokenName;

      @Value("${jwt.prefix}")
      private String tokenPrefix;

      @Value("${jwt.expire}")
      private long tokenExpire;

      @Autowired
      private UserContext userContext;

      @Override
      public void filter(ContainerRequestContext requestContext) throws IOException {

          final String method = requestContext.getMethod();
          final String path = info.getPath();
          final String address = request.getRemoteAddr();

          System.out.printf("Request %s %s from IP %s", method, path, address);
          System.out.println();

          if (path.contains("/actuator/")) return;

          // 不能在PreMatching中使用，此时还是null
          ResourceMethodInvoker methodInvoker = (ResourceMethodInvoker) requestContext.getProperty("org.jboss.resteasy.core.ResourceMethodInvoker");

          Method handler = methodInvoker.getMethod();
          Class<?> clazz = handler.getDeclaringClass();
          if (clazz.isAnnotationPresent(PermitAll.class) || handler.isAnnotationPresent(PermitAll.class)) {
              return;
          }
          if (clazz.isAnnotationPresent(DenyAll.class) || handler.isAnnotationPresent(DenyAll.class)) {
              requestContext.abortWith(ACCESS_FORBIDDEN);
              return;
          }

          final String token = requestContext.getHeaderString(AUTHORIZATION_PROPERTY).replaceFirst(AUTHENTICATION_SCHEME, "");
          if (token.isEmpty() || token.isBlank()) {
              requestContext.abortWith(ACCESS_DENIED);
              return;
          }

          ResteasyUser user = userContext.getUser(token);
          if (!JwtUtil.validate(token, tokenExpire * 1000, user)) {
              requestContext.abortWith(ACCESS_DENIED);
              return;
          }
      }


  }

  ```

此时则可以使用`javax`提供的`PermitAll`、`DenyAll`、`RolesAllowed`注解，直接在类和方法上标注访问权限。当然也可以直接在上面的`filter`里针对路径做处理。

另外需要注意的是，Resteasy需要使用`@ApplicationPath`标注Application：

![resteasy](/assets/images/2019-08-20-resteasy-path.jpg)

另外需要在`resources/application.yml`里面配置resteasy的注册项，这样resteasy才会扫描相应的类：

  ```yaml

  resteasy:
    jaxrs:
      app:
        registration: property
        classes: com.moicen.resteasy.ResteasyApplication
  ```

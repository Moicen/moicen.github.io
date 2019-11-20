---
layout: post
title: jaxrs集成Swagger
abstract: jaxrs集成Swagger
category: technical
permalink: jaxrs-swagger
author: 木逸辰
tags: [tech, jaxrs, resteasy, swagger, openapi]
---

### {{ page.title }}


**本文中所用项目地址：[jaxrs-swagger-sample](https://github.com/alchemy-studio/jaxrs-swagger-sample)**

[Swagger](https://swagger.io/)是目前最流行也是最方便的API工具，在Java开发中使用起来也十分方便。

如果使用了Spring全家桶开发，那么只需在依赖中加入springfox提供的`springfox-swagger2`和`springfox-swagger-ui`即可，它会自动提取使用了`@RequestMapping`注解的方法，并在`spring-boot:run`的时候启动一个swagger-ui的服务，除了必要的openAPI注解，基本不需要什么额外工作。

但是spring-fox只能识别spring提供的map注解，[不支持识别`@Path`等`javax.ws.rs`提供的映射注解](https://github.com/springfox/springfox/issues/2650)，因此我这里需要一些额外的工作（其实就是按照openAPI的标准用法来用）来集成swagger。

用[Spring Initializr](https://start.spring.io/)创建一个maven项目（我这里直接用IDEA创建）：

![create jaxrs swagger sample](/assets/images/2019-10-24-create-jaxrs-swagger-sample.png)

在`pom.xml`中添加相关依赖和插件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cn.alchemystudio.swagger</groupId>
    <artifactId>jaxrs-swagger-sample</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>jaxrs-swagger-sample</name>
    <description>Sample project for integrating swagger with jaxrs</description>

    <properties>
        <java.version>11</java.version>
        <swagger.version>2.0.10</swagger.version>
        <resteasy.version>4.3.1.Final</resteasy.version>
        <resteasy.springboot.version>4.3.1.Final</resteasy.springboot.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- jaxrs 依赖 我这里使用了resteasy-->
        <dependency>
            <groupId>javax.ws.rs</groupId>
            <artifactId>javax.ws.rs-api</artifactId>
            <version>2.1.1</version>
        </dependency>
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-core</artifactId>
            <version>${resteasy.version}</version>
        </dependency>
        <dependency>
            <groupId>org.jboss.resteasy</groupId>
            <artifactId>resteasy-spring-boot-starter</artifactId>
            <version>${resteasy.springboot.version}</version>
            <scope>runtime</scope>
        </dependency>

        <!-- swagger 依赖 -->
        <dependency>
            <groupId>io.swagger.core.v3</groupId>
            <artifactId>swagger-jaxrs2</artifactId>
            <version>${swagger.version}</version>
        </dependency>

        <dependency>
            <groupId>io.swagger.core.v3</groupId>
            <artifactId>swagger-jaxrs2-servlet-initializer</artifactId>
            <version>${swagger.version}</version>
        </dependency>

        <dependency>
            <groupId>io.swagger.core.v3</groupId>
            <artifactId>swagger-annotations</artifactId>
            <version>${swagger.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- 用于生成openapi数据文件的插件 -->
            <plugin>
                <groupId>io.swagger.core.v3</groupId>
                <artifactId>swagger-maven-plugin</artifactId>
                <version>${swagger.version}</version>
                <configuration>
                    <outputFileName>jaxrs-api</outputFileName>
                    <outputPath>${project.build.directory}/swagger</outputPath>
                    <outputFormat>JSON</outputFormat>
                    <configurationFilePath>${project.basedir}/src/main/resources/openapi.json</configurationFilePath>
                </configuration>
                <executions>
                    <execution>
                        <phase>compile</phase>
                        <goals>
                            <goal>resolve</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

要生成openapi的数据文件，还需要添加一个名为`openapi`的配置文件，可以是`json`或者`yaml`等格式，用来描述所提供的api服务的相关信息：


`src/main/resources/openapi.json`：

```json
{
  "resourcePackages": [
    "cn.alchemystudio.swagger.jaxrs.rest"
  ],
  "prettyPrint": true,
  "cacheTTL": 0,
  "openAPI": {
    "info": {
      "version": 1.0,
      "title": "sample jaxrs api",
      "description": "jaxrs api docs",
      "termsOfService": "https://blog.moicen.com",
      "contact": {
        "email": "i@moicen.com"
      },
      "license": {
        "name": "Apache 2.0",
        "url": "http://www.apache.org/licenses/LICENSE-2.0.html"
      }
    }
  }
}
```

注册JaxRS的Application：
首先是`src/main/java/cn/alchemystudio/swagger/jaxrs/JaxrsSwaggerSampleApplication.java`：

```java
package cn.alchemystudio.swagger.jaxrs;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import javax.ws.rs.ApplicationPath;
import javax.ws.rs.core.Application;

@SpringBootApplication
@ApplicationPath("/rest/")
public class JaxrsSwaggerSampleApplication extends Application {

    public static void main(String[] args) {
        SpringApplication.run(JaxrsSwaggerSampleApplication.class, args);
    }

}
```

然后需要在`src/main/resources/application.properties`文件中配置一下：

```
resteasy.jaxrs.app.registration=property
resteasy.jaxrs.app.classes=cn.alchemystudio.swagger.jaxrs.JaxrsSwaggerSampleApplication
```

接下来我们实一个简单的`/dummy`服务，使用openapi的annotation

`src/main/java/cn/alchemystudio/swagger/jaxrs/rest/DummyService.java`：

```java
package cn.alchemystudio.swagger.jaxrs.rest;

import cn.alchemystudio.swagger.jaxrs.common.HttpWebResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springframework.stereotype.Component;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Component
@Path("/dummy")
@Produces(MediaType.APPLICATION_JSON)
public class DummyService {


    @GET
    @Operation(
            summary = "get dummy",
            description = "Get Dummy",
            tags = {"dummy"}
    )
    @ApiResponse(responseCode = "200", description = "OK")
    public HttpWebResponse get() {
        return new HttpWebResponse<>(true, "dummy");
    }
}
```

这里我们使用了`@Operation`和`@ApiResponse`给方法添加一些描述信息，swagger从这些annotation中读取信息，从而生成api数据。

然后我们就可以使用命令生成openAPI数据文件了：

```bash
$ mvn clean && mvn compile 
```

可以看到生成的api数据文件：

![generate openapi data file](assets/images/2019-10-24-generate-openapi.jpg)

查看一下生成的文件内容，需要格式化一下：

```json
{
    "openapi": "3.0.1",
    "info": {
        "title": "sample jaxrs api",
        "description": "jaxrs api docs",
        "termsOfService": "https://blog.moicen.com",
        "contact": {
            "email": "i@moicen.com"
        },
        "license": {
            "name": "Apache 2.0",
            "url": "http://www.apache.org/licenses/LICENSE-2.0.html"
        },
        "version": "1.0"
    },
    "paths": {
        "/dummy": {
            "get": {
                "tags": ["dummy"],
                "summary": "get dummy",
                "description": "Get Dummy",
                "operationId": "get",
                "responses": {
                    "200": {
                        "description": "OK"
                    }
                }
            }
        }
    },
    "components": {
        "schemas": {
            "HttpWebResponse": {
                "type": "object",
                "properties": {
                    "success": {
                        "type": "boolean"
                    },
                    "result": {
                        "type": "object"
                    },
                    "message": {
                        "type": "string"
                    }
                }
            }
        }
    }
}
```

可以看到，我们刚刚添加的`/dummy`的接口已经生成了。接下来要用生成的数据提供一个web服务，方便使用浏览器访问。为了简单，我这里直接使用已有的docker镜像来启动服务：

```bash
# 拉取官方提供的ui服务镜像
$ docker pull swaggerapi/swagger-ui
# 启动一个容器，使用刚才生成的文件提供数据
$ docker run -itd -v ~/docker-share:/docker-share -e SWAGGER_JSON=/docker-share/swagger/jaxrs-api.json -p 8081:8080 swaggerapi/swagger-ui
```

然后打开浏览器，访问`http://localhost:8081`，即可访问到swagger服务：

![swagger ui](assets/images/2019-10-24-swagger-ui.jpg)


这样我们就搭建了一个最简单的swagger服务，`src/main/resources/openapi.json`中还可以添加很多配置，比如jwt认证、过滤某些不需要展示的接口等。

swagger提供的annotation也很丰富，详细的使用可以参考[官方文档](https://github.com/OAI/OpenAPI-Specification/blob/3.0.1/versions/3.0.1.md)。


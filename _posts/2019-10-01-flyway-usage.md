---
layout: post
title: 使用Flyway做数据库结构的自动化迁移
abstract: 使用Flyway做数据库结构的自动化迁移
category: technical
permalink: flyway-migration-example
author: 木逸辰
tags: [tech, flyway, database]
---

### {{ page.title }}


[Flyway](https://flywaydb.org/)是Java下面执行数据库迁移的工具。可以很方便的进行数据库的版本控制。

在POM文件里添加Flyway的dependency和plugin：

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>${flyway.version}</version>
</dependency>



<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    <version>${flyway.version}</version>
</plugin>
```

安装好之后，在Maven的窗体的Plugins里面可以看到Flyway：

![flyway maven](/asserts/images/2019-10-01-flyway-mvn.jpg)

Flyway从`resource/db/migration`目录读取SQL迁移脚本：

![flyway scripts](/asserts/images/2019-10-01-flyway-scripts.jpg)

双击执行`flyway:info`：

![flyway info](/asserts/images/2019-10-01-flyway-info.jpg)

如果有多个数据库，需要针对不同的数据库分别执行迁移操作，那么可以在POM里配置多个`execution`：

```xml
<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    <version>${flyway.version}</version>
    <executions>
        <execution>
            <id>prod</id>
            <goals>
                <goal>migrate</goal>
            </goals>
            <configuration>
                <url>jdbc:postgresql://localhost:5432/math_ocr</url>
                <user>ocr</user>
                <password/>
            </configuration>
        </execution>
        <execution>
            <id>test</id>
            <goals>
                <goal>migrate</goal>
            </goals>
            <configuration>
                <url>jdbc:postgresql://localhost:5432/math_ocr_test</url>
                <user>ocr_test</user>
                <password/>
            </configuration>
        </execution>
    </executions>
</plugin>
```

此时双击执行mvn的任务会无法找到数据库，因为找不到默认的数据库配置。可以使用命令行执行：

![flyway multi db](/asserts/images/2019-10-01-flyway-multi-db.jpg)

使用`flyway:clean`可以清除掉数据库的所有版本记录（一般不应在生产环境使用）

![flyway clean](/asserts/images/2019-10-01-flyway-clean.jpg)

可以看到，版本记录的状态已变成`pending`

`flyway:migrate`是最常用的命令，用于执行所有未执行的数据库变更。

![flyway migrate](/asserts/images/2019-10-01-flyway-migrate.jpg)

此时数据库的版本记录都是`success`状态了。

Flyway在数据库中使用`flyway_schema_history`表做版本控制：

![flyway version](/asserts/images/2019-10-01-flyway-version.jpg)

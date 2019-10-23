---
layout: post
title: 使用Hibernate映射实体类到数据库
abstract: 使用Hibernate映射实体类到数据库
category: technical
permalink: database-map-with-hibernate
author: 木逸辰
tags: [tech, database, hibernate]
---

### {{ page.title }}


[Hibernate](http://hibernate.org/)能够很方便地将实体类映射为数据库表，并生成相应地SQL语句。

首先给我们定义的实体类加上JPA的注解：

```java
package com.moicen.hibernate;

import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonProperty;
import org.hibernate.annotations.UpdateTimestamp;
import org.hibernate.validator.constraints.Length;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.OneToMany;
import java.util.Date;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

@Entity
public class User {

    @Id
    @Length(min = 36, max = 36)
    private String id;

    public String getId() {
        return id;
    }

    @Column(unique = true)
    private String openid;

    @Column(unique = true)
    private String unionid;

    @Column(unique = true)
    private String distinctId;

    @Column(unique = true)
    private String username;

    @Column
    private String password;

    @Column
    private String nickname;

    @Column
    private String mobile;

    @Column
    @UpdateTimestamp
    private Date lastLogin;


    @Column(nullable = false)
    private boolean disabled;

    public User() {
        this.id = UUID.randomUUID().toString();
    }

}

```

如上（省略了getter、setter和constructor），Hibernate可以直接读取`javax.persistence`的注解，将类和属性映射为数据库表。

下面我们实现一个输出实体类变化的SQL语句的`Migration`类。

```java
package com.moicen.hibernate;

import org.hibernate.boot.Metadata;
import org.hibernate.tool.hbm2ddl.SchemaExport;
import org.hibernate.tool.hbm2ddl.SchemaUpdate;
import org.hibernate.tool.schema.TargetType;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.EnumSet;

/**
 * 改变数据结构用这个输出sql，形成flyway的migration。
 */
public class Migration {

    /**
     * 比对当前关系数据库表，与JPA class的区别，生成变化的sql。·A -> B·
     */
    public static void update(Metadata metadata) {

        SchemaUpdate schemaUpdate = new SchemaUpdate();
        schemaUpdate.setHaltOnError(true);
        schemaUpdate.setFormat(true);
        schemaUpdate.setDelimiter(";");

        SimpleDateFormat formatter = new SimpleDateFormat("yyyyMMdd");
        schemaUpdate.setOutputFile(String.format("/tmp/%s.sql", formatter.format(new Date())));
        schemaUpdate.execute(EnumSet.of(TargetType.STDOUT, TargetType.SCRIPT), metadata);
    }

    /**
     * 不管当前数据库表的结构，根据JPA classes生成一份完整的SQL建库语句。
     */
    public static void create(Metadata metadata) {
        SchemaExport schemaExport = new SchemaExport();
        schemaExport.setHaltOnError(true);
        schemaExport.setFormat(true);
        schemaExport.setDelimiter(";");

        SimpleDateFormat formatter = new SimpleDateFormat("yyyyMMdd");
        schemaExport.setOutputFile(String.format("/tmp/%s.sql", formatter.format(new Date())));
        schemaExport.execute(EnumSet.of(TargetType.STDOUT, TargetType.SCRIPT), SchemaExport.Action.CREATE, metadata);
    }

}

```

如上，这个`Migration`类有两个方法，`create`方法使用`SchemaExport`输出完整的建库语句。`Update`方法对比JPA class和当前数据库结构，生成变化的SQL。
这里需要配置`hibernate.hbm2ddl.auto=none`，这样就只输出SQL语句而不会操作数据库。另外也可以配置`show_sql=true`在控制台输出SQL。

接下来要设置配置文件，并从配置文件中读取配置。Hibernate会读取`resources/META-INF/persistenc.xml`文件中的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.1"
             xmlns="http://xmlns.jcp.org/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd">
    <persistence-unit name="default" transaction-type="RESOURCE_LOCAL">
        <properties>
            <!--jdbc-->
            <property name="javax.persistence.jdbc.driver" value="org.postgresql.Driver" />
            <property name="javax.persistence.jdbc.url" value="jdbc:postgresql://localhost:5432/foo" />
            <property name="javax.persistence.jdbc.user" value="foo" />
            <property name="javax.persistence.jdbc.password" value="" />
            <!--hibernate-->
            <property name="show_sql" value="true"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQL92Dialect"/>
            <property name="hibernate.hbm2ddl.auto" value="none"/>
            <property name="hibernate.temp.use_jdbc_metadata_defaults" value="false"/>
            <property name="hibernate.implicit_naming_strategy" value="org.springframework.boot.orm.jpa.hibernate.SpringImplicitNamingStrategy"/>
            <property name="hibernate.physical_naming_strategy" value="org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy"/>
        </properties>
    </persistence-unit>
</persistence>
```

`jdbc`部分配置数据库连接相关，`hibernate`部分配置hibernate相关。如果只需执行`schemaExport`，也可以不配置`jdbc`部分，而把数据库连接配置到`hibernate`里面：

```xml
<property name="connection.driver_class" value="org.postgresql.Driver"/>
<property name="connection.url" value="jdbc:postgresql://localhost:5432/math_ocr"/>
<property name="connection.username" value="ocr"/>
<property name="connection.password" value=""/>
```


但是如果要执行`schemaUpdate`，则必须配置`jdbc`部分，否则在执行`schemaUpdate`的时候会报`UnsupportedOperationException`异常:

```
java.lang.UnsupportedOperationException: The application must supply JDBC connections
```

另外再定义一个方法来读取配置生成hibernate需要的metadata:

```java
package com.moicen.resteasy.data.ddl;

import org.hibernate.boot.Metadata;
import org.hibernate.jpa.boot.internal.EntityManagerFactoryBuilderImpl;

import java.util.HashMap;

public class MigrationFactory {

    public static Metadata getMetaData(String persistenceUnit) {

        EntityManagerFactoryBuilderImpl builder = new CustomHibernatePersistenceProvider()
                .getEntityManagerFactoryBuilder(persistenceUnit, new HashMap<>());
        builder.build();

        return builder.getMetadata();
    }
}

```

测试代码：

```java
package com.moicen.resteasy.test;

import com.moicen.resteasy.data.ddl.Migration;
import com.moicen.resteasy.data.ddl.MigrationFactory;
import org.junit.Test;

public class MigrationTest {
    @Test
    public void testCreate() {
        Migration.create(MigrationFactory.getMetaData("test"));
    }

    @Test
    public void testUpdate() {
        Migration.update(MigrationFactory.getMetaData("test"));
    }

}

```

这里需要注意的是，hibernate只会正确识别到`main`目录下的`persistence.xml`，如果是写在`test`目录下，hibernate可以读取到，测试可以通过，但是无法输出SQL语句。
如果需要针对多个数据库，可以在`persistence.xml`里添加多个`persistence-unit`，执行的时候指定名称就可以了：

```xml
<persistence-unit name="dev" transaction-type="RESOURCE_LOCAL">
    <!--dev database config-->
</persistence-unit>
<persistence-unit name="test" transaction-type="RESOURCE_LOCAL">
    <!--test database config-->
</persistence-unit>
```


跑一把测试看生成的SQL语句，首先是`create`生成现有的完整SQL：

```sql

    create table user (
       id varchar(36) not null,
        disabled boolean not null,
        distinct_id varchar(255),
        last_login timestamp,
        mobile varchar(255),
        nickname varchar(255),
        openid varchar(255),
        password varchar(255),
        unionid varchar(255),
        username varchar(255),
        primary key (id)
    );

    alter table user 
       add constraint UK_sfnsk2qd4eain0esb3a5qe6yp unique (distinct_id);

    alter table user 
       add constraint UK_jq6t15r1hbsjnl3jpu8x3n8ca unique (openid);

    alter table user 
       add constraint UK_f178j5qnv4c7ivg8rw36x437k unique (unionid);

    alter table user 
       add constraint UK_q9cbi25jeglogs7bc72063ive unique (username);

```

给实体类添加一个字段：

```java
public class User {
    // ...
    @Column
    private int age;
    // ...
}
```

调用`update`方法看生成的迁移SQL：

```sql
    alter table user 
       add column age int4;
```

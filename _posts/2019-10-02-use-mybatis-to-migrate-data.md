---
layout: post
title: 使用MyBatis做数据迁移
abstract: 使用MyBatis做数据迁移
category: technical
permalink: data-migration-with-mybatis
author: 木逸辰
tags: [tech, database, mybatis]
---

### {{ page.title }}


上一篇里面我们使用Flyway做了数据库的版本控制，对于数据库结构的变化迁移是非常方便的，但是对于数据的迁移并不是很方便。因为Flyway是直接执行SQL脚本，如果有数据迁移，那么对数据的操作也需要写在SQL里面，对于一些比较复杂的迁移过程就会很不方便。[MyBatis](https://mybatis.org/)作为一个轻量级的ORM，非常适合做数据迁移。

首先安装MyBatis的依赖：

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>${mybatis.version}</version>
</dependency>

<dependency>
    <groupId>org.mybatis.dynamic-sql</groupId>
    <artifactId>mybatis-dynamic-sql</artifactId>
    <version>1.1.3</version>
</dependency>
```

添加一张表用于记录变更记录；

```sql
create table data_migration_history (
    version varchar(255) not null unique ,
    execute_at timestamp not null
);
```

```java
import java.util.Date;


public class DataMigrationHistory {

    private String version;

    private Date executeAt;


    public String getVersion() {
        return version;
    }

    public void setVersion(String version) {
        this.version = version;
    }

    public Date getExecuteAt() {
        return executeAt;
    }

    public void setExecuteAt(Date executeAt) {
        this.executeAt = executeAt;
    }
}

```


为实体类定义一个映射接口：

```java
@Mapper
public interface DataMigrationHistoryMapper {

    @InsertProvider(type = DataMigrationHistoryHandler.class, method = "insert")
    int insert(@Param("version") String version, @Param("executeAt") Date executeAt);

    @SelectProvider(type = DataMigrationHistoryHandler.class, method = "count")
    int count(@Param("version") String version);

    @SelectProvider(type = DataMigrationHistoryHandler.class, method = "list")
    @Results(id = "DataMigrationHistoryResult", value = {
            @Result(column = "version", property = "version", jdbcType = JdbcType.VARCHAR),
            @Result(column = "execute_at", property = "executeAt", jdbcType = JdbcType.TIMESTAMP)
    })
    List<DataMigrationHistory> list();
}
```

定义接口里面使用的执行类，使用MyBatis 3提供的Dynamic SQL，可以很好地防止SQL语句的书写错误。

![dynamic sql](/assets/images/2019-10-02-mybatis-dynamic-sql.jpg)

定义`Migration`类执行具体的迁移方法：

```java
// interface
import org.apache.ibatis.session.SqlSession;

public interface IMigration {
    boolean migrate(SqlSession session);
}

// 这个抽象类只是用来方便读取所有的迁移类
public abstract class Migration {
}

/**
 * Migration 20191003
 * Add a record to user table
 * id: 1, username: moicen, lastLogin: now
 */
public class Migration20191003 extends Migration implements IMigration {
    public boolean migrate(SqlSession session) {
        UserMapper mapper = session.getMapper(UserMapper.class);
        return mapper.insert(UUID.randomUUID().toString(), "moicen", new Date()) > 0;
    }
}

```

最后定义一个从外部操作的入口：

![migrator](/assets/images/2019-10-02-mybatis-migrator.jpg)

`getMigrations`方法根据配置文件配置的规则读取所有的迁移类。`migrate`方法检查迁移是否已执行，对未执行过的类按顺序执行。`info`方法输出当前数据库的迁移状态。

测试效果：

![migrator](/assets/images/2019-10-02-mybatis-migrator.jpg)

![migrator](/assets/images/2019-10-02-mybatis-migrate.jpg)

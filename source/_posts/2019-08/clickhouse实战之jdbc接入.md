---
title: clickhouse实战之jdbc接入
date: 2019-08-25 18:29:55
tags: 
- clickhouse
- java
categories: 
- clickhouse

---


# 配置
安装好clickhouse后有几个关键的配置需要调整。
`clickhouse`的配置文件主要有两个:
```shell 
vi /etc/clickhouse-server/config.xml # 服务器配置 
vi /etc/clickhouse-server/users.xml #  客户端连接的默认配置
```
`config.xml`里需要调整的主要是数据文件的目录、http端口、监听地址:
```xml
<http_port>8080</http_port> <!-- 默认是8123-->
<listen_host>0.0.0.0</listen_host> <!-- 默认只监听本地127.0.0.1-->
<path>/var/lib/clickhouse/</path> <!-- 需要chown -R clickhouse:clickhouse这个目录 如果后续要修改，也可以停服后通过软链接移动到别的目录-->

<uncompressed_cache_size>8589934592</uncompressed_cache_size>
<mark_cache_size>5368709120</mark_cache_size>
```
此外也能通过配置文件发现默认的tcp端口是9000,interserver_http_port是9009。

`users.xml`里需要调整的主要是:
```xml
<max_memory_usage>20000000000</max_memory_usage><!-- 单个查询的最大内存使用bytes-->
<password></password>
```

## 启动命令
```shell
# 服务端:
sudo service clickhouse-server start 
# 客户端(多行模式):
clickhouse-client -m 
# 客户端执行sql文件:
clickhouse-client -mn < 1.sql
```

# jdbc
首先是引入依赖:
```xml
<!-- https://mvnrepository.com/artifact/ru.yandex.clickhouse/clickhouse-jdbc -->
<dependency>
    <groupId>ru.yandex.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.1.54</version>
    <exclusions>
    <exclusion>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>*</artifactId>
    </exclusion>
    </exclusions>
</dependency>
```
这里我项目里其他地方用fasterxml，有版本冲突，因此这里exclude掉了。

然后配置yml:
```yml
spring.clickhouse.hikari:
  idle-timeout: 1000
  jdbc-url: jdbc:clickhouse://<your_host_or_ip_address>:8080/default
  driverClassName: ru.yandex.clickhouse.ClickHouseDriver
  maximumPoolSize: 10
  rewriteBatchedStatements: true
```

最后像普通jdbc一样使用即可:
```java
// 1. 配置：
@Configuration
@ConfigurationProperties(prefix = "spring.clickhouse.hikari")
public class ClickhouseDSConfig extends HikariConfig {
    @Bean(name="chds")
    public DataSource dataSource() throws SQLException {
        return new HikariDataSource(this);
    }
}

// 2. 存储层:
@Repository
public class ChDao extends JdbcDaoSupport implements IChDao {
    @Autowired
    public ChDao(@Qualifier("chds") DataSource ds) {
        setDataSource(ds);
    }

    @Override
    public Map<String, Object> query(String sql, Object... args) {
        return this.getJdbcTemplate().queryForMap(sql, args);
    }

    @Override
    public List<Map<String, Object>> queryForList(String sql, Object... args) {
        return this.getJdbcTemplate().queryForList(sql, args);
    }
}

// 3. 使用: 
@Autowired
IChDao dao;

public void run() {
    System.out.println("begin~~~");
    String sqlDB = "show databases";//查询数据库
    System.out.println(dao.queryForList(sqlDB));
    System.out.println(dao.queryForList("show tables;"));
}
```


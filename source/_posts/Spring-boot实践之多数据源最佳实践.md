---
title: Spring boot实践之多数据源最佳实践
date: 2019-01-05 10:31:16
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---

>多数据源主要用于mysql主从，多库等场景，笔者初始接触时也在网上找了很多资料如何配置，但做法百花齐放，有很多用到了ThreadLocal，注解，数据源路由等技术，最终选择了一个简单，易用，易理解的方式：每一个数据源只扫描自己的mapper

## 思路
基于以上思想，只需要以下步骤：
1. 配置文件中采用不同的前缀配置各个数据源
2. 为每个数据源初始化DataSource，SqlSessionFactory，TransactionManager
3. 每个数据源都有自己的配置，扫描自己的mapper.xml，DAO接口

## 实践

### 数据源配置

注意，笔者遇到的情况是多库，以订单库和用户库举例，如果是主从，可以起名master，slave
配置文件如下：

```
#db1
spring.datasource.default.url = jdbc:mysql://localhost:3306/order?characterEncoding=utf-8&connectTimeout=2000&socketTimeout=2000&zeroDateTimeBehavior=convertToNull
spring.datasource.default.username = root
spring.datasource.default.password = 123

#db2
spring.datasource.user.url = jdbc:mysql://localhost:3306/user?characterEncoding=utf-8&connectTimeout=2000&socketTimeout=2000&zeroDateTimeBehavior=convertToNull
spring.datasource.user.username = root
spring.datasource.user.password = 123
```


### 配置数据源相关对象

#### 抽取公共类

如果数据源很多，建议抽取公共类，封装一些公共方法

```java
public class BaseDataSourceConfig {

    /**
     * 初始化SqlSessionFactory
     * @param dataSource 数据源
     * @param location mapper.xml位置
     * @return SqlSessionFactory
     * @throws Exception
     */
    SqlSessionFactory getSqlSessionFactory(DataSource dataSource, String location)
            throws Exception {
        final SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setConfiguration(configuration());
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources(location));
        return sqlSessionFactoryBean.getObject();
    }

    /**
     * 配置下划线转驼峰
     * @return
     */
    private Configuration configuration() {
        Configuration configuration = new Configuration();
        configuration.setMapUnderscoreToCamelCase(true);
        return configuration;
    }
}
```

#### 配置主数据源

将order库作为项目的主数据源,@MapperScan用于扫描DAO接口，MAPPER_LOCATION传入父类指定mapper.xml位置
同时@Primary标注这是我们项目的主数据源
`@ConfigurationProperties("spring.datasource.default")` 表示数据源配置采用的前缀

```java
@Configuration
@MapperScan(basePackages = "com.ttyc.dao.order", sqlSessionFactoryRef = "defaultSqlSessionFactory")
public class DefaultDataSourceConfig extends BaseDataSourceConfig {

    private static final String MAPPER_LOCATION = "classpath:mapper/order/*.xml";

    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.default")
    public DataSource defaultDataSource() {
        return DruidDataSourceBuilder.create()
                .build();
    }

    @Primary
    @Bean(name = "defaultSqlSessionFactory")
    public SqlSessionFactory defaultSqlSessionFactory(@Qualifier("defaultDataSource") DataSource defaultDataSource)
            throws Exception {
        return getSqlSessionFactory(defaultDataSource, MAPPER_LOCATION);
    }

    @Primary
    @Bean(name = "defaultTransactionManager")
    public DataSourceTransactionManager defaultTransactionManager(
            @Qualifier("defaultDataSource") DataSource defaultDataSource) {
        return new DataSourceTransactionManager(defaultDataSource);
    }

}
```

同理配置user库数据源，只不过去除@Primary注解，至此配置结束

### 如何使用

正常编程即可，因为数据源已经按路径扫描了DAO接口和mapper.xml文件


### 思考

以微服务的思想，多库是不应该存在的， 每个服务应该数据自治，职责单一，对于user库应该调用用户微服务接口，而不应该访问用户DB，造成耦合，目前笔者遇到的需求属于临时需求，并且将来会废弃该接口，所以从成本考虑，采用直接访问DB的形式，希望大家引以为戒！




































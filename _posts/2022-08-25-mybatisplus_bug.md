---
layout: post
title: mybatis-plus 踩坑
date: 2022-08-25
tags: 计算机基础
---
### 问题描述
1. 当配置多数据源（mysql jdbc、trino jdbc）时，增加第二个jdbc后，原来的jdbc也失效。
2. 完全多数据源配置后，MetaObjectHandler失效

### 原因
1. 单数据源（datasource）时，SqlSessionFactory bean会采用default对象。多数据源时，2个数据源的SqlSessionFactory需要显式配置。
2. SqlSessionFactory采用缺省配置时，缺省的bean会读取MetaObjectHandler.class的bean对象。显式配置SqlSessionFactory时，需要显式地在globalConfig里配置MetaObjectHandler对象。

### 配置代码
##### datasource配置
```
@Configuration
@MapperScan(value = {"com.***.***.dal.mysql.mapper"}, sqlSessionFactoryRef = "mysqlSqlSessionFactory")
public class MysqlDataConfig {

   /```省略@value注入的datasource属性```/

    @Bean("dataSource") //新建bean实例
    public DataSource dataSource() {
        HikariDataSource hikariDataSource = new HikariDataSource();
        hikariDataSource.setDriverClassName(driveClassName);
        hikariDataSource.setJdbcUrl(jdbcUrl);
        hikariDataSource.setUsername(userName);
        hikariDataSource.setPassword(password);
        hikariDataSource.setMaximumPoolSize(maxPoolSize);
        hikariDataSource.setMinimumIdle(minIdle);
        hikariDataSource.setConnectionTimeout(connectionTimeout);
        hikariDataSource.setIdleTimeout(idleTimeout);
        hikariDataSource.setMaxLifetime(maxLifetime);
        hikariDataSource.setConnectionTestQuery(connectionTestQuery);
        hikariDataSource.setConnectionInitSql(connectionInitSql);
        return hikariDataSource;
    }

    @Bean(name = "templateReport")
    public JdbcTemplate templateXbRisk() {
        return new JdbcTemplate(dataSource());
    }

    @Bean("mysqlSqlSessionFactory")
    @Primary
    public SqlSessionFactory mysqlSqlSessionFactory(@Qualifier("dataSource") DataSource dataSource)
            throws Exception {
        MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();
        bean.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/mysql/*.xml"));
        bean.setDataSource(dataSource);
        MybatisConfiguration config = new MybatisConfiguration();
        // 开启驼峰命名映射
        config.setMapUnderscoreToCamelCase(true);
        bean.setConfiguration(config);
        GlobalConfig globalConfig = new GlobalConfig();
        globalConfig.setMetaObjectHandler(new MysqlMetaObjectHandler());
        bean.setGlobalConfig(globalConfig);
        return bean.getObject();
    }
}
```
##### MetaObjectHandler实现类
```
public class MysqlMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        Object createdBy = this.getFieldValByName("createdBy", metaObject);
        String username = Objects.nonNull(createdBy) ? String.valueOf(createdBy) : "SYSTEM";
        this.setFieldValByName("createdBy", username, metaObject);
        this.setFieldValByName("createdAt", new Date(), metaObject);
        this.setFieldValByName("updatedBy", username, metaObject);
        this.setFieldValByName("updatedAt", new Date(), metaObject);
        //有值，则写入
        if (metaObject.hasGetter("deleteFlag")) {
            this.setFieldValByName("deleteFlag", false, metaObject);
        }
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        Object updatedBy = this.getFieldValByName("updatedBy", metaObject);
        String username = Objects.nonNull(updatedBy) ? String.valueOf(updatedBy) : "SYSTEM";
        this.setFieldValByName("updatedBy", username, metaObject);
        this.setFieldValByName("updatedAt", new Date(), metaObject);
    }
}
```
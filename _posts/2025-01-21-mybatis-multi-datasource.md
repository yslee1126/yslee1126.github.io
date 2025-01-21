---
title: Mybatis multiple datasource 설정 feat spring boot, hikariCP
date: 2025-01-21 17:00:00 +0900
categories: [Library]
math: true
mermaid: true
---

### 개요
- 회사 모듈중에 mybatis 를 사용하는 모듈이 있었는데 datasource 를 여러개 사용해야 하는 상황이 발생했다
- 설정 방법을 정리해본다 

### 환경 
- spring boot 3.x
- mybatis 3.x

### 설정 방법
- application.yml 에 datasource 설정을 두개로 구성한다
```yaml
spring:
  primary-db:
    datasource:
      driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
      url: jdbc:sqlserver://xxxx;databaseName=xxxx;SelectMethod=direct;sendStringParametersAsUnicode=false;responseBuffering=adaptive;noAccessToProcedureBodies=true;
      username: xxxx
      password: xxxx
      hikari:
        pool-name: PrimaryHikariCP
        auto-commit: true
        connection-timeout: 30000
        idle-timeout: 200000
        max-lifetime: 300000
        connection-test-query: SELECT 1
        validation-timeout: 5000
        maximum-pool-size: 10
        minimum-idle: 5
        read-only: false
        leak-detection-threshold: 60000
        transaction-isolation: TRANSACTION_READ_UNCOMMITTED
  comment-db:
    datasource:
      driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
      url: jdbc:sqlserver://xxxx;databaseName=xxxx;SelectMethod=direct;sendStringParametersAsUnicode=false;responseBuffering=adaptive;noAccessToProcedureBodies=true;
      username: xxxx
      password: xxxx
      hikari:
        pool-name: CommentHikariCP
        auto-commit: true
        connection-timeout: 30000
        idle-timeout: 200000
        max-lifetime: 300000
        connection-test-query: SELECT 1
        validation-timeout: 5000
        maximum-pool-size: 10
        minimum-idle: 5
        read-only: false
        leak-detection-threshold: 60000
        transaction-isolation: TRANSACTION_READ_UNCOMMITTED
```

- datasource 를 정의하기 위한 configuration 클래스를 작성한다
- primary db 를 정의하는 경우에는 @Primary 어노테이션을 붙여준다
- 두번째 DB 정의하는 경우 Primary 어노테이션을 제외하고 bean 이름을 다르게 지정한다
```java
@Configuration
@MapperScan(
        basePackages = "xxxxx",
        sqlSessionFactoryRef = "primarySqlSessionFactory"
)
public class PrimaryDatasourceConfig {

    @Primary
    @Bean(name = "primaryDataSourceProperties")
    @ConfigurationProperties(prefix = "spring.primary-db.datasource")
    public DataSourceProperties primaryDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties("spring.primary-db.datasource.hikari")
    public HikariDataSource primaryDataSource(
            @Qualifier("primaryDataSourceProperties") DataSourceProperties primaryDataSourceProperties) {
        return primaryDataSourceProperties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    @Primary
    @Bean(name = "primarySqlSessionFactory")
    public SqlSessionFactory primarySqlSessionFactory(
            @Qualifier("primaryDataSource") DataSource primaryDataSource) throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(primaryDataSource);
        return factoryBean.getObject();
    }

    @Primary
    @Bean(name = "primarySqlSessionTemplate")
    public SqlSessionTemplate primarySqlSessionTemplate(
            @Qualifier("primarySqlSessionFactory") SqlSessionFactory primarySqlSessionFactory) {
        return new SqlSessionTemplate(primarySqlSessionFactory);
    }

}
```

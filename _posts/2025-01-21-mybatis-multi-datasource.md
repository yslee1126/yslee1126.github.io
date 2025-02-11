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

---

### 환경 
- spring boot 3.x
- mybatis 3.x

---

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

```java
@Configuration
@MapperScan(
        basePackages = "aa.bb",
        sqlSessionFactoryRef = "primarySqlSessionFactory"
)
@Import({CommentDatasourceConfig.class})
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

      // MyBatis 설정 파일 추가
      Resource configLocation = new PathMatchingResourcePatternResolver()
        .getResource("classpath:query-config/mybatis-config.xml");
      factoryBean.setConfigLocation(configLocation);

      // 매퍼 파일 추가
      Resource[] mapperLocations = new PathMatchingResourcePatternResolver()
        .getResources("classpath:query-mapper/*.xml");
      factoryBean.setMapperLocations(mapperLocations);
        
        return factoryBean.getObject();
    }

    @Primary
    @Bean(name = "primarySqlSessionTemplate")
    public SqlSessionTemplate primarySqlSessionTemplate(
            @Qualifier("primarySqlSessionFactory") SqlSessionFactory primarySqlSessionFactory) {
        return new SqlSessionTemplate(primarySqlSessionFactory);
    }

}

@MapperScan(
  basePackages = "aa.bb.cc", 
  sqlSessionFactoryRef = "commentSqlSessionFactory"
)
public class CommentDatasourceConfig {

  @Bean(name = "commentDataSourceProperties")
  @ConfigurationProperties(prefix = "spring.comment-db.datasource")
  public DataSourceProperties commentDataSourceProperties() {
    return new DataSourceProperties();
  }

  @Bean(name = "commentDataSource")
  @ConfigurationProperties("spring.comment-db.datasource.hikari")
  public HikariDataSource commentDataSource(
    @Qualifier("commentDataSourceProperties") DataSourceProperties commentDataSourceProperties) {
    return commentDataSourceProperties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
  }

  @Bean(name = "commentSqlSessionFactory")
  public SqlSessionFactory commentSqlSessionFactory(
    @Qualifier("commentDataSource") DataSource commentDataSource) throws Exception {
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(commentDataSource);

    // MyBatis 설정 파일 추가
    Resource configLocation = new PathMatchingResourcePatternResolver()
      .getResource("classpath:query-config/mybatis-config.xml");
    factoryBean.setConfigLocation(configLocation);

    // 매퍼 파일 추가
    Resource[] mapperLocations = new PathMatchingResourcePatternResolver()
      .getResources("classpath:query-mapper/*.xml");
    factoryBean.setMapperLocations(mapperLocations);
    
    return factoryBean.getObject();
  }

  @Bean(name = "commentSqlSessionTemplate")
  public SqlSessionTemplate commentSqlSessionTemplate(
    @Qualifier("commentSqlSessionFactory") SqlSessionFactory commentSqlSessionFactory) {
    return new SqlSessionTemplate(commentSqlSessionFactory);
  }

}
```

--- 

### 회고
- MapperScan 으로 Primary DB 설정은 aa.bb 패키지, Comment DB 설정은 aa.bb.cc 패키지에 각각 적용하려고 한다  
  - Comment 쪽 MapperScan 이 먼저 실행되어야 aa.bb.cc 패키지의 Mapper 들이 Comment datasource 를 사용할 수 있다
  - 만약 Primary 쪽 MapperScan 이 먼저 실행된다면 aa.bb 패키지 아래 모든 Mapper 들이 Primary datasource 를 사용하게 되어 Comment datasource 를 사용할 수 없다
- CommentDatasourceConfig 를 먼저 로딩하려면 어떻게 해야할까 
  - 설정이 로딩되는 순서는 MapperScannerConfigurer.class 에 postProcessBeanDefinitionRegistry() 내부 scan() 부분을 디버깅하면 확인할 수 있다 
  - Config 클래스가 로딩되는 순서는 클래스명의 알파벳 순서로 확인된다 따라서 CommentDatasourceConfig 설정이 PrimaryDatasourceConfig 보다 먼저 로딩된다    
  - 하지만 알파벳 순서 보다 명확하게 Config 클래드의 로딩 순서를 지정하기 위해 Import 어노테이션을 사용할 수 있다 
  - PrimaryDatasourceConfig 에 @Import({CommentDatasourceConfig.class}) 를 추가하면 CommentDatasourceConfig 가 먼저 로딩된다
  - CommentDatasourceConfig 에는 @Configuration 어노테이션을 사용할 필요가 없다 
- Datasource 설정 후 쿼리 실행할때 DatasourceUtils.class 에 getConnection() 을 디버깅하면 실제로 어떤 datasource 를 사용하는지 확인할 수 있다

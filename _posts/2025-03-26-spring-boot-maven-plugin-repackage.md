---
title: 한번의 빌드로 spring-boot executable war 와 was 배포용 war 동시에 만들기
date: 2025-03-26 19:00:00 +0900
categories: [Framework]
math: true
mermaid: true
---

### 개요 
- 한번의 빌드로 실행가능한 war 파일과 설치형 was 에 배포 하기위한 war 파일을 동시에 만들어야하는 상황 발생  
- legacy 서버에는 설치형 was 에 war 를 배포하고 신규 서버에는 실행가능한 war 를 이미지 형태로 배포해야한다  
- spring boot 버전은 2.1.x 사용중인 시스템 
- 멀티 모듈 브로젝트 구조이며 빌드는 maven 을 사용 

--- 

### 설정 
- 최상위 pom.xml 에서 spring-boot-maven-plugin 을 설정한다
- 실행가능한 war 파일을 만들기 위해 repackage 를 선언한다 
```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <version>2.1.10.RELEASE</version>
  <executions>
    <execution>
      <goals>
        <goal>repackage</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

- 빌드하고자 하는 자식 모듈의 pom.xml 에서 maven-war-plugin 을 설정한다
- was 는 wildfly 를 사용하고 있다 wildfly 에 배포하기위한 war 를 생성하게 된다
- was 에 배포하기위한 패키징에는 tomcat 이 필요없다 그리고 컴파일 단계에서만 undertow 가 필요하여 provided 로 설정한다
- 부모에서 설정한 spring-boot-maven-plugin repackage 가 기본적으로 skip 되어 있기 때문에 properties 에 repackage.skip 을 false 로 설정해준다
```xml

<packaging>war</packaging>

<properties>
  <spring-boot.repackage.skip>false</spring-boot.repackage.skip>
</properties>

<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-web</artifactId>
<exclusions>
  <exclusion>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-websocket</artifactId>
  </exclusion>
  <exclusion>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
  </exclusion>
</exclusions>
</dependency>

<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-undertow</artifactId>
<scope>provided</scope>
</dependency>

<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-war-plugin</artifactId>
  <configuration>
    <archive>
      <manifestEntries>
        <Dependencies>jdk.unsupported</Dependencies>
      </manifestEntries>
    </archive>
  </configuration>
</plugin>

```

- was 에 배포하기위해 spring boot servlet initializer 를 상속받은 클래스를 생성한다

```java
public class ServletInitializer extends SpringBootServletInitializer {

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		return application.sources(MokaBbsApplication.class);
	}

}
```

- 이렇게 설정을 완료하고 package 를 실행하면 실행가능하면서도 was 에 배포가능한 war 파일이 생성된다 

--- 

### 회고 
- 프로파일에 따라서 빌드를 분기하여 패키지를 두개로 만드는 방법도 있지만 CI 파이프라인에 변화를 최소화 하면서 빌드를 하기위해 이 방법을 선택했다
- 추후에 설치형 was 배포가 필요없어지면 실행가능한 jar 만 빌드되도록 변경해야 한다 
- spring boot 과 빌드 환경 버전을 꾸준히 올려야 할 필요성을 느낀다 
  - 과거의 기술은 옵션을 찾기가 점점 어려워진다 

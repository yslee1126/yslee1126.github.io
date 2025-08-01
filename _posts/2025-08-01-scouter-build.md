---
title: WebClient 와 Scouter 호환성 오류 대응 
date: 2025-08-01 17:00:00 +0900
categories: [infra]
math: true
mermaid: true
---

### 개요 
- Spring Boot 3.x 에서 WebClient 를 사용하는 경우 Scoter 의 최신 Agent 2.20.0 사용해도 오류가 발생한다 
  - 에러 내용은 ClientHttpResponse.getRawStatusCode() 에 대한 NoSuchMethodError 이다 
  - Scouter Git 에 가면 이슈가 오픈되어있지만 2년 가까이 수정되지 않고 있다 
- Scouter 코드를 내려 받아서 해당 부분을 직접 수정하고 빌드해본다 

### 해결 과정  

- 소스 다운로드

```
> git clone https://github.com/scouter-project/scouter.git
```

- 프로젝트는 jdk 1.8 환경에서 구현된걸로 보이지만 현재 스카우터를 사용할 프로젝트는 17 이여서 빌드를 17 로 실행했는데 상관없다 
- 빌드하기전에 환경 변수 설정이 필요하다 특정 모듈이 java 20을 사용하는 걸로 보인다  
  - JAVA_20_HOME=자바20이상의 jdk home 경로 지정 
- 에러가 나는 부분은 scouter.xtra.httpclient.WebClient 이다 해당 부분을 수정해본다 

```java

/* 
 * res.getRawStatusCode() 이 부분에서 에이전트의 WebFlux 5.x 와 우리 서비스의 WebFlux 6.x 는 HttpStatus 객체가 다르기 때문에 오류가 나게 된다 
 * NoSuchMethodError 를 Catch 하여 아래와 같은 메소드를 실행하도록 수정했다 
 * Scouter 의 WebFlux 버전을 올리는 것은 부작용이 클것 같아서 
 * 성능에는 나쁘겠지만 리플렉션을 사용해서 WebFlux 6.x 의 HttpStatusCode 를 접근해보았다 
 */
@Nullable
private static Integer getStatusCodeSpringBoot3(ClientHttpResponse res) {
  try {
    Method getStatusCodeMethod = ClientHttpResponse.class.getMethod("getStatusCode");
    Object statusCode = getStatusCodeMethod.invoke(res);
    // HttpStatusCode (Spring Framework 6.x)
    if (statusCode != null && statusCode.getClass().getName().equals("org.springframework.http.HttpStatusCode")) {
      Method valueMethod = statusCode.getClass().getMethod("value");
      return (Integer) valueMethod.invoke(statusCode);
    }
  } catch (Exception e) {
    e.printStackTrace(); //로깅할 적당한 방법이 필요하다 
  }

  return null;
}


```

- 전체 프로젝트 중 에이전트 모듈만 빌드해본다 
  - clean package -pl scouter.agent.java -am
- 필드 결과물 target/scouter.agent.jar 를 우리 프로젝트에 적용한다 

### 회고 
- 스카우터의 빌드환경이 예상보다 오래된걸로 보인다 업데이트도 잘 안되는거 같다
- 다른 APM 으로 바꾸는 걸 고려해봐야겠다

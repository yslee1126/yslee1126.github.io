---
title: RedisTemplate host 당 커넥션 수 설정   
date: 2025-09-09 17:00:00 +0900
categories: [library]
math: true
mermaid: true
---

### 개요
- 외부 API 호출하는 부분에서 처리시간이 지연되어 Scouter 에 올라오는 경우가 발생 
  - Scouter 외부 call 응답 소요시간이 많이 찍혀서 외부 API 가 느린건가 했는데 T-Gap 쪽에도 소요시간이 많이 나옴 
  - Scouter 에서 해당 외부 API 링크 클릭해보면 API 소요시간 얼마 없음
- 동시간대 같은 API 가 많이 몰려있는 상황임이 확인되어 해당 코드 파악 


### 설정 
- RestTemplate 사용하는 부분에 max per route 옵션이 없음 
- 기본 값은 5로 확인되어 해당 값을 높게 반영 
- 예상 동시접속 수 산출 
  - TPS x 평균응답시간 = 20 정도로 생각됨 
  - 평균 응답시간말고 느릴때 응답시간도 고려를 해야할지.. 

```java

// http client 가 노후화 되었네..  
Dispatcher dispatcher = new Dispatcher();
        dispatcher.setMaxRequests(maxTotal);
        dispatcher.setMaxRequestsPerHost(maxPerRoute);

OkHttpClient client = new OkHttpClient.Builder()
  .dispatcher(dispatcher)
  .readTimeout(connectionReadTimeout, TimeUnit.MILLISECONDS)
  .connectTimeout(connectionTimeout, TimeUnit.MILLISECONDS)
  .connectionPool(new ConnectionPool(maxIdleConnections, keepAliveDuration, TimeUnit.MILLISECONDS)) //커넥션풀 적용
  .build() 

```


### 회고 
- 동시 요청 값을 초과하여 요청이 쌓이는 경우 큐에 대기하는 걸로 보이는데 아무런 메세지가 안나오니 API 가 어떤 상태인지 확인이 어려웠다
  - APM (스카우터) 에서도 어느 부분에서 대기 하고 있는지 안보였다 더 좋은 APM 을 사용하면 보일려나 
- 예상 동시접속자 수를 좀더 보수적으로 많이 잡아야할지 고민이다 
  - 평균 응답시간이 아닌 느릴때 응답시간을 적용
  - 받아주는 API 서버가 어느정도 TPS 나오는지 확인 필요 
- WebClient 는 설정하지 않았다 
  - 기본값이 50으로 확인되는데 동기식으로 통신하는 부분이 없어서 
  - RestTemplate 보다는 병목이 훨씬 없을 것으로 예상된다

---
title: spring boot undertow memory 최적화 일지 
date: 2025-04-23 17:00:00 +0900
categories: [infra]
math: true
mermaid: true
---

### 개요 
- 일부 서비스 was 를 wildfly 에서 spring boot embedded undertow 로 변경 
- 기존보다 메모리 사용량이 30% 증가되어 연락옴 

### 현상 확인 과정 
- xms, xmx 기존과 12g로 동일하게 설정되었으나 메모리 사용량이 더 많은 걸로 확인됨 
- gc 는 주기적으로 실행되고 있어서 heap used 그래프 모양은 정상적이지만 우상향 중임 그러나 xmx 를 초과 하지 않고 있음 
- old 영역에 누수가 있는지 살펴봄 
```
> jstat -gc 5591 1000 60   # 1초 간격으로 60초 동안 확인
```
- old 영역은 일정하여 문제 없어보임 
  - OC 전체 영역 OU 사용중인 영역  
  - OC 가 늘어나면 메모리 누수 
- metaspace 도 안정적 
  - MC, MU 가 자바옵션으로 지정한것 보다 밑돌고 있음  
- young 영역은 급격하게 차오르는 걸로 보임 
  - Eden 영역이 매우 빠르게 차고 비워짐  
  - Supervisor 영역이 한쪽만 사용되는데 g1gc 에서는 정상으로 판단 SOU, S1U   
  - Eden 이 차는 속도를 young gc 가 못따라가는 형태로 보임 
- 인프라쪽에서 쓰레드가 너무 많다고 연락와서 동작중인 쓰레드 확인 
```
jcmd 프로세스아이디 Thread.print | awk '
  /^"/ {
    name=$0;
    getline;
    state=$0;
    if (name ~ /XNIO-.*task-/) {
      if (state ~ /RUNNABLE/) runnable++;
      else if (state ~ /WAITING/) waiting++;
      else if (state ~ /TIMED_WAITING/) timed_waiting++;
    }
  }
  END {
    print "XNIO RUNNABLE: " runnable;
    print "XNIO WAITING: " waiting;
    print "XNIO TIMED_WAITING: " timed_waiting;
  }
'
```
- 스레드 풀이 너무 큰걸로 확인 실제 돌고 있는 쓰레드는 매우 적음 
  - CPU 도 3% 대로 놀고 있음
  - server.undertow.worker-threads 값 줄여서 테스트 필요 
  - 필요한 워커 스레드 수 ≈ TPS × 요청 처리 시간 (초) 
    - 대략 256 이면 충분할것으로 예상됨 

### 앞으로 할일 
- 덤프가 좀더 자세하게 남아야하는데 옵션 더 필요함
```
-XX:+UseG1GC
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xlog:gc*:file=로그경로/gclog/gc.log:time,uptime,level,tags

-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=로그경로/heapdump.hprof

-XX:+UseStringDeduplication

# 어플리케이션 성능 향상을 위한 옵션 
-Dspring.main.lazy-initialization=true   
-Djava.security.egd=file:/dev/./urandom  
```
- young gc 를 좀더 자주 부르도록
```
-XX:G1NewSizePercent=10 -XX:G1MaxNewSizePercent=50
```
- supervisor 사용 유도 필요한지?? 
```
-XX:MaxTenuringThreshold=12로 늘려 객체가 Survivor Space에 더 오래 머물게 함.
-XX:G1NewSizePercent=15로 Young Generation 크기를 약간 늘려 Survivor Regions 확보.
-XX:+PrintTenuringDistribution 추가해 어떤 객체가 살아남는지 로그로 확인.
```
- xms, xmx 를 서버 메모리의 60% 수준으로 낮춤 

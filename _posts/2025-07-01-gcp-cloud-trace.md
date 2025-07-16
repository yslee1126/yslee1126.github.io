---
title: Spring boot 어플리케이션의 GCP cloud trace 사용 설정  
date: 2025-07-01 17:00:00 +0900
categories: [infra]
math: true
mermaid: true
---

### 개요 
- GCP로 이관되는 Spring boot 어플리케이션에 대한 APM 이 필요한 상황 
- Scouter, Datadog 등 준비되기까지 오래걸리기 때문에 선제적으로 GCP Cloud Trace 사용하기로 결정 
- Spring boot 3.5.x 에서 cloud trace 에 정보 연동하는 기능을 구현해본다 

### 설정 

- 의존성 추가 
```groovy
    implementation 'com.google.cloud:spring-cloud-gcp-starter-trace:6.2.2'
    implementation 'com.google.cloud:google-cloud-trace:2.66.0' // Cloud Trace v2 gRPC client
    implementation 'io.grpc:grpc-services:1.73.0'
```

- application.yml 에 cloud gcp 관련 기본 설정 필요 

### 구현 

- AOP 를 사용하여 필요한 메소드 실행후 추가정보를 Cloud Trace 로 전송 

```java
private final Tracer tracer;

    @Autowired
    public CloudTraceAspect(Tracer tracer) {
        this.tracer = tracer;
    }

    @Around("execution(* ...*Controller.*(..)) || execution(* ...*Service.*(..)) || execution(* ...*Mapper.*(..))")
    public Object traceSelectMethods(ProceedingJoinPoint joinPoint) throws Throwable {
        String className = joinPoint.getSignature().getDeclaringType().getSimpleName();
        String methodName = joinPoint.getSignature().getName();
        String spanName = className + "." + methodName + "()";

        Span span = null;
        Tracer.SpanInScope scope = null;
        try {
            span = tracer.nextSpan().name(spanName).start();
            scope = tracer.withSpanInScope(span); // 스코프 설정
            String traceId = span.context().traceIdString();
            log.info("Trace Method: {}, TraceId: {}", spanName, traceId);

            // SQL Mapper 어노테이션 추출 및 로깅
            extractAndLogSqlAnnotation(joinPoint, span);

            // 파라미터 로깅
            extractAndLogMethodArguments(joinPoint, span);

        } catch (Exception e) {
            log.error("Tracer init failed: {}", e.getMessage());
        }

        try {
            return joinPoint.proceed();
        } catch (Throwable t) {
            if (span != null) span.error(t);
            throw t;
        } finally {
            if (scope != null) scope.close();
            if (span != null) span.finish();
        }
    }
```

- span 은 클라우드 트레이스의 작업단위이다  
- 생성된 span 을 현재 실행 컨텍스트 내에서 활성화 한다  
- 작업이 끝나면 span 을 닫고 finish 하여 cloud trace 로 전송한다 
- 자세한 로그를 보고 싶다면 io.grpc, zipkin2 로그 레벨을 조정한다
- cloud trace 로 통신은 비동기로 이루어진다
  - 스레드명 worker-ELG-1-x
  - 프로토콜은 gRPC 로 확인된다 

### 회고 
- API 호출이 발생하고나서 trace 탐색기를 살펴보면 2초정도의 시차를 두고 발송한 정보가 표시된다
  - 실제 운영환경에서는 trace 로그를 벌크로 전송할텐데 시차가 더 벌어질 수 있다  
- 기존에 사용하던 APM 처럼 쿼리나 파라메터를 알아서 프로파일링 하여보여주지 않기 때문에 AOP 로 해당 부분을 직접 구현해야하는게 번거로웠다 
  - 이 부분을 구현하지 않으면 단순히 API 기본 정보와 총 소요시간만 trace 에 표시된다  
- 비용이 거의 안들어간다고 하니깐 다른 대안이 없으면 쓸만하다 
  - 그러나 가능하면 다른 APM 을 쓰고 싶다 

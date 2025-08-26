---
title: Redis Stream 설정 with SpringBoot  
date: 2025-08-22 17:00:00 +0900
categories: [framework]
math: true
mermaid: true
---

### 개요 
- 메시지 스트리밍 플랫폼이 필요하지만 카프카를 도입하기 여려운 상황인 경우 
- 이미 사용중인 Redis 가 있다면 Stream 기능으로 대체 할 수 있다  

### 설정 
- SpringBoot data redis 의존성은 이미 사용중이라고 가정한다
  - lettuce 설정 역시 커스텀 되어있어야 한다 
- 그외 Redis Stream 주요 설정은 아래와 같다 

```yaml
    redis:
      timeout: 3000 # stream 쓴다면 read timeout은 컨슈머의 poll timeout 보다는 길게 해야함 (RedisStreamConfig.java pollTimout() 부분) 
      connect-timeout: 2000
    lettuce:
      pool:
        max-active: 16 # 컨슈머 수에 따라서 조절필요, 컨슈머가 구독할때 커넥션을 블록하기 때문에  
        max-idle: 8
        min-idle: 1
        time-between-eviction-runs: 5s //idle 커넥션 관리 주기 
```

```java
@Configuration
@Slf4j
public class RedisStreamConfig {

    // 컨슈머를 어떻게 구성할지에 따라서 쓰레드 풀 설정 조정 필요 
    @Bean(name = "redisStreamTaskExecutor")
    public ThreadPoolTaskExecutor redisStreamTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("redis-stream-");
        executor.setWaitForTasksToCompleteOnShutdown(true); // 셧다운 시 작업 완료 대기
        executor.setAwaitTerminationSeconds(30); // 최대 30초 대기
        executor.initialize();
        return executor;
    }

    @Bean
    public StreamMessageListenerContainer<String, MapRecord<String, String, String>> streamMessageListenerContainer(
            RedisConnectionFactory connectionFactory) {

        // 옵션 세팅
        StreamMessageListenerContainer.StreamMessageListenerContainerOptions<String, MapRecord<String, String, String>> options =
                StreamMessageListenerContainer.StreamMessageListenerContainerOptions.builder()
                        .executor(redisStreamTaskExecutor())
                        .pollTimeout(Duration.ofMillis(500)) //너무 짧게 하면 잠시 운영하다가 오류 발생 (커넥션 반납완료 전에 다시 poll 이 발생하여 계속 새로운 커넥션을 생성하면서 오류)  
                        .build();

        return StreamMessageListenerContainer.create(connectionFactory, options);
    }
    
}

@Configuration
@Slf4j
public class StreamSubscriptionConfig {

  @Autowired
  private List<AbstractRedisStreamHandler> streamHandlers;

  @Autowired
  private StreamMessageListenerContainer<String, MapRecord<String, String, String>> container;

  private final Map<String, Subscription> subscriptions = new ConcurrentHashMap<>();

  @Bean
  public Map<String, Subscription> streamSubscriptions() {
    // 각 핸들러에 대해 구독 생성
    for (AbstractRedisStreamHandler handler : streamHandlers) {
      String streamName = handler.getStreamName();
      String groupName = handler.getGroupName();
      String consumerId = handler.generateConsumerId("main"); // 이 부분에서 컨슈머를 여러개 만들 경우에 대한 고민이 필요하다 

      try {
        Subscription subscription = container.receive(
          Consumer.from(groupName, consumerId),
          StreamOffset.create(streamName, ReadOffset.lastConsumed()),
          handler::handleMessage);

        subscriptions.put(streamName, subscription);
        log.info("스트림 {} 구독 설정 완료", streamName);
      } catch (Exception e) {
        log.error("스트림 {} 구독 설정 실패: {}", streamName, e.getMessage(), e);
      }
    }

    // 컨테이너 시작
    if (!container.isRunning()) {
      container.start();
      log.info("스트림 메시지 리스너 컨테이너 시작");
    }

    return subscriptions;
  }
}

```

- AbstractRedisStreamHandler 핸들러 구현하고 각 업무에서 상속 받아서 processMessage 를 구현하자 
```java
@Slf4j
public abstract class AbstractRedisStreamHandler {

  @Autowired
  protected StringRedisTemplate redisTemplate;

  // 등록된 컨슈머 ID들을 추적
  protected final Set<String> consumerIds = ConcurrentHashMap.newKeySet();

  // 자식 클래스에서 구현해야 할 추상 메서드

  /**
   * 처리할 스트림 이름을 반환합니다.
   */
  public abstract String getStreamName();

  /**
   * 처리할 컨슈머 그룹 이름을 반환합니다.
   */
  public abstract String getGroupName();

  /**
   * 최대 유지할 메시지 수를 반환합니다. (trim 용도)
   */
  protected abstract int getMaxMessages();

  /**
   * 스트림 메시지 처리 로직.
   * 실제 비즈니스 로직은 이 메서드에서 구현해야 합니다.
   *
   * @param message 처리할 메시지
   */
  protected abstract void processMessage(MapRecord<String, String, String> message) throws JsonProcessingException;

  /**
   * 메시지를 처리하는 메서드.
   * 공통 로직(로깅, 예외 처리, ACK, 삭제)을 처리하고 실제 비즈니스 로직은
   * processMessage()에 위임합니다.
   */
  public void handleMessage(MapRecord<String, String, String> message) {
    try {
      log.info("{} 메시지 도착: {}", getStreamName(), message.getValue());

      // 실제 메시지 처리 로직 호출 (자식 클래스에서 구현)
      processMessage(message);

    } catch (Exception e) {
      log.error("메시지 처리 실패: ID={}, 에러: {}", message.getId(), e.getMessage(), e);
      // 특정 실패 처리 로직은 자식 클래스에서 재정의 가능
      handleMessageProcessingFailure(message, e);
    } finally {
      try {
        // 메시지 처리 완료 후 ACK 및 삭제
        acknowledgeAndDeleteMessage(message);
      } catch (Exception e) {
        log.error("메시지 ACK/삭제 실패: ID={}, 에러: {}", message.getId(), e.getMessage(), e);
      }
    }
  }

  /**
   * 메시지 처리 실패 시 호출되는 메서드.
   * 자식 클래스에서 필요시 오버라이드하여 사용.
   */
  protected void handleMessageProcessingFailure(MapRecord<String, String, String> message, Exception exception) {
    // 기본 구현은 로깅만 수행. 자식 클래스에서 재정의 가능
    log.error("메시지 처리 실패. 스트림: {}, 그룹: {}, ID: {}, 에러: {}",
      getStreamName(), getGroupName(), message.getId(), exception.getMessage());
  }

  /**
   * 메시지 ACK 및 삭제 처리.
   */
  protected void acknowledgeAndDeleteMessage(MapRecord<String, String, String> message) {
    redisTemplate.opsForStream().acknowledge(getStreamName(), getGroupName(), message.getId());
    log.debug("ACK 완료: 메시지 ID {}", message.getId());

    redisTemplate.opsForStream().delete(getStreamName(), message.getId());
    log.debug("Stream 메시지 삭제 완료: ID {}", message.getId());
  }

  /**
   * 컨슈머 ID 생성.
   */
  public String generateConsumerId(String customerId) {
    String machineName = "";
    try {
      machineName = System.getenv("HOSTNAME"); // K8s에서 자동으로 설정됨
      if (machineName == null || machineName.isEmpty()) {
        machineName = InetAddress.getLocalHost().getHostName();
        log.warn("pod name is null or empty, use local hostname: {}", machineName);
      }
    } catch (Exception e) {
      log.error("Failed to get pod name: {}", e.getMessage(), e);
      machineName = "unknown-host";
    }

    String id = String.format("%s-consumer-%s-%s", getStreamName(), customerId, machineName);
    consumerIds.add(id);
    log.info("생성된 컨슈머 ID: {}", id);
    return id;
  }

  /**
   * 처리되지 않은 메시지(Pending) 처리.
   * 스케줄러로 주기적으로 실행됨.
   */
  @Scheduled(fixedRate = 60000) // 1분 마다
  public void processPendingMessages() {
    try {
      StreamOperations<String, String, String> streamOps = redisTemplate.opsForStream();
      PendingMessagesSummary pendingSummary = streamOps.pending(getStreamName(), getGroupName());

      if (Objects.requireNonNull(pendingSummary).getTotalPendingMessages() == 0) {
        log.debug("{} Pending 메시지 없음", getStreamName());
        return;
      }

      log.info("총 {} Pending 메시지 개수: {}", getStreamName(), pendingSummary.getTotalPendingMessages());

      for (String consumerId : consumerIds) {
        try {
          processPendingMessagesForConsumer(streamOps, consumerId);
        } catch (Exception e) {
          log.error("Consumer {} Pending 메시지 처리 실패: {}", consumerId, e.getMessage(), e);
        }
      }
    } catch (Exception e) {
      log.error("{} Pending 메시지 처리 중 예외 발생: {}", getStreamName(), e.getMessage(), e);
    }
  }

  /**
   * 특정 컨슈머의 처리되지 않은 메시지(Pending) 처리.
   */
  protected void processPendingMessagesForConsumer(StreamOperations<String, String, String> streamOps, String consumerId) {
    try {
      // 특정 consumer의 pending 메시지 조회
      PendingMessages pendingMessages = streamOps.pending(
        getStreamName(),
        Consumer.from(getGroupName(), consumerId),
        Range.unbounded(),
        100L); // 최대 100개 조회

      if (pendingMessages.isEmpty()) {
        log.debug("컨슈머 {}의 Pending 메시지 없음", consumerId);
        return;
      }

      log.info("컨슈머 {}의 Pending 메시지 {}개 처리 시작", consumerId, pendingMessages.size());

      // Pending 메시지 재처리
      for (PendingMessage pending : pendingMessages) {
        String messageId = pending.getId().getValue();

        try {
          List<MapRecord<String, String, String>> claimed = streamOps.claim(
            getStreamName(),
            getGroupName(),
            consumerId,
            Duration.ofSeconds(60),  // 최소 60초 이상 idle
            pending.getId()
          );

          if (!claimed.isEmpty()) {
            MapRecord<String, String, String> message = claimed.get(0);
            log.info("{} Pending 메시지 재처리: {}", getStreamName(), messageId);
            handleMessage(message);
          }
        } catch (Exception e) {
          log.error("{} 메시지 claim 실패: {}, 에러: {}", getStreamName(), messageId, e.getMessage(), e);
        }
      }
    } catch (Exception e) {
      log.error("{} 컨슈머 {} Pending 메시지 처리 중 오류: {}",
        getStreamName(), consumerId, e.getMessage(), e);
    }
  }

  /**
   * 스트림 메시지 정리 (트림).
   * 스케줄러로 주기적으로 실행됨.
   */
  @Scheduled(fixedRate = 300000) // 5분 마다
  public void trimStreams() {
    try {
      int maxMessages = getMaxMessages();
      redisTemplate.opsForStream().trim(getStreamName(), maxMessages, true);
      log.info("스트림 {} 정리 완료 (최대 {} 메시지 유지)", getStreamName(), maxMessages);
    } catch (Exception e) {
      log.warn("스트림 {} 정리 실패: {}", getStreamName(), e.getMessage(), e);
    }
  }

  /**
   * 스트림과 컨슈머 그룹이 존재하는지 확인하고, 없으면 생성합니다.
   */
  @PostConstruct
  public void initStreamAndGroup() {
    try {
      boolean streamExists = Boolean.TRUE.equals(redisTemplate.hasKey(getStreamName()));

      if (!streamExists) {
        log.info("스트림 {}이(가) 존재하지 않아 새로 생성합니다.", getStreamName());
        // 스트림 생성 (더미 메시지 추가)
        redisTemplate.opsForStream().add(getStreamName(),
          Collections.singletonMap("init", "true"));
        // 바로 삭제 (스트림 생성 목적으로만 사용)
        redisTemplate.opsForStream().trim(getStreamName(), 0);
      }

      try {
        // 컨슈머 그룹 생성 시도
        redisTemplate.opsForStream().createGroup(getStreamName(), getGroupName());
        log.info("컨슈머 그룹 {} 생성 완료", getGroupName());
      } catch (Exception e) {
        // 이미 존재하는 경우 무시
        log.debug("컨슈머 그룹 {} 이미 존재함: {}", getGroupName(), e.getMessage());
      }

    } catch (Exception e) {
      log.error("스트림 {}와 그룹 {} 초기화 중 오류: {}",
        getStreamName(), getGroupName(), e.getMessage(), e);
    }
  }
}

// 참고 handleMessage 에서 받아온 messgae 를 DTO 로 전환하는 코드 예시 
public static <T> TestDto<T> fromStreamMessage(MapRecord<String, String, String> message, Class<T> dataClass) throws JsonProcessingException {
  Map<String, String> messageValue = message.getValue();
  Object payloadObj = messageValue.get("payload");
  String jsonString = payloadObj.toString();
  ObjectMapper objectMapper = new ObjectMapper();
  objectMapper.findAndRegisterModules(); // 날짜/시간 타입 모듈 등록
  return objectMapper.readValue(jsonString, objectMapper.getTypeFactory().constructParametricType(TestDto.class, dataClass));
}

```

### 회고 
- 예외 상황을 안정적으로 처리하는 것도 중요하지만 try catch 가 너무 많아지는 건 추가로 고민해야 겠다  
  - 각종 예외 상황에서 알림 메세지 혹은 DLQ 를 이용한 재처리 로직이 필요하다 
- 비지니스 로직이 길어질때 펜딩된 메세지가 의도치 않게 재처리 되는 가능성을 점검해야한다 
  - 주기적으로 펜딩 메세지를 재처리하고 있기 때문에..
- 스트림이 넘치면 잘라버리도록 되어있는데 이런 상황이 오기전에 알림 메세지라던가 서킷브레이커 같은 역할이 필요하다
- 실제로 해당 엄무가 얼마나 QoS 를 보장해야 하는지에 따라서 구현해야하는 예외 처리가 많아지게 된다
- 캐시로만 Redis 를 쓸때에는 lettuce pool 설정 안했는데 Stream 을 사용하면서 pool 크기를 신경쓰게 되었다 
- 급하게 활용하기에는 Redis Stream도 좋은 솔루션이다 
  - 그러나 메세징이 핵심인 업무에는 카프카를 사용하거나 Redis 를 별도로 구축해서 사용하는게 맞는 것 같다
  - Redis Stream 도 그리 간단하게 적용할 수 있는 솔루션은 아닌것 같다 

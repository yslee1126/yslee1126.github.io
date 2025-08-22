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

    @Autowired
    @Lazy // 순환 참조 이슈가 있다면 Lazy 사용, 구독하는 부분을 따로 분리한다면 그럴일 없을듯 
    private RcvStreamHandler rcvStreamHandler;

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


    // 수신 업무 관련 구독, 이런 부분은 다른 설정으로 분리 필요 
    @Bean
    public Subscription rcvSubscription(StreamMessageListenerContainer<String, MapRecord<String, String, String>> container
            , StringRedisTemplate redisTemplate) {

        // 수신 Stream, 그룹이 없다면 생성
        createGroupIfNotExists(redisTemplate, RcvStreamHandler.STREAM_NAME, RcvStreamHandler.GROUP);

        // 구독 시작
        Subscription subscription = container.receive( // 수동 Ack 형태의 구독 
                Consumer.from(RcvStreamHandler.GROUP, rcvStreamHandler.generateConsumerName()), // 컨슈머의 이름을 유니크하게 구분해야함, 그리고 생성한 consumer id 는 저장 해놓고 펜딩메세지 처리할때 사용 
                StreamOffset.create(RcvStreamHandler.STREAM_NAME, ReadOffset.lastConsumed()),
                rcvStreamHandler::handleMessage);

        container.start();
        return subscription;
    }

    private static void createGroupIfNotExists(StringRedisTemplate redisTemplate, String streamName, String groupName) {
        try {
            //stream key 에 ttl 적용은 장애를 유발하므로 추천하지 않고 주기적으로 trim 을 이용해서 유지할 데이터 개수를 제한 하는게 난거 같다  
            redisTemplate.opsForStream().createGroup(streamName, groupName);
        } catch (Exception e) {
            log.info("Group already exists : {}", e.getMessage());
        }
    }
}
```

- 핸들러 구현 
```java
private final Set<String> consumerIds = ConcurrentHashMap.newKeySet();

public void handleMessage(MapRecord<String, String, String> message) {
        try {
            log.info("{} 메세지 도착: {}", STREAM_NAME, message.getValue());

            //비지니스 로직 

        } catch (Exception e) {
            log.error("메시지 처리 또는 ACK/삭제 실패: ID={}, 에러: {}", message.getId(), e.getMessage(), e);
            // 실패 시 재시도 로직 또는 dead-letter queue로 전송
        } finally {
            // 메시지 처리 완료 후 ACK 및 삭제
            // Ack 가 안되면 계속 pending 상태 
          
            redisTemplate.opsForStream().acknowledge(STREAM_NAME, GROUP, message.getId()); // 수동 ACK
            log.info("ACK 완료: 메시지 ID {}", message.getId());

            // ack가 확실히 된 상태에서 delete 필요 
            redisTemplate.opsForStream().delete(STREAM_NAME, message.getId()); // 메시지 삭제
            log.info("Stream 메시지 삭제 완료: ID {}, 값 {}", message.getId(), message.getValue());
        }
    }

// handleMessage 에서 받아온 messgae 를 DTO 로 전환하는 코드 예시 
public static <T> TestDto<T> fromStreamMessage(MapRecord<String, String, String> message, Class<T> dataClass) throws JsonProcessingException {
  Map<String, String> messageValue = message.getValue();
  Object payloadObj = messageValue.get("payload");
  String jsonString = payloadObj.toString();
  ObjectMapper objectMapper = new ObjectMapper();
  objectMapper.findAndRegisterModules(); // 날짜/시간 타입 모듈 등록
  return objectMapper.readValue(jsonString, objectMapper.getTypeFactory().constructParametricType(TestDto.class, dataClass));
}

// pending 상태로 남아있는 메세지에 대한 재처리 
@Scheduled(fixedRate = 60000) // 1분 마다
public void processPendingMessages() {
  StreamOperations<String, String, String> streamOps = redisTemplate.opsForStream();
  PendingMessagesSummary pendingSummary = streamOps.pending(STREAM_NAME, GROUP);

  if (Objects.requireNonNull(pendingSummary).getTotalPendingMessages() == 0) {
    log.info("{} Pending 메시지 없음", STREAM_NAME);
    return;
  }

  log.info("총 RCV Pending 메시지 개수: {}", pendingSummary.getTotalPendingMessages());

  for (String consumerId : consumerIds) {
    try {
      processPendingMessage(streamOps, consumerId);
    } catch (Exception e) {
      log.error("Failed to process pending messages for consumer {}: {}", consumerId, e.getMessage());
    }
  }
}

private void processPendingMessage(StreamOperations<String, String, String> streamOps, String consumerId) {

  try {

    // 특정 consumer의 pending 메시지 조회
    PendingMessages pendingMessages = streamOps.pending(STREAM_NAME,
      Consumer.from(GROUP, consumerId),
      Range.unbounded(),
      100L); // 최대 100개 조회

    if (pendingMessages.isEmpty()) {
      log.info("해당 consumer의 {} Pending 메시지 없음", STREAM_NAME);
      return;
    }

    log.info("처리할 {} Pending 메시지 개수: {}", STREAM_NAME, pendingMessages.size());

    // Pending 메시지 재처리
    for (PendingMessage pending : pendingMessages) {
      String messageId = pending.getId().getValue();

      try {
        List<MapRecord<String, String, String>> claimed = streamOps.claim(
          STREAM_NAME,
          GROUP,
          consumerId,
          Duration.ofSeconds(60),  // 최소 60초 이상 idle
          pending.getId()
        );

        if (!claimed.isEmpty()) {
          MapRecord<String, String, String> message = claimed.get(0);
          try {
            log.info("{} Pending 재처리: {}", STREAM_NAME, message.getValue());
            // 메시지 처리 로직
            handleMessage(message); // 기존 handleMessage 메서드 재사용
          } catch (Exception e) {
            // 필요하면 재처리 dead-letter queue 전송
            log.error("{} Pending 재처리 실패: {}, 에러: {}",STREAM_NAME , messageId, e.getMessage());
          }
        }
      } catch (Exception e) {
        log.error("{} 메시지 claim 실패: {}, 에러: {}",STREAM_NAME, messageId, e.getMessage());
      }
    }
  } catch (Exception e) {
    log.error("{} Pending 메시지 처리 중 오류 발생: {}", STREAM_NAME, e.getMessage(), e);
  }
}

public String generateConsumerName() {
  // 8자리 UUID 생성 (하이픈 제거)
  String uuid = UUID.randomUUID().toString().replace("-", "");
  consumerIds.add(uuid);
  return String.format("%s-consumer-%s", STREAM_NAME, uuid);
}

// 데이터를 스트림에 유지하는 개수에 제한을 둔다
@Scheduled(fixedRate = 300000) // 5분 마다
public void trimStreams() {
  try {
    redisTemplate.opsForStream().trim(STREAM_NAME, 10000, true);
    log.debug("Trimmed stream {} to max 10000 messages", STREAM_NAME);
  } catch (Exception e) {
    log.warn("Failed to trim stream {}: {}", STREAM_NAME, e.getMessage());
  }

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

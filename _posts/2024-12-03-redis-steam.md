---
title: Redis Steam
date: 2024-12-04 10:00:00 +0900
categories: [NoSql]
math: true
mermaid: true
---

1. 개요
- 메세징 시스템
- Producer, Consumer 가 있으며 Consumer Group 지원 
- Kafka 의 topic 개념을 Redis 의 Stream 으로 이해하면 쉽다
  - 그러나 정확히는 Stream 이 Redis 자료구조 형태중 하나이다  
- Consumer Group 에 속한 Consumer 들이 하나의 Steam 에 순서대로 들어온 메세지를 병렬처리 한다
  - 그러나 각각의 처리 종료 시점은 순서대로 진행되지 않는다 
- Pub/Sub 과 달리 메세지가 휘발성이 아니며 하나의 메세지가 모든 Consumer 에게 Broadcasting 되는게 아닌 Message : Consumer = 1 : 1 관계   
- 메세지 재처리 구현 가능 
- Kafka 운영은 부담되지만 Pub/Sub 보다는 안정적인 메세지 처리가 필요한 시스템에 추천 
- 메세지 상태
  - DELIVERED : steam 에 전달됨 
  - PENDING : consumer 에 전달됨  
  - ACK : consumer 가 처리 완료함 


---
2. 코드  
- operator 
- RedisTemplate 이용하여 스트림 처리하기 위한 여러가지 도구들 

```kotlin
@Component
class RedisOperator(private val redisTemplate: RedisTemplate<String, String>,
                    @Value("\${redis.stream.default-ttl-sec}")
                    private val ttl: Long) {

    private val log = LoggerFactory.getLogger(this.javaClass)!!

    fun createStreamIfNotExists(streamKey: String): Boolean {
        return try {
            val opsForStream = redisTemplate.opsForStream<String, String>()
            if (!redisTemplate.hasKey(streamKey)) {
                // 스트림 초기화
                opsForStream.add(
                    MapRecord.create(streamKey, mapOf("type" to "stream-init"))
                )
                redisTemplate.expire(streamKey, Duration.ofSeconds(ttl))
                log.debug("create stream $streamKey with ttl $ttl")
            }
            true
        } catch (e: Exception) {
            log.error("create stream error", e)
            false
        }
    }

    fun streamExists(streamKey: String): Boolean {
        return redisTemplate.hasKey(streamKey)
    }

    fun xAdd(streamKey: String, data: Map<String, String>): RecordId? {
        createStreamIfNotExists(streamKey)
        val record = MapRecord.create(streamKey, data)
        return redisTemplate.opsForStream<String, String>().add(record)
    }

    fun xRange(streamKey: String, start: String, end: String, size: Int): List<MapRecord<String, String, String>> {
        return redisTemplate.opsForStream<String, String>()
            .range(streamKey, Range.closed(start, end))?.takeLast(size) ?: emptyList()
    }

    fun xReadById(streamKey: String, recordId: String): MapRecord<String, String, String>? {
        val range = redisTemplate.opsForStream<String, String>()
            .range(streamKey, Range.closed(recordId, recordId))
        return range?.firstOrNull()
    }

    fun findPendingMessages(
        streamKey: String,
        consumerGroupName: String,
        consumerName: String? = null
    ): PendingMessages {
        log.debug("find pending $streamKey $consumerGroupName $consumerName")
        return if (consumerName != null) {
            // 특정 컨슈머의 Pending 메시지 조회
            this.redisTemplate.opsForStream<String, String>()
                .pending(
                    streamKey,
                    Consumer.from(consumerGroupName, consumerName),
                    Range.unbounded<Long>(),
                    100L
                )
        } else {
            // 그룹의 모든 컨슈머의 Pending 메시지 조회
            this.redisTemplate.opsForStream<String, String>()
                .pending(streamKey, consumerGroupName, Range.unbounded<Long>(), 100L)
        }
    }

    fun xClaim(
        streamKey: String,
        groupName: String,
        newOwner: String,
        idleTime: Long,
        messageIds: List<String>
    ): List<MapRecord<String, String, Any>> {
        // XClaimOptions 설정
        val options = RedisStreamCommands.XClaimOptions.minIdle(Duration.ofMillis(idleTime))
            .ids(*messageIds.map { RecordId.of(it) }.toTypedArray())

        // 메시지 클레임
        return redisTemplate.opsForStream<String, Any>()
            .claim(streamKey, groupName, newOwner, options) ?: emptyList()
    }

    fun xReadGroup(
        streamKey: String,
        groupName: String,
        consumerName: String,
        count: Long = 10,
        block: Long = 0
    ): List<MapRecord<String, String, String>> {
        val readOptions = StreamReadOptions.empty().count(count).block(Duration.ofMillis(block))
        val streams = StreamOffset.create(streamKey, ReadOffset.lastConsumed())
        return redisTemplate.opsForStream<String, String>()
            .read(Consumer.from(groupName, consumerName), readOptions, streams) ?: emptyList()
    }

    fun xAck(streamKey: String, groupName: String, messageId: String): Long? {
        return redisTemplate.opsForStream<String, Any>().acknowledge(streamKey, groupName, messageId)
    }

    fun createStreamConsumerGroup(streamKey: String, consumerGroupName: String) {

        if (!redisTemplate.hasKey(streamKey)) { // Stream 이 존재하지 않는 경우 생성
            //createStreamConsumerGroupAsync(streamKey, consumerGroupName) ttl 지정 실패 
            createStreamIfNotExists(streamKey)
        }

        if (!isStreamConsumerGroupExist(streamKey, consumerGroupName)) { // consumer group 이 없는 경우
            log.debug("create group $consumerGroupName")
            redisTemplate.opsForStream<String, Any>()
                .createGroup(streamKey,
                    ReadOffset.from("0"), //stream 에 있는 모든 메세지 읽도록, ReadOffset.latest() 사용하면 신규 메세지만 처리
                    consumerGroupName)
        }

    }

    /**
     * 비동기 명령어로 스트림과 consumer group 생성했는데 ttl 설정이 안됨
     */
    private fun createStreamConsumerGroupAsync(streamKey: String, consumerGroupName: String) {
        val connection = redisTemplate.connectionFactory?.connection?.nativeConnection
        if (connection is RedisAsyncCommands<*, *>) {
            val commands = connection as RedisAsyncCommands<String, String>
            val commandArgs = CommandArgs(StringCodec.UTF8)
                .add(CommandKeyword.CREATE)
                .add(streamKey) // key
                .add(consumerGroupName) // group
                .add("0") // 0을 사용하면 전체 메세지를 가져옴, $를 사용하면 마지막 메세지만 가져옴
                .add("MKSTREAM") // Stream이 없으면 자동 생성

            commands.dispatch(CommandType.XGROUP, StatusOutput(StringCodec.UTF8), commandArgs)
            log.debug("create stream and group $streamKey $consumerGroupName")

            //ttl 설정 하고 싶은데 잘 안됨
        }
    }

    fun isStreamConsumerGroupExist(streamKey: String, consumerGroupName: String): Boolean {
        val groups = redisTemplate.opsForStream<String, Any>().groups(streamKey)
        return groups.any { it.groupName() == consumerGroupName }
    }

    fun createStreamMessageListenerContainer(): StreamMessageListenerContainer<String, MapRecord<String, String, String>> {
        return StreamMessageListenerContainer.create(
            redisTemplate.connectionFactory,
            StreamMessageListenerContainer.StreamMessageListenerContainerOptions.builder()
                .pollTimeout(Duration.ofMillis(2000))
                //.batchSize() 한번 폴링해서 가져올 메세지 개수
                //.executor() custom task executor 설정 권장
                .build()
        )
    }

}
```

- consumer 
- 메세지를 받아서 핸들링하기위한 리스너 설정 
- 리스너는 Bean 생성할때 초기화 하고 Stream 과 Consumer 그룹생성은 스케줄러에서 호출하여 실행하도록 준비함  
- 새 스트림 생성할때 TTL 지정하여 일정시간이 지나면 없어지도록 메모리 관리에 주의함 

```kotlin
@Component
@Profile("dev")
class RedisStreamConsumer(private val redisOperator: RedisOperator): InitializingBean, DisposableBean {

    private val log = LoggerFactory.getLogger(this.javaClass)!!

    private var listenerContainer: StreamMessageListenerContainer<String, MapRecord<String, String, String>>? = null
    private val subscriptions = mutableListOf<Subscription>()

    fun handleStreamMessage(streamKey: String, consumerGroupName: String, consumerName: String, message: MapRecord<String, String, String>?) {
        val stream = message?.stream
        val recordId = message?.id?.value
        var messageValue = message?.value

        log.debug("handle message {} {} {} {} {}", streamKey, consumerGroupName, consumerName, recordId, messageValue)

        try {
            if (!messageValue.isNullOrEmpty()) {
                // To-Do: 메시지 처리 로직 구현
                log.debug("messaged received : {}", messageValue)
                // 처리 후 stream 에서 삭제?
            }

            // 이후, ack stream
            recordId?.let {
                val ackResult = redisOperator.xAck(streamKey, consumerGroupName, it)
                if (ackResult == 0L) {
                    // ack 실패에 대한 처리
                    log.error("ack fail $recordId")
                } else {
                    log.debug("ack success $recordId")
                }
            } ?: log.error("Record ID is null, skipping acknowledgment.")

        } catch (e: Exception) {
            log.error("message processing error $message", e)
        }
    }

    override fun afterPropertiesSet() {
        // StreamMessageListenerContainer 설정
        listenerContainer = redisOperator.createStreamMessageListenerContainer()
        listenerContainer?.start()
    }

    fun createStreamAndConsumerGroup(streamKey: String, consumerGroupName: String) {
        // Consumer Group 초기화
        redisOperator.createStreamConsumerGroup(streamKey, consumerGroupName)
    }

    fun subscribeToStream(streamKey: String, consumerGroupName: String, consumerName: String) {
        val subscription = listenerContainer?.receive(
            Consumer.from(consumerGroupName, consumerName),
            StreamOffset.create(streamKey, ReadOffset.lastConsumed()),
            StreamListener { record ->
                handleStreamMessage(streamKey, consumerGroupName, consumerName, record)
            }
        )
        subscription?.await(Duration.ofSeconds(1)) // 구독 성공을 위해 1초 기다림

        subscription?.let { subscriptions.add(it) }
    }

    override fun destroy() {
        cleanUpSubscription()
        listenerContainer?.stop()
    }

    fun cleanUpSubscription() {
        subscriptions.forEach{ it.cancel()}
    }

}
```

- 스케줄러 
- 스트림과 구독을 시간별로 생성하고 펜딩상태인 메세지가 있으면 처리한다

```kotlin
@Component
@Profile("dev")
class RedisStreamScheduler(private val redisOperator: RedisOperator,
                           private val redisStreamConsumer: RedisStreamConsumer) {

    private val log = LoggerFactory.getLogger(this.javaClass)!!

    private var consumerName1 = "testConsumer1"
    private var consumerName2 = "testConsumer2"
    private val consumerGroup = "consumer_group"

    // 1분마다 stream 체크해서 없으면 생성
    @Scheduled(fixedRate = 60000)
    fun createStreamAndConsumerGroup() {
        val streamKey = getStreamKey()
        if (redisOperator.streamExists(streamKey)) {
            log.debug("stream already exists $streamKey")
        } else {
            log.debug("")
            redisStreamConsumer.createStreamAndConsumerGroup(streamKey, consumerGroup)
            redisStreamConsumer.cleanUpSubscription()
            redisStreamConsumer.subscribeToStream(streamKey, consumerGroup, consumerName1)
            redisStreamConsumer.subscribeToStream(streamKey, consumerGroup, consumerName2)
        }

    }

    fun getStreamKey(): String {
        val currentTime = LocalDateTime.now()
        val formatter = DateTimeFormatter.ofPattern("'test_'yyyyMMddHH")
        return currentTime.format(formatter)
    }

    // 3분마다 펜딩 체크
    @Scheduled(fixedRate = 180000)
    fun pendingMessagesScheduler() {
        val streamKey = getStreamKey()
        if (!redisOperator.streamExists(streamKey)) {
            log.debug("stream not exists $streamKey")
            return
        }

        var pendingMessages = redisOperator.findPendingMessages(streamKey, consumerGroup)

        // Pending 메시지가 없으면 더 이상 진행하지 않음
        if (pendingMessages.isEmpty) {
            log.debug("no pending messages")
            return
        }

        pendingMessages.forEach { pendingMessage ->
            try {
                // 메시지 ID 로깅
                log.debug("Pending Message ID: {}", pendingMessage.id)

                // 메시지 데이터 로깅
                val messageData = redisOperator.xReadById(streamKey, pendingMessage.id.value)
                if (messageData != null) {
                    log.debug("Message Data: ID={}, Stream={}, Values={}",
                        messageData.id,
                        messageData.stream,
                        messageData.value.entries.joinToString { "${it.key}=${it.value}" }
                    )
                } else {
                    log.debug("No message data found for ID={}", pendingMessage.id)
                }

                // 메시지 처리 시간 체크 (비정상 처리 감지)
                var elapsedTime = pendingMessage.elapsedTimeSinceLastDelivery.toSeconds()
                val isLongPending = elapsedTime > 30 // 30초 이상 Pending

                if (isLongPending) {
                    log.warn("Long Pending Message Detected - ID: ${pendingMessage.id}, Elapsed Time: $elapsedTime ms")
                    // 비정상 처리로 간주되는 메시지에 대한 특별 처리 로직

                }

                // 메시지 처리 로직 (실제 비즈니스 로직)

                // 성공적으로 처리되면 ACK
                redisOperator.xAck(streamKey, consumerGroup, pendingMessage.id.value)
                log.debug("Message {} successfully processed and acknowledged", pendingMessage.id)

            } catch (e: Exception) {
                // 메시지 처리 중 예외 발생 시 로깅
                log.error("Error processing pending message ${pendingMessage.id}", e)

                // 실패한 메시지에 대한 처리 방법 (재시도, 알림 등)
            }
        }

    }

    @Scheduled(fixedRate = 300000)
    fun cleanUpStream() {
        // ack 되었다고 stream 에서 메세지가 지워지지 않기 때문에 너무 많이 쌓인 경우 적당한 삭제로직이 필요
    }

}
```


---

3. 주의사항
- Steam 이름은 key 값이므로 하나의 Steam 에 저장되는 데이터는 하나의 노드에만 쌓인다 
  - Steam 을 자주 생성하는 방식 추천 
- Redis 에서 권장되는 데이터 크기가 500kb 이하인데 1mb 를 초과하는 연산은 오래 걸리므로 싱글 스레드인 Redis 성능에 나쁜 영향을 끼친다
- PENDING 상태 메세지가 쌓여갈 수 있으므로 메모리 모니터링에 신경쓰면서 재처리 로직을 스케줄러 형태로 구현하는걸 추천 

---
4. 회고 
- 메세지 펜딩상태가 정상처리중인지 비정상 처리중인지 구분하는 명확한 기준을 찾지 못함    
- 부하를 여러 노드로 분산하기 위해 시간마다 새로운 스트림을 미리 생성하는 방식을 생각해 봤는데 실제 업무에 따라서 다르게 접근해도 될 것 같다
  - 스트림을 분산하기 위한 고민을 해야한다는게 다소 번거롭다 
  - 하지만 기존에 Redis 를 사용하고 있는 중소규모 서비스에서 도입하기가 쉽다는 점이 큰 장점인거 같긴하다  
- ACK 상태이면 Stream 에서 없어질 줄 알았는데 남아있는 걸로 보인다 
  - 메세지를 지우거나 스트림 TTL 을 짧게 가져가는게 좋을 것 같다  

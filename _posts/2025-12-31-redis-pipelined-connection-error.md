---
title: SpringBoot RedisTemplate Pipelined 명령어 사용시 Connection Timeout 이슈     
date: 2025-12-31 17:00:00 +0900
categories: [library]
math: true
mermaid: true
---

### 개요 
- 새로 런칭한 API 모듈에 Redis 캐시 호출 증가
- multiset 메소드를 사용하는 부분에서 ConnectTimeoutException 이 발생하기 시작

### 주요 로그 

```shell

org.springframework.data.redis.connection.RedisPipelineException: Pipeline contained one or more invalid commands
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.closePipeline(LettuceConnection.java:638)
	at jdk.internal.reflect.GeneratedMethodAccessor416.invoke(Unknown Source)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at org.springframework.data.redis.core.CloseSuppressingInvocationHandler.invoke(CloseSuppressingInvocationHandler.java:61)
	at jdk.proxy2/jdk.proxy2.$Proxy248.closePipeline(Unknown Source)
	at org.springframework.data.redis.core.RedisTemplate.lambda$executePipelined$1(RedisTemplate.java:490)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:406)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:373)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:360)
	at org.springframework.data.redis.core.RedisTemplate.executePipelined(RedisTemplate.java:481)
	at org.springframework.data.redis.core.RedisTemplate.executePipelined(RedisTemplate.java:475)

Caused by: io.lettuce.core.RedisException: io.netty.channel.ConnectTimeoutException: connection timed out after 500 ms
	at io.lettuce.core.internal.Exceptions.bubble(Exceptions.java:83)
	... 24 common frames omitted


```


### 주요 코드 

```java

redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            RedisSerializer<String> keySerializer = (RedisSerializer<String>) redisTemplate.getKeySerializer();
            RedisSerializer<T> valueSerializer = (RedisSerializer<T>) redisTemplate.getValueSerializer();

            for (Map.Entry<String, T> entry : dataToCache.entrySet()) {
                byte[] rawKey = keySerializer.serialize(entry.getKey());
                byte[] rawValue = valueSerializer.serialize(entry.getValue());

                if (rawKey != null && rawValue != null) {
                    connection.stringCommands().set(rawKey,
                            rawValue,
                            Expiration.from(cacheSchema.getTtl(), TimeUnit.SECONDS),
                            RedisStringCommands.SetOption.UPSERT);
                }
            }
            return null;
        });

```

### 현상 확인  

- opsForValue().get() 와 같은 유형의 다른 일반적인 명령어는 shared connection 을 사용 
- executePipelined() 에서는 dedicated connection 을 새로 연결하여 상태를 유지하고 명령어륾 모아서 한번에 flush 

### 이슈 해결 

- oltp 환경에서 pipeline 명령어 쓰지 말자 


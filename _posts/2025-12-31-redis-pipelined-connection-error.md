---
title: SpringBoot RedisTemplate Pipelined 명령어 사용시 Connection Timeout 이슈     
date: 2025-12-31 17:00:00 +0900
categories: [library]
math: true
mermaid: true
---

### 개요 
- 새로 런칭한 API 모듈에 Redis 캐시 호출이 증가하고 있는 상황 
- multi set 메소드를 사용하는 부분에서 ConnectTimeoutException 이 발생하기 시작

### 해당 로그 

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


### 이슈 발생 코드 


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


### 현상 분석   

- opsForValue().get() 같은 유형의 일반적인 명령어는 shared connection 을 사용하지만  
- executePipelined() 에서는 dedicated connection 을 새로 연결하는데 이 과정에서 connection 을 원할하게 얻어오지 못하여 timeout 발생 
  - 파이프라인 명령어는 상태를 유지하고 명령어를 모아서 한번에 flush 하기 위한 메소드이다 
  - 그래서 독립적인 새 connection 이 필요하다 


### 이슈 해결 

- 파이프라인 명령어를 사용하려면 기존의 shared connection 을 사용하는 방식말고 다른 방식의 통신이 필요하다 
- pool 사용 설정을 하면 shared connection 을 사용하지 않고 pooling 된 connection 을 사용하게 된다 https://docs.spring.io/spring-data/redis/reference/redis/drivers.html 
  - 그러나 pool 사용하는 경우 redis 는 어짜피 FIFO 이기 때문에 성능상 별로 이점이 없다는 벤치마킹 결과들이 검색된다 
- 기존의 redisTemplate 을 유지한 상태에서 파이프라인 전용 poolingRedisTemplate 를 하나더 만들어 놓고 파이프라인 명령어를 사용하는 것도 대안이 될 수 있다
  - 일반적인 경우 restTemplate 사용하고 파이프라인 명령어 쓸때만 poolingRedisTemplate 사용하도록 해보자 
  - pool 사용하려면 factory 설정을 분리해서 factory.setShareNativeConnection(false); 셋팅하고 
  - shared 가 아닌 pool 로 관리하는 connection 을 설정하자   

```java
// 2가지 factory 선언 
@Bean
@Primary
public RedisConnectionFactory redisConnectionFactory() {
    RedisClusterConfiguration configuration = getRedisClusterConfiguration();

    return new LettuceConnectionFactory(configuration, getClientConfigForCluster());
}

@Bean(name = "poolingConnectionFactory")
public RedisConnectionFactory poolingConnectionFactory() {
    RedisClusterConfiguration configuration = getRedisClusterConfiguration();

    /* 
     * pooling cluster 를 설정할때는 commons-pool2 라이브러리의 GenericObjectPoolConfig 사용했다 
     * max total 은 너무 많이 잡지 않고 8 정도로 했다 
     * pooling 비용만 늘어나고 성능상 별로 이점이 없을 것 같다 
     */
    LettuceConnectionFactory factory = new LettuceConnectionFactory(configuration, getClientPoolConfigForCluster());
    factory.setShareNativeConnection(false); // shared 연결 사용 안함
    return factory;
}

// 2가지 redis template 선언 
@Bean
@Primary
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
  RedisTemplate<String, Object> template = new RedisTemplate<>();
  template.setConnectionFactory(redisConnectionFactory);
  // 키와 값을 위한 직렬화 전략 설정
  template.setKeySerializer(new StringRedisSerializer());
  template.setValueSerializer(new GenericJackson2JsonRedisSerializer(getObjectMapper()));
  return template;
}


/**
 * 일반적인 shared 연결이 아닌 별도의 연결을 사용하고 싶을때
 */
@Bean(name = "poolingRedisTemplate")
public RedisTemplate<String, Object> poolingRedisTemplate(
  @Qualifier("poolingConnectionFactory") RedisConnectionFactory poolingConnectionFactory) {
  RedisTemplate<String, Object> template = new RedisTemplate<>();
  template.setConnectionFactory(poolingConnectionFactory);
  template.setKeySerializer(new StringRedisSerializer());
  template.setValueSerializer(new GenericJackson2JsonRedisSerializer(getObjectMapper()));
  return template;
}


```


### 회고 
- 왠만하면 oltp 환경에서는 pipeline 명령어 쓰지 말자
  - 가능하면 shared connection 모드로 사용하자 
- 파이프라인 명령어 사용이 과연 얼마나 될까? 너무 자주 쓰지 않는 방안을 논의해보자 


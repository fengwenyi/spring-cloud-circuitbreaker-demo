# Resilience4j

- [spring cloud circuitbreaker doc](https://docs.spring.io/spring-cloud-circuitbreaker/docs/current/reference/html/)

## Starters

### Non Reactive Applications

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

### Reaction Applications

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

## 禁用自动配置

```yaml
spring.cloud.circuitbreaker.resilience4j.enabled: false
```

## 默认配置

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
        .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(4)).build())
        .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
        .build());
        }
```

**Reactive Example**

```java
@Bean
public Customizer<ReactiveResilience4JCircuitBreakerFactory> defaultCustomizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(4)).build()).build());
}
```

## 特定的断路器配置

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
    return factory -> factory.configure(builder -> builder.circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(2)).build()), "slow");
}
```

> 除了配置创建的断路器之外，您还可以在断路器创建之后但在返回给调用者之前自定义断路器。 
> 为此，您可以使用 addCircuitBreakerCustomizer 方法。 
> 这对于将事件处理程序添加到 Resilience4J 断路器非常有用。

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> slowCustomizer() {
    return factory -> factory.addCircuitBreakerCustomizer(circuitBreaker -> circuitBreaker.getEventPublisher()
    .onError(normalFluxErrorConsumer).onSuccess(normalFluxSuccessConsumer), "normalflux");
}
```

**Reactive Example**

```java
@Bean
public Customizer<ReactiveResilience4JCircuitBreakerFactory> slowCustomizer() {
    return factory -> {
        factory.configure(builder -> builder
        .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(2)).build())
        .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults()), "slow", "slowflux");
        factory.addCircuitBreakerCustomizer(circuitBreaker -> circuitBreaker.getEventPublisher()
            .onError(normalFluxErrorConsumer).onSuccess(normalFluxSuccessConsumer), "normalflux");
     };
}
```

## 属性配置

`CircuitBreaker` and `TimeLimiter`

属性配置比 Java 定制器配置具有更高的优先级。

示例：

```yaml
resilience4j.circuitbreaker:
 instances:
     backendA:
         registerHealthIndicator: true
         slidingWindowSize: 100
     backendB:
         registerHealthIndicator: true
         slidingWindowSize: 10
         permittedNumberOfCallsInHalfOpenState: 3
         slidingWindowType: TIME_BASED
         recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate

resilience4j.timelimiter:
 instances:
     backendA:
         timeoutDuration: 2s
         cancelRunningFuture: true
     backendB:
         timeoutDuration: 1s
         cancelRunningFuture: false
```


### 更多自定义配置

[Resilience4j configuration](https://resilience4j.readme.io/docs/getting-started-3#configuration)

```yaml
resilience4j.circuitbreaker:
    instances:
        backendA:
            registerHealthIndicator: true
            slidingWindowSize: 100
        backendB:
            registerHealthIndicator: true
            slidingWindowSize: 10
            permittedNumberOfCallsInHalfOpenState: 3
            slidingWindowType: TIME_BASED
            minimumNumberOfCalls: 20
            waitDurationInOpenState: 50s
            failureRateThreshold: 50
            eventConsumerBufferSize: 10
            recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate
            
resilience4j.retry:
    instances:
        backendA:
            maxAttempts: 3
            waitDuration: 10s
            enableExponentialBackoff: true
            exponentialBackoffMultiplier: 2
            retryExceptions:
                - org.springframework.web.client.HttpServerErrorException
                - java.io.IOException
            ignoreExceptions:
                - io.github.robwin.exception.BusinessException
        backendB:
            maxAttempts: 3
            waitDuration: 10s
            retryExceptions:
                - org.springframework.web.client.HttpServerErrorException
                - java.io.IOException
            ignoreExceptions:
                - io.github.robwin.exception.BusinessException
                
resilience4j.bulkhead:
    instances:
        backendA:
            maxConcurrentCalls: 10
        backendB:
            maxWaitDuration: 10ms
            maxConcurrentCalls: 20
            
resilience4j.thread-pool-bulkhead:
  instances:
    backendC:
      maxThreadPoolSize: 1
      coreThreadPoolSize: 1
      queueCapacity: 1
        
resilience4j.ratelimiter:
    instances:
        backendA:
            limitForPeriod: 10
            limitRefreshPeriod: 1s
            timeoutDuration: 0
            registerHealthIndicator: true
            eventConsumerBufferSize: 100
        backendB:
            limitForPeriod: 6
            limitRefreshPeriod: 500ms
            timeoutDuration: 3s
            
resilience4j.timelimiter:
    instances:
        backendA:
            timeoutDuration: 2s
            cancelRunningFuture: true
        backendB:
            timeoutDuration: 1s
            cancelRunningFuture: false
```

## 隔断规则支持

`resilience4j-bulkhead`

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-bulkhead</artifactId>
    <version>1.7.1</version>
</dependency>
```

默认支持，可以通过配置关闭

```yaml
spring.cloud.circuitbreaker.bulkhead.resilience4j.enabled: false
```

提供了两种实现方式：

- a `SemaphoreBulkhead` which uses Semaphores

- a `FixedThreadPoolBulkhead` which uses a bounded queue and a fixed thread pool.

默认使用 `FixedThreadPoolBulkhead`，更多请看：[Resilience4j Bulkhead](https://resilience4j.readme.io/docs/bulkhead)


`Bulkhead` and `ThreadPoolBulkhead` 配置

```java
@Bean
public Customizer<Resilience4jBulkheadProvider> defaultBulkheadCustomizer() {
    return provider -> provider.configureDefault(id -> new Resilience4jBulkheadConfigurationBuilder()
        .bulkheadConfig(BulkheadConfig.custom().maxConcurrentCalls(4).build())
        .threadPoolBulkheadConfig(ThreadPoolBulkheadConfig.custom().coreThreadPoolSize(1).maxThreadPoolSize(1).build())
        .build()
);
}
```

## 特定的隔板配置

默认：

```java
@Bean
public Customizer<Resilience4jBulkheadProvider> slowBulkheadProviderCustomizer() {
    return provider -> provider.configure(builder -> builder
        .bulkheadConfig(BulkheadConfig.custom().maxConcurrentCalls(1).build())
        .threadPoolBulkheadConfig(ThreadPoolBulkheadConfig.ofDefaults()), "slowBulkhead");
}
```

> 除了配置创建的 Bulkhead 之外，
> 您还可以在它们被创建之后但在它们返回给调用者之前自定义它们。 
> 为此，
> 您可以使用 addBulkheadCustomizer 和 addThreadPoolBulkheadCustomizer 方法。

**Bulkhead Example**

```java
@Bean
public Customizer<Resilience4jBulkheadProvider> customizer() {
    return provider -> provider.addBulkheadCustomizer(bulkhead -> bulkhead.getEventPublisher()
        .onCallRejected(slowRejectedConsumer)
        .onCallFinished(slowFinishedConsumer), "slowBulkhead");
}
```

**Thread Pool Bulkhead Example**

```java
@Bean
public Customizer<Resilience4jBulkheadProvider> slowThreadPoolBulkheadCustomizer() {
    return provider -> provider.addThreadPoolBulkheadCustomizer(threadPoolBulkhead -> threadPoolBulkhead.getEventPublisher()
        .onCallRejected(slowThreadPoolRejectedConsumer)
        .onCallFinished(slowThreadPoolFinishedConsumer), "slowThreadPoolBulkhead");
}
```

## Bulkhead 属性配置

您可以在应用程序的配置属性文件中配置 ThreadPoolBulkhead 和 SemaphoreBulkhead 实例。 

属性配置比 Java 定制器配置具有更高的优先级。

```yaml
resilience4j.thread-pool-bulkhead:
    instances:
        backendA:
            maxThreadPoolSize: 1
            coreThreadPoolSize: 1
resilience4j.bulkhead:
    instances:
        backendB:
            maxConcurrentCalls: 10
```

更多配置，看[Reilience4j Spring Boot 2 Configuration](https://resilience4j.readme.io/docs/getting-started-3#configuration)

## 收集指标

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-micrometer</artifactId>
</dependency>
```

更多，看 [Reilience4j Micrometer](https://resilience4j.readme.io/docs/micrometer)

> ! 您不必直接包含 micrometer-core，因为它是由 spring-boot-starter-actuator 引入的
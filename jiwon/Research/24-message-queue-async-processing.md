# 2.4 메시지 큐와 비동기 처리

## Overview
메시지 큐는 시스템 간 비동기 통신을 가능하게 하는 미들웨어입니다. **메시지 큐**를 통해 시스템 간 결합도를 낮추고, 확장성과 안정성을 향상시킬 수 있습니다.

## 메시지 큐 패턴

### Point-to-Point (Queue)
```
Producer → [Queue] → Consumer
           │     │
           └─ M1 ─┘  (하나의 메시지는 하나의 Consumer만 처리)
           └─ M2 ─┘
```

**특징**:
- 1:1 메시지 전달
- 메시지는 한 번만 처리
- 작업 분산에 적합

### Publish-Subscribe (Topic)
```
Producer → [Topic] → Consumer 1
           │     └→ Consumer 2
           │     └→ Consumer 3
           └─ M1, M2, M3... (모든 Consumer가 동일한 메시지 수신)
```

**특징**:
- 1:N 메시지 전달
- 모든 구독자가 메시지 사본 수신
- 이벤트 알림에 적합

### Request-Response
```
Requester → [Request Queue] → Service
    ↑                           │
    └── [Response Queue] ←──────┘
```

**특징**:
- 비동기 RPC 패턴
- 상관 ID로 요청-응답 매칭
- 긴 처리 시간 작업에 적합

## RabbitMQ 구현

### 기본 설정
```yaml
# docker-compose.yml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3.11-management
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
```

### Spring Boot 설정
```java
@Configuration
@EnableRabbit
public class RabbitMQConfig {
    
    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_QUEUE = "order.processing.queue";
    public static final String ORDER_ROUTING_KEY = "order.created";
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(new Jackson2JsonMessageConverter());
        template.setConfirmCallback(confirmCallback());
        return template;
    }
    
    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder
            .topicExchange(ORDER_EXCHANGE)
            .durable(true)
            .build();
    }
    
    @Bean
    public Queue orderQueue() {
        return QueueBuilder
            .durable(ORDER_QUEUE)
            .withArgument("x-message-ttl", 60000)
            .withArgument("x-dead-letter-exchange", "order.dlx")
            .build();
    }
    
    @Bean
    public Binding orderBinding() {
        return BindingBuilder
            .bind(orderQueue())
            .to(orderExchange())
            .with(ORDER_ROUTING_KEY);
    }
}
```

### 메시지 발행
```java
@Component
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        try {
            MessageProperties properties = new MessageProperties();
            properties.setCorrelationId(event.getOrderId());
            properties.setTimestamp(new Date());
            properties.setHeader("eventType", "ORDER_CREATED");
            
            Message message = new Message(
                objectMapper.writeValueAsBytes(event), properties);
            
            rabbitTemplate.send(ORDER_EXCHANGE, ORDER_ROUTING_KEY, message);
            
            log.info("Order created event published: {}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("Failed to publish order created event", e);
            throw new MessagePublishException("Failed to publish order event", e);
        }
    }
}
```

### 메시지 소비
```java
@Component
public class OrderEventConsumer {
    
    @RabbitListener(queues = ORDER_QUEUE, concurrency = "5-10")
    public void handleOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header Map<String, Object> headers,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
        
        try {
            log.info("Processing order created event: {}", event.getOrderId());
            
            // 중복 처리 방지
            if (isAlreadyProcessed(event.getOrderId())) {
                channel.basicAck(deliveryTag, false);
                return;
            }
            
            // 비즈니스 로직 처리
            processOrderCreation(event);
            
            // 처리 완료 표시
            markAsProcessed(event.getOrderId());
            
            // 메시지 확인
            channel.basicAck(deliveryTag, false);
            
        } catch (RecoverableException e) {
            // 재시도 가능한 예외
            channel.basicNack(deliveryTag, false, true);
            
        } catch (Exception e) {
            // 복구 불가능한 예외 - DLQ로 이동
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

## Apache Kafka 구현

### 클러스터 설정
```yaml
# kafka-cluster.yml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka-1:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_IN_SYNC_REPLICAS: 2
```

### Spring Kafka 설정
```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // 성능 최적화
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        configProps.put(ProducerConfig.RETRIES_CONFIG, 10);
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        // Idempotent producer
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "ecommerce-group");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        
        // 오프셋 관리
        configProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        
        return new DefaultKafkaConsumerFactory<>(configProps);
    }
    
    @Bean
    public NewTopic userEventsTopic() {
        return TopicBuilder.name("user-events")
            .partitions(12)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "604800000") // 7일
            .build();
    }
}
```

### Kafka Producer
```java
@Service
public class KafkaEventPublisher {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public CompletableFuture<SendResult<String, Object>> publishUserEvent(UserEvent event) {
        String topic = "user-events";
        String key = event.getUserId(); // 파티션 키
        
        return kafkaTemplate.send(topic, key, event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    log.info("User event sent: {}, offset: {}", 
                        event.getUserId(), result.getRecordMetadata().offset());
                } else {
                    log.error("Failed to send user event: {}", event.getUserId(), ex);
                }
            });
    }
    
    public CompletableFuture<SendResult<String, Object>> publishOrderEvent(OrderEvent event) {
        String topic = "order-events";
        String key = event.getOrderId();
        
        // 헤더 추가
        ProducerRecord<String, Object> record = new ProducerRecord<>(topic, key, event);
        record.headers().add("eventType", event.getEventType().getBytes());
        record.headers().add("version", "1.0".getBytes());
        
        return kafkaTemplate.send(record);
    }
}
```

### Kafka Consumer
```java
@Component
public class KafkaEventConsumer {
    
    @KafkaListener(topics = "user-events", groupId = "user-service-group")
    public void handleUserEvents(
            @Payload UserEvent event,
            @Header KafkaHeaders headers,
            Acknowledgment ack) {
        
        String eventId = generateEventId(event, headers);
        
        try {
            // 중복 처리 방지
            if (isAlreadyProcessed(eventId)) {
                ack.acknowledge();
                return;
            }
            
            switch (event.getEventType()) {
                case USER_REGISTERED:
                    userService.handleUserRegistration(event);
                    break;
                case USER_PROFILE_UPDATED:
                    userService.handleProfileUpdate(event);
                    break;
                default:
                    log.warn("Unknown event type: {}", event.getEventType());
            }
            
            markAsProcessed(eventId);
            ack.acknowledge();
            
        } catch (RetryableException e) {
            log.warn("Retryable error: {}", e.getMessage());
            throw e; // 재시도를 위해 예외 재던짐
            
        } catch (Exception e) {
            log.error("Failed to process event: {}", eventId, e);
            handleFailedEvent(event, e);
            ack.acknowledge(); // 실패한 메시지도 ACK
        }
    }
}
```

## 메시지 안정성 보장

### 중복 처리 방지
```java
@Service
public class IdempotentMessageProcessor {
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean processMessageIdempotently(String messageId, Supplier<Void> processor) {
        String lockKey = "lock:" + messageId;
        String processedKey = "processed:" + messageId;
        
        // 분산 락 획득
        Boolean lockAcquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "locked", Duration.ofMinutes(5));
        
        if (!lockAcquired) {
            return false; // 다른 인스턴스에서 처리 중
        }
        
        try {
            if (redisTemplate.hasKey(processedKey)) {
                return true; // 이미 처리됨
            }
            
            processor.get();
            
            redisTemplate.opsForValue().set(processedKey, "completed", Duration.ofDays(1));
            return true;
            
        } finally {
            redisTemplate.delete(lockKey);
        }
    }
}
```

### Dead Letter Queue 처리
```java
@Component
public class DeadLetterQueueHandler {
    
    @RabbitListener(queues = "order.dlq")
    public void handleDeadLetterMessages(
            @Payload String message,
            @Header Map<String, Object> headers) {
        
        log.error("Processing dead letter message: {}", message);
        
        // 관리자 알림
        alertService.sendDeadLetterAlert(message, headers);
        
        // 실패 로그 저장
        messageFailureRepository.save(MessageFailure.builder()
            .message(message)
            .headers(headers)
            .timestamp(Instant.now())
            .build());
    }
    
    @Scheduled(fixedRate = 300000) // 5분마다
    public void retryFailedMessages() {
        List<MessageFailure> retryableMessages = 
            messageFailureRepository.findRetryableMessages();
        
        for (MessageFailure failure : retryableMessages) {
            try {
                reprocessMessage(failure);
                failure.markAsRetried();
                messageFailureRepository.save(failure);
                
            } catch (Exception e) {
                failure.incrementRetryCount();
                messageFailureRepository.save(failure);
            }
        }
    }
}
```

## 백프레셔 처리

### 처리량 제한
```java
@Component
public class BackpressureHandler {
    private final Semaphore processingPermits;
    private final RateLimiter rateLimiter;
    
    public BackpressureHandler() {
        this.processingPermits = new Semaphore(100);
        this.rateLimiter = RateLimiter.create(50.0); // 초당 50개 처리
    }
    
    public void processWithBackpressure(String messageId, Runnable processor) {
        try {
            // Rate Limiting
            if (!rateLimiter.tryAcquire(1, TimeUnit.SECONDS)) {
                throw new BackpressureException("Rate limit exceeded");
            }
            
            // 동시 처리량 제한
            boolean acquired = processingPermits.tryAcquire(5, TimeUnit.SECONDS);
            if (!acquired) {
                throw new BackpressureException("Processing capacity exceeded");
            }
            
            try {
                processor.run();
            } finally {
                processingPermits.release();
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ProcessingException("Processing interrupted", e);
        }
    }
}
```

### Circuit Breaker 패턴
```java
@Component
public class MessageProcessingCircuitBreaker {
    private final CircuitBreaker circuitBreaker;
    
    public MessageProcessingCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(20)
            .minimumNumberOfCalls(10)
            .build();
        
        this.circuitBreaker = CircuitBreaker.of("messageProcessing", config);
    }
    
    public void processWithCircuitBreaker(String messageId, Runnable processor) {
        Supplier<Void> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> {
                processor.run();
                return null;
            });
        
        try {
            decoratedSupplier.get();
        } catch (CallNotPermittedException e) {
            log.warn("Circuit breaker is OPEN, rejecting message: {}", messageId);
            moveToDelayQueue(messageId);
        }
    }
}
```

## 성능 모니터링

### 메트릭 수집
```java
@Component
public class MessageMetricsCollector {
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void handleMessageProcessed(MessageProcessedEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        sample.stop(Timer.builder("message.process.duration")
            .tag("queue", event.getQueueName())
            .tag("status", event.getStatus())
            .register(meterRegistry));
        
        Counter.builder("message.process.count")
            .tag("queue", event.getQueueName())
            .tag("status", event.getStatus())
            .register(meterRegistry)
            .increment();
    }
    
    @Scheduled(fixedRate = 30000)
    public void collectQueueMetrics() {
        for (String queueName : monitoredQueues) {
            int queueSize = getQueueSize(queueName);
            int consumerCount = getConsumerCount(queueName);
            
            meterRegistry.gauge("queue.size", Tags.of("queue", queueName), queueSize);
            meterRegistry.gauge("queue.consumers", Tags.of("queue", queueName), consumerCount);
        }
    }
}
```

## Best Practices

### 메시지 설계
- **작은 크기**: 메시지 크기 최소화 (< 1MB)
- **스키마 진화**: 하위 호환성 보장
- **멱등성**: 중복 처리 방지

### 큐 설계
- **적절한 파티셔닝**: 병렬 처리 최적화
- **TTL 설정**: 오래된 메시지 자동 제거
- **Dead Letter Queue**: 실패 메시지 처리

### 에러 처리
- **재시도 정책**: 지수 백오프 적용
- **Circuit Breaker**: 연쇄 장애 방지
- **모니터링**: 실시간 메트릭 수집

## Benefits and Challenges

### Benefits
- **비동기 처리**: 시스템 간 결합도 감소
- **확장성**: 독립적인 확장 가능
- **탄력성**: 장애 격리와 복구 용이
- **처리량**: 병렬 처리로 성능 향상

### Challenges
- **복잡성**: 분산 시스템 복잡성 증가
- **순서 보장**: 메시지 순서 처리 어려움
- **중복 처리**: 정확히 한 번 처리 보장 어려움
- **디버깅**: 비동기 처리로 디버깅 복잡
# 3.1 고급 메시지 패턴

## Overview
고급 메시지 패턴은 복잡한 비즈니스 요구사항을 처리하기 위한 이벤트 기반 아키텍처 패턴입니다. **이벤트 소싱(Event Sourcing)**과 **CQRS(Command Query Responsibility Segregation)** 패턴을 통해 확장 가능하고 유지보수하기 쉬운 시스템을 구축할 수 있습니다.

## 이벤트 소싱 패턴

### 기본 개념
이벤트 소싱은 애플리케이션 상태의 변경을 일련의 이벤트로 저장하는 패턴입니다.

```
Traditional State Storage:
[Current State] ← 직접 상태 저장

Event Sourcing:
[Event 1] → [Event 2] → [Event 3] → [Current State]
```

### 이벤트 스토어 구현
```java
@Entity
public class Event {
    @Id
    private String eventId;
    private String aggregateId;
    private String eventType;
    private String eventData;
    private LocalDateTime timestamp;
    private Long version;
    
    // getters, setters
}

@Repository
public interface EventStore {
    void saveEvent(Event event);
    List<Event> getEventsForAggregate(String aggregateId);
    List<Event> getEventsAfterVersion(String aggregateId, Long version);
}
```

### 애그리게이트 구현
```java
public class OrderAggregate {
    private String orderId;
    private String customerId;
    private OrderStatus status;
    private List<OrderItem> items;
    private List<Event> uncommittedEvents = new ArrayList<>();
    
    // 이벤트 적용
    public void apply(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.customerId = event.getCustomerId();
        this.status = OrderStatus.CREATED;
        this.items = event.getItems();
    }
    
    public void apply(OrderPaidEvent event) {
        this.status = OrderStatus.PAID;
    }
    
    // 커맨드 처리
    public void createOrder(String customerId, List<OrderItem> items) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            UUID.randomUUID().toString(),
            customerId,
            items,
            Instant.now()
        );
        
        applyEvent(event);
    }
    
    public void payOrder(String paymentId) {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("Order cannot be paid in current status");
        }
        
        OrderPaidEvent event = new OrderPaidEvent(
            this.orderId,
            paymentId,
            Instant.now()
        );
        
        applyEvent(event);
    }
    
    private void applyEvent(Object event) {
        apply(event);
        uncommittedEvents.add(event);
    }
    
    // 이벤트 스트림으로부터 애그리게이트 복원
    public static OrderAggregate fromEvents(List<Event> events) {
        OrderAggregate aggregate = new OrderAggregate();
        
        for (Event event : events) {
            switch (event.getEventType()) {
                case "OrderCreated":
                    aggregate.apply(deserialize(event.getEventData(), OrderCreatedEvent.class));
                    break;
                case "OrderPaid":
                    aggregate.apply(deserialize(event.getEventData(), OrderPaidEvent.class));
                    break;
            }
        }
        
        return aggregate;
    }
}
```

### 스냅샷 구현
```java
@Service
public class SnapshotService {
    private final SnapshotRepository snapshotRepository;
    private final EventStore eventStore;
    
    public OrderAggregate loadAggregate(String aggregateId) {
        // 최신 스냅샷 조회
        Optional<Snapshot> snapshot = snapshotRepository.findLatest(aggregateId);
        
        if (snapshot.isPresent()) {
            // 스냅샷 이후 이벤트만 조회
            List<Event> events = eventStore.getEventsAfterVersion(
                aggregateId, snapshot.get().getVersion());
            
            OrderAggregate aggregate = deserialize(snapshot.get().getData(), OrderAggregate.class);
            return aggregate.applyEvents(events);
        } else {
            // 스냅샷이 없으면 모든 이벤트 조회
            List<Event> events = eventStore.getEventsForAggregate(aggregateId);
            return OrderAggregate.fromEvents(events);
        }
    }
    
    @Scheduled(fixedRate = 300000) // 5분마다
    public void createSnapshots() {
        List<String> aggregateIds = getActiveAggregateIds();
        
        for (String aggregateId : aggregateIds) {
            List<Event> events = eventStore.getEventsForAggregate(aggregateId);
            
            if (events.size() > 100) { // 이벤트가 100개 이상이면 스냅샷 생성
                OrderAggregate aggregate = OrderAggregate.fromEvents(events);
                
                Snapshot snapshot = Snapshot.builder()
                    .aggregateId(aggregateId)
                    .version(events.get(events.size() - 1).getVersion())
                    .data(serialize(aggregate))
                    .timestamp(Instant.now())
                    .build();
                
                snapshotRepository.save(snapshot);
            }
        }
    }
}
```

## CQRS 패턴

### 커맨드 모델
```java
// Command
public class CreateOrderCommand {
    private String customerId;
    private List<OrderItem> items;
    // getters, setters
}

// Command Handler
@Service
public class OrderCommandHandler {
    private final EventStore eventStore;
    private final EventPublisher eventPublisher;
    
    @CommandHandler
    public void handle(CreateOrderCommand command) {
        // 비즈니스 로직 검증
        validateCommand(command);
        
        // 애그리게이트 생성
        OrderAggregate aggregate = new OrderAggregate();
        aggregate.createOrder(command.getCustomerId(), command.getItems());
        
        // 이벤트 저장
        List<Event> events = aggregate.getUncommittedEvents();
        for (Event event : events) {
            eventStore.saveEvent(event);
            eventPublisher.publish(event);
        }
        
        aggregate.markEventsAsCommitted();
    }
    
    @CommandHandler
    public void handle(PayOrderCommand command) {
        // 애그리게이트 로드
        OrderAggregate aggregate = loadAggregate(command.getOrderId());
        
        // 커맨드 처리
        aggregate.payOrder(command.getPaymentId());
        
        // 이벤트 저장 및 발행
        saveAndPublishEvents(aggregate);
    }
}
```

### 쿼리 모델
```java
// Read Model
@Entity
@Table(name = "order_view")
public class OrderView {
    @Id
    private String orderId;
    private String customerId;
    private String customerName;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // getters, setters
}

// Query Handler
@Service
public class OrderQueryHandler {
    private final OrderViewRepository repository;
    
    public OrderView getOrder(String orderId) {
        return repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    public Page<OrderView> getOrdersByCustomer(String customerId, Pageable pageable) {
        return repository.findByCustomerId(customerId, pageable);
    }
    
    public List<OrderSummary> getOrderSummaryByDate(LocalDate date) {
        return repository.findOrderSummaryByDate(date);
    }
}
```

### 프로젝션 업데이트
```java
@Component
public class OrderProjectionHandler {
    private final OrderViewRepository orderViewRepository;
    private final CustomerService customerService;
    
    @EventHandler
    public void handle(OrderCreatedEvent event) {
        Customer customer = customerService.getCustomer(event.getCustomerId());
        
        OrderView orderView = new OrderView();
        orderView.setOrderId(event.getOrderId());
        orderView.setCustomerId(event.getCustomerId());
        orderView.setCustomerName(customer.getName());
        orderView.setStatus(OrderStatus.CREATED);
        orderView.setTotalAmount(calculateTotal(event.getItems()));
        orderView.setCreatedAt(event.getTimestamp());
        
        orderViewRepository.save(orderView);
    }
    
    @EventHandler
    public void handle(OrderPaidEvent event) {
        OrderView orderView = orderViewRepository.findById(event.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
        
        orderView.setStatus(OrderStatus.PAID);
        orderView.setUpdatedAt(event.getTimestamp());
        
        orderViewRepository.save(orderView);
    }
    
    @EventHandler
    public void handle(OrderShippedEvent event) {
        OrderView orderView = orderViewRepository.findById(event.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
        
        orderView.setStatus(OrderStatus.SHIPPED);
        orderView.setUpdatedAt(event.getTimestamp());
        
        orderViewRepository.save(orderView);
    }
}
```

## 이벤트 재생과 복구

### 이벤트 재생 서비스
```java
@Service
public class EventReplayService {
    private final EventStore eventStore;
    private final List<EventHandler> eventHandlers;
    
    public void replayEventsForAggregate(String aggregateId) {
        List<Event> events = eventStore.getEventsForAggregate(aggregateId);
        
        for (Event event : events) {
            for (EventHandler handler : eventHandlers) {
                try {
                    handler.handle(event);
                } catch (Exception e) {
                    log.error("Failed to replay event {} with handler {}", 
                        event.getEventId(), handler.getClass().getName(), e);
                }
            }
        }
    }
    
    public void replayEventsFromTimestamp(Instant fromTimestamp) {
        List<Event> events = eventStore.getEventsAfterTimestamp(fromTimestamp);
        
        for (Event event : events) {
            publishEvent(event);
        }
    }
    
    public void rebuildProjection(String projectionName, LocalDateTime fromDate) {
        // 프로젝션 초기화
        clearProjection(projectionName);
        
        // 지정된 날짜부터 이벤트 재생
        List<Event> events = eventStore.getEventsAfterDate(fromDate);
        
        EventHandler handler = getProjectionHandler(projectionName);
        for (Event event : events) {
            handler.handle(event);
        }
    }
}
```

### 이벤트 버전 관리
```java
@Component
public class EventVersionManager {
    
    public Object upgradeEvent(Event event) {
        switch (event.getEventType()) {
            case "OrderCreated":
                return upgradeOrderCreatedEvent(event);
            case "OrderPaid":
                return upgradeOrderPaidEvent(event);
            default:
                return deserialize(event.getEventData());
        }
    }
    
    private OrderCreatedEvent upgradeOrderCreatedEvent(Event event) {
        if (event.getVersion() == 1) {
            // V1에서 V2로 업그레이드
            OrderCreatedEventV1 v1Event = deserialize(event.getEventData(), OrderCreatedEventV1.class);
            
            return OrderCreatedEvent.builder()
                .orderId(v1Event.getOrderId())
                .customerId(v1Event.getCustomerId())
                .items(convertItems(v1Event.getItems()))
                .shippingAddress(getDefaultShippingAddress(v1Event.getCustomerId()))
                .timestamp(v1Event.getTimestamp())
                .build();
        }
        
        return deserialize(event.getEventData(), OrderCreatedEvent.class);
    }
}
```

## 이벤트 스트리밍

### Kafka를 활용한 이벤트 스트리밍
```java
@Service
public class EventStreamProcessor {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    @EventHandler
    public void publishToStream(DomainEvent event) {
        String topic = getTopicForEvent(event);
        String key = event.getAggregateId();
        
        EventEnvelope envelope = EventEnvelope.builder()
            .eventId(event.getEventId())
            .eventType(event.getClass().getSimpleName())
            .aggregateId(event.getAggregateId())
            .timestamp(event.getTimestamp())
            .data(event)
            .version(event.getVersion())
            .build();
        
        kafkaTemplate.send(topic, key, envelope)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event to stream", ex);
                } else {
                    log.debug("Event published to stream: {}", event.getEventId());
                }
            });
    }
    
    @KafkaListener(topics = "order-events", groupId = "analytics-service")
    public void processOrderEvents(EventEnvelope envelope) {
        try {
            DomainEvent event = deserializeEvent(envelope);
            analyticsEventHandler.handle(event);
        } catch (Exception e) {
            log.error("Failed to process order event", e);
        }
    }
}
```

### 이벤트 상관관계 추적
```java
@Component
public class EventCorrelationTracker {
    private final RedisTemplate<String, Object> redisTemplate;
    
    public void trackCorrelation(String correlationId, String eventId) {
        String key = "correlation:" + correlationId;
        redisTemplate.opsForList().rightPush(key, eventId);
        redisTemplate.expire(key, Duration.ofHours(24));
    }
    
    public List<String> getCorrelatedEvents(String correlationId) {
        String key = "correlation:" + correlationId;
        return redisTemplate.opsForList().range(key, 0, -1)
            .stream()
            .map(Object::toString)
            .collect(Collectors.toList());
    }
    
    public void createSaga(String sagaId, String correlationId) {
        String key = "saga:" + sagaId;
        Map<String, Object> sagaData = new HashMap<>();
        sagaData.put("correlationId", correlationId);
        sagaData.put("status", "STARTED");
        sagaData.put("events", new ArrayList<>());
        
        redisTemplate.opsForHash().putAll(key, sagaData);
        redisTemplate.expire(key, Duration.ofDays(7));
    }
}
```

## 이벤트 소싱 최적화

### 성능 최적화
```java
@Service
public class OptimizedEventStore {
    private final JdbcTemplate jdbcTemplate;
    private final RedisTemplate<String, Object> redisTemplate;
    
    public void saveEventsBatch(List<Event> events) {
        // 배치 삽입으로 성능 향상
        String sql = "INSERT INTO events (event_id, aggregate_id, event_type, event_data, timestamp, version) VALUES (?, ?, ?, ?, ?, ?)";
        
        jdbcTemplate.batchUpdate(sql, events, events.size(), (ps, event) -> {
            ps.setString(1, event.getEventId());
            ps.setString(2, event.getAggregateId());
            ps.setString(3, event.getEventType());
            ps.setString(4, event.getEventData());
            ps.setTimestamp(5, Timestamp.from(event.getTimestamp()));
            ps.setLong(6, event.getVersion());
        });
        
        // 캐시 업데이트
        for (Event event : events) {
            updateEventCache(event);
        }
    }
    
    public List<Event> getEventsForAggregateOptimized(String aggregateId) {
        // 캐시에서 먼저 확인
        String cacheKey = "events:" + aggregateId;
        List<Event> cachedEvents = (List<Event>) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedEvents != null) {
            return cachedEvents;
        }
        
        // 캐시 미스 시 DB에서 조회
        List<Event> events = jdbcTemplate.query(
            "SELECT * FROM events WHERE aggregate_id = ? ORDER BY version",
            eventRowMapper,
            aggregateId
        );
        
        // 캐시에 저장 (최대 1시간)
        redisTemplate.opsForValue().set(cacheKey, events, Duration.ofHours(1));
        
        return events;
    }
}
```

## Best Practices

### 이벤트 설계
- **불변성**: 이벤트는 한 번 생성되면 변경되지 않음
- **완전성**: 상태 복원에 필요한 모든 정보 포함
- **버전 관리**: 스키마 진화를 위한 버전 정보

### 애그리게이트 설계
- **일관성 경계**: 애그리게이트 내부에서만 강한 일관성 보장
- **크기 제한**: 너무 큰 애그리게이트는 성능 문제 야기
- **비즈니스 규칙**: 도메인 규칙을 애그리게이트에서 집중 관리

### 프로젝션 관리
- **최적화**: 쿼리 패턴에 맞는 프로젝션 설계
- **일관성**: Eventually Consistent 모델 수용
- **복구**: 프로젝션 재구축 메커니즘 제공

## Benefits and Challenges

### Benefits
- **감사 추적**: 모든 변경사항의 완전한 이력 보존
- **시점 복원**: 특정 시점의 상태로 복원 가능
- **확장성**: 읽기와 쓰기 모델 독립적 확장
- **유연성**: 새로운 비즈니스 요구사항에 유연하게 대응

### Challenges
- **복잡성**: 전통적인 CRUD보다 복잡한 구현
- **학습 곡선**: 개발팀의 이벤트 소싱 이해 필요
- **일관성**: Eventually Consistent 모델의 복잡성
- **성능**: 이벤트 재생 시 성능 고려사항
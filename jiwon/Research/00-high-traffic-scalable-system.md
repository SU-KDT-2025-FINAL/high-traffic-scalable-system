# 고트래픽 확장 가능한 시스템 학습 로드맵

## 📚 학습 개요

이 로드맵은 **고트래픽 확장 가능한 시스템 완전 가이드**를 기반으로 체계적인 학습 계획을 제공합니다. 각 단계별로 이론 학습과 실습을 병행합니다.

### 🎯 학습 목표
- 대규모 트래픽을 처리할 수 있는 시스템 설계 능력 확보
- 분산 시스템의 핵심 개념과 패턴 이해
- 실제 운영 환경에서 사용되는 기술 스택 경험
- 성능 최적화와 장애 대응 능력 개발

### 📋 전체 학습 일정
```
Phase 1: 기초 개념    - 확장성 개념, 분산 시스템 기초
Phase 2: 핵심 기술    - 로드밸런싱, 캐싱, 데이터베이스 확장
Phase 3: 고급 아키텍처 - 오토스케일링, 메시지큐, 마이크로서비스
Phase 4: 운영과 최적화 - 모니터링, 실전 프로젝트
```

---

## 🚀 Phase 1: 기초 개념 및 아키텍처

### Step 1: 시스템 확장성 개념과 원리
**학습 목표**: 확장 가능한 시스템의 기본 개념 이해

#### 📖 이론 학습
- [ ] 수직 확장 vs 수평 확장 개념 이해
- [ ] Stateless vs Stateful 설계 차이점
- [ ] 확장성 설계 원칙과 베스트 프랙티스
- [ ] 장애 격리(Fault Isolation) 개념

#### 🛠️ 실습 과제
1. **간단한 웹 애플리케이션 작성**
   ```bash
   # Spring Boot 프로젝트 생성
   spring init --dependencies=web,actuator simple-web-app
   ```
   - Stateless REST API 구현 (사용자 정보 조회/등록)
   - 세션을 사용하지 않는 인증 구현 (JWT)
   - Health Check 엔드포인트 구현

2. **부하 테스트 환경 구축**
   ```bash
   # Apache Bench 설치 및 테스트
   ab -n 1000 -c 10 http://localhost:8080/api/users
   
   # JMeter를 사용한 부하 테스트 시나리오 작성
   ```

#### 📚 추천 자료
- "Designing Data-Intensive Applications" Ch.1
- AWS Well-Architected Framework 문서
- Martin Fowler의 "Patterns of Enterprise Application Architecture"

---

### Step 2: 분산 시스템 아키텍처
**학습 목표**: 분산 시스템의 핵심 개념과 CAP 정리 이해

#### 📖 이론 학습
- [ ] CAP 정리 (Consistency, Availability, Partition Tolerance)
- [ ] 분산 시스템 패턴 (Master-Slave, Peer-to-Peer, Sharding)
- [ ] 일관성 모델 (Strong, Eventual, Weak Consistency)
- [ ] 분산 합의 알고리즘 (Raft, PBFT) 기초

#### 🛠️ 실습 과제
1. **분산 데이터 저장소 구현**
   ```java
   // 간단한 일관된 해싱 구현
   public class ConsistentHashing {
       private TreeMap<Long, String> ring = new TreeMap<>();
       
       public void addNode(String node) {
           // 가상 노드 생성 및 링에 추가
       }
       
       public String getNode(String key) {
           // 키에 해당하는 노드 반환
       }
   }
   ```

2. **Master-Slave 패턴 구현**
   - Redis Master-Slave 설정
   - 읽기 전용 복제본에서 데이터 조회
   - 장애 시 Failover 테스트

#### 🔍 실습 검증
- [ ] 일관된 해싱으로 노드 추가/제거 시 재분배 확인
- [ ] Master 장애 시 Slave 승격 과정 관찰
- [ ] 다양한 일관성 레벨에서의 데이터 동기화 테스트

---

### Step 3: 로드 밸런싱과 트래픽 분산
**학습 목표**: 로드 밸런서의 종류와 알고리즘 이해 및 구현

#### 📖 이론 학습
- [ ] L4 vs L7 로드 밸런서 차이점
- [ ] 로드 밸런싱 알고리즘 (Round Robin, Weighted RR, Least Connections, IP Hash)
- [ ] 세션 지속성(Session Affinity) 문제와 해결책
- [ ] 글로벌 로드 밸런싱과 DNS 기반 분산

#### 🛠️ 실습 과제
1. **NGINX 로드 밸런서 설정**
   ```nginx
   upstream backend {
       least_conn;
       server app1:8080 weight=3;
       server app2:8080 weight=2;
       server app3:8080 weight=1;
   }
   
   server {
       listen 80;
       location / {
           proxy_pass http://backend;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```

2. **애플리케이션 클러스터 구성**
   ```docker-compose
   # docker-compose.yml
   version: '3.8'
   services:
     nginx:
       image: nginx:alpine
       ports:
         - "80:80"
       volumes:
         - ./nginx.conf:/etc/nginx/nginx.conf
     
     app1:
       build: .
       environment:
         - SERVER_ID=app1
     
     app2:
       build: .
       environment:
         - SERVER_ID=app2
   ```

3. **부하 분산 효과 측정**
   - 각 서버별 요청 분배 비율 확인
   - 세션 지속성 테스트
   - 서버 장애 시 자동 배제 확인

#### 📈 성능 측정
- [ ] 로드 밸런서 유무에 따른 응답시간 비교
- [ ] 다양한 알고리즘별 성능 특성 분석
- [ ] 동시 사용자 수 증가에 따른 처리량 변화

---

### Step 4: 캐싱 전략과 구현
**학습 목표**: 다층 캐싱 구조와 캐시 패턴 이해

#### 📖 이론 학습
- [ ] 캐시 레벨별 특성 (Browser, CDN, Reverse Proxy, Application, Database)
- [ ] 캐싱 패턴 (Cache-Aside, Write-Through, Write-Behind)
- [ ] 캐시 무효화 전략과 TTL 정책
- [ ] 분산 캐시와 일관성 문제

#### 🛠️ 실습 과제
1. **Redis를 활용한 분산 캐시 구현**
   ```java
   @Service
   public class ProductService {
       @Autowired
       private RedisTemplate<String, Object> redisTemplate;
       
       @Cacheable(value = "products", key = "#productId")
       public Product getProduct(String productId) {
           // 캐시 미스 시 DB에서 조회
           return productRepository.findById(productId);
       }
       
       @CacheEvict(value = "products", key = "#product.id")
       public void updateProduct(Product product) {
           productRepository.save(product);
       }
   }
   ```

2. **멀티레벨 캐싱 구현**
   - L1: 애플리케이션 메모리 캐시 (Caffeine)
   - L2: 분산 캐시 (Redis)
   - L3: 데이터베이스
   ```java
   public Product getProductWithMultiLevelCache(String id) {
       // L1 캐시 확인
       Product product = l1Cache.get(id);
       if (product != null) return product;
       
       // L2 캐시 확인
       product = (Product) redisTemplate.opsForValue().get(id);
       if (product != null) {
           l1Cache.put(id, product);
           return product;
       }
       
       // DB 조회 및 캐시 저장
       product = productRepository.findById(id);
       if (product != null) {
           redisTemplate.opsForValue().set(id, product, Duration.ofMinutes(10));
           l1Cache.put(id, product);
       }
       return product;
   }
   ```

3. **캐시 성능 측정 도구 구현**
   - 캐시 적중률 측정
   - 응답시간 개선 효과 분석
   - 메모리 사용량 모니터링

#### 🔍 Phase 1 종합 평가
**프로젝트**: 간단한 전자상거래 시스템 구축
- [ ] 상품 조회 API (캐싱 적용)
- [ ] 사용자 인증 시스템 (Stateless JWT)
- [ ] 로드 밸런서를 통한 다중 인스턴스 운영
- [ ] 부하 테스트를 통한 성능 검증

---

## 🏗️ Phase 2: 핵심 기술 구현

### Step 2.1: 데이터베이스 확장 기법 - 읽기 확장
**학습 목표**: 데이터베이스 읽기 성능 최적화 기법 학습

#### 📖 이론 학습
- [ ] 읽기 복제본(Read Replica) 아키텍처
- [ ] Master-Slave 복제 메커니즘
- [ ] 복제 지연(Replication Lag) 문제와 해결책
- [ ] 읽기/쓰기 분리 패턴

#### 🛠️ 실습 과제
1. **MySQL Master-Slave 복제 설정**
   ```yaml
   # docker-compose.yml
   version: '3.8'
   services:
     mysql-master:
       image: mysql:8.0
       environment:
         MYSQL_ROOT_PASSWORD: password
         MYSQL_DATABASE: ecommerce
       command: >
         --server-id=1
         --log-bin=mysql-bin
         --binlog-format=ROW
     
     mysql-slave:
       image: mysql:8.0
       environment:
         MYSQL_ROOT_PASSWORD: password
       command: >
         --server-id=2
         --relay-log=relay-log
         --read-only=1
   ```

2. **동적 데이터소스 라우팅 구현**
   ```java
   @Configuration
   public class DatabaseConfig {
       @Bean
       public DataSource routingDataSource() {
           RoutingDataSource routingDataSource = new RoutingDataSource();
           
           Map<Object, Object> dataSourceMap = new HashMap<>();
           dataSourceMap.put("write", masterDataSource());
           dataSourceMap.put("read", slaveDataSource());
           
           routingDataSource.setTargetDataSources(dataSourceMap);
           routingDataSource.setDefaultTargetDataSource(masterDataSource());
           return routingDataSource;
       }
   }
   
   // 사용 예시
   @ReadDataSource
   public List<Product> findAllProducts() {
       return productRepository.findAll();
   }
   
   @WriteDataSource  
   public Product createProduct(Product product) {
       return productRepository.save(product);
   }
   ```

3. **복제 지연 모니터링**
   - 복제 지연 시간 측정
   - 일관성 검증 도구 구현
   - 장애 시 자동 전환 메커니즘

---

### Step 2.2: 데이터베이스 확장 기법 - 샤딩
**학습 목표**: 데이터 분산 저장을 위한 샤딩 기법 학습

#### 📖 이론 학습
- [ ] 샤딩 전략 (Range-based, Hash-based, Directory-based)
- [ ] 샤드 키 선택 기준과 핫스팟 방지
- [ ] 크로스 샤드 쿼리 문제와 해결책
- [ ] 데이터베이스 파티셔닝 (Horizontal/Vertical)

#### 🛠️ 실습 과제
1. **해시 기반 샤딩 구현**
   ```java
   @Service
   public class ShardingService {
       private List<DataSource> shards;
       
       public DataSource getShard(String shardKey) {
           int hash = shardKey.hashCode();
           int shardIndex = Math.abs(hash) % shards.size();
           return shards.get(shardIndex);
       }
       
       public void saveUser(User user) {
           DataSource shard = getShard(user.getUserId());
           // 해당 샤드에 데이터 저장
           JdbcTemplate jdbcTemplate = new JdbcTemplate(shard);
           jdbcTemplate.update("INSERT INTO users ...", user);
       }
   }
   ```

2. **MySQL 파티셔닝 설정**
   ```sql
   -- 날짜 기반 파티셔닝
   CREATE TABLE orders (
       order_id BIGINT,
       user_id BIGINT,
       order_date DATE,
       total_amount DECIMAL
   ) PARTITION BY RANGE (TO_DAYS(order_date)) (
       PARTITION p2024_01 VALUES LESS THAN (TO_DAYS('2024-02-01')),
       PARTITION p2024_02 VALUES LESS THAN (TO_DAYS('2024-03-01')),
       PARTITION p_future VALUES LESS THAN MAXVALUE
   );
   ```

3. **샤드 간 집계 쿼리 구현**
   - 모든 샤드에서 데이터 조회
   - 애플리케이션 레벨에서 집계 처리
   - 성능 최적화 (병렬 처리, 캐싱)

#### 🔍 실습 검증
- [ ] 샤드별 데이터 분포 균등성 확인
- [ ] 신규 샤드 추가 시 데이터 재분배 테스트
- [ ] 크로스 샤드 쿼리 성능 측정

---

### Step 2.3: 오토스케일링과 리소스 관리
**학습 목표**: 동적 리소스 관리를 위한 오토스케일링 구현

#### 📖 이론 학습
- [ ] 수평적 자동 확장 (HPA) vs 수직적 자동 확장 (VPA)
- [ ] 스케일링 메트릭 (CPU, Memory, Custom Metrics)
- [ ] 스케일링 정책과 안정화 윈도우
- [ ] 예측적 스케일링과 스케줄 기반 스케일링

#### 🛠️ 실습 과제
1. **Kubernetes HPA 설정**
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: web-app-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: web-app
     minReplicas: 2
     maxReplicas: 10
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: Utilization
           averageUtilization: 70
     - type: Resource
       resource:
         name: memory
         target:
           type: Utilization
           averageUtilization: 80
   ```

2. **커스텀 메트릭 기반 스케일링**
   ```java
   // 큐 길이 기반 스케일링을 위한 메트릭 노출
   @RestController
   public class MetricsController {
       @Autowired
       private MeterRegistry meterRegistry;
       
       @GetMapping("/metrics/queue-length")
       public void updateQueueLength() {
           int queueLength = messageQueue.getQueueLength();
           Gauge.builder("queue.length")
               .register(meterRegistry)
               .set(queueLength);
       }
   }
   ```

3. **AWS Auto Scaling 구성**
   ```json
   {
     "AutoScalingGroupName": "web-app-asg",
     "MinSize": 2,
     "MaxSize": 20,
     "DesiredCapacity": 3,
     "LaunchTemplate": {
       "LaunchTemplateName": "web-app-template",
       "Version": "$Latest"
     },
     "TargetGroupARNs": ["arn:aws:elasticloadbalancing:..."],
     "HealthCheckType": "ELB",
     "HealthCheckGracePeriod": 300
   }
   ```

#### 📊 성능 분석
- [ ] 스케일링 이벤트 발생 조건과 반응 시간 측정
- [ ] 다양한 부하 패턴에서의 스케일링 효과 분석
- [ ] 비용 최적화를 위한 스케일링 정책 조정

---

### Step 2.4: 메시지 큐와 비동기 처리 기초
**학습 목표**: 비동기 메시징을 통한 시스템 분리와 확장성 향상

#### 📖 이론 학습
- [ ] 메시지 큐 패턴 (Point-to-Point, Publish-Subscribe)
- [ ] 메시지 브로커 비교 (RabbitMQ, Apache Kafka, Amazon SQS)
- [ ] 메시지 순서 보장과 중복 처리 문제
- [ ] 백프레셔(Backpressure)와 플로우 제어

#### 🛠️ 실습 과제
1. **RabbitMQ를 활용한 비동기 처리**
   ```java
   // 주문 처리 시스템
   @Component
   public class OrderProducer {
       @Autowired
       private RabbitTemplate rabbitTemplate;
       
       public void sendOrderEvent(OrderEvent event) {
           rabbitTemplate.convertAndSend("order.exchange", 
               "order.created", event);
       }
   }
   
   @RabbitListener(queues = "inventory.queue")
   @Component
   public class InventoryConsumer {
       public void processOrderEvent(OrderEvent event) {
           // 재고 확인 및 예약 로직
           inventoryService.reserveStock(event.getItems());
       }
   }
   ```

2. **Apache Kafka 이벤트 스트리밍**
   ```java
   @Service
   public class EventPublisher {
       @Autowired
       private KafkaTemplate<String, Object> kafkaTemplate;
       
       public void publishEvent(String topic, Object event) {
           kafkaTemplate.send(topic, event)
               .addCallback(
                   result -> log.info("Event sent successfully"),
                   failure -> log.error("Failed to send event", failure)
               );
       }
   }
   
   @KafkaListener(topics = "user-events", groupId = "notification-service")
   public void handleUserEvent(UserEvent event) {
       // 사용자 이벤트 처리
       notificationService.sendWelcomeEmail(event.getUserId());
   }
   ```

3. **메시지 처리 안정성 보장**
   - Dead Letter Queue 구현
   - 재시도 메커니즘과 Circuit Breaker 패턴
   - 메시지 중복 처리 방지 (Idempotency)

#### 🔍 Phase 2 종합 평가
**프로젝트**: 확장 가능한 주문 처리 시스템
- [ ] 읽기 복제본을 활용한 상품 조회 최적화
- [ ] 사용자 기반 샤딩으로 데이터 분산
- [ ] HPA를 통한 자동 스케일링 구현  
- [ ] 비동기 메시징으로 주문-재고-결제 프로세스 분리

---

## 🏛️ Phase 3: 고급 아키텍처 패턴 (9-12주)

### Step 3.1: 메시지 큐 고급 패턴
**학습 목표**: 이벤트 소싱과 CQRS 패턴을 통한 확장 가능한 아키텍처

#### 📖 이론 학습
- [ ] 이벤트 소싱(Event Sourcing) 개념과 장단점
- [ ] CQRS (Command Query Responsibility Segregation) 패턴
- [ ] 이벤트 스토어 설계와 스냅샷 생성
- [ ] 이벤트 재생(Event Replay)과 프로젝션 업데이트

#### 🛠️ 실습 과제
1. **이벤트 소싱 구현**
   ```java
   @Entity
   public class OrderAggregate {
       private String orderId;
       private List<OrderEvent> events = new ArrayList<>();
       
       public void createOrder(String customerId, List<OrderItem> items) {
           OrderCreatedEvent event = new OrderCreatedEvent(orderId, customerId, items);
           applyEvent(event);
       }
       
       public void payOrder(String paymentId) {
           OrderPaidEvent event = new OrderPaidEvent(orderId, paymentId);
           applyEvent(event);
       }
       
       private void applyEvent(OrderEvent event) {
           events.add(event);
           eventStore.save(event);
           eventPublisher.publish(event);
       }
       
       // 이벤트 스트림으로부터 현재 상태 복원
       public static OrderAggregate fromEvents(List<OrderEvent> events) {
           OrderAggregate order = new OrderAggregate();
           for (OrderEvent event : events) {
               order.apply(event);
           }
           return order;
       }
   }
   ```

2. **CQRS 패턴 구현**
   ```java
   // Command 모델
   @Service
   public class OrderCommandService {
       public void createOrder(CreateOrderCommand command) {
           OrderAggregate order = new OrderAggregate();
           order.createOrder(command.getCustomerId(), command.getItems());
           orderRepository.save(order);
       }
   }
   
   // Query 모델
   @Service
   public class OrderQueryService {
       public OrderView getOrder(String orderId) {
           return readOnlyOrderRepository.findOrderView(orderId);
       }
       
       public List<OrderSummary> getOrdersByCustomer(String customerId) {
           return readOnlyOrderRepository.findOrderSummariesByCustomerId(customerId);
       }
   }
   
   // 이벤트 핸들러로 읽기 모델 업데이트
   @EventHandler
   @Component
   public class OrderViewUpdater {
       @KafkaListener(topics = "order-events")
       public void handle(OrderCreatedEvent event) {
           OrderView view = new OrderView();
           view.setOrderId(event.getOrderId());
           view.setCustomerId(event.getCustomerId());
           view.setStatus("CREATED");
           orderViewRepository.save(view);
       }
   }
   ```

3. **이벤트 스토어 구현**
   - MySQL을 활용한 이벤트 저장소
   - 이벤트 스트림 조회 API
   - 스냅샷 생성 및 복원 메커니즘

---

### Step 3.2: 마이크로서비스 아키텍처 기초
**학습 목표**: 마이크로서비스 분해와 서비스 간 통신 패턴

#### 📖 이론 학습
- [ ] 마이크로서비스 분해 전략 (Domain-Driven Design)
- [ ] 서비스 간 통신 (동기 vs 비동기, REST vs gRPC)
- [ ] API Gateway 패턴과 서비스 디스커버리
- [ ] 분산 데이터 관리와 데이터 일관성

#### 🛠️ 실습 과제
1. **도메인별 마이크로서비스 분리**
   ```
   ┌─ User Service ─────┐    ┌─ Product Service ──┐
   │ - 사용자 관리      │    │ - 상품 관리        │
   │ - 인증/인가        │    │ - 카테고리 관리    │
   │ - 프로필 관리      │    │ - 재고 관리        │
   └────────────────────┘    └─────────────────────┘
            │                          │
            └──────────┬─────────────────┘
                      │
   ┌─ Order Service ───▼───┐    ┌─ Payment Service ──┐
   │ - 주문 생성/조회      │    │ - 결제 처리        │
   │ - 주문 상태 관리      │    │ - 결제 내역 조회   │
   │ - 배송 추적           │    │ - 환불 처리        │
   └───────────────────────┘    └─────────────────────┘
   ```

2. **Spring Cloud Gateway 구성**
   ```java
   @Configuration
   public class GatewayConfig {
       @Bean
       public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
           return builder.routes()
               .route("user-service", r -> r.path("/api/users/**")
                   .filters(f -> f
                       .stripPrefix(2)
                       .circuitBreaker(config -> config.setName("user-service-cb"))
                       .retry(retryConfig -> retryConfig.setRetries(3))
                   )
                   .uri("lb://user-service"))
               
               .route("order-service", r -> r.path("/api/orders/**")
                   .filters(f -> f
                       .stripPrefix(2)
                       .requestRateLimiter(config -> config
                           .setRateLimiter(redisRateLimiter())
                           .setKeyResolver(userKeyResolver())
                       )
                   )
                   .uri("lb://order-service"))
               .build();
       }
   }
   ```

3. **서비스 디스커버리 (Eureka) 설정**
   ```yaml
   # eureka-server application.yml
   server:
     port: 8761
   
   eureka:
     instance:
       hostname: localhost
     client:
       register-with-eureka: false
       fetch-registry: false
   
   # 마이크로서비스 application.yml
   eureka:
     client:
       service-url:
         defaultZone: http://localhost:8761/eureka/
     instance:
       prefer-ip-address: true
       lease-renewal-interval-in-seconds: 10
   ```

---

### Step 3.3: 분산 트랜잭션과 Saga 패턴
**학습 목표**: 마이크로서비스 환경에서의 트랜잭션 관리

#### 📖 이론 학습
- [ ] 분산 트랜잭션의 문제점과 2PC/3PC 프로토콜
- [ ] Saga 패턴 (Choreography vs Orchestration)
- [ ] 보상 트랜잭션(Compensating Transaction) 설계
- [ ] 최종 일관성(Eventual Consistency)과 BASE 모델

#### 🛠️ 실습 과제
1. **Choreography-based Saga 구현**
   ```java
   // Order Service
   @Service
   public class OrderSagaService {
       public void processOrder(CreateOrderRequest request) {
           try {
               Order order = createOrder(request);
               
               // 재고 예약 이벤트 발행
               eventPublisher.publish("inventory.reserve", 
                   new ReserveInventoryEvent(order.getId(), order.getItems()));
               
           } catch (Exception e) {
               compensateCreateOrder(request.getOrderId());
           }
       }
       
       @EventListener
       public void handleInventoryReserved(InventoryReservedEvent event) {
           // 결제 요청 이벤트 발행
           eventPublisher.publish("payment.process", 
               new ProcessPaymentEvent(event.getOrderId(), event.getAmount()));
       }
       
       @EventListener
       public void handlePaymentFailed(PaymentFailedEvent event) {
           // 보상 트랜잭션: 재고 예약 취소
           eventPublisher.publish("inventory.cancel", 
               new CancelInventoryEvent(event.getOrderId()));
           compensateCreateOrder(event.getOrderId());
       }
   }
   ```

2. **Orchestration-based Saga 구현**
   ```java
   @Component
   public class OrderSagaOrchestrator {
       @SagaOrchestrationStart
       public void processOrder(CreateOrderCommand command) {
           SagaInstance saga = sagaManager.begin("order-saga", command.getOrderId());
           
           saga.choreography()
               .step("reserve-inventory")
               .invokeParticipant("inventory-service")
               .withCompensation("cancel-inventory")
               
               .step("process-payment")
               .invokeParticipant("payment-service")
               .withCompensation("refund-payment")
               
               .step("confirm-order")
               .invokeLocal(this::confirmOrder)
               .withCompensation(this::cancelOrder)
               
               .execute();
       }
   }
   ```

3. **분산 트랜잭션 모니터링**
   - Saga 실행 상태 추적
   - 실패한 트랜잭션 재시도 메커니즘
   - 보상 트랜잭션 실행 로그

---

### Step 3.4: 서비스 메시와 관찰 가능성
**학습 목표**: 마이크로서비스 간 통신 최적화와 모니터링

#### 📖 이론 학습
- [ ] 서비스 메시 개념과 Istio/Linkerd 비교
- [ ] 사이드카 패턴과 프록시 기반 통신
- [ ] 분산 추적(Distributed Tracing)과 OpenTelemetry
- [ ] 서비스 간 보안 (mTLS, JWT, RBAC)

#### 🛠️ 실습 과제
1. **Istio 서비스 메시 구성**
   ```yaml
   # istio-gateway.yaml
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: ecommerce-gateway
   spec:
     selector:
       istio: ingressgateway
     servers:
     - port:
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - "*"
   
   # virtual-service.yaml
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: order-service-vs
   spec:
     http:
     - match:
       - uri:
           prefix: /api/orders
       route:
       - destination:
           host: order-service
         weight: 90
       - destination:
           host: order-service-v2
         weight: 10
       fault:
         delay:
           percentage:
             value: 0.1
           fixedDelay: 5s
   ```

2. **분산 추적 구현 (Jaeger + OpenTelemetry)**
   ```java
   @RestController
   public class OrderController {
       private final Tracer tracer;
       
       @PostMapping("/orders")
       public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
           Span span = tracer.spanBuilder("create-order")
               .setSpanKind(SpanKind.SERVER)
               .setAttribute("user.id", request.getUserId())
               .setAttribute("order.items.count", request.getItems().size())
               .startSpan();
           
           try (Scope scope = span.makeCurrent()) {
               Order order = orderService.createOrder(request);
               
               span.setAttribute("order.id", order.getId())
                   .setAttribute("order.amount", order.getTotalAmount())
                   .setStatus(StatusCode.OK);
               
               return ResponseEntity.ok(order);
               
           } catch (Exception e) {
               span.setStatus(StatusCode.ERROR, e.getMessage())
                   .recordException(e);
               throw e;
           } finally {
               span.end();
           }
       }
   }
   ```

3. **카나리 배포와 A/B 테스트**
   - 트래픽 분할을 통한 점진적 배포
   - 성능 메트릭 기반 자동 롤백
   - 사용자 세그먼트별 기능 플래그

#### 🔍 Phase 3 종합 평가
**프로젝트**: 완전한 마이크로서비스 전자상거래 플랫폼
- [ ] 도메인별로 분리된 독립적인 서비스
- [ ] 이벤트 소싱과 CQRS를 활용한 주문 시스템
- [ ] Saga 패턴으로 분산 트랜잭션 처리
- [ ] 서비스 메시를 통한 안전한 서비스 간 통신

---

## 🔧 Phase 4: 운영과 최적화

### Step 4.1: 성능 모니터링과 관찰 가능성
**학습 목표**: 프로덕션 환경의 종합적인 모니터링 시스템 구축

#### 📖 이론 학습
- [ ] 관찰 가능성의 3요소 (Metrics, Logs, Traces)
- [ ] SLI/SLO/SLA 정의와 에러 버짓 관리
- [ ] 알림 피로도 방지를 위한 알림 정책 설계
- [ ] 성능 병목점 식별과 최적화 방법론

#### 🛠️ 실습 과제
1. **종합 모니터링 스택 구축**
   ```yaml
   # prometheus-stack.yaml
   version: '3.8'
   services:
     prometheus:
       image: prom/prometheus:latest
       ports:
         - "9090:9090"
       volumes:
         - ./prometheus.yml:/etc/prometheus/prometheus.yml
         - ./alert_rules.yml:/etc/prometheus/alert_rules.yml
     
     grafana:
       image: grafana/grafana:latest
       ports:
         - "3000:3000"
       environment:
         - GF_SECURITY_ADMIN_PASSWORD=admin
       volumes:
         - grafana-storage:/var/lib/grafana
         - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
     
     alertmanager:
       image: prom/alertmanager:latest
       ports:
         - "9093:9093"
       volumes:
         - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
   ```

2. **커스텀 대시보드 생성**
   ```json
   {
     "dashboard": {
       "title": "E-commerce System Overview",
       "panels": [
         {
           "title": "Request Rate",
           "type": "graph",
           "targets": [
             {
               "expr": "rate(http_requests_total[5m])",
               "legendFormat": "{{service}}"
             }
           ]
         },
         {
           "title": "Error Rate",
           "type": "stat",
           "targets": [
             {
               "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m]) * 100"
             }
           ]
         },
         {
           "title": "Response Time P95",
           "type": "graph",
           "targets": [
             {
               "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
             }
           ]
         }
       ]
     }
   }
   ```

3. **지능형 알림 시스템 구현**
   ```yaml
   # alert_rules.yml
   groups:
   - name: ecommerce.rules
     rules:
     - alert: HighErrorRate
       expr: |
         (
           rate(http_requests_total{status=~"5.."}[5m]) /
           rate(http_requests_total[5m])
         ) > 0.05
       for: 5m
       labels:
         severity: critical
         service: "{{ $labels.service }}"
       annotations:
         summary: "High error rate detected"
         description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
         runbook_url: "https://wiki.company.com/runbooks/high-error-rate"
     
     - alert: DatabaseConnectionPoolExhausted
       expr: |
         hikaricp_connections_active >= hikaricp_connections_max
       for: 1m
       labels:
         severity: critical
       annotations:
         summary: "Database connection pool exhausted"
         description: "All database connections are in use for {{ $labels.service }}"
   ```

---

### Step 4.2: 보안과 컴플라이언스
**학습 목표**: 대규모 시스템의 보안 강화와 규정 준수

#### 📖 이론 학습
- [ ] 제로 트러스트 보안 모델
- [ ] API 보안 (인증, 인가, Rate Limiting, API Gateway 보안)
- [ ] 데이터 암호화 (전송 중/저장 중 암호화)
- [ ] 규정 준수 (GDPR, PCI DSS) 요구사항

#### 🛠️ 실습 과제
1. **OAuth 2.0 + JWT 인증 시스템**
   ```java
   @Configuration
   @EnableWebSecurity
   public class SecurityConfig {
       
       @Bean
       public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
           http
               .authorizeHttpRequests(authz -> authz
                   .requestMatchers("/api/public/**").permitAll()
                   .requestMatchers("/api/admin/**").hasRole("ADMIN")
                   .requestMatchers("/api/orders/**").hasAnyRole("USER", "PREMIUM")
                   .anyRequest().authenticated()
               )
               .oauth2ResourceServer(oauth2 -> oauth2
                   .jwt(jwt -> jwt.jwtDecoder(jwtDecoder()))
               )
               .sessionManagement(session -> session
                   .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
               );
           
           return http.build();
       }
   }
   ```

2. **데이터 암호화 구현**
   ```java
   @Entity
   public class User {
       @Id
       private String userId;
       
       @Column
       @Convert(converter = EncryptedStringConverter.class)
       private String email;
       
       @Column
       @Convert(converter = EncryptedStringConverter.class)
       private String phoneNumber;
   }
   
   @Converter
   public class EncryptedStringConverter implements AttributeConverter<String, String> {
       @Override
       public String convertToDatabaseColumn(String attribute) {
           return encryptionService.encrypt(attribute);
       }
       
       @Override
       public String convertToEntityAttribute(String dbData) {
           return encryptionService.decrypt(dbData);
       }
   }
   ```

3. **보안 감사 로그 시스템**
   ```java
   @Component
   @Slf4j
   public class SecurityAuditLogger {
       
       @EventListener
       public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
           String username = event.getAuthentication().getName();
           String clientIp = getClientIp(event);
           
           auditLog.info("Authentication successful: user={}, ip={}, timestamp={}", 
               username, clientIp, Instant.now());
       }
       
       @EventListener
       public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
           String username = event.getAuthentication().getName();
           String reason = event.getException().getMessage();
           
           auditLog.warn("Authentication failed: user={}, reason={}, timestamp={}", 
               username, reason, Instant.now());
       }
   }
   ```

---

### Step 4.3: 성능 최적화와 튜닝
**학습 목표**: 시스템 성능 병목점 식별과 최적화

#### 📖 이론 학습
- [ ] 성능 프로파일링 도구와 기법
- [ ] JVM 튜닝과 가비지 컬렉션 최적화
- [ ] 데이터베이스 쿼리 최적화와 인덱스 전략
- [ ] 네트워크 최적화와 CDN 활용

#### 🛠️ 실습 과제
1. **애플리케이션 성능 프로파일링**
   ```java
   // JProfiler, YourKit, 또는 Async Profiler 사용
   @Component
   public class PerformanceProfiler {
       
       @EventListener
       @Async
       public void profileSlowRequests(RequestProcessedEvent event) {
           if (event.getDuration() > Duration.ofSeconds(1)) {
               // 느린 요청에 대한 상세 프로파일링
               ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(true, true);
               profilerService.analyzeThreadDump(threadInfos, event);
           }
       }
   }
   ```

2. **데이터베이스 성능 최적화**
   ```sql
   -- 인덱스 전략 최적화
   CREATE INDEX idx_orders_user_id_created_at 
   ON orders(user_id, created_at DESC);
   
   CREATE INDEX idx_products_category_status 
   ON products(category_id, status) 
   WHERE status = 'ACTIVE';
   
   -- 쿼리 성능 분석
   EXPLAIN ANALYZE 
   SELECT o.*, u.username 
   FROM orders o 
   JOIN users u ON o.user_id = u.user_id 
   WHERE o.created_at >= NOW() - INTERVAL '30 days'
   ORDER BY o.created_at DESC 
   LIMIT 100;
   ```

3. **JVM 메모리 최적화**
   ```bash
   # JVM 튜닝 옵션
   -Xms2g -Xmx4g
   -XX:+UseG1GC
   -XX:MaxGCPauseMillis=200
   -XX:+HeapDumpOnOutOfMemoryError
   -XX:HeapDumpPath=/app/logs/
   -XX:+UseStringDeduplication
   -XX:+PrintGCDetails
   -XX:+PrintGCTimeStamps
   -Xloggc:/app/logs/gc.log
   ```

4. **캐시 최적화 전략**
   ```java
   @Service
   public class OptimizedProductService {
       
       // 지역성을 고려한 캐시 키 설계
       @Cacheable(value = "products", 
                  key = "#productId + ':' + #locale.language",
                  condition = "#locale.language != null")
       public ProductDto getProduct(String productId, Locale locale) {
           return productRepository.findByIdAndLocale(productId, locale);
       }
       
       // 배치 로딩으로 N+1 문제 해결
       @Cacheable(value = "product-batch")
       public Map<String, ProductDto> getProducts(List<String> productIds) {
           return productRepository.findByIdIn(productIds)
               .stream()
               .collect(Collectors.toMap(Product::getId, ProductDto::from));
       }
   }
   ```

---

## 📚 추가 학습 자료

### 📖 필수 도서
1. **"Designing Data-Intensive Applications"** - Martin Kleppmann
   - 분산 시스템의 핵심 개념과 패턴
   - 데이터 모델링과 저장 시스템
   - 분산 처리와 일관성 모델

2. **"Building Microservices"** - Sam Newman
   - 마이크로서비스 아키텍처 설계
   - 서비스 분해 전략
   - 운영과 모니터링

3. **"Release It!"** - Michael Nygard
   - 안정성 패턴과 장애 대응
   - 운영 환경에서의 베스트 프랙티스
   - 성능 최적화 기법

### 🌐 온라인 코스
1. **"Distributed Systems"** - MIT 6.824
2. **"System Design Interview"** - ByteByteGo
3. **"Kubernetes in Action"** - Manning Publications
4. **"Apache Kafka Series"** - Udemy

### 🛠️ 실습 플랫폼
1. **Docker Hub**: 컨테이너 이미지 저장소
2. **GitHub Actions**: CI/CD 파이프라인
3. **AWS Free Tier**: 클라우드 리소스 실습
4. **Kubernetes Playground**: 온라인 K8s 환경
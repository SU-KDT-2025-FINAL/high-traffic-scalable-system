# 대규모 트래픽 처리를 위한 데이터베이스 성능 최적화 및 분산 처리 가이드

## 목차
1. [데이터베이스 병목 현상과 해결 방향](#데이터베이스-병목-현상과-해결-방향)
2. [읽기 성능 최적화 전략](#읽기-성능-최적화-전략)
3. [쓰기 성능 최적화 전략](#쓰기-성능-최적화-전략)
4. [데이터베이스 샤딩과 파티셔닝](#데이터베이스-샤딩과-파티셔닝)
5. [분산 트랜잭션 처리](#분산-트랜잭션-처리)
6. [NoSQL vs RDBMS 선택 기준](#nosql-vs-rdbms-선택-기준)
7. [데이터 일관성과 성능 트레이드오프](#데이터-일관성과-성능-트레이드오프)
8. [실제 구현 예시](#실제-구현-예시)
9. [모니터링 및 튜닝](#모니터링-및-튜닝)
10. [성능 최적화 체크리스트](#성능-최적화-체크리스트)

## 데이터베이스 병목 현상과 해결 방향

### 1. 주요 병목 지점
- **단일 DB 서버**: 모든 요청이 하나의 DB로 집중
- **디스크 I/O**: 느린 디스크 접근으로 인한 지연
- **락 경합**: 동시 접근 시 발생하는 데이터 락
- **네트워크 지연**: DB 서버와 애플리케이션 간 통신 지연

### 2. 해결 방향
```
애플리케이션 서버 (다중)
        ↓
로드 밸런서 (DB 요청 분산)
        ↓
┌─────────────┬─────────────┬─────────────┐
│ Read Replica│ Read Replica│ Master DB   │
│     (읽기)   │     (읽기)   │   (쓰기)    │
└─────────────┴─────────────┴─────────────┘
```

## 읽기 성능 최적화 전략

### 1. Read Replica 구성
```yaml
# Docker Compose 예시
version: '3.8'
services:
  db-master:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: myapp
    volumes:
      - master-data:/var/lib/mysql
    ports:
      - "3306:3306"
  
  db-replica-1:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: myapp
    volumes:
      - replica1-data:/var/lib/mysql
    ports:
      - "3307:3306"
    depends_on:
      - db-master
```

### 2. 읽기 전용 서비스 분리
```java
@Service
public class ProductService {
    @Autowired
    @Qualifier("masterDataSource")
    private JdbcTemplate masterJdbcTemplate;
    
    @Autowired
    @Qualifier("replicaDataSource")
    private JdbcTemplate replicaJdbcTemplate;
    
    // 읽기 전용 - Replica 사용
    public List<Product> getProducts() {
        return replicaJdbcTemplate.query(
            "SELECT * FROM products", 
            new ProductRowMapper()
        );
    }
    
    // 쓰기 작업 - Master 사용
    public void createProduct(Product product) {
        masterJdbcTemplate.update(
            "INSERT INTO products (name, price) VALUES (?, ?)",
            product.getName(), product.getPrice()
        );
    }
}
```

### 3. 인덱스 최적화
```sql
-- 복합 인덱스 생성
CREATE INDEX idx_user_status_created 
ON orders (user_id, status, created_at);

-- 부분 인덱스 (조건부 인덱스)
CREATE INDEX idx_active_users 
ON users (email) 
WHERE status = 'ACTIVE';

-- 커버링 인덱스
CREATE INDEX idx_product_cover 
ON products (category_id, price, name, description);
```

## 쓰기 성능 최적화 전략

### 1. 배치 처리
```java
@Service
public class OrderBatchService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void batchInsertOrders(List<Order> orders) {
        String sql = "INSERT INTO orders (user_id, product_id, quantity, price) VALUES (?, ?, ?, ?)";
        
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Order order = orders.get(i);
                ps.setLong(1, order.getUserId());
                ps.setLong(2, order.getProductId());
                ps.setInt(3, order.getQuantity());
                ps.setBigDecimal(4, order.getPrice());
            }
            
            @Override
            public int getBatchSize() {
                return orders.size();
            }
        });
    }
}
```

### 2. 비동기 쓰기 처리
```java
@Component
public class AsyncOrderProcessor {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Async
    public CompletableFuture<Void> processOrderAsync(Order order) {
        // 주문 이벤트를 Kafka로 전송
        OrderEvent event = new OrderEvent(order);
        kafkaTemplate.send("order-events", event);
        
        return CompletableFuture.completedFuture(null);
    }
}

@KafkaListener(topics = "order-events")
public void handleOrderEvent(OrderEvent event) {
    // 비동기로 DB 저장 처리
    orderRepository.save(event.getOrder());
}
```

## 데이터베이스 샤딩과 파티셔닝

### 1. 수평 샤딩 (Horizontal Sharding)
```java
@Component
public class ShardingKeyGenerator {
    
    private static final int SHARD_COUNT = 4;
    
    public int getShardKey(Long userId) {
        return (int) (userId % SHARD_COUNT);
    }
    
    public String getShardDataSource(Long userId) {
        int shardKey = getShardKey(userId);
        return "dataSource" + shardKey;
    }
}

@Service
public class UserShardingService {
    
    @Autowired
    private ShardingKeyGenerator shardingKeyGenerator;
    
    @Autowired
    private Map<String, JdbcTemplate> shardDataSources;
    
    public User findUserById(Long userId) {
        String dataSourceKey = shardingKeyGenerator.getShardDataSource(userId);
        JdbcTemplate jdbcTemplate = shardDataSources.get(dataSourceKey);
        
        return jdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new UserRowMapper(),
            userId
        );
    }
}
```

### 2. 파티셔닝 예시
```sql
-- 날짜 기반 파티셔닝
CREATE TABLE orders (
    id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    -- 기타 컬럼들
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 해시 기반 파티셔닝
CREATE TABLE user_activities (
    id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    activity_type VARCHAR(50),
    created_at DATETIME NOT NULL
) PARTITION BY HASH(user_id) PARTITIONS 8;
```

## 분산 트랜잭션 처리

### 1. 2PC (Two-Phase Commit) 패턴
```java
@Service
@Transactional
public class DistributedTransactionService {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private OrderService orderService;
    
    public void processOrder(OrderRequest request) {
        try {
            // Phase 1: Prepare
            paymentService.preparePayment(request.getPaymentInfo());
            inventoryService.reserveInventory(request.getProducts());
            
            // Phase 2: Commit
            Long orderId = orderService.createOrder(request);
            paymentService.commitPayment(request.getPaymentInfo());
            inventoryService.commitReservation(request.getProducts());
            
        } catch (Exception e) {
            // Rollback 모든 변경사항
            paymentService.rollbackPayment(request.getPaymentInfo());
            inventoryService.rollbackReservation(request.getProducts());
            throw new OrderProcessingException("주문 처리 실패", e);
        }
    }
}
```

### 2. Saga 패턴
```java
@Component
public class OrderSagaOrchestrator {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private NotificationService notificationService;
    
    public void executeOrderSaga(OrderRequest request) {
        SagaTransaction saga = SagaTransaction.builder()
            .addStep(
                () -> paymentService.processPayment(request.getPaymentInfo()),
                () -> paymentService.refundPayment(request.getPaymentInfo())
            )
            .addStep(
                () -> inventoryService.reduceInventory(request.getProducts()),
                () -> inventoryService.restoreInventory(request.getProducts())
            )
            .addStep(
                () -> notificationService.sendOrderConfirmation(request.getUserId()),
                () -> notificationService.sendOrderCancellation(request.getUserId())
            )
            .build();
            
        saga.execute();
    }
}
```

## NoSQL vs RDBMS 선택 기준

### 1. 사용 사례별 선택 가이드

| 특성 | RDBMS | NoSQL |
|------|-------|--------|
| **데이터 일관성** | 강한 일관성 필요 시 | 최종 일관성 허용 가능 시 |
| **스키마** | 고정된 스키마 | 유연한 스키마 |
| **확장성** | 수직 확장 중심 | 수평 확장 중심 |
| **트랜잭션** | ACID 트랜잭션 필요 | 단순한 읽기/쓰기 중심 |
| **쿼리 복잡도** | 복잡한 조인, 집계 | 단순한 키-값 조회 |

### 2. 하이브리드 아키텍처
```java
@Service
public class HybridDataService {
    
    // 트랜잭션 데이터는 RDBMS
    @Autowired
    private OrderRepository orderRepository; // JPA/MySQL
    
    // 캐시 데이터는 NoSQL
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 로그/분석 데이터는 Document DB
    @Autowired
    private MongoTemplate mongoTemplate;
    
    public Order createOrder(OrderRequest request) {
        // 1. 트랜잭션 데이터 저장 (RDBMS)
        Order order = orderRepository.save(new Order(request));
        
        // 2. 캐시 업데이트 (Redis)
        redisTemplate.opsForValue().set(
            "order:" + order.getId(), 
            order, 
            Duration.ofHours(24)
        );
        
        // 3. 분석 데이터 저장 (MongoDB)
        OrderAnalytics analytics = new OrderAnalytics(order);
        mongoTemplate.save(analytics);
        
        return order;
    }
}
```

## 데이터 일관성과 성능 트레이드오프

### 1. CAP 정리에 따른 선택
```java
// 강한 일관성 (Consistency) - 성능 희생
@Transactional(isolation = Isolation.SERIALIZABLE)
public void updateAccountBalance(Long accountId, BigDecimal amount) {
    Account account = accountRepository.findByIdWithLock(accountId);
    account.updateBalance(amount);
    accountRepository.save(account);
}

// 최종 일관성 (Eventually Consistent) - 성능 우선
@Async
public void updateAccountBalanceAsync(Long accountId, BigDecimal amount) {
    // 이벤트 발행으로 비동기 처리
    eventPublisher.publishEvent(new BalanceUpdateEvent(accountId, amount));
}
```

### 2. 읽기 전용 복제본 활용
```java
@Service
public class BalanceService {
    
    // 실시간 정확한 잔액 (Master DB)
    public BigDecimal getCurrentBalance(Long accountId) {
        return masterAccountRepository.findById(accountId)
            .map(Account::getBalance)
            .orElse(BigDecimal.ZERO);
    }
    
    // 빠른 잔액 조회 (Read Replica - 약간의 지연 허용)
    public BigDecimal getApproximateBalance(Long accountId) {
        return replicaAccountRepository.findById(accountId)
            .map(Account::getBalance)
            .orElse(BigDecimal.ZERO);
    }
}
```

## 실제 구현 예시

### 1. Spring Boot + MySQL Master-Slave 설정
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("master", masterDataSource());
        dataSourceMap.put("slave", slaveDataSource());
        
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        
        return routingDataSource;
    }
}

// 읽기/쓰기 라우팅
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOnlyConnection {
}

@Aspect
@Component
public class DataSourceAspect {
    
    @Before("@annotation(readOnlyConnection)")
    public void setReadOnlyDataSource(ReadOnlyConnection readOnlyConnection) {
        DataSourceContextHolder.setDataSourceKey("slave");
    }
    
    @After("@annotation(readOnlyConnection)")
    public void clearDataSource() {
        DataSourceContextHolder.clear();
    }
}
```

### 2. Redis를 활용한 분산 캐시
```java
@Service
public class ProductCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private ProductRepository productRepository;
    
    public Product getProduct(Long productId) {
        String cacheKey = "product:" + productId;
        
        // 1. 캐시에서 조회
        Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cachedProduct != null) {
            return cachedProduct;
        }
        
        // 2. DB에서 조회
        Product product = productRepository.findById(productId).orElse(null);
        if (product != null) {
            // 3. 캐시에 저장 (TTL 1시간)
            redisTemplate.opsForValue().set(cacheKey, product, Duration.ofHours(1));
        }
        
        return product;
    }
    
    // 상품 업데이트 시 캐시 무효화
    public void updateProduct(Product product) {
        productRepository.save(product);
        
        String cacheKey = "product:" + product.getId();
        redisTemplate.delete(cacheKey);
    }
}
```

## 모니터링 및 튜닝

### 1. 데이터베이스 성능 지표
```yaml
# Prometheus 모니터링 설정
- name: mysql-exporter
  image: prom/mysqld-exporter
  env:
    - name: DATA_SOURCE_NAME
      value: "user:password@(mysql:3306)/"
  ports:
    - containerPort: 9104
```

### 2. 슬로우 쿼리 모니터링
```sql
-- MySQL 슬로우 쿼리 로그 활성화
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- 슬로우 쿼리 분석
SELECT 
    query_time,
    lock_time,
    rows_sent,
    rows_examined,
    sql_text
FROM mysql.slow_log
WHERE start_time >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY query_time DESC
LIMIT 10;
```

### 3. 자동 인덱스 추천
```java
@Service
public class QueryAnalysisService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public List<IndexRecommendation> analyzeQueries() {
        // 실행 계획 분석
        String sql = "EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE user_id = ? AND status = ?";
        
        Map<String, Object> plan = jdbcTemplate.queryForMap(sql, 123L, "PENDING");
        
        // 인덱스 추천 로직
        return generateIndexRecommendations(plan);
    }
}
```

## 성능 최적화 체크리스트

### 1. 읽기 성능 최적화
- [ ] Read Replica 구성 완료
- [ ] 적절한 인덱스 생성 및 관리
- [ ] 쿼리 최적화 (N+1 문제 해결)
- [ ] 캐시 전략 수립 (Redis, CDN)
- [ ] 커넥션 풀 튜닝

### 2. 쓰기 성능 최적화
- [ ] 배치 처리 구현
- [ ] 비동기 처리 도입
- [ ] 불필요한 인덱스 제거
- [ ] 파티셔닝 적용
- [ ] 트랜잭션 범위 최소화

### 3. 확장성 확보
- [ ] 샤딩 전략 수립 및 구현
- [ ] 분산 트랜잭션 처리 방안
- [ ] 데이터 일관성 정책 정의
- [ ] 장애 복구 시나리오 준비
- [ ] 모니터링 및 알림 체계 구축

### 4. 운영 최적화
- [ ] 슬로우 쿼리 모니터링
- [ ] 리소스 사용량 추적
- [ ] 자동 백업 및 복구 체계
- [ ] 데이터베이스 버전 관리
- [ ] 성능 테스트 자동화

## 모범 사례 요약

### 1. 아키텍처 설계
- **읽기/쓰기 분리**: Master-Slave 구조로 읽기 성능 향상
- **적절한 샤딩**: 데이터 특성에 맞는 샤딩 키 선택
- **캐시 활용**: 다층 캐시 구조로 DB 부하 감소

### 2. 성능 최적화
- **인덱스 전략**: 쿼리 패턴 분석 후 최적 인덱스 생성
- **배치 처리**: 대량 데이터 처리 시 배치 연산 활용
- **비동기 처리**: 즉시 응답이 불필요한 작업의 비동기화

### 3. 운영 관리
- **지속적 모니터링**: 성능 지표 실시간 추적
- **자동화**: 백업, 복구, 스케일링 자동화
- **장애 대응**: 빠른 감지와 자동 복구 체계

이러한 전략을 통해 대규모 트래픽 상황에서도 안정적이고 효율적인 데이터베이스 운영이 가능합니다.
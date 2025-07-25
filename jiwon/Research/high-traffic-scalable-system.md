# 고트래픽 확장 가능한 시스템 완전 가이드

## 목차
1. [시스템 확장성 개념과 원리](#1-시스템-확장성-개념과-원리)
2. [분산 시스템 아키텍처](#2-분산-시스템-아키텍처)
3. [로드 밸런싱과 트래픽 분산](#3-로드-밸런싱과-트래픽-분산)
4. [캐싱 전략과 구현](#4-캐싱-전략과-구현)
5. [데이터베이스 확장 기법](#5-데이터베이스-확장-기법)
6. [오토스케일링과 리소스 관리](#6-오토스케일링과-리소스-관리)
7. [메시지 큐와 비동기 처리](#7-메시지-큐와-비동기-처리)
8. [마이크로서비스 아키텍처](#8-마이크로서비스-아키텍처)
9. [성능 모니터링과 관찰 가능성](#9-성능-모니터링과-관찰-가능성)
10. [실전 구현 예제](#10-실전-구현-예제)

---

## 1. 시스템 확장성 개념과 원리

### 1.1 확장성(Scalability)이란?

확장성은 시스템이 증가하는 요구사항(사용자 수, 데이터 양, 트랜잭션 수)에 대응하여 성능을 유지하거나 향상시킬 수 있는 능력입니다.

#### 수직 확장(Vertical Scaling / Scale Up)
- **개념**: 기존 서버의 성능을 향상시키는 방법
- **방법**: CPU, 메모리, 디스크 용량 증가
- **장점**: 구현이 간단, 데이터 일관성 유지 용이
- **단점**: 물리적 한계 존재, 단일 장애점, 비용 급증
- **적용 예시**: 
  ```
  기존: 4코어 CPU, 8GB RAM
  확장: 16코어 CPU, 64GB RAM
  ```

#### 수평 확장(Horizontal Scaling / Scale Out)
- **개념**: 서버의 수를 늘려 부하를 분산하는 방법
- **방법**: 동일한 역할을 하는 서버를 여러 대 추가
- **장점**: 이론적으로 무한 확장 가능, 내결함성 향상, 비용 효율적
- **단점**: 복잡성 증가, 데이터 일관성 문제, 네트워크 오버헤드
- **적용 예시**:
  ```
  기존: 서버 1대
  확장: 서버 10대로 부하 분산
  ```

### 1.2 확장성 설계 원칙

#### Stateless 설계
모든 서버가 동일한 요청을 처리할 수 있도록 상태 정보를 서버에 저장하지 않는 설계 방식입니다.

**Stateful vs Stateless 비교:**
```java
// Stateful (잘못된 예)
public class StatefulService {
    private String userSession; // 서버에 상태 저장
    
    public void processRequest(String request) {
        // 특정 서버에서만 처리 가능
        userSession = extractSession(request);
    }
}

// Stateless (올바른 예)
public class StatelessService {
    public void processRequest(String request, String sessionToken) {
        // 어떤 서버에서든 처리 가능
        String userSession = extractSession(sessionToken);
    }
}
```

#### 분산 처리 가능 설계
작업을 여러 노드에서 병렬로 처리할 수 있도록 설계합니다.

#### 장애 격리(Fault Isolation)
한 부분의 장애가 전체 시스템에 영향을 주지 않도록 격리합니다.

---

## 2. 분산 시스템 아키텍처

### 2.1 분산 시스템의 특징

#### CAP 정리 (CAP Theorem)
분산 시스템에서는 다음 세 가지 중 최대 두 가지만 보장할 수 있습니다:

- **Consistency(일관성)**: 모든 노드가 동시에 같은 데이터를 보여줌
- **Availability(가용성)**: 시스템이 항상 읽기/쓰기 요청에 응답
- **Partition Tolerance(분할 내성)**: 네트워크 분할이 발생해도 시스템이 계속 작동

**실제 적용 예시:**
- **CP 시스템**: MongoDB, HBase (일관성 + 분할내성)
- **AP 시스템**: Cassandra, DynamoDB (가용성 + 분할내성)
- **CA 시스템**: MySQL, PostgreSQL (일관성 + 가용성, 단일 노드)

### 2.2 분산 시스템 패턴

#### Master-Slave 패턴
```yaml
# Master-Slave Redis 설정 예시
version: '3.8'
services:
  redis-master:
    image: redis:7.0
    command: redis-server --port 6379
    ports:
      - "6379:6379"
  
  redis-slave-1:
    image: redis:7.0
    command: redis-server --port 6379 --slaveof redis-master 6379
    depends_on:
      - redis-master
  
  redis-slave-2:
    image: redis:7.0
    command: redis-server --port 6379 --slaveof redis-master 6379
    depends_on:
      - redis-master
```

#### Peer-to-Peer 패턴
모든 노드가 동등한 역할을 수행하는 구조입니다.

#### 샤딩(Sharding) 패턴
데이터를 여러 노드에 분산 저장하는 방식입니다.

```java
// 일관된 해싱을 사용한 샤딩 예시
public class ConsistentHashSharding {
    private TreeMap<Long, String> ring = new TreeMap<>();
    
    public void addNode(String node) {
        for (int i = 0; i < 150; i++) { // 가상 노드 생성
            long hash = hash(node + ":" + i);
            ring.put(hash, node);
        }
    }
    
    public String getNode(String key) {
        long hash = hash(key);
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
        if (entry == null) {
            entry = ring.firstEntry();
        }
        return entry.getValue();
    }
}
```

---

## 3. 로드 밸런싱과 트래픽 분산

### 3.1 로드 밸런싱 알고리즘

#### Round Robin
순서대로 서버에 요청을 분배합니다.
```nginx
upstream backend {
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}
```

#### Weighted Round Robin
서버별로 가중치를 부여하여 성능에 따라 요청을 분배합니다.
```nginx
upstream backend {
    server server1.example.com weight=3;
    server server2.example.com weight=2;
    server server3.example.com weight=1;
}
```

#### Least Connections
현재 연결 수가 가장 적은 서버로 요청을 전달합니다.
```nginx
upstream backend {
    least_conn;
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}
```

#### IP Hash
클라이언트 IP를 기반으로 특정 서버로 요청을 전달합니다.
```nginx
upstream backend {
    ip_hash;
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}
```

### 3.2 로드 밸런서 계층

#### L4 로드 밸런서 (Layer 4)
TCP/UDP 레벨에서 동작하며, IP 주소와 포트를 기반으로 트래픽을 분산합니다.

**특징:**
- 빠른 처리 속도
- 애플리케이션 내용 인식 불가
- 세션 지속성 제한적

**구현 예시 (HAProxy):**
```
backend webservers
    balance roundrobin
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
    server web3 192.168.1.12:80 check
```

#### L7 로드 밸런서 (Layer 7)
HTTP 레벨에서 동작하며, URL, 헤더, 쿠키 등을 기반으로 트래픽을 분산합니다.

**특징:**
- 애플리케이션 내용 기반 라우팅 가능
- SSL 터미네이션 지원
- 더 정교한 트래픽 제어

**구현 예시 (NGINX):**
```nginx
location /api/ {
    proxy_pass http://api-backend;
}

location /static/ {
    proxy_pass http://static-backend;
}

location / {
    proxy_pass http://web-backend;
}
```

### 3.3 글로벌 로드 밸런싱

#### DNS 기반 로드 밸런싱
```
# 지역별 DNS 레코드
api.example.com. 300 IN A 1.2.3.4   # 아시아 서버
api.example.com. 300 IN A 5.6.7.8   # 유럽 서버
api.example.com. 300 IN A 9.10.11.12 # 미국 서버
```

#### CDN을 활용한 트래픽 분산
CloudFlare, AWS CloudFront 등을 활용하여 전 세계적으로 트래픽을 분산합니다.

---

## 4. 캐싱 전략과 구현

### 4.1 캐싱 레벨

#### 브라우저 캐시
클라이언트 브라우저에서 리소스를 저장합니다.
```http
Cache-Control: public, max-age=3600
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

#### CDN 캐시
전 세계에 분산된 서버에서 정적 콘텐츠를 캐싱합니다.

#### 리버스 프록시 캐시
애플리케이션 서버 앞단에서 응답을 캐싱합니다.
```nginx
location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 302 10m;
    proxy_cache_valid 404 1m;
    proxy_pass http://backend;
}
```

#### 애플리케이션 레벨 캐시
메모리나 분산 캐시에 데이터를 저장합니다.

### 4.2 캐싱 패턴

#### Cache-Aside (Lazy Loading)
애플리케이션이 직접 캐시를 관리하는 방식입니다.

```java
@Service
public class ProductService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private ProductRepository productRepository;
    
    public Product getProduct(String productId) {
        String cacheKey = "product:" + productId;
        
        // 1. 캐시에서 조회 시도
        Product product = (Product) redisTemplate.opsForValue().get(cacheKey);
        
        if (product == null) {
            // 2. 캐시 미스 시 DB에서 조회
            product = productRepository.findById(productId).orElse(null);
            
            if (product != null) {
                // 3. 조회된 데이터를 캐시에 저장
                redisTemplate.opsForValue().set(
                    cacheKey, product, 
                    Duration.ofMinutes(10)
                );
            }
        }
        
        return product;
    }
    
    public void updateProduct(Product product) {
        // 1. DB 업데이트
        productRepository.save(product);
        
        // 2. 캐시 무효화
        String cacheKey = "product:" + product.getId();
        redisTemplate.delete(cacheKey);
    }
}
```

#### Write-Through
데이터를 저장할 때 DB와 캐시에 동시에 저장하는 방식입니다.

```java
public void updateProduct(Product product) {
    // 1. DB와 캐시에 동시 저장
    productRepository.save(product);
    
    String cacheKey = "product:" + product.getId();
    redisTemplate.opsForValue().set(
        cacheKey, product, 
        Duration.ofMinutes(10)
    );
}
```

#### Write-Behind (Write-Back)
캐시에만 먼저 저장하고, 나중에 비동기적으로 DB에 저장하는 방식입니다.

```java
@Component
public class WriteBehindCache {
    private final Map<String, Product> writeBuffer = new ConcurrentHashMap<>();
    
    @Scheduled(fixedDelay = 5000) // 5초마다 실행
    public void flushToDatabase() {
        if (!writeBuffer.isEmpty()) {
            List<Product> products = new ArrayList<>(writeBuffer.values());
            productRepository.saveAll(products);
            writeBuffer.clear();
        }
    }
    
    public void updateProduct(Product product) {
        // 캐시에만 저장
        String cacheKey = "product:" + product.getId();
        redisTemplate.opsForValue().set(cacheKey, product);
        
        // 나중에 DB에 저장하기 위해 버퍼에 추가
        writeBuffer.put(cacheKey, product);
    }
}
```

### 4.3 분산 캐시 구현

#### Redis Cluster 설정
```yaml
version: '3.8'
services:
  redis-node-1:
    image: redis:7.0
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 7001
    ports:
      - "7001:7001"
    volumes:
      - redis-data-1:/data

  redis-node-2:
    image: redis:7.0
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 7002
    ports:
      - "7002:7002"
    volumes:
      - redis-data-2:/data

  redis-node-3:
    image: redis:7.0
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 7003
    ports:
      - "7003:7003"
    volumes:
      - redis-data-3:/data

volumes:
  redis-data-1:
  redis-data-2:
  redis-data-3:
```

#### Spring Boot Redis Cluster 설정
```java
@Configuration
public class RedisConfig {
    
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration config = new RedisClusterConfiguration();
        config.addClusterNode(new RedisNode("redis-node-1", 7001));
        config.addClusterNode(new RedisNode("redis-node-2", 7002));
        config.addClusterNode(new RedisNode("redis-node-3", 7003));
        
        JedisConnectionFactory factory = new JedisConnectionFactory(config);
        return factory;
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

### 4.4 캐시 운영 전략

#### 캐시 워밍(Cache Warming)
시스템 시작 시 자주 사용되는 데이터를 미리 캐시에 로드합니다.

```java
@Component
public class CacheWarmer {
    
    @EventListener(ApplicationReadyEvent.class)
    public void warmUpCache() {
        // 인기 상품 100개를 미리 캐시에 로드
        List<Product> popularProducts = productRepository.findTop100ByOrderByViewCountDesc();
        
        for (Product product : popularProducts) {
            String cacheKey = "product:" + product.getId();
            redisTemplate.opsForValue().set(
                cacheKey, product, 
                Duration.ofHours(1)
            );
        }
    }
}
```

#### 캐시 무효화 전략
```java
@Service
public class CacheInvalidationService {
    
    // 태그 기반 무효화
    public void invalidateProductCache(String productId) {
        Set<String> keys = redisTemplate.keys("product:" + productId + "*");
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
    
    // 패턴 기반 무효화
    public void invalidateCategoryCache(String categoryId) {
        Set<String> keys = redisTemplate.keys("category:" + categoryId + "*");
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
    
    // TTL 기반 자동 만료
    public void setCacheWithTTL(String key, Object value, long seconds) {
        redisTemplate.opsForValue().set(key, value, Duration.ofSeconds(seconds));
    }
}
```

---

## 5. 데이터베이스 확장 기법

### 5.1 수직적 확장

#### 읽기 복제본 (Read Replica)
읽기 전용 복제본을 생성하여 읽기 부하를 분산합니다.

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.write")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.read")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("write", writeDataSource());
        dataSourceMap.put("read", readDataSource());
        
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(writeDataSource());
        
        return routingDataSource;
    }
}

// 사용 예시
@Service
public class UserService {
    
    @WriteDataSource
    public void createUser(User user) {
        userRepository.save(user);
    }
    
    @ReadDataSource
    public User findUser(Long userId) {
        return userRepository.findById(userId);
    }
}
```

### 5.2 수평적 확장 (샤딩)

#### 범위 기반 샤딩
데이터를 특정 범위로 나누어 분산 저장합니다.

```java
@Service
public class RangeBasedSharding {
    
    private Map<String, DataSource> shards = new HashMap<>();
    
    public DataSource getShard(Long userId) {
        if (userId <= 100000) {
            return shards.get("shard1");
        } else if (userId <= 200000) {
            return shards.get("shard2");
        } else {
            return shards.get("shard3");
        }
    }
}
```

#### 해시 기반 샤딩
해시 함수를 사용하여 데이터를 분산합니다.

```java
@Service
public class HashBasedSharding {
    
    private List<DataSource> shards;
    
    public DataSource getShard(String key) {
        int hash = key.hashCode();
        int shardIndex = Math.abs(hash) % shards.size();
        return shards.get(shardIndex);
    }
}
```

#### 디렉토리 기반 샤딩
별도의 디렉토리 서비스를 통해 데이터 위치를 관리합니다.

```java
@Service
public class DirectoryBasedSharding {
    
    @Autowired
    private ShardDirectory shardDirectory;
    
    public DataSource getShard(String key) {
        String shardId = shardDirectory.getShardId(key);
        return getDataSourceByShardId(shardId);
    }
}
```

### 5.3 데이터베이스 분할 (Partitioning)

#### 수평 분할 (Horizontal Partitioning)
```sql
-- 날짜 기반 파티셔닝
CREATE TABLE orders_2024_01 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

#### 수직 분할 (Vertical Partitioning)
```sql
-- 사용자 기본 정보 테이블
CREATE TABLE users_basic (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP
);

-- 사용자 프로필 정보 테이블
CREATE TABLE users_profile (
    user_id BIGINT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    bio TEXT,
    avatar_url VARCHAR(500)
);
```

---

## 6. 오토스케일링과 리소스 관리

### 6.1 수평적 자동 확장 (HPA)

#### CPU 기반 HPA
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
  maxReplicas: 20
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

#### 커스텀 메트릭 기반 HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-consumer-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-consumer
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: rabbitmq_queue_messages
        selector:
          matchLabels:
            queue: "processing-queue"
      target:
        type: AverageValue
        averageValue: "10"
```

### 6.2 수직적 자동 확장 (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      maxAllowed:
        cpu: 2
        memory: 4Gi
      minAllowed:
        cpu: 100m
        memory: 128Mi
```

### 6.3 예측적 스케일링

```python
# AWS Lambda를 사용한 예측적 스케일링
import boto3
import pandas as pd
from datetime import datetime, timedelta

def lambda_handler(event, context):
    # 과거 트래픽 데이터 분석
    cloudwatch = boto3.client('cloudwatch')
    
    # 지난 주 같은 시간대 트래픽 패턴 조회
    end_time = datetime.now()
    start_time = end_time - timedelta(days=7)
    
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/ApplicationELB',
        MetricName='RequestCount',
        Dimensions=[
            {
                'Name': 'LoadBalancer',
                'Value': 'app/my-load-balancer/1234567890123456'
            }
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Sum']
    )
    
    # 예측 모델 적용 (간단한 예시)
    current_hour = datetime.now().hour
    if current_hour in [9, 12, 18]:  # 피크 시간대
        desired_capacity = 10
    else:
        desired_capacity = 3
    
    # Auto Scaling Group 업데이트
    autoscaling = boto3.client('autoscaling')
    autoscaling.set_desired_capacity(
        AutoScalingGroupName='my-asg',
        DesiredCapacity=desired_capacity,
        HonorCooldown=False
    )
    
    return {
        'statusCode': 200,
        'body': f'Scaled to {desired_capacity} instances'
    }
```

### 6.4 리소스 제한과 요청

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: web-app
        image: web-app:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## 7. 메시지 큐와 비동기 처리

### 7.1 메시지 큐 패턴

#### 점대점 (Point-to-Point)
하나의 메시지를 하나의 컨슈머가 처리하는 방식입니다.

```java
// RabbitMQ 구현 예시
@Component
public class OrderProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendOrder(Order order) {
        rabbitTemplate.convertAndSend("order.queue", order);
    }
}

@RabbitListener(queues = "order.queue")
@Component
public class OrderConsumer {
    
    public void processOrder(Order order) {
        // 주문 처리 로직
        System.out.println("Processing order: " + order.getId());
    }
}
```

#### 발행-구독 (Publish-Subscribe)
하나의 메시지를 여러 컨슈머가 받아 처리하는 방식입니다.

```java
// Apache Kafka 구현 예시
@Service
public class EventPublisher {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishOrderEvent(OrderEvent event) {
        kafkaTemplate.send("order-events", event);
    }
}

@KafkaListener(topics = "order-events", groupId = "inventory-service")
@Component
public class InventoryEventConsumer {
    
    public void handleOrderEvent(OrderEvent event) {
        // 재고 업데이트 로직
        inventoryService.updateStock(event.getProductId(), event.getQuantity());
    }
}

@KafkaListener(topics = "order-events", groupId = "notification-service")
@Component
public class NotificationEventConsumer {
    
    public void handleOrderEvent(OrderEvent event) {
        // 알림 발송 로직
        notificationService.sendOrderConfirmation(event.getUserId());
    }
}
```

### 7.2 메시지 큐 설정과 운영

#### RabbitMQ 클러스터 설정
```yaml
version: '3.8'
services:
  rabbitmq-1:
    image: rabbitmq:3.11-management
    hostname: rabbitmq-1
    environment:
      RABBITMQ_ERLANG_COOKIE: "secret-cookie"
      RABBITMQ_DEFAULT_USER: "admin"
      RABBITMQ_DEFAULT_PASS: "password"
    volumes:
      - rabbitmq-1-data:/var/lib/rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"

  rabbitmq-2:
    image: rabbitmq:3.11-management
    hostname: rabbitmq-2
    environment:
      RABBITMQ_ERLANG_COOKIE: "secret-cookie"
    volumes:
      - rabbitmq-2-data:/var/lib/rabbitmq
    depends_on:
      - rabbitmq-1
    command: >
      bash -c "
        rabbitmq-server &
        sleep 10
        rabbitmqctl stop_app
        rabbitmqctl join_cluster rabbit@rabbitmq-1
        rabbitmqctl start_app
        wait"

volumes:
  rabbitmq-1-data:
  rabbitmq-2-data:
```

#### Apache Kafka 클러스터 설정
```yaml
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
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3

  kafka-2:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:29093,PLAINTEXT_HOST://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3

  kafka-3:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:29094,PLAINTEXT_HOST://localhost:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
```

### 7.3 비동기 처리 패턴

#### 이벤트 소싱 (Event Sourcing)
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
        // 이벤트를 이벤트 스토어에 저장
        eventStore.save(event);
        // 이벤트를 메시지 큐에 발행
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

#### CQRS (Command Query Responsibility Segregation)
```java
// Command 모델
@Service
public class OrderCommandService {
    
    public void createOrder(CreateOrderCommand command) {
        OrderAggregate order = new OrderAggregate();
        order.createOrder(command.getCustomerId(), command.getItems());
        orderRepository.save(order);
    }
    
    public void payOrder(PayOrderCommand command) {
        OrderAggregate order = orderRepository.findById(command.getOrderId());
        order.payOrder(command.getPaymentId());
        orderRepository.save(order);
    }
}

// Query 모델
@Service
public class OrderQueryService {
    
    @Autowired
    private ReadOnlyOrderRepository readOnlyOrderRepository;
    
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
    
    @KafkaListener(topics = "order-events")
    public void handle(OrderPaidEvent event) {
        OrderView view = orderViewRepository.findById(event.getOrderId());
        view.setStatus("PAID");
        orderViewRepository.save(view);
    }
}
```

### 7.4 백프레셔와 플로우 제어

```java
@Service
public class BackpressureAwareConsumer {
    
    private final Semaphore semaphore = new Semaphore(10); // 동시 처리 제한
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("consumer");
    
    @KafkaListener(topics = "heavy-processing-topic", 
                   containerFactory = "backpressureKafkaListenerContainerFactory")
    public void processMessage(String message) {
        try {
            semaphore.acquire(); // 세마포어로 동시 처리 수 제한
            
            Supplier<String> decoratedSupplier = CircuitBreaker
                .decorateSupplier(circuitBreaker, () -> {
                    return heavyProcessingService.process(message);
                });
            
            String result = decoratedSupplier.get();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            semaphore.release();
        }
    }
}

// Kafka 컨슈머 설정
@Configuration
public class KafkaConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> 
            backpressureKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3); // 동시 컨슈머 수 제한
        
        // 백프레셔 설정
        factory.getContainerProperties().setPollTimeout(3000);
        factory.getContainerProperties().setIdleEventInterval(30000L);
        
        return factory;
    }
}
```

---

## 8. 마이크로서비스 아키텍처

### 8.1 마이크로서비스 설계 원칙

#### 단일 책임 원칙
각 서비스는 하나의 비즈니스 기능만 담당해야 합니다.

```
잘못된 예:
- UserOrderPaymentService (사용자, 주문, 결제가 모두 섞임)

올바른 예:
- UserService (사용자 관리)
- OrderService (주문 관리)  
- PaymentService (결제 처리)
```

#### 데이터베이스 분리
각 서비스는 독립적인 데이터베이스를 가져야 합니다.

```yaml
# docker-compose.yml
version: '3.8'
services:
  user-service:
    image: user-service:latest
    depends_on:
      - user-db
    environment:
      DATABASE_URL: postgres://user-db:5432/userdb

  user-db:
    image: postgres:13
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: userservice
      POSTGRES_PASSWORD: password

  order-service:
    image: order-service:latest
    depends_on:
      - order-db
    environment:
      DATABASE_URL: postgres://order-db:5432/orderdb

  order-db:
    image: postgres:13
    environment:
      POSTGRES_DB: orderdb
      POSTGRES_USER: orderservice
      POSTGRES_PASSWORD: password
```

### 8.2 서비스 간 통신

#### 동기식 통신 (REST API)
```java
// Order Service
@RestController
public class OrderController {
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        // 사용자 정보 검증을 위해 User Service 호출
        User user = userServiceClient.getUser(request.getUserId());
        
        if (user == null) {
            return ResponseEntity.badRequest().build();
        }
        
        Order order = orderService.createOrder(request);
        return ResponseEntity.ok(order);
    }
}

// Feign Client를 사용한 서비스 호출
@FeignClient(name = "user-service", url = "${user-service.url}")
public interface UserServiceClient {
    
    @GetMapping("/users/{userId}")
    User getUser(@PathVariable("userId") String userId);
}
```

#### 비동기식 통신 (이벤트 기반)
```java
// Order Service - 이벤트 발행
@Service
public class OrderService {
    
    @Autowired
    private EventPublisher eventPublisher;
    
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        
        // 이벤트 발행
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), 
            order.getUserId(), 
            order.getItems()
        );
        eventPublisher.publish("order.created", event);
        
        return order;
    }
}

// Inventory Service - 이벤트 구독
@Component
public class InventoryEventHandler {
    
    @EventListener
    @KafkaListener(topics = "order.created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        for (OrderItem item : event.getItems()) {
            inventoryService.reserveStock(item.getProductId(), item.getQuantity());
        }
        
        // 재고 예약 완료 이벤트 발행
        StockReservedEvent stockEvent = new StockReservedEvent(event.getOrderId());
        eventPublisher.publish("stock.reserved", stockEvent);
    }
}
```

### 8.3 API Gateway

```java
// Spring Cloud Gateway 설정
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r.path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestHeader("X-Gateway", "true")
                    .circuitBreaker(config -> config.setName("user-service-cb"))
                    .retry(retryConfig -> retryConfig.setRetries(3))
                )
                .uri("lb://user-service"))
            
            .route("order-service", r -> r.path("/api/orders/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestHeader("X-Gateway", "true")
                    .circuitBreaker(config -> config.setName("order-service-cb"))
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())
                    )
                )
                .uri("lb://order-service"))
            
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20); // 초당 10개, 버스트 20개
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> exchange.getRequest().getHeaders()
            .getFirst("X-User-Id");
    }
}
```

### 8.4 서비스 디스커버리

#### Eureka 서버 설정
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml (Eureka Server)
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

#### 서비스 등록
```java
@SpringBootApplication
@EnableEurekaClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml (User Service)
spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

### 8.5 분산 트랜잭션 (Saga Pattern)

#### Choreography-based Saga
```java
// Order Service
@Service
public class OrderSagaService {
    
    public void processOrder(CreateOrderRequest request) {
        try {
            // 1. 주문 생성
            Order order = createOrder(request);
            
            // 2. 재고 예약 이벤트 발행
            eventPublisher.publish("inventory.reserve", 
                new ReserveInventoryEvent(order.getId(), order.getItems()));
            
        } catch (Exception e) {
            // 보상 트랜잭션 실행
            compensateCreateOrder(request.getOrderId());
        }
    }
    
    @EventListener
    public void handleInventoryReserved(InventoryReservedEvent event) {
        // 3. 결제 요청 이벤트 발행
        eventPublisher.publish("payment.process", 
            new ProcessPaymentEvent(event.getOrderId(), event.getAmount()));
    }
    
    @EventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        // 보상 트랜잭션: 재고 예약 취소
        eventPublisher.publish("inventory.cancel", 
            new CancelInventoryEvent(event.getOrderId()));
        
        // 보상 트랜잭션: 주문 취소
        compensateCreateOrder(event.getOrderId());
    }
}
```

#### Orchestration-based Saga
```java
@Component
public class OrderSagaOrchestrator {
    
    @SagaOrchestrationStart
    public void processOrder(CreateOrderCommand command) {
        // 사가 인스턴스 생성
        SagaInstance saga = sagaManager.begin("order-saga", command.getOrderId());
        
        // 1. 재고 예약 단계
        saga.choreography()
            .step("reserve-inventory")
            .invokeParticipant("inventory-service")
            .withCompensation("cancel-inventory")
            
            // 2. 결제 처리 단계
            .step("process-payment")
            .invokeParticipant("payment-service")
            .withCompensation("refund-payment")
            
            // 3. 주문 확정 단계
            .step("confirm-order")
            .invokeLocal(this::confirmOrder)
            .withCompensation(this::cancelOrder)
            
            .execute();
    }
    
    @SagaCompensation
    public void handleSagaFailure(SagaFailureEvent event) {
        // 실패한 단계부터 역순으로 보상 트랜잭션 실행
        sagaManager.compensate(event.getSagaId());
    }
}
```

---

## 9. 성능 모니터링과 관찰 가능성

### 9.1 메트릭 수집

#### Prometheus 설정
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['user-service:8080', 'order-service:8081']
    scrape_interval: 5s

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

#### Spring Boot 애플리케이션에서 커스텀 메트릭
```java
@RestController
public class OrderController {
    
    private final MeterRegistry meterRegistry;
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;
    
    public OrderController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Number of orders created")
            .tag("service", "order-service")
            .register(meterRegistry);
        
        this.orderProcessingTimer = Timer.builder("order.processing.time")
            .description("Order processing time")
            .register(meterRegistry);
    }
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        return Timer.Sample.start(meterRegistry)
            .stop(orderProcessingTimer, () -> {
                try {
                    Order order = orderService.createOrder(request);
                    orderCreatedCounter.increment(
                        Tags.of("status", "success", "user_tier", order.getUserTier())
                    );
                    return ResponseEntity.ok(order);
                } catch (Exception e) {
                    orderCreatedCounter.increment(
                        Tags.of("status", "error", "error_type", e.getClass().getSimpleName())
                    );
                    throw e;
                }
            });
    }
}
```

### 9.2 로그 집계와 분석

#### ELK Stack 설정
```yaml
# docker-compose.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.5.0
    ports:
      - "5044:5044"
    volumes:
      - ./logstash/config:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  elasticsearch-data:
```

#### Logstash 파이프라인 설정
```ruby
# logstash/config/logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][service] {
    mutate {
      add_field => { "service_name" => "%{[fields][service]}" }
    }
  }

  # Spring Boot 로그 파싱
  if [service_name] == "order-service" {
    grok {
      match => { 
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:service},%{DATA:trace_id},%{DATA:span_id}\] %{DATA:thread} %{DATA:class} : %{GREEDYDATA:log_message}" 
      }
    }
    
    date {
      match => [ "timestamp", "yyyy-MM-dd HH:mm:ss.SSS" ]
    }
    
    if [trace_id] and [trace_id] != "" {
      mutate {
        add_field => { "trace_id" => "%{trace_id}" }
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "application-logs-%{+YYYY.MM.dd}"
  }
  
  stdout {
    codec => rubydebug
  }
}
```

#### 구조화된 로깅
```java
@RestController
public class OrderController {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderController.class);
    private static final String ORDER_CREATED = "order.created";
    private static final String ORDER_FAILED = "order.creation.failed";
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        String traceId = TraceContext.current().traceId();
        
        try {
            logger.info("Creating order for user: {}", 
                kv("user_id", request.getUserId()),
                kv("trace_id", traceId),
                kv("event", ORDER_CREATED)
            );
            
            Order order = orderService.createOrder(request);
            
            logger.info("Order created successfully: {}", 
                kv("order_id", order.getId()),
                kv("user_id", order.getUserId()),
                kv("amount", order.getTotalAmount()),
                kv("trace_id", traceId),
                kv("event", ORDER_CREATED),
                kv("processing_time_ms", processingTime)
            );
            
            return ResponseEntity.ok(order);
            
        } catch (Exception e) {
            logger.error("Failed to create order: {}", 
                kv("user_id", request.getUserId()),
                kv("error", e.getMessage()),
                kv("trace_id", traceId),
                kv("event", ORDER_FAILED),
                e
            );
            
            throw e;
        }
    }
}
```

### 9.3 분산 추적 (Distributed Tracing)

#### Jaeger 설정
```yaml
# docker-compose.yml
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:1.41
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

#### Spring Boot 애플리케이션에서 OpenTelemetry 설정
```java
@Configuration
public class TracingConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        return OpenTelemetrySdk.builder()
            .setTracerProvider(
                SdkTracerProvider.builder()
                    .addSpanProcessor(BatchSpanProcessor.builder(
                            JaegerGrpcSpanExporter.builder()
                                .setEndpoint("http://jaeger:14250")
                                .build())
                        .build())
                    .setResource(Resource.getDefault()
                        .merge(Resource.builder()
                            .put(ResourceAttributes.SERVICE_NAME, "order-service")
                            .put(ResourceAttributes.SERVICE_VERSION, "1.0.0")
                            .build()))
                    .build())
            .build();
    }
}

// 컨트롤러에서 추적 사용
@RestController
public class OrderController {
    
    private final Tracer tracer;
    
    public OrderController(OpenTelemetry openTelemetry) {
        this.tracer = openTelemetry.getTracer("order-service");
    }
    
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

### 9.4 알림과 경고

#### Alertmanager 설정
```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@company.com'
  smtp_auth_username: 'alerts@company.com'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
  routes:
  - match:
      severity: critical
    receiver: 'critical-alerts'
  - match:
      severity: warning
    receiver: 'warning-alerts'

receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://slack-webhook-service:8080/webhook'
    
- name: 'critical-alerts'
  email_configs:
  - to: 'oncall@company.com'
    subject: 'CRITICAL: {{ .GroupLabels.alertname }}'
    body: |
      {{ range .Alerts }}
      Alert: {{ .Annotations.summary }}
      Description: {{ .Annotations.description }}
      {{ end }}
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/...'
    channel: '#critical-alerts'
    title: 'CRITICAL ALERT'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

- name: 'warning-alerts'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/...'
    channel: '#monitoring'
    title: 'Warning Alert'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

#### 알림 규칙 정의
```yaml
# alert_rules.yml
groups:
- name: application.rules
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
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"

  - alert: HighMemoryUsage
    expr: |
      (
        container_memory_usage_bytes / 
        container_spec_memory_limit_bytes
      ) > 0.9
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage"
      description: "Memory usage is {{ $value | humanizePercentage }} for {{ $labels.container_name }}"

  - alert: DatabaseConnectionPoolExhausted
    expr: |
      hikaricp_connections_active >= hikaricp_connections_max
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Database connection pool exhausted"
      description: "All database connections are in use for {{ $labels.service }}"

  - alert: KafkaConsumerLag
    expr: |
      kafka_consumer_lag_max > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kafka consumer lag is high"
      description: "Consumer lag is {{ $value }} for topic {{ $labels.topic }}"
```

---

## 10. 실전 구현 예제

### 10.1 고트래픽 이커머스 시스템

#### 시스템 아키텍처 개요
```
[Load Balancer] 
    |
[API Gateway] - [Rate Limiter] - [Auth Service]
    |
+-- [User Service] - [User DB]
+-- [Product Service] - [Product DB] - [Redis Cache]
+-- [Order Service] - [Order DB] - [Message Queue]
+-- [Payment Service] - [Payment DB]
+-- [Inventory Service] - [Inventory DB]
+-- [Notification Service] - [Email/SMS Gateway]
```

#### 제품 조회 서비스 (고성능 캐싱)
```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping("/{productId}")
    @Cacheable(value = "products", key = "#productId")
    public ResponseEntity<ProductResponse> getProduct(
            @PathVariable String productId,
            @RequestHeader(value = "Accept-Language", defaultValue = "ko") String language) {
        
        ProductResponse product = productService.getProduct(productId, language);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(300, TimeUnit.SECONDS).cachePublic())
            .eTag(product.getVersion())
            .body(product);
    }
    
    @GetMapping("/search")
    public ResponseEntity<PagedResult<ProductSummary>> searchProducts(
            @RequestParam String query,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "relevance") String sort) {
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .page(page)
            .size(size)
            .sort(sort)
            .build();
            
        PagedResult<ProductSummary> results = productService.searchProducts(request);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
            .body(results);
    }
}

@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    // 멀티레벨 캐싱 전략
    public ProductResponse getProduct(String productId, String language) {
        String cacheKey = String.format("product:%s:%s", productId, language);
        
        // L1 캐시: 애플리케이션 메모리 (Caffeine)
        ProductResponse product = applicationCache.get(cacheKey);
        if (product != null) {
            return product;
        }
        
        // L2 캐시: 분산 캐시 (Redis)
        product = (ProductResponse) redisTemplate.opsForValue().get(cacheKey);
        if (product != null) {
            applicationCache.put(cacheKey, product);
            return product;
        }
        
        // L3: 데이터베이스
        Product dbProduct = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product = ProductResponse.from(dbProduct, language);
        
        // 캐시에 저장 (비동기)
        CompletableFuture.runAsync(() -> {
            redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(10));
            applicationCache.put(cacheKey, product);
        });
        
        return product;
    }
    
    // 검색 기능 (Elasticsearch)
    public PagedResult<ProductSummary> searchProducts(SearchRequest request) {
        String cacheKey = "search:" + request.hashCode();
        
        PagedResult<ProductSummary> cachedResult = 
            (PagedResult<ProductSummary>) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedResult != null) {
            return cachedResult;
        }
        
        // Elasticsearch 쿼리
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
            .withQuery(QueryBuilders.multiMatchQuery(request.getQuery())
                .field("name", 2.0f)
                .field("description", 1.0f)
                .field("tags", 1.5f))
            .withFilter(QueryBuilders.termQuery("status", "ACTIVE"))
            .withSort(SortBuilders.fieldSort(request.getSort()))
            .withPageable(PageRequest.of(request.getPage(), request.getSize()))
            .build();
        
        SearchHits<Product> searchHits = elasticsearchTemplate.search(searchQuery, Product.class);
        
        List<ProductSummary> products = searchHits.stream()
            .map(hit -> ProductSummary.from(hit.getContent()))
            .collect(Collectors.toList());
        
        PagedResult<ProductSummary> result = new PagedResult<>(
            products, 
            request.getPage(), 
            request.getSize(), 
            searchHits.getTotalHits()
        );
        
        // 캐시 저장 (1분)
        redisTemplate.opsForValue().set(cacheKey, result, Duration.ofMinutes(1));
        
        return result;
    }
}
```

#### 주문 처리 서비스 (이벤트 기반)
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    @RateLimiter(name = "order-creation", fallbackMethod = "fallbackCreateOrder")
    @CircuitBreaker(name = "order-service", fallbackMethod = "fallbackCreateOrder")
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        
        OrderResponse order = orderService.createOrder(request, user.getId());
        
        return ResponseEntity.status(HttpStatus.CREATED)
            .location(URI.create("/api/orders/" + order.getId()))
            .body(order);
    }
    
    public ResponseEntity<OrderResponse> fallbackCreateOrder(
            CreateOrderRequest request, UserPrincipal user, Exception ex) {
        
        // 장애 시 대체 응답
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(OrderResponse.builder()
                .message("주문 서비스가 일시적으로 사용할 수 없습니다. 잠시 후 다시 시도해주세요.")
                .build());
    }
}

@Service
@Transactional
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @Autowired
    private InventoryServiceClient inventoryServiceClient;
    
    @Autowired
    private RedissonClient redissonClient;
    
    public OrderResponse createOrder(CreateOrderRequest request, String userId) {
        // 분산 락을 사용한 동시성 제어
        String lockKey = "order:create:" + userId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 5초 대기, 30초 후 자동 해제
            if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                return processOrderCreation(request, userId);
            } else {
                throw new OrderCreationException("주문 처리 중입니다. 잠시 후 다시 시도해주세요.");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new OrderCreationException("주문 처리가 중단되었습니다.");
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
    
    private OrderResponse processOrderCreation(CreateOrderRequest request, String userId) {
        // 1. 주문 생성
        Order order = Order.builder()
            .userId(userId)
            .items(request.getItems())
            .totalAmount(calculateTotalAmount(request.getItems()))
            .status(OrderStatus.PENDING)
            .build();
        
        order = orderRepository.save(order);
        
        // 2. 재고 확인 (동기 호출)
        InventoryCheckResult inventoryResult = inventoryServiceClient
            .checkInventory(order.getItems());
        
        if (!inventoryResult.isAvailable()) {
            order.setStatus(OrderStatus.CANCELLED);
            order.setCancelReason("재고 부족");
            orderRepository.save(order);
            
            throw new InsufficientInventoryException("재고가 부족합니다.");
        }
        
        // 3. 주문 처리 이벤트 발행 (비동기)
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .items(order.getItems())
            .totalAmount(order.getTotalAmount())
            .timestamp(Instant.now())
            .build();
        
        eventPublisher.publishEvent("order.created", event);
        
        return OrderResponse.from(order);
    }
    
    // 이벤트 처리
    @EventListener
    @KafkaListener(topics = "inventory.reserved")
    public void handleInventoryReserved(InventoryReservedEvent event) {
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
        
        order.setStatus(OrderStatus.INVENTORY_RESERVED);
        orderRepository.save(order);
        
        // 결제 처리 이벤트 발행
        PaymentRequestEvent paymentEvent = PaymentRequestEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .amount(order.getTotalAmount())
            .build();
        
        eventPublisher.publishEvent("payment.process", paymentEvent);
    }
    
    @EventListener
    @KafkaListener(topics = "payment.completed")
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
        
        order.setStatus(OrderStatus.PAID);
        order.setPaymentId(event.getPaymentId());
        orderRepository.save(order);
        
        // 주문 확정 이벤트 발행
        OrderConfirmedEvent confirmedEvent = OrderConfirmedEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .build();
        
        eventPublisher.publishEvent("order.confirmed", confirmedEvent);
    }
    
    @EventListener
    @KafkaListener(topics = "payment.failed")
    public void handlePaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
        
        order.setStatus(OrderStatus.PAYMENT_FAILED);
        orderRepository.save(order);
        
        // 보상 트랜잭션: 재고 예약 취소
        InventoryCancelEvent cancelEvent = InventoryCancelEvent.builder()
            .orderId(order.getId())
            .items(order.getItems())
            .build();
        
        eventPublisher.publishEvent("inventory.cancel", cancelEvent);
    }
}
```

#### 재고 관리 서비스 (동시성 제어)
```java
@Service
public class InventoryService {
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @Autowired
    private RedissonClient redissonClient;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @EventListener
    @KafkaListener(topics = "order.created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            reserveInventory(event.getOrderId(), event.getItems());
            
            InventoryReservedEvent reservedEvent = InventoryReservedEvent.builder()
                .orderId(event.getOrderId())
                .build();
            
            eventPublisher.publishEvent("inventory.reserved", reservedEvent);
            
        } catch (InsufficientInventoryException e) {
            InventoryReservationFailedEvent failedEvent = InventoryReservationFailedEvent.builder()
                .orderId(event.getOrderId())
                .reason(e.getMessage())
                .build();
            
            eventPublisher.publishEvent("inventory.reservation.failed", failedEvent);
        }
    }
    
    @Transactional
    public void reserveInventory(String orderId, List<OrderItem> items) {
        Map<String, RLock> locks = new HashMap<>();
        
        try {
            // 모든 상품에 대해 락 획득
            for (OrderItem item : items) {
                String lockKey = "inventory:product:" + item.getProductId();
                RLock lock = redissonClient.getLock(lockKey);
                
                if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                    locks.put(item.getProductId(), lock);
                } else {
                    throw new InventoryLockException("재고 잠금을 획득할 수 없습니다.");
                }
            }
            
            // 재고 확인 및 예약
            for (OrderItem item : items) {
                Inventory inventory = inventoryRepository
                    .findByProductId(item.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
                
                if (inventory.getAvailableQuantity() < item.getQuantity()) {
                    throw new InsufficientInventoryException(
                        String.format("상품 %s의 재고가 부족합니다. 요청: %d, 가용: %d", 
                            item.getProductId(), 
                            item.getQuantity(), 
                            inventory.getAvailableQuantity())
                    );
                }
                
                // 재고 예약
                inventory.reserve(item.getQuantity());
                inventoryRepository.save(inventory);
                
                // 예약 기록 저장
                InventoryReservation reservation = InventoryReservation.builder()
                    .orderId(orderId)
                    .productId(item.getProductId())
                    .quantity(item.getQuantity())
                    .reservedAt(Instant.now())
                    .expiresAt(Instant.now().plus(Duration.ofMinutes(15))) // 15분 후 자동 해제
                    .build();
                
                inventoryReservationRepository.save(reservation);
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new InventoryException("재고 처리가 중단되었습니다.");
            
        } finally {
            // 모든 락 해제
            locks.values().forEach(lock -> {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            });
        }
    }
    
    // 예약 만료된 재고 자동 해제
    @Scheduled(fixedDelay = 60000) // 1분마다 실행
    public void releaseExpiredReservations() {
        List<InventoryReservation> expiredReservations = 
            inventoryReservationRepository.findExpiredReservations(Instant.now());
        
        for (InventoryReservation reservation : expiredReservations) {
            releaseReservation(reservation);
        }
    }
    
    @Transactional
    public void releaseReservation(InventoryReservation reservation) {
        String lockKey = "inventory:product:" + reservation.getProductId();
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                Inventory inventory = inventoryRepository
                    .findByProductId(reservation.getProductId())
                    .orElse(null);
                
                if (inventory != null) {
                    inventory.release(reservation.getQuantity());
                    inventoryRepository.save(inventory);
                }
                
                inventoryReservationRepository.delete(reservation);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### 10.2 Kubernetes 배포 설정

#### 애플리케이션 배포 매니페스트
```yaml
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
      - name: order-service
        image: order-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: order-service-config
              key: redis-url
        - name: KAFKA_BROKERS
          valueFrom:
            configMapKeyRef:
              name: order-service-config
              key: kafka-brokers
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        - name: log-volume
          mountPath: /app/logs
      volumes:
      - name: config-volume
        configMap:
          name: order-service-config
      - name: log-volume
        emptyDir: {}
      imagePullSecrets:
      - name: docker-registry-secret

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
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
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag_max
      target:
        type: AverageValue
        averageValue: "50"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

#### ConfigMap과 Secret
```yaml
# order-service-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  application.yml: |
    server:
      port: 8080
      
    spring:
      application:
        name: order-service
      datasource:
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
          idle-timeout: 300000
          max-lifetime: 1800000
      
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
      endpoint:
        health:
          show-details: always
      health:
        circuitbreakers:
          enabled: true
        ratelimiters:
          enabled: true
          
    resilience4j:
      circuitbreaker:
        instances:
          inventory-service:
            slidingWindowSize: 10
            permittedNumberOfCallsInHalfOpenState: 3
            slidingWindowType: COUNT_BASED
            minimumNumberOfCalls: 5
            waitDurationInOpenState: 30s
            failureRateThreshold: 50
            eventConsumerBufferSize: 10
      
      ratelimiter:
        instances:
          order-creation:
            limitForPeriod: 10
            limitRefreshPeriod: 1s
            timeoutDuration: 5s
            
  redis-url: "redis://redis-cluster:6379"
  kafka-brokers: "kafka-1:9092,kafka-2:9092,kafka-3:9092"

---
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
type: Opaque
data:
  database-url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc3dvcmRAcG9zdGdyZXM6NTQzMi9vcmRlcmRi # base64 encoded
  jwt-secret: bXlfc2VjcmV0X2tleQ== # base64 encoded
  encryption-key: ZW5jcnlwdGlvbl9rZXk= # base64 encoded
```

#### Ingress 설정
```yaml
# api-gateway-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: "/$2"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1/users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /api/v1/products(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 80
      - path: /api/v1/orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /api/v1/payments(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
```

이 가이드는 고트래픽 확장 가능한 시스템 구축을 위한 포괄적인 내용을 다루고 있습니다. 각 섹션의 개념을 이해하고 실제 프로젝트에 적용해보면서 경험을 쌓는 것이 중요합니다.
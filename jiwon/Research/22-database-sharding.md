# 2.2 데이터베이스 샤딩

## Overview
데이터베이스 샤딩은 대용량 데이터를 여러 데이터베이스에 수평적으로 분할하여 저장하는 기법입니다. **샤딩(Sharding)**을 통해 단일 데이터베이스의 한계를 극복하고 확장성을 확보할 수 있습니다.

## 샤딩 전략

### Range-based Sharding
```
Shard 1: user_id 1 ~ 1,000,000
Shard 2: user_id 1,000,001 ~ 2,000,000
Shard 3: user_id 2,000,001 ~ 3,000,000
```

**특징**:
- 범위 기반으로 데이터 분할
- 구현이 간단함
- 핫스팟 발생 가능성

### Hash-based Sharding
```java
public int getShardId(String key) {
    return Math.abs(key.hashCode()) % shardCount;
}
```

**특징**:
- 해시 함수를 사용한 균등 분산
- 핫스팟 방지
- 범위 쿼리 비효율적

### Directory-based Sharding
```
Lookup Service
├── user_id 1-500K    → Shard 1
├── user_id 500K-1M   → Shard 2
└── user_id 1M-1.5M   → Shard 3
```

**특징**:
- 별도 디렉터리 서비스로 샤드 위치 관리
- 유연한 데이터 분배
- 추가 조회 오버헤드

## 일관된 해싱 구현

### 기본 구현
```java
public class ConsistentHashing {
    private final SortedMap<Long, Integer> ring = new TreeMap<>();
    private final int virtualNodes = 150;
    
    public ConsistentHashing(int shardCount) {
        for (int i = 0; i < shardCount; i++) {
            for (int j = 0; j < virtualNodes; j++) {
                String virtualNodeKey = "shard-" + i + "-virtual-" + j;
                long hash = hash(virtualNodeKey);
                ring.put(hash, i);
            }
        }
    }
    
    public int getShardIndex(String key) {
        long hash = hash(key);
        SortedMap<Long, Integer> tailMap = ring.tailMap(hash);
        Long nodeHash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        return ring.get(nodeHash);
    }
}
```

### 샤딩 라우터
```java
@Component
public class ShardingRouter {
    private final List<DataSource> shards;
    private final ConsistentHashing consistentHash;
    
    public DataSource getShard(String shardKey) {
        int shardIndex = consistentHash.getShardIndex(shardKey);
        return shards.get(shardIndex);
    }
    
    public List<DataSource> getAllShards() {
        return new ArrayList<>(shards);
    }
}
```

## MySQL 파티셔닝

### Range 파티셔닝
```sql
CREATE TABLE orders (
    order_id BIGINT,
    user_id BIGINT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (TO_DAYS(order_date)) (
    PARTITION p2024_q1 VALUES LESS THAN (TO_DAYS('2024-04-01')),
    PARTITION p2024_q2 VALUES LESS THAN (TO_DAYS('2024-07-01')),
    PARTITION p2024_q3 VALUES LESS THAN (TO_DAYS('2024-10-01')),
    PARTITION p2024_q4 VALUES LESS THAN (TO_DAYS('2025-01-01'))
);
```

### Hash 파티셔닝
```sql
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT,
    email VARCHAR(255),
    username VARCHAR(100),
    PRIMARY KEY (user_id)
) PARTITION BY HASH(user_id) PARTITIONS 8;
```

### List 파티셔닝
```sql
CREATE TABLE user_profiles (
    profile_id BIGINT,
    user_id BIGINT,
    region VARCHAR(10),
    PRIMARY KEY (profile_id, region)
) PARTITION BY LIST COLUMNS(region) (
    PARTITION p_asia VALUES IN('KR', 'JP', 'CN', 'SG'),
    PARTITION p_america VALUES IN('US', 'CA', 'BR', 'MX'),
    PARTITION p_europe VALUES IN('DE', 'FR', 'GB', 'IT')
);
```

## 애플리케이션 구현

### 샤딩 서비스
```java
@Service
public class ShardedUserService {
    private final ShardingJdbcTemplate shardingTemplate;
    
    public User findById(Long userId) {
        String shardKey = String.valueOf(userId);
        String sql = "SELECT * FROM users WHERE user_id = ?";
        return shardingTemplate.queryForObject(shardKey, sql, userRowMapper, userId);
    }
    
    public User save(User user) {
        String shardKey = String.valueOf(user.getUserId());
        
        if (user.getUserId() == null) {
            return createUser(user, shardKey);
        } else {
            return updateUser(user, shardKey);
        }
    }
}
```

### 크로스 샤드 쿼리
```java
@Service
public class CrossShardQueryService {
    private final ExecutorService executorService;
    
    public <T> List<T> queryAllShards(String sql, RowMapper<T> rowMapper, Object... args) {
        List<CompletableFuture<List<T>>> futures = new ArrayList<>();
        
        for (DataSource shard : getAllShards()) {
            CompletableFuture<List<T>> future = CompletableFuture.supplyAsync(() -> {
                JdbcTemplate jdbcTemplate = new JdbcTemplate(shard);
                return jdbcTemplate.query(sql, rowMapper, args);
            }, executorService);
            
            futures.add(future);
        }
        
        return futures.stream()
            .map(CompletableFuture::join)
            .flatMap(List::stream)
            .collect(Collectors.toList());
    }
}
```

## 샤드 재분배

### 온라인 마이그레이션
```java
@Service
public class ShardMigrationService {
    
    public void addNewShard(DataSource newShard) {
        // 1. 새 샤드를 라우터에 추가
        shardingRouter.addShard(newShard);
        
        // 2. 데이터 재분배 계획 수립
        MigrationPlan plan = createMigrationPlan();
        
        // 3. 점진적 데이터 마이그레이션
        executeMigrationPlan(plan);
        
        // 4. 라우팅 테이블 업데이트
        updateRoutingTable();
    }
    
    private void executeMigrationPlan(MigrationPlan plan) {
        for (MigrationTask task : plan.getTasks()) {
            // 소스에서 데이터 읽기
            List<DataRecord> records = readFromSource(task);
            
            // 타겟에 데이터 쓰기
            writeToTarget(task, records);
            
            // 일관성 검증
            if (verifyConsistency(task)) {
                deleteFromSource(task);
            }
        }
    }
}
```

### 듀얼 쓰기 패턴
```java
@Service
public class DualWriteService {
    
    public void enableDualWrite(String tableName) {
        // 새로운 쓰기는 기존 샤드와 새 샤드 모두에 적용
        dualWriteManager.enable(tableName);
    }
    
    @Transactional
    public void writeWithDualWrite(String tableName, Object data) {
        try {
            // 기존 샤드에 쓰기
            writeToOldShard(tableName, data);
            
            // 새 샤드에도 쓰기
            writeToNewShard(tableName, data);
            
        } catch (Exception e) {
            // 롤백 처리
            rollbackDualWrite(tableName, data);
            throw e;
        }
    }
}
```

## 성능 모니터링

### 샤드별 메트릭
```java
@Component
public class ShardMetricsCollector {
    
    @Scheduled(fixedRate = 30000)
    public void collectMetrics() {
        for (int i = 0; i < shards.size(); i++) {
            DataSource shard = shards.get(i);
            String shardId = "shard-" + i;
            
            // 테이블 크기
            long tableSize = getTableSize(shard);
            meterRegistry.gauge("shard.table.size", Tags.of("shard", shardId), tableSize);
            
            // 쿼리 응답시간
            long avgQueryTime = getAverageQueryTime(shard);
            meterRegistry.gauge("shard.query.time", Tags.of("shard", shardId), avgQueryTime);
            
            // 활성 연결 수
            int activeConnections = getActiveConnections(shard);
            meterRegistry.gauge("shard.connections", Tags.of("shard", shardId), activeConnections);
        }
    }
}
```

### 불균형 감지
```java
@Component
public class ShardBalanceMonitor {
    
    @Scheduled(fixedRate = 300000) // 5분마다
    public void checkBalance() {
        Map<String, Long> shardSizes = getShardSizes();
        
        long minSize = Collections.min(shardSizes.values());
        long maxSize = Collections.max(shardSizes.values());
        
        double imbalanceRatio = (double) maxSize / minSize;
        
        if (imbalanceRatio > 3.0) {
            alertService.sendAlert("Shard imbalance detected: " + imbalanceRatio);
            suggestRebalancing(shardSizes);
        }
    }
}
```

## Best Practices

### 샤드 키 선택
- **높은 카디널리티**: 균등 분산을 위한 다양한 값
- **불변성**: 변경되지 않는 키 사용
- **쿼리 패턴**: 자주 사용되는 쿼리 조건

### 크로스 샤드 쿼리 최적화
- **병렬 처리**: 여러 샤드 동시 쿼리
- **결과 집계**: 애플리케이션 레벨에서 집계
- **캐싱**: 자주 사용되는 집계 결과 캐싱

### 데이터 일관성
- **분산 트랜잭션**: 2PC 또는 Saga 패턴 사용
- **이벤트 소싱**: 변경사항을 이벤트로 관리
- **최종 일관성**: Eventually Consistent 모델 적용

## Benefits and Challenges

### Benefits
- **확장성**: 데이터 크기와 처리량 선형 확장
- **성능**: 샤드별 독립적인 처리로 성능 향상
- **가용성**: 일부 샤드 장애가 전체 시스템에 미치는 영향 최소화
- **비용 효율성**: 필요에 따른 점진적 확장

### Challenges
- **복잡성**: 애플리케이션 로직과 운영 복잡성 증가
- **크로스 샤드 쿼리**: 여러 샤드에 걸친 조인과 집계 어려움
- **재분배**: 샤드 추가/제거 시 데이터 마이그레이션 복잡
- **트랜잭션**: 분산 트랜잭션 처리의 어려움
# 2.1 데이터베이스 읽기 확장

## Overview
데이터베이스 읽기 확장은 **읽기 복제본(Read Replica)**을 활용하여 읽기 트래픽을 분산시키는 기법입니다. 마스터-슬레이브 아키텍처를 통해 읽기 성능을 향상시키고 시스템 가용성을 높일 수 있습니다.

## 읽기 복제 아키텍처

### 기본 구조
```
Application
    ├── Write Requests → Master Database
    └── Read Requests  → Slave Database 1, 2, 3...
```

**마스터 데이터베이스**는 모든 쓰기 작업을 처리하고, **슬레이브 데이터베이스**들은 마스터로부터 데이터를 복제받아 읽기 작업만 처리합니다.

### 복제 메커니즘
1. **바이너리 로그**: 마스터에서 모든 변경사항을 바이너리 로그에 기록
2. **릴레이 로그**: 슬레이브가 마스터의 바이너리 로그를 가져와서 저장
3. **SQL 쓰레드**: 릴레이 로그의 내용을 슬레이브 데이터베이스에 적용

## MySQL Master-Slave 설정

### 마스터 서버 설정
```sql
-- my.cnf 설정
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW

-- 복제 사용자 생성
CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;
```

### 슬레이브 서버 설정
```sql
-- my.cnf 설정
[mysqld]
server-id = 2
read_only = 1
relay-log = relay-log

-- 복제 시작
CHANGE MASTER TO
    MASTER_HOST = 'master-host',
    MASTER_USER = 'replication',
    MASTER_PASSWORD = 'password';
START SLAVE;
```

## 애플리케이션 구현

### Spring Boot 라우팅 데이터소스
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource());
        targetDataSources.put("slave", slaveDataSource());
        
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        
        return routingDataSource;
    }
}
```

### 어노테이션 기반 라우팅
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadDataSource {
}

@Service
public class ProductService {
    
    @ReadDataSource
    @Transactional(readOnly = true)
    public List<Product> findAllProducts() {
        return productRepository.findAll();
    }
    
    @WriteDataSource
    public Product createProduct(Product product) {
        return productRepository.save(product);
    }
}
```

## 복제 지연 처리

### 지연 모니터링
```java
@Component
public class ReplicationLagMonitor {
    
    @Scheduled(fixedRate = 30000)
    public void checkReplicationLag() {
        try (Connection conn = slaveDataSource.getConnection()) {
            String sql = "SHOW SLAVE STATUS";
            // Seconds_Behind_Master 값 확인
            // 임계값 초과 시 알림 발송
        }
    }
}
```

### 적응적 라우팅
```java
public class AdaptiveReadService {
    
    public <T> T readWithConsistency(Supplier<T> readOperation) {
        Duration currentLag = lagMonitor.getCurrentLag();
        
        if (currentLag.compareTo(MAX_ACCEPTABLE_LAG) > 0) {
            // 지연이 크면 마스터에서 읽기
            useDataSource("master");
        } else {
            // 지연이 허용 범위면 슬레이브에서 읽기
            useDataSource("slave");
        }
        
        return readOperation.get();
    }
}
```

## 장애 처리

### 자동 페일오버
```java
@Component
public class DatabaseFailoverManager {
    
    @EventListener
    public void handleMasterFailure(MasterFailureEvent event) {
        // 1. 가장 적합한 슬레이브 선택
        DataSource newMaster = selectBestSlave();
        
        // 2. 슬레이브를 마스터로 승격
        promoteSlaveToMaster(newMaster);
        
        // 3. 라우팅 업데이트
        updateRouting(newMaster);
    }
}
```

### 헬스 체크
```java
@Component
public class DatabaseHealthChecker {
    
    @Scheduled(fixedRate = 10000)
    public void checkHealth() {
        // 마스터와 슬레이브 상태 확인
        // 장애 발생 시 이벤트 발행
    }
}
```

## Best Practices

### 연결 풀 최적화
- **마스터**: 적당한 크기의 연결 풀 (쓰기 위주)
- **슬레이브**: 큰 크기의 연결 풀 (읽기 위주)

### 일관성 보장
- **세션 일관성**: 사용자별 최근 쓰기 추적
- **단조 읽기**: 시간이 지나면서 더 오래된 데이터를 읽지 않도록 보장

### 부하 분산
- **라운드 로빈**: 슬레이브들 간 균등 분배
- **가중치 기반**: 서버 성능에 따른 차등 분배

## Benefits and Challenges

### Benefits
- **읽기 성능 향상**: 읽기 트래픽 분산으로 응답 시간 단축
- **가용성 증대**: 마스터 장애 시 슬레이브로 페일오버 가능
- **확장성**: 슬레이브 추가로 읽기 처리량 확장
- **백업**: 프로덕션 영향 없이 백업 및 분석 수행

### Challenges
- **복제 지연**: 마스터와 슬레이브 간 데이터 불일치 가능성
- **복잡성 증가**: 애플리케이션 로직 복잡화
- **일관성 문제**: 읽기 후 즉시 쓰기 시 데이터 불일치
- **운영 오버헤드**: 여러 데이터베이스 관리 부담
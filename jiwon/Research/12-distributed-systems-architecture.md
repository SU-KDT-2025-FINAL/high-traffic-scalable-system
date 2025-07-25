# 1-2.분산 시스템 아키텍처: CAP 정리와 분산 합의 알고리즘

## 개요

분산 시스템은 현대 고트래픽 애플리케이션의 기반입니다. 이 문서는 CAP 정리의 실제적 이해, 일관성 모델의 트레이드오프, 그리고 분산 합의 알고리즘의 구현을 다룹니다. 시니어 개발자로서 이론적 개념을 실제 시스템 설계 결정으로 전환하는 방법을 학습합니다.

## CAP 정리 깊이 있는 이해

### CAP 정리의 실제적 의미

**일관성 (Consistency)**
- **정의**: 모든 노드가 동시에 같은 데이터를 볼 수 있음
- **강한 일관성**: 모든 읽기가 가장 최근 쓰기를 반환
- **실제 구현**: 동기식 복제, 분산 락, 합의 프로토콜
- **비용**: 높은 지연시간, 낮은 가용성

**가용성 (Availability)**
- **정의**: 시스템이 장애 중에도 계속 운영됨
- **실제 의미**: 모든 요청이 응답을 받음 (성공 또는 실패)
- **구현**: 복제, 자동 장애극복, 느슨한 일관성
- **비용**: 데이터 불일치 가능성

**분할 내성 (Partition Tolerance)**
- **정의**: 네트워크 분할 중에도 시스템이 계속 동작
- **현실적 필요성**: 네트워크 장애는 불가피
- **구현**: 복제, 분산 데이터 저장, 비동기 통신

### CAP 정리의 현실적 적용

```java
// CP 시스템 예시 - 강한 일관성, 낮은 가용성
@Service
public class StrongConsistencyService {
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transferMoney(String fromAccount, String toAccount, BigDecimal amount) {
        // 분산 락 획득
        DistributedLock lock1 = lockService.acquire(fromAccount);
        DistributedLock lock2 = lockService.acquire(toAccount);
        
        try {
            Account from = accountRepository.findByIdForUpdate(fromAccount);
            Account to = accountRepository.findByIdForUpdate(toAccount);
            
            if (from.getBalance().compareTo(amount) < 0) {
                throw new InsufficientFundsException();
            }
            
            from.withdraw(amount);
            to.deposit(amount);
            
            // 모든 노드에 동기식 복제
            accountRepository.saveWithSyncReplication(from);
            accountRepository.saveWithSyncReplication(to);
            
        } finally {
            lock1.release();
            lock2.release();
        }
    }
}
```

```java
// AP 시스템 예시 - 높은 가용성, 최종 일관성
@Service
public class EventualConsistencyService {
    
    public void createPost(CreatePostRequest request) {
        Post post = new Post(request);
        
        // 로컬에 즉시 저장 (가용성 우선)
        postRepository.save(post);
        
        // 비동기로 다른 노드에 전파
        eventPublisher.publishAsync(new PostCreatedEvent(post));
        
        // 검색 인덱스 업데이트는 최종 일관성
        CompletableFuture.runAsync(() -> {
            try {
                searchIndexService.index(post);
            } catch (Exception e) {
                log.warn("검색 인덱스 업데이트 실패, 재시도 예정", e);
                retryService.scheduleRetry(() -> searchIndexService.index(post));
            }
        });
    }
}
```

## 분산 시스템 패턴

### Master-Slave 패턴 상세 구현

**MySQL Master-Slave 복제 설정**
```yaml
# docker-compose.yml
version: '3.8'
services:
  mysql-master:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: ecommerce
      MYSQL_USER: replication
      MYSQL_PASSWORD: replication_password
    command: >
      --server-id=1
      --log-bin=mysql-bin
      --binlog-format=ROW
      --gtid-mode=ON
      --enforce-gtid-consistency=ON
    ports:
      - "3306:3306"
    volumes:
      - master_data:/var/lib/mysql

  mysql-slave:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
    command: >
      --server-id=2
      --relay-log=relay-log
      --read-only=1
      --gtid-mode=ON
      --enforce-gtid-consistency=ON
    ports:
      - "3307:3306"
    volumes:
      - slave_data:/var/lib/mysql
    depends_on:
      - mysql-master
```

**동적 데이터소스 라우팅**
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(DatabaseType.WRITE, masterDataSource());
        dataSourceMap.put(DatabaseType.READ, slaveDataSource());
        
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        
        return routingDataSource;
    }
    
    @Bean
    public DataSource masterDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://mysql-master:3306/ecommerce");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
    
    @Bean
    public DataSource slaveDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://mysql-slave:3306/ecommerce");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setReadOnly(true);
        return new HikariDataSource(config);
    }
}

// 사용 예시
@Service
public class ProductService {
    
    @ReadDataSource
    @Transactional(readOnly = true)
    public List<Product> findAllProducts() {
        return productRepository.findAll(); // 슬레이브에서 읽기
    }
    
    @WriteDataSource
    @Transactional
    public Product createProduct(Product product) {
        return productRepository.save(product); // 마스터에 쓰기
    }
}
```

### 일관된 해싱 (Consistent Hashing) 구현

```java
public class ConsistentHashRing {
    private final TreeMap<Long, String> ring = new TreeMap<>();
    private final int virtualNodes;
    private final MessageDigest md5;
    
    public ConsistentHashRing(int virtualNodes) {
        this.virtualNodes = virtualNodes;
        try {
            this.md5 = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("MD5 알고리즘을 찾을 수 없습니다", e);
        }
    }
    
    public void addNode(String node) {
        for (int i = 0; i < virtualNodes; i++) {
            String virtualNodeName = node + ":" + i;
            long hash = hash(virtualNodeName);
            ring.put(hash, node);
        }
        log.info("노드 {} 추가됨. 가상 노드 수: {}", node, virtualNodes);
    }
    
    public void removeNode(String node) {
        for (int i = 0; i < virtualNodes; i++) {
            String virtualNodeName = node + ":" + i;
            long hash = hash(virtualNodeName);
            ring.remove(hash);
        }
        log.info("노드 {} 제거됨", node);
    }
    
    public String getNode(String key) {
        if (ring.isEmpty()) {
            return null;
        }
        
        long hash = hash(key);
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
        
        // 링의 끝에 도달하면 첫 번째 노드로 되돌아감
        return entry != null ? entry.getValue() : ring.firstEntry().getValue();
    }
    
    private long hash(String key) {
        md5.reset();
        md5.update(key.getBytes());
        byte[] digest = md5.digest();
        
        long hash = 0;
        for (int i = 0; i < 4; i++) {
            hash <<= 8;
            hash |= ((int) digest[i]) & 0xFF;
        }
        return hash;
    }
    
    // 데이터 분포 분석을 위한 메서드
    public Map<String, Integer> getDistribution(List<String> keys) {
        Map<String, Integer> distribution = new HashMap<>();
        
        for (String key : keys) {
            String node = getNode(key);
            distribution.merge(node, 1, Integer::sum);
        }
        
        return distribution;
    }
}
```

### 샤딩 구현 예시

```java
@Service
public class ShardedUserService {
    
    private final List<DataSource> shards;
    private final ConsistentHashRing hashRing;
    
    public ShardedUserService(List<DataSource> shards) {
        this.shards = shards;
        this.hashRing = new ConsistentHashRing(150); // 노드당 150개 가상 노드
        
        // 샤드를 해시 링에 추가
        for (int i = 0; i < shards.size(); i++) {
            hashRing.addNode("shard-" + i);
        }
    }
    
    public User getUserById(String userId) {
        DataSource shard = getShard(userId);
        JdbcTemplate jdbcTemplate = new JdbcTemplate(shard);
        
        try {
            return jdbcTemplate.queryForObject(
                "SELECT * FROM users WHERE user_id = ?",
                new Object[]{userId},
                new UserRowMapper()
            );
        } catch (EmptyResultDataAccessException e) {
            return null;
        }
    }
    
    public void saveUser(User user) {
        DataSource shard = getShard(user.getUserId());
        JdbcTemplate jdbcTemplate = new JdbcTemplate(shard);
        
        jdbcTemplate.update(
            "INSERT INTO users (user_id, username, email, created_at) VALUES (?, ?, ?, ?)",
            user.getUserId(),
            user.getUsername(), 
            user.getEmail(),
            user.getCreatedAt()
        );
    }
    
    // 크로스 샤드 쿼리 - 모든 샤드에서 검색
    public List<User> findUsersByUsername(String username) {
        List<CompletableFuture<List<User>>> futures = new ArrayList<>();
        
        for (DataSource shard : shards) {
            CompletableFuture<List<User>> future = CompletableFuture.supplyAsync(() -> {
                JdbcTemplate jdbcTemplate = new JdbcTemplate(shard);
                return jdbcTemplate.query(
                    "SELECT * FROM users WHERE username LIKE ?",
                    new Object[]{"%" + username + "%"},
                    new UserRowMapper()
                );
            });
            futures.add(future);
        }
        
        // 모든 샤드의 결과를 병합
        return futures.stream()
            .map(CompletableFuture::join)
            .flatMap(List::stream)
            .collect(Collectors.toList());
    }
    
    private DataSource getShard(String key) {
        String nodeName = hashRing.getNode(key);
        int shardIndex = Integer.parseInt(nodeName.split("-")[1]);
        return shards.get(shardIndex);
    }
}
```

## 일관성 모델

### 강한 일관성 (Strong Consistency)

```java
@Service
public class StrongConsistentInventoryService {
    
    @Autowired
    private DistributedLockService lockService;
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public boolean reserveInventory(String productId, int quantity) {
        // 분산 락으로 강한 일관성 보장
        DistributedLock lock = lockService.acquire("inventory:" + productId, 
            Duration.ofSeconds(30));
        
        try {
            Inventory inventory = inventoryRepository.findByProductId(productId);
            
            if (inventory.getAvailableQuantity() >= quantity) {
                inventory.reserve(quantity);
                
                // 모든 복제본에 동기식 업데이트
                inventoryRepository.saveWithSyncReplication(inventory);
                
                // 모든 노드에서 일관성 확인
                verifyConsistencyAcrossNodes(productId, inventory.getAvailableQuantity());
                
                return true;
            }
            
            return false;
            
        } finally {
            lock.release();
        }
    }
    
    private void verifyConsistencyAcrossNodes(String productId, int expectedQuantity) {
        List<String> nodes = clusterService.getAllNodes();
        
        for (String node : nodes) {
            int actualQuantity = inventoryRepository.getQuantityFromNode(node, productId);
            if (actualQuantity != expectedQuantity) {
                throw new ConsistencyViolationException(
                    String.format("노드 %s의 수량 불일치: 예상=%d, 실제=%d", 
                        node, expectedQuantity, actualQuantity)
                );
            }
        }
    }
}
```

### 최종 일관성 (Eventual Consistency)

```java
@Service
public class EventuallyConsistentInventoryService {
    
    @EventListener
    @Async
    public void handleInventoryUpdate(InventoryUpdateEvent event) {
        // 비동기로 모든 노드에 전파
        List<String> nodes = clusterService.getAllNodes();
        
        for (String node : nodes) {
            CompletableFuture.runAsync(() -> {
                try {
                    inventoryRepository.updateOnNode(node, event.getProductId(), 
                        event.getNewQuantity());
                } catch (Exception e) {
                    log.warn("노드 {}에 재고 업데이트 실패: {}", node, e.getMessage());
                    // 재시도 큐에 추가
                    retryQueue.add(new RetryTask(node, event));
                }
            });
        }
    }
    
    // 주기적으로 일관성 복구
    @Scheduled(fixedDelay = 60000) // 1분마다
    public void reconcileInventory() {
        List<String> productIds = inventoryRepository.getAllProductIds();
        
        for (String productId : productIds) {
            reconcileProductInventory(productId);
        }
    }
    
    private void reconcileProductInventory(String productId) {
        Map<String, Integer> nodeQuantities = new HashMap<>();
        
        // 모든 노드에서 현재 수량 조회
        for (String node : clusterService.getAllNodes()) {
            try {
                int quantity = inventoryRepository.getQuantityFromNode(node, productId);
                nodeQuantities.put(node, quantity);
            } catch (Exception e) {
                log.warn("노드 {}에서 재고 조회 실패: {}", node, e.getMessage());
            }
        }
        
        // 최신 버전 결정 (타임스탬프 기반)
        Integer correctQuantity = determineCorrectQuantity(productId, nodeQuantities);
        
        // 불일치하는 노드들 수정
        for (Map.Entry<String, Integer> entry : nodeQuantities.entrySet()) {
            if (!entry.getValue().equals(correctQuantity)) {
                try {
                    inventoryRepository.updateOnNode(entry.getKey(), productId, correctQuantity);
                    log.info("노드 {}의 재고 수정: {} -> {}", 
                        entry.getKey(), entry.getValue(), correctQuantity);
                } catch (Exception e) {
                    log.error("재고 수정 실패", e);
                }
            }
        }
    }
}
```

## 분산 합의 알고리즘

### Raft 알고리즘 기본 구현

```java
@Component
public class RaftNode {
    
    private volatile NodeState state = NodeState.FOLLOWER;
    private volatile String currentLeader;
    private volatile int currentTerm = 0;
    private volatile String votedFor;
    
    private final List<String> cluster;
    private final String nodeId;
    private final Map<String, Integer> nextIndex = new HashMap<>();
    private final Map<String, Integer> matchIndex = new HashMap<>();
    
    @Scheduled(fixedDelay = 150) // 150ms 하트비트
    public void heartbeatTimer() {
        if (state == NodeState.LEADER) {
            sendHeartbeats();
        }
    }
    
    @Scheduled(fixedDelay = 300) // 300ms 선거 타임아웃
    public void electionTimer() {
        if (state == NodeState.FOLLOWER && !hasRecentHeartbeat()) {
            startElection();
        }
    }
    
    private void startElection() {
        state = NodeState.CANDIDATE;
        currentTerm++;
        votedFor = nodeId;
        
        log.info("선거 시작: 임기 {}", currentTerm);
        
        int votes = 1; // 자신에게 투표
        
        for (String node : cluster) {
            if (!node.equals(nodeId)) {
                VoteResponse response = requestVote(node);
                if (response.isVoteGranted()) {
                    votes++;
                }
            }
        }
        
        if (votes > cluster.size() / 2) {
            becomeLeader();
        } else {
            state = NodeState.FOLLOWER;
        }
    }
    
    private void becomeLeader() {
        state = NodeState.LEADER;
        currentLeader = nodeId;
        
        log.info("리더가 됨: 임기 {}", currentTerm);
        
        // 모든 팔로워의 nextIndex 초기화
        for (String node : cluster) {
            if (!node.equals(nodeId)) {
                nextIndex.put(node, getLastLogIndex() + 1);
                matchIndex.put(node, 0);
            }
        }
        
        sendHeartbeats();
    }
    
    private void sendHeartbeats() {
        for (String node : cluster) {
            if (!node.equals(nodeId)) {
                AppendEntriesRequest request = AppendEntriesRequest.builder()
                    .term(currentTerm)
                    .leaderId(nodeId)
                    .prevLogIndex(nextIndex.get(node) - 1)
                    .prevLogTerm(getLogTerm(nextIndex.get(node) - 1))
                    .entries(Collections.emptyList()) // 하트비트는 빈 엔트리
                    .leaderCommit(getCommitIndex())
                    .build();
                
                CompletableFuture.runAsync(() -> {
                    AppendEntriesResponse response = sendAppendEntries(node, request);
                    handleAppendEntriesResponse(node, response);
                });
            }
        }
    }
    
    public AppendEntriesResponse handleAppendEntries(AppendEntriesRequest request) {
        if (request.getTerm() > currentTerm) {
            currentTerm = request.getTerm();
            votedFor = null;
            state = NodeState.FOLLOWER;
        }
        
        if (request.getTerm() < currentTerm) {
            return AppendEntriesResponse.builder()
                .term(currentTerm)
                .success(false)
                .build();
        }
        
        currentLeader = request.getLeaderId();
        resetElectionTimer();
        
        // 로그 일관성 검사
        if (request.getPrevLogIndex() > 0) {
            if (getLastLogIndex() < request.getPrevLogIndex() ||
                getLogTerm(request.getPrevLogIndex()) != request.getPrevLogTerm()) {
                
                return AppendEntriesResponse.builder()
                    .term(currentTerm)
                    .success(false)
                    .build();
            }
        }
        
        // 새 엔트리 추가
        if (!request.getEntries().isEmpty()) {
            appendLogEntries(request.getEntries(), request.getPrevLogIndex() + 1);
        }
        
        // 커밋 인덱스 업데이트
        if (request.getLeaderCommit() > getCommitIndex()) {
            setCommitIndex(Math.min(request.getLeaderCommit(), getLastLogIndex()));
        }
        
        return AppendEntriesResponse.builder()
            .term(currentTerm)
            .success(true)
            .build();
    }
}
```

## 실습 프로젝트: 분산 상품 관리 시스템

```java
@Service
public class DistributedProductService {
    
    private final ConsistentHashRing hashRing;
    private final List<ProductRepository> shardRepositories;
    private final EventPublisher eventPublisher;
    
    public Product createProduct(CreateProductRequest request) {
        // 1. 일관된 해싱으로 샤드 결정
        String shard = hashRing.getNode(request.getCategoryId());
        ProductRepository repository = getRepositoryForShard(shard);
        
        // 2. 로컬 샤드에 저장
        Product product = new Product(request);
        repository.save(product);
        
        // 3. 다른 시스템에 이벤트 발행 (최종 일관성)
        eventPublisher.publish(new ProductCreatedEvent(product));
        
        return product;
    }
    
    public List<Product> searchProducts(String query) {
        // 모든 샤드에 병렬 쿼리
        List<CompletableFuture<List<Product>>> futures = shardRepositories.stream()
            .map(repo -> CompletableFuture.supplyAsync(() -> 
                repo.findByNameContaining(query)))
            .collect(Collectors.toList());
        
        return futures.stream()
            .map(CompletableFuture::join)
            .flatMap(List::stream)
            .sorted(Comparator.comparing(Product::getCreatedAt).reversed())
            .collect(Collectors.toList());
    }
}
```

## 핵심 포인트

1. **CAP는 이분법이 아님**: 네트워크 분할 시 일관성과 가용성 중 하나를 선택하는 것이지, 평상시에는 둘 다 가질 수 있습니다.

2. **최종 일관성이 현실적**: 대부분의 대규모 시스템은 강한 일관성보다 최종 일관성을 선택합니다.

3. **일관된 해싱은 확장의 핵심**: 노드 추가/제거 시 최소한의 데이터 재분배로 시스템 확장이 가능합니다.

4. **분산 합의는 복잡함**: Raft 같은 알고리즘을 직접 구현하기보다는 검증된 라이브러리 사용을 권장합니다.

5. **모니터링과 관찰성**: 분산 시스템에서는 각 노드의 상태와 일관성을 지속적으로 모니터링해야 합니다.

이러한 분산 시스템의 기초는 다음 단계의 로드 밸런싱과 트래픽 분산 패턴을 이해하는 데 필수적입니다.
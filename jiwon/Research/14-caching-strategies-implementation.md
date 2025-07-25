# 1-4.캐싱 전략과 구현: 다층 캐싱 아키텍처와 분산 캐시 최적화

## 개요

캐싱은 고성능 시스템의 핵심 구성 요소입니다. 이 문서는 브라우저부터 데이터베이스까지의 다층 캐싱 아키텍처, 다양한 캐싱 패턴의 특성과 적용 시나리오, 그리고 분산 캐시 환경에서의 일관성 관리를 다룹니다. 시니어 개발자로서 캐싱 계층별 최적화 전략과 성능 향상 기법을 학습합니다.

## 다층 캐싱 아키텍처

### 캐시 계층 구조

```
┌─────────────────┐
│   Browser Cache │ ← HTTP 헤더 기반, 클라이언트 제어
└─────────────────┘
         ↓
┌─────────────────┐
│   CDN Cache     │ ← 지리적 분산, 정적 콘텐츠
└─────────────────┘
         ↓
┌─────────────────┐
│ Reverse Proxy   │ ← Varnish/NGINX, 애플리케이션 레벨
└─────────────────┘
         ↓
┌─────────────────┐
│Application Cache│ ← Redis/Memcached, 분산 캐시
└─────────────────┘
         ↓
┌─────────────────┐
│ Database Cache  │ ← 쿼리 결과, 버퍼 풀
└─────────────────┘
```

### 각 계층별 특성과 최적화

**1. 브라우저 캐시 (Browser Cache)**
```java
@RestController
public class StaticResourceController {
    
    @GetMapping("/api/config")
    public ResponseEntity<ConfigResponse> getConfig() {
        ConfigResponse config = configService.getConfig();
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(Duration.ofHours(1))
                .cachePublic()
                .mustRevalidate())
            .eTag(config.getVersion())
            .body(config);
    }
    
    @GetMapping("/api/user/profile")
    public ResponseEntity<UserProfile> getUserProfile(@PathVariable String userId) {
        UserProfile profile = userService.getProfile(userId);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5))
                .cachePrivate())
            .lastModified(profile.getLastModified())
            .body(profile);
    }
    
    @GetMapping("/assets/**")
    public ResponseEntity<Resource> getStaticAsset(HttpServletRequest request) {
        // 정적 자원은 장기간 캐싱
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(Duration.ofDays(365))
                .cachePublic()
                .immutable())
            .body(resourceLoader.getResource(request.getRequestURI()));
    }
}
```

**2. CDN 캐시 최적화**
```java
@Configuration
public class CdnConfiguration {
    
    @Bean
    public CloudFrontConfiguration cloudFrontConfig() {
        return CloudFrontConfiguration.builder()
            .distributionConfig(DistributionConfig.builder()
                .callerReference(UUID.randomUUID().toString())
                .comment("E-commerce CDN")
                .defaultRootObject("index.html")
                .enabled(true)
                
                // 캐시 동작 설정
                .defaultCacheBehavior(DefaultCacheBehavior.builder()
                    .targetOriginId("api-origin")
                    .viewerProtocolPolicy(ViewerProtocolPolicy.REDIRECT_TO_HTTPS)
                    .cachePolicyId("managed-caching-optimized")
                    .compress(true)
                    .build())
                
                // 정적 자원 캐시 동작
                .cacheBehaviors(CacheBehaviors.builder()
                    .items(CacheBehavior.builder()
                        .pathPattern("/assets/*")
                        .targetOriginId("s3-origin")
                        .viewerProtocolPolicy(ViewerProtocolPolicy.REDIRECT_TO_HTTPS)
                        .cachePolicyId("managed-caching-optimized")
                        .compress(true)
                        .ttl(Duration.ofDays(30))
                        .build())
                    .build())
                    
                .build())
            .build();
    }
}

@Service
public class CdnInvalidationService {
    
    private final CloudFrontClient cloudFrontClient;
    
    public void invalidateCache(List<String> paths) {
        InvalidationBatch batch = InvalidationBatch.builder()
            .paths(Paths.builder()
                .quantity(paths.size())
                .items(paths)
                .build())
            .callerReference(UUID.randomUUID().toString())
            .build();
        
        CreateInvalidationRequest request = CreateInvalidationRequest.builder()
            .distributionId("E1PA6795UKMFR9")
            .invalidationBatch(batch)
            .build();
        
        cloudFrontClient.createInvalidation(request);
        log.info("CDN 캐시 무효화 요청: {}", paths);
    }
    
    @EventListener
    public void handleProductUpdate(ProductUpdatedEvent event) {
        // 상품 정보 변경 시 관련 캐시 무효화
        List<String> paths = Arrays.asList(
            "/api/products/" + event.getProductId(),
            "/api/categories/" + event.getCategoryId(),
            "/assets/images/products/" + event.getProductId() + "/*"
        );
        
        invalidateCache(paths);
    }
}
```

**3. 리버스 프록시 캐시 (NGINX)**
```nginx
# nginx.conf
http {
    # 캐시 경로 설정
    proxy_cache_path /var/cache/nginx/api 
                     levels=1:2 
                     keys_zone=api_cache:100m 
                     max_size=1g 
                     inactive=60m 
                     use_temp_path=off;
    
    proxy_cache_path /var/cache/nginx/static 
                     levels=1:2 
                     keys_zone=static_cache:100m 
                     max_size=5g 
                     inactive=30d 
                     use_temp_path=off;
    
    # API 캐싱 설정
    upstream api_servers {
        server app1:8080;
        server app2:8080;
        server app3:8080;
    }
    
    server {
        listen 80;
        
        # API 응답 캐싱
        location /api/products {
            proxy_pass http://api_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            
            # 캐시 설정
            proxy_cache api_cache;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            proxy_cache_valid 200 302 5m;
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;
            proxy_cache_lock on;
            
            # 캐시 상태 헤더 추가
            add_header X-Cache-Status $upstream_cache_status;
            
            # 캐시 우회 조건
            proxy_cache_bypass $http_x_no_cache $arg_nocache;
            proxy_no_cache $http_x_no_cache $arg_nocache;
        }
        
        # 정적 파일 캐싱
        location /assets/ {
            proxy_pass http://api_servers;
            
            proxy_cache static_cache;
            proxy_cache_key "$host$request_uri";
            proxy_cache_valid 200 30d;
            proxy_cache_valid 404 1h;
            
            # 긴 캐시 기간
            expires 1y;
            add_header Cache-Control "public, immutable";
            add_header X-Cache-Status $upstream_cache_status;
        }
        
        # 캐시 퍼지 엔드포인트 (관리용)
        location /nginx-cache-purge {
            allow 192.168.1.0/24;
            deny all;
            
            proxy_cache_purge api_cache "$scheme$request_method$host$arg_uri";
        }
    }
}
```

## 캐싱 패턴 구현

### Cache-Aside (Lazy Loading) 패턴

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    private static final String CACHE_KEY_PREFIX = "product:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(30);
    
    public Product getProduct(String productId) {
        String cacheKey = CACHE_KEY_PREFIX + productId;
        
        // 1. 캐시에서 조회
        Product cachedProduct = redisTemplate.opsForValue().get(cacheKey);
        if (cachedProduct != null) {
            log.debug("캐시 히트: {}", productId);
            return cachedProduct;
        }
        
        // 2. 캐시 미스 - 데이터베이스에서 조회
        log.debug("캐시 미스: {}", productId);
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        // 3. 캐시에 저장
        redisTemplate.opsForValue().set(cacheKey, product, CACHE_TTL);
        
        return product;
    }
    
    public Product updateProduct(Product product) {
        // 1. 데이터베이스 업데이트
        Product updatedProduct = productRepository.save(product);
        
        // 2. 캐시 무효화
        String cacheKey = CACHE_KEY_PREFIX + product.getId();
        redisTemplate.delete(cacheKey);
        
        // 3. 이벤트 발행 (다른 캐시 레이어 무효화용)
        eventPublisher.publishEvent(new ProductUpdatedEvent(updatedProduct));
        
        return updatedProduct;
    }
}
```

### Write-Through 패턴

```java
@Service
public class WriteThoughProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    @Transactional
    public Product createProduct(CreateProductRequest request) {
        Product product = new Product(request);
        
        // 1. 데이터베이스에 저장
        Product savedProduct = productRepository.save(product);
        
        // 2. 즉시 캐시에 저장
        String cacheKey = CACHE_KEY_PREFIX + savedProduct.getId();
        redisTemplate.opsForValue().set(cacheKey, savedProduct, CACHE_TTL);
        
        log.info("상품 생성 및 캐시 저장: {}", savedProduct.getId());
        return savedProduct;
    }
    
    @Transactional
    public Product updateProduct(String productId, UpdateProductRequest request) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product.update(request);
        
        // 1. 데이터베이스 업데이트
        Product updatedProduct = productRepository.save(product);
        
        // 2. 캐시도 즉시 업데이트
        String cacheKey = CACHE_KEY_PREFIX + productId;
        redisTemplate.opsForValue().set(cacheKey, updatedProduct, CACHE_TTL);
        
        log.info("상품 업데이트 및 캐시 갱신: {}", productId);
        return updatedProduct;
    }
}
```

### Write-Behind (Write-Back) 패턴

```java
@Service
public class WriteBehindProductService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    @Autowired
    private TaskExecutor taskExecutor;
    
    private final BlockingQueue<Product> writeQueue = new LinkedBlockingQueue<>();
    
    @PostConstruct
    public void initializeWriteBehindProcessor() {
        // 백그라운드에서 주기적으로 데이터베이스에 쓰기
        taskExecutor.execute(this::processWriteQueue);
    }
    
    public Product updateProductAsync(String productId, UpdateProductRequest request) {
        Product product = getProduct(productId);
        product.update(request);
        
        // 1. 캐시에 즉시 저장
        String cacheKey = CACHE_KEY_PREFIX + productId;
        redisTemplate.opsForValue().set(cacheKey, product, CACHE_TTL);
        
        // 2. 쓰기 큐에 추가 (비동기 데이터베이스 쓰기)
        writeQueue.offer(product);
        
        log.info("상품 캐시 업데이트 완료, DB 쓰기 예약: {}", productId);
        return product;
    }
    
    private void processWriteQueue() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                List<Product> batch = new ArrayList<>();
                
                // 배치 단위로 처리
                Product product = writeQueue.take(); // 블로킹
                batch.add(product);
                
                // 추가 항목들을 배치에 추가 (최대 100개)
                writeQueue.drainTo(batch, 99);
                
                // 배치 단위로 데이터베이스에 저장
                productRepository.saveAll(batch);
                
                log.info("배치 DB 쓰기 완료: {} 건", batch.size());
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                log.error("DB 쓰기 실패", e);
                // 실패한 항목들을 다시 큐에 추가하거나 별도 처리
            }
        }
    }
    
    // 애플리케이션 종료 시 남은 데이터 처리
    @PreDestroy
    public void flushPendingWrites() {
        log.info("애플리케이션 종료 - 남은 쓰기 작업 처리: {} 건", writeQueue.size());
        
        List<Product> remaining = new ArrayList<>();
        writeQueue.drainTo(remaining);
        
        if (!remaining.isEmpty()) {
            productRepository.saveAll(remaining);
            log.info("종료 시 DB 쓰기 완료: {} 건", remaining.size());
        }
    }
}
```

## 다중 레벨 캐싱 구현

### L1 (로컬) + L2 (분산) 캐시 조합

```java
@Configuration
public class MultiLevelCacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CompositeCacheManager compositeCacheManager = new CompositeCacheManager();
        
        List<CacheManager> cacheManagers = Arrays.asList(
            caffeineL1CacheManager(),
            redisL2CacheManager()
        );
        
        compositeCacheManager.setCacheManagers(cacheManagers);
        compositeCacheManager.setFallbackToNoOpCache(false);
        
        return compositeCacheManager;
    }
    
    @Bean
    public CacheManager caffeineL1CacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats());
        
        return cacheManager;
    }
    
    @Bean
    public CacheManager redisL2CacheManager() {
        RedisCacheManager.Builder builder = RedisCacheManager.RedisCacheManagerBuilder
            .fromConnectionFactory(redisConnectionFactory())
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())));
        
        return builder.build();
    }
}

@Service
public class MultiLevelCacheService {
    
    @Autowired
    private CaffeineCache l1Cache;
    
    @Autowired
    private RedisTemplate<String, Object> l2Cache;
    
    @Autowired
    private ProductRepository repository;
    
    public Product getProduct(String productId) {
        String cacheKey = "product:" + productId;
        
        // L1 캐시 확인 (로컬 메모리)
        Product product = l1Cache.get(cacheKey, Product.class);
        if (product != null) {
            cacheMetrics.recordL1Hit();
            return product;
        }
        
        // L2 캐시 확인 (Redis)
        product = (Product) l2Cache.opsForValue().get(cacheKey);
        if (product != null) {
            // L1 캐시에 저장
            l1Cache.put(cacheKey, product);
            cacheMetrics.recordL2Hit();
            return product;
        }
        
        // 데이터베이스에서 조회
        product = repository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        // 양쪽 캐시에 저장
        l2Cache.opsForValue().set(cacheKey, product, Duration.ofMinutes(30));
        l1Cache.put(cacheKey, product);
        
        cacheMetrics.recordCacheMiss();
        return product;
    }
    
    public void evictProduct(String productId) {
        String cacheKey = "product:" + productId;
        
        // 양쪽 캐시에서 제거
        l1Cache.evict(cacheKey);
        l2Cache.delete(cacheKey);
        
        // 다른 인스턴스의 L1 캐시 무효화를 위한 이벤트 발행
        eventPublisher.publishEvent(new CacheEvictionEvent(cacheKey));
    }
}

@EventListener
@Component
public class CacheEvictionEventHandler {
    
    @Autowired
    private CaffeineCache l1Cache;
    
    @EventListener
    public void handleCacheEviction(CacheEvictionEvent event) {
        // 다른 인스턴스에서 발생한 캐시 무효화 이벤트 처리
        l1Cache.evict(event.getCacheKey());
        log.debug("L1 캐시 무효화: {}", event.getCacheKey());
    }
}
```

### 캐시 워밍 (Cache Warming) 전략

```java
@Component
public class CacheWarmupService {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private CategoryService categoryService;
    
    @Autowired
    private PopularProductService popularProductService;
    
    @EventListener(ApplicationReadyEvent.class)
    public void warmupCache() {
        log.info("캐시 워밍 시작");
        
        CompletableFuture.allOf(
            CompletableFuture.runAsync(this::warmupPopularProducts),
            CompletableFuture.runAsync(this::warmupCategories),
            CompletableFuture.runAsync(this::warmupConfiguration)
        ).join();
        
        log.info("캐시 워밍 완료");
    }
    
    private void warmupPopularProducts() {
        try {
            List<String> popularProductIds = popularProductService.getTop100ProductIds();
            
            popularProductIds.parallelStream()
                .forEach(productId -> {
                    try {
                        productService.getProduct(productId);
                        log.debug("인기 상품 캐시 워밍: {}", productId);
                    } catch (Exception e) {
                        log.warn("상품 캐시 워밍 실패: {}", productId, e);
                    }
                });
            
            log.info("인기 상품 캐시 워밍 완료: {} 개", popularProductIds.size());
            
        } catch (Exception e) {
            log.error("인기 상품 캐시 워밍 실패", e);
        }
    }
    
    private void warmupCategories() {
        try {
            List<Category> categories = categoryService.getAllCategories();
            
            categories.forEach(category -> {
                try {
                    categoryService.getCategoryWithProducts(category.getId());
                    log.debug("카테고리 캐시 워밍: {}", category.getId());
                } catch (Exception e) {
                    log.warn("카테고리 캐시 워밍 실패: {}", category.getId(), e);
                }
            });
            
            log.info("카테고리 캐시 워밍 완료: {} 개", categories.size());
            
        } catch (Exception e) {
            log.error("카테고리 캐시 워밍 실패", e);
        }
    }
    
    @Scheduled(cron = "0 0 2 * * ?") // 매일 새벽 2시
    public void scheduledCacheWarmup() {
        log.info("정기 캐시 워밍 시작");
        warmupCache();
    }
}
```

## 분산 캐시 일관성 관리

### 캐시 무효화 전략

```java
@Service
public class CacheInvalidationService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void invalidateProductCache(String productId) {
        List<String> keysToInvalidate = Arrays.asList(
            "product:" + productId,
            "product:detail:" + productId,
            "product:reviews:" + productId,
            "category:products:*" // 와일드카드 패턴
        );
        
        // Redis에서 키 삭제
        keysToInvalidate.forEach(pattern -> {
            if (pattern.contains("*")) {
                Set<String> keys = redisTemplate.keys(pattern);
                if (!keys.isEmpty()) {
                    redisTemplate.delete(keys);
                }
            } else {
                redisTemplate.delete(pattern);
            }
        });
        
        // 다른 애플리케이션 인스턴스에 무효화 이벤트 발행
        eventPublisher.publishEvent(new CacheInvalidationEvent(productId, keysToInvalidate));
        
        log.info("상품 캐시 무효화 완료: {}", productId);
    }
    
    public void invalidateCategoryCache(String categoryId) {
        // 카테고리와 관련된 모든 캐시 무효화
        Set<String> keys = redisTemplate.keys("category:" + categoryId + ":*");
        keys.addAll(redisTemplate.keys("product:category:" + categoryId + ":*"));
        
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
        
        log.info("카테고리 캐시 무효화 완료: {} ({} 키)", categoryId, keys.size());
    }
    
    // 캐시 태그 기반 무효화
    public void invalidateByTags(Set<String> tags) {
        tags.forEach(tag -> {
            Set<String> taggedKeys = redisTemplate.opsForSet().members("tag:" + tag);
            if (taggedKeys != null && !taggedKeys.isEmpty()) {
                redisTemplate.delete(taggedKeys);
                redisTemplate.delete("tag:" + tag); // 태그 자체도 삭제
            }
        });
        
        log.info("태그 기반 캐시 무효화 완료: {}", tags);
    }
}

@Component
public class CacheTagManager {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void addCacheTag(String cacheKey, String... tags) {
        for (String tag : tags) {
            redisTemplate.opsForSet().add("tag:" + tag, cacheKey);
        }
    }
    
    public void addProductToCache(String productId, Product product, String... additionalTags) {
        String cacheKey = "product:" + productId;
        
        // 캐시에 저장
        redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(30));
        
        // 태그 추가
        List<String> tags = new ArrayList<>();
        tags.add("product");
        tags.add("category:" + product.getCategoryId());
        tags.add("brand:" + product.getBrandId());
        tags.addAll(Arrays.asList(additionalTags));
        
        addCacheTag(cacheKey, tags.toArray(new String[0]));
    }
}
```

### Redis Cluster 구성

```java
@Configuration
public class RedisClusterConfig {
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration clusterConfiguration = new RedisClusterConfiguration();
        
        // 클러스터 노드 설정
        clusterConfiguration.clusterNode("redis-node1", 7000);
        clusterConfiguration.clusterNode("redis-node2", 7001);
        clusterConfiguration.clusterNode("redis-node3", 7002);
        clusterConfiguration.clusterNode("redis-node4", 7003);
        clusterConfiguration.clusterNode("redis-node5", 7004);
        clusterConfiguration.clusterNode("redis-node6", 7005);
        
        // 클러스터 옵션
        clusterConfiguration.setMaxRedirects(3);
        
        // 연결 풀 설정
        GenericObjectPoolConfig<Connection> poolConfig = new GenericObjectPoolConfig<>();
        poolConfig.setMaxTotal(100);
        poolConfig.setMaxIdle(20);
        poolConfig.setMinIdle(5);
        poolConfig.setTestOnBorrow(true);
        poolConfig.setTestOnReturn(true);
        poolConfig.setTestWhileIdle(true);
        
        LettucePoolingClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
            .poolConfig(poolConfig)
            .commandTimeout(Duration.ofSeconds(5))
            .shutdownTimeout(Duration.ofSeconds(2))
            .build();
        
        return new LettuceConnectionFactory(clusterConfiguration, clientConfig);
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        // 직렬화 설정
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, 
            ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(objectMapper);
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
}
```

## 캐시 성능 모니터링

### 캐시 메트릭 수집

```java
@Component
public class CacheMetricsService {
    
    private final MeterRegistry meterRegistry;
    private final Counter cacheHitCounter;
    private final Counter cacheMissCounter;
    private final Timer cacheOperationTimer;
    private final Gauge cacheSize;
    
    public CacheMetricsService(MeterRegistry meterRegistry, CacheManager cacheManager) {
        this.meterRegistry = meterRegistry;
        
        this.cacheHitCounter = Counter.builder("cache.hits")
            .description("캐시 히트 수")
            .register(meterRegistry);
            
        this.cacheMissCounter = Counter.builder("cache.misses")
            .description("캐시 미스 수")
            .register(meterRegistry);
            
        this.cacheOperationTimer = Timer.builder("cache.operation.time")
            .description("캐시 작업 소요 시간")
            .register(meterRegistry);
            
        // 캐시 크기 게이지
        this.cacheSize = Gauge.builder("cache.size")
            .description("캐시 아이템 수")
            .register(meterRegistry, this, CacheMetricsService::getCurrentCacheSize);
    }
    
    public void recordCacheHit(String cacheName) {
        cacheHitCounter.increment(Tags.of("cache", cacheName));
    }
    
    public void recordCacheMiss(String cacheName) {
        cacheMissCounter.increment(Tags.of("cache", cacheName));
    }
    
    public void recordCacheOperation(String operation, String cacheName, Duration duration) {
        cacheOperationTimer.record(duration, Tags.of("operation", operation, "cache", cacheName));
    }
    
    // 캐시 히트율 계산
    @Scheduled(fixedDelay = 60000) // 1분마다
    public void calculateHitRatio() {
        double hits = cacheHitCounter.count();
        double misses = cacheMissCounter.count();
        double total = hits + misses;
        
        if (total > 0) {
            double hitRatio = hits / total * 100;
            Gauge.builder("cache.hit.ratio")
                .description("캐시 히트율 (%)")
                .register(meterRegistry)
                .set(hitRatio);
            
            log.info("캐시 히트율: {:.2f}% (히트: {}, 미스: {})", hitRatio, hits, misses);
        }
    }
    
    private double getCurrentCacheSize() {
        // Redis 클러스터에서 총 키 수 조회
        try {
            return redisTemplate.getConnectionFactory()
                .getConnection()
                .dbSize();
        } catch (Exception e) {
            log.warn("캐시 크기 조회 실패", e);
            return 0;
        }
    }
}

@RestController
@RequestMapping("/admin/cache")
public class CacheManagementController {
    
    @Autowired
    private CacheMetricsService cacheMetrics;
    
    @Autowired
    private CacheInvalidationService cacheInvalidation;
    
    @GetMapping("/stats")
    public ResponseEntity<CacheStats> getCacheStats() {
        CacheStats stats = CacheStats.builder()
            .hitCount(cacheMetrics.getHitCount())
            .missCount(cacheMetrics.getMissCount())
            .hitRatio(cacheMetrics.getHitRatio())
            .evictionCount(cacheMetrics.getEvictionCount())
            .averageLoadTime(cacheMetrics.getAverageLoadTime())
            .build();
        
        return ResponseEntity.ok(stats);
    }
    
    @DeleteMapping("/invalidate/product/{productId}")
    public ResponseEntity<String> invalidateProduct(@PathVariable String productId) {
        cacheInvalidation.invalidateProductCache(productId);
        return ResponseEntity.ok("상품 캐시 무효화 완료: " + productId);
    }
    
    @DeleteMapping("/invalidate/category/{categoryId}")
    public ResponseEntity<String> invalidateCategory(@PathVariable String categoryId) {
        cacheInvalidation.invalidateCategoryCache(categoryId);
        return ResponseEntity.ok("카테고리 캐시 무효화 완료: " + categoryId);
    }
    
    @DeleteMapping("/clear")
    public ResponseEntity<String> clearAllCache() {
        redisTemplate.getConnectionFactory()
            .getConnection()
            .flushAll();
        return ResponseEntity.ok("모든 캐시 클리어 완료");
    }
}
```

### 캐시 성능 최적화

```java
@Component
public class CacheOptimizer {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 배치 캐시 로딩으로 N+1 문제 해결
    public Map<String, Product> getProductsBatch(List<String> productIds) {
        if (productIds.isEmpty()) {
            return Collections.emptyMap();
        }
        
        // Redis Pipeline을 사용한 배치 조회
        List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            StringRedisConnection stringRedisConn = (StringRedisConnection) connection;
            
            productIds.forEach(id -> {
                String key = "product:" + id;
                stringRedisConn.get(key);
            });
            
            return null;
        });
        
        Map<String, Product> cachedProducts = new HashMap<>();
        List<String> missedProductIds = new ArrayList<>();
        
        for (int i = 0; i < productIds.size(); i++) {
            String productId = productIds.get(i);
            Object result = results.get(i);
            
            if (result != null) {
                cachedProducts.put(productId, (Product) result);
            } else {
                missedProductIds.add(productId);
            }
        }
        
        // 캐시 미스된 상품들을 데이터베이스에서 배치 조회
        if (!missedProductIds.isEmpty()) {
            List<Product> productsFromDb = productRepository.findByIdIn(missedProductIds);
            
            // 조회된 상품들을 캐시에 배치 저장
            redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
                StringRedisConnection stringRedisConn = (StringRedisConnection) connection;
                
                productsFromDb.forEach(product -> {
                    String key = "product:" + product.getId();
                    try {
                        String value = objectMapper.writeValueAsString(product);
                        stringRedisConn.setEx(key, 1800, value); // 30분 TTL
                    } catch (Exception e) {
                        log.error("캐시 저장 실패: {}", product.getId(), e);
                    }
                });
                
                return null;
            });
            
            productsFromDb.forEach(product -> 
                cachedProducts.put(product.getId(), product));
        }
        
        return cachedProducts;
    }
    
    // 캐시 키 압축으로 메모리 절약
    public String compressKey(String originalKey) {
        return DigestUtils.md5Hex(originalKey);
    }
    
    // 캐시 값 압축
    public void setCachedValueCompressed(String key, Object value, Duration ttl) {
        try {
            String jsonValue = objectMapper.writeValueAsString(value);
            byte[] compressedValue = compress(jsonValue.getBytes());
            
            redisTemplate.opsForValue().set(key + ":compressed", compressedValue, ttl);
            
        } catch (Exception e) {
            log.error("압축 캐시 저장 실패: {}", key, e);
        }
    }
    
    public <T> T getCachedValueCompressed(String key, Class<T> type) {
        try {
            byte[] compressedValue = (byte[]) redisTemplate.opsForValue().get(key + ":compressed");
            
            if (compressedValue != null) {
                byte[] decompressedValue = decompress(compressedValue);
                String jsonValue = new String(decompressedValue);
                return objectMapper.readValue(jsonValue, type);
            }
            
        } catch (Exception e) {
            log.error("압축 캐시 조회 실패: {}", key, e);
        }
        
        return null;
    }
    
    private byte[] compress(byte[] data) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (GZIPOutputStream gzipOut = new GZIPOutputStream(baos)) {
            gzipOut.write(data);
        }
        return baos.toByteArray();
    }
    
    private byte[] decompress(byte[] compressedData) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (GZIPInputStream gzipIn = new GZIPInputStream(new ByteArrayInputStream(compressedData))) {
            byte[] buffer = new byte[1024];
            int len;
            while ((len = gzipIn.read(buffer)) != -1) {
                baos.write(buffer, 0, len);
            }
        }
        return baos.toByteArray();
    }
}
```

## 실습 프로젝트: 전자상거래 캐싱 시스템

```yaml
# docker-compose.yml (캐싱 시스템)
version: '3.8'

services:
  # 애플리케이션 서버들
  app1:
    build: .
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - REDIS_CLUSTER_NODES=redis-node1:7000,redis-node2:7001,redis-node3:7002
    depends_on:
      - redis-node1
      - redis-node2
      - redis-node3

  app2:
    build: .
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - REDIS_CLUSTER_NODES=redis-node1:7000,redis-node2:7001,redis-node3:7002
    depends_on:
      - redis-node1
      - redis-node2
      - redis-node3

  # NGINX 리버스 프록시 (캐싱 포함)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - nginx_cache:/var/cache/nginx
    depends_on:
      - app1
      - app2

  # Redis 클러스터
  redis-node1:
    image: redis:alpine
    command: redis-server --port 7000 --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes
    ports:
      - "7000:7000"
    volumes:
      - redis1_data:/data

  redis-node2:
    image: redis:alpine  
    command: redis-server --port 7001 --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes
    ports:
      - "7001:7001"
    volumes:
      - redis2_data:/data

  redis-node3:
    image: redis:alpine
    command: redis-server --port 7002 --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes
    ports:
      - "7002:7002"
    volumes:
      - redis3_data:/data

  # 캐시 모니터링
  redis-commander:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=cluster:redis-node1:7000,redis-node2:7001,redis-node3:7002
    ports:
      - "8081:8081"

volumes:
  nginx_cache:
  redis1_data:
  redis2_data:
  redis3_data:
```

## 핵심 포인트

1. **캐시 계층 선택**: 데이터 특성과 액세스 패턴에 따라 적절한 캐시 레벨 선택

2. **캐싱 패턴**: Cache-Aside는 읽기 중심, Write-Through는 일관성 중시, Write-Behind는 쓰기 성능 중시

3. **분산 캐시 일관성**: 캐시 무효화 전략과 이벤트 기반 동기화가 핵심

4. **성능 모니터링**: 히트율, 응답시간, 메모리 사용량을 지속적으로 모니터링

5. **최적화 기법**: 배치 로딩, 압축, 파이프라이닝으로 성능 향상 가능

이러한 캐싱 전략은 Phase 1의 마지막 단계로서, 시스템의 전반적인 성능을 크게 향상시키며 Phase 2의 고급 확장 기법을 위한 기반을 제공합니다.
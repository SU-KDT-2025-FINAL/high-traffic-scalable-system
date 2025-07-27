# 성능 최적화와 튜닝

## 목차
1. [성능 최적화 개요](#1-성능-최적화-개요)
2. [애플리케이션 성능 프로파일링](#2-애플리케이션-성능-프로파일링)
3. [JVM 튜닝과 가비지 컬렉션 최적화](#3-jvm-튜닝과-가비지-컬렉션-최적화)
4. [데이터베이스 쿼리 최적화](#4-데이터베이스-쿼리-최적화)
5. [네트워크 최적화와 CDN 활용](#5-네트워크-최적화와-cdn-활용)
6. [캐시 최적화 전략](#6-캐시-최적화-전략)
7. [실전 성능 최적화 사례](#7-실전-성능-최적화-사례)

---

## 1. 성능 최적화 개요

### 1.1 성능 최적화의 원칙

#### 측정 우선 원칙 (Measure First)
최적화하기 전에 반드시 현재 성능을 측정하고 병목점을 식별해야 합니다.

```java
// 성능 측정을 위한 기본 구조
@Component
public class PerformanceProfiler {
    private final MeterRegistry meterRegistry;
    private final Timer performanceTimer;
    
    public PerformanceProfiler(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.performanceTimer = Timer.builder("method.execution.time")
            .description("Method execution time")
            .register(meterRegistry);
    }
    
    public <T> T measurePerformance(String methodName, Supplier<T> operation) {
        return Timer.Sample.start(meterRegistry)
            .stop(performanceTimer.tags("method", methodName), operation);
    }
}
```

#### 파레토 법칙 (80/20 Rule)
시스템 성능의 80%는 20%의 코드에서 결정됩니다. 핵심 병목점에 집중해야 합니다.

#### 조기 최적화 금지
"조기 최적화는 모든 악의 근원"이라는 원칙을 따라 실제 병목점이 확인된 후 최적화를 진행합니다.

### 1.2 성능 지표와 측정

#### 핵심 성능 지표
```java
@Component
public class PerformanceMetrics {
    
    // 응답 시간 측정
    @EventListener
    public void recordResponseTime(HttpRequestProcessedEvent event) {
        Timer.Sample.start()
            .stop(Timer.builder("http.request.duration")
                .tag("uri", event.getUri())
                .tag("method", event.getMethod())
                .tag("status", String.valueOf(event.getStatus()))
                .register(meterRegistry));
    }
    
    // 처리량 측정
    @EventListener
    public void recordThroughput(RequestCompletedEvent event) {
        Counter.builder("http.requests.total")
            .tag("endpoint", event.getEndpoint())
            .tag("status", event.getStatus())
            .register(meterRegistry)
            .increment();
    }
    
    // 에러율 측정
    @EventListener
    public void recordErrorRate(ErrorEvent event) {
        Counter.builder("http.errors.total")
            .tag("type", event.getErrorType())
            .tag("severity", event.getSeverity())
            .register(meterRegistry)
            .increment();
    }
}
```

#### 성능 벤치마킹
```bash
# Apache Bench를 사용한 부하 테스트
ab -n 10000 -c 100 -H "Authorization: Bearer token" \
   http://localhost:8080/api/products

# JMeter를 사용한 복합 시나리오 테스트
jmeter -n -t load-test.jmx -l results.jtl -e -o report/

# wrk를 사용한 고성능 부하 테스트
wrk -t12 -c400 -d30s --script=auth.lua http://localhost:8080/api/orders
```

---

## 2. 애플리케이션 성능 프로파일링

### 2.1 프로파일링 도구

#### Java 애플리케이션 프로파일링
```java
// JFR (Java Flight Recorder) 설정
@Configuration
public class ProfilingConfig {
    
    @Bean
    @ConditionalOnProperty("profiling.jfr.enabled")
    public JfrEventHandler jfrEventHandler() {
        return new JfrEventHandler();
    }
}

@Component
public class JfrEventHandler {
    
    @EventListener
    public void handleSlowQuery(SlowQueryEvent event) {
        try (var jfrEvent = new DatabaseQueryEvent()) {
            jfrEvent.setQuery(event.getQuery());
            jfrEvent.setDuration(event.getDuration());
            jfrEvent.setDatabase(event.getDatabase());
            jfrEvent.commit();
        }
    }
}

// 커스텀 JFR 이벤트 정의
@Category("Application")
@Label("Database Query")
public class DatabaseQueryEvent extends Event {
    @Label("Query")
    public String query;
    
    @Label("Duration")
    @Timespan(Timespan.MILLISECONDS)
    public long duration;
    
    @Label("Database")
    public String database;
}
```

#### Async Profiler 통합
```java
@RestController
public class ProfilingController {
    
    private final AsyncProfiler profiler = AsyncProfiler.getInstance();
    
    @PostMapping("/profiling/start")
    public ResponseEntity<String> startProfiling(
            @RequestParam(defaultValue = "cpu") String event,
            @RequestParam(defaultValue = "60") int duration) {
        
        try {
            String command = String.format("start,event=%s,file=profile-%d.html", 
                event, System.currentTimeMillis());
            profiler.execute(command);
            
            // duration 후 자동 정지
            CompletableFuture.delayedExecutor(duration, TimeUnit.SECONDS)
                .execute(() -> {
                    try {
                        profiler.execute("stop");
                    } catch (Exception e) {
                        log.error("Failed to stop profiler", e);
                    }
                });
            
            return ResponseEntity.ok("Profiling started for " + duration + " seconds");
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Failed to start profiling: " + e.getMessage());
        }
    }
}
```

### 2.2 성능 병목점 식별

#### 느린 메서드 자동 감지
```java
@Aspect
@Component
public class SlowMethodDetector {
    
    private static final long SLOW_THRESHOLD_MS = 1000;
    
    @Around("@annotation(Timed) || execution(* com.example.service.*.*(..))")
    public Object detectSlowMethods(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            return result;
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            
            if (duration > SLOW_THRESHOLD_MS) {
                String methodName = joinPoint.getSignature().toShortString();
                log.warn("Slow method detected: {} took {}ms", methodName, duration);
                
                // 느린 메서드 이벤트 발행
                applicationEventPublisher.publishEvent(
                    new SlowMethodEvent(methodName, duration, joinPoint.getArgs())
                );
            }
        }
    }
}

// 느린 메서드 분석
@EventListener
@Component
public class SlowMethodAnalyzer {
    
    private final Map<String, SlowMethodStats> methodStats = new ConcurrentHashMap<>();
    
    @EventListener
    public void analyzeSlowMethod(SlowMethodEvent event) {
        methodStats.compute(event.getMethodName(), (key, stats) -> {
            if (stats == null) {
                stats = new SlowMethodStats();
            }
            stats.addSample(event.getDuration());
            return stats;
        });
        
        // 임계값 초과 시 알림
        SlowMethodStats stats = methodStats.get(event.getMethodName());
        if (stats.getCallCount() > 10 && stats.getAverageDuration() > 2000) {
            sendSlowMethodAlert(event.getMethodName(), stats);
        }
    }
}
```

#### 메모리 누수 감지
```java
@Component
public class MemoryLeakDetector {
    
    private final MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
    private final Map<String, Long> previousHeapUsage = new ConcurrentHashMap<>();
    
    @Scheduled(fixedDelay = 30000) // 30초마다 실행
    public void detectMemoryLeaks() {
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        long currentUsed = heapUsage.getUsed();
        long maxHeap = heapUsage.getMax();
        
        // 힙 사용률이 80% 이상이면 경고
        double usageRatio = (double) currentUsed / maxHeap;
        if (usageRatio > 0.8) {
            log.warn("High heap usage detected: {:.2f}% ({}/{})", 
                usageRatio * 100, formatBytes(currentUsed), formatBytes(maxHeap));
        }
        
        // 힙 사용량 증가 추세 분석
        String timestamp = LocalDateTime.now().truncatedTo(ChronoUnit.MINUTES).toString();
        Long previousUsage = previousHeapUsage.put(timestamp, currentUsed);
        
        if (previousUsage != null && currentUsed > previousUsage * 1.1) {
            log.warn("Potential memory leak detected: heap usage increased from {} to {} in last minute",
                formatBytes(previousUsage), formatBytes(currentUsed));
        }
    }
    
    @EventListener
    public void handleOutOfMemoryError(OutOfMemoryErrorEvent event) {
        // 힙 덤프 생성
        try {
            String dumpPath = "/tmp/heapdump-" + System.currentTimeMillis() + ".hprof";
            MBeanServer server = ManagementFactory.getPlatformMBeanServer();
            HotSpotDiagnosticMXBean mxBean = ManagementFactory.newPlatformMXBeanProxy(
                server, "com.sun.management:type=HotSpotDiagnostic", HotSpotDiagnosticMXBean.class);
            mxBean.dumpHeap(dumpPath, true);
            
            log.error("OutOfMemoryError occurred, heap dump saved to: {}", dumpPath);
        } catch (Exception e) {
            log.error("Failed to create heap dump", e);
        }
    }
}
```

---

## 3. JVM 튜닝과 가비지 컬렉션 최적화

### 3.1 JVM 메모리 구조 최적화

#### 힙 메모리 튜닝
```bash
# 기본 힙 설정
-Xms2g                    # 초기 힙 크기
-Xmx4g                    # 최대 힙 크기
-XX:NewRatio=3            # Old:Young = 3:1
-XX:SurvivorRatio=8       # Eden:Survivor = 8:1

# 대용량 힙을 위한 설정
-XX:+UseCompressedOops    # 객체 포인터 압축 (32GB 이하)
-XX:+UseCompressedClassPointers  # 클래스 포인터 압축

# 힙 덤프 설정
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/app/logs/heapdump.hprof
```

#### 메타스페이스 튜닝
```bash
# 메타스페이스 설정 (Java 8+)
-XX:MetaspaceSize=256m    # 초기 메타스페이스 크기
-XX:MaxMetaspaceSize=512m # 최대 메타스페이스 크기
-XX:CompressedClassSpaceSize=128m  # 압축된 클래스 공간 크기
```

### 3.2 가비지 컬렉션 최적화

#### G1GC 튜닝
```bash
# G1GC 기본 설정
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200     # 목표 일시정지 시간
-XX:G1HeapRegionSize=16m     # 힙 영역 크기
-XX:G1NewSizePercent=30      # Young Generation 최소 비율
-XX:G1MaxNewSizePercent=40   # Young Generation 최대 비율
-XX:G1MixedGCCountTarget=8   # Mixed GC 목표 사이클 수

# 대용량 힙을 위한 G1GC 설정
-XX:+UnlockExperimentalVMOptions
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:G1HeapRegionSize=32m
-XX:G1MixedGCLiveThresholdPercent=85
```

#### ZGC 설정 (Java 11+)
```bash
# ZGC 설정 (매우 낮은 지연시간 요구시)
-XX:+UnlockExperimentalVMOptions
-XX:+UseZGC
-XX:+UnlockDiagnosticVMOptions
-XX:+LogVMOutput
-XX:LogFile=/app/logs/zgc.log
```

#### GC 로깅과 분석
```bash
# 상세한 GC 로깅
-Xlog:gc*:gc.log:time,tags
-XX:+UseStringDeduplication  # 문자열 중복 제거

# GC 분석을 위한 추가 옵션
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintStringDeduplicationStatistics
```

### 3.3 JIT 컴파일러 최적화

```bash
# JIT 컴파일러 튜닝
-XX:+TieredCompilation           # 계층적 컴파일레이션
-XX:TieredStopAtLevel=4          # 최고 레벨까지 컴파일
-XX:CompileThreshold=10000       # 컴파일 임계값
-XX:+UseCodeCacheFlushing        # 코드 캐시 정리

# 대규모 애플리케이션을 위한 설정
-XX:ReservedCodeCacheSize=256m   # 코드 캐시 크기
-XX:InitialCodeCacheSize=64m     # 초기 코드 캐시 크기
```

### 3.4 JVM 모니터링

```java
@Component
public class JvmMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final MemoryMXBean memoryBean;
    private final List<GarbageCollectorMXBean> gcBeans;
    
    public JvmMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.memoryBean = ManagementFactory.getMemoryMXBean();
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        // JVM 메트릭 등록
        new JvmMemoryMetrics().bindTo(meterRegistry);
        new JvmGcMetrics().bindTo(meterRegistry);
        new JvmThreadMetrics().bindTo(meterRegistry);
        new ProcessorMetrics().bindTo(meterRegistry);
    }
    
    @Scheduled(fixedDelay = 10000)
    public void collectCustomMetrics() {
        // 힙 사용률
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        double heapUsageRatio = (double) heapUsage.getUsed() / heapUsage.getMax();
        
        Gauge.builder("jvm.heap.usage.ratio")
            .description("Heap usage ratio")
            .register(meterRegistry, () -> heapUsageRatio);
        
        // GC 효율성
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            long collections = gcBean.getCollectionCount();
            long time = gcBean.getCollectionTime();
            
            if (collections > 0) {
                double avgGcTime = (double) time / collections;
                
                Gauge.builder("jvm.gc.average.time")
                    .tag("gc", gcBean.getName())
                    .description("Average GC time")
                    .register(meterRegistry, () -> avgGcTime);
            }
        }
    }
}
```

---

## 4. 데이터베이스 쿼리 최적화

### 4.1 쿼리 성능 분석

#### 느린 쿼리 감지
```java
@Component
public class SlowQueryDetector {
    
    private static final long SLOW_QUERY_THRESHOLD_MS = 1000;
    
    @EventListener
    public void detectSlowQueries(DatabaseQueryEvent event) {
        if (event.getDuration() > SLOW_QUERY_THRESHOLD_MS) {
            log.warn("Slow query detected: {} took {}ms", 
                event.getQuery(), event.getDuration());
            
            // 쿼리 실행 계획 분석 요청
            analyzeQueryPlan(event.getQuery());
        }
    }
    
    private void analyzeQueryPlan(String query) {
        try {
            String explainQuery = "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) " + query;
            // 실행 계획 분석 로직
        } catch (Exception e) {
            log.error("Failed to analyze query plan", e);
        }
    }
}
```

#### 쿼리 캐싱 전략
```java
@Repository
public class OptimizedProductRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 쿼리 결과 캐싱
    @Cacheable(value = "query-results", key = "#query + ':' + #params.hashCode()")
    public List<Product> executeQuery(String query, Map<String, Object> params) {
        return jdbcTemplate.query(query, params, new ProductRowMapper());
    }
    
    // 배치 쿼리 최적화
    public void batchUpdateProducts(List<Product> products) {
        String sql = "UPDATE products SET name = ?, price = ?, updated_at = ? WHERE id = ?";
        
        List<Object[]> batchArgs = products.stream()
            .map(p -> new Object[]{p.getName(), p.getPrice(), Instant.now(), p.getId()})
            .collect(Collectors.toList());
        
        jdbcTemplate.batchUpdate(sql, batchArgs);
    }
    
    // 페이징 최적화 (커서 기반)
    public Page<Product> findProductsWithCursor(String cursor, int limit) {
        String sql = cursor == null ? 
            "SELECT * FROM products ORDER BY id LIMIT ?" :
            "SELECT * FROM products WHERE id > ? ORDER BY id LIMIT ?";
        
        Object[] params = cursor == null ? 
            new Object[]{limit} : 
            new Object[]{cursor, limit};
        
        List<Product> products = jdbcTemplate.query(sql, params, new ProductRowMapper());
        
        String nextCursor = products.isEmpty() ? null : 
            products.get(products.size() - 1).getId();
        
        return new CursorPage<>(products, nextCursor, limit);
    }
}
```

### 4.2 인덱스 최적화

#### 인덱스 전략
```sql
-- 복합 인덱스 최적화
CREATE INDEX idx_orders_user_status_created 
ON orders(user_id, status, created_at DESC) 
WHERE status IN ('PENDING', 'PROCESSING');

-- 부분 인덱스 (조건부 인덱스)
CREATE INDEX idx_products_active 
ON products(category_id, price) 
WHERE status = 'ACTIVE';

-- 표현식 인덱스
CREATE INDEX idx_users_email_lower 
ON users(LOWER(email));

-- GIN 인덱스 (전문 검색)
CREATE INDEX idx_products_search 
ON products USING GIN(to_tsvector('english', name || ' ' || description));
```

#### 인덱스 모니터링
```java
@Component
public class IndexMonitor {
    
    @Scheduled(fixedDelay = 3600000) // 1시간마다
    public void analyzeIndexUsage() {
        String sql = """
            SELECT schemaname, tablename, indexname, idx_tup_read, idx_tup_fetch,
                   pg_size_pretty(pg_relation_size(indexrelid)) as size
            FROM pg_stat_user_indexes
            WHERE idx_tup_read = 0 AND idx_tup_fetch = 0
            ORDER BY pg_relation_size(indexrelid) DESC
            """;
        
        List<UnusedIndex> unusedIndexes = jdbcTemplate.query(sql, 
            (rs, rowNum) -> UnusedIndex.builder()
                .schemaName(rs.getString("schemaname"))
                .tableName(rs.getString("tablename"))
                .indexName(rs.getString("indexname"))
                .size(rs.getString("size"))
                .build());
        
        if (!unusedIndexes.isEmpty()) {
            log.warn("Found {} unused indexes consuming space", unusedIndexes.size());
            unusedIndexes.forEach(index -> 
                log.warn("Unused index: {}.{} ({})", 
                    index.getTableName(), index.getIndexName(), index.getSize()));
        }
    }
}
```

### 4.3 연결 풀 최적화

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        HikariConfig config = new HikariConfig();
        
        // 연결 풀 크기 최적화
        config.setMaximumPoolSize(20);           // CPU 코어 수 * 2-4
        config.setMinimumIdle(5);                // 최소 유지 연결 수
        config.setConnectionTimeout(30000);      // 연결 대기 시간
        config.setIdleTimeout(600000);           // 유휴 연결 타임아웃
        config.setMaxLifetime(1800000);          // 연결 최대 수명
        config.setLeakDetectionThreshold(60000); // 연결 누수 감지
        
        // 성능 최적화 설정
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");
        config.addDataSourceProperty("cacheResultSetMetadata", "true");
        config.addDataSourceProperty("cacheServerConfiguration", "true");
        config.addDataSourceProperty("elideSetAutoCommits", "true");
        config.addDataSourceProperty("maintainTimeStats", "false");
        
        return config;
    }
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(hikariConfig());
    }
}
```

---

## 5. 네트워크 최적화와 CDN 활용

### 5.1 HTTP 최적화

#### HTTP/2 활용
```yaml
# application.yml
server:
  http2:
    enabled: true
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,text/css,text/javascript,application/javascript,application/json
    min-response-size: 1024
```

#### 응답 압축
```java
@Configuration
public class CompressionConfig {
    
    @Bean
    public FilterRegistrationBean<CompressionFilter> compressionFilter() {
        FilterRegistrationBean<CompressionFilter> registration = 
            new FilterRegistrationBean<>();
        
        CompressionFilter filter = new CompressionFilter();
        registration.setFilter(filter);
        registration.addUrlPatterns("/api/*");
        registration.setOrder(1);
        
        return registration;
    }
}

@Component
public class CompressionFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String acceptEncoding = httpRequest.getHeader("Accept-Encoding");
        
        if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
            GzipResponseWrapper wrappedResponse = 
                new GzipResponseWrapper(httpResponse);
            chain.doFilter(request, wrappedResponse);
            wrappedResponse.finishResponse();
        } else {
            chain.doFilter(request, response);
        }
    }
}
```

### 5.2 CDN 최적화

#### CloudFront 설정
```yaml
# CDN 캐시 정책
cache_behaviors:
  - path_pattern: "/api/*"
    target_origin_id: "api-origin"
    viewer_protocol_policy: "redirect-to-https"
    cache_policy_id: "dynamic-content"
    ttl:
      default: 0
      max: 300
    headers:
      - "Authorization"
      - "User-Agent"
  
  - path_pattern: "/static/*"
    target_origin_id: "static-origin"
    viewer_protocol_policy: "redirect-to-https"
    cache_policy_id: "static-content"
    ttl:
      default: 86400
      max: 31536000
    compress: true
```

#### 적응형 이미지 최적화
```java
@RestController
public class ImageOptimizationController {
    
    @GetMapping("/images/{imageId}")
    public ResponseEntity<Resource> getOptimizedImage(
            @PathVariable String imageId,
            @RequestParam(defaultValue = "webp") String format,
            @RequestParam(required = false) Integer width,
            @RequestParam(required = false) Integer height,
            @RequestHeader(value = "Accept", required = false) String accept) {
        
        // 브라우저 지원 형식 감지
        String targetFormat = determineBestFormat(accept, format);
        
        // 이미지 캐시 키 생성
        String cacheKey = String.format("image:%s:%s:%dx%d", 
            imageId, targetFormat, width, height);
        
        // 캐시에서 확인
        Resource cachedImage = imageCache.get(cacheKey);
        if (cachedImage != null) {
            return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(365, TimeUnit.DAYS))
                .body(cachedImage);
        }
        
        // 이미지 최적화 처리
        Resource optimizedImage = imageOptimizer.optimize(
            imageId, targetFormat, width, height);
        
        // 캐시에 저장
        imageCache.put(cacheKey, optimizedImage);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(365, TimeUnit.DAYS))
            .body(optimizedImage);
    }
    
    private String determineBestFormat(String accept, String requestedFormat) {
        if (accept != null) {
            if (accept.contains("image/avif")) return "avif";
            if (accept.contains("image/webp")) return "webp";
        }
        return requestedFormat;
    }
}
```

### 5.3 연결 최적화

#### HTTP 클라이언트 최적화
```java
@Configuration
public class HttpClientConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        CloseableHttpClient httpClient = HttpClients.custom()
            .setMaxConnTotal(200)                    // 전체 최대 연결 수
            .setMaxConnPerRoute(50)                  // 라우트당 최대 연결 수
            .setConnectionTimeToLive(30, TimeUnit.SECONDS)
            .setKeepAliveStrategy((response, context) -> 20 * 1000) // 20초
            .setRetryHandler(new DefaultHttpRequestRetryHandler(3, true))
            .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectTimeout(5000)             // 연결 타임아웃
                .setSocketTimeout(10000)             // 소켓 타임아웃
                .setConnectionRequestTimeout(3000)   // 연결 요청 타임아웃
                .build())
            .build();
        
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory(httpClient);
        
        RestTemplate restTemplate = new RestTemplate(factory);
        restTemplate.setInterceptors(Arrays.asList(
            new LoggingClientHttpRequestInterceptor(),
            new RetryClientHttpRequestInterceptor()
        ));
        
        return restTemplate;
    }
}
```

---

## 6. 캐시 최적화 전략

### 6.1 지능형 캐시 전략

#### 적응형 TTL
```java
@Service
public class AdaptiveCacheService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final Map<String, CacheStats> cacheStats = new ConcurrentHashMap<>();
    
    public void put(String key, Object value, String category) {
        Duration ttl = calculateOptimalTTL(key, category);
        redisTemplate.opsForValue().set(key, value, ttl);
        
        // 캐시 통계 업데이트
        updateCacheStats(key, category, ttl);
    }
    
    private Duration calculateOptimalTTL(String key, String category) {
        CacheStats stats = cacheStats.get(category);
        if (stats == null) {
            return Duration.ofMinutes(10); // 기본 TTL
        }
        
        // 히트율과 업데이트 빈도를 고려한 TTL 계산
        double hitRate = stats.getHitRate();
        long avgUpdateInterval = stats.getAverageUpdateInterval();
        
        if (hitRate > 0.8 && avgUpdateInterval > Duration.ofHours(1).toMillis()) {
            return Duration.ofHours(2); // 높은 히트율, 낮은 업데이트 빈도
        } else if (hitRate > 0.5) {
            return Duration.ofMinutes(30);
        } else {
            return Duration.ofMinutes(5); // 낮은 히트율
        }
    }
}
```

#### 계층적 캐시 무효화
```java
@Component
public class HierarchicalCacheInvalidator {
    
    private final Map<String, Set<String>> cacheHierarchy = Map.of(
        "user", Set.of("user:profile", "user:preferences", "user:orders"),
        "product", Set.of("product:details", "product:reviews", "product:recommendations"),
        "order", Set.of("order:details", "order:items", "order:history")
    );
    
    @EventListener
    public void handleEntityUpdate(EntityUpdateEvent event) {
        String entityType = event.getEntityType();
        String entityId = event.getEntityId();
        
        // 직접적인 캐시 무효화
        String directKey = entityType + ":" + entityId;
        cacheManager.evict(directKey);
        
        // 계층적 캐시 무효화
        Set<String> relatedCaches = cacheHierarchy.get(entityType);
        if (relatedCaches != null) {
            for (String cachePrefix : relatedCaches) {
                Set<String> keysToEvict = redisTemplate.keys(cachePrefix + ":" + entityId + "*");
                if (!keysToEvict.isEmpty()) {
                    redisTemplate.delete(keysToEvict);
                    log.debug("Evicted {} keys with prefix: {}", keysToEvict.size(), cachePrefix);
                }
            }
        }
    }
}
```

### 6.2 캐시 성능 모니터링

```java
@Component
public class CachePerformanceMonitor {
    
    @EventListener
    public void monitorCacheHit(CacheHitEvent event) {
        Timer.Sample.start(meterRegistry)
            .stop("cache.access.time", 
                "cache", event.getCacheName(),
                "result", "hit");
        
        Counter.builder("cache.requests.total")
            .tag("cache", event.getCacheName())
            .tag("result", "hit")
            .register(meterRegistry)
            .increment();
    }
    
    @EventListener  
    public void monitorCacheMiss(CacheMissEvent event) {
        Timer.Sample.start(meterRegistry)
            .stop("cache.access.time",
                "cache", event.getCacheName(),
                "result", "miss");
        
        Counter.builder("cache.requests.total")
            .tag("cache", event.getCacheName())
            .tag("result", "miss")
            .register(meterRegistry)
            .increment();
    }
    
    @Scheduled(fixedDelay = 60000)
    public void reportCacheStatistics() {
        for (String cacheName : cacheManager.getCacheNames()) {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache instanceof RedisCacheImpl) {
                RedisCacheImpl redisCache = (RedisCacheImpl) cache;
                CacheStatistics stats = redisCache.getStatistics();
                
                Gauge.builder("cache.hit.ratio")
                    .tag("cache", cacheName)
                    .register(meterRegistry, () -> stats.getHitRatio());
                
                Gauge.builder("cache.size")
                    .tag("cache", cacheName)
                    .register(meterRegistry, () -> stats.getSize());
            }
        }
    }
}
```

---

## 7. 실전 성능 최적화 사례

### 7.1 대용량 배치 처리 최적화

```java
@Component
public class OptimizedBatchProcessor {
    
    private static final int BATCH_SIZE = 1000;
    private static final int THREAD_COUNT = Runtime.getRuntime().availableProcessors();
    
    @Autowired
    private TaskExecutor taskExecutor;
    
    public CompletableFuture<BatchResult> processBatch(List<DataItem> items) {
        // 청크 단위로 분할
        List<List<DataItem>> chunks = Lists.partition(items, BATCH_SIZE);
        
        // 병렬 처리
        List<CompletableFuture<Void>> futures = chunks.stream()
            .map(chunk -> CompletableFuture.runAsync(
                () -> processChunk(chunk), taskExecutor))
            .collect(Collectors.toList());
        
        // 모든 청크 처리 완료 대기
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> BatchResult.success(items.size()));
    }
    
    private void processChunk(List<DataItem> chunk) {
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            
            try (PreparedStatement stmt = conn.prepareStatement(
                "INSERT INTO processed_data (id, data, processed_at) VALUES (?, ?, ?)")) {
                
                for (DataItem item : chunk) {
                    stmt.setString(1, item.getId());
                    stmt.setString(2, item.getData());
                    stmt.setTimestamp(3, Timestamp.from(Instant.now()));
                    stmt.addBatch();
                }
                
                stmt.executeBatch();
                conn.commit();
                
            } catch (SQLException e) {
                conn.rollback();
                throw new BatchProcessingException("Failed to process chunk", e);
            }
        } catch (SQLException e) {
            throw new BatchProcessingException("Failed to get connection", e);
        }
    }
}
```

### 7.2 API 응답 시간 최적화

```java
@RestController
public class OptimizedApiController {
    
    @GetMapping("/products/{categoryId}")
    @Timed(name = "api.products.get", description = "Get products by category")
    public CompletableFuture<ResponseEntity<ProductListResponse>> getProducts(
            @PathVariable String categoryId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        return CompletableFuture.supplyAsync(() -> {
            // 병렬로 데이터 조회
            CompletableFuture<List<Product>> productsFuture = 
                CompletableFuture.supplyAsync(() -> 
                    productService.findByCategory(categoryId, page, size));
            
            CompletableFuture<Long> countFuture = 
                CompletableFuture.supplyAsync(() -> 
                    productService.countByCategory(categoryId));
            
            CompletableFuture<List<Category>> categoriesFuture = 
                CompletableFuture.supplyAsync(() -> 
                    categoryService.findSubCategories(categoryId));
            
            // 모든 비동기 작업 완료 대기
            CompletableFuture.allOf(productsFuture, countFuture, categoriesFuture)
                .join();
            
            try {
                List<Product> products = productsFuture.get();
                Long totalCount = countFuture.get();
                List<Category> subCategories = categoriesFuture.get();
                
                ProductListResponse response = ProductListResponse.builder()
                    .products(products)
                    .totalCount(totalCount)
                    .subCategories(subCategories)
                    .page(page)
                    .size(size)
                    .build();
                
                return ResponseEntity.ok()
                    .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES))
                    .body(response);
                    
            } catch (InterruptedException | ExecutionException e) {
                Thread.currentThread().interrupt();
                throw new ProductServiceException("Failed to fetch products", e);
            }
        }, asyncExecutor);
    }
}
```

### 7.3 메모리 사용량 최적화

```java
@Service
public class MemoryOptimizedService {
    
    // 객체 풀을 사용한 메모리 최적화
    private final ObjectPool<StringBuilder> stringBuilderPool = 
        new GenericObjectPool<>(new BasePooledObjectFactory<StringBuilder>() {
            @Override
            public StringBuilder create() {
                return new StringBuilder(1024);
            }
            
            @Override
            public PooledObject<StringBuilder> wrap(StringBuilder obj) {
                obj.setLength(0); // 재사용 시 초기화
                return new DefaultPooledObject<>(obj);
            }
        });
    
    public String processLargeText(List<String> textParts) {
        StringBuilder sb = null;
        try {
            sb = stringBuilderPool.borrowObject();
            
            for (String part : textParts) {
                sb.append(part).append("\n");
            }
            
            return sb.toString();
            
        } catch (Exception e) {
            throw new TextProcessingException("Failed to process text", e);
        } finally {
            if (sb != null) {
                try {
                    stringBuilderPool.returnObject(sb);
                } catch (Exception e) {
                    log.warn("Failed to return StringBuilder to pool", e);
                }
            }
        }
    }
    
    // 스트림을 사용한 메모리 효율적 처리
    public void processLargeDataset(String filePath) {
        try (Stream<String> lines = Files.lines(Paths.get(filePath))) {
            lines.parallel()
                .filter(line -> !line.trim().isEmpty())
                .map(this::transformLine)
                .collect(Collectors.groupingByConcurrent(
                    this::getCategory,
                    Collectors.counting()))
                .forEach(this::processCategory);
        } catch (IOException e) {
            throw new DataProcessingException("Failed to process file", e);
        }
    }
    
    // WeakReference를 사용한 캐시
    private final Map<String, WeakReference<ExpensiveObject>> weakCache = 
        new ConcurrentHashMap<>();
    
    public ExpensiveObject getExpensiveObject(String key) {
        WeakReference<ExpensiveObject> ref = weakCache.get(key);
        ExpensiveObject obj = (ref != null) ? ref.get() : null;
        
        if (obj == null) {
            obj = createExpensiveObject(key);
            weakCache.put(key, new WeakReference<>(obj));
        }
        
        return obj;
    }
}
```

### 7.4 I/O 성능 최적화

```java
@Component
public class OptimizedFileProcessor {
    
    // NIO를 사용한 대용량 파일 처리
    public void processLargeFile(Path filePath, Consumer<String> lineProcessor) {
        try (FileChannel channel = FileChannel.open(filePath, StandardOpenOption.READ)) {
            ByteBuffer buffer = ByteBuffer.allocateDirect(8192);
            StringBuilder lineBuilder = new StringBuilder();
            
            while (channel.read(buffer) > 0) {
                buffer.flip();
                
                while (buffer.hasRemaining()) {
                    char c = (char) buffer.get();
                    
                    if (c == '\n') {
                        if (lineBuilder.length() > 0) {
                            lineProcessor.accept(lineBuilder.toString());
                            lineBuilder.setLength(0);
                        }
                    } else {
                        lineBuilder.append(c);
                    }
                }
                
                buffer.clear();
            }
            
            // 마지막 라인 처리
            if (lineBuilder.length() > 0) {
                lineProcessor.accept(lineBuilder.toString());
            }
            
        } catch (IOException e) {
            throw new FileProcessingException("Failed to process file", e);
        }
    }
    
    // 비동기 파일 업로드
    @Async
    public CompletableFuture<String> uploadFileAsync(MultipartFile file) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String fileName = generateUniqueFileName(file.getOriginalFilename());
                Path targetPath = Paths.get(uploadDirectory, fileName);
                
                // Files.copy를 사용한 효율적 복사
                try (InputStream inputStream = file.getInputStream()) {
                    Files.copy(inputStream, targetPath, StandardCopyOption.REPLACE_EXISTING);
                }
                
                // 업로드 후 처리 (썸네일 생성, 바이러스 검사 등)
                postProcessFile(targetPath);
                
                return fileName;
                
            } catch (IOException e) {
                throw new FileUploadException("Failed to upload file", e);
            }
        });
    }
}
```

이 문서는 대규모 시스템의 성능 최적화를 위한 포괄적인 가이드를 제공합니다. 각 최적화 기법은 실제 프로덕션 환경에서 검증된 방법들이며, 시스템의 특성에 맞게 적절히 조합하여 사용해야 합니다.
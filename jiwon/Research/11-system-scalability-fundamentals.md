# 1-1.시스템 확장성 기초: 수직/수평 확장과 무상태 설계

## 개요

시스템 확장성은 고트래픽 애플리케이션 아키텍처의 핵심입니다. 이 문서는 모든 시니어 개발자가 숙달해야 할 기본 개념을 다룹니다: 수직 확장과 수평 확장 간의 트레이드오프, 무상태 설계의 중요성, 그리고 연쇄 장애를 방지하는 장애 격리 패턴을 포함합니다.

## 핵심 확장성 개념

### 수직 확장 vs 수평 확장

**수직 확장 (Scale Up)**
- **정의**: 기존 머신에 더 많은 성능 추가 (CPU, RAM, 스토리지)
- **구현**: 서버 하드웨어 업그레이드, 클라우드에서 인스턴스 크기 증가
- **장점**:
  - 간단한 애플리케이션 아키텍처 - 분산 시스템 복잡성 없음
  - 데이터 분산이나 일관성 문제 없음
  - 구현과 유지보수가 쉬움
  - 단일 스레드 애플리케이션에 적합

- **단점**:
  - 하드웨어 제한 - 단일 머신 용량에 한계 존재
  - 단일 장애점 - 전체 시스템이 하나의 머신에 의존
  - 지수적 비용 증가 - 고성능 하드웨어는 비용 대비 효율이 낮음
  - 업그레이드 시 다운타임 필요

**수평 확장 (Scale Out)**
- **정의**: 증가된 부하를 처리하기 위해 더 많은 머신 추가
- **구현**: 여러 서버에 걸친 로드 밸런싱, 마이크로서비스 아키텍처
- **장점**:
  - 용량에 이론적 제한 없음
  - 중복성을 통한 장애 내성
  - 선형 비용 확장
  - 대규모 트래픽 급증 처리 가능

- **단점**:
  - 복잡한 애플리케이션 아키텍처
  - 데이터 일관성 도전
  - 네트워크 지연과 파티셔닝 문제
  - 관리할 구성 요소 증가

### 무상태 vs 상태 유지 설계

**무상태 설계 구현**
```java
@RestController
public class ProductController {
    
    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(
            @PathVariable String id,
            @RequestHeader("Authorization") String token) {
        
        // JWT 토큰에서 사용자 컨텍스트 추출 (서버 측 세션 없음)
        UserContext user = jwtService.parseToken(token);
        Product product = productService.getProduct(id, user);
        
        return ResponseEntity.ok(product);
    }
}
```

**주요 이점**:
- 각 요청이 필요한 모든 정보를 포함
- 어떤 서버든 모든 요청을 처리 가능
- 수평 확장에 완벽히 적합
- 원활한 로드 밸런싱 가능

### 장애 격리 패턴

**벌크헤드 패턴**
```java
@Component
public class PaymentService {
    
    // 서로 다른 작업을 위한 분리된 스레드 풀
    private final Executor criticalPaymentExecutor = 
        Executors.newFixedThreadPool(10); // 높은 우선순위
    
    private final Executor reportingExecutor = 
        Executors.newFixedThreadPool(2);  // 낮은 우선순위
    
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // 중요한 결제 처리
            return paymentProcessor.process(request);
        }, criticalPaymentExecutor);
    }
    
    public CompletableFuture<Report> generateReport(ReportRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // 중요하지 않은 리포팅
            return reportGenerator.generate(request);
        }, reportingExecutor);
    }
}
```

**서킷 브레이커 패턴**
```java
@Service
public class ExternalApiService {
    
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("externalApi");
    
    public Optional<ExternalData> fetchData(String id) {
        Supplier<ExternalData> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> externalApiClient.getData(id));
        
        try {
            return Optional.of(decoratedSupplier.get());
        } catch (Exception e) {
            log.warn("외부 API에 대한 서킷 브레이커가 열렸습니다", e);
            return Optional.empty(); // 우아한 성능 저하
        }
    }
}
```

**타임아웃과 재시도 메커니즘**
```java
@Service
public class DatabaseService {
    
    @Retryable(
        value = {TransientDataAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public User getUserById(String id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    @Recover
    public User recover(TransientDataAccessException ex, String id) {
        log.error("재시도 후에도 사용자 조회 실패: {}", id, ex);
        throw new ServiceUnavailableException("사용자 서비스가 일시적으로 사용 불가능합니다");
    }
}
```

## 실제 구현

### 무상태 REST API 구축

**JWT 인증 구현**
```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @Autowired
    private AuthService authService;
    
    @Autowired
    private JwtService jwtService;
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        try {
            // 사용자 자격증명 인증
            User user = authService.authenticate(request.getUsername(), request.getPassword());
            
            // 사용자 컨텍스트로 JWT 토큰 생성
            String token = jwtService.generateToken(user);
            
            return ResponseEntity.ok(new AuthResponse(token, user.getId(), user.getUsername()));
            
        } catch (AuthenticationException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(new AuthResponse(null, null, "잘못된 자격증명"));
        }
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refreshToken(@RequestHeader("Authorization") String token) {
        try {
            String refreshedToken = jwtService.refreshToken(token);
            User user = jwtService.getUserFromToken(refreshedToken);
            
            return ResponseEntity.ok(new AuthResponse(refreshedToken, user.getId(), user.getUsername()));
            
        } catch (JwtException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(new AuthResponse(null, null, "잘못된 토큰"));
        }
    }
}
```

**헬스 체크 엔드포인트**
```java
@RestController
@RequestMapping("/health")
public class HealthController {
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @GetMapping
    public ResponseEntity<HealthStatus> health() {
        HealthStatus status = new HealthStatus();
        status.setTimestamp(Instant.now());
        status.setStatus("UP");
        
        // 데이터베이스 연결 확인
        try {
            dataSource.getConnection().close();
            status.addCheck("database", "UP");
        } catch (SQLException e) {
            status.addCheck("database", "DOWN");
            status.setStatus("DOWN");
        }
        
        // Redis 연결 확인
        try {
            redisTemplate.opsForValue().set("health-check", "ok", Duration.ofSeconds(10));
            status.addCheck("redis", "UP");
        } catch (Exception e) {
            status.addCheck("redis", "DOWN");
            status.setStatus("DOWN");
        }
        
        HttpStatus httpStatus = "UP".equals(status.getStatus()) ? 
            HttpStatus.OK : HttpStatus.SERVICE_UNAVAILABLE;
            
        return ResponseEntity.status(httpStatus).body(status);
    }
}
```

## 부하 테스트와 성능 검증

### Apache Bench 테스트 전략

**기본 처리량 테스트**
```bash
# 기준 성능 테스트
ab -n 1000 -c 10 http://localhost:8080/api/users/123

# 인증 헤더와 함께 테스트
ab -n 1000 -c 10 -H "Authorization: Bearer <jwt-token>" http://localhost:8080/api/users/123

# keep-alive 연결과 함께 테스트 (더 현실적)
ab -n 1000 -c 10 -k http://localhost:8080/api/users/123

# POST 요청 테스트
ab -n 1000 -c 10 -p user.json -T application/json http://localhost:8080/api/users
```

**JMeter 고급 시나리오**
- 점진적 부하 증가 테스트 (ramp-up testing)
- 스파이크 테스트 (급작스러운 부하 급증)
- 지구력 테스트 (지속적인 부하)
- 사용자 여정 시뮬레이션 (현실적인 사용 패턴)

### 모니터링할 주요 지표

**애플리케이션 지표**
- 응답 시간 백분위수 (P50, P95, P99)
- 처리량 (초당 요청 수)
- 오류율 (4xx 및 5xx 응답)
- 지원 가능한 동시 사용자 수

**시스템 지표**
- CPU 사용률
- 메모리 사용량
- 네트워크 I/O
- 데이터베이스 연결 풀 사용량

**비즈니스 지표**
- 사용자 세션 지속 시간
- API 엔드포인트 사용 패턴
- 피크 트래픽 시간
- 요청의 지리적 분포

### 확장성 테스트 시나리오

1. **기준 성능**: 정상 부하 하에서 단일 인스턴스
2. **스트레스 테스트**: 단일 인스턴스의 한계점
3. **수평 확장**: 로드 밸런서를 통한 다중 인스턴스
4. **장애 극복 테스트**: 인스턴스 장애와 복구
5. **자동 확장**: 부하에 따른 동적 인스턴스 확장

## 시니어 개발자를 위한 핵심 포인트

1. **무상태 설계는 필수**: 수평 확장을 기대하는 시스템에서는 무상태 설계가 필수입니다. JWT 기반 인증과 외부 세션 저장소에 대한 투자는 즉시 효과를 발휘합니다.

2. **첫날부터 장애 격리**: 서킷 브레이커, 벌크헤드, 적절한 타임아웃을 처음부터 구현하는 것이 나중에 개조하는 것보다 훨씬 쉽습니다.

3. **모든 것을 모니터링**: 기술적 지표와 비즈니스 지표에 대한 포괄적인 모니터링은 확장 결정에 필요한 데이터를 제공합니다.

4. **부하 테스트는 지속적**: 성능 테스트는 출시 전 일회성 활동이 아니라 CI/CD 파이프라인의 일부여야 합니다.

5. **용량 계획으로 사고**: 시스템의 한계와 성장 패턴을 이해하면 사후 대응이 아닌 사전 예방적 확장이 가능합니다.

이러한 확장성 개념의 기초는 후속 단계에서 다룰 더 고급의 분산 시스템 패턴을 위한 준비가 됩니다.
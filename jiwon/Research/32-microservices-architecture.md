# 3.2 마이크로서비스 아키텍처

## Overview
마이크로서비스 아키텍처는 애플리케이션을 독립적으로 배포 가능한 작은 서비스들로 분해하는 설계 방법입니다. **도메인 주도 설계(Domain-Driven Design)**를 기반으로 서비스를 분리하고, **API Gateway**와 **서비스 디스커버리**를 통해 서비스 간 통신을 관리합니다.

## 마이크로서비스 분해 전략

### 도메인 기반 분해
```
E-commerce System Domain Decomposition:

┌─ User Management ─┐  ┌─ Product Catalog ─┐  ┌─ Order Processing ─┐
│ - User Registration│  │ - Product Info    │  │ - Order Creation   │
│ - Authentication   │  │ - Inventory       │  │ - Payment          │
│ - Profile Management│  │ - Categories      │  │ - Shipping         │
└────────────────────┘  └───────────────────┘  └────────────────────┘

┌─ Notification ────┐  ┌─ Analytics ──────┐  ┌─ Customer Support ─┐
│ - Email Service   │  │ - Usage Metrics  │  │ - Ticket System    │
│ - SMS Service     │  │ - Reporting      │  │ - Chat Support     │
│ - Push Notifications│  │ - Recommendations│  │ - FAQ Management   │
└───────────────────┘  └──────────────────┘  └────────────────────┘
```

### 데이터 분해 패턴
```java
// Before: Monolithic Database
@Entity
public class User {
    private Long id;
    private String email;
    private String profile;
    private List<Order> orders;
    private List<PaymentMethod> paymentMethods;
}

// After: Domain-specific Entities
// User Service
@Entity
public class User {
    private Long id;
    private String email;
    private String hashedPassword;
    private UserStatus status;
}

// Profile Service
@Entity
public class UserProfile {
    private Long userId;
    private String firstName;
    private String lastName;
    private String phoneNumber;
    private Address address;
}

// Order Service
@Entity
public class Order {
    private String orderId;
    private Long userId; // Reference only
    private OrderStatus status;
    private List<OrderItem> items;
}
```

## Spring Cloud Gateway 구현

### Gateway 설정
```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r.path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .circuitBreaker(config -> config
                        .setName("user-service-cb")
                        .setFallbackUri("forward:/fallback/user"))
                    .retry(retryConfig -> retryConfig.setRetries(3))
                )
                .uri("lb://user-service"))
            
            .route("product-service", r -> r.path("/api/products/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                )
                .uri("lb://product-service"))
                
            .route("order-service", r -> r.path("/api/orders/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestHeader("X-Gateway-Request", "true")
                )
                .uri("lb://order-service"))
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20, 1); // replenishRate, burstCapacity, requestedTokens
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> exchange.getRequest().getHeaders()
            .getFirst("X-User-Id")
            .map(Mono::just)
            .orElse(Mono.just("anonymous"));
    }
}
```

### 인증 및 인가 필터
```java
@Component
public class AuthenticationFilter implements GlobalFilter, Ordered {
    
    private final JwtTokenProvider jwtTokenProvider;
    private final ReactiveRedisTemplate<String, String> redisTemplate;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // 인증이 필요없는 경로 체크
        if (isPublicPath(request.getPath().toString())) {
            return chain.filter(exchange);
        }
        
        String token = extractToken(request);
        if (token == null) {
            return unauthorized(exchange);
        }
        
        return validateToken(token)
            .flatMap(claims -> {
                // 요청에 사용자 정보 추가
                ServerHttpRequest modifiedRequest = request.mutate()
                    .header("X-User-Id", claims.getUserId())
                    .header("X-User-Role", claims.getRole())
                    .build();
                
                return chain.filter(exchange.mutate().request(modifiedRequest).build());
            })
            .onErrorResume(throwable -> unauthorized(exchange));
    }
    
    private Mono<JwtClaims> validateToken(String token) {
        try {
            JwtClaims claims = jwtTokenProvider.parseToken(token);
            
            // Redis에서 토큰 유효성 확인
            return redisTemplate.hasKey("token:" + token)
                .flatMap(exists -> exists ? Mono.just(claims) : Mono.error(new InvalidTokenException()));
                
        } catch (Exception e) {
            return Mono.error(new InvalidTokenException());
        }
    }
    
    @Override
    public int getOrder() {
        return -100; // 높은 우선순위
    }
}
```

## 서비스 디스커버리

### Eureka 서버 설정
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
# eureka-server application.yml
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

spring:
  application:
    name: eureka-server
```

### 서비스 클라이언트 설정
```yaml
# user-service application.yml
spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
    fetch-registry: true
    register-with-eureka: true
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
    metadata-map:
      version: 1.0
      region: us-west-2
```

### 서비스 간 통신
```java
@Service
public class OrderService {
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @Autowired
    private ProductServiceClient productServiceClient;
    
    @Autowired
    private PaymentServiceClient paymentServiceClient;
    
    public Order createOrder(CreateOrderRequest request) {
        // 사용자 정보 조회
        User user = userServiceClient.getUser(request.getUserId());
        if (user == null) {
            throw new UserNotFoundException(request.getUserId());
        }
        
        // 상품 정보 및 재고 확인
        List<Product> products = productServiceClient.getProducts(request.getProductIds());
        validateProductAvailability(products, request.getItems());
        
        // 주문 생성
        Order order = Order.builder()
            .orderId(generateOrderId())
            .userId(request.getUserId())
            .items(request.getItems())
            .totalAmount(calculateTotal(products, request.getItems()))
            .status(OrderStatus.CREATED)
            .build();
        
        orderRepository.save(order);
        
        // 비동기 처리를 위한 이벤트 발행
        eventPublisher.publishOrderCreated(new OrderCreatedEvent(order));
        
        return order;
    }
}
```

### Feign 클라이언트 구현
```java
@FeignClient(name = "user-service", fallback = UserServiceClientFallback.class)
public interface UserServiceClient {
    
    @GetMapping("/users/{userId}")
    User getUser(@PathVariable("userId") String userId);
    
    @GetMapping("/users/{userId}/profile")
    UserProfile getUserProfile(@PathVariable("userId") String userId);
    
    @PostMapping("/users")
    User createUser(@RequestBody CreateUserRequest request);
}

@Component
public class UserServiceClientFallback implements UserServiceClient {
    
    @Override
    public User getUser(String userId) {
        log.warn("Fallback: Unable to get user {}", userId);
        return User.builder()
            .id(userId)
            .email("unknown@example.com")
            .status(UserStatus.UNKNOWN)
            .build();
    }
    
    @Override
    public UserProfile getUserProfile(String userId) {
        log.warn("Fallback: Unable to get user profile {}", userId);
        return null;
    }
    
    @Override
    public User createUser(CreateUserRequest request) {
        log.error("Fallback: Cannot create user during service unavailability");
        throw new ServiceUnavailableException("User service is currently unavailable");
    }
}
```

## 분산 데이터 관리

### Database per Service 패턴
```java
// User Service Database
@Configuration
public class UserDatabaseConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.user")
    public DataSource userDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @Primary
    public JdbcTemplate userJdbcTemplate(@Qualifier("userDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}

// Order Service Database
@Configuration
public class OrderDatabaseConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.order")
    public DataSource orderDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public JdbcTemplate orderJdbcTemplate(@Qualifier("orderDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 이벤트 기반 데이터 동기화
```java
@Component
public class UserEventHandler {
    
    private final OrderUserProjectionRepository projectionRepository;
    
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        // Order Service에서 필요한 사용자 정보만 저장
        OrderUserProjection projection = OrderUserProjection.builder()
            .userId(event.getUserId())
            .email(event.getEmail())
            .status(UserStatus.ACTIVE)
            .createdAt(event.getTimestamp())
            .build();
        
        projectionRepository.save(projection);
        
        log.info("User projection created for order service: {}", event.getUserId());
    }
    
    @EventListener
    public void handleUserUpdated(UserUpdatedEvent event) {
        OrderUserProjection projection = projectionRepository.findByUserId(event.getUserId());
        if (projection != null) {
            projection.setEmail(event.getEmail());
            projection.setUpdatedAt(event.getTimestamp());
            projectionRepository.save(projection);
        }
    }
    
    @EventListener
    public void handleUserDeleted(UserDeletedEvent event) {
        projectionRepository.deleteByUserId(event.getUserId());
        log.info("User projection deleted from order service: {}", event.getUserId());
    }
}
```

## 분산 설정 관리

### Spring Cloud Config
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# config-server application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/microservices-config
          clone-on-start: true
          default-label: main
        health:
          repositories:
            microservices-config:
              label: main
              name: microservices-config
              profiles: development,production
```

### 동적 설정 업데이트
```java
@RestController
@RefreshScope
public class ConfigurableController {
    
    @Value("${feature.new-algorithm.enabled:false}")
    private boolean newAlgorithmEnabled;
    
    @Value("${api.rate-limit.requests-per-minute:100}")
    private int rateLimitPerMinute;
    
    @GetMapping("/config/status")
    public Map<String, Object> getConfigStatus() {
        Map<String, Object> config = new HashMap<>();
        config.put("newAlgorithmEnabled", newAlgorithmEnabled);
        config.put("rateLimitPerMinute", rateLimitPerMinute);
        return config;
    }
    
    @PostMapping("/actuator/refresh")
    public ResponseEntity<String> refresh() {
        // 설정 새로고침은 Spring Cloud Bus를 통해 자동으로 처리됨
        return ResponseEntity.ok("Configuration refreshed");
    }
}
```

## 서비스 메시 준비

### 서비스 간 통신 추상화
```java
@Component
public class ServiceCommunicationManager {
    
    private final List<ServiceClient> serviceClients;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final RetryRegistry retryRegistry;
    
    public <T> T callService(String serviceName, String operation, 
                            Function<ServiceClient, T> serviceCall) {
        
        ServiceClient client = getServiceClient(serviceName);
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(serviceName);
        Retry retry = retryRegistry.retry(serviceName);
        
        Supplier<T> decoratedSupplier = Decorators
            .ofSupplier(() -> serviceCall.apply(client))
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .decorate();
        
        return decoratedSupplier.get();
    }
    
    private ServiceClient getServiceClient(String serviceName) {
        return serviceClients.stream()
            .filter(client -> client.getServiceName().equals(serviceName))
            .findFirst()
            .orElseThrow(() -> new ServiceNotFoundException(serviceName));
    }
}
```

### 분산 추적 준비
```java
@Configuration
public class TracingConfiguration {
    
    @Bean
    public Sender sender() {
        return OkHttpSender.create("http://zipkin:9411/api/v2/spans");
    }
    
    @Bean
    public AsyncReporter<Span> spanReporter() {
        return AsyncReporter.create(sender());
    }
    
    @Bean
    public Tracing tracing() {
        return Tracing.newBuilder()
            .localServiceName("gateway-service")
            .spanReporter(spanReporter())
            .sampler(Sampler.create(1.0f)) // 100% 샘플링 (프로덕션에서는 조정 필요)
            .build();
    }
    
    @Bean
    public HttpTracing httpTracing(Tracing tracing) {
        return HttpTracing.create(tracing);
    }
}
```

## 모니터링과 로깅

### 중앙집중식 로깅
```java
@Component
public class StructuredLoggingAspect {
    
    private final ObjectMapper objectMapper;
    
    @Around("@annotation(Loggable)")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        StructuredLogEntry logEntry = StructuredLogEntry.builder()
            .timestamp(Instant.now())
            .service("order-service")
            .method(className + "." + methodName)
            .traceId(getTraceId())
            .spanId(getSpanId())
            .build();
        
        Instant start = Instant.now();
        Object result = null;
        Exception exception = null;
        
        try {
            result = joinPoint.proceed();
            logEntry.setStatus("SUCCESS");
            return result;
            
        } catch (Exception e) {
            exception = e;
            logEntry.setStatus("ERROR");
            logEntry.setErrorMessage(e.getMessage());
            throw e;
            
        } finally {
            Duration duration = Duration.between(start, Instant.now());
            logEntry.setDurationMs(duration.toMillis());
            
            String logMessage = objectMapper.writeValueAsString(logEntry);
            if (exception != null) {
                log.error(logMessage);
            } else {
                log.info(logMessage);
            }
        }
    }
}
```

### 헬스 체크 구현
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    private final DatabaseHealthChecker databaseChecker;
    private final ExternalServiceHealthChecker externalServiceChecker;
    
    @Override
    public Health health() {
        Health.Builder builder = Health.up();
        
        // 데이터베이스 상태 확인
        try {
            databaseChecker.checkHealth();
            builder.withDetail("database", "UP");
        } catch (Exception e) {
            builder.down()
                .withDetail("database", "DOWN")
                .withDetail("database-error", e.getMessage());
        }
        
        // 외부 서비스 상태 확인
        Map<String, String> externalServices = externalServiceChecker.checkAllServices();
        builder.withDetail("external-services", externalServices);
        
        // 커스텀 메트릭 추가
        builder.withDetail("active-connections", getActiveConnections());
        builder.withDetail("memory-usage", getMemoryUsage());
        
        return builder.build();
    }
}
```

## Best Practices

### 서비스 분해 원칙
- **단일 책임**: 각 서비스는 하나의 비즈니스 능력에 집중
- **자율성**: 서비스는 독립적으로 개발, 배포, 확장 가능
- **데이터 소유권**: 각 서비스가 자신의 데이터를 소유

### API 설계
- **RESTful**: 일관된 REST API 설계
- **버전 관리**: API 버전 관리 전략
- **문서화**: OpenAPI/Swagger를 통한 API 문서 자동 생성

### 장애 처리
- **Circuit Breaker**: 연쇄 장애 방지
- **Timeout**: 적절한 타임아웃 설정
- **Fallback**: 장애 시 대체 응답 제공

## Benefits and Challenges

### Benefits
- **독립 배포**: 서비스별 독립적인 배포 사이클
- **기술 다양성**: 서비스별 최적 기술 스택 선택
- **확장성**: 필요한 서비스만 선택적 확장
- **팀 자율성**: 작은 팀이 서비스 전체 생명주기 관리

### Challenges
- **복잡성**: 분산 시스템의 복잡성 증가
- **네트워크 지연**: 서비스 간 네트워크 호출 오버헤드
- **데이터 일관성**: 분산 데이터 관리의 어려움
- **운영 오버헤드**: 다수 서비스 모니터링과 관리
# 1-3.로드 밸런싱과 트래픽 분산: L4/L7 로드 밸런서와 분산 알고리즘

## 개요

로드 밸런싱은 고가용성과 확장성을 달성하는 핵심 기술입니다. 이 문서는 L4와 L7 로드 밸런서의 차이점, 다양한 분산 알고리즘의 특성, 그리고 세션 지속성 문제의 해결책을 다룹니다. 시니어 개발자로서 트래픽 패턴과 애플리케이션 특성에 맞는 로드 밸런싱 전략을 설계하는 방법을 학습합니다.

## 로드 밸런서 유형

### Layer 4 (전송 계층) 로드 밸런서

**특징 및 동작 원리**
- TCP/UDP 패킷의 IP 주소와 포트 정보만 사용
- 애플리케이션 내용을 검사하지 않음
- 네트워크 계층에서 빠른 패킷 전달
- 연결별 로드 밸런싱 (Connection-based)

**장점**:
- 높은 처리량과 낮은 지연시간
- 프로토콜에 독립적 (HTTP, HTTPS, TCP, UDP 모두 지원)
- 간단한 구성과 관리
- 낮은 CPU 사용률

**단점**:
- 콘텐츠 기반 라우팅 불가능
- 고급 헬스 체크 제한적
- 세션 기반 라우팅 어려움

**구현 예시 - HAProxy L4 설정**
```
global
    daemon
    maxconn 4096

defaults
    mode tcp
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend tcp_frontend
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
    server web3 192.168.1.12:8080 check
```

### Layer 7 (애플리케이션 계층) 로드 밸런서

**특징 및 동작 원리**
- HTTP/HTTPS 헤더, URL, 쿠키 등 애플리케이션 데이터 검사
- 콘텐츠 기반 라우팅 가능
- SSL 종료 및 압축 기능
- 요청별 로드 밸런싱 (Request-based)

**장점**:
- 정교한 라우팅 규칙
- SSL 오프로딩
- 압축 및 캐싱 기능
- 상세한 헬스 체크

**단점**:
- 높은 CPU 사용률
- 애플리케이션 계층 처리로 인한 지연
- HTTP/HTTPS에 제한적

**구현 예시 - NGINX L7 설정**
```nginx
upstream api_servers {
    least_conn;
    server api1.example.com:8080 weight=3 max_fails=3 fail_timeout=30s;
    server api2.example.com:8080 weight=2 max_fails=3 fail_timeout=30s;
    server api3.example.com:8080 weight=1 max_fails=3 fail_timeout=30s;
    
    # 헬스 체크
    keepalive 32;
}

upstream static_servers {
    server static1.example.com:8080;
    server static2.example.com:8080;
}

server {
    listen 80;
    server_name example.com;
    
    # SSL 리다이렉트
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;
    
    # SSL 설정
    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE+AESGCM:ECDHE+AES256:ECDHE+AES128:!aNULL:!MD5:!DSS;
    
    # 압축 설정
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript;
    
    # API 요청 라우팅
    location /api/ {
        proxy_pass http://api_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 타임아웃 설정
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # 연결 재사용
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
    
    # 정적 콘텐츠 라우팅
    location /static/ {
        proxy_pass http://static_servers;
        proxy_cache_valid 200 1h;
        add_header Cache-Control "public, max-age=3600";
    }
    
    # 관리자 인터페이스 (특정 IP만 허용)
    location /admin/ {
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        deny all;
        
        proxy_pass http://api_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 로드 밸런싱 알고리즘

### Round Robin

**동작 원리**: 요청을 순서대로 각 서버에 분배

**구현 예시**:
```java
@Component
public class RoundRobinLoadBalancer implements LoadBalancer {
    
    private final AtomicInteger currentIndex = new AtomicInteger(0);
    private final List<Server> servers;
    
    public RoundRobinLoadBalancer(List<Server> servers) {
        this.servers = new ArrayList<>(servers);
    }
    
    @Override
    public Server selectServer() {
        if (servers.isEmpty()) {
            return null;
        }
        
        int index = currentIndex.getAndIncrement() % servers.size();
        return servers.get(index);
    }
    
    @Override
    public void addServer(Server server) {
        servers.add(server);
    }
    
    @Override
    public void removeServer(Server server) {
        servers.remove(server);
        // 인덱스 조정
        if (currentIndex.get() >= servers.size()) {
            currentIndex.set(0);
        }
    }
}
```

**특징**:
- 간단하고 공평한 분배
- 서버 용량이 동일할 때 적합
- 무상태 요청에 이상적

### Weighted Round Robin

**동작 원리**: 서버 가중치에 따라 요청 분배

```java
@Component  
public class WeightedRoundRobinLoadBalancer implements LoadBalancer {
    
    private final List<WeightedServer> servers;
    private final AtomicInteger currentIndex = new AtomicInteger(0);
    
    public WeightedRoundRobinLoadBalancer(List<WeightedServer> servers) {
        this.servers = new ArrayList<>(servers);
    }
    
    @Override
    public Server selectServer() {
        if (servers.isEmpty()) {
            return null;
        }
        
        // 총 가중치 계산
        int totalWeight = servers.stream()
            .mapToInt(WeightedServer::getWeight)
            .sum();
        
        if (totalWeight == 0) {
            return servers.get(0).getServer();
        }
        
        // 현재 가중치 업데이트
        for (WeightedServer ws : servers) {
            ws.addCurrentWeight(ws.getWeight());
        }
        
        // 가장 높은 현재 가중치를 가진 서버 선택
        WeightedServer selected = servers.stream()
            .max(Comparator.comparingInt(WeightedServer::getCurrentWeight))
            .orElse(servers.get(0));
        
        // 선택된 서버의 현재 가중치 감소
        selected.subtractCurrentWeight(totalWeight);
        
        return selected.getServer();
    }
    
    static class WeightedServer {
        private final Server server;
        private final int weight;
        private int currentWeight;
        
        public WeightedServer(Server server, int weight) {
            this.server = server;
            this.weight = weight;
            this.currentWeight = 0;
        }
        
        public void addCurrentWeight(int weight) {
            this.currentWeight += weight;
        }
        
        public void subtractCurrentWeight(int weight) {
            this.currentWeight -= weight;
        }
        
        // getters...
    }
}
```

### Least Connections

**동작 원리**: 현재 연결 수가 가장 적은 서버 선택

```java
@Component
public class LeastConnectionsLoadBalancer implements LoadBalancer {
    
    private final Map<Server, AtomicInteger> connectionCounts = new ConcurrentHashMap<>();
    private final List<Server> servers;
    
    public LeastConnectionsLoadBalancer(List<Server> servers) {
        this.servers = new ArrayList<>(servers);
        servers.forEach(server -> connectionCounts.put(server, new AtomicInteger(0)));
    }
    
    @Override
    public Server selectServer() {
        if (servers.isEmpty()) {
            return null;
        }
        
        return servers.stream()
            .min(Comparator.comparingInt(server -> 
                connectionCounts.get(server).get()))
            .orElse(servers.get(0));
    }
    
    public void incrementConnections(Server server) {
        connectionCounts.get(server).incrementAndGet();
    }
    
    public void decrementConnections(Server server) {
        connectionCounts.get(server).decrementAndGet();
    }
    
    @EventListener
    public void handleConnectionEstablished(ConnectionEstablishedEvent event) {
        incrementConnections(event.getServer());
    }
    
    @EventListener  
    public void handleConnectionClosed(ConnectionClosedEvent event) {
        decrementConnections(event.getServer());
    }
}
```

### IP Hash

**동작 원리**: 클라이언트 IP 해시값으로 서버 결정

```java
@Component
public class IpHashLoadBalancer implements LoadBalancer {
    
    private final List<Server> servers;
    private final ConsistentHashRing hashRing;
    
    public IpHashLoadBalancer(List<Server> servers) {
        this.servers = new ArrayList<>(servers);
        this.hashRing = new ConsistentHashRing(150); // 가상 노드 수
        
        servers.forEach(server -> 
            hashRing.addNode(server.getId()));
    }
    
    @Override
    public Server selectServer(String clientIp) {
        if (servers.isEmpty()) {
            return null;
        }
        
        String serverId = hashRing.getNode(clientIp);
        return servers.stream()
            .filter(server -> server.getId().equals(serverId))
            .findFirst()
            .orElse(servers.get(0));
    }
    
    public Server selectServer(HttpServletRequest request) {
        String clientIp = getClientIp(request);
        return selectServer(clientIp);
    }
    
    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        
        String xRealIp = request.getHeader("X-Real-IP");
        if (xRealIp != null && !xRealIp.isEmpty()) {
            return xRealIp;
        }
        
        return request.getRemoteAddr();
    }
}
```

## 세션 지속성 해결책

### 문제점
- HTTP는 무상태 프로토콜이지만 웹 애플리케이션은 세션 상태 필요
- 로드 밸런서가 요청을 다른 서버로 분산하면 세션 정보 손실

### 해결책 1: Sticky Sessions (Session Affinity)

```nginx
upstream backend {
    ip_hash; # IP 기반 세션 고정
    server web1:8080;
    server web2:8080;
    server web3:8080;
}

# 또는 쿠키 기반
upstream backend {
    hash $cookie_sessionid consistent;
    server web1:8080;
    server web2:8080;
    server web3:8080;
}
```

**장점**: 기존 애플리케이션 코드 변경 불필요
**단점**: 확장성 제한, 서버 장애 시 세션 손실

### 해결책 2: 외부 세션 저장소

```java
@Configuration
@EnableRedisHttpSession
public class SessionConfig {
    
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration("redis-server", 6379));
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory());
        template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}

@RestController
public class SessionController {
    
    @PostMapping("/login")
    public ResponseEntity<String> login(
            @RequestBody LoginRequest request,
            HttpServletRequest httpRequest) {
        
        // 사용자 인증
        User user = authService.authenticate(request.getUsername(), request.getPassword());
        
        // 세션에 사용자 정보 저장 (Redis에 자동 저장됨)
        HttpSession session = httpRequest.getSession();
        session.setAttribute("user", user);
        session.setAttribute("loginTime", Instant.now());
        
        return ResponseEntity.ok("로그인 성공");
    }
    
    @GetMapping("/profile")
    public ResponseEntity<User> getProfile(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        if (session == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        
        User user = (User) session.getAttribute("user");
        if (user == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        
        return ResponseEntity.ok(user);
    }
}
```

### 해결책 3: JWT 토큰 (권장)

```java
@Service
public class JwtService {
    
    private final String secretKey = "your-secret-key";
    private final long expirationTime = 86400000; // 24시간
    
    public String generateToken(User user) {
        return Jwts.builder()
            .setSubject(user.getUsername())
            .claim("userId", user.getId())
            .claim("roles", user.getRoles())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expirationTime))
            .signWith(SignatureAlgorithm.HS512, secretKey)
            .compact();
    }
    
    public Claims parseToken(String token) {
        return Jwts.parser()
            .setSigningKey(secretKey)
            .parseClaimsJws(token)
            .getBody();
    }
    
    public boolean isTokenValid(String token) {
        try {
            Claims claims = parseToken(token);
            return !claims.getExpiration().before(new Date());
        } catch (Exception e) {
            return false;
        }
    }
}

@RestController
public class JwtAuthController {
    
    @PostMapping("/api/auth/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        User user = authService.authenticate(request.getUsername(), request.getPassword());
        String token = jwtService.generateToken(user);
        
        return ResponseEntity.ok(new AuthResponse(token, user.getId()));
    }
    
    @GetMapping("/api/user/profile")
    public ResponseEntity<User> getProfile(@RequestHeader("Authorization") String authHeader) {
        String token = authHeader.replace("Bearer ", "");
        
        if (!jwtService.isTokenValid(token)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        
        Claims claims = jwtService.parseToken(token);
        String userId = claims.get("userId", String.class);
        User user = userService.findById(userId);
        
        return ResponseEntity.ok(user);
    }
}
```

## 헬스 체크와 장애 처리

### 다단계 헬스 체크

```java
@Component
public class HealthCheckService {
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private ApplicationContext applicationContext;
    
    public HealthStatus checkHealth() {
        HealthStatus status = new HealthStatus();
        status.setTimestamp(Instant.now());
        status.setOverallStatus("UP");
        
        // 기본 애플리케이션 상태
        status.addCheck("application", checkApplication());
        
        // 데이터베이스 연결
        status.addCheck("database", checkDatabase());
        
        // Redis 연결
        status.addCheck("redis", checkRedis());
        
        // 외부 서비스 연결
        status.addCheck("external-api", checkExternalServices());
        
        // 디스크 공간
        status.addCheck("diskspace", checkDiskSpace());
        
        // 전체 상태 결정
        boolean hasFailure = status.getChecks().values().stream()
            .anyMatch(check -> "DOWN".equals(check.getStatus()));
        
        if (hasFailure) {
            status.setOverallStatus("DOWN");
        }
        
        return status;
    }
    
    private HealthCheck checkDatabase() {
        try {
            Connection conn = dataSource.getConnection();
            PreparedStatement stmt = conn.prepareStatement("SELECT 1");
            ResultSet rs = stmt.executeQuery();
            
            if (rs.next() && rs.getInt(1) == 1) {
                return HealthCheck.builder()
                    .status("UP")
                    .details(Map.of("database", "연결 정상"))
                    .build();
            }
        } catch (SQLException e) {
            return HealthCheck.builder()
                .status("DOWN")
                .details(Map.of("database", "연결 실패: " + e.getMessage()))
                .build();
        }
        
        return HealthCheck.builder()
            .status("DOWN")
            .details(Map.of("database", "알 수 없는 오류"))
            .build();
    }
    
    private HealthCheck checkRedis() {
        try {
            String testKey = "health:check:" + System.currentTimeMillis();
            redisTemplate.opsForValue().set(testKey, "ok", Duration.ofSeconds(10));
            String result = redisTemplate.opsForValue().get(testKey);
            redisTemplate.delete(testKey);
            
            if ("ok".equals(result)) {
                return HealthCheck.builder()
                    .status("UP")
                    .details(Map.of("redis", "연결 정상"))
                    .build();
            }
        } catch (Exception e) {
            return HealthCheck.builder()
                .status("DOWN")
                .details(Map.of("redis", "연결 실패: " + e.getMessage()))
                .build();
        }
        
        return HealthCheck.builder()
            .status("DOWN")
            .details(Map.of("redis", "테스트 실패"))
            .build();
    }
}

@RestController
public class HealthController {
    
    @Autowired
    private HealthCheckService healthCheckService;
    
    @GetMapping("/health")
    public ResponseEntity<HealthStatus> health() {
        HealthStatus status = healthCheckService.checkHealth();
        
        HttpStatus httpStatus = "UP".equals(status.getOverallStatus()) 
            ? HttpStatus.OK 
            : HttpStatus.SERVICE_UNAVAILABLE;
        
        return ResponseEntity.status(httpStatus).body(status);
    }
    
    @GetMapping("/health/liveness")
    public ResponseEntity<Map<String, String>> liveness() {
        // 기본적인 생존 확인 (Kubernetes liveness probe용)
        return ResponseEntity.ok(Map.of("status", "UP"));
    }
    
    @GetMapping("/health/readiness")
    public ResponseEntity<Map<String, String>> readiness() {
        // 트래픽 수신 준비 상태 확인 (Kubernetes readiness probe용)
        HealthStatus status = healthCheckService.checkHealth();
        
        HttpStatus httpStatus = "UP".equals(status.getOverallStatus()) 
            ? HttpStatus.OK 
            : HttpStatus.SERVICE_UNAVAILABLE;
        
        return ResponseEntity.status(httpStatus)
            .body(Map.of("status", status.getOverallStatus()));
    }
}
```

## 성능 모니터링과 최적화

### NGINX 성능 모니터링

```nginx
# nginx.conf에 모니터링 설정 추가
http {
    # 로그 형식 정의
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for" '
                   'rt=$request_time uct="$upstream_connect_time" '
                   'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    access_log /var/log/nginx/access.log main;
    
    # 상태 페이지 설정 (모니터링용)
    server {
        listen 8080;
        server_name localhost;
        
        location /nginx_status {
            stub_status on;
            access_log off;
            allow 127.0.0.1;
            allow 10.0.0.0/8;
            deny all;
        }
        
        location /upstream_status {
            access_log off;
            allow 127.0.0.1;
            allow 10.0.0.0/8;
            deny all;
            
            proxy_pass http://backend/health;
        }
    }
}
```

### 성능 지표 수집

```java
@Component
public class LoadBalancerMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter requestCounter;
    private final Timer responseTimer;
    private final Gauge activeConnections;
    
    public LoadBalancerMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.requestCounter = Counter.builder("lb.requests.total")
            .description("총 요청 수")
            .register(meterRegistry);
            
        this.responseTimer = Timer.builder("lb.response.time")
            .description("응답 시간")
            .register(meterRegistry);
            
        this.activeConnections = Gauge.builder("lb.connections.active")
            .description("활성 연결 수")
            .register(meterRegistry, this, LoadBalancerMetrics::getActiveConnectionCount);
    }
    
    public void recordRequest(String method, String uri, int statusCode, long duration) {
        requestCounter.increment(
            Tags.of(
                "method", method,
                "uri", uri,
                "status", String.valueOf(statusCode)
            )
        );
        
        responseTimer.record(duration, TimeUnit.MILLISECONDS,
            Tags.of(
                "method", method,
                "status", String.valueOf(statusCode)
            )
        );
    }
    
    private double getActiveConnectionCount() {
        // 실제 활성 연결 수 조회 로직
        return connectionPool.getActiveCount();
    }
}

@Component
public class MetricsFilter implements Filter {
    
    @Autowired
    private LoadBalancerMetrics metrics;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        long startTime = System.currentTimeMillis();
        
        try {
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            
            metrics.recordRequest(
                httpRequest.getMethod(),
                httpRequest.getRequestURI(),
                httpResponse.getStatus(),
                duration
            );
        }
    }
}
```

## 실습 프로젝트: Docker Compose 기반 로드 밸런싱

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl
    depends_on:
      - app1
      - app2
      - app3
    restart: unless-stopped

  app1:
    build: .
    environment:
      - SERVER_ID=app1
      - SERVER_PORT=8080
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
      - mysql
    restart: unless-stopped

  app2:
    build: .
    environment:
      - SERVER_ID=app2
      - SERVER_PORT=8080
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
      - mysql
    restart: unless-stopped

  app3:
    build: .
    environment:
      - SERVER_ID=app3
      - SERVER_PORT=8080
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
      - mysql
    restart: unless-stopped

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: ecommerce
    volumes:
      - mysql_data:/var/lib/mysql
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

volumes:
  redis_data:
  mysql_data:
  grafana_data:
```

## 핵심 포인트

1. **L4 vs L7 선택**: 단순한 TCP 로드 밸런싱이면 L4, 콘텐츠 기반 라우팅이 필요하면 L7

2. **알고리즘 선택**: 서버 용량이 같으면 Round Robin, 다르면 Weighted, 긴 연결이면 Least Connections

3. **세션 지속성**: Sticky Session보다는 외부 세션 저장소나 JWT 사용 권장

4. **헬스 체크**: 단순한 TCP 연결 확인을 넘어 애플리케이션 레벨 상태 확인 필요

5. **모니터링**: 응답 시간, 처리량, 오류율을 지속적으로 모니터링하여 성능 최적화

이러한 로드 밸런싱 기법은 다음 단계의 캐싱 전략과 결합되어 더욱 강력한 확장성을 제공합니다.
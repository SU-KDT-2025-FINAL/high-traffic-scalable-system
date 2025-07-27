# 3.4 서비스 메시와 관찰 가능성

## Overview
서비스 메시는 마이크로서비스 간 통신을 관리하는 인프라 계층입니다. **Istio**와 같은 서비스 메시를 통해 **사이드카 패턴**으로 서비스 간 통신을 제어하고, **분산 추적**과 **관찰 가능성**을 확보할 수 있습니다.

## 서비스 메시 아키텍처

### 사이드카 패턴
```
┌─ Service A ─┐    ┌─ Service B ─┐
│ App │ Envoy │───▶│ Envoy │ App │
└─────────────┘    └─────────────┘
      │                    │
      └──── Control Plane ─┘
           (Istiod)
```

**특징**:
- 각 서비스 인스턴스마다 사이드카 프록시 배치
- 모든 네트워크 트래픽이 사이드카를 통과
- 애플리케이션 코드 변경 없이 기능 추가

### 데이터 플레인 vs 컨트롤 플레인
```
Control Plane (Istiod):
├─ Pilot: 트래픽 관리 규칙 배포
├─ Citadel: 보안 정책 관리
└─ Galley: 설정 검증 및 배포

Data Plane (Envoy Proxy):
├─ 트래픽 라우팅
├─ 로드 밸런싱
├─ 회로 차단
├─ 재시도 및 타임아웃
└─ 텔레메트리 수집
```

## Istio 설치 및 설정

### Istio 설치
```bash
# Istio 다운로드
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.19.0
export PATH=$PWD/bin:$PATH

# Istio 설치
istioctl install --set values.defaultRevision=default

# 네임스페이스에 사이드카 자동 주입 활성화
kubectl label namespace default istio-injection=enabled
```

### Gateway 설정
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ecommerce-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: ecommerce-tls
    hosts:
    - api.ecommerce.com
```

### VirtualService 설정
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service-vs
spec:
  hosts:
  - api.ecommerce.com
  gateways:
  - ecommerce-gateway
  http:
  - match:
    - uri:
        prefix: /api/orders
    route:
    - destination:
        host: order-service
        port:
          number: 8080
      weight: 90
    - destination:
        host: order-service-v2
        port:
          number: 8080
      weight: 10
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
```

### DestinationRule 설정
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service-dr
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 10
    circuitBreaker:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 50
```

## 트래픽 관리

### 카나리 배포
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: canary-deployment
spec:
  hosts:
  - user-service
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: user-service
        subset: v2
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 95
    - destination:
        host: user-service
        subset: v2
      weight: 5
```

### A/B 테스트
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ab-testing
spec:
  hosts:
  - product-service
  http:
  - match:
    - headers:
        user-segment:
          exact: "premium"
    route:
    - destination:
        host: product-service
        subset: premium-features
  - match:
    - headers:
        user-segment:
          exact: "standard"
    route:
    - destination:
        host: product-service
        subset: standard-features
  - route:
    - destination:
        host: product-service
        subset: standard-features
```

### 장애 주입
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chaos-testing
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        chaos-test:
          exact: "true"
    fault:
      abort:
        percentage:
          value: 10
        httpStatus: 500
      delay:
        percentage:
          value: 20
        fixedDelay: 3s
    route:
    - destination:
        host: payment-service
```

## 보안 설정

### mTLS 설정
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: default
  namespace: production
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

### 인가 정책
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/api-gateway"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/orders", "/orders/*"]
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/admin"]
    to:
    - operation:
        methods: ["*"]
```

### JWT 검증
```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  jwtRules:
  - issuer: "https://auth.ecommerce.com"
    jwksUri: "https://auth.ecommerce.com/.well-known/jwks.json"
    audiences:
    - "ecommerce-api"
    forwardOriginalToken: true
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-gateway
  rules:
  - from:
    - source:
        requestPrincipals: ["https://auth.ecommerce.com/*"]
```

## 분산 추적 구현

### OpenTelemetry 설정
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
    
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      memory_limiter:
        limit_mib: 256
    
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
      prometheus:
        endpoint: "0.0.0.0:8889"
    
    service:
      pipelines:
        traces:
          receivers: [otlp, jaeger]
          processors: [memory_limiter, batch]
          exporters: [jaeger]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
```

### 애플리케이션 트레이싱
```java
@RestController
public class OrderController {
    
    private final Tracer tracer;
    private final OrderService orderService;
    
    @PostMapping("/orders")
    public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
        Span span = tracer.spanBuilder("create-order")
            .setSpanKind(SpanKind.SERVER)
            .setAttribute("user.id", request.getUserId())
            .setAttribute("order.items.count", request.getItems().size())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 비즈니스 로직 실행
            Order order = orderService.createOrder(request);
            
            span.setAttribute("order.id", order.getId())
                .setAttribute("order.amount", order.getTotalAmount().toString())
                .setStatus(StatusCode.OK);
            
            return ResponseEntity.ok(OrderDto.from(order));
            
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

### 커스텀 메트릭 수집
```java
@Component
public class BusinessMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final Counter orderCounter;
    private final Timer orderProcessingTime;
    private final Gauge activeOrders;
    
    public BusinessMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.orderCounter = Counter.builder("orders.created.total")
            .description("Total number of orders created")
            .register(meterRegistry);
        
        this.orderProcessingTime = Timer.builder("order.processing.duration")
            .description("Order processing time")
            .register(meterRegistry);
        
        this.activeOrders = Gauge.builder("orders.active.count")
            .description("Number of active orders")
            .register(meterRegistry, this, BusinessMetricsCollector::getActiveOrderCount);
    }
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        orderCounter.increment(
            Tags.of("user.type", event.getUserType(), 
                   "order.type", event.getOrderType())
        );
    }
    
    @EventListener
    public void handleOrderProcessed(OrderProcessedEvent event) {
        orderProcessingTime.record(event.getProcessingDuration(), TimeUnit.MILLISECONDS);
    }
    
    private double getActiveOrderCount() {
        return orderService.getActiveOrderCount();
    }
}
```

## 로깅과 모니터링

### 구조화된 로깅
```java
@Aspect
@Component
public class StructuredLoggingAspect {
    
    private final ObjectMapper objectMapper;
    
    @Around("@annotation(Loggable)")
    public Object logMethodExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        // 분산 추적 정보 추출
        Span currentSpan = Span.current();
        String traceId = currentSpan.getSpanContext().getTraceId();
        String spanId = currentSpan.getSpanContext().getSpanId();
        
        StructuredLogEntry logEntry = StructuredLogEntry.builder()
            .timestamp(Instant.now())
            .service("order-service")
            .method(className + "." + methodName)
            .traceId(traceId)
            .spanId(spanId)
            .build();
        
        Instant start = Instant.now();
        try {
            Object result = joinPoint.proceed();
            
            Duration duration = Duration.between(start, Instant.now());
            logEntry.setStatus("SUCCESS");
            logEntry.setDurationMs(duration.toMillis());
            
            log.info(objectMapper.writeValueAsString(logEntry));
            
            return result;
            
        } catch (Exception e) {
            Duration duration = Duration.between(start, Instant.now());
            logEntry.setStatus("ERROR");
            logEntry.setDurationMs(duration.toMillis());
            logEntry.setErrorMessage(e.getMessage());
            
            log.error(objectMapper.writeValueAsString(logEntry));
            
            throw e;
        }
    }
}
```

### Prometheus 메트릭 수집
```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: istio-mesh-metrics
spec:
  selector:
    matchLabels:
      app: istiod
  endpoints:
  - port: http-monitoring
    interval: 30s
    path: /stats/prometheus
---
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: application-metrics
spec:
  selector:
    matchLabels:
      monitoring: enabled
  endpoints:
  - port: metrics
    interval: 15s
    path: /actuator/prometheus
```

### Grafana 대시보드
```json
{
  "dashboard": {
    "title": "Service Mesh Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total[5m])) by (destination_service_name)",
            "legendFormat": "{{destination_service_name}}"
          }
        ]
      },
      {
        "title": "Success Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{response_code!~\"5.*\"}[5m])) / sum(rate(istio_requests_total[5m])) * 100"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (destination_service_name, le))"
          }
        ]
      },
      {
        "title": "Circuit Breaker Status",
        "type": "table",
        "targets": [
          {
            "expr": "envoy_cluster_circuit_breakers_cx_open"
          }
        ]
      }
    ]
  }
}
```

## 성능 최적화

### Envoy 프록시 최적화
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-proxy-config
data:
  custom_bootstrap.json: |
    {
      "static_resources": {
        "clusters": [
          {
            "name": "local_service",
            "connect_timeout": "0.25s",
            "type": "STATIC",
            "lb_policy": "ROUND_ROBIN",
            "circuit_breakers": {
              "thresholds": [
                {
                  "priority": "DEFAULT",
                  "max_connections": 1000,
                  "max_pending_requests": 100,
                  "max_requests": 1000,
                  "max_retries": 3
                }
              ]
            }
          }
        ]
      },
      "admin": {
        "access_log_path": "/dev/null",
        "address": {
          "socket_address": {
            "address": "127.0.0.1",
            "port_value": 15000
          }
        }
      }
    }
```

### 리소스 최적화
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/proxyCPU: "100m"
        sidecar.istio.io/proxyMemory: "128Mi"
        sidecar.istio.io/proxyCPULimit: "200m"
        sidecar.istio.io/proxyMemoryLimit: "256Mi"
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
```

## 장애 대응

### 자동 복구 설정
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker-config
spec:
  host: payment-service
  trafficPolicy:
    circuitBreaker:
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
    outlierDetection:
      consecutiveGatewayErrors: 3
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 10s
      maxEjectionPercent: 20
```

### 알림 설정
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: istio-alerts
spec:
  groups:
  - name: istio.rules
    rules:
    - alert: HighErrorRate
      expr: |
        (
          sum(rate(istio_requests_total{response_code=~"5.*"}[5m])) by (destination_service_name) /
          sum(rate(istio_requests_total[5m])) by (destination_service_name)
        ) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.destination_service_name }}"
    
    - alert: HighLatency
      expr: |
        histogram_quantile(0.99,
          sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (destination_service_name, le)
        ) > 1000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High latency detected"
        description: "P99 latency is {{ $value }}ms for {{ $labels.destination_service_name }}"
```

## Best Practices

### 트래픽 관리
- **점진적 배포**: 카나리/블루-그린 배포 활용
- **Circuit Breaker**: 연쇄 장애 방지
- **재시도 정책**: 적절한 재시도와 백오프 설정

### 보안
- **mTLS**: 서비스 간 통신 암호화
- **인가 정책**: 최소 권한 원칙 적용
- **JWT 검증**: 토큰 기반 인증 구현

### 관찰 가능성
- **구조화된 로깅**: 일관된 로그 형식 사용
- **분산 추적**: 요청 흐름 추적
- **메트릭 수집**: 비즈니스와 기술 메트릭 수집

## Benefits and Challenges

### Benefits
- **투명성**: 애플리케이션 코드 변경 없이 기능 추가
- **관찰 가능성**: 자동 메트릭, 로그, 추적 수집
- **보안**: 자동 mTLS와 정책 기반 접근 제어
- **트래픽 관리**: 세밀한 트래픽 제어와 라우팅

### Challenges
- **복잡성**: 추가적인 인프라 구성 요소
- **성능 오버헤드**: 사이드카 프록시로 인한 지연
- **운영 부담**: 서비스 메시 자체의 모니터링과 관리
- **학습 곡선**: 새로운 개념과 도구 습득 필요
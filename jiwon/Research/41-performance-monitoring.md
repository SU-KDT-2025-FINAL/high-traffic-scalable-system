# 4.1 성능 모니터링과 관찰 가능성

## Overview
성능 모니터링은 시스템의 **관찰 가능성(Observability)**을 확보하여 문제를 사전에 감지하고 해결하는 핵심 활동입니다. **메트릭(Metrics)**, **로그(Logs)**, **추적(Traces)**의 3요소를 통해 시스템 상태를 종합적으로 파악할 수 있습니다.

## 관찰 가능성의 3요소

### 메트릭 (Metrics)
```
시계열 데이터로 시스템 상태 수치화
- Counter: 누적 카운트 (요청 수, 에러 수)
- Gauge: 현재 값 (CPU 사용률, 메모리 사용량)
- Histogram: 분포도 (응답 시간 분포)
- Summary: 요약 통계 (평균, 백분위수)
```

### 로그 (Logs)
```
시스템에서 발생하는 이벤트 기록
- 구조화된 로그: JSON 형태의 일관된 형식
- 로그 레벨: ERROR, WARN, INFO, DEBUG
- 컨텍스트 정보: 타임스탬프, 서비스명, 요청 ID
```

### 추적 (Traces)
```
분산 시스템에서 요청의 전체 흐름 추적
- Trace: 하나의 요청에 대한 전체 여정
- Span: 요청 처리의 개별 작업 단위
- Context: 서비스 간 추적 정보 전파
```

## SLI/SLO/SLA 설정

### SLI (Service Level Indicators)
```java
@Component
public class SLICollector {
    
    private final MeterRegistry meterRegistry;
    
    // 가용성 SLI
    @EventListener
    public void collectAvailability(RequestCompletedEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        sample.stop(Timer.builder("http.request.duration")
            .tag("method", event.getMethod())
            .tag("status", event.getStatus())
            .tag("endpoint", event.getEndpoint())
            .register(meterRegistry));
        
        // 성공률 계산
        Counter.builder("http.requests.total")
            .tag("status_class", getStatusClass(event.getStatus()))
            .register(meterRegistry)
            .increment();
    }
    
    // 지연시간 SLI
    public void recordLatency(String operation, Duration duration) {
        Timer.builder("operation.duration")
            .tag("operation", operation)
            .register(meterRegistry)
            .record(duration);
    }
    
    // 처리량 SLI
    public void recordThroughput(String service, int requestCount) {
        Counter.builder("service.throughput")
            .tag("service", service)
            .register(meterRegistry)
            .increment(requestCount);
    }
    
    private String getStatusClass(int status) {
        if (status >= 200 && status < 300) return "2xx";
        if (status >= 300 && status < 400) return "3xx";
        if (status >= 400 && status < 500) return "4xx";
        if (status >= 500) return "5xx";
        return "unknown";
    }
}
```

### SLO (Service Level Objectives) 정의
```yaml
# slo-config.yaml
slos:
  order-service:
    availability:
      target: 99.9%  # 99.9% 가용성
      measurement_window: 30d
      
    latency:
      target: 95%    # 95%의 요청이 200ms 이하
      threshold: 200ms
      measurement_window: 30d
      
    throughput:
      target: 1000   # 초당 1000 요청 처리
      measurement_window: 1h
      
  user-service:
    availability:
      target: 99.95%
      measurement_window: 30d
      
    latency:
      target: 99%
      threshold: 100ms
      measurement_window: 30d
```

### Error Budget 관리
```java
@Service
public class ErrorBudgetService {
    
    private final MetricsRepository metricsRepository;
    private final AlertService alertService;
    
    @Scheduled(fixedRate = 300000) // 5분마다
    public void calculateErrorBudget() {
        for (ServiceSLO slo : getSLOList()) {
            ErrorBudget budget = calculateBudget(slo);
            
            if (budget.isExhausted()) {
                alertService.sendErrorBudgetAlert(slo.getServiceName(), budget);
                freezeDeployments(slo.getServiceName());
            } else if (budget.getBurnRate() > BURN_RATE_THRESHOLD) {
                alertService.sendHighBurnRateAlert(slo.getServiceName(), budget);
            }
            
            updateErrorBudgetMetrics(slo.getServiceName(), budget);
        }
    }
    
    private ErrorBudget calculateBudget(ServiceSLO slo) {
        Duration window = slo.getMeasurementWindow();
        Instant start = Instant.now().minus(window);
        
        // 총 요청 수
        long totalRequests = metricsRepository.getTotalRequests(
            slo.getServiceName(), start, Instant.now());
        
        // 성공 요청 수
        long successRequests = metricsRepository.getSuccessRequests(
            slo.getServiceName(), start, Instant.now());
        
        // 현재 가용성
        double currentAvailability = (double) successRequests / totalRequests;
        
        // 에러 버짓 계산
        double allowedErrorRate = 1.0 - slo.getAvailabilityTarget();
        long allowedErrors = (long) (totalRequests * allowedErrorRate);
        long actualErrors = totalRequests - successRequests;
        
        return ErrorBudget.builder()
            .totalBudget(allowedErrors)
            .consumed(actualErrors)
            .remaining(allowedErrors - actualErrors)
            .burnRate(calculateBurnRate(actualErrors, allowedErrors, window))
            .build();
    }
}
```

## Prometheus와 Grafana 구축

### Prometheus 설정
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"
  - "recording_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'spring-actuator'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['order-service:8080', 'user-service:8080']
    scrape_interval: 10s
    
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
```

### 알림 규칙
```yaml
# alert_rules.yml
groups:
- name: ecommerce.rules
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
      service: "{{ $labels.service }}"
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
      runbook_url: "https://wiki.company.com/runbooks/high-error-rate"

  - alert: HighLatency
    expr: |
      histogram_quantile(0.95, 
        rate(http_request_duration_seconds_bucket[5m])
      ) > 0.2
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High latency detected"
      description: "95th percentile latency is {{ $value }}s"

  - alert: ErrorBudgetExhausted
    expr: |
      (
        1 - (
          rate(http_requests_total{status!~"5.."}[30d]) /
          rate(http_requests_total[30d])
        )
      ) > 0.001
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Error budget exhausted"
      description: "Service {{ $labels.service }} has exhausted its error budget"
```

### Grafana 대시보드
```json
{
  "dashboard": {
    "title": "Service Performance Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"color": "green", "value": 0},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        }
      },
      {
        "title": "Response Time Percentiles",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P99"
          }
        ]
      },
      {
        "title": "Error Budget Status",
        "type": "table",
        "targets": [
          {
            "expr": "error_budget_remaining",
            "format": "table"
          }
        ]
      }
    ]
  }
}
```

## 중앙집중식 로깅

### ELK Stack 구성
```yaml
# docker-compose.yml
version: '3.7'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.0
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
    depends_on:
      - elasticsearch
```

### Logstash 파이프라인
```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
  
  http {
    port => 5000
    codec => json
  }
}

filter {
  if [fields][service] {
    mutate {
      add_field => { "service_name" => "%{[fields][service]}" }
    }
  }
  
  # JSON 로그 파싱
  if [message] =~ /^{.*}$/ {
    json {
      source => "message"
    }
  }
  
  # 타임스탬프 파싱
  if [timestamp] {
    date {
      match => [ "timestamp", "ISO8601" ]
    }
  }
  
  # 에러 로그 분류
  if [level] == "ERROR" {
    mutate {
      add_tag => [ "error" ]
    }
  }
  
  # 트레이스 ID 추출
  if [traceId] {
    mutate {
      add_field => { "trace_id" => "%{traceId}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  if "error" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
  }
}
```

### 구조화된 로깅
```java
@Component
public class StructuredLogger {
    
    private final ObjectMapper objectMapper;
    private final Logger log = LoggerFactory.getLogger(StructuredLogger.class);
    
    public void logBusinessEvent(String eventType, Map<String, Object> eventData) {
        LogEntry entry = LogEntry.builder()
            .timestamp(Instant.now())
            .service("order-service")
            .eventType(eventType)
            .traceId(getCurrentTraceId())
            .spanId(getCurrentSpanId())
            .data(eventData)
            .build();
        
        try {
            String jsonLog = objectMapper.writeValueAsString(entry);
            log.info(jsonLog);
        } catch (Exception e) {
            log.error("Failed to serialize log entry", e);
        }
    }
    
    public void logError(String operation, Exception error, Map<String, Object> context) {
        ErrorLogEntry entry = ErrorLogEntry.builder()
            .timestamp(Instant.now())
            .service("order-service")
            .operation(operation)
            .errorType(error.getClass().getSimpleName())
            .errorMessage(error.getMessage())
            .stackTrace(getStackTrace(error))
            .traceId(getCurrentTraceId())
            .spanId(getCurrentSpanId())
            .context(context)
            .build();
        
        try {
            String jsonLog = objectMapper.writeValueAsString(entry);
            log.error(jsonLog);
        } catch (Exception e) {
            log.error("Failed to serialize error log entry", e);
        }
    }
}
```

## 애플리케이션 성능 모니터링 (APM)

### 커스텀 메트릭 수집
```java
@Component
public class BusinessMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    
    // 비즈니스 메트릭
    private final Counter orderCounter;
    private final Timer orderProcessingTime;
    private final Gauge inventoryLevel;
    private final Counter paymentCounter;
    
    public BusinessMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.orderCounter = Counter.builder("business.orders.created")
            .description("Number of orders created")
            .register(meterRegistry);
        
        this.orderProcessingTime = Timer.builder("business.order.processing.time")
            .description("Time taken to process an order")
            .register(meterRegistry);
        
        this.inventoryLevel = Gauge.builder("business.inventory.level")
            .description("Current inventory level")
            .register(meterRegistry, this, BusinessMetricsCollector::getCurrentInventoryLevel);
        
        this.paymentCounter = Counter.builder("business.payments.processed")
            .description("Number of payments processed")
            .register(meterRegistry);
    }
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        orderCounter.increment(
            Tags.of(
                "customer_type", event.getCustomerType(),
                "order_channel", event.getChannel(),
                "order_value_range", categorizeOrderValue(event.getTotalAmount())
            )
        );
    }
    
    @EventListener
    public void handleOrderProcessed(OrderProcessedEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        sample.stop(orderProcessingTime.tag("status", event.getStatus()));
    }
    
    @EventListener
    public void handlePaymentProcessed(PaymentProcessedEvent event) {
        paymentCounter.increment(
            Tags.of(
                "payment_method", event.getPaymentMethod(),
                "payment_status", event.getStatus(),
                "amount_range", categorizePaymentAmount(event.getAmount())
            )
        );
    }
    
    private double getCurrentInventoryLevel() {
        return inventoryService.getTotalInventoryValue();
    }
    
    private String categorizeOrderValue(BigDecimal amount) {
        if (amount.compareTo(new BigDecimal("100")) < 0) return "small";
        if (amount.compareTo(new BigDecimal("500")) < 0) return "medium";
        return "large";
    }
}
```

### 성능 프로파일링
```java
@Component
@Profile("performance-monitoring")
public class PerformanceProfiler {
    
    private final MeterRegistry meterRegistry;
    
    @Around("@annotation(ProfilePerformance)")
    public Object profileMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            Object result = joinPoint.proceed();
            
            sample.stop(Timer.builder("method.execution.time")
                .tag("class", className)
                .tag("method", methodName)
                .tag("status", "success")
                .register(meterRegistry));
            
            return result;
            
        } catch (Exception e) {
            sample.stop(Timer.builder("method.execution.time")
                .tag("class", className)
                .tag("method", methodName)
                .tag("status", "error")
                .register(meterRegistry));
            
            // 에러 카운터
            Counter.builder("method.execution.errors")
                .tag("class", className)
                .tag("method", methodName)
                .tag("error_type", e.getClass().getSimpleName())
                .register(meterRegistry)
                .increment();
            
            throw e;
        }
    }
    
    @EventListener
    @Async
    public void profileSlowQueries(SlowQueryEvent event) {
        if (event.getExecutionTime() > Duration.ofSeconds(1)) {
            Timer.builder("database.slow.query")
                .tag("query_type", event.getQueryType())
                .tag("table", event.getTableName())
                .register(meterRegistry)
                .record(event.getExecutionTime());
        }
    }
}
```

## 알림 시스템

### Alertmanager 설정
```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@ecommerce.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default'
  routes:
  - match:
      severity: critical
    receiver: 'critical-alerts'
  - match:
      severity: warning
    receiver: 'warning-alerts'

receivers:
- name: 'default'
  email_configs:
  - to: 'team@ecommerce.com'
    subject: 'Alert: {{ .GroupLabels.alertname }}'
    body: |
      {{ range .Alerts }}
      Alert: {{ .Annotations.summary }}
      Description: {{ .Annotations.description }}
      {{ end }}

- name: 'critical-alerts'
  email_configs:
  - to: 'oncall@ecommerce.com'
    subject: 'CRITICAL: {{ .GroupLabels.alertname }}'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    channel: '#alerts'
    title: 'Critical Alert'
    text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

- name: 'warning-alerts'
  email_configs:
  - to: 'dev-team@ecommerce.com'
    subject: 'WARNING: {{ .GroupLabels.alertname }}'
```

### 지능형 알림
```java
@Service
public class IntelligentAlertingService {
    
    private final AlertRepository alertRepository;
    private final NotificationService notificationService;
    
    public void processAlert(Alert alert) {
        // 중복 알림 방지
        if (isDuplicate(alert)) {
            log.debug("Duplicate alert filtered: {}", alert.getAlertName());
            return;
        }
        
        // 알림 심각도 평가
        AlertSeverity severity = evaluateSeverity(alert);
        
        // 비즈니스 시간 고려
        if (isBusinessHours() && severity.ordinal() >= AlertSeverity.HIGH.ordinal()) {
            sendImmediateAlert(alert);
        } else if (severity == AlertSeverity.CRITICAL) {
            sendImmediateAlert(alert);
        } else {
            scheduleAlert(alert);
        }
        
        // 알림 이력 저장
        alertRepository.save(AlertHistory.from(alert));
        
        // 연관 알림 분석
        analyzeCorrelatedAlerts(alert);
    }
    
    private boolean isDuplicate(Alert alert) {
        return alertRepository.existsByAlertNameAndStatusAndCreatedAtAfter(
            alert.getAlertName(),
            AlertStatus.ACTIVE,
            Instant.now().minus(Duration.ofMinutes(5))
        );
    }
    
    private AlertSeverity evaluateSeverity(Alert alert) {
        // 규칙 기반 심각도 평가
        if (alert.getMetrics().containsKey("error_rate")) {
            double errorRate = (Double) alert.getMetrics().get("error_rate");
            if (errorRate > 0.1) return AlertSeverity.CRITICAL;
            if (errorRate > 0.05) return AlertSeverity.HIGH;
        }
        
        if (alert.getMetrics().containsKey("response_time")) {
            double responseTime = (Double) alert.getMetrics().get("response_time");
            if (responseTime > 5000) return AlertSeverity.HIGH;
            if (responseTime > 1000) return AlertSeverity.MEDIUM;
        }
        
        return AlertSeverity.LOW;
    }
    
    private void analyzeCorrelatedAlerts(Alert alert) {
        // 최근 5분간 유사한 알림 분석
        List<Alert> recentAlerts = alertRepository.findRecentAlerts(
            Duration.ofMinutes(5));
        
        if (recentAlerts.size() > 3) {
            // 다수의 알림이 발생하면 시스템 전체 문제로 판단
            notificationService.sendSystemWideAlert(
                "Multiple services experiencing issues", recentAlerts);
        }
    }
}
```

## Best Practices

### 메트릭 설계
- **Golden Signals**: Latency, Traffic, Errors, Saturation 중심 모니터링
- **비즈니스 메트릭**: 기술 메트릭과 함께 비즈니스 KPI 추적
- **카디널리티 관리**: 높은 카디널리티 라벨 사용 주의

### 로깅 전략
- **구조화**: 일관된 JSON 형태로 로그 구조화
- **컨텍스트**: 추적 가능한 상관관계 ID 포함
- **레벨링**: 적절한 로그 레벨 사용

### 알림 관리
- **피로도 방지**: 중복 알림 제거와 적절한 임계값 설정
- **액션 가능**: 실행 가능한 정보를 포함한 알림
- **에스컬레이션**: 심각도에 따른 알림 전달 체계

## Benefits and Challenges

### Benefits
- **사전 예방**: 문제 발생 전 조기 감지
- **신속 대응**: 장애 발생 시 빠른 문제 파악과 해결
- **성능 최적화**: 데이터 기반 성능 개선 의사결정
- **사용자 경험**: 서비스 품질 향상을 통한 사용자 만족도 증대

### Challenges
- **데이터 홍수**: 과도한 메트릭과 로그로 인한 정보 과부하
- **저장 비용**: 대용량 모니터링 데이터 저장 비용
- **복잡성**: 다양한 모니터링 도구 통합의 복잡성
- **알림 피로**: 과도한 알림으로 인한 중요 알림 무시
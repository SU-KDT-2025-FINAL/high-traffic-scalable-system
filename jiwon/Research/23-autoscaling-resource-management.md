# 2.3 오토스케일링과 리소스 관리

## Overview
오토스케일링은 시스템 부하에 따라 리소스를 자동으로 조정하는 기법입니다. **수평적 확장(Horizontal Scaling)**과 **수직적 확장(Vertical Scaling)**을 통해 트래픽 변화에 탄력적으로 대응할 수 있습니다.

## 스케일링 유형

### 수평적 확장 (HPA)
```
Normal Load:  [Pod1] [Pod2]
High Load:    [Pod1] [Pod2] [Pod3] [Pod4] [Pod5]
```

**특징**:
- 인스턴스 개수 증감
- 무상태 애플리케이션에 적합
- 로드 분산 필수

### 수직적 확장 (VPA)
```
Before: Pod (CPU: 100m, Memory: 128Mi)
After:  Pod (CPU: 500m, Memory: 512Mi)
```

**특징**:
- CPU/메모리 할당량 증감
- 상태가 있는 애플리케이션에 적합
- Pod 재시작 필요

## Kubernetes HPA 구현

### 기본 HPA 설정
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 커스텀 메트릭 HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-processor-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-processor
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: queue_length
      target:
        type: AverageValue
        averageValue: "5"
  - type: Object
    object:
      metric:
        name: requests_per_second
      describedObject:
        apiVersion: v1
        kind: Service
        name: api-service
      target:
        type: Value
        value: "100"
```

## 애플리케이션 메트릭 노출

### Spring Boot Actuator
```java
@RestController
public class MetricsController {
    private final MeterRegistry meterRegistry;
    private final QueueService queueService;
    
    @Bean
    public Gauge queueLengthGauge() {
        return Gauge.builder("queue_length")
            .description("Current queue length")
            .register(meterRegistry, queueService, QueueService::getQueueLength);
    }
    
    @GetMapping("/metrics/custom")
    public Map<String, Object> getCustomMetrics() {
        Map<String, Object> metrics = new HashMap<>();
        metrics.put("active_connections", connectionPool.getActiveConnections());
        metrics.put("processing_time", getAverageProcessingTime());
        metrics.put("error_rate", calculateErrorRate());
        return metrics;
    }
}
```

### Prometheus 메트릭
```java
@Component
public class CustomMetricsExporter {
    private final Counter requestCounter;
    private final Timer responseTimer;
    private final Gauge activeConnections;
    
    public CustomMetricsExporter(MeterRegistry registry) {
        this.requestCounter = Counter.builder("http_requests_total")
            .description("Total HTTP requests")
            .register(registry);
        
        this.responseTimer = Timer.builder("http_response_time")
            .description("HTTP response time")
            .register(registry);
        
        this.activeConnections = Gauge.builder("active_connections")
            .description("Active database connections")
            .register(registry, this, CustomMetricsExporter::getActiveConnections);
    }
    
    @EventListener
    public void handleRequest(RequestEvent event) {
        requestCounter.increment(Tags.of("method", event.getMethod(), 
                                        "status", event.getStatus()));
    }
}
```

## VPA 구현

### VPA 리소스 정의
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: database-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: database
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: database
      minAllowed:
        cpu: 100m
        memory: 256Mi
      maxAllowed:
        cpu: 2000m
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
```

### VPA 권장사항 확인
```bash
kubectl describe vpa database-vpa

# 출력 예시:
# Recommendation:
#   Container Recommendations:
#     Container Name: database
#     Lower Bound:
#       Cpu:     250m
#       Memory:  1Gi
#     Target:
#       Cpu:     500m
#       Memory:  2Gi
#     Upper Bound:
#       Cpu:     1000m
#       Memory:  4Gi
```

## 예측적 스케일링

### 시계열 기반 예측
```java
@Service
public class PredictiveScalingService {
    private final MetricsService metricsService;
    private final TimeSeriesPredictor predictor;
    
    @Scheduled(fixedRate = 300000) // 5분마다
    public void performPredictiveScaling() {
        // 과거 메트릭 수집
        List<MetricData> historicalData = metricsService.getHistoricalData(
            Duration.ofDays(7), Duration.ofMinutes(5));
        
        // 다음 시간대 부하 예측
        PredictionResult prediction = predictor.predictLoad(
            historicalData, Duration.ofMinutes(30));
        
        // 예측 결과에 따른 사전 스케일링
        if (prediction.getConfidence() > 0.8) {
            adjustReplicasBasedOnPrediction(prediction);
        }
    }
    
    private void adjustReplicasBasedOnPrediction(PredictionResult prediction) {
        int targetReplicas = calculateTargetReplicas(prediction);
        
        if (targetReplicas != getCurrentReplicas()) {
            scaleDeployment(targetReplicas);
            log.info("Predictive scaling: {} replicas", targetReplicas);
        }
    }
}
```

### 스케줄 기반 스케일링
```java
@Service
public class ScheduledScalingService {
    
    @Scheduled(cron = "0 30 8 * * MON-FRI") // 평일 오전 8:30
    public void scaleUpForBusinessHours() {
        scaleDeployment("web-app", 10);
        log.info("Scaled up for business hours");
    }
    
    @Scheduled(cron = "0 0 18 * * MON-FRI") // 평일 오후 6:00
    public void scaleDownAfterBusinessHours() {
        scaleDeployment("web-app", 3);
        log.info("Scaled down after business hours");
    }
    
    @Scheduled(cron = "0 0 22 * * *") // 매일 오후 10:00
    public void nightlyScaleDown() {
        List<String> deployments = Arrays.asList("web-app", "api-gateway", "worker");
        deployments.forEach(deployment -> scaleDeployment(deployment, 2));
        log.info("Nightly scale down completed");
    }
}
```

## 클러스터 오토스케일링

### Cluster Autoscaler 설정
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled
        - --balance-similar-node-groups
```

### Node Pool 설정
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
  region: us-west-2

nodeGroups:
  - name: worker-nodes
    instanceType: m5.large
    minSize: 2
    maxSize: 20
    desiredCapacity: 5
    
    asgSuspendProcesses:
      - AZRebalance
    
    instancesDistribution:
      maxPrice: 0.2
      instanceTypes: ["m5.large", "m5.xlarge", "m4.large"]
      onDemandBaseCapacity: 2
      onDemandPercentageAboveBaseCapacity: 20
      spotAllocationStrategy: "diversified"
```

## 백프레셔 처리

### 처리 용량 제한
```java
@Component
public class BackpressureHandler {
    private final Semaphore processingPermits;
    
    public BackpressureHandler() {
        this.processingPermits = new Semaphore(100); // 최대 100개 동시 처리
    }
    
    public void processWithBackpressure(Runnable processor) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            boolean acquired = processingPermits.tryAcquire(5, TimeUnit.SECONDS);
            
            if (!acquired) {
                meterRegistry.counter("backpressure.rejected").increment();
                throw new BackpressureException("Processing capacity exceeded");
            }
            
            try {
                processor.run();
                sample.stop(Timer.builder("process.duration")
                    .tag("status", "success").register(meterRegistry));
            } finally {
                processingPermits.release();
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ProcessingException("Processing interrupted", e);
        }
    }
}
```

### Circuit Breaker 패턴
```java
@Component
public class CircuitBreakerService {
    private final CircuitBreaker circuitBreaker;
    
    public CircuitBreakerService() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(20)
            .minimumNumberOfCalls(10)
            .build();
        
        this.circuitBreaker = CircuitBreaker.of("service", config);
    }
    
    public void processWithCircuitBreaker(Runnable processor) {
        Supplier<Void> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> {
                processor.run();
                return null;
            });
        
        try {
            decoratedSupplier.get();
        } catch (CallNotPermittedException e) {
            log.warn("Circuit breaker is OPEN, rejecting request");
            throw new ServiceUnavailableException("Service temporarily unavailable");
        }
    }
}
```

## 비용 최적화

### 스팟 인스턴스 활용
```java
@Component
public class SpotInstanceOptimizer {
    
    @Scheduled(fixedRate = 300000) // 5분마다
    public void optimizeSpotInstances() {
        Map<String, Double> spotPrices = getCurrentSpotPrices();
        List<InstanceRecommendation> recommendations = 
            calculateOptimalInstanceMix(spotPrices);
        
        updateNodeGroupConfiguration(recommendations);
    }
    
    private List<InstanceRecommendation> calculateOptimalInstanceMix(
            Map<String, Double> spotPrices) {
        
        return spotPrices.entrySet().stream()
            .map(entry -> {
                String instanceType = entry.getKey();
                Double price = entry.getValue();
                double efficiency = calculatePricePerformanceRatio(instanceType, price);
                return new InstanceRecommendation(instanceType, price, efficiency);
            })
            .filter(rec -> rec.getEfficiency() > 0.7)
            .sorted(Comparator.comparing(InstanceRecommendation::getEfficiency).reversed())
            .limit(3)
            .collect(Collectors.toList());
    }
}
```

### 리소스 사용률 분석
```java
@Service
public class ResourceOptimizationService {
    
    @Scheduled(cron = "0 0 6 * * *") // 매일 오전 6시
    public void generateOptimizationReport() {
        OptimizationReport report = analyzeResourceUtilization();
        List<OptimizationRecommendation> recommendations = 
            generateRecommendations(report);
        
        sendOptimizationReport(report, recommendations);
    }
    
    private OptimizationReport analyzeResourceUtilization() {
        Duration analysisWindow = Duration.ofDays(7);
        Map<String, ResourceUtilization> utilizationData = 
            metricsService.getResourceUtilization(analysisWindow);
        
        OptimizationReport.Builder builder = OptimizationReport.builder();
        
        utilizationData.forEach((deployment, utilization) -> {
            if (utilization.getAverageCpuUtilization() < 30) {
                builder.addUnderutilizedDeployment(deployment, "CPU");
            }
            if (utilization.getAverageMemoryUtilization() < 40) {
                builder.addUnderutilizedDeployment(deployment, "Memory");
            }
        });
        
        return builder.build();
    }
}
```

## Best Practices

### 메트릭 설계
- **Golden Signals**: Latency, Traffic, Errors, Saturation
- **비즈니스 메트릭**: 주문 수, 활성 사용자 수
- **인프라 메트릭**: CPU, 메모리, 네트워크, 디스크

### 스케일링 정책
- **안정화 윈도우**: 급격한 스케일링 방지
- **스케일업**: 빠르게 반응 (30초-1분)
- **스케일다운**: 천천히 반응 (5-10분)

### 리소스 할당
- **Request**: 최소 필요 리소스
- **Limit**: 최대 허용 리소스
- **Headroom**: 여유 공간 확보 (20-30%)

## Benefits and Challenges

### Benefits
- **탄력성**: 트래픽 변화에 자동 대응
- **비용 효율성**: 필요한 만큼만 리소스 사용
- **가용성**: 부하 분산으로 안정성 향상
- **운영 효율성**: 수동 개입 최소화

### Challenges
- **복잡성**: 스케일링 정책 설계 복잡
- **지연 시간**: 스케일링 반응 시간
- **메트릭 설계**: 적절한 메트릭 선택 어려움
- **상태 관리**: 스케일링 시 상태 보존 문제
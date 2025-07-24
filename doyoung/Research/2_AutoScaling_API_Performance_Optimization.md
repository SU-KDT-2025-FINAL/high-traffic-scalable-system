# 동적 오토스케일링 기반 API 서버 성능 최적화 가이드 (공식적 설명)

## 목차
1. [오토스케일링이란?](#오토스케일링이란)
2. [API 서버가 느려지는 이유](#api-서버가-느려지는-이유)
3. [기본 오토스케일링 방법](#기본-오토스케일링-방법)
4. [최신 오토스케일링 방법](#최신-오토스케일링-방법)
5. [많이 쓰는 고도화 방법](#많이-쓰는-고도화-방법)
6. [실제 적용 예시](#실제-적용-예시)
7. [모니터링과 알림](#모니터링과-알림)
8. [성능을 높이는 팁](#성능을-높이는-팁)

## 오토스케일링이란?

- 오토스케일링은 서버가 바쁠 때 자동으로 서버 수를 늘리고, 한가할 때는 줄여주는 기술이다.
- 예를 들어, 쇼핑몰에 사용자가 몰리면 서버가 자동으로 늘어나 서비스가 느려지지 않도록 한다.
- 반대로, 사용자가 적은 시간에는 서버 수를 줄여 비용을 절감할 수 있다.

## API 서버가 느려지는 이유

1. 사용자가 한꺼번에 많이 접속하면 서버의 부하가 증가한다.
2. 서버의 CPU, 메모리, 네트워크 대역폭이 한계에 도달하면 응답이 느려진다.
3. 이러한 상황에서 오토스케일링이 필요하다.

## 기본 오토스케일링 방법

- 서버의 CPU나 메모리 사용량이 일정 수준(예: 60%)을 넘으면 서버 인스턴스 수를 자동으로 늘린다.
- 사용량이 낮아지면 서버 인스턴스 수를 줄인다.
- 예시: 아래 설정은 CPU가 60%를 넘으면 서버가 최대 10개까지 늘어난다.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: classic-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60
```

## 최신 오토스케일링 방법

- CPU, 메모리뿐 아니라 네트워크 트래픽, 요청 수 등 다양한 기준으로 서버를 확장할 수 있다.
- 인공지능(AI) 기반으로 트래픽을 예측하여 미리 서버를 확장할 수도 있다.
- 예시: 아래 설정은 여러 기준을 동시에 적용하는 방법이다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: advanced-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

## 많이 쓰는 고도화 방법

- 실시간 트래픽을 기반으로 서버를 자동으로 확장 또는 축소한다.
- AI 기반 트래픽 예측, 예약 기반 스케일링 등 다양한 방법을 조합하여 사용한다.
- 서버 장애 발생 시 자동 복구(Auto Healing)를 적용한다.
- 서버가 바쁠 때만 임시로 서버를 빌려 쓰는 스팟 인스턴스, 서버리스 구조를 활용한다.
- 세션(로그인 정보 등)은 Redis와 같은 외부 저장소에 보관하여 서버 확장 시에도 문제가 발생하지 않도록 한다.

## 실제 적용 예시

### 1. Spring Boot + Kubernetes HPA
```java
@RestController
public class HealthController {
    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
```

### 2. Node.js + AWS Lambda (서버리스)
```js
exports.handler = async (event) => {
  // API 요청 처리
  return {
    statusCode: 200,
    body: JSON.stringify({ message: "Hello from Lambda!" })
  };
};
```

- Lambda는 요청이 많아지면 자동으로 서버 인스턴스가 늘어나고, 요청이 줄어들면 인스턴스 수도 줄어든다.

## 모니터링과 알림

- Prometheus, Grafana 등 모니터링 도구를 통해 서버 상태를 실시간으로 확인할 수 있다.
- 서버가 임계값을 초과하면 Slack 등 메신저로 알림을 받을 수 있다.

```yaml
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        api_url: 'https://hooks.slack.com/services/...'
```

## 성능을 높이는 팁

- 캐시(임시 저장소, CDN)를 적극적으로 활용하면 서버의 응답 속도가 빨라진다.
- 비동기 처리, 배치 처리 등으로 트래픽이 몰릴 때 부하를 분산할 수 있다.
- 데이터베이스 연결, HTTP 연결, 압축 등 다양한 설정을 최적화하면 성능이 향상된다.
- 무중단 배포, 롤백 자동화 등으로 서비스 중단 없이 서버를 교체할 수 있다.
- 오토스케일링이 너무 자주 발생하지 않도록 쿨다운(일시 정지) 시간을 적절히 설정한다.

## 모범 사례 요약

- 오토스케일링과 무상태 구조(Stateless)로 서버를 쉽게 확장 및 축소할 수 있다.
- AI 예측, 실시간, 예약 기반 등 다양한 스케일링 방법을 조합하면 효과적이다.
- 장애 발생 시 자동 복구, 실시간 모니터링, 무중단 배포 등 운영 안정성을 높여야 한다.
- 성능과 비용을 모두 고려한 운영 전략이 중요하다.

이러한 전략을 적용하면 많은 사용자가 동시에 접속해도 빠르고 안정적으로 서비스를 제공할 수 있다. 
# 3.3 분산 트랜잭션과 Saga 패턴

## Overview
분산 시스템에서는 여러 서비스에 걸친 트랜잭션 처리가 복잡해집니다. **Saga 패턴**은 분산 트랜잭션을 일련의 로컬 트랜잭션으로 분해하여 **최종 일관성(Eventually Consistency)**을 보장하는 패턴입니다.

## 분산 트랜잭션의 문제

### 전통적인 2PC 문제점
```
Two-Phase Commit (2PC):
1. Prepare Phase: 모든 참여자에게 준비 요청
2. Commit Phase: 모든 참여자가 준비되면 커밋

문제점:
- 단일 장애점 (Coordinator)
- 블로킹 프로토콜
- 성능 저하
- 확장성 제한
```

### CAP 정리와 트레이드오프
```
CAP Theorem:
- Consistency (일관성)
- Availability (가용성)  
- Partition Tolerance (분할 내성)

분산 시스템에서는 3개 중 2개만 선택 가능
→ 마이크로서비스는 보통 AP를 선택 (가용성 + 분할 내성)
```

## Saga 패턴 개념

### Choreography vs Orchestration

#### Choreography (분산 조정)
```
Order Service → Inventory Service → Payment Service → Shipping Service
     ↓               ↓                   ↓               ↓
Order Created → Stock Reserved → Payment Processed → Order Shipped
     ↓               ↓                   ↓               ↓
각 서비스가 다음 단계를 직접 호출
```

#### Orchestration (중앙 조정)
```
Saga Orchestrator
    ├─ Step 1: Reserve Stock
    ├─ Step 2: Process Payment  
    ├─ Step 3: Ship Order
    └─ Handle Compensation
```

## Choreography-based Saga 구현

### 이벤트 기반 Saga
```java
// Order Service
@Service
@Transactional
public class OrderSagaService {
    
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;
    
    public void processOrder(CreateOrderRequest request) {
        try {
            // 1. 주문 생성
            Order order = createOrder(request);
            orderRepository.save(order);
            
            // 2. 재고 예약 이벤트 발행
            InventoryReservationEvent event = InventoryReservationEvent.builder()
                .orderId(order.getId())
                .customerId(order.getCustomerId())
                .items(order.getItems())
                .sagaId(UUID.randomUUID().toString())
                .build();
            
            eventPublisher.publish("inventory.reserve", event);
            
        } catch (Exception e) {
            log.error("Failed to process order: {}", request.getOrderId(), e);
            throw e;
        }
    }
    
    @EventListener
    public void handleInventoryReserved(InventoryReservedEvent event) {
        try {
            // 주문 상태 업데이트
            Order order = orderRepository.findById(event.getOrderId())
                .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
            
            order.updateStatus(OrderStatus.INVENTORY_RESERVED);
            orderRepository.save(order);
            
            // 결제 처리 이벤트 발행
            PaymentProcessingEvent paymentEvent = PaymentProcessingEvent.builder()
                .orderId(event.getOrderId())
                .amount(order.getTotalAmount())
                .customerId(order.getCustomerId())
                .sagaId(event.getSagaId())
                .build();
            
            eventPublisher.publish("payment.process", paymentEvent);
            
        } catch (Exception e) {
            // 보상 트랜잭션 시작
            compensateInventoryReservation(event);
        }
    }
    
    @EventListener
    public void handlePaymentProcessed(PaymentProcessedEvent event) {
        // 배송 시작 이벤트 발행
        ShippingStartEvent shippingEvent = ShippingStartEvent.builder()
            .orderId(event.getOrderId())
            .shippingAddress(getShippingAddress(event.getOrderId()))
            .sagaId(event.getSagaId())
            .build();
        
        eventPublisher.publish("shipping.start", shippingEvent);
    }
    
    @EventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        // 보상 트랜잭션: 재고 예약 취소
        InventoryCancellationEvent compensationEvent = InventoryCancellationEvent.builder()
            .orderId(event.getOrderId())
            .reason("Payment failed")
            .sagaId(event.getSagaId())
            .build();
        
        eventPublisher.publish("inventory.cancel", compensationEvent);
        
        // 주문 상태 업데이트
        updateOrderStatus(event.getOrderId(), OrderStatus.PAYMENT_FAILED);
    }
}
```

### 재고 서비스 구현
```java
@Service
@Transactional
public class InventorySagaService {
    
    private final InventoryRepository inventoryRepository;
    private final EventPublisher eventPublisher;
    
    @EventListener
    public void handleInventoryReservation(InventoryReservationEvent event) {
        try {
            // 재고 확인 및 예약
            for (OrderItem item : event.getItems()) {
                Inventory inventory = inventoryRepository.findByProductId(item.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
                
                if (inventory.getAvailableQuantity() < item.getQuantity()) {
                    throw new InsufficientStockException(item.getProductId());
                }
                
                // 재고 예약
                inventory.reserve(item.getQuantity());
                inventoryRepository.save(inventory);
            }
            
            // 재고 예약 성공 이벤트 발행
            InventoryReservedEvent successEvent = InventoryReservedEvent.builder()
                .orderId(event.getOrderId())
                .items(event.getItems())
                .sagaId(event.getSagaId())
                .timestamp(Instant.now())
                .build();
            
            eventPublisher.publish("inventory.reserved", successEvent);
            
        } catch (Exception e) {
            // 재고 예약 실패 이벤트 발행
            InventoryReservationFailedEvent failedEvent = InventoryReservationFailedEvent.builder()
                .orderId(event.getOrderId())
                .reason(e.getMessage())
                .sagaId(event.getSagaId())
                .build();
            
            eventPublisher.publish("inventory.reservation.failed", failedEvent);
        }
    }
    
    @EventListener
    public void handleInventoryCancellation(InventoryCancellationEvent event) {
        try {
            // 예약된 재고 해제
            Order order = getOrderDetails(event.getOrderId());
            
            for (OrderItem item : order.getItems()) {
                Inventory inventory = inventoryRepository.findByProductId(item.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
                
                inventory.cancelReservation(item.getQuantity());
                inventoryRepository.save(inventory);
            }
            
            log.info("Inventory reservation cancelled for order: {}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("Failed to cancel inventory reservation for order: {}", 
                event.getOrderId(), e);
        }
    }
}
```

## Orchestration-based Saga 구현

### Saga Orchestrator
```java
@Component
public class OrderSagaOrchestrator {
    
    private final SagaManager sagaManager;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    @SagaOrchestrationStart
    public void processOrder(CreateOrderCommand command) {
        String sagaId = UUID.randomUUID().toString();
        
        SagaInstance saga = sagaManager.begin("order-saga", sagaId);
        
        try {
            saga.addStep("reserve-inventory")
                .invokeParticipant("inventory-service")
                .withCommand(new ReserveInventoryCommand(command.getOrderId(), command.getItems()))
                .withCompensation(new CancelInventoryCommand(command.getOrderId()))
                
                .addStep("process-payment")
                .invokeParticipant("payment-service")
                .withCommand(new ProcessPaymentCommand(command.getOrderId(), command.getAmount()))
                .withCompensation(new RefundPaymentCommand(command.getOrderId()))
                
                .addStep("start-shipping")
                .invokeParticipant("shipping-service")
                .withCommand(new StartShippingCommand(command.getOrderId(), command.getShippingAddress()))
                .withCompensation(new CancelShippingCommand(command.getOrderId()))
                
                .execute();
                
        } catch (SagaExecutionException e) {
            saga.compensate();
            throw e;
        }
    }
}
```

### Saga Manager 구현
```java
@Service
public class SagaManager {
    
    private final SagaInstanceRepository sagaRepository;
    private final SagaStepExecutor stepExecutor;
    
    public SagaInstance begin(String sagaType, String sagaId) {
        SagaInstance instance = SagaInstance.builder()
            .sagaId(sagaId)
            .sagaType(sagaType)
            .status(SagaStatus.STARTED)
            .startedAt(Instant.now())
            .build();
        
        sagaRepository.save(instance);
        
        return instance;
    }
    
    public void executeStep(SagaInstance saga, SagaStep step) {
        try {
            // 단계 실행
            stepExecutor.execute(step);
            
            // 성공 시 상태 업데이트
            step.markAsCompleted();
            updateSagaProgress(saga);
            
        } catch (Exception e) {
            // 실패 시 보상 트랜잭션 시작
            step.markAsFailed(e.getMessage());
            startCompensation(saga, step);
        }
    }
    
    private void startCompensation(SagaInstance saga, SagaStep failedStep) {
        saga.setStatus(SagaStatus.COMPENSATING);
        sagaRepository.save(saga);
        
        // 실행된 단계들을 역순으로 보상
        List<SagaStep> completedSteps = saga.getCompletedSteps();
        Collections.reverse(completedSteps);
        
        for (SagaStep step : completedSteps) {
            try {
                stepExecutor.compensate(step);
                step.markAsCompensated();
                
            } catch (Exception e) {
                log.error("Compensation failed for step: {}", step.getStepName(), e);
                // 보상 실패 처리 (수동 개입 필요)
                alertService.sendCompensationFailureAlert(saga, step, e);
            }
        }
        
        saga.setStatus(SagaStatus.COMPENSATED);
        sagaRepository.save(saga);
    }
}
```

### 단계 실행기
```java
@Component
public class SagaStepExecutor {
    
    private final Map<String, SagaParticipant> participants;
    
    public void execute(SagaStep step) {
        SagaParticipant participant = participants.get(step.getParticipantName());
        if (participant == null) {
            throw new SagaParticipantNotFoundException(step.getParticipantName());
        }
        
        try {
            SagaStepResult result = participant.execute(step.getCommand());
            
            if (result.isSuccess()) {
                step.setResult(result);
                log.info("Saga step executed successfully: {}", step.getStepName());
            } else {
                throw new SagaStepExecutionException(result.getErrorMessage());
            }
            
        } catch (Exception e) {
            log.error("Saga step execution failed: {}", step.getStepName(), e);
            throw new SagaStepExecutionException(e.getMessage(), e);
        }
    }
    
    public void compensate(SagaStep step) {
        SagaParticipant participant = participants.get(step.getParticipantName());
        if (participant == null) {
            log.warn("Participant not found for compensation: {}", step.getParticipantName());
            return;
        }
        
        try {
            participant.compensate(step.getCompensationCommand());
            log.info("Saga step compensated successfully: {}", step.getStepName());
            
        } catch (Exception e) {
            log.error("Saga step compensation failed: {}", step.getStepName(), e);
            throw new SagaCompensationException(e.getMessage(), e);
        }
    }
}
```

## 보상 트랜잭션 설계

### 보상 액션 구현
```java
@Service
public class PaymentCompensationService {
    
    private final PaymentRepository paymentRepository;
    private final RefundService refundService;
    
    public void compensatePayment(CompensatePaymentCommand command) {
        try {
            Payment payment = paymentRepository.findByOrderId(command.getOrderId())
                .orElse(null);
            
            if (payment == null) {
                log.info("No payment found for order: {}, compensation not needed", 
                    command.getOrderId());
                return;
            }
            
            if (payment.getStatus() == PaymentStatus.COMPLETED) {
                // 결제 완료된 경우 환불 처리
                Refund refund = refundService.processRefund(
                    payment.getPaymentId(), 
                    payment.getAmount(),
                    "Order cancelled - Saga compensation"
                );
                
                payment.addRefund(refund);
                payment.setStatus(PaymentStatus.REFUNDED);
                paymentRepository.save(payment);
                
                log.info("Payment compensated with refund: {}", refund.getRefundId());
                
            } else {
                // 결제 진행 중인 경우 취소
                payment.setStatus(PaymentStatus.CANCELLED);
                paymentRepository.save(payment);
                
                log.info("Payment cancelled: {}", payment.getPaymentId());
            }
            
        } catch (Exception e) {
            log.error("Payment compensation failed for order: {}", command.getOrderId(), e);
            throw new CompensationException("Failed to compensate payment", e);
        }
    }
}
```

### 멱등성 보장
```java
@Service
public class IdempotentSagaService {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean executeIdempotently(String sagaId, String stepName, Runnable action) {
        String lockKey = "saga:lock:" + sagaId + ":" + stepName;
        String executionKey = "saga:executed:" + sagaId + ":" + stepName;
        
        // 이미 실행되었는지 확인
        if (redisTemplate.hasKey(executionKey)) {
            log.info("Step already executed: {} - {}", sagaId, stepName);
            return true;
        }
        
        // 분산 락 획득
        Boolean lockAcquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "locked", Duration.ofMinutes(5));
        
        if (!lockAcquired) {
            log.warn("Failed to acquire lock for saga step: {} - {}", sagaId, stepName);
            return false;
        }
        
        try {
            // 다시 한 번 실행 여부 확인 (double-checked locking)
            if (redisTemplate.hasKey(executionKey)) {
                return true;
            }
            
            // 액션 실행
            action.run();
            
            // 실행 완료 표시
            redisTemplate.opsForValue().set(executionKey, "completed", Duration.ofDays(7));
            
            return true;
            
        } finally {
            redisTemplate.delete(lockKey);
        }
    }
}
```

## Saga 모니터링과 관리

### Saga 상태 추적
```java
@RestController
public class SagaMonitoringController {
    
    private final SagaInstanceRepository sagaRepository;
    private final SagaMetricsService metricsService;
    
    @GetMapping("/sagas/{sagaId}")
    public SagaInstanceDto getSagaStatus(@PathVariable String sagaId) {
        SagaInstance saga = sagaRepository.findBySagaId(sagaId)
            .orElseThrow(() -> new SagaNotFoundException(sagaId));
        
        return SagaInstanceDto.from(saga);
    }
    
    @GetMapping("/sagas")
    public Page<SagaInstanceDto> getSagas(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) SagaStatus status) {
        
        Pageable pageable = PageRequest.of(page, size);
        
        Page<SagaInstance> sagas;
        if (status != null) {
            sagas = sagaRepository.findByStatus(status, pageable);
        } else {
            sagas = sagaRepository.findAll(pageable);
        }
        
        return sagas.map(SagaInstanceDto::from);
    }
    
    @GetMapping("/sagas/metrics")
    public SagaMetrics getSagaMetrics() {
        return metricsService.generateMetrics();
    }
    
    @PostMapping("/sagas/{sagaId}/retry")
    public ResponseEntity<String> retrySaga(@PathVariable String sagaId) {
        try {
            sagaManager.retry(sagaId);
            return ResponseEntity.ok("Saga retry initiated");
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Failed to retry saga: " + e.getMessage());
        }
    }
}
```

### 실패한 Saga 처리
```java
@Service
public class SagaRecoveryService {
    
    @Scheduled(fixedRate = 300000) // 5분마다
    public void recoverFailedSagas() {
        List<SagaInstance> failedSagas = sagaRepository.findByStatusAndUpdatedAtBefore(
            SagaStatus.FAILED, 
            Instant.now().minus(Duration.ofMinutes(10))
        );
        
        for (SagaInstance saga : failedSagas) {
            try {
                if (saga.getRetryCount() < MAX_RETRY_COUNT) {
                    retrySaga(saga);
                } else {
                    moveToManualIntervention(saga);
                }
            } catch (Exception e) {
                log.error("Failed to recover saga: {}", saga.getSagaId(), e);
            }
        }
    }
    
    private void retrySaga(SagaInstance saga) {
        saga.incrementRetryCount();
        saga.setStatus(SagaStatus.RETRYING);
        sagaRepository.save(saga);
        
        // 실패한 단계부터 재시작
        SagaStep failedStep = saga.getFailedStep();
        if (failedStep != null) {
            sagaManager.retryFromStep(saga, failedStep);
        }
    }
    
    private void moveToManualIntervention(SagaInstance saga) {
        saga.setStatus(SagaStatus.REQUIRES_MANUAL_INTERVENTION);
        sagaRepository.save(saga);
        
        // 관리자 알림
        alertService.sendSagaManualInterventionAlert(saga);
    }
}
```

## Best Practices

### Saga 설계 원칙
- **원자성**: 각 로컬 트랜잭션은 원자적이어야 함
- **일관성**: 전체 Saga 완료 시 비즈니스 규칙 만족
- **격리성**: 동시 실행되는 Saga 간 격리
- **지속성**: Saga 상태는 지속적으로 저장

### 보상 트랜잭션 설계
- **멱등성**: 여러 번 실행해도 같은 결과
- **되돌리기 가능**: 원래 상태로 복원 가능
- **시맨틱 보상**: 비즈니스 의미를 고려한 보상

### 에러 처리
- **재시도 정책**: 일시적 오류에 대한 재시도
- **타임아웃**: 무한 대기 방지
- **모니터링**: 실시간 Saga 상태 추적

## Benefits and Challenges

### Benefits
- **확장성**: 서비스별 독립적 확장
- **가용성**: 일부 서비스 장애가 전체에 미치는 영향 최소화
- **유연성**: 비즈니스 로직 변경에 유연하게 대응
- **성능**: 블로킹 없는 비동기 처리

### Challenges
- **복잡성**: 분산 트랜잭션 로직의 복잡성
- **디버깅**: 여러 서비스에 걸친 트랜잭션 추적 어려움
- **일관성**: 최종 일관성 모델의 복잡성
- **보상 설계**: 적절한 보상 트랜잭션 설계 어려움
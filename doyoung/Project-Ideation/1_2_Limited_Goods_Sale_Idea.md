# 핵심 서비스 아이디어
- 한정 수량 상품의 공정하고 효율적인 판매를 위한 고성능 시스템 설계
- 동시 주문 폭주 상황에서 재고 오차 및 데이터 불일치 방지
- 주문, 결제, 알림 등 각 기능을 독립적으로 확장 가능한 MSA 구조로 분리
- 장애 발생 시에도 데이터 정합성과 사용자 경험을 보장하는 보상 트랜잭션 적용

## MSA 서비스 구성
- Product-Service: 상품 정보 및 재고 관리 (분산 락을 이용한 동시성 제어)
- Order-Service: 주문 요청을 받아 대기열(Queue)에 등록하고 순차 처리
- Payment-Service: 결제 처리 연동
- Notification-Service: 주문 성공/실패/대기 등 상태 알림

### MSA 시스템 유형
- Orchestration 기반 Saga 패턴: Order-Service가 오케스트레이터(지휘자) 역할을 함
  - 주문 요청이 들어오면 (1)Product-Service의 재고 확인/차감 → (2)Payment-Service의 결제 요청 → (3)주문 완료 순서로 각 서비스를 순차적으로 호출
  - 중간에 결제 실패 등 문제가 발생하면 재고를 다시 복구시키는 보상 트랜잭션(Compensating Transaction)을 호출하여 데이터 정합성을 보장

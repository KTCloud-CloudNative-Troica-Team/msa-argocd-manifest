# ADR-0006: order state machine 7-state 확장

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

TROICA_SPEC.md §0의 시연 시나리오는 주문 → 결제 → 배송 흐름을 포함했으나, 모노레포의 `OrderLineItemStatus` enum은 단순한 2-state (PENDING, INVENTORY_RESERVED)뿐. 결제/배송에 해당하는 상태가 없어서 시연 시나리오 구현 불가.

또한:
- order-service의 `OrderTerminalEvent` 발행 로직 부재 → `order.confirmed` / `order.cancelled` Kafka 토픽에 producer 없음.
- 관리자가 주문 상태를 변경할 admin endpoint도 없음.

세 가지 부재가 동시 발견. 어떻게 채울지 결정 필요.

## 결정

order state machine을 **7-state**로 확장하고, terminal event 발행과 admin endpoint를 함께 구현한다.

### 상태 전이도

```
                                   ┌──────────────┐
                              ┌──→│  CONFIRMED   │ (정상 종료)
                              │   └──────────────┘
PENDING                       │
  ↓                           │
INVENTORY_RESERVED  ──→ PAID  ──→  SHIPPED  ──→ CONFIRMED
  ↓ (재고 부족)                                         
FAILED                        │   ┌──────────────┐
  ↓ (관리자 / 사용자)         └──→│  CANCELLED   │ (관리자 취소)
CANCELLED                         └──────────────┘
```

7 상태:
1. **PENDING** — 주문 생성. inventory 예약 시도 전.
2. **INVENTORY_RESERVED** — 재고 예약 완료.
3. **PAID** — 결제 완료 (시연: admin endpoint로 시뮬레이션).
4. **SHIPPED** — 배송 시작 (시연: admin endpoint로 시뮬레이션).
5. **CONFIRMED** — 주문 완료. terminal.
6. **FAILED** — 재고 부족으로 실패. terminal.
7. **CANCELLED** — 관리자/사용자 취소. terminal.

### 부속 구현

- **`OrderStatus.checkTransitive(next)`**: 모든 상태 전이를 enum 안에 명시. 잘못된 전이 → 도메인 예외.
- **`OrderStatus.isPublishableTerminal()`**: CONFIRMED / CANCELLED만 true (FAILED는 발행 안 함 — D7 일관성).
- **`OrderStatusOutbox` 신규 entity**: 상태 변경 시 outbox row 생성. publisher가 polling.
- **`OrderTerminalEventKafkaPublisher`**: outbox poll → JSON 직렬화 → Kafka publish (ADR-0003 wire format).
- **Admin endpoint**: `POST /admin/v1/orders/{id}/status` — 상태 강제 변경 (시연 시 결제/배송 시뮬레이션용).

## 대안

| 옵션 | 거부 사유 |
|---|---|
| (a) state machine 확장 (채택) | 시연 시나리오 완전 구현. 발표 콘텐츠 풍부. |
| (b) INVENTORY_RESERVED = CONFIRMED로 단순화 (결제/배송 생략) | 시연 임팩트 약화. 발표에서 보여줄 게 없음. |
| (c) 별도 작업으로 미루기 | Phase 4 안에서 처리 가능한 범위. 미루면 발표 시점에 부재. |

## 결과

**긍정**:
- 시연 시나리오 (주문 → 재고 예약 → 결제 → 배송 → 완료) 완전 구현.
- terminal event (`order.confirmed`, `order.cancelled`) Kafka 발행으로 다른 서비스 (analytics, alerting 등) subscribe 가능 (D7).
- admin endpoint로 demo 시 흐름 시뮬레이션 용이.

**부정 / 후속**:
- enum 전이 검증 로직 복잡도 증가 — 잘못된 전이 시도가 명확한 예외로 노출되므로 디버깅은 오히려 쉬워짐.
- 결제/배송이 실제 구현 아닌 simulation (시연용). 운영 시에는 실제 PG / 배송 시스템 연동 필요.

**관련 후속 작업**:
- R-21 (Phase 4 #3): `OrderStatusOutbox` + `OrderTerminalEventKafkaPublisher` + admin endpoint 구현.
- D7: `order.confirmed`/`order.cancelled` 토픽 발행 유지 (consumer 없어도). ADR-0008 참조.

## 관련

- BACKLOG: Q6, R-21, D5
- SPEC: §0 시연 시나리오, §9 페이로드
- ADR-0003 (Kafka wire format JSON)
- ADR-0004 (Kafka 토픽명 표준)
- ADR-0008 (terminal events 발행 유지)

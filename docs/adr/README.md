# Architecture Decision Records (ADR)

Troica 팀의 주요 아키텍처 결정 기록. 시간 순으로 번호 매김, 각 ADR은 독립적으로 읽을 수 있도록 컨텍스트 포함.

## 형식 (MADR 4.0 기반)

```
- 상태: Proposed / Accepted / Deprecated / Superseded by ADR-XXX
- 결정일: YYYY-MM-DD
- 결정자: Troica 팀 (3인 합의)

## 컨텍스트
의사결정이 필요했던 배경 + 제약.

## 결정
채택한 옵션 + 근거.

## 대안
검토한 다른 옵션들 + 거부 사유.

## 결과
긍정 영향 / 부정 영향 / 후속 작업.

## 관련
BACKLOG R-XX, TROUBLESHOOTING 항목, SPEC §X, 다른 ADR.
```

## 목록

| ADR | 제목 | 상태 |
|---|---|---|
| [ADR-0001](./0001-polyrepo-with-auth-service.md) | Polyrepo 구조 + auth-service 분리 + notification 폐기 | Accepted |
| [ADR-0002](./0002-client-libraries-distribution.md) | client-redis는 JitPack, client-ses 제거 | Accepted |
| [ADR-0003](./0003-kafka-wire-format-json.md) | Kafka 직렬화 = JSON 유지 (Protobuf 마이그레이션 보류) | Accepted |
| [ADR-0004](./0004-kafka-topic-naming.md) | Kafka 토픽명은 SPEC 표준 (`order.pending` 등) | Accepted |
| [ADR-0005](./0005-api-gateway-bff-with-cloud-gateway.md) | api-gateway = BFF (REST→gRPC) + Spring Cloud Gateway routes 혼합 | Accepted |
| [ADR-0006](./0006-order-state-machine-extension.md) | order state machine 7-state 확장 | Accepted |
| [ADR-0007](./0007-inventory-event-sourcing-batch-worker.md) | inventory = Event Sourcing + Spring Batch worker | Accepted |
| [ADR-0008](./0008-order-terminal-events.md) | order.confirmed/order.cancelled 토픽 발행 유지 (consumer 없어도) | Accepted |
| [ADR-0009](./0009-api-gateway-istio-mesh-collaboration.md) | api-gateway (BFF + SC Gateway) ↔ Istio Service Mesh 책임 분담 | Accepted |
| [ADR-0010](./0010-security-alerting-strategy.md) | 보안 알림 채널 `#security-report` 분리 전략 | Accepted |

## 가이드

- **새 ADR 추가 시**: 다음 번호 + 본 README 표에 한 줄 추가.
- **ADR 변경 시**: 새 ADR로 작성하고 기존 ADR 상태를 `Superseded by ADR-XXX`로.
- **결정 폐기 시**: 기존 ADR을 `Deprecated`로 표시하고 사유를 ADR에 추가.
- 회의 후 결정 사항을 ADR로 남기면 추후 신규 멤버 onboarding + 회고 시 컨텍스트 복원 용이.

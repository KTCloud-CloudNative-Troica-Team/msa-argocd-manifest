# ADR-0007: inventory = Event Sourcing + Spring Batch worker

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

모노레포의 `inventory-event` 모듈 구조가 의도를 알기 어려웠음:

- 테이블 `inventory_event`에 `processed: Boolean` 플래그 컬럼 존재 → "Outbox 패턴인가?"
- 그러나 inventory의 현재 재고 값은 `inventory_event` 자체에서 누적 계산 → "Event Sourcing인가?"
- 둘이 섞여 있음. SPEC §0 / §9에는 두 패턴 모두 언급되지만 inventory가 어디에 속하는지 불명확.

또한 모노레포에는 worker 분기 없음. order-service는 Phase 3에서 `@Profile("worker")` + `@Scheduled` 폴러로 worker를 명확히 분리했는데, inventory도 같은 패턴 필요.

## 결정

inventory는 **Event Sourcing 패턴 + Spring Batch worker**로 명확히 정의한다.

### Event Sourcing
- `inventory_event` 테이블은 **append-only event store** (immutable).
- 도메인 이벤트: `InventoryReserved`, `InventoryReleased`, `InventoryReplenished` 등.
- 현재 재고 = 이벤트들의 누적 적용 (snapshot 캐시는 후속 최적화).

### Spring Batch worker
- Kafka에서 `order.pending` 수신 → inventory 예약 시도 → `inventory_event` row append → `order.inventory-reserved` 발행.
- 이 처리는 **`@Profile("worker")` + Spring Batch Job/Step**으로 구현.
- chunk-oriented `ItemReader` (Kafka topic) / `ItemProcessor` (도메인 로직) / `ItemWriter` (event store append).
- inventory-service 본체 Pod는 gRPC 서버 (조회) 담당. worker Pod는 위 Batch Job 담당.

### `processed` flag 처리
- 기존 모놀레포의 `processed: Boolean`은 **Event Sourcing 모델에서는 부적합** (이벤트는 immutable, processed 상태는 별도 read-side 책임).
- polyrepo 이관 시 `processed` 컬럼은 일관성을 위해 유지하되, 미사용 + 향후 마이그레이션에서 제거 검토.

## 대안

| 옵션 | 거부 사유 |
|---|---|
| (a) Outbox 패턴으로 SPEC 정정 | 현재 재고 값을 `inventory_event` 누적으로 계산하는 코드와 모순. event store가 outbox 역할까지 하면 single responsibility 깨짐. |
| (b) Event Sourcing 패턴 (채택) | 모노레포 코드의 누적 계산 + append-only 의도와 일치. Spring Batch와 자연스러운 페어링. |
| (c) 다른 패턴 (CQRS + 별도 store) | PoC 단계 over-engineering. 추후 read-side 최적화 시 분리 검토. |

worker 분기:
| 옵션 | 거부 사유 |
|---|---|
| Spring `@Scheduled` 단순 폴러 (order처럼) | Kafka 메시지 처리에는 Spring Batch의 chunk-oriented 패러다임이 더 적합 (에러 처리, restart, skip 정책). |
| Spring Batch (채택) | Job/Step 메타데이터로 운영 가시성 좋음. 시연 시 batch run history 보여줄 수 있음. |

## 결과

**긍정**:
- inventory의 책임 명확: event store + worker.
- 시연 시 batch job 실행 → event 누적 → 재고 변화를 코드 + DB 양쪽에서 보여줄 수 있음.
- 향후 snapshot 캐시 / CQRS read-side 추가 시 자연스러운 확장.

**부정 / 후속**:
- Spring Batch 의존성 추가 (`spring-boot-starter-batch`).
- worker가 Batch Job step 단위로 동작 → 처리 latency 증가 (chunk size에 비례). 실시간성 떨어지는 점은 시연 시 단점.
- **ItemWriter Kotlin lambda 타입 추론 실패**: SAM 변환이 Spring Batch generic wildcard와 충돌. `object : ItemWriter<...>` 명시 필요 (TROUBLESHOOTING §1.5).
- **`:inventory-event` transitive 미인식**: 명시적 `implementation(project(":inventory-event"))` 필요 (TROUBLESHOOTING §1.6).

**관련 후속 작업**:
- R-22 (Phase 4 #4): event store + worker 구현.
- worker는 별도 K8s Deployment (`profile=worker`, `spring.main.web-application-type=none`, `grpc.server.port=-1`).

## 관련

- BACKLOG: Q7, R-22, D5
- SPEC: §0, §1.1, §9
- TROUBLESHOOTING: §1.5 (ItemWriter lambda), §1.6 (inventory-event transitive)
- ADR-0003 (Kafka wire format JSON), ADR-0004 (토픽명 표준) — worker가 producer/consumer

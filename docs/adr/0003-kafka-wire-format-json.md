# ADR-0003: Kafka wire format = JSON 유지 (Protobuf 마이그레이션 보류)

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

`msa-common-libs:events` 모듈에는 Protobuf 정의가 있다 (ADR-0010 — TBD, 이벤트 스키마 = Protobuf 단독). 그러나 모노레포의 기존 Kafka producer/consumer 구현은 **JSON serializer** (`KafkaJsonSchemaSerializer`)를 사용 중.

polyrepo 이관 시 wire format을 어떻게 할지 결정 필요:

- order-service가 Kafka publish (Outbox publisher).
- inventory-service가 Kafka consume (event sourcing 입력).
- 두 서비스가 동시에 wire format을 바꿔야만 양립 가능 — 한쪽만 바꾸면 메시지 deserialization 실패.

Phase 3 시점에 order-service만 polyrepo 이관, inventory-service는 아직 모노레포. 동시 변경 불가.

## 결정

**Kafka wire format은 JSON 유지**. Protobuf 마이그레이션은 Phase 4 이후로 보류.

- `events` 모듈은 Protobuf 정의 + 생성된 Kotlin/Java 클래스를 제공하되, **wire 직렬화는 JSON으로 우회**.
- order-service publisher는 Protobuf message → Java POJO → Jackson JSON.
- inventory-service consumer는 Jackson JSON → Java POJO → 도메인 변환.

## 대안

| 옵션 | 거부 사유 |
|---|---|
| (a) JSON 유지 (채택) | order/inventory 동시 마이그레이션 부담 회피. Phase 3 스코프 폭주 방지. |
| (b) Phase 3에 Protobuf 동시 마이그레이션 | order/inventory 양쪽 ~5 파일씩 수정 + 통합 테스트. Phase 3 일정 초과. |
| (c) order만 Protobuf로 보내고 inventory는 fallback | wire 호환성 깨짐. 운영 risk 큼. |

## 결과

**긍정**:
- Phase 3 일정 안에서 order-service polyrepo 이전 완료.
- inventory-service polyrepo 이전 (Phase 4)도 동일 wire format 유지로 단순.
- 디버깅 시 JSON 메시지를 사람이 직접 읽을 수 있음 (kafka-console-consumer).

**부정 / 후속**:
- **타입 안전성 손실**: JSON은 schema-less. consumer가 field 누락 / type mismatch를 런타임에 발견.
- **schema evolution 가드 부재**: Protobuf의 필드번호 불변 같은 safeguard 없음. 송수신측이 별도 규약 필요.
- **Protobuf 정의는 있는데 wire로는 안 씀** — 일관성 깨짐. proto 정의가 단순 "도메인 객체 schema" 역할로만 사용.

**후속 작업 (BACKLOG R-13)**: Phase 4 이후 order/inventory 동시 마이그레이션. wire format 전환 시 호환 기간 동안 dual publish (JSON + Protobuf 양쪽) 검토.

## 관련

- BACKLOG: R-13, Q4
- SPEC: §9.3 페이로드 스키마
- ADR-0010 (TBD) 이벤트 스키마 = Protobuf 단독 (스키마 정의 자체는 Protobuf, wire만 JSON)
- ADR-0004 (Kafka 토픽명 SPEC 표준) — 본 ADR과 함께 Kafka 운영 표준 형성

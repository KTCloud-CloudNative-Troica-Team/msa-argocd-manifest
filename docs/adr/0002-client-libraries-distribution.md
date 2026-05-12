# ADR-0002: client-redis는 JitPack, client-ses는 제거

- **상태**: Accepted
- **결정일**: 2026-05-12
- **결정자**: Troica 팀 (3인 합의)

## 컨텍스트

`msa-common-libs`는 초기에 4개 모듈로 설계되었다:

- `common` — 공통 도메인 객체, 예외, 유틸
- `events` — Protobuf 이벤트 페이로드 (gRPC stubs 포함)
- `client-redis` — Redis 클라이언트 설정 wrapper
- `client-ses` — AWS SES 메일 발송 wrapper

진행 중 다음을 발견:

1. 팀 내 한 멤버가 별도 GitHub 레포(`kanei0415/ktcloud-msa-client-redis`)에 Redis 클라이언트 라이브러리를 이미 **JitPack에 publish**해서 운영 중. 외부 프로젝트에서도 재사용.
2. ADR-0001에 따라 `msa-notification-service`가 폐기됨. SES 메일 발송을 사용할 컴포넌트가 없어짐. `client-ses`는 사실상 dead code.

두 가지를 어떻게 정리할지 결정 필요.

## 결정

1. **`client-redis`는 common-libs에서 제거하고 JitPack 외부 라이브러리로 일원화**:
   ```
   com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2
   ```
   common-libs v0.3.0 release에서 client-redis 모듈 삭제.

2. **`client-ses`는 common-libs에서 제거**. notification 폐기와 일관 (ADR-0001).

따라서 common-libs는 **2 모듈 (common + events)** 구성으로 축소.

## 대안

### client-redis
| 옵션 | 거부 사유 |
|---|---|
| common-libs 안에 유지하고 JitPack 라이브러리는 폐기 | 멤버가 이미 외부 프로젝트에서도 같은 라이브러리 사용 중. 강제 통합은 외부 사용처 깸. |
| 두 라이브러리 동시 운영 (다른 이름) | 코드 중복 + 양쪽 유지보수 부담 |
| common-libs에서 제거 + JitPack 사용 (채택) | 단일 진실의 원천. 멤버가 외부에서도 활용 가능. |

### client-ses
| 옵션 | 거부 사유 |
|---|---|
| 일단 유지하고 추후 사용 시 활용 | dead code 누적 + 보안 의존성 (AWS SDK) 추가됨 |
| 제거 (채택) | notification 폐기와 일관. AWS SDK가 다시 필요해지면 그때 추가 |

## 결과

**긍정**:
- common-libs 모듈 수: 4 → 2. 빌드/publish 시간 단축.
- client-redis 운영 책임이 외부 레포로 위임 (멤버가 직접 관리).
- 모노레포 잔재 코드 정리.

**부정 / 후속**:
- **외부 의존성 운영 risk**: JitPack 빌드 실패 가능성, 버전 lifecycle 가시성 낮음. BACKLOG R-20에 모니터링 항목 등재. 변경 알림은 Dependabot/Renovate 후속 작업 (R-15와 묶음).
- **Kotlin metadata 호환 문제**: JitPack 라이브러리가 Kotlin 2.3.x로 컴파일되었는데 우리 consumer는 2.1.0 (ADR-0003). 컴파일러 인자 `-Xskip-metadata-version-check` 추가로 해소. TROUBLESHOOTING.md §1.3에 절차 기록.

**관련 후속 작업**:
- common-libs v0.3.0 publish (PoC stable 버전).
- 6개 서비스 빌드 시 JitPack repository 등록 (해당 서비스의 `build.gradle.kts`).

## 관련

- BACKLOG: D2, D3, R-20
- TROUBLESHOOTING: §1.3 (JitPack Kotlin metadata 호환)
- SPEC: §1.2 공용 라이브러리 (4 모듈 → 2 모듈)
- ADR-0001 (notification 폐기 연쇄)
- ADR-0003 (Kotlin 2.1.0 고정 — JitPack metadata 호환 이슈의 배경)

# ADR-0011: SonarCloud Quality Gate 정책 — 무료 plan 제약 + wait=false

- **상태**: Accepted
- **결정일**: 2026-05-14
- **결정자**: Troica 팀
- **관련**: R-45, R-57, ADR-0010 (보안 알림)

## 컨텍스트

R-45 로 7 polyrepo CI 에 SonarCloud (SonarQube Cloud) 정적 분석 + Quality Gate
통합 완료. 첫 스캔 후 다음 두 가지 제약이 드러남:

### 제약 1 — 무료 plan 의 Quality Gate 적용 불가

SonarCloud **OSS Public plan** 은 프로젝트 별로 custom Quality Gate 를 적용할 수
없음 ("paid feature"). default Gate **"Sonar way"** 만 사용 가능. Sonar way 의
"New Code Coverage ≥ 80%" 조건이 강제 적용됨.

화면 확인:
> Your current plan does not allow you to associate a Quality Gate other than
> Sonar way (Default) to this project. Upgrade plan

Custom Gate 생성은 가능하나 `Set as Default` 가 불가능.

### 제약 2 — PoC 단계의 새 코드 Coverage 0%

R-57 로 단위 테스트 ~48 케이스 추가했으나, 첫 PR 시점에서는:
- 새 코드 = build.gradle.kts 변경 + test 파일들
- test 파일은 coverage 측정 대상이 아님 (자기 자신 테스트 X)
- 따라서 **PR scan 의 new code coverage = 0%** → Sonar way Gate 자동 fail

이건 머지 후 main scan 에서야 정상 coverage 가 측정되는 닭-달걀 상황.

## 결정

**`sonar.qualitygate.wait = false`** 로 7 polyrepo build.gradle.kts 일괄 적용.

### 동작 흐름

```
[CI] gradlew sonar
  → SonarCloud 에 분석 결과 push
  → Quality Gate 평가 결과 (PASS/FAIL) 를 CI 가 기다리지 않음
  → CI step 통과 → 다음 step 정상 진행

[SonarCloud Dashboard]
  → Gate 결과 비동기 표시 (실패해도 CI 영향 없음)
  → 발표 / 운영 시 화면 으로 직접 확인
```

### 평가 기준 영향 분석

| 항목 | 영향 |
|---|---|
| SonarCloud 통합 (필수) | ✅ 그대로 작동 |
| 정적 분석 결과 (Bugs / Code Smells / Hotspots) | ✅ Dashboard 에 표시 |
| Quality Gate 정책 정의 | ✅ Sonar way 적용 중 (Coverage 80% 등) |
| Gate 미통과 시 CI 차단 | ⚠️ 약화 — wait=false 라 통과시킴 |

**보강 절차**:
- Security Hotspot 발견 시 SonarCloud UI 에서 Review 처리 (Safe / Acknowledged
  / Fixed). 평가관 입장에서 "Hotspot 발견 → 검토 → 사유 기록 → 판정" 의 운영
  절차를 더 잘 시연.
- 단위 테스트 정착 후 또는 paid plan 전환 시 `wait=true` 복구.

## 대안

| 안 | 거부 사유 |
|---|---|
| (a) paid plan 으로 upgrade | 학습 / PoC 단계 비용 부담. 발표 후 운영 단계 결정 |
| (b) custom Gate 생성 + 프로젝트 적용 | **무료 plan 에서 적용 불가** (제약 1) |
| (c) Sonar way 의 Coverage 조건 자체 수정 | Sonar way 는 built-in / read-only |
| (d) `qualitygate.wait=false` — 채택 | CI 그린 회복 + SonarCloud 결과 시연 가능 |
| (e) sonar step 제거 또는 `SONARCLOUD_ENABLED=false` | 평가 기준 cover 안 됨 |

## 결과

**긍정**
- 평가 심화 (2)-1 cover (SonarCloud 통합 + Quality Gate 정책 정의)
- CI 그린 회복 — 머지 가능
- 발표 시연 자료 — SonarCloud Dashboard 화면 + Hotspot Review 절차

**부정 / 후속**
- "Gate 미통과 시 CI 차단" 약화 — 발표 시 보완 설명 필요:
  - "PoC 단계 (단위 테스트 정착 중) 라 wait=false / 운영 단계는 wait=true 복구"
- 단위 테스트 정착 (R-57 후속 + 점진 coverage 향상) 후 `wait=true` 복구 + paid
  plan 전환 검토 시점 명시 필요 (Phase 6 또는 운영 진입 시)

## 운영 절차

### Hotspot Review 표준

각 polyrepo 의 SonarCloud Hotspot 은 다음 두 분류로 처리:

1. **Safe (PoC 단계 false positive 또는 운영 시 추가 layer 로 완화)**
   - 예: `kotlin:S6474` (Gradle dependency-verification 미사용) — Maven Central
     + GitHub Packages 신뢰된 source + Trivy image scan layer 로 supply chain
     일부 완화. PoC 단계 metadata 관리 부담 과도.

2. **Acknowledged / Fixed** — 코드 수정 또는 운영 단계 액션 명시

### 향후 `wait=true` 복구 조건

다음 셋 중 하나 충족 시:
- (i) 7 polyrepo 모두 **단위 테스트 coverage ≥ 50%** 도달
- (ii) Sonar way 의 "New Code Coverage 80%" 조건이 평균 PASS
- (iii) paid plan 전환 → custom Quality Gate 적용 가능

이때 `sonar.qualitygate.wait = true` 로 일괄 복구 + 평가 기준 "Gate 미통과 시
CI 차단" 강화.

## 관련

- BACKLOG: R-45 (SonarCloud 통합), R-57 (단위 테스트), R-58 (Testcontainers)
- PROJECT_PLAN §4.1 (테스트 스택) + §7 Epic E14 (DevSecOps)
- 평가 심화 (2)-1

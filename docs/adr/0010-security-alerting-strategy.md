# ADR-0010: 보안 알림 채널 전략 — `#security-report` 분리

- **상태**: Accepted
- **결정일**: 2026-05-13
- **결정자**: Troica 팀
- **관련**: R-47, R-46, R-55 (Falco — Phase 6 선택)

## 컨텍스트

평가 심화 (2)-3 "보안 스캔 결과를 Slack `#security-report` 채널에 연동" 에 대응.
일반 운영 알림 채널(`#alerts` 또는 유사)과 **보안 알림을 분리**해야 하는 이유:

- 보안 사고는 응답 시간 (MTTR) 이 일반 운영보다 짧아야 함 (CVE / 권한 침해 등).
- 보안 담당 청자가 운영 담당과 다를 수 있음 (SRE vs Security).
- 일반 운영 알림에 묻혀 보안 신호가 손실될 위험.

## 결정

**Slack 채널을 2개로 분리** 함.

| 채널 | 발신자 | 사례 |
|---|---|---|
| `#alerts` | AlertManager (severity ∈ {warning, critical}) | Pod CrashLoop, HPA 한계 도달, Kafka consumer lag 등 |
| **`#security-report`** | (1) GitHub Actions — Trivy/SonarQube/ZAP 스캔 실패 (2) AlertManager (severity=`security`) — Falco 런타임 탐지 등 | privileged 컨테이너 발견 / CVE-HIGH 검출 / OPA 정책 위반 / 비정상 shell |

### 발신 경로

1. **CI 단계** — GitHub Actions step에서 `slackapi/slack-github-action`로 webhook 직접
   호출. 본 레포의 Trivy manifest scan (R-46) 이미 적용됨. 6 polyrepo CI 의 Trivy
   image scan + SonarCloud scan도 동일 패턴 적용 예정 (R-45 작업 시).
2. **런타임 단계** — Prometheus alert rule 의 `severity: security` 라벨 → AlertManager
   route → Slack receiver `security`.

### Secrets / 설정

| 키 | 위치 | 용도 |
|---|---|---|
| `SLACK_SECURITY_WEBHOOK_URL` | GitHub Org Secret | CI step 의 webhook URL |
| `slack.api_url` (security receiver) | AlertManager values (kube-prometheus-stack) | 런타임 알림 |

GitHub Org Secret으로 두면 6 polyrepo + manifest repo 모두 접근 가능. 보안 채널
webhook은 일반 알림 채널 webhook과 **별도 URL** 사용 — 채널 분리 강제.

### AlertManager 라우팅 예시 (Phase 5 R-35 (e) 와 함께 적용)

```yaml
alertmanager:
  config:
    route:
      receiver: default-slack
      routes:
        - matchers:
            - severity = "security"
          receiver: security-slack
          continue: false       # 보안 알림은 다른 receiver로 fallthrough 안 시킴
    receivers:
      - name: default-slack
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack-default-url
            channel: "#alerts"
      - name: security-slack
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/slack-security-url
            channel: "#security-report"
            send_resolved: true
```

ExternalSecrets Operator (R-25/R-33) 가 AWS Secrets Manager 에서 두 URL을
가져와 Secret 생성. AlertManager Pod 에 mount.

## 대안 비교

| 안 | 거부 사유 |
|---|---|
| (a) 단일 채널 (`#alerts`) + 라벨로만 구분 | 알림 폭주 시 보안 신호 손실 위험. |
| (b) **2개 채널 (`#alerts` + `#security-report`)** — **채택** | 평가 기준 부합. 청자 분리 효과 큼. |
| (c) 3개 이상 채널 (severity 별) | 운영 부담 증가. 본 PoC 규모에 과함. |

## 결과

**긍정**
- 평가 심화 (2)-3 cover.
- 보안 사고 응답 시간 단축 (전용 채널 모니터링).
- Trivy/SonarCloud/Falco 결과가 자연스럽게 한 곳에 집결.

**부정/후속**
- Slack 채널 생성 + webhook 발급은 수동 작업 (Phase 5 진입 시 1회).
- AlertManager 실 매니페스트는 R-35 (e) 와 함께 적용 (kube-prometheus-stack values).
- 본 ADR은 결정만 명시 — 실 매니페스트는 Phase 5에서.

## 관련

- BACKLOG: R-47 (본 ADR 작성 + Trivy CI Slack webhook 적용), R-35 (e) (AlertManager 본 배포)
- 평가 심화 (2)-3
- ADR-0005 (api-gateway), ADR-0009 (api-gateway ↔ Istio 협업)

# R-47 (B) Demo — AlertManager severity 라우팅 → Slack 채널 분기

> **평가 cover**: 심화 (2)-3 (필수) — Slack #security-report 보안 알림 채널
>
> **ADR**: [ADR-0010 — 보안 알림 채널 분리 전략](../adr/0010-security-alerting-strategy.md)
>
> **소요**: ~10 분

## 사전 조건

cluster up + R-42 (B) 와 비슷한 monitoring stack ready 상태.

```bash
# 1. AlertManager pod Running
kubectl -n monitoring get pods -l app.kubernetes.io/name=alertmanager

# 2. Slack webhook ExternalSecret synced (R-33)
kubectl -n monitoring get externalsecret alertmanager-slack-webhooks
# 기대: STATUS=SecretSynced READY=True

# 3. 마운트되는 Secret 자체 존재 + 두 key
kubectl -n monitoring get secret alertmanager-slack-webhooks -o jsonpath='{.data}' | tr ',' '\n'
# 기대: "slack-default-url":"...base64...","slack-security-url":"...base64..."

# 4. PromRule 4 개 등록 (PR #91)
kubectl get prometheusrule -A | grep troica
# 기대: troica-domain-alerts (KafkaConsumerLagHigh, HighHttpErrorRate, JvmHeapPressure, CircuitBreakerOpened)
```

---

## Part 1 — AlertManager config 정합성 확인

### Step 1 — AlertManager pod 안 mount 경로 검증

```bash
# pod 안의 webhook URL 파일 존재 + 내용 확인
ALERT_POD=$(kubectl -n monitoring get pod -l app.kubernetes.io/name=alertmanager -o jsonpath='{.items[0].metadata.name}')

# 두 파일 모두 존재해야 함
kubectl -n monitoring exec $ALERT_POD -c alertmanager -- \
    ls -la /etc/alertmanager/secrets/alertmanager-slack-webhooks/

# 기대 출력:
# slack-default-url
# slack-security-url

# 파일 내용 = Slack webhook URL (https://hooks.slack.com/...)
kubectl -n monitoring exec $ALERT_POD -c alertmanager -- \
    cat /etc/alertmanager/secrets/alertmanager-slack-webhooks/slack-security-url
```

**기대**: `https://hooks.slack.com/services/T.../B.../...` 한 줄.

### Step 2 — AlertManager 가 로드한 config 확인

```bash
# AlertManager 의 active config (HTTP /api/v2/status)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 &
sleep 3

curl -s http://localhost:9093/api/v2/status | grep -A 30 "configYAML\|route\|receivers" | head -60

# 또는 web UI 접근 (브라우저 또는 ssh tunnel)
# http://localhost:9093 — Status 탭에 active config
```

**확인 포인트**:
- `route.routes[0].matchers` 에 `severity = "security"` 매칭
- `receivers[].name` 에 `default-slack` + `security-slack`
- `slack_configs[].api_url_file` 정확한 mount 경로

---

## Part 2 — alert fire 시연 (3 가지 path)

### Path A — R-41 demo 의 자연 발생 (권장, 가장 정직한 demo)

R-41 (B) demo 시 inventory-service down → api-gateway 의 CircuitBreaker OPEN → PromRule `CircuitBreakerOpened` (severity=security) 자동 fire.

```bash
# R-41 (B) demo 진행 후 alertmanager 의 active alert 확인
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 &
sleep 3

curl -s http://localhost:9093/api/v2/alerts | grep -oE '"alertname":"[^"]+","[^"]+":"[^"]+"' | head -10

# 기대: "alertname":"CircuitBreakerOpened" + "severity":"security"

# Slack #security-report 채널에 :rotating_light: CircuitBreakerOpened 메시지 수신 확인
```

### Path B — amtool 로 manual fire (CB demo 없이 단독 검증)

```bash
# AlertManager pod 안에서 amtool 으로 alert push
kubectl -n monitoring exec $ALERT_POD -c alertmanager -- \
    amtool alert add \
    --alertmanager.url=http://localhost:9093 \
    alertname=ManualSecurityTest \
    severity=security \
    instance=demo \
    summary="R-47 demo manual security alert"

# active alerts 확인
kubectl -n monitoring exec $ALERT_POD -c alertmanager -- \
    amtool alert query --alertmanager.url=http://localhost:9093

# Slack #security-report 채널 확인
```

### Path C — HTTP POST 직접 (amtool 없을 때 fallback)

```bash
# Alertmanager 의 /api/v2/alerts 에 직접 POST
curl -X POST http://localhost:9093/api/v2/alerts \
    -H "Content-Type: application/json" \
    -d '[{
      "labels": {
        "alertname": "ManualSecurityTest",
        "severity": "security",
        "instance": "demo"
      },
      "annotations": {
        "summary": "R-47 demo manual security alert",
        "description": "Manual fire via HTTP API to verify Slack #security-report routing"
      },
      "generatorURL": "http://demo"
    }]'

# active alerts 확인
curl -s http://localhost:9093/api/v2/alerts | grep -oE '"alertname":"ManualSecurityTest"'

# Slack #security-report 채널 확인 (10-30 초 후 메시지 도착)
```

---

## Part 3 — severity=warning / critical 분기 검증 (선택)

`#alerts` 채널 도 작동하는지:

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
    -H "Content-Type: application/json" \
    -d '[{
      "labels": {
        "alertname": "ManualWarningTest",
        "severity": "warning",
        "instance": "demo"
      },
      "annotations": {
        "summary": "R-47 demo warning routing"
      },
      "generatorURL": "http://demo"
    }]'
```

**기대**: Slack **#alerts** 채널 (default-slack receiver) 에 메시지 수신.

severity=security 와 분기 구분 demo.

---

## 평가 발표 포인트

| 평가요소 | 시연 |
|---|---|
| 심화 (2)-3 Slack #security-report 보안 알림 | severity=security → security-slack receiver → #security-report 메시지 |
| ADR-0010 채널 분리 | AlertManager route 매니페스트 (kube-prometheus-stack values inline) |
| Trivy workflow Slack notify | trivy-manifest-scan.yml 의 slack-github-action step (R-46) |

---

## Troubleshooting

| 증상 | 가능한 원인 |
|---|---|
| AlertManager log 에 `api_url_file: open ...: no such file` | secret mount 경로 mismatch — Step 1 의 `ls` 확인 |
| Slack 채널 메시지 안 옴 | (a) webhook URL 잘못 (b) outbound 차단 (NAT/SG) (c) AlertManager log `dial tcp ...` |
| ExternalSecret SecretSyncedError | troica/slack/webhooks (AWS Secrets Manager) 미등록 또는 IAM 권한 X (PR #12 확인) |
| amtool 없음 | Path C (HTTP POST) 또는 amtool 별도 다운로드 |
| alert fire 했는데 라우팅 잘못 | AlertManager Status 페이지의 active config 의 route.matchers 확인 |
| Slack 채널 ID mismatch | 매니페스트 의 channel: "#security-report" 와 실 채널 이름 일치 |

---

## AlertManager Slack message 형식 확인 (선택)

`#security-report` 메시지 형식:
```
:rotating_light: CircuitBreakerOpened
[FIRING:1] CircuitBreakerOpened severity=security ...
```

`title: ':rotating_light: {{ .CommonLabels.alertname }}'` — 빨간 회전등 emoji + alertname.

---

## Reference

- [AlertManager Slack receiver docs](https://prometheus.io/docs/alerting/latest/configuration/#slack_config)
- [prometheus-operator alertmanagerSpec.secrets](https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.AlertmanagerSpec)
- [ADR-0010 보안 알림 채널 분리](../adr/0010-security-alerting-strategy.md)
- [PromRule 매니페스트](../../platform/31-prometheus-rules/manifests/troica-domain-alerts.yaml)

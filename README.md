# msa-argocd-manifest

Troica Market Service MSA의 **GitOps 매니페스트 단일 진실의 원천**.

> 프로젝트 계획서 (실제 구현 반영 갱신본): [docs/PROJECT_PLAN.md](./docs/PROJECT_PLAN.md)
> SPEC: [docs/TROICA_SPEC.md](./docs/TROICA_SPEC.md)
> 작업 백로그: [docs/BACKLOG.md](./docs/BACKLOG.md)
> 트러블슈팅: [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md)
> 아키텍처 결정: [docs/adr/](./docs/adr/)

---

## 시스템 아키텍처

![AWS 배포 아키텍처](./docs/images/AWS-arch.png)

상세 다이어그램 (Mermaid, `docs/images/`):
- [AWS-architecture.md](./docs/images/AWS-architecture.md) — AWS 리소스 7개 섹션 (Network / Compute+Storage / Security Groups / CI-CD / Kubelet ECR pull / 영구·임시·수동 분류 + 통합 요약)
- [GitOps-flow.md](./docs/images/GitOps-flow.md) — Developer → polyrepo CI → ECR → manifest → ArgoCD app-of-apps → 클러스터 배포 흐름
- [MSA-services.md](./docs/images/MSA-services.md) — 6개 마이크로서비스 토폴로지 + Kafka 토픽 기반 saga (서비스 간 직접 호출 0건)

핵심 컴포넌트:
- **AWS Phase 0** (영구 자원): IAM OIDC + 6 ECR repo + KMS — `msa-provisioning` Terraform 관리
- **K8s 클러스터** (임시 자원): EC2 + NAT + NLB + EFS — `msa-provisioning` Ansible 셋업
- **GitOps**: ArgoCD가 본 레포 watch → 클러스터 sync
- **6개 서비스**: 각 polyrepo의 CI → ECR push → manifest auto-bump PR → ArgoCD sync

---

## 디렉토리 구조

```
.
├── README.md                       # 본 파일
├── docs/                           # 문서
│   ├── TROICA_SPEC.md              # 청사진 (단일 진실의 원천)
│   ├── BACKLOG.md                  # Phase별 task 진행 상태
│   ├── TROUBLESHOOTING.md          # 누적 디버깅 자료
│   ├── PHASE_4_RUNBOOK.md          # Phase 4 작업 절차
│   ├── PROGRESS_LOG.md             # 컨텍스트 스냅샷
│   ├── adr/                        # Architecture Decision Records
│   └── images/                     # 다이어그램 (AWS-arch.png 등 — 모든 diagram의 단일 위치)
├── bootstrap/
│   ├── root-app.yaml               # ArgoCD 진입점 (app-of-apps)
│   ├── apps.yaml                   # 자식 Application 3개 (projects/platform/applications)
│   └── README.md
├── projects/                       # ArgoCD AppProject CRD
│   ├── market-project.yaml
│   └── platform-project.yaml
├── applications/                   # 마이크로서비스 레이어 (ApplicationSet)
│   ├── appset.yaml                 # 6 service × 2 env = 12 Application 자동 생성
│   ├── charts/microservice/        # 공통 Helm 차트
│   └── values/<service>/           # 서비스별 + 환경별 values
├── platform/                       # 플랫폼 레이어 (App-of-Apps, 19 Application)
│   ├── root.yaml                   # platform 측 App-of-Apps root (Wave -10)
│   ├── 00-cert-manager/            # cert-manager (Wave 0)
│   ├── 05-ebs-csi/                 # AWS EBS CSI driver (Wave 0)
│   ├── 05-rbac/                    # RBAC roles + bindings (Wave -8, R-48)
│   ├── 10-istio-base/              # Istio CRD (Wave -5, R-03)
│   ├── 11-istio-cp/                # istiod (Wave -2, R-62 분리 — gateway chart 제거)
│   ├── 12-istio-gateway/           # Gateway+VS+PeerAuth+DR+namespace label (Wave -3)
│   ├── 13-istio-ingressgateway/    # istio-ingressgateway (Wave -1, R-62 PR #118 별도 분리)
│   ├── 30-kube-prometheus-stack/   # Prometheus + Grafana + AlertManager (R-35/R-47)
│   ├── 30-loki/                    # Loki (Wave 0)
│   ├── 30-tempo/                   # Tempo (Wave 0)
│   ├── 31-prometheus-rules/        # AlertManager 측 룰 (R-47 security 라우팅)
│   ├── 40-cnpg-operator/           # CloudNativePG operator (Wave 1)
│   ├── 40-external-secrets-operator/  # ESO (Wave 1, R-25)
│   ├── 40-strimzi-operator/        # Strimzi Kafka operator (Wave 1)
│   ├── 50-kafka-cluster/           # Kafka KRaft 1 broker + 4 topic (Wave 2)
│   ├── 50-postgres-clusters/       # 6 CNPG Cluster (Wave 2)
│   ├── 50-redis-cluster/           # Redis (Wave 2)
│   ├── 60-network-policies/        # default-deny + 의도 통신 allow (R-49)
│   └── 91-external-secrets-config/ # 24 ExternalSecret (Wave 3, R-25/R-33)
├── tests/e2e/                       # Newman collection (R-42)
│   └── troica-market-e2e.postman_collection.json   # 7 step 시나리오
└── .github/workflows/               # GitHub Actions
    ├── e2e-newman.yml               # Newman cluster 호출 (R-42)
    └── trivy-manifest-scan.yml      # 매니페스트 보안 스캔 (R-46)
```

---

## 빠른 시작 — 클러스터에 부트스트랩 (L3)

### 사전 요구사항

- `msa-provisioning` Phase 0 완료 — ECR/OIDC 영구 자원 생성됨
- `msa-provisioning` 임시 자원 apply + `ansible-playbook main.yaml` 완료 — k8s 클러스터 + ArgoCD 설치됨
- master 노드에 SSH 가능 + `kubectl`/`argocd` CLI 설치됨

### 1. ArgoCD root-app 적용 (1회)

자동: ansible의 `argocd-setup.yaml`이 본 레포의 `bootstrap/root-app.yaml`을 자동 fetch + apply.

수동 (필요 시):
```bash
kubectl apply -f https://raw.githubusercontent.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest/main/bootstrap/root-app.yaml
```

### 2. 자식 Application 자동 생성 확인

```bash
kubectl get applications -n argocd
```

기대 (모두 `Synced + Healthy` — 약 5-10 분 대기):
- `root-app` (Wave -10) — bootstrap 진입점
- 3 자식 (Wave -10): `app-projects`, `app-platform`, `app-applications`
- **19 platform Application** (Wave -8 ~ 5): cert-manager / ebs-csi / rbac / istio-{base,cp,gateway,ingressgateway} / kube-prometheus-stack / loki / tempo / troica-prometheus-rules / cnpg-operator / external-secrets-operator / strimzi-operator / kafka-cluster / postgres-clusters / redis-cluster / platform-network-policies / external-secrets-config + platform-root
- **6 service Application** (Wave 10, PoC diet dev only): `api-gateway-dev` / `auth-service-dev` / `user-service-dev` / `product-service-dev` / `inventory-service-dev` / `order-service-dev`

**OutOfSync 정상 7 개** (cosmetic — R-61 후속): istio-base / istio-cp / external-secrets-config / strimzi-operator / tempo / platform-root / root-app. mutating webhook inject + CRD default field 의 git ↔ cluster diff. 작동 무영향.

### 3. ArgoCD UI 접근

```bash
# admin 비밀번호 (master 측)
kubectl get secret -n argocd argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

ArgoCD CLI / UI 접근 (master 측 NodePort 30090, R-21 PR #21 의 정정 후):

```bash
# CLI 로그인 (master 측)
argocd login <master-private-ip>:30090 --username admin --insecure

# 동기화 (자동 sync 이미 활성. 수동 trigger 시)
argocd app sync root-app --prune
```

브라우저 UI 접근 — local PC 에서 SSH tunnel + port-forward:

```powershell
# local PowerShell (bastion 경유 main-master)
ssh -i ~/.ssh/ktcloud-bastion-node-key -J ec2-user@<bastion-public-ip> -L 30090:<main-master-private-ip>:30090 ec2-user@<main-master-private-ip>
```

브라우저: `http://localhost:30090` (admin / 위 비밀번호)

### 4. 6 service pod 측 정상 작동 확인

Phase 5 의 platform Application 들 (postgres / redis / kafka / external-secrets) 측 sync 완료 후 6 service pod startup:

```bash
# 8 pod (api-gateway + 5 service + 2 worker) 측 2/2 Running 까지 watch
kubectl get pods -n market-dev -w
# Ctrl-C 로 종료

# 기대 출력:
# api-gateway-XXX                 2/2 Running   0   1m
# auth-service-XXX                2/2 Running   0   1m
# user-service-XXX                2/2 Running   0   1m
# product-service-XXX             2/2 Running   0   1m
# inventory-service-XXX           2/2 Running   0   1m
# inventory-service-worker-XXX    2/2 Running   0   1m
# order-service-XXX               2/2 Running   0   1m
# order-service-worker-XXX        2/2 Running   0   1m
```

`2/2` = main app container + istio-proxy sidecar (mTLS STRICT, R-03). Istio sidecar 자동 inject — `istio-injection=enabled` namespace label (PR #111).

### 5. 외부 진입 path 검증 (NLB → Istio → api-gateway)

```bash
# msa-provisioning 측 nlb_dns_name output 확인
# 또는 AWS CLI:
NLB="http://$(aws elbv2 describe-load-balancers --names kt-cloud-nlb --query 'LoadBalancers[0].DNSName' --output text)"

curl -i --max-time 10 "$NLB/healthz"
# 기대: HTTP/1.1 200 OK + {"status":"UP",...} (api-gateway actuator)

curl -i --max-time 10 "$NLB/api/v1/products"
# 기대: HTTP/1.1 200 OK + {"products":[]} (BFF → product-service gRPC)
```

---

## 평가 시연 path (재현 가능 — 평가 후 후속 cycle 측에도 동일 적용)

### R-41 Circuit Breaker 시연 (장애 격리, 기본 (2)-3 + (2)-4)

```bash
# 1. 정상 응답 baseline (3 회)
NLB="http://kt-cloud-nlb-XXXXXX.elb.ap-northeast-2.amazonaws.com"
for i in 1 2 3; do curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" $NLB/api/v1/products; done
# 기대: 200 0.0Xs × 3

# 2. ArgoCD selfHeal 일시 disable (fault injection 위해 — selfHeal 가 scale 즉시 복원)
kubectl patch application product-service-dev -n argocd --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":false,"prune":false}}}}'

# 3. product-service scale 0
kubectl scale deployment -n market-dev product-service --replicas=0

# 4. fallback 응답 확인 — CB OPEN signature (110-280ms fail-fast)
for i in $(seq 1 10); do
  curl -s -o /tmp/out -w "$i: %{http_code} %{time_total}s\n" $NLB/api/v1/products
done
# 기대: 1: 500 0.4Xs  ← CB CLOSED, fail count
#       2-10: 500 0.1Xs ⚡ CB OPEN, fail-fast

# 5. 다른 service 격리 확인
curl -i --max-time 10 $NLB/api/v1/users    # 기대: 401 (격리됨)
curl -i --max-time 10 $NLB/api/v1/orders   # 기대: 401 (격리됨)
curl -i --max-time 10 $NLB/healthz          # 기대: 200 (api-gateway 자체)

# 6. 복구 — selfHeal 다시 활성
kubectl scale deployment -n market-dev product-service --replicas=1
kubectl patch application product-service-dev -n argocd --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":true,"prune":true}}}}'
```

### R-42 Newman E2E 시연 (기본 (3)-2 + (3)-5)

GitHub Actions UI 측:
1. https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest/actions/workflows/e2e-newman.yml
2. **Run workflow** → baseUrl input = `http://kt-cloud-nlb-XXXXXX.elb.ap-northeast-2.amazonaws.com`
3. 결과: 5/5 시나리오 PASS (signup → signin/JWT → products → inventories → JWT check, orders 측 data 부족 시 skip 로직)

### R-49 NetworkPolicy 차단 시연 (심화 (3)-3)

```bash
# 1. ALLOW case: api-gateway label → auth-service:9005
kubectl run test-allow --rm -it --image=busybox:1.36 -n market-dev \
  --labels=app.kubernetes.io/name=api-gateway --restart=Never -- \
  sh -c 'time nc -zv auth-service.market-dev.svc.cluster.local 9005'
# 기대: 0.71s 통과 (Connection ... open)

# 2. DENY case: 다른 label → 차단
kubectl run test-deny --rm -it --image=busybox:1.36 -n market-dev \
  --labels=app.kubernetes.io/name=product-service --restart=Never -- \
  sh -c 'time nc -zv -w 5 auth-service.market-dev.svc.cluster.local 9005'
# 기대: 5s timeout
```

### R-47 Slack #security-report 시연 (심화 (2)-3)

```bash
# AlertManager port-forward
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093 &

# manual fire (severity=security)
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"ManualSecurityTest","severity":"security"},"annotations":{"summary":"R-47 시연"}}]'

# Slack #security-report 채널 도착 확인
```

### Grafana UI 접근 (built-in dashboard 시각화)

```powershell
# local PC PowerShell (port-forward + ssh tunnel)
ssh -i ~/.ssh/ktcloud-bastion-node-key -J ec2-user@<bastion-public-ip> \
  -L 3000:localhost:3000 ec2-user@<main-master-private-ip>
# ssh session 의 main-master shell 에서:
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

브라우저: `http://localhost:3000`

admin 비밀번호:
```bash
kubectl get secret -n monitoring grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

시연 대상 dashboard:
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace (Pods)
- Node Exporter / Nodes
- Prometheus / Overview

---

## 로컬 검증 (PR 전 dry-run)

### Helm chart 렌더링

```bash
helm template applications/charts/microservice \
  -f applications/values/_template/values.yaml \
  -f applications/values/_template/values-dev.yaml
```

### kubectl dry-run (서버 측 검증)

```bash
# 클러스터 연결된 경우
kubectl apply --dry-run=server -f bootstrap/root-app.yaml
kubectl apply --dry-run=server -f bootstrap/apps.yaml
kubectl apply --dry-run=server -f projects/
kubectl apply --dry-run=server -f applications/appset.yaml
```

### ApplicationSet template 렌더링 미리보기

```bash
argocd appset list
argocd appset get market-services
```

---

## 변경 워크플로우

### 새 서비스 추가

1. `applications/values/<service>/` 디렉토리 생성:
   - `values.yaml` (공통)
   - `values-dev.yaml` (dev 환경)
   - (PoC diet — prod 비활성. 운영 단계 진입 시 `values-prod.yaml` 추가)
2. `_template/`을 베이스로 복사 후 image.repository + port 등 조정
3. PR 머지 → ApplicationSet이 자동 감지 → 6 → 7 Application (dev only)

### 새 platform 컴포넌트 추가

1. `platform/<NN>-<name>/application.yaml` 생성 (Application CR 또는 Helm chart reference)
2. PR 머지 → `platform-root`가 `**/application.yaml` glob으로 자동 발견 → sync

### image tag 자동 bump

각 서비스의 CI가 main push 시 자동:
- `values-dev.yaml`의 `image.tag` 직접 commit (자동 dev 배포)
- `values-prod.yaml`의 `image.tag` PR 생성 (prod 승인 게이트)

---

## 의사결정 + 변경 원칙

- **`main` 직접 커밋 금지** — 모든 변경은 feature branch + PR
- **아키텍처 결정 변경 시**: SPEC.md + BACKLOG.md 동시 갱신 + 새 ADR 추가 (`docs/adr/NNNN-*.md`)
- **다이어그램**: 모든 PNG/SVG는 `docs/images/`에 모음. 다른 레포는 GitHub raw URL로 참조.

주요 ADR:
- [ADR-0001](./docs/adr/0001-polyrepo-with-auth-service.md) Polyrepo + auth-service 분리
- [ADR-0005](./docs/adr/0005-api-gateway-bff-with-cloud-gateway.md) api-gateway BFF + Spring Cloud Gateway 혼합
- [ADR-0006](./docs/adr/0006-order-state-machine-extension.md) order state machine 7-state
- [ADR-0007](./docs/adr/0007-inventory-event-sourcing-batch-worker.md) inventory Event Sourcing + Spring Batch

전체 목록: [docs/adr/README.md](./docs/adr/README.md)

---

## 관련 레포

| 레포 | 역할 |
|---|---|
| [msa-provisioning](https://github.com/KTCloud-CloudNative-Troica-Team/msa-provisioning) | Terraform + Ansible (AWS 인프라 + k8s 클러스터) |
| [msa-common-libs](https://github.com/KTCloud-CloudNative-Troica-Team/msa-common-libs) | 공용 라이브러리 (common + events Protobuf) |
| [msa-product-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-product-service) | 상품 도메인 |
| [msa-order-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-order-service) | 주문 도메인 + Outbox worker |
| [msa-inventory-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-inventory-service) | 재고 도메인 + Event Sourcing Batch worker |
| [msa-user-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-user-service) | 사용자 도메인 |
| [msa-auth-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-auth-service) | 인증 (JWT 발급/검증) |
| [msa-api-gateway](https://github.com/KTCloud-CloudNative-Troica-Team/msa-api-gateway) | 외부 진입점 (BFF + SC Gateway) |
| [msa-frontend](https://github.com/KTCloud-CloudNative-Troica-Team/msa-frontend) | 프론트엔드 SPA (React 19 + Vite + TS, `market-msa-app/`) — 팀장님 작성 |

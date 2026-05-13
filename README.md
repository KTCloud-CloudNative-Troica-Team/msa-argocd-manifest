# msa-argocd-manifest

Troica Market Service MSA의 **GitOps 매니페스트 단일 진실의 원천**.

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
└── platform/                       # 플랫폼 레이어 (App-of-Apps)
    └── root.yaml
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
kubectl -n argocd get application -A
```

기대:
- `root-app` (Synced + Healthy)
- 3개 자식: `app-projects`, `app-platform`, `app-applications`
- ApplicationSet `market-services`가 생성한 12개 서비스 Application (`<service>-{dev,prod}`)

### 3. ArgoCD UI 접근

```bash
# admin 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# CLI 로그인 (NodePort 30080)
argocd login <master-private-ip>:30080 --username admin --insecure

# sync
argocd app sync root-app --prune
```

### 4. pod 상태 확인 (Phase 5 platform 배포 전까지는 CrashLoop 정상)

```bash
kubectl get pods -n market-dev
kubectl get pods -n market-prod
```

ECR image pull은 성공 (kubelet credential provider 작동). Spring Boot가 PostgreSQL/Redis/Kafka 없어서 startup 실패 → CrashLoopBackOff가 **Phase 0의 자연스러운 경계**. Phase 5에서 platform 배포 시 해소.

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
   - `values-prod.yaml` (prod 환경)
2. `_template/`을 베이스로 복사 후 image.repository + port 등 조정
3. PR 머지 → ApplicationSet이 자동 감지 → 12 → 14 Application

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

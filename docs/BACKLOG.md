# Troica MSA 작업 백로그 (Phase별)

> **참조**: 모든 작업은 [TROICA_SPEC.md](./TROICA_SPEC.md) 의 Phase 구조를 따른다.
> 트러블슈팅 / 디버깅 세부 내용은 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) 참조 — 본 문서는 task 진행 상태 + 평가요소 매핑만 추적한다.
>
> **작업 원칙**
> - `main` 직접 커밋 금지. 모든 변경은 feature branch + PR.
> - **`msa-spring-boot` 레포는 read-only**. 다른 작업자가 main 에서 작업 중이므로 push 금지.
> - Phase 단위로 진행하며 SPEC §14 검증 체크리스트를 통과해야 다음 Phase 진입.

## 상태 범례

| 기호 | 상태 |
|---|---|
| ✅ | 완료 (PR 머지) |
| 🔄 | 진행 중 (PR 작업 중 또는 머지 대기) |
| ⏸ | 대기 (선결 조건 필요) |
| 🔍 | 검토 필요 |
| 📌 | 보류 / 미래 작업 |
| ❌ | 폐기 (작업하지 않기로 결정) |

## 평가요소 컬럼 표기

각 Phase 표의 **평가요소** 컬럼은 task 가 KT Cloud Tech UP 2기 발표 평가 기준의 어디에 매핑되는지 표기한다.

| 표기 | 의미 |
|---|---|
| `기본 (N)-M (필수)` / `심화 (N)-M (필수)` | 발표 평가 기준의 필수 항목에 직접 매핑 |
| `기본 (N)-M (선택)` / `심화 (N)-M (선택)` | 발표 평가 기준의 선택 항목 |
| `(필수)` (괄호만) | 평가 기준에 직접 매핑되지는 않으나 선결 조건인 필수 작업 (인프라 / GitOps 기반 / 도메인 베이스 등) |
| `(ADR-NNNN)` | 의사결정 — [ADR](./adr/) 단일 진실 |
| `(운영)` / `(품질)` | 운영 / 코드 품질 트랙 |
| `—` | 평가와 무관한 정리 / 회고 |

평가요소 → R 역인덱스는 본 문서 마지막의 [평가요소 매핑](#평가요소-매핑-kt-cloud-tech-up-2기-발표) 섹션 참조.

## ID convention

| prefix | 의미 | 예시 | 현재 정책 |
|---|---|---|---|
| **R-NN** | Risk / follow-up / task | R-27, R-38, R-57 | ✏️ **신규 task 는 R-NN 사용** (다음 번호 = 마지막 R 번호 + 1) |
| **P*<phase>.<seq>* | Phase 작업 단위 | P0.1, P3.2 | 📜 historical — commit history 에 immutable. 신규 사용 X |
| **D*N*** | 누적 결정 (Decision) | D1, D2, D6 | 📜 historical — 모두 [ADR](./adr/) 로 흡수. 본 문서는 링크 only |
| **Q*N*** | 회의 검토 질문 (Question) | Q1, Q4, Q7 | 📜 historical — D 와 동일 |

---

## Phase 0 — AWS 인프라 부트스트랩 (완료)

> Terraform + Ansible 로 AWS 영구/임시 자원 + ECR + IAM OIDC + kubelet ECR pull 까지 자동화. polyrepo 마이그레이션 의 인프라 베이스.

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| P0.1 | Terraform OIDC + ECR repo × 6 + VPC Endpoint + KMS | ✅ | (필수) | `msa-provisioning phase-0/aws-ecr-oidc` 머지 |
| P0.2 | kubelet ECR credential provider — Ansible playbook 골격 | ✅ | (필수) | `msa-provisioning phase-0/kubelet-ecr-credential-provider` |
| P0.3 | S3 backend + 영구/임시 자원 분리 (`prevent_destroy`) | ✅ | (필수) | `phase-0/permanent-resources-and-backend`. 비용 사이클 가능 |
| P0.4 | README S3 부트스트랩 절차 fix | ✅ | (필수) | `chore/readme-s3-bootstrap-fix` |
| R-19 | CI push-gated step `AWS_DEPLOYMENTS_ENABLED` gate | ✅ | (필수) | SPEC §7 + 6 서비스 ci.yml 가드 |
| R-26 | 매니페스트 `image.repository` placeholder → 실제 account_id | ✅ | (필수) | `phase-0/manifest-account-id` |
| R-29 | cloud-provider-aws ecr-credential-provider 바이너리 빌드 패턴 | ✅ | (필수) | `fix/ansible-ecr-credential-provider-integrated`. 자세히 [TROUBLESHOOTING §5.1](./TROUBLESHOOTING.md) |
| R-30 | kubelet `kubeadm-flags.env` 직접 통합 (drop-in 미평가 우회) | ✅ | (필수) | 같은 PR |
| R-31 | bootstrap/root-app.yaml app-of-apps 정식 패턴 | ✅ | (필수) | `fix/r-31-root-app-app-of-apps`. brace expansion → doublestar |
| R-32 | ansible argocd-setup.yaml root-app repo URL | ✅ | (필수) | `fix/r-32-argocd-setup-repo-url` |
| R-34 | `destroy-temp.sh` Windows 호환 (PowerShell 추가) | ✅ | (운영) | `chore/destroy-temp-powershell` + `chore/gitattributes-lf-enforcement` |
| R-37 | platform/root.yaml `**/application.yaml` glob | ✅ | (필수) | `fix/r-37-platform-root-glob` (R-31 동일 패턴) |

**Phase 0 검증 완료**: terraform apply/destroy 사이클 작동 / 6 서비스 ECR push 성공 / ApplicationSet 12 자식 app 자동 생성 / kubelet credential provider 정상 (`Successfully pulled image`).

---

## Phase 1 — ArgoCD 매니페스트 골격 (완료)

> GitOps 단일 진실의 원천 (manifest repo) 디렉토리 구조 정립 + 공통 Helm 차트 작성. 모든 후속 서비스 배포의 기반.

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| P1.1 | `TROICA_SPEC.md` 청사진 + `BACKLOG.md` 신설 | ✅ | (필수) | `bootstrap/spec-and-backlog` |
| P1.2 | 매니페스트 디렉토리 구조 (bootstrap / projects / applications / platform) | ✅ | (필수) | `phase-1/manifest-skeleton` |
| P1.3 | Helm chart `microservice` (deployment / service / hpa / sa / configmap / virtualservice) | ✅ | (필수) | 동일 PR. `helm lint` + `helm template` 통과 |
| R-01 | `values-worker.yaml` 미사용 → SPEC 정정 | ✅ | — | SPEC §2 dead file 제거 |
| R-02 | App-of-Apps multi-source 스니펫 정정 | ✅ | — | SPEC §6 `source:+sources:` 중복 정리 |
| P1-V | 클러스터-사이드 검증 (`kubectl apply --dry-run=server`) | ✅ | 기본 (2)-1 (필수) | Phase 0 클러스터에서 검증 |

---

## Phase 2 — common-libs 멀티모듈 + Protobuf (완료)

> 6 서비스가 공유할 공용 라이브러리 + 이벤트 스키마 분리. GH Packages publish 까지 자동화.

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| P2.1 | 4-module Gradle (common / client-redis / client-ses / events) | ✅ | (필수) | `msa-common-libs phase-2/initial-setup` |
| P2.2 | Protobuf 코드젠 + Kotlin codegen | ✅ | (필수) | 동일 PR. order / notification 도메인 proto |
| P2.3 | GitHub Packages publish workflow | ✅ | (필수) | tag push 시 `publish.yml` |
| P2.4 | gradlew 100755 fix | ✅ | (운영) | Windows 권한 fix |
| R-09 | Kotlin/Gradle toolchain 충돌 (2.3.x → 2.1.0) | ✅ | (운영) | [TROUBLESHOOTING §1.2](./TROUBLESHOOTING.md) |
| R-10 | 이벤트 스키마 Protobuf 채택 | ✅ | (ADR-0003) | SPEC §9.3 |
| R-11 | common-libs GH Packages publish 실 검증 | ✅ | (필수) | v0.1.0/v0.2.0/v0.3.0 tag push 검증 |
| D2 | client-redis JitPack 으로 일원화 | ✅ | (ADR-0002) | common-libs v0.3.0 에서 client-redis 모듈 제거 |
| D3 | client-ses 제거 (notification 폐기 일관) | ✅ | (ADR-0002) | common-libs v0.3.0 |

---

## Phase 3 — order-service polyrepo + worker 패턴 정립 (완료)

> 모노레포 → 첫 polyrepo 추출. `@Profile("worker")` 분기 + 별도 Deployment 로 worker 패턴 정립 (이후 inventory Spring Batch worker 도 동일 패턴 적용).

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| P3.1 | 2-module Gradle (order / order-service) | ✅ | 기본 (1)-3 (필수) | `msa-order-service phase-3/initial-setup` |
| P3.2 | order-worker `@Profile("worker")` + `@Scheduled` Outbox poller | ✅ | 기본 (1)-5 (선택) Outbox | 동일 PR |
| P3.3 | order-service values 매니페스트 (dev/prod) | ✅ | (필수) | `phase-3/order-service-values`. api + worker 2 Deployment |
| P3.4 | Spring Boot 3.3.0 → 3.5.13 upgrade | ✅ | (운영 / 보안) | `upgrade/spring-boot-3.5`. CVE-2025-22235 fix + EOL 회피. [TROUBLESHOOTING §2.2](./TROUBLESHOOTING.md) |
| R-06 | bootJar `JarLauncher` 경로 검증 | ✅ | (운영) | MANIFEST 확인 |
| R-07 | worker profile 분기 코드 | ✅ | 기본 (1)-5 (선택) | 신규 1 파일로 해소 |
| R-12 | 로컬 Docker build 검증 | ✅ | (운영) | Phase 0 CI ECR push 로 입증 |
| R-14 | Trivy action `@v0.36.0` pin (공급망 사고 대응) | ✅ | 심화 (2)-2 (필수) 일부 | [TROUBLESHOOTING §2.1](./TROUBLESHOOTING.md) |
| R-17 | common-libs v0.2.0 publish 후 deps 재bump | ✅ | (운영) | 자연스럽게 0.3.0 까지 진행 |

---

## Phase 4 — 6 서비스 polyrepo + 프론트엔드 (완료)

> 모노레포의 6 도메인을 polyrepo 로 분리 + auth-service 신규 + api-gateway BFF+SC Gateway 혼합 구현 + 프론트엔드 별도 polyrepo.

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| P4.1 | msa-product-service | ✅ | 기본 (1)-1 (필수) | `phase-4/initial-setup` |
| P4.2 | msa-order-service 토픽명 + state machine 확장 | ✅ | 기본 (1)-2 (필수) + ADR-0004/0006 | `phase-4/topics-and-state-machine`. `OrderLineItemStatus` 확장 + admin endpoint |
| P4.3 | msa-inventory-service Event Sourcing + Spring Batch worker | ✅ | 기본 (1)-5 (선택) | `phase-4/initial-setup`. `@Profile("worker")` Batch Job/Step |
| P4.4 | msa-user-service (user lib publish + service app) | ✅ | 기본 (1)-1 (필수) | dual delivery 패턴 |
| P4.5 | msa-auth-service 신규 레포 | ✅ | 기본 (2)-2 (필수) JWT + (ADR-0001) | gRPC `AuthService.{SignUp, SignIn, CheckValidity}` |
| P4.6 | msa-api-gateway 단일 모듈 BFF + SC Gateway 혼합 | ✅ | 기본 (2)-2 (필수) + (ADR-0005) | Spring Cloud 2025.0.2 Northfields |
| P4.7 | 6 매니페스트 values (dev/prod) | ✅ | (필수) | argocd-manifest 6 PR 머지 |
| R-04 | api-gateway 명칭 결정 | ✅ | (ADR-0005) | `msa-api-gateway` 신규 레포 |
| R-13 | Kafka JSON wire 유지 | ✅ | (ADR-0003) | Protobuf 마이그레이션 보류 |
| R-21 | order state machine 7-state 확장 | ✅ | (ADR-0006) | P4.2 동일 |
| R-22 | inventory Event Sourcing + Spring Batch | ✅ | 기본 (1)-5 (선택) + (ADR-0007) | P4.3 동일 |
| R-23 | api-gateway BFF + SC Gateway 라우팅 매트릭스 | ✅ | (ADR-0005) | application.yaml |
| R-24 | identification 모듈 폐기 영향 정리 | ✅ | — | 의존 흔적 제거 확인 |
| R-25 | auth-service JWT secret placeholder fallback (1차 fix) | 🔄 | 심화 (3)-2 (필수) 일부 | `fix/r-25-jwt-secret-placeholder`. Phase 5 ExternalSecrets 정식 해결 |
| R-39 | msa-frontend 신규 레포 (팀장님 작성) | ✅ | (필수) 프론트 | React 19.2 + Vite 8 + TS 6 + TanStack/MUI/Tailwind/Zustand. 10번째 polyrepo. 호스팅 배포는 R-40 |
| D1 | auth-service 신규 + notification 폐기 | ✅ | (ADR-0001) | msa-auth-service 신규 + msa-notification-service archive |
| D6 | notification-service archive | ✅ | (ADR-0001) | GitHub UI archive |

---

## Phase 5 — 플랫폼 배포 + 운영·보안·테스트 + 평가 필수 cover (진행 중)

> **목적**: Phase 4 까지 배포된 6 서비스 위에 stateful 백엔드 (Postgres / Redis / Kafka) + 운영 컴포넌트 (Istio / LGTM 스택 / AlertManager / ExternalSecrets) + 보안 컴포넌트 (RBAC / NetworkPolicy / Trivy / SonarCloud) + 단위/E2E 테스트를 모두 cover. 발표 평가 **기본 + 심화 필수 18 개** 를 100% 충족하는 것이 Phase 5 종료 조건.
>
> **분류**: (A) 매니페스트 / 코드 / 문서 — 이미 ✅ 또는 🔄. (B) cluster up 후 실 검증 — ⏸. (C) 외부 수동 셋업 — Slack 채널 등.

### (A) 매니페스트 / 코드 / 문서 작업

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| R-43 | PodDisruptionBudget templates (공통 차트) | ✅ | 심화 (1)-3 (필수) | `applications/charts/microservice/templates/pdb.yaml` + values 기본값. api: minAvailable=1, worker: maxUnavailable=1 |
| R-44 | api-gateway ↔ Istio Service Mesh 책임 분담 ADR | ✅ | 심화 (1)-2 (필수) | [ADR-0009](./adr/0009-api-gateway-istio-mesh-collaboration.md). 4 축 책임 분리 (north-south / east-west / BFF / 회복성 / JWT) |
| R-46 | Trivy config (K8s 매니페스트 스캔) | ✅ | 심화 (2)-2 (필수) | `.github/workflows/trivy-manifest-scan.yml`. SARIF → GitHub Security 탭 업로드 + Slack webhook step. PoC 단계 정보 수집 모드 (`exit-code: "0"`) — 운영 전환 시 `"1"` 로 승격. 누적 fix 3건: [TROUBLESHOOTING §4.6](./TROUBLESHOOTING.md) |
| R-48 | K8s RBAC 역할 분리 (사람 + 서비스) | ✅ | 심화 (3)-1·(3)-2 (필수) | `platform/05-rbac/` Sync Wave -8 + Helm 차트 `templates/role.yaml` |
| R-49 | NetworkPolicy 매니페스트 (default-deny + 의도 통신만 allow) | ✅ | 심화 (3)-3 (필수) | `platform/60-network-policies/` Sync Wave 5. market-{dev,prod} 각 6 정책 |
| R-50 | api-gateway Rate Limit (SC Gateway RequestRateLimiter + Redis Token Bucket) | ✅ | 기본 (2)-5 (선택) | msa-api-gateway PR #7. default-filters + order-admin 별도 보수적 |
| R-41 | api-gateway Response Aggregate Fallback (Resilience4j Circuit Breaker) | ✅ | 기본 (2)-3 (필수) + (2)-4 (선택) | msa-api-gateway PR #8. InventoryQueryService runCatching fallback. ADR-0009 책임 분담 |
| R-45 | SonarCloud (public 무료) CI 통합 + Quality Gate 정책 | ✅ | 심화 (2)-1 (필수) | 7 polyrepo CI sonar step + sonarcloud.io 셋업 + [ADR-0011](./adr/0011-sonarcloud-quality-gate-policy.md) `wait=false` 결정 + Hotspot Review 절차. 자세히 [TROUBLESHOOTING §4.5](./TROUBLESHOOTING.md) |
| R-47 | Slack `#security-report` 보안 알림 채널 (ADR + CI webhook + AlertManager) | 🔄 | 심화 (2)-3 (필수) | [ADR-0010](./adr/0010-security-alerting-strategy.md) ✅. Trivy workflow Slack step ✅. AlertManager severity=security 라우팅 매니페스트 ✅ (R-35 e 와 함께). 채널 생성 + Org Secret 등록 ⏸ (C) |
| R-57 | JUnit 단위 테스트 (7 polyrepo, ~48 케이스) | ✅ | 기본 (3)-1 (필수) | 7 PR 머지. 도메인 entity / state machine / JWT round-trip / CB Fallback. 작업 중 발견 3건: [TROUBLESHOOTING §1.8, §1.9](./TROUBLESHOOTING.md) |
| R-35 (A) | Platform 매니페스트 7 set 작성 (cert-manager → external-secrets-config) | ✅ | 기본 (1)-3·(1)-4 (필수/선택) + (3)-3·(3)-6 (필수/선택) + 심화 (1)-2·(2)-3·(3)-2 (필수) | PR #78 / commit `974e01d`. 매니페스트 22 파일 — `platform/{00-cert-manager, 10-istio-base, 11-istio-cp, 30-{kube-prometheus-stack, loki, tempo}, 40-{cnpg, strimzi, redis, external-secrets}-operator, 50-{kafka-cluster, postgres-clusters, redis-cluster}, 91-external-secrets-config}/`. Sync Wave -10 ~ 3 |
| R-03 (A) | Istio Gateway + VirtualService + PeerAuthentication + DestinationRule + namespace injection | ✅ | 심화 (1)-2 (필수) | PR #81 / commit `e7c35a8`. `platform/12-istio-gateway/{application.yaml, manifests/{gateway, virtualservice, peer-authentication, destination-rule, namespace-injection}.yaml}` 6 파일. Sync Wave -3. mesh-wide mTLS STRICT + market-{dev,prod} 각 outlierDetection |
| R-42 (A) | Postman Collection + Newman CI + Grafana 도메인 대시보드 | ✅ | 기본 (3)-2·(3)-3 (필수) + (3)-5 (선택) | PR #79 / commit `d2d371a`. `tests/e2e/troica-market-e2e.postman_collection.json` 7 step 시나리오 (실 `*RestControllerAdapter.kt` endpoint 기준) + `.github/workflows/e2e-newman.yml` (workflow_dispatch + 매일 KST 10:00 + paths) + `platform/30-kube-prometheus-stack/dashboards/troica-services.yaml` (req rate / 5xx 에러 / P99 latency / Kafka lag / JVM heap / CB state 6 panel) |

### (B) cluster up 후 실 검증 (R-35 platform 배포 후 진행)

> (A) 매니페스트는 모두 main 머지 ✅. cluster up 시 ArgoCD 가 Sync Wave 순으로 자동 reconcile. 본 섹션은 reconcile 후 실 작동 검증만 추적.

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| R-35 (B) | Platform 배포 reconcile 검증 (a~g) | ⏸ | (A) 와 동일 평가요소들의 실 검증 | (a) CNPG Cluster × 6 primary/replica up (b) RedisFailover × 2 (c) Kafka KRaft 3 broker + 4 KafkaTopic up + consumer lag exporter scrape (d) istio-cp pod + ingressgateway NodePort 30080/30443 (e) AlertManager → Slack #alerts / #security-report 라우팅 (f) Prometheus targets scrape + Grafana datasource up (g) ExternalSecret sync → Secret 생성. **Phase 5 본 작업, 비용 영향 大** |
| R-03 (B) | Istio mesh 실 작동 검증 | ⏸ | 심화 (1)-2 (필수) | (1) `istioctl proxy-status` 모든 pod SYNCED (2) Gateway 외부 진입 — curl → api-gateway 도달 (3) PeerAuthentication mTLS — `tcpdump` 으로 sidecar 간 통신 암호화 확인 (4) DestinationRule outlier ejection 시연 |
| R-25 | JWT secret 정식 외부화 (ExternalSecrets) | ⏸ | 심화 (3)-2 (필수) | R-33 / R-35 (g) 와 묶음. ExternalSecret → AWS Secrets Manager → Secret 생성 → auth-service / api-gateway 가 ENV 로 주입 |
| R-33 | 6×2 Secret + ConfigMap 실값 | ⏸ | 심화 (3)-2 (필수) | ExternalSecrets Operator + AWS Secrets Manager. 9 secrets (DB credentials × 6 + JWT × 2 + Slack webhooks + Grafana admin) |
| R-41 (B) | K8s 디스커버리 + 장애 격리 실 검증 | ⏸ | 기본 (2)-1·(2)-3 (필수) | (1) ClusterIP+DNS `kubectl exec` + nslookup + curl (2) Fallback 시연 (inventory Pod down → 빈 list 반환) |
| R-42 (B) | Postman + Newman E2E + Prometheus + Grafana 실 통과 | ⏸ | 기본 (3)-2·(3)-3 (필수) + (3)-5 (선택) | workflow_dispatch 로 baseUrl=Istio Gateway 지정 → 7 step 시나리오 통과. Grafana 6 panel 실 데이터 확인 |
| R-44 (B) | 독립 배포 E2E 무영향 검증 (Newman) | ⏸ | 심화 (1)-1 (필수) | 한 서비스만 image bump → 다른 서비스 트래픽 무영향 시연. Newman 으로 비교 |
| R-47 (B) | AlertManager `#security-report` 라우팅 실 배포 | ⏸ | 심화 (2)-3 (필수) | severity=security label PromRule 실 fire → #security-report 메시지 수신 확인 |
| R-48 (B) | RBAC 실 작동 검증 (`kubectl auth can-i`) | ⏸ | 심화 (3)-1·(3)-2 (필수) | developer / operator / sre 각 역할 검증 |
| R-49 (B) | NetworkPolicy 통신 차단 검증 | ⏸ | 심화 (3)-3 (필수) | `kubectl exec` 다른 서비스에서 curl 거부 확인 |
| R-50 (B) | RequestRateLimiter 실 작동 검증 | ⏸ | 기본 (2)-5 (선택) | k6 burst → 429 응답 확인 |

### (C) 외부 수동 셋업

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| R-47 | Slack `#security-report` 채널 생성 + webhook 발급 | ⏸ | 심화 (2)-3 (필수) | Slack workspace 작업 (5 분). Org Secret `SLACK_SECURITY_WEBHOOK_URL` 등록 |

---

## Phase 6 — 심화 옵션 + 운영 전환 + 발표 (대기)

> **목적**: 평가 **선택 항목** 으로 발표 임팩트 강화 + 운영 단계 진입을 위한 정리 작업 + 최종 발표 자료. 필수 항목은 Phase 5 종료 시점에 이미 cover 된 상태.

### (A) 심화 선택 평가요소 (발표 임팩트)

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| R-51 | KEDA (Kafka lag 기반 ScaledObject) | 📌 | 심화 (1)-4 (선택) | inventory consumer 가 `order.pending` lag 임계 도달 시 자동 확장 |
| R-52 | ArgoCD Rollouts Canary | 📌 | 심화 (1)-5 (선택) | order-service 등 1 서비스 단계별 분기 + 자동 롤백 |
| R-53 | OPA Gatekeeper | 📌 | 심화 (2)-5 (선택) | ConstraintTemplate (`runAsNonRoot` / `readOnlyRootFilesystem`) Namespace 전체 강제 |
| R-54 | OWASP ZAP DAST + SARIF | 📌 | 심화 (2)-4 (선택) | staging ZAP scan → GitHub Security 탭 |
| R-55 | Falco DaemonSet 런타임 탐지 | 📌 | 심화 (3)-5 (선택) | 비정상 shell 실행 등 → Slack `#security-report` |
| R-56 | Incident Response 5 단계 자동화 | 📌 | 심화 (3)-4 (선택) | 탐지 → 격리 → 분석 → 복구 → 회고 runbook + 각 단계 스크립트 |
| R-58 | Testcontainers 통합 테스트 | 📌 | 기본 (3)-4 (선택) | 7 polyrepo build.gradle.kts + 실 DB/Kafka 컨테이너로 통합 테스트 |

### (B) 운영 전환 + 정리

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| R-40 | msa-frontend 정적 호스팅 배포 (S3 + CloudFront + Route 53 + ACM) | ⏸ | (배포 마무리) | terraform 모듈 + CI build artifact S3 sync |
| R-08 | `msa-spring-boot` 모노레포 archive | ⏸ | (정리) | GitHub UI archive. 모든 서비스 polyrepo 이전 완료 후 |
| R-15 | GitHub Actions SHA pin + Dependabot/Renovate 정책 재검토 | ⏸ | (보안 강화) | R-27 (g) Dependabot 폐기 후 별도 가벼운 대안 |
| R-18 | Spring Boot 3.5 OSS EOL 2026-06-30 모니터링 | ⏸ | (운영) | 3.6 / 4.0 마이그레이션 검토 |

### (C) 발표 자료

| ID | Task | 상태 | 평가요소 | 산출물 |
|---|---|---|---|---|
| P6.1 | 발표 자료 (architecture diagram + decision log + 비용 분석 + SonarCloud Dashboard 캡쳐) | ⏸ | (발표) | KT Cloud Tech UP 2기 발표 |

---

## 비-Phase / 운영 + 코드 품질 / 폐기

> Phase 분류에 묶이지 않는 누적 운영 / CI 최적화 / 외부 의존성 모니터링 / 폐기된 결정.

### 운영 / 코드 품질

| ID | Task | 상태 | 산출물 |
|---|---|---|---|
| R-13 | Kafka wire JSON 유지 | ✅ | (ADR-0003). Protobuf 마이그레이션 보류 |
| R-16 | Kotlin 2.3.x + Gradle 9.x JetBrains 버그 모니터링 | 📌 | R-09 동일. 2.4.x 또는 2.3.22+ release 후 재시도 |
| R-20 | JitPack `client-redis` 버전 변경 모니터링 | 📌 | 팀장님 통제 외, monthly check |
| R-27 (a) | Gradle 중복 빌드 제거 (호스트 + Docker → Docker 단순화) | ✅ | 6 PR 머지. 빌드 ~5분 → 2분 25초. [TROUBLESHOOTING §4.3](./TROUBLESHOOTING.md) |
| R-27 (e) | `paths-ignore` (docs PR build skip) | ✅ | `msa-product-service` 적용. 5 서비스 + common-libs propagate 보류 |
| R-27 (b/c/d/f) | BuildKit cache / Trivy DB cache / Reusable workflow / ECR retry | 📌 | Phase 6 일괄 검토. (a) 검증 후 우선순위 낮음 |
| R-28 | netty 4.2.x BOM 전환 → CVE-2026-42577 해제 | 📌 | Spring Boot 3.5.15+ 또는 reactor-netty 1.3.x release 모니터링. 외부 의존성 |
| R-36 | ansible playbook + README 일본어 → 한국어 | ✅ | `i18n/ansible-japanese-to-korean`. 14 파일 일괄 |
| R-38 | common-libs `QuerydslConfig` final → Spring CGLIB proxy 실패 | ✅ | `fix/r-38-kotlin-spring-plugin` + v0.3.1 publish + 6 서비스 의존성 bump. [TROUBLESHOOTING §1.7](./TROUBLESHOOTING.md) |

### 폐기 (작업 안 하기로 결정)

| ID | Task | 상태 | 사유 |
|---|---|---|---|
| R-05 | notification-service 그린필드 | ❌ 폐기 | D6 — notification 도메인 폐기 (Grafana → Slack/PagerDuty 로 대체) |
| R-27 (g) | Dependabot 자동 활성화 | ❌ 폐기 | 첫 주에 7+ PR 폭주 → review 부담. Spring Boot BOM 으로 대부분 cover + Gradle bump 가 Kotlin 2.1 toolchain 결정과 충돌. [TROUBLESHOOTING §4.4](./TROUBLESHOOTING.md). 추후 Renovate 검토 (R-15) |

---

## SPEC 변경 이력

> SPEC.md 의 누적 변경 기록. 자세한 변경 사유는 git log + 본 BACKLOG 또는 TROUBLESHOOTING 참조.

| 일자 | SPEC 위치 | 변경 | 트리거 |
|---|---|---|---|
| 2026-05-12 | §2 디렉토리 트리 | `values-worker.yaml` 제거, worker = flag | R-01 |
| 2026-05-12 | §6 multi-source 예시 | `source:+sources:` 중복 정리 | R-02 |
| 2026-05-12 | §0 결정표 | Protobuf / Istio 행 추가 | R-10 / R-03 |
| 2026-05-12 | §1.2 공용 라이브러리 | 멀티모듈 4 서브모듈 명시 | Phase 2 |
| 2026-05-12 | §9.3 페이로드 스키마 | "Protobuf 또는 Avro" → "Protobuf only + no registry for PoC" | R-10 |
| 2026-05-12 | §12 msa-common-libs | 단일 → 멀티모듈 publish 패턴 + 빌드환경 § | Phase 2 |
| 2026-05-12 | §13.2 / §13.3 | Phase 작업 순서 확장 (멀티모듈 + Protobuf + worker + gradlew 100755) | Phase 2/3 |
| 2026-05-12 | §7 CI Trivy 버전 | `0.24.0` → `v0.36.0` + 공급망 사고 주석 | R-14 |
| 2026-05-12 | §0 / §12.0.1 | SB `3.3.0` → `3.5.13`, Spring Cloud 2025.0.2 명시 | CVE-2025-22235 / Q1 |
| 2026-05-12 | §13.3 step 7 | "SB 3.3.0 검증" → "SB 3.5.13 검증" | upgrade 결과 |
| 2026-05-12 | §7 / §7.1 | `AWS_DEPLOYMENTS_ENABLED` 게이트 + Variable 행 | R-19 |
| 2026-05-13 | §0 / §1 | Polyrepo 9개 → 10개 (msa-frontend 추가) | R-39 |

---

## 평가요소 매핑 (KT Cloud Tech UP 2기 발표)

> 발표 평가 기준 → BACKLOG task 역인덱스. **필수 항목 18 개 (기본 9 + 심화 9)** 모두 R-NN 매핑 필요.

### 기본 프로젝트

| 평가요소 | 필수/선택 | 대응 R / Phase | 상태 |
|---|---|---|---|
| (1)-1 이벤트 스토밍 + 독립 배포/확장/격리 | 필수 | P4.1 + Phase 4 polyrepo 전체 + W1 이벤트 스토밍 | ✅ |
| (1)-2 동기 REST vs 비동기 이벤트 적용 기준 | 필수 | ADR-0003 / ADR-0005 / PROJECT_PLAN §5.2 | ✅ |
| (1)-3 Database per Service | 필수 | Phase 4 전체 + PROJECT_PLAN §5.3 + R-35 (A) postgres-clusters 매니페스트 | ✅ |
| (1)-4 Kafka StatefulSet + 토픽 + 비동기 | 선택 | R-35 (A) kafka-cluster 매니페스트 ✅ + ADR-0004. cluster 검증 R-35 (B) ⏸ | 🔄 |
| (1)-5 Event Sourcing 1 서비스 | 선택 | R-22 / ADR-0007 (inventory) | ✅ |
| (2)-1 K8s ClusterIP + DNS 디스커버리 | 필수 | Helm chart Service template (P1.3) ✅ + R-41 (B) `kubectl exec` 시연 ⏸ | 🔄 |
| (2)-2 API GW + 경로 라우팅 + JWT 필터 | 필수 | ADR-0005 / P4.5 / P4.6 | ✅ |
| (2)-3 장애 격리 시나리오 설계 + 코드 | 필수 | **R-41** (A) 코드 머지 | ✅ |
| (2)-4 Resilience4j + Fault Injection | 선택 | **R-41** CB ✅ / Fault Injection 은 R-41 (B) 검증 | 🔄 |
| (2)-5 Rate Limit | 선택 | **R-50** | ✅ |
| (3)-1 JUnit 단위 + CI | 필수 | **R-57** (7 polyrepo ~48 케이스) | ✅ |
| (3)-2 Postman E2E (서비스 연계) | 필수 | **R-42 (A)** collection + workflow 머지 ✅ + R-42 (B) cluster 통과 ⏸ | 🔄 |
| (3)-3 Prometheus + Grafana | 필수 | **R-35 (A)** kube-prometheus-stack + R-42 (A) Grafana 대시보드 ConfigMap ✅ + cluster 검증 ⏸ | 🔄 |
| (3)-4 Testcontainers | 선택 | **R-58** | 📌 |
| (3)-5 Newman CLI in CI | 선택 | **R-42 (A)** `.github/workflows/e2e-newman.yml` ✅ + cluster 실 실행 ⏸ | 🔄 |
| (3)-6 Kafka consumer lag Exporter | 선택 | R-35 (A) Strimzi metrics + Grafana 대시보드 panel ✅ + cluster scrape ⏸ | 🔄 |

### 심화 프로젝트

| 평가요소 | 필수/선택 | 대응 R / Phase | 상태 |
|---|---|---|---|
| (1)-1 독립 배포 + E2E 무영향 검증 | 필수 | **R-44** (B) Newman 비교 검증 | 🔄 |
| (1)-2 API GW ↔ Service Mesh 협업 | 필수 | **R-44** ADR-0009 ✅ + **R-03 (A)** Istio 매니페스트 ✅ | ✅ |
| (1)-3 PodDisruptionBudget | 필수 | **R-43** | ✅ |
| (1)-4 KEDA | 선택 | **R-51** | 📌 |
| (1)-5 ArgoCD Rollouts Canary | 선택 | **R-52** | 📌 |
| (2)-1 SonarQube 80% 게이트 | 필수 | **R-45** + ADR-0011 | ✅ |
| (2)-2 Trivy config (매니페스트 스캔) | 필수 | **R-46** workflow 머지 (info-mode) | ✅ |
| (2)-3 Slack #security-report | 필수 | **R-47** ADR-0010 ✅ + R-46 webhook ✅ + R-35 (A) AlertManager 매니페스트 ✅ + 채널 생성 ⏸ (C) | 🔄 |
| (2)-4 OWASP ZAP DAST | 선택 | **R-54** | 📌 |
| (2)-5 OPA Gatekeeper | 선택 | **R-53** | 📌 |
| (3)-1 RBAC 역할 분리 | 필수 | **R-48** (사람 RBAC) | ✅ |
| (3)-2 ServiceAccount + Role/RoleBinding | 필수 | **R-48** (서비스 RBAC) + R-25 + R-33 + R-35 (A) ExternalSecret 매니페스트 | ✅ |
| (3)-3 NetworkPolicy + 차단 테스트 | 필수 | **R-49** (A) 매니페스트 ✅ / (B) 차단 검증 ⏸ | 🔄 |
| (3)-4 Incident Response 5 단계 | 선택 | **R-56** | 📌 |
| (3)-5 Falco DaemonSet | 선택 | **R-55** | 📌 |

**필수 18 개 cover 현황 (코드 직접 검증 기준)**
- 기본 필수 9 중 ✅ 6 / 🔄 3 (R-35 (B) Kafka cluster up, R-41 (B) DNS demo, R-42 (B) E2E run)
- 심화 필수 9 중 ✅ 6 / 🔄 3 (R-44 (B) Newman 비교, R-47 채널 생성, R-49 (B) 차단 검증)
- 합계: **✅ 12 / 🔄 6** — 모든 매니페스트 / 코드 / 문서 머지 완료. cluster up + Slack 채널 생성 → 18/18 ✅

(A) 매니페스트 작업이 모두 머지된 상태로 ⏸ 는 0. 🔄 6 건은 모두 "cluster up 후 자동 검증" 또는 "외부 5분 작업" 단위.

---

## 주요 의사결정 (ADR)

> 자세한 컨텍스트 / 대안 / 결과는 [docs/adr/](./adr/) 의 각 ADR 참조.

| ADR | 결정 | 관련 R / Q / D |
|---|---|---|
| [ADR-0001](./adr/0001-polyrepo-with-auth-service.md) | Polyrepo 구조 + auth-service 분리 + notification 폐기 | D1, D6, Q2, Q3, R-05 |
| [ADR-0002](./adr/0002-client-libraries-distribution.md) | client-redis 는 JitPack, client-ses 제거 | D2, D3, R-20 |
| [ADR-0003](./adr/0003-kafka-wire-format-json.md) | Kafka 직렬화 = JSON (Protobuf 마이그레이션 보류) | Q4, R-13 |
| [ADR-0004](./adr/0004-kafka-topic-naming.md) | Kafka 토픽명 = SPEC 표준 (`order.pending` 등) | Q5 |
| [ADR-0005](./adr/0005-api-gateway-bff-with-cloud-gateway.md) | api-gateway = BFF + Spring Cloud Gateway 혼합 | Q1, R-04, R-23 |
| [ADR-0006](./adr/0006-order-state-machine-extension.md) | order state machine 7-state 확장 | Q6, R-21 |
| [ADR-0007](./adr/0007-inventory-event-sourcing-batch-worker.md) | inventory = Event Sourcing + Spring Batch worker | Q7, R-22 |
| [ADR-0008](./adr/0008-order-terminal-events.md) | order.confirmed / cancelled 토픽 발행 유지 | D7 |
| [ADR-0009](./adr/0009-api-gateway-istio-mesh-collaboration.md) | api-gateway ↔ Istio Service Mesh 책임 분담 | R-44 |
| [ADR-0010](./adr/0010-security-alerting-strategy.md) | 보안 알림 채널 `#security-report` 분리 | R-47 |
| [ADR-0011](./adr/0011-sonarcloud-quality-gate-policy.md) | SonarCloud Quality Gate 정책 — 무료 plan + `wait=false` | R-45, R-57 |

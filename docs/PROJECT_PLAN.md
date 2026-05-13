# Market Service MSA 구축 프로젝트 제안 및 계획서 (2026-05-13 갱신본)

> **교육과정**: KT Cloud Tech UP 2기 (클라우드 네이티브)
> **팀명**: **Troica** (1팀)
> **팀원**: 김영호(팀장) · 김정엽 · 우문주
> **프로젝트명**: Market Service MSA 구축 프로젝트
> **프로젝트 기간**: 2026.04.24 ~ 2026.05.21 (약 4주)

> ⚠️ **본 문서는 원본 계획서를 실제 구현 기준으로 정정한 갱신본입니다.**
> 본문은 2026-05-13 시점 실제 코드/매니페스트/ADR과 일치하도록 갱신되었습니다. 원본 계획서와의 차이는 [부록 C — 계획서 대비 변경 요약](#부록-c-계획서-대비-변경-요약)에 정리되어 있습니다.
> **단일 진실의 원천**: [TROICA_SPEC.md](./TROICA_SPEC.md) · [BACKLOG.md](./BACKLOG.md) · [ADR](./adr/) · [다이어그램](./images/)

---

## 1. 프로젝트 배경 및 목표

### 1.1 배경

전통적인 모놀리식 커머스 시스템은 단일 데이터베이스와 단일 배포 단위에 의존하기 때문에, 트래픽이 증가하거나 일부 도메인에 장애가 발생하면 전체 서비스 가용성이 저하됩니다. 본 프로젝트는 **단일 데이터베이스 환경을 분산 추적·메시 네트워크 기반 MSA로 확장**하는 것을 목표로 합니다.

### 1.2 목표

- **기본 단계**: 클라우드 스토리지 및 분산 DB·MQ 클러스터링을 구축하고, 도메인별로 분리된 마이크로서비스를 동기/비동기 통신으로 연결한다.
- **심화 단계**: Service Mesh(Istio/Envoy)를 도입해 통신 보안·트래픽 제어·관측성을 강화하고, OpenTelemetry 기반의 분산 추적과 Grafana LGTM 스택으로 통합 가시성을 확보한다.

### 1.3 성공 기준

| 구분 | 지표 | 목표값 |
| --- | --- | --- |
| 가용성 | 단일 서비스 장애 시 부분 응답 성공률 | 95% 이상 |
| 성능 | 재고 조회 P99 latency | 50ms 이하 |
| 신뢰성 | Kafka 메시지 손실 | 0건 (Outbox로 At-least-once 보증) |
| 관측성 | 분산 트레이스 커버리지 | 전 서비스 100% |

---

## 2. 서비스 기획 및 소개

### 2.1 서비스 개요

**Market Service**는 일반적인 온라인 마켓플레이스의 핵심 도메인 — 회원/인증, 상품/재고, 주문/결제 — 을 마이크로서비스로 구성한 학습용 레퍼런스 시스템입니다. 단순한 CRUD 수준에 그치지 않고 **고동시성 재고 관리, 분산 트랜잭션, 이벤트 기반 통합** 등 실무에서 부딪히는 난제를 직접 다룹니다.

### 2.2 도메인 경계 (Bounded Context) — **6개 서비스**

이벤트 스토밍 후 실제 구현은 다음 6개 서비스로 경계가 정착되었습니다.

| 서비스 | 책임 | 주요 데이터 | 비고 |
| --- | --- | --- | --- |
| **user-service** | 회원 가입·프로필 관리 (REST only) | 회원 | 모노레포 `user/` 단독 |
| **auth-service** | 인증·JWT 발급/검증 (gRPC) | 인증 이력, refresh token | **신규 분리** (ADR-0001) |
| **product-service** | 상품 정보·카테고리 관리 (gRPC) | 상품, 카테고리 | |
| **inventory-service** | 재고 조회·차감 + **Event Sourcing + Spring Batch projection** (gRPC) | 재고(상태 DB) + 재고 이벤트(이벤트 스토어) | ADR-0007 |
| **order-service** | 주문 생성·**확장된 state machine** + Outbox + Saga 오케스트레이션 (gRPC) | 주문, Outbox | ADR-0006 (7-state) |
| **api-gateway** | 외부 진입점 — **BFF (gRPC client) + Spring Cloud Gateway 혼합** (HTTP) | (stateless) | ADR-0005 |

> 원본 계획서의 `notification-service`는 **폐기 (archive)** — 시스템 알림은 Prometheus → AlertManager → Slack/PagerDuty 경로로, 비즈니스 이벤트는 발행만 유지하되 현재 consumer 없음 (ADR-0001).
> 각 서비스는 **Database per Service** 원칙을 따라 독립 DB를 가지며, 데이터 공유는 **Kafka 이벤트**를 통해서만 이루어집니다. **서비스 간 직접 gRPC 호출 0건** — 모든 서비스가 gRPC server만 노출하고 client 설정이 없음 (api-gateway만 gRPC client).

---

## 3. 주요 기능

### 3.1 사용자 관점 기능

- 회원 가입·로그인 (JWT 기반)
- 상품 카탈로그 조회 / 검색
- 실시간 재고 조회
- 주문 생성 → 결제(시뮬레이션) → 배송(시뮬레이션) 상태 전이
- 운영 알림 (Grafana → Slack webhook) — 별도 notification 서비스 없음

### 3.2 시스템 관점 기능 (핵심)

- **고동시성 재고 차감**: Redis 단일 스레드 Lua 스크립트 원자적 연산
- **Event Sourcing**: 재고 변경 이력을 이벤트 저장소에 append-only로 보관, Spring Batch projection으로 상태 DB 반영 (ADR-0007)
- **Transactional Outbox**: 주문 생성 시 DB 트랜잭션과 Kafka 발행을 원자적으로 보장. 별도 `order-worker` Pod(`@Profile("worker")`)가 Outbox 폴링
- **Orchestration Saga**: 주문 → 재고 예약 → 결제(state) → 배송(state) → 확정/취소. **state machine 7-state**로 payment/shipping 마이크로서비스 분리 대신 시뮬레이션 (ADR-0006)
- **분산 락 (Redisson)**: 캐시 갱신 가환성(commutativity) 부재 문제 해결. Redisson 클라이언트는 **JitPack 외부 의존성** (`com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2`, ADR-0002)
- **Circuit Breaker (Resilience4j)**: 의존 서비스 장애 시 부분 응답 Fallback
- **Rate Limiting**: api-gateway에서 Token Bucket 기반 분당 요청 제한
- **API Gateway 인증**: `JwtHeaderCheckFilter`가 모든 요청에 auth-service `CheckValidity` gRPC 호출로 토큰 검증 (중앙화)

---

## 4. 개발 환경 (기술 스택 · 도구)

### 4.1 기술 스택

| 계층 | 채택 기술 | 선정 이유 / 실 구현 |
| --- | --- | --- |
| **언어/런타임** | **Java 21 (LTS) + Kotlin 2.1.0** | Virtual Thread 안정 지원. Kotlin 2.3.x는 `BuildUtilKt.clearJarCaches()` JetBrains 버그로 보류 (R-09/R-16) |
| **빌드 도구** | **Gradle 8.10.2** | Kotlin 2.1.0과 짝. 9.x는 동일 JetBrains 버그로 보류 |
| **백엔드 프레임워크** | **Spring Boot 3.5.13** | CVE-2025-22235 fix 포함. SB 3.5는 2026-06-30 OSS EOL — Phase 6 후 3.6/4.0 마이그레이션 검토 (R-18) |
| **API Gateway** | **Spring Cloud Gateway 2025.0.2 (Northfields) + WebFlux BFF 혼합** | **단일 deployable에서 일부 path는 SC Gateway routes로 reverse proxy(REST), 일부는 BFF가 gRPC client로 백엔드 호출** (ADR-0005). 모든 요청에 JWT 필터 적용 |
| **공통 라이브러리** | **`msa-common-libs`** (멀티모듈 Gradle: `common` + `events`) | JPA/QueryDSL/예외/유틸 + Protobuf 스키마. GH Packages publish (v0.3.1 최신) |
| **회복성** | **Resilience4j (Circuit Breaker, Retry, Bulkhead)** | Hystrix 대체 표준 |
| **관계형 DB** | **PostgreSQL** (서비스별 독립 인스턴스 × 6) | Read Replica·논리 복제 (Phase 5에서 CNPG Operator 적용 예정) |
| **캐시 / 분산 락** | **Redis** (Redisson 클라이언트) | Lua 원자 연산, RedLock. 클라이언트는 JitPack 외부 의존성 (ADR-0002) |
| **이벤트 스트리밍** | **Apache Kafka (KRaft 모드)** | 표준 비동기 메시징. 현재 single broker, Phase 5에서 Strimzi Operator + 3 broker cluster 전환 |
| **이벤트 직렬화** | **JSON wire** (Protobuf 코드젠 보유, wire 미사용) | ADR-0003 — 모노레포 호환 + 학습 부담 회피. Protobuf wire 마이그레이션은 SemVer breaking change로 추후 트랙 |
| **구성 관리 (CM)** | **Ansible (로컬 ansible-playbook)** | EC2 OS 부트스트랩 + kubeadm init/join + CNI(Calico) 설치 자동화. R-30: `kubeadm-flags.env` 직접 통합 (drop-in 미평가 우회) |
| **컨테이너 오케스트레이션** | **Kubernetes (EC2 self-managed, kubeadm)** | Master 3 + Worker 3 + Bastion 2 구성. SPOF·Split Brain을 직접 통제하며 학습 |
| **Container Runtime** | **containerd** | |
| **Service Mesh (심화)** | **Istio + Envoy** (Phase 5 예정) | 현 `msa-provisioning`의 Traefik은 Phase 5에서 제거 (R-03 / SPEC §0) |
| **계측 (Instrumentation)** | **OpenTelemetry SDK** (Phase 5 예정) | CNCF 표준, 벤더 중립 |
| **메트릭** | **Prometheus + Mimir** (Phase 5 예정) | 장기 보관·Multi-tenant |
| **트레이싱** | **Tempo** (Phase 5 예정) | LGTM 스택 통합, 비용 효율 |
| **로깅** | **Loki + Promtail** (Phase 5 예정) | 라벨 기반 인덱싱 |
| **시각화** | **Grafana** (Phase 5 예정) | 메트릭·로그·트레이스 단일 뷰 + Slack/PagerDuty webhook 알림 |
| **알림** | **AlertManager → Slack (warning/info) + PagerDuty (critical)** | 별도 notification 서비스 폐기 (ADR-0001) |
| **프론트엔드** | **React 18 + Vite + TypeScript** | 정적 호스팅 (S3 + CloudFront) — Phase 6 |
| **테스트** | **JUnit 5, Mockito, Testcontainers, Postman + Newman CLI** | Mock-less 통합 테스트 |
| **CI** | **GitHub Actions** | 빌드·단위 테스트·Trivy 스캔·**ECR push** (OIDC AssumeRole)·매니페스트 image tag bump (dev=direct commit, prod=PR) |
| **컨테이너 레지스트리** | **AWS ECR × 6** (`msa/<service>`) | GitHub OIDC Provider 기반 (long-lived AWS key 미사용). KMS 암호화, IMMUTABLE tag, 30-image lifecycle |
| **CD (GitOps)** | **Argo CD** | 단일 GitOps 도구로 플랫폼 + 마이크로서비스 모두 관리. 플랫폼은 App-of-Apps + Sync Wave, 마이크로서비스는 ApplicationSet Git Generator (6 svc × 2 env = 12 Application 자동 생성) |

### 4.2 개발 도구

| 분류 | 도구 |
| --- | --- |
| IDE | IntelliJ IDEA Community |
| 협업 | Notion, Discord, GitHub Projects |
| API 설계 | OpenAPI 3.1, Swagger UI, Protobuf (gRPC IDL) |
| 형상 관리 | GitHub (**Polyrepo 9개** + Trunk-based + PR 리뷰) |
| 인프라 코드 | Terraform, Ansible, Helm |
| 부하 테스트 | k6 (Phase 6 예정) |
| 다이어그램 | Mermaid (모두 `docs/images/`에 모음 — [AWS-architecture](./images/AWS-architecture.md), [GitOps-flow](./images/GitOps-flow.md), [MSA-services](./images/MSA-services.md)) |

### 4.3 레포지토리 구조 — Polyrepo 9개

GitHub Organization: `KTCloud-CloudNative-Troica-Team`

| # | 레포 | 역할 |
| --- | --- | --- |
| 1 | [msa-user-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-user-service) | 회원 도메인 (REST :8004) |
| 2 | [msa-auth-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-auth-service) | 인증 도메인 (gRPC :9005) — **신규** (ADR-0001) |
| 3 | [msa-product-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-product-service) | 상품 도메인 (gRPC :9001) |
| 4 | [msa-order-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-order-service) | 주문 도메인 (gRPC :9002 + admin REST :8002) + Outbox worker |
| 5 | [msa-inventory-service](https://github.com/KTCloud-CloudNative-Troica-Team/msa-inventory-service) | 재고 도메인 (gRPC :9003) + Spring Batch worker |
| 6 | [msa-api-gateway](https://github.com/KTCloud-CloudNative-Troica-Team/msa-api-gateway) | 외부 진입점 (HTTP :8100) |
| 7 | [msa-common-libs](https://github.com/KTCloud-CloudNative-Troica-Team/msa-common-libs) | 공용 라이브러리 (common + events Protobuf) |
| 8 | [msa-argocd-manifest](https://github.com/KTCloud-CloudNative-Troica-Team/msa-argocd-manifest) | GitOps 매니페스트 (단일 진실의 원천) |
| 9 | [msa-provisioning](https://github.com/KTCloud-CloudNative-Troica-Team/msa-provisioning) | Terraform + Ansible |

> archive: ~~`msa-notification-service`~~ (ADR-0001) · `msa-spring-boot` 모노레포 (Phase 6 archive 예정, R-08)

---

## 5. 아키텍처

### 5.1 AWS 배포 아키텍처

상세 다이어그램: [docs/images/AWS-architecture.md](./images/AWS-architecture.md) (Mermaid, 7개 섹션)

```
┌──────────────────────────────────────────────────────────────────────────┐
│  AWS Account (영구 자원: IAM OIDC + IAM Role + KMS + ECR × 6)            │
│                                                                          │
│  ┌────────────────────── VPC kt-cloud-vpc (10.0.0.0/16) ────────────┐    │
│  │                              IGW                                 │    │
│  │                               │                                  │    │
│  │  ┌────── AZ ap-northeast-2a ──┴─┐  ┌── AZ ap-northeast-2b ────┐  │    │
│  │  │ Public 10.0.1.0/24           │  │ Public 10.0.3.0/24       │  │    │
│  │  │  NAT-a · bastion-a · NLB EIP │  │  NAT-b · bastion-b · EIP │  │    │
│  │  │ ◀──────── NLB :6443 ─────────┼──┼──────────────────────────┤  │    │
│  │  │                              │  │                          │  │    │
│  │  │ Private 10.0.2.0/24          │  │ Private 10.0.4.0/24      │  │    │
│  │  │  master-1, master-2,         │  │  master-3, worker-2,     │  │    │
│  │  │  worker-1                    │  │  worker-3                │  │    │
│  │  │  + EBS × 1 (worker)          │  │  + EBS × 2 (worker)      │  │    │
│  │  │  + EFS Mount Target          │  │  + EFS Mount Target      │  │    │
│  │  │  + VPCE ENI (ecr.api/dkr)    │  │  + VPCE ENI (ecr.api/dkr)│  │    │
│  │  └──────────────────────────────┘  └──────────────────────────┘  │    │
│  │                                                                  │    │
│  │  VPC Endpoints: ecr.api · ecr.dkr (Interface) + s3 (Gateway)     │    │
│  │  EFS: kt-cloud-cluster-efs (리전 스코프)                         │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

**핵심 설계 결정**

- **Master 노드 3중화** — SPOF 방지 + **Split Brain 차단** (과반수 쿼럼)
- **Multi-AZ 배포** — AZ 단위 장애에도 가용성 유지
- **Private Subnet 내 워크로드 격리** — master/worker 모두 비공개 망. NLB(공개)는 control-plane 6443 노출만
- **NAT Gateway × 2** — 각 AZ 독립 (장애 격리 + 비용 트레이드오프)
- **VPC Endpoint** — ECR API/DKR(Interface) + S3(Gateway). 외부 인터넷 경유 없이 ECR pull
- **kubelet ECR pull** — instance profile (`AmazonEC2ContainerRegistryReadOnly`) + `ecr-credential-provider` 바이너리 (Go 빌드 R-29). kubelet args는 `kubeadm-flags.env` 직접 통합 (R-30)
- **CI 경로 ECR push** — GitHub Actions OIDC AssumeRole (계정 레벨, long-lived key 미사용)
- **영구/임시 자원 분리** — IAM/KMS/ECR/VPC는 `prevent_destroy=true` (월 ~$1), EC2/NAT/NLB/EFS는 destroy 사이클 (월 ~$300)
- **S3 backend** — Terraform 외부 수동 생성 (`troica-tfstate-<SUFFIX>`)

> 외부 진입은 **현재 NLB :6443 (kube-apiserver)만 노출** — 애플리케이션 트래픽용 ALB는 Phase 5에서 Istio Gateway로 대체 예정 (R-03).

### 5.2 MSA 통신 구조

상세 다이어그램: [docs/images/MSA-services.md](./images/MSA-services.md)

| 통신 패턴 | 적용 기준 | 사용 기술 |
| --- | --- | --- |
| **동기 HTTP (외부)** | 클라이언트 → api-gateway 진입 | WebFlux + JWT 검증 |
| **동기 gRPC (BFF → 백엔드)** | api-gateway → auth/product/order/inventory | Protobuf |
| **동기 REST 패스스루** | api-gateway → user-service, /admin/v1/orders/** → order-service | SC Gateway routes |
| **비동기 이벤트 (서비스 간)** | 도메인 이벤트 전파 — **inter-service 통신의 유일한 경로** | Kafka 토픽 (JSON wire) |

> **서비스 간 직접 gRPC 호출 없음**. 모든 서비스가 `grpc.server.port`만 설정하고 client 설정이 없음 — 진정한 decoupling.

**주요 토픽** (ADR-0004 표준명)

- `order.pending` — 주문 발생, 재고 차감 요청 (order Outbox worker → inventory consumer)
- `order.inventory-reserved` — 재고 예약 완료 응답 (inventory → order consumer)
- `order.confirmed` — 주문 확정 (terminal)
- `order.cancelled` — 주문 취소 (terminal)

> 원본 계획서의 `notification.requested` 토픽은 폐기 (consumer 없음, ADR-0001).

**api-gateway 라우팅 매트릭스** (ADR-0005)

| Path | 패턴 | 대상 |
| --- | --- | --- |
| `/api/v1/auth/**` | BFF (gRPC) | auth-service :9005 |
| `/api/v1/products/**` | BFF (gRPC) | product-service :9001 |
| `/api/v1/orders/**` | BFF (gRPC) | order-service :9002 |
| `/api/v1/inventory/**` | BFF (gRPC) | inventory-service :9003 |
| `/api/v1/users/**` | SC Gateway 패스스루 (REST) | user-service :8004 |
| `/admin/v1/orders/**` | SC Gateway 패스스루 (REST) | order-service :8002 (admin) |

### 5.3 데이터 아키텍처

- **Database per Service**: 서비스별 독립 PostgreSQL 인스턴스 6개 (auth-db, user-db, product-db, order-db + outbox, inventory-db = event store + snapshot)
- **Read Replica**: product/inventory에 적용 예정 (Phase 5)
- **Redis**: auth-service (refresh token), inventory-service (stock cache) — Phase 5에서 Redis Cluster 3 master + 3 replica 전환
- **Kafka**: 현재 single broker (KRaft), Phase 5에서 Strimzi Operator + StatefulSet 3 broker
- **Object Storage(S3)**: 상품 이미지·정적 자산 (Phase 6) + S3 Lifecycle → Glacier 이관
- **EFS**: 공유 로그·설정 (현재 미사용, Phase 5 활용)
- **EBS**: StatefulSet PVC (worker 노드 × 3에 20GB씩)
- **암호화**: 모든 EBS/S3/EFS/ECR는 **AWS KMS** Customer Managed Key로 SSE 암호화

### 5.4 정적 웹 호스팅 (Phase 6 예정)

- React 빌드 산출물 → **S3 + CloudFront**
- **Route 53** 도메인 연결
- HTTPS는 ACM 인증서 사용

### 5.5 배포 파이프라인 (환경 부트스트랩 + GitOps 런타임)

본 프로젝트는 **환경 부트스트랩**과 **런타임 배포**를 명확히 분리하고, 각 단계마다 책임이 다른 도구를 사용한다. CI/CD에서도 마찬가지로, CI는 GitHub Actions가 담당하고 CD(클러스터 동기화)는 **Argo CD가 단독으로 책임**진다. 클러스터에 직접 `kubectl apply`하는 경로는 존재하지 않으며, **모든 런타임 변경은 Git을 통해서만 발생**한다.

#### 5.5.1 환경 부트스트랩 (0 → 운영 가능 클러스터)

```
[Step 0] 수동 (1회)                       [Step 1] Terraform
─────────────────────                     ────────────────────────────────
S3 backend bucket 생성                    terraform apply
(troica-tfstate-<SUFFIX>)                    ├─ 영구 자원 (prevent_destroy):
                                             │   IAM OIDC + IAM Role + KMS
                                             │   + ECR × 6 + VPC + Subnet × 4
                                             │   + Route Tables + SG × 4
                                             │
                                             └─ 임시 자원 (destroy-temp.sh 대상):
                                                 EC2 × 8 (master 3 + worker 3 + bastion 2)
                                                 + EBS × 3 + NAT × 2 + NLB
                                                 + VPC Endpoint × 3 + EFS

                            ▼

[Step 2] Ansible (로컬)                    [Step 3] ArgoCD root sync
─────────────────────────                  ─────────────────────────────
ansible-playbook main.yaml                 자동: argocd-setup.yaml이
   │                                       msa-argocd-manifest의 root-app.yaml
   ▼                                       을 fetch + apply (R-32)
EC2 OS 부트스트랩                              │
 · containerd 설치                             ▼
 · swap off, kernel module, sysctl         App-of-Apps 자동 전개
 · kubeadm / kubelet / kubectl 설치         · projects (AppProject × 2)
 · kubeadm init  (master-1)                 · platform (Sync Wave):
 · kubeadm join (master-2/3, worker-1~3)      cert-manager → Istio CRD → Istio CP
 · CNI (Calico) 설치                          → LGTM 스택 → Strimzi → CNPG → Redis Op
 · ecr-credential-provider 배포 (R-29)      · applications (ApplicationSet):
 · ArgoCD 설치 (argocd-setup.yaml)            6 service × 2 env = 12 Application 자동 생성
```

> **DR 시나리오 = Step 1~3 그대로 재실행**. 빈 AWS 계정 (+ Step 0의 S3 backend bucket) 과 매니페스트 리포 URL만 있으면 전체 환경이 복원된다. Step 3에서 ArgoCD CLI 명령 없이 ansible playbook이 root-app.yaml을 가져와 적용 — 클러스터에 직접 손대는 일조차 자동화 (R-31, R-32).

#### 5.5.2 도구별 책임 (환경 부트스트랩)

| 도구 | 책임 | 다루는 대상 |
| --- | --- | --- |
| **Terraform** | AWS 인프라 프로비저닝 | VPC, Subnet, EC2, EBS, EFS, S3, KMS, ECR, IAM(OIDC/Role), SG, NLB |
| **Ansible (로컬 실행)** | OS 레벨 구성 + K8s 부트스트랩 + ArgoCD 부트스트랩 | 패키지(containerd, kubeadm, kubelet), 커널 파라미터, swap off, kubeadm init/join, CNI 설치, join token, **ecr-credential-provider 배포 (R-29)**, **ArgoCD 설치 + root-app.yaml 적용 (R-32)** |
| **kubeadm** | K8s 컨트롤 플레인 부트스트랩 | (Ansible이 호출하는 도구) |
| **Argo CD** | 클러스터 *내부*의 GitOps CD | 마이크로서비스 + 플랫폼 컴포넌트 + AppProject |

#### 5.5.3 런타임 배포 흐름 (CI + GitOps)

상세 다이어그램: [docs/images/GitOps-flow.md](./images/GitOps-flow.md)

```
개발자 push ──▶ 6개 polyrepo (소스 리포 각각)
                   │
                   ▼  GitHub Actions (CI, 각 repo)
            빌드 · 단위 + Testcontainers 통합 테스트 · Trivy CVE 스캔
                   │
                   ▼  OIDC AssumeRole → ECR push
            AWS ECR (msa/<service>:main-<sha>)
                   │
                   ▼  manifest auto-bump
            msa-argocd-manifest (GitOps SoT)
              ├─ dev: values-dev.yaml 직접 commit  → 즉시 sync
              └─ prod: values-prod.yaml PR 생성    → 승인 후 sync
                   │
                   ▼  poll / webhook
                Argo CD ──sync──▶ Kubernetes Cluster
                                    ├─ App Layer       (ApplicationSet · Git Generator)
                                    │   └─ user / auth / product / order / inventory / api-gateway
                                    │       × {dev, prod} = 12 Application
                                    │
                                    └─ Platform Layer  (App-of-Apps + Sync Wave) ⏸ Phase 5
                                        cert-manager → Istio CRD → Istio CP
                                        OTel Collector · Prometheus · Mimir · Tempo · Loki · Grafana
                                        AlertManager · Strimzi(Kafka) · Redis Operator · CNPG(PostgreSQL)
                                        ExternalSecrets Operator (R-25/R-33)
```

**Argo CD를 두 가지 용도로 동시 사용**

| 구분 | 관리 대상 | 채택 패턴 | 근거 |
| --- | --- | --- | --- |
| **애플리케이션 CD** | 마이크로서비스 6개 × 환경 2개 = 12 Application | **ApplicationSet** + Git Generator (matrix) | 동일한 구조의 앱을 환경(dev/prod)별로 일괄 생성 — 동질적인 앱에 권장 |
| **클러스터 CD (Cluster Bootstrap GitOps)** | 플랫폼 컴포넌트 — Istio, LGTM, AlertManager, Strimzi, CNPG, Redis Operator, ExternalSecrets 등 | **App-of-Apps** + **Sync Wave** | 컴포넌트마다 unique한 Helm values와 *명시적 의존성 순서* 필요 — 이질적 플랫폼 컴포넌트에 권장 |

> 두 패턴을 혼용하는 것이 2026 기준 ArgoCD 공식 모범 사례. Troica 프로젝트는 마이크로서비스가 동질적이고(같은 Spring Boot 베이스, 같은 Helm 차트 구조), 플랫폼은 의존성 순서(예: cert-manager → Istio CRD → Istio control plane → 마이크로서비스)가 중요하므로 이 권장 구성이 그대로 들어맞는다.

**Sync Policy 표준화**

- `automated.prune: true` — Git에서 사라진 리소스는 클러스터에서도 제거
- `automated.selfHeal: true` — 클러스터에 수동 변경이 있어도 Git 상태로 자동 복원
- `syncOptions: [CreateNamespace=true, ServerSideApply=true]`
- 리소스 삭제 보호용 `finalizer: resources-finalizer.argocd.argoproj.io`

**왜 "클러스터 CD까지" 한 도구로 묶는가**

- 클러스터 재구축(DR) 시 **빈 클러스터 + 매니페스트 리포 URL 하나만 있으면** 전체 환경이 복원된다.
- 운영자가 클러스터에 직접 손대지 않으므로 *"클러스터에서 본 상태 ≠ Git 상태"* 라는 drift 자체가 사라진다.
- 모든 변경이 PR을 통과하므로 감사 추적과 권한 분리가 자연스럽다.

---

## 6. 핵심 플로우

### 6.1 재고 조회 — 캐시 히트

```
사용자 ──재고조회──▶ inventory-service ◀── 캐시히트 ── Redis(Inventory)
       ◀─재고정보──
                       │
                       ▼ (캐시 미스 시 조회)
                    PostgreSQL(Inventory)
```

> **핵심**: 캐시 히트 시 재고 조회 처리량을 매우 높게 유지.

### 6.2 재고 조회 — 캐시 미스

```
                       ┌─ 분산 락 (Redisson) ─┐
                       │   재고 락 획득       │
                       ▼                      │
사용자 ──재고조회──▶ inventory-service ──캐싱──▶ Redis(Inventory)
       ◀─재고저장──         │ ◀── 캐시 미스 ──┘
                            │
              ┌─재고정보─────┴────── 미반영 이벤트 조회 ──┐
              ▼                                         ▼
       PostgreSQL(Inventory)                 PostgreSQL(InventoryEvent)
```

> **핵심**
> - 캐시 미스 시 **상태 DB(소스 데이터) + 이벤트 스토어(미반영 이벤트)** 두 정보를 조합해 *현재 재고*를 계산.
> - Redis 캐시 갱신은 **가환성(commutativity)이 없으므로** 분산 락이 반드시 필요.

### 6.3 재고 감소 (가장 중요한 핫스팟)

```
                       ┌── 멱등성 키 ──┐
                       │  주문·재고    │
                       │  멱등성 정보  │
                       ▼               │
order.pending ─소비자─▶ inventory-service ─생산자─▶ order.inventory-reserved
                          │       │
                       재고감소  이벤트생성
                          ▼       ▼
                   Redis(Inventory)  PostgreSQL(InventoryEvent)
```

> **설계 포인트**
> - 관계형 DB의 격리 수준에 의존하지 않고, **Redis 단일 스레드 + Lua 스크립트**의 원자적 연산으로 차감.
> - **Event Sourcing** 패턴이라 RDB는 *append-only* IO만 발생 → 매우 높은 처리량 + Redis로 낮은 latency 동시 달성.
> - Lua 원자 연산 덕분에 **이 경로에서는 분산 락이 불필요**.
> - Kafka는 *at-least-once* 보증이므로 **멱등성**이 필수. Redis 멱등성 키로 중복 처리 방지.

### 6.4 재고 배치 처리 (Spring Batch worker)

```
PostgreSQL ─이벤트 갱신─▶ │              │ ◀── 재고 조회 ── PostgreSQL
(InventoryEvent) ─이벤트 조회─▶ inventory-batch ── 재고 갱신 ─▶ (Inventory)
                                  │
                               캐시 갱신
                                  ▼
                            Redis(Inventory)
                                  ▲
                                  │ 분산 락 (재고 락 획득)
                              Redisson
```

> **설계 포인트** (ADR-0007)
> - Event Sourcing으로 누적된 이벤트 이력을 **Spring Batch Job**이 상태 DB에 머지(materialize).
> - inventory-service 코드베이스 내 `@Profile("worker")` 분기 + 별도 K8s Deployment.
> - Redis 캐시는 가환적이지 않아 분산 락 필요.
> - **캐시 값에 이벤트 시퀀스 번호(seq)를 함께 저장**하여 늦게 도착한 배치가 최신 캐시를 덮어쓰는 사고를 방지 (Redisson non-fair 분산 락 함정 우회).

### 6.5 주문 생성 — Transactional Outbox

```
사용자 ──주문발행──▶ order-service ──Outbox(동일 트랜잭션)──▶ PostgreSQL
       ◀─발행응답──                                       (OrderInventoryRequestOutbox)
```

> 주문 도메인은 **Orchestration Saga의 허브**. 주문 INSERT와 메시지 발행을 같은 트랜잭션 안에서 보장하기 위해 **Transactional Outbox 테이블**에 메시지를 함께 기록.

### 6.6 주문 메시지 발신 — Outbox Poller (별도 Pod)

```
PostgreSQL                       Kafka
(OrderInventoryRequestOutbox) ──▶ order-worker ──생산자──▶ order.pending
```

> 별도의 **order-worker Pod** (`@Profile("worker")` + 별도 K8s Deployment, `web-application-type=none` + `grpc.server.port=-1`) 가 Outbox를 5초 주기로 폴링하여 Kafka로 발행 → **At-least-once 메시지 보증**. 컨슈머 측에서는 위 6.3의 멱등성 키로 중복을 흡수.

### 6.7 주문 상태 머신 — 7-state 확장 (ADR-0006)

```
CREATED → PENDING → PAID → SHIPPING → SHIPPED → CONFIRMED
              │       │       │           │
              └───────┴───────┴───────────┴──→ CANCELLED (보상)
```

> 별도의 payment-service / shipping-service를 만들지 않고 order-service의 **state machine 확장**으로 시뮬레이션. 각 terminal 상태(`CONFIRMED`/`CANCELLED`)는 Kafka 토픽 발행 (ADR-0008).

### 6.8 장애 격리 시나리오 (Resilience4j)

의존 서비스 다운 시, Circuit Breaker가 OPEN 상태로 전환되어 **재고 정보 없이도 부분 응답을 반환**합니다 (Response Aggregate 패턴). 실제 장애 주입(Chaos)을 통해 동작을 검증.

### 6.9 JWT 검증 흐름 (ADR-0005)

```
모든 요청 ──▶ api-gateway ──JwtHeaderCheckFilter──▶ auth-service.CheckValidity (gRPC :9005)
                              ◀── valid/invalid ──
                              │
                              ▼ valid 시
                         BFF/SC Gateway 라우팅 → 백엔드
```

> JWT 검증은 매 요청마다 auth-service에 위임 (중앙화). JWT secret은 auth-service에만 존재 (보안 영역 격리).

---

## 7. 백로그 구성

### 7.1 Epic 단위

| Epic | 단계 | 주요 스토리 | 진행 상태 |
| --- | --- | --- | --- |
| **E1. 인프라 부트스트랩** | 기본 | VPC·Subnet·EC2·KMS·S3·ECR·IAM OIDC Terraform, Ansible 플레이북으로 OS 부트스트랩 + kubeadm 자동화 + Calico 설치, EFS/EBS 마운트 검증 | ✅ Phase 0 |
| **E2. GitOps 부트스트랩** | 기본 | Argo CD 설치, 매니페스트 리포 구조(`platform/`, `applications/`, `projects/`), App-of-Apps Root + Sync Wave, ApplicationSet Git Generator로 12 Application 일괄 생성, Auto-sync/Self-heal/Prune 정책 표준화 | ✅ Phase 1 |
| **E3. 분산 데이터 계층** | 기본 | PostgreSQL HA (CNPG Operator), Redis Cluster, Kafka KRaft StatefulSet, DR/백업 정책 | ⏸ Phase 5 (R-35) |
| **E4. 도메인 서비스 구현** | 기본 | **6개 서비스** user/auth/product/order/inventory/api-gateway (계획서 5개 → 실제 6개) | ✅ Phase 2-4 |
| **E5. 게이트웨이·인증** | 기본 | api-gateway (BFF + SC Gateway 혼합, ADR-0005), JWT 필터, Rate Limit | ✅ Phase 4 |
| **E6. 이벤트 통합** | 기본 | Outbox(order), Saga(7-state), Event Sourcing(inventory) | ✅ Phase 3-4 |
| **E7. 회복성** | 기본 | Resilience4j Circuit Breaker, Fallback, Chaos 검증 | ⏸ Phase 5 |
| **E8. 정적 웹** | 기본 | React + S3 + CloudFront + Route 53 | ⏸ Phase 6 |
| **E9. 테스트 자동화** | 기본 | JUnit, Testcontainers, Postman + Newman in CI | 🔄 부분 (Phase 5/6) |
| **E10. Service Mesh** | 심화 | Istio 설치 (Argo CD App-of-Apps로 관리, Sync Wave -5/-4), Envoy Sidecar 주입, mTLS, 트래픽 시프트 | ⏸ Phase 5 (R-03) |
| **E11. 관측성** | 심화 | OpenTelemetry, Prometheus, Tempo, Loki, Mimir, Grafana 대시보드 | ⏸ Phase 5 |
| **E12. 알림** | 심화 | Kafka consumer lag 알림, AlertManager → Slack/PagerDuty (별도 notification 서비스 폐기, ADR-0001) | ⏸ Phase 5 |

### 7.2 우선순위 기준

**MoSCoW** 방식 — Must(기본 E1~E8) → Should(E9~E10) → Could(E11) → Won't(이번 스프린트 제외)

### 7.3 Phase 진행 (2026-05-13 기준)

| Phase | 상태 | 산출물 |
| --- | --- | --- |
| **Phase 0** AWS 인프라 + ECR + 부트스트랩 | ✅ | terraform apply/destroy 사이클, OIDC ECR push, ArgoCD ApplicationSet 12 app 자동 생성 검증 |
| **Phase 1** ArgoCD 매니페스트 골격 | ✅ | bootstrap/projects/applications/platform 구조, 공통 Helm 차트 |
| **Phase 2** common-libs 멀티모듈 + Protobuf | ✅ | v0.3.1 publish, kotlin-spring plugin (R-38) |
| **Phase 3** order-service polyrepo + worker | ✅ | 모노레포 추출 + worker 분기 + SB 3.5.13 upgrade |
| **Phase 4** 6개 서비스 polyrepo | ✅ | product/order/inventory/user/auth/api-gateway 모두 polyrepo 완료 |
| **Phase 5** Platform 배포 (Postgres/Redis/Kafka/Istio/ExternalSecrets) | ⏸ | R-35 본 작업 |
| **Phase 6** 정리 + 발표 | ⏸ | 모노레포 archive (R-08), 발표 자료, SB EOL 대응 |

상세는 [BACKLOG.md](./BACKLOG.md) 참조.

---

## 8. 역할 분담

| 영역 | 1차 담당 | 2차(리뷰/페어) | 비고 |
| --- | --- | --- | --- |
| **인프라 / Kubernetes / Terraform / IaC / GitOps** | 우문주 | 김정엽 | EC2 K8s(kubeadm)·VPC·KMS·EFS/EBS·ECR, Ansible 플레이북 + Argo CD + Helm 차트 |
| **Service Mesh / Observability(LGTM)** | 김영호 | 김정엽 | Istio, OTel, Tempo, Grafana |
| **백엔드(주문·재고·이벤트 처리)** | 김정엽 | 김영호 | Outbox, Saga(7-state), Event Sourcing, Redisson |
| **백엔드(회원·인증·상품·게이트웨이)** | 김정엽 | 우문주 | JWT(auth-service 분리), Rate Limit, Circuit Breaker, BFF+SC Gateway 혼합 |
| **프론트엔드(React) / 정적 호스팅** | 김영호 | 우문주 | S3+CloudFront, Route 53 |
| **테스트 자동화 / CI(GitHub Actions)** | 우문주 | 김영호 | JUnit, Testcontainers, Newman, Trivy, OIDC ECR push |
| **문서·발표 자료** | 전원 | — | 주차별 회고 + 최종 발표 |

> 페어 프로그래밍과 코드 리뷰는 PR 단위로 *최소 1인 승인* 원칙을 준수합니다.

---

## 9. 프로젝트 일정 (4주)

### 9.1 마일스톤

| 주차 | 기간 | 마일스톤 | 산출물 | 상태 |
| --- | --- | --- | --- | --- |
| **W1** | 04.24 ~ 04.30 | 설계·환경 구축 | 이벤트 스토밍, 아키텍처 다이어그램, Terraform skeleton, Ansible 인벤토리 + 플레이북으로 K8s 클러스터 부트스트랩, Argo CD 설치 + 매니페스트 리포 초기 구조 + Root Application | ✅ Phase 0-1 |
| **W2** | 05.01 ~ 05.07 | 기본 데이터·서비스 구현 | PostgreSQL/Redis/Kafka 클러스터, 6개 서비스 polyrepo 추출 + image push 검증 | ✅ Phase 2-3 |
| **W3** | 05.08 ~ 05.14 | MSA 핵심 통합 | order/inventory state machine + saga, Circuit Breaker, api-gateway BFF+SC Gateway 혼합, JWT, 모든 polyrepo CI/CD 검증 | ✅ Phase 4 + 다이어그램 |
| **W4** | 05.15 ~ 05.21 | 심화·통합·발표 | Istio·OTel·Grafana LGTM (플랫폼 레이어 Argo CD 등록), Newman E2E, AlertManager, 최종 발표 | ⏸ Phase 5 진행 예정 |

### 9.2 데일리/위클리 리듬

- **Daily Standup**: 평일 10:00 (15분)
- **Weekly Demo & Retro**: 매주 금요일 16:00
- **Code Freeze**: 5월 19일(화)
- **최종 발표**: 5월 21일(목)

---

## 10. 위험 요소 및 대응

| 위험 | 영향 | 완화책 | 상태 |
| --- | --- | --- | --- |
| Istio 학습 곡선 | 높음 | W3 말부터 사이드카 자동 주입 우선, 정책은 W4 | ⏸ Phase 5 |
| ArgoCD App-of-Apps + Sync Wave 학습 곡선 | 중 | W1 단일 Application으로 시작, W2부터 점진 도입 | ✅ Phase 1 검증 |
| Kafka consumer lag 폭증 | 중 | Prometheus Kafka Exporter + 사전 알림 임계값 | ⏸ Phase 5 |
| Outbox 폴링 지연 | 중 | 폴링 주기 5초, 인덱스 최적화, 워커 수평 확장 | ✅ order-worker 검증 |
| AWS 비용 초과 | 중 | t3 instance, **destroy-temp.sh로 임시 자원 일괄 destroy**, S3 Lifecycle | ✅ 현재 ~$1/월 (영구 자원만) |
| 분산 락 데드락 | 낮음 | Redisson lease time + watchdog, 단위 테스트로 검증 | ⏸ Phase 5 |
| **netty 4.2.x BOM CVE-2026-42577** (api-gateway) | 중 | reactor-netty 1.3.x 또는 SB 3.5.15+ 출시 대기 (R-28). 현재는 `.trivyignore` accepted risk | 📌 외부 의존성 모니터링 |
| **Spring Boot 3.5 OSS EOL 2026-06-30** | 중 | Phase 6 후 3.6/4.0 마이그레이션 검토 (R-18) | 📌 monitoring |
| **JWT secret 평문 노출** (auth-service) | 중 | placeholder fallback로 1차 hygiene → Phase 5에서 ExternalSecrets Operator + AWS Secrets Manager (R-25/R-33) | 🔄 1차 fix 완료 |

---

## 11. 기대 효과

- **실무형 클라우드 네이티브 역량 확보** — Kubernetes (self-managed kubeadm), Service Mesh, OpenTelemetry, IaC를 한 사이클로 경험.
- **GitOps 운영 체계 정착** — 클러스터·앱 상태가 단일 Git 리포지토리로 수렴. ApplicationSet으로 12 Application 자동 생성, App-of-Apps + Sync Wave로 플랫폼 컴포넌트 의존성 순서 명시.
- **4단계 DR 자동화 체계** — *S3 backend 수동 → Terraform → Ansible (kubeadm + ArgoCD 설치) → root sync 자동* 네 단계로 빈 AWS 계정에서 전체 운영 환경을 재구축 가능.
- **Polyrepo + 공용 라이브러리 패턴 정착** — 9개 polyrepo + GH Packages publish, ArgoCD ApplicationSet으로 동질 앱 일괄 관리.
- **분산 시스템 트레이드오프 체득** — 일관성 vs 가용성, Saga(orchestration) vs 2PC, 캐시 가환성, Outbox vs CDC, Event Sourcing 등을 코드 레벨로 이해.
- **운영 가시성 확보** (Phase 5) — 메트릭·로그·트레이스를 단일 Grafana 뷰에서 상관 분석.
- **재사용 가능한 자산** — Terraform 모듈, Ansible 플레이북/롤, Helm 공통 차트, Argo CD 매니페스트 리포 템플릿, GitHub Actions 워크플로 (OIDC + Trivy + auto-bump 표준).

---

## 부록 A. 토픽·이벤트 카탈로그 (실제 적용본 — ADR-0004)

| 토픽 | 발행자 | 구독자 | 페이로드 핵심 필드 |
| --- | --- | --- | --- |
| `order.pending` | order-worker (Outbox poller) | inventory-service | orderId, items[], idempotencyKey |
| `order.inventory-reserved` | inventory-service | order-service | orderId, reservationId, success |
| `order.confirmed` | order-service (state CONFIRMED 도달 시) | (현재 consumer 없음 — 향후 분석 등) | orderId, userId |
| `order.cancelled` | order-service (state CANCELLED 도달 시) | (현재 consumer 없음) | orderId, reason |

> 원본 계획서의 `notification.requested` 토픽은 폐기. 명명 규약: 소문자 + `<도메인>.<상태/동작>` + 다중 단어는 hyphen (`order.inventory-reserved` 등). ADR-0004.

## 부록 B. SLO 초안

| 서비스 | 가용성 | Latency P99 | Error Budget(월) |
| --- | --- | --- | --- |
| user-service | 99.9% | 200ms | 43.2분 |
| auth-service | 99.95% | 100ms | 21.6분 |
| product-service | 99.9% | 150ms | 43.2분 |
| inventory-service | 99.95% | 50ms | 21.6분 |
| order-service | 99.9% | 300ms | 43.2분 |
| api-gateway | 99.95% | (백엔드 + 20ms JWT 검증) | 21.6분 |

## 부록 C. 계획서 대비 변경 요약

| 영역 | 원본 계획서 | 실제 구현 | 사유 / 근거 |
| --- | --- | --- | --- |
| **서비스 개수** | 5개 (user/product/inventory/order/notification) | **6개** (user/auth/product/order/inventory/api-gateway) | auth 분리 + notification 폐기 + api-gateway 별 서비스화 (ADR-0001) |
| **notification-service** | 핵심 도메인 중 1개 | **폐기 (archive)** | 학습 범위 축소. 시스템 알림은 Grafana → Slack/PagerDuty (ADR-0001) |
| **auth 도메인** | user-service 내부 | **`msa-auth-service` 별도 레포 분리** | JWT secret 보안 영역 격리 + gRPC 단일 책임 (ADR-0001) |
| **api-gateway** | Spring Cloud Gateway 단독 | **BFF (gRPC client) + SC Gateway 혼합** | path별로 BFF 또는 패스스루 선택 (ADR-0005) |
| **레포 구조** | 명시 안 됨 | **Polyrepo 9개** (서비스 6 + common-libs + manifest + provisioning) | Polyrepo + 공용 라이브러리 패턴 (ADR-0001) |
| **공용 라이브러리** | 명시 안 됨 | `msa-common-libs` 멀티모듈 (`common` + `events` Protobuf) — GH Packages publish | dual delivery (lib + service) |
| **Redis 클라이언트** | 명시 안 됨 | **JitPack 외부 의존성** (`com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2`) | 팀장님 publish 사용. `msa-common-libs:client-redis`는 제거 (ADR-0002) |
| **Kafka 직렬화** | 명시 안 됨 | **JSON wire** 유지 (Protobuf 코드젠 있지만 wire 미사용) | 모노레포 호환 + 학습 부담 회피. wire→Protobuf 마이그레이션은 추후 트랙 (ADR-0003) |
| **Kafka 토픽명** | `order.pending`, `order.inventory_reserved`(underscore), `order.confirmed`, `order.cancelled`, `notification.requested` | `order.pending`, `order.**inventory-reserved**`(hyphen), `order.confirmed`, `order.cancelled` — **`notification.requested` 폐기** | 다중 단어 표기를 hyphen으로 통일 (ADR-0004) |
| **Kafka 모드** | KRaft StatefulSet 3 broker | 현재 single broker, **Phase 5에서 Strimzi Operator + 3 broker** 전환 예정 | 단계적 배포 |
| **Service Mesh** | 처음부터 Istio | **현재 Traefik (msa-provisioning 임시), Phase 5에서 Istio 전환** | 학습 곡선 분산 (R-03) |
| **외부 진입** | NLB + ALB + Spring Cloud Gateway | **NLB (kube-apiserver 6443) 만 노출**. 애플리케이션 진입은 Phase 5 Istio Gateway로 | ALB 미사용 |
| **컨테이너 레지스트리** | ECR or GHCR | **ECR 단일** + GitHub OIDC AssumeRole (long-lived key 미사용) | 보안 강화 |
| **AWS 인프라 추가** | 명시 안 됨 | **VPC Endpoint × 3** (ecr.api / ecr.dkr / s3) + **NAT × 2 (multi-AZ)** + **IAM OIDC** + **KMS** + **ECR × 6** | ECR pull 사설 경로 + 비용 절감 |
| **K8s 노드 구성** | Master 3 + Worker N | **Master 3 + Worker 3 + Bastion 2** | bastion은 SSH ProxyJump 진입점 |
| **payment / shipping** | 별도 마이크로서비스 (암시) | **order-service의 state machine 7-state 확장으로 시뮬레이션** | 학습 범위 + ADR-0006 |
| **order state machine** | 단순 saga | **7-state**: CREATED → PENDING → PAID → SHIPPING → SHIPPED → CONFIRMED or CANCELLED | ADR-0006 |
| **inventory worker** | 배치 잡 (암시) | **Spring Batch + `@Profile("worker")` + 별도 Deployment** | ADR-0007 |
| **Spring Boot 버전** | 3.5.x | **3.5.13** (CVE-2025-22235 fix) | 정확한 버전 명시 |
| **Kotlin/Gradle 버전** | 명시 안 됨 | **Kotlin 2.1.0 + Gradle 8.10.2** | Kotlin 2.3.x JetBrains 버그 보류 (R-09/R-16) |
| **Spring Cloud 버전** | 명시 안 됨 | **2025.0.2 Northfields** (api-gateway 전용) | SB 3.5.x 공식 짝 |
| **JWT 검증 위치** | api-gateway 자체 검증 (암시) | **api-gateway → auth-service `CheckValidity` gRPC 위임** | 중앙화 + 보안 영역 격리 (ADR-0005) |
| **CI 최적화** | 명시 안 됨 | **Gradle 중복 빌드 제거** (호스트 + Docker → Docker만 packaging), 빌드 시간 4분 → **2분 25초** | R-27 (a) |
| **kubelet ECR pull** | 명시 안 됨 | **ecr-credential-provider Go 빌드 + kubeadm-flags.env 직접 통합** | R-29/R-30 (drop-in 미평가 우회) |
| **다이어그램** | draw.io, Mermaid | **모두 Mermaid + `docs/images/` 단일 위치** | AWS-architecture / GitOps-flow / MSA-services 3종 |
| **서비스 간 직접 호출** | 동기 gRPC (선택) | **0건** — 모든 서비스가 gRPC server만, client 설정 없음 | 진정한 decoupling. inter-service는 Kafka만 |

### 누적 결정 (ADR)

| ADR | 결정 |
| --- | --- |
| [ADR-0001](./adr/0001-polyrepo-with-auth-service.md) | Polyrepo 9개 + auth-service 분리 + notification 폐기 |
| [ADR-0002](./adr/0002-client-libraries-distribution.md) | client-redis JitPack, client-ses 제거 |
| [ADR-0003](./adr/0003-kafka-wire-format-json.md) | Kafka 직렬화 = JSON 유지 |
| [ADR-0004](./adr/0004-kafka-topic-naming.md) | 토픽명 = `<도메인>.<상태/동작>` (hyphen for multi-word) |
| [ADR-0005](./adr/0005-api-gateway-bff-with-cloud-gateway.md) | api-gateway = BFF + SC Gateway 혼합 |
| [ADR-0006](./adr/0006-order-state-machine-extension.md) | order 7-state machine |
| [ADR-0007](./adr/0007-inventory-event-sourcing-batch-worker.md) | inventory Event Sourcing + Spring Batch worker |
| [ADR-0008](./adr/0008-order-terminal-events.md) | order.confirmed/cancelled 토픽 발행 유지 |

### 추적 중인 R 항목 (Phase 5+ 대상)

- **R-03** Traefik → Istio 교체 (Phase 5)
- **R-08** 모노레포 archive (Phase 6)
- **R-18** Spring Boot 3.5 EOL 2026-06-30
- **R-25** JWT secret 정식 외부화 (Phase 5, ExternalSecrets)
- **R-27 (b/c/d/f)** CI 추가 최적화
- **R-28** netty 4.2.x BOM CVE-2026-42577 (외부 의존성)
- **R-33** 6×2 Secret/ConfigMap 실값 (Phase 5)
- **R-35** Phase 5 platform 본 작업

---

## 변경 이력

| 일자 | 변경 | 비고 |
| --- | --- | --- |
| 2026-04-24 | 원본 계획서 작성 | KT Cloud Tech UP 2기 제출본 |
| 2026-05-13 | **실제 구현 기준 갱신본** | Phase 0-4 완료 + ADR 0001-0008 + 다이어그램 3종 반영 (본 문서) |

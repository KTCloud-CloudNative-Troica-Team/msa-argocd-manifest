# Troica GitOps / ArgoCD 배포 흐름

6개 polyrepo + msa-argocd-manifest (단일 진실의 원천) + ArgoCD app-of-apps 가 어떻게 맞물려 클러스터까지 배포되는지 한 장 요약.

> 본 레포 README: [../../README.md](../../README.md)
> ArgoCD 설치: `msa-provisioning/ansible-playbook main.yaml` (Phase 0 ~ 1 자동화)

---

## 전체 흐름 (Developer → 클러스터)

```mermaid
flowchart TB
  Dev([👨‍💻 Developer])

  subgraph Polyrepos["🐙 6개 Polyrepo (서비스별)"]
    direction TB
    R1["msa-user-service"]
    R2["msa-auth-service"]
    R3["msa-product-service"]
    R4["msa-order-service"]
    R5["msa-inventory-service"]
    R6["msa-api-gateway"]
  end

  subgraph CI["⚙️ GitHub Actions (각 repo)"]
    direction TB
    Build["build + test"]
    Trivy["Trivy CVE scan"]
    Push["docker push → ECR"]
    Bump["manifest auto-bump<br/>dev: direct commit<br/>prod: PR (승인 게이트)"]
    Build --> Trivy --> Push --> Bump
  end

  ECR[("📦 ECR × 6<br/>msa/&lt;service&gt;:main-&lt;sha&gt;")]

  subgraph Manifest["📘 msa-argocd-manifest (GitOps SoT)"]
    direction TB
    BootDir["bootstrap/<br/>root-app.yaml + apps.yaml"]
    ProjDir["projects/<br/>AppProject × 2"]
    PlatDir["platform/<br/>root.yaml (App-of-Apps)"]
    AppDir["applications/<br/>appset.yaml + charts/ + values/"]
  end

  subgraph Cluster["☸️ Kubernetes Cluster (master × 3 + worker × 3)"]
    direction TB
    subgraph ArgoNS["📂 namespace argocd"]
      direction TB
      RootApp["Application root-app"]
      Child1["Application app-projects"]
      Child2["Application app-platform"]
      Child3["Application app-applications"]
      AppSet["ApplicationSet market-services<br/>6 svc × 2 env = 12 Application"]
      RootApp --> Child1
      RootApp --> Child2
      RootApp --> Child3
      Child3 --> AppSet
    end
    subgraph DevNS["📂 namespace market-dev"]
      DevPods["Deployments × 6<br/>Services × 6<br/>ConfigMap / Secret"]
    end
    subgraph ProdNS["📂 namespace market-prod"]
      ProdPods["Deployments × 6<br/>Services × 6<br/>ConfigMap / Secret"]
    end
  end

  Dev -->|git push main| Polyrepos
  Polyrepos -->|trigger| CI
  Push -.->|"image push"| ECR
  Bump -->|"commit / PR"| Manifest

  Manifest -.->|"poll / webhook"| ArgoNS
  AppSet -->|"sync apply"| DevNS
  AppSet -->|"sync apply"| ProdNS

  DevPods -.->|"kubelet pull"| ECR
  ProdPods -.->|"kubelet pull"| ECR

  classDef dev fill:#e0e0e0,stroke:#212121,color:#000
  classDef repo fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000
  classDef ci fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
  classDef ecr fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
  classDef manifest fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
  classDef cluster fill:#f1f8e9,stroke:#558b2f,stroke-width:2px,color:#000
  classDef argons fill:#b2dfdb,stroke:#00695c,color:#000
  classDef ns fill:#e1bee7,stroke:#7b1fa2,color:#000
  classDef app fill:#bbdefb,stroke:#1976d2,color:#000
  classDef workload fill:#c8e6c9,stroke:#2e7d32,color:#000

  class Dev dev
  class Polyrepos repo
  class R1,R2,R3,R4,R5,R6 repo
  class CI ci
  class Build,Trivy,Push,Bump ci
  class ECR ecr
  class Manifest manifest
  class BootDir,ProjDir,PlatDir,AppDir manifest
  class Cluster cluster
  class ArgoNS argons
  class RootApp,Child1,Child2,Child3,AppSet app
  class DevNS,ProdNS ns
  class DevPods,ProdPods workload
```

---

## 단계별 설명

### 1. 개발자 commit
- 6개 polyrepo 중 하나의 `main` 브랜치에 push (직접 또는 PR merge)

### 2. CI (GitHub Actions) — 각 repo의 `.github/workflows/ci.yml`
- `build + test` (R-27 (a) 적용 후 ~2-3분)
- `Trivy CVE scan` (HIGH/CRITICAL gate)
- `docker push → ECR` (OIDC AssumeRole, `msa/<service>:main-<short-sha>` 태그)
- `manifest auto-bump`:
  - **dev**: msa-argocd-manifest의 `values-dev.yaml`을 **직접 commit** → 자동 배포
  - **prod**: `values-prod.yaml` 변경 **PR 생성** → 승인 게이트

### 3. msa-argocd-manifest (GitOps 단일 진실의 원천)
- `bootstrap/`: ArgoCD 진입점 (root-app + 3 child Application)
- `projects/`: AppProject CRD × 2 (market / platform)
- `platform/`: app-of-apps 패턴 (Phase 5에서 Postgres/Redis/Kafka 등 추가)
- `applications/`: 6 service × 2 env Helm values + 공통 차트

### 4. ArgoCD (cluster, namespace `argocd`)
- `root-app` Application이 본 레포 watch → 3 자식 Application 생성
- `app-applications`가 ApplicationSet `market-services`를 sync
- ApplicationSet이 **6 service × 2 env = 12 Application** 자동 생성

### 5. K8s workload sync
- 각 child Application이 `market-dev` 또는 `market-prod` namespace에 Deployment/Service/ConfigMap/Secret apply
- kubelet은 ECR에서 image pull (별도 경로 — instance profile 사용, [AWS-architecture §5 참조](./AWS-architecture.md))

---

## 검증 포인트

- ✅ CI는 image push **+** manifest commit/PR 두 작업 분리 (BACKLOG R-27 (a) 빌드 시간 최적화 적용)
- ✅ dev = direct commit (속도) / prod = PR (안전 게이트) — 환경별 정책 차등
- ✅ ApplicationSet으로 12개 Application 선언적 생성 (각 서비스 추가 시 `values/<service>/` 만 만들면 자동)
- ✅ app-of-apps로 layered sync: projects → platform → applications
- ✅ kubelet ECR pull은 ArgoCD와 별개 경로 (image-credential-provider, R-29/R-30)
- ✅ ArgoCD 자체는 ansible로 부트스트랩 (Phase 0~1) → 본 레포는 ArgoCD 설치를 가정

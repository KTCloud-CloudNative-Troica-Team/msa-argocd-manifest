# Troica GitOps / ArgoCD 배포 흐름

6개 polyrepo + msa-argocd-manifest (단일 진실의 원천) + ArgoCD app-of-apps 가 어떻게 맞물려 클러스터까지 배포되는지 한 장 요약.

> 본 레포 README: [../../README.md](../../README.md)
> ArgoCD 설치: `msa-provisioning/ansible-playbook main.yaml` (Phase 0 ~ 1 자동화)

---

## 전체 흐름 (Developer → 클러스터)

![gitops.png](gitops.png)

```mermaid
flowchart TB
  Dev([👨‍💻 Developer])

  subgraph Polyrepos["🐙 9개 Polyrepo (서비스별 + 공통 + 매니페스트 + 프론트)"]
    direction TB
    R1["msa-user-service"]
    R2["msa-auth-service"]
    R3["msa-product-service"]
    R4["msa-order-service"]
    R5["msa-inventory-service"]
    R6["msa-api-gateway"]
    R7["msa-common-libs"]
    R8["msa-argocd-manifest"]
    R9["msa-frontend"]
    R0["msa-provisioning"]
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
    ProjDir["projects/<br/>AppProject × 2 (market + platform)"]
    PlatDir["platform/ — 19 Application<br/>cert-manager / ebs-csi / rbac<br/>istio-{base,cp,gateway,ingressgateway}<br/>kube-prometheus-stack / loki / tempo<br/>troica-prometheus-rules<br/>cnpg / strimzi / external-secrets-op<br/>kafka / postgres / redis<br/>network-policies / external-secrets-config"]
    AppDir["applications/<br/>appset.yaml + charts/microservice<br/>+ values/{6 service}/{values, values-dev}.yaml"]
    Tests["tests/e2e/<br/>troica-market-e2e.postman_collection.json"]
    Wf[".github/workflows/<br/>e2e-newman.yml + trivy-manifest-scan.yml"]
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
      DevPods["Deployments × 8 (api + 5 service + 2 worker)<br/>Services × 6<br/>2/2 sidecar inject (Istio)<br/>ConfigMap (envFrom) + Secret (ExternalSecret 결과)"]
    end
    subgraph ProdNS["📂 namespace market-prod<br/>(PoC diet — 비활성)"]
      ProdPods["AppSet 의 list generator 측<br/>dev only — 평가 demo 는 dev"]
    end
    subgraph IstioNS["📂 namespace istio-system"]
      IstioPods["istiod (Wave -2)<br/>istio-ingressgateway (Wave -1, R-62 PR #118 분리)<br/>Gateway + VirtualService"]
    end
    subgraph DataNS["📂 stateful namespaces"]
      DataPods["databases / kafka / redis<br/>monitoring / observability<br/>cnpg-system / external-secrets"]
    end
  end

  SM[("🔐 AWS Secrets Manager<br/>troica/* secrets<br/>(DB password, JWT secret, Slack webhook)")]
  ESO["External Secrets Operator<br/>(platform/40-external-secrets-operator)"]

  Dev -->|git push main| Polyrepos
  Polyrepos -->|trigger| CI
  Push -.->|"image push"| ECR
  Bump -->|"yq write + commit / PR<br/>values-dev.yaml: image.tag"| Manifest

  Manifest -.->|"poll / webhook<br/>3 min default"| ArgoNS
  AppSet -->|"sync apply<br/>Wave 10"| DevNS
  Child2 -->|"sync<br/>Wave -5 ~ 5"| IstioNS
  Child2 -->|"Wave 2 ~ 3"| DataNS

  DevPods -.->|"kubelet ECR pull<br/>(instance profile, R-29)"| ECR
  SM -.->|"IMDS auth<br/>(node IAM role)"| ESO
  ESO -.->|"sync K8s Secret<br/>SecretSynced=True"| DataNS
  ESO -.->|"envFrom secretRef"| DevNS

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
  class R0,R1,R2,R3,R4,R5,R6,R7,R8,R9 repo
  class CI ci
  class Build,Trivy,Push,Bump ci
  class ECR ecr
  class Manifest manifest
  class BootDir,ProjDir,PlatDir,AppDir,Tests,Wf manifest
  class Cluster cluster
  class ArgoNS argons
  class RootApp,Child1,Child2,Child3,AppSet app
  class DevNS,ProdNS,IstioNS,DataNS ns
  class DevPods,ProdPods,IstioPods,DataPods workload
  class SM,ESO ecr
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
- `platform/`: **19 Application** (Sync Wave -8 ~ 5):
  - **인프라 (3)**: 00-cert-manager / 05-ebs-csi / 05-rbac (Wave -8, R-48)
  - **Istio mesh (4)**: 10-istio-base (Wave -5, CRD) / 11-istio-cp (Wave -2, istiod only) / 12-istio-gateway (Wave -3, Gateway+VS+PeerAuth+DestinationRule+namespace label) / **13-istio-ingressgateway** (Wave -1, R-62 PR #118 분리)
  - **Observability (4)**: 30-kube-prometheus-stack / 30-loki / 30-tempo / 31-prometheus-rules (troica-prometheus-rules, R-47 AlertManager 룰)
  - **Operators (3)**: 40-cnpg-operator / 40-external-secrets-operator / 40-strimzi-operator
  - **Stateful (3)**: 50-kafka-cluster / 50-postgres-clusters / 50-redis-cluster
  - **Security (1)**: 60-network-policies (R-49 default-deny)
  - **Config (1)**: 91-external-secrets-config (R-25/R-33 service envFrom binding)
- `applications/`: 6 service Helm values (`values/<service>/values.yaml + values-dev.yaml`) + 공통 차트 (`charts/microservice/`)
- `tests/e2e/`: Newman collection + Postman 시나리오 7 step
- `.github/workflows/`: e2e-newman.yml (R-42) + trivy-manifest-scan.yml (R-46)

### 4. ArgoCD (cluster, namespace `argocd`)
- `root-app` Application이 본 레포 watch → 3 자식 Application 생성 (`app-projects` / `app-platform` / `app-applications`)
- `app-platform` 측 platform/ 디렉토리의 18 Application sync
- `app-applications` 측 ApplicationSet `market-services` sync
- ApplicationSet이 **6 service × {env: dev} = 6 Application** 자동 생성 (PoC diet — prod 일시 비활성, AppSet list generator 측)

### 5. ExternalSecrets 측 secret 동기화
- ESO operator (platform/40-external-secrets-operator) 가 ClusterSecretStore + 24 ExternalSecret 통해 AWS Secrets Manager 측 secret 을 K8s Secret 으로 sync
- IRSA 가 아닌 **IMDS 인증** (self-managed kubeadm 측, R-29 동일 패턴)
- 12 K8s Secret + 12 ConfigMap (envFrom secretRef + configMapRef) 자동 mount

### 6. K8s workload sync
- 각 Application이 `market-dev` namespace에 Deployment / Service / ConfigMap / Secret apply (Wave 10)
- platform 측 Application 들은 Wave -8 ~ 5 순서로 sync (stateful → service 의존성 보장)
- kubelet은 ECR에서 image pull (별도 경로 — instance profile 사용, [AWS-architecture §5 참조](./AWS-architecture.md))
- Istio sidecar 자동 inject (`istio-injection=enabled` namespace label) → pod 2/2 Running

---

## 검증 포인트

- ✅ CI는 image push **+** manifest commit/PR 두 작업 분리 (R-27 (a) 빌드 시간 최적화 적용 ~2분 25초)
- ✅ dev = direct commit (속도) / prod = PR (안전 게이트) — 환경별 정책 차등
- ✅ ApplicationSet으로 **6개 Application** 선언적 생성 (PoC diet — dev only. `values/<service>/` 만 만들면 자동)
- ✅ app-of-apps로 layered sync: projects → platform → applications (Wave 기반 순서)
- ✅ **istio-cp (Wave -2) 와 istio-ingressgateway (Wave -1) 분리** (R-62 PR #118) — multi-source race 영구 제거
- ✅ kubelet ECR pull은 ArgoCD와 별개 경로 (image-credential-provider, R-29/R-30)
- ✅ ArgoCD 자체는 ansible로 부트스트랩 (Phase 0~1) → 본 레포는 ArgoCD 설치를 가정
- ✅ AWS Secrets Manager → ExternalSecrets Operator (IMDS 인증, self-managed kubeadm) → K8s Secret → envFrom secretRef
- ✅ Newman E2E workflow (`.github/workflows/e2e-newman.yml`) workflow_dispatch trigger 시 baseUrl input + NLB DNS 측 cluster 호출 chain 검증

## 평가 18/18 통과 후 cluster 검증 흐름

발표 시연 path:
1. **Cluster up** → ansible (`msa-provisioning/ansible-playbook main.yaml`) → kubeadm + Calico + ArgoCD bootstrap
2. **ArgoCD root-app** → main 측 manifest 18 platform + 6 service Application 자동 sync
3. **ExternalSecrets** → Secrets Manager → 24 K8s Secret SecretSynced
4. **Istio mesh ready** → 8 service pod 2/2 (sidecar inject)
5. **외부 진입** → NLB DNS:80 → Istio Gateway → VirtualService 4 prefix → api-gateway
6. **검증**: Newman workflow 5/5 PASS + Grafana built-in dashboard + Prometheus targets + R-41 CB OPEN 시연

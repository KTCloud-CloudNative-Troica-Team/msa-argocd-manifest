# Troica AWS Architecture (상세)

`msa-provisioning` Terraform이 만드는 모든 AWS 리소스를 레벨별로 분할 시각화.

> 검증 기준: 각 리소스의 정확한 scope (계정/리전/VPC/AZ/Subnet) + 트래픽 방향.
> 자세한 코드: [msa-provisioning/terraform](https://github.com/KTCloud-CloudNative-Troica-Team/msa-provisioning/tree/main/terraform)
>
> **누적 변경**:
> - R-59 (msa-provisioning PR #13): worker 사양 t3.medium → t3.large (Phase 5 메모리 대비)
> - R-35 (d) / PR #22: NLB 에 Istio Gateway path 추가 — listener 80/443, target group istio-http-tg (30080) + istio-https-tg (30443), worker 3 attachment × 2 port, cluster-node-sg ingress 30080/30443 from `0.0.0.0/0`

---

## 1. Network 레이어 (VPC + AZ + Subnet + IGW + NAT + NLB)

![aws_arch_1_network.png](aws_arch_1_network.png)

```mermaid
flowchart TB
  Internet([🌐 Internet])
  User([👤 운영자<br/>ssh / kubectl])

  subgraph VPC["☁️ VPC kt-cloud-vpc — 10.0.0.0/16"]
    IGW["Internet Gateway"]
    NLB["NLB kt-cloud-nlb<br/>L4 TCP passthrough<br/>listener 6443/80/443<br/>R-35 (d) Istio Gateway + k8s API"]

    subgraph AZ2A["📍 ap-northeast-2a"]
      direction TB
      subgraph PubA["Public Subnet 10.0.1.0/24"]
        NATA["NAT Gateway 2a<br/>+ EIP"]
        BastA["bastion-a<br/>t3.nano<br/>auto public IP"]
      end
      subgraph PriA["Private Subnet 10.0.2.0/24"]
        MA1["master-1<br/>t3.medium"]
        MA2["master-2<br/>t3.medium"]
        WA["worker-1<br/>t3.large"]
      end
    end

    subgraph AZ2B["📍 ap-northeast-2b"]
      direction TB
      subgraph PubB["Public Subnet 10.0.3.0/24"]
        NATB["NAT Gateway 2b<br/>+ EIP"]
        BastB["bastion-b<br/>t3.nano<br/>auto public IP"]
      end
      subgraph PriB["Private Subnet 10.0.4.0/24"]
        MB1["master-3<br/>t3.medium"]
        WB1["worker-2<br/>t3.large"]
        WB2["worker-3<br/>t3.large"]
      end
    end
  end

  User -->|ssh :22| BastA
  User -->|ssh :22| BastB
  User -->|kubectl :6443| NLB
  Internet -->|"inbound 6443<br/>(k8s API)"| NLB
  Internet -->|"inbound 80/443<br/>(Istio Gateway, PR #22)"| NLB

  NLB -->|"6443 → master"| MA1
  NLB -->|"6443 → master"| MA2
  NLB -->|"6443 → master"| MB1
  NLB -->|"80 → NodePort 30080<br/>443 → NodePort 30443<br/>(Istio ingressgateway)"| WA
  NLB -->|"80 → NodePort 30080<br/>443 → NodePort 30443"| WB1
  NLB -->|"80 → NodePort 30080<br/>443 → NodePort 30443"| WB2

  BastA -.ssh jump.-> MA1
  BastB -.ssh jump.-> MB1

  PriA -.egress via NAT.-> NATA
  PriB -.egress via NAT.-> NATB
  NATA --> IGW
  NATB --> IGW
  BastA --> IGW
  BastB --> IGW
  IGW --> Internet

  classDef vpc fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
  classDef az2a fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
  classDef az2b fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000
  classDef pub fill:#bbdefb,stroke:#1976d2,color:#000
  classDef pri fill:#e1bee7,stroke:#7b1fa2,color:#000
  classDef master fill:#ffcdd2,stroke:#c62828,color:#000
  classDef worker fill:#c8e6c9,stroke:#2e7d32,color:#000
  classDef bastion fill:#ffe0b2,stroke:#ef6c00,color:#000
  classDef nat fill:#ffccbc,stroke:#d84315,color:#000
  classDef nlb fill:#b2dfdb,stroke:#00695c,stroke-width:2px,color:#000
  classDef igw fill:#d1c4e9,stroke:#4527a0,color:#000
  classDef ext fill:#f5f5f5,stroke:#424242,color:#000

  class VPC vpc
  class AZ2A az2a
  class AZ2B az2b
  class PubA,PubB pub
  class PriA,PriB pri
  class MA1,MA2,MB1 master
  class WA,WB1,WB2 worker
  class BastA,BastB bastion
  class NATA,NATB nat
  class NLB nlb
  class IGW igw
  class Internet,User ext
```

**검증 포인트**:
- VPC `10.0.0.0/16`, subnet 4개 (public/private × 2 AZ)
- NLB는 양 AZ public subnet에 attach (subnet_mapping × 2 + EIP × 2)
- **NLB listener 3개**: 6443 (k8s API → master target group) + 80 (HTTP) + 443 (HTTPS) → worker NodePort 30080/30443 → istio-ingressgateway (PR #22, R-35 (d) Istio Gateway 외부 진입 path)
- private subnet의 egress는 각 AZ의 NAT 경유 → IGW (route 분리)
- bastion만 public subnet (외부 SSH 진입). 별도 bastion subnet 없음. `associate_public_ip_address=true` 의 auto public IP (EIP attach 아님)
- master 3대 (2a:2 + 2b:1, t3.medium), worker 3대 (2a:1 + 2b:2, **t3.large** - R-59 메모리 상향)

---

## 2. Compute + Storage (EC2 + EFS + EBS + VPC Endpoint)

![aws_arch_2_compute_storage.png](aws_arch_2_compute_storage.png)

```mermaid
flowchart LR
  subgraph VPC["☁️ VPC kt-cloud-vpc"]
    direction TB

    subgraph EndpointGroup["VPC Endpoints (intra-VPC)"]
      direction LR
      VPCE_API["VPCE Interface<br/>ecr.api<br/>private DNS"]
      VPCE_DKR["VPCE Interface<br/>ecr.dkr<br/>private DNS"]
      VPCE_S3[("VPCE Gateway<br/>s3<br/>route-table 매핑")]
    end

    EFS[("EFS File System<br/>kt-cloud-cluster-efs<br/>리전 스코프")]

    subgraph AZ2A["📍 ap-northeast-2a"]
      subgraph PriA["Private Subnet 10.0.2.0/24"]
        MA1["master-1<br/>t3.medium"]
        MA2["master-2<br/>t3.medium"]
        WA["worker-1<br/>t3.medium"]
        EBSa["EBS 20GB<br/>/dev/sdh"]
        EFSmA["EFS Mount Target<br/>ENI (2a)"]
        ENI_A_API["VPCE ENI<br/>ecr.api"]
        ENI_A_DKR["VPCE ENI<br/>ecr.dkr"]
      end
    end

    subgraph AZ2B["📍 ap-northeast-2b"]
      subgraph PriB["Private Subnet 10.0.4.0/24"]
        MB1["master-3<br/>t3.medium"]
        WB1["worker-2<br/>t3.medium"]
        WB2["worker-3<br/>t3.medium"]
        EBSb1["EBS 20GB<br/>/dev/sdh"]
        EBSb2["EBS 20GB<br/>/dev/sdh"]
        EFSmB["EFS Mount Target<br/>ENI (2b)"]
        ENI_B_API["VPCE ENI<br/>ecr.api"]
        ENI_B_DKR["VPCE ENI<br/>ecr.dkr"]
      end
    end
  end

  WA -.volume_attachment.- EBSa
  WB1 -.volume_attachment.- EBSb1
  WB2 -.volume_attachment.- EBSb2

  EFS -.mount.- EFSmA
  EFS -.mount.- EFSmB
  MA1 & MA2 & WA -->|NFS 2049| EFSmA
  MB1 & WB1 & WB2 -->|NFS 2049| EFSmB

  VPCE_API -.subnet_ids.- ENI_A_API
  VPCE_API -.subnet_ids.- ENI_B_API
  VPCE_DKR -.subnet_ids.- ENI_A_DKR
  VPCE_DKR -.subnet_ids.- ENI_B_DKR

  classDef vpc fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
  classDef az2a fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
  classDef az2b fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000
  classDef pri fill:#e1bee7,stroke:#7b1fa2,color:#000
  classDef master fill:#ffcdd2,stroke:#c62828,color:#000
  classDef worker fill:#c8e6c9,stroke:#2e7d32,color:#000
  classDef storage fill:#cfd8dc,stroke:#37474f,color:#000
  classDef vpce fill:#fff3e0,stroke:#e65100,color:#000
  classDef eni fill:#ffe0b2,stroke:#ef6c00,color:#000

  class VPC vpc
  class AZ2A az2a
  class AZ2B az2b
  class PriA,PriB pri
  class MA1,MA2,MB1 master
  class WA,WB1,WB2 worker
  class EBSa,EBSb1,EBSb2,EFS,EFSmA,EFSmB storage
  class VPCE_API,VPCE_DKR,VPCE_S3,EndpointGroup vpce
  class ENI_A_API,ENI_A_DKR,ENI_B_API,ENI_B_DKR eni
```

**검증 포인트**:
- EBS는 worker × 3에만 부착 (master/bastion은 root volume만)
- EFS는 리전 스코프, 각 AZ private subnet에 mount target ENI 1개씩
- VPC Endpoint Interface (ecr.api / ecr.dkr): 양 AZ private subnet에 ENI (subnet_ids로 attach)
- VPC Endpoint Gateway (s3): route table 매핑 (private RT × 2) — 본 그림에서는 ENI 없는 별개 표시

---

## 3. Security Groups + 트래픽 흐름

![aws_arch_3_sg_traffic.png](aws_arch_3_sg_traffic.png)

```mermaid
flowchart TB
  User([👤 User])
  Internet([🌐 Internet])

  subgraph VPC["☁️ VPC kt-cloud-vpc"]
    direction TB

    subgraph SGBastion["🛡 SG bastion-node-sg"]
      Bastion["bastion × 2<br/>public subnets"]
    end

    subgraph SGCluster["🛡 SG cluster-node-sg<br/>(self-ref ALL, intra-VPC)"]
      ClusterNodes["master × 3<br/>worker × 3<br/>private subnets"]
    end

    subgraph SGEFS["🛡 SG efs-sg<br/>(NFS from VPC CIDR)"]
      EFSMounts["EFS Mount Target × 2"]
    end

    subgraph SGVPCE["🛡 SG vpc-endpoint-sg<br/>(HTTPS from VPC CIDR)"]
      VPCEs["VPCE ENI<br/>ecr.api + ecr.dkr"]
    end

    NLB["NLB kt-cloud-nlb"]
  end

  User -->|"22 SSH<br/>+ ICMP<br/>(my_ip /32)"| Bastion
  Internet -->|"6443 from 0.0.0.0/0<br/>30080 from 0.0.0.0/0<br/>30443 from 0.0.0.0/0<br/>(PR #22)"| NLB
  NLB -->|"6443 → master<br/>30080/30443 → worker"| ClusterNodes
  Bastion -->|"22 from bastion-sg"| ClusterNodes
  ClusterNodes -->|"self-ref ALL"| ClusterNodes
  ClusterNodes -->|"2049 NFS"| EFSMounts
  ClusterNodes -->|"443 HTTPS"| VPCEs

  classDef vpc fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
  classDef sgBastion fill:#ffe0b2,stroke:#ef6c00,stroke-width:2px,color:#000
  classDef sgCluster fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
  classDef sgEFS fill:#cfd8dc,stroke:#37474f,stroke-width:2px,color:#000
  classDef sgVPCE fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
  classDef nlb fill:#b2dfdb,stroke:#00695c,stroke-width:2px,color:#000
  classDef ext fill:#f5f5f5,stroke:#424242,color:#000

  class VPC vpc
  class SGBastion sgBastion
  class SGCluster sgCluster
  class SGEFS sgEFS
  class SGVPCE sgVPCE
  class NLB nlb
  class User,Internet ext
```

**검증 포인트**:
- `bastion-node-sg`: SSH 22 + ICMP only from `data.http.my_ip` (운영자 PC IP `/32`)
- `cluster-node-sg` ingress:
  - 6443 from `0.0.0.0/0` (NLB → master k8s API)
  - **30080 from `0.0.0.0/0`** (PR #22, NLB → worker NodePort http) — `aws_security_group_rule.cluster_node_istio_http_ingress`
  - **30443 from `0.0.0.0/0`** (PR #22, NLB → worker NodePort https) — `aws_security_group_rule.cluster_node_istio_https_ingress`
  - intra-VPC TCP/UDP/ICMP from `10.0.0.0/16`
  - SSH 22 from `bastion-node-sg`
- `cluster-node-sg` self-ref (`aws_security_group_rule.cluster_node_self_ingress`): 같은 SG 멤버끼리 ALL protocol → Calico IP-in-IP / worker ↔ master 통신
- **NLB source IP preservation** = true (default for instance target) → client public IP 가 worker 까지 그대로 → SG 의 30080/30443 `0.0.0.0/0` ingress 필수 (없으면 NLB listener 추가해도 SG drop)
- `efs-sg`: NFS 2049 from `10.0.0.0/16` (intra-VPC)
- `vpc-endpoint-sg`: HTTPS 443 from `10.0.0.0/16` (intra-VPC)

---

## 4. CI/CD — GitHub Actions OIDC → ECR push

![aws_arch_4_cicd.png](aws_arch_4_cicd.png)

```mermaid
flowchart LR
  subgraph GH["🐙 GitHub Organization<br/>KTCloud-CloudNative-Troica-Team"]
    direction TB
    Repo["msa-* repository<br/>main branch push"]
    Workflow["CI workflow<br/>aws-actions/configure-aws-credentials@v4"]
    Repo --> Workflow
  end

  subgraph AWS["☁️ AWS Account 601766312629 (account-level)"]
    direction TB

    subgraph IAM["🔐 IAM"]
      direction TB
      OIDC["OIDC Provider<br/>token.actions.githubusercontent.com"]
      Role["IAM Role<br/>troica-gha-ecr-push<br/>Trust: repo:org/msa-*:ref:refs/heads/main"]
      Policy["Inline Policy<br/>ECR push actions"]
      OIDC -->|Federated| Role
      Role --> Policy
    end

    subgraph ECRGroup["📦 ECR Repositories"]
      direction TB
      ECR1["msa/user-service"]
      ECR2["msa/auth-service"]
      ECR3["msa/product-service"]
      ECR4["msa/order-service"]
      ECR5["msa/inventory-service"]
      ECR6["msa/api-gateway"]
    end

    KMS["🔑 KMS Key<br/>alias/troica-ecr<br/>rotation enabled"]

    Policy -.scoped to.- ECRGroup
    KMS -.encrypts.- ECRGroup
  end

  Workflow -->|"1 AssumeRoleWithWebIdentity via STS"| OIDC
  Workflow -->|"2 docker push"| ECR3

  classDef gh fill:#e0e0e0,stroke:#212121,stroke-width:2px,color:#000
  classDef aws fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
  classDef iam fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000
  classDef ecr fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
  classDef kms fill:#ffe0b2,stroke:#bf360c,color:#000
  classDef role fill:#ffcdd2,stroke:#c62828,color:#000

  class GH gh
  class AWS aws
  class IAM iam
  class ECRGroup ecr
  class ECR1,ECR2,ECR3,ECR4,ECR5,ECR6 ecr
  class KMS kms
  class OIDC,Role,Policy role
  class Repo,Workflow gh
```

**검증 포인트**:
- IAM OIDC Provider, IAM Role (`troica-gha-ecr-push`), Policy 모두 **계정 레벨** (VPC 외부)
- Trust policy 조건: `repo:KTCloud-CloudNative-Troica-Team/msa-*:ref:refs/heads/main` — main 브랜치만 assume 허용
- IAM Role의 name-based ARN → destroy/apply 사이클 안정 (GitHub Org Secret `AWS_ACCOUNT_ID` 영구 유효)
- ECR Repo × 6, KMS 암호화, IMMUTABLE tag, 30-image lifecycle

---

## 5. EC2 노드의 ECR Image Pull (kubelet 경로 — CI push와 별개)

![aws_arch_5_ecr_image_pull.png](aws_arch_5_ecr_image_pull.png)

```mermaid
flowchart TB
  subgraph Node["🖥 EC2 Node (master/worker, private subnet)"]
    direction TB
    Kubelet["kubelet<br/>--image-credential-provider-config<br/>--image-credential-provider-bin-dir"]
    Provider["ecr-credential-provider binary<br/>/usr/local/bin/<br/>Go-built (R-29)"]
    Profile["EC2 instance_profile<br/>ktcloud-node-profile<br/>+ AmazonEC2ContainerRegistryReadOnly"]
    Kubelet --> Provider
    Provider --> Profile
  end

  subgraph VPCEndpoints["🔌 VPC Endpoints (intra-VPC, private DNS)"]
    direction LR
    VPCE_API["VPCE Interface<br/>ecr.api"]
    VPCE_DKR["VPCE Interface<br/>ecr.dkr"]
    VPCE_S3["VPCE Gateway<br/>s3"]
  end

  subgraph ECR["📦 ECR (account-level)"]
    direction TB
    ECRRepo["ECR Repository<br/>msa/&lt;service&gt;"]
    S3Backing[("S3 image layers<br/>AWS-managed")]
  end

  Provider -->|"1 GetAuthorizationToken"| VPCE_API
  VPCE_API --> ECRRepo
  Kubelet -->|"2 pull manifest"| VPCE_DKR
  VPCE_DKR --> ECRRepo
  Kubelet -->|"3 pull layers"| VPCE_S3
  VPCE_S3 --> S3Backing

  classDef node fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
  classDef vpce fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
  classDef ecr fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
  classDef internal fill:#e1f5fe,stroke:#0277bd,color:#000

  class Node node
  class VPCEndpoints vpce
  class VPCE_API,VPCE_DKR,VPCE_S3 vpce
  class ECR ecr
  class ECRRepo,S3Backing ecr
  class Kubelet,Provider,Profile internal
```

**검증 포인트**:
- CI push와 별개 경로 — kubelet의 ECR pull은 instance profile (`AmazonEC2ContainerRegistryReadOnly`) 사용
- ecr-credential-provider는 GitHub release에 binary 없음 → Go 빌드 후 ansible copy (R-29)
- kubelet args 적용: drop-in 미평가 회피 위해 `kubeadm-flags.env` 직접 통합 (R-30)
- 트래픽: ECR API/DKR은 VPCE Interface, 이미지 layer는 S3 Gateway endpoint

---

## 6. 영구 / 임시 / 수동 자원 분류

```mermaid
flowchart LR
  subgraph Permanent["✅ 영구 (prevent_destroy = true)<br/>월 ~$1"]
    direction TB
    P1["IAM OIDC Provider"]
    P2["IAM Role troica-gha-ecr-push"]
    P3["IAM Role Policy (inline)"]
    P4["KMS Key + Alias"]
    P5["ECR Repo × 6"]
    P6["VPC + Subnet × 4"]
    P7["IGW + Route Tables × 3"]
    P8["Security Groups × 4"]
    P9["Key Pair"]
    P10["IAM Instance Profile"]
  end

  subgraph Temporary["♻️ 임시 (destroy-temp.sh 대상)<br/>월 ~$300"]
    direction TB
    T1["EC2 × 8 (master 3 t3.medium + worker 3 t3.large + bastion 2 t3.nano)"]
    T2["EBS: PVC dynamic provisioning (EBS CSI)"]
    T3["NAT Gateway × 2 + NAT EIP × 2"]
    T4["NLB + Target Group × 3 (k8s-api + istio-http + istio-https) + Listener × 3 (6443/80/443) + Attach × 9 (master 3 + worker 3 × 2 port)"]
    T5["NLB EIP × 2"]
    T6["VPC Endpoint × 3 (ecr.api + ecr.dkr + s3)"]
    T7["EFS File System + Mount Target × 2 + EFS SG"]
    T8["SG rules × 3 (self-ingress + istio-http + istio-https)"]
    T9["local_file ansible_inventory"]
  end

  subgraph Manual["✋ 수동 (Terraform 외부)<br/>월 ~$0.5"]
    direction TB
    M1["S3 Backend Bucket<br/>troica-tfstate-&lt;SUFFIX&gt;"]
  end

  classDef perm fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
  classDef temp fill:#ffe0b2,stroke:#ef6c00,stroke-width:2px,color:#000
  classDef manual fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000

  class Permanent perm
  class Temporary temp
  class Manual manual
  class P1,P2,P3,P4,P5,P6,P7,P8,P9,P10 perm
  class T1,T2,T3,T4,T5,T6,T7,T8,T9 temp
  class M1 manual
```

---

## 7. 통합 토폴로지 (한 화면 요약)

![aws_arch_7_topology.png](aws_arch_7_topology.png)

```mermaid
flowchart TB
  Internet([🌐 Internet])
  GH([🐙 GitHub Actions])
  User([👤 User])

  subgraph AcctLevel["☁️ AWS Account (영구 자원, account-level)"]
    direction LR
    OIDC["IAM OIDC"]
    Role["IAM Role<br/>gha-ecr-push"]
    KMS["🔑 KMS"]
    ECR["📦 ECR × 6"]
    OIDC --> Role
    Role -.scoped.- ECR
    KMS -.encrypts.- ECR
  end

  subgraph Region["📍 Region ap-northeast-2"]
    subgraph VPCBox["☁️ VPC kt-cloud-vpc 10.0.0.0/16"]
      IGW["IGW"]
      NLB["NLB<br/>:6443 (k8s API)<br/>:80/:443 (Istio)"]
      EFS["EFS"]
      VPCE["VPCE × 3<br/>ecr.api/dkr/s3"]

      subgraph A2A["📍 AZ 2a"]
        PubA["public 10.0.1.0/24<br/>NAT-a · bastion-a"]
        PriA["private 10.0.2.0/24<br/>master-1/2 (t3.medium)<br/>worker-1 (t3.large)"]
      end

      subgraph A2B["📍 AZ 2b"]
        PubB["public 10.0.3.0/24<br/>NAT-b · bastion-b"]
        PriB["private 10.0.4.0/24<br/>master-3 (t3.medium)<br/>worker-2/3 (t3.large)"]
      end
    end
  end

  subgraph S3Ext["✋ 수동 (외부)"]
    Bucket["S3 backend bucket"]
  end

  GH -->|"OIDC AssumeRole"| OIDC
  GH -->|"docker push"| ECR
  User -->|ssh| PubA
  User -->|ssh| PubB
  User -->|kubectl| NLB
  Internet -->|"6443 (k8s API)<br/>80/443 (Istio Gateway)"| NLB
  NLB -->|"6443 → master<br/>30080/30443 → worker NodePort"| PriA
  NLB -->|"6443 → master<br/>30080/30443 → worker NodePort"| PriB
  PubA --- IGW
  PubB --- IGW
  PriA -.via NAT.- PubA
  PriB -.via NAT.- PubB
  PriA -.NFS.- EFS
  PriB -.NFS.- EFS
  PriA -.443.- VPCE
  PriB -.443.- VPCE
  VPCE -.ECR pull.- ECR
  User -->|terraform init| Bucket

  classDef vpc fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
  classDef az2a fill:#e3f2fd,stroke:#1565c0,color:#000
  classDef az2b fill:#f3e5f5,stroke:#6a1b9a,color:#000
  classDef pub fill:#bbdefb,stroke:#1976d2,color:#000
  classDef pri fill:#e1bee7,stroke:#7b1fa2,color:#000
  classDef acct fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
  classDef region fill:#f1f8e9,stroke:#558b2f,stroke-width:2px,color:#000
  classDef ext fill:#f5f5f5,stroke:#424242,color:#000
  classDef nlb fill:#b2dfdb,stroke:#00695c,color:#000
  classDef ecr fill:#ffcdd2,stroke:#c62828,color:#000
  classDef storage fill:#cfd8dc,stroke:#37474f,color:#000

  class VPCBox vpc
  class A2A az2a
  class A2B az2b
  class PubA,PubB pub
  class PriA,PriB pri
  class AcctLevel acct
  class Region region
  class S3Ext ext
  class Internet,User,GH ext
  class NLB nlb
  class ECR,Role,OIDC,KMS ecr
  class EFS,VPCE,Bucket storage
  class IGW pri
```

---

## 검증 체크리스트

- ✅ VPC CIDR `10.0.0.0/16`, 4개 subnet CIDR 정확
  - Public 2a `10.0.1.0/24` + Public 2b `10.0.3.0/24` (NAT Gateway + bastion)
  - Private 2a `10.0.2.0/24` + Private 2b `10.0.4.0/24` (master/worker/EFS Mount Target/VPC Endpoint ENI)
- ✅ public subnet에 IGW 경유 / private subnet에 NAT 경유 (route table 분리, RT × 3)
- ✅ NLB는 subnet_mapping으로 양 AZ public subnet에 attach + 각 AZ EIP
- ✅ **NLB listener × 3** = 6443 (k8s API → master target group) + 80 (HTTP → istio-http-tg) + 443 (HTTPS → istio-https-tg) — R-35 (d) / PR #22
- ✅ **NLB target group × 3** = k8s-api-tg (6443, master × 3) + istio-http-tg (30080, worker × 3) + istio-https-tg (30443, worker × 3) — attachment 9 개
- ✅ bastion은 public subnet (`associate_public_ip_address=true`, EIP attach 아님). 별도 bastion subnet 없음
- ✅ master/worker는 private subnet + instance profile (`ktcloud-node-profile`)
  - master 3 × t3.medium (control plane only)
  - worker 3 × **t3.large** (R-59 메모리 상향)
- ✅ EFS mount target은 private subnet ENI (양 AZ)
- ✅ VPC Endpoint Interface (ecr.api/ecr.dkr)는 private subnet ENI, private DNS enabled
- ✅ VPC Endpoint Gateway (s3)는 route_table_ids 매핑 (private RT × 2)
- ✅ SG 4개 + 의도된 ingress rule 명시
  - `cluster-node-sg` 에 **30080 + 30443 from `0.0.0.0/0`** (PR #22, NLB source IP preservation 대응)
- ✅ IAM OIDC + Role trust policy = `repo:.../msa-*:ref:refs/heads/main`
- ✅ ECR push (CI) vs ECR pull (kubelet) — 다른 경로, 분리 표현
- ✅ 영구/임시/수동 자원 분류 정확

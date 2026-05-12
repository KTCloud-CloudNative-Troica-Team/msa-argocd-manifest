# Troica AWS Architecture (상세)

`msa-provisioning` Terraform이 만드는 모든 AWS 리소스를 레벨별로 분할 시각화.

> 검증 기준: 각 리소스의 정확한 scope (계정/리전/VPC/AZ/Subnet) + 트래픽 방향.
> 자세한 코드: [msa-provisioning/terraform](https://github.com/KTCloud-CloudNative-Troica-Team/msa-provisioning/tree/main/terraform)

---

## 1. Network 계층 (VPC + AZ + Subnet + Route)

**검증 포인트**:
- VPC CIDR `10.0.0.0/16`, AZ 2개 (2a/2b), public/private subnet 각 2개 = 총 4 subnet
- public RT 1개 공유 (0.0.0.0/0 → IGW)
- private RT 2개 (각 AZ별, 0.0.0.0/0 → 해당 AZ의 NAT)
- NLB는 VPC 레벨이지만 양 AZ public subnet에 매핑 (subnet_mapping × 2)

```mermaid
flowchart TB
  Internet([🌐 Internet])

  subgraph VPC["VPC: kt-cloud-vpc (10.0.0.0/16)"]
    IGW[Internet Gateway<br>main_igw]

    subgraph PubRT["Public Route Table (kt-cloud-public-rt)"]
      PubRoute["route: 0.0.0.0/0 → IGW"]
    end

    subgraph AZ2A["AZ: ap-northeast-2a"]
      direction TB
      PubA["Public Subnet<br>10.0.1.0/24<br>tag: kubernetes.io/role/elb=1"]
      PriA["Private Subnet<br>10.0.2.0/24<br>tag: kubernetes.io/role/internal-elb=1"]
      PriRTA["Private RT 2a<br>route: 0.0.0.0/0 → NAT 2a"]
      NATA["NAT Gateway 2a<br>(in Public Subnet 2a)"]
      NEIPa[EIP nat-eip-2a]
    end

    subgraph AZ2B["AZ: ap-northeast-2b"]
      direction TB
      PubB["Public Subnet<br>10.0.3.0/24<br>tag: kubernetes.io/role/elb=1"]
      PriB["Private Subnet<br>10.0.4.0/24<br>tag: kubernetes.io/role/internal-elb=1"]
      PriRTB["Private RT 2b<br>route: 0.0.0.0/0 → NAT 2b"]
      NATB["NAT Gateway 2b<br>(in Public Subnet 2b)"]
      NEIPb[EIP nat-eip-2b]
    end

    NLB["NLB kt-cloud-nlb<br>(VPC-scoped, subnet_mapping × 2)"]
    NEIP1[EIP nlb_eip_2a]
    NEIP2[EIP nlb_eip_2b]
  end

  Internet -->|inbound 6443| NLB
  NLB -.assoc.- NEIP1
  NLB -.assoc.- NEIP2
  NLB -->|subnet_mapping| PubA
  NLB -->|subnet_mapping| PubB

  Internet -->|inbound 22 SSH| IGW
  IGW <-->|attach| VPC

  PubA -.RT assoc.- PubRT
  PubB -.RT assoc.- PubRT
  PubRT -->|0.0.0.0/0| IGW

  PriA -.RT assoc.- PriRTA
  PriRTA -->|0.0.0.0/0| NATA
  NATA -.assoc.- NEIPa
  NATA --> IGW

  PriB -.RT assoc.- PriRTB
  PriRTB -->|0.0.0.0/0| NATB
  NATB -.assoc.- NEIPb
  NATB --> IGW
```

---

## 2. Compute + Storage (EC2 + EFS + EBS + VPC Endpoint)

**검증 포인트**:
- EC2 8대: master 3 (a:2 + b:1) + worker 3 (a:1 + b:2) + bastion 2 (a:1 + b:1)
- bastion만 public subnet (`associate_public_ip_address=true`)
- master/worker는 private subnet (IAM instance profile = `ktcloud-node-profile` with AmazonEC2ContainerRegistryReadOnly)
- EBS × 3: worker 노드에만 (20GB, /dev/sdh)
- EFS file system 1개 + Mount Target 2개 (각 AZ private subnet에 ENI)
- VPC Endpoint Interface × 2 (ecr.api + ecr.dkr): private DNS enabled, ENI는 양 AZ private subnet
- VPC Endpoint Gateway × 1 (s3): route table 매핑 (private RT 2a + 2b)

```mermaid
flowchart LR
  subgraph VPC["VPC kt-cloud-vpc"]
    direction LR

    subgraph AZ2A["AZ ap-northeast-2a"]
      direction TB
      subgraph PubA["Public Subnet 10.0.1.0/24"]
        BastA["EC2 bastion-2a<br>t3.nano<br>public IP"]
      end
      subgraph PriA["Private Subnet 10.0.2.0/24"]
        MA1["EC2 master-01<br>t3.medium<br>instance-profile"]
        MA2["EC2 master-02<br>t3.medium<br>instance-profile"]
        WA["EC2 worker-01<br>t3.medium<br>instance-profile"]
        EBSa["EBS 20GB<br>/dev/sdh"]
        EFSmA["EFS Mount Target<br>(ENI in 2a)"]
        VPCEa1["VPCE ENI<br>ecr.api"]
        VPCEa2["VPCE ENI<br>ecr.dkr"]
      end
    end

    subgraph AZ2B["AZ ap-northeast-2b"]
      direction TB
      subgraph PubB["Public Subnet 10.0.3.0/24"]
        BastB["EC2 bastion-2b<br>t3.nano<br>public IP"]
      end
      subgraph PriB["Private Subnet 10.0.4.0/24"]
        MB1["EC2 master-01<br>t3.medium<br>instance-profile"]
        WB1["EC2 worker-01<br>t3.medium<br>instance-profile"]
        WB2["EC2 worker-02<br>t3.medium<br>instance-profile"]
        EBSb1["EBS 20GB<br>/dev/sdh"]
        EBSb2["EBS 20GB<br>/dev/sdh"]
        EFSmB["EFS Mount Target<br>(ENI in 2b)"]
        VPCEb1["VPCE ENI<br>ecr.api"]
        VPCEb2["VPCE ENI<br>ecr.dkr"]
      end
    end

    EFS["EFS File System<br>kt-cloud-cluster-efs<br>(region-scoped)"]
    VPCE_API["VPC Endpoint<br>ecr.api Interface<br>private DNS"]
    VPCE_DKR["VPC Endpoint<br>ecr.dkr Interface<br>private DNS"]
    VPCE_S3["VPC Endpoint<br>s3 Gateway"]
  end

  WA -.volume_attachment.- EBSa
  WB1 -.volume_attachment.- EBSb1
  WB2 -.volume_attachment.- EBSb2

  EFS -.mount_target_2a.- EFSmA
  EFS -.mount_target_2b.- EFSmB
  EFSmA -->|NFS 2049 from VPC| PriA
  EFSmB -->|NFS 2049 from VPC| PriB

  VPCE_API -.subnet_ids.- VPCEa1
  VPCE_API -.subnet_ids.- VPCEb1
  VPCE_DKR -.subnet_ids.- VPCEa2
  VPCE_DKR -.subnet_ids.- VPCEb2

  VPCE_S3 -.route_table_ids.- PriA
  VPCE_S3 -.route_table_ids.- PriB
```

---

## 3. Security Groups + 트래픽 흐름

**검증 포인트**:
- 4 SG: `cluster-node-sg` (master/worker), `bastion-node-sg` (bastion), `kt-cloud-cluster-efs-sg` (EFS), `troica-vpc-endpoint-sg` (VPC Endpoint ENI)
- `cluster-node-sg`: 6443 from 0.0.0.0/0 (NLB ingress), intra-VPC TCP/UDP, ICMP, SSH from bastion-node-sg
- `bastion-node-sg`: SSH 22 + ICMP from user's IP (`data.http.my_ip`)
- `kt-cloud-cluster-efs-sg`: NFS 2049 from 10.0.0.0/16 (intra-VPC)
- `troica-vpc-endpoint-sg`: HTTPS 443 from 10.0.0.0/16 (intra-VPC)

```mermaid
flowchart TB
  User([👤 User<br>data.http.my_ip])
  GitHub([🐙 GitHub Actions<br>OIDC])
  Internet([🌐 Internet])

  subgraph VPC["VPC kt-cloud-vpc (10.0.0.0/16)"]
    direction TB

    subgraph SGBastion["SG bastion-node-sg"]
      direction LR
      Bastion[bastion × 2<br>public subnets]
    end

    subgraph SGCluster["SG cluster-node-sg"]
      direction LR
      Cluster[master × 3 + worker × 3<br>private subnets]
    end

    subgraph SGEFS["SG kt-cloud-cluster-efs-sg"]
      EFSMounts[EFS Mount Targets × 2]
    end

    subgraph SGVPCE["SG troica-vpc-endpoint-sg"]
      VPCEs[VPC Endpoint ENIs<br>ecr.api + ecr.dkr]
    end

    NLB[NLB kt-cloud-nlb]
  end

  User -->|22 SSH<br>ICMP| Bastion
  Internet -->|6443<br>k8s API| NLB
  NLB -->|6443| Cluster

  Bastion -->|22 SSH<br>jump host| Cluster
  Cluster -->|intra-VPC<br>10.0.0.0/16<br>TCP/UDP all| Cluster
  Cluster -->|2049 NFS| EFSMounts
  Cluster -->|443 HTTPS| VPCEs
```

---

## 4. CI/CD (계정 레벨 IAM + ECR + KMS — GitHub Actions OIDC flow)

**검증 포인트**:
- IAM OIDC Provider: `token.actions.githubusercontent.com` (계정 레벨, 단일)
- IAM Role `troica-gha-ecr-push`: name-based ARN (destroy/apply 사이클 안정)
- Trust policy: `repo:KTCloud-CloudNative-Troica-Team/msa-*:ref:refs/heads/main` 만 assume 허용
- Inline Policy: ECR push (BatchGetImage, PutImage, UploadLayerPart 등) on `repository/msa/*`
- 6개 ECR Repo: KMS encrypted (`AES_256` via `aws_kms_key.ecr`), IMMUTABLE tag, 30-image lifecycle
- 노드의 ECR pull은 별도 경로: EC2 instance profile에 `AmazonEC2ContainerRegistryReadOnly` attached (kubelet credential provider가 호출)

```mermaid
flowchart LR
  subgraph GH["GitHub Organization<br>KTCloud-CloudNative-Troica-Team"]
    direction TB
    Repo[msa-* repository<br>main branch push]
    Workflow[CI workflow<br>aws-actions/configure-aws-credentials]
    Repo --> Workflow
  end

  subgraph AWS["AWS Account 601766312629"]
    direction TB

    subgraph IAM["IAM (account-level)"]
      direction TB
      OIDC["OIDC Provider<br>token.actions.githubusercontent.com<br>thumbprint 6938fd4d..."]
      Role["IAM Role<br>troica-gha-ecr-push<br>Trust: repo:.../msa-*:ref:refs/heads/main"]
      Policy["Inline Policy<br>ecr-push:<br>GetAuthorizationToken<br>BatchGet/PutImage<br>UploadLayerPart"]
      OIDC -->|Federated principal| Role
      Role --> Policy
    end

    subgraph KMSGroup["KMS"]
      KMS[KMS Key ecr<br>rotation enabled<br>deletion_window 7d]
      Alias[Alias alias/troica-ecr]
      KMS --- Alias
    end

    subgraph ECRGroup["ECR Repositories (× 6, IMMUTABLE)"]
      direction TB
      ECR1[msa/user-service]
      ECR2[msa/auth-service]
      ECR3[msa/product-service]
      ECR4[msa/order-service]
      ECR5[msa/inventory-service]
      ECR6[msa/api-gateway]
    end

    Policy -.scoped to.- ECRGroup
    KMS -.encryption.- ECR1
    KMS -.encryption.- ECR2
    KMS -.encryption.- ECR3
    KMS -.encryption.- ECR4
    KMS -.encryption.- ECR5
    KMS -.encryption.- ECR6
  end

  Workflow -->|1. AssumeRoleWithWebIdentity<br>(STS)| OIDC
  Workflow -->|2. session credentials| Role
  Workflow -->|3. docker push| ECR3
```

---

## 5. EC2 노드의 ECR Image Pull (kubelet 경로)

**검증 포인트** — 위 CI/CD flow와 별개 (push vs pull 분리):
- EC2 instance profile `ktcloud-node-profile` → IAM Role (`ktcloud-cluster-node-role`, data source) → `AmazonEC2ContainerRegistryReadOnly` AWS managed policy attached
- kubelet에 ecr-credential-provider binary 설치 (R-29: Go 빌드 후 ansible copy)
- kubelet config `/etc/kubernetes/credential-provider-config.yaml`로 ECR matchImages 지정
- kubelet args `--image-credential-provider-config=...` 활성 (R-30: `kubeadm-flags.env` 통합)
- 트래픽 흐름: kubelet → ECR API/DKR (VPC Endpoint Interface) + 이미지 layer 다운로드는 S3 (VPC Endpoint Gateway)

```mermaid
flowchart TB
  subgraph Node["EC2 Node (master/worker, private subnet)"]
    direction TB
    Kubelet["kubelet<br>--image-credential-provider-config<br>--image-credential-provider-bin-dir"]
    Provider["ecr-credential-provider binary<br>/usr/local/bin/<br>(Go-built, R-29)"]
    Profile["EC2 instance_profile<br>ktcloud-node-profile<br>→ ECR ReadOnly"]
    Kubelet --> Provider
    Provider --> Profile
  end

  subgraph VPC["VPC (intra)"]
    VPCE_API[VPC Endpoint<br>ecr.api Interface<br>private DNS]
    VPCE_DKR[VPC Endpoint<br>ecr.dkr Interface<br>private DNS]
    VPCE_S3[VPC Endpoint<br>s3 Gateway]
  end

  subgraph ECR["ECR (account-level)"]
    ECRRepo[ECR Repository<br>msa/&lt;service&gt;]
    S3Backing[(S3<br>image layers<br>AWS-managed)]
  end

  Provider -->|1. GetAuthorizationToken| VPCE_API
  VPCE_API --> ECRRepo
  Kubelet -->|2. pull manifest| VPCE_DKR
  VPCE_DKR --> ECRRepo
  Kubelet -->|3. pull layers| VPCE_S3
  VPCE_S3 --> S3Backing
```

---

## 6. 영구 / 임시 / 수동 자원 분류

**검증**: 비용 사이클(`destroy-temp.sh`) 시 무엇이 destroy되고 무엇이 유지되는지.

```mermaid
flowchart LR
  subgraph Permanent["영구 (prevent_destroy = true, ~$1/월)"]
    direction TB
    P1[IAM OIDC Provider]
    P2[IAM Role troica-gha-ecr-push]
    P3[IAM Role Policy]
    P4[KMS Key + Alias]
    P5[ECR Repo × 6]
    P6[VPC + Subnet × 4]
    P7[Internet Gateway]
    P8[Route Tables × 3]
    P9[Security Groups × 4]
    P10[Key Pair]
    P11[IAM Instance Profile<br>+ Role attachment]
  end

  subgraph Temporary["임시 (destroy-temp.sh 대상, ~$300/월)"]
    direction TB
    T1[EC2 × 8<br>master 3 + worker 3 + bastion 2]
    T2[EBS × 3 + Volume Attachment × 3]
    T3[NAT Gateway × 2 + NAT EIP × 2]
    T4[NLB + Target Group + Listener + Attachments × 3]
    T5[NLB EIP × 2]
    T6[VPC Endpoint × 3<br>ecr.api + ecr.dkr + s3]
    T7[EFS File System + Mount Target × 2 + EFS SG]
    T8[local_file ansible_inventory]
  end

  subgraph Manual["수동 (Terraform 외부, ~$0.5/월)"]
    M1[S3 Backend Bucket<br>troica-tfstate-&lt;SUFFIX&gt;]
  end
```

---

## 통합 토폴로지 (한 화면 요약)

```mermaid
flowchart TB
  Internet([🌐 Internet])
  GH([🐙 GitHub Actions])
  User([👤 User])

  subgraph Acct["AWS Account (601766312629) — 영구"]
    OIDC[IAM OIDC]
    Role[Role gha-ecr-push]
    KMS[KMS]
    ECR[ECR × 6]
    OIDC --> Role
    Role -.- ECR
    KMS -.- ECR
  end

  subgraph Region["Region ap-northeast-2"]
    subgraph VPCBox["VPC kt-cloud-vpc 10.0.0.0/16"]
      IGW[IGW]
      NLB[NLB]
      EFS[EFS]
      VPCE[VPC Endpoint × 3]

      subgraph A2A["AZ 2a"]
        PubA["public 10.0.1.0/24<br>NAT-2a, bastion-2a"]
        PriA["private 10.0.2.0/24<br>master-01/02, worker-01"]
      end

      subgraph A2B["AZ 2b"]
        PubB["public 10.0.3.0/24<br>NAT-2b, bastion-2b"]
        PriB["private 10.0.4.0/24<br>master-01, worker-01/02"]
      end
    end
  end

  subgraph S3Ext["수동"]
    Bucket[S3 backend bucket]
  end

  GH -->|OIDC AssumeRole| OIDC
  GH -->|docker push| ECR
  Internet -->|22| PubA
  Internet -->|22| PubB
  Internet -->|6443| NLB
  NLB --> PriA
  NLB --> PriB
  PubA --- IGW
  PubB --- IGW
  PriA -.NAT.- PubA
  PriB -.NAT.- PubB
  PriA -.NFS.- EFS
  PriB -.NFS.- EFS
  PriA -.443.- VPCE
  PriB -.443.- VPCE
  VPCE -.ECR pull.- ECR
  User -->|terraform init| Bucket
```

---

## 검증 체크리스트 (다이어그램 작성 시 확인)

- ✅ VPC CIDR `10.0.0.0/16`, 4개 subnet CIDR 정확 (1/2/3/4 .0/24)
- ✅ public subnet에 IGW route (kt-cloud-public-rt 공유)
- ✅ private subnet에 NAT route (각 AZ별 분리)
- ✅ NLB는 subnet_mapping으로 양 AZ public subnet에 attach + 각 AZ EIP
- ✅ bastion은 public subnet (`associate_public_ip_address=true`)
- ✅ master/worker는 private subnet + instance profile
- ✅ EFS mount target은 private subnet ENI
- ✅ VPC Endpoint Interface(ecr.api/ecr.dkr)는 private subnet ENI, private DNS enabled
- ✅ VPC Endpoint Gateway(s3)는 route_table_ids 매핑 (private RT × 2)
- ✅ SG 4개 + 의도된 ingress rule 명시
- ✅ IAM OIDC + Role trust policy = `repo:.../msa-*:ref:refs/heads/main`
- ✅ ECR push (CI) vs ECR pull (kubelet)은 다른 경로 — 분리 표현
- ✅ 영구/임시/수동 자원 분류 정확

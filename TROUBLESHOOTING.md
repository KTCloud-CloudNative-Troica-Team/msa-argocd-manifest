# Troica Troubleshooting

> Troica polyrepo 마이그레이션 (Phase 0~) 진행 중 마주친 이슈와 해결책 누적.
>
> - **BACKLOG.md**: 작업 진행 상태 (완료/진행 중/대기)
> - **본 문서**: 디버깅 자료 — 증상 / 원인 / 해결 / 재발 방지
>
> 같은 이슈 재발 시 본 문서 검색 → 빠른 복구.

## 목차

1. [빌드 / 의존성](#1-빌드--의존성)
2. [보안 / CVE](#2-보안--cve)
3. [컨테이너 빌드](#3-컨테이너-빌드)
4. [CI/CD](#4-cicd)
5. [Terraform / AWS 인프라](#5-terraform--aws-인프라)
6. [Windows / PowerShell](#6-windows--powershell)
7. [Kubernetes / ArgoCD](#7-kubernetes--argocd)
8. [kubelet / Ansible](#8-kubelet--ansible)

---

## 1. 빌드 / 의존성

### 1.1 gradlew permission denied (CI)

**증상**: GitHub Actions에서 `./gradlew build` 실행 시 `Permission denied (exit 126)`.

**원인**: Windows에서 만든 `gradlew`가 git index에 `100644`로 저장됨. Linux CI runner에서 실행 권한 없음.

**해결**: 첫 commit 전 한 번 실행:
```bash
git update-index --chmod=+x gradlew
git commit -m "fix(gradlew): mark executable"
```
이후 git에서 `100755`로 추적되어 어느 OS에서 clone해도 정상.

**재발 방지**: 신규 Gradle 레포 생성 시 첫 commit에 매번 적용. `MEMORY.md`에 등재됨.

---

### 1.2 Kotlin 2.3.x + Gradle 9.x `BuildUtilKt.clearJarCaches` NoClassDefFoundError

**증상**: `./gradlew build` 실행 시:
```
java.lang.NoClassDefFoundError:
  org/jetbrains/kotlin/incremental/ClasspathEntrySnapshotter$Settings
  at BuildUtilKt.clearJarCaches(BuildUtilKt.kt:...)
```

**원인**: Kotlin 2.3.20/2.3.21의 Build Tools API JAR 내부 클래스 참조 깨짐. JetBrains upstream 버그.

**시도한 워크어라운드 (모두 실패)**:
- `kotlin.compiler.runViaBuildToolsApi=false`
- `kotlin.incremental.useClasspathSnapshot=false`
- `kotlin.incremental=false`

**해결**: **Kotlin 2.1.0 + Gradle 8.10.2 고정.** 모노레포 toolchain(2.3.20/9.0.0) 사용 금지.

**재발 방지**: 모든 신규 서비스 레포의 `build.gradle.kts` + `gradle/wrapper/gradle-wrapper.properties`에서 버전 고정.

---

### 1.3 JitPack `client-redis` Kotlin metadata 호환

**증상**: inventory-service 빌드 시:
```
The class file ... was compiled by a newer version of Kotlin compiler than 2.1
```

**원인**: 팀장님이 JitPack에 게시한 `com.github.kanei0415:ktcloud-msa-client-redis:v1.0.2`가 Kotlin 2.3.x로 컴파일됨. 우리 컴파일러는 2.1.0.

**해결**: consumer 측 (inventory-service `build.gradle.kts`)에 컴파일러 인자 추가:
```kotlin
kotlin {
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict", "-Xskip-metadata-version-check")
    }
}
```

**재발 방지**: JitPack 의존성 사용하는 모든 서비스에 동일 컴파일러 인자.

---

### 1.4 common-libs `0.x.0-SNAPSHOT` GitHub Packages 부재

**증상**: 서비스 CI에서 `Could not find com.troica.msa:common:0.x.0-SNAPSHOT`.

**원인**: common-libs는 git tag (`v0.x.0`) push 시 GH Packages에 publish됨. SNAPSHOT은 publish 안 됨.

**해결**: 머지 직전에 의존성 버전을 `0.x.0-SNAPSHOT` → `0.x.0` (stable)로 bump. common-libs는 tag push로 publish.

**재발 방지**: 패턴화됨. v0.1.0, v0.2.0, v0.3.0 모두 동일 절차로 처리.

---

### 1.5 Spring Batch ItemWriter Kotlin lambda 타입 추론 실패

**증상**: `inventory-service`의 Spring Batch worker 빌드 시:
```kotlin
Type mismatch: inferred type is (Chunk<...>) -> Unit but ItemWriter<...> was expected
```

**원인**: Kotlin SAM(single abstract method) 변환이 Spring Batch의 `ItemWriter` 제네릭 wildcard와 충돌.

**해결**: 람다 대신 `object : ItemWriter<...>` 명시:
```kotlin
object : ItemWriter<InventoryEventDomainEntity> {
    override fun write(chunk: Chunk<out InventoryEventDomainEntity>) {
        // ...
    }
}
```

---

### 1.6 inventory-service가 `:inventory-event` 모듈 transitive 미인식

**증상**: `inventory-service` 빌드 시 `Unresolved reference: InventoryEventDomainEntity`.

**원인**: 의존성 그래프상 transitive로 들어올 줄 알았으나 명시적 선언 필요.

**해결**: `inventory-service/build.gradle.kts`에 직접 추가:
```kotlin
implementation(project(":inventory-event"))
```

---

## 2. 보안 / CVE

### 2.1 Trivy action `@0.24.0` 미존재 + 공급망 공격

**증상**: CI에서 `aquasecurity/trivy-action@0.24.0` → `Prepare all required actions` 단계 실패 (conditional 평가 전).

**원인**: 2026-03-19 trivy-action 공급망 사고로 태그 `0.0.1~0.34.2`가 모두 compromised. 사고 후 새 컨벤션은 `v` prefix.

**해결**: `@v0.36.0` (2026-04-22, post-incident)로 pin. SPEC §7 + 모든 6개 서비스 ci.yml 적용.

**재발 방지**: 향후 SHA pin + Dependabot 도입 권장 (R-15).

---

### 2.2 Spring Boot 3.3.0 CVE-2025-22235 + EOL

**증상**: 기술 스택 audit 중 Spring Boot 3.3.x EOL (2025-06-30) + CVE-2025-22235 (HIGH, CVSS 7.3) 발견.

**원인**: 모노레포가 SB 3.3.0 사용 중. polyrepo로 옮기면서 그대로 따라옴.

**해결**: SB 3.5.13으로 일괄 bump (common-libs + 모든 서비스). Spring Cloud 2025.0.2 (Northfields, SB 3.5.x 공식 페어) 호환 확인.

**잘못된 가정 정정**: 팀장님은 "3.5.14 호환되는 Spring Cloud 없다"고 함. 실제로는 2025.0.2가 SB 3.5.x 전체 호환. 2025.1.0이 SB 4.x 전용으로 헷갈렸음.

---

### 2.3 Trivy가 main push 첫 시도에 12개 HIGH CVE 차단

**상황**: Phase 0 `AWS_DEPLOYMENTS_ENABLED=true` 활성화 후 첫 ECR push 시도. Trivy 보안 게이트가 작동.

**잡힌 CVE**:
| CVE | Library | Fix | 처리 |
|---|---|---|---|
| CVE-2026-40973 | spring-boot 3.5.13 | 3.5.14 | dependency bump |
| CVE-2026-34483/34487 | tomcat-embed-core 10.1.53 | 10.1.54 | SB 3.5.14 BOM 자동 |
| CVE-2026-42198 | postgresql 42.7.10 | 42.7.11 | 명시적 override (SB BOM 미반영) |
| CVE-2025-55163 | grpc-netty-shaded 1.68.1 | 1.75.0 | grpc 명시 bump |
| CVE-2026-42583 | netty-codec 4.1.132 (Lettuce/Kafka에서) | 4.1.133 | 명시 override |
| CVE-2026-42579 | netty-codec-dns 4.1.132 | 4.1.133 | 명시 override |
| CVE-2026-42584/42587 | netty-codec-http(2) 4.1.132 | 4.1.133 | 명시 override (api-gateway) |
| CVE-2026-42577 | netty-transport-native-epoll 4.1.x | 4.2.13 only | `.trivyignore` accepted risk |
| CVE-2026-5598 | bcprov-jdk18on 1.80 | 1.84 | 명시 override |

**해결 결과**: 모든 CVE 해결 또는 정당화. CI 통과 → ECR push 성공.

**시연 가치**: "자동 보안 게이트가 prod로 가는 길목에서 12개 HIGH CVE 차단" — Tech UP 발표 핵심.

---

### 2.4 `.trivyignore` 명시적 accepted risk (api-gateway)

**상황**: CVE-2026-42577 (netty-transport-native-epoll DoS via RST) — fix가 netty 4.2.13.Final에만 있음 (4.1.x line은 fix 없음).

**원인**: reactor-netty 1.2.x (Spring Boot 3.5.x BOM)는 netty 4.1.x 기준. 4.2.x로 bump 시 ABI 호환 보장 안 됨.

**해결**: `.trivyignore`로 명시적 accepted risk. 위협 모델 근거 (private k8s + Istio Gateway 뒤 → 직접 TCP RST exploit 비현실적). 해제 조건은 R-28 (SB가 netty 4.2.x BOM 갱신 시).

---

### 2.5 평문 JWT secret 노출 (auth-service)

**증상**: `application.yaml`에 `secret: ${JWT_SECRET:v7S6A9yB2E5H8KcNfUjXnZr4u7x!A%D*G-}` — 평문 fallback이 GitHub public repo에 노출.

**원인**: 모노레포 PoC의 평문이 polyrepo로 그대로 따라옴.

**해결 (1차)**: fallback을 명백한 placeholder로 치환:
```yaml
secret: ${JWT_SECRET:DEV_ONLY_INSECURE_REPLACE_ME_WITH_REAL_SECRET}
```
**해결 (정식, Phase 5)**: ExternalSecrets Operator + AWS Secrets Manager.

---

## 3. 컨테이너 빌드

### 3.1 Alpine(musl) + protoc-gen-grpc-java glibc 비호환

**증상**: Docker build의 `:product-service:generateProto` 단계에서:
```
protoc-gen-grpc-java-1.68.1-linux-x86_64.exe:
program not found or is not executable
```

**오해의 소지**: "program not found"는 파일은 있지만 dynamic linker가 못 로드하는 상황.

**원인**: Dockerfile build stage가 `eclipse-temurin:21-jdk-alpine` (musl libc). Maven Central의 `protoc-gen-grpc-java` 바이너리는 glibc 동적 링크.

**해결**: build stage만 glibc 베이스로:
```dockerfile
# Before
FROM eclipse-temurin:21-jdk-alpine AS build

# After
FROM eclipse-temurin:21-jdk AS build      # Ubuntu/Debian, glibc
```
Runtime stage는 alpine 유지 → 최종 image 크기 그대로. 6개 서비스 모두 적용.

**왜 이제 발견됐나**: `AWS_DEPLOYMENTS_ENABLED=false`일 때 Docker build step 자체가 skip됨 → 게이트 활성화 첫 시도에 노출.

---

## 4. CI/CD

### 4.1 CI 영구 red — OIDC AssumeRole 실패

**증상**: Phase 3 머지 후 main 빌드가 OIDC role 없어서 영구 빨강. 팀이 "원래 빨갛다"에 익숙해질 위험.

**원인**: Phase 0 (OIDC role 생성) 전에 push-gated step이 무조건 실행됨.

**해결**: 모든 push-gated step에 게이트 추가:
```yaml
if: github.event_name == 'push' && vars.AWS_DEPLOYMENTS_ENABLED == 'true'
```
Phase 0 완료 + Org Variable `AWS_DEPLOYMENTS_ENABLED=true` 설정 시 자동 활성.

---

### 4.2 ECR `GetAuthorizationToken` rate limit (병렬 머지)

**증상**: 5개 서비스 동시 머지 후 3개에서 ECR login timeout:
```
net/http: request canceled while waiting for connection
(Client.Timeout exceeded while awaiting headers)
```

**원인**: GitHub Actions runner의 egress IP에서 5개 동시 `aws ecr get-login-password` → AWS가 source IP rate limit. Fresh account라 limit이 더 낮음.

**해결**: 머지를 직렬로 (한 번에 하나, 1-2분 간격). 또는 Re-run을 1개씩.

**재발 방지**: R-27 (f) CI 워크플로우에 ECR login retry wrapper 추가 권장.

---

### 4.3 CI 빌드 ~4분 — 정상이지만 비효율

**구성 요소**:
- Gradle 호스트 빌드 (테스트 포함): ~1.5분
- Docker build (Gradle 내부 재실행): ~1.5분 (중복!)
- Trivy DB 다운로드 (첫 회): ~1분 (vuln-db 92MB + java-db 865MB)
- ECR push + manifest bump: ~30s

**개선 여지** (R-27, Phase 5):
- Gradle 중복 제거 → -1.5분
- Docker BuildKit cache mount → -30~60s
- Trivy cache key 주 단위 → -1분 (daily first-run only)

---

## 5. Terraform / AWS 인프라

### 5.1 cloud-provider-aws release에 binary asset 없음

**증상**: ansible `get_url`로 `ecr-credential-provider-linux-amd64` 다운로드 → 404.

**원인**: cloud-provider-aws v1.30.x release 페이지의 `assets`가 0개. AWS EKS AMI에 미리 설치되어 있어서 binary 별도 배포 안 함.

**시도한 버전 점검**:
- v1.30.5 (우리가 처음 시도) — release 자체 없음
- v1.30.10 (1.30 라인 최신) — release 있지만 assets=0

**해결**: Go 소스에서 직접 빌드 → ansible로 노드에 copy:
```bash
sudo apt install -y golang-go
git clone --depth 1 --branch v1.30.10 https://github.com/kubernetes/cloud-provider-aws.git
cd cloud-provider-aws
CGO_ENABLED=0 go build -o /tmp/ecr-credential-provider ./cmd/ecr-credential-provider
```
ansible playbook이 `/tmp/ecr-credential-provider`를 노드로 copy.

---

### 5.2 terraform fmt 오류 — JSON StringLike 줄 분리

**증상**: `terraform fmt -check` 실패 — JSON `StringLike` 내 줄 분리 인식 안 됨.

**해결**: 해당 JSON 값을 한 줄로 합침.

---

### 5.3 PR 2가 PR 1 자원 참조 → terraform validate 실패

**증상**: PR 2의 `aws_iam_role_policy_attachment`가 PR 1의 `data.aws_iam_role` 참조. PR 2 단독 validate 실패.

**해결**: stacked PR — PR 2를 PR 1 branch에 rebase. PR 1 머지 후 PR 2가 main 기준으로 정렬.

---

## 6. Windows / PowerShell

### 6.1 PowerShell `-flag=value` 인수 파싱 깨짐

**증상**:
```
terraform plan -out=phase-0.tfplan
→ Error: Too many command line arguments
```

**원인**: PowerShell 5.1이 native exe에 `-flag=value`를 두 개 인수로 쪼갬.

**해결 옵션**:
- 공백 분리: `terraform plan -out phase-0.tfplan`
- 인용부호: `terraform plan "-out=phase-0.tfplan"`
- Stop-parsing 토큰: `terraform plan --% -out=phase-0.tfplan`
- Array splatting (여러 개일 때):
  ```powershell
  $t = @("-target=...", "-target=...")
  terraform destroy @t -auto-approve
  ```

---

### 6.2 PowerShell JSON quoting (aws s3api)

**증상**:
```
aws s3api put-bucket-encryption ... --server-side-encryption-configuration '{"Rules": [...]}'
→ Invalid JSON: Expecting property name enclosed in double quotes
```

**원인**: PowerShell 5.1이 single-quoted JSON을 native exe에 전달 시 따옴표 보존 깨짐.

**해결**: AWS CLI shorthand 사용 (JSON 회피):
```powershell
aws s3api put-bucket-encryption `
  --bucket $BUCKET `
  --server-side-encryption-configuration "Rules=[{ApplyServerSideEncryptionByDefault={SSEAlgorithm=AES256}}]"
```

---

### 6.3 `.sh` 파일이 git에서 CRLF로 저장 → bash 실패

**증상**:
```bash
bash destroy-temp.sh
→ $'\r': command not found
→ set: pipefail: invalid option name
```

**원인**: Windows의 git `core.autocrlf` 기본값이 `.sh`를 CRLF로 변환.

**해결**:
- 단기: PowerShell 네이티브 `destroy-temp.ps1` 추가 (별도 PR로 머지됨).
- 영구: `.gitattributes`에 `*.sh text eol=lf` 강제. Windows에서 clone해도 워킹 디렉토리는 LF.

---

### 6.4 `aws s3api put-public-access-block`의 잘못된 파라미터명

**증상**:
```
Unknown parameter in PublicAccessBlockConfiguration: "RestrictPublicAccess"
must be one of: BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, RestrictPublicBuckets
```

**원인**: 정확한 이름은 `RestrictPublicBuckets`. AWS docs 작성 시 흔한 오타.

**해결**: 명령어의 `RestrictPublicAccess=true` → `RestrictPublicBuckets=true`.

---

## 7. Kubernetes / ArgoCD

### 7.1 ArgoCD `directory.include` brace expansion 미동작

**증상**: `bootstrap/root-app.yaml`의:
```yaml
directory:
  include: '{projects/x.yaml,platform/y.yaml,applications/z.yaml}'
```
→ 4개 파일 중 0개 매칭. 자식 Application 0개 생성.

**원인**: ArgoCD가 `glob.Glob` 패턴만 지원. brace expansion `{a,b,c}`는 매칭 안 함.

**해결**: app-of-apps 정식 패턴으로 전환.
- `bootstrap/root-app.yaml`: path=bootstrap, recurse=false
- `bootstrap/apps.yaml` 신규: multi-doc으로 3개 자식 Application

**동일 패턴 fix**: `platform/root.yaml`의 `{*/application.yaml,*/*/application.yaml}` → `**/application.yaml` doublestar.

---

### 7.2 root-app이 팀장님 개인 레포 가리킴

**증상**: 클러스터 부트스트랩 후 `argocd app list`에서:
```
root-app    https://github.com/kanei0415/ktcloud-k8s-argocd-manifest.git    Setup
```
우리 팀 매니페스트가 무시됨.

**원인**: `ansible/argocd-setup.yaml`이 root Application을 inline yaml로 정의하면서 팀장님 PoC 레포 URL hardcoded.

**해결**: ansible playbook이 우리 매니페스트 레포의 `bootstrap/root-app.yaml`을 fetch + apply하는 패턴으로 전환. 매니페스트가 단일 진실의 원천.

---

### 7.3 ArgoCD auto-sync가 "all resources를 wipe out" 가드로 멈춤

**증상**: 새 root-app이 `Synced` 안 되고:
```
Skipping sync attempt: auto-sync will wipe out all resources
```

**원인**: 이전 root-app이 남긴 `charts-app` ApplicationSet이 orphan으로 살아있음. 새 root-app과 desired state diff가 "전부 wipe" → ArgoCD 안전 가드 발동.

**해결**: orphan ApplicationSet 수동 제거 후 재sync:
```bash
kubectl -n argocd delete applicationset charts-app addons-app apps-app 2>/dev/null
argocd app sync root-app
```

---

### 7.4 pod `CreateContainerConfigError` — Secret/ConfigMap 미존재

**증상**: pod 상태 `CreateContainerConfigError` + events:
```
Error: secret "api-gateway-secrets" not found
```

**원인**: Helm chart의 `envFrom`이 `{service}-secrets`, `{service}-config` 참조하지만 외부에서 생성된 적 없음.

**해결 (PoC)**: placeholder Secret/ConfigMap 일괄 생성:
```bash
for svc in api-gateway auth-service user-service product-service order-service inventory-service; do
  for ns in market-dev market-prod; do
    kubectl create secret generic ${svc}-secrets -n $ns --from-literal=PLACEHOLDER=tbd
    kubectl create configmap ${svc}-config -n $ns --from-literal=PLACEHOLDER=tbd
  done
done
```

**정식 해결 (Phase 5)**: ExternalSecrets Operator + AWS Secrets Manager (R-33).

**현상 정정**: 위 임시 fix로 pod는 startup 가능하지만 DB 없어서 Spring Boot가 `CrashLoopBackOff`. 이게 Phase 0의 자연스러운 경계.

---

### 7.5 K8S_VARS_HOLDER unreachable

**증상**: ansible playbook 실행 중:
```
fatal: [K8S_VARS_HOLDER]: UNREACHABLE!
```

**원인**: `join-master.yaml`에서 vars 저장용 가상 호스트. inventory에 없는 상태.

**영향**: 다른 노드의 task는 모두 정상. 가상 호스트의 `gather facts` 실패만 발생. **무시 가능**.

---

## 8. kubelet / Ansible

### 8.1 kubelet `KUBELET_EXTRA_ARGS` drop-in 미평가

**증상**: ansible playbook으로 drop-in 작성:
```ini
# /etc/systemd/system/kubelet.service.d/30-ecr-credential-provider.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--image-credential-provider-config=..."
```
`systemctl show kubelet --property=ExecStart`에 `$KUBELET_EXTRA_ARGS` 보이지만, `ps aux | grep kubelet`에는 ECR 인자 미반영.

**원인**: Amazon Linux 2023 + kubeadm 1.30 환경에서 systemd가 `Environment="KUBELET_EXTRA_ARGS=..."`를 어떤 이유로 평가 안 함. 근본 원인 미규명.

**해결**: `/var/lib/kubelet/kubeadm-flags.env`의 `KUBELET_KUBEADM_ARGS`에 ECR 인자를 직접 추가:
```ini
KUBELET_KUBEADM_ARGS="--image-credential-provider-config=... --image-credential-provider-bin-dir=/usr/local/bin --container-runtime-endpoint=... --pod-infra-container-image=..."
```
playbook의 lineinfile task로 통합. drop-in은 다른 환경 호환 위해 유지.

**검증**: `ps aux | grep kubelet`에 ECR 인자 보이면 성공. ansible playbook에 자동 검증 task 추가:
```yaml
- name: kubelet args에 ECR provider 인자 실제 반영 확인
  shell: ps -ef | grep '[k]ubelet' | grep -q image-credential-provider-config
  failed_when: result.rc != 0
```

---

### 8.2 Docker image pull 시 `no basic auth credentials`

**증상**: pod events:
```
Failed to pull image "601766312629.dkr.ecr.ap-northeast-2.amazonaws.com/msa/api-gateway:..."
pull access denied, repository does not exist or may require authorization:
authorization failed: no basic auth credentials
```

**원인**: kubelet에 ECR credential provider 인자가 안 들어감 (위 8.1 참조). credential provider가 호출되지 않아서 ECR 인증 토큰 미발급.

**해결**: 8.1 fix 적용 후 kubelet 재시작 → ECR pull 정상 작동.

**검증 신호**: events에 `Successfully pulled image ... Image size: XXXMb` 보이면 ECR credential provider 정상.

---

## 부록 — 유용한 디버깅 명령

### kubelet
```bash
# kubelet 실제 args 확인
ps aux | grep '[k]ubelet'

# kubelet drop-in 적용 상태
sudo systemctl show kubelet --property=Environment --property=ExecStart

# kubelet 로그 (특정 키워드)
sudo journalctl -u kubelet --since "10 minutes ago" | grep -iE "credential|ecr|pull"
```

### ArgoCD
```bash
# 모든 application 상태
argocd app list
kubectl -n argocd get application,applicationset,appproject

# 특정 app 디버깅
argocd app get <app-name> --refresh
kubectl -n argocd describe application <app-name>
```

### Pod
```bash
# pod 이벤트 + 상세
kubectl describe pod -n <ns> <pod>
kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -30

# ConfigMap/Secret 참조 확인
kubectl get pod -n <ns> <pod> -o yaml | grep -A 2 -E "configMap|secret|envFrom"
```

### Trivy 로컬 검증
```bash
trivy image <image-ref> --severity HIGH,CRITICAL --ignore-unfixed
```

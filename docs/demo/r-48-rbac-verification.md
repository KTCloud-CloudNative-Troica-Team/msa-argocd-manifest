# R-48 (B) Demo — K8s RBAC 역할 분리 검증 (kubectl auth can-i)

> **평가 cover**:
> - 심화 (3)-1 (필수): 사람 RBAC 역할 분리 (developer / operator / sre)
> - 심화 (3)-2 (필수): ServiceAccount + Role/RoleBinding (R-25/R-33 ExternalSecret + Service 측 RBAC)
>
> **소요**: ~10 분

## 사전 조건

cluster up + Sync Wave -8 (rbac) 완료.

```bash
# 매니페스트 sync 확인 — 3 role + 4 bindings (developer 1 + operator dev/prod 2 + sre 1)
kubectl get clusterrole troica-developer
kubectl get clusterrolebinding troica-developer-binding troica-sre-binding
kubectl -n market-dev get role troica-operator
kubectl -n market-dev get rolebinding troica-operator-binding
kubectl -n market-prod get role troica-operator
kubectl -n market-prod get rolebinding troica-operator-binding
```

**기대**: 모두 NAME 확인됨 (NotFound 없음).

---

## Part 1 — impersonation 으로 RBAC 검증

### 배경 — Group subject + OIDC 미설정

매니페스트의 RoleBinding subjects 가 `kind: Group`. 운영 단계 OIDC provider 통합 후 group 매칭 (Phase 6). **PoC 단계 = `kubectl --as / --as-group` impersonation 으로 검증**.

master 노드의 admin kubeconfig (`system:masters` group) 가 모든 자원의 impersonate verb 보유 → 임의 user/group 흉내 가능.

### 권한 매트릭스 (검증할 것)

| 역할 | Group | 권한 | 차단 |
|---|---|---|---|
| developer | `troica-developers` | get/list/watch (cluster-wide read-only) | create / delete / exec |
| operator | `troica-operators` | market-{dev,prod} 에서 pod delete + deployment patch | 다른 namespace, RBAC 자체 변경 |
| sre | `troica-sre` | cluster-admin (모든 것) | (없음) |

---

## Part 2 — Developer Role 검증 (cluster-wide read-only)

master 노드에서:

### Step 1 — Allow 검증 (성공해야 할 것들)

```bash
# A. 모든 namespace 의 pod 조회 — yes
kubectl auth can-i list pods --all-namespaces \
    --as=dev-user@example.com --as-group=troica-developers

# B. log 조회 — yes
kubectl auth can-i get pods/log -n market-dev \
    --as=dev-user@example.com --as-group=troica-developers

# C. argo application 조회 — yes
kubectl auth can-i get applications -n argocd \
    --as=dev-user@example.com --as-group=troica-developers

# D. NetworkPolicy 조회 — yes
kubectl auth can-i get networkpolicies -n market-dev \
    --as=dev-user@example.com --as-group=troica-developers
```

**기대**: 모두 `yes`.

### Step 2 — Deny 검증 (실패해야 할 것들)

```bash
# A. pod 삭제 — no
kubectl auth can-i delete pods -n market-dev \
    --as=dev-user@example.com --as-group=troica-developers

# B. deployment 수정 — no
kubectl auth can-i patch deployments -n market-dev \
    --as=dev-user@example.com --as-group=troica-developers

# C. exec into pod — no (pods/exec subresource)
kubectl auth can-i create pods/exec -n market-dev \
    --as=dev-user@example.com --as-group=troica-developers

# D. Secret 조회 — no (read-only 라도 Secret 은 권한 매니페스트에 미포함)
kubectl auth can-i get secrets -n market-dev \
    --as=dev-user@example.com --as-group=troica-developers
```

**기대**: 모두 `no`.

---

## Part 3 — Operator Role 검증 (market-{dev,prod} 한정)

### Step 1 — Allow 검증 (market-dev 에서)

```bash
# A. pod 삭제 (재시작용) — yes
kubectl auth can-i delete pods -n market-dev \
    --as=ops-user@example.com --as-group=troica-operators

# B. deployment patch — yes
kubectl auth can-i patch deployments -n market-dev \
    --as=ops-user@example.com --as-group=troica-operators

# C. deployment scale — yes
kubectl auth can-i patch deployments/scale -n market-dev \
    --as=ops-user@example.com --as-group=troica-operators

# D. argo application patch — yes (운영 단계 sync trigger 가능)
kubectl auth can-i patch applications -n market-dev \
    --as=ops-user@example.com --as-group=troica-operators
```

**기대**: 모두 `yes`.

### Step 2 — market-prod 도 동일

```bash
kubectl auth can-i delete pods -n market-prod \
    --as=ops-user@example.com --as-group=troica-operators
# 기대: yes
```

### Step 3 — Deny 검증 (다른 namespace + RBAC 자체 변경)

```bash
# A. monitoring namespace 의 pod 삭제 — no (Role 이 market-{dev,prod} 에만 적용)
kubectl auth can-i delete pods -n monitoring \
    --as=ops-user@example.com --as-group=troica-operators

# B. argocd namespace 의 application 변경 — no
kubectl auth can-i patch applications -n argocd \
    --as=ops-user@example.com --as-group=troica-operators

# C. ClusterRole 생성 — no (RBAC 자체 변경 차단)
kubectl auth can-i create clusterroles \
    --as=ops-user@example.com --as-group=troica-operators

# D. namespace 생성 — no
kubectl auth can-i create namespaces \
    --as=ops-user@example.com --as-group=troica-operators

# E. node 조회 — no (cluster-wide 자원)
kubectl auth can-i get nodes \
    --as=ops-user@example.com --as-group=troica-operators
```

**기대**: 모두 `no`.

---

## Part 4 — SRE Role 검증 (cluster-admin)

```bash
# A. 모든 verb / resource — yes
kubectl auth can-i '*' '*' --all-namespaces \
    --as=sre-user@example.com --as-group=troica-sre

# B. RBAC 자체 변경 — yes (운영 단계 권한 부여)
kubectl auth can-i create clusterroles \
    --as=sre-user@example.com --as-group=troica-sre

# C. node drain — yes
kubectl auth can-i patch nodes \
    --as=sre-user@example.com --as-group=troica-sre

# D. Secret 조회 — yes
kubectl auth can-i get secrets -n market-dev \
    --as=sre-user@example.com --as-group=troica-sre
```

**기대**: 모두 `yes`.

---

## Part 5 — ServiceAccount RBAC 자동 검증 (심화 (3)-2 일부)

service Helm chart 의 ServiceAccount + Role (R-25/R-33 의 R-48 일부) 도 자동 검증:

```bash
# 6 service 의 ServiceAccount 등록
kubectl -n market-dev get serviceaccount

# api-gateway ServiceAccount 의 권한 (chart 의 templates/role.yaml)
kubectl auth can-i get configmaps -n market-dev \
    --as=system:serviceaccount:market-dev:api-gateway
```

**기대**: ServiceAccount 가 자기 namespace 의 configmap / secret 등 일부 자원 접근 가능.

---

## 평가 발표 포인트 정리

| 평가요소 | 시연 |
|---|---|
| 심화 (3)-1 사람 RBAC | 3 역할 (developer/operator/sre) — Part 2~4 의 Allow + Deny matrix |
| 심화 (3)-2 ServiceAccount RBAC | Part 5 — chart 의 templates/role.yaml + RoleBinding |
| 최소 권한 원칙 | operator 가 RBAC 자체 변경 못 함 (Part 3 Step 3) — 권한 격리 |
| namespace 격리 | operator 가 monitoring/argocd 자원 못 건드림 (Part 3 Step 3) |

---

## Troubleshooting

| 증상 | 가능한 원인 |
|---|---|
| 모든 명령이 `yes` (deny 검증도 yes) | 매니페스트 미sync 또는 group 이름 mismatch — `kubectl get clusterrolebinding troica-developer-binding -o yaml` 로 subjects 확인 |
| `kubectl auth can-i` 가 error | impersonation 권한 부재 — master 의 admin.conf 사용 (system:masters 그룹) |
| Group 이름 typo | 매니페스트의 정확한 이름: `troica-developers` (s 끝, developer 의 복수형) / `troica-operators` / `troica-sre` (sre 는 단수) |
| ServiceAccount 권한 fail | service Helm chart 의 templates/role.yaml 미작성 — 별도 확인 |

---

## 한 줄 요약 명령어 (발표 demo 1 분)

```bash
# 3 role × allow / deny × 2 = 12 명령. 모두 yes/no 한 줄 출력 — 시각적 명확
echo "=== developer ===" ; \
kubectl auth can-i list pods -A --as=d --as-group=troica-developers ; \
kubectl auth can-i delete pods -n market-dev --as=d --as-group=troica-developers ; \
echo "=== operator ===" ; \
kubectl auth can-i delete pods -n market-dev --as=o --as-group=troica-operators ; \
kubectl auth can-i delete pods -n monitoring --as=o --as-group=troica-operators ; \
echo "=== sre ===" ; \
kubectl auth can-i '*' '*' --all-namespaces --as=s --as-group=troica-sre
```

**기대 출력**:
```
=== developer ===
yes
no
=== operator ===
yes
no
=== sre ===
yes
```

---

## 정적/논리/실행/실패 흐름 검증

| 단계 | 결과 |
|---|---|
| 정적 | 3 role 매니페스트 RBAC 표준 ✅ (이미 머지) |
| 논리 | impersonation = master admin kubeconfig 의 system:masters 권한 사용 |
| 실행 | `kubectl auth can-i --as --as-group` → API server 의 RBAC authorizer 호출 → 매칭/거부 |
| 실패 | (a) 매니페스트 미sync (Allow 명령도 no) (b) group 이름 typo (모든 명령 no) (c) impersonation 권한 X (error) — 모두 troubleshooting 명시 |

---

## Reference

- [ClusterRole / RoleBinding docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [kubectl impersonation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)
- [troica-developer 매니페스트](../../platform/05-rbac/manifests/troica-developer.yaml)
- [troica-operator 매니페스트](../../platform/05-rbac/manifests/troica-operator.yaml)
- [troica-sre 매니페스트](../../platform/05-rbac/manifests/troica-sre.yaml)

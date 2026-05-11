# Applications values

ApplicationSet(`../appset.yaml`)이 본 디렉토리 하위 각 서비스 폴더를 스캔해 환경(dev/prod)별 Application을 자동 생성한다.

## 컨벤션

| 디렉토리 | ApplicationSet 동작 |
|---------|---------------------|
| `<service-name>/` | `<service-name>-dev`, `<service-name>-prod` Application 자동 생성 |
| `_template/` 등 underscore prefix | exclude — 무시됨 |

## 새 서비스 추가

```bash
cp -R applications/values/_template applications/values/<service-name>
```

이후 `<service-name>/values.yaml`의 `nameOverride`, `image.repository`, `envFrom` 이름을 수정한다.

## 로컬 helm template 검증

```bash
helm template applications/charts/microservice \
  -f applications/values/_template/values.yaml \
  -f applications/values/_template/values-dev.yaml
```

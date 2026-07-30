# Day 7 — Helm (패키지 매니저)

> Kubernetes v1.36 기준 · Helm의 필요성과 핵심 개념을 이해하고, 공개 차트 활용부터
> 나만의 차트 작성, 그리고 배운 리소스 전체를 하나의 차트로 묶어 배포합니다.

---

## 학습 목표

- Helm이 필요한 이유와 **chart · release · repository** 개념을 이해하고 Helm을 설치할 수 있다.
- 공개 차트를 설치·업그레이드·롤백하고, `values` 오버라이드로 배포를 커스터마이징할 수 있다.
- 차트 구조(template/values 분리)를 이해하고, 환경별(dev/prod) 구성을 가진 **나만의 차트**를 작성할 수 있다.
- Deployment + Service + Ingress + ConfigMap/Secret + PVC를 **하나의 Helm 차트**로 묶어 배포할 수 있다.

---

## 목차

1. [Helm 개요 ① 패키지 매니저가 왜 필요한가](#1-helm-개요-①-패키지-매니저가-왜-필요한가)
2. [Helm 개요 ② Chart · Release · Repository](#2-helm-개요-②-chart--release--repository)
3. [Helm 설치](#3-helm-설치)
4. [Helm 활용 ① Chart 설치 · 업그레이드 · 롤백](#4-helm-활용-①-chart-설치--업그레이드--롤백)
5. [Helm 활용 ② values 오버라이드](#5-helm-활용-②-values-오버라이드)
6. [[실습 1] 공개 Chart 사용](#실습-1-공개-chart-사용)
7. [나만의 Chart 만들기 ① 구조](#7-나만의-chart-만들기-①-구조)
8. [나만의 Chart 만들기 ② template · values 분리](#8-나만의-chart-만들기-②-template--values-분리)
9. [나만의 Chart 만들기 ③ 환경별(dev/prod) 구성](#9-나만의-chart-만들기-③-환경별devprod-구성)
10. [[실습 2] 첫 차트 만들고 배포하기](#실습-2-첫-차트-만들고-배포하기)
11. [종합 실습 — 전체 스택을 하나의 Chart로](#11-종합-실습--전체-스택을-하나의-chart로)
12. [[실습 3] 종합 배포 — Deployment+Service+Ingress+ConfigMap/Secret+PVC](#실습-3-종합-배포--deploymentserviceingressconfigmapsecretpvc)
13. [7일차 종합 리뷰 및 정리](#13-7일차-종합-리뷰-및-정리)

---

## 1. Helm 개요 ① 패키지 매니저가 왜 필요한가

### 1.1 YAML 직접 관리의 한계

Day 4~6에서 만든 리소스를 떠올려 보면, 하나의 애플리케이션을 배포하는 데도
Deployment, Service, Ingress, ConfigMap, Secret, PVC 등 **여러 YAML 파일**이 필요했습니다.

| 문제 | 설명 |
|---|---|
| 파일이 흩어짐 | 앱 하나당 5~10개 YAML을 따로 관리 |
| 환경별 중복 | dev/staging/prod마다 거의 같은 YAML을 복사·수정 |
| 버전 관리 어려움 | "지난 배포로 되돌리기"를 위한 표준 방법이 없음 |
| 재사용 어려움 | 같은 패턴(웹+DB)을 다른 프로젝트에 쓰려면 처음부터 재작성 |
| 배포 단위가 불명확 | "이 앱의 리소스 전체"를 하나로 묶어 관리·삭제하기 어려움 |

### 1.2 Helm = Kubernetes의 패키지 매니저

`apt`(Linux), `npm`(Node.js), `brew`(macOS)처럼, **Helm은 쿠버네티스 애플리케이션을
패키징·배포·버전 관리**하는 표준 도구입니다.

```
                apt install nginx      (OS 패키지)
                npm install express    (Node 패키지)
                helm install myapp ... (K8s 애플리케이션 패키지)
```

- 여러 YAML을 **하나의 패키지(chart)**로 묶는다.
- 환경별 값만 바꿔 **같은 템플릿을 재사용**한다 (values).
- 배포 이력을 저장해 **한 줄로 롤백**할 수 있다 (release revision).
- Kubernetes 공식 CNCF 프로젝트이며, 사실상 업계 표준입니다.

---

## 2. Helm 개요 ② Chart · Release · Repository

| 개념 | 비유 | 설명 |
|---|---|---|
| **Chart** | 설치 프로그램(설계도) | 템플릿화된 K8s 매니페스트 묶음 + 기본값(values.yaml). 배포 가능한 "패키지" 단위 |
| **Release** | 설치된 앱 인스턴스 | 클러스터에 실제로 설치된 Chart의 한 개체. 같은 Chart를 다른 이름/네임스페이스로 여러 번 설치 가능 |
| **Repository** | 앱 스토어 | Chart를 모아 배포하는 곳 (HTTP 서버). `helm repo add` 로 등록 |
| **Values** | 설치 옵션 | Chart의 기본값을 덮어써서 환경에 맞게 커스터마이징하는 입력 |
| **Revision** | 설치 버전 이력 | Release가 upgrade될 때마다 증가하는 번호 — 롤백의 기준 |

```
Repository (예: bitnami)
   └── Chart: nginx (버전 15.x)
          │  helm install my-nginx bitnami/nginx --values my-values.yaml
          ▼
        Release: my-nginx  (namespace: web)
          ├─ Revision 1 (최초 설치)
          ├─ Revision 2 (helm upgrade 후)
          └─ Revision 3 (helm upgrade 후) ← 현재
```

> 지난 과정에서 이미 Helm을 사용했습니다 — kube-prometheus-stack, ingress-nginx, cert-manager
> 모두 **공개 Chart**를 `helm install` 로 설치한 것입니다.

---

## 3. Helm 설치

```bash
# 스크립트 설치 (Linux/macOS)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# 설치 확인
helm version
```

| 환경 | 설치 방법 |
|---|---|
| macOS | `brew install helm` |
| Windows | `choco install kubernetes-helm` 또는 `winget install Helm.Helm` |
| Linux | 위 스크립트 또는 배포판 패키지 매니저 |

```bash
# 자동완성 (선택)
helm completion bash > /etc/bash_completion.d/helm   # 셸에 맞게 조정
```

> Helm 3부터는 별도 서버 컴포넌트(Tiller) 없이 **클라이언트만으로 동작**합니다 —
> Helm 2와의 큰 차이점이자 보안이 개선된 지점입니다.

---

## 4. Helm 활용 ① Chart 설치 · 업그레이드 · 롤백

```bash
# 리포지토리 추가·검색
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx

# 설치 (release 이름: my-nginx)
helm install my-nginx bitnami/nginx --namespace web --create-namespace

# 설치된 release 목록
helm list -n web
helm status my-nginx -n web

# 업그레이드 (예: replicaCount 변경)
helm upgrade my-nginx bitnami/nginx -n web --set replicaCount=3

# 이력 확인
helm history my-nginx -n web

# 롤백 (직전 리비전으로, 또는 특정 리비전 지정)
helm rollback my-nginx -n web
helm rollback my-nginx 1 -n web

# 삭제
helm uninstall my-nginx -n web
```

| 명령 | kubectl 대응 개념 |
|---|---|
| `helm install` | 여러 `kubectl apply` 를 한 번에 |
| `helm upgrade` | 변경분만 반영하는 `kubectl apply` + 히스토리 기록 |
| `helm rollback` | 이전 리비전의 매니페스트로 재적용 (Deployment의 `rollout undo`와 유사한 개념을 차트 전체에 적용) |
| `helm uninstall` | release가 만든 리소스 전체를 한 번에 `kubectl delete` |

> **install --dry-run** 으로 실제 적용 없이 렌더링 결과만 미리 볼 수 있습니다.
> ```bash
> helm install my-nginx bitnami/nginx --dry-run --debug
> ```

---

## 5. Helm 활용 ② values 오버라이드

Chart는 `values.yaml` 이라는 **기본값 세트**를 갖고 있고, 설치 시 이를 덮어써서 커스터마이징합니다.

### 5.1 오버라이드 방법 3가지 (우선순위: 아래로 갈수록 높음)

```bash
# ① Chart 기본 values.yaml (가장 낮은 우선순위)
helm show values bitnami/nginx > default-values.yaml

# ② --set 으로 개별 값 지정 (커맨드라인)
helm install my-nginx bitnami/nginx --set replicaCount=3 --set service.type=NodePort

# ③ -f/--values 로 파일 지정 (가장 흔히 사용, 여러 개 지정 가능)
helm install my-nginx bitnami/nginx -f my-values.yaml

# 여러 -f 를 함께 쓰면 뒤에 오는 파일이 앞의 값을 덮어씀
helm install my-nginx bitnami/nginx -f base-values.yaml -f prod-values.yaml
```

`my-values.yaml` 예시:

```yaml
replicaCount: 3
service:
  type: LoadBalancer
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 500m, memory: 256Mi }
```

### 5.2 현재 적용된 값 확인

```bash
helm get values my-nginx -n web            # 사용자가 지정한 값만
helm get values my-nginx -n web -a         # 최종 병합된 전체 값 (all)
helm get manifest my-nginx -n web          # 실제 렌더링된 YAML 전체
```

> **우선순위 요약**: chart 기본값 < `-f` 파일(먼저 지정한 것) < `-f` 파일(나중 지정한 것) < `--set`
> — `--set` 이 가장 강하게 덮어씁니다.

---

## [실습 1] 공개 Chart 사용

**목표**: 공개 Chart(ingress-nginx)를 설치·values 커스터마이징·업그레이드·롤백해 본다.

```bash
# 1) 설치
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-ingress ingress-nginx/ingress-nginx -n ingress-demo --create-namespace

kubectl get pods -n ingress-demo
helm list -n ingress-demo

# 2) values 커스터마이징하여 업그레이드 (replicaCount 2로, NodePort로 전환)
cat > ingress-values.yaml << 'EOF'
controller:
  replicaCount: 2
  service:
    type: NodePort
EOF

helm upgrade my-ingress ingress-nginx/ingress-nginx -n ingress-demo -f ingress-values.yaml
kubectl get pods -n ingress-demo -w        # 2개로 늘어나는 과정 관찰

# 3) 이력 확인 후 롤백
helm history my-ingress -n ingress-demo
helm rollback my-ingress 1 -n ingress-demo
kubectl get pods -n ingress-demo           # 1개로 돌아옴

# 4) 정리
helm uninstall my-ingress -n ingress-demo
kubectl delete namespace ingress-demo
```

> ✔ `helm upgrade` 후 replica 수가 2로 늘어나는 것을 확인했다.
> ✔ `helm rollback` 후 리비전 1(replica 1개)로 되돌아갔다.
> ✔ (질문) `helm rollback` 은 클러스터에서 실제로 어떤 kubectl 동작에 해당할까?

---

## 7. 나만의 Chart 만들기 ① 구조

```bash
helm create mychart
```

```
mychart/
├── Chart.yaml           # 차트 메타데이터 (이름, 버전, 설명)
├── values.yaml           # 기본 설정값
├── charts/               # 의존 차트(서브차트)를 담는 디렉토리
├── templates/             # 실제 K8s 매니페스트 템플릿
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl       # 재사용 템플릿 함수(이름 규칙 등)
│   ├── NOTES.txt          # 설치 후 안내 메시지
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

`Chart.yaml` 핵심 필드:

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application
version: 0.1.0        # Chart 자체의 버전
appVersion: "1.0.0"    # 담고 있는 애플리케이션의 버전
```

> `version` 과 `appVersion` 은 별개입니다. Chart 구조를 바꾸면 `version`을,
> 애플리케이션 이미지를 올리면 `appVersion`을 올리는 것이 관례입니다.

---

## 8. 나만의 Chart 만들기 ② template · values 분리

**핵심 원칙**: 템플릿(구조)과 값(내용)을 분리해, 같은 템플릿을 여러 값으로 재사용합니다.

### 8.1 values.yaml (값)

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.27"
service:
  type: ClusterIP
  port: 80
```

### 8.2 templates/deployment.yaml (템플릿 — Go template 문법)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

### 8.3 템플릿 문법 핵심

| 문법 | 의미 |
|---|---|
| `{{ .Values.xxx }}` | values.yaml의 값 참조 |
| `{{ .Release.Name }}` | 설치 시 지정한 release 이름 |
| `{{ .Chart.Name }}` | Chart.yaml의 name |
| `{{ if .Values.x }} ... {{ end }}` | 조건부 렌더링 |
| `{{ range .Values.list }} ... {{ end }}` | 반복 렌더링 |
| `{{ include "mychart.fullname" . }}` | `_helpers.tpl`에 정의한 재사용 템플릿 호출 |

```bash
# 렌더링 결과 미리보기 (클러스터에 적용하지 않음) — 문법 오류·값 확인에 필수
helm template myrelease ./mychart

# lint로 차트 문법 검증
helm lint ./mychart
```

---

## 9. 나만의 Chart 만들기 ③ 환경별(dev/prod) 구성

같은 Chart를 환경별로 다른 값만 바꿔 배포합니다 — **템플릿은 하나, values는 여러 개**.

```
mychart/
├── values.yaml          # 공통 기본값
├── values-dev.yaml       # dev 전용 오버라이드
└── values-prod.yaml      # prod 전용 오버라이드
```

`values-dev.yaml`:

```yaml
replicaCount: 1
image: { tag: "dev-latest" }
ingress:
  enabled: true
  host: dev.example.local
resources:
  requests: { cpu: 50m, memory: 64Mi }
```

`values-prod.yaml`:

```yaml
replicaCount: 3
image: { tag: "1.27" }
ingress:
  enabled: true
  host: app.example.com
  tls: true
resources:
  requests: { cpu: 200m, memory: 256Mi }
  limits:   { cpu: 1,    memory: 512Mi }
```

```bash
# 환경별 배포 — 같은 차트, 다른 values, 다른 release/namespace
helm install myapp-dev  ./mychart -f mychart/values.yaml -f mychart/values-dev.yaml  -n dev  --create-namespace
helm install myapp-prod ./mychart -f mychart/values.yaml -f mychart/values-prod.yaml -n prod --create-namespace
```

> 이 패턴이 "Dockerfile 하나로 이미지는 한 번 빌드하고, 환경별 설정은 ConfigMap/values로 분리한다"는
> 12-Factor App 원칙과 정확히 같은 철학입니다.

---

## [실습 2] 첫 차트 만들고 배포하기

**목표**: `helm create` 로 시작해 간단한 웹 서버 Chart를 dev/prod 값으로 각각 배포한다.

```bash
helm create mychart
cd mychart

# templates/ 기본 파일 중 이번 실습에 불필요한 것 정리(선택)
rm -f templates/tests/test-connection.yaml

# values.yaml 을 6장 예시로 교체, values-dev.yaml/values-prod.yaml 를 9장 예시로 작성

# 렌더링 확인 (적용 전 검증)
helm template dev-test . -f values.yaml -f values-dev.yaml
helm lint .

# 실제 설치
helm install myapp-dev . -f values.yaml -f values-dev.yaml -n dev --create-namespace
kubectl get deploy,svc -n dev

helm install myapp-prod . -f values.yaml -f values-prod.yaml -n prod --create-namespace
kubectl get deploy,svc -n prod
```

```bash
# replica 수 등 환경별 차이 비교
kubectl get deploy -n dev  -o jsonpath='{.items[0].spec.replicas}'; echo
kubectl get deploy -n prod -o jsonpath='{.items[0].spec.replicas}'; echo
```

> ✔ `helm template` 결과에서 `{{ .Values... }}` 자리에 실제 값이 채워져 있다.
> ✔ dev는 replica 1, prod는 replica 3으로 **같은 템플릿이 다르게 렌더링**되었다.
> ✔ (질문) values.yaml에는 있지만 values-dev.yaml에는 없는 키는 어떤 값이 적용되는가?

---

## 11. 종합 실습 — 전체 스택을 하나의 Chart로

Day 4~6에서 배운 리소스를 모두 Helm 템플릿으로 통합합니다.

```
mychart/templates/
├── deployment.yaml     # 앱 컨테이너, ConfigMap/Secret 참조, PVC 마운트
├── service.yaml        # ClusterIP
├── ingress.yaml        # 호스트 라우팅 (values.ingress.enabled 로 on/off)
├── configmap.yaml       # 애플리케이션 설정
├── secret.yaml          # 민감정보 (values로 직접 노출하지 않는 것을 권장 - 13장 참고)
└── pvc.yaml             # 데이터 영속화
```

`values.yaml` 설계 예시:

```yaml
replicaCount: 2
image: { repository: myapp, tag: "1.0" }

config:
  APP_MODE: production

secret:
  dbPassword: ""            # 실제 값은 --set 또는 별도 관리 (14장 참고)

persistence:
  enabled: true
  size: 1Gi
  accessMode: ReadWriteOnce

ingress:
  enabled: true
  host: myapp.example.local
  path: /

service:
  port: 80
  targetPort: 8080
```

`templates/ingress.yaml`의 조건부 렌더링 예시:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-svc
                port: { number: {{ .Values.service.port }} }
{{- end }}
```

> `ingress.enabled: false` 인 환경(예: 내부 전용 dev)에서는 Ingress 리소스 자체가 생성되지 않습니다 —
> 조건부 템플릿으로 **환경별 리소스 유무**까지 제어할 수 있습니다.

---

## [실습 3] 종합 배포 — Deployment+Service+Ingress+ConfigMap/Secret+PVC

**목표**: 지금까지 배운 6개 리소스 유형을 하나의 Release로 배포·확인·정리한다.

### 3-1. Chart 완성

11장 구조를 기반으로 `templates/*.yaml` 6개 파일과 `values.yaml`을 작성합니다.
(Secret 값은 실습 편의상 `--set` 으로 주입 — 실제 값을 Git에 커밋하지 않기 위함)

### 3-2. 설치

```bash
helm install fullstack ./mychart \
  -n fullstack --create-namespace \
  --set secret.dbPassword='S3cure!pw'

kubectl get all,ingress,configmap,secret,pvc -n fullstack
```

### 3-3. 리소스 전체 확인

```bash
kubectl get deployment -n fullstack
kubectl get svc -n fullstack
kubectl get ingress -n fullstack
kubectl get configmap -n fullstack
kubectl get secret -n fullstack
kubectl get pvc -n fullstack

# release가 만든 리소스만 모아보기 (label 기반)
kubectl get all -n fullstack -l app.kubernetes.io/instance=fullstack
```

### 3-4. 업그레이드 시나리오

```bash
# 이미지 태그 변경 + replica 확장을 한 번에
helm upgrade fullstack ./mychart -n fullstack \
  --set image.tag=1.1 --set replicaCount=3 \
  --reuse-values                      # 기존 values(예: dbPassword)는 유지

kubectl rollout status deployment/fullstack-mychart -n fullstack
helm history fullstack -n fullstack
```

### 3-5. 정리

```bash
helm uninstall fullstack -n fullstack
kubectl get all -n fullstack             # 리소스가 모두 사라졌는지 확인
kubectl delete namespace fullstack
```

> ✔ 6개 리소스 유형이 **하나의 `helm install` 명령**으로 동시에 생성되었다.
> ✔ `helm uninstall` 한 번으로 PVC를 제외한(또는 포함한, chart 설계에 따라) 모든 리소스가 정리되었다.
> ✔ (질문) PVC까지 함께 삭제되길 원한다면 template에 어떤 장치가 필요할까? (힌트: `helm.sh/resource-policy` 어노테이션 여부)
> ✔ (질문) Deployment 개별로 `kubectl apply` 하던 지난 실습들과 비교했을 때, Helm이 가장 크게 줄여준 작업은 무엇인가?

---

## 13. 7일차 종합 리뷰 및 정리

### 13.1 오늘 배운 것 요약

| 주제 | 핵심 문장 |
|---|---|
| Helm 필요성 | 흩어진 YAML·환경별 중복·버전 관리 문제를 패키지 매니저 모델로 해결 |
| Chart/Release/Repo | Chart=설계도, Release=설치된 인스턴스, Repository=Chart 저장소 |
| 설치·업그레이드·롤백 | `install` → `upgrade` → `rollback`, 모두 리비전으로 관리 |
| values 오버라이드 | 우선순위: 기본값 < `-f`(순서대로) < `--set` |
| Chart 구조 | Chart.yaml(메타) / values.yaml(값) / templates/(구조) 분리 |
| 환경별 구성 | 템플릿 하나 + values 여러 개 = dev/prod 재사용 |
| 종합 배포 | Deployment+Service+Ingress+ConfigMap/Secret+PVC를 한 Release로 관리 |

### 13.2 Day 1~7 전체 여정 되짚기

```
[Day 1-2] 컨테이너·이미지, Dockerfile 빌드
[Day 3]   네트워크·스토리지, 레지스트리, Compose
[Day 4]   컨트롤 플레인 6대 컴포넌트, Deployment(롤링·롤백·스케일), Service
[Day 5]   Ingress(라우팅·TLS), ConfigMap/Secret, Volume·PV/PVC
[Day 6]   PV 심화(reclaim policy), StatefulSet(DB, 순서·식별자·partition)
[Day 7]   Helm — 위 모든 리소스를 하나의 패키지로 묶어 재사용 가능하게
```

- Day 1~6에서 각 리소스를 **개별적으로 이해**했다면, Day 7의 Helm은 그것들을
  **하나의 배포 단위**로 조립하는 방법이었습니다.
- 실무에서는 처음부터 YAML을 손으로 쓰기보다, **공개 Chart를 설치하며 구조를 배우고,
  필요한 부분만 커스터마이징**하는 흐름이 일반적입니다.

### 13.3 셀프 체크 (복습 질문)

1. Chart와 Release의 차이를 한 문장으로 설명하라.
2. `--set` 과 `-f values.yaml` 을 동시에 쓰면 어떤 것이 우선하는가?
3. `helm template` 과 `helm install` 의 차이는 무엇인가?
4. `values.yaml` 의 값을 템플릿에서 참조할 때 사용하는 문법은?
5. dev/prod 환경을 같은 Chart로 관리할 때, 무엇을 공유하고 무엇을 다르게 하는가?
6. `helm uninstall` 이후에도 남아있을 수 있는 리소스는 무엇이며 왜 그런가?

### 13.4 과제 (선택)

- [실습 3]의 Chart에 `_helpers.tpl` 을 활용해 리소스 이름 규칙을 함수화해 보기
- `helm package ./mychart` 로 `.tgz` 패키지를 만들고, 로컬 리포지토리(`helm repo index`)로 배포해 보기
- Bitnami의 PostgreSQL Chart(`bitnami/postgresql`)를 values 오버라이드로 설치해,
  Day 6에서 직접 만든 StatefulSet과 어떤 부분이 대응되는지 비교해 보기



---


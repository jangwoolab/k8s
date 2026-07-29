# Day 5 — Ingress · ConfigMap/Secret · Volume · 리뷰

> Kubernetes v1.36 기준 · HTTP 라우팅(Ingress), 설정과 민감정보 분리(ConfigMap/Secret),
> 데이터 영속화(Volume·PV·PVC)를 학습하고 과정 전체를 리뷰합니다.

---

## 학습 목표

- Ingress의 개념과 **Ingress 리소스 / Ingress Controller의 분리 구조**를 이해하고, 호스트·경로 기반 라우팅과 TLS 구성을 설명할 수 있다.
- ConfigMap과 Secret으로 **설정·민감정보를 이미지에서 분리**하고, 환경변수·볼륨으로 주입할 수 있다. (base64 ≠ 암호화 이해)
- 컨테이너의 임시성을 이해하고, emptyDir·hostPath와 **PV·PVC·StorageClass 동적 프로비저닝**으로 데이터를 영속화할 수 있다.

---

## 목차

1. [Ingress 개념](#1-ingress-개념)
2. [Ingress Controller(nginx) 설치](#2-ingress-controllernginx-설치)
3. [호스트 / 경로 기반 라우팅](#3-호스트--경로-기반-라우팅)
4. [Ingress TLS + cert-manager 소개](#4-ingress-tls--cert-manager-소개)
5. [[실습 1] 여러 서비스 도메인·경로 라우팅](#실습-1-여러-서비스-도메인경로-라우팅)
6. [ConfigMap](#6-configmap)
7. [Secret](#7-secret)
8. [[실습 2] 설정 분리](#실습-2-설정-분리)
9. [Volume 기초 — 임시성 · emptyDir · hostPath](#9-volume-기초--임시성--emptydir--hostpath)
10. [PV · PVC · StorageClass](#10-pv--pvc--storageclass)
11. [[실습 3] PVC 마운트](#실습-3-pvc-마운트)
12. [5일차 리뷰 및 정리](#12-5일차-리뷰-및-정리)

---

## 1. Ingress 개념

### 1.1 왜 Ingress인가

Day 4의 Service만으로는 부족한 지점이 있습니다.

| 요구 | Service(L4)의 한계 | Ingress(L7)의 해법 |
|---|---|---|
| 서비스마다 외부 노출 | NodePort/LB를 서비스 수만큼 생성 (포트·비용 증가) | **진입점 1개**로 여러 서비스 라우팅 |
| 도메인별 분기 | 불가 (IP:포트 수준) | `shop.example.com` / `blog.example.com` 호스트 라우팅 |
| 경로별 분기 | 불가 | `/api` → api-svc, `/` → web-svc 경로 라우팅 |
| HTTPS 종료 | 앱이 직접 처리 | Ingress에서 **TLS 종료** 후 내부는 HTTP |

---
쿠버네티스에서 인그레스(Ingress)를 사용하는 주요 이유는 **클러스터 외부에서 내부 서비스(Service)로 들어오는 HTTP/HTTPS 트래픽을 효율적이고 안전하게 관리하기 위함**입니다.

인그레스를 사용하지 않을 때 발생하는 문제점과 비교하면 인그레스의 필요성이 훨씬 명확해집니다.

---

## 1. 인그레스가 필요한 주요 이유

### 1) 비용 절감 (단일 엔드포인트 / L7 로드밸런싱)

* **인그레스 미사용 시:** 외부 트래픽을 받기 위해 서비스마다 `type: LoadBalancer`를 생성해야 합니다. 이 경우 클라우드 업체(AWS, GCP, Azure 등)에서 서비스 개수만큼 실제 로드밸런서를 생성하므로 **비용이 비례해서 증가**합니다.
* **인그레스 사용 시:** 단 하나의 인그레스(및 인그레스 콘트롤러)만 생성하고, L7(애플리케이션) 레이어에서 URL 경로(Path)나 호스트 이름(Host)을 기준으로 여러 내부 서비스로 트래픽을 분기합니다. 로드밸런서를 하나만 띄우므로 비용을 크게 아낄 수 있습니다.

### 2) 도메인 및 경로 기반 라우팅 (URL 라우팅)

하나의 IP 주소(또는 도메인)로 들어오는 요청을 조건에 따라 서로 다른 파드로 연결할 수 있습니다.

* **호스트(Host) 기반 라우팅:**
* `api.example.com` → `api-service`로 전달
* `web.example.com` → `web-service`로 전달


* **경로(Path) 기반 라우팅:**
* `[example.com/users](https://example.com/users)` → `user-service`로 전달
* `[example.com/orders](https://example.com/orders)` → `order-service`로 전달



### 3) TLS/SSL 암호화 (HTTPS) 중앙 집중 관리

* 개별 애플리케이션 파드마다 SSL/TLS 인증서를 설치하고 관리할 필요 없이, 인그레스 단에서 HTTPS 인증서를 일괄 적용(SSL Termination)할 수 있습니다.
* 외부 ↔ 인그레스 구간은 암호화(HTTPS)로 통신하고, 인그레스 ↔ 내부 파드 구간은 HTTP로 통신하여 개별 애플리케이션의 암호화 연산 부담을 줄여줍니다.

### 4) 고급 트래픽 제어 기능 제공

인그레스 콘트롤러(예: NGINX Ingress Controller, Traefik, Istio 등)를 활용하면 아래와 같은 수준 높은 라우팅 기능을 설정 하나로 적용할 수 있습니다.

* **카나리 배포(Canary Deployment):** 트래픽의 10%만 신규 버전 서비스로 보내는 등 비율 기반 배포 가능
* **CORS 설정 및 헤더 조작:** 클라이언트 요청/응답 헤더 재작성
* **Rate Limiting:** 특정 IP의 과도한 요청 차단

---

## 2. 한눈에 보는 비교 (NodePort / LoadBalancer vs Ingress)

| 구분 | NodePort | LoadBalancer | Ingress |
| --- | --- | --- | --- |
| **작동 계층** | L4 (전송 계층) | L4 (전송 계층) | **L7 (애플리케이션 계층)** |
| **접속 방식** | `IP:30000~32767` 포트 지정 | 서비스별 외부 IP 할당 | **단일 IP / 도메인 / URL 경로** |
| **비용/효율** | 개발/테스트용 (포트 관리 어려움) | 서비스당 로드밸런서 생성 (비용 증가) | **하나의 로드밸런서로 여러 서비스 공유 (비용 절감)** |
| **주요 기능** | 단순 포트 포워딩 | 단순 L4 로드밸런싱 | **URL 라우팅, SSL/TLS 종단, 헤더 제어 등** |

---

> 💡 **주의할 점**
> 인그레스 리소스(YAML)를 작성하는 것만으로는 작동하지 않으며, 이를 실제로 처리해 줄 **인그레스 콘트롤러(Ingress Controller)**(예: NGINX Ingress Controller, ALB Ingress Controller 등)가 클러스터에 설치되어 있어야 합니다.
---

### 1.2 구조 — 리소스와 컨트롤러의 분리

```
                    ┌──────────────── 클러스터 ────────────────────┐
 사용자 ──HTTP(S)──▶│  Ingress Controller (nginx Pod)              │
 (80/443)           │   │  ▲                                       │
                    │   │  └─ watch: Ingress 리소스(규칙)           │
                    │   ├── host: shop.example.com ──▶ shop-svc ─▶ Pods
                    │   ├── path: /api            ──▶ api-svc  ─▶ Pods
                    │   └── path: /               ──▶ web-svc  ─▶ Pods
                    └──────────────────────────────────────────────┘
```

- **Ingress 리소스** = 라우팅 "규칙" 선언 (YAML). 그 자체로는 아무것도 안 함.
- **Ingress Controller** = 규칙을 읽어 실제로 트래픽을 처리하는 리버스 프록시(nginx 등).
- 컨트롤러가 없으면 Ingress 리소스는 **무시됩니다** — 반드시 별도 설치가 필요합니다.
- 참고: 장기적으로는 후속 표준인 **Gateway API**로의 전환이 진행 중입니다(개념은 동일 계열).

---

## 2. Ingress Controller(nginx) 설치

### Helm 설치를 위한 준비 
```bash
sudo dnf install -y tar git openssl
```

### Helm 설치
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

```

### Helm 설치 확인 
```bash
helm

helm version
```


### **ingress-nginx** 를 Helm으로 설치
  - 가장 널리 쓰이는 방식

```bash
# 인그레스 레파지토리 등록
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# 헬름 업데이트
helm repo update

# 헬름을 사용해서 인그레스 설치 [기본 : LoadBalancer]
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace --namespace ingress-nginx

# 설치 확인
kubectl get pods -n ingress-nginx
kubectl get svc  -n ingress-nginx   # controller 서비스 유형 확인

# ingress-nginx-controller  LoadBalancer의 EXTERNAL-IP 확인
# http://10.10.10.50     으로 접속시 404 Not Found [nginx] 출력되면 설치 성공 
```

| 환경 | controller Service 접근 방법 |
|---|---|
| 클라우드 | `LoadBalancer` — EXTERNAL-IP 발급 대기 후 사용 |
| 온프렘/실습(kubeadm 등) | `NodePort` 로 전환하거나 아래처럼 설치 시 지정 [LoadBalancer 가 설치되지 않은 환경인 경우] |
| minikube | `minikube addons enable ingress` 로 간단 설치 가능 |

## [참고] : 아래는 참고 사항임. NodePort를 사용해서 인그레스 설치 [테스트용/실습환경용] - LoadBalancer가 없는 환경인 경우 
```bash
# 실습 환경: NodePort로 설치 (80/443 → 노드 포트)
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort

kubectl get svc -n ingress-nginx ingress-nginx-controller
# 예: 80:30080/TCP, 443:30443/TCP
```

> ✔ 체크포인트 — controller Pod가 Running, 서비스에서 80/443 매핑 포트를 확인했다.

---

## 3. 호스트 / 경로 기반 라우팅

### 3.1 기본 Ingress 리소스

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # 경로 재작성(필요 시)
spec:
  ingressClassName: nginx            # 어떤 컨트롤러가 처리할지 지정
  rules:
    # ── 호스트 기반 ─────────────────────────────
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: shop-svc, port: { number: 80 } }
    # ── 경로 기반 (같은 호스트에서 분기) ──────────
    - host: www.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api-svc, port: { number: 80 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: web-svc, port: { number: 80 } }
```

### 3.2 pathType 3가지

| pathType | 매칭 방식 | 예 |
|---|---|---|
| `Prefix` | 경로 접두어 일치 (가장 일반적) | `/api` → /api, /api/v1 모두 매칭 |
| `Exact` | 정확히 일치 | `/health` 만 |
| `ImplementationSpecific` | 컨트롤러 구현에 위임 | nginx 정규식 등 |

- 여러 path가 겹치면 **더 긴(구체적인) 경로가 우선**합니다 (`/api` > `/`).
- `ingressClassName` 으로 컨트롤러를 명시하는 것이 v1 표준입니다.

---

## 4. Ingress TLS + cert-manager 소개

### 4.1 Ingress에서 TLS 종료

인증서(+키)를 **kubernetes.io/tls 타입 Secret** 으로 만들고 Ingress에 연결합니다.

```bash
# (실습용) 자체 서명 인증서 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=shop.example.com"

kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key
```

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts: [shop.example.com]
      secretName: shop-tls          # TLS Secret 참조
  rules:
    - host: shop.example.com
      ...
```

→ 443으로 들어온 트래픽을 Ingress가 복호화(TLS 종료)하고, 내부는 HTTP로 전달합니다.

### 4.2 cert-manager 소개 (자동 발급·갱신)

수동 인증서 관리의 한계(만료·갱신)를 해결하는 표준 도구입니다.

- **cert-manager**: 인증서를 쿠버네티스 리소스(Certificate)로 선언하면 자동 발급·갱신
- **Issuer / ClusterIssuer**: 발급처 정의 — Let's Encrypt(ACME), 사설 CA 등
- Ingress에 어노테이션 한 줄이면 자동으로 TLS Secret 생성·갱신:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [shop.example.com]
      secretName: shop-tls        # cert-manager가 자동 생성·갱신
```

```bash
# 설치 개요 (오늘은 소개까지 - 상세 구성은 심화 차시)
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace --set crds.enabled=true
```

> Let's Encrypt 사용 시 도메인 소유 검증(HTTP-01/DNS-01)이 필요하므로
> 공인 도메인·외부 접근이 가능한 환경에서 실습합니다.

---

## [실습 1] 여러 서비스 도메인·경로 라우팅

**목표**: 앱 2개를 배포하고, 하나의 Ingress로 호스트·경로 라우팅을 구성한다.

### 1-1. 대상 서비스 2개 준비

```bash
# 응답 내용이 구분되는 데모 이미지 사용
kubectl create deployment web --image=nginxdemos/hello:plain-text --replicas=2
kubectl expose deployment web --name=web-svc --port=80 --target-port=80

kubectl create deployment api --image=hashicorp/http-echo -- \
  /http-echo -listen=:8080 -text="API RESPONSE"
kubectl expose deployment api --name=api-svc --port=80 --target-port=8080
```

### 1-2. Ingress 작성·적용

`day5-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: web.example.local          # 호스트 기반
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: web-svc, port: { number: 80 } } }
    - host: api.example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api-svc, port: { number: 80 } } }
    - host: all.example.local          # 경로 기반 (한 호스트에서 분기)
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend: { service: { name: api-svc, port: { number: 80 } } }
          - path: /
            pathType: Prefix
            backend: { service: { name: web-svc, port: { number: 80 } } }
```

```bash
kubectl apply -f day5-ingress.yaml
kubectl get ingress demo-ingress
kubectl describe ingress demo-ingress    # 규칙·백엔드 확인
```

### 1-3. 도메인 테스트 (curl Host 헤더 / hosts 파일)

DNS 없이 테스트하는 두 가지 방법:

```bash
# 방법 A. curl --resolve (권장: hosts 수정 불필요)
NODE_IP=<노드IP>; NP=<80매핑 NodePort>     # 예: 30080
curl --resolve web.example.local:$NP:$NODE_IP http://web.example.local:$NP/
curl --resolve api.example.local:$NP:$NODE_IP http://api.example.local:$NP/
curl --resolve all.example.local:$NP:$NODE_IP http://all.example.local:$NP/api
curl --resolve all.example.local:$NP:$NODE_IP http://all.example.local:$NP/

# 방법 B. /etc/hosts 에 등록 후 브라우저 접속
# <노드IP>  web.example.local api.example.local all.example.local
```

> ✔ web.* 은 nginx hello, api.* 는 "API RESPONSE" 가 응답한다 (호스트 라우팅).
> ✔ all.*/api 와 all.*/ 가 서로 다른 서비스로 분기한다 (경로 라우팅, 긴 경로 우선).
> ✔ (질문) 규칙에 없는 호스트로 접속하면 404가 나는 이유는? (default backend)

### 1-4. 정리

```bash
kubectl delete ingress demo-ingress
kubectl delete svc web-svc api-svc
kubectl delete deployment web api
```

---

## 6. ConfigMap

**설정을 이미지에서 분리**해, 같은 이미지를 환경(개발/운영)마다 다르게 실행하기 위한 리소스입니다.

### 6.1 생성 방법 3가지

```bash
# ① 리터럴
kubectl create configmap app-config \
  --from-literal=APP_MODE=production \
  --from-literal=APP_COLOR=blue

# ② 파일 (파일명이 key, 내용이 value)
kubectl create configmap nginx-conf --from-file=nginx.conf

# ③ YAML 선언
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: production
  APP_COLOR: blue
  app.properties: |          # 파일 형태의 멀티라인 값
    greeting=hello
    retry=3
```

### 6.2 사용 방법 ① 환경변수 주입

```yaml
containers:
  - name: app
    image: busybox
    env:                                   # 개별 키 주입
      - name: APP_MODE
        valueFrom:
          configMapKeyRef: { name: app-config, key: APP_MODE }
    envFrom:                               # 전체 키 일괄 주입
      - configMapRef: { name: app-config }
```

### 6.3 사용 방법 ② 볼륨 마운트 (파일로 제공)

```yaml
containers:
  - name: app
    volumeMounts:
      - name: config-vol
        mountPath: /etc/config             # key들이 파일로 나타남
volumes:
  - name: config-vol
    configMap: { name: app-config }
```

**env vs 볼륨 비교**

| 방식 | 특징 |
|---|---|
| 환경변수 | 간단. 단, **Pod 재시작 전까지 갱신 반영 안 됨** |
| 볼륨 마운트 | 설정 "파일" 그대로 제공. ConfigMap 수정 시 **파일이 자동 갱신**(약간의 지연) — 단, 앱이 파일 변경을 다시 읽어야 함 |

---

## 7. Secret

민감정보(비밀번호·토큰·키)를 위한 리소스 — ConfigMap과 사용법은 유사하지만 취급이 다릅니다.

### 7.1 유형

| type | 용도 |
|---|---|
| `Opaque` (기본) | 임의의 키-값 (DB 비밀번호 등) |
| `kubernetes.io/tls` | TLS 인증서/키 (Ingress TLS에서 사용) |
| `kubernetes.io/dockerconfigjson` | 사설 레지스트리 인증 (imagePullSecrets) |
| `kubernetes.io/basic-auth` 등 | 특정 형식 강제용 내장 타입 |

### 7.2 생성과 주입

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=app \
  --from-literal=DB_PASSWORD='S3cure!pw'

kubectl get secret db-secret -o yaml     # data에 base64로 표시됨
```

```yaml
# env 주입
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef: { name: db-secret, key: DB_PASSWORD }
# 볼륨 주입 (파일 권한 제어 가능)
volumes:
  - name: secret-vol
    secret: { secretName: db-secret, defaultMode: 0400 }
```

### 7.3 보안 주의 — base64 ≠ 암호화

```bash
echo 'UzNjdXJlIXB3' | base64 -d     # → S3cure!pw  (누구나 복원 가능)
```

- Secret의 base64는 **인코딩일 뿐 암호화가 아닙니다.** `get secret -o yaml` 권한이 있으면 그대로 읽힙니다.
- 실무 보강책:
  - **RBAC**으로 Secret 조회 권한 최소화
  - etcd **encryption at rest** 활성화 (관리형 클러스터는 옵션 제공)
  - Git에 Secret YAML 평문 커밋 금지 → Sealed Secrets / SOPS
  - 외부 비밀 관리와 연동: **External Secrets Operator**, Vault, 클라우드 Secret Manager
- ConfigMap에는 민감정보를 절대 넣지 않습니다.

---

## [실습 2] 설정 분리

**목표**: 하드코딩된 설정을 ConfigMap/Secret으로 분리해 같은 이미지로 환경만 바꾼다.

### 2-1. ConfigMap · Secret 생성

```bash
kubectl create configmap demo-config \
  --from-literal=APP_MODE=production \
  --from-literal=GREETING="Hello from ConfigMap"

kubectl create secret generic demo-secret \
  --from-literal=DB_PASSWORD='S3cure!pw'
```

### 2-2. 주입 Pod 작성·확인

`day5-config-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env | grep -E 'APP_|GREETING|DB_'; ls -l /etc/appcfg; sleep 3600"]
      envFrom:
        - configMapRef: { name: demo-config }
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: { name: demo-secret, key: DB_PASSWORD }
      volumeMounts:
        - name: cfg
          mountPath: /etc/appcfg
  volumes:
    - name: cfg
      configMap: { name: demo-config }
```

```bash
kubectl apply -f day5-config-pod.yaml
kubectl logs config-demo          # 환경변수 · 마운트 파일 확인
kubectl exec config-demo -- cat /etc/appcfg/GREETING
```

### 2-3. 갱신 동작 비교

```bash
kubectl create configmap demo-config --from-literal=APP_MODE=staging \
  --from-literal=GREETING="Updated!" -o yaml --dry-run=client | kubectl apply -f -

# 잠시 후(수십 초) 볼륨 파일은 갱신, env는 그대로임을 확인
kubectl exec config-demo -- cat /etc/appcfg/GREETING     # Updated!
kubectl exec config-demo -- sh -c 'echo $GREETING'        # 이전 값
```

> ✔ envFrom/secretKeyRef 값이 컨테이너 안에서 보인다.
> ✔ ConfigMap 수정 시 볼륨 파일은 갱신되지만 환경변수는 Pod 재시작 전까지 유지된다.
> ✔ (질문) DB 비밀번호를 ConfigMap이 아닌 Secret에 넣어야 하는 이유를 base64 관점에서 설명해 보자.

### 2-4. 정리

```bash
kubectl delete pod config-demo
kubectl delete configmap demo-config
kubectl delete secret demo-secret
```

---

## 9. Volume 기초 — 임시성 · emptyDir · hostPath

### 9.1 컨테이너의 임시성

- 컨테이너 파일시스템은 **컨테이너 재시작 시 초기화**, Pod가 삭제되면 완전히 사라집니다.
- 같은 Pod 안의 컨테이너끼리도 파일시스템이 분리되어 파일 공유가 안 됩니다.
- → Pod 수준의 **Volume**을 정의하고 컨테이너에 마운트해 해결합니다.

### 9.2 기본 볼륨 유형

| 유형 | 수명 | 용도 | 주의 |
|---|---|---|---|
| `emptyDir` | **Pod와 동일** (Pod 삭제 시 소멸) | 컨테이너 간 공유, 캐시·임시 작업 공간 | 영속 아님. `medium: Memory` 옵션(tmpfs) |
| `hostPath` | 노드 디스크 | 노드의 특정 경로 마운트 (에이전트류) | Pod가 **다른 노드로 가면 데이터 없음**, 보안 위험 — 일반 앱에 사용 금지 |
| `configMap` / `secret` | 리소스와 동일 | 설정·인증서를 파일로 제공 | 읽기 전용 사용 권장 (6~7장) |

```yaml
# emptyDir 로 두 컨테이너가 파일 공유
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh","-c","while true; do date >> /work/log.txt; sleep 5; done"]
      volumeMounts: [{ name: work, mountPath: /work }]
    - name: reader
      image: busybox:1.36
      command: ["sh","-c","tail -f /work/log.txt"]
      volumeMounts: [{ name: work, mountPath: /work }]
  volumes:
    - name: work
      emptyDir: {}
```

---

## 10. PV · PVC · StorageClass

Pod보다 오래 살아야 하는 데이터(DB 등)는 **클러스터 수준의 영속 스토리지**를 사용합니다.

### 10.1 역할 분담

```
 [ 개발자 ]                    [ 클러스터/플랫폼 ]
 PVC: "10Gi, RWO 주세요" ──▶  StorageClass: "요청 오면 이렇게 만들어" (프로비저너)
        │                              │ (동적 프로비저닝)
        └────────── 바인딩 ◀────────── PV: 실제 스토리지 조각 (EBS, NFS, local...)
        
 Pod ──▶ volumes.persistentVolumeClaim: pvc-이름   (Pod는 PVC만 알면 됨)
```

| 리소스 | 역할 | 만드는 사람 |
|---|---|---|
| **PV** (PersistentVolume) | 실제 스토리지 한 조각의 표현 | 관리자(정적) 또는 프로비저너(동적) |
| **PVC** (PersistentVolumeClaim) | "이만큼 필요하다"는 요청서 | 개발자 |
| **StorageClass** | PVC 요청 시 PV를 자동 생성하는 방법 정의 | 관리자 (동적 프로비저닝의 핵심) |

### 10.2 accessModes

| 모드 | 의미 | 대표 스토리지 |
|---|---|---|
| `ReadWriteOnce` (RWO) | **한 노드**에서 읽기/쓰기 | 블록 스토리지(EBS 등) — 가장 흔함 |
| `ReadOnlyMany` (ROX) | 여러 노드에서 읽기 전용 | 공유 스토리지 |
| `ReadWriteMany` (RWX) | **여러 노드**에서 읽기/쓰기 | NFS, CephFS, EFS |
| `ReadWriteOncePod` (RWOP) | 단 하나의 Pod만 | 단독 점유 보장 |

### 10.3 동적 프로비저닝 흐름

1. 관리자가 StorageClass 준비 (`kubectl get storageclass`, 기본 SC에 `(default)` 표시)
2. 개발자가 PVC 생성 → 프로비저너가 조건에 맞는 **PV를 자동 생성·바인딩**
3. Pod가 PVC를 마운트 → Pod/노드가 바뀌어도 데이터 유지
4. `reclaimPolicy` — `Delete`(PVC 삭제 시 PV·데이터 삭제, 동적 기본) / `Retain`(보존)

---

## [실습 3] PVC 마운트

**목표**: PVC로 볼륨을 요청해 Pod에 마운트하고, Pod를 지워도 데이터가 남는지 확인한다.

### 3-1. StorageClass 확인

```bash
kubectl get storageclass
# 기본(default) SC가 없다면(온프렘 kubeadm 등) local-path-provisioner 설치:
# kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
# kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 3-2. PVC 생성

`day5-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
  # storageClassName 생략 시 기본 SC 사용
```

```bash
kubectl apply -f day5-pvc.yaml
kubectl get pvc data-pvc        # STATUS: Bound (또는 첫 Pod 생성 시 바인딩)
kubectl get pv                  # 자동 생성된 PV 확인 (동적 프로비저닝)
```

### 3-3. Pod에 마운트하고 데이터 기록

`day5-pvc-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
kubectl apply -f day5-pvc-pod.yaml
kubectl exec pvc-demo -- sh -c 'echo "persist me $(date)" >> /data/note.txt; cat /data/note.txt'
```

### 3-4. Pod 삭제 후 데이터 생존 확인

```bash
kubectl delete pod pvc-demo
kubectl apply -f day5-pvc-pod.yaml           # 같은 PVC로 새 Pod
kubectl exec pvc-demo -- cat /data/note.txt  # 이전 기록이 그대로!
```

> ✔ PVC가 Bound 되고 PV가 자동 생성되었다 (동적 프로비저닝).
> ✔ Pod를 삭제·재생성해도 /data/note.txt 내용이 유지된다.
> ✔ (질문) emptyDir였다면 이 실습 결과가 어떻게 달랐을까?
> ✔ (질문) PVC까지 삭제하면 데이터는? — reclaimPolicy(Delete/Retain)로 설명해 보자.

### 3-5. 정리

```bash
kubectl delete pod pvc-demo
kubectl delete pvc data-pvc
```

---

## 12. 5일차 리뷰 및 정리

### 12.1 오늘 배운 것 한 장 요약

| 주제 | 핵심 문장 |
|---|---|
| Ingress | 규칙(리소스)과 처리기(Controller)는 분리 — 진입점 하나로 호스트·경로 라우팅, TLS 종료 |
| cert-manager | 인증서를 선언하면 자동 발급·갱신 (ClusterIssuer + 어노테이션) |
| ConfigMap | 설정을 이미지에서 분리 — env는 재시작 필요, 볼륨은 파일 자동 갱신 |
| Secret | 민감정보 전용 — **base64는 암호화가 아니다** → RBAC·암호화·외부 비밀 관리로 보강 |
| Volume | emptyDir(Pod 수명) · hostPath(노드 종속, 주의) · configMap/secret 볼륨 |
| PV·PVC·SC | 개발자는 PVC로 요청, StorageClass가 PV를 동적 생성 — Pod가 바뀌어도 데이터 유지 |

### 12.2 과정 전체 리소스 지도 (Day 1~5)

```
 [빌드·배포]                [네트워크]              [설정·데이터]
 이미지 ──▶ Pod            Service (L4, 고정 접점)   ConfigMap (설정)
        ReplicaSet          └ ClusterIP/NodePort/LB  Secret (민감정보)
        Deployment ────────▶ Ingress (L7, 도메인/경로) Volume/PVC (영속 데이터)
        (선언·롤링·롤백)
              ▲ 모두 컨트롤러의 "조정 루프" 위에서 동작 (Day 4)
```

### 12.3 셀프 체크 (복습 질문)

1. Ingress 리소스만 만들고 Controller를 설치하지 않으면 어떻게 되는가?
2. `/api` 와 `/` 규칙이 함께 있을 때 `/api/v1/users` 요청은 어디로 가는가?
3. ConfigMap을 수정했는데 앱에 반영되지 않았다 — env 주입과 볼륨 마운트 각각에서 이유는?
4. `kubectl get secret -o yaml` 의 값이 안전하지 않은 이유와 3가지 보강책은?
5. emptyDir · hostPath · PVC의 데이터 수명을 각각 한 줄로 설명하라.
6. PVC 생성 시 PV가 자동으로 생기는 원리는 무엇이라 부르며, 무엇이 그 역할을 하는가?

### 12.4 과제 (선택)

- [실습 1]의 Ingress에 자체 서명 TLS(4.1)를 적용해 `https://` 로 접속해 보기
- [실습 3]의 Pod를 Deployment(replicas=1)로 바꾸고, 롤링 업데이트 후에도 데이터가 유지되는지 확인하기



---


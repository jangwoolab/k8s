아래는 기존 내용을 **Markdown 문서** 형식으로 재구성한 버전입니다.


# Amazon EKS 실습 가이드

## 실습 목표

본 실습에서는 Amazon EKS(Elastic Kubernetes Service)를 직접 구축하고 애플리케이션을 배포하는 과정을 수행합니다.

실습을 완료하면 다음 내용을 이해할 수 있습니다.

- Docker 이미지 생성 및 Registry 업로드
- Bastion Host 구성
- Amazon EKS 클러스터 생성
- Deployment 및 Service 생성
- LoadBalancer 서비스 구성
- Ingress Controller 설치
- Kubernetes 리소스 관리
- AWS 리소스 정리

---
# 제출 안내

## 제출 방법

- **Notion**에 모든 실습 내용을 정리
- 각 단계별
  - 사용한 명령어
  - 실행 결과
  - 화면 캡처
  - 간단한 설명
  를 함께 작성

완료 후 PDF로 변환하여 제출합니다.

### 파일명

1조_홍길동.pdf


### 제출처

p.jangwoo@gmail.com


---

# 실습 진행 순서

---

# STEP 1. Local Kubernetes에서 Ingress-NGINX 설치

## 목표

Local Kubernetes 환경에서 Ingress Controller를 설치하고 동작을 확인합니다.

### 수행 내용

- Ingress-NGINX 설치
- Pod 생성 확인
- Service 확인
- Ingress Controller 정상 동작 확인

### 정리 내용

- 설치 명령어
- 설치 결과
- kubectl get pod
- kubectl get svc
- 실행 화면 캡처

---

# STEP 2. PetClinic Docker 이미지 생성

## 목표

Spring PetClinic 프로젝트를 Docker 이미지로 생성합니다.

### 이미지 이름

```
petclinic:v1.0
```

### 작업 환경

Windows Docker Desktop

### 수행 내용

- Dockerfile 작성
- Docker Build
- Docker 실행
- 정상 동작 확인

### Registry 업로드

#### Docker Hub

- 로그인
- Image Push

#### Amazon ECR

- Private Repository 생성
- Login
- Tag 변경
- Push

### 정리 내용

- Dockerfile
- Build 명령어
- Push 명령어
- Docker Hub 화면
- AWS ECR 화면

---

# STEP 3. Bastion Host 구성

## 목표

EKS를 관리하기 위한 Bastion Host를 생성합니다.

### EC2 설정

| 항목 | 값 |
|------|-----|
| OS | Amazon Linux 2023 |
| Instance Type | t3.micro |
| VPC | Default VPC |
| Public IP | Enabled |

### 보안 그룹

본인의 공인 IP만 SSH 접근 가능하도록 설정합니다.

### 설치 항목

- kubectl
- AWS CLI
- eksctl

### 설치 확인

```
kubectl version --client

aws --version

eksctl version
```

### 정리 내용

- EC2 생성 화면
- 보안 그룹
- 설치 명령어
- 버전 확인 결과

---

# STEP 4. Amazon EKS 클러스터 생성

## 목표

eksctl을 이용하여 Amazon EKS 클러스터를 생성합니다.

### 클러스터 정보

| 항목 | 값 |
|------|-----|
| Cluster Name | demo-eks |
| Region | ap-northeast-2 |
| Node Group | demo-ng |
| Instance Type | t3.medium |
| Node Count | 실습 환경에 맞게 |
| Volume | 20GB |

### 수행 내용

- eksctl 명령어 작성
- 클러스터 생성
- Node 확인

### 확인 명령어

```
kubectl get nodes

kubectl get pods -A
```

### 정리 내용

- eksctl 명령어
- 생성 로그
- Node 확인 결과

---

# STEP 5. PetClinic Deployment 생성

## 목표

Deployment를 이용하여 애플리케이션을 배포합니다.

### Deployment 이름

```
petclinic-app
```

### 이미지

```
petclinic:v1.0
```

### Replica 수

```
3
```

### 확인

```
kubectl get deployment

kubectl get pod

kubectl describe deployment
```

### 정리 내용

- Deployment YAML
- 적용 명령어
- Pod 실행 결과

---

# STEP 6. LoadBalancer Service 생성

## 목표

Deployment를 외부에서 접근할 수 있도록 Service를 생성합니다.

### Service Type

```
LoadBalancer
```

### 확인

```
kubectl get svc
```

### 수행 내용

- EXTERNAL-IP 확인
- DNS Name 확인
- 브라우저 접속 테스트

### 정리 내용

- Service YAML
- Service 생성 결과
- 접속 화면 캡처

---

# STEP 7. EKS에 Ingress-NGINX 설치

## 목표

Amazon EKS 환경에서 Ingress Controller를 설치합니다.

### 수행 내용

- Ingress-NGINX 설치
- Controller Pod 확인
- Service 확인

### 확인 명령어

```
kubectl get pods -n ingress-nginx

kubectl get svc -n ingress-nginx
```

### 정리 내용

- 설치 명령어
- 설치 결과
- Pod 상태
- Service 상태

---

# STEP 8. Kubernetes 리소스 삭제

## 목표

생성한 Kubernetes 리소스를 삭제합니다.

### 삭제 대상

- Deployment
- Service

### 확인

```
kubectl get deployment

kubectl get svc

kubectl get pod
```

### 정리 내용

- 삭제 명령어
- 삭제 결과

---

# STEP 9. AWS 리소스 삭제

## 목표

실습 종료 후 AWS 리소스를 모두 삭제합니다.

### 삭제 대상

- EKS Cluster
- Node Group
- Load Balancer
- EBS
- Security Group
- ECR Repository (필요 시)
- Bastion Host

### 확인

AWS Console에서 리소스가 모두 삭제되었는지 확인합니다.

---

# 제출 체크리스트

| 항목 | 완료 |
|------|------|
| STEP 1 완료 | ☐ |
| STEP 2 완료 | ☐ |
| STEP 3 완료 | ☐ |
| STEP 4 완료 | ☐ |
| STEP 5 완료 | ☐ |
| STEP 6 완료 | ☐ |
| STEP 7 완료 | ☐ |
| STEP 8 완료 | ☐ |
| STEP 9 완료 | ☐ |

---

# 제출 전 확인사항

- Notion에 단계별로 정리했는가?
- 모든 명령어를 기록했는가?
- 주요 실행 결과를 캡처했는가?
- 오류가 발생했다면 해결 과정을 작성했는가?
- PDF로 변환했는가?
- 파일명을 규칙에 맞게 저장했는가?
- 제출 메일 주소를 확인했는가?

---

# 학습 목표 요약

이번 실습을 완료하면 다음 기술을 직접 경험하게 됩니다.

- Docker 이미지 생성
- Docker Hub 사용
- Amazon ECR 사용
- Amazon EC2(Bastion Host)
- Amazon EKS 구축
- Kubernetes Deployment
- Kubernetes Service
- LoadBalancer
- Ingress-NGINX
- kubectl 활용
- eksctl 활용
- AWS 리소스 관리 및 삭제
````



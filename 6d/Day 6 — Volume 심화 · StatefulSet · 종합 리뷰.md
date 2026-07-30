# Day 6 — Volume 심화 · StatefulSet · 종합 리뷰

> Kubernetes v1.36 기준 · PV의 reclaim policy와 프로비저닝 방식을 깊이 이해하고,
> StatefulSet으로 안정적 식별자를 가진 상태 유지 워크로드(DB)를 배포·운영합니다.

---

## 학습 목표

- PV의 **reclaim policy**(Retain/Delete/Recycle)와 **정적 vs 동적 프로비저닝**의 차이를 이해하고, StorageClass의 상세 옵션을 설명할 수 있다.
- StatefulSet과 Deployment의 차이(안정적 식별자, 순서 보장, Headless Service)를 이해하고, `volumeClaimTemplates`로 Pod별 전용 PVC를 가진 DB를 배포·스케일할 수 있다.
- 업데이트 전략(`partition`)과 PVC 보존 동작을 이해하고, 상태 유지 워크로드의 문제를 종합적으로 디버깅할 수 있다.

---

## 목차

1. [PV 심화 ① Reclaim Policy](#1-pv-심화-①-reclaim-policy)
2. [PV 심화 ② 정적 vs 동적 프로비저닝](#2-pv-심화-②-정적-vs-동적-프로비저닝)
3. [PV 심화 ③ StorageClass 상세](#3-pv-심화-③-storageclass-상세)
4. [StatefulSet 개념 ① Deployment와 차이](#4-statefulset-개념-①-deployment와-차이)
5. [StatefulSet 개념 ② 안정적 식별자 · 순서 보장 · Headless Service](#5-statefulset-개념-②-안정적-식별자--순서-보장--headless-service)
6. [StatefulSet 실습 준비 — volumeClaimTemplates](#6-statefulset-실습-준비--volumeclaimtemplates)
7. [[실습 1] DB 배포](#실습-1-db-배포)
8. [스케일과 파드별 PVC 확인](#8-스케일과-파드별-pvc-확인)
9. [업데이트 전략 — partition](#9-업데이트-전략--partition)
10. [PVC 보존 동작](#10-pvc-보존-동작)
11. [[실습 2] 종합 디버깅](#실습-2-종합-디버깅)
12. [6일차 리뷰 및 정리](#12-6일차-리뷰-및-정리)

---

## 1. PV 심화 ① Reclaim Policy

PVC가 삭제된 후 남은 PV(와 실제 스토리지)를 **어떻게 처리할지** 결정하는 정책입니다.

| 정책 | 동작 | 상태 전이 | 사용 시점 |
|---|---|---|---|
| **Delete** | PV와 **실제 스토리지까지 삭제** | PVC 삭제 → PV·데이터 즉시 삭제 | 동적 프로비저닝 기본값. 임시성 데이터, 테스트 환경 |
| **Retain** | PV는 남지만 **재사용 불가 상태(Released)** 로 보존 | PVC 삭제 → PV `Released` (수동 개입 필요) | DB 등 **중요 데이터**, 실수로 인한 삭제 방지 |
| **Recycle** (사실상 폐지) | 데이터 삭제 후 PV 재사용 가능 상태로 | - | v1.20+ 제거 예정 경고 — 사용 금지, 동적 프로비저닝으로 대체 |

```bash
# 정책 확인·변경
kubectl get pv <PV이름> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
kubectl patch pv <PV이름> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

**Retain 상태의 PV를 재사용하려면?**

```bash
# 1) Released 상태 확인
kubectl get pv

# 2) claimRef 제거 → Available로 전환
kubectl patch pv <PV이름> -p '{"spec":{"claimRef": null}}'

# 3) 새 PVC로 다시 바인딩 (동일 조건이면 자동 매칭)
```

> **실무 원칙** — DB·업로드 파일처럼 잃으면 안 되는 데이터의 StorageClass는
> `reclaimPolicy: Retain` 으로 설정합니다 (기본은 Delete임에 주의).

---

## 2. PV 심화 ② 정적 vs 동적 프로비저닝

| 구분 | 정적(Static) Provisioning | 동적(Dynamic) Provisioning |
|---|---|---|
| PV 생성 주체 | **관리자가 미리** YAML로 생성 | PVC 요청 시 **프로비저너가 자동 생성** |
| 사용 흐름 | PV 생성 → PVC가 매칭되는 PV를 찾아 바인딩 | PVC 생성 → StorageClass가 즉시 PV 생성 |
| 적합한 경우 | 기존 스토리지 재사용, 세밀한 통제가 필요할 때 | 대부분의 클라우드 환경 (표준) |
| StorageClass | 불필요(`storageClassName: ""` 가능) | 필수 |

### 2.1 정적 프로비저닝 예시

```yaml
# 관리자가 먼저 PV 생성
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual-01
spec:
  capacity: { storage: 5Gi }
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath: { path: /mnt/data/pv01 }     # 예시(실습용) - 운영에선 NFS/블록스토리지 등
---
# 개발자가 조건에 맞는 PVC 생성 → 위 PV와 자동 바인딩
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manual
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: manual
  resources: { requests: { storage: 5Gi } }
```

### 2.2 동적 프로비저닝 흐름 (복습 + 심화)

```
PVC 생성 (storageClassName 지정 또는 기본 SC 사용)
   │
   ▼
StorageClass가 지정한 provisioner 동작
   │  (예: ebs.csi.aws.com, kubernetes.io/no-provisioner 는 동적 불가)
   ▼
CSI 드라이버가 실제 스토리지 생성 + PV 오브젝트 자동 등록
   │
   ▼
PVC ↔ PV 자동 Bound
```

- CSI(Container Storage Interface) 드라이버가 각 스토리지 벤더와의 연동을 표준화합니다.
- `kubernetes.io/no-provisioner` 는 **동적 생성이 불가능**함을 의미(로컬 볼륨 등, 수동 PV 필요).

---

## 3. PV 심화 ③ StorageClass 상세

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com          # 어떤 CSI 드라이버를 쓸지
parameters:                            # 프로비저너별 세부 옵션
  type: gp3
  iops: "3000"
reclaimPolicy: Retain                  # 이 SC로 만든 PV의 기본 정책 (기본값 Delete)
volumeBindingMode: WaitForFirstConsumer  # 아래 참고
allowVolumeExpansion: true             # PVC 크기 확장 허용 여부
```

| 필드 | 설명 |
|---|---|
| `provisioner` | 스토리지를 실제로 만드는 드라이버(CSI). 클라우드별로 다름 |
| `parameters` | 볼륨 타입·IOPS·파일시스템 등 프로비저너 종속 옵션 |
| `reclaimPolicy` | 이 클래스로 만들어진 PV의 기본 회수 정책 |
| `volumeBindingMode` | `Immediate`(PVC 생성 즉시 PV 생성) vs **`WaitForFirstConsumer`**(Pod가 스케줄될 때까지 대기 → 같은 가용영역에 볼륨 생성, 권장) |
| `allowVolumeExpansion` | `true` 면 PVC의 `resources.requests.storage` 를 늘려 온라인 확장 가능 |

```bash
kubectl get storageclass
kubectl describe storageclass <이름>
kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

> **volumeBindingMode 팁** — 멀티 AZ 클러스터에서 `Immediate`를 쓰면 볼륨과 Pod가
> 다른 AZ에 생성되어 마운트가 실패할 수 있습니다. 클라우드 환경은 `WaitForFirstConsumer`가 사실상 표준입니다.

---

## 4. StatefulSet 개념 ① Deployment와 차이

Deployment는 "교체 가능한 동일한 Pod 여러 개"를 다루기에 최적화되어 있습니다.
하지만 **DB처럼 각 인스턴스가 고유한 신원과 데이터를 가져야 하는 경우** 적합하지 않습니다.

| 항목 | Deployment | StatefulSet |
|---|---|---|
| Pod 이름 | 랜덤 해시 (`web-7f9c8d-x2k9p`) | **고정 순번** (`db-0`, `db-1`, `db-2`) |
| Pod 정체성 | 서로 교체 가능(interchangeable) | **각자 고유** — db-0은 항상 db-0 |
| 생성/삭제 순서 | 동시(병렬) | **순차적** (0 → 1 → 2, 삭제는 역순) |
| 스토리지 | 모든 Pod가 같은 PVC 공유 가능 | **Pod마다 전용 PVC** (volumeClaimTemplates) |
| 네트워크 식별 | Service 뒤에서 IP 구분 없음 | Headless Service로 **Pod별 고정 DNS** |
| 재스케줄 후 | 새 이름/IP로 아무 데나 재생성 | **같은 이름 유지**, 기존 PVC 재연결 |

**언제 무엇을 쓰나**

- **Deployment**: 웹 서버, API 서버 등 무상태(stateless) — 어떤 Pod가 응답해도 무관
- **StatefulSet**: MySQL, PostgreSQL, Kafka, Elasticsearch, Zookeeper 등 — 각 노드가 역할·데이터를 가짐

---

## 5. StatefulSet 개념 ② 안정적 식별자 · 순서 보장 · Headless Service

### 5.1 안정적 네트워크 식별자

```
Pod 이름:  db-0            db-1            db-2
DNS:       db-0.db-hl.default.svc.cluster.local
           db-1.db-hl.default.svc.cluster.local
           db-2.db-hl.default.svc.cluster.local
```

- Pod가 재생성되어도 **이름·DNS·PVC가 그대로 유지**됩니다.
- Primary/Replica 구성에서 "db-0 = Primary"처럼 **역할을 이름에 고정**할 수 있습니다.

### 5.2 순서 보장 (Ordinal Index)

- **생성**: db-0 Running+Ready 확인 후 db-1 생성 → ... 순차 진행 (기본 `podManagementPolicy: OrderedReady`)
- **삭제/축소**: 역순(높은 인덱스부터)
- 병렬이 필요하면 `podManagementPolicy: Parallel` 로 완화 가능 (신원 유지 자체는 동일)

### 5.3 Headless Service (clusterIP: None)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-hl
spec:
  clusterIP: None                # Headless
  selector: { app: db }
  ports: [{ port: 5432 }]
```

- 일반 Service처럼 **단일 VIP로 뭉뚱그리지 않고**, DNS 조회 시 **각 Pod의 IP를 개별 반환**합니다.
- StatefulSet은 이 Headless Service와 짝을 이뤄 `spec.serviceName` 에 지정합니다.
- 클라이언트(또는 앱)가 `db-0.db-hl`, `db-1.db-hl` 처럼 **특정 개체를 지목**해 접속할 수 있습니다.

---

## 6. StatefulSet 실습 준비 — volumeClaimTemplates

`volumeClaimTemplates` 는 StatefulSet이 **Pod마다 자동으로 전용 PVC를 생성**하게 하는 핵심 필드입니다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-hl              # 반드시 Headless Service 지정
  replicas: 3
  selector:
    matchLabels: { app: db }
  template:
    metadata:
      labels: { app: db }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: db-secret, key: password } }
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:             # ★ Pod마다 PVC 자동 생성
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 2Gi } }
```

- 생성되는 PVC 이름 규칙: **`<template이름>-<Pod이름>`** → `data-db-0`, `data-db-1`, `data-db-2`
- 각 PVC는 **독립적인 PV**에 바인딩되어, db-1의 데이터가 db-0에 섞이지 않습니다.

---

## [실습 1] DB 배포

**목표**: PostgreSQL을 StatefulSet으로 배포하고, Pod별 전용 볼륨과 고정 DNS를 확인한다.

### 1-1. Secret과 Headless Service

```bash
kubectl create secret generic db-secret --from-literal=password='S3cure!pw'
```

`day6-headless.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-hl
spec:
  clusterIP: None
  selector: { app: db }
  ports: [{ port: 5432, name: postgres }]
```

### 1-2. StatefulSet 적용

```bash
kubectl apply -f day6-headless.yaml
kubectl apply -f day6-statefulset.yaml     # 6장의 정의 사용, replicas: 3

kubectl get pods -l app=db -w
# db-0 → Running/Ready 후 db-1 → db-2 순으로 생성되는 것을 관찰
```

### 1-3. 식별자 확인

```bash
kubectl get pods -l app=db -o wide
kubectl get pvc -l app=db                   # data-db-0, data-db-1, data-db-2

# Pod별 고정 DNS 확인
kubectl run client --rm -it --image=busybox:1.36 --restart=Never -- \
  nslookup db-1.db-hl.default.svc.cluster.local
```

### 1-4. 데이터 격리 확인

```bash
kubectl exec -it db-0 -- psql -U postgres -c "CREATE TABLE t(id int); INSERT INTO t VALUES (0);"
kubectl exec -it db-1 -- psql -U postgres -c "SELECT * FROM t;"   # 에러 또는 빈 결과 - 별개 데이터
```

> ✔ db-0, db-1, db-2 가 순차적으로 Ready 되었다.
> ✔ PVC가 Pod 수만큼 개별 생성되었다 (`data-db-0` 등).
> ✔ db-1.db-hl 로 db-1 Pod의 IP를 직접 조회했다.
> ✔ db-0에 넣은 데이터가 db-1에는 없다 — **각자 독립된 스토리지**를 사용함을 확인했다.

---

## 8. 스케일과 파드별 PVC 확인

```bash
# 확장 — 순차적으로 db-3 추가, 새 PVC(data-db-3) 자동 생성
kubectl scale statefulset db --replicas=4
kubectl get pods -l app=db -w
kubectl get pvc -l app=db

# 축소 — 역순으로 삭제 (db-3부터)
kubectl scale statefulset db --replicas=2
kubectl get pods -l app=db
kubectl get pvc -l app=db          # ★ data-db-2, data-db-3 PVC는 그대로 남아있음!
```

> ✔ 스케일 축소 후에도 PVC는 삭제되지 않는다 — StatefulSet의 **안전장치**(데이터 유실 방지).
> ✔ (질문) 다시 `--replicas=4` 로 늘리면 db-2는 예전 데이터를 그대로 이어받을까? 직접 확인해 보자.

---

## 9. 업데이트 전략 — partition

StatefulSet의 기본 업데이트 전략은 `RollingUpdate`이며, **높은 순번부터 역순**으로 갱신합니다.
`partition` 값을 지정하면 **일부 Pod만 카나리 방식으로 먼저 업데이트**할 수 있습니다.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2        # 인덱스 2 이상만 업데이트 대상 (replicas=4 라면 db-2, db-3만)
```

```bash
# partition=2로 설정 후 이미지 변경 → db-3, db-2만 갱신되고 db-1, db-0은 유지됨
kubectl patch statefulset db -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
kubectl set image statefulset/db postgres=postgres:16.4

kubectl get pods -l app=db -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.containers[0].image}{"\n"}{end}'
# db-0, db-1: postgres:16   /   db-2, db-3: postgres:16.4
```

- 검증이 끝나면 `partition` 을 0으로 낮춰 **나머지도 순차 업데이트**합니다.
- Deployment의 `maxSurge/maxUnavailable` 과 달리, StatefulSet은 **"어디서부터"** 를 인덱스로 통제한다는 점이 핵심 차이입니다.

---

## 10. PVC 보존 동작

StatefulSet의 PVC는 기본적으로 **StatefulSet/Pod가 삭제되어도 남습니다.**

| 동작 | PVC 결과 |
|---|---|
| Pod 삭제(재스케줄) | 유지 — 같은 이름의 새 Pod가 동일 PVC 재연결 |
| `replicas` 축소 | 유지 (8장 확인) |
| `kubectl delete statefulset db` | 유지 (StatefulSet만 삭제, PVC는 그대로) |
| `kubectl delete statefulset db --cascade=orphan` | Pod도 유지한 채 컨트롤러만 제거 (고급) |
| **PVC를 직접 삭제**해야 비로소 정리됨 | `kubectl delete pvc -l app=db` |

v1.27+ 부터는 `persistentVolumeClaimRetentionPolicy` 로 세밀 제어가 가능합니다.

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain     # StatefulSet 삭제 시: Retain(기본, 보존) | Delete
    whenScaled: Retain      # 스케일 축소 시: Retain(기본, 보존) | Delete
```

> **왜 기본이 보존인가** — DB StatefulSet을 실수로 지워도 데이터(PVC)는 남아,
> 같은 이름으로 재생성하면 복구됩니다. 완전 삭제는 **항상 PVC를 별도로, 의도적으로** 지워야 합니다.

---

## [실습 2] 종합 디버깅

**목표**: 흔한 장애 3가지를 재현하고 원인을 찾아 복구한다.

### 시나리오 A — Pod가 Pending에서 멈춤

```bash
# 존재하지 않는 StorageClass로 volumeClaimTemplates를 바꿔 재배포했다고 가정
kubectl describe pod db-0 | tail -15
kubectl get pvc -l app=db                          # Pending 상태 확인
kubectl describe pvc data-db-0 | grep -A3 Events    # 원인 이벤트 확인
```

> 원인 예시: 존재하지 않는 StorageClass, 용량 부족, `WaitForFirstConsumer`인데 스케줄 불가 등.

### 시나리오 B — 순서에 막혀 전체가 멈춤

```bash
# db-0이 CrashLoopBackOff 이면 OrderedReady 정책상 db-1 이후가 생성되지 않는다
kubectl get pods -l app=db
kubectl logs db-0 --previous
kubectl describe pod db-0 | grep -A10 Events
```

> ✔ db-0 하나의 장애가 전체 StatefulSet 롤아웃을 막을 수 있음을 확인 —
> `podManagementPolicy: Parallel` 이 필요한 상황인지 판단해 보자.

### 시나리오 C — 재배포했는데 이전 데이터가 안 보임

```bash
# PVC가 새로 생성됐는지, 기존 PVC에 다시 연결됐는지 확인
kubectl get pvc -l app=db -o custom-columns=NAME:.metadata.name,VOLUME:.spec.volumeName,STATUS:.status.phase
kubectl get pv | grep <PVC에 매칭된 볼륨>
```

> 원인 후보: PVC를 함께 삭제한 뒤 재생성(신규 PV 할당됨), reclaimPolicy가 Delete라 실제 데이터가 사라짐,
> 다른 네임스페이스/다른 이름으로 배포해 새 PVC 세트가 만들어짐.

### 디버깅 체크리스트

```bash
kubectl get statefulset db -o wide
kubectl get pods -l app=db -o wide
kubectl get pvc -l app=db
kubectl get pv
kubectl describe statefulset db | tail -20
kubectl get events --sort-by=.lastTimestamp | grep -i db
```

> ✔ 세 시나리오 각각에서 "증상 → 확인 명령 → 근본 원인"을 한 문장으로 정리해 본다.

---

## 12. 6일차 리뷰 및 정리

### 12.1 오늘 배운 것 요약

| 주제 | 핵심 문장 |
|---|---|
| Reclaim Policy | Delete(기본, 삭제) vs Retain(중요 데이터는 이걸로) — Recycle은 폐지 |
| 정적 vs 동적 | 정적은 관리자가 PV 선생성, 동적은 StorageClass+CSI가 자동 생성 |
| StorageClass | provisioner·volumeBindingMode(WaitForFirstConsumer 권장)·allowVolumeExpansion |
| StatefulSet vs Deployment | 고정 이름/순서 보장/Pod별 전용 PVC/Headless DNS — DB 등 상태 유지 워크로드용 |
| volumeClaimTemplates | Pod마다 `<template>-<Pod이름>` PVC 자동 생성, 데이터 격리 |
| partition | 인덱스 기준 카나리 업데이트 — 상위 번호부터, 검증 후 0으로 낮춰 전체 반영 |
| PVC 보존 | 기본은 보존 — Pod/StatefulSet 삭제·스케일 축소해도 유지, 완전 삭제는 PVC를 별도로 |

### 12.2 Deployment ↔ StatefulSet 선택 가이드 (복습)

```
"이 워크로드의 각 인스턴스가 서로 바꿔써도 무방한가?"
        │
   YES ─┴─ NO
   │        │
 Deployment  StatefulSet
 (웹/API)    (DB, 메시지 큐, 분산 코디네이터)
```

### 12.3 셀프 체크 (복습 질문)

1. `reclaimPolicy: Retain` 인 PV를 재사용하려면 어떤 필드를 정리해야 하는가?
2. `volumeBindingMode: Immediate` 가 멀티 AZ 환경에서 문제가 될 수 있는 이유는?
3. StatefulSet의 Pod가 `db-0, db-1, db-2` 처럼 이름이 고정되는 것이 왜 중요한가? (Deployment와 비교)
4. `volumeClaimTemplates` 로 만들어진 PVC의 이름 규칙은 무엇인가?
5. replicas를 4→2로 줄였다가 다시 4로 늘리면 db-2, db-3의 데이터는 어떻게 되는가?
6. `partition: 2` 의 의미와, 검증 후 전체 반영을 위해 어떤 값으로 바꿔야 하는가?

### 12.4 과제 (선택)

- Retain 정책의 PV를 Released 상태로 만든 뒤, `claimRef` 를 정리해 새 PVC로 재바인딩해 보기
- `persistentVolumeClaimRetentionPolicy.whenScaled: Delete` 로 바꾸고 스케일 축소 시 PVC가 실제로 삭제되는지 비교 실험

**Day 1~6 과정 마무리** — 다음 심화 주제 예고: HPA(자동 확장), RBAC, Helm 차트 작성, Operator 패턴, kube-prometheus-stack 모니터링.

---

*Kubernetes v1.36 · K8s Essentials Day 6 교안 · 2026-07*

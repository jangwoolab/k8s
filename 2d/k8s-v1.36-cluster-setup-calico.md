# Kubernetes v1.36 클러스터 구성 가이드 (kubeadm + containerd + Calico)

> 이 문서는 **사전 셋팅이 완료된** 상태에서 `kubeadm init`으로 컨트롤 플레인을 초기화하고, **Calico CNI**를 설치한 뒤 **워커 노드를 join**해 클러스터를 완성하는 과정을 단계별로 정리한 것입니다.
> 환경: **Rocky Linux 10.2 / Kubernetes v1.36 / CRI: containerd / CNI: Calico v3.32.0**

---

## 전제 조건

이 문서는 다음 사전 셋팅이 모든 노드에 적용되어 있다고 가정합니다. (선행 문서 "설치 전 셋팅 가이드" 참조)

- swap 비활성화 / SELinux permissive / 방화벽 포트 개방
- overlay·br_netfilter 모듈, sysctl(ip_forward 등) 설정
- containerd 설치 + `SystemdCgroup = true` + 실행 중
- 모든 노드에 kubelet·kubeadm·kubectl(v1.36) 설치, kubelet enable

## 적용 범위 표기

- **[CP]** : 컨트롤 플레인 노드에서만 수행
- **[WORKER]** : 각 워커 노드에서 수행
- **[ALL]** : 모든 노드에서 수행

## 예시 환경 (본인 환경에 맞게 변경)

| 노드 | 호스트네임 | IP |
|---|---|---|
| 베스천 호스트 | bastion | 10.10.10.100 |
| 컨트롤 플레인 | k8s-cp01 | 10.10.10.10 |
| 워커 노드 1 | k8s-worker01 | 10.10.10.21 |
| 워커 노드 2 | k8s-worker02 | 10.10.10.22 |

| 항목 | 값 |
|---|---|
| Pod 네트워크 CIDR | 192.168.0.0/16 (Calico operator 기본값) |
| CRI 소켓 | unix:///run/containerd/containerd.sock |

---

# STEP 0. CNI 설치 전 NetworkManager 설정 [ALL]

Rocky의 NetworkManager가 Calico 인터페이스를 건드리지 않도록 예외를 등록합니다. (생략 시 네트워크가 불안정할 수 있음)

```bash
cat <<EOF | sudo tee /etc/NetworkManager/conf.d/calico.conf
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
EOF

sudo systemctl reload NetworkManager
```

---

# STEP 1. 컨트롤 플레인 초기화 (kubeadm init) [CP]

컨트롤 플레인 노드에서만 실행합니다.

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=10.10.10.10 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

옵션 설명:
- `--pod-network-cidr` : Pod IP 대역. **Calico operator 기본값(192.168.0.0/16)과 일치**시킵니다. 다른 대역을 쓰려면 STEP 3에서 Calico 설정도 함께 바꿔야 합니다.
- `--apiserver-advertise-address` : API 서버가 광고할 컨트롤 플레인 IP.
- `--cri-socket` : containerd만 설치된 경우 자동 감지되지만 명시하면 안전합니다.

> 초기화가 성공하면 출력 맨 아래에 워커 노드용 **`kubeadm join ...` 명령**이 나옵니다. 이 명령을 반드시 복사해 두세요. (STEP 4에서 사용)

> ### 주의 : 아래는 kubeadm init 출력 결과임 반드시 출력 결과를 복사해야 다음 STEP을 진행 할 수 있음. 
```bash
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.10.10.10:6443 --token swtrv8.ndh9byn6ibk4ywur \
        --discovery-token-ca-cert-hash sha256:2016b5c9c1897f382e8e0e3b50a64c69718d7bfaa9d8a6c7edbf6f9df64ed1ff

```

미리 이미지를 받아 시간을 단축하려면 init 전에 실행할 수 있습니다.

```bash
sudo kubeadm config images pull
```

---

# STEP 2. kubectl 사용 설정 [CP]

일반 사용자가 kubectl을 쓸 수 있도록 kubeconfig를 설정합니다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

확인 (이 시점에는 CNI 미설치라 노드가 **NotReady**인 것이 정상):

```bash
kubectl get nodes
# NAME       STATUS     ROLES           AGE   VERSION
# k8s-cp01   NotReady   control-plane   1m    v1.36.x

kubectl get pods -n kube-system
# coredns 파드는 CNI 설치 전까지 Pending 상태로 대기
```

---

# STEP 3. Calico CNI 설치 (operator 방식) [CP]

Calico는 operator로 설치하는 것이 권장 방식입니다. operator를 배포한 뒤, Calico 구성을 정의하는 커스텀 리소스를 적용합니다.

## 3-1. Tigera operator 설치

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
```

## 3-2. Calico 설치 정의(custom resources) 적용

Pod CIDR을 STEP 1과 **다르게** 지정했다면 먼저 파일을 받아 수정한 뒤 적용하세요.

```bash
# 기본값(192.168.0.0/16) 그대로 사용하는 경우 — 바로 적용
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
```

CIDR을 바꿔야 할 경우:

```bash
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
# custom-resources.yaml 안의 cidr 값을 kubeadm init 시 사용한 값과 동일하게 수정
#   spec.calicoNetwork.ipPools[0].cidr: 192.168.0.0/16  ->  원하는 대역
kubectl create -f custom-resources.yaml
```

## 3-3. Calico 기동 확인

```bash
# calico-system 네임스페이스의 파드가 모두 Running 될 때까지 대기
kubectl get pods -n calico-system -w

# operator 상태
kubectl get pods -n tigera-operator

# Installation 리소스 진행 상태
kubectl get tigerastatus
```

`kubectl get tigerastatus`의 모든 항목이 `AVAILABLE: True`가 되면 완료입니다. 이후 노드 상태를 확인합니다.

```bash
kubectl get nodes
# k8s-cp01   Ready   control-plane   8m   v1.36.x   <- Ready 로 전환
```

> 참고: Calico v3.32는 Linux 커널 5.10 이상을 요구합니다. Rocky 10.2(6.12 계열)는 충족합니다.

---

# STEP 4. 워커 노드 join [WORKER]

각 워커 노드에서, STEP 1의 init 출력에 나온 join 명령을 실행합니다. (토큰·해시는 본인 출력값 사용)
> ### 적용 구문 : 주의 :  token, hash 는 자신의 시스템에 맞게 적용함.  
```bash
sudo kubeadm join 10.0.0.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket=unix:///run/containerd/containerd.sock
```
> ### 실제 적용예) : <== token, hash값을 자신의 시스템에 맞게 부여 할 것 
```bash
sudo kubeadm join 10.10.10.10:6443 \
  --token swtrv8.ndh9byn6ibk4ywur \
  --discovery-token-ca-cert-hash sha256:2016b5c9c1897f382e8e0e3b50a64c69718d7bfaa9d8a6c7edbf6f9df64ed1ff \
  --cri-socket=unix:///run/containerd/containerd.sock
```

## 참고) join 명령을 잃어버렸거나 토큰이 만료된 경우 [CP]

토큰 기본 유효기간은 24시간입니다. 컨트롤 플레인에서 새 join 명령을 생성하세요.

```bash
kubeadm token create --print-join-command
```

---

# STEP 5. 클러스터 검증 [CP]

```bash
# 모든 노드가 Ready 인지 확인
kubectl get nodes -o wide

# 예상 출력
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-cp01       Ready    control-plane   15m   v1.36.x
# k8s-worker01   Ready    <none>          3m    v1.36.x
# k8s-worker02   Ready    <none>          3m    v1.36.x

# 시스템 파드 확인
kubectl get pods -A

# 컴포넌트 헬스 확인
kubectl get --raw='/readyz?verbose'
```

---

# STEP 6. 동작 테스트 배포 [CP]

CNI와 노드 스케줄링이 정상인지 간단한 배포로 검증합니다.

```bash
# 테스트 매니페스트 생성
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
spec:
  type: NodePort
  selector:
    app: nginx-test
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
EOF

# 파드가 여러 노드에 분산 배치되는지 확인
kubectl get pods -o wide

# 접속 테스트 (임의 노드 IP로)
curl http://10.0.0.21:30080

# 정리
kubectl delete deployment nginx-test
kubectl delete service nginx-test
```

세 개의 파드가 서로 다른 노드에 뜨고 NodePort로 응답하면 클러스터 네트워킹이 정상입니다.

---

# 클러스터 구성 완료

여기까지 진행하면 다음이 완성됩니다.

- containerd 런타임 기반의 컨트롤 플레인 1대 + 워커 2대
- Calico CNI로 Pod 간 통신 가능
- NodePort로 외부 노출 검증 완료

다음으로 진행할 만한 주제: Ingress Controller 설치, MetalLB(온프레미스 LoadBalancer), 메트릭 서버(HPA 전제), 스토리지(CSI) 구성 등.

---

## 부록 A. 컨트롤 플레인에 워크로드 배치 (단일 노드 실습용)

기본적으로 컨트롤 플레인에는 일반 파드가 스케줄되지 않습니다. 단일 노드로 실습한다면 taint를 제거합니다. (멀티 노드 운영 환경에서는 권장하지 않음)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 부록 B. 노드 제거 / 초기화

```bash
# [CP] 노드를 안전하게 비우고 제거
kubectl drain k8s-worker01 --ignore-daemonsets --delete-emptydir-data
kubectl delete node k8s-worker01

# [해당 노드] kubeadm 상태 초기화
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
# iptables/ipvs 규칙이 남았다면 추가 정리 필요
```

## 부록 C. 자주 겪는 문제

| 증상 | 원인 / 조치 |
|---|---|
| 노드가 계속 NotReady | CNI(Calico) 미설치 또는 기동 실패 — `kubectl get pods -n calico-system` 확인 |
| coredns가 Pending | 동일하게 CNI 설치 전 정상. Calico 설치 후 해소 |
| join 시 cgroup/CRI 오류 | containerd `SystemdCgroup=true` 및 `--cri-socket` 확인 |
| join 토큰 만료 | `[CP]` 에서 `kubeadm token create --print-join-command` |
| Pod CIDR 불일치로 통신 안 됨 | kubeadm init의 `--pod-network-cidr`와 Calico custom-resources의 cidr 일치 확인 |
| 방화벽으로 노드 간 통신 불가 | 6443·10250·NodePort 및 Calico 포트(BGP 179/tcp, VXLAN 4789/udp 등) 확인 |
| calico 인터페이스 불안정 | STEP 0의 NetworkManager 예외 설정 확인 |
```

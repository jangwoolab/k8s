# Kubernetes v1.36 설치 전 셋팅 가이드 (Rocky Linux 10.2)

> 이 문서는 `kubeadm`으로 클러스터를 초기화(`kubeadm init`)하기 **전에** 모든 노드에서 수행해야 하는 사전 셋팅을 단계별로 정리한 것입니다.

> 대상 환경: **Rocky Linux v10.2 minimal / Kubernetes v1.36 / 컨테이너 런타임(CRI) : containerd / CNI : Calico**

---

## 적용 범위 표기

- **[ALL]** : 컨트롤 플레인·워커 노드 **모두**에서 수행
- **[CP]**  : 컨트롤 플레인(Control Plane) 노드에서만 수행
- **[WORKER]** : 워커(Worker) 노드에서만 수행

대부분의 단계는 [ALL]이며, 방화벽 포트 설정만 역할에 따라 다릅니다.

---

## 사전 요구사항 체크리스트

| 항목 | 요구사항 |
|---|---|
| RAM | 노드당 2GB 이상 (컨트롤 플레인 권장 4GB+) |
| CPU | 컨트롤 플레인 2 vCPU 이상 |
| 네트워크 | 모든 노드 간 완전한 통신 가능 : VMnet8 (NAT) |
| 고유성 | 노드별 hostname, MAC 주소, product_uuid가 모두 고유 |
| 권한 | root 또는 sudo 권한 |
| 커널 | Rocky 10.2 기본 커널(6.12 계열) — kubeadm 지원 범위 |

> 참고: RHEL/Rocky 8의 4.18 커널은 Kubernetes 1.32 이상에서 미지원이지만, Rocky 10.2는 최신 커널이므로 문제없습니다.

## 전체 VM 정보 

| HostName | CPU | RAM | IP Address |
|---|---|---|---|
| bastion | 2 vCPU | 4GB | 10.10.10.100 |   
| k8s-cp01 |  2 vCPU | 4GB | 10.10.10.10 |  
| k8s-worker01 | 2 vCPU | 4GB | 10.10.10.21 |  
| k8s-worker02 | 2 vCPU | 4GB | 10.10.10.22 |  

> 배스천(Bastion) 서버 : 외부의 공격으로부터 내부 네트워크를 보호하고, 관리자가 외부에서 내부 서버로 안전하게 접속할 수 있도록 해주는 보안 게이트웨이(점프 서버)입니다. 클라우드 환경에서는 프라이빗 서브넷에 숨겨진 서버들을 관리하기 위해 주로 사용됩니다.
---

# STEP 0. 노드 고유성 확인 [ALL]

**주의 :** VM을 복제(clone)해서 노드를 만든 경우 product_uuid나 MAC, IP Address가 겹칠 수 있습니다. 클러스터 구성 전에 반드시 확인하세요.

```bash
# MAC 주소 확인
ip link

# IP Address, MAC 주소 확인 
ip addr show     
ip a

# product_uuid 확인 (노드마다 값이 달라야 함)
sudo cat /sys/class/dmi/id/product_uuid
```

모든 노드에서 위 세 값이 서로 달라야 합니다. 같다면 VM을 새로 생성하거나 UUID를 재설정해야 합니다.

---

# STEP 1. 시스템 업데이트 [ALL]

```bash
sudo dnf -y update
```

> 커널이 업데이트된 경우 이후 단계(특히 모듈 로드) 전에 재부팅을 권장합니다.
> `sudo reboot`

---

# STEP 2. 호스트네임 및 호스트 파일 설정 [ALL]

각 노드에서 자신의 호스트네임을 설정합니다. (예시)

```bash
# bastion 노드에서
sudo hostnamectl set-hostname bastion

# 컨트롤 플레인 노드에서
sudo hostnamectl set-hostname k8s-cp01

# 워커 노드에서 (각각)
sudo hostnamectl set-hostname k8s-worker01
sudo hostnamectl set-hostname k8s-worker02
```

**모든 노드**의 `/etc/hosts`에 클러스터 노드 정보를 추가합니다. (IP는 환경에 맞게 변경)

```bash
sudo tee -a /etc/hosts <<EOF
10.10.10.10   k8s-cp01
10.10.10.21   k8s-worker01
10.10.10.22   k8s-worker02
10.10.10.100  bastion
EOF
```

---

# STEP 3. Swap 비활성화 [ALL]

kubeadm은 기본적으로 swap이 꺼져 있어야 합니다.

```bash
# 즉시 비활성화
sudo swapoff -a

# 재부팅 후에도 유지되도록 fstab에서 swap 항목 주석 처리
sudo sed -i.bak '/\bswap\b/s/^/#/' /etc/fstab

# (Rocky에서 zram-generator로 swap이 설정된 경우 추가 처리)
sudo systemctl mask swap.target 2>/dev/null || true

# 확인 (출력이 비어 있어야 함)
swapon --show
free -h
```

---

# STEP 4. SELinux 설정 [ALL]

Rocky 10.2는 SELinux가 기본 enforcing입니다. kubeadm 표준 절차에서는 permissive로 전환합니다.

```bash
# 즉시 permissive 적용
sudo setenforce 0

# 재부팅 후에도 유지
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 확인
getenforce   # Permissive
```

> 운영 환경에서 SELinux를 enforcing으로 유지하려면 별도의 정책 구성이 필요하며, 이는 kubeadm 기본 지원 범위를 벗어납니다.

---

# STEP 5. 방화벽 포트 설정

## 옵션 A. 필요한 포트만 개방 (권장)

### [CP] 컨트롤 플레인 노드

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp        # API 서버
sudo firewall-cmd --permanent --add-port=2379-2380/tcp   # etcd
sudo firewall-cmd --permanent --add-port=10250/tcp       # kubelet
sudo firewall-cmd --permanent --add-port=10257/tcp       # controller-manager
sudo firewall-cmd --permanent --add-port=10259/tcp       # scheduler
sudo firewall-cmd --reload
```

### [WORKER] 워커 노드

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp           # kubelet
sudo firewall-cmd --permanent --add-port=10256/tcp           # kube-proxy
sudo firewall-cmd --permanent --add-port=30000-32767/tcp     # NodePort 범위
sudo firewall-cmd --reload
```

> CNI 플러그인(Calico, Flannel 등)에 따라 추가 포트(예: VXLAN 8472/udp, BGP 179/tcp)가 필요할 수 있습니다. 사용할 CNI 문서를 확인하세요.

## 옵션 B. 실습/랩 환경 — firewalld 비활성화 [ALL]

학습용 격리 환경에서는 간단히 방화벽을 끄기도 합니다. (운영 환경 비권장)

```bash
sudo systemctl disable --now firewalld
```

---

# STEP 6. 커널 모듈 로드 [ALL]

```bash
# 부팅 시 자동 로드되도록 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# 즉시 로드
sudo modprobe overlay
sudo modprobe br_netfilter

# 확인
lsmod | grep -E 'overlay|br_netfilter'
```

---

# STEP 7. 커널 네트워크 파라미터(sysctl) 설정 [ALL]

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 적용
sudo sysctl --system

# 확인 (세 값 모두 1이어야 함)
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

---

# STEP 8. 컨테이너 런타임(containerd) 설치 [ALL]

Rocky 기본 저장소에는 containerd가 없으므로 Docker 공식 저장소를 추가해 설치합니다.

```bash
# Docker 공식 저장소 추가
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# containerd 설치
sudo dnf install -y containerd.io
```

> 만약 el10용 docker-ce 패키지가 아직 제공되지 않아 위 단계가 실패하면, STEP 8-ALT(CRI-O)로 대체하세요.

## STEP 8-1. containerd 설정 (SystemdCgroup 활성화) [ALL]

kubeadm은 cgroup 드라이버로 `systemd`를 권장합니다. 이를 위해 containerd 설정을 변경합니다.

```bash
# 기본 설정 파일 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# SystemdCgroup = true 로 변경
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 적용되었는지 확인 (true 가 출력되어야 함)
grep SystemdCgroup /etc/containerd/config.toml

# containerd 시작 및 부팅 시 자동 실행
sudo systemctl enable --now containerd
sudo systemctl restart containerd

# 상태 확인
sudo systemctl status containerd --no-pager
```

## STEP 8-ALT. (대안) CRI-O 설치 [ALL] , [주의: containerd가 설정되지 않는 경우만 실행]

containerd 대신 CRI-O를 쓰려면 아래를 사용합니다. CRI-O는 Kubernetes 버전과 1:1로 정렬됩니다(v1.36).

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.36/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y cri-o
sudo systemctl enable --now crio
```

---

# STEP 9. Kubernetes 저장소 추가 (v1.36) [ALL]

v1.36 전용 커뮤니티 저장소(pkgs.k8s.io)를 추가합니다. `exclude` 항목은 `dnf update` 시 쿠버네티스 패키지가 임의로 올라가는 것을 막아줍니다.

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

> 저장소는 마이너 버전별로 분리되어 있습니다. 다른 버전을 쓰려면 URL의 `v1.36`을 해당 버전으로 바꾸세요.

---

# STEP 10. kubelet · kubeadm · kubectl 설치 [ALL]

```bash
# exclude를 일시 해제하여 설치
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# kubelet 부팅 시 자동 실행 등록 (cluster init 전까지는 대기 상태로 crashloop이 정상)
sudo systemctl enable --now kubelet
```

> 설치 직후 `kubectl`/`kubeadm`은 사용 가능하지만, `kubelet`은 `kubeadm init`(또는 `join`) 전까지 정상 기동되지 않습니다. 이는 정상 동작입니다.

---

# STEP 11. 설치 전 최종 점검 [ALL]

```bash
# 버전 확인
kubeadm version
kubelet --version
kubectl version --client

# 런타임 소켓 확인 (containerd 사용 시)
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock version 2>/dev/null || \
  ls -l /run/containerd/containerd.sock

# 시스템 상태 요약
echo "== swap ==" ; swapon --show
echo "== selinux ==" ; getenforce
echo "== modules ==" ; lsmod | grep -E 'overlay|br_netfilter'
echo "== sysctl ==" ; sysctl net.ipv4.ip_forward
```

## (선택) [CP] 컨트롤 플레인 이미지 미리 받기

네트워크가 느린 환경에서 `kubeadm init` 시간을 단축하려면 필요한 이미지를 미리 내려받을 수 있습니다.

```bash
sudo kubeadm config images pull --kubernetes-version v1.36.0
```

---

# 셋팅 완료 — 다음 단계 안내

여기까지가 **설치 전 셋팅**입니다. 이후 실제 클러스터 구성은 다음 순서로 진행됩니다. (이 문서의 범위 밖)

1. **[CP]** `kubeadm init` 으로 컨트롤 플레인 초기화
2. **[CP]** kubeconfig 설정 (`$HOME/.kube/config`)
3. **[CP]** CNI 플러그인 설치 (Calico / Flannel 등)
4. **[WORKER]** `kubeadm join` 으로 워커 노드 합류
5. **[CP]** `kubectl get nodes` 로 모든 노드 Ready 확인

---

## 부록 A. 사전 셋팅 요약 체크리스트

- [ ] 노드 product_uuid · MAC 고유성 확인
- [ ] 시스템 업데이트 완료
- [ ] hostname 및 /etc/hosts 설정
- [ ] swap 비활성화 (`swapon --show` 비어 있음)
- [ ] SELinux permissive (`getenforce`)
- [ ] 방화벽 포트 개방 또는 firewalld 비활성화
- [ ] overlay · br_netfilter 모듈 로드
- [ ] sysctl 3개 파라미터 = 1
- [ ] containerd 설치 + SystemdCgroup=true + 실행 중
- [ ] Kubernetes v1.36 저장소 추가
- [ ] kubelet · kubeadm · kubectl 설치 + kubelet enable
- [ ] 버전 및 시스템 상태 최종 점검

## 부록 B. 자주 겪는 문제

| 증상 | 원인 / 조치 |
|---|---|
| `kubeadm init` preflight에서 swap 오류 | STEP 3 재확인, `swapon --show`가 비어야 함 |
| cgroup 드라이버 불일치 경고 | STEP 8-1의 `SystemdCgroup = true` 확인 후 containerd 재시작 |
| 노드 간 통신 불가 | 방화벽 포트(STEP 5)·CNI 포트·/etc/hosts 확인 |
| docker-ce 저장소 el10 패키지 없음 | STEP 8-ALT(CRI-O)로 대체 |
| product_uuid 중복 (VM 복제) | VM 재생성 또는 UUID 재설정 |

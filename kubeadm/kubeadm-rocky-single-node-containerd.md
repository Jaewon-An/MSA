# kubeadm 기반 Kubernetes 단일 노드 설치 (Rocky Linux + containerd)

Rocky Linux 서버 1대에 kubeadm으로 Kubernetes를 설치하는 가이드입니다. 컨테이너 런타임은 **containerd**를 사용하며, 동일 서버를 control-plane 및 worker 노드로 사용합니다.

---

## 사전 요구사항

- Rocky Linux 8.x 또는 9.x (x86_64)
- root 또는 sudo 권한
- 최소 2 CPU, 2GB RAM 권장 (실 서비스 시 4 CPU, 8GB RAM 이상 권장)
- 네트워크 연결 (패키지 다운로드 및 이미지 pull)

---

## 1단계: 시스템 사전 설정

### 1-1. 시스템 업데이트 및 필수 패키지
```bash
# 시스템 업데이트
sudo dnf update -y

# 필수 패키지
sudo dnf install -y curl wget git vim net-tools conntrack-tools
```

### 1-2. 스왑 비활성화 (Kubernetes 권장)
```bash
# 스왑 끄기
sudo swapoff -a

# 부팅 시 스왑 비활성화 (fstab에서 swap 주석 처리)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab

# 확인
free -h
```

**예상:** `Swap` 행의 사용량이 0B로 표시됩니다.

### 1-3. 커널 모듅 로드 및 영구 설정
```bash
# 필요한 모듅 로드
sudo modprobe overlay
sudo modprobe br_netfilter

# 부팅 시 자동 로드
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

### 1-4. 네트워크 설정 (sysctl)
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 1-5. SELinux 설정 (Permissive)
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
getenforce
```

**예상 출력:** `Permissive`

### 1-6. 방화벽 포트 오픈
```bash
sudo firewall-cmd --permanent --add-port=6443/tcp      # Kubernetes API
sudo firewall-cmd --permanent --add-port=2379-2380/tcp # etcd
sudo firewall-cmd --permanent --add-port=10250/tcp      # Kubelet
sudo firewall-cmd --permanent --add-port=10251/tcp      # kube-scheduler
sudo firewall-cmd --permanent --add-port=10252/tcp      # kube-controller-manager
sudo firewall-cmd --permanent --add-port=30000-32767/tcp # NodePort
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

---

## 2단계: containerd 설치 및 설정

### 2-1. containerd 설치
```bash
# DNF 저장소에서 설치 (Rocky Linux)
sudo dnf install -y containerd

# 버전 확인
containerd --version
```

**예상 출력:** `containerd containerd.io 1.6.x ...`

### 2-2. containerd 설정 (systemd cgroup, CRI 활성화)
```bash
# 기본 설정 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# systemd cgroup driver 사용으로 수정
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 서비스 시작 및 활성화
sudo systemctl enable containerd
sudo systemctl start containerd
sudo systemctl status containerd
```

**예상:** `Active: active (running)`

### 2-3. CRI 소켓 확인
```bash
ls -la /run/containerd/containerd.sock
```

**예상:** 소켓 파일이 존재해야 합니다. kubeadm은 이 경로(`unix:///run/containerd/containerd.sock`)를 사용합니다.

---

## 3단계: kubeadm, kubelet, kubectl 설치

### 3-1. Kubernetes 패키지 저장소 추가
```bash
# Kubernetes 공식 저장소 (v1.28 기준, 필요 시 버전 변경)
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
EOF
```

> 다른 버전이 필요하면 `v1.28`을 `v1.29` 등으로 변경하고, [공식 문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)의 저장소 URL을 참고하세요.

### 3-2. kubeadm, kubelet, kubectl 설치
```bash
# 설치 (버전 고정 권장)
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# 부팅 시 kubelet 자동 시작
sudo systemctl enable kubelet
```

### 3-3. 설치 확인
```bash
kubeadm version
kubectl version --client
kubelet --version
```

**예상 출력 예:**
```
kubeadm version: &version.Info{Major:"1", Minor:"28", ...}
Client Version: v1.28.x
Kubernetes v1.28.x
```

---

## 4단계: kubeadm으로 클러스터 초기화 (단일 노드)

### 4-1. control-plane 노드 초기화
```bash
# containerd CRI 소켓 지정하여 초기화
sudo kubeadm init \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --pod-network-cidr=10.244.0.0/16

# Flannel 사용 시 위 pod-network-cidr 사용. Calico 사용 시 아래처럼 할 수 있음:
# sudo kubeadm init --cri-socket=unix:///run/containerd/containerd.sock --pod-network-cidr=192.168.0.0/16
```

**실행 후 출력 끝부분에 다음이 포함됩니다:**
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4-2. kubectl 설정 (일반 사용자)
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4-3. kubelet 시작
```bash
sudo systemctl start kubelet
sudo systemctl status kubelet
```

---

## 5단계: Pod 네트워크(CNI) 설치

단일 노드에서도 Pod 간 통신을 위해 CNI가 필요합니다. Flannel 또는 Calico 중 하나를 설치합니다.

### 5-1. Flannel 설치 (권장: 간단한 구성)
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 5-2. (대안) Calico 설치
```bash
# Flannel 대신 Calico 사용 시 4-1에서 --pod-network-cidr=192.168.0.0/16 사용 후
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
```

### 5-3. CNI Pod 준비 대기
```bash
kubectl get pods -n kube-flannel -w
# 또는 Calico: kubectl get pods -n calico-system -w
# Ctrl+C로 중단 후 아래로 확인
kubectl get pods -A
```

**예상:** `kube-system`, `kube-flannel`(또는 calico) 네임스페이스의 Pod들이 `Running` 상태가 됩니다.

---

## 6단계: 단일 노드에서 워크로드 스케줄 허용 (선택)

control-plane 노드는 기본적으로 taint가 걸려 있어 일반 Pod가 스케줄되지 않습니다. 서버 1대만 사용할 경우 이 taint를 제거하면 같은 노드에 워크로드를 띄울 수 있습니다.

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

**예상 출력:** `node/<노드이름> untainted`

> 제거하지 않으면 control-plane 전용 노드로만 사용하고, worker 노드를 별도로 추가해야 합니다.

---

## 7단계: 동작 확인

### 7-1. 노드 및 시스템 Pod 확인
```bash
kubectl get nodes
kubectl get pods -A
```

**예상 출력 예:**
```
NAME     STATUS   ROLES           AGE   VERSION
rocky1   Ready    control-plane   10m   v1.28.x

NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-xxx                         1/1     Running   0          8m
kube-system   etcd-rocky1                         1/1     Running   0          10m
kube-system   kube-apiserver-rocky1               1/1     Running   0          10m
kube-system   kube-controller-manager-rocky1      1/1     Running   0          10m
kube-system   kube-proxy-xxx                      1/1     Running   0          8m
kube-system   kube-scheduler-rocky1               1/1     Running   0          10m
kube-flannel  kube-flannel-ds-xxx                 1/1     Running   0          5m
```

### 7-2. 테스트 Pod 실행
```bash
kubectl run nginx --image=nginx --restart=Never
kubectl get pods -o wide
kubectl delete pod nginx
```

---

## 요약 및 참고

| 항목 | 내용 |
|------|------|
| OS | Rocky Linux 8/9 |
| 컨테이너 런타임 | containerd |
| 구성 | kubeadm 단일 노드 (control-plane + worker) |
| CNI | Flannel (또는 Calico) |
| CRI 소켓 | `unix:///run/containerd/containerd.sock` |

- **kubeadm 업그레이드:** [공식 업그레이드 가이드](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) 참고  
- **worker 노드 추가:** `kubeadm init` 출력에 있는 `kubeadm join ...` 명령을 worker 서버에서 실행  
- **문제 발생 시:** `journalctl -xeu kubelet`, `crictl ps`(containerd 컨테이너 확인)로 점검

---

## 한 번에 확인하는 명령어

```bash
echo "=== 노드 ==="
kubectl get nodes

echo -e "\n=== 시스템 Pod ==="
kubectl get pods -A

echo -e "\n=== containerd ==="
sudo systemctl is-active containerd

echo -e "\n=== kubelet ==="
sudo systemctl is-active kubelet
```

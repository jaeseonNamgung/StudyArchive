# 설치 환경

- CPU 2core 이상
- 메모리 2GB 이상
- 3대의 host 필요 (master node 1대, worker node 2대)
- 우분투 24.04 LTS OS가 설치된 host 3대
- Mac OS (M2)
- UTM 가상머신

# 도커 설치

```bash
#  패키지 목록 업데이트 & 필수 패키지 설치
sudo apt-get update
sudo apt-get install ca-certificates curl
# GPG 키 저장소 생성 및 Docker GPG 키 추가
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Docker 저장소를 APT 소스 목록에 추가
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt-get update
```

```bash
# 도커 설치
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
# 도커 소켓 권한 설정
sudo chmod 666 /var/run/docker.sock
# 현재 사용자를 Docker 그룹에 추가
sudo usermod -aG docker $USER
```

# 쿠버네티스 설치

## 쿠버네티스 런타임 구성

> Kubernetes v1.23.x와 v1.24.x 사이에는 큰 차이점이 존재한다. Kubernetes v1.24.0부터는 docker를 버렸다는 것이다. 공식 문서의 내용을 간단히 요약하면 다음과 같다.

- 1.24 버전 이전에는 docker라는 specific한 CRI를 사용하고 있었다.
- Kubernetes에서 docker 외에도 다양한 CRI를 지원하기 위해, CRI standard라는 것을 만들었다.
- Docker는 CRI standard를 따르지 않고 있다.
- Kubernetes는 docker 지원을 위해 dockershim이라는 코드를 만들어서 제공했다.
- Kubernetes 개발 참여자들이 docker라는 특수 CRI를 위해 별도로 시간을 할애하는 것이 부담스럽다.
- Kubernetes v1.24부터 dockershim 지원 안하기로 했다.

쿠버네티스 공식 문서: https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/
참고 블로그:
> 

1. 메모리 스왑 비활성화

```bash
sudo swapoff -a && sudo sed -i '/swap/s/^/#/' /etc/fstab
```

1. 방화벽 비활성화 [방화벽이 켜져있는 경우]

```bash
# firewalld 설치 필요 
sudo apt-get install -y firewalld

sudo systemctl stop firewalld && sudo systemctl disable firewalld

```

1. 노드 간 통신을 위한 iptables 설정

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 필요한 sysctl 파라미터를 설정하면, 재부팅 후에도 값이 유지된다.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 재부팅하지 않고 sysctl 파라미터 적용하기
sudo sysctl --system
```

1. Containerd 설정

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

- containerd config default: 기본 설정을 출력하는 명령어
- `| sudo tee /etc/containerd/config.toml`: 해당 출력을 `/etc/containerd/config.toml` 파일에 저장
- tee 명령어를 사용하면 출력을 파일에 저장하면서 동시에 화면에도 표시할 수 있다.
1. Cgroup 설정 (systemd 사용)

> 리눅스에서, [control group](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-cgroup)은 프로세스에 할당된 리소스를 제한하는데 사용된다.
[kubelet](https://kubernetes.io/ko/docs/reference/generated/kubelet)과 그에 연계된 컨테이너 런타임 모두 컨트롤 그룹(control group)들과 상호작용 해야 하는데, 이는 [파드 및 컨테이너 자원 관리](https://kubernetes.io/ko/docs/concepts/configuration/manage-resources-containers/)가 수정될 수 있도록 하고 cpu 혹은 메모리와 같은 자원의 요청(request)과 상한(limit)을 설정하기 위함이다. 컨트롤 그룹과 상호작용하기 위해서는, kubelet과 컨테이너 런타임이 *cgroup 드라이버*를 사용해야 한다. 매우 중요한 점은, kubelet과 컨테이너 런타임이 같은 cgroup group 드라이버를 사용해야 하며 구성도 동일해야 한다는 것이다.

두 가지의 cgroup 드라이버가 이용 가능하다.
- [`cgroupfs`](https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/#cgroupfs-cgroup-driver)
- [`systemd`](https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/#systemd-cgroup-driver) 

참고: 쿠버네티스 공식 문서
> 
- `/etc/containerd/config.toml` 파일에서 SystemdCgroup을 true로 변경
- 주의사항:

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

1. Containerd 재시작

```bash
sudo systemctl restart containerd
# containerd 상태 확인
sudo systemctl status containerd
```

## **kubeadm으로 클러스터 구성**

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

1. Google Public Key 다운로드

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

1. Kubernetes 패키지 저장소 URL 추가

```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo apt-get update
sudo apt-get install -y kubelet=1.32.1-1.1 kubeadm=1.32.1-1.1  kubectl=1.32.1-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

## Kubernetes 클러스터 초기화 (Master Node 에서만 실행!)

1. kubeadm 초기화

```bash
sudo kubeadm init --apiserver-advertise-address 192.168.64.20  --pod-network-cidr=192.168.0.0/16
```

- `—apiserver-advertise-address` : 마스터노드의 IP
- `--pod-network-cidr=` : 설치하고자 하는 CRI의 IP주소 대역
    - Calico: 192.168.0.0/16
1. kubectl 권한 설정

```bash

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

1. Pod CNI 애드온 설치 (Calico)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
```

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

# 자동 완성 설정

```bash
source <(kubectl completion bash) 
echo "source <(kubectl completion bash)" >> ~/.bashrc 
```

```bash
alias k=kubectl
complete -o default -F __start_kubectl k
```

## 워커 노드 구성

- Master Node를 초기화 후 출력된 `kubeadm join` 명령어를 사용하여 워커 노드를 구성

```bash
sudo kubeadm join 192.168.64.20:6443 --token 3s23ar.yhfowtvgyj88x5o7 \
	--discovery-token-ca-cert-hash sha256:111654d61fa28e3812a8a18b2e8f8f606b108ceeebcca45e6ff89f6d639cb813
```

## 각종 오류 상항

[Kubernetes 저장소 오류]

```bash
E: The repository 'https://pkgs.k8s.io/core:/stable:/v1.24/deb  InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

- GPG 키 다시 추가

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key |
sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.asc >/dev/null
```

- 설치된 kubernetes.list 파일을 제거하고 다시 알맞은 Kubernetes 저장소 다시 설치

```bash
sudo rm -rf /etc/apt/sources.list.d/kubernetes.list
```

[일반적인 오류와 해결 방법]

- **오류:** `container runtime is not running`

```
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
```

- 권한 오류

```json
I0209 08:14:08.141011    3771 version.go:256] remote version is much newer: v1.32.1; falling back to: stable-1.30
[init] Using Kubernetes version: v1.30.9
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR IsPrivilegedUser]: **user is not running as root**
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

root 권한이 아니라서 생기는 오류 `sudo` 를 사용하면 된다.

- `kubeadm init` 오류

```json
I0209 08:14:35.166408    3952 version.go:256] remote version is much newer: v1.32.1; falling back to: stable-1.30
[init] Using Kubernetes version: v1.30.9
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR Port-6443]: Port 6443 is in use
	[ERROR Port-10259]: Port 10259 is in use
	[ERROR Port-10257]: Port 10257 is in use
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml **already exists**
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml **already exists**
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml **already exists**
	[ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml **already exists**
	[ERROR Port-10250]: Port 10250 is in use
	[ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

이 오류는 이미 `kubeadm init` 을 했기 때문에 yaml 파일이 존재해서 생기는 오류이다. 아래와 같이 기존에 존재한 yaml 파일을 제거해준 후 다시 초기화 해주면 된다.

```bash
# kubeadm 초기화
sudo kubeadm reset -f
```

### 범외 Minikube 시작 예시

```yaml
minikube start --driver='docker' --profile='thred-node' --cni='calico' --kubernetes-version='stable' --nodes=3
```
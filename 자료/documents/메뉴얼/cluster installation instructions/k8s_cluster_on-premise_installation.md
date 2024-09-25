# Kubernetes 클러스터 구축

ubuntu 22.04 server 버전의 온프레미스 환경에서 k8s 클러스터를 구축하기 위한 단계별 방법을 기록한다.

## 초기 네트워크 환경 설정

1. hostname 설정

`/etc/hosts` 파일을 편집해서 노드로 사용할 각각의 컴퓨터들의 hostname을 기록해두어야 할 필요가 있다.

이를 통해 클러스터에서 각각의 노드를 ip가 아니라 hostname으로 찾을 수 있다.

예시를 들어서, ip가 `192.168.0.100`인 컴퓨터를 master 노드로 사용하고, ip 주소가 각각 `192.168.0.101`, `192.168.0.102`인 컴퓨터 2대를 worker 노드로 사용하려고 한다고 가정하자.

그렇다면 `/etc/hosts`파일에 다음과 같은 내용을 추가해야한다.

```config
...
192.168.0.100 com0
192.168.0.101 com1
192.168.0.102 com2
```

위에서 추가한 내용을 통해, `com0`, `com1`, `com2`라는 hostname으로 각각의 노드에 접근할 수 있다.

만약 hostname을 변경할 필요가 있을 경우 다음의 명령어를 통해 hostname을 재설정할 수 있다.

```bash
sudo hostnamectl set-hostname HOSTNAME_TO_CHANGE
```

이때 `HOSTNAME_TO_CHANGE`는 변경하길 원하는 hostname으로 대체하면 된다.

현재 세션을 종료한 다음 다시 로그인하면 변경된다.

2. ssh 연결

기본적으로 클러스터를 구축하고자 한다면 클러스터를 구성하는 노드들이 다른 노드에 ssh를 통해 접근할 때 비밀번호 또는 로그인을 요구하지 않아야 할 필요가 있다.

다음의 명령어를 통해 ssh 키를 새롭게 생성한다.

이때 passphrase는 없이 생성한다.

```bash
ssh-keygen
```

명령어를 실행한다면 default로 home 디렉토리 내부의 `.ssh/` 디렉토리 아래에 `id_rsa`와 `id_rsa.pub`라는 이름의 private/public ssh 키가 생성된다.

추가적으로 다음의 명령어를 통해 비밀번호 없이 로그인이 가능하도록 설정한다.

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

위 명령어를 통해 `~/.ssh` 디렉토리의 아래에 `authorized_keys` 파일에 `id_rsa.pub` 파일의 내용을 추가해 같은 `id_rsa.pub` 파일을 가지고 있다면 비밀번호없이 ssh로 접근 가능하도록 할 수 있다.

`id_rsa`, `id_rsa.pub`, `authorized_keys` 총 3개의 파일을 master 노드에서 worker 노드로 삼을 컴퓨터들에 다음 명령어를 통해 복제해준다.

```bash
cd ~/.ssh && scp id_rsa id_rsa.pub authorized_keys user@com1:/home/user/.ssh/
```

각 노드의 유저이름에 알맞게 `user`라는 이름을 변경해준다.

추가적으로, ssh key 파일들은 적절한 권한이 설정되어야 ssh를 통해 접속할 때 문제가 발생하지 않는다.

각 노드에서 ssh를 통해 접속해서 또는 직접적으로 터미널에서 다음 명령어를 실행해준다.

```bash
chmod 700 /home/user/.ssh
chmod 600 /home/user/.ssh/id_rsa
chmod 644 /home/user/.ssh/id_rsa.pub
chmod 600 /home/user/.ssh/authorized_keys
```

## 스왑 메모리 비활성화

swap 메모리는 OS가 RAM에 올려놓은 메모리 중 일부를 HDD 혹은 SSD에 옮겨 총 가용 메모리의 용량을 늘리는 방식이다.

이 방식의 장점은 실제 RAM의 용량보다 더 큰 메모리를 확보할 수 있다는 점이지만, 동시에 RAM이 아닌 디스크에 올려놓은 데이터를 로딩하는 경우의 오버헤드가 커져서 전체적인 속도가 느려진다는 단점 또한 존재한다.

k8s 클러스터를 구축하기 위해서는 swap 메모리를 비활성화하는 것이 필요하다.

k8s의 스케줄러는 각 pod에 할당된 cpu 및 메모리 자원을 정확하게 관리할 필요가 있는데, 만약 swap 메모리가 활성화되어 있다면 스케줄러가 정확한 메모리 잔여량을 알지 못하고 잘못된 양을 할당하거나 노드의 성능이 저하될 가능성이 있다.

swap 메모리를 비활성화하기 위해서는 다음의 명령어를 실행한다.

```bash
sudo swapoff --all
```

그 후 다음의 명령어를 통해 swap 메모리의 활성화 여부를 확인할 수 있다.

```bash
free -h
```

명령어를 실행한 결과 swap 메모리의 크기가 0이면 정상적으로 비활성화된 것이다.

다만 위 명령어를 통해서 swap 메모리를 비활성화하는 것은 재부팅시 초기화되므로 설정 파일을 수정해 영구적으로 swap 메모리의 크기를 0으로 설정할 필요가 있다.

이를 위해 다음의 명령어를 실행한다.

```bash
sudo vim /etc/fstab
```

위 명령어를 통해 `/etc/fstab` 파일을 열어 `swap.img`라는 이름이 적힌 라인을 주석으로 처리해주면 된다.

```bash
...
#/swap.img      none    swap    sw      0       0
...
```

설정 파일도 저장했으면 재부팅 이후에도 swap 메모리가 0으로 유지된다.

## containerd 설치

**Kubernetes**는 컨테이너화된 애플리케이션의 배포, 관리 및 확장을 예약하고 자동화하기 위한 *컨테이너 오케스트레이션 플랫폼*이다.

일반적으로 단어가 길기 때문에 k와 s 사이에 8개의 글자가 있다는 점을 살려 **k8s**라고 불린다.

컨테이너의 생성, 실행, 관리 및 종료를 담당하는 소프트웨어를 `컨테이너 런타임`이라고 지칭한다.

`CRI(Container Runtime Interface)`는 k8s가 컨테이너 런타임과 상호작용하여 컨테이너를 생성하거나 제거하기 위한 인터페이스이다.

`CRI` 개념에는 컨테이너의 생성과 삭제, 상태 조회와 배포, 모니터링, 로그 수집 등 다양한 기능이 포함된다.

containerd는 `CRI` 표준을 직접적으로 구현한 런타임 인터페이스이다.

다음 명령어를 통해 containerd를 설치할 수 있다.

```bash
sudo apt update && sudo apt install containerd.io -y
```

## 네트워크 모듈 설정

### overlay, br_netfilter

linux 커널에서 제공하는 네트워크 드라이버 중 `overlay` 네트워크 드라이버는 여러 개의 독립적인 네트워크를 겹쳐서 하나의 연결된 네트워크를 생성할 수 있게 해주는 드라이버다.

k8s에서 `overlay`를 통해 물리적 네트워크 위에 가상 네트워크를 겹쳐 다양한 네임스페이스 간의 통신을 지원하고 네트워크 격리를 구현할 수 있다.

`br_netfilter` 모듈은 Linux 커널에서 브리지된 네트워크 트래픽을 iptables와 같은 네트워크 필터링 툴로 처리하기 위한 모듈이다.

k8s에서 `br_netfilter`는 클러스터 노드 간의 네트워크 트래픽을 관리하고 보안 규칙을 적용하기 위해 사용된다.

### 모듈 설치

다음 명령어를 통해 `/etc/modules-load.d/` 디렉토리 아래에 `k8s.conf`를 작성해 앞서 설명한 네트워크 모듈들을 사용하도록 설정할 수 있다.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

다음 명령어를 통해 `overlay`, `br_netfilter` 모듈을 메모리에 로드할 수 있다.

```bash
sudo modprobe overlay && sudo modprobe br_netfilter
```

모듈들이 정상적으로 설치되었는지 확인하려면 각각 다음의 명령어를 통해 메모리에 로드된 모듈의 목록에서 검색하는 방법으로 확인할 수 있다.

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

### 네트워크 트래픽 규칙 추가

그 후 추가적으로 다음 명령어를 통해 k8s 클러스터에서 ipv4 또는 ipv6 트래픽이 `iptable` 규칙으로 처리될 수 있도록 한다.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables   = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward  = 1
EOF
```

위의 규칙들을 모두 작성한 이후 다음 명령어를 통해 `sysctl` 규칙들이 커널에 적용되도록 한다.

```bash
sudo sysctl --system
```

## containerd 설정

k8s는 기본적으로 현대 리눅스에서 시스템 및 서비스 관리 데몬으로 일반적으로 활용되는 `systemd`를 Cgroup 관리자로 사용하는 것을 선호한다.

그렇기 때문에 k8s에서 `containerd`를 기본적인 `CRI`로 사용할 때는 호환성을 높이고자 `SystemdCgroup` 변수를 true로 잡아 `containerd`에서 Cgroup을 관리할 때 `systemd` 데몬을 사용하도록 설정하는 것이 공식에서 권장되는 설정이다.

만약 해당 변수가 false 상태라면 `containerd`가 Cgroup을 관리할 때 `cgroupfs`를 사용하게 되는데, 이 경우 `kubelet`이 `systemd`를 사용하기 때문에 충돌이 발생할 가능성이 있다.

다음 명령어를 통해 `containerd`에 적용하기 위한 설정 파일을 덤핑할 수 있다.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

그 후 해당 설정 파일에서 `SystemdCgroup` 변수의 값을 false에서 true로 바꿔줄 필요가 있다.

```bash
vim /etc/containerd/config.toml
```

```toml
    ...
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            ...
            SystemdCgroup=true
    ...
```

값을 변경해준 이후 해당 설정을 적용해주기 위해 `containerd`를 재시작한다.

```bash
sudo systemctl restart containerd.service
sudo systemctl status containerd.service
```

## kubernetes 설치

k8s를 구성하는 기본적인 패키지들로는 `kubeadm`, `kubelet`, `kubectl`이 있다.

`kubeadm`은 k8s의 클러스터 초기 설정과 부트스트랩 과정을 간편하게 하기 위한 관리 및 설정 툴이다.

`kubelet`은 k8s 클러스터에서 각 노드의 핵심 에이전트로, pod의 실행, 모니터링, 보고 등을 수행해 컨테이너의 라이프사이클을 관리한다.

`kubectl`은 k8s와 개발자 또는 관리자가 직접적으로 상호작용하기 위한 cli 툴로, 클러스터의 상태를 조회하고 리소스를 관리하는데 사용된다.

위 패키지들은 기본적인 apt 미러 사이트 목록에서 설치할 수 없기 때문에 추가적으로 외부에서 apt repository를 목록에 추가해준 다음에 설치하는 과정이 필요하다.

다음 명령어를 통해 외부 apt repository를 끌어오기 위한 설치를 진행해준다.

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl
```

다음 명령어를 통해 kubernetes 패키지 repository에 접근하기 위한 public signing 키를 다운로드한다.

동일한 signing key가 모든 k8s repository에 접근하기 위해 사용되기 때문에 key를 다운로드할 때 version은 크게 중요하지 않다.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.26/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

적절한 k8s 버전의 apt repository를 추가해준다.

현재 구성에서 사용한 k8s의 버전은 1.26이다.

다음 명령어를 통해 apt를 등록하되, 해당 명령어는 `/etc/apt/sources.list.d/kubernetes.list`파일을 덮어씌운다는 점을 유의해야 한다.

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.26/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

apt 목록에 k8s를 설치하기 위해 필요한 패키지 목록이 포함된 repository가 추가되었기 때문에 다음 명령어를 통해 apt를 업데이트하고 필요한 패키지들을 설치한다.

```bash
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

설치가 완료되었다면 다음 명령어를 통해 설치된 패키지들의 버전이 동일한지 확인할 수 있다.

```bash
sudo kubelet --version
sudo kubeadm version
sudo kubectl version --output=yaml
```

## 필요 이미지 pull

k8s 클러스터를 구성하기 위한 모든 패키지들이 설치되었으면 master 노드에서 다음 명령어를 진행해 클러스터를 구성하기 위해 필요한 초기 이미지들을 pull해 올 수 있다.

```bash
sudo kubeadm certs check-expiration && sudo kubeadm config images list && sudo kubeadm config images pull
sudo kubeadm config images pull --cri-socket /run/containerd/containerd.sock
```

## master 노드 초기화

모든 이미지들을 pull하는데 성공했다면, master 노드에서 다음 명령어를 실행하는 것으로 노드 초기화를 진행할 수 있다.

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.x.y --pod-network-cidr=192.168.0.0/16 --cri-socket /run/containerd/containerd.sock
```

이때 위의 명령어에서 ip 주소를 master 노드의 주소값으로 변경해 명령어를 실행하면 된다.

초기화를 완료하게 된다면 여러 결과 메시지가 표시되는데 개중 중요한 것들을 표시하자면 다음과 같다.

1. home 디렉토리 아래에 `.kube/`디렉토리 생성

결과 메시지로도 동일하게 표시되겠지만, 다음의 명령어를 통해 `/etc/kubernetes/admin.conf` 파일을 home 디렉토리 아래의 `.kube/config` 파일로도 복사해줘야 클러스터가 설정 파일을 인식한다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

2. 클러스터 참여 명령어

앞서 [k8s 기본 패키지 설치](#kubernetes-설치)단계까지 모든 worker 노드들에서 동일하게 진행해줘야 `kubeadm`을 통해 클러스터에 참여할 수 있다.

마스터 노드의 초기화 이후에 표시되는 메시지에서 다음과 같은 형식의 명령어가 표시된다.

```bash
sudo kubeadm join 192.168.x.y:6443 --token TOKEN --discovery-token-ca-cert-hash sha256:HASH
```

worker 노드에서 위 명령어를 실행하는 것으로 클러스터에 참여할 수 있다.

주의할 점은 master 노드에서 ufw를 통해 6443 포트를 열어둬야 한다는 점이다.

따라서 추가적으로 다음의 명령어로 실행해줄 필요가 있다.

```bash
sudo ufw allow 6443/tcp
```

## CNI 구축

k8s 클러스터를 구축했다고 해도 단순히 control-plane에서의 구축이 끝난 것이지, 실제로 각 노드의 pod들이 통신하기 위해서는 추가적으로 설치가 필요한 것이 있다.

`CNI(Container Network Interface)`는 컨테이너화된 애플리케이션의 네트워킹을 표준화하기 위해 제공되는 인터페이스로, 네트워크 설정을 다양한 플러그인으로 컨테이너의 네트워크를 설정할 수 있다.

여러 네트워크 플러그인 중 `flannel`은 기본적으로 L2 오버레이 네트워크를 사용하여 컨테이너 간의 통신을 구현한다.

`VXLAN` 또는 다른 오버레이 프로토콜을 사용하여 물리적 네트워크 인프라와 독립적으로 가상 네트워크를 설정하며, 물리적 네트워크의 제약을 받지 않지만 네트워킹 성능에 영향을 미칠 수 있다.

또 다른 `CNI` 플러그인 `calico`는 주로 L3 네트워크를 사용하여 Pod 간의 통신을 설정한다.

`BGP(Border Gateway Protocol)`를 사용하여 라우팅을 수행하고, 네이티브 IP 네트워크를 설정하여 IP 레벨에서 라우팅을 수행하기 때문에 네트워크 트래픽이 물리적 네트워크 인프라를 직접 통과하여 오버헤드가 줄어든다는 장점이 있다.

여러 `CNI`들이 있지만 그 중에서 `calico`를 사용해 구축하겠다.

현재 작성되는 명령어는 k8s v1.26을 기준으로 하기 때문에, `calico` 또한 v3.26.3을 기준으로 작성되었다.

(추후 업데이트가 필요할 수 있음.)

다음 명령어를 실행해 `calico` 프로젝트의 공식 Github로부터 다운로드한다.

```bash
git clone -b v3.26.3 https://github.com/projectcalico/calico.git
```

명령어를 실행하고 나면 실행한 위치에 `calico/`라는 이름의 디렉토리가 생긴다.

동일한 위치에서 다음 명령어를 실행한다.

```bash
kubectl create -f ./calico/manifests/tigera-operator.yaml
kubectl create -f ./calico/manifests/custom-resources.yaml
```

실행된 명령어를 통해 클러스터에 `tigera-operator`의 설정을 적용할 수 있다.

`tigera-operator`는 `calico`의 설치 및 관리를 자동화하는 k8s의 오퍼레이터이다.

오퍼레이터는 k8s 클러스터에서 리소스를 관리하고 유지보수하는 자동화된 방식으로, 복잡한 배포 및 운영 작업을 단순화할 수 있다는 장점이 있다.

설정을 클러스터에 적용하고 나면 다음 명령어를 통해 pod들이 실행에 성공하는지 여부를 확인할 수 있다.

```bash
kubectl get pods -n calico-system
```

추가적으로, ubuntu 22.04 server 버전에서 발생하는 버그 중에 하나로 클러스터에서 소켓을 생성하기 위한 권한 일부를 `apparmor`에서 막아서 생성이 안되면서 `calico`의 pod들이 pending 상태에서 멈춰있는 경우가 있다.

이 경우, 다음 명령어를 통해 `apparmor`를 정지하고 실행하면 제대로 running 상태로 전환되는 것을 확인할 수 있다.

```bash
sudo systemctl stop apparmor && sudo systemctl disable apparmor
```

## helm 설치

`helm`은 Kubernetes 애플리케이션의 배포 및 관리를 간소화하기 위해 설계된 패키지 관리 도구이다.

`helm`은 애플리케이션의 구성, 배포, 업그레이드 및 관리 작업을 자동화하는 데 유용하며, `차트(Charts)`를 사용하여 Kubernetes 리소스를 정의하고 관리한다.

다음 명령어를 통해 `helm`을 설치할 수 있다.

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 && chmod 700 get_helm.sh
./get_helm.sh
```

shell 코드로 설치가 완료되었다면 다음 명령어를 통해 `helm`의 버전을 확인할 수 있다.

```bash
helm version
```

## metallb 설치

클라우드 환경이 아니라 온프레미스 또는 프라이빗 네트워크 환경에서는 로드밸런싱 기능이 k8s 클러스터에 기본적으로 제공되지 않는다.

그렇기 때문에 loadbalancer 서비스를 정의하더라도 실제 외부 ip 주소값이 할당되지 않고 계속 pending 상태에 남아있게 된다.

`metallb`는 Kubernetes 클러스터에서 로드 밸런서 기능을 제공하는 네트워크 솔루션이다.

`metallb`는 `helm`을 통해 간단하게 설치할 수 있다.

다음 명령어를 통해 `metallb` 공식 repo를 `helm`의 repo 목록에 추가할 수 있다.

```bash
helm repo add metallb https://metallb.github.io/metallb && helm repo update
```

추가된 `metallb`를 설치하기 위한 namespace를 클러스터에 다음 명령어를 통해 생성할 수 있다.

```bash
kubectl create namespace metallb
```

`metallb`라는 이름의 namespace 내부에 `helm`을 통해 `metallb`를 다음 명령어를 통해 설치할 수 있다.

```bash
helm install metallb metallb/metallb --namespace metallb
```

그 후, k8s 클러스터에 `metallb-config.yaml`이라는 이름으로 다음과 같은 내용의 yaml을 작성해 적용한다.

```bash
vim metallb-config.yaml
```

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb-addrpool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.100-192.168.0.120
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - metallb-addrpool
```

```bash
kubectl apply -f metallb-config.yaml
```

위 `metallb-config.yaml`까지 클러스터에 잘 적용되었다면 k8s 클러스터에 로드밸런싱 기능이 구현된 것이다.
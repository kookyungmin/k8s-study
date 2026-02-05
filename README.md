# Kubernetes Study

## 테스트 환경

VM (Ubuntu 24.04) Node 4개 (Master 1, Worker 3)

- k8s-master
- k8s-node1
- k8s-node2
- k8s-node3

## VM 환경 구성

CPU: 4 Core

Memory: 4096MB

Disk: 100GB

![img_1.png](img/img_1.png)

내부 네트워크

![img.png](img/img2.png)

외부 네트워크

![img.png](img/img3.png)

IP : 192.168.56.100 ~ 102

Network Mask : 255.255.255.0

Gateway : 192.168.56.1

![img_2.png](img/img_2.png)

## k8s 를 위한 환경 구성

유용한 패키지 설치

```
$ sudo apt -y update
$ sudo apt -y install openssh-server vim tree htop
```

테스트를 위해 방화벽 해제

```
$ sudo ufw disable
```

Pod 가 Swap 을 사용하지 않도록 하여 성능 유지를 위해 swap 해제

```
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

chrony 를 통해 시간 동기화

```
$ sudo apt install chrony
$ chronyc sources -v

# NTP 서버 추가
$ sudo vi /etc/chrony/chrony.conf
server time.google.com iburst
server time.cloudflare.com iburst

# DNS 서버 추가
$ sudo resolvectl dns enp0s8 8.8.8.8 1.1.1.1
$ sudo resolvectl domain enp0s8 "~."

$ sudo systemctl restart chrony
$ sudo chronyc makestep
```

네트워크 패킷을 올바르게 포워딩하기 위해 커널에서 IP 포워딩 활성화

```
$ sudo -i
$ sudo echo '1' > /proc/sys/net/ipv4/ip_forward
```

containerd 를 이용한 container runtime 구성

```
$ sudo cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

```
# modprobe 프로그램은 요청된 모듈이 동작할 수 있도록,
부수적인 모듈을 depmod 프로그램을 이용하여 검색해 필요한 모듈을 커널에 차례로 등록한다.
$ sudo modprobe overlay
$ sudo modprobe br_netfilter
```

노드 간 통신을 위한 iptables 에 브릿지 관련 설정 추가

```
$ sudo cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

$ sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

$ sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

$ sudo sysctl --system
```

k8s runtime 준비 -> containerd 사용

```
# apt 가 https로 리포지터리를 사용하는 것을 허용하기 위한 패키지 및 docker 에 필요한 패키지 설치

$ sudo apt-get update && sudo apt-get install -y \
apt-transport-https ca-certificates curl gnupg2 software-properties-common
```

도커 공식 GPG 키 추가

```
$ sudo install -m 0755 -d /etc/apt/keyrings

$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
 | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
 
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg

# arm64 버전 (amd64 인 경우 변경)
$ echo \
"deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$ sudo apt-get update
```

도커 관련 패키지 추가

```
$ sudo apt-get -y install docker-ce docker-ce-cli \
containerd.io docker-buildx-plugin docker-compose-plugin
```

Container.d 관련 설정

```
$ sudo sh -c "containerd config default > /etc/containerd/config.toml"
$ sudo vi /etc/containerd/config.toml (disabled 된 plugin 이 있는지 확인 -> 없어야함)
$ sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/g' /etc/containerd/config.toml
$ sudo systemctl restart containerd
```

docker daemon 설정

```
$ sudo vi /etc/docker/daemon.json
{ 
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m"
    },
    "storage-driver": "overlay2"
}

$ sudo mkdir -p /etc/systemd/system/docker.service.d
$ sudo usermod -aG docker $USER
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
$ sudo systemctl restart containerd
$ sudo reboot

$ docker info (Cgroup Driver 가 systemd 로 변경되었는지 확인)
...
 Cgroup Driver: systemd
```

k8s 도구 설치 (1.28.15)

```
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

$ sudo apt update

$ sudo apt -y install kubelet kubeadm kubectl

# 신규 version으로 자동 업데이트를 방지하기 위해 hold
$ sudo apt-mark hold kubelet kubeadm kubectl

$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet
$ sudo systemctl enable --now kubelet
```

VM 복제 전 ip 및 host 미리 셋팅

```
$ sudo vi /etc/hosts

...
127.0.1.1 k8s-master
127.0.1.100 k8s-master
127.0.1.101 k8s-node1
127.0.1.102 k8s-node2
127.0.1.103 k8s-node3
```

shutdown 후 VM 복제 -> 상호간 ssh 설정

```
$ sudo shutdown -h now

# 부팅 후 ip 변경, hostname 변경, /etc/hosts 변경
# 상호 간 ssh 접속

$ ssh $USER@k8s-node1
$ ssh $USER@k8s-node2
```

![img_3.png](img/img_3.png)


## 3-node Cluster 환경에서 Kubernetes 초기화

Pod Network IP 주소 범위를 CIDR 로 지정해서 kubenetes init

```
$ sudo kubeadm init --pod-network-cidr=10.96.0.0/12 \
--apiserver-advertise-address=192.168.56.100
```

kubernetes init 성공 후 안내한 방법대로 다음 명령어 실행

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

각 노드에 다음 명령 실행

```
$ sudo kubeadm join 192.168.56.100:6443 --token xblxtf.7bf0jks48dgjq8ho \
	--discovery-token-ca-cert-hash sha256:018d776e93ba3f2ee523abfdd8f271a8d615ed4c322fb18e237fc8d1a1466524
```

kubectl 자동완성 기능 설치

```
$ sudo apt-get install bash-completion -y
$ source <(kubectl completion bash)
$ echo "source <(kubectl completion bash)" >> ~/.bashrc
$ complete -F __start_kubectl k
$ source ~/.bashrc
```

kubernetes 관련 서비스 포트 열렸는지 확인

```
$ sudo netstat -ntlp
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      12338/kubelet
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      12425/kube-proxy
tcp        0      0 127.0.0.1:10259         0.0.0.0:*               LISTEN      12121/kube-schedule
tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      12153/kube-controll
tcp        0      0 127.0.0.1:41991         0.0.0.0:*               LISTEN      1391/containerd
tcp6       0      0 :::6443                 :::*                    LISTEN      12145/kube-apiserve
tcp6       0      0 :::10256                :::*                    LISTEN      12425/kube-proxy
tcp6       0      0 :::10250                :::*                    LISTEN      12338/kubelet
```

Node들이 Cluster에 제대로 연결 되었는지 확인

```
$ kubectl get node
NAME         STATUS     ROLES           AGE   VERSION
k8s-master   NotReady   control-plane   16m   v1.28.15
k8s-node1    NotReady   <none>          62s   v1.28.15
k8s-node2    NotReady   <none>          42s   v1.28.15

# coredns 제외하고 모두 Running 상태여야 함
$ kubectl get pods -A
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-5dd5756b68-fbz5m             0/1     Pending   0          18m
kube-system   coredns-5dd5756b68-rd4tf             0/1     Pending   0          18m
kube-system   etcd-k8s-master                      1/1     Running   0          18m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          18m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          18m
kube-system   kube-proxy-p8wfc                     1/1     Running   0          18m
kube-system   kube-proxy-qhmzg                     1/1     Running   0          2m48s
kube-system   kube-proxy-zrww8                     1/1     Running   0          2m28s
kube-system   kube-scheduler-k8s-master            1/1     Running   0          18m
```

CNI(Container Network Interface) 변경 (Calico 사용)

```
# CNI는 컨테이너(Pod) 간의 네트워킹을 제어할 수 있는 Plugin 을 만들기 위한 표준
# CNI는 클러스터 환경의 Pod 들의 IP를 중복되지 않게 할당하고, 해당 Pod 들이 어느 노드에 있는지 알게 해줌
# k8s 는 기본적으로 kubenet 이라는 자체 CNI를 제공하지만 기능이 제한적이라 Calico 로 변경

$ curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
$ kubectl apply -f calico.yaml
$ kubectl get po -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-658d97c59c-dsf28   1/1     Running   0          45s
kube-system   calico-node-88f8h                          1/1     Running   0          45s
kube-system   calico-node-m67k9                          1/1     Running   0          45s
kube-system   calico-node-zv4qg                          1/1     Running   0          45s
```

## k8s node 확장

```
# 새로운 token 생성
$ kubeadm token create --print-join-command

join 명령 새로운 노드에 실행
$ sudo kubeadm join 192.168.56.100:6443 --token xblxtf.7bf0jks48dgjq8ho \
	--discovery-token-ca-cert-hash sha256:018d776e93ba3f2ee523abfdd8f271a8d615ed4c322fb18e237fc8d1a1466524
```





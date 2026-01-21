---
title: Kubernetes 설치방법
author: G.G
date: 2024-12-30 08:07:00 +0900
categories: [Blog, Orchestration]
tags: [Kubernetes, K8s]
---

## 📘 개요

**Kubernetes 설치 매뉴얼**
Kubernetes는 분산 시스템에서 컨테이너 애플리케이션을 자동으로 배포, 관리, 확장하는 데 널리 사용되는 강력한 오픈소스 도구입니다. 이 매뉴얼은 컨테이너 오케스트레이션 플랫폼인 **Kubernetes(K8s)**를 설치하고 구성하는 데 필요한 모든 과정을 단계별로 안내합니다.


## 환경구성

### 사용자 계정 생성
사용자 계정 생성 후 운영하고 싶은 경우 사용자 계정을 생성합니다.

```bash
#!/bin/bash

# --- 설정 변수 ---
username="k8s"
password="1234"
comment="Kubernetes User"

# 1. 사용자 생성
echo "Creating user: $username..."
sudo useradd -m "$username" \
  -d "/home/$username" \
  -p "$(openssl passwd -6 "$password")" \
  -s /bin/bash \
  -c "$comment"

# 2. sudoers 설정 파일 생성 (/etc/sudoers.d/ 이용)
# 핵심 변경 사항: NOPASSWD: 옵션 추가
echo "Configuring passwordless sudo..."
echo "$username ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee "/etc/sudoers.d/$username" > /dev/null

# 3. 권한 설정 (0440 필수)
sudo chmod 0440 "/etc/sudoers.d/$username"

# 4. 설정 검증
if sudo visudo -c; then
    echo "✅ Success! User '$username' created with passwordless sudo access."
else
    echo "❌ Error: Sudoers configuration failed."
fi
```

### Swap 메모리 비활성화
Kubernetes는 Pod와 컨테이너의 리소스를 세밀하게 관리합니다. 스왑이 활성화되어 있으면, 노드의 메모리 사용 상태가 정확하지 않게 되며, 이는 Kubernetes의 스케줄링 및 리소스 제한 기능에 영향을 미칠 수 있습니다.

```bash
sudo swapoff -a
sudo sed -i '/swap/ s|^|# |' /etc/fstab
sudo swapon --show
```

### 네트워크 포워딩 활성화
Kubernetes 노드에서 iptables가 브리지된 트래픽(bridge traffic)을 올바르게 처리하도록 설정하기 위해 설정한다.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
# 필요한 sysctl 파라미터를 설정하면, 재부팅 후에도 값이 유지된다.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 재부팅하지 않고 sysctl 파라미터 적용하기
sudo sysctl --system
```

### 컨테이너 런타임 설치
Kubernetes는 컨테이너를 스케줄링하고 관리하기 위해 컨테이너 런타임과 상호작용합니다. Kubernetes는 **CRI(Container Runtime Interface)**라는 표준 인터페이스를 통해 여러 컨테이너 런타임과 통신합니다.

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -pv /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i \
    -e 's|SystemdCgroup = .*|SystemdCgroup = true|' \
    -e 's|sandbox_image = .*|sandbox_image = "registry.k8s.io/pause:3.10"|' /etc/containerd/config.toml

sudo systemctl enable --now containerd
```
 
| **컨테이너 런타임** | **설명**                                             | **주요 특징**                                             | **적합한 사용 사례**                                          |
|--------------------|----------------------------------------------------|--------------------------------------------------------|-----------------------------------------------------------|
| **containerd**      | Docker에서 분리된 독립적인 런타임, Kubernetes에서 기본 런타임으로 사용                      | - Kubernetes와 기본적으로 호환됨<br>- 이미지 관리 및 컨테이너 라이프사이클 관리 | - Kubernetes 클러스터에서 기본 런타임                         |
| **CRI-O**           | Kubernetes와의 통합을 위해 설계된 경량 런타임                                        | - Kubernetes와 최적화된 통합<br>- OCI 이미지 표준 준수<br>- 경량화된 설계   | - Kubernetes 환경에서만 사용                                  |
| **runc**            | OCI 표준을 준수하는 컨테이너 런타임의 기본 구현체                                    | - 낮은 수준의 시스템 호출로 컨테이너 실행<br>- OCI 표준 준수               | - 기본적인 컨테이너 실행에 사용, 다른 런타임의 핵심 구성 요소    |

> - 컨테이너 런타임 종류: containerd, CRI-O, docker
{: .prompt-info }

## Kubernetes 설치

### **kubeadm**, **kubelet**, **kubectl** 설치
kubeadm: 클러스터를 부트스트랩하는 명령이다.

kubelet: 클러스터의 모든 머신에서 실행되는 파드와 컨테이너 시작과 같은 작업을 수행하는 컴포넌트이다.

kubectl: 클러스터와 통신하기 위한 커맨드 라인 유틸리티이다.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

### kubeadm을 사용하여 클러스터 생성
**클러스터 이미지** 다운로드

```bash
kubeadm config images pull
```

설치 버전 확인

```bash
kubeadm version -o short
kubectl version
kubelet --version
```

### Kubernetes 클러스터 초기화

- `--apiserver-advertise-address` 마스터 노드 IP 입력
- `--pod-network-cidr` pod 네트워크 대역 설정
  
```bash
# kubeadm init --pod-network-cidr=10.244.0.0/16 --v=5
echo "y" | kubeadm reset
kubeadm init
```

```bash
kubeadm reset
```

> - Kubernetes 클러스터 환경을 초기화
{: .prompt-info }

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>
  
```bash
[init] Using Kubernetes version: v1.31.3                                                                                                                                                            
[preflight] Running pre-flight checks                                                                                                                                                               
[preflight] Pulling images required for setting up a Kubernetes cluster                                                                                                                             
[preflight] This might take a minute or two, depending on the speed of your internet connection                                                                                                     
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'                                                                                                          
W1129 12:29:09.369676    4743 checks.go:846] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use
 "registry.k8s.io/pause:3.10" as the CRI sandbox image.                                                                                                                                             
[certs] Using certificateDir folder "/etc/kubernetes/pki"                                                                                                                                           
[certs] Generating "ca" certificate and key                                                                                                                                                         
[certs] Generating "apiserver" certificate and key                                                                                                                                                  
[certs] apiserver serving cert is signed for DNS names [awx kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.0.240]            
[certs] Generating "apiserver-kubelet-client" certificate and key                                                                                                                                   
[certs] Generating "front-proxy-ca" certificate and key                                                                                                                                             
[certs] Generating "front-proxy-client" certificate and key                                                                                                                                         
[certs] Generating "etcd/ca" certificate and key                                                                                                                                                    
[certs] Generating "etcd/server" certificate and key                                                                                                                                                
[certs] etcd/server serving cert is signed for DNS names [awx localhost] and IPs [192.168.0.240 127.0.0.1 ::1]                                                                                      
[certs] Generating "etcd/peer" certificate and key                                                                                                                                                  
[certs] etcd/peer serving cert is signed for DNS names [awx localhost] and IPs [192.168.0.240 127.0.0.1 ::1]                                                                                        
[certs] Generating "etcd/healthcheck-client" certificate and key                                                                                                                                    
[certs] Generating "apiserver-etcd-client" certificate and key                                                                                                                                      
[certs] Generating "sa" key and public key                                                                                                                                                          
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"                                                                                                                                              
[kubeconfig] Writing "admin.conf" kubeconfig file                                                                                                                                                   
[kubeconfig] Writing "super-admin.conf" kubeconfig file                                                                                                                                             
[kubeconfig] Writing "kubelet.conf" kubeconfig file                                                                                                                                                 
[kubeconfig] Writing "controller-manager.conf" kubeconfig file                                                                                                                                      
[kubeconfig] Writing "scheduler.conf" kubeconfig file                                                                                                                                               
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"                                                                                                                   
[control-plane] Using manifest folder "/etc/kubernetes/manifests"                                                                                                                                   
[control-plane] Creating static Pod manifest for "kube-apiserver"                                                                                                                                   
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 1.502300159s
[api-check] Waiting for a healthy API server. This can take up to 4m0s
[api-check] The API server is healthy after 27.505794599s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node awx as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node awx as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: ayzlvn.agpd3mm6rfooxf11
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
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

kubeadm join 192.168.0.240:6443 --token ayzlvn.agpd3mm6rfooxf11 \
        --discovery-token-ca-cert-hash sha256:87ea1bdbf12fb51a73efe8441b62e685702e7860864c4cb595c607ed6fc97d05  
```

</details>

루트가 아닌 사용자가 kubectl 명령어를 작동시키려면 다음 명령어를 실행한다.

```bash
mkdir -pv $HOME/.kube
sudo cp -iv /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown -v $(id -u):$(id -g) $HOME/.kube/config
```

루트 사용자인 경우 환경변수를 추가한다.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 플러그인 설치
CNI는 Kubernetes 클러스터 내 Pod 간 통신 및 외부 네트워크 연결을 관리하는 네트워크 플러그인입니다. Kubernetes는 다양한 CNI 플러그인을 지원하며, Calico, Flannel, Weave Net 등이 가장 널리 사용됩니다.

| **항목**              | **Calico**                              | **Flannel**                           |
|----------------------|-----------------------------------------|---------------------------------------|
| **목적**              | 고성능, 고급 네트워크 정책 지원                 | 간단한 네트워크 연결                  |
| **확장성**             | 대규모 클러스터에 적합                        | 중소규모 클러스터에 적합              |
| **설정 복잡도**         | 복잡                                     | 간단                                  |
| **네트워크 정책 지원**   | Kubernetes 네트워크 정책 지원               | 기본적으로 지원하지 않음              |
| **리소스 사용량**       | 높음                                    | 낮음                                  |
| **트래픽 처리 성능**     | 높음                                    | 상대적으로 낮음                       |
| **기술**              | BGP, IP-in-IP, NAT, 네트워크 정책         | VXLAN, host-gw 등 오버레이 네트워크    |

> **Calico** vs **Flannel** 비교
{: .prompt-info }

A. **Calico CNI** 설치

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

B. **Flannel CNI** 설치

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```bash
kubeadm init --pod-network-cidr=10.2.0.0/16
curl -sO https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i 's/"Network": "10.244.0.0\/16"/"Network": "10.2.0.0\/16"/' kube-flannel.yml
kubectl apply -f kube-flannel.yml
```
> 사용자 정의 --pod-network-cidr
{: .prompt-info }

### kubeadm, kubectl 자동완성 기능

```bash
kubeadm completion bash >/etc/bash_completion.d/kubeadm
source /etc/bash_completion.d/kubeadm
kubectl completion bash >/etc/bash_completion.d/kubectl
source /etc/bash_completion.d/kubectl

source <(kubectl completion bash) # bash-completion 패키지를 먼저 설치한 후, bash의 자동 완성을 현재 셸에 설정한다
echo "source <(kubectl completion bash)" >> ~/.bashrc # 자동 완성을 bash 셸에 영구적으로 추가한다
source <(kubeadm completion bash) # bash-completion 패키지를 먼저 설치한 후, bash의 자동 완성을 현재 셸에 설정한다
echo "source <(kubeadm completion bash)" >> ~/.bashrc # 자동 완성을 bash 셸에 영구적으로 추가한다
```

### Control plane 노드 격리
**마스터 노드(Control Plane)**에 일반 워크로드 Pod가 생성되지 않도록 방지하는 것입니다.

```bash
kubectl taint nodes master_node01 node-role.kubernetes.io/control-plane:NoSchedule
```

```bash
kubectl taint nodes worker_node01 node-role.kubernetes.io/control-plane-
kubectl taint nodes worker_node02 node-role.kubernetes.io/control-plane-
```

### 기타 명령어

**Kubernetes 클러스터** 필수 컴포넌트 상태 확인

```bash
kubectl get pods -n kube-system
```

**Token** Hash 값 확인

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

**Token** 목록 확인

```bash
kubeadm token list
```

**Token** 유효 기간 만료 무제한 설정

```bash
kubeadm token create --ttl=0
```

**워커 노드**를 **마스터 노드**에 연결

```bash
kubeadm join 192.168.0.240:6443 --token ayzlvn.agpd3mm6rfooxf11 \
        --discovery-token-ca-cert-hash sha256:87ea1bdbf12fb51a73efe8441b62e685702e7860864c4cb595c607ed6fc97d05
```

**클러스터** 노드 정보 조회

```bash
kubectl get nodes -A -o wide
kubectl get nodes --all-namespaces --output=wide
```

**클러스터** 파드 정보 조회

```bash
kubectl get pod -A -o wide
kubectl get pods --all-namespaces --output=wide
```

**클러스터** 서비스 정보 조회

```bash
kubectl get svc -A -o wide
kubectl get svc --all-namespaces --output=wide
```

<details markdown="block" style="margin: 1em 0; padding: 0.8em; border: 2px solid #007acc; border-radius: 10px; background-color: #f5faff; box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);">
  <summary>
    펼치기/접기
  </summary>

```bash
root@MasterNode:~:> kubectl get pods --all-namespaces --output=wide
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE     IP              NODE         NOMINATED NODE   READINESS GATES
kube-system            calico-kube-controllers-85b5b5888d-mpwgt     1/1     Running   0             44s     192.168.181.6   masternode   <none>           <none>
kube-system            calico-node-45vnr                            1/1     Running   0             44s     220.71.21.173   masternode   <none>           <none>
kube-system            coredns-5cc6c6d8b5-jrk2z                     1/1     Running   0             8m44s   192.168.181.5   masternode   <none>           <none>
kube-system            coredns-5cc6c6d8b5-nnctc                     1/1     Running   0             8m44s   192.168.181.4   masternode   <none>           <none>
kube-system            etcd-masternode                              1/1     Running   3             76m     220.71.21.173   masternode   <none>           <none>
kube-system            kube-apiserver-masternode                    1/1     Running   2             76m     220.71.21.173   masternode   <none>           <none>
kube-system            kube-controller-manager-masternode           1/1     Running   2             76m     220.71.21.173   masternode   <none>           <none>
kube-system            kube-proxy-5rh4w                             1/1     Running   0             76m     220.71.21.173   masternode   <none>           <none>
kube-system            kube-scheduler-masternode                    1/1     Running   3             76m     220.71.21.173   masternode   <none>           <none>
kube-system            weave-net-qzsql                              2/2     Running   1 (20m ago)   20m     220.71.21.173   masternode   <none>           <none>
kubernetes-dashboard   dashboard-metrics-scraper-799d786dbf-7nmcq   1/1     Running   0             17m     192.168.181.1   masternode   <none>           <none>
kubernetes-dashboard   kubernetes-dashboard-6b6b86c4c5-flx5z        1/1     Running   0             17m     192.168.181.2   masternode   <none>           <none>
```

</details>

**대시보드** UI 배포

```bash
curl -s https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml -o recommended.yaml
```

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml
```

**대시보드로**의 접속을 활성화

```bash
kubectl proxy
```

```bash
kubectl proxy --port=30080 --address='0.0.0.0' &
```

**대시보드** UI 접속

```bash
 http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

```bash
kubectl get svc -n kubernetes-dashboard
```

![kubeadm-token-auth](https://user-images.githubusercontent.com/36792594/150469046-c7dcceaf-8d50-4e27-9a56-535d9b11864f.png){: .align-center}
[그림1]

### Error

```bash
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.
```

```bash
swapoff > daemon.json > systemctl restart docker;systemctl restart kubelet;
```

## 참조
- [컨테이너 런타임 가이드](https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/)
- [k8s 설치가이드](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#kubeadm-kubelet-및-kubectl-설치/)
- [k8s 대시보드 가이드](https://kubernetes.io/ko/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [칼리코 CNI 설치가이드](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
- [Ansible Kubernetes설치 가이드](https://github.com/kubernetes-sigs/kubespray)

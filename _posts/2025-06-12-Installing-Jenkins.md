---
title: Jenkins 설치
author: G.G
date: 2025-06-12 15:42 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Helm, Jenkins]
---

## 📘 개요 (Overview)
이 문서는 RKE2 기반 쿠버네티스 클러스터 환경에서 Jenkins CI/CD 시스템을 Helm Chart를 통해 설치하고 구성하는 방법을 안내합니다.
기본적인 클러스터 설치부터 Jenkins의 Helm 배포, 스토리지(NFS), 로드밸런서(MetalLB), Ingress 설정까지 포함되어 있어 온프레미스 환경에서도 실용적으로 활용 가능합니다.

## 🧭 등장배경
Jenkins 기반 내부 CI/CD 인프라 구축 과정에서 다음과 같은 문제가 있었습니다:

- `/etc/hosts` 기반 수동 IP 관리의 **확장성 한계**
- **사설 DNS 인프라 구축 필요성**
- **GUI와 API를 통한 레코드 관리** 요구
- 기존 Docker Compose 기반 시스템의 **Kubernetes 전환 필요성**

이에 따라 **RKE2 기반 K8s 클러스터 + Helm + NFS + MetalLB**를 이용해 Jenkins를 안정적으로 배포하는 방안을 설계하게 되었습니다.


## 📝 구성 요소 (Components)

| 구성 요소            | 역할 및 설명                  |
| ---------------- | ------------------------ |
| **RKE2**         | 쿠버네티스 배포 (Rancher 제공)    |
| **Helm**         | Jenkins 설치 및 패키지 관리      |
| **k9s**          | 쿠버네티스 CLI 관리도구           |
| **NFS**          | Jenkins 영속 볼륨 스토리지 제공    |
| **MetalLB**      | 로드밸런서를 통한 외부 접근 설정       |
| **cert-manager** | Ingress TLS 인증서 자동화 (옵션) |
| **Jenkins**      | CI/CD 시스템                |

## 🏗️ 아키텍처

```bash
┌─────────────┐       (1) Push Code & Manifests
│  Developer  │───────▶│ GitHub / GitLab (SCM) │
└─────────────┘       ◀─────────────────────────┘
                        │
                        ▼ (Webhook Trigger)
┌─────────────┐       (2) Build, Test, Package (Maven/Gradle)
│   Jenkins   │───────▶│ Nexus (Artifacts)       │
│ (CI Server) │       │                         │
└──────┬──────┘       ◀─────────────────────────┘
       │
       ▼ (3) Build & Push Docker Image
┌─────────────┐
│    Harbor   │
│(Image Registry)│
└───────┬───────┘
        │
        ▼ (4) Image Update (Webhook/Git Commit)
┌───────────────────┐       (5) Sync with Git Repo (Manifests)
│      ArgoCD       │───────▶│ GitHub / GitLab     │
│   (CD Controller) │       │ (GitOps - Deployment)│
└───────┬───────────┘       ◀─────────────────────────┘
        │
        ▼ (6) Apply Manifests (Helm Charts etc.)
┌───────────────────────────┐        ┌────────────┐
│   Kubernetes Cluster      │──────▶│   Ingress  │
│ (RKE2, Pods, Services)    │        └────┬───────┘
└───────────┬───────────────┘             │
            │                           ┌──▼───┐
            ▼                           │MetalLB│
 ┌────────────────────────┐             └───┬───┘
 │  NFS Persistent Volume │                 │
 │ (Storage for Pods etc.)│◀────────────────┘
 └────────────────────────┘
```

## ⚙️ 설지방법

### RKE2, k9s, Helm 설치

```bash
# 📌 스왑 메모리 비활성화 (K8s는 swap을 비활성화해야 안정적임)
sudo swapoff -a

# 📌 RKE2 설치
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -

# 📌 RKE2 서비스 자동 시작
systemctl enable rke2-server --now
```

```bash
# 📌 K9s CLI 설치 (TUI 기반 쿠버네티스 관리 도구)
curl -sS https://webinstall.dev/k9s | bash

# 📌 Helm 설치 (Kubernetes 패키지 매니저)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### PATH, alias 설정

```bash
cat <<'EOF' | sudo tee -a ~/.bashrc

###############################################
# ✅ 사용자 환경 설정 (PATH 및 KUBECONFIG)
# - 목적: rke2 관련 CLI 도구 및 kubectl 명령어 정상 인식
# - 대상: 현재 로그인 사용자 환경에만 적용됨
###############################################


# 📌 kubectl 명령어에서 사용할 kubeconfig 경로 설정
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

# 📌 rke2 CLI 바이너리 경로 추가 (예: rke2-killall.sh, kubectl 등 포함)
export PATH=$PATH:/var/lib/rancher/rke2/bin

# 📌 시스템 전체 바이너리 경로 추가 (예: k3s, helm, k9s 등)
export PATH="/usr/local/bin:$PATH"
EOF

# ✨ 설정 적용 (현재 쉘에 즉시 반영)
source ~/.bashrc

kubectl get nodes

mkdir -p ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
chown $(id -u):$(id -g) ~/.kube/config

# kubectl 자동완성 등록
kubectl completion bash >/etc/bash_completion.d/kubectl
source /etc/bash_completion.d/kubectl

# Helm 자동완성 등록
helm completion bash > /etc/bash_completion.d/helm
source /etc/bash_completion.d/helm
```

### NFS 서버 (PVC 제공용) 구성

```bash
# NFS 설치 & 복잡 조치
# nfs-kernel-server
sudo dnf install -y nfs-utils
sudo systemctl enable nfs-server --now

# Export 디렉토리 생성
sudo mkdir -p /data
sudo chown -R 5000:5000 /data

# /etc/exports 설정
echo "/data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports > /dev/null
sudo exportfs -rv

# 방화벽 설정
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
```

### NFS StorageClass 설치

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm repo update

IP=$(ip -br address | grep -E 'eth' | awk '{print $3}' | cut -d'/' -f1)
helm install nfs-subdir nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-system \
  --create-namespace \
  --set nfs.server=$IP \
  --set nfs.path=/data \
  --set storageClass.name=nfs \
  --set storageClass.defaultClass=true \
  --set storageClass.reclaimPolicy=Retain \
  --set storageClass.allowVolumeExpansion=true

kubectl get storageclass
```

### MetalLB 로드버더 설치

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace \
  --set webhook.enabled=true
```

### 🔒 Cert-manager 설치 (Ingress TLS 인증서 자동화용)

```bash
# 📌 cert-manager CRD(커스텀 리소스 정의) 설치 (선택사항: Helm에서 crds.enabled를 true로 하면 생략 가능)
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.crds.yaml
# 📌 cert-manager Helm 저장소 추가 및 업데이트
helm repo add jetstack https://charts.jetstack.io
helm repo update

# 📌 최신 cert-manager 버전 자동 추출
CERT_MANAGER_VERSION=$(helm search repo jetstack/cert-manager --versions | \
                       awk 'NR > 1 {print $2}' | \
                       head -n 2 | \
                       tail -n 1)

# 📌 cert-manager 설치 (Ingress에서 TLS 자동화에 활용)
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version $CERT_MANAGER_VERSION \
  --set crds.enabled=true
```

### 확인

```bash
helm repo list
helm list -A
kubectl get all --all-namespaces
```

### 🧩 Jenkins Helm 설치 (Ingress + NFS + ClusterIP 구성)

```bash
# 📌 Jenkins 공식 Helm Chart 저장소 등록
helm repo add jenkins https://charts.jenkins.io
helm repo update

# 📌 Jenkins 설치 (NFS PVC 사용 + Ingress 구성)
sudo tee values.yaml <<'EOF'
# values.yaml for Jenkins Helm Chart (Corrected)

controller:
  serviceType: ClusterIP
  ingress:
    enabled: true
    ingressClassName: nginx
    hostName: jenkins.infra.com
    annotations:
      "cert-manager.io/cluster-issuer": "letsencrypt-prod"
    hosts:
      - host: jenkins.infra.com
        paths:
          - path: /
            pathType: ImplementationSpecific
    tls:
      - hosts:
          - jenkins.infra.com
        secretName: jenkins-tls-secret
persistence:
  storageClass: "nfs"
EOF

helm upgrade --install jenkins jenkins/jenkins \
  --namespace jenkins --create-namespace \
  -f values.yaml
```

### 확인

```bash
helm list -n jenkins
helm status jenkins -n jenkins
helm get manifest jenkins -n jenkins
helm get values jenkins -n jenkins
kubectl get all -n jenkins
kubectl get ingress -n jenkins
kubectl get certificate -n jenkins
kubectl describe pod jenkins-0 -n jenkins
kubectl logs -f pod/jenkins-0 -c jenkins -n jenkins
kubectl logs jenkins-0 -n jenkins --all-containers=true
```

### 🛠️ CoreDNS에 Ingress 도메인 반영 (선택. DNS 통신 안될 경우)

```bash
# 📌 CoreDNS 설정 확인 및 편집 (jenkins.infra.com 도메인 인식되도록)
kubectl get configmaps -n kube-system
kubectl edit configmap rke2-coredns-rke2-coredns -n kube-system

# 📌 CoreDNS 재시작 (변경 적용)
kubectl delete pods -n kube-system -l k8s-app=kube-dns
# 또는
kubectl rollout restart deployment rke2-coredns-rke2-coredns -n kube-system

# 📌 Jenkins Helm Chart 제거 (PVC 등은 남아 있음)
helm uninstall jenkins -n jenkins
```

### 🔥 방화벽 설정 (HTTP 서비스 허용)

```bash
# 로그가 필요하면 아래처럼 Drop 패킷 로깅 규칙 추가 (한시적)
sudo firewall-cmd --set-log-denied=all   # all, unicast, broadcast, multicast

# Drop된 패킷 확인
sudo journalctl -xef | grep 'REJECT'
# 또는
sudo dmesg -Tw | grep 'REJECT'

# 방화벽 다시 로드
sudo systemctl stop firewalld.service
```

📝 PodMan Jenkins 설치 방법

```bash
# Podman 및 Podman Compose 패키지 설치
sudo dnf install -y podman podman-compose

# 컨테이너 레지스트리 기본값 설정 (docker.io)
sudo sed -i 's/^unqualified-search-registries = .*$/unqualified-search-registries = ["docker.io"]/' /etc/containers/registries.conf

# Podman Compose 설치 버전 확인
podman-compose --version

# Jenkins 및 데이터용 디렉토리 생성
mkdir -pv /opt/jenkins
mkdir -pv /data/jenkins

# 데이터 디렉토리들에 개별 컨테이너 파일 컨텍스트 영구 적용 규칙 추가
sudo semanage fcontext -a -t container_file_t "/data/jenkins(/.*)?"

# 영구 규칙 적용
sudo restorecon -Rv /data

# DevOps 네트워크 대역 생성
podman network create \
  --driver bridge \
  --subnet 10.90.0.0/24 \
  --gateway 10.90.0.1 \
  --ip-range 10.90.0.64/26 \
  net_devops

# Jenkins 및 데이터용 디렉토리 생성
cat << EOF > /opt/jenkins/docker-compose.yml
# /opt/jenkins/docker-compose.yml

services:
  jenkins:
    image: jenkins/jenkins:latest-jdk21
    container_name: jenkins
    hostname: jenkins
    restart: unless-stopped
    ports:
      - "8090:8080"   # 웹 UI 접속용
      - "50000:50000" # 에이전트 통신용
    volumes:
      - jenkins_home:/var/jenkins_home
    environment:
      TZ: 'Asia/Seoul'
      # 로그 설정 파일이 실제 경로에 없으면 오류가 날 수 있으니 유의하세요!
      JAVA_OPTS: "-Djava.util.logging.config.file=/var/jenkins_home/log.properties"
    networks:
      - net_devops

  jenkins-agent:
    image: jenkins/ssh-agent:latest-jdk21 # 오타 수정 완료
    container_name: jenkins-agent
    hostname: jenkins-agent
    env_file:        # <--- 여기에 추가
      - .env         # <--- .env 파일의 내용을 환경 변수로 불러오겠다
    environment:
      - JENKINS_AGENT_SSH_PUBKEY=\${AGENT_SSH_PUBKEY}
    networks:
      - net_devops

volumes:
  jenkins_home: # 선언식으로 자동 생성 관리
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      # 실제 파일이 저장될 로컬의 절대 경로를 적어주세요.
      # (예: /home/user/jenkins/jenkins_home)
      device: '${PWD}/jenkins_home'

networks:
  net_devops:
    external: true # 미리 생성된 네트워크 사용
EOF
```

```bash
# Jenkins 설치 디렉토리로 이동
cd /opt/jenkins

# Podman Compose를 이용한 컨테이너 실행 (백그라운드)
podman compose up -d

# 🛠️ 컨테이너 systemd 서비스 파일 생성
podman generate systemd --name jenkins --files --new

# 📂 생성된 서비스 파일 시스템에 등록
cp -v container-* /usr/lib/systemd/system/

# 🚀 서비스 활성화 및 즉시 시작
systemctl enable --now container-jenkins
```

### 🔐 Jenkins 초기 설정 가이드

```bash
# Jenkins 설치 후 다음 명령어로 초기 비밀번호 확인
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```

```bash
# Jenkins 설치 후 다음 명령어로 초기 비밀번호 확인
podman exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

![그림_1](/assets/img/2025-06-15/그림1.png)
_Jenkins 로그인_

![그림_2](/assets/img/2025-06-15/그림2.png)
_Jenkins 대시보드_

## 참고 자료
- [Jenkins 공식 문서](https://www.jenkins.io/doc/book/installing/kubernetes/)
- [Github 공식 문서](https://github.com/jenkinsci/docker)
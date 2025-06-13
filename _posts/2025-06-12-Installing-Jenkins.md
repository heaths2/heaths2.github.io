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
┌─────────────┐        ┌────────────┐
│  Developer  │──────▶│   Ingress   │
└─────────────┘        └────┬───────┘
                             │
                    ┌────────▼────────┐
                    │   MetalLB LB    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │    Jenkins Pod  │
                    │(Helm Chart 기반)│
                    └────────┬────────┘
                             │
                ┌────────────▼────────────┐
                │ NFS Persistent Volume   │
                │(nfs-subdir-provisioner)│
                └────────────────────────┘
```

## ⚙️ 설지방법

### RKE2, k9s, Helm 설치

```bash
# 📌 스왑 메모리 비활성화 (K8s는 swap을 비활성화해야 안정적임)
sudo swapoff -a

# 📌 RKE2 설치 (서버 타입으로 지정, 버전 명시)
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="v1.31.8+rke2r1" INSTALL_RKE2_TYPE="server" sh -

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
sudo mkdir -p /data/db
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

IP=$(ip -br address | grep -E 'eth|enp0s' | awk '{print $3}' | cut -d'/' -f1)
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

### 확인

```bash
helm repo list
helm list -A
kubectl get all --all-namespaces
```

### 🔒 cert-manager 설치 (Ingress TLS 인증서 자동화용)

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

### 🧩 Jenkins Helm 설치 (Ingress + NFS + ClusterIP 구성)

```bash
# 📌 Jenkins 공식 Helm Chart 저장소 등록
helm repo add jenkins https://charts.jenkins.io
helm repo update

# 📌 Jenkins 설치 (NFS PVC 사용 + Ingress 구성)
helm upgrade --install jenkins jenkins/jenkins \
--namespace jenkins \
--create-namespace \
--set persistence.storageClass=nfs \                        # NFS 스토리지 사용
--set controller.serviceType=ClusterIP \                   # 내부 서비스용
--set ingress.enabled=true \                               # Ingress 사용
--set ingress.className=nginx \                            # IngressClass 설정
--set ingress.hosts[0].name=jenkins.infra.com \            # 호스트 이름
--set ingress.hosts[0].path=/ \                            # 경로
--set ingress.service.port=8080                            # 서비스 포트

# NFS 스토리지 사용
# 내부 서비스용
# Ingress 사용
# IngressClass 설정
# 호스트 이름
# 경로
helm upgrade --install jenkins jenkins/jenkins \
--namespace jenkins \
--create-namespace \
--set persistence.storageClass=nfs \
--set controller.serviceType=ClusterIP \
--set ingress.enabled=true \
--set ingress.className=nginx \
--set ingress.hosts[0].name=jenkins.infra.com \
--set ingress.hosts[0].path=/
```

### 🛠️ CoreDNS에 Ingress 도메인 반영 (선택. DNS 통신 안될 경우)

```bash
# 📌 CoreDNS 설정 확인 및 편집 (jenkins.infra.com 도메인 인식되도록)
kubectl get configmaps -n kube-system
kubectl edit configmap rke2-coredns-rke2-coredns -n kube-system

# 📌 CoreDNS 재시작 (변경 적용)
kubectl rollout restart deployment rke2-coredns-rke2-coredns -n kube-system

# 📌 Jenkins Helm Chart 제거 (PVC 등은 남아 있음)
helm uninstall jenkins -n jenkins
```

### 🔥 방화벽 설정 (HTTP 서비스 허용)

```bash
# 📌 HTTP 포트 방화벽 허용 (Ingress 접근을 위해 필요)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

### 🔐 Jenkins 초기 설정 가이드

```bash
# Jenkins 설치 후 다음 명령어로 초기 비밀번호 확인
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```

### 릴리스가 존재하면 업그레이드, 없으면 신규 설치

```bash
helm upgrade --install powerdns-admin ~/PowerDNS-Admin \
  --namespace pdns \
  --values ~/PowerDNS-Admin/values.yaml
```

### 전체 설정 내용 확인

```bash
kubectl get all -A -o wide
```

### DNS 등록

![그림_1](/assets/img/2025-05-04/그림1.png)
_PowerDNS-Admin 계정생성 클릭_

![그림_2](/assets/img/2025-05-04/그림2.png)
_PowerDNS-Admin 계정생성_

![그림_3](/assets/img/2025-05-04/그림3.png)
_PowerDNS-Admin 계정 로그인인_

![그림_4](/assets/img/2025-05-04/그림4.png)
_PowerDNS-Admin 계정생성 클릭_

![그림_5](/assets/img/2025-05-04/그림5.png)
_PowerDNS-Admin Zone 생성_

![그림_6](/assets/img/2025-05-04/그림6.png)
_PowerDNS-Admin Zone 클릭_

![그림_7](/assets/img/2025-05-04/그림7.png)
_PowerDNS-Admin Zone에 레코드 등록_

![그림_8](/assets/img/2025-05-04/그림8.png)
_PowerDNS-Admin Zone에 레코드 적용 확인_

![그림_9](/assets/img/2025-05-04/그림9.png)
_PowerDNS-Admin Zone에 레코드 목록 확인_

## 참고 자료
- [Jenkins 공식식 문서](https://www.jenkins.io/doc/book/installing/kubernetes/)

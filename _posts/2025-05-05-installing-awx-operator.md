---
title: AWX & AWX-Operator 설치
author: G.G
date: 2025-05-05 20:48 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, AWX, AWX-Operator, Helm]
---

## 📘 개요 (Overview)
AWX는 Ansible Tower의 오픈소스 버전으로, 웹 기반 GUI 및 REST API를 통해 Ansible Playbook을 중앙에서 관리하고 자동화할 수 있는 플랫폼입니다.
AWX Operator는 Kubernetes 환경에서 AWX 인스턴스를 Custom Resource(사용자 정의 리소스) 형태로 배포 및 유지 관리할 수 있도록 설계된 컨트롤러입니다.

이 문서는 RKE2 기반의 고가용성 Kubernetes 환경에서 Helm Chart를 활용하여 AWX Operator 및 AWX 인스턴스를 설치하는 절차를 담고 있으며,
실제 운영 환경에서 사용할 수 있도록 보안 설정, Helm 기반 버전 관리, Secret 관리, Ingress 설정 등의 항목을 포함합니다.

## 🧭 등장 배경 (Background)
- 단일 서버 기반 Ansible 실행의 복잡성 증가
- 멀티 사용자의 GUI 기반 작업 스케줄링 요구
- Kubernetes 환경에 쉽게 배포 가능한 자동화 도구 필요
- Helm Chart를 통한 선언적 설치 및 유지보수

## 🧩 주요 구성 요소 (Components)

| 구성 요소            | 설명                                                  |
| ---------------- | --------------------------------------------------- |
| **AWX Operator** | Kubernetes에서 AWX 인스턴스를 관리하는 컨트롤러(Operator 패턴 적용)    |
| **AWX**          | Ansible Playbook을 GUI로 실행하고, RBAC, 스케줄링, 인벤토리 관리 제공 |
| **PostgreSQL**   | AWX 내부 메타데이터 저장소 (Operator가 자동 관리)                  |
| **Helm**         | Kubernetes 리소스 설치 및 버전 관리를 위한 패키지 관리 도구             |

## 🛠️ 설치 방법 (Installation)

### RKE2, k9s, Helm 설치

```bash
# 스왑 메모리 비활성화
sudo swapoff -a

# curl -s https://update.rke2.io/v1-release/channels/stable
# curl -sfL https://get.rke2.io | sh -
# RKE2 CLI 설치
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="v1.31.8+rke2r1" INSTALL_RKE2_TYPE="server" sh -
systemctl enable rke2-server --now
```

```bash
# K9s CLI 설치
curl -sS https://webinstall.dev/k9s | bash

# Helm 설치
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
sudo dnf install -y nfs-utils
sudo systemctl enable --now nfs-server

# Export 디렉토리 생성
sudo mkdir -p /data/db
sudo chown -R 5000:5000 /data

# /etc/exports 설정
echo "/data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee /etc/exports > /dev/null
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

### AWX & AWX-Operator Helm Chart 배포

```bash
helm repo add awx-operator https://ansible-community.github.io/awx-operator-helm/
helm repo update
# helm search repo awx-operator --versions | head -5

helm create AWX-Operator
rm -f  AWX-Operator/templates/{deployment.yaml,hpa.yaml,serviceaccount.yaml,service.yaml,tests/*}

helm upgrade --install awx-operator ./AWX-Operator \
  --namespace awx \
  --create-namespace \
  -f AWX-Operator/values.yaml
```

```bash
systemctl stop firewalld.service

LTS_TAG=`curl -s https://api.github.com/repos/ansible/awx-operator/releases/latest | grep tag_name | cut -d '"' -f 4`
kubectl apply -k github.com/ansible/awx-operator/config/default?ref=${LTS_TAG}

mkdir -pv awx
# Basic Install
LTS_TAG=`curl -s https://api.github.com/repos/ansible/awx-operator/releases/latest | grep tag_name | cut -d '"' -f 4`

tee ~/awx/kustomization.yaml << EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=$LTS_TAG
  - awx-server.yaml
  - awx-ingress.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: $LTS_TAG

# Specify a custom namespace in which to install AWX
namespace: awx
EOF


tee ~/awx/awx-server.yaml << EOF
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-server
spec:
  service_type: ClusterIP
EOF

tee ~/awx/awx-ingress.yaml << EOF
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-ingress
  namespace: awx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: awx.infra
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: awx-server-service
            port:
              number: 80
EOF
```

## 참고 자료
> - [Helm 설치 가이드](https://helm.sh/ko/docs/intro/install/)
> - [awx-operator 가이드](https://ansible.readthedocs.io/projects/awx-operator/en/latest/)
> - [awx-operator git](https://github.com/ansible/awx-operator/blob/devel/docs/installation/basic-install.md)
> - [kustomize 참고](https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/)
> - [kustomize git 참고](https://github.com/kubernetes-sigs/kustomize)
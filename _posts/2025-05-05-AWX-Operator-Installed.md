---
title: AWX & AWX-Operator 설치
author: G.G
date: 2025-05-05 20:48 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, AWX, AWX-Operator, Helm]
---

## 📘 개요
PowerDNS는 유연하고 확장 가능한 오픈소스 DNS 서버이며, PowerDNS-Admin은 이를 위한 웹 기반 관리 인터페이스입니다.
이 문서는 Kubernetes(K3s) 환경에서 Helm Chart를 활용해 PowerDNS + PowerDNS-Admin 스택을 설치하고,
내부망 DNS 서버로 구성하는 과정을 담고 있습니다.

## 🧭 등장배경
- /etc/hosts 기반 수동 관리의 확장성 한계
- 내부망에서 독립된 DNS 인프라 필요성
- GUI 기반의 레코드 관리와 API 자동화를 고려한 선택
- Docker Compose → Helm Chart 기반 Kubernetes 전환 필요

## 🧩 주요 특징 및 구성 요소

| 구성 요소                      | 설명                                 |
| -------------------------- | ---------------------------------- |
| **PowerDNS Authoritative** | PostgreSQL Backend 기반 권한 있는 DNS 서버 |
| **PowerDNS-Admin**         | GUI 기반 웹 인터페이스 (API 지원 포함)         |
| **PostgreSQL**             | 레코드 메타데이터 저장소                      |
| **MetalLB**                | K3s 환경에서 LoadBalancer 타입의 외부 IP 제공 |
| **Helm**                   | 배포 자동화 및 재사용 가능한 Chart 관리 도구       |

## 🏗️ 아키텍처

```bash
[Browser]
   |
   | HTTP
   v
[MetalLB LoadBalancer: 172.16.0.242:8080]
   |
   v
[PowerDNS-Admin Pod] ---> [PowerDNS API: 8081]
                        |
                        v
                  [PowerDNS Pod: 53/tcp,udp]
                        |
                        v
                [PostgreSQL Pod (DB Backend)]
```

- **네트워크 포트 정리**

| 서비스            | 포트             | 설명            |
| -------------- | -------------- | ------------- |
| PowerDNS       | 53/tcp,udp     | DNS 서비스 기본 포트 |
| PowerDNS API   | 8081/tcp       | 관리용 REST API  |
| PowerDNS Admin | 8080 (→ 외부 80) | GUI 인터페이스     |
| PostgreSQL     | 5432/tcp       | 데이터베이스 연결     |

## 📁 파일 구조

```bash
PowerDNS-Admin
├── charts
├── [Chart.yaml](#chartyaml)
├── templates
│   ├── [deployment-postgresql.yaml](#deployment-postgresqlyaml)
│   ├── [deployment-powerdns-admin.yaml](#deployment-powerdns-adminyaml)
│   ├── [deployment-powerdns.yaml](#deployment-powerdnsyaml)
│   ├── [metallb-config.yaml](#metallb-config.yaml)
│   ├── [pvc-postgresql.yaml](#pvc-postgresqlyaml)
│   ├── [service-postgresql.yaml](#service-postgresqlyaml)
│   ├── [service-powerdns-admin.yaml](#service-powerdns-adminyaml)
│   ├── [service-powerdns.yaml](#service-powerdnsyaml)
└── [values.yaml](#valuesyaml)
```

## ⚙️ 사용법

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

helm install nfs-subdir nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --create-namespace \
  --set nfs.server=172.16.0.51 \
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
kubectl get all --all-namespaces
```

### PowerDNS & PowerDNS-Admin Helm Chart 배포

```bash
helm create PowerDNS-Admin
rm -f PowerDNS-Admin/templates/{deployment.yaml,hpa.yaml,serviceaccount.yaml,service.yaml,tests/*}
```

### AWX

```bash
systemctl stop firewalld.service

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
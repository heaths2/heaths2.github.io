---
title: Rancher Kubernetes Engine 2 설치
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

### K3s, kubectl, k9s, Helm 설치 


```bash
sudo swapoff -a
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

```bash
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

```bash
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update
# cert-manager 버전 목록 조회 (최신 안정 버전 확인용)
# helm search repo jetstack/cert-manager --versions | head -20

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --set crds.enabled=true

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rke2.infra.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=it@infra.com \
  --set letsEncrypt.ingress.class=nginx
```

```bash
cat <<'EOF' | sudo tee /etc/rancher/rke2/config.yaml
cluster-cidr: "10.42.0.0/16,2001:cafe:42::/56"
service-cidr: "10.43.0.0/16,2001:cafe:43::/112"
EOF
```


```bash
echo https://rke2.infra.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')

kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

### Worker Node

```bash
sudo swapoff -a
sudo systemctl stop firewalld
sudo systemctl disable firewalld
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="v1.31.8+rke2r1" INSTALL_RKE2_TYPE="agent" sh -
```

```bash
cat <<'EOF' | sudo tee /etc/rancher/rke2/config.yaml
server: https://192.168.1.51:9345
token: K10c80dfd26c5d8eb362b216f134523c053a834678f0d4e1b609da96c78faad38db::server:167bf371331b9a8636cbdf9b2681e672
EOF
```

## 참고 자료
> - [RKE2 공식 문서](https://docs.rke2.io/install/quickstart)
> - [RKE2 깃허브 문서](https://github.com/rancher/rke2)
> - [cert-manager 공식 문서](https://cert-manager.io/docs/installation/helm)
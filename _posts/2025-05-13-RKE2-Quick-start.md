---
title: RKE2 + Rancher 설치 매뉴얼
author: G.G
date: 2025-05-05 20:48 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, RKE2, Rancher, Helm, K9s]
---

## 📘 개요
이 문서는 경량 Kubernetes 배포판인 RKE2(Rancher Kubernetes Engine 2)를 기반으로,
Rancher를 설치하여 웹 기반 Kubernetes 관리 플랫폼을 구축하는 전체 과정을 설명합니다.
테스트 및 개발 환경에서 빠르게 RKE2 클러스터를 구성하고 Rancher UI로 관리하기 위한 목적에 최적화되어 있습니다.

## 🧭 등장배경
- 기존 VM 기반 Kubernetes 관리의 복잡성 해소 
- Rancher UI를 통한 멀티 클러스터 관리 수요
- K3s보다 보안 강화를 요구하는 환경(RKE2는 SELinux/PSP/etcd 내장)
- 스크립트 기반 빠른 재구축 및 테스트 환경 자동화를 위한 설계

## 🧩 주요 구성 요소

| 구성 요소            | 설명                                        |
| ---------------- | ----------------------------------------- |
| **RKE2 Server**  | Control-plane 역할을 하는 메인 노드                |
| **RKE2 Agent**   | Worker 역할을 하는 데이터 처리 전용 노드                |
| **Rancher**      | Kubernetes 클러스터 관리용 웹 UI 및 API 플랫폼        |
| **Helm**         | Kubernetes 패키지 배포 도구                      |
| **cert-manager** | TLS 인증서를 자동으로 발급 및 갱신하는 Kubernetes 리소스    |
| **MetalLB**      | LoadBalancer IP를 내부망에서 할당할 수 있는 네트워크 컴포넌트 |

## 🛠️ 설치 방법

### 🖥️ Control/Worker Node 공통 RKE2, k9s, Helm 설치

```bash
# 스왑 메모리 비활성화
sudo swapoff -a

# curl -s https://update.rke2.io/v1-release/channels/stable
# curl -sfL https://get.rke2.io | sh -
# Control Node RKE2 CLI 설치
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="v1.31.8+rke2r1" INSTALL_RKE2_TYPE="server" sh -
# Worker Node RKE2 CLI 설치
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="v1.31.8+rke2r1" INSTALL_RKE2_TYPE="agent" sh -
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

### 📌 cert-manager 설치

```bash
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update
# cert-manager 버전 목록 조회 (최신 안정 버전 확인용)
# helm search repo jetstack/cert-manager --versions | head -20

# cert-manager 설치 (v1.17.2, CRD 자동 설치 옵션 포함)
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --set crds.enabled=true
```

### 🌐 Rancher 설치

```bash
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

### 🔍 클러스터 정보 확인

```bash
# 각 노드의 Pod CIDR 대역 확인 (CNI 설정 확인용)
kubectl get nodes -o jsonpath="{range .items[*]}{.metadata.name} → {.spec.podCIDR}{'\n'}{end}"
```
> - IP 대역 확인
{: .prompt-info }

```bash
# Rancher Web UI 접속 URL 자동 생성 (초기 비밀번호 포함)
echo https://rke2.infra.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')

# Rancher 초기 관리자 패스워드 확인
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

### 🔐 Rancher Admin 비밀번호 초기화 

```bash
kubectl exec -it -n cattle-system rancher-5f6d98bff8-5b8sx -- reset-password
W0515 01:02:24.741810     653 client_config.go:667] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
New password for default admin user (user-4nnzl):
RuMIDJM_N3AYqUshnlsG
```

### 🖥️ Worker Node 설정

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
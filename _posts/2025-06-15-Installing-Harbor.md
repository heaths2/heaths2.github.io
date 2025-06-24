---
title: Harbor 설치
author: G.G
date: 2025-06-15 21:00 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Helm, Harbor]
---

## 📘 개요 (Overview)
이 문서는 RKE2 기반 쿠버네티스 클러스터 환경에서 Harbor 프라이빗 컨테이너 이미지 레지스트리를 Helm Chart를 통해 설치하고 구성하는 방법을 안내합니다. 기본적인 클러스터 설치부터 Harbor의 Helm 배포, 스토리지(NFS), 로드밸런서(MetalLB), Ingress 설정까지 포함되어 있어 온프레미스 환경에서 안전하고 효율적인 컨테이너 이미지 관리에 실용적으로 활용 가능합니다.

## 🧭 등장배경
CI/CD 파이프라인 구축 및 컨테이너 기반 애플리케이션 운영 과정에서 다음과 같은 문제에 직면했습니다.

- /etc/hosts 기반 수동 IP 관리의 확장성 한계: 증가하는 서비스 및 시스템의 IP를 수동으로 관리하는 것은 복잡하고 오류 발생 가능성이 높았습니다.
- 사설 DNS 인프라 구축 필요성: 내부망에서 컨테이너 이미지 레지스트리 및 기타 서비스에 안정적으로 접근하기 위한 사설 DNS 인프라의 필요성이 대두되었습니다.
- GUI와 API를 통한 레코드 관리 요구: DNS 레코드 관리를 더욱 편리하고 자동화하기 위해 GUI 및 API 기반의 관리 시스템이 필요했습니다.
- Docker Hub 등 공개 레지스트리 의존성 및 보안 문제: 민감한 기업 내부 이미지를 공개 레지스트리에 저장하는 것은 보안상 위험하며, 네트워크 문제 발생 시 외부 의존성으로 인해 CI/CD 파이프라인이 중단될 수 있었습니다.
- 이미지 보안 취약점 관리 및 서명 기능 부재: 컨테이너 이미지의 보안 취약점을 스캔하고 신뢰할 수 있는 이미지만 배포할 수 있도록 서명하는 기능이 필요했습니다.

이에 따라 RKE2 기반 K8s 클러스터 + Helm + NFS + MetalLB를 이용하여 Harbor를 안정적으로 배포하고 운영하는 방안을 설계하게 되었습니다


## 📝 구성 요소 (Components)

| 역할                                | 설명                                           |
| --------------------------------- | -------------------------------------------- |
| 🧱 **컨테이너 이미지 저장소**               | Docker, OCI 이미지들을 저장하고 배포                    |
| 🔐 **보안 스캔 (Vulnerability Scan)** | Trivy 연동으로 이미지 취약점 진단                        |
| 🔐 **접근 제어 / 인증 연동**              | LDAP, OIDC, SSO 등과 연동하여 권한 기반 이미지 접근 제어      |
| 🔄 **이미지 복제 (Replication)**       | 다른 레지스트리(Docker Hub, Quay, GCR 등)와 양방향 복제 가능 |
| 📝 **프로젝트 단위 관리**                 | 사용자별/팀별 이미지 관리 공간을 제공 (이름공간 분리)              |
| 📜 **정책 기반 거버넌스**                 | 레이블, 자동 삭제, 푸시 제한, 태그 정책 등 관리 가능             |
| 📡 **웹 UI + API 제공**              | GUI와 REST API로 모두 관리 가능                      |

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

### 🧩 Harbor Helm 설치 (Ingress + NFS + ClusterIP 구성)

```bash
# 📌 Harbor 공식 Helm Chart 저장소 등록
helm repo add harbor https://helm.goharbor.io
helm repo update

# 📌 Harbor 설치
sudo tee values.yaml <<'EOF'
# values.yaml for Harbor Helm Chart
externalURL: https://harbor.infra.com
expose:
  type: ingress
  tls:
    enabled: true
    secretName: "harbor-ingress"
  ingress:
    className: "nginx"
    hosts:
      core: harbor.infra.com
    annotations:
      "cert-manager.io/cluster-issuer": "letsencrypt-prod"
      "nginx.ingress.kubernetes.io/proxy-body-size": "0"
      "nginx.ingress.kubernetes.io/ssl-redirect": "true"
harborAdminPassword: "Harbor12345"
persistence:
  enabled: true
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      storageClass: nfs
      accessMode: ReadWriteOnce
      size: 50Gi
    chartmuseum:
      storageClass: nfs
      accessMode: ReadWriteOnce
      size: 5Gi
    jobservice:
      storageClass: nfs
      accessMode: ReadWriteOnce
      size: 1Gi
    database:
      storageClass: nfs
      accessMode: ReadWriteOnce
      size: 5Gi
    redis:
      storageClass: nfs
      accessMode: ReadWriteOnce
      size: 1Gi
    trivy:
      storageClass: nfs
      accessMode: ReadWriteOnce
      size: 5Gi
EOF

helm upgrade --install harbor harbor/harbor \
  --namespace harbor --create-namespace \
  -f values.yaml
```

### 확인

```bash
helm list -n harbor
helm status harbor -n harbor
helm get manifest harbor -n harbor
helm get values harbor -n harbor
kubectl get all -n harbor
kubectl get ingress -n harbor
kubectl get certificate -n harbor
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
# 로그가 필요하면 아래처럼 Drop 패킷 로깅 규칙 추가 (한시적)
sudo firewall-cmd --set-log-denied=all   # all, unicast, broadcast, multicast

# Drop된 패킷 확인
sudo journalctl -xef | grep 'REJECT'
# 또는
sudo dmesg -Tw | grep 'REJECT'

# 방화벽 다시 로드
sudo systemctl stop firewalld.service
```

### 🔐 Harbor 초기 설정 가이드

```bash
# Jenkins 설치 후 다음 명령어로 초기 비밀번호 확인
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```

![그림_1](/assets/img/2025-06-15/그림1.png)
_Jenkins 로그인_

![그림_2](/assets/img/2025-06-15/그림2.png)
_Jenkins 대시보드_

## 참고 자료
- [Jenkins 공식 문서](https://www.jenkins.io/doc/book/installing/kubernetes/)

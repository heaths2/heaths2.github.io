---
title: Nginx Proxy Manager 설치
author: G.G
date: 2025-06-22 21:20 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Helm, Nginx Proxy Manager]
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
cat << 'EOF' > /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

sudo dnf update
sudo dnf install nginx nginx-mod-headers-more

sudo dnf install -y podman
sudo dnf install -y python3-pip  # pip3 설치
pip3 install podman-compose      # podman-compose 설치
echo 'export PATH="$PATH:/usr/local/bin"' >> ~/.bashrc

podman --version
podman-compose --version # 설치 확인

# /opt/nginx-proxy-manager/docker-compose.yml
# Podman Compose 환경에서도 동일하게 작동합니다.

version: '3.8'

services:
  app:
    image: 'docker.io/jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager_app
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    volumes:
      - ./data:/data
      - ./data/letsencrypt:/etc/letsencrypt
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"

  db:
    image: 'docker.io/library/mariadb:10.6'
    container_name: nginx-proxy-manager_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 'npm'
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm'
    volumes:
      - ./data/mysql:/var/lib/mysql



https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/


# 터미널 위치: /path/to/your/charts/ (예: /home/user/charts/)
# Chart 경로: /path/to/your/charts/nginx-proxy-manager-chart/
# values.yaml 경로: /path/to/your/charts/nginx-proxy-manager-chart/values.yaml
helm create nginx-proxy-manager-chart
helm upgrade --install nginx-proxy-manager ./nginx-proxy-manager-chart \
  --namespace npm --create-namespace \
  -f ./nginx-proxy-manager-chart/values.yaml


초기 사용자 이름 (Email): admin@example.com
초기 비밀번호 (Password): changeme
```

```bash
cat nginx-proxy-manager-chart/Chart.yaml
# Chart.yaml
apiVersion: v2
name: nginx-proxy-manager-chart
description: A Helm chart for Nginx Proxy Manager
version: 0.1.0 # Chart의 버전
appVersion: "latest" # Nginx Proxy Manager 앱의 버전
```

```bash
# Nginx Proxy Manager를 위한 Helm Chart values.yaml 입니다.
# 'was' 노드 (172.16.0.35)에 Pod를 강제로 배포하도록 설정되었습니다.

app:
  image:
    repository: docker.io/jc21/nginx-proxy-manager
    tag: latest
    pullPolicy: IfNotPresent
  name: nginx-proxy-manager-app
  replicaCount: 1
  ports:
    http: 80
    https: 443
    admin: 81
  persistence:
    enabled: true
    data:
      name: npm-data-volume
      mountPath: /data
      size: 5Gi
      storageClassName: nfs # 스토리지 클래스 이름 확인 필수
    letsencrypt:
      name: npm-letsencrypt-volume
      mountPath: /etc/letsencrypt
      size: 1Gi
      storageClassName: nfs # 스토리지 클래스 이름 확인 필수
  environment:
    DB_PG_HOST: "nginx-proxy-manager-postgresql"
    DB_PG_PORT: "5432"
    DB_PG_USER: "npm"
    DB_PG_PASSWORD: "npmpass" # 중요: 프로덕션에서는 Kubernetes Secret으로 관리하는 것을 강력히 권장합니다.
    DB_PG_DATABASE: "npm"
  resources:
    requests:
      cpu: 50m  # 최소 CPU 요청
      memory: 64Mi # 최소 메모리 요청
    limits:
      cpu: 200m # CPU 제한
      memory: 128Mi # 메모리 제한
  nodeSelector:
    kubernetes.io/hostname: was

postgresql:
  enabled: true
  name: nginx-proxy-manager-postgresql
  image:
    repository: docker.io/library/postgres
    tag: 16
    pullPolicy: IfNotPresent
  environment:
    POSTGRES_USER: "npm"
    POSTGRES_PASSWORD: "npmpass" # 중요: 프로덕션에서는 Kubernetes Secret으로 관리하는 것을 강력히 권장합니다.
    POSTGRES_DB: "npm"
  persistence:
    enabled: true
    data:
      name: npm-postgresql-data-volume
      mountPath: /var/lib/postgresql/data
      size: 10Gi
      storageClassName: nfs # 스토리지 클래스 이름 확인 필수
  resources:
    requests:
      cpu: 100m # 최소 CPU 요청
      memory: 128Mi # 최소 메모리 요청
    limits:
      cpu: 400m # CPU 제한
      memory: 256Mi # 메모리 제한
  nodeSelector:
    kubernetes.io/hostname: was
  service:
    port: 5432
    targetPort: 5432
    type: ClusterIP

service:
  type: ClusterIP
  http:
    port: 80
    targetPort: 80
  https:
    port: 443
    targetPort: 443
  admin:
    port: 81
    targetPort: 81

ingress:
  enabled: true
  className: "nginx"
  host:
    - npm.infra.com # 실제 사용하실 도메인 이름으로 변경해주세요!
  annotations:
    "cert-manager.io/cluster-issuer": "letsencrypt-prod"
      #    "nginx.ingress.kubernetes.io/ssl-redirect": "true" # HTTPS 강제 리다이렉트
  tls:
    enabled: true # Cert-Manager 사용 시 true
    secretName: npm-tls-secret # Cert-Manager가 생성할 TLS Secret 이름
```

```bash
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" . }}-app
  labels:
    {{- include "nginx-proxy-manager-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.app.replicaCount }}
  selector:
    matchLabels:
      {{- include "nginx-proxy-manager-chart.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: app
  template:
    metadata:
      labels:
        {{- include "nginx-proxy-manager-chart.labels" . | nindent 8 }}
        app.kubernetes.io/component: app
    spec:
      {{- if .Values.app.nodeSelector }}
      nodeSelector:
        {{- toYaml .Values.app.nodeSelector | nindent 8 }}
      {{- end }}
      {{- if .Values.app.tolerations }}
      tolerations:
        {{- toYaml .Values.app.tolerations | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Values.app.name }}
          image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
          imagePullPolicy: {{ .Values.app.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.app.ports.http }}
            - name: https
              containerPort: {{ .Values.app.ports.https }}
            - name: admin
              containerPort: {{ .Values.app.ports.admin }}
          env:
            {{- range $key, $value := .Values.app.environment }}
            - name: {{ $key }}
              value: "{{ $value }}"
            {{- end }}
          volumeMounts:
            {{- if .Values.app.persistence.enabled }}
            - name: {{ .Values.app.persistence.data.name }}
              mountPath: {{ .Values.app.persistence.data.mountPath }}
            - name: {{ .Values.app.persistence.letsencrypt.name }}
              mountPath: {{ .Values.app.persistence.letsencrypt.mountPath }}
            {{- end }}
          resources:
            {{- toYaml .Values.app.resources | nindent 12 }}
      volumes:
        {{- if .Values.app.persistence.enabled }}
        - name: {{ .Values.app.persistence.data.name }}
          persistentVolumeClaim:
            claimName: {{ include "nginx-proxy-manager-chart.fullname" . }}-{{ .Values.app.persistence.data.name }}
        - name: {{ .Values.app.persistence.letsencrypt.name }}
          persistentVolumeClaim:
            claimName: {{ include "nginx-proxy-manager-chart.fullname" . }}-{{ .Values.app.persistence.letsencrypt.name }}
        {{- end }}
```

```bash
# templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "nginx-proxy-manager-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If release name contains chart name it will be used as a full name.
*/}}
{{- define "nginx-proxy-manager-chart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}

{{/*
Common labels
*/}}
{{- define "nginx-proxy-manager-chart.labels" -}}
helm.sh/chart: {{ include "nginx-proxy-manager-chart.name" . }}-{{ .Chart.Version }}
{{ include "nginx-proxy-manager-chart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{/*
Selector labels
*/}}
{{- define "nginx-proxy-manager-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "nginx-proxy-manager-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

```bash
# nginx-proxy-manager-chart/templates/ingress.yaml
{{- $root := . }}
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" $root }} # <-- 여기를 . 대신 $root 로 변경
  labels:
    {{- include "nginx-proxy-manager-chart.labels" $root | nindent 4 }} # <-- 여기도 . 대신 $root 로 변경
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  rules:
    {{- range $root.Values.ingress.host }}
    - host: {{ . }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "nginx-proxy-manager-chart.fullname" $root }}
                port:
                  number: {{ $root.Values.service.http.targetPort }}
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: {{ include "nginx-proxy-manager-chart.fullname" $root }}
                port:
                  number: {{ $root.Values.service.admin.targetPort }}
    {{- end }}
  {{- if $root.Values.ingress.tls.enabled }}
  tls:
    - hosts:
        {{- range $root.Values.ingress.host }}
        - {{ . }}
        {{- end }}
      secretName: {{ $root.Values.ingress.tls.secretName }}
  {{- end }}
{{- end }}
```

```bash
# templates/postgresql-deployment.yaml
{{- if .Values.postgresql.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" . }}-postgresql
  labels:
    {{- include "nginx-proxy-manager-chart.labels" . | nindent 4 }}
    app.kubernetes.io/component: database
spec:
  replicas: 1 # 데이터베이스는 보통 단일 복제본으로 운영됩니다.
  selector:
    matchLabels:
      {{- include "nginx-proxy-manager-chart.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: database
  template:
    metadata:
      labels:
        {{- include "nginx-proxy-manager-chart.labels" . | nindent 8 }}
        app.kubernetes.io/component: database
    spec:
      {{- if .Values.postgresql.nodeSelector }}
      nodeSelector:
        {{- toYaml .Values.postgresql.nodeSelector | nindent 8 }}
      {{- end }}
      {{- if .Values.postgresql.tolerations }}
      tolerations:
        {{- toYaml .Values.postgresql.tolerations | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Values.postgresql.name }}
          image: "{{ .Values.postgresql.image.repository }}:{{ .Values.postgresql.image.tag }}"
          imagePullPolicy: {{ .Values.postgresql.image.pullPolicy }}
          env:
            - name: POSTGRES_USER
              value: "{{ .Values.postgresql.environment.POSTGRES_USER }}"
            - name: POSTGRES_PASSWORD
              value: "{{ .Values.postgresql.environment.POSTGRES_PASSWORD }}"
            - name: POSTGRES_DB
              value: "{{ .Values.postgresql.environment.POSTGRES_DB }}"
          ports:
            - name: postgresql
              containerPort: 5432
          volumeMounts:
            {{- if .Values.postgresql.persistence.enabled }}
            - name: {{ .Values.postgresql.persistence.data.name }}
              mountPath: {{ .Values.postgresql.persistence.data.mountPath }}
            {{- end }}
          resources:
            {{- toYaml .Values.postgresql.resources | nindent 12 }}
      volumes:
        {{- if .Values.postgresql.persistence.enabled }}
        - name: {{ .Values.postgresql.persistence.data.name }}
          persistentVolumeClaim:
            claimName: {{ include "nginx-proxy-manager-chart.fullname" . }}-{{ .Values.postgresql.persistence.data.name }}
        {{- end }}
{{- end }}
```

```bash
# templates/postgresql-service.yaml
{{- if .Values.postgresql.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" . }}-postgresql # Nginx Proxy Manager 앱에서 이 서비스 이름으로 PostgreSQL에 접근합니다.
  labels:
    {{- include "nginx-proxy-manager-chart.labels" . | nindent 4 }}
    app.kubernetes.io/component: database
spec:
  type: ClusterIP # 데이터베이스는 보통 클러스터 내부에서만 접근하므로 ClusterIP가 적합합니다.
  ports:
    - port: {{ .Values.postgresql.service.port }}
      targetPort: {{ .Values.postgresql.service.targetPort }}
      protocol: TCP
      name: postgresql
  selector:
    {{- include "nginx-proxy-manager-chart.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: database
{{- end }}
```

```bash
# templates/pvc-app.yaml
{{- if .Values.app.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" . }}-{{ .Values.app.persistence.data.name }}
  labels:
    {{- include "nginx-proxy-manager-chart.labels" . | nindent 4 }}
spec:
  accessModes:
    - ReadWriteOnce # 대부분의 경우 ReadWriteOnce (RWO)를 사용합니다.
  resources:
    requests:
      storage: {{ .Values.app.persistence.data.size }}
  storageClassName: {{ .Values.app.persistence.data.storageClassName }}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" . }}-{{ .Values.app.persistence.letsencrypt.name }}
  labels:
    {{- include "nginx-proxy-manager-chart.labels" . | nindent 4 }}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.app.persistence.letsencrypt.size }}
  storageClassName: {{ .Values.app.persistence.letsencrypt.storageClassName }}
{{- end }}
```

```bash
# templates/pvc-postgresql.yaml
{{- if .Values.postgresql.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" . }}-{{ .Values.postgresql.persistence.data.name }}
  labels:
    {{- include "nginx-proxy-manager-chart.labels" . | nindent 4 }}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.postgresql.persistence.data.size }}
  storageClassName: {{ .Values.postgresql.persistence.data.storageClassName }}
{{- end }}
```

```bash
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "nginx-proxy-manager-chart.fullname" . }}-app
  labels:
    {{- include "nginx-proxy-manager-chart.labels" . | nindent 4 }}
    app.kubernetes.io/component: app
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.http.port }}
      targetPort: {{ .Values.app.ports.http }} # ContainerPort를 참조
      protocol: TCP
      name: http
      {{- if eq .Values.service.type "NodePort" }}
      nodePort: {{ .Values.service.http.nodePort }}
      {{- end }}
    - port: {{ .Values.service.https.port }}
      targetPort: {{ .Values.app.ports.https }} # ContainerPort를 참조
      protocol: TCP
      name: https
      {{- if eq .Values.service.type "NodePort" }}
      nodePort: {{ .Values.service.https.nodePort }}
      {{- end }}
    - port: {{ .Values.service.admin.port }}
      targetPort: {{ .Values.app.ports.admin }} # ContainerPort를 참조
      protocol: TCP
      name: admin
      {{- if eq .Values.service.type "NodePort" }}
      nodePort: {{ .Values.service.admin.nodePort }}
      {{- end }}
  selector:
    {{- include "nginx-proxy-manager-chart.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: app
```

![그림_1](/assets/img/2025-06-15/그림1.png)
_Jenkins 로그인_

![그림_2](/assets/img/2025-06-15/그림2.png)
_Jenkins 대시보드_

## 참고 자료
- [Nginx 공식 문서](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/)

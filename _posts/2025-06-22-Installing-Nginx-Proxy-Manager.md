---
title: Nginx Proxy Manager 설치
author: G.G
date: 2025-06-22 21:20 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Helm, Nginx Proxy Manager]
---

## 📘 개요 (Overview)
이 매뉴얼은 Kubernetes 환경에서 **Nginx Proxy Manager(NPM)**를 Helm Chart로 배포하는 과정을 정리한 것입니다. MariaDB를 백엔드 DB로 사용하며, NFS를 활용한 영구 저장소와 cert-manager를 통한 SSL 인증서 자동 발급(자체 서명, Let's Encrypt 등)을 포함합니다.

## 🧭 등장배경
- 컨테이너 환경의 프록시 관리 자동화와 웹 UI 기반 설정의 필요성이 증가
- 수동 Docker Compose에서 Helm Chart 기반 인프라 표준화/자동화로 전환 추세
- NFS 기반 스토리지, cert-manager 등 최신 K8s 구성요소와의 연동 사례가 많아지면서 실제 적용 사례 매뉴얼 요구 증가

## 📝 주요 특징 및 구성요소 (Components)

- Helm Chart 템플릿 구조(Chart.yaml/values.yaml/템플릿)
- DB 분리(MariaDB), Secret 기반 환경변수, PVC 기반 영구 저장소
- Ingress+cert-manager를 통한 SSL 자동화, NFS StorageClass 활용
- K8s/DevOps 표준에 맞춘 Best Practice 적용

## 📁 파일 구조

```bash
nginx-proxy-manager/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   ├── db-secret.yaml
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── mariadb-deployment.yaml
│   ├── mariadb-pvc.yaml
│   ├── mariadb-service.yaml
│   ├── pvc.yaml
│   ├── service.yaml
│   └── selfsigned-clusterissuer.yaml
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

### 📌 확인

```bash
helm repo list
helm list -A
kubectl get all --all-namespaces
```

```bash
# 📌 NPM Chart 템플릿 생성
helm create nginx-proxy-manager
rm -rf nginx-proxy-manager/templates/{hpa.yaml,serviceaccount.yaml,tests/*}
```

### 🧩 NPM Helm 설치 (Ingress + NFS + ClusterIP 구성)

#### A. MariaDB

```bash
# 📌 Chart.yaml
cat << 'EOF' > nginx-proxy-manager/Chart.yaml
apiVersion: v2
name: nginx-proxy-manager
description: A Helm chart for deploying Nginx Proxy Manager with MariaDB backend
version: 1.0.0
appVersion: "2.10.0"
EOF
```

```bash
# 📌 values.yaml
cat << 'EOF' > nginx-proxy-manager/values.yaml
app:
  image:
    repository: docker.io/jc21/nginx-proxy-manager
    tag: latest
    pullPolicy: IfNotPresent
  replicaCount: 1
  service:
    type: ClusterIP
    ports:
      http: 80
      https: 443
      admin: 81
  ingress:
    enabled: true
    className: nginx
    annotations:
      cert-manager.io/cluster-issuer: "selfsigned-cluster-issuer"
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
    hosts:
      - host: npm.infra.com
        paths:
          - path: /
            pathType: Prefix
          - path: /admin
            pathType: Prefix
    tls:
      - hosts:
          - npm.infra.com
        secretName: npm-infra-com-tls
  persistence:
    enabled: true
    storageClass: nfs
    accessModes:
      - ReadWriteMany
    size: 10Gi
    dataSubPath: "data"
    letsencryptSubPath: "letsencrypt"
db:
  enabled: true
  type: mariadb
  image:
    repository: docker.io/library/mariadb
    tag: "10.6"
    pullPolicy: IfNotPresent
  auth:
    rootPassword: "npm"
    username: "npm"
    password: "npm"
    database: "npm"
  persistence:
    enabled: true
    storageClass: nfs
    accessModes:
      - ReadWriteOnce
    size: 5Gi
certManager:
  clusterIssuer:
    enabled: true
    name: selfsigned-cluster-issuer
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
EOF
```

```bash
# 📌 templates/_helpers.tpl
cat << 'EOF' > nginx-proxy-manager/templates/_helpers.tpl
{{/* Helm helper template for names */}}
{{- define "nginx-proxy-manager.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "nginx-proxy-manager.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- printf "%s" $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
EOF
```

```bash
# 📌 templates/NOTES.txt
cat << 'EOF' > nginx-proxy-manager/templates/NOTES.txt
Thank you for installing {{ include "nginx-proxy-manager.fullname" . }}.

Your release is named {{ .Release.Name }}.

To access the Nginx Proxy Manager admin panel, visit:
{{- if .Values.app.ingress.enabled }}
  https://{{ (index .Values.app.ingress.hosts 0).host }}/admin
{{- else if eq .Values.app.service.type "LoadBalancer" }}
  # LoadBalancer IP는 클러스터 환경에 따라 다르게 할당됩니다.
  # 다음 명령어로 확인 가능: kubectl get svc -n {{ .Release.Namespace }} {{ include "nginx-proxy-manager.fullname" . }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
  http://<LoadBalancer-IP>:81
{{- else if eq .Values.app.service.type "NodePort" }}
  http://<node-ip>:81 # NodePort의 경우 실제 노드 IP를 사용
{{- else }}
  http://{{ include "nginx-proxy-manager.fullname" . }}.{{ .Release.Namespace }}.svc.cluster.local:81
{{- end }}

Initial Login Credentials:
  Email: admin@example.com
  Password: changeme

Remember to change your password after the first login.
EOF
```

```bash
# 📌 templates/db-secret.yaml
cat << EOF > nginx-proxy-manager/templates/db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-proxy-manager-db-secret
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
type: Opaque
data:
  db-password: $(echo -n 'npm' | base64)
EOF
```

```bash
# 📌 templates/deployment.yaml
cat << 'EOF' > nginx-proxy-manager/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-proxy-manager
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx-proxy-manager
      app.kubernetes.io/instance: nginx-proxy-manager
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx-proxy-manager
        app.kubernetes.io/instance: nginx-proxy-manager
    spec:
      containers:
        - name: nginx-proxy-manager
          image: "docker.io/jc21/nginx-proxy-manager:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
            - name: admin
              containerPort: 81
              protocol: TCP
          env:
            - name: DB_MYSQL_HOST
              value: mariadb
            - name: DB_MYSQL_PORT
              value: "3306"
            - name: DB_MYSQL_USER
              value: npm
            - name: DB_MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: nginx-proxy-manager-db-secret
                  key: db-password
            - name: DB_MYSQL_NAME
              value: npm
          volumeMounts:
            - name: npm-data
              mountPath: /data
            - name: npm-letsencrypt
              mountPath: /etc/letsencrypt
      volumes:
        - name: npm-data
          persistentVolumeClaim:
            claimName: nginx-proxy-manager-data
        - name: npm-letsencrypt
          persistentVolumeClaim:
            claimName: nginx-proxy-manager-letsencrypt
EOF
```

```bash
# 📌 templates/ingress.yaml
cat << 'EOF' > nginx-proxy-manager/templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-proxy-manager
  namespace: npm
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-cluster-issuer
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: npm.infra.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-proxy-manager
                port:
                  number: 81
  tls:
    - hosts:
        - npm.infra.com
      secretName: npm-infra-com-tls
EOF
```

```bash
# 📌 templates/mariadb-deployment.yaml
cat << 'EOF' > nginx-proxy-manager/templates/mariadb-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx-proxy-manager
      app.kubernetes.io/instance: nginx-proxy-manager
      app.kubernetes.io/component: mariadb
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx-proxy-manager
        app.kubernetes.io/instance: nginx-proxy-manager
        app.kubernetes.io/component: mariadb
    spec:
      containers:
        - name: mariadb
          image: "docker.io/library/mariadb:10.6"
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: nginx-proxy-manager-db-secret
                  key: db-password
            - name: MYSQL_DATABASE
              value: npm
            - name: MYSQL_USER
              value: npm
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: nginx-proxy-manager-db-secret
                  key: db-password
          ports:
            - name: mariadb
              containerPort: 3306
              protocol: TCP
          volumeMounts:
            - name: mariadb-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mariadb-data
          persistentVolumeClaim:
            claimName: mariadb-data
EOF
```

```bash
# 📌 templates/mariadb-pvc.yaml
cat << 'EOF' > nginx-proxy-manager/templates/mariadb-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-data
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs
EOF
```

```bash
# 📌 templates/mariadb-service.yaml
cat << 'EOF' > nginx-proxy-manager/templates/mariadb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: mariadb
spec:
  type: ClusterIP
  ports:
    - port: 3306
      targetPort: mariadb
      protocol: TCP
      name: mariadb
  selector:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/component: mariadb
EOF
```

```bash
# 📌 templates/pvc.yaml (data + letsencrypt)
cat << 'EOF' > nginx-proxy-manager/templates/pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-proxy-manager-data
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-proxy-manager-letsencrypt
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
EOF
```

```bash
# 📌 templates/service.yaml
cat << 'EOF' > nginx-proxy-manager/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-proxy-manager
  labels:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
    - port: 443
      targetPort: https
      protocol: TCP
      name: https
    - port: 81
      targetPort: admin
      protocol: TCP
      name: admin
  selector:
    app.kubernetes.io/name: nginx-proxy-manager
    app.kubernetes.io/instance: nginx-proxy-manager
EOF
```

```bash
# 📌 templates/selfsigned-clusterissuer.yaml
cat << 'EOF' > nginx-proxy-manager/templates/selfsigned-clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
EOF
```

#### B. PostgreSQL

```bash
# 📌 nginx-proxy-manager 설치
helm upgrade --install nginx-proxy-manager ./nginx-proxy-manager \
  --namespace npm --create-namespace \
  -f ./nginx-proxy-manager/values.yaml
```

### 🔐 Nginx Proxy Manager 초기 접속 정보

```
초기 사용자 이름 (Email): admin@example.com
초기 비밀번호 (Password): changeme
```

### PodMan 설치 방법

```bash
# Podman & podman-compose 패키지 설치
sudo dnf install -y podman podman-compose

# unqualified-search-registries 값 "docker.io"로 변경
sudo sed -i 's/^unqualified-search-registries = .*$/unqualified-search-registries = ["docker.io"]/' /etc/containers/registries.conf

# podman-compose 설치 버전 확인
podman-compose --version

# Nginx Proxy Manager 및 데이터용 디렉토리 생성
mkdir -pv /opt/nginx-proxy-manager
mkdir -pv /data/{letsencrypt,nginx-proxy-manager,pgsql}

# Nginx Proxy Manager용 docker-compose.yml 파일 생성
cat << EOF > /opt/nginx-proxy-manager/docker-compose.yml
# /opt/nginx-proxy-manager/docker-compose.yml
version: '3.8'

services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager_app
    restart: unless-stopped
    ports:
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
    environment:
      DB_POSTGRES_HOST: 'db'
      DB_POSTGRES_PORT: '5432'
      DB_POSTGRES_USER: 'npm'
      DB_POSTGRES_PASSWORD: 'npm'
      DB_POSTGRES_NAME: 'npm'
    volumes:
      - /data/nginx-proxy-manager:/data
      - /data/letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: postgres:latest
    container_name: nginx-proxy-manager_db
    restart: unless-stopped
    environment:
      POSTGRES_USER: 'npm'
      POSTGRES_PASSWORD: 'npm'
      POSTGRES_DB: 'npm'
    volumes:
      - /data/pgsql:/var/lib/postgresql/data
EOF
```


![그림_1](/assets/img/2025-06-15/그림1.png)
_Jenkins 로그인_

![그림_2](/assets/img/2025-06-15/그림2.png)
_Jenkins 대시보드_

## 참고 자료
- [Nginx 공식 문서](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/)

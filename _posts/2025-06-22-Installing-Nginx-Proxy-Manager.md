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

### 📝 PodMan 설치 방법

```bash
# Podman 및 Podman Compose 패키지 설치
sudo dnf install -y podman podman-compose

# 컨테이너 레지스트리 기본값 설정 (docker.io)
sudo sed -i 's/^unqualified-search-registries = .*$/unqualified-search-registries = ["docker.io"]/' /etc/containers/registries.conf

# Podman Compose 설치 버전 확인
podman-compose --version

# Nginx Proxy Manager 및 데이터용 디렉토리 생성
mkdir -pv /opt/nginx-proxy-manager
mkdir -pv /data/npm/{letsencrypt,nginx,pgsql,logrotate.d}

# 데이터 디렉토리들에 개별 컨테이너 파일 컨텍스트 영구 적용 규칙 추가
sudo semanage fcontext -a -t container_file_t "/data/npm(/.*)?"

# 영구 규칙 적용
sudo restorecon -Rv /data

# Nginx Proxy Manager docker-compose 파일 생성
cat << EOF > /opt/nginx-proxy-manager/docker-compose.yml
# /opt/nginx-proxy-manager/docker-compose.yml
version: '3.8'

services:
  # 🌐 Nginx Proxy Manager (웹 프록시 관리)
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager_app
    hostname: nginx-proxy-manager_app
    restart: unless-stopped
    ports:
      # These ports are in format <host-port>:<container-port>
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
      # Add any other Stream port you want to expose
      # - '21:21' # FTP
    environment:
      TZ: 'Asia/Seoul'
      PUID: 0
      PGID: 0
      INITIAL_ADMIN_EMAIL: admin@infra.local
      INITIAL_ADMIN_PASSWORD: mail1234
      # Postgres parameters:
      DB_POSTGRES_HOST: 'db'
      DB_POSTGRES_PORT: '5432'
      DB_POSTGRES_USER: 'npm'
      DB_POSTGRES_PASSWORD: 'npm'
      DB_POSTGRES_NAME: 'npm'
      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'
    volumes:
      - npm_nginx:/data
      - npm_letsencrypt:/etc/letsencrypt
      - /data/npm/logrotate.d/logrotate.custom:/etc/logrotate.d/nginx-proxy-manager
      - /data/profile/.bashrc:/root/.bashrc
    depends_on:
      - db
    deploy:
      resources:
        limits:
          cpus: '0.50'        # 트래픽 처리를 위해 CPU는 조금 넉넉히
          memory: 256M        # 현재 101MB 사용 중 -> 256MB로 제한
        reservations:
          cpus: '0.10'
          memory: 128M

  # 🗄️ PostgreSQL 데이터베이스
  db:
    image: postgres:17
    container_name: nginx-proxy-manager_db
    hostname: nginx-proxy-manager_db
    environment:
      TZ: 'Asia/Seoul'
      PUID: 999
      PGID: 999
      POSTGRES_USER: 'npm'
      POSTGRES_PASSWORD: 'npm'
      POSTGRES_DB: 'npm'
    volumes:
      - npm_pgsql:/var/lib/postgresql
      - /data/profile/.bashrc:/root/.bashrc
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 128M        # 현재 34MB 사용 중 -> 128MB로 제한
        reservations:
          cpus: '0.05'
          memory: 64M

# ✅ Podman 볼륨 정의 (모두 /data/npm/ 하위)
volumes:
  npm_nginx:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/npm/

  npm_letsencrypt:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/npm/letsencrypt

  npm_pgsql:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/npm/pgsql
EOF

# 로그 순환 설정
cat << 'EOF' > /data/npm/logrotate.d/logrotate.custom
# /data/logrotate.d/logrotate.custom
/data/npm/logs/*_access.log /data/npm/logs/*/access.log {
    create 0644
    weekly
    rotate 4
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
    kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}

/data/npm/logs/*_error.log /data/npm/logs/*/error.log {
    create 0644
    weekly
    rotate 10
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
    kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
EOF
```

```bash
# Nginx Proxy Manager 설치 디렉토리로 이동
cd /opt/nginx-proxy-manager

# Podman Compose를 이용한 컨테이너 실행 (백그라운드)
podman compose -f /opt/nginx-proxy-manager/docker-compose.yml up -d
```

![그림_1](/assets/img/2025-06-22/그림1.png)
_NPM 로그인_

![그림_2](/assets/img/2025-06-22/그림2.png)
_NPM 로그인 계정정보 변경_

![그림_3](/assets/img/2025-06-22/그림3.png)
_NPM 로그인 계정 비밀번호 변경_

![그림_3](/assets/img/2025-06-22/그림3.png)
_NPM 로그인 계정 비밀번호 변경_

![그림_4](/assets/img/2025-06-22/그림4.png)
_NPM Proxy 호스트 선택_

![그림_5](/assets/img/2025-06-22/그림5.png)
_NPM Proxy 호스트 추가_

- **UPStream 등록**

```bash
# eth0 인터페이스의 IP 주소만 추출
IP=$(ip -br a show eth0 | awk '{print $3}' | cut -d'/' -f1)

# Upstream 디렉토리 생성
mkdir -pv /data/npm/nginx/custom/

# http.conf 생성
cat << EOF > /data/npm/nginx/custom/http.conf
    upstream npm {
        server ${IP}:81;
    }
    upstream backend {
        server ${IP}:8081;
        server ${IP}:8082;
    }
EOF

# root_top.conf 생성
cat << EOF > /data/npm/nginx/custom/root_top.conf
# GEOIP2 모듈 활성화
load_module /usr/lib/nginx/modules/ngx_http_geoip2_module.so;
load_module /usr/lib/nginx/modules/ngx_stream_geoip2_module.so;
EOF

# 변경된 Nginx 설정을 적용하기 위한 컨테이너 재시작
podman restart nginx-proxy-manager_app
```

- **기본 프록시 설정 확인**

```bash
# 기본 프록시 설정 확인
podman exec -it nginx-proxy-manager_app cat /etc/nginx/conf.d/include/proxy.conf
```

> ```bash
> add_header       X-Served-By $host;
> proxy_set_header Host $host;
> proxy_set_header X-Forwarded-Scheme $scheme;
> proxy_set_header X-Forwarded-Proto  $scheme;
> proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
> proxy_set_header X-Real-IP          $remote_addr;
> proxy_pass       $forward_scheme://$server:$port$request_uri;
> ```

- **NPM 호스트 프록시 설정 확인**

```bash
# NPM 호스트 프록시 설정 확인
podman exec -it nginx-proxy-manager_app cat /data/nginx/proxy_host/1.conf
```

> ```bash
> # ------------------------------------------------------------
> # npm.infra.local
> # ------------------------------------------------------------
> 
> 
> 
> map $scheme $hsts_header {
>     https   "max-age=63072000; preload";
> }
> 
> server {
>   set $forward_scheme http;
>   set $server         "127.0.0.1";
>   set $port           80;
> 
>   listen 80;
> listen [::]:80;
> 
> 
>   server_name npm.infra.local;
> http2 off;
> 
> 
> 
> 
> 
> 
> 
> 
> 
> 
> 
> 
>   access_log /data/logs/proxy-host-1_access.log proxy;
>   error_log /data/logs/proxy-host-1_error.log warn;
> 
> 
> 
> 
> 
> 
> 
>   location / {
> 
> 
> 
> 
> 
> 
> 
> 
>     # Proxy!
>     include conf.d/include/proxy.conf;
>   }
> 
> 
>   # Custom
>   include /data/nginx/custom/server_proxy[.]conf;
> }
> ```

![그림_6](/assets/img/2025-06-22/그림6.png)
_NPM Proxy 호스트 등록_

![그림_7](/assets/img/2025-06-22/그림7.png)
_NPM Proxy Upstrem 설정_


> * NPM 
>
> ```bash
>   access_log /data/logs/npm_access.log;
>   error_log /data/logs/npm_error.log;
> 
>   location / {
>     add_header       X-Served-By $host;
>     proxy_set_header Host $host;
>     proxy_set_header X-Forwarded-Scheme $scheme;
>     proxy_set_header X-Forwarded-Proto  $scheme;
>     proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
>     proxy_set_header X-Real-IP          $remote_addr;
>     proxy_pass       http://npm;
>   }
> ```

> * www 예시 
>
> ```bash
>   access_log /data/logs/www_access.log;
>   error_log /data/logs/www_error.log;
> 
>   location / {
>     add_header       X-Served-By $host;
>     proxy_set_header Host $host;
>     proxy_set_header X-Forwarded-Scheme $scheme;
>     proxy_set_header X-Forwarded-Proto  $scheme;
>     proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
>     proxy_set_header X-Real-IP          $remote_addr;
>     proxy_pass       http://backend;
>   }
> ```

- **NPM 호스트 프록시 설정 변경 확인**

> ```bash
> # ------------------------------------------------------------
> # npm.infra.local
> # ------------------------------------------------------------
> 
> 
> 
> map $scheme $hsts_header {
>     https   "max-age=63072000; preload";
> }
> 
> server {
>   set $forward_scheme http;
>   set $server         "127.0.0.1";
>   set $port           80;
> 
>   listen 80;
> listen [::]:80;
> 
> 
>   server_name npm.infra.local;
> http2 off;
> 
> 
> 
> 
> 
> 
> 
> 
> 
> 
> 
> 
>   access_log /data/logs/proxy-host-1_access.log proxy;
>   error_log /data/logs/proxy-host-1_error.log warn;
> 
>   access_log /data/logs/npm_access.log;
>   error_log /data/logs/npm_error.log;
> 
>   location / {
>     add_header       X-Served-By $host;
>     proxy_set_header Host $host;
>     proxy_set_header X-Forwarded-Scheme $scheme;
>     proxy_set_header X-Forwarded-Proto  $scheme;
>     proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
>     proxy_set_header X-Real-IP          $remote_addr;
>     proxy_pass       http://npm;
>   }
> 
> 
> 
> 
> 
>   # Custom
>   include /data/nginx/custom/server_proxy[.]conf;
> }
> ```

![그림_8](/assets/img/2025-06-22/그림8.png)
_NPM Proxy 호스트 목록 확인_

![그림_9](/assets/img/2025-06-22/그림9.png)
_NPM Proxy Load Balancing 확인_

- **자체 서명(Self-Signed) 인증서** 발급 및 등록

> ```bash
> #!//usr/bin/env bash
> # =============================================
> # Script: Certificate.sh
> # Author: GG
> # Purpose: *.infra.com 용 self-signed wildcard SSL cert 생성
> # Requires: openssl
> # =============================================
> 
> DOMAIN="infra.com"
> CERT_DIR="./certs"
> DAYS_VALID=365
> 
> mkdir -p ${CERT_DIR}
> 
> echo "[+] Generating Private Key..."
> openssl genrsa -out ${CERT_DIR}/${DOMAIN}.key 2048
> 
> echo "[+] Creating CSR with SAN (Subject Alternative Name)..."
> cat > ${CERT_DIR}/${DOMAIN}.cnf <<EOF
> [req]
> default_bits       = 2048
> prompt             = no
> default_md         = sha256
> req_extensions     = req_ext
> distinguished_name = dn
> 
> [dn]
> C=KR
> ST=Seoul
> L=Seoul
> O=Infra
> OU=Dev
> CN=*.${DOMAIN}
> 
> [req_ext]
> subjectAltName = @alt_names
> 
> [alt_names]
> DNS.1 = *.${DOMAIN}
> DNS.2 = ${DOMAIN}
> EOF
> 
> openssl req -new -key ${CERT_DIR}/${DOMAIN}.key \
>     -out ${CERT_DIR}/${DOMAIN}.csr \
>     -config ${CERT_DIR}/${DOMAIN}.cnf
> 
> echo "[+] Generating Self-Signed Certificate..."
> openssl x509 -req -in ${CERT_DIR}/${DOMAIN}.csr \
>     -signkey ${CERT_DIR}/${DOMAIN}.key \
>     -out ${CERT_DIR}/${DOMAIN}.crt \
>     -days ${DAYS_VALID} -extfile ${CERT_DIR}/${DOMAIN}.cnf -extensions req_ext
> 
> echo "[+] Done!"
> echo "Generated files:"
> ls -l ${CERT_DIR}/${DOMAIN}.key ${CERT_DIR}/${DOMAIN}.crt ${CERT_DIR}/${DOMAIN}.csr
> 
> echo "[+] Certificate details:"
> openssl x509 -in ${CERT_DIR}/${DOMAIN}.crt -noout -text | grep "Not"
> # openssl x509 -in certs/infra.com.crt -noout -dates
> ```

![그림_10](/assets/img/2025-06-22/그림10.png)
_NPM Proxy Certificate 등록_

![그림_11](/assets/img/2025-06-22/그림11.png)
_NPM Proxy Certificate 인증서 목록(비활성화)_

![그림_12](/assets/img/2025-06-22/그림12.png)
_NPM Proxy Certificate 인증서 SSL 적용_

![그림_13](/assets/img/2025-06-22/그림13.png)
_NPM Proxy Certificate 인증서 목록(활성화)_

## 참고 자료
- [Nginx 공식 문서](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/)

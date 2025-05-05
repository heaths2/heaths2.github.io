---
title: PowerDNS & PowerDNS Admin Docker 설치
author: G.G
date: 2025-05-04 12:17 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, PowerDNS, PowerDNS-Admin]
---

## 📌 개요
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

## ⚙️ 사용법

### K3s, kubectl, k9s, Helm 설치 

```bash
# K3s 설치 (Ubuntu/Rocky)
curl -sfL https://get.k3s.io | sh -

# kubectl 자동완성 등록
kubectl completion bash >/etc/bash_completion.d/kubectl
source /etc/bash_completion.d/kubectl

# kubectl 설정 파일 복사
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chown $(id -u):$(id -g) ~/.kube/config

# K9s CLI 설치
curl -sS https://webinstall.dev/k9s | bash

# Helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Helm 자동완성 등록
helm completion bash > /etc/bash_completion.d/helm
source /etc/bash_completion.d/helm
```

### PATH, alias 설정

```bash
cat <<EOF | sudo tee -a ~/.bashrc
###############################################
# ✅ 사용자 환경 설정 (PATH 및 alias)
# - 목적: k3s, helm 등 CLI 도구를 정상 인식시키기 위함
# - 대상: 현재 사용자 기준 설정
###############################################

# 시스템 전체 바이너리 경로 추가 (예: k3s, helm, k9s 등)
export PATH="/usr/local/bin:$PATH"

# 사용자 전용 바이너리 경로 추가 (예: Webinstall, pipx 설치 등)
export PATH="$HOME/.local/bin:$PATH"

# k3s에서 기본 제공하는 kubectl을 사용하도록 별칭 설정
alias kubectl='k3s kubectl'
EOF

source ~/.bashrc
kubectl get node
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
kubectl get pods -n pdns
kubectl get svc -n pdns
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

### values.yaml

```yaml
---
# 📁 values.yaml (설정값 중심 관리)

postgresql:
  enabled: true
  image: postgres:16-alpine
  database: pdns
  username: pdns
  password: pdns
  storageSize: 5Gi

  securityContext:
    runAsUser: 5000
    runAsGroup: 5000
    fsGroup: 5000

powerdns:
  enabled: true
  image: pschiffe/pdns-pgsql:latest
  apiKey: changeme
  dbHost: postgresql
  dbUser: pdns
  dbPassword: pdns
  dbName: pdns
  apiAllowFrom: 0.0.0.0/0
  webserver: true

powerdnsAdmin:
  enabled: true
  image: pschiffe/pdns-admin
  db:
    host: postgresql
    port: 5432
    user: pdns
    password: pdns
  api:
    url: http://powerdns:8081
    key: changeme

recursor:
  enabled: true
  image: pschiffe/pdns-recursor:latest

service:
  type: ClusterIP

ingress:
  enabled: false
  hostname: ""
  className: ""

metallb:
  enabled: true
  poolName: pdns-pool
  advertisementName: pdns-l2adv
  addressRange: 172.16.0.240-172.16.0.250
```
{: file='PowerDNS-Admin/values.yaml'}


### deployment-postgresql.yaml

```yaml
---
# 📁 templates/deployment-postgresql.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      securityContext:
        runAsUser: {{ .Values.postgresql.securityContext.runAsUser }}
        runAsGroup: {{ .Values.postgresql.securityContext.runAsGroup }}
        fsGroup: {{ .Values.postgresql.securityContext.fsGroup }}
      initContainers:
        - name: init-permissions
          image: busybox
          command: ["sh", "-c", "chown -R 5000:5000 /var/lib/postgresql/data && chmod -R 755 /var/lib/postgresql/data"]
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data
          securityContext:
            runAsUser: 0
      containers:
        - name: postgresql
          image: {{ .Values.postgresql.image }}
          env:
            - name: POSTGRES_DB
              value: {{ .Values.postgresql.database }}
            - name: POSTGRES_USER
              value: {{ .Values.postgresql.username }}
            - name: POSTGRES_PASSWORD
              value: {{ .Values.postgresql.password }}
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: db-data
          persistentVolumeClaim:
            claimName: pvc-postgresql
```
{: file='PowerDNS-Admin/templates/deployment-postgresql.yaml'}

### deployment-powerdns-admin.yaml

```yaml
---
# 📁 templates/deployment-powerdns-admin.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: powerdns-admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: powerdns-admin
  template:
    metadata:
      labels:
        app: powerdns-admin
    spec:
      containers:
        - name: pdns-admin
          image: {{ .Values.powerdnsAdmin.image }}
          env:
            - name: PDNS_ADMIN_SQLA_DB_TYPE
              value: postgres
            - name: PDNS_ADMIN_SQLA_DB_HOST
              value: {{ .Values.powerdnsAdmin.db.host }}
            - name: PDNS_ADMIN_SQLA_DB_PORT
              value: "5432"
            - name: PDNS_ADMIN_SQLA_DB_USER
              value: {{ .Values.powerdnsAdmin.db.user }}
            - name: PDNS_ADMIN_SQLA_DB_PASSWORD
              value: {{ .Values.powerdnsAdmin.db.password }}
            - name: PDNS_VERSION
              value: "4.9"
            - name: PDNS_API_URL
              value: {{ .Values.powerdnsAdmin.api.url }}
            - name: PDNS_API_KEY
              value: {{ .Values.powerdnsAdmin.api.key }}
          ports:
            - containerPort: 8080
```
{: file='PowerDNS-Admin/templates/deployment-powerdns-admin.yaml'}

### deployment-powerdns.yaml

```yaml
---
# 📁 templates/deployment-powerdns.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: powerdns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: powerdns
  template:
    metadata:
      labels:
        app: powerdns
    spec:
      containers:
        - name: powerdns
          image: {{ .Values.powerdns.image }}
          env:
            - name: PDNS_gpgsql_password
              value: {{ .Values.powerdns.dbPassword }}
            - name: PDNS_gpgsql_user
              value: {{ .Values.powerdns.dbUser }}
            - name: PDNS_gpgsql_dbname
              value: {{ .Values.powerdns.dbName }}
            - name: PDNS_gpgsql_host
              value: {{ .Values.powerdns.dbHost }}
            - name: PDNS_api
              value: "yes"
            - name: PDNS_api_key
              value: {{ .Values.powerdns.apiKey }}
            - name: PDNS_webserver
              value: "yes"
            - name: PDNS_webserver_address
              value: "0.0.0.0"
            - name: PDNS_webserver_allow_from
              value: {{ .Values.powerdns.apiAllowFrom }}
          ports:
            - containerPort: 53
              protocol: TCP
            - containerPort: 53
              protocol: UDP
            - containerPort: 8081
              protocol: TCP
```
{: file='PowerDNS-Admin/templates/deployment-powerdns.yaml'}

### metallb-config.yaml

```yaml
---
# 📁 PowerDNS-Admin/templates/metallb-config.yaml
# Helm Chart에 포함되는 MetalLB 설정 파일입니다.

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: {{ .Values.metallb.poolName }}
  namespace: metallb-system
spec:
  addresses:
    - {{ .Values.metallb.addressRange }}

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: {{ .Values.metallb.advertisementName }}
  namespace: metallb-system
spec:
  ipAddressPools:
    - {{ .Values.metallb.poolName }}
```
{: file='PowerDNS-Admin/templates/metallb-config.yaml'}

### pvc-postgresql.yaml

```yaml
---
# 📁 templates/pvc-postgresql.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-postgresql
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  resources:
    requests:
      storage: {{ .Values.postgresql.storageSize }}
```
{: file='PowerDNS-Admin/templates/pvc-postgresql.yaml'}

### service-postgresql.yaml

```yaml
---
# 📁 templates/service-postgresql.yaml

apiVersion: v1
kind: Service
metadata:
  name: postgresql
spec:
  selector:
    app: postgresql
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  type: ClusterIP
```
{: file='PowerDNS-Admin/templates/service-postgresql.yaml'}

### service-powerdns-admin.yaml

```yaml
---
# 📁 templates/service-powerdns-admin.yaml

apiVersion: v1
kind: Service
metadata:
  name: powerdns-admin
spec:
  selector:
    app: powerdns-admin
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  type: LoadBalancer
  loadBalancerIP: {{ .Values.powerdnsAdmin.serviceIP }}
```
{: file='PowerDNS-Admin/templates/service-powerdns-admin.yaml'}

### service-powerdns.yaml

```yaml
---
# 📁 templates/service-powerdns.yaml

apiVersion: v1
kind: Service
metadata:
  name: powerdns
spec:
  selector:
    app: powerdns
  ports:
    - name: dns-tcp
      port: 53
      targetPort: 53
      protocol: TCP
    - name: dns-udp
      port: 53
      targetPort: 53
      protocol: UDP
    - name: web
      port: 8081
      targetPort: 8081
      protocol: TCP
  type: LoadBalancer
  loadBalancerIP: {{ .Values.powerdns.serviceIP }}
```
{: file='PowerDNS-Admin/templates/service-powerdns.yaml'}

## 참고 자료
- [PowerDNS-Admin Github 문서](https://github.com/PowerDNS-Admin/PowerDNS-Admin)
- [PowerDNS Github 문서](https://github.com/pschiffe/docker-pdns)
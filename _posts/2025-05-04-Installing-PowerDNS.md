---
title: PowerDNS & PowerDNS Admin Docker 설치
author: G.G
date: 2025-05-04 12:17 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Helm, PowerDNS, PowerDNS-Admin]
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
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -
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

### PowerDNS & PowerDNS-Admin Helm Chart 배포

```bash
helm create PowerDNS-Admin
rm -f PowerDNS-Admin/templates/{deployment.yaml,hpa.yaml,serviceaccount.yaml,service.yaml,tests/*}
```

### values.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/values.yaml
---
# 📁 PowerDNS-Admin/values.yaml (설정값 중심 관리)

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
  serviceType: LoadBalancer
  serviceIP: 172.16.0.241

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
  serviceType: ClusterIP

recursor:
  enabled: true
  image: pschiffe/pdns-recursor:latest

metallb:
  enabled: true
  poolName: pdns-pool
  advertisementName: pdns-l2adv
  addressRange: 172.16.0.241-172.16.0.250

ingress:
  enabled: true # Ingress 활성화
  hostname: "pdns.infra.com" # PowerDNS Admin 접속을 위한 호스트 이름 설정
  className: "nginx" # 사용하는 Ingress Controller의 클래스 이름 (예: nginx, traefik 등)
  annotations: {} # 추가 Ingress 어노테이션 (필요시)
EOF
```
{: file='PowerDNS-Admin/values.yaml'}

### deployment-postgresql.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/deployment-postgresql.yaml
---
# 📁 PowerDNS-Admin/templates/deployment-postgresql.yaml

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
EOF
```
{: file='PowerDNS-Admin/templates/deployment-postgresql.yaml'}

### deployment-powerdns-admin.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/deployment-powerdns-admin.yaml
---
# 📁 PowerDNS-Admin/templates/deployment-powerdns-admin.yaml

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
EOF
```
{: file='PowerDNS-Admin/templates/deployment-powerdns-admin.yaml'}

### deployment-powerdns.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/deployment-powerdns.yaml
---
# 📁 PowerDNS-Admin/templates/deployment-powerdns.yaml

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
EOF
```
{: file='PowerDNS-Admin/templates/deployment-powerdns.yaml'}

### metallb-config.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/metallb-config.yaml
---
# 📁 PowerDNS-Admin/templates/metallb-config.yaml
# Helm Chart에 포함되는 MetalLB 설정 파일입니다.

{{- if .Values.metallb.enabled }}
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: {{ .Values.metallb.poolName }}
  namespace: metallb-system # MetalLB가 설치된 네임스페이스
spec:
  addresses:
    - {{ .Values.metallb.addressRange }}

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: {{ .Values.metallb.advertisementName }}
  namespace: metallb-system # MetalLB가 설치된 네임스페이스
spec:
  ipAddressPools:
    - {{ .Values.metallb.poolName }}
{{- end }}
EOF
```
{: file='PowerDNS-Admin/templates/metallb-config.yaml'}

### ingress.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/ingress.yaml
---
# 📁 PowerDNS-Admin/templates/ingress.yaml
# PowerDNS Admin을 위한 Ingress 설정 파일입니다.

{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: powerdns-ingress
  annotations:
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
  labels:
    app.kubernetes.io/name: powerdns-admin
    helm.sh/chart: {{ include "powerdns-admin.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.hostname }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: powerdns-admin
                port:
                  number: 8080
{{- end }}
EOF
```
{: file='PowerDNS-Admin/templates/ingress.yaml'}

### pvc-postgresql.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/pvc-postgresql.yaml
---
# 📁 PowerDNS-Admin/templates/pvc-postgresql.yaml

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
EOF
```
{: file='PowerDNS-Admin/templates/pvc-postgresql.yaml'}

### service-postgresql.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/service-postgresql.yaml
---
# 📁 PowerDNS-Admin/templates/service-postgresql.yaml

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
EOF
```
{: file='PowerDNS-Admin/templates/service-postgresql.yaml'}

### service-powerdns-admin.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/service-powerdns-admin.yaml
---
# 📁 PowerDNS-Admin/templates/service-powerdns-admin.yaml

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
EOF
```
{: file='PowerDNS-Admin/templates/service-powerdns-admin.yaml'}

### service-powerdns.yaml

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/service-powerdns.yaml
---
# 📁 PowerDNS-Admin/templates/service-powerdns.yaml

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
  type: {{ .Values.powerdns.serviceType }} # values.yaml에서 설정한 LoadBalancer 사용
  loadBalancerIP: {{ .Values.powerdns.serviceIP }} # LoadBalancer IP 설정
EOF
```
{: file='PowerDNS-Admin/templates/service-powerdns.yaml'}

### _helpers.tpl

```yaml
cat <<'EOF' | sudo tee PowerDNS-Admin/templates/_helpers.tpl
---
# 📁 PowerDNS-Admin/templates/_helpers.tpl
# Helm Chart에서 공통적으로 사용되는 헬퍼 템플릿들을 정의합니다.

{{/*
Chart 이름과 버전 조합을 반환합니다.
*/}}
{{- define "powerdns-admin.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
전체 이름 접두사를 생성합니다.
*/}}
{{- define "powerdns-admin.fullname" -}}
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
앱 라벨을 생성합니다.
*/}}
{{- define "powerdns-admin.labels" -}}
helm.sh/chart: {{ include "powerdns-admin.chart" . }}
{{ include "powerdns-admin.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{/*
셀렉터 라벨을 생성합니다.
*/}}
{{- define "powerdns-admin.selectorLabels" -}}
app.kubernetes.io/name: {{ include "powerdns-admin.fullname" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
EOF
```
{: file='PowerDNS-Admin/templates/_helpers.tpl'}

### Helm Chart 설치

```bash
helm install powerdns-admin ~/PowerDNS-Admin \
  --namespace pdns \
  --create-namespace \
  --values ~/PowerDNS-Admin/values.yaml
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

### DNS 호스트 적용

```bash
cat <<EOF | sudo tee /etc/resolv.conf
# ======================= DNS 설정 파일 ========================
# 이 설정은 Linux 시스템이 도메인 이름을 IP로 해석할 때
# 어떤 DNS 서버를 우선적으로 사용할지 정의합니다.
# =============================================================

nameserver 172.16.0.51      # ✅ 내부 DNS (PowerDNS Authoritative 서버) - 사내 도메인 이름 해석
nameserver 8.8.8.8          # ✅ 외부 DNS (Google Public DNS) - 외부 도메인 요청 처리
nameserver 210.220.163.82   # ✅ 외부 DNS (SK 브로드밴드 기본 DNS)
nameserver 219.250.36.130   # ✅ 외부 DNS (SK 브로드밴드 보조 DNS)
nameserver 168.126.63.1     # ✅ 외부 DNS (KT 기본 DNS)
nameserver 168.126.63.2     # ✅ 외부 DNS (KT 보조 DNS)
nameserver 164.124.107.9    # ✅ 외부 DNS (LG U+ 기본 DNS)
nameserver 203.248.242.2    # ✅ 외부 DNS (LG U+ 보조 DNS)

search in.infra.com         # ✅ 도메인 자동 완성 - 예: wk01 → wk01.in.infra.com
EOF
```

### DNS 확인

```bash
dig +short +search wk01
dig +short +search wk02
dig +short @172.16.0.51 wk01.in.infra.com
dig +short @172.16.0.51 wk02.in.infra.com
```

## 참고 자료
- [PowerDNS-Admin Github 문서](https://github.com/PowerDNS-Admin/PowerDNS-Admin)
- [PowerDNS Github 문서](https://github.com/pschiffe/docker-pdns)
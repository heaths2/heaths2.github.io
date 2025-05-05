---
title: PowerDNS & PowerDNS Admin Docker 설치
author: G.G
date: 2025-05-04 12:17 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, PowerDNS, PowerDNS-Admin]
---

## 명령어 사용법


## 📁 파일 구조

```text
PowerDNS-Admin
├── charts
├── [Chart.yaml](#chartyaml)
├── templates
│   ├── [deployment-postgresql.yaml](#deployment-postgresqlyaml)
│   ├── [deployment-powerdns-admin.yaml](#deployment-powerdns-adminyaml)
│   ├── [deployment-powerdns.yaml](#deployment-powerdnsyaml)
│   ├── [pvc-postgresql.yaml](#pvc-postgresqlyaml)
│   ├── [service-postgresql.yaml](#service-postgresqlyaml)
│   ├── [service-powerdns-admin.yaml](#service-powerdns-adminyaml)
│   ├── [service-powerdns.yaml](#service-powerdnsyaml)
└── [values.yaml](#valuesyaml)
```

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
```
{: file='PowerDNS-Admin/values.yaml'}

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
```
{: file='PowerDNS-Admin/templates/service-postgresql.yaml'}

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
      nodePort: 30080
      protocol: TCP
  type: NodePort
```
{: file='PowerDNS-Admin/templates/service-powerdns-admin.yaml'}

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
      protocol: TCP
    - name: dns-udp
      port: 53
      protocol: UDP
    - name: web
      port: 8081
      protocol: TCP
  type: ClusterIP
```
{: file='PowerDNS-Admin/templates/service-powerdns.yaml'}

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

## 참고 자료
- [공식 문서서](https://python-poetry.org/docs/#installing-with-the-official-installer)
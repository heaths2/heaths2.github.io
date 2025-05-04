---
title: PowerDNS & PowerDNS Admin Docker 설치
author: G.G
date: 2025-05-04 12:17 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, PowerDNS, PowerDNS-Admin]
---

## 명령어 사용법

```yaml
postgresql:
  image: postgres:16-alpine
  database: pdns
  username: pdns
  password: pdns
  storageClass: nfs
  storageSize: 5Gi
  accessMode: ReadWriteOnce

  securityContext:
    runAsUser: 5000
    runAsGroup: 5000
    fsGroup: 5000
    readOnlyRootFilesystem: false
    runAsNonRoot: true

  initVolumePermissions:
    enabled: true
    image: busybox
    command: "chown -R 5000:5000 /data && chmod -R 755 /data"
    mountPath: /data
    securityContext:
      runAsUser: 0
      readOnlyRootFilesystem: false

  service:
    type: ClusterIP

ingress:
  enabled: false
```
{: file='PowerDNS-Admin/values.yaml'}

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-postgresql
spec:
  accessModes:
    - {{ .Values.postgresql.accessMode | default "ReadWriteOnce" }}
  storageClassName: {{ .Values.postgresql.storageClass }}
  resources:
    requests:
      storage: {{ .Values.postgresql.storageSize }}
```
{: file='PowerDNS-Admin/templates/pvc-postgresql.yaml'}

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  labels:
    app: postgresql
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
        fsGroup: {{ .Values.postgresql.securityContext.fsGroup }}

      {{- if .Values.postgresql.initVolumePermissions.enabled }}
      initContainers:
        - name: fix-permissions
          image: {{ .Values.postgresql.initVolumePermissions.image }}
          command: ["sh", "-c", "{{ .Values.postgresql.initVolumePermissions.command }}"]
          volumeMounts:
            - name: db-storage
              mountPath: {{ .Values.postgresql.initVolumePermissions.mountPath }}
          securityContext:
            runAsUser: {{ .Values.postgresql.initVolumePermissions.securityContext.runAsUser | default 0 }}
            readOnlyRootFilesystem: {{ .Values.postgresql.initVolumePermissions.securityContext.readOnlyRootFilesystem | default false }}
      {{- end }}

      containers:
        - name: postgres
          image: {{ .Values.postgresql.image }}
          env:
            - name: POSTGRES_DB
              value: {{ .Values.postgresql.database }}
            - name: POSTGRES_USER
              value: {{ .Values.postgresql.username }}
            - name: POSTGRES_PASSWORD
              value: {{ .Values.postgresql.password }}
          ports:
            - containerPort: 5432
          securityContext:
            runAsUser: {{ .Values.postgresql.securityContext.runAsUser }}
            runAsGroup: {{ .Values.postgresql.securityContext.runAsGroup }}
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: {{ .Values.postgresql.securityContext.readOnlyRootFilesystem }}
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data

      volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: pvc-postgresql
```
{: file='PowerDNS-Admin/templates/deployment-postgresql.yaml'}

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  labels:
    app: postgresql
spec:
  type: {{ .Values.postgresql.service.type }}
  selector:
    app: postgresql
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
      protocol: TCP
```
{: file='PowerDNS-Admin/templates/service-postgresql.yaml'}

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postgresql-init-schema
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: schema-loader
          image: {{ .Values.postgresql.image }}
          command: ["sh", "-c", "psql -h postgresql -U {{ .Values.postgresql.username }} -d {{ .Values.postgresql.database }} -f /data/db/schema.pgsql.sql"]
          env:
            - name: PGPASSWORD
              value: {{ .Values.postgresql.password }}
          volumeMounts:
            - name: schema-volume
              mountPath: /data
      restartPolicy: OnFailure
      volumes:
        - name: schema-volume
          hostPath:
            path: /data
            type: Directory
```
{: file='PowerDNS-Admin/templates/job-schema.yaml'}

## 참고 자료
- [공식 문서서](https://python-poetry.org/docs/#installing-with-the-official-installer)
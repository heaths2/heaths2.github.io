---
title: Ansible AWX 외부 데이터베이스 연동하는 방법
author: G.G
date: 2025-01-10 16:01:00 +0900
categories: [Blog, Orchestration]
tags: [ansible, awx, postgresql]
---

> 관련글 :
> [ Kubespray 설치방법 (1)](https://heaths2.github.io/posts/kubespray_install/)
> [ Kubespray 설치방법 (1)](https://heaths2.github.io/posts/kubespray_install/)

## 개요
AWX Operator를 설치할 때 기본적으로 제공되는 PostgreSQL 데이터베이스를 사용하지 않고, 별도로 구축된 외부 데이터베이스(NFS 스토리지를 사용하는 PostgreSQL)를 활용합니다. 이를 통해 AWX의 데이터 관리를 더욱 효율적으로 하고, 외부 데이터베이스와의 통합 환경을 구성합니다.

## 설치

### PostgreSQL 구성 시크릿 생성

```bash
tee ./awx-pgsql-configuration.yaml << EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: awx-pgsql-configuration
  namespace: awx
stringData:
  host: "10.1.81.249"
  port: "5432"
  database: "awx"
  username: "awx"
  password: "awx"
  sslmode: prefer
  type: unmanaged
type: Opaque
EOF
```

### PersistentVolumeClaim(PVC) 설정

```bash
tee ./awx-pgsql-pvc.yaml << EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pgsql-15-awx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: local-path
EOF
```

### AWX Operator 배포 설정

```bash
tee ./awx.yaml << EOF
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
  nodeport_port: 30080
  projects_persistence: true
  projects_storage_access_mode: ReadWriteOnce
  postgres_configuration_secret: awx-pgsql-configuration
  web_extra_volume_mounts: |
    - name: external-volume
      mountPath: /var/lib/projects
  extra_volumes: |
    - name: external-volume
      persistentVolumeClaim:
        claimName: pgsql-15-awx-pvc
EOF
```

### Namespace 생성

```bash
kubectl create namespace awx
```

### 리소스 배포

```bash
kubectl apply -f awx-pgsql-pvc.yaml
kubectl apply -f awx-pgsql-configuration.yaml
kubectl apply -k .
```

## 참조
- [Ansible AWX Operator 공식 문서](https://ansible.readthedocs.io/projects/awx-operator/en/latest/user-guide/advanced-configuration/custom-volume-and-volume-mount-options.html#custom-awx-configuration)

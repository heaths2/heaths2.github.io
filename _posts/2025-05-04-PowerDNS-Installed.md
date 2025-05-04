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
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  readOnlyRootFilesystem: true   # 파일시스템 보호
  runAsNonRoot: true             # root 실행 차단

initVolumePermissions:
  enabled: true
  image: busybox
  command: "chown -R 1000:1000 /data && chmod -R 755 /data"
  mountPath: /data

service:
  type: NodePort

ingress:
  enabled: false
```
{: file='PowerDNS-Admin/values.yaml'}

## 참고 자료
- [공식 문서서](https://python-poetry.org/docs/#installing-with-the-official-installer)
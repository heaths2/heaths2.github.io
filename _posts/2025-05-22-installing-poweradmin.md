---
title: PowerDNS & PowerAdmin Web UI 설치
author: G.G
date: 2025-05-22 17:59 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, PowerDNS, PowerAdmin]
---

## 📘 개요
📘 개요 (Overview)
PowerDNS는 고성능 DNS 서버 소프트웨어로, 데이터베이스와의 통합이 강점입니다. 여기에 PowerAdmin이라는 웹 기반 관리 UI를 연동하면, 복잡한 DNS 설정도 GUI로 쉽게 관리할 수 있습니다.

이 문서는 RHEL 9 기반 서버에서 PostgreSQL과 함께 PowerDNS를 설치하고, PowerAdmin을 연동하여 웹 기반 DNS 레코드 관리 환경을 구축하는 방법을 단계적으로 설명합니다. 이 구성은 내부 네트워크(IDC 또는 사내망) 환경에서 마스터 DNS 서버를 운영하는 데 적합합니다.

## 🧭 등장배경
전통적인 BIND 기반 DNS는 높은 유연성과 표준 준수에도 불구하고, 다음과 같은 한계가 존재합니다.
- 텍스트 기반 구성 파일 관리의 불편함
- 실시간 변경의 어려움 (서비스 재시작 필요)
- GUI 미지원

이에 따라, DB 기반으로 레코드를 저장하고 실시간 변경, 백업/복원 용이성, 웹 UI 관리가 가능한 솔루션의 수요가 증가하였습니다.

PowerDNS는 이러한 요구를 충족하며, 특히 다음과 같은 환경에서 적합합니다.
- 사내 IDC에서 내부 도메인 분리 관리
- 자동화된 DNS 레코드 등록 및 삭제
- 멀티 테넌시 또는 UI 접근이 필요한 보안 네트워크 환경

## 🧩 주요 특징 및 구성 요소

| 구성 요소                       | 설명                                      |
| --------------------------- | --------------------------------------- |
| **PowerDNS (pdns)**         | PostgreSQL 기반 DNS Authoritative 서버      |
| **pdns-backend-postgresql** | PostgreSQL과 연동되는 PowerDNS 백엔드 모듈        |
| **PostgreSQL**              | DNS 레코드 및 설정 정보를 저장하는 RDBMS             |
| **PowerAdmin**              | PowerDNS 관리용 PHP 기반 웹 UI                |
| **nginx + php-fpm**         | PowerAdmin UI 제공을 위한 웹서버 및 PHP 처리기      |
| **SELinux / Firewall**      | 보안을 위한 Linux 보안 설정과 포트 허용 관리            |
| **systemd**                 | PowerDNS, PostgreSQL, php-fpm 등의 서비스 관리 |

- 🔄 DB 백엔드 연동: PostgreSQL 같은 RDBMS를 사용하여 DNS 데이터 저장
- ⚡ 빠른 응답 속도: 고성능 캐시 시스템 내장
- 🔐 보안 기능: DNSSEC 및 다양한 인증 기능 지원
- 🧱 유연한 아키텍처: 권한 DNS(Authoritative)와 재귀 DNS(Recursor)를 독립 구성 가능

## 🏗️ 아키텍처
아래는 PowerDNS + PowerAdmin 설치 구조의 논리적 아키텍처입니다.

```bash
┌────────────┐
│  Browser   │  ⇄  PowerAdmin UI (PHP)
└────┬───────┘
     │ HTTP (80)
┌────▼───────┐
│   Nginx    │
│ + php-fpm  │
└────┬───────┘
     │ PHP 연결
┌────▼───────┐       ┌────────────────────┐
│ PowerAdmin │  ⇄   │     PostgreSQL     │
└────────────┘       │   DNS Zone &      │
                     │     Record DB     │
                     └────────────────────┘
                             ▲
                             │
                     ┌───────┴────────┐
                     │   PowerDNS     │ ← Authoritative DNS Server
                     │ (gpgsql mode)  │
                     └────────────────┘
```

## ⚙️ 사용법

### EPEL 및 기본 패키지 설치

```bash
sudo dnf install epel-release -y
sudo dnf install pdns pdns-backend-postgresql -y
```

### PHP 8.2 환경 구성 (PowerAdmin 호환을 위한 권장 버전)

```bash
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.2 -y
sudo dnf install -y php php-intl php-gettext php-pdo php-fpm php-pgsql
```

### PHP-FPM 설정 (Nginx 연동용)

```bash
sudo sed -i \
  -e 's/^user *= *.*/user = nginx/' \
  -e 's/^group *= *.*/group = nginx/' \
  /etc/php-fpm.d/www.conf

chown -R nginx:nginx /var/lib/php/session
chmod 1733 /var/lib/php/session
```

### PowerDNS 설정 (PostgreSQL 백엔드 연동)

```bash
sudo sed -i 's/^\s*launch=bind/# launch=bind/' /etc/pdns/pdns.conf
cat <<'EOF' | sudo tee -a /etc/pdns/pdns.conf
# DB 백엔드 활성화 (PostgreSQL 기준)
launch=gpgsql

gpgsql-host=127.0.0.1
gpgsql-port=5432
gpgsql-user=pdns
gpgsql-password=pdns
gpgsql-dbname=pdns
EOF
```

### Nginx 설치 및 PowerAdmin용 웹 서버 설정

```bash
sudo dnf install -y nginx

cat <<'EOF' | sudo tee /etc/nginx/conf.d/poweradmin.conf
server {
    listen 80;
    server_name localhost; # Replace with your domain

    root /usr/share/nginx/html; # Path to Poweradmin files
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php-fpm/www.sock;  # RHEL/CentOS path
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_index index.php;
    }

    # Deny access to .htaccess and .htpasswd files for security reasons
    location ~ /\.ht {
        deny all;
    }
}
EOF
```

### PostgreSQL 17 수동 설치 및 초기화

```bash
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-libs-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-server-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-contrib-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-devel-17.5-2PGDG.rhel9.x86_64.rpm

sudo dnf install -y ./postgresql17* --skip-broken
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
sudo systemctl enable postgresql-17 --now
```

### PowerAdmin 다운로드 및 배포

```bash
curl -Lo v3.9.2.zip https://github.com/poweradmin/poweradmin/archive/refs/tags/v3.9.2.zip
unzip v3.9.2.zip
# For Nginx (if using a different directory)
rm -rf /usr/share/nginx/html/*
cp -r poweradmin-3.9.2/* /usr/share/nginx/html/
chown -R nginx:nginx /usr/share/nginx/html/
```

### PostgreSQL 사용자 및 DB 초기 생성

```bash
cat <<'EOF'> ~/pdns.sql
-- PowerDNS PGSQL Create DB File
CREATE USER pdns WITH ENCRYPTED PASSWORD 'pdns';
CREATE DATABASE pdns OWNER pdns;
GRANT ALL PRIVILEGES ON DATABASE pdns TO pdns;
EOF
sudo -u postgres psql < ~/pdns.sql

echo "127.0.0.1:5432:pdns:pdns:pdns" >> ~/.pgpass
chmod 600 ~/.pgpass
```

### PowerDNS용 스키마 생성

```bash
psql -U pdns -h 127.0.0.1 -d pdns < "/usr/share/doc/pdns/schema.pgsql.sql"

psql -U pdns -h 127.0.0.1 -d pdns -c '\dt'
psql -U pdns -h 127.0.0.1 -d pdns -c '\l'
```

### 서비스 시작 및 자동 실행 등록

```bash
systemctl enable pdns php-fpm nginx --now
```

### 방화벽 및 SELinux 보안 설정

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload

sudo setsebool -P httpd_can_network_connect_db 1
sudo restorecon -Rv /usr/share/nginx/html
```

### 설치 마법사 실행

![그림_1](/assets/img/2025-05-23/그림1.png)
_PowerAdmin 설치 마법사 실행_

![그림_2](/assets/img/2025-05-23/그림2.png)
_PowerAdmin 언어 선택_

![그림_3](/assets/img/2025-05-23/그림3.png)
_PowerAdmin 설치 마법사 설명1_

![그림_4](/assets/img/2025-05-23/그림4.png)
_PowerAdmin PostgreSQL 데이터베이스 연결_

![그림_5](/assets/img/2025-05-23/그림5.png)
_PowerAdmin PostgreSQL 데이터베이스 연결_

![그림_6](/assets/img/2025-05-23/그림6.png)
_PowerAdmin 계정 및 네임서버 설정_

### PowerAdmin UI에서 사용하는 DB 권한 부여

![그림_7](/assets/img/2025-05-23/그림7.png)
_PowerAdmin pdns 스키마 권한 설정_

```bash
cat <<'EOF' > ~/pdns-grants.sql
-- PowerDNS: Restricted rights GRANT script for user `pdns`

-- 권한 부여 (기존 사용자에게)
GRANT SELECT, INSERT, DELETE, UPDATE ON supermasters TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON domains TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON domainmetadata TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON cryptokeys TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON records TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON comments TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON perm_items TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON perm_templ TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON perm_templ_items TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON users TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON zones TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON zone_templ TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON zone_templ_records TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON records_zone_templ TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON migrations TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON log_zones TO pdns;
GRANT SELECT, INSERT, DELETE, UPDATE ON log_users TO pdns;

-- 시퀀스 권한 부여
GRANT USAGE, SELECT ON SEQUENCE domains_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE domainmetadata_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE cryptokeys_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE records_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE comments_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE perm_items_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE perm_templ_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE perm_templ_items_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE users_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE zones_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE zone_templ_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE zone_templ_records_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE log_zones_id_seq TO pdns;
GRANT USAGE, SELECT ON SEQUENCE log_users_id_seq TO pdns;
EOF

psql -U pdns -h 127.0.0.1 -d pdns < ~/pdns-grants.sql
```

### 최종 config.inc.php 직접 삽입 (보안 키 포함)

![그림_7](/assets/img/2025-05-23/그림7.png)
_PowerAdmin 설정 내역 저장_

```bash
cat <<EOF > /usr/share/nginx/html/inc/config.inc.php
<?php
\$db_host = 'localhost';
\$db_name = 'pdns';
\$db_user = 'pdns';
\$db_pass = 'pdns';
\$db_type = 'pgsql';

\$session_key = 'aH*N4axD7%q%QvEctt))9FVaztW5VSg^Wf3DsZ636YVKz2';

\$iface_lang = 'en_EN';

\$dns_hostmaster = 'hostmaster.infra.com';
\$dns_ns1 = 'ns1.infra.com';
\$dns_ns2 = 'ns2.infra.com';
EOF
```

### 설치 마법사 디렉토리 삭제

![그림_8](/assets/img/2025-05-23/그림8.png)
_PowerAdmin 설정 완료_

![그림_9](/assets/img/2025-05-23/그림9.png)
_PowerAdmin 설치 마법사 삭제_

```bash
rm -rf /usr/share/nginx/html/install
```

### PowerAdmin Web UI 사용 방법

![그림_10](/assets/img/2025-05-23/그림10.png)
_PowerAdmin Web UI 로그인_

![그림_11](/assets/img/2025-05-23/그림11.png)
_PowerAdmin Web UI Zone 생성 클릭_

![그림_12](/assets/img/2025-05-23/그림12.png)
_PowerAdmin Web UI Zone 생성_

![그림_13](/assets/img/2025-05-23/그림13.png)
_PowerAdmin Web UI Zone 수정 선택_

![그림_14](/assets/img/2025-05-23/그림14.png)
_PowerAdmin Web UI Zone 레코드 추가1_

![그림_15](/assets/img/2025-05-23/그림15.png)
_PowerAdmin Web UI Zone 레코드 추가2_

![그림_16](/assets/img/2025-05-23/그림16.png)
_PowerAdmin Web UI Zone 레코드 목록_

### DNS 호스트 적용

```bash
cat <<EOF | sudo tee /etc/resolv.conf
# ======================= DNS 설정 파일 ========================
# 이 설정은 Linux 시스템이 도메인 이름을 IP로 해석할 때
# 어떤 DNS 서버를 우선적으로 사용할지 정의합니다.
# =============================================================

nameserver 172.16.0.52      # ✅ 내부 DNS (PowerDNS Authoritative 서버) - 사내 도메인 이름 해석
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
dig +short +search ns1
dig +short +search k8s
dig +short @172.16.0.52 ns1.in.infra.com
dig +short @172.16.0.52 k8s.in.infra.com
```

### values.yaml

```yaml
---
# 📁 PowerAdmin/values.yaml

postgresql:
  enabled: true
  image: postgres:16-alpine
  database: pdns
  username: pdns
  password: pdns
  storageSize: 5Gi
  storageClass: nfs
  securityContext:
    runAsUser: 5000
    runAsGroup: 5000
    fsGroup: 5000

powerdns:
  image: alpine:3.19
  dbHost: postgresql
  dbUser: pdns
  dbPassword: pdns
  dbName: pdns
  apiKey: changeme
  apiAllowFrom: 0.0.0.0/0
  webserver: "yes"
  ports:
    dnsTcp: 53
    dnsUdp: 53
    api: 8081

poweradmin:
  enabled: true
  image: php:8.2-fpm-alpine
  nginxImage: nginx:stable
  phpPort: 9000
  mountPath: /usr/share/nginx/html
  nginxConfigName: poweradmin-nginx-config
  service:
    port: 80
  ingress:
    enabled: false
    hostname: poweradmin.infra.com
    className: nginx
    tls: false
  volume:
    storageClass: ""
    size: 1Gi

service:
  type: ClusterIP

ingress:
  enabled: false
  hostname: ""
  className: ""
```
{: file='PowerAdmin/values.yaml'}

### values.yaml

```yaml
---
# 📁 PowerAdmin/values.yaml

postgresql:
  enabled: true
  image: postgres:16-alpine
  database: pdns
  username: pdns
  password: pdns
  storageSize: 5Gi
  storageClass: nfs
  securityContext:
    runAsUser: 5000
    runAsGroup: 5000
    fsGroup: 5000

powerdns:
  image: alpine:3.19
  dbHost: postgresql
  dbUser: pdns
  dbPassword: pdns
  dbName: pdns
  apiKey: changeme
  apiAllowFrom: 0.0.0.0/0
  webserver: "yes"
  ports:
    dnsTcp: 53
    dnsUdp: 53
    api: 8081

poweradmin:
  enabled: true
  image: php:8.2-fpm-alpine
  nginxImage: nginx:stable
  phpPort: 9000
  mountPath: /usr/share/nginx/html
  nginxConfigName: poweradmin-nginx-config
  service:
    port: 80
  ingress:
    enabled: false
    hostname: poweradmin.infra.com
    className: nginx
    tls: false
  volume:
    storageClass: ""
    size: 1Gi

service:
  type: ClusterIP

ingress:
  enabled: false
  hostname: ""
  className: ""
```
{: file='PowerAdmin/values.yaml'}

### deployment-postgresql.yaml

```yaml
---
# 📁 PowerAdmin/templates/deployment-postgresql.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql-16
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
        runAsUser: {{ .Values.postgresql.securityContext.runAsUser }}
        runAsGroup: {{ .Values.postgresql.securityContext.runAsGroup }}
        fsGroup: {{ .Values.postgresql.securityContext.fsGroup }}
      initContainers:
        - name: init-permissions
          image: busybox
          command:
            - sh
            - -c
            - "chown -R 5000:5000 /var/lib/postgresql/data && chmod -R 755 /var/lib/postgresql/data"
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
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data

      volumes:
        - name: db-data
          persistentVolumeClaim:
            claimName: pvc-postgresql
        - name: init-sql
          configMap:
            name: pdns-init-sql
```
{: file='PowerAdmin/templates/deployment-postgresql.yaml'}


### service-postgresql.yaml

```yaml
---
# 📁 PowerAdmin/templates/service-postgresql.yaml

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
{: file='PowerAdmin/templates/service-postgresql.yaml'}

### pvc-postgresql.yaml

```yaml
---
# 📁 PowerAdmin/templates/pvc-postgresql.yaml

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
{: file='PowerAdmin/templates/pvc-postgresql.yaml'}

### configmap-initdb.yaml

```yaml
---
# 📁 PowerAdmin/templates/configmap-initdb.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: pdns-init-sql
  namespace: {{ .Release.Namespace }}
data:
  pdns.sql: |-
{{ .Files.Get "sql/pdns.sql" | indent 4 }}
  schema.pgsql.sql: |-
{{ .Files.Get "sql/schema.pgsql.sql" | indent 4 }}
  pdns-grants.sql: |-
{{ .Files.Get "sql/pdns-grants.sql" | indent 4 }}
```
{: file='PowerAdmin/templates/configmap-initdb.yaml'}

### job-postgresql.yaml

```yaml
---
# 📁 PowerAdmin/templates/job-postgresql.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: postgresql-init
  namespace: pdns
spec:
  template:
    spec:
      containers:
        - name: psql
          image: postgres:16-alpine
          env:
            - name: PGPASSWORD
              value: {{ .Values.postgresql.password }}
          command:
            - sh
            - -c
            - |
              echo "Waiting for PostgreSQL..."
              until pg_isready -h postgresql -U {{ .Values.postgresql.username }}; do sleep 2; done

              echo "Running SQL scripts..."
              psql -h postgresql -U {{ .Values.postgresql.username }} -f /init/pdns.sql &&
              psql -h postgresql -U {{ .Values.postgresql.username }} -d {{ .Values.postgresql.database }} -f /init/schema.pgsql.sql &&
              psql -h postgresql -U {{ .Values.postgresql.username }} -d {{ .Values.postgresql.database }} -f /init/pdns-grants.sql
          volumeMounts:
            - name: init-sql
              mountPath: /init
      restartPolicy: OnFailure
      volumes:
        - name: init-sql
          configMap:
            name: pdns-init-sql
```
{: file='PowerAdmin/templates/job-postgresql.yaml'}

## 참고 자료
- [PowerDNS 공식문서](https://repo.powerdns.com)
- [PowerAdmin Github 문서](https://github.com/poweradmin/poweradmin)
- [Postgresql 공식 저장소](https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64)



```bash
# schema.pgsql.sql
CREATE TABLE domains (
  id                    SERIAL PRIMARY KEY,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  TEXT NOT NULL,
  notified_serial       BIGINT DEFAULT NULL,
  account               VARCHAR(40) DEFAULT NULL,
  options               TEXT DEFAULT NULL,
  catalog               TEXT DEFAULT NULL,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE UNIQUE INDEX name_index ON domains(name);
CREATE INDEX catalog_idx ON domains(catalog);


CREATE TABLE records (
  id                    BIGSERIAL PRIMARY KEY,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(65535) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  disabled              BOOL DEFAULT 'f',
  ordername             VARCHAR(255),
  auth                  BOOL DEFAULT 't',
  CONSTRAINT domain_exists
  FOREIGN KEY(domain_id) REFERENCES domains(id)
  ON DELETE CASCADE,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE INDEX rec_name_index ON records(name);
CREATE INDEX nametype_index ON records(name,type);
CREATE INDEX domain_id ON records(domain_id);
CREATE INDEX recordorder ON records (domain_id, ordername text_pattern_ops);


CREATE TABLE supermasters (
  ip                    INET NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) NOT NULL,
  PRIMARY KEY(ip, nameserver)
);


CREATE TABLE comments (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) DEFAULT NULL,
  comment               VARCHAR(65535) NOT NULL,
  CONSTRAINT domain_exists
  FOREIGN KEY(domain_id) REFERENCES domains(id)
  ON DELETE CASCADE,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE INDEX comments_domain_id_idx ON comments (domain_id);
CREATE INDEX comments_name_type_idx ON comments (name, type);
CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);


CREATE TABLE domainmetadata (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT REFERENCES domains(id) ON DELETE CASCADE,
  kind                  VARCHAR(32),
  content               TEXT
);

CREATE INDEX domainidmetaindex ON domainmetadata(domain_id);


CREATE TABLE cryptokeys (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT REFERENCES domains(id) ON DELETE CASCADE,
  flags                 INT NOT NULL,
  active                BOOL,
  published             BOOL DEFAULT TRUE,
  content               TEXT
);

CREATE INDEX domainidindex ON cryptokeys(domain_id);


CREATE TABLE tsigkeys (
  id                    SERIAL PRIMARY KEY,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```
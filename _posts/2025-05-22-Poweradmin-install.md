---
title: PowerDNS & PowerAdmin Web UI 설치
author: G.G
date: 2025-05-22 17:59 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, PowerDNS, PowerAdmin]
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
sudo dnf install epel-release -y
sudo dnf install pdns pdns-backend-postgresql
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.2 -y
sudo dnf install -y php php-intl php-gettext php-pdo php-fpm php-pgsql

sudo sed -i \
  -e 's/^user *= *.*/user = nginx/' \
  -e 's/^group *= *.*/group = nginx/' \
  /etc/php-fpm.d/www.conf

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

wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-libs-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-server-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-contrib-17.5-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/17/redhat/rhel-9-x86_64/postgresql17-devel-17.5-2PGDG.rhel9.x86_64.rpm

sudo dnf install -y ./postgresql17* --skip-broken
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
sudo systemctl enable postgresql-17 --now

curl -Lo v3.9.2.zip https://github.com/poweradmin/poweradmin/archive/refs/tags/v3.9.2.zip
unzip v3.9.2.zip
# For Nginx (if using a different directory)
rm -rf /usr/share/nginx/html/*
cp -r poweradmin-3.9.2/* /usr/share/nginx/html/
chown -R nginx:nginx /usr/share/nginx/html/
sed -i \
-e "s/^\(\$db_host *= *\).*/\1'localhost';/" \
-e "s/^\(\$db_port *= *\).*/\1'5432';/" \
-e "s/^\(\$db_user *= *\).*/\1'pdns';/" \
-e "s/^\(\$db_pass *= *\).*/\1'pdns';/" \
-e "s/^\(\$db_name *= *\).*/\1'pdns';/" \
-e "s/^\(\$db_type *= *\).*/\1'pgsql';/" \
-e "s/^\(\$dns_hostmaster *= *\).*/\1'hostmaster.infra.com';/" \
-e "s/^\(\$dns_ns1 *= *\).*/\1'ns1.infra.com';/" \
-e "s/^\(\$dns_ns2 *= *\).*/\1'ns2.infra.com';/" \
/usr/share/nginx/html/inc/config-defaults.inc.php
mv -v /usr/share/nginx/html/inc/config-defaults.inc.php /usr/share/nginx/html/inc/config.inc.php
chown nginx:nginx /usr/share/nginx/html/inc/config.inc.php

cat <<'EOF'> ~/pdns.sql
-- PowerDNS PGSQL Create DB File
CREATE USER pdns WITH ENCRYPTED PASSWORD 'pdns';
CREATE DATABASE pdns OWNER pdns;
GRANT ALL PRIVILEGES ON DATABASE pdns TO pdns;
EOF
sudo -u postgres psql < ~/pdns.sql

echo "127.0.0.1:5432:pdns:pdns:pdns" >> ~/.pgpass
chmod 600 ~/.pgpass

psql -U pdns -h 127.0.0.1 -d pdns < "/usr/share/doc/pdns/schema.pgsql.sql"


psql -U pdns -h 127.0.0.1 -d pdns -c '\dt'
psql -U pdns -h 127.0.0.1 -d pdns -c '\l'

systemctl enable pdns php-fpm nginx --now

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

sudo setsebool -P httpd_can_network_connect_db 1
sudo restorecon -Rv /usr/share/nginx/html

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

psql -U pdns -h 127.0.0.1 -d pdns < "~/pdns-grants.sql"
```


---
title: NetBox Community 설치 가이드
author: G.G
date: 2026-05-10 19:30 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, IAPM, Netbox]
---

## 📘 개요 (Overview)

NetBox는 네트워크 인프라의 **단일 진실 공급원(Source of Truth)** 역할을 하는 오픈소스 IPAM/DCIM 도구입니다. IP 대역과 VLAN, 랙과 장비, 케이블 연결, 회선과 가상머신까지 하나의 데이터 모델로 관리합니다.

인프라 정보가 엑셀과 위키에 흩어져 있으면 "이 IP를 누가 쓰고 있나", "이 스위치 포트에 뭐가 물려 있나"에 아무도 답하지 못하게 됩니다. NetBox는 이 정보를 구조화된 모델로 강제하고 REST/GraphQL API로 노출하기 때문에, Ansible이나 모니터링 도구가 인벤토리를 자동으로 끌어다 쓸 수 있습니다.

이 가이드는 `netbox-docker`를 기반으로 Podman 환경에 NetBox Community를 구축하고, DNS·IP 계산기·토폴로지 뷰 플러그인을 포함한 커스텀 이미지를 빌드하는 절차를 다룹니다.

## 📂 디렉토리 구조 (Tree 구조)

```bash
/opt/netbox/
├── src/
├── postgres-data/        # PostgreSQL 데이터
├── redis-data/           # Redis (작업 큐)
├── redis-cache-data/     # Redis (캐시)
├── netbox-media/         # 업로드 파일
├── netbox-reports/       # 리포트 스크립트
├── netbox-scripts/       # 커스텀 스크립트
├── backup/
└── logs/
```

## 🛠️ 설치 방법 (Installation)

- 📁 작업 디렉토리 및 컨테이너 네트워크 준비

```bash
# 📁 작업 디렉토리 생성
APP_NAME="netbox"
BASE_DIR="/opt"
APP_DIR="${BASE_DIR}/${APP_NAME}"
mkdir -pv $APP_DIR
cd $APP_DIR
mkdir -pv "${PWD}"/{src,postgres-data,redis-data,redis-cache-data,netbox-media,netbox-reports,netbox-scripts,backup,logs}

# 🔐 SELinux Context 설정
sudo semanage fcontext -a -t container_file_t "${APP_DIR}(/.*)?"
sudo restorecon -Rv "${APP_DIR}"
sudo semanage fcontext -l | grep "${APP_DIR}"
ls -ldZ "${APP_DIR}"

# DevOps 네트워크 대역 생성
podman network create \
  --driver bridge \
  --subnet 10.90.0.0/24 \
  --gateway 10.90.0.1 \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  --ip-range 10.90.0.64/26 \
  net_devops
```

- 📦 netbox-docker 클론 및 플러그인 이미지 빌드

```bash
# rm -rfv "${APP_DIR}" && \
git clone -b release \
  https://github.com/netbox-community/netbox-docker.git \
  "${APP_DIR}"
 
cd "${APP_DIR}"

cp -v docker-compose.override.yml.example docker-compose.override.yml

# DNS Plugin 요구사항 파일 생성
cat > plugin_requirements.txt <<'EOF'
netbox-plugin-dns
netbox-ipcalculator
netbox-topology-views
EOF

cat >> configuration/plugins.py <<'EOF'

PLUGINS = [
    "netbox_dns",
    "netbox_ipcalculator",
    "netbox_topology_views",
]

PLUGINS_CONFIG = {
    "netbox_dns": {},
    "netbox_ipcalculator": {},
    "netbox_topology_views": {},
}
EOF

cat > Dockerfile-Plugins <<'EOF'
FROM netboxcommunity/netbox:latest

COPY ./plugin_requirements.txt /opt/netbox/
RUN /usr/local/bin/uv pip install -r /opt/netbox/plugin_requirements.txt

# These lines are only required if your plugin has its own static files.
COPY configuration/configuration.py /etc/netbox/config/configuration.py
COPY configuration/plugins.py /etc/netbox/config/plugins.py
RUN DEBUG="true" SECRET_KEY="dummydummydummydummydummydummydummydummydummydummy" \
    /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py collectstatic --no-input
EOF

cat > docker-compose.override.yml <<'EOF'
services:
  netbox:
    image: netbox:latest-plugins
    pull_policy: never
    ports:
      - 8000:8080
    build:
      context: .
      dockerfile: Dockerfile-Plugins
  netbox-worker:
    image: netbox:latest-plugins
    pull_policy: never
EOF

podman compose build --no-cache
podman compose up -d

#POSTGRES_PASSWORD="$(LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom 2>/dev/null | head -c 32)"
#REDIS_PASSWORD="$(LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom 2>/dev/null | head -c 32)"
#SECRET_KEY="$(LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom 2>/dev/null | head -c 64)"
#SUPERUSER_API_TOKEN="$(LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom 2>/dev/null | head -c 40)"
#
## .env 생성
#cat > .env <<EOF
#POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
#REDIS_PASSWORD=${REDIS_PASSWORD}
#SECRET_KEY=${SECRET_KEY}
#SUPERUSER_API_TOKEN=${SUPERUSER_API_TOKEN}
#EOF
```

> 💡 **목적**
> - 플러그인은 `netboxcommunity/netbox` 기본 이미지에 포함되지 않아 **직접 빌드**해야 합니다
> - `pull_policy: never` 를 지정해야 로컬 빌드 이미지가 원격 이미지로 덮어써지지 않습니다
> - `netbox-worker` 도 동일 이미지를 써야 플러그인이 백그라운드 작업에서 동작합니다
{: .prompt-tip }

## ✅ 설치 확인 (Verification)

```bash
cd "${APP_DIR}"

# 컨테이너 상태 — netbox, netbox-worker, postgres, redis 전부 Up 이어야 정상
podman compose ps

# 초기 마이그레이션 진행 상황 (첫 기동은 수 분 소요)
podman compose logs -f netbox | grep -iE "migrat|ready|listening"

# HTTP 응답 확인
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:8000/login/

# 플러그인 적재 확인
podman compose exec netbox /opt/netbox/netbox/manage.py shell -c \
  "from django.conf import settings; print(settings.PLUGINS)"
```

```bash
# 예상 출력
200
['netbox_dns', 'netbox_ipcalculator', 'netbox_topology_views']
```
{: .prompt-info }

- 관리자 계정 생성

```bash
podman compose exec netbox /opt/netbox/netbox/manage.py createsuperuser
```

## 참고 자료

- [NetBox 공식 문서](https://netboxlabs.com/docs/netbox/)
- [netbox-docker — 플러그인 사용법](https://github.com/netbox-community/netbox-docker/wiki/Using-Netbox-Plugins)

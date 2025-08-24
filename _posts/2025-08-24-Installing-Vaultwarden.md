---
title: Vaultwarden 설치 가이드
author: G.G
date: 2025-08-24 21:01 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Vaultwarden, Bitwarden]
---

## 📘 개요 (Overview)
Vaultwarden은 경량화된 오픈 소스 Bitwarden 서버이며, 비밀번호 관리 시스템을 개인 서버에 안전하게 구축할 수 있도록 도와줍니다. 도커(Docker) 또는 포드맨(Podman)과 같은 컨테이너 기술을 활용하여 쉽게 배포하고 관리할 수 있습니다.

## 📌 목적 (Purpose)
- 🧭 등장배경: Bitwarden은 편리하지만, 민감한 비밀번호 데이터를 외부에 저장하는 것에 대한 우려가 존재했습니다. 이에 사용자가 자신의 서버에서 직접 비밀번호 관리 솔루션을 운영하며 데이터 주권을 확보하고자 하는 요구가 커졌습니다.
- 🎯 기대효과: Vaultwarden을 통해 사용자는 개인 서버에 안전하고 독립적인 비밀번호 관리 시스템을 구축할 수 있습니다. 이를 통해 클라우드 서비스에 대한 의존성을 줄이고, 데이터 보안을 강화하며, 모든 기기에서 비밀번호를 편리하게 동기화하고 관리할 수 있습니다.


## 📝 구성 요소 (Components)

| 역할 | 설명 | 특징 |
|---|---|---|
| **Vaultwarden Engine** | Vaultwarden 서버를 실행하는 핵심 컨테이너입니다. | 데이터베이스, 이메일 발송 등 다양한 기능을 포함하고 있으며, **컨테이너화**되어 있어 배포가 간편합니다. |
| **Nginx Proxy Manager (NPM)** | Vaultwarden에 대한 **리버스 프록시** 및 **SSL/HTTPS** 인증을 담당하는 도구입니다. | 하나의 서버에서 여러 웹 서비스를 관리할 수 있게 하며, Let's Encrypt를 이용해 무료로 HTTPS를 적용하여 보안을 강화합니다. |
| **PostgreSQL / SQLite** | Vaultwarden의 비밀번호 데이터를 저장하는 **데이터베이스**입니다. | 기본적으로 SQLite를 사용하지만, 대규모 환경에서는 PostgreSQL을 사용해 안정성과 성능을 높일 수 있습니다. 데이터의 영구 저장을 위해 컨테이너 외부 볼륨에 저장하는 것이 권장됩니다. |
| **Bitwarden Client** | Vaultwarden 서버에 접속하여 비밀번호를 관리하는 **클라이언트 앱**입니다. | 웹 볼트, 브라우저 확장팩, 모바일 앱 등 다양한 형태로 제공되며, Vaultwarden 서버와 완벽하게 호환됩니다. |

## ⚙️ 설지방법

- 패키지 설치

```bash
# Podman & podman-compose 패키지 설치
sudo dnf install -y podman podman-compose

# podman-compose 설치 버전 확인
podman-compose --version
```

- 기존 이미지 저장소 변경

```bash
# unqualified-search-registries 값 "docker.io"로 변경
sudo sed -i 's/^unqualified-search-registries = .*$/unqualified-search-registries = ["docker.io"]/' /etc/containers/registries.conf
```

- 기본 저장소 위치 변경

```bash
#  Vaultwarden Password Manager 및 데이터용 디렉토리 생성
mkdir -pv /opt/vaultwarden
mkdir -pv /data/{letsencrypt,nginx,pgsql,logrotate.d,vaultwarden}

# 데이터 디렉토리들에 개별 컨테이너 파일 컨텍스트 영구 적용 규칙 추가
sudo semanage fcontext -a -t container_file_t "/data/letsencrypt(/.*)?"
sudo semanage fcontext -a -t container_file_t "/data/nginx(/.*)?"
sudo semanage fcontext -a -t container_file_t "/data/pgsql(/.*)?"
sudo semanage fcontext -a -t container_file_t "/data/logrotate.d(/.*)?"
sudo semanage fcontext -a -t container_file_t "/data/vaultwarden(/.*)?"

# 영구 규칙 적용
sudo restorecon -Rv /data
```

```bash
# Vaultwarden Password Manager docker-compose 파일 생성
cat << 'EOF' > /opt/vaultwarden/docker-compose.yml
# /opt/vaultwarden/docker-compose.yml
version: '3.8'

services:
  # Nginx Proxy Manager (웹 프록시 관리)
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager_app
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    environment:
      DB_POSTGRES_HOST: 'pgsql'
      DB_POSTGRES_PORT: '5432'
      DB_POSTGRES_USER: 'npm'
      DB_POSTGRES_PASSWORD: 'npm'
      DB_POSTGRES_NAME: 'npm'
      TZ: 'Asia/Seoul'
    volumes:
      - /data/nginx:/data
      - /data/letsencrypt:/etc/letsencrypt
      - /data/logrotate.d/logrotate.custom:/etc/logrotate.d/nginx-proxy-manager
    healthcheck:
      test: ["CMD", "/usr/bin/check-health"]
      interval: 10s
      timeout: 3s
    depends_on:
      - pgsql

  # Nginx Proxy Manager용 PostgreSQL 데이터베이스
  pgsql:
    image: postgres:latest
    container_name: nginx-proxy-manager_db
    restart: unless-stopped
    environment:
      POSTGRES_USER: 'npm'
      POSTGRES_PASSWORD: 'npm'
      POSTGRES_DB: 'npm'
      TZ: 'Asia/Seoul'
    volumes:
      - /data/pgsql:/var/lib/postgresql/data

  # Vaultwarden (비밀번호 관리)
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: https://pw.infra.com
      ADMIN_TOKEN: "$(openssl rand -base64 32)"
      TZ: "Asia/Seoul"
    volumes:
      - /data/vaultwarden:/data
    expose:
      - "80"
EOF
```

```bash
# Vaultwarden Password Manager 설치 디렉토리로 이동
cd /opt/vaultwarden

# Podman Compose를 이용한 컨테이너 실행 (백그라운드)
podman-compose up -d
```

![그림_1](/assets/img/2025-08-24/그림1.png)
_Vaultwarden Proxy 호스트 등록_

![그림_2](/assets/img/2025-08-24/그림2.png)
_Vaultwarden Proxy 호스트 ssl 등록_

![그림_3](/assets/img/2025-08-24/그림3.png)
_Vaultwarden 계정만들기 선택_

![그림_4](/assets/img/2025-08-24/그림4.png)
_Vaultwarden 계정 입력_

![그림_5](/assets/img/2025-08-24/그림5.png)
_Vaultwarden 계정 비밀번호 입력 및 생성_

![그림_6](/assets/img/2025-08-24/그림6.png)
_Vaultwarden 비밀번호 관리자 로그인 화면_

![그림_7](/assets/img/2025-08-24/그림7.png)
_Vaultwarden Bitwarden 비밀번호 관리자 확장 프로그램 설치_

![그림_8](/assets/img/2025-08-24/그림8.png)
_Vaultwarden Bitwarden 비밀번호 관리자 확장 프로그램 추가_

![그림_9](/assets/img/2025-08-24/그림9.png)
_Vaultwarden Bitwarden 비밀번호 관리자 자체 호스팅 등록 선택_

![그림_10](/assets/img/2025-08-24/그림10.png)
_Vaultwarden Bitwarden 비밀번호 관리자 자체 호스팅 주소 입력_

![그림_11](/assets/img/2025-08-24/그림11.png)
_Vaultwarden Bitwarden 비밀번호 관리자 로그인_

![그림_12](/assets/img/2025-08-24/그림12.png)
_Vaultwarden Bitwarden 비밀번호 관리자 웹사이트 등록_

![그림_13](/assets/img/2025-08-24/그림13.png)
_Vaultwarden Bitwarden 비밀번호 관리자 웹사이트 목록_

![그림_14](/assets/img/2025-08-24/그림14.png)
_Vaultwarden Bitwarden 비밀번호 관리자 웹사이트 접속 테스트-1_

![그림_15](/assets/img/2025-08-24/그림15.png)
_Vaultwarden Bitwarden 비밀번호 관리자 웹사이트 접속 테스트-2_

## 참고 자료
- [Vaultwarden Github 공식 문서](https://github.com/dani-garcia/vaultwarden)

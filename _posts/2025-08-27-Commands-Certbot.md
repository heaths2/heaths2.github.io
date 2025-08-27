---
title: Certbot 사용법
author: G.G
date: 2025-08-27 21:10 +0900
categories: [Blog, Command]
tags: [Command, Certbot]
---

## 📘 개요 (Overview)
Certbot은 Let's Encrypt에서 제공하는 무료 SSL/TLS 인증서 발급 자동화 도구입니다. 웹 서버에 설치하여 도메인 소유권을 자동으로 확인하고, SSL 인증서를 발급 및 갱신해주는 핵심적인 역할을 수행합니다. 이를 통해 웹사이트의 HTTPS 통신을 손쉽게 구현하여 데이터 암호화와 보안을 강화할 수 있습니다.

## 📌 목적 (Purpose)
- 🧭 등장 배경: 현대 웹 환경에서 SSL/TLS 인증서는 웹사이트의 보안과 사용자 신뢰를 위해 필수적입니다. 하지만 유료 인증서를 구매하거나 복잡한 수동 설정을 해야 하는 어려움이 있었습니다. 이에 Let's Encrypt는 무료 인증서를 제공하며, Certbot은 이 과정을 자동화하여 누구나 쉽게 HTTPS를 적용할 수 있도록 만들어졌습니다.

- 🎯 기대 효과: Certbot을 사용하면 웹 서버에 무료로 SSL 인증서를 발급하고 적용할 수 있습니다. 또한, Certbot이 인증서 갱신을 자동으로 처리해주기 때문에 인증서 만료로 인한 서비스 중단 걱정을 줄이고, 전반적인 보안을 강화할 수 있습니다.


## 📝 구성 요소 (Components)

| 역할 | 설명 | 특징 |
|---|---|---|
| **Certbot Client** | Certbot의 핵심 CLI(명령줄 인터페이스) 도구입니다. | SSL 인증서 발급, 갱신, 폐기 등의 작업을 수행하는 메인 프로그램입니다. |
| **인증 플러그인 (Authentication Plugin)** | 도메인 소유권을 확인하는 역할을 담당합니다. | **HTTP-01**, **DNS-01** 등 다양한 챌린지 방식을 지원하며, 각 웹 서버나 DNS 서비스에 맞는 플러그인을 사용합니다. |
| **설치 플러그인 (Installer Plugin)** | 발급받은 인증서를 웹 서버에 적용하는 역할을 합니다. | **Apache**, **Nginx** 등 특정 웹 서버의 설정 파일을 자동으로 수정하여 HTTPS를 활성화합니다. |
| **Let's Encrypt CA (Certificate Authority)** | SSL 인증서를 발급해주는 **인증 기관**입니다. | Certbot이 보낸 도메인 소유권 증명을 확인한 후, 실제 인증서를 발급해주는 신뢰 기관입니다. |

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
cat << EOF > /opt/vaultwarden/docker-compose.yml
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
      DOMAIN: https://vault.infra.local
      ADMIN_TOKEN: "$(openssl rand -base64 32)"
      TZ: "Asia/Seoul"
    volumes:
      - /data/vaultwarden:/data
    expose:
      - "80"
EOF

# 로그 순환 설정
cat << 'EOF' > /data/logrotate.d/logrotate.custom
# /data/logrotate.d/logrotate.custom
/data/logs/*_access.log /data/logs/*/access.log {
    su npm npm
    create 0644
    weekly
    rotate 4
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
    kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}

/data/logs/*_error.log /data/logs/*/error.log {
    su npm npm
    create 0644
    weekly
    rotate 10
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
    kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
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

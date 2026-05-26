---
title: Pi-hole 설치 가이드
author: G.G
date: 2025-09-12 13:22 +0900
categories: [Blog, Provisioning]
tags: [Provisioning, Pi-hole]
---

## 📘 개요 (Overview)
Pi-hole은 네트워크 전체의 광고와 추적기를 차단하는 DNS 싱크홀(Sinkhole) 입니다. 집이나 사무실 네트워크에 설치하면, 모든 기기(PC, 스마트폰, 스마트 TV 등)가 이 Pi-hole을 통해 인터넷에 접속하게 되어 광고가 자동으로 필터링됩니다.

## 📌 목적 (Purpose)
* 🧭 등장 배경: 인터넷 사용량이 급증하면서 웹사이트, 앱, 영상 등에서 다양한 형태의 광고와 추적기가 넘쳐나게 되었습니다. 기존의 브라우저 확장 프로그램은 특정 기기에서만 작동하고, 네트워크 전체의 광고를 막는 데 한계가 있었습니다. 이러한 문제를 해결하고, 네트워크 수준에서 모든 기기의 광고를 한 번에 차단하기 위해 Pi-hole이 등장했습니다.

* 🎯 기대 효과
- 광고 차단: 모든 네트워크 기기에서 웹 광고, 팝업 광고, 동영상 광고 등을 효과적으로 차단하여 쾌적한 웹 서핑 환경을 제공합니다.
- 보안 강화: 악성 도메인, 피싱 사이트, 추적기 등을 차단하여 개인정보 보호 및 보안을 강화할 수 있습니다.
- 성능 향상: 불필요한 광고 콘텐츠를 다운로드하지 않으므로 네트워크 트래픽이 줄어들고, 웹페이지 로딩 속도가 빨라지는 효과를 얻을 수 있습니다.
- 중앙 집중식 관리: 하나의 Pi-hole을 설치하면 네트워크에 연결된 모든 기기를 중앙에서 관리할 수 있어 편리합니다.

## 🧩 주요 구성 요소 (Components)

| 역할 | 설명 | 특징 |
| :--- | :--- | :--- |
| **DNS 서버** | 도메인 이름을 IP 주소로 변환하는 역할을 합니다. Pi-hole은 이 역할을 수행하며, 차단 목록에 있는 도메인은 0.0.0.0(싱크홀)으로 응답합니다. | 네트워크의 모든 기기는 **Pi-hole의 IP 주소**를 DNS 서버로 설정하여 사용합니다. |
| **블랙리스트 (Blacklist)** | 광고, 추적기, 악성 웹사이트의 도메인 목록입니다. | 사용자가 직접 추가하거나, 커뮤니티에서 공유되는 **공개 목록**을 구독하여 자동 업데이트할 수 있습니다. |
| **화이트리스트 (Whitelist)** | 실수로 차단된 정상적인 도메인이나 허용하고 싶은 도메인 목록입니다. | 블랙리스트보다 **우선순위**가 높아서, 블랙리스트에 있더라도 화이트리스트에 있으면 차단되지 않습니다. |
| **대시보드 (Dashboard)** | Pi-hole의 상태, 차단된 광고 수, 네트워크 트래픽 통계 등을 시각적으로 보여주는 웹 인터페이스입니다. | 차단 목록을 관리하고, 시스템 설정을 변경하는 등 **Pi-hole을 편리하게 제어**할 수 있습니다. |
| **`pihole-FTL`** | Pi-hole의 핵심 엔진인 FTL (Faster Than Light) 서비스입니다. | 네트워크 트래픽을 모니터링하고 **데이터베이스에 저장**하며, 대시보드에 필요한 정보를 제공합니다. |

## 🛠️ 설치 방법 (Installation)

- 🛠️ Podman 설치

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

- Portainer 설치

```bash
# 포드맨 소켓이 활성화되었는지 확인
sudo systemctl enable --now podman.socket

# 디렉토리 생성
mkdir -pv /opt/pi-hole
mkdir -pv /data/pi-hole

# 데이터 디렉토리들에 개별 컨테이너 파일 컨텍스트 영구 적용 규칙 추가
sudo semanage fcontext -a -t container_file_t "/data(/.*)?"

# 영구 규칙 적용
sudo restorecon -Rv /data
```

```bash
# docker-compose 파일 생성
cat << "EOF" > /opt/pi-hole/docker-compose.yml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      # DNS Ports
      - "53:53/tcp"
      - "53:53/udp"
      # Default HTTP Port
      - "80:80/tcp"
      # Default HTTPs Port. FTL will generate a self-signed certificate
      - "443:443/tcp"
      # Uncomment the below if using Pi-hole as your DHCP Server
      #- "67:67/udp"
      # Uncomment the line below if you are using Pi-hole as your NTP server
      #- "123:123/udp"
    environment:
      # Set the appropriate timezone for your location from
      # https://en.wikipedia.org/wiki/List_of_tz_database_time_zones, e.g:
      TZ: 'Asia/Seoul'
      # Set a password to access the web interface. Not setting one will result in a random password being assigned
      FTLCONF_webserver_api_password: 'correct horse battery staple'
      # If using Docker's default `bridge` network setting the dns listening mode should be set to 'all'
      FTLCONF_dns_listeningMode: 'all'
    # Volumes store your data between container upgrades
    volumes:
      # For persisting Pi-hole's databases and common configuration file
      - '/data/pi-hole/etc-pihole:/etc/pihole'
      - '/data/profile/.bashrc:/root/.bashrc'  # bashrc 파일 매핑
      # Uncomment the below if you have custom dnsmasq config files that you want to persist. Not needed for most starting fresh with Pi-hole v6. If you're upgrading from v5 you and have used this directory before, you should keep it enabled for the first v6 container start to allow for a complete migration. It can be removed afterwards. Needs environment variable FTLCONF_misc_etc_dnsmasq_d: 'true'
      #- './etc-dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      # See https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
      # Required if you are using Pi-hole as your DHCP server, else not needed
      - NET_ADMIN
      # Required if you are using Pi-hole as your NTP client to be able to set the host's system time
      - SYS_TIME
      # Optional, if Pi-hole should get some more processing time
      - SYS_NICE
    restart: unless-stopped
EOF
```

```bash
# 설치 디렉토리로 이동
cd /opt/pi-hole

# Podman Compose를 이용한 컨테이너 실행 (백그라운드)
podman-compose up -d

# 🛠️ 컨테이너 systemd 서비스 파일 생성
podman generate systemd --name pi-hole --files --new

# 📂 생성된 서비스 파일 시스템에 등록
cp -v container-* /usr/lib/systemd/system/

# 🚀 서비스 활성화 및 즉시 시작
systemctl enable --now container-pi-hole
```

![그림_1](/assets/img/2025-09-06/그림1.png)
_Portainer 계정 생성_

## 참고 자료
- [Pi-hole 공식 문서](https://docs.pi-hole.net/docker)
